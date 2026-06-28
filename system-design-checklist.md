# Checklist de entrevista: los conceptos que te inyecta el interviewer

**Los problemas y conceptos de system design que caen en rondas senior/staff (Google L5 · Meta E5 · Amazon L6 · Uber/Salesforce staff) · cada uno resuelto en 4 capas · 2026**

> Cómo usar este checklist: no es un módulo para leer de corrido, es un **mapa de auto-evaluación**. Para cada concepto, preguntate: *si el interviewer me lo inyecta en mi diseño o lo pide como follow-up, ¿puedo razonarlo en voz alta con claridad?* Si la respuesta es "más o menos", seguí el link al módulo del hub que lo desarrolla. El objetivo es que en una ronda de 1 hora puedas tomar cualquiera de estos y responder con estructura, no con un nombre suelto.

**El marco — 4 capas.** Una respuesta fuerte a cualquiera de estos tiene cuatro capas (el mismo marco que aplicamos a las fallas en [Diseño de sistemas](system-design.md) y [Concurrencia](concurrencia.md)):
1. **Qué se rompe** exactamente (o, para un building block: qué problema resuelve).
2. **Por qué** a *esta* escala o forma de tráfico (no genérico).
3. **Control de corto plazo** que frena el *blast radius*.
4. **Cambio de diseño de largo plazo** que baja la probabilidad de que se repita.

Y toda respuesta de diseño tiene **3 cosas**: una **forma para el happy path**, un **modelo de falla** y un **proceso de recuperación**. El ejemplo canónico es el *retry storm*: respuesta débil = "usá backoff"; respuesta fuerte = "la latencia sube → los timeouts disparan → los clientes reintentan en paralelo → más carga en la dependencia → más latencia: el mecanismo de recuperación se volvió **amplificador de carga**" + las 4 capas.

**Cómo leer cada entrada:** *qué se rompe* → **(1)** qué · **(2)** por qué a escala · **(3)** control corto · **(4)** rediseño largo → **dónde estudiarlo** en el hub. La marca ✅ = cubierto a fondo en el hub; 🟡 = parcial; 🆕 = módulo a fondo en el roadmap (acá tenés la respuesta de entrevista).

**Tipos de ejercicio:** 🧠 criterio/análisis. Al final hay **drills de auto-evaluación** ("el interviewer inyecta X, respondé en 4 capas") con soluciones agrupadas.

---

## Parte 1 — Los conceptos que más se inyectan (el set de 27)

### Nivel principiante — los fallos básicos que todos deben saber

**1. Thundering herd.** Muchos clientes despiertan a la vez sobre el mismo recurso recién disponible. **(1)** una cache que expira, un servicio que vuelve o un lock que se libera recibe una avalancha simultánea. **(2)** miles de clientes **sincronizados** (mismo TTL, mismo evento de recuperación) golpean en el mismo instante. **(3)** *jitter* para desincronizar + *request coalescing* (single-flight: una sola llamada real, las demás esperan su resultado). **(4)** colas que absorben el pico, backoff con jitter, calentamiento gradual. → ✅ [redis.md](redis.md) (cache stampede), [system-design.md](system-design.md) mód 5.

**2. Cache stampede.** El caso específico del herd en caché: una key caliente expira y N requests van todas a la base. **(1)** la key popular vence → todos los *misses* pegan a la DB a la vez. **(2)** con alta lectura, una sola key concentra miles de QPS que la DB no aguanta de golpe. **(3)** TTL con *jitter* + lock/single-flight (solo uno recomputa, el resto espera). **(4)** *refresh-ahead* (recomputar antes de expirar), o no-expiry con refresh asíncrono. → ✅ [redis.md](redis.md) mód 4, [system-design.md](system-design.md) mód 5.

**3. N+1 query.** Una query para la lista + N queries, una por cada hijo. **(1)** cargás N entidades y por cada una hacés otra consulta → N+1 round-trips. **(2)** a escala la latencia se multiplica por N y satura el connection pool. **(3)** *eager load*/JOIN, batch con `IN (...)`, DataLoader en GraphQL. **(4)** proyecciones a medida, denormalización selectiva, detección en tests/logs (contar queries por request). → ✅ [postgresql.md](postgresql.md) mód 11, [graphql.md](graphql.md).

**4. Hot partition / hot key.** Una partición o key recibe muchísimo más tráfico que el resto. **(1)** la clave de sharding concentra carga (una celebridad, un país, un producto viral) → un nodo al rojo mientras los demás duermen. **(2)** la distribución real es ley de potencias, no uniforme; el hash "balanceado" no salva un valor súper-popular. **(3)** cachear esa key, replicarla, rate-limit por key. **(4)** elegir mejor clave de sharding (alta cardinalidad), *split* de la hot key (sufijo), o tratamiento especial del caso patológico (ver fan-out de celebridad). → ✅ [system-design.md](system-design.md) mód 4, [nosql.md](nosql.md), [system-design-casos.md](system-design-casos.md) (typeahead/feed).

**5. Single point of failure (SPOF).** Un componente sin redundancia cuya caída tumba todo. **(1)** un único nodo/instancia/coordinador en el camino crítico → si muere, cae el sistema. **(2)** el scaling vertical y los componentes "uno solo" (un NAT, un líder, una DB master) son SPOF por diseño. **(3)** health checks + failover manual, hot standby. **(4)** redundancia (N+1, multi-AZ), servicios *stateless* detrás de un LB, replicación con failover automático. → ✅ [system-design.md](system-design.md) mód 3/10, [aws-practica.md](aws-practica.md) (NAT/multi-AZ).

**6. Retry storm.** Los reintentos se vuelven amplificador de carga. **(1)** una dependencia se pone lenta → timeouts → todos reintentan en paralelo → más carga → más lenta: cascada. **(2)** con miles de clientes y reintentos sincronizados e inmediatos, el propio retry multiplica el tráfico justo cuando el sistema está caído. **(3)** backoff **exponencial + jitter**, límite de reintentos, *circuit breaker* que corta. **(4)** *token bucket* de reintentos, *deadlines* propagados, *load shedding* en el servidor. → ✅ [system-design.md](system-design.md) mód 10, [concurrencia.md](concurrencia.md) mód 19, [redis.md](redis.md).

**7. Backpressure.** El productor va más rápido de lo que el consumidor puede procesar. **(1)** una cola crece sin límite (o un stream de Node) → memoria que explota o latencia que se dispara. **(2)** a escala, picos de entrada superan la capacidad sostenida del consumidor; sin freno, el sistema se cae en vez de degradar. **(3)** colas **acotadas** que bloquean/rechazan al productor, `pause()`/`drain` en streams. **(4)** flujo *pull*, créditos/ventanas, autoscaling del consumidor, *load shedding*. → ✅ [concurrencia.md](concurrencia.md) mód 17, [system-design.md](system-design.md) mód 8, [nodejs.md](nodejs.md) (streams).

**8. Duplicate requests / idempotency gap.** Un reintento ejecuta dos veces un efecto que debía ocurrir una. **(1)** se pierde el ACK de un `POST` con efecto (un cobro), el cliente reintenta y cobra dos veces. **(2)** la red pierde respuestas siempre; bajo timeouts/reintentos automáticos la ventana ACK-perdido es permanente. **(3)** **idempotency key** + reserva atómica antes del efecto (INSERT único / `SET NX`). **(4)** contrato idempotente documentado, dedup en el sink, store-and-return de la respuesta. → ✅ [api-design.md](api-design.md) mód 4, [system-design.md](system-design.md) mód 8, [system-design-casos.md](system-design-casos.md).

**9. Stale cache / read-after-write inconsistency.** Escribís y al leer ves el valor viejo. **(1)** tras un write, la lectura pega a una réplica rezagada o a una cache no invalidada → el usuario no ve su propio cambio. **(2)** con réplicas de lectura y *replication lag*, o con cache + TTL, la ventana de inconsistencia es real bajo carga. **(3)** *read-your-writes* (leer del primario tras escribir, o *sticky* por sesión), invalidar la cache en el write. **(4)** *write-through*/*write-around* deliberado, versionado (ETag/If-Match), consistencia causal donde importe. → 🟡 [system-design.md](system-design.md) mód 5/4, [api-design.md](api-design.md) mód 12 (ETag) — *profundización: módulo de Replicación.*

### Nivel intermedio — coordinación y entrega

**10. Distributed rate limiting.** Limitar por cliente a través de todas las instancias. **(1)** un contador en memoria por instancia deja pasar N×(instancias); el límite real se filtra. **(2)** a 500K req/s repartidas en miles de nodos, el estado del límite tiene que ser compartido y atómico. **(3)** estado en Redis con **script Lua atómico** (token bucket / fixed-sliding window). **(4)** límite local de respaldo (fail-open), cuotas por tier, decisión fail-open vs fail-closed. → ✅ [system-design-casos.md](system-design-casos.md) caso 6, [redis.md](redis.md) mód 6, [api-design.md](api-design.md) mód 11.

**11. Leader election.** Elegir un único coordinador entre nodos iguales, y reelegir si cae. **(1)** sin líder único, dos nodos creen ser el dueño (split-brain) y corrompen estado; con un líder fijo, su caída paraliza. **(2)** a escala distribuida las particiones de red y los crashes son normales; necesitás acordar quién manda **sin** un árbitro central. **(3)** un lock con lease/TTL en un store fuerte (etcd/ZooKeeper) como elección simple. **(4)** consenso real: **Raft** (election timeout aleatorizado, voto por mayoría = **quorum N/2+1**, log replication) o Paxos; reads linealizables vía *read index*. → 🆕 *módulo Consenso (en el roadmap)* — relacionado: [event-driven.md](event-driven.md).

**12. Distributed locking & lease expiry.** Un lock que vale entre procesos/máquinas, y qué pasa si el dueño muere. **(1)** un lock sin expiración se queda tomado para siempre si el dueño crashea (deadlock distribuido); uno con TTL puede expirar **mientras** el dueño sigue trabajando → dos dueños. **(2)** sin reloj compartido ni garantía de que el trabajo termine antes del TTL, el doble dueño es real. **(3)** lock con TTL + renovación (lease), `SET NX PX`. **(4)** **fencing tokens** (un nº monotónico que el recurso protegido valida y rechaza al dueño viejo), Redlock con sus caveats (Kleppmann). → 🟡 [redis.md](redis.md) (Redlock/fencing), [concurrencia.md](concurrencia.md) — *profundización: módulo Consenso.*

**13. Quorum reads vs quorum writes.** Cuántas réplicas deben confirmar una lectura/escritura. **(1)** si leés de pocas réplicas tras escribir en pocas, podés leer un valor viejo (inconsistencia). **(2)** en un store replicado N veces, sin coordinación las réplicas divergen temporalmente. **(3)** exigir **R + W > N** (p. ej. N=3, W=2, R=2) garantiza solapamiento → la lectura ve la última escritura. **(4)** *read repair* + *hinted handoff* + anti-entropy (Merkle) para converger; ajustar R/W según prioridad lectura vs escritura. → 🆕 *módulo Consenso/Replicación* — base CAP/PACELC en [system-design.md](system-design.md) mód 6.

**14. Fan-out on write vs on read.** ¿Precomputás al escribir o juntás al leer? **(1)** push (fan-out on write) hace lecturas O(1) pero una escritura de celebridad son millones de writes; pull abarata escritura pero la lectura es cara. **(2)** el grafo social es desbalanceado: el costo de un write no es uniforme. **(3)** detectar la cuenta caliente y saltear su fan-out. **(4)** **híbrido**: push para usuarios normales, pull para celebridades, merge en lectura. → ✅ [system-design-casos.md](system-design-casos.md) caso 1.

**15. Out-of-order event processing.** Los eventos llegan en distinto orden del que ocurrieron. **(1)** procesás un "update" antes que el "create", o aplicás un estado viejo sobre uno nuevo → estado corrupto. **(2)** con múltiples productores, particiones y reintentos, el orden global no existe; solo hay orden por partición/clave. **(3)** clave de partición por entidad (todos sus eventos en orden), número de secuencia/versión que descarta lo viejo. **(4)** event-time + **watermarks** (ver #26), CRDTs/LWW para conmutatividad, procesar por *stream key*. → 🟡 [event-driven.md](event-driven.md) — *profundización: módulo Streams.*

**16. Dead letter queues & poison messages.** Un mensaje que siempre falla bloquea o se pierde. **(1)** un mensaje "veneno" (malformado, o que dispara un bug) se reintenta infinito y traba el consumidor, o se descarta en silencio. **(2)** a escala siempre hay mensajes corruptos; sin red de contención, uno solo frena la cola entera. **(3)** límite de reintentos → mandar a una **DLQ** tras N fallos. **(4)** alerta sobre la DLQ (un mensaje muerto es algo que alguien debe ver), reproceso manual, esquema/validación en el borde. → ✅ [redis.md](redis.md) mód 9, [event-driven.md](event-driven.md), [system-design.md](system-design.md) mód 8.

**17. Zero-downtime schema migration.** Cambiar el esquema sin parar el servicio. **(1)** un `ALTER` bloqueante, o renombrar/quitar una columna que el código viejo aún usa, rompe la app en pleno deploy. **(2)** con miles de filas y despliegues *rolling*, conviven código viejo y nuevo contra el mismo esquema. **(3)** cambios **aditivos** primero (columna nueva nullable), migración de datos en *batches* no bloqueantes. **(4)** **expand-contract**: expand (agregar lo nuevo) → backfill → dual-write/dual-read → switch → contract (quitar lo viejo), cada paso compatible hacia atrás. → 🟡 [api-design.md](api-design.md) mód 8 (expand-contract), [aws-practica.md](aws-practica.md) — *profundización: módulo Datos a escala.*

**18. Circuit breaker & cascading failure.** Cortar la llamada a una dependencia caída para no arrastrar al resto. **(1)** un servicio lento consume todos los threads/conexiones del que lo llama → la falla se propaga hacia arriba (cascada). **(2)** con dependencias sincrónicas encadenadas, un eslabón lento agota recursos de toda la cadena. **(3)** **circuit breaker** (open/half-open/closed) que falla rápido cuando la dependencia está mal. **(4)** *bulkheads* (aislar pools por dependencia), timeouts agresivos, degradación graceful (responder con caché/parcial). → ✅ [system-design.md](system-design.md) mód 10, [nestjs-senior.md](nestjs-senior.md), [concurrencia.md](concurrencia.md).

### Nivel avanzado — escala global y casos patológicos

**19. Split a monolith safely.** Extraer servicios de un monolito sin romper. **(1)** un "big bang" reescribe todo y rompe integraciones invisibles; o el servicio extraído comparte la DB y siguen acoplados. **(2)** a escala de equipo, el monolito frena los deploys, pero partirlo mal crea un "monolito distribuido" peor. **(3)** **strangler fig**: poner un proxy/fachada y migrar endpoints de a uno. **(4)** separar por **bounded context** (DDD), una DB por servicio, comunicación por eventos (outbox/CDC), medir antes de cortar. → 🟡 [ddd.md](ddd.md) (bounded contexts), [event-driven.md](event-driven.md), [microfrontends.md](microfrontends.md) (strangler) — *profundización: módulo Datos a escala.*

**20. Multi-region failover.** Sobrevivir a la caída de una región entera. **(1)** todo en una región → un outage regional te baja del todo; replicar mal → datos inconsistentes o *split-brain* al failover. **(2)** a escala global, latencia y soberanía de datos empujan a multi-región, pero la consistencia entre regiones es cara. **(3)** **active-passive**: una región sirve, otra es standby con replicación + DNS/health-check failover. **(4)** **active-active**: ambas sirven y escriben → necesitás resolución de conflictos (#21), o particionar por región (datos de cada usuario en su región). → 🟡 [aws.md](aws.md)/[aws-practica.md](aws-practica.md) — *profundización: módulo Replicación.*

**21. Active-active conflict resolution.** Dos regiones escriben el mismo dato a la vez. **(1)** sin coordinación, dos writes concurrentes en regiones distintas → ¿cuál gana? Uno se pierde o quedan divergentes. **(2)** en active-active la latencia inter-región hace inviable coordinar cada write (volverías a single-master). **(3)** **last-write-wins** por timestamp (simple, pierde escrituras concurrentes). **(4)** **CRDTs** (estructuras que convergen sin coordinación: contadores, OR-sets), **vector clocks** para detectar concurrencia y resolver por política de dominio. → 🟡 [mobile-system-design.md](mobile-system-design.md) (CRDTs/LWW offline) — *profundización: módulo Replicación.*

**22. Change Data Capture vs dual writes.** Propagar cambios de la DB a otros sistemas (cache, search, eventos). **(1)** el *dual write* (escribir a la DB y publicar el evento por separado) no es atómico: si una falla, quedan desincronizados. **(2)** a escala, esa ventana de fallo ocurre seguido y corrompe los consumidores downstream. **(3)** patrón **outbox** (escribir el evento en la misma transacción que el dato, publicar después). **(4)** **CDC** real leyendo el *log* de la DB (WAL/binlog → Debezium), que captura todo cambio sin tocar el código de escritura. → ✅ [event-driven.md](event-driven.md) mód 3, [nosql.md](nosql.md) (Streams), [system-design-casos.md](system-design-casos.md) caso 5.

**23. Search index freshness vs ranking quality.** El índice de búsqueda fresco vs el ranking bueno. **(1)** indexar en tiempo real da resultados frescos pero ranking pobre (sin señales agregadas); recalcular ranking en batch da calidad pero datos viejos. **(2)** a escala, reindexar todo con ranking completo es caro y lento; servir en vivo exige precómputo. **(3)** índice incremental para frescura + boost simple en vivo. **(4)** arquitectura de dos capas: ingest rápido (fresco) + pipeline batch de ranking (calidad) que publica versiones; conciliar al servir. → 🟡 [postgresql.md](postgresql.md) (FTS), [vector-dbs.md](vector-dbs.md)/[rag.md](rag.md) — *profundización: módulo Datos a escala.*

**24. Rebalancing shards under skewed traffic.** Redistribuir datos cuando un shard se calienta. **(1)** el tráfico sesgado convierte un shard "balanceado" en hot shard; mover datos para rebalancear genera más carga justo cuando ya está saturado. **(2)** a escala, agregar/quitar nodos con hashing simple (`mod N`) **remapea casi todo** → tormenta de movimiento de datos. **(3)** **consistent hashing** con *virtual nodes* (mover solo una fracción al cambiar la topología). **(4)** **shuffle sharding** y split de hot shards, rebalanceo gradual con límites de tasa, aislar tenants ruidosos en su shard. → 🟡 [system-design.md](system-design.md) mód 4 (sharding) — *profundización: módulos Consenso (consistent hashing) y Datos a escala.*

**25. Noisy neighbor (multi-tenant).** Un tenant consume recursos y degrada a los demás. **(1)** un cliente manda un pico (o una query cara) y satura CPU/IO/conexiones compartidas → todos los demás tenants sufren. **(2)** en infraestructura compartida, sin aislamiento un solo tenant puede monopolizar el recurso. **(3)** rate-limit y cuotas por tenant, *timeouts* por query, prioridades. **(4)** aislamiento: pools/colas separadas por tier (*fair queuing*), cgroups/ResourceQuota (k8s), **shuffle sharding**, shard dedicado para el ruidoso. → 🟡 [kubernetes.md](kubernetes.md) (limits/quotas), [nestjs-senior.md](nestjs-senior.md) (multi-tenant) — *profundización: módulo Datos a escala.*

**26. Watermarks / late-arriving events.** En streams, saber cuándo una ventana de tiempo está "completa". **(1)** agregás por ventana de tiempo (clics por minuto), pero un evento llega tarde (red, móvil offline) tras cerrar la ventana → conteo incorrecto. **(2)** en *stream processing* el *event time* (cuándo pasó) difiere del *processing time* (cuándo llega); sin manejo, perdés o duplicás. **(3)** **watermark** = "ya no espero eventos anteriores a T" → cierra la ventana; *allowed lateness* para los rezagados. **(4)** event-time windows (tumbling/sliding/session) + política de *late data* (descartar, side-output, recomputar), checkpointing para exactly-once. → 🆕 *módulo Streams (en el roadmap).*

**27. Exactly-once vs practical deduplication.** "Exactly-once" no existe en la red; se aproxima. **(1)** la entrega es *at-least-once* (reintentos) o *at-most-once* (puede perder); "exactly-once" puro entre sistemas con efectos externos es imposible. **(2)** a escala, los reintentos y crashes hacen que los duplicados sean la norma. **(3)** *at-least-once* + **consumidor idempotente** (dedup por id de evento) = *effectively-once*. **(4)** dedup con ventana/store de ids procesados, idempotency keys end-to-end, transacciones donde el sink lo permita (Kafka EOS dentro del mismo sistema). → ✅ [system-design.md](system-design.md) mód 8, [event-driven.md](event-driven.md), [redis.md](redis.md) mód 10.

---

## Parte 2 — Building blocks y conceptos distribuidos que también caen

No son "fallas" sino piezas y técnicas que el interviewer espera que conozcas. Formato: *qué es / qué resuelve* → **cómo** · **trade-off** · **cuándo**.

**A. Consistent hashing.** Distribuir keys entre nodos minimizando el remapeo al agregar/quitar nodos. **Cómo:** nodos y keys sobre un *anillo* de hash; cada key va al nodo siguiente en el anillo; *virtual nodes* para balancear. **Trade-off:** más complejo que `hash mod N`, pero `mod N` remapea casi todo al cambiar N. **Cuándo:** caches distribuidas, sharding de datos, balanceo con afinidad. → 🆕 *módulo Consenso.* (Base: [system-design.md](system-design.md) mód 4.)

**B. Consenso: Raft / Paxos.** Hacer que N nodos acuerden un valor/orden a pesar de fallos. **Cómo:** Raft = leader election (quorum) + log replication + safety; un valor se *commitea* cuando la mayoría lo replica. **Trade-off:** correctitud fuerte a costa de latencia (round-trips de quorum) y disponibilidad bajo partición (CP). **Cuándo:** config/metadata coordinada (etcd→Kubernetes, Consul; ZooKeeper en HBase/Solr — Kafka hoy usa su propio Raft, **KRaft**, y quitó ZooKeeper en la 4.0). → 🆕 *módulo Consenso.*

**C. 2PC / 3PC (commit distribuido).** Atomicidad *all-or-nothing* entre varias bases/servicios. **Cómo:** un coordinador hace *prepare* → si todos votan sí, *commit*; si no, *rollback*. **Trade-off:** consistencia fuerte pero **bloqueante** (locks tomados durante todo el round-trip) y el coordinador es SPOF; 3PC mitiga el bloqueo, no lo elimina. **Cuándo:** transacciones cortas con cero tolerancia a anomalías; si no, **Saga** (compensaciones, eventual). → 🆕 *módulo Replicación.* (Saga ✅ en [event-driven.md](event-driven.md) mód 9.)

**D. Gossip / failure detection / heartbeat.** Que cada nodo sepa quién está vivo sin un registro central. **Cómo:** cada nodo manda *heartbeats* a un subconjunto aleatorio; la membresía se propaga epidémicamente (gossip). **Trade-off:** detección eventual (no instantánea) a cambio de escalar a miles de nodos sin SPOF. **Cuándo:** clusters grandes (Cassandra, DynamoDB-style), service mesh. → 🆕 *módulo Consenso.* (Heartbeats WS en [tiempo-real.md](tiempo-real.md).)

**E. Anti-entropy: Merkle trees / read repair / hinted handoff.** Reconciliar réplicas divergentes barato. **Cómo:** *Merkle tree* compara hashes de rangos para transferir solo lo que difiere; *read repair* corrige en la lectura; *hinted handoff* guarda escrituras para un nodo caído y se las entrega al volver. **Trade-off:** convergencia eventual con mínimo tráfico vs complejidad operativa. **Cuándo:** stores AP con quorum (Cassandra/Dynamo). → 🆕 *módulo Consenso/Replicación.*

**F. Logical clocks: Lamport / vector clocks.** Ordenar eventos sin reloj global confiable. **Cómo:** *Lamport* da un **orden parcial** consistente con la causalidad (si a→b entonces L(a)<L(b)), del que se deriva un orden total rompiendo empates por id de nodo — pero **no distingue concurrente de causal** (L(a)<L(b) no implica a→b); *vector clocks* sí detectan si dos eventos son concurrentes o causales, que es justamente por lo que existen. **Trade-off:** los vector clocks crecen con la cantidad de actores, pero distinguen "concurrente" de "viejo" (clave para resolver conflictos). **Cuándo:** active-active, versionado causal, CRDTs. → 🆕 *módulo Replicación.*

**G. Diseño geoespacial: geohash / quadtree.** Buscar "lo cercano" eficientemente. **Cómo:** *geohash* codifica lat/long en un string donde el prefijo = proximidad; *quadtree* particiona el espacio recursivamente; consultás celdas vecinas. **Trade-off:** geohash simple e indexable (string) pero con bordes; quadtree se adapta a densidad pero es más complejo. **Cuándo:** "drivers cerca" (Uber), "amigos cerca", mapas, delivery. → 🆕 *módulo Geoespacial.*

**H. Unique ID distribuido (Snowflake).** Generar IDs únicos, ordenables, sin un contador central. **Cómo:** Snowflake = timestamp + machine id + secuencia (64 bits), ordenable por tiempo y sin coordinación; alternativa ULID. **Trade-off:** ordenable y descentralizado vs depende de relojes razonablemente sincronizados. **Cuándo:** PKs distribuidas, `msgId`/`eventId` ordenables. → 🟡 mencionado en [postgresql.md](postgresql.md)/[system-design-casos.md](system-design-casos.md) — *profundización: módulo Datos a escala.*

**I. Storage engines: WAL / LSM-tree vs B-tree / compaction.** Cómo la DB persiste y lee. **Cómo:** *WAL* (write-ahead log) garantiza durabilidad; *B-tree* optimiza lecturas (Postgres/MySQL); *LSM-tree* optimiza escrituras (memtable + SSTables + *compaction*: Cassandra/RocksDB). **Trade-off:** B-tree = lecturas rápidas, escrituras in-place (y también usa WAL para durabilidad); LSM = escrituras rápidas, pero *read amplification* (una lectura toca varias SSTables) y *write amplification* (la compaction reescribe datos varias veces). **Cuándo:** elegir DB según ratio lectura/escritura. → 🟡 [postgresql.md](postgresql.md) (MVCC/índices) — *profundización: módulo Datos a escala.*

**J. Batch vs stream (Lambda / Kappa).** Procesar datos en lotes vs en tiempo real. **Cómo:** *Lambda* = capa batch (exacta, lenta) + capa speed (aproximada, rápida) que se concilian; *Kappa* = solo stream, reprocesando desde el log cuando hace falta. **Trade-off:** Lambda da exactitud + frescura a costa de mantener dos pipelines; Kappa simplifica con uno solo. **Cuándo:** analytics/ML features, métricas, agregaciones a escala. → 🆕 *módulo Streams.*

**K. OLTP vs OLAP / warehouse / columnar.** Transaccional vs analítico. **Cómo:** OLTP = muchas transacciones chicas (row-store, normalizado); OLAP = pocas queries que escanean mucho (column-store, denormalizado, *data warehouse*). **Trade-off:** mezclar ambos en una DB mata a las dos; separarlos exige ETL/CDC. **Cuándo:** cuando los reportes/analytics empiezan a competir con el tráfico transaccional. → 🆕 *módulo Streams/Datos.*

**L. Kafka internals.** Cómo funciona un log distribuido por debajo. **Cómo:** *topics* en **particiones** (orden por partición), *offsets* por consumidor, *consumer groups* (cada partición a un consumidor del grupo), retención por tiempo/tamaño. **Trade-off:** orden y paralelismo se equilibran con el nº de particiones (más particiones = más paralelismo, menos orden global). **Cuándo:** event streaming, desacople, *replay*. → 🟡 conceptual en [event-driven.md](event-driven.md) — *profundización: módulo Streams.*

**M. CDN (push vs pull).** Servir contenido desde el borde, cerca del usuario. **Cómo:** *pull* (la CDN cachea on-demand en el primer miss); *push* (vos subís el contenido al borde). **Trade-off:** pull es simple pero el primer request es lento (cold); push controla pero hay que gestionar la distribución. **Cuándo:** estáticos, media, y hasta respuestas de API cacheables. → 🟡 [reverse-proxy.md](reverse-proxy.md)/[system-design.md](system-design.md) — *profundización: módulo Building blocks.*

**N. Estructuras probabilísticas: Bloom / count-min / HyperLogLog.** Responder aproximado con memoria mínima. **Cómo:** *Bloom* = "¿está?" (falsos positivos, nunca negativos); *count-min* = frecuencia aproximada; *HLL* = conteo de únicos (cardinalidad). **Trade-off:** error acotado a cambio de usar una fracción de la memoria de la estructura exacta. **Cuándo:** dedup a escala (crawler), "ya lo vi", top-k, usuarios únicos. → 🟡 Bloom ✅ en [system-design-casos.md](system-design-casos.md) caso 7 — *count-min/HLL: módulo Building blocks.*

**O. Cell-based architecture (blast radius).** Particionar el sistema en **celdas** independientes (cada una con su stack completo) para acotar el radio de una falla. **Cómo:** cada celda atiende a un subconjunto de usuarios/tenants; un *router* fino los mapea; un problema en una celda afecta solo a esa fracción, no a todos. **Trade-off:** aislamiento de fallas y de *noisy neighbors* a costa de más complejidad operativa y routing. **Cuándo:** servicios críticos a gran escala (Amazon lo inyecta seguido como follow-up de multi-región/multi-tenant). → 🆕 *módulo Datos a escala / Replicación.*

**P. Feature flags / kill-switch.** Activar/desactivar comportamiento sin deploy — un control de primer orden para frenar el *blast radius*. **Cómo:** un flag remoto decide en runtime si una feature corre; el *kill-switch* la apaga al instante ante un incidente; los *canary*/rollouts graduales exponen el cambio a un % creciente. **Trade-off:** control y reversibilidad inmediata a cambio de deuda de flags (hay que limpiarlos) y combinatoria de estados a testear. **Cuándo:** todo deploy riesgoso, migraciones (expand-contract #17), y como freno de emergencia que no requiere rollback. → 🆕 *módulo Datos a escala* — relacionado con degradación graceful (#18).

---

## Cómo se conecta con el hub (mapa de estudio)

| Tema | Dónde está hoy | Módulo nuevo en el roadmap |
|---|---|---|
| Fallas de caché/carga (1,2,4,6,7) | [redis.md](redis.md), [system-design.md](system-design.md), [concurrencia.md](concurrencia.md) | — (cubierto) |
| Idempotencia/entrega (8,27,16) | [api-design.md](api-design.md), [event-driven.md](event-driven.md) | — (cubierto) |
| Consenso/coordinación (11,12,13 + A,B,D,E,F) | parcial | **Consenso y coordinación distribuida** |
| Replicación/multi-región (9,20,21 + C) | parcial | **Replicación, multi-región y conflictos** |
| Streams/analytics (15,26 + J,K,L) | parcial | **Datos en movimiento: streams, batch y analytics** |
| Datos a escala/operación (17,19,23,24,25 + H,I) | parcial | **Datos a escala y multi-tenancy** |
| Building blocks/geo (G,M,N + leaderboard) | parcial | **Building blocks y diseño geoespacial** |

---

## Drills de auto-evaluación

> Simulá la ronda: el interviewer **inyecta** el problema en tu diseño. Respondé en **4 capas** (qué se rompe / por qué a esta escala / control corto / rediseño largo) en voz alta, y recién después mirá la solución.

8.1 🧠 Estás diseñando un feed. El interviewer dice: "una celebridad con 50M seguidores postea. ¿Qué pasa?" Respondé en 4 capas.
8.2 🧠 "Tu cache de productos usa TTL de 60s. A medianoche expira la home y la DB se cae. ¿Por qué y cómo lo evitás?" (nombrá el fenómeno).
8.3 🧠 "Tu servicio de pagos a veces cobra dos veces. ¿Por qué pasa y cómo lo arreglás de raíz?"
8.4 🧠 "Necesitás un único job que corra en uno solo de tus 10 nodos, y que si ese nodo muere, otro lo tome. ¿Cómo lo coordinás sin split-brain?" *(cubrí los dos ángulos: la elección/lock y qué pasa si el dueño "revive").*
8.5 🧠 "Tenés 3 réplicas de tu KV store. ¿Cómo garantizás que una lectura vea la última escritura sin leer siempre del primario?" *(cubrí el lag de réplica y la garantía de quorum).*
8.6 🧠 "Agregás clics por minuto, pero llegan eventos con 5 min de retraso. ¿Cómo no contás mal?"
8.7 🧠 "Un cliente enterprise manda 100× el tráfico del resto y degrada a todos. ¿Cómo aislás?"
8.8 🧠 "Pasás de 4 a 6 shards y casi todas las keys se remapean. ¿Por qué y qué usás en su lugar?"

---

# Soluciones

> Mirá esto solo después de responder en voz alta. En system design no hay una respuesta única: si tu razonamiento cubre las 4 capas y justifica los trade-offs, está bien.

**8.1 — Fan-out de celebridad (#14, #4).** (1) El fan-out on write intenta 50M escrituras de timeline → la cola se satura y los timelines de todos se atrasan. (2) El grafo es ley de potencias: el costo de un post no es uniforme, una cuenta puede tener fan-out 10⁶× la mediana. (3) Detectar la cuenta caliente por umbral de seguidores y **saltear** su fan-out. (4) Modelo **híbrido**: push para usuarios normales, los posts de celebridades se **pullean** al leer y se mezclan con el timeline precomputado. → [system-design-casos.md](system-design-casos.md) caso 1.

**8.2 — Cache stampede / thundering herd (#2, #1).** (1) La key de la home expira y todos los misses pegan a la DB simultáneamente. (2) Con TTL fijo y compartido, miles de requests se sincronizan en el mismo instante de expiración → pico que la DB no aguanta. (3) **TTL con jitter** + **single-flight** (solo un request recomputa, el resto espera su resultado). (4) **Refresh-ahead** (recomputar antes de expirar) o no-expiry con refresh asíncrono. → [redis.md](redis.md) mód 4.

**8.3 — Idempotency gap (#8).** (1) Se pierde el ACK de un `POST /pagos`, el cliente reintenta y cobra dos veces. (2) La red pierde respuestas siempre; bajo reintentos automáticos la ventana ACK-perdido es permanente. (3) Exigir **Idempotency-Key** y deduplicar por ella. (4) **Reserva atómica** de la key **antes** del efecto (INSERT único / `SET NX`) + guardar la respuesta para devolverla en el reintento (store-and-return). → [api-design.md](api-design.md) mód 4.

**8.4 — Leader election / distributed lock (#11, #12).** (1) Sin coordinación, dos nodos creen ser el dueño del job (split-brain) y lo corren dos veces; con un dueño fijo, su caída lo detiene. (2) Particiones y crashes son normales; no hay árbitro central. (3) Un **lock con lease/TTL** en un store fuerte (etcd/ZooKeeper/Redis) + renovación mientras vive. (4) Consenso (**Raft**) para la elección, y **fencing token** que el recurso valida para rechazar a un dueño viejo que "revivió". → *módulo Consenso.*

**8.5 — Quorum / read-after-write (#13, #9).** (1) Leer de una réplica rezagada tras escribir devuelve el valor viejo. (2) Con replicación asíncrona hay *lag* real bajo carga. (3) **Quorum**: con N=3, escribir en W=2 y leer en R=2 garantiza **R+W>N** → la lectura solapa con la última escritura. (4) Read repair + anti-entropy para converger; o consistencia causal / *read-your-writes* (sticky al primario tras escribir). → *módulo Consenso/Replicación.*

**8.6 — Watermarks / late events (#26).** (1) Cerrás la ventana del minuto y un evento llega 5 min tarde → el conteo de ese minuto quedó mal. (2) En streams el *event time* difiere del *processing time*; sin manejo, perdés o subcontás. (3) **Watermark** = "ya no espero eventos anteriores a T" para cerrar la ventana, con **allowed lateness** que mantiene la ventana abierta un rato extra. (4) Windows por event-time + política de *late data* (side-output/recomputar) + checkpointing para exactly-once. → *módulo Streams.*

**8.7 — Noisy neighbor (#25).** (1) El cliente enterprise satura CPU/conexiones compartidas → todos los tenants se degradan. (2) En infraestructura compartida, sin aislamiento un tenant monopoliza el recurso. (3) **Rate-limit y cuotas por tenant** + timeouts por query. (4) Aislamiento: **fair queuing** (pools/colas por tier), cgroups/ResourceQuota, **shuffle sharding** o un shard dedicado para el ruidoso. → *módulo Datos a escala.*

**8.8 — Rebalanceo / consistent hashing (#24, A).** (1) Con `hash mod N`, cambiar N remapea casi todas las keys → tormenta de movimiento de datos y cache misses masivos. (2) A escala, ese remapeo total satura la red y las DBs justo durante el cambio de topología. (3) **Consistent hashing** con *virtual nodes*: al cambiar la topología solo se mueve una fracción de las keys. (4) Rebalanceo gradual con límites de tasa, split de hot shards y aislamiento de tenants ruidosos. → *módulo Consenso (consistent hashing).*

---

## Siguientes pasos

Este checklist es tu **mapa de repaso**: pasá por los conceptos y marcá cuáles podés razonar en 4 capas sin dudar. Los que tengan 🆕 o 🟡 son los que el hub va a profundizar en los módulos del roadmap:

- **Consenso y coordinación distribuida** — Raft/leader election, quorum, consistent hashing, locks+fencing, gossip, logical clocks, anti-entropy.
- **Replicación, multi-región y conflictos** — replication lag/read-after-write, active-active vs active-passive, CRDTs/LWW, 2PC/3PC vs Saga.
- **Datos en movimiento** — watermarks, out-of-order, batch vs stream (Lambda/Kappa), OLTP vs OLAP, Kafka internals.
- **Datos a escala y multi-tenancy** — zero-downtime migration, rebalanceo bajo skew, noisy neighbor, storage engines, Snowflake.
- **Building blocks y diseño geoespacial** — geohash/quadtree, CDN, service discovery, estructuras probabilísticas, leaderboard.

Y la base que ya tenés: el método y el criterio en [Diseño de sistemas](system-design.md), los casos resueltos en [Banco de casos](system-design-casos.md), el contrato en [Diseño de APIs](api-design.md), y las fallas de concurrencia en [Concurrencia](concurrencia.md). La regla de siempre: **no alcanza con nombrar el concepto — hay que razonarlo en 4 capas.**
