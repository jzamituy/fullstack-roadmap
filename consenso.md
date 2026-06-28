# Consenso y coordinación distribuida a fondo

**Cómo varias máquinas se ponen de acuerdo cuando la red miente y los nodos se caen · Node + Go · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo te da los conceptos que más se prueban en los *deep-dives* de entrevistas senior/staff: **consenso (Raft)**, **quórums**, **relojes lógicos**, **detección de fallas**, **locks distribuidos con fencing**, **anti-entropía** y **consistent hashing**. Su complemento es el [laboratorio práctico](consenso-practica.md): acá entendés, allá **construís** (relojes de Lamport y vectoriales, quórum R+W>N, elección de líder estilo Raft, un lock con fencing token y un hash ring).

**Lo que asumimos.** Que ya tenés el método de [Diseño de sistemas backend](system-design.md) (CAP, sharding, replicación a alto nivel) y que viste [Concurrencia en system design](concurrencia.md) — porque coordinar máquinas es "concurrencia, pero la memoria compartida ahora es la red, y la red miente". No asumimos que sepas Go: los ejemplos en Go se explican línea por línea.

> ⚠️ **Sobre los lenguajes.** El consenso es **lógica de sistema distribuido**, no sintaxis. Los ejemplos van en **Node/TS** (tu stack) y en **Go** donde la concurrencia real (goroutines, channels, `select` con timeout) hace el código más claro que en un runtime single-thread. Ver el mismo algoritmo en los dos planos es lo que te hace sonar senior: entendés la idea, no la API de una librería.

### El marco que usamos en todo el módulo

Una entrevista de system design no se gana recitando "Raft elige un líder". Se gana **razonando fallas**. Por eso, para cada mecanismo de coordinación, lo atacamos en **cuatro capas**:

1. **Qué se rompe** exactamente.
2. **Por qué se rompe** a *esta* escala, con *esta* topología de red o *este* modo de falla (no en abstracto).
3. **Control de corto plazo** que frena el *blast radius* (el daño inmediato).
4. **Cambio de diseño de largo plazo** que baja la probabilidad de que se repita.

> **El ejemplo que fija el tono — el *split-brain*.** Respuesta débil: *"elegís un líder con Raft"*. Respuesta profunda, en cascada: **un líder se pone lento (GC pause, red saturada) → los followers no reciben heartbeat → eligen un nuevo líder con un término mayor → el viejo líder revive y todavía se cree líder → ahora hay dos nodos aceptando escrituras → escrituras divergentes que después no se pueden mergear**. Eso es ver el sistema, no la línea de código. Lo resolvemos con *terms* monotónicos y fencing; volvemos a esto en los módulos 9 y 12.

Y cuando tengas que **diseñar** algo coordinado, una buena respuesta tiene **tres cosas**: una **forma para el happy path**, un **modelo de falla**, y un **proceso de recuperación**. Lo cerramos en el módulo 19.

**Tipos de ejercicio** (para saber cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (el grueso de la implementación está en el [laboratorio](consenso-practica.md)).

**Índice de módulos**

*Etapa 1 — Por qué coordinar es difícil*
1. Fallas parciales y por qué no hay reloj global
2. Modelos de falla y de red (crash-stop, bizantino; síncrono/asíncrono)
3. Detección de fallas: heartbeats y el problema "¿caído o lento?"

*Etapa 2 — Tiempo y orden sin un reloj compartido*
4. Relojes físicos y por qué no alcanzan
5. Relojes lógicos de Lamport (happened-before)
6. Relojes vectoriales (detectar concurrencia)

*Etapa 3 — Acuerdo: consenso*
7. Qué es consenso y la máquina de estados replicada
8. Quórums (N/2+1 y R+W>N)
9. Raft I — elección de líder (terms, votos, timeout aleatorio)
10. Raft II — replicación del log (commit index, safety)
11. Raft III — casos borde (split-brain, catch-up, cambio de membresía)

*Etapa 4 — Coordinación práctica*
12. Locks distribuidos: lease + fencing tokens
13. Elección de líder en producción (single-writer, etcd/ZooKeeper)
14. Membresía y gossip (SWIM)

*Etapa 5 — Réplicas sin líder (leaderless)*
15. Quórum de lectura/escritura estilo Dynamo (R+W>N, sloppy quorum)
16. Anti-entropía: read-repair, hinted handoff y Merkle trees
17. Consistent hashing (hash ring, virtual nodes, rebalanceo)

*Etapa 6 — Trade-offs y cierre*
18. El costo de coordinar: cuándo SÍ y cuándo NO
19. Cómo hablar de consenso en una entrevista

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Etapa 1 — Por qué coordinar es difícil

## Módulo 1 — Fallas parciales y por qué no hay reloj global

**Teoría.** En un solo proceso, cuando algo falla, **falla todo**: el proceso muere y listo. En un sistema distribuido aparece el monstruo que define el campo: la **falla parcial**. Un nodo está vivo, otro está muerto, un tercero está vivo pero **no te llega el mensaje** porque la red se tragó el paquete. Y desde donde vos estás parado, **los tres se ven igual**: mandaste un mensaje y no volvió respuesta.

Eso lleva a la verdad incómoda del área: **no podés distinguir "el nodo se cayó" de "el nodo está lento" de "la red perdió tu mensaje"**. Un timeout no te dice *cuál* de las tres pasó. Solo te dice "no tuve respuesta a tiempo".

A eso se suma que **no hay un reloj global**. Cada máquina tiene su reloj físico, y esos relojes **derivan** (clock skew): dos eventos en dos máquinas pueden tener timestamps que mienten sobre cuál pasó primero. Entonces no podés ordenar eventos del sistema mirando la hora.

La frase mental: **un sistema distribuido es un montón de máquinas que solo se pueden hablar por mensajes que tardan, se pierden o llegan desordenados, sin un reloj común y sin saber si el otro está vivo.** Todo lo que sigue —relojes lógicos, consenso, quórums— son respuestas a *esa* lista de problemas.

> El resultado teórico que conviene nombrar: el teorema **FLP** (Fischer-Lynch-Paterson, 1985) dice que en un sistema **asíncrono** (sin cota de tiempo de mensajes) con **al menos un nodo que puede fallar**, **no existe** algoritmo de consenso que garantice terminar siempre. La salida práctica no es "es imposible", es "usamos timeouts y aceptamos un sistema *parcialmente* síncrono" — por eso Raft y Paxos funcionan en la vida real aunque FLP diga que en el peor caso teórico podrían no terminar.

**Ejercicios 1**
1.1 🔁 Nombrá las tres cosas que un timeout **no** te deja distinguir.
1.2 🧠 ¿Por qué no podés ordenar eventos de dos máquinas mirando sus timestamps físicos?
1.3 🧠 FLP dice que el consenso es imposible en sistemas asíncronos con fallas. ¿Por qué entonces Raft funciona en producción?

---

## Módulo 2 — Modelos de falla y de red

**Teoría.** Antes de diseñar nada, hay que declarar **contra qué fallas** te protegés. No es lo mismo "un nodo se apaga" que "un nodo miente". Los modelos de falla, de menos a más feo:

- **Crash-stop:** el nodo funciona bien hasta que se cae, y cuando se cae, se queda caído. El más simple.
- **Crash-recovery:** el nodo se cae y **vuelve** (reinicio), posiblemente perdiendo lo que tenía en memoria pero conservando lo que escribió en disco. Es el modelo realista para Raft/etcd.
- **Omisión:** el nodo (o la red) **pierde** mensajes —algunos llegan, otros no— pero los que llegan son correctos.
- **Bizantino:** el nodo puede hacer **cualquier cosa**: mandar mensajes contradictorios a distintos pares, mentir, corromper datos. Es el modelo de blockchains y aviónica. Tolerarlo cuesta caro (necesitás `3f+1` nodos para tolerar `f` traidores, vs `2f+1` para crash). **La mayoría de los sistemas de backend NO son bizantinos**: asumís crash-recovery y confiás en que tus propios nodos no mienten.

Y el **modelo de red**, que es lo que cambia si el consenso es posible:

- **Síncrono:** hay una cota conocida del tiempo de mensaje. Irreal en una red de verdad.
- **Asíncrono:** los mensajes tardan lo que tardan, sin cota. Es donde pega FLP.
- **Parcialmente síncrono:** la red **a veces** se porta bien (hay cotas) y a veces no. Es el modelo real, y es donde Raft/Paxos garantizan **safety siempre** y **liveness cuando la red se estabiliza**.

> **La distinción que te separa de un junior:** safety vs liveness. **Safety** = "nunca pasa algo malo" (nunca hay dos líderes en el mismo término, nunca se commitea un valor que se pierde). **Liveness** = "eventualmente pasa algo bueno" (eventualmente se elige un líder y el sistema avanza). Raft **nunca** sacrifica safety; lo que sacrifica bajo partición es liveness (puede quedarse sin poder avanzar hasta que la red mejore). Saber qué garantía cede un sistema bajo falla es la pregunta de fondo de CAP.

**Ejercicios 2**
2.1 🔁 Diferencia entre crash-stop y crash-recovery.
2.2 🧠 ¿Por qué tolerar fallas bizantinas necesita más nodos que tolerar crashes? ¿Cuántos nodos para tolerar `f` de cada tipo?
2.3 🧠 Un sistema "elige CP" en CAP. En términos de safety/liveness, ¿qué está priorizando y qué está cediendo bajo partición?

---

## Módulo 3 — Detección de fallas: heartbeats y "¿caído o lento?"

**Teoría.** Como no podés *saber* si un nodo está vivo, lo **estimás**: un **detector de fallas** marca un nodo como sospechoso si no da señales de vida a tiempo. El mecanismo base es el **heartbeat**: cada nodo manda "estoy vivo" cada `T` ms; si no recibís `k` heartbeats seguidos, lo declarás caído.

Acá aparece el trade-off central, y es el del módulo 1 disfrazado: **¿qué timeout poné?**

- **Timeout corto:** detectás caídas rápido (buena disponibilidad), pero **te equivocás más**: un nodo que solo estaba lento (GC pause, pico de red) lo declarás muerto. Eso es un **falso positivo**, y dispara un *failover* innecesario.
- **Timeout largo:** menos falsos positivos, pero **tardás** en reaccionar a una caída real (el sistema queda degradado más tiempo).

> 🔥 **Falla en 4 capas — el falso positivo que dispara un failover en cascada.**
> 1. **Qué se rompe:** declarás caído a un líder que solo estaba lento; promovés a otro; el viejo revive → dos líderes (split-brain) o, peor, un *flapping* (el líder cambia una y otra vez).
> 2. **Por qué a esta escala:** con timeout agresivo y una pausa de GC de 2s (normal bajo carga), el detector dispara; cuanto más cargado el sistema, más pausas, más failovers, y cada failover **agrega** carga (re-elección, re-sincronización de logs) → más pausas. La recuperación se volvió amplificador.
> 3. **Control de corto plazo:** subir el timeout / exigir `k` heartbeats perdidos seguidos; **fencing** para que el viejo líder no pueda escribir aunque se crea vivo (módulo 12).
> 4. **Largo plazo:** detectores adaptativos como **φ-accrual** (no un sí/no binario, sino un *nivel de sospecha* que sube con la tardanza y se calibra con el histograma real de latencias); leases en vez de heartbeats puros (módulo 13).

El **φ-accrual failure detector** (usado por Cassandra y Akka) es la respuesta madura: en vez de un umbral fijo, calcula la probabilidad de que el nodo esté caído dado el patrón histórico de llegada de heartbeats, y vos elegís el umbral φ según cuánto te duele un falso positivo.

**Ejercicios 3**
3.1 🔁 ¿Qué es un falso positivo en detección de fallas y qué lo dispara?
3.2 🧠 Tenés un servicio con pausas de GC de hasta 1.5s. ¿Qué problema trae un heartbeat con timeout de 1s y cómo lo mitigás sin volverte ciego a caídas reales?
3.3 🧠 ¿Qué le agrega un detector φ-accrual sobre un umbral fijo de "3 heartbeats perdidos"?

---

## Etapa 2 — Tiempo y orden sin un reloj compartido

## Módulo 4 — Relojes físicos y por qué no alcanzan

**Teoría.** Cada máquina tiene un reloj de cuarzo que **deriva**: corre un poquito más rápido o más lento que el real. Lo corregís con **NTP** (Network Time Protocol), que sincroniza contra servidores de tiempo, pero NTP tiene su propio error (decenas de ms en la práctica, a veces más) y, peor, puede hacer **saltar el reloj hacia atrás** cuando corrige una deriva. Un reloj que retrocede es una bomba: timestamps duplicados, TTLs que se rompen, ordenamientos que mienten.

Por eso, para **medir duraciones** se usa un **reloj monotónico** (el que nunca retrocede, p. ej. `process.hrtime` en Node, `time.monotonic` en otros), y el reloj "de pared" (wall clock) **solo** para mostrarle la hora a un humano.

¿Y para **ordenar eventos** entre máquinas? El wall clock no sirve: con skew de 100ms, un evento que pasó *después* en la máquina B puede tener timestamp *menor* que uno de la máquina A. Si decidís un conflicto con "gana el timestamp más alto" (**Last-Write-Wins**), estás dejando que la deriva de relojes decida quién gana — y podés **perder escrituras** silenciosamente.

> Google **TrueTime** (Spanner) ataca esto con relojes atómicos + GPS que acotan el error a un intervalo `[earliest, latest]` y **esperan** ese intervalo antes de commitear, garantizando orden externo. Es la excepción que confirma la regla: hace falta hardware especial y esperar a propósito para que el wall clock sea confiable a escala global. Sin eso, **no ordenes eventos distribuidos por reloj físico**.

**Ejercicios 4**
4.1 🔁 ¿Por qué usás un reloj monotónico para medir una duración y no `Date.now()`?
4.2 🧠 Mostrá con un ejemplo cómo Last-Write-Wins por timestamp físico puede **perder** una escritura.
4.3 🧠 ¿Qué problema resuelve TrueTime y por qué la mayoría de los sistemas no lo tienen?

---

## Módulo 5 — Relojes lógicos de Lamport

**Teoría.** Si el reloj físico no sirve para ordenar, usá un **reloj lógico**: un contador que no mide segundos, mide **causalidad**. Lamport (1978) define la relación **happened-before** (`→`): el evento `a` ocurrió-antes que `b` si pasaron en el mismo nodo en ese orden, o si `a` es el envío de un mensaje y `b` su recepción. Si ni `a → b` ni `b → a`, los eventos son **concurrentes**.

El **reloj de Lamport** es un entero por nodo con dos reglas:

1. Antes de cada evento local, incrementás tu contador: `L = L + 1`.
2. Al **enviar** un mensaje, le pegás tu `L`. Al **recibir** un mensaje con timestamp `Lm`, hacés `L = max(L, Lm) + 1`.

La garantía: si `a → b`, entonces `L(a) < L(b)`. Es decir, **respeta la causalidad**: una causa siempre tiene timestamp menor que su efecto.

> ⚠️ **El matiz que se equivoca el 90%:** la **vuelta NO vale**. Que `L(a) < L(b)` **no** implica que `a → b`; pueden ser concurrentes y tener timestamps distintos por casualidad. El reloj de Lamport da un **orden total** (rompés empates con el id del nodo) que es **consistente con la causalidad**, pero **no te deja detectar si dos eventos fueron concurrentes**. Para eso necesitás vector clocks (módulo 6).

¿Para qué sirve entonces? Para cualquier cosa que necesite un **orden total consistente con causalidad** sin pretender detectar conflictos: asignar números de versión, ordenar operaciones en un log, romper empates de forma determinística en todos los nodos.

**Ejercicios 5**
5.1 🔁 Escribí las dos reglas del reloj de Lamport (evento local y recepción de mensaje).
5.2 🧠 Si `L(a) < L(b)`, ¿podés afirmar que `a` causó `b`? Justificá.
5.3 ✍️ Dos nodos, eventos: A1, A2 en el nodo A; B1 en B. A2 envía un mensaje que B recibe como B1. Calculá los timestamps de Lamport. (Ampliado en el [laboratorio](consenso-practica.md).)

---

## Módulo 6 — Relojes vectoriales

**Teoría.** Un **vector clock** es la respuesta al límite de Lamport: en vez de un solo entero, cada nodo mantiene un **vector** con un contador **por nodo**. El nodo `i` tiene `V[i]` y lleva su idea de cuántos eventos vio de cada otro nodo.

Reglas:
1. En un evento local, el nodo `i` incrementa **su** posición: `V[i] += 1`.
2. Al enviar, manda todo el vector. Al recibir `Vm`, hace `V[j] = max(V[j], Vm[j])` para todo `j`, y después incrementa su propia posición.

Para comparar dos vectores `V` y `W`:
- `V < W` (V ocurrió-antes) si `V[i] ≤ W[i]` para **todo** `i` y es estrictamente menor en al menos uno.
- Si ni `V < W` ni `W < V` → son **concurrentes** (cada uno tiene algo que el otro no vio). **Esto es lo que Lamport no podía decirte.**

> Esa capacidad de detectar concurrencia es la base de la resolución de conflictos en bases leaderless (Dynamo, Riak): cuando dos escrituras tienen vectores concurrentes, el sistema sabe que **chocaron de verdad** (no es que una sea más vieja) y guarda **ambas versiones** (*siblings*) para que la app o un CRDT las resuelva. Lo retomamos en [Replicación y conflictos](system-design.md) y en el banco de casos.

El costo: el vector crece con la cantidad de nodos que escriben, y hay que podarlo (version vectors / dotted version vectors en sistemas reales). Pero conceptualmente, **vector clock = el reloj que sí distingue "viejo" de "concurrente"**.

| | Lamport | Vector clock |
|---|---|---|
| Estructura | Un entero | Un entero por nodo |
| Detecta orden causal | Sí (en una dirección) | Sí |
| Detecta **concurrencia** | **No** | **Sí** |
| Costo | O(1) | O(nodos) |
| Uso típico | Orden total de un log | Resolver conflictos en réplicas |

**Ejercicios 6**
6.1 🔁 ¿Qué puede decir un vector clock que un reloj de Lamport no?
6.2 🧠 Dados `V = [2,1,0]` y `W = [2,0,1]` (nodos A,B,C), ¿qué relación hay entre los eventos? ¿Qué implica para una base de datos?
6.3 🧠 ¿Cuál es el costo de los vector clocks a medida que crece el cluster y cómo lo acotan los sistemas reales?

---

## Etapa 3 — Acuerdo: consenso

## Módulo 7 — Qué es consenso y la máquina de estados replicada

**Teoría.** **Consenso** es el problema de que un conjunto de nodos **se pongan de acuerdo en un valor** (o en una secuencia de valores), aun cuando algunos fallen y la red se porte mal. Un algoritmo de consenso correcto cumple tres propiedades:

- **Acuerdo (agreement):** dos nodos no deciden valores distintos. *(safety)*
- **Validez (validity):** el valor decidido fue propuesto por alguien (no se inventa). *(safety)*
- **Terminación (termination):** eventualmente todos deciden. *(liveness — la que FLP dice que no podés garantizar siempre)*

¿Para qué querés acuerdo? Para construir una **máquina de estados replicada (RSM)**: la idea más importante del módulo. Si **todas** las réplicas arrancan en el mismo estado y aplican **exactamente la misma secuencia de comandos en el mismo orden**, terminan en el mismo estado. El problema de "mantener N réplicas idénticas" se reduce entonces a **un solo problema**: ponerse de acuerdo en el **orden del log de comandos**. Eso es lo que hacen Raft, Paxos y Zab: acuerdan un log replicado, ordenado e idéntico.

> La frase mental: **consenso = acordar el orden de un log; replicar estado = reproducir ese log.** Una base de datos replicada, un scheduler de Kubernetes, un sistema de locks: todos son una RSM por encima de un log consensuado. Por eso etcd (que usa Raft) está en el corazón de Kubernetes — guarda el estado del cluster como un log replicado.

**Ejercicios 7**
7.1 🔁 Nombrá las tres propiedades del consenso y marcá cuál es de liveness.
7.2 🧠 Explicá por qué "mantener N réplicas idénticas" se reduce a "acordar el orden de un log".
7.3 🧠 ¿Por qué la validez importa? Dá un ejemplo de un algoritmo que cumpla acuerdo y terminación pero viole validez (y por qué es inútil).

---

## Módulo 8 — Quórums (N/2+1 y R+W>N)

**Teoría.** Un **quórum** es "la cantidad mínima de nodos que tienen que estar de acuerdo para que una decisión cuente". La magia está en una sola propiedad: **dos quórums siempre se solapan en al menos un nodo**. Ese solapamiento es lo que evita que dos decisiones contradictorias pasen "por separado".

El quórum más común es la **mayoría**: `N/2 + 1` (con `N=5`, hace falta 3). Dos mayorías de 5 comparten sí o sí un nodo (3+3 > 5). Por eso Raft y Paxos usan mayorías: una elección de líder necesita mayoría de votos, así que **no puede haber dos líderes** con mayoría en el mismo término. Y por eso los clusters de consenso van en **números impares** (3, 5, 7): con 4 nodos tolerás los mismos fallos que con 3 (uno), pero arriesgás más empates — el número par no compra nada.

En sistemas **leaderless estilo Dynamo** el quórum se parte en dos parámetros que vos elegís:

- **N** = cantidad de réplicas de cada dato.
- **W** = cuántas tienen que confirmar una **escritura**.
- **R** = cuántas tenés que leer.

La regla de oro: **si `R + W > N`, toda lectura se solapa con la última escritura** → leés el dato fresco (consistencia fuerte de lectura). Si `R + W ≤ N`, podés leer una réplica que se perdió la última escritura → **lectura stale**.

| Config (N=3) | Garantía | Para qué |
|---|---|---|
| W=3, R=1 | Lectura siempre fresca, escritura cara y frágil | Lecturas dominantes, escrituras raras |
| W=1, R=3 | Escritura rápida, lectura cara | Escrituras dominantes |
| **W=2, R=2** | `R+W=4 > 3` → fresca y balanceada | El default sano |
| W=1, R=1 | `R+W=2 ≤ 3` → rápido pero **stale posible** | Máxima disponibilidad, tolera stale |

> El quórum es **el mismo principio** en los dos mundos: en consenso (Raft) es interno y fijo (mayoría); en leaderless (Dynamo) es un dial que vos girás por operación. Lo construís en el [laboratorio](consenso-practica.md).

**Ejercicios 8**
8.1 🔁 ¿Cuál es la propiedad clave de los quórums por mayoría que evita dos líderes?
8.2 🧠 ¿Por qué los clusters de Raft se despliegan con un número impar de nodos?
8.3 🧠 Con `N=3`, elegís `W=2, R=1`. ¿Podés leer un valor stale? Mostralo con un escenario concreto. ¿Y con `R=2`?

---

## Módulo 9 — Raft I: elección de líder

**Teoría.** Raft (Ongaro & Ousterhout, 2014) se diseñó para ser **entendible** (a diferencia de Paxos). Su idea organizadora: en vez de que todos los nodos negocien cada valor, **elegí un líder** y que él mande. Todo pasa por el líder; los followers solo copian. Si el líder se cae, se elige otro.

El tiempo se parte en **términos (terms)**: enteros que crecen, cada uno con **a lo sumo un líder**. El término es un reloj lógico de Lamport disfrazado, y es la pieza de safety central. Cada nodo está en uno de tres roles: **follower**, **candidate**, **leader**.

El baile de la elección:
1. Todos arrancan como **follower**. Si un follower no recibe heartbeat del líder en su **election timeout**, sospecha que no hay líder.
2. Se vuelve **candidate**, incrementa el término, **vota por sí mismo** y pide votos a los demás (`RequestVote`).
3. Cada nodo vota **una sola vez por término**, y solo si el candidato está **al menos tan actualizado** como él (su log no es más viejo — esto es clave para la safety del módulo 11).
4. Si el candidato junta **mayoría** de votos → es líder y empieza a mandar heartbeats. Si nadie junta mayoría (**split vote**), el término muere sin líder y se reintenta.

> **El truco que evita que el split vote se repita para siempre: el election timeout ALEATORIO.** Si todos los followers usaran el mismo timeout, se volverían candidatos a la vez una y otra vez, partiendo el voto eternamente (un livelock — ver [concurrencia](concurrencia.md), módulo 14). Raft le da a cada nodo un timeout **aleatorio** en un rango (p. ej. 150–300ms): casi siempre uno arranca primero, junta su mayoría y corta antes de que los otros despierten. **El jitter desincroniza** — exactamente la misma idea que rompe el retry storm. Lo ves fallar y arreglarse en el [laboratorio](consenso-practica.md).

> 🔥 **Falla en 4 capas — split-brain por término viejo.**
> 1. **Qué se rompe:** un líder viejo, particionado de la mayoría, sigue creyéndose líder y aceptando escrituras que **nunca se commitearán** (o peor, si no hay fencing, corrompe un recurso externo).
> 2. **Por qué a esta escala:** una partición de red deja al líder en la minoría; la mayoría elige uno nuevo con término mayor; el viejo no se enteró.
> 3. **Control de corto plazo:** Raft ya lo resuelve para sus propias escrituras — el viejo líder, al contactar a cualquiera con término mayor, **se entera de que quedó obsoleto y se degrada a follower**; sus escrituras del término viejo nunca juntaron mayoría, así que no se commitean. El número de término es la barrera.
> 4. **Largo plazo:** para efectos **fuera** de Raft (escribir en S3, tomar un lock externo) el término no alcanza → necesitás **fencing tokens** (módulo 12), porque Raft solo protege su propio log.

**Ejercicios 9**
9.1 🔁 ¿Qué es un *term* en Raft y cuántos líderes puede haber por término?
9.2 🧠 ¿Por qué el election timeout es aleatorio y no fijo? ¿Qué patología evita?
9.3 🧠 Un líder queda en la minoría de una partición y sigue aceptando escrituras. ¿Por qué esas escrituras no se commitean, y por qué eso NO alcanza para un efecto externo como tomar un lock?

---

## Módulo 10 — Raft II: replicación del log

**Teoría.** Ya hay líder. Ahora **replica comandos**. Cada entrada del log lleva `(term, índice, comando)`. El flujo del happy path:

1. El cliente le manda un comando al líder.
2. El líder lo **agrega a su log** (sin aplicarlo todavía) y lo manda a los followers con `AppendEntries`.
3. Cuando una **mayoría** confirmó que escribió la entrada en su log, el líder la marca como **committed**, **la aplica a su máquina de estados** y le responde OK al cliente.
4. En los siguientes `AppendEntries` (incluidos los heartbeats), el líder avisa el `commitIndex`, y los followers aplican hasta ahí.

Dos punteros importan: `commitIndex` (hasta dónde es seguro aplicar) y, por follower, `matchIndex` (hasta dónde sé que ese follower copió). `AppendEntries` también es el heartbeat: si no hay comandos, el líder manda un `AppendEntries` vacío para decir "sigo vivo y este es el commitIndex".

> **Por qué "mayoría confirmó" = durable:** si la entrada está en una mayoría y el líder se cae, **cualquier** líder nuevo necesita mayoría de votos, y esa mayoría **se solapa** con la mayoría que tiene la entrada → el nuevo líder, por la regla de votación (módulo 9, paso 3), va a tener esa entrada. El quórum del módulo 8 es lo que hace que un commit sobreviva a la caída del líder.

`AppendEntries` también **repara** logs divergentes: lleva `(prevLogIndex, prevLogTerm)`; si no matchean en el follower, el follower **rechaza** y el líder retrocede el índice hasta encontrar el punto donde coinciden, y desde ahí **sobrescribe** lo que el follower tenía de más. El log del líder es la verdad.

> ⚠️ El líder **solo puede commitear entradas de su propio término** directamente. Una entrada vieja (de un término anterior) se commitea **indirectamente**, recién cuando se commitea una entrada del término actual por encima. Es un caso borde sutil de Raft que evita un escenario donde una entrada "replicada en mayoría" se pierde igual. No hace falta memorizar la prueba, sí saber que existe.

**Ejercicios 10**
10.1 🔁 ¿En qué momento exacto el líder aplica un comando a su máquina de estados y le responde OK al cliente?
10.2 🧠 Explicá por qué "replicado en una mayoría" implica que el commit sobrevive a la caída del líder.
10.3 🧠 Un follower tiene su log divergente del líder. ¿Cómo lo reconcilia `AppendEntries` y quién gana?

---

## Módulo 11 — Raft III: casos borde y seguridad

**Teoría.** Lo que hace a Raft *correcto* (no solo plausible) son las restricciones de safety. Las tres que conviene tener en la cabeza:

- **Election restriction (la más importante):** un nodo solo vota por un candidato cuyo log sea **al menos tan actualizado** como el suyo ("up-to-date" = término mayor en la última entrada, o mismo término pero log más largo). Esto garantiza que **un líder nuevo siempre tiene todas las entradas ya committeadas** — no podés elegir un líder que se haya perdido un commit, porque no juntaría mayoría.
- **Leader append-only:** el líder **nunca** borra ni pisa entradas de su propio log; solo agrega. La divergencia se arregla en los followers, no en el líder.
- **Log matching:** si dos logs tienen una entrada con el mismo índice y término, entonces **todas** las entradas anteriores son idénticas. (Lo garantiza el chequeo `prevLogIndex/prevLogTerm`.)

Dos operaciones de producción que todo sistema Raft necesita:

- **Snapshots / log compaction:** el log no puede crecer para siempre. Cada tanto, cada nodo toma un **snapshot** del estado de la máquina y descarta el log anterior a ese punto. Un follower muy atrasado recibe el snapshot entero (`InstallSnapshot`) en vez de replays infinitos.
- **Cambio de membresía (reconfiguración):** agregar o sacar nodos **en caliente** es peligroso — si pasás de la config vieja a la nueva de golpe, puede haber un instante con **dos mayorías disjuntas** (split-brain de configuración). Raft lo resuelve con **joint consensus** (una fase intermedia donde se requiere mayoría de **ambas** configuraciones a la vez) o cambiando **de a un nodo** (single-server changes), que es lo que hace etcd.

> El mensaje senior: Raft no es "elegí un líder y replicá". Es "elegí un líder **de forma que nunca pierda un commit**, replicá **de forma que la mayoría sea durable**, y cambiá la membresía **sin abrir una ventana de dos mayorías**". Las restricciones *son* el algoritmo.

**Ejercicios 11**
11.1 🔁 ¿Qué garantiza la *election restriction* (que solo votás por logs al menos tan actualizados como el tuyo)?
11.2 🧠 ¿Por qué hacen falta snapshots y qué problema traería no tenerlos?
11.3 🧠 ¿Qué puede salir mal si cambiás la membresía del cluster de la config vieja a la nueva "de golpe", y cómo lo evita Raft?

---

## Etapa 4 — Coordinación práctica

## Módulo 12 — Locks distribuidos: lease + fencing tokens

**Teoría.** Querés que **un solo** worker procese un job, o que **un solo** nodo sea el que escribe. La tentación es un "lock distribuido" en Redis/etcd: `SET lock NX EX 30`. Funciona... hasta que no.

El problema es el del módulo 1: **no podés saber si el que tiene el lock sigue vivo**. Si le das al lock un TTL (lease), evitás que un worker muerto se quede el lock para siempre — pero abrís otra ventana:

> 🔥 **Falla en 4 capas — el lock que dos clientes creen tener (el caso de Kleppmann).**
> 1. **Qué se rompe:** el cliente A toma el lock (TTL 30s), arranca a escribir, pero se **congela** justo después (GC pause largo, el SO lo desaloja). Pasan 30s, el lease **expira**, el cliente B toma el lock y empieza a escribir. A **despierta** sin saber que pasó el tiempo, se cree dueño del lock, y escribe encima. **Dos escritores. Corrupción.**
> 2. **Por qué a esta escala:** las pausas de GC de varios segundos son **normales** bajo carga; el TTL del lock no congela el reloj del mundo. Cuanto más cargado el sistema, más probable la pausa larga justo en la sección crítica.
> 3. **Control de corto plazo:** TTL más largo (reduce la probabilidad, no la elimina) y chequear el lock seguido (tampoco alcanza: la pausa puede caer entre el chequeo y la escritura).
> 4. **Largo plazo — FENCING TOKENS:** el servicio de lock devuelve, junto al lock, un **número monotónicamente creciente** (un token). Cada vez que escribís al recurso protegido, mandás el token. El recurso **rechaza** cualquier escritura con un token **menor** que el último que vio. Cuando B toma el lock, recibe un token mayor que el de A; cuando A despierta y escribe con su token viejo, el recurso lo **rechaza**. El recurso, no el lock, es el que hace cumplir la exclusión.

La lección que se cita en toda entrevista: **un lock distribuido no es seguro por sí solo; necesita que el recurso protegido valide un fencing token.** Si el recurso no puede validar un token (no tenés control de él), entonces el lock es *best-effort* y tu diseño tiene que tolerar dos dueños. Lo construís en el [laboratorio](consenso-practica.md).

> Por eso para locks "de verdad" (correctitud, no solo eficiencia) se usa **etcd/ZooKeeper** (consenso real + leases + revocación), no un `SET NX` en un solo Redis. Redlock (Redis multi-nodo) sigue sin resolver el problema de la pausa sin fencing — es la crítica famosa de Kleppmann. Para *eficiencia* (evitar trabajo duplicado, no correctitud), un lock best-effort en Redis está bien.

**Ejercicios 12**
12.1 🔁 ¿Qué es un fencing token y quién lo valida?
12.2 🧠 Contá el escenario de los dos dueños del lock paso a paso. ¿En qué momento exacto se rompe?
12.3 🧠 ¿Cuándo un lock best-effort en Redis (sin fencing) es aceptable y cuándo no? Distinguí "eficiencia" de "correctitud".

---

## Módulo 13 — Elección de líder en producción

**Teoría.** Muchas veces no querés montar tu propio Raft; querés **el resultado**: "que haya exactamente un nodo haciendo X". El patrón **single-writer** (un solo líder escribe, los demás esperan) elimina de un saque una clase entera de conflictos — es la forma más barata de consistencia. La pregunta es cómo elegir ese único.

En la práctica delegás el consenso a un sistema que ya lo resolvió:

- **etcd** (Raft) o **ZooKeeper** (Zab): exponen primitivas de **lease** y de **nodos efímeros**. El patrón: cada candidato crea una clave con un lease que tiene que **renovar** (keep-alive); el que la tiene es el líder; si se cae y no renueva, el lease expira y otro la toma. ZooKeeper tiene además el patrón de **ephemeral sequential nodes** para una cola de espera justa de candidatos.
- **Lease vs lock:** un **lease** es un lock con **vencimiento automático**. Es lo que querés en distribuido: si el dueño muere, el recurso se libera solo (no queda trabado para siempre). El precio es el del módulo 12 — el lease puede expirar mientras el dueño está vivo-pero-pausado → fencing.

> La conexión con todo lo anterior: la elección de líder **es** un caso de consenso (acordar quién es el líder), por eso etcd/ZooKeeper —que son RSMs sobre Raft/Zab— son la herramienta correcta, y un `SET NX` en un Redis single-node **no** lo es (un solo Redis es un SPOF y no da las garantías de un quórum). Cuando en una entrevista te pidan "elegí un líder", la respuesta es "lease renovable sobre etcd/ZooKeeper + fencing token en el recurso", no "un flag en una tabla".

**Ejercicios 13**
13.1 🔁 ¿Qué diferencia hay entre un lock y un lease?
13.2 🧠 ¿Por qué el patrón single-writer es "la forma más barata de consistencia"?
13.3 🧠 ¿Por qué etcd/ZooKeeper son la herramienta correcta para elegir líder y un Redis single-node no?

---

## Módulo 14 — Membresía y gossip (SWIM)

**Teoría.** Hasta acá los nodos sabían quiénes eran sus pares. ¿Pero cómo un cluster de 1000 nodos mantiene **quién está vivo** sin que todos hagan heartbeat contra todos (`O(N²)` mensajes, que no escala)? Con **gossip**: cada nodo le cuenta a unos pocos vecinos al azar lo que sabe, y la información **se propaga como un rumor** (epidémicamente), llegando a todos en `O(log N)` rondas.

El protocolo canónico es **SWIM** (Scalable Weakly-consistent Infection-style Membership), usado por Consul, Serf y derivados:

- **Detección de fallas:** cada nodo, cada tanto, le hace *ping* a **un** par al azar. Si no responde, no lo declara muerto de una — pide a **otros `k` nodos** que le hagan ping también (*indirect probing*). Recién si nadie lo alcanza, lo marca **sospechoso**, lo difunde, y tras un tiempo, **muerto**. Esto baja muchísimo los falsos positivos del módulo 3: una pérdida de paquete puntual no mata a un nodo sano.
- **Diseminación:** los cambios de membresía (se unió X, murió Y) viajan **piggyback** en los mismos pings de gossip, sin un canal aparte.

> Gossip te da **consistencia débil** de la membresía: no todos los nodos ven el cambio al mismo tiempo, pero **convergen** rápido. Es el trade-off correcto para *membership* y *anti-entropía* (módulo 16), donde no necesitás acuerdo instantáneo, necesitás que **eventualmente** todos sepan lo mismo, y que escale a miles de nodos. **Gossip NO es consenso:** no decide un valor único, solo difunde un estado que converge. Usás Raft cuando necesitás acuerdo fuerte (quién es el líder, el orden del log) y gossip cuando necesitás difusión escalable y eventual (quién está vivo).

**Ejercicios 14**
14.1 🔁 ¿Por qué heartbeat de todos contra todos no escala, y qué complejidad de propagación da gossip?
14.2 🧠 ¿Qué agrega el *indirect probing* de SWIM sobre un ping directo, en términos del módulo 3?
14.3 🧠 ¿Cuándo usás gossip y cuándo Raft? Dá un ejemplo de cada uno en un mismo sistema.

---

## Etapa 5 — Réplicas sin líder (leaderless)

## Módulo 15 — Quórum de lectura/escritura estilo Dynamo

**Teoría.** Raft tiene un líder: todo pasa por él, lo que da consistencia fuerte pero el líder es un cuello y un punto de re-elección. El estilo **leaderless** (Dynamo, Cassandra, Riak) tira el líder: **cualquier** réplica acepta lecturas y escrituras, y la consistencia se controla con los quórums `R`/`W`/`N` del módulo 8.

El flujo de una escritura: el cliente (o un coordinador) manda la escritura a **las N réplicas** del dato (definidas por consistent hashing, módulo 17) y espera **W** confirmaciones. La lectura va a las réplicas y espera **R** respuestas; si vuelven versiones distintas, se queda con la más nueva (por versión/vector clock) y **repara** las viejas (read-repair, módulo 16).

Dos conceptos que distinguen al que estudió:

- **`R + W > N` da lectura fresca**, pero **no** es lo mismo que la linealizabilidad de Raft: sin coordinación, dos escrituras concurrentes pueden chocar y necesitás vector clocks + resolución de conflictos (módulo 6). El quórum te da "leés la última *committeada*", no "hay un único orden total de todo".
- **Sloppy quorum + hinted handoff:** si las N réplicas "naturales" de un dato están caídas/inalcanzables durante una partición, Dynamo no rechaza la escritura: la guarda en **otras** réplicas vivas con una **nota (hint)** de a quién pertenece. Cuando las réplicas naturales vuelven, esos nodos les **entregan** los datos guardados (hinted **handoff**). Sube la disponibilidad de escritura (elegís A de CAP) al precio de que durante la partición el quórum es "flojo" y podés leer stale.

> El eje mental: Raft = **CP** (consistente, se planta bajo partición). Dynamo-style con sloppy quorum = sintonizable hacia **AP** (disponible, acepta stale y resuelve después). No es "uno mejor que otro": es qué le duele más a tu caso, perder disponibilidad o servir un dato viejo. Esa frase es la respuesta a la mitad de las preguntas de CAP.

**Ejercicios 15**
15.1 🔁 En un sistema leaderless, ¿quién acepta escrituras y cómo se controla la consistencia?
15.2 🧠 ¿Por qué `R+W>N` "da lectura fresca" pero NO equivale a la linealizabilidad de Raft?
15.3 🧠 Explicá sloppy quorum + hinted handoff y qué letra de CAP está priorizando.

---

## Módulo 16 — Anti-entropía: read-repair, hinted handoff y Merkle trees

**Teoría.** En leaderless, las réplicas **divergen**: una se perdió una escritura (estaba caída), otra recibió un hint. **Anti-entropía** es el conjunto de mecanismos que las hace **converger** de nuevo. Tres, de oportunista a sistemático:

1. **Read-repair (oportunista):** cuando una lectura con quórum `R` detecta que una réplica devolvió una versión vieja, el coordinador le **escribe de vuelta** la versión nueva. Repara *lo que se lee*, gratis, en el camino de la lectura. Lo que nadie lee, no se repara así.
2. **Hinted handoff (durante/post-falla):** el del módulo 15 — los datos guardados "de prestado" durante una caída se entregan a su dueño cuando vuelve. Repara la divergencia causada por nodos temporalmente caídos.
3. **Anti-entropía de fondo con Merkle trees (sistemático):** para reparar **todo**, incluso lo que nadie lee, dos réplicas tienen que comparar **rangos enteros** de datos sin transferirlos completos. Un **Merkle tree** es un árbol de hashes: las hojas son hashes de bloques de datos, cada nodo interno es el hash de sus hijos. Para comparar dos réplicas, intercambian **la raíz**: si coincide, **todo el subárbol es idéntico** y no hace falta transferir nada. Si difiere, bajan por las ramas que difieren hasta encontrar **exactamente** los bloques distintos. Reconcilian transfiriendo solo eso, en `O(log n)` en vez de comparar todo el dataset.

> Los Merkle trees son la misma idea que usa Git (para comparar árboles de commits) y las blockchains (para probar que una transacción está en un bloque sin bajar el bloque entero). El concepto reutilizable: **comparar dos conjuntos grandes que casi siempre son iguales, transfiriendo solo lo que difiere.** Si en una entrevista te preguntan "cómo sincronizás dos réplicas de un TB que difieren en 3 registros sin mandar el TB", la respuesta es Merkle tree.

**Ejercicios 16**
16.1 🔁 Nombrá los tres mecanismos de anti-entropía de oportunista a sistemático.
16.2 🧠 ¿Qué limitación tiene read-repair que obliga a tener también anti-entropía de fondo?
16.3 🧠 Explicá cómo dos réplicas comparan un dataset de 1 TB que difiere en 3 registros sin transferir el TB.

---

## Módulo 17 — Consistent hashing

**Teoría.** Tenés N réplicas/shards y querés repartir claves entre ellas. La forma ingenua: `nodo = hash(clave) % N`. Funciona... hasta que **cambiás N**. Si agregás o sacás un nodo, el `% N` cambia para **casi todas** las claves → **rehash masivo**: casi todo el dataset se tiene que mover de nodo. A escala, eso es un apocalipsis de tráfico y un cache que se invalida entero.

**Consistent hashing** lo arregla. La idea: hasheá tanto las **claves** como los **nodos** en el mismo espacio circular (el **ring**, p. ej. 0 a 2³²−1). Una clave pertenece al **primer nodo que encontrás girando en sentido horario** desde su posición. Cuando agregás un nodo, **solo** las claves entre el nodo nuevo y su predecesor se mueven; el resto **no se toca**. Sacar un nodo mueve solo *sus* claves al siguiente. El movimiento es `~1/N` del dataset, no todo.

> ⚠️ **El problema que tiene el ring ingenuo y la solución que SÍ se usa — virtual nodes.** Con pocos nodos físicos, el reparto en el ring es **desbalanceado** (un nodo puede quedar a cargo de un arco enorme → *hot node*), y cuando un nodo se cae, **toda** su carga cae en su único vecino. La solución: cada nodo físico se coloca en el ring en **muchas** posiciones (vnodes — cientos por nodo, con hashes distintos). Así (a) la carga se reparte parejo (ley de los grandes números), y (b) cuando un nodo se cae, sus vnodes están dispersos, así que su carga se redistribuye entre **muchos** nodos, no uno solo. Esto es lo que hacen Cassandra, DynamoDB y los caches consistentes. Lo construís en el [laboratorio](consenso-practica.md) y verás el rehash masivo del `% N` con tus ojos.

Consistent hashing es el **building block** que conecta esta etapa: define *qué N réplicas* tiene cada dato (módulo 15), y por ende a quién le hace handoff y con quién corre anti-entropía (módulo 16).

**Ejercicios 17**
17.1 🔁 ¿Qué pasa con `hash(clave) % N` cuando cambiás N, y por qué es un problema?
17.2 🧠 ¿Cuánto del dataset se mueve al agregar un nodo en un ring consistente, y por qué?
17.3 🧠 ¿Qué dos problemas del ring ingenuo resuelven los virtual nodes?

---

## Etapa 6 — Trade-offs y cierre

## Módulo 18 — El costo de coordinar: cuándo SÍ y cuándo NO

**Teoría.** El error de quien recién aprendió consenso es **meterlo en todos lados**. Coordinar **cuesta**: cada decisión por consenso es al menos un round-trip a una mayoría (latencia), el cluster tiene un techo de throughput (todo pasa por el líder), y bajo partición **se planta** (cede liveness). Un senior sabe **evitar** coordinación tanto como aplicarla.

La jerarquía, de barato a caro:

1. **No compartas estado.** Particioná de forma que cada request toque **un solo** owner (sharding por entidad). Sin estado compartido, no hay nada que coordinar. Es por lejos lo más barato.
2. **Single-writer por partición.** Si una entidad la escribe siempre el mismo nodo (módulo 13), tenés orden y consistencia **sin** consenso por operación. Las RSMs y el sharding de Kafka (una partición, un orden) viven acá.
3. **Eventual + resolución de conflictos.** Si tolerás stale temporal, leaderless + quórum + anti-entropía (módulos 15-16) te da disponibilidad alta sin el cuello del líder.
4. **Consenso fuerte (Raft/Paxos).** Reservalo para lo que **de verdad** necesita acuerdo: quién es el líder, el orden de un log, una config global, un lock de correctitud. Es la opción más cara; usala donde un dato viejo o dos dueños **corrompen** algo.

> La pregunta que tenés que hacerte en la entrevista: **"¿esto necesita acuerdo, o me alcanza con que cada cosa tenga un único dueño?"**. Casi siempre la respuesta es la segunda. El consenso es el martillo más pesado de la caja — impresionante, pero la mayoría de los problemas son tornillos, no clavos.

**Ejercicios 18**
18.1 🔁 Nombrá tres costos concretos de usar consenso.
18.2 🧠 Ordená de más barato a más caro: leaderless+quórum, no compartir estado, consenso fuerte, single-writer. Justificá el orden.
18.3 🧠 Te piden "un contador global exacto de likes en tiempo real a escala de Twitter". ¿Meterías consenso? ¿Qué harías en su lugar?

---

## Módulo 19 — Cómo hablar de consenso en una entrevista

**Teoría.** Cierre operativo. Cuando te toque un tema de coordinación, no recites el algoritmo: estructurá la respuesta con **las tres cosas** y, cuando aparezca una falla, con **las cuatro capas**.

**Las tres cosas:**
- **(a) Happy path:** "hay un líder elegido por mayoría que ordena las escrituras en un log replicado; una escritura se confirma cuando una mayoría la persistió".
- **(b) Modelo de falla:** "si el líder se cae o se particiona, los followers no reciben heartbeat y eligen uno nuevo con término mayor; el viejo, al ver el término mayor, se degrada".
- **(c) Recuperación:** "el líder nuevo, por la election restriction, ya tiene todos los commits; los followers atrasados se ponen al día con AppendEntries o un snapshot".

**El reflejo de las cuatro capas** ante "¿qué pasa si...?": *qué se rompe → por qué a esta escala/topología → control de corto plazo (fencing, subir timeout, degradar) → rediseño (term/quorum/fencing/φ-accrual)*.

Y tres muletillas que suenan a que entendiste, no memorizaste:
- *"No puedo distinguir caído de lento; por eso uso quórums, no un nodo que confío que está vivo."*
- *"Un lock distribuido no es seguro sin un fencing token que valide el recurso."*
- *"¿Necesito acuerdo, o me alcanza con un único dueño por partición?"*

> **Cierre.** Todo este módulo es una sola pregunta repetida: *¿cómo decido algo cuando no puedo confiar en que el otro está vivo, en que mi mensaje llegó, ni en mi reloj?*. Las respuestas escalan en costo: no compartir estado → un único dueño → eventual con anti-entropía → consenso fuerte. El que sabe **cuál** de esas cuatro aplica a cada problema —y puede contar la cascada de 4 capas cuando algo falla— ya razona como senior. Ahora pasá al [laboratorio práctico](consenso-practica.md) y **construí** los cinco: relojes de Lamport y vectoriales, un quórum R+W>N, una elección de líder estilo Raft con su split vote, un lock con fencing token y un hash ring con su rehash masivo. Entender es la mitad; romperlo y arreglarlo con tus manos es la otra.

**Ejercicios 19**
19.1 🔁 Listá "las tres cosas" de una respuesta de system design.
19.2 🧠 Te preguntan "¿qué pasa si el líder se pausa 20s por GC?". Respondé en las cuatro capas.
19.3 🧠 ¿Por qué la frase "no puedo distinguir caído de lento" es el corazón de casi todo este módulo?

---

## Soluciones

**1.1** Un timeout no distingue (1) el nodo se cayó, (2) el nodo está vivo pero lento, (3) la red perdió tu mensaje (o la respuesta). Solo te dice "no hubo respuesta a tiempo".

**1.2** Porque los relojes físicos derivan (clock skew) y NTP solo los acerca con error de decenas de ms. Con skew, un evento posterior en una máquina puede tener timestamp menor que uno anterior en otra → el orden por timestamp miente.

**1.3** Porque FLP aplica al modelo **asíncrono puro** (sin cota de tiempo). En la práctica usamos un modelo **parcialmente síncrono**: timeouts + la realidad de que la red *casi siempre* se porta bien. Raft garantiza **safety siempre** y **liveness cuando la red se estabiliza**; FLP solo dice que en el peor caso adversarial podría no terminar, no que falle siempre.

**2.1** Crash-stop: el nodo se cae y queda caído para siempre. Crash-recovery: el nodo se cae y **vuelve** (reinicia), perdiendo el estado en memoria pero conservando lo persistido en disco. Es el modelo realista de Raft/etcd.

**2.2** Porque un nodo bizantino puede mandar mensajes **contradictorios** a distintos pares (mentir), así que necesitás más redundancia para que la mayoría honesta domine: `3f+1` nodos para tolerar `f` bizantinos, contra `2f+1` para tolerar `f` crashes.

**2.3** Prioriza **safety/consistencia** (nunca devolver datos divergentes) y cede **liveness/disponibilidad** bajo partición: el lado minoritario deja de responder hasta que la partición se cure. Es exactamente "C sobre A".

**3.1** Un falso positivo es declarar **caído** a un nodo que en realidad estaba **vivo pero lento** (o tuvo una pérdida de paquete puntual). Lo dispara un timeout demasiado agresivo combinado con una pausa real del nodo (GC, pico de red).

**3.2** Con timeout de 1s y pausas de hasta 1.5s, vas a declarar muerto a un nodo sano en cada GC largo → failover innecesario, posible flapping/split-brain. Mitigación sin volverte ciego: subir el timeout por encima de la pausa esperada y/o exigir `k` heartbeats perdidos seguidos; idealmente un detector adaptativo (φ-accrual) y fencing para que el "muerto" que revive no pueda escribir.

**3.3** φ-accrual no da un sí/no binario sino un **nivel de sospecha continuo** calculado sobre el **histograma real** de tiempos de llegada de heartbeats. Te deja elegir el umbral según cuánto te duele un falso positivo, y se adapta solo a una red más lenta sin reconfigurar un número fijo.

**4.1** Porque `Date.now()` (wall clock) puede **saltar hacia atrás** cuando NTP corrige la deriva → una duración te puede dar negativa o absurda. El reloj monotónico nunca retrocede, así que `fin - inicio` siempre es correcto.

**4.2** Nodo A escribe `x=1` a las 10:00:00.100 (su reloj adelantado 80ms; el real era 10:00:00.020). Nodo B escribe `x=2` a las 10:00:00.060 (reloj exacto) — **después** de verdad. Con LWW por timestamp, gana `x=1` (100 > 60) aunque `x=2` fue la escritura real más reciente → se **perdió** una escritura por skew.

**4.3** TrueTime acota el error del reloj a un intervalo `[earliest, latest]` (con relojes atómicos + GPS) y hace que las transacciones **esperen** ese intervalo antes de commitear, garantizando orden externo (linealizabilidad) global. La mayoría no lo tiene porque requiere hardware dedicado en cada datacenter — es de Google/Spanner.

**5.1** (1) Antes de un evento local: `L = L + 1`. (2) Al recibir un mensaje con timestamp `Lm`: `L = max(L, Lm) + 1` (y al enviar, adjuntás tu `L` actual).

**5.2** No. `L(a) < L(b)` es **necesario** si `a → b`, pero **no suficiente**: dos eventos concurrentes (sin relación causal) también pueden tener timestamps distintos. Lamport solo garantiza la implicación en una dirección (`a→b ⟹ L(a)<L(b)`), no la vuelta.

**5.3** A1: `L=1`. A2: `L=2` (y manda 2 en el mensaje). B1 recibe con `Lm=2`: `L = max(L_B, 2) + 1`; si B no tenía eventos previos (`L_B=0`), `B1 = 3`. Orden de Lamport: A1(1) < A2(2) < B1(3), consistente con la causalidad A2 → B1.

**6.1** Un vector clock puede distinguir **concurrencia** de **causalidad**: dado dos eventos te dice si uno ocurrió-antes del otro o si son **concurrentes** (chocaron de verdad). Lamport solo da un orden total consistente con causalidad, pero no detecta concurrencia.

**6.2** `V=[2,1,0]` y `W=[2,0,1]`: V tiene `B=1>0` pero W tiene `C=1>0`; ninguno es `≤` en todas las posiciones → **concurrentes**. Para una base leaderless significa que son dos escrituras que **chocaron**; el sistema guarda **ambas versiones (siblings)** para que la app/un CRDT resuelva, en vez de descartar una.

**6.3** El vector crece con la cantidad de nodos que escriben (`O(nodos)` por valor). Los sistemas reales lo acotan con **version vectors / dotted version vectors** (una entrada por *réplica*, no por cliente), poda de entradas viejas y límites de siblings.

**7.1** Acuerdo (no deciden valores distintos — safety), validez (el valor decidido fue propuesto — safety), terminación (eventualmente todos deciden — **liveness**).

**7.2** Porque si todas las réplicas arrancan igual y aplican **la misma secuencia de comandos en el mismo orden** (máquina de estados determinística), terminan en el mismo estado. Entonces replicar estado = acordar **el orden del log**; lo demás es reproducirlo localmente.

**7.3** Sin validez, un algoritmo podría "decidir" siempre un valor fijo (p. ej. siempre `0`) ignorando lo propuesto: cumpliría acuerdo (todos coinciden en `0`) y terminación (deciden ya), pero es **inútil** porque no refleja ninguna propuesta real. La validez ata la decisión a la entrada.

**8.1** Que **dos quórums por mayoría siempre se solapan** en al menos un nodo (`N/2+1` + `N/2+1` > `N`). Ese nodo compartido no puede votar dos cosas contradictorias en el mismo término → no puede haber dos líderes con mayoría.

**8.2** Porque un número impar tolera los mismos fallos que el par inmediato inferior (5 y 6 toleran 2; 3 y 4 toleran 1) **con un nodo menos**, y los pares aumentan el riesgo/costo de empates sin subir la tolerancia. 3, 5, 7 maximizan tolerancia por nodo y evitan empates.

**8.3** Con `W=2, R=1` (N=3): escribís en réplicas {1,2}; después leés de {3} (que no recibió la escritura) → **stale**. `R+W=3 ≤ N=3`, no garantiza solape. Con `R=2`: cualquier par de réplicas que leas **se solapa** con el par que escribiste (`R+W=4 > 3`), así que al menos una tiene el dato fresco → no stale.

**9.1** Un term es un entero monotónico que marca una "época" con **a lo sumo un líder**. Sirve de reloj lógico para detectar información obsoleta: un mensaje con término menor al actual se ignora.

**9.2** Para evitar el **split vote** repetido (un livelock): con timeouts iguales, todos se vuelven candidatos a la vez, parten el voto y nadie junta mayoría, una y otra vez. El timeout aleatorio **desincroniza** — casi siempre uno arranca primero y corta antes de que despierten los demás. Es la misma idea del jitter contra el retry storm.

**9.3** No se commitean porque el líder viejo está en la **minoría**: nunca junta la mayoría que Raft exige para commitear. Al recontactar a cualquier nodo con término mayor, **se entera y se degrada a follower**. No alcanza para un efecto externo (tomar un lock, escribir en S3) porque ahí Raft no controla el recurso: el líder viejo, antes de degradarse, podría haber ejecutado el efecto → hace falta un **fencing token** que el recurso valide.

**10.1** Cuando una **mayoría** de réplicas confirmó que escribió la entrada en su log: ahí el líder la marca **committed**, la **aplica** a su máquina de estados y recién **responde OK** al cliente.

**10.2** Porque la entrada está en una mayoría, y cualquier líder futuro necesita **mayoría de votos**, que **se solapa** con la mayoría que tiene la entrada. Por la election restriction, ese nodo compartido no vota por un candidato sin esa entrada → el líder nuevo la tiene. El solape de quórums hace durable al commit.

**10.3** `AppendEntries` lleva `(prevLogIndex, prevLogTerm)`; el follower rechaza si no matchea, el líder **retrocede** el índice hasta el punto común y desde ahí **sobrescribe** lo divergente del follower. **Gana el log del líder** (leader append-only: el líder nunca toca el suyo).

**11.1** Garantiza que **un líder nuevo siempre tiene todas las entradas ya committeadas**: como solo votás por candidatos con log al menos tan actualizado como el tuyo, un candidato al que le falta un commit no junta mayoría (la mayoría que tiene el commit no lo vota). Así ningún commit se "pierde" al cambiar de líder.

**11.2** Porque el log crece sin límite y no podés guardarlo ni reproducirlo entero por siempre. Snapshots capturan el estado y descartan el log viejo; sin ellos, el disco se llena, el recovery tarda eternidades reproduciendo todo, y un follower nuevo/atrasado nunca se pone al día.

**11.3** Pasar de la config vieja a la nueva de golpe puede abrir una ventana con **dos mayorías disjuntas** (una en cada config) que eligen **dos líderes** a la vez → split-brain. Raft lo evita con **joint consensus** (fase intermedia que exige mayoría de **ambas** configs) o cambiando **de a un nodo** (etcd), de modo que las mayorías siempre se solapan.

**12.1** Un fencing token es un número **monotónicamente creciente** que el servicio de lock entrega junto al lock. Lo valida el **recurso protegido** (no el lock): rechaza cualquier escritura con un token menor que el último que vio.

**12.2** A toma el lock (TTL 30s) → A se **congela** (GC) → pasan 30s, el lease **expira** → B toma el lock y escribe → A **despierta**, se cree dueño y escribe encima. Se rompe en el momento en que A escribe con el lock ya expirado: hay dos escritores sobre el mismo recurso. El fencing lo corta porque A escribe con token viejo y el recurso lo rechaza.

**12.3** Es aceptable para **eficiencia** (evitar que dos workers hagan el mismo trabajo caro; si por una pausa lo hacen los dos, perdés algo de CPU pero nada se corrompe). **No** es aceptable para **correctitud** (si dos dueños escribiendo a la vez corrompen datos o doble-cobran): ahí necesitás fencing token validado por el recurso, o consenso real.

**13.1** Un **lock** no vence solo: si el dueño muere sin liberarlo, queda trabado. Un **lease** es un lock con **vencimiento automático (TTL)**: si el dueño no lo renueva, se libera solo. En distribuido querés lease (tolera que el dueño muera), al precio de la ventana de expiración-mientras-vivo → fencing.

**13.2** Porque tener un único escritor por dato/partición **elimina los conflictos de escritura en el origen**: no hay dos updates concurrentes que reconciliar, no necesitás locks por operación ni resolución de conflictos. Ordenás "gratis" (un solo escritor define el orden). Es más barato que coordinar cada operación.

**13.3** Porque elegir líder **es** consenso, y etcd/ZooKeeper son RSMs sobre Raft/Zab: dan quórum, leases con revocación y nodos efímeros, garantizando un único líder aun con fallas. Un Redis single-node es un **SPOF** y no da garantías de quórum: si se cae o se particiona, podés terminar con dos "líderes".

**14.1** Todos-contra-todos es `O(N²)` mensajes por ronda → no escala a miles de nodos. Gossip propaga la info como rumor a unos pocos vecinos al azar, llegando a todos en `O(log N)` rondas con carga acotada por nodo.

**14.2** El indirect probing: antes de declarar caído a un nodo que no respondió tu ping, pedís a **otros `k` nodos** que también le hagan ping. Si alguno lo alcanza, no estaba caído (era una pérdida de paquete puntual entre vos y él) → baja drásticamente los **falsos positivos** del módulo 3.

**14.3** Gossip: para difundir **membresía** (quién está vivo) a escala con consistencia eventual — no necesitás acuerdo instantáneo. Raft: para acuerdo **fuerte** (quién es el líder, el orden del log de comandos). Ejemplo: Consul usa gossip (SWIM) para membership/health y Raft para el estado consistente del catálogo/KV.

**15.1** **Cualquier** réplica acepta lecturas y escrituras (no hay líder). La consistencia se controla con los quórums: una escritura espera `W` confirmaciones de las `N` réplicas y una lectura espera `R`; con `R+W>N` la lectura ve la última escritura.

**15.2** Porque `R+W>N` solo garantiza que **leés un valor que estuvo en una escritura con quórum**, pero sin un líder que ordene, **dos escrituras concurrentes** pueden haber pasado ambas con quórum y chocar → necesitás vector clocks + resolución de conflictos. La linealizabilidad de Raft da un **único orden total** de todas las operaciones; el quórum leaderless no.

**15.3** Si las N réplicas naturales del dato están caídas durante una partición, la escritura se guarda en **otras** réplicas vivas con un **hint** de a quién pertenece (sloppy quorum → no rechaza, sube disponibilidad de escritura). Cuando las naturales vuelven, los nodos que guardaron entregan los datos (**hinted handoff**). Prioriza **A** (disponibilidad) de CAP, al precio de stale temporal.

**16.1** (1) Read-repair (oportunista, en el camino de la lectura), (2) hinted handoff (durante/post-falla, entrega lo guardado de prestado), (3) anti-entropía de fondo con Merkle trees (sistemático, compara y repara todo el dataset).

**16.2** Read-repair solo arregla **lo que se lee**: los datos fríos (que nadie consulta) quedan divergentes para siempre. Por eso hace falta una pasada de fondo (Merkle) que reconcilie también lo que nadie lee.

**16.3** Cada réplica arma un Merkle tree (árbol de hashes) de su dataset. Intercambian la **raíz**: si difiere, bajan solo por las ramas cuyo hash no coincide, en `O(log n)`, hasta aislar los 3 registros distintos, y transfieren **solo esos**. Nunca mandan el TB porque los subárboles idénticos se descartan por el hash de un nivel superior.

**17.1** Cambia `% N` para casi **todas** las claves → rehash masivo: casi todo el dataset se reasigna de nodo. A escala es una avalancha de movimiento de datos y un cache que se invalida entero por agregar/sacar un solo nodo.

**17.2** Aproximadamente `1/N` del dataset (solo las claves que caen en el arco entre el nodo nuevo y su predecesor en el ring). El resto de las claves siguen apuntando al mismo nodo porque su "primer nodo en sentido horario" no cambió.

**17.3** (1) **Desbalance:** con pocos nodos físicos, los arcos del ring quedan disparejos (hot nodes); muchos vnodes por nodo reparten parejo por la ley de los grandes números. (2) **Carga concentrada al caer un nodo:** sin vnodes, toda la carga del caído va a su único vecino; con vnodes dispersos, se redistribuye entre muchos nodos.

**18.1** Tres de: latencia (round-trip a una mayoría por decisión), throughput limitado (todo pasa por el líder / hay un cuello), pérdida de liveness bajo partición (se planta), y complejidad operativa (re-elecciones, snapshots, membresía).

**18.2** De más barato a más caro: (1) **no compartir estado** (sharding por owner — no hay nada que coordinar), (2) **single-writer por partición** (orden gratis, sin consenso por operación), (3) **leaderless + quórum** (disponibilidad alta, paga anti-entropía y tolera stale), (4) **consenso fuerte** (round-trip a mayoría por decisión + se planta bajo partición). El costo sube con la cantidad de coordinación por operación.

**18.3** No metería consenso: un contador exacto global por consenso sería un cuello brutal. Haría **sharding del contador** (N contadores parciales, cada uno con su único dueño) + **agregación** asíncrona, tolerando un valor *aproximado*/levemente atrasado. Likes es un caso donde "eventual y casi-exacto" vale infinitamente más que "exacto y lento". (No compartir estado → agregar.)

**19.1** (a) Una forma para el **happy path**, (b) un **modelo de falla**, (c) un **proceso de recuperación**.

**19.2** (1) **Qué se rompe:** los followers dejan de recibir heartbeat y, si el timeout es menor a 20s, eligen un nuevo líder; el viejo, al volver, podría haber ejecutado efectos externos. (2) **Por qué a esta escala:** pausas de GC de segundos son normales bajo carga; con timeouts agresivos, una pausa = un failover. (3) **Corto plazo:** election timeout mayor que la pausa esperada + degradación del líder viejo al ver término mayor; fencing token para efectos externos. (4) **Largo plazo:** detector φ-accrual, leases con fencing, y bajar la presión de GC / aislar el path crítico.

**19.3** Porque de "no puedo distinguir caído de lento" salen **todos** los mecanismos del módulo: quórums (decido con una mayoría en vez de confiar en un nodo), terms/fencing (me protejo del que revive), detección de fallas con timeout (estimo lo que no puedo saber) y consenso (acuerdo robusto a que algunos no respondan). Es la incertidumbre fundamental que todo lo demás intenta domar.
