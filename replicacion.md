# Replicación, multi-región y conflictos a fondo

**Copias de los datos que mienten un rato: lag, multi-región, LWW, CRDTs y por qué evitás 2PC · Node + Go · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es el **M2** del roadmap profundo de system design (el M1 es [Consenso y coordinación distribuida](consenso.md)): se enfoca en qué pasa cuando tenés **varias copias** del mismo dato — cómo se desincronizan, qué anomalías de lectura producen, cómo resolver conflictos (LWW vs vector clocks vs CRDTs) y por qué las transacciones cross-servicio (2PC) son una trampa. Su complemento es el [laboratorio práctico](replicacion-practica.md): acá entendés, allá **construís** (replication lag, LWW perdiendo escrituras, CRDTs que convergen y el bloqueo de 2PC).

**Lo que asumimos.** Que tenés el método de [Diseño de sistemas backend](system-design.md) (CAP, sharding, replicación a alto nivel), que pasaste por [Consenso](consenso.md) (quórums, relojes lógicos, **vector clocks**, fencing — los reusamos sin re-explicarlos) y que viste [Event-driven](event-driven.md) (saga, outbox — los referenciamos al hablar de transacciones distribuidas). No asumimos Go: los ejemplos se explican línea por línea.

> ⚠️ **Sobre los lenguajes.** La replicación es **lógica de sistema distribuido**. Los ejemplos van en **Node/TS** (tu stack) y en **Go** donde la concurrencia real (varias réplicas escribiendo a la vez) se ve más clara. Ver el mismo problema en los dos planos es lo que te hace sonar senior.

### El marco que usamos en todo el módulo

Igual que en M1: para cada mecanismo, **cuatro capas** —(1) qué se rompe, (2) por qué a *esta* escala/topología, (3) control de corto plazo (blast radius), (4) rediseño de largo plazo— y, al diseñar, **las tres cosas**: happy path, modelo de falla, recuperación.

> **El ejemplo que fija el tono — "leí mi propia escritura y no estaba".** Respuesta débil: *"hay lag de replicación"*. Respuesta profunda, en cascada: **escribís en el líder → la respuesta vuelve OK → tu siguiente request, por balanceo, cae en un follower que todavía no recibió esa escritura → leés el valor viejo → como usuario, "guardé y desapareció"**. No es un bug de tu código: es la consistencia eventual asomando. Lo arreglás con *read-your-writes* (módulo 5), no rezando para que el lag baje. Volvemos a esto en los módulos 4-5.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (el grueso va al [laboratorio](replicacion-practica.md)).

**Índice de módulos**

*Etapa 1 — Por qué replicar y el mapa*
1. Por qué replicar (y qué cuesta)
2. Las tres topologías: single-leader, multi-leader, leaderless

*Etapa 2 — Replicación con un líder*
3. Síncrona vs asíncrona (y semi-síncrona)
4. Replication lag y sus tres anomalías de lectura
5. Cómo se arreglan las anomalías
6. Failover del líder y sus peligros

*Etapa 3 — Multi-líder y multi-región*
7. Multi-leader: cuándo y el problema de los conflictos
8. Topologías multi-región: active-passive vs active-active (RPO/RTO)
9. Resolución de conflictos: detección, LWW, siblings

*Etapa 4 — CRDTs en serio*
10. CRDTs: convergencia sin coordinación
11. El zoo de CRDTs (y sus límites)

*Etapa 5 — Transacciones distribuidas*
12. Dual-write: por qué cruzar servicios es difícil
13. 2PC / 3PC: el coordinador, el bloqueo
14. Saga + outbox: la alternativa práctica

*Etapa 6 — Consistencia y cierre*
15. El espectro de consistencia (linealizable → causal → eventual) + PACELC
16. Cómo hablar de replicación en una entrevista

Las soluciones de **todos** los ejercicios están al final.

---

## Etapa 1 — Por qué replicar y el mapa

## Módulo 1 — Por qué replicar (y qué cuesta)

**Teoría.** **Replicar** = mantener copias del mismo dato en varias máquinas. Se hace por tres razones, y conviene tenerlas separadas porque cada una empuja a un diseño distinto:

1. **Disponibilidad:** si una réplica se cae, otra responde. Sin réplicas, una máquina caída = datos inaccesibles.
2. **Latencia geográfica:** una réplica cerca del usuario (en su región) responde en 10ms en vez de 200ms cruzando el océano.
3. **Throughput de lectura:** repartís las lecturas entre N réplicas. (Ojo: **no** escala las escrituras si hay un solo líder — todas pasan por él.)

El precio, y es **el** tema del módulo: mantener las copias **consistentes** cuesta. En el instante en que aceptás una escritura en una réplica, las demás están **desactualizadas** hasta que les llega. Tenés dos salidas y no hay una tercera gratis:

- **Esperar** a que todas confirmen antes de responder → consistencia fuerte, pero **latencia alta** y **disponibilidad baja** (si una réplica está lenta/caída, la escritura espera o falla).
- **No esperar** → respondés rápido y disponible, pero las réplicas quedan temporalmente divergentes → **consistencia eventual** y todas las anomalías que vienen.

> La frase mental: **replicar es fácil; mantener las copias de acuerdo es el problema entero.** Todo este módulo es ese trade-off mirado desde distintos ángulos. Es el CAP/PACELC de [system-design](system-design.md) hecho carne: ante partición, ¿consistencia o disponibilidad?; y aun sin partición, ¿latencia o consistencia?

**Ejercicios 1**
1.1 🔁 Nombrá las tres razones para replicar y marcá cuál NO ayuda a escalar escrituras.
1.2 🧠 ¿Por qué replicar para throughput de lectura no sirve para escalar escrituras con un solo líder?
1.3 🧠 Explicá el trade-off fundamental entre "esperar a todas las réplicas" y "no esperar" en términos de latencia y consistencia.

---

## Módulo 2 — Las tres topologías

**Teoría.** Toda la replicación cae en una de tres formas, según **quién puede aceptar escrituras**. Es el mapa mental que organiza el módulo entero:

| Topología | Quién escribe | Conflictos | Ejemplos |
|---|---|---|---|
| **Single-leader** | Un solo nodo (líder); los demás copian | **No hay** (un solo orden de escritura) | Postgres/MySQL replication, MongoDB replica set |
| **Multi-leader** | Varios líderes, c/u acepta escrituras | **Sí** (escrituras concurrentes a la misma clave) | Multi-datacenter, CRDTs colaborativos, clientes offline |
| **Leaderless** | Cualquier réplica (quórum) | **Sí** (resuelve con versiones) | Dynamo, Cassandra, Riak |

La lógica detrás del mapa:

- **Single-leader** es lo más simple y lo más común. El líder define **el** orden de las escrituras, así que **no hay conflictos**: todos los followers aplican la misma secuencia. El precio: el líder es un cuello de escritura y un punto de failover (módulo 6).
- **Multi-leader** y **leaderless** aceptan escrituras en más de un lugar → ganan disponibilidad/latencia de escritura, pero **pagan con conflictos**: dos escrituras a la misma clave que no se vieron entre sí, y alguien tiene que decidir quién gana (módulos 9-11).

> Single-leader = "delegá el orden en un jefe". Leaderless = "no hay jefe, votá con quórums" (es el mundo de [consenso](consenso.md), módulos 15-17: quórum R+W>N, anti-entropía, consistent hashing). Multi-leader = "varios jefes que después tienen que reconciliar". **Casi todo lo que diseñes va a ser single-leader** salvo que tengas una razón fuerte (multi-región con escritura local, colaboración en vivo, offline-first).

**Ejercicios 2**
2.1 🔁 ¿Cuál de las tres topologías NO tiene conflictos de escritura y por qué?
2.2 🧠 ¿Qué ganan multi-leader y leaderless a cambio de tener que resolver conflictos?
2.3 🧠 Te piden el default para un CRUD de backend típico. ¿Qué topología elegís y qué razón te haría cambiar?

---

## Etapa 2 — Replicación con un líder

## Módulo 3 — Síncrona vs asíncrona (y semi-síncrona)

**Teoría.** En single-leader, el líder recibe la escritura y la propaga a los followers. La pregunta de oro: **¿espera la confirmación de los followers antes de responderle al cliente?**

- **Asíncrona:** el líder responde OK apenas escribe **localmente**, y propaga a los followers "en segundo plano". **Rápida** y **disponible** (un follower lento/caído no frena las escrituras), pero si el líder se cae antes de propagar, esas escrituras confirmadas **se pierden** (módulo 6). Es el default de la mayoría.
- **Síncrona:** el líder espera que **el** (o los) follower(s) confirmen antes de responder OK. **Durable** (la escritura sobrevive a la caída del líder), pero **lenta** (pagás el round-trip al follower) y **frágil** (si el follower sync se cae, las escrituras se **bloquean**).

El punto que distingue al que entendió: **síncrona con TODOS los followers es inviable** — cualquier réplica caída frena todas las escrituras. Por eso existe la **semi-síncrona**: **al menos uno** (o `k`) followers confirman de forma síncrona, el resto async. Te garantiza que la escritura está en **dos** nodos (durable ante una caída) sin atarte a que estén **todos** vivos.

| | Async | Sync (1 follower) | Semi-sync (k de N) |
|---|---|---|---|
| Latencia de escritura | Baja | Alta | Media |
| Durabilidad ante caída del líder | **Puede perder** | Segura | Segura (k copias) |
| Disponibilidad de escritura | Alta | Baja (follower caído frena) | Media |

> En Postgres lo configurás con `synchronous_commit` + `synchronous_standby_names` (`ANY 1 (...)` = semi-sync); MySQL tiene `rpl_semi_sync`. La decisión es de negocio: ¿podés perder los últimos N ms de escrituras si se cae el líder? Para un like, sí (async). Para un pago confirmado, no (sync/semi-sync, o directamente consenso).

**Ejercicios 3**
3.1 🔁 ¿Qué arriesga la replicación asíncrona cuando el líder se cae?
3.2 🧠 ¿Por qué la replicación síncrona contra **todos** los followers es inviable en la práctica?
3.3 🧠 ¿Qué garantiza la semi-síncrona que la async no, y a qué costo respecto de la async?

---

## Módulo 4 — Replication lag y sus tres anomalías de lectura

**Teoría.** Con replicación asíncrona, los followers van **atrasados** respecto del líder: ese atraso es el **replication lag** (normalmente ms, pero bajo carga o re-sync puede ser segundos o minutos). Si leés de un follower atrasado, ves el pasado. Eso produce **tres anomalías** clásicas que tenés que saber nombrar (son de *Designing Data-Intensive Applications*):

1. **Read-your-writes (lectura de tu propia escritura):** escribís algo, e inmediatamente lo leés de un follower que aún no lo recibió → **no ves tu propia escritura**. ("Edité mi perfil y aparece el viejo.") Es la del ejemplo de arriba.
2. **Monotonic reads (lecturas monótonas):** hacés dos lecturas seguidas; la primera cae en un follower adelantado (ve el dato nuevo), la segunda en uno más atrasado (ve el viejo) → **el tiempo parece ir para atrás**. ("Vi un comentario y al refrescar desapareció.")
3. **Consistent prefix reads (prefijo consistente):** en datos particionados, ves escrituras **fuera de orden causal** → ves la respuesta de una pregunta **antes** que la pregunta. Pasa cuando distintas particiones replican a distinto ritmo y no hay garantía de orden entre ellas.

> 🔥 **Falla en 4 capas — "guardé y desapareció" (read-your-writes).**
> 1. **Qué se rompe:** el usuario escribe, recibe OK, y su próxima lectura muestra el valor viejo.
> 2. **Por qué a esta escala:** balanceás lecturas entre N followers async; la escritura tardó 50ms en propagar pero la lectura siguiente llegó a los 10ms a otro follower. No pasa en dev (una sola DB), explota en prod con réplicas.
> 3. **Control de corto plazo:** rutear las lecturas **del propio usuario** al líder por una ventana corta tras escribir (sticky-to-leader); o leer del líder lo que el usuario **puede haber editado** (su propio perfil).
> 4. **Largo plazo:** read-your-writes consistente con **timestamp lógico** — el cliente guarda el LSN/posición de su última escritura y exige leer de una réplica al menos tan nueva (módulo 5).

**Ejercicios 4**
4.1 🔁 Nombrá las tres anomalías de lectura del replication lag.
4.2 🧠 Explicá monotonic reads con un ejemplo de UI y por qué "el tiempo va para atrás".
4.3 🧠 ¿Por qué consistent prefix reads es un problema sobre todo en datos **particionados**?

---

## Módulo 5 — Cómo se arreglan las anomalías

**Teoría.** Cada anomalía tiene una técnica concreta; saberlas distingue una respuesta de senior de un "subí réplicas":

- **Read-your-writes:**
  - Leé del **líder** las cosas que el usuario pudo editar (su propio perfil); del follower lo demás.
  - **Sticky routing:** tras una escritura, mandá las lecturas de ese usuario al líder por unos segundos.
  - **Timestamp lógico (la robusta):** el cliente recuerda la posición del log (LSN) de su última escritura; al leer, exige una réplica con `posición ≥ esa` (o lee del líder si ninguna llegó). Lo construís en el [laboratorio](replicacion-practica.md).
- **Monotonic reads:** que cada usuario lea **siempre de la misma réplica** (sticky por hash del user-id), así nunca "salta hacia atrás" a una más vieja. No te da lo último, pero te da **monotonía** (nunca retrocede).
- **Consistent prefix reads:** asegurá que las escrituras causalmente relacionadas vayan a la **misma partición** o lleven metadata causal (vector clocks / dependencias) para reordenar en el lector.

> La idea transversal: **no hace falta consistencia fuerte para arreglar la anomalía concreta que te duele.** Read-your-writes te da garantías sobre *tus* escrituras; monotonic reads, sobre *no retroceder*. Son **garantías de sesión** — más baratas que linealizabilidad global y suficientes para que la UX no se sienta rota. Elegir la garantía mínima que resuelve el síntoma es la jugada senior.

**Ejercicios 5**
5.1 🔁 Dá las tres técnicas para read-your-writes.
5.2 🧠 ¿Cómo arregla monotonic reads el "leer siempre de la misma réplica" y qué NO te garantiza?
5.3 🧠 ¿Por qué "garantías de sesión" (read-your-writes, monotonic) son preferibles a "consistencia fuerte global" cuando solo te duele una anomalía puntual?

---

## Módulo 6 — Failover del líder y sus peligros

**Teoría.** El líder se cae. Hay que **promover** un follower a líder nuevo (failover). Suena simple; es donde se pierden datos y nacen los split-brain. Tres peligros que tenés que nombrar:

1. **Pérdida de escrituras (con async):** el follower promovido puede estar **atrasado** — le faltan las últimas escrituras que el líder viejo confirmó pero no propagó. Esas escrituras **se descartan**. Si esos IDs/datos se usaron en otro sistema (p. ej. el viejo líder ya le mandó el ID a Redis), quedás **inconsistente cross-sistema**.
2. **Split-brain:** el líder viejo no estaba muerto, estaba **lento/particionado**; revive y se cree líder mientras el nuevo también lo es → **dos líderes aceptando escrituras** → divergencia que después no se mergea. (Es el caso de M1; se corta con quórum + fencing — que el storage rechace al líder viejo.)
3. **Timeout mal calibrado:** muy corto → failovers innecesarios por una pausa de GC (falso positivo del [módulo 3 de consenso](consenso.md)); muy largo → caída real degrada el sistema más tiempo.

> 🔥 **Falla en 4 capas — failover que pierde escrituras y rompe otro sistema.**
> 1. **Qué se rompe:** se promueve un follower atrasado; las últimas escrituras confirmadas del líder viejo desaparecen.
> 2. **Por qué a esta escala:** con replicación async y lag de cientos de ms bajo carga, la ventana de escrituras "confirmadas pero no propagadas" no es vacía; y si esos datos ya cruzaron a otro sistema (cache, cola), el descarte deja huérfanos.
> 3. **Control de corto plazo:** promover **el follower más adelantado** (mayor LSN); fencing para que el líder viejo no escriba al volver.
> 4. **Largo plazo:** replicación **semi-síncrona** (la escritura confirmada está en ≥2 nodos, así que el follower promovido la tiene) o **consenso** (Raft: el líder nuevo, por la *election restriction*, ya tiene todos los commits — [consenso módulo 11](consenso.md)). El failover sin pérdida **es** un problema de consenso.

**Ejercicios 6**
6.1 🔁 ¿Por qué un failover con replicación async puede perder escrituras?
6.2 🧠 ¿Por qué conviene promover el follower con mayor LSN, y qué problema sigue abierto respecto del líder viejo?
6.3 🧠 ¿Cómo elimina la pérdida de escrituras la replicación semi-síncrona (o el consenso)? Conectalo con la election restriction de Raft.

---

## Etapa 3 — Multi-líder y multi-región

## Módulo 7 — Multi-leader: cuándo y el problema de los conflictos

**Teoría.** Multi-leader = **más de un nodo acepta escrituras**, y los líderes se replican entre sí (async). Lo usás cuando un solo líder no alcanza:

- **Multi-datacenter:** un líder por región → las escrituras son **locales** (baja latencia) y la región sobrevive a la caída de otra. Cada líder replica a los demás de fondo.
- **Clientes offline:** una app que escribe offline (calendario en el celu) es, de hecho, un líder; al reconectar, sincroniza. Cada dispositivo es un "datacenter de uno".
- **Colaboración en vivo:** Google Docs, Figma — cada cliente edita su copia y se mergea.

El precio, inevitable: **conflictos de escritura**. Dos líderes aceptan, sin verse, escrituras a la **misma clave** (Ana pone el título "X" en la región US, Beto pone "Y" en EU "al mismo tiempo"). Cuando las réplicas se cruzan, hay **dos valores concurrentes** y el sistema tiene que decidir. En single-leader esto **no existe** (un solo orden); en multi-leader es el problema central.

> La regla para la entrevista: **multi-leader solo si lo justificás** (multi-región con escritura local, offline, colaboración). Si lo metés "para escalar escrituras" sin necesitar escritura en varios lugares, te compraste el problema de conflictos gratis. La mejor resolución de conflictos es **no tenerlos**: particioná para que cada clave tenga un solo líder (sharding por entidad/región).

**Ejercicios 7**
7.1 🔁 Dá los tres escenarios donde multi-leader se justifica.
7.2 🧠 ¿Por qué single-leader no tiene conflictos de escritura y multi-leader sí?
7.3 🧠 ¿Cuál es "la mejor resolución de conflictos" y cómo se logra?

---

## Módulo 8 — Topologías multi-región: active-passive vs active-active

**Teoría.** Cuando el sistema cruza regiones, dos diseños:

- **Active-passive (failover regional):** una región **activa** sirve todo; las otras son **réplicas pasivas** en espera. Si la activa cae, **promovés** otra (failover regional). Simple, sin conflictos (una sola escribe), pero la región pasiva está "ociosa" y el failover tiene latencia + riesgo de pérdida (módulo 6).
- **Active-active (multi-master regional):** **todas** las regiones sirven lecturas **y escrituras** locales. Máxima disponibilidad y latencia mínima, pero es multi-leader → **conflictos** entre regiones que hay que resolver (módulo 9).

Dos métricas que **tenés que** nombrar al hablar de recuperación regional (y todo arquitecto las pregunta):

- **RPO (Recovery Point Objective):** cuántos datos podés perder, medido en tiempo. RPO=0 → no perdés nada (exige replicación síncrona/consenso). RPO=5min → tolerás perder los últimos 5 min.
- **RTO (Recovery Time Objective):** cuánto podés tardar en volver a estar arriba. RTO=30s → el failover tiene que completar en 30s.

> RPO y RTO **son decisiones de negocio, no técnicas**: definen cuánta plata gastás en replicación y automatización del failover. Active-passive con async = RPO de segundos/minutos, RTO de minutos. Active-active = RPO≈0 y RTO≈0 (otra región ya sirve), al precio de resolver conflictos. La respuesta senior empieza por **"¿cuál es el RPO/RTO del negocio?"** antes de elegir topología.

**Ejercicios 8**
8.1 🔁 Definí RPO y RTO en una frase cada uno.
8.2 🧠 ¿Por qué active-active da mejor RPO/RTO que active-passive, y qué problema nuevo trae?
8.3 🧠 Un banco exige RPO=0 para transacciones. ¿Qué tipo de replicación te obliga eso y qué le hace a la latencia de escritura?

---

## Módulo 9 — Resolución de conflictos: detección, LWW, siblings

**Teoría.** Hay dos escrituras concurrentes a la misma clave. ¿Cómo decidís? Primero **detectás** la concurrencia (no son "una más vieja que otra": son concurrentes — ninguna vio a la otra). Eso lo hacen los **vector clocks** de [consenso módulo 6](consenso.md): si los vectores son comparables, una sucede a la otra (la nueva gana); si son **concurrentes**, hay conflicto real. Después, tres estrategias:

1. **Last-Write-Wins (LWW):** quedate con la de **timestamp más alto**. **Simple y peligroso:** descarta silenciosamente la otra escritura, y peor, el "más alto" depende de relojes físicos que **derivan** ([consenso módulo 4](consenso.md)) → con clock skew, **perdés la escritura que de verdad fue última**. Cassandra usa LWW por celda; es aceptable solo si **perder escrituras concurrentes es tolerable** (último estado de un sensor, sí; saldo de cuenta, jamás).
2. **Guardar ambas (siblings):** el sistema detecta el conflicto y conserva **las dos versiones** (Dynamo/Riak: *siblings*). La próxima lectura recibe ambas y la **aplicación** (o el usuario) decide cómo mergear. No pierde datos, pero traslada la complejidad a la app.
3. **Conflict-free (CRDTs):** estructurás el dato para que **no pueda haber conflicto** — cualquier orden de merge da el mismo resultado. Es el módulo 10-11. La opción más elegante cuando tu dato encaja en un CRDT.

> El reflejo de entrevista: cuando te digan "dos escrituras concurrentes", **no contestes "LWW"** de una. Decí: *"primero detecto si son concurrentes con vector clocks; si lo son, LWW pierde datos por clock skew, así que o guardo siblings y mergeo en la app, o uso un CRDT si el dato lo permite — LWW solo si perder una escritura concurrente es aceptable para este dato."* Esa frase es la diferencia.

**Ejercicios 9**
9.1 🔁 ¿Qué estructura usás para **detectar** que dos escrituras son concurrentes (no una más vieja)?
9.2 🧠 ¿Por qué LWW por timestamp físico puede descartar la escritura que realmente fue última?
9.3 🧠 ¿Cuándo es aceptable LWW y cuándo tenés que ir a siblings o CRDTs? Dá un ejemplo de cada uno.

---

## Etapa 4 — CRDTs en serio

## Módulo 10 — CRDTs: convergencia sin coordinación

**Teoría.** Un **CRDT** (Conflict-free Replicated Data Type) es una estructura de datos diseñada para que **réplicas que reciben las mismas actualizaciones en cualquier orden converjan al mismo estado, sin coordinación**. Esa es la magia: no hay líder, no hay locks, no hay consenso — y aun así todos terminan iguales. La propiedad se llama **Strong Eventual Consistency**: dos réplicas que vieron el mismo conjunto de updates están en el mismo estado (no solo "eventualmente", sino *garantizado* por la matemática).

¿Cómo? Las operaciones de merge cumplen tres propiedades algebraicas (forman un *semilattice*):

- **Conmutativa:** `a ⊔ b = b ⊔ a` (el orden no importa).
- **Asociativa:** `(a ⊔ b) ⊔ c = a ⊔ (b ⊔ c)` (el agrupamiento no importa).
- **Idempotente:** `a ⊔ a = a` (recibir lo mismo dos veces no rompe nada).

Esas tres juntas significan que **no importa en qué orden ni cuántas veces llegan los updates**: el merge siempre da el mismo resultado. Por eso toleran red que reordena, duplica y reentrega (que es exactamente lo que hace una red real).

Dos familias:
- **State-based (CvRDT):** cada réplica manda **su estado entero**; el merge es una función que toma el "máximo" (LUB) de dos estados. Robusto (idempotente, tolera reentrega), pero manda mucho dato.
- **Op-based (CmRDT):** cada réplica manda **la operación** ("incrementá en 1"); las ops tienen que ser conmutativas y la entrega **confiable y exactly-once** (o idempotente). Manda menos, pero exige más de la red.

> El insight: los CRDTs **cambian dónde ponés el esfuerzo**. En vez de coordinar *al escribir* (locks, consenso — caro, lento, se planta bajo partición), diseñás el *tipo de dato* para que el merge sea trivial y **siempre disponible** (escribís local, mergeás después). Es la base de Figma, Google Docs, Redis CRDTs (Active-Active), Riak. El límite: no todo problema encaja en un CRDT (módulo 11).

**Ejercicios 10**
10.1 🔁 ¿Qué garantiza un CRDT y cómo se llama esa propiedad?
10.2 🧠 ¿Por qué conmutatividad + asociatividad + idempotencia hacen que el orden de llegada de los updates no importe?
10.3 🧠 Diferencia entre CRDT state-based y op-based, y qué le exige cada uno a la red.

---

## Módulo 11 — El zoo de CRDTs (y sus límites)

**Teoría.** Los CRDTs canónicos que conviene conocer (los construís en el [laboratorio](replicacion-practica.md)):

- **G-Counter (grow-only counter):** un contador que solo sube. Cada réplica tiene **su propia** casilla (un vector, una por nodo); incrementar = subir tu casilla; **merge = máximo casilla a casilla**; **valor = suma de todas**. Como cada réplica solo toca su casilla, nunca hay conflicto, y el `max` es idempotente/conmutativo. Sirve para likes, vistas, métricas.
- **PN-Counter:** un contador que sube **y baja**. Son **dos** G-Counters (uno de incrementos `P`, otro de decrementos `N`); el valor es `sum(P) - sum(N)`. No podés restar directo en un grow-only, así que decrementar = incrementar el contador `N`.
- **LWW-Register:** un valor único con timestamp; el merge se queda con el de mayor timestamp. Es LWW (módulo 9) hecho CRDT — **puede perder** una escritura concurrente, pero converge.
- **OR-Set (Observed-Remove Set):** un conjunto donde podés agregar y quitar. El truco: cada `add(x)` lleva un **tag único**; `remove(x)` solo borra **los tags que esa réplica observó**. Así, un `add` concurrente con un `remove` → **gana el add** (add-wins), porque su tag nuevo no fue observado por el remove. Resuelve el problema naíf de "un set donde add y remove concurrentes se pisan".

> ⚠️ **El límite que tenés que decir:** **no todo es un CRDT.** Cosas que **requieren una invariante global** —"el saldo no puede ser negativo", "máximo 100 asientos", "este username es único"— **no** se pueden hacer conflict-free, porque mantener la invariante exige **coordinación** (alguien tiene que ver todas las operaciones antes de decidir). Para eso necesitás un líder/consenso. Los CRDTs brillan en datos **acumulativos o de conjunto** sin invariantes cross-réplica (contadores, carritos, presencia, texto colaborativo); fallan donde hace falta un "no" global.

**Ejercicios 11**
11.1 🔁 ¿Cómo evita conflictos un G-Counter y cómo se calcula su valor?
11.2 🧠 ¿Por qué un PN-Counter necesita dos contadores en vez de sumar y restar en uno?
11.3 🧠 Dá un ejemplo de un requisito que **NO** se puede implementar como CRDT y explicá por qué.

---

## Etapa 5 — Transacciones distribuidas

## Módulo 12 — Dual-write: por qué cruzar servicios es difícil

**Teoría.** Tenés que actualizar **dos sistemas** en una operación: guardar el pedido en Postgres **y** publicar un evento en Kafka. O debitar la cuenta en el servicio A **y** acreditar en el servicio B. El **dual-write problem**: no hay forma de escribir en dos sistemas distintos **atómicamente** sin un protocolo especial. Si escribís en uno y el proceso muere antes del segundo, quedás **inconsistente** (pedido guardado, evento nunca publicado → nadie le cobra; o cuenta debitada, nunca acreditada → plata que se evaporó).

No te salva el orden ni los try/catch:
- Escribís DB, después Kafka: si Kafka falla o el proceso muere en el medio, DB tiene el pedido pero no hay evento.
- Escribís Kafka, después DB: si DB falla, hay un evento de un pedido que no existe.
- "Hago rollback del primero si falla el segundo": el rollback **también** puede fallar, y si ya respondiste al cliente, no podés des-responder.

> La trampa mental: en una sola base, `BEGIN/COMMIT` te da atomicidad gratis. **Cruzando servicios/sistemas no hay un COMMIT compartido.** Las salidas son tres: (a) **2PC** — un protocolo de commit atómico distribuido (módulo 13, casi siempre la respuesta equivocada); (b) **outbox + evento** — escribís el evento en la **misma** transacción de la DB (una tabla outbox) y un proceso lo publica después, convirtiendo el dual-write en un single-write ([event-driven](event-driven.md)); (c) **saga** — una secuencia de transacciones locales con compensaciones (módulo 14). La (b) y la (c) son lo que se usa en serio.

**Ejercicios 12**
12.1 🔁 ¿Qué es el dual-write problem en una frase?
12.2 🧠 Mostrá por qué "escribo DB y después Kafka, con rollback si falla" no resuelve el problema.
12.3 🧠 ¿Cómo convierte el patrón **outbox** un dual-write en un single-write? (Recordá [event-driven](event-driven.md).)

---

## Módulo 13 — 2PC / 3PC: el coordinador, el bloqueo

**Teoría.** **Two-Phase Commit (2PC)** es el protocolo clásico para commit atómico distribuido: que N participantes commiteen **todos** o **ninguno**. Hay un **coordinador** y dos fases:

1. **Prepare (votar):** el coordinador pregunta a cada participante "¿podés commitear?". Cada uno hace el trabajo, escribe a su WAL, **toma los locks**, y responde **YES** (me comprometo, ya no puedo echarme atrás) o **NO**. Quedó *in-doubt*: bloqueado, esperando la orden.
2. **Commit/Abort (decidir):** si **todos** votaron YES, el coordinador manda **COMMIT**; si alguno votó NO, manda **ABORT**. Los participantes ejecutan y **liberan los locks**.

La garantía de atomicidad es real. El problema es **fatal** y por eso 2PC casi no se usa en sistemas modernos de alta disponibilidad:

> 🔥 **Falla en 4 capas — el coordinador se cae con todos in-doubt (blocking problem).**
> 1. **Qué se rompe:** los participantes votaron YES (locks tomados, in-doubt) y el coordinador **se cae antes de mandar la decisión**. Los participantes **no pueden** decidir solos (no saben si los otros votaron YES) → quedan **bloqueados con los locks tomados, indefinidamente**, hasta que el coordinador vuelva.
> 2. **Por qué a esta escala:** los locks tomados bloquean a **todos** los demás que quieran esos datos → la indisponibilidad de un coordinador se propaga como un freeze. Y el coordinador es un **SPOF** que serializa todo.
> 3. **Control de corto plazo:** timeouts + intervención manual; coordinador con su propio log durable para recuperar al reiniciar.
> 4. **Largo plazo — no usar 2PC para esto:** rediseñar con **saga** (transacciones locales + compensación, sin locks distribuidos ni coordinador bloqueante) o **outbox**. 2PC se reserva para casos acotados (XA en un solo datacenter, transacciones entre tablas de un mismo motor distribuido).

**3PC (three-phase commit)** agrega una fase intermedia (*pre-commit*) para ser no-bloqueante... pero **asume una red síncrona** (cotas de tiempo) y **no tolera particiones de red** → en una red real (asíncrona, parcialmente síncrona) **tampoco sirve**. Por eso, en la práctica, "commit atómico distribuido sin bloqueo" se resuelve con **consenso** (Raft/Paxos detrás de la decisión de commit, como hace Spanner/Calvin), no con 3PC.

**Ejercicios 13**
13.1 🔁 ¿Qué pasa en cada una de las dos fases de 2PC?
13.2 🧠 Explicá el blocking problem: ¿por qué los participantes no pueden decidir solos si el coordinador se cae tras el prepare?
13.3 🧠 ¿Por qué 3PC tampoco se usa, y qué lo reemplazó para el commit atómico sin bloqueo?

---

## Módulo 14 — Saga + outbox: la alternativa práctica

**Teoría.** Como 2PC bloquea y no escala, los sistemas distribuidos modernos resuelven las transacciones cross-servicio con **sagas**: en vez de un commit atómico global, una **secuencia de transacciones locales**, cada una en su servicio, y si una falla, **compensás** (deshacés) las anteriores con operaciones inversas. (La teoría a fondo está en [event-driven](event-driven.md); acá la ubicamos en el mapa de replicación/transacciones.)

- **Coreografía:** cada servicio escucha eventos y reacciona (sin orquestador central). Simple para pocos pasos; se vuelve un espagueti de eventos con muchos.
- **Orquestación:** un orquestador (o un workflow engine: Temporal, Step Functions) dirige la secuencia y dispara las compensaciones. Más control y visibilidad para flujos largos.

La diferencia conceptual con 2PC, que es lo que tenés que decir:

| | 2PC | Saga |
|---|---|---|
| Atomicidad | **Atómica** (todo o nada, real) | **Eventual** (pasa por estados intermedios visibles) |
| Aislamiento | Sí (locks durante el protocolo) | **No** (otros ven estados parciales) → necesitás contramedidas |
| Bloqueo | **Bloquea** (in-doubt con locks) | No bloquea (transacciones locales cortas) |
| Falla del coordinador | Freeze | La saga reintenta/compensa; el estado está en el log |

> El precio de la saga: **no hay aislamiento**. Entre el paso 1 y el paso 3, otros pueden ver el estado a medias (el pedido existe pero el pago no se confirmó). Lo manejás con estados explícitos (`PENDIENTE`/`CONFIRMADO`), semántica de "reserva con TTL", e **idempotencia** en cada paso (porque vas a reintentar). Y como el dual-write dentro de cada servicio sigue existiendo (escribir DB + emitir evento), cada paso usa **outbox** para que sea atómico localmente. Saga + outbox + idempotencia es el combo que se usa en producción — no 2PC.

**Ejercicios 14**
14.1 🔁 ¿Qué hace una saga cuando un paso intermedio falla?
14.2 🧠 ¿Cuál es el precio principal de una saga respecto de 2PC, y cómo lo mitigás?
14.3 🧠 ¿Por qué cada paso de una saga necesita outbox e idempotencia?

---

## Etapa 6 — Consistencia y cierre

## Módulo 15 — El espectro de consistencia + PACELC

**Teoría.** "Consistencia" no es un sí/no: es un **espectro** de garantías, de la más fuerte (y cara) a la más débil (y barata). Tenés que poder ubicar cada modelo:

- **Linealizabilidad (linealizable):** el sistema se comporta como si hubiera **una sola copia** y cada operación ocurre en un **instante** entre su inicio y su fin (respeta el orden real/wall-clock). Si una escritura terminó, **toda** lectura posterior la ve. Es lo más fuerte para **un objeto**; lo da Raft/consenso. Caro: round-trip a quórum, se planta bajo partición.
- **Serializabilidad:** las **transacciones** (multi-objeto) se comportan como si se hubieran ejecutado en **algún** orden serial. Es la garantía de aislamiento de las DBs (el `SERIALIZABLE` de [postgresql](postgresql.md)). **Ojo:** es **ortogonal** a linealizabilidad — una habla de recencia de un objeto, la otra de aislamiento de transacciones. **Strict serializability** = las dos juntas (Spanner).
- **Consistencia causal:** respeta el orden **causa→efecto** (happened-before, los vector clocks de [consenso](consenso.md)) pero **no** ordena operaciones concurrentes. Es **la más fuerte que podés tener sin coordinación / bajo partición** — por eso es el sweet spot de muchos sistemas: te da "la respuesta nunca aparece antes que la pregunta" sin pagar consenso.
- **Eventual:** "si parás de escribir, eventualmente todas las réplicas convergen". No dice **cuándo** ni ordena nada. La más barata y disponible; es lo que da Dynamo/Cassandra por default y los CRDTs (en su versión *strong eventual*).

> ¿Y dónde caen las **garantías de sesión** del módulo 5 (read-your-writes, monotonic reads)? Por debajo de causal y por encima de eventual, pero en **otro eje**: son **per-cliente** (garantizan algo sobre *tu* sesión), no per-objeto global. Linealizabilidad las implica gratis (si todo está al día para todos, también lo está para vos); por eso elegirlas sueltas es pagar mucho menos cuando solo te duele *tu* propia lectura.

Y el marco que lo ata, más útil que CAP: **PACELC**. *If **P**artition → choose **A** or **C**; **E**lse (operación normal) → choose **L**atency or **C**onsistency.* CAP solo habla del caso de partición; PACELC agrega la verdad de todos los días: **aun sin partición**, más consistencia = más latencia (más réplicas que esperar). Cassandra es **PA/EL** (disponible y de baja latencia, eventual); Spanner es **PC/EC** (consistente siempre, paga latencia).

> **La trampa de CAP — "CA no existe" y las dos C.** Dos errores que un entrevistador caza al toque:
> 1. **No podés "elegir CA".** En un sistema distribuido la **P** (partición) no es una opción que aceptás o rechazás: las redes fallan, los cables se cortan, una pausa de GC larga **es indistinguible de una partición** para el resto del sistema (un nodo pausado no responde, igual que uno aislado). **P siempre puede pasar**, así que el único momento de decisión real es *durante* una partición, y ahí elegís **C o A** — nunca las dos a la vez. Lo que la gente llama un sistema "CA" es en realidad **un solo nodo** (Postgres en una máquina) —o, más en general, un sistema que **asume una red sin particiones** y deja de dar garantías si una ocurre—: consistente y disponible mientras no haya red que particionar, pero ni escala horizontalmente ni sobrevive a la caída de esa máquina. *"Elijo CA"* es, en los hechos, *"no tengo un sistema distribuido"*. Por eso PACELC arranca asumiendo que la partición **va** a ocurrir y pregunta por el **else**: el trade-off de todos los días es latencia vs consistencia, no la foto excepcional de la partición.
> 2. **Las dos C no son la misma C.** La **C de CAP** es **linealizabilidad** (recencia y orden real de **un objeto**); la **C de ACID** es **serializabilidad** (aislamiento de **transacciones** multi-objeto). Son ejes **ortogonales** (lo viste arriba): CAP no habla de transacciones. Confundirlas produce frases sin sentido como *"NoSQL sacrifica ACID por estar en AP"* — mezcla dos cosas distintas. **Spanner** es "CP" **y** strict-serializable a la vez; y un store que opera en modo "AP" como **Cassandra** puede igual ofrecer transacciones ligeras (LWT) acotadas a una sola partición. La C de CAP y la de ACID se deciden por separado (y muchas veces, por operación).

> El reflejo senior: cuando te pregunten "¿qué consistencia?", no digas "fuerte" por reflejo. Ubicá: *"¿necesito recencia de un objeto (linealizable), aislamiento de transacciones (serializable), o me alcanza con causal/eventual que es más disponible y barato?"*. Y usá PACELC para nombrar el costo en latencia **además** del de disponibilidad.

**Ejercicios 15**
15.1 🔁 Ordená de más fuerte a más débil: causal, eventual, linealizable. (Dejá serializabilidad afuera: es ortogonal.)
15.2 🧠 ¿Por qué linealizabilidad y serializabilidad son ortogonales? ¿Qué es strict serializability?
15.3 🧠 ¿Qué agrega PACELC sobre CAP, y qué significa que Cassandra sea "PA/EL" y Spanner "PC/EC"?
15.4 🧠 Un candidato dice *"mi diseño es CA"* y *"mi DB NoSQL está en AP, así que no es ACID"*. Señalá por qué **ambas** frases están mal.

---

## Módulo 16 — Cómo hablar de replicación en una entrevista

**Teoría.** Cierre operativo. Ante un tema de replicación/multi-región/consistencia, estructurá con **las tres cosas** y, ante fallas, **las cuatro capas**.

**Las tres cosas (ejemplo: "diseñá la capa de datos multi-región"):**
- **(a) Happy path:** "single-leader por shard con réplicas de lectura; multi-región active-passive con replicación semi-síncrona a una región para RPO bajo; lecturas locales, escrituras al líder de la región activa".
- **(b) Modelo de falla:** "si cae el líder, failover al follower más adelantado con fencing; si cae la región activa, promuevo la pasiva (RTO de minutos); el lag produce read-your-writes, lo tapo con sticky/LSN".
- **(c) Recuperación:** "el follower promovido tiene los commits por semi-sync; el líder viejo, al volver, es fenced; reconcilio con anti-entropía si hubo divergencia".

**Las muletillas que suenan a que entendiste:**
- *"¿Cuál es el RPO/RTO del negocio?"* — antes de elegir topología.
- *"Multi-leader solo si necesito escritura local en varias regiones; si no, single-leader y particiono."*
- *"No digo LWW de una: detecto concurrencia con vector clocks; LWW pierde datos por clock skew; CRDT si el dato encaja, siblings si no."*
- *"¿Necesito un commit atómico de verdad (raro), o me alcanza saga + outbox + idempotencia?"*
- *"Elijo la garantía mínima que tapa la anomalía: read-your-writes, no linealizabilidad global."*

> **Cierre.** Replicar es asumir que vas a tener **copias que mienten un rato**, y todo el oficio es **elegir cuánto rato y qué hacés con los desacuerdos**. La escalera: single-leader (sin conflictos) → semi-sync/consenso (failover sin pérdida) → multi-leader/CRDTs (escritura local, convergencia sin coordinar) → saga (transacciones sin 2PC). El que sabe **cuál** de esos peldaños pide el problema —y nombra el RPO/RTO, la anomalía concreta y el costo en PACELC— ya razona como senior. Ahora pasá al [laboratorio práctico](replicacion-practica.md) y **construí**: el lag que te hace leer el pasado, LWW tirando una escritura, un CRDT que converge venga el orden que venga, y el 2PC que se congela cuando muere el coordinador.

**Ejercicios 16**
16.1 🔁 Listá "las tres cosas" aplicadas a una capa de datos multi-región.
16.2 🧠 Te preguntan "¿qué pasa si se cae la región activa?". Respondé nombrando RPO/RTO y el failover.
16.3 🧠 ¿Por qué "elegí la garantía mínima que tapa la anomalía" es mejor consejo que "usá consistencia fuerte"?

---

## Soluciones

**1.1** (1) Disponibilidad, (2) latencia geográfica, (3) throughput de lectura. La que **no** escala escrituras es el throughput de lectura (con single-leader, todas las escrituras pasan igual por el líder).

**1.2** Porque con un solo líder, **toda** escritura tiene que pasar por él para mantener un orden único; las réplicas solo copian. Sumar followers reparte las **lecturas**, pero el líder sigue siendo el único que acepta escrituras → no sube el techo de escritura. Para escalar escrituras necesitás **sharding** (partir los datos) o multi-leader.

**1.3** Esperar a todas las réplicas → consistencia fuerte (todas tienen el dato antes de responder) pero **alta latencia** (pagás el round-trip a la más lenta) y baja disponibilidad (una réplica caída frena). No esperar → **baja latencia** y alta disponibilidad, pero las réplicas quedan divergentes un rato → consistencia eventual y sus anomalías. No hay opción que dé las dos cosas gratis.

**2.1** **Single-leader**: un solo nodo acepta escrituras, así que define **un único orden** y todos los followers aplican la misma secuencia → no hay dos escrituras "concurrentes a la misma clave que no se vieron".

**2.2** Ganan **escritura en más de un lugar**: disponibilidad de escritura (si un nodo/región cae, otro acepta) y **latencia baja de escritura** (escribís local). A cambio aparecen escrituras concurrentes a la misma clave que hay que reconciliar.

**2.3** **Single-leader** por default (simple, sin conflictos, cubre el 90% de los CRUD). Cambiaría si necesito **escritura local en varias regiones** (multi-región active-active), **clientes offline** que escriben sin conexión, o **colaboración en vivo** — ahí justifico multi-leader/CRDTs.

**3.1** Que las escrituras **confirmadas pero todavía no propagadas** a los followers **se pierden**: el líder respondió OK y se cayó antes de mandarlas; el follower promovido no las tiene.

**3.2** Porque **cualquier** follower caído o lento frenaría **todas** las escrituras (el líder espera la confirmación de todos). Con muchas réplicas, la probabilidad de que al menos una esté lenta tiende a 1 → el sistema no podría escribir casi nunca.

**3.3** Garantiza que la escritura confirmada está en **al menos dos nodos** (líder + ≥1 follower sync) → sobrevive a la caída del líder (durabilidad), cosa que la async no asegura. El costo respecto de la async: **más latencia** de escritura (esperás el round-trip al follower sync) y algo menos de disponibilidad (necesitás que ese follower esté vivo, aunque no todos).

**4.1** (1) Read-your-writes (no ves tu propia escritura), (2) monotonic reads (lecturas que retroceden en el tiempo), (3) consistent prefix reads (ves escrituras fuera de orden causal).

**4.2** Hacés dos lecturas: la 1ª cae en un follower adelantado (ves un comentario nuevo), la 2ª en uno más atrasado (el comentario "todavía no existe") → la UI muestra que algo que viste **desapareció**: el tiempo retrocedió. Pasa porque cada lectura puede ir a una réplica con distinto lag.

**4.3** Porque distintas particiones replican **independientemente** y a distinto ritmo; si una escritura causalmente posterior (la respuesta) está en una partición más rápida que la causalmente anterior (la pregunta), un lector ve la respuesta antes que la pregunta. Con una sola partición hay un solo flujo ordenado; al particionar, perdés el orden entre particiones salvo que lo fuerces.

**5.1** (1) Leer del **líder** lo que el usuario pudo editar; (2) **sticky routing** al líder por una ventana tras escribir; (3) **timestamp lógico/LSN**: el cliente recuerda la posición de su última escritura y exige una réplica al menos tan nueva (o lee del líder).

**5.2** Si un usuario lee **siempre de la misma réplica**, nunca "salta" a una más atrasada que la anterior → sus lecturas nunca retroceden (monotonía). NO te garantiza ver **lo último** (esa réplica puede estar atrasada respecto del líder); solo que no vas hacia atrás.

**5.3** Porque las garantías de sesión resuelven **exactamente** el síntoma (tus escrituras / no retroceder) a un costo mucho menor: no exigen coordinación global ni round-trips a quórum por cada operación. La consistencia fuerte global tapa todo pero es cara (latencia, se planta bajo partición); si solo te duele una anomalía, pagar por todo es sobre-ingeniería.

**6.1** Porque el follower promovido puede estar **atrasado** (replicación async): le faltan las escrituras que el líder viejo confirmó al cliente pero no alcanzó a propagar. Esas escrituras se descartan al promover.

**6.2** El follower con mayor LSN es el **más adelantado** → perdés menos (o nada). Pero sigue abierto el **líder viejo**: si no estaba muerto sino particionado, puede volver y creerse líder (split-brain) → hay que **fencing** (que el storage rechace sus escrituras) o asegurarse de que se degrade al ver un término mayor.

**6.3** Semi-síncrona: la escritura confirmada está en ≥2 nodos, así que el follower promovido (que era uno de los sync) **la tiene** → no se pierde. Consenso (Raft): por la **election restriction**, solo se elige líder a un nodo con todos los commits (un candidato al que le falta un commit no junta mayoría) → el líder nuevo ya tiene todo lo committeado. En ambos, "failover sin pérdida" = garantizar que el promovido tenga las escrituras durables.

**7.1** (1) Multi-datacenter (un líder por región, escritura local), (2) clientes offline (cada dispositivo es un líder que sincroniza al reconectar), (3) colaboración en vivo (cada cliente edita su copia y mergea).

**7.2** Single-leader: un solo nodo ordena todas las escrituras → no existen dos escrituras "que no se vieron". Multi-leader: varios nodos aceptan escrituras sin verse entre sí (se replican async) → dos escrituras a la misma clave pueden ser concurrentes y entrar en conflicto al cruzarse.

**7.3** "No tener conflictos": **particionar** para que cada clave/entidad tenga un **solo** líder (sharding por entidad o por región). Si cada dato tiene un único dueño que lo escribe, no hay escrituras concurrentes que reconciliar.

**8.1** RPO: cuántos datos (medidos en tiempo) podés perder ante un desastre (RPO=0 → nada). RTO: cuánto tiempo podés tardar en volver a estar operativo tras el desastre.

**8.2** Active-active ya tiene **otra región sirviendo** (lecturas y escrituras), así que ante la caída de una, las demás siguen → RTO≈0 y, con replicación apretada, RPO≈0. El problema nuevo: es multi-leader → **conflictos** entre regiones que hay que resolver (LWW/CRDT/siblings). Active-passive deja la pasiva ociosa y el failover tarda (RTO mayor) y puede perder datos (RPO mayor con async).

**8.3** RPO=0 obliga a replicación **síncrona** (o consenso): la transacción no se confirma hasta que está durable en ≥2 nodos/regiones, así nada se pierde. Eso **sube la latencia de escritura** porque cada commit paga el round-trip a la otra réplica/región (decenas de ms si es cross-región).

**9.1** **Vector clocks** ([consenso módulo 6](consenso.md)): si los vectores de las dos escrituras son comparables (`V<W` o `W<V`), una sucede a la otra; si son **concurrentes** (ninguno ≤ el otro), es un conflicto real (ninguna vio a la otra).

**9.2** Porque "más alto" se mide con **relojes físicos que derivan** (clock skew). Una escritura que ocurrió **después** en el tiempo real puede llevar un timestamp **menor** (su nodo tenía el reloj atrasado) → LWW se queda con la otra y descarta la que de verdad fue última. El reloj decide, no la causalidad.

**9.3** **LWW aceptable:** el dato es un "último estado" donde perder una escritura concurrente no duele (lectura de un sensor, último visto, cache de presencia). **Siblings/CRDT necesarios:** cuando perder una escritura corrompe (carrito de compras → CRDT OR-Set para no perder ítems; saldo/contador → PN-Counter; documento colaborativo → CRDT de texto; o siblings + merge en la app si no encaja en un CRDT).

**10.1** Garantiza que réplicas que recibieron **el mismo conjunto** de updates convergen al **mismo estado**, sin coordinación, en cualquier orden de llegada. La propiedad se llama **Strong Eventual Consistency**.

**10.2** Conmutativa → el orden de merge no cambia el resultado; asociativa → el agrupamiento tampoco; idempotente → recibir el mismo update repetido no lo cambia. Juntas, el estado final depende **solo del conjunto** de updates recibidos, no del orden, la cantidad de veces ni el agrupamiento con que llegaron → tolera la red real (reordena, duplica, reentrega).

**10.3** State-based (CvRDT): manda el **estado entero**, el merge toma el LUB (máximo) de dos estados; tolera reentrega/duplicados (idempotente) pero manda mucho dato. Op-based (CmRDT): manda **la operación**; exige que las ops sean conmutativas y que la entrega sea **confiable y sin duplicados** (exactly-once o idempotente), a cambio manda menos.

**11.1** Cada réplica incrementa **solo su propia casilla** (un vector con una posición por nodo), así nunca hay dos réplicas tocando la misma casilla → sin conflicto. Merge = **máximo casilla a casilla**; **valor = suma de todas las casillas**.

**11.2** Porque un G-Counter es **grow-only**: el merge es `max`, que es monótono creciente; si restaras, el `max` "olvidaría" el decremento (una réplica con valor menor perdería contra otra mayor). Solución: dos G-Counters, `P` (incrementos) y `N` (decrementos); decrementar = subir `N`; valor = `sum(P) - sum(N)`. Ambos solo crecen, así que el `max` sigue siendo válido.

**11.3** Cualquier requisito con **invariante global**: "el saldo nunca negativo", "máximo 100 asientos", "username único". No se puede hacer conflict-free porque mantener la invariante exige que **alguien vea todas las operaciones** antes de aceptar una (coordinación) → necesitás líder/consenso. Los CRDTs convergen, pero no pueden **rechazar** una operación que violaría una invariante que depende de otras réplicas.

**12.1** Que no podés escribir en **dos sistemas distintos** de forma **atómica** sin un protocolo especial: si el proceso falla entre la primera y la segunda escritura, quedás inconsistente (uno escrito, el otro no).

**12.2** Si escribís DB y después Kafka: el proceso puede morir **entre** las dos → DB tiene el pedido, no hay evento. El rollback "si falla el segundo" tampoco salva: el rollback puede fallar también, y si el proceso muere **antes** de ejecutarlo, no hay quién haga rollback; además, si ya respondiste OK al cliente, no podés deshacer esa respuesta. No hay un COMMIT que abarque los dos sistemas.

**12.3** Outbox: escribís el evento en una **tabla de la misma DB** dentro de la **misma transacción** que el dato de negocio → el dual-write se vuelve un **single-write** atómico (DB local). Un proceso aparte (relay/CDC) lee la tabla outbox y publica el evento a Kafka de forma asíncrona y reintentable. Si el publish falla, el evento sigue en la tabla → se reintenta; nunca hay "dato sin evento".

**13.1** **Prepare:** el coordinador pregunta "¿podés commitear?"; cada participante hace el trabajo, escribe a su WAL, toma locks y vota YES (comprometido, in-doubt) o NO. **Commit/Abort:** si todos votaron YES, el coordinador manda COMMIT; si alguno NO, manda ABORT; los participantes ejecutan y liberan locks.

**13.2** Porque tras votar YES un participante está **comprometido pero no sabe la decisión global** (no sabe si los **otros** votaron YES o NO). Solo el coordinador junta todos los votos. Si el coordinador se cae antes de comunicar la decisión, el participante no puede commitear (quizás otro votó NO) ni abortar (quizás todos votaron YES y ya hay un COMMIT en camino) → queda **bloqueado con los locks tomados** hasta que el coordinador vuelva.

**13.3** 3PC agrega una fase pre-commit para ser no-bloqueante, pero **asume red síncrona** (cotas de tiempo conocidas) y **no tolera particiones** → en una red real falla. Lo reemplazó el **consenso** (Raft/Paxos): la decisión de commit se replica por consenso (como en Spanner/Calvin), que es no-bloqueante y tolera particiones cediendo liveness, no safety.

**14.1** **Compensa**: ejecuta transacciones inversas que deshacen el efecto de los pasos ya completados (p. ej. reembolsar el cobro, liberar el stock reservado). No hay rollback global; se "deshace hacia atrás" con operaciones de negocio.

**14.2** El precio es **la falta de aislamiento**: entre pasos, otros pueden ver estados intermedios (el pedido existe pero el pago no se confirmó). Se mitiga con estados explícitos (`PENDIENTE`/`CONFIRMADO`), reservas con TTL, y diseño para que el estado parcial no engañe (no mostrar como "comprado" hasta confirmar).

**14.3** **Outbox** porque cada paso hace su propio dual-write (escribir su DB + emitir el evento del siguiente paso) y eso tiene que ser atómico localmente. **Idempotencia** porque los pasos se **reintentan** (at-least-once): si un evento se entrega dos veces, ejecutar el paso de nuevo no debe duplicar el efecto (cobrar dos veces).

**15.1** De más fuerte a más débil: **linealizable** → **causal** → **eventual**. (La serializabilidad es ortogonal: aplica a transacciones multi-objeto, no entra en esta línea de "recencia de un objeto".)

**15.2** Porque hablan de cosas distintas: **linealizabilidad** = recencia/orden real de operaciones sobre **un objeto** (una sola copia, lo último visible); **serializabilidad** = las **transacciones** multi-objeto equivalen a *algún* orden serial (aislamiento), sin exigir que ese orden respete el tiempo real. Podés tener una sin la otra. **Strict serializability** = las dos juntas: transacciones serializables **en** orden de tiempo real (lo que da Spanner).

**15.3** PACELC agrega el caso **sin partición**: aun cuando todo funciona, hay un trade-off **Latencia vs Consistencia** (más consistencia = esperar más réplicas). CAP solo cubre la partición. "Cassandra PA/EL" = ante partición elige Disponibilidad, y en operación normal elige Latencia (eventual, rápida). "Spanner PC/EC" = ante partición elige Consistencia, y en operación normal también Consistencia (paga latencia con TrueTime).

**15.4** *"Mi diseño es CA"*: la partición **no es opcional** en un sistema distribuido (siempre puede ocurrir, y una pausa larga es indistinguible de una), así que no podés renunciar a la **P**; "CA" describe un solo nodo o un sistema que asume una red sin particiones, no una elección de diseño distribuido. La decisión real es C **o** A *durante* la partición. *"NoSQL en AP no es ACID"*: mezcla **las dos C** — la C de CAP (linealizabilidad, recencia de un objeto) es **ortogonal** a las garantías ACID (serializabilidad, aislamiento de transacciones). Un store en modo AP puede ofrecer transacciones acotadas a una partición (LWT de Cassandra, transacciones single-item de DynamoDB), y además el nivel AP/CP suele elegirse **por operación**, no es etiqueta fija del sistema.

**16.1** (a) Happy path: single-leader por shard + réplicas de lectura, multi-región active-passive con semi-sync, lecturas locales/escrituras al líder activo. (b) Modelo de falla: failover al follower más adelantado con fencing; promoción de la región pasiva ante caída regional; el lag tapado con sticky/LSN. (c) Recuperación: el promovido tiene los commits por semi-sync, el líder viejo fenced, reconciliación por anti-entropía si hubo divergencia.

**16.2** "Depende del RPO/RTO del negocio. Con active-passive y replicación semi-síncrona a la región pasiva, el RPO es cercano a 0 (no pierdo transacciones confirmadas) y el RTO es de minutos (lo que tarda promover y redirigir el tráfico). Promuevo la región pasiva a activa, hago fencing de la vieja, y redirijo con DNS/anycast. Si el negocio exige RTO≈0, voy a active-active y asumo resolver conflictos."

**16.3** Porque la consistencia fuerte global es cara (latencia por round-trips a quórum, se planta bajo partición) y muchas veces **sobra**: si el síntoma es "no veo mi propia escritura", read-your-writes lo tapa con una fracción del costo. Elegir la garantía mínima suficiente es lo que mantiene el sistema **disponible y rápido** sin la anomalía concreta — sobre-pedir consistencia es pagar de más y perder disponibilidad sin necesidad.
