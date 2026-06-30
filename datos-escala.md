# Datos a escala y multi-tenancy a fondo

**Sharding, skew, noisy neighbor, migraciones sin downtime, B-tree vs LSM y Snowflake IDs · Node + Go · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **M4** del roadmap profundo de system design (M1 [Consenso](consenso.md), M2 [Replicación](replicacion.md), M3 [Streams](streams.md)). Trata lo que pasa cuando **los datos no entran en una sola máquina** y **muchos clientes comparten el sistema**: cómo partís (sharding), qué hacés con el desbalance (skew/hot shard), cómo aislás inquilinos (noisy neighbor/shuffle sharding), cómo cambiás el esquema **sin apagar nada** (expand-contract), cómo guardan los datos los motores por dentro (B-tree vs LSM), cómo generás IDs sin coordinación (Snowflake) y cómo partís un monolito (strangler). Su complemento es el [laboratorio práctico](datos-escala-practica.md): acá entendés, allá **construís** (range vs hash sharding, un generador Snowflake, una migración expand-contract, un mini LSM-tree y shuffle sharding).

**Lo que asumimos.** El método de [system-design](system-design.md), [PostgreSQL](postgresql.md) (índices, MVCC, ACID) y [NoSQL](nosql.md) (particiones, hot partitions). De [Consenso](consenso.md) reusamos **consistent hashing** (módulo 17) para el rebalanceo; de [Event-driven](event-driven.md)/[DDD](ddd.md), la idea de partir un monolito. No asumimos Go: los ejemplos se explican línea por línea.

> ⚠️ **Sobre los lenguajes.** Acá la lógica es de **datos y particionado**. Los ejemplos van en **Node/TS** (tu stack) y en **Go** donde ayuda. La idea importa más que el runtime.

### El marco que usamos en todo el módulo

Igual que en M1-M3: **cuatro capas** ante fallas —(1) qué se rompe, (2) por qué a *esta* escala/forma de tráfico, (3) control de corto plazo, (4) rediseño— y **las tres cosas** al diseñar: happy path, modelo de falla, recuperación.

> **El ejemplo que fija el tono — "un cliente tiró abajo a todos los demás".** Respuesta débil: *"escalá los servidores"*. Respuesta profunda, en cascada: **un solo tenant manda un pico de 100× (o una query pesada) → satura la conexión/CPU/IO del shard que comparte con otros → los demás tenants de ese shard ven latencia y timeouts → reintentan → más carga → caen todos los de ese shard, aunque no hicieron nada**. No es "faltan servidores": es que **no aislaste el blast radius**. Lo arreglás con quotas + shuffle sharding (módulo 5), no comprando máquinas.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (el grueso va al [laboratorio](datos-escala-practica.md)).

**Índice de módulos**

*Etapa 1 — Partir los datos*
1. Cuándo (y cuándo NO) shardear
2. Elegir la sharding key: range vs hash

*Etapa 2 — Rebalanceo y aislamiento*
3. Rebalanceo de shards sin mover todo
4. Skew y hot shards
5. Noisy neighbor, shuffle sharding y celdas

*Etapa 3 — Cambiar el esquema sin downtime*
6. Expand-contract (parallel change)
7. Backfill de datos grandes
8. Doble escritura / doble lectura en la transición

*Etapa 4 — Storage engines por dentro*
9. WAL y durabilidad
10. B-tree (lectura en su lugar)
11. LSM-tree, SSTables y compaction
12. B-tree vs LSM: el trade-off

*Etapa 5 — IDs y búsqueda a escala*
13. IDs distribuidos: UUID, Snowflake, ULID
14. Búsqueda a escala: freshness vs ranking

*Etapa 6 — Migrar el sistema y cierre*
15. Partir el monolito: strangler fig
16. Cómo hablar de datos a escala en una entrevista

Las soluciones de **todos** los ejercicios están al final.

---

## Etapa 1 — Partir los datos

## Módulo 1 — Cuándo (y cuándo NO) shardear

**Teoría.** **Sharding** = partir los datos **horizontalmente** (por filas) entre varias máquinas, cada una con un subconjunto. Es la última carta para escalar **escrituras** y volumen de datos cuando una sola base ya no da. Pero la regla número uno, la que te separa de quien leyó un blog:

> **No shardees hasta que no te quede otra.** Sharding multiplica la complejidad: los **joins cross-shard** se vuelven caros o imposibles, las **transacciones** dejan de ser ACID locales (volvés a M2: saga/2PC), el **rebalanceo** mueve datos, y las queries que antes eran un `WHERE` ahora tienen que saber **en qué shard** vive el dato. Antes de shardear, agotá: **vertical scaling** (una máquina más grande — sorprende cuánto aguanta), **read replicas** (M2, para lecturas), **caching** ([Redis](redis.md)), e **índices** ([PostgreSQL](postgresql.md)).

Dos formas de partir:
- **Vertical (por feature/tabla):** mandás distintas tablas/dominios a distintas bases (usuarios en una, pedidos en otra). Es lo primero y más simple; alinea con [DDD](ddd.md) (un servicio, su base). Tope: una sola tabla que crece sin fin sigue sin entrar.
- **Horizontal (sharding real, por filas):** partís **una** tabla enorme por una **sharding key**: `usuarios` se reparte en shard A (ids 1-1M), shard B (1M-2M), etc. Es lo que resuelve "una tabla con miles de millones de filas".

> La frase mental: **shardear es comprar escala de escritura con complejidad de queries y operación.** Un senior sabe **postergarlo** (vertical + replicas + cache + índices) y, cuando ya no hay opción, elegir bien la **key** (módulo 2) — porque la key mal elegida es la que después no podés cambiar sin re-shardear todo.

**Ejercicios 1**
1.1 🔁 ¿Qué multiplica el sharding que hay que pagar (nombrá tres costos)?
1.2 🧠 ¿Qué cuatro cosas agotás **antes** de shardear y por qué?
1.3 🧠 Diferencia entre partición vertical y horizontal, con un ejemplo de cada una.

---

## Módulo 2 — Elegir la sharding key: range vs hash

**Teoría.** La decisión que define todo: **¿cómo asignás cada fila a un shard?**. La key tiene que estar en casi todas tus queries (si no, cada query golpea todos los shards = *scatter-gather*). Tres estrategias:

- **Range-based (por rango):** shards por rangos ordenados de la key (`A-F`, `G-M`, …, o por fecha: cada mes un shard). **Pro:** range scans eficientes (`WHERE fecha BETWEEN`), datos cercanos juntos. **Contra brutal:** si la key crece **secuencialmente** (timestamp, auto-increment), **todas** las escrituras nuevas caen en el **último** shard → *hot shard* de escritura mientras los demás están ociosos.
- **Hash-based (por hash):** `shard = hash(key) % N` (o consistent hashing). **Pro:** reparte **parejo**, mata el hot shard de inserción secuencial. **Contra:** perdés los range scans (datos contiguos quedan desparramados), y una **key única muy popular** (un usuario celebridad) sigue siendo un hot shard.
- **Directory/lookup (por directorio):** un servicio que mapea `key → shard` explícitamente. Flexible (podés mover una key a otro shard), pero el directorio es un nivel de indirección y un posible cuello/SPOF.

| | Range | Hash | Directory |
|---|---|---|---|
| Range scans | ✅ | ❌ | depende |
| Reparto parejo | ❌ (secuencial → hot) | ✅ | ✅ (lo controlás) |
| Mover una key | difícil | difícil (rehash) | ✅ fácil |
| Complejidad | baja | baja | media (servicio extra) |

> La elección sale del **patrón de acceso**: ¿hacés muchos range scans por tiempo? → range (y mitigás el hot shard con una key compuesta, p. ej. `hash(userId)+timestamp`). ¿Acceso por clave puntual y querés reparto parejo? → hash. La trampa clásica de entrevista: *"sharding por `created_at`"* → te comés el hot shard de escritura. Lo ves en el [laboratorio](datos-escala-practica.md).

**Ejercicios 2**
2.1 🔁 ¿Por qué la sharding key tiene que estar en casi todas tus queries?
2.2 🧠 ¿Por qué sharding range-based por un timestamp produce un hot shard, y hash-based no?
2.3 🧠 ¿Qué resuelve el sharding por directorio que range/hash no, y a qué costo?

---

## Etapa 2 — Rebalanceo y aislamiento

## Módulo 3 — Rebalanceo de shards sin mover todo

**Teoría.** Agregás capacidad (un shard más) o un nodo se cae: hay que **redistribuir** los datos. Cómo lo hacés define si el rebalanceo es un evento rutinario o un apocalipsis:

- **`hash % N` (la trampa):** ya lo vimos en [Consenso módulo 17](consenso.md) — cambiar `N` remapea **casi todas** las keys → mover casi todo el dataset. **No se usa para rebalancear.**
- **Cantidad fija de particiones (la práctica):** creás **muchas más** particiones que nodos desde el día uno (p. ej. 1000 particiones, 10 nodos → 100 cada uno). Agregar un nodo = mover **algunas particiones enteras** a él, sin recalcular nada. Es lo que hace **Cassandra** (vnodes) y **Riak**. El número de particiones es fijo; lo que cambia es **qué nodo** las hospeda.
- **Consistent hashing con vnodes** ([Consenso módulo 17](consenso.md)): mueve solo `~1/N` al agregar un nodo. Es la base de Dynamo/Cassandra.

> La idea clave: **desacoplá el número de particiones del número de nodos.** Si la cantidad de particiones es fija y grande, sumar/sacar nodos solo **reasigna** particiones (mover datos de a bloques), nunca **rehashea** el dataset. El rebalanceo se vuelve "mover 100 de 1000 particiones al nodo nuevo", no "recalcular dónde va cada fila". El precio: el rebalanceo **igual mueve datos** (I/O, ancho de banda) → se hace **throttled** (de a poco) para no saturar el sistema en producción.

**Ejercicios 3**
3.1 🔁 ¿Por qué `hash % N` no sirve para rebalancear al agregar un nodo?
3.2 🧠 ¿Cómo permite "muchas particiones fijas, pocos nodos" rebalancear sin rehashear?
3.3 🧠 ¿Por qué el rebalanceo se hace throttled aunque tengas el ancho de banda?

---

## Módulo 4 — Skew y hot shards

**Teoría.** El enemigo del sharding es el **skew**: la carga **no** se reparte pareja, y un shard recibe muchísimo más que los otros (*hot shard*). Aunque tengas 100 shards, si el 50% del tráfico va a uno, ese uno se satura y los otros 99 están ociosos — no escalaste nada. Causas y remedios:

- **Key secuencial** (timestamp, auto-increment) con sharding range → todo lo nuevo al último shard. **Remedio:** hash de la key, o key compuesta.
- **Hot key / celebridad:** una sola key recibe muchísimo tráfico (el tweet de una estrella, un producto en oferta). El hash **no** ayuda — una key va a **un** shard. **Remedios:** (a) **salting** — partir la hot key en `key:0`…`key:n` subkeys que caen en shards distintos, y agregar al leer; (b) **shard dedicado** para esa key; (c) **caché** delante (la hot key es justo la más cacheable).

> 🔥 **Falla en 4 capas — hot shard por una key celebridad.**
> 1. **Qué se rompe:** un shard se satura (CPU/IO/conexiones) por una sola key con tráfico desproporcionado; sus queries se degradan y arrastran a las otras keys que viven en ese shard.
> 2. **Por qué a esta escala:** la distribución de tráfico real es **power-law** (Zipf), no uniforme — siempre hay un puñado de keys con órdenes de magnitud más tráfico; el hash reparte keys parejo pero **no** reparte la *carga por key*.
> 3. **Control de corto plazo:** caché agresiva delante de la hot key (absorbe la mayoría de las lecturas); rate-limit a esa key.
> 4. **Largo plazo:** **salting** de la hot key (varias subkeys en shards distintos) o un **shard dedicado**; detectar hot keys automáticamente (telemetría por key) y reubicarlas. El sharding reparte *keys*; vos tenés que repartir *carga*.

**Ejercicios 4**
4.1 🔁 ¿Qué es un hot shard y por qué "tener muchos shards" no lo evita?
4.2 🧠 ¿Por qué el hashing reparte keys parejo pero NO resuelve una hot key celebridad?
4.3 🧠 Explicá el salting de una hot key: cómo escribís y cómo leés.

---

## Módulo 5 — Noisy neighbor, shuffle sharding y celdas

**Teoría.** En **multi-tenancy** (muchos clientes comparten infra), aparece el **noisy neighbor**: un tenant que consume de más (pico de tráfico, query pesada, bug) **degrada a los demás** que comparten sus recursos. El objetivo es **aislar el blast radius**: que el daño de un tenant no se propague a todos.

Herramientas, de menos a más sofisticada:

1. **Quotas / rate limits por tenant:** un techo de recursos (requests/s, conexiones, CPU) por cliente. Frena al ruidoso, pero si todos comparten el mismo pool, un tenant igual puede consumir su quota y rozar a los demás.
2. **Shuffle sharding (la joya de AWS):** tenés N workers. En vez de que **todos** los tenants usen **todos** los workers (un tenant tóxico afecta a todos), a **cada tenant** le asignás un **subconjunto aleatorio de `k`** workers. Dos tenants rara vez comparten **exactamente** el mismo subconjunto. Si un tenant se vuelve tóxico (envenena requests, satura), solo afecta a **sus `k` workers**, y la **superposición** con cualquier otro tenant es pequeña → la mayoría de los demás tenants tienen al menos un worker sano y siguen funcionando. Reduce el blast radius de "todos" a "una fracción chica".
3. **Cell-based architecture:** partís **todo el stack** (no solo workers) en **celdas** independientes; cada tenant vive en **una** celda; una falla (o un deploy malo) queda **contenida** en su celda. Es el aislamiento más fuerte (lo inyecta Amazon en entrevistas).

> 🔥 **Falla en 4 capas — noisy neighbor sin aislamiento.** (1) Qué se rompe: un tenant satura el pool compartido → todos los tenants de ese pool caen. (2) Por qué a esta escala: con un pool único, el blast radius de **cualquier** tenant es **el 100%**. (3) Corto plazo: quotas/rate-limit por tenant + circuit breaker. (4) Largo plazo: **shuffle sharding** (blast radius = `k/N` con poca superposición) o **celdas** (blast radius = una celda). La pregunta senior: *"¿cuál es el blast radius de un solo tenant que se vuelve loco?"* — si la respuesta es "todos", rediseñá.

> La matemática que impresiona: con N=100 workers y k=5 por tenant, hay `C(100,5)` ≈ 75 millones de combinaciones posibles — dos tenants comparten su set completo casi nunca. Pero ojo: más combinaciones reducen la **probabilidad** de solapamiento total, no garantizan un blast radius trivial — el blast radius real depende de **N, K y la cantidad de tenants**, y hay que **medirlo** (en el [laboratorio](datos-escala-practica.md) verás que con N=12, K=3, solo 3 de 50 tenants quedan totalmente expuestos a uno tóxico).

**Ejercicios 5**
5.1 🔁 ¿Qué es el noisy neighbor y qué significa "aislar el blast radius"?
5.2 🧠 Explicá shuffle sharding: ¿por qué un tenant tóxico afecta a pocos en vez de a todos?
5.3 🧠 Diferencia entre shuffle sharding y cell-based architecture en qué aíslan.

---

## Etapa 3 — Cambiar el esquema sin downtime

## Módulo 6 — Expand-contract (parallel change)

**Teoría.** Tenés que cambiar el esquema (renombrar una columna, cambiar un tipo, partir una tabla) de un sistema **que no podés apagar** y donde el **código viejo y el nuevo conviven** durante el deploy (rolling deploy: por un rato corren las dos versiones a la vez). Un `ALTER TABLE ... RENAME` de un solo paso **rompe** el código viejo que todavía usa el nombre anterior. La técnica es **expand-contract** (o *parallel change*), en tres fases:

1. **EXPAND (agregar, compatible hacia atrás):** agregás lo **nuevo** sin tocar lo viejo. Para un rename: agregás la **columna nueva** (la vieja sigue ahí). El esquema ahora soporta **ambas** versiones del código.
2. **MIGRATE (transición):** desplegás código que **escribe en ambas** (vieja y nueva) y **backfilleás** los datos viejos a la columna nueva (módulo 7). Cuando todo está doble-escrito y backfilleado, cambiás las **lecturas** a la columna nueva.
3. **CONTRACT (quitar lo viejo):** cuando **ningún** código usa lo viejo, lo borrás (drop de la columna vieja). Recién acá.

> 🔥 **Falla en 4 capas — el rename de un solo paso.**
> 1. **Qué se rompe:** hacés `RENAME columna` y deployás; durante el rolling deploy, las instancias viejas (que aún corren) buscan la columna con el nombre viejo → **errores en producción** hasta que termina el deploy.
> 2. **Por qué a esta escala:** con deploy continuo y varias instancias, **siempre** hay un intervalo donde conviven código viejo y nuevo; un cambio no-compatible rompe en ese intervalo.
> 3. **Control de corto plazo:** ventana de mantenimiento (downtime) — lo que justo querés evitar.
> 4. **Largo plazo:** **expand-contract** — cada paso es compatible con la versión anterior, así que viejo y nuevo conviven sin romperse, y el cambio se hace en varios deploys chicos en vez de uno grande y riesgoso.

La regla: **cada migración de esquema debe ser compatible hacia atrás con el código que todavía está corriendo.** Nunca borres ni renombres en el mismo paso que introducís lo nuevo.

**Ejercicios 6**
6.1 🔁 ¿Cuáles son las tres fases de expand-contract?
6.2 🧠 ¿Por qué un `RENAME` de un solo paso rompe en un rolling deploy?
6.3 🧠 ¿En qué momento exacto es seguro hacer el DROP de la columna vieja?

---

## Módulo 7 — Backfill de datos grandes

**Teoría.** En la fase MIGRATE casi siempre hay que **rellenar** datos: copiar la columna vieja a la nueva, recalcular un campo, poblar una tabla nueva — sobre **millones o miles de millones** de filas. Hacerlo con un solo `UPDATE tabla SET nueva = vieja` es un desastre: bloquea la tabla, infla el WAL, y puede tardar horas reventando la DB de producción. Un backfill bien hecho es:

- **Por lotes (batched):** procesás N filas por vez (p. ej. 1000), commiteando cada lote. No tomás un lock gigante ni una transacción de horas.
- **Throttled:** pausás entre lotes para no saturar IO/CPU/replicación (un backfill agresivo dispara replication lag — M2 — y degrada el tráfico real).
- **Idempotente y reanudable:** si se corta a la mitad, reanudás desde donde quedaste (trackeás el último id procesado) sin re-hacer ni romper lo ya hecho. Re-correr el backfill no debe duplicar ni corromper.
- **Online:** no bloquea las escrituras normales. Como el código nuevo **ya escribe en ambas** columnas (módulo 8), el backfill solo se ocupa de las filas **viejas** (las nuevas ya vienen bien).

> La frase mental: **un backfill es un proceso de fondo, por lotes, throttled e idempotente — no un `UPDATE` masivo.** El error clásico de un junior es correr el `UPDATE` entero y tirar abajo la base (lock + WAL + lag). El senior lo trata como una migración de datos cuidada que convive con el tráfico vivo.

**Ejercicios 7**
7.1 🔁 Nombrá las cuatro propiedades de un backfill bien hecho.
7.2 🧠 ¿Qué tres cosas malas pasan si hacés `UPDATE tabla SET nueva = vieja` de una en una tabla enorme?
7.3 🧠 ¿Por qué el backfill solo necesita ocuparse de las filas viejas, y no de las nuevas?

---

## Módulo 8 — Doble escritura / doble lectura en la transición

**Teoría.** El corazón de la fase MIGRATE es la convivencia, y tiene un orden estricto que no podés alterar:

1. **Empezás leyendo y escribiendo lo viejo** (estado inicial).
2. **Doble escritura:** desplegás código que escribe en **viejo Y nuevo** a la vez (cada escritura mantiene las dos sincronizadas). Las lecturas siguen en lo **viejo** (lo nuevo todavía no está completo: faltan los datos históricos).
3. **Backfill:** rellenás lo nuevo con los datos históricos (módulo 7). Ahora lo nuevo está **completo** (histórico + escrituras recientes).
4. **Cambiás las lecturas a lo nuevo** (y lo verificás: comparás viejo vs nuevo un tiempo, *shadow reads*, para confirmar que coinciden).
5. **Dejás de escribir lo viejo** y, finalmente, lo borrás (CONTRACT).

El orden importa porque **no podés leer de lo nuevo antes de que esté completo** (doble escritura + backfill), y **no podés borrar lo viejo antes de que nadie lo lea**. Saltarte un paso = leer datos incompletos o romper código que todavía depende de lo viejo.

> Esta coreografía es la misma idea de M2 (convivencia de versiones, garantías de sesión) aplicada al esquema: **cada paso es reversible y compatible**, así que si algo sale mal en cualquier punto, volvés atrás sin pérdida. Lo construís en el [laboratorio](datos-escala-practica.md). Herramientas reales que lo automatizan: gh-ost / pt-online-schema-change (MySQL), o migraciones expand-contract a mano en Postgres.

**Ejercicios 8**
8.1 🔁 Ordená los pasos de la transición: backfill, doble escritura, cambiar lecturas, drop de lo viejo.
8.2 🧠 ¿Por qué las lecturas se cambian a lo nuevo DESPUÉS del backfill y no antes?
8.3 🧠 ¿Qué son las *shadow reads* y para qué sirven antes de hacer el switch?

---

## Etapa 4 — Storage engines por dentro

## Módulo 9 — WAL y durabilidad

**Teoría.** ¿Cómo garantiza una base que un dato confirmado **sobrevive** a un corte de luz justo después del commit? Con el **WAL (Write-Ahead Log)**: **antes** de modificar las estructuras de datos en disco (las páginas, el índice), la base **escribe el cambio en un log secuencial append-only** y lo fuerza a disco (`fsync`). Recién entonces confirma el commit. La regla: **log primero, datos después** ("write-ahead").

¿Por qué funciona? Dos razones:

1. **Durabilidad + recuperación:** si el proceso muere a mitad de aplicar el cambio a las páginas, al reiniciar la base **reproduce el WAL** desde el último checkpoint y completa lo que faltaba. El WAL es la verdad; las páginas son una materialización que se puede reconstruir. (Es la misma idea del log de [Consenso módulo 7](consenso.md) y del changelog de [Streams módulo 12](streams.md): **el log es la fuente de verdad, el estado se reconstruye reproduciéndolo**.)
2. **Velocidad:** escribir **secuencialmente** al final de un log es **mucho** más rápido que hacer escrituras **aleatorias** a las páginas dispersas en disco (el disco —incluso SSD— prefiere lo secuencial). El WAL convierte muchas escrituras aleatorias en una secuencial.

> El WAL es también lo que habilita la **replicación** (M2: el follower aplica el WAL del líder) y el **CDC** ([Streams módulo 14](streams.md): Debezium lee el WAL). Un solo mecanismo —el log de cambios— da durabilidad, recuperación, replicación y captura de cambios. Por eso "¿cómo garantizás durabilidad?" se responde "WAL + fsync antes del commit", no "guardo en disco".

**Ejercicios 9**
9.1 🔁 ¿Qué significa "write-ahead" y en qué orden escribe el WAL respecto de los datos?
9.2 🧠 ¿Cómo recupera la base un commit confirmado tras un crash usando el WAL?
9.3 🧠 ¿Por qué escribir al WAL (secuencial) es más rápido que escribir directo a las páginas?

---

## Módulo 10 — B-tree (lectura en su lugar)

**Teoría.** El **B-tree** (en realidad B+tree) es la estructura de índice **dominante** en las bases relacionales (Postgres, MySQL/InnoDB) y muchas key-value. Es un árbol balanceado de **páginas** (bloques de tamaño fijo, típico 4-16 KB): los nodos internos guardan claves que **guían** la búsqueda, las hojas guardan los datos (o punteros a ellos), todas a la misma profundidad. Propiedades:

- **Lectura `O(log n)`:** bajás del root a la hoja en pocos saltos (un árbol de profundidad 4 indexa miles de millones de filas). Excelente para **lecturas puntuales y range scans** (las hojas están enlazadas en orden).
- **Escritura en su lugar (in-place):** para insertar/actualizar, la base **modifica la página** donde va la clave. Si la página se llena, se **parte** (page split) en dos, lo que puede propagarse hacia arriba. Eso implica **escrituras aleatorias** (la página puede estar en cualquier parte del disco) y, combinado con el WAL, **write amplification** (escribís el WAL + la página + a veces el split).

> El B-tree está optimizado para **lecturas**: la estructura ordenada y balanceada hace que encontrar y leer (puntual o por rango) sea rapidísimo y predecible. El precio lo paga la **escritura**: modificar en su lugar genera I/O aleatorio y splits. Por eso es el default de las DBs OLTP (donde las lecturas y los range scans dominan), y por eso aparece su contrincante (LSM, módulo 11) cuando las **escrituras** son el cuello.

**Ejercicios 10**
10.1 🔁 ¿Para qué tipo de operación está optimizado un B-tree y por qué?
10.2 🧠 ¿Qué es un page split y por qué contribuye a la write amplification?
10.3 🧠 ¿Por qué las escrituras de un B-tree tienden a ser aleatorias en disco?

---

## Módulo 11 — LSM-tree, SSTables y compaction

**Teoría.** El **LSM-tree (Log-Structured Merge-tree)** es la estructura **write-optimized** detrás de Cassandra, RocksDB, LevelDB, ScyllaDB. Invierte la lógica del B-tree: en vez de modificar en su lugar, **solo agrega**. Las piezas:

- **Memtable:** las escrituras van primero a una estructura **ordenada en memoria** (un árbol balanceado / skip list). Rapidísimo (es RAM) y respaldado por un WAL para durabilidad.
- **SSTable (Sorted String Table):** cuando la memtable se llena, se **vuelca a disco** como un archivo **inmutable y ordenado**. Nunca se modifica; las escrituras nuevas van a una memtable nueva → otra SSTable. Con el tiempo se acumulan **muchas** SSTables.
- **Compaction:** un proceso de fondo **mergea** SSTables (merge-sort de archivos ordenados), descartando versiones viejas de una key y los **tombstones** (marcas de borrado). Mantiene la cantidad de SSTables acotada.

El **trade-off** que define al LSM:
- **Escritura rapidísima:** todo es **append secuencial** (memtable + flush + compaction secuencial), sin I/O aleatorio. Por eso gana en cargas write-heavy.
- **Lectura más cara (read amplification):** una key puede estar en la memtable **o en cualquiera** de las N SSTables → una lectura podría chequear varias. **Se mitiga con Bloom filters** ([System Design](system-design.md)/checklist) — un filtro **probabilístico** y compacto con **cero falsos negativos** (responde "seguro que esta key NO está acá" sin equivocarse nunca en el "no"; puede tener falsos positivos, que solo cuestan un chequeo de más). Cada SSTable tiene el suyo → la lectura **saltea** las SSTables que con seguridad no tienen la key, chequeando solo las candidatas.
- **Write amplification por compaction:** el mismo dato se **reescribe varias veces** a medida que sube de nivel en las compactions → más I/O total de escritura en el tiempo (aunque cada escritura individual sea barata). Es el costo oculto.

> La frase mental: **el B-tree modifica en su lugar (lecturas baratas, escrituras aleatorias); el LSM solo agrega y mergea después (escrituras baratas/secuenciales, lecturas que tocan varios archivos + Bloom filters).** Lo construís —memtable, flush a SSTables, lectura que recorre la más nueva primero, y compaction— en el [laboratorio](datos-escala-practica.md).

**Ejercicios 11**
11.1 🔁 ¿Qué es una memtable, qué es una SSTable y qué hace la compaction?
11.2 🧠 ¿Por qué un LSM tiene read amplification y cómo lo mitigan los Bloom filters?
11.3 🧠 Explicá la write amplification de la compaction (por qué el mismo dato se escribe varias veces).

---

## Módulo 12 — B-tree vs LSM: el trade-off

**Teoría.** La comparación directa, que es una pregunta de entrevista frecuente. No hay "mejor": hay "para qué carga".

| | B-tree | LSM-tree |
|---|---|---|
| Escritura | En su lugar, I/O **aleatorio**, splits | **Append** secuencial, rapidísimo |
| Lectura | **Una** estructura, `O(log n)`, predecible | Varias SSTables + Bloom filters (read amp) |
| Write amplification | WAL + página + splits | Compaction reescribe varias veces |
| Espacio | Fragmentación, páginas a medio llenar | Mejor compresión (SSTables ordenadas/inmutables) |
| Latencia | Predecible | Picos por compaction de fondo |
| Brilla en | **Read-heavy**, range scans, OLTP | **Write-heavy**, ingestión alta, time-series |
| Ejemplos | Postgres, InnoDB | Cassandra, RocksDB, ScyllaDB |

La elección:
- **Read-heavy / OLTP / muchos range scans / latencia predecible** → **B-tree** (Postgres). El default para la mayoría de las apps.
- **Write-heavy / ingestión masiva / time-series / logs / métricas** → **LSM** (Cassandra, RocksDB). Cuando las escrituras son el cuello y tolerás read amp + picos de compaction.

> El cierre senior: *"¿tu carga es read-heavy o write-heavy?"*. Si dudás, B-tree (Postgres) es el default sano y aguanta muchísimo. LSM lo elegís cuando **medís** que las escrituras no dan abasto y el patrón es de ingestión/append (eventos, series temporales) más que de updates puntuales. Y recordá que muchos sistemas son **híbridos** o configurables (MyRocks = MySQL sobre RocksDB).

**Ejercicios 12**
12.1 🔁 ¿Cuál gana en escritura y cuál en lectura, y por qué?
12.2 🧠 Te dan un sistema de ingestión de métricas (millones de writes/s, lecturas por rango de tiempo). ¿Qué engine y por qué?
12.3 🧠 ¿Por qué el LSM suele comprimir mejor y tener picos de latencia que el B-tree no?

---

## Etapa 5 — IDs y búsqueda a escala

## Módulo 13 — IDs distribuidos: UUID, Snowflake, ULID

**Teoría.** ¿Cómo generás IDs únicos cuando **no hay una sola base** que los emita (sharding, varios servicios)? El auto-increment de una tabla no sirve: necesita un único emisor (cuello y SPOF) y no funciona entre shards. Las opciones:

- **UUID v4 (random):** 128 bits aleatorios, generado **localmente** sin coordinación. **Pro:** único con probabilidad altísima, cero coordinación. **Contra serio:** es **aleatorio**, así que como clave de un índice **B-tree** tiene **pésima localidad** — cada insert cae en una página al azar → page splits por todos lados, cache misses, fragmentación. Insertar mil UUIDs toca mil páginas distintas.
- **Snowflake (Twitter):** un entero de **64 bits** compuesto: **timestamp** (≈41 bits, ms) + **machine/worker id** (≈10 bits) + **secuencia** (≈12 bits, contador dentro del mismo ms). **Pro:** generado **localmente** (sin coordinación: cada nodo tiene su worker id), **aproximadamente ordenado por tiempo** (el timestamp va adelante) → **buena localidad** en el índice (los inserts nuevos van juntos al final), entra en un `bigint`. Es lo que querés a escala.
- **ULID / UUID v7:** misma idea que Snowflake pero en 128 bits y **lexicográficamente ordenable** (timestamp adelante + aleatorio atrás). UUID v7 es el estándar moderno que arregla la localidad de v4.

> El insight que sorprende en entrevista: **el ID no es solo "único", también define la localidad de tu índice.** Un UUID v4 random destruye la performance de inserción de un B-tree (M10); un ID **time-ordered** (Snowflake/ULID/UUIDv7) mantiene los inserts contiguos (van al final del árbol) → menos splits, mejor cache. Por eso a escala se usa Snowflake/ULID, no UUID v4. Lo construís en el [laboratorio](datos-escala-practica.md).

> **El asterisco del "sin coordinación" — worker id y reloj.** Snowflake genera sin coordinación *en caliente*, pero esconde dos dependencias que un entrevistador staff busca:
> 1. **El `worker id` tiene que ser único — y eso sí se coordina.** Los ≈10 bits dan **1024** ids de nodo; como cada proceso lleva su **propia** secuencia, dos nodos que arrancan con el **mismo** worker id generan en paralelo el mismo `(timestamp, worker, secuencia)` → **IDs idénticos**. Asignarlos sin choque necesita un árbitro: o **config estática** (frágil cuando los pods se reciclan/autoescalan) o un **registro central** (ZooKeeper/etcd con nodos efímeros, como hizo Twitter). Es un punto de coordinación y, si el registro es único, un **SPOF de *arranque*** — no de generación: una vez que el nodo tomó su id, genera solo. La coordinación se corrió del *path* de cada ID al *bootstrap* del nodo.
> 2. **El reloj que retrocede rompe la unicidad.** El timestamp embebido **es hora de pared** (epoch en ms) —tiene que serlo, para ser comparable entre nodos y traducible a tiempo real— y va adelante asumiendo que el reloj **solo avanza**; pero NTP corrige hacia atrás, una VM migra, un *leap second*… si el reloj **retrocede**, podés re-emitir un `(timestamp, secuencia)` ya usado → **colisión**. La garantía de no-retroceso **no la da un reloj monótono del SO** (ese no se traduce a hora real): la impone el **propio generador**, que recuerda el **último timestamp emitido** y, si el reloj actual es **menor**, **se niega a generar** (el Snowflake original de Twitter lanza *"Clock is moving backwards"*); algunas implementaciones más tolerantes **esperan** a que el reloj alcance en vez de fallar. (No confundir: el "esperar al próximo ms" del original es para cuando se **agota la secuencia** dentro del mismo milisegundo, no para el reloj que retrocede.)
>
> Es el mismo *clock skew* del que se cuida [Consenso](consenso.md): en distribuido, **la hora de pared no es una fuente de verdad**.

**Ejercicios 13**
13.1 🔁 ¿Por qué un auto-increment de una tabla no sirve para IDs en un sistema shardeado?
13.2 🧠 ¿Por qué un UUID v4 random es malo como clave de un índice B-tree?
13.3 🧠 ¿Qué tres partes tiene un Snowflake ID y qué propiedad gana por tener el timestamp adelante?
13.4 🧠 Snowflake "genera sin coordinación". ¿Qué necesita coordinarse igual al arrancar un nodo, y qué falla concreta produce si el reloj de un nodo **retrocede**?

---

## Módulo 14 — Búsqueda a escala: freshness vs ranking

**Teoría.** Buscar texto a escala (productos, posts, logs) no se hace con `LIKE %query%` (escanea todo, no escala). Se hace con un **índice invertido**: un mapa `término → lista de documentos que lo contienen` (posting list). Buscar "zapatos rojos" = intersecar las posting lists de "zapatos" y "rojos". Es lo que hace Elasticsearch/OpenSearch/Lucene. Dos tensiones a escala:

- **Freshness (qué tan nuevo es lo indexado):** cuando insertás un documento, **no** está buscable hasta que se **indexa**. Indexar en tiempo real es caro (actualizar el índice invertido por cada doc), así que los motores hacen **near-real-time**: bufferizan y refrescan el índice cada cierto intervalo (Elasticsearch: ~1s por default). Hay un **lag** entre escribir y poder buscar — el primo del replication lag de M2.
- **Ranking (qué tan relevante es el orden):** no alcanza con encontrar los docs; hay que **ordenarlos por relevancia** (BM25/TF-IDF, señales de negocio: popularidad, recencia, personalización). Un buen ranking necesita **estadísticas globales** (qué tan raro es un término en todo el corpus) y cómputo — que es caro de mantener fresco sobre un índice que cambia todo el tiempo.

> 🔥 **El trade-off freshness vs ranking en 4 capas.** (1) Qué se rompe: querés resultados **frescos** (lo recién subido aparece ya) **y** bien **rankeados** (lo más relevante primero), pero un buen ranking necesita estadísticas globales que recalcular sobre un índice que muta constantemente es caro. (2) Por qué a esta escala: a millones de docs y escrituras/s, reindexar+rerankear en tiempo real no da. (3) Corto plazo: refresh interval (near-real-time) + ranking aproximado con stats levemente atrasadas. (4) Largo plazo: separar **indexación rápida** (freshness, near-real-time) de **ranking** (a veces en una segunda pasada / batch, o con stats actualizadas periódicamente) — la misma idea Lambda/Kappa de M3 aplicada a búsqueda: respuesta fresca y aproximada ya, ranking refinado después.

> Nota de alcance: este módulo trata la búsqueda a nivel **conceptual** (el trade-off freshness↔ranking es lo que se inyecta en una entrevista de datos a escala); no tiene un build en el laboratorio porque la implementación de un motor de búsqueda (índice invertido, BM25, sharding del índice) da para un módulo aparte. Lo que tenés que poder razonar acá es *por qué* freshness y ranking pelean a escala, no construir Lucene.

**Ejercicios 14**
14.1 🔁 ¿Qué es un índice invertido y por qué `LIKE %x%` no escala?
14.2 🧠 ¿Por qué hay un lag entre escribir un documento y poder buscarlo (freshness)?
14.3 🧠 ¿Por qué freshness y ranking están en tensión a escala?

---

## Etapa 6 — Migrar el sistema y cierre

## Módulo 15 — Partir el monolito: strangler fig

**Teoría.** Tenés un monolito grande y querés migrar a servicios (o a una arquitectura nueva) **sin reescribir todo de cero**. El **big-bang rewrite** (reescribir todo y hacer el switch un día) es **el** anti-patrón clásico: tarda años, el sistema viejo sigue cambiando mientras tanto, y el día del switch fallan mil cosas. La alternativa es el **strangler fig** (Fowler, por la higuera estranguladora que crece alrededor de un árbol hasta reemplazarlo):

1. **Poné un facade/proxy** (un API gateway o reverse proxy) delante del monolito: todo el tráfico pasa por ahí.
2. **Migrá una pieza a la vez:** extraés **una** funcionalidad (un endpoint, un dominio) a un servicio nuevo, y en el proxy **enrutás** ese path al servicio nuevo; el resto sigue yendo al monolito.
3. **Repetí**, extrayendo funcionalidad de a poco, hasta que el monolito quede "estrangulado" (vacío) y lo apagás.

> La clave: **cada paso es chico, reversible y entrega valor** — si la extracción de un servicio sale mal, volvés a enrutar ese path al monolito (rollback barato). Nunca hay un "día del big bang". Es la misma filosofía de expand-contract (módulo 6) a nivel de sistema: **migración incremental, cada paso compatible y reversible**, en vez de un salto grande y riesgoso. Alinea con [DDD](ddd.md) (extraés por *bounded context*) y con [Event-driven](event-driven.md) (los servicios nuevos se integran por eventos/outbox). El dolor real: los **datos** — el servicio nuevo a menudo necesita su propia base, y ahí volvés a expand-contract + doble escritura (módulos 6-8) para migrar los datos sin downtime.

**Ejercicios 15**
15.1 🔁 ¿Por qué el big-bang rewrite es un anti-patrón?
15.2 🧠 Explicá los pasos del strangler fig y por qué cada uno es reversible.
15.3 🧠 ¿Cuál es el dolor real al partir un monolito (más allá del código) y con qué técnica de este módulo lo resolvés?

---

## Módulo 16 — Cómo hablar de datos a escala en una entrevista

**Teoría.** Cierre operativo. Ante un tema de escala de datos/multi-tenancy, estructurá con **las tres cosas** y, ante fallas, **las cuatro capas**.

**Las tres cosas (ejemplo: "escalá la base de datos de un sistema con millones de usuarios"):**
- **(a) Happy path:** "primero agoto vertical + read replicas + cache + índices; cuando no da, shardeo horizontal por `hash(userId)` (reparto parejo), con muchas particiones fijas para rebalancear sin rehashear; IDs Snowflake para localidad de índice".
- **(b) Modelo de falla:** "una hot key celebridad satura un shard → caché + salting + shard dedicado; un tenant ruidoso → quotas + shuffle sharding para acotar el blast radius; un cambio de esquema → expand-contract sin downtime".
- **(c) Recuperación:** "rebalanceo throttled para no degradar; backfill por lotes idempotente; shadow reads antes de cada switch; rollback enrutando al monolito/columna vieja".

**Las muletillas que suenan a que entendiste:**
- *"No shardeo hasta agotar vertical + replicas + cache + índices."*
- *"Hash reparte keys, no carga: una hot key sigue siendo un hot shard → salting o caché."*
- *"Cada migración de esquema es compatible hacia atrás: expand-contract, nunca un rename de un paso."*
- *"¿Read-heavy o write-heavy? B-tree o LSM según eso."*
- *"¿Cuál es el blast radius de un tenant que se vuelve loco? Si es 'todos', falta shuffle sharding o celdas."*
- *"Un ID time-ordered (Snowflake) no es solo único: cuida la localidad del índice."*

> **Cierre.** Operar datos a escala es, en el fondo, **repartir** (sharding) **sin desbalancear** (skew), **aislar** (multi-tenancy) **sin que un cliente arrastre a todos** (blast radius), y **cambiar las cosas en caliente** (expand-contract, strangler) **sin apagar nada**. Y por debajo, entender **cómo el motor guarda los bytes** (WAL, B-tree, LSM) es lo que te deja elegir la herramienta por la carga, no por moda. El que sabe **postergar** el sharding, elegir la **key** con criterio, **acotar el blast radius** y **migrar incremental** —y contar la cascada de 4 capas cuando un shard se calienta o un vecino hace ruido— ya razona como senior. Ahora pasá al [laboratorio práctico](datos-escala-practica.md) y **construí**: el hot shard del sharding range, un Snowflake ID, una migración expand-contract, un mini LSM-tree con su compaction, y el blast radius del shuffle sharding.

**Ejercicios 16**
16.1 🔁 Listá "las tres cosas" aplicadas a escalar la base de datos de un sistema grande.
16.2 🧠 Te preguntan "¿cómo cambiás el tipo de una columna sin downtime?". Respondé con expand-contract.
16.3 🧠 ¿Por qué "postergá el sharding" es de los mejores consejos de este módulo?

---

## Soluciones

**1.1** Tres de: joins cross-shard caros/imposibles; transacciones que dejan de ser ACID locales (saga/2PC); rebalanceo que mueve datos; queries que ahora tienen que saber en qué shard vive el dato (routing); operación más compleja (backups, migraciones por shard).

**1.2** Antes de shardear agotás: (1) **vertical scaling** (máquina más grande — aguanta mucho), (2) **read replicas** (escalan lecturas, M2), (3) **caching** (Redis, saca presión de lectura), (4) **índices** (resuelven muchas queries lentas sin tocar la arquitectura). Porque todas dan escala con **mucha menos** complejidad que el sharding, que es irreversible y caro de operar.

**1.3** Vertical: separás **distintas tablas/dominios** a distintas bases (usuarios en una, pedidos en otra) — por feature. Horizontal: partís **una misma tabla** por filas según una sharding key (usuarios 1-1M en shard A, 1M-2M en shard B) — por volumen de esa tabla.

**2.1** Porque si la key no está en la query, la base no sabe **en qué shard** está el dato → tiene que preguntarle a **todos** los shards (scatter-gather), que es lento y no escala. Con la key en la query, va directo al shard correcto.

**2.2** Range por timestamp: las filas se ordenan por tiempo, así que **todas** las inserciones nuevas (tiempo creciente) caen en el **último** rango/shard → hot shard de escritura, el resto ocioso. Hash: `hash(timestamp)` desparrama los valores cercanos por **todos** los shards → las escrituras nuevas se reparten parejo (al precio de perder los range scans por tiempo).

**2.3** El directorio permite **mover una key a otro shard** fácilmente (solo actualizás el mapeo), cosa que range/hash no (cambiar la asignación implica mover/rehashear). El costo: un **servicio de lookup** extra (indirección en cada query, posible cuello/SPOF, hay que mantenerlo consistente y disponible).

**3.1** Porque `hash % N` ata la asignación al número de nodos: cambiar `N` cambia el resultado para **casi todas** las keys → habría que mover casi todo el dataset. El rebalanceo dejaría el sistema horas moviendo datos.

**3.2** Creás muchas más particiones que nodos desde el inicio (fijas). La key se mapea siempre a la **misma partición** (no depende de N nodos). Agregar/sacar un nodo solo **reasigna particiones enteras** entre nodos (mover bloques), sin recalcular a qué partición va cada key → no hay rehash del dataset.

**3.3** Porque mover datos compite con el **tráfico real**: satura IO/CPU y dispara replication lag (M2), degradando la latencia de los usuarios. Throttled (de a poco, con pausas) mantiene el rebalanceo en segundo plano sin afectar el servicio, aunque tarde más.

**4.1** Un hot shard es un shard que recibe **muchísima más** carga que los demás. Tener muchos shards no lo evita porque el problema no es la **cantidad** de shards sino la **distribución** de la carga: si el tráfico se concentra en una key/rango, ese shard se satura y los demás quedan ociosos — no escalaste.

**4.2** Porque el hash reparte **keys distintas** parejo entre shards, pero **una** key va siempre a **un** shard. Si esa única key (un usuario celebridad, un producto viral) concentra muchísimo tráfico, todo ese tráfico golpea un solo shard. El hash equilibra cantidad de keys, no carga por key.

**4.3** Salting: partís la hot key en varias subkeys (`key:0`, `key:1`, …, `key:n`) que hashean a shards **distintos**. Al **escribir**, elegís una subkey (round-robin/aleatoria) → la carga se reparte entre n shards. Al **leer**, consultás **todas** las subkeys y **agregás** los resultados. Cambiás simplicidad de lectura por reparto de carga.

**5.1** Noisy neighbor: un tenant que consume de más (pico, query pesada, bug) **degrada a los demás** que comparten sus recursos. "Aislar el blast radius" = limitar **a cuántos** alcanza el daño de un tenant, para que no se propague a todos.

**5.2** A cada tenant le asignás un **subconjunto aleatorio de `k`** workers de los N (no todos). Como hay muchísimas combinaciones posibles de `k` sobre N, dos tenants rara vez comparten el mismo subconjunto, y la superposición es chica. Si un tenant se vuelve tóxico, solo afecta a **sus `k`** workers → los demás tenants tienen, casi siempre, al menos un worker sano y siguen funcionando. El blast radius pasa de "todos" a una fracción.

**5.3** Shuffle sharding aísla a nivel de **workers/instancias** (cada tenant usa un subset aleatorio de workers de un pool compartido). Cell-based aísla **todo el stack**: cada tenant vive en una **celda** independiente y completa (sus propios workers, base, etc.), así una falla o deploy malo queda contenido en la celda. Cell-based es aislamiento más fuerte (y más caro); shuffle sharding es más barato y probabilístico.

**6.1** (1) **EXPAND** (agregar lo nuevo, compatible hacia atrás, sin tocar lo viejo), (2) **MIGRATE** (doble escritura + backfill + cambiar lecturas a lo nuevo), (3) **CONTRACT** (borrar lo viejo cuando nadie lo usa).

**6.2** Porque en un rolling deploy conviven instancias **viejas** (aún corriendo) y **nuevas**. Si renombrás la columna, las instancias viejas siguen buscando el nombre **viejo** → fallan hasta que termina el deploy. El cambio no es compatible con la versión que todavía corre.

**6.3** Recién cuando **ningún** código en producción usa la columna vieja: o sea, después de haber cambiado **todas** las lecturas a la nueva y de haber dejado de escribir la vieja, y de confirmar (idealmente con telemetría) que nada la referencia. Antes de eso, el drop rompería código vivo.

**7.1** (1) Por lotes (batched), (2) throttled (pausas para no saturar), (3) idempotente y reanudable, (4) online (no bloquea las escrituras normales).

**7.2** (1) **Lock**: un `UPDATE` masivo puede tomar locks que bloquean el tráfico real; (2) **WAL/transacción gigante**: infla el log y una transacción de horas presiona la DB; (3) **replication lag**: el volumen de cambios atrasa a los followers (M2) y degrada lecturas. Puede tirar abajo la base de producción.

**7.3** Porque durante la fase MIGRATE el código nuevo **ya escribe en ambas** columnas (módulo 8): las filas **nuevas** ya nacen con la columna nueva poblada. El backfill solo tiene que rellenar las filas **viejas** (las que existían antes de activar la doble escritura).

**8.1** Orden: (1) doble escritura (escribir viejo y nuevo), (2) backfill (rellenar lo nuevo con lo histórico), (3) cambiar lecturas a lo nuevo, (4) drop de lo viejo.

**8.2** Porque antes del backfill, lo nuevo tiene **solo** las escrituras recientes (las que hizo la doble escritura), le faltan **todos los datos históricos** → leer de ahí devolvería datos incompletos. Recién tras el backfill lo nuevo está completo (histórico + reciente) y es seguro leerlo.

**8.3** Shadow reads: leés de **ambas** fuentes (vieja y nueva) y **comparás** los resultados (sin servir la nueva todavía), durante un tiempo, para **verificar** que coinciden antes de hacer el switch. Sirve para detectar bugs en la doble escritura/backfill **antes** de depender de lo nuevo en producción.

**9.1** "Write-ahead" = escribir el cambio en el **log primero**, y forzarlo a disco (`fsync`), **antes** de modificar las páginas/estructuras de datos reales. Log primero, datos después.

**9.2** Al reiniciar tras el crash, la base **reproduce el WAL** desde el último checkpoint: reaplica los cambios confirmados que quizás no llegaron a las páginas. Como el commit solo se confirmó **después** de persistir el WAL, todo lo confirmado está en el log → se recupera sin pérdida.

**9.3** Porque escribir al WAL es **secuencial** (append al final de un archivo), y el disco (HDD y también SSD) es mucho más rápido en escrituras secuenciales que en **aleatorias** (que es lo que serían las escrituras directas a páginas dispersas). El WAL convierte muchas escrituras aleatorias en una secuencial.

**10.1** Para **lecturas** (puntuales y range scans): el árbol balanceado y ordenado permite encontrar una clave en `O(log n)` saltos y recorrer rangos por las hojas enlazadas, de forma rápida y predecible.

**10.2** Un page split ocurre cuando una página se llena y hay que insertar más: se **parte en dos** y se reparte el contenido, lo que puede propagarse hacia el padre. Contribuye a la write amplification porque una sola inserción lógica termina escribiendo **varias** páginas (la dividida, la nueva, el padre) más el WAL.

**10.3** Porque la clave a insertar/actualizar puede ir a **cualquier** página del árbol, y esas páginas están dispersas en el disco → modificar "en su lugar" implica I/O **aleatorio** (saltar a la página que toque), a diferencia del append secuencial del WAL/LSM.

**11.1** Memtable: estructura **ordenada en memoria** que recibe las escrituras (rápida, respaldada por WAL). SSTable: archivo **inmutable y ordenado** en disco al que se vuelca la memtable cuando se llena. Compaction: proceso de fondo que **mergea** SSTables, descartando versiones viejas y tombstones, para acotar su cantidad.

**11.2** Read amplification: una key puede estar en la memtable o en **cualquiera** de las N SSTables, así que una lectura podría tener que chequear varias. Los **Bloom filters** (uno por SSTable) responden "esta key **seguro no** está acá" con cero falsos negativos → la lectura **saltea** las SSTables que no la tienen y solo abre las candidatas, reduciendo drásticamente cuántos archivos toca.

**11.3** Porque las SSTables son inmutables: cuando se mergean en compaction (para descartar duplicados/tombstones y mantener pocos archivos), el dato se **copia** a una SSTable nueva. A medida que el dato sube de nivel en sucesivas compactions, se **reescribe varias veces** → el total de bytes escritos en el tiempo es varias veces el tamaño del dato (write amplification), aunque cada escritura individual sea un append barato.

**12.1** **Escritura**: gana el **LSM** (todo append secuencial, sin I/O aleatorio ni splits). **Lectura**: gana el **B-tree** (una sola estructura ordenada, `O(log n)`, predecible; el LSM toca varias SSTables aunque los Bloom filters ayuden).

**12.2** **LSM** (Cassandra/RocksDB/ScyllaDB): la ingestión masiva de métricas es **write-heavy** y de **append** (no updates puntuales), justo donde el LSM brilla (escrituras secuenciales baratas); las lecturas por rango de tiempo se sirven bien con SSTables ordenadas. Un B-tree se ahogaría en el I/O aleatorio de tantas escrituras.

**12.3** Comprime mejor porque las SSTables son **inmutables y ordenadas** (datos homogéneos y contiguos comprimen bien, sin la fragmentación ni las páginas a medio llenar del B-tree). Tiene picos de latencia porque la **compaction** corre en segundo plano y consume IO/CPU de golpe, afectando puntualmente las lecturas/escrituras — el B-tree no tiene ese proceso de fondo.

**13.1** Porque el auto-increment necesita un **único emisor** (la tabla/secuencia) que serialice los IDs → es un cuello y un SPOF, y entre **varios shards** no hay un único emisor compartido (dos shards generarían el mismo id, o necesitarías coordinar, que es lento). No escala horizontalmente.

**13.2** Porque es **aleatorio**: cada inserción cae en una **página al azar** del B-tree → page splits dispersos, cache misses y fragmentación (el árbol no puede agregar al final, tiene que insertar en cualquier lado). Insertar muchos UUIDs v4 toca muchas páginas distintas → I/O aleatorio y mala performance de inserción.

**13.3** (1) **Timestamp** (≈41 bits, ms), (2) **machine/worker id** (≈10 bits), (3) **secuencia** (≈12 bits, contador dentro del mismo ms). Con el timestamp **adelante**, los IDs quedan **aproximadamente ordenados por tiempo** → buena **localidad** de índice (los inserts nuevos van contiguos al final del B-tree), generados localmente sin coordinación.

**13.4** Lo que se coordina al arrancar es la **asignación del `worker id` único**: como cada proceso lleva su propia secuencia, dos nodos con el mismo worker id generan en paralelo el mismo `(timestamp, worker, secuencia)` → IDs **duplicados**, así que el id se reparte por config estática o por un registro central (ZooKeeper/etcd) — coordinación en el *bootstrap*, no en cada ID (y un posible SPOF de arranque). Si el **reloj retrocede** (corrección de NTP, migración de VM, leap second), el generador puede re-emitir un `(timestamp, secuencia)` ya usado → **colisión de IDs**. El timestamp embebido **es wall-clock** (no un reloj monótono, que no se traduce a tiempo real); la garantía de no-retroceso la pone el **software**: recuerda el último timestamp y, si el reloj actual es menor, **se niega a generar** (el Snowflake original falla con *"Clock is moving backwards"*; otras implementaciones esperan a que el reloj alcance).

**14.1** Un índice invertido es un mapa `término → lista de documentos que lo contienen` (posting list); buscar = intersecar/unir posting lists. `LIKE %x%` no escala porque tiene que **escanear cada fila** comparando el patrón (full scan, sin poder usar un índice por el comodín inicial) → O(n) sobre todo el corpus.

**14.2** Porque indexar (actualizar el índice invertido) en tiempo real por cada documento es **caro**; los motores **bufferizan** y **refrescan** el índice cada cierto intervalo (near-real-time, ~1s en Elasticsearch). Entre escribir el doc y el siguiente refresh, el doc **existe pero no es buscable** → lag de freshness (primo del replication lag de M2).

**14.3** Porque un buen **ranking** necesita **estadísticas globales** del corpus (rareza de términos, popularidad) y cómputo, que es caro de mantener **fresco** sobre un índice que **cambia constantemente** (cada escritura cambia las stats). Querés resultados frescos **y** bien rankeados, pero recalcular el ranking en tiempo real a escala no da → hay que separar indexación rápida (freshness) de ranking (refinado después/con stats algo atrasadas).

**15.1** Porque reescribir todo y hacer el switch un día: tarda **años** durante los cuales el sistema viejo sigue cambiando (el nuevo persigue un blanco móvil), concentra **todo el riesgo** en un único big-bang (mil cosas fallan a la vez el día del switch), y **no entrega valor** hasta el final. Es alto riesgo, alto costo, retorno tardío.

**15.2** (1) Facade/proxy delante del monolito (todo el tráfico pasa por ahí). (2) Extraés **una** funcionalidad a un servicio nuevo y enrutás **ese path** al nuevo; el resto sigue en el monolito. (3) Repetís hasta vaciar el monolito. Cada paso es reversible porque si la extracción falla, **volvés a enrutar ese path al monolito** (rollback barato, sin big-bang).

**15.3** El dolor real son los **datos**: el servicio nuevo suele necesitar su propia base, así que hay que **migrar los datos** del monolito sin downtime. Se resuelve con **expand-contract + doble escritura + backfill** (módulos 6-8): el servicio nuevo y el monolito conviven escribiendo/leyendo de forma compatible hasta completar la migración de datos.

**16.1** (a) Happy path: agotar vertical+replicas+cache+índices → shardear por `hash(userId)` con muchas particiones fijas → IDs Snowflake. (b) Modelo de falla: hot key → caché+salting+shard dedicado; tenant ruidoso → quotas+shuffle sharding; cambio de esquema → expand-contract. (c) Recuperación: rebalanceo throttled, backfill por lotes idempotente, shadow reads antes del switch, rollback enrutando a lo viejo.

**16.2** "Con expand-contract: (1) **agrego** una columna nueva con el tipo nuevo (no toco la vieja); (2) deployo código que **escribe en ambas** y **backfilleo** la nueva desde la vieja por lotes idempotentes; (3) hago **shadow reads** para confirmar que coinciden, cambio las **lecturas** a la nueva; (4) dejo de escribir la vieja y la **dropeo**. Cada paso es compatible con el código que sigue corriendo, así que cero downtime."

**16.3** Porque el sharding es **irreversible, caro de operar y multiplica la complejidad** (joins/transacciones/routing/rebalanceo), mientras que vertical scaling + read replicas + cache + índices dan **muchísima** escala con una fracción del costo y del riesgo. Postergarlo evita comerte la complejidad antes de necesitarla — y muchas veces no la necesitás nunca. Shardear temprano es de los errores más caros.
