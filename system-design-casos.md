# Banco de casos de diseño de sistemas: 17 problemas resueltos de punta a punta

**Los casos clásicos de entrevista senior, resueltos con el método · feed, chat, notificaciones, typeahead, archivos, rate limiter, crawler, reservas/pagos, proximidad, métricas en streaming, edición colaborativa, caché distribuida, streaming de video, ledger de pagos, matching de órdenes, asistente RAG/LLM, scheduler distribuido · el recorrido importa más que la respuesta · 2026**

> Cómo usar este banco: este módulo es la **práctica** de [Diseño de sistemas backend](system-design.md). Ahí aprendiste el método y el criterio; acá los aplicamos a los 17 problemas que más caen en entrevistas. Para cada caso, **tapá la solución y resolvelo vos primero en voz alta** siguiendo los 6 pasos; después contrastá. Lo que se evalúa no es si llegás a *mi* diseño, sino si recorrés bien: requisitos → estimación → API/datos → alto nivel → profundizar donde duele → trade-offs. Son **17 casos**.

**El método, en una línea** (de [system-design](system-design.md) módulo 1): **(1)** aclarar requisitos (sobre todo los **no funcionales**: escala, lectura/escritura, latencia, consistencia), **(2)** estimar (back-of-the-envelope), **(3)** API + modelo de datos, **(4)** diseño de alto nivel, **(5)** profundizar **donde duele**, **(6)** discutir trade-offs. No hay respuestas correctas, hay **decisiones justificadas**.

**Los dos marcos que usamos en cada caso.** Para los modos de falla, las **4 capas**: (1) qué se rompe, (2) por qué a *esta* escala/tráfico, (3) control de corto plazo que frena el *blast radius*, (4) cambio de diseño de largo plazo. Y toda respuesta completa tiene **3 cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación. Los ejercicios de cada caso son **follow-ups** — las preguntas que el entrevistador encadena después del diseño base ("¿y si agregás X?", "¿qué se rompe cuando Y?"). Las soluciones están **todas al final**.

**Índice de casos** (cada uno introduce un **patrón reutilizable** — el ladrillo que después reconocés en problemas nuevos)
1. Feed / timeline de red social (Twitter/Instagram) — *fan-out: write vs read*
2. Chat y mensajería en tiempo real (WhatsApp) — *conexiones persistentes + entrega exactly-once*
3. Sistema de notificaciones push — *async con colas + idempotencia*
4. Typeahead / autocompletar de búsqueda — *precómputo + caché del camino caliente*
5. Almacenamiento y sincronización de archivos (Dropbox) — *separar metadata/contenido + content-addressing*
6. Rate limiter distribuido — *estado compartido atómico*
7. Web crawler / cola de trabajos a escala — *frontera acotada + dedup probabilístico*
8. Sistema de reservas / pagos — *consistencia fuerte + idempotencia transaccional*
9. Uber / ride-sharing: proximidad y matching en tiempo real — *indexación espacial (geohash/quadtree)*
10. Ad-click aggregator / métricas en tiempo real — *agregación en streaming por ventanas de event-time*
11. Google Docs / edición colaborativa en tiempo real — *convergencia de ediciones concurrentes (OT / CRDT)*
12. Caché distribuida (diseñar un Redis/Memcached a escala) — *consistent hashing + defensa contra stampede*
13. YouTube / Netflix: streaming de video — *transcoding paralelo + adaptive bitrate + CDN*
14. Payment ledger / billetera — *contabilidad de doble entrada + log inmutable*
15. Order matching engine (exchange) — *order book en memoria + matching secuencial determinista (event sourcing)*
16. Sistema RAG / asistente LLM — *recuperación + generación fundamentada con citas (el LLM como dependencia cara/lenta/no determinista)*
17. Distributed job scheduler — *disparo programado a escala: índice por tiempo + claim atómico con lease*

> **El patrón que se repite.** Vas a ver que los mismos ladrillos reaparecen en casi todos los casos: **fan-out** (¿escribo a muchos en la escritura o leo de muchos en la lectura?), **caché del camino caliente**, **idempotencia ante reintentos**, **colas para desacoplar y absorber picos**, **sharding por la clave de acceso**, y **el caso patológico** (la celebridad, el prefijo popular, el archivo gigante). Aprender a *reconocer* qué ladrillo aplica es la mitad de la entrevista.

---

## Caso 1 — Feed / timeline de red social (Twitter/Instagram)

**1. Requisitos.**
- Funcionales: un usuario postea; sus seguidores ven ese post en su **home timeline** (los posts de la gente que siguen, ordenados por recencia).
- No funcionales: **lectura-intensivo** (mirás el feed mucho más de lo que posteás); el timeline tolera **consistencia eventual** (que un post tarde segundos en aparecer está bien); latencia de lectura baja (abrir la app tiene que ser instantáneo); grafo social muy desbalanceado (un famoso tiene 100M seguidores, vos 200).

**2. Estimación.** 300M usuarios activos, cada uno abre el feed ~10 veces/día → ~35K lecturas/s promedio, picos de varios cientos de miles. Escrituras (posts): mucho menos, ~5K/s. Ratio lectura:escritura ~100:1 o más. Cada timeline muestra ~20 posts por pantalla. El número que **decide** el diseño no es el promedio de lecturas sino el **fan-out de escritura**: un post normal se copia a ~cientos de timelines (barato), pero un post de una cuenta con 100M seguidores son **100M escrituras** — un solo famoso posteando una vez por segundo ya empuja el fan-out on write puro al orden de **10⁸ escrituras/s**. El número exacto no importa: basta que se acerque para que el push puro sea inviable, y esa sola cuenta ya fuerza el modelo híbrido (módulos 4-5).

**3. API y datos.**
```
POST /posts            { texto, media? }      → { postId }
GET  /feed?cursor=...                          → { posts[], nextCursor }
POST /follow           { targetUserId }
```
- `posts`: `postId (PK) | authorId | texto | createdAt` (sharded por `postId`).
- `follows`: `followerId | followeeId` (con índices en ambas direcciones).
- `timeline` (el feed precomputado): `userId → lista de postIds` (en Redis, una lista/sorted-set por usuario).

**4. La decisión clave — fan-out on write vs on read.** Cuando posteás, ¿quién hace el trabajo de armar timelines?
- **Fan-out on write (push):** al postear, **escribís el postId en el timeline de cada seguidor** (en Redis). Leer el feed es entonces baratísimo (ya está armado: un `LRANGE`). Costo: una escritura se multiplica por la cantidad de seguidores → un post de alguien con 1M seguidores son 1M escrituras.
- **Fan-out on read (pull):** no precomputás nada; al abrir el feed, **buscás los posts recientes de toda la gente que seguís** y los mezclás en el momento. Escribir es barato (un insert); leer es caro (N queries + merge), y se repite en cada apertura.

El default para lectura-intensivo es **push** (pagás una vez al escribir, leés gratis muchas veces). Pero rompe con las celebridades.

**5. Donde duele — la celebridad (hot key + write amplification).** Hacer push de un post de un famoso a 100M timelines es inviable (lentísimo y carísimo, y la mayoría ni va a abrir la app). La solución que usa la industria es **híbrida**: push para usuarios normales; para las cuentas con millones de seguidores, **no** hacés fan-out — sus posts se **pullean en el momento de leer** y se mezclan con el timeline precomputado. Así, abrir el feed = (timeline pre-armado por push) ⊕ (posts recientes de los pocos famosos que seguís, traídos por pull y cacheados).

> **4 capas — el fan-out de la celebridad:**
> 1. **Qué se rompe:** un famoso postea, el sistema intenta 100M escrituras de fan-out; la cola de fan-out se satura, los timelines de todos (no solo sus seguidores) se atrasan minutos.
> 2. **Por qué a esta escala:** el grafo social es ley de potencias — unas pocas cuentas tienen un fan-out 10⁶ veces mayor que la mediana; el costo de una escritura no es uniforme.
> 3. **Control de corto plazo:** detectar la cuenta caliente (umbral de seguidores) y **saltear su fan-out**; encolar el trabajo y procesarlo con prioridad baja en vez de bloquear.
> 4. **Cambio de diseño:** modelo **híbrido push/pull** — pull para cuentas por encima del umbral, push para el resto; el merge en lectura combina ambos. El feed es eventual por diseño.

**6. Trade-offs.** Push da lecturas O(1) pero escrituras caras y desperdicio (armás timelines de gente que no entra a la app); pull invierte el costo. El híbrido es más complejo (dos caminos + merge) pero es el único que escala con un grafo desbalanceado. El orden del feed: estricta recencia es simple; *ranking* (ML) es otro proyecto entero — declaralo fuera de scope salvo que lo pidan.

> **Deep-dive (Staff) — servir el feed en multi-región.** Si el entrevistador sube la apuesta a *"300M usuarios en todo el mundo, latencia de lectura baja en todos lados"*, la respuesta que muestra criterio es: el camino **post → timeline** es el caso **fácil** de multi-región, justo por ser lectura-intensivo y eventual. Los movimientos:
> - **Lecturas locales:** una réplica del feed en cada región; el usuario lee de la suya (~10 ms en vez de ~200 ms cruzando el océano — ver [Replicación](replicacion.md) módulo 1). El timeline ya es eventual por diseño, así que servir una copia con unos segundos de *lag* es aceptable. El orden exacto entre posts de regiones distintas puede variar un poco (*skew* de reloj), pero la recencia del feed ya es aproximada → matiz aceptado a conciencia.
> - **Sin conflictos en el timeline:** un post es **inmutable** (se crea una vez, no se edita concurrentemente) y el timeline es una **proyección derivada**. Eso elimina el problema central del multi-leader —escrituras concurrentes a la misma clave que reconciliar con LWW/CRDT ([Replicación](replicacion.md) módulos 7-9)— **en el camino post→timeline**: replicás los posts a todas las regiones **async** y cada región hace su fan-out local. El único costo de coordinación que queda es el `postId` **único global**, que se resuelve **sin consenso** con IDs tipo Snowflake/ULID (región + timestamp + secuencia embebidos — ver [Datos a escala](datos-escala.md) módulo 13), no con un contador central.
> - **El engagement NO es gratis:** los **contadores de likes/retweets** y los **follows** (el `POST /follow` del caso) **sí son mutables y concurrentes** → ahí reaparece el multi-leader clásico, que se resuelve con **counters CRDT** o **conteo aproximado** ([Replicación](replicacion.md) módulos 7-9). Por eso "el caso fácil" aplica a la **proyección del timeline**, no a todo el producto.
> - **El costo real del timeline es el fan-out cross-región:** alguien en EU sigue cuentas que postean en US. La línea la marca el **volumen de seguidores cross-región por cuenta**: si una cuenta de US tiene muchos seguidores en EU, conviene **replicar** sus posts a EU (push, egress + storage ×N); si son cuatro gatos, **pullear** bajo demanda (latencia). Es el híbrido push/pull de la sección 5 extendido al eje geográfico: replicás lo masivo, pulleás lo raro.
>
> **El contraste que muestra criterio:** ese camino va **active-active** (todas las regiones sirven lecturas y escrituras locales — [Replicación](replicacion.md) módulo 8) sin dolor porque es **AP + append-only**. El **caso 8** (reservas) **no puede**: dos regiones no pueden vender el mismo asiento sin coordinar → o una sola región es dueña del inventario (active-passive) o pagás **consenso** cross-región en el camino crítico (cada commit paga un RTT intercontinental, **~100-150 ms por escritura**). "Multi-región" no es una sola respuesta: depende de si el dato **tolera conflictos**.

> **El cierre de criterio — no sobre-diseñar.** ¿Una red social interna de 5000 empleados? **Fan-out on read y listo**: una query con `WHERE authorId IN (seguidos) ORDER BY createdAt LIMIT 20` sobre Postgres con un índice alcanza y sobra. El híbrido push/pull es la respuesta a 300M usuarios, no a 5000.

**Ejercicios — Caso 1**
1.1 🔁 ¿Qué diferencia hay entre fan-out on write y fan-out on read, y cuál abarata las lecturas?
1.2 🧠 ¿Por qué el fan-out on write puro se rompe con las cuentas de millones de seguidores, y cómo lo resuelve el modelo híbrido?
1.3 🧠 Un usuario nuevo sigue a 500 cuentas de golpe y su timeline está vacío. ¿Cómo le armás el feed inicial sin esperar a que cada una postee? ¿Push o pull para ese arranque?

---

## Caso 2 — Chat y mensajería en tiempo real (WhatsApp)

**1. Requisitos.**
- Funcionales: enviar/recibir mensajes 1-a-1 (y grupos chicos); entrega en tiempo real; estados (enviado/entregado/leído); historial persistente; funciona offline (recibís lo perdido al reconectar).
- No funcionales: latencia de entrega de **sub-segundo**; **ninguna pérdida de mensajes**; orden por conversación; presencia (online/last seen); altísima cantidad de conexiones **persistentes** simultáneas.

**2. Estimación.** 500M usuarios, 50M conexiones WebSocket vivas en pico, ~40 mensajes/usuario/día → ~230K mensajes/s promedio, picos mayores. El cuello no es CPU: es **mantener millones de conexiones abiertas** (memoria/FD por conexión) y rutear entre ellas. El dimensionado se hace por **conexiones concurrentes, no por mensajes/s**: 50M sockets × ~10 KB de estado cada uno (buffers + metadata) ≈ **500 GB** solo en sostenerlos vivos, repartidos en **decenas a cientos de gateways** (~100K-1M conexiones por nodo según RAM y file descriptors). Por eso el diseño gira alrededor de la capa de presencia/routing, no del throughput de mensajes.

**3. API y datos.** Transporte: **WebSocket** (conexión persistente bidireccional, ver [Tiempo real](tiempo-real.md)), no HTTP polling.
```
WS  → { tipo: "msg", to, clientMsgId, texto }
WS  ← { tipo: "msg", from, msgId, texto, ts }
WS  ← { tipo: "ack", clientMsgId, msgId }        # el server confirmó la recepción
WS  ← { tipo: "delivered" | "read", msgId }
```
- `messages`: `msgId | conversationId | senderId | texto | createdAt`, **sharded por `conversationId`** (los mensajes de una charla viven juntos y se leen juntos). `msgId` ordenable (snowflake/ULID) para orden total por conversación.
- `inbox`/cola por usuario para los mensajes pendientes mientras está offline.

**4. La decisión clave — ¿cómo encuentro a quién está conectado a otro server?** Con millones de conexiones repartidas en miles de gateways WebSocket, el server de Ana no sabe en qué gateway está Bruno. Solución: una **capa de presencia/routing** — un registro (`userId → gatewayId`) en Redis, y un **bus de mensajes** (pub/sub o colas) entre gateways. Ana manda → su gateway persiste el mensaje y consulta dónde está Bruno → publica al gateway de Bruno → ese gateway lo empuja por el socket de Bruno. Si Bruno está offline, queda en su **inbox** y se entrega al reconectar.

**5. Donde duele — no perder ni duplicar en reconexiones.** Las conexiones móviles se caen todo el tiempo. El happy path es fácil; lo difícil es la **recuperación**: cada mensaje del cliente lleva un `clientMsgId` (idempotency key, ver [API design](api-design.md) módulo 4); el server **dedup-ea** por esa key, así un reenvío tras un ACK perdido no duplica. Al reconectar, el cliente manda "tengo hasta el `msgId` X" y el server le envía el delta desde su inbox. La entrega es **at-least-once + dedup en el cliente** por `msgId` = efecto exactly-once.

> **4 capas — el mensaje "perdido" en la reconexión:**
> 1. **Qué se rompe:** el cliente envía, el server lo guarda pero el ACK se pierde en la caída de red; el cliente reintenta y el mensaje aparece **dos veces** (o, si no reintenta, se "pierde" a ojos del usuario).
> 2. **Por qué a esta escala/uso:** las redes móviles cortan constantemente; con cientos de miles de mensajes/s, las ventanas ACK-perdido no son raras, son permanentes.
> 3. **Control de corto plazo:** ACK explícito de extremo a extremo; el cliente reintenta los no-ackeados con backoff.
> 4. **Cambio de diseño:** `clientMsgId` para dedup en el server + `msgId` para dedup en el cliente + sync por delta al reconectar (el cliente declara su último `msgId` visto). Entrega at-least-once con idempotencia en ambas puntas.

**6. Trade-offs.** WebSocket vs long-polling: WS escala mejor para muchas conexiones vivas pero exige *sticky sessions* y manejar reconexión (ver [Tiempo real](tiempo-real.md)). Almacenar todo el historial para siempre vs retención/archivado a almacenamiento frío. Cifrado end-to-end (E2EE) cambia el diseño: el server rutea pero **no puede leer** el contenido, así que la búsqueda server-side y algunos features se hacen imposibles — es una decisión de producto, no solo técnica. (Ojo: con E2EE, el routing y los acks/recibos de estado viajan como **metadata aparte** del payload cifrado — el server los necesita para entregar, así que el contenido va cifrado pero "quién le habla a quién y cuándo" sigue siendo visible para el server salvo que lo mitigues explícitamente.)

> **El cierre de criterio — no sobre-diseñar.** ¿Un chat de soporte para tu SaaS con cientos de usuarios concurrentes? Socket.IO sobre un par de instancias con el adaptador de Redis ([Tiempo real](tiempo-real.md)) y Postgres para el historial es la respuesta. La capa de presencia distribuida y el sharding por conversación son para WhatsApp, no para tu widget.

**Ejercicios — Caso 2**
2.1 🔁 ¿Por qué se usa WebSocket en vez de HTTP polling para un chat, y cuál es el principal cuello de botella de recursos?
2.2 🧠 ¿Cómo logra el sistema entrega **exactly-once a ojos del usuario** si la red garantiza solo at-least-once? Nombrá las dos idempotency keys y dónde actúa cada una.
2.3 🧠 ¿Por qué shardear los mensajes por `conversationId` y no por `userId` o por `msgId`? ¿Qué patrón de acceso optimiza?

---

## Caso 3 — Sistema de notificaciones push

**1. Requisitos.**
- Funcionales: un evento del sistema ("te mencionaron", "tu pago se acreditó") genera una notificación que llega al usuario por uno o más **canales** (push móvil/APNs+FCM, email, SMS, in-app); respetar las **preferencias** del usuario (qué quiere recibir y por dónde).
- No funcionales: **alta disponibilidad** (no perder una notificación importante); tolerar picos enormes (un evento masivo dispara millones de golpe); **no duplicar** (recibir el mismo push 3 veces enoja); latencia de segundos es aceptable (es async por naturaleza).

**2. Estimación.** Decenas de millones de notificaciones/día, con picos brutales (un partido, una caída que dispara alertas a todos). El sistema tiene que **absorber el pico** sin tumbar a los proveedores externos (APNs/FCM/el proveedor de SMS tienen sus propios rate limits). El promedio (decenas de millones/día ≈ cientos/s) no dimensiona nada; lo que manda es el **pico**: un evento que avisa a 10M usuarios, drenados en ~10 s, son **~1M/s**, muy por encima de lo que esos proveedores aceptan sostenido → dimensionás para **absorber el frente y drenar al ritmo del proveedor**, no para el promedio (mismo principio que el frente de medianoche del **caso 17**).

**3. API y datos.** Asíncrono de punta a punta (ver [Event-driven](event-driven.md)).
```
POST /notify   { userId, tipo, payload, dedupKey }   → 202 Accepted
```
- Productores publican eventos a una **cola/stream**; un servicio de notificaciones consume, resuelve preferencias y plantilla, y despacha por canal.
- `notification_log`: `dedupKey (unique) | userId | canal | estado | sentAt` — para idempotencia y auditoría.
- `preferences`: `userId | tipo | canales[]`.

**4. La decisión clave — desacoplar con una cola y fan-out por canal.** El productor del evento **no** debe esperar a APNs. Publica a una cola y devuelve `202` (ver [API design](api-design.md) módulo 4). Workers consumen, expanden a los canales habilitados (fan-out por canal) y entregan. La cola **absorbe el pico**: si entran 1M eventos en un segundo, se encolan y se drenan al ritmo que los proveedores aguantan (**backpressure**, ver [Concurrencia](concurrencia.md)). Cada canal es un worker pool con su propio rate hacia el proveedor externo.

**5. Donde duele — no mandar el mismo push N veces.** La entrega de colas es **at-least-once**: un worker procesa un evento, manda el push, pero muere antes de hacer el ACK → otro worker reprocesa y manda **otra vez**. La defensa es una **idempotency key** (`dedupKey`, derivada del evento): antes de despachar, el worker hace un insert atómico de la key en `notification_log` (`INSERT ... ON CONFLICT DO NOTHING`); si ya estaba, **no reenvía**. Mismo patrón que el consumidor idempotente de [Event-driven](event-driven.md) y las idempotency keys de [API design](api-design.md).

> **4 capas — la tormenta de notificaciones duplicadas:**
> 1. **Qué se rompe:** un deploy reinicia los workers a mitad de proceso; los eventos sin ACK se reprocesan y miles de usuarios reciben el mismo push 2-3 veces.
> 2. **Por qué a esta escala/uso:** at-least-once es el contrato de cualquier cola; con millones de mensajes y workers que se reciclan (deploys, autoscaling, crashes), el reproceso es la norma, no la excepción.
> 3. **Control de corto plazo:** dedup por `dedupKey` con insert atómico antes de despachar; ventana de dedup en Redis para los recién enviados.
> 4. **Cambio de diseño:** idempotencia de extremo a extremo (key estable por evento), workers que ackean **después** de registrar el envío, y DLQ para los que fallan repetidamente (un push no entregable es algo que alguien debe ver — ver [Redis](redis.md)/[Event-driven](event-driven.md)).

**6. Trade-offs.** Prioridades: un OTP de login no puede esperar detrás de un millón de newsletters → **colas separadas por prioridad**. Reintentos: APNs/FCM fallan transitoriamente → backoff + DLQ, pero ojo con reintentar una notificación *time-sensitive* que ya no tiene sentido (un "tu Uber llegó" viejo). Agregación: 100 likes no son 100 pushes, son uno ("a 100 personas les gustó") — es feature de producto que ahorra muchísimo.

> **El cierre de criterio — no sobre-diseñar.** ¿Mandás emails transaccionales de tu app? Una cola ([Redis](redis.md)/BullMQ) + un worker + un proveedor (SendGrid/SES) con idempotencia por key alcanza. El fan-out multi-canal con colas por prioridad y agregación es para cuando notificar **es** el producto.

**Ejercicios — Caso 3**
3.1 🔁 ¿Por qué el endpoint que dispara una notificación devuelve `202` y encola, en vez de entregar en línea?
3.2 🧠 La cola garantiza at-least-once. ¿Qué mecanismo evita que el usuario reciba el mismo push dos veces, y en qué momento exacto del worker actúa?
3.3 🧠 Un OTP de login y un newsletter masivo comparten el sistema. ¿Qué problema hay si van por la misma cola y cómo lo resolvés?

---

## Caso 4 — Typeahead / autocompletar de búsqueda

**1. Requisitos.**
- Funcionales: mientras el usuario tipea, sugerir las **top-k** completaciones más probables para ese prefijo, ordenadas por popularidad.
- No funcionales: latencia **brutalmente baja** (cada tecla dispara una consulta — tiene que responder en pocos ms o se siente trabado); lectura-intensivo extremo; las sugerencias pueden ser **levemente desactualizadas** (que un término nuevo tarde minutos/horas en aparecer está bien).

**2. Estimación.** Si la búsqueda recibe 100K búsquedas/s y cada una son ~5 teclas que disparan request → **~500K requests/s** de typeahead. Es de los endpoints más calientes que vas a diseñar. La latencia y el QPS mandan todo.

**3. API y datos.**
```
GET /suggest?q=lap        → { sugerencias: ["laptop", "lapicera", ...] }   # top-k por el prefijo "lap"
```
- Estructura central: un **trie** (árbol de prefijos) donde cada nodo guarda las **top-k completaciones precomputadas** de ese prefijo (con su score de popularidad). Buscar "lap" = navegar 3 nodos y devolver la lista ya lista. O(longitud del prefijo), no O(cantidad de términos).

**4. La decisión clave — precomputar el top-k en cada nodo, separar lectura de escritura.** No calculás el ranking en la request (sería carísimo). El trie de **lectura** sirve respuestas en memoria (o desde caché/CDN), inmutable. Un pipeline **offline** (batch, cada X minutos/horas) recolecta las búsquedas reales, recalcula las frecuencias y **reconstruye el trie** con los nuevos top-k; luego se publica una versión nueva y los servidores la cargan. Lectura rápida e inmutable; escritura desacoplada y eventual.

**5. Donde duele — el QPS sobre el camino caliente.** A 500K req/s, hasta navegar un trie en memoria por instancia se multiplica. Capas: **CDN/edge cache** para los prefijos más populares (un puñado de prefijos cortos concentran la mayoría del tráfico — "a", "fa", "lo"), **caché en memoria** por instancia, y el trie precomputado detrás. Los prefijos de 1-2 letras son los más pedidos y los que más conviene cachear agresivamente.

> **4 capas — el prefijo corto y popular que satura:**
> 1. **Qué se rompe:** millones piden `?q=a` por segundo; si cada uno recalcula o pega a un backend central, ese nodo se funde (hot key).
> 2. **Por qué a esta escala/uso:** la distribución de prefijos es muy sesgada — unos pocos prefijos cortos concentran un porcentaje enorme del tráfico.
> 3. **Control de corto plazo:** cachear top-k por prefijo con TTL en CDN/edge y en memoria; servir los prefijos calientes sin tocar el backend.
> 4. **Cambio de diseño:** top-k **precomputado** por nodo del trie (la respuesta ya existe, no se calcula en la request) + actualización batch desacoplada; la frescura se sacrifica deliberadamente por latencia.

**6. Trade-offs.** Frescura vs latencia: actualizar el trie en tiempo real daría sugerencias frescas pero a costo prohibitivo; el batch acepta minutos de retraso a cambio de lecturas instantáneas. Personalización (sugerencias según *tu* historial) multiplica el costo (ya no hay un trie global) — feature aparte. Manejo de typos/fuzzy matching es otro nivel (distancia de edición) — declaralo fuera de scope salvo que lo pidan.

> **El cierre de criterio — no sobre-diseñar.** ¿Autocompletar sobre un catálogo de 10K productos? `SELECT ... WHERE nombre ILIKE 'lap%' ORDER BY ventas LIMIT 10` sobre Postgres con un índice de prefijo (o `pg_trgm`) responde en milisegundos. El trie precomputado con pipeline batch y CDN es para escala de buscador web.

**Ejercicios — Caso 4**
4.1 🔁 ¿Qué estructura de datos está en el corazón del typeahead y por qué su costo de lectura no depende de la cantidad total de términos?
4.2 🧠 ¿Por qué se precomputa el top-k en cada nodo en vez de calcularlo en la request, y qué se sacrifica con esa decisión?
4.3 🧠 ¿Por qué los prefijos de 1-2 letras son a la vez los más peligrosos (hot keys) y los más fáciles de mitigar?

---

## Caso 5 — Almacenamiento y sincronización de archivos (Dropbox)

**1. Requisitos.**
- Funcionales: subir/bajar archivos; sincronizarlos entre los dispositivos de un usuario; versionado; compartir.
- No funcionales: **durabilidad altísima** (no perder archivos jamás); manejar archivos grandes; subidas resumibles (las redes cortan); uso eficiente de ancho de banda y almacenamiento (no resubir lo que no cambió).

**2. Estimación.** Cientos de millones de usuarios, archivos de KB a GB. El volumen total son **petabytes**; el costo dominante es **almacenamiento y ancho de banda**, no CPU. Muchos archivos son idénticos entre usuarios (el mismo PDF, el mismo instalador) → la **deduplicación** ahorra una fortuna. Concreto: cientos de millones de usuarios × varios GB = un bruto de **petabytes**, y muchos chunks se repiten entre cuentas (el mismo instalador, el mismo adjunto). Ese número fuerza una decisión: la dedup **global** (cross-account) recorta típicamente **30-50%** del almacenamiento —decenas a cientos de millones de dólares al año a esta escala— pero exige content-addressing (el hash del chunk como clave) y un índice de chunks compartido, con costo en privacidad (un atacante infiere si un chunk ya existe midiendo el tiempo de subida) y en borrado (reference counting); la dedup **por cuenta** es más simple y segura pero ahorra mucho menos. La estimación acá no busca un QPS: es el eje que decide **global vs aislada** y por qué el diseño optimiza **bytes únicos**, no cómputo.

**3. API y datos.** Separá **metadata** de **contenido**.
```
POST /files/{id}/chunks/{idx}     (bytes del chunk)      → 200
POST /files                       { nombre, chunkHashes[] } → { fileId }
GET  /files/{id}/manifest                                 → { chunks[], version }
```
- **Blobs en object storage** (S3/GCS): durable, barato, escala infinito. Los archivos se parten en **chunks** (ej. 4MB).
- **Metadata en una base** (Postgres): `files (fileId | userId | nombre | version)`, `chunks (chunkHash | fileId | idx)`. La metadata es chica y transaccional; el contenido es enorme y va al blob store.

**4. La decisión clave — chunking + deduplicación por contenido.** Partís cada archivo en chunks y a cada uno lo identificás por el **hash de su contenido** (content-addressed): `chunkId = sha256(bytes)`.
```ts
import { createHash } from "node:crypto";
// El id de un chunk ES el hash de su contenido (compila en --strict).
function chunkId(bytes: Buffer): string {
  return createHash("sha256").update(bytes).digest("hex");
}
```
Esto da tres cosas gratis: **dedup** (si dos archivos comparten un chunk, el hash coincide y lo guardás una sola vez), **sync por delta** (al editar un archivo, solo subís los chunks cuyo hash cambió — no el archivo entero) y **integridad** (el hash verifica que el chunk no se corrompió). Subir un archivo = el cliente calcula los hashes, le pregunta al server cuáles **no** existen aún, y sube **solo esos**.

**5. Donde duele — consistencia entre metadata y blob.** Un archivo "existe" cuando su metadata lo dice; los bytes viven en otro sistema (S3). Si registrás la metadata **antes** de que todos los chunks estén subidos, un lector puede pedir un archivo cuyos bytes no están → error. **El orden importa:** subí los chunks primero (a S3), verificá que están, y **recién entonces** commiteás la metadata que apunta a ellos. Los chunks huérfanos (subidos pero nunca referenciados, ej. una subida abandonada) se limpian con un **garbage collector** offline.

> **4 capas — la metadata que apunta a un blob que no está:**
> 1. **Qué se rompe:** se registra la versión nueva del archivo, pero un chunk no terminó de subir (o la subida se abortó); al bajarlo, falta un pedazo → archivo corrupto.
> 2. **Por qué a esta escala/uso:** metadata y contenido son **dos sistemas distintos** sin transacción común (Postgres + S3); cualquier fallo entre los dos pasos deja un estado inconsistente (el *dual-write problem* de [Event-driven](event-driven.md)).
> 3. **Control de corto plazo:** escribir en orden (blobs → verificar → metadata); nunca exponer una versión cuya metadata no tenga todos los chunks confirmados.
> 4. **Cambio de diseño:** metadata **inmutable por versión** (una versión se publica solo cuando está completa), content-addressing para verificar integridad, y GC de chunks huérfanos. El archivo "salta" de una versión completa a la siguiente, nunca queda a medias.

**6. Trade-offs.** Tamaño de chunk: chunks chicos = mejor dedup y deltas más finos, pero más metadata y más requests; chunks grandes = lo inverso. Dedup global (entre todos los usuarios) ahorra más pero complica el borrado (no podés borrar un chunk que otro usuario referencia → *reference counting*) y tiene implicaciones de privacidad. Sync en tiempo real (watchers + push de cambios) vs polling periódico.

> **El cierre de criterio — no sobre-diseñar.** ¿Tu app sube avatares y algún PDF? Subí directo a S3 con una **presigned URL** y guardá la URL en una columna. Chunking, content-addressing y GC son para cuando **almacenar y sincronizar archivos ES el producto**, no un feature lateral.

**Ejercicios — Caso 5**
5.1 🔁 ¿Por qué se separan la metadata (en una base) y el contenido (en object storage)?
5.2 🧠 ¿Qué tres beneficios da identificar cada chunk por el hash de su contenido (content-addressing)?
5.3 🧠 ¿En qué orden escribís metadata y blobs al subir un archivo, y por qué el orden inverso corrompería lecturas?

---

## Caso 6 — Rate limiter distribuido

**1. Requisitos.**
- Funcionales: limitar a cada cliente a N requests por ventana de tiempo; al pasarse, rechazar con `429` (ver [API design](api-design.md) módulo 10).
- No funcionales: el límite debe valer **a través de todas las instancias** del API (no N por instancia); latencia mínima añadida (se ejecuta en cada request); preciso bajo concurrencia; tolerante a fallos (si el limiter se cae, decidir si *fail-open* o *fail-closed*).

**2. Estimación.** Si el API hace 500K req/s, el rate limiter se ejecuta **500K veces/s** y agrega latencia a cada una → tiene que ser sub-milisegundo. El estado del contador es chico (por cliente), pero compartido entre todas las instancias.

**3. API y datos.** No es un endpoint: es un **middleware** delante de todos. El estado compartido va a **Redis** (single-thread, atómico, rapidísimo — ver [Redis](redis.md)). Clave: `rl:{clienteId}` → contador/tokens.

**4. La decisión clave — el algoritmo y su atomicidad.** El [system-design módulo 12](system-design.md) mostró el **token bucket** en memoria; acá el reto es hacerlo **distribuido y atómico**.
- **Fixed window:** contás requests por ventana fija (`rl:user:minuto-actual`, `INCR`, expira en 60s). Simple pero tiene el **boundary burst**: alguien manda N al final de un minuto y N al principio del siguiente → 2N en pocos segundos.
- **Sliding window** (log o counter): suaviza el borde, más justo, algo más caro.
- **Token bucket:** permite ráfagas controladas; el estado (tokens + último refill) se guarda y actualiza atómicamente.

La trampa clásica: hacer `INCR` y `EXPIRE` en **dos comandos** no es atómico — si el proceso muere entre ambos, la clave queda **sin TTL** y el cliente queda bloqueado para siempre. La solución es un **script Lua** (Redis lo ejecuta atómicamente, ver [Redis](redis.md)):
```lua
-- Fixed window atómico: incrementa y, si es la primera vez, fija el TTL. Devuelve el conteo.
local c = redis.call('INCR', KEYS[1])
if c == 1 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end
return c
```
*(En producción se suele blindar un paso más: el `EXPIRE` solo se fija cuando `c == 1`, así que si la clave se creó por otra vía sin TTL, nunca vencería. Defensa: re-checar `if redis.call('TTL', KEYS[1]) < 0 then redis.call('EXPIRE', KEYS[1], ARGV[1]) end`, o usar `SET key 1 EX ttl NX` para la primera escritura. La regla: toda clave de rate limiting debe tener TTL garantizado.)*

**5. Donde duele — atomicidad y qué pasa si Redis se cae.** Sin atomicidad, dos requests concurrentes del mismo cliente pueden leer el mismo valor y ambas pasar (check-then-act, ver [Concurrencia](concurrencia.md) módulo 5): el límite se filtra. Lua lo resuelve. Y la pregunta senior: **si Redis no responde, ¿qué hacés?** *Fail-open* (dejás pasar todo) protege la disponibilidad de tu API pero deja caer el límite; *fail-closed* (rechazás todo) protege el recurso pero te tumba el API por un problema del limiter. La respuesta depende de qué protege el límite (¿abuso/costo? probablemente fail-open con un límite local de respaldo; ¿un recurso frágil? fail-closed).

> **4 capas — el `INCR` + `EXPIRE` no atómico:**
> 1. **Qué se rompe:** se hace `INCR` y el proceso muere antes del `EXPIRE`; la clave del cliente queda **sin vencimiento** y nunca se resetea → ese cliente queda limitado para siempre (o, según la lógica, sin límite nunca).
> 2. **Por qué a esta escala/uso:** a cientos de miles de ops/s, la ventana entre dos comandos separados **se toca seguro**; "raro" a baja escala es "constante" a alta.
> 3. **Control de corto plazo:** colapsar las dos operaciones en una atómica (script Lua o `SET key val EX ttl NX`); detectar claves sin TTL.
> 4. **Cambio de diseño:** toda la lógica del limiter (leer, decidir, actualizar, fijar TTL) en **un script Lua** que Redis corre atómicamente; reloj y refill calculados dentro del script. Atomicidad por diseño, no por suerte.

**6. Trade-offs.** Precisión vs costo: fixed window es barato pero injusto en el borde; sliding window log es exacto pero guarda un timestamp por request (memoria); el counter aproximado es el punto medio que usa casi todo el mundo. Centralizado en Redis (un round-trip por request, simple y consistente) vs límite local por instancia (cero latencia pero N× más permisivo). Granularidad: por usuario, por IP, por endpoint, o combinaciones.

> **El cierre de criterio — no sobre-diseñar.** ¿Una sola instancia de API? El token bucket **en memoria** del [system-design módulo 12](system-design.md) alcanza — no necesitás Redis ni Lua. El limiter distribuido es la respuesta cuando tenés **varias instancias** que deben compartir el mismo contador.

**Ejercicios — Caso 6**
6.1 🔁 ¿Por qué el estado del rate limiter va en Redis y no en la memoria de cada instancia del API?
6.2 🧠 ¿Por qué `INCR` seguido de `EXPIRE` en dos comandos es un bug, qué falla concreta produce, y cómo lo arregla un script Lua?
6.3 🧠 Si Redis deja de responder, ¿qué significan *fail-open* y *fail-closed*, y cómo elegís según lo que protege el límite?
6.4 ✍️ Escribí un script Lua de **fixed window** que reciba `KEYS[1]` (la clave), `ARGV[1]` (el TTL en segundos) y `ARGV[2]` (el límite), incremente atómicamente, fije el TTL en la primera escritura y devuelva `1` si la request se permite o `0` si se pasó del límite.

---

## Caso 7 — Web crawler / cola de trabajos a escala

**1. Requisitos.**
- Funcionales: dado un conjunto de URLs semilla, descargar páginas, extraer links y seguirlos, hasta cubrir la web (o un dominio); guardar el contenido para indexar.
- No funcionales: **escala masiva** (miles de millones de páginas); **no recrawlear** lo mismo ni entrar en loops; **politeness** (no martillar un dominio — respetar `robots.txt` y un rate por host); resumible y tolerante a fallos (un crawl dura días); priorizable (páginas importantes primero).

**2. Estimación.** 10B páginas en un mes → ~4000 páginas/s sostenido. Cada página ~100KB → **≈1 PB** de contenido (10×10⁹ × 100KB = 10⁶ GB). El cuello: la **cantidad de URLs en vuelo** (la frontera) y **no procesar duplicados** entre miles de millones.

**3. API y datos.** Es un **pipeline de cola de trabajos** (el patrón general detrás de cualquier procesamiento masivo async — ver [Redis](redis.md)/[Event-driven](event-driven.md)).
```
URL frontier (cola priorizada por host) → workers (fetch + parse) → extract links → dedup → re-encolar
```
- **Frontier:** la cola de URLs por visitar, particionada **por host** (para aplicar politeness por dominio) y priorizada (PageRank/frescura).
- **"Visto":** un conjunto de URLs ya encoladas/visitadas. A miles de millones, un `SET` exacto no entra en memoria → **Bloom filter** (estructura probabilística: dice "seguro que no lo vi" o "probablemente sí", con falsos positivos pero nunca falsos negativos, a una fracción de la memoria).
- Contenido crudo a **object storage**; metadata e índice aparte.

**4. La decisión clave — dedup de URLs a escala + politeness.** Dos problemas que definen el diseño:
- **Dedup:** antes de encolar una URL, ¿ya la vi? Un Bloom filter responde en O(1) con poca memoria; el costo es que un falso positivo descarta una URL nueva ocasionalmente (aceptable: perdés una página, no rompés nada).
- **Politeness:** no podés mandar 4000 req/s a un solo dominio. La frontera se particiona **por host** y cada host tiene su propio rate (cola por dominio + un planificador que respeta el `crawl-delay`). Esto desacopla "tengo mucho trabajo" de "puedo golpear este sitio ahora".

**5. Donde duele — la cola que explota y las trampas.** Cada página extrae ~100 links → la frontera **crece más rápido de lo que se drena** si no hay control. Y hay **spider traps** (calendarios infinitos, URLs generadas dinámicamente que nunca terminan). Defensas: **backpressure** (acotar la frontera; si se llena, dejar de extraer o bajar prioridad — ver [Concurrencia](concurrencia.md)), límites de profundidad y de páginas por dominio, normalización de URLs (para que `?utm=...` no genere infinitas variantes del mismo recurso), y detección de patrones de trampa.

> **4 capas — la frontera que crece sin límite:**
> 1. **Qué se rompe:** cada página agrega más URLs de las que se procesan; la cola crece sin techo hasta agotar el almacenamiento y el crawler se detiene (o un spider trap la inunda con basura).
> 2. **Por qué a esta escala/uso:** el factor de ramificación de la web es enorme (decenas-cientos de links por página); sin control, el crecimiento es exponencial y los sitios maliciosos/mal hechos lo explotan.
> 3. **Control de corto plazo:** acotar el tamaño de la frontera (backpressure: frenar la extracción cuando se llena), límites de profundidad y de páginas por host, deduplicar agresivo.
> 4. **Cambio de diseño:** frontera priorizada y acotada + Bloom filter para dedup + normalización de URLs + detección de trampas; el crawler procesa lo **valioso** primero y descarta el ruido, en vez de intentar "toda la web" ciegamente.

**6. Trade-offs.** Bloom filter: ahorra memoria brutal pero introduce falsos positivos (descartás alguna URL nueva) — aceptable acá. Exactitud del "visto" vs memoria. Frescura (recrawlear seguido para detectar cambios) vs cobertura (descubrir páginas nuevas) — compiten por el mismo ancho de banda. Este caso es el **patrón general de cola de trabajos a escala**: frontier = cola, workers = consumidores, dedup = idempotencia, politeness = rate limiting, backpressure = control de flujo. Cambiá "URL" por "video a transcodificar" o "imagen a procesar" y es el mismo diseño.

> **El cierre de criterio — no sobre-diseñar.** ¿Procesás miles de trabajos en background (thumbnails, emails, reportes)? [Redis](redis.md)/BullMQ con workers, reintentos y DLQ es la respuesta — la cola de trabajos "de manual". El Bloom filter, la partición por host y la detección de spider traps son específicos de crawlear la web abierta a escala de buscador.

**Ejercicios — Caso 7**
7.1 🔁 ¿Para qué sirve un Bloom filter en un crawler y qué tipo de error puede cometer (y cuál nunca)?
7.2 🧠 ¿Por qué la frontera se particiona por host, y qué problema resuelve eso respecto de la *politeness*?
7.3 🧠 ¿Por qué la frontera puede crecer sin control, y qué dos mecanismos la mantienen acotada?

---

## Caso 8 — Sistema de reservas / pagos (consistencia fuerte e idempotencia transaccional)

**1. Requisitos.**
- Funcionales: reservar un recurso **único y escaso** (un asiento, una habitación, una unidad de stock) y pagarlo; el recurso queda no disponible para otros.
- No funcionales: **consistencia fuerte** — a diferencia del feed (caso 1), acá un dato desactualizado **vende de más**; jamás dos personas se quedan con el mismo asiento (**no oversell**) ni se cobra dos veces (**no doble cargo**); contención altísima sobre **pocos recursos calientes** en el pico de venta. Correctitud sobre disponibilidad.

**2. Estimación.** Una ticketera: una venta popular dispara cientos de miles de personas peleando por **los mismos pocos miles de asientos** en segundos. El problema **no es throughput** (no son tantas filas) sino **contención y corrección**: muchísimos intentos concurrentes sobre el mismo recurso. Es el caso opuesto al feed — ahí el reto era la escala de lectura; acá, la integridad bajo escritura concurrente.

**3. API y datos.** Tres pasos, no uno: reservar (hold temporal) → pagar → confirmar.
```
POST /holds         { seatId }                    → { holdId, expiraEn }   # reserva temporal
POST /bookings      { holdId, Idempotency-Key }   → { bookingId }          # confirma + cobra
```
- `seats`: `seatId (PK) | estado ('available'|'held'|'booked') | heldBy | heldUntil | version`. *(La columna `version` está para la **variante de lock optimista** que comparamos en el paso 4; si elegís el UPDATE condicional por `estado`, no la usás — la dejamos para que veas las dos opciones.)*
- `bookings`: `bookingId | seatId | userId | idempotencyKey (unique) | estado`.
La consistencia vive en una base **transaccional** (Postgres), no en un store eventual.

**4. La decisión clave — reservar sin vender dos veces (atomicidad bajo contención).** El corazón es el *check-then-act*: "¿está libre? entonces tomalo". Hacerlo en dos pasos es la receta del oversell (ver [Concurrencia](concurrencia.md) módulo 5). Las opciones:
- **Lock pesimista** (`SELECT ... FOR UPDATE`): bloqueás la fila del asiento, verificás y actualizás. Simple y correcto, pero bajo contención extrema los locks se encolan y pueden dar contención/deadlocks (ver [PostgreSQL](postgresql.md)).
- **Lock optimista** (columna `version`): leés con su versión, actualizás `WHERE version = leída`; si otro ganó, 0 filas → reintentás. Mejor en baja contención; en alta, muchos reintentos.
- **UPDATE condicional atómico** (lo más simple y efectivo acá): un solo statement que solo tiene éxito si seguía libre.
```sql
-- Reserva atómica: solo afecta la fila si el asiento seguía 'available'. 0 filas = ya lo tomaron.
UPDATE seats SET estado = 'held', held_by = $1, held_until = now() + interval '5 minutes'
WHERE seat_id = $2 AND estado = 'available';
```
Chequeás las filas afectadas: 1 = tuyo, 0 = alguien te ganó (devolvés `409 Conflict`, ver [API design](api-design.md)). El propio motor serializa el `UPDATE`, así que no hay ventana de carrera.

*(Reabsorber holds vencidos: el `WHERE` de arriba solo toma asientos `available`, pero un hold abandonado queda en `held` con `held_until` pasado. Para reciclar ese inventario, ampliá la condición a `WHERE seat_id = $2 AND (estado = 'available' OR (estado = 'held' AND held_until < now()))`, o corré un *sweeper* periódico que devuelva a `available` los holds expirados.)*

> **No confundas dos carreras distintas:** la **atomicidad del instante** (lock / `version` / UPDATE condicional) evita que dos requests *simultáneas* tomen el mismo asiento; el **TTL del hold** resuelve la carrera *diferida en el tiempo* (alguien reserva y nunca paga, congelando inventario). Necesitás las dos: una para el choque inmediato, otra para el abandono.

**5. Donde duele — el pago externo lento y la doble cobranza.** Pagar tarda (Stripe responde en cientos de ms o segundos) y **falla**. Error de junior: tener la transacción de DB abierta **mientras esperás al proveedor** → mantenés locks tomados segundos, matás la concurrencia y arriesgás timeouts. El patrón correcto separa las fases: **(1) hold corto** (reserva atómica con TTL, transacción rapidísima), **(2) cobrar fuera de la transacción** con una **idempotency key** (no cobrar dos veces si el cliente reintenta — el mismo patrón de [API design](api-design.md) módulo 4), **(3) confirmar** (pasar el asiento de `held` a `booked`). Si el pago falla o el hold expira, el asiento **se libera** y vuelve a `available`. Coordinar las tres fases entre servicios (inventario + pagos) es una **saga** con compensación (ver [Event-driven](event-driven.md)): si el paso 2 falla, el paso compensatorio libera el hold del paso 1.

Tres matices que un entrevistador senior espera oír:
- **La idempotency key es store-and-return, no solo un `unique`.** Ante un reintento con la misma key, devolvés el `bookingId` **ya creado** (la respuesta guardada), no un error: el índice `unique` es la red de seguridad contra duplicados, no el mecanismo de respuesta (ver [API design](api-design.md) módulo 4).
- **El pago puede quedar en estado *desconocido*.** Si el proveedor no responde (timeout), no sabés si cobró: **nunca asumas que falló**. Reintentás con la misma idempotency key (el proveedor deduplica) y **consultás el estado** (polling o webhook de confirmación) antes de decidir; recién ahí confirmás o compensás.
- **Si la compensación también falla**, el **TTL del hold es el respaldo final**: aunque el paso que libera el asiento falle, el hold expira solo y el inventario se recicla. Diseñá para que el peor caso sea "el asiento queda bloqueado unos minutos", no "se pierde para siempre".

> **4 capas — el oversell del último asiento:**
> 1. **Qué se rompe:** dos usuarios compran el último asiento casi a la vez; ambos pagan; uno se queda sin lugar (oversell) — o, peor, a alguien se le cobra dos veces por un reintento.
> 2. **Por qué a esta escala/uso:** en el pico de venta hay contención brutal sobre el mismo recurso; un check-then-act sin atomicidad deja pasar a los dos (ambos leen 'available' antes de que ninguno escriba).
> 3. **Control de corto plazo:** reserva **atómica** (UPDATE condicional / `FOR UPDATE`) que solo un intento puede ganar; idempotency key en el cobro para que un reintento no duplique el cargo.
> 4. **Cambio de diseño:** hold con TTL + saga reserva→pago→confirmar con compensación (liberar el hold si el pago falla o expira), inventario en una base con consistencia fuerte, y para picos masivos una **sala de espera virtual** (cola que admite usuarios al inventario de a tandas, en vez de dejar entrar a todos a pelear a la vez).

**6. Trade-offs.** Consistencia fuerte = elegís **CP** del teorema CAP (ver [system-design](system-design.md) módulo 6): ante una partición de red, preferís **rechazar o bloquear** antes que arriesgar un oversell — sacrificás disponibilidad **conscientemente**, al revés del feed (AP). No lo recites como dogma: el punto es que en inventario escaso el costo de un dato inconsistente (vender de más) es peor que el de rechazar una compra. Pesimista (simple, pero locks bajo contención) vs optimista (sin locks, pero reintentos en alta contención) vs UPDATE condicional (el punto dulce para "tomar un recurso único"). El hold con TTL evita que un carrito abandonado congele inventario para siempre, a costa de complejidad (expiración + liberación). La sala de espera agrega latencia de cola pero salva al sistema en los picos.

> **El cierre de criterio — no sobre-diseñar.** ¿Vendés productos con **stock amplio** y baja contención (mil unidades, pocas compras concurrentes)? Un `UPDATE productos SET stock = stock - 1 WHERE id = $1 AND stock > 0` atómico, chequeando filas afectadas, alcanza y sobra — sin holds, sin saga, sin sala de espera. Toda esa maquinaria es para **inventario escaso con contención alta** (entradas de un recital, asientos de un vuelo), no para un e-commerce común.

**Ejercicios — Caso 8**
8.1 🔁 ¿Por qué el inventario de reservas necesita consistencia **fuerte** y no eventual, a diferencia del timeline del caso 1?
8.2 🧠 ¿Por qué **no** tener la transacción de base de datos abierta mientras esperás la respuesta del proveedor de pago, y qué patrón de tres fases usás en su lugar?
8.3 🧠 Para "tomar el último asiento" bajo alta contención, compará lock pesimista, lock optimista y UPDATE condicional: ¿cuál elegís y por qué?
8.4 ✍️ Escribí `reservarAsiento(seatId, userId, repo)` que reserve **atómicamente** solo si el asiento está libre, devolviendo `true` si lo tomó y `false` si ya estaba tomado. Usá los tipos de abajo.
```ts
interface SeatRepo {
  // ejecuta el UPDATE condicional y devuelve cuántas filas afectó (0 o 1)
  marcarHeldSiLibre(seatId: string, userId: string): Promise<number>;
}
```
8.5 🧠 ¿Cuándo justificás una **sala de espera virtual** (admitir usuarios al inventario de a tandas) y qué problema resuelve que el lock/UPDATE condicional **no** resuelve?

---

## Caso 9 — Uber / ride-sharing: proximidad y matching en tiempo real

**1. Requisitos.**
- Funcionales: un pasajero pide un viaje desde su ubicación; el sistema encuentra **conductores cercanos disponibles** y hace el **matching** (asigna uno); durante el viaje, pasajero y conductor ven la ubicación del otro en tiempo real.
- No funcionales: los conductores **reportan su ubicación cada pocos segundos** → escritura-intensivo de ubicaciones; la búsqueda "conductores cerca de mí" tiene que ser de **baja latencia** sobre datos que cambian sin parar; un conductor **no** puede asignarse a dos pasajeros a la vez (consistencia en la asignación); casi todos los viajes son **locales** (el sistema escala por ciudad/región, no global por viaje).

**2. Estimación.** 5M conductores activos, cada uno manda su posición cada ~4 s → 5×10⁶ ÷ 4 ≈ **~1.25M escrituras de ubicación/s**. Los pedidos de viaje son órdenes de magnitud menos (cientos de miles/hora en pico → ~cientos/s). La conclusión que *conduce* el diseño: el cuello **no** es el matching (poco volumen) sino (a) absorber el **fire-hose de ubicaciones** y (b) responder "¿quién está cerca?" rápido sobre puntos que se mueven todo el tiempo.

**3. API y datos.**
```
POST /drivers/location   { driverId, lat, lng }          → 200        # altísima frecuencia
POST /rides              { riderId, origen: {lat, lng} }  → { rideId, driver }
WS   ← { tipo: "ubicacion", lat, lng, ts }                            # tracking en vivo del viaje
```
- Ubicaciones en un store **en memoria** (Redis) indexado por **celda geoespacial**, no en una tabla con columnas `lat`/`lng`: un `WHERE lat BETWEEN ... AND lng BETWEEN ...` no usa un índice eficiente para 2-D y termina escaneando una franja enorme.
- `rides`: `rideId | riderId | driverId | estado` — en una base **transaccional**, porque la asignación necesita consistencia (como el inventario del caso 8).

**4. La decisión clave — indexar el espacio (geohash / quadtree).** El problema es "dame los conductores en ~2 km a la redonda" sobre millones de puntos móviles. No hay un índice 1-D que sirva para buscar en 2-D. La técnica: **mapear 2-D a 1-D con un geohash** — dividís el mundo en una grilla de celdas y a cada una le das un id de string donde el **prefijo** es la celda más grande que la contiene. Puntos cercanos comparten prefijo → "cerca de mí" se vuelve una búsqueda **por prefijo/rango en una dimensión**, que sí es indexable.
- **Geohash:** simple, 1-D, encaja directo en Redis; celdas de tamaño fijo según el nivel de precisión.
- **Quadtree:** un árbol que subdivide **solo las zonas densas** (el centro lleno se parte más fino que las afueras vacías) → se adapta a la densidad, pero es una estructura en memoria que hay que mantener.

Redis trae comandos geoespaciales nativos (`GEOADD` para insertar, `GEOSEARCH` para "dentro de N km") construidos sobre geohash — en producción arrancás con eso antes de implementar un quadtree a mano.

**5. Donde duele — el fire-hose de ubicaciones y el borde de celda.** 1.25M escrituras/s de posiciones es brutal, y la mayoría son **efímeras** (solo importa la última de cada conductor). Persistir cada ping en disco es desperdicio y funde el índice. Además, un conductor a 50 m tuyo pero **del otro lado del borde** de una celda cae en otro geohash: si solo mirás tu celda, lo perdés. Por eso la búsqueda mira **tu celda + las 8 vecinas** (grilla 3×3) y después refina por distancia real. Y el **matching** tiene su propia carrera: dos pasajeros que piden a la vez no pueden quedarse con el mismo conductor → reusás el **claim atómico del caso 8** (un `UPDATE drivers SET estado='asignado' WHERE driverId=$1 AND estado='disponible'`; 1 fila = es tuyo, 0 = ya lo tomaron, seguís con el próximo candidato).

> **4 capas — el fire-hose de actualizaciones de ubicación:**
> 1. **Qué se rompe:** millones de updates/s saturan la base si cada uno es un write durable; la búsqueda "cerca de mí" se hace lenta porque el índice se reescribe sin parar.
> 2. **Por qué a esta escala/uso:** los conductores reportan cada pocos segundos; el volumen de writes de ubicación supera por órdenes de magnitud al de viajes, y persistir cada ping es inútil (solo vale la posición actual).
> 3. **Control de corto plazo:** ubicaciones **en memoria** con TTL (no en disco); **sobrescribir** la última posición en vez de hacer append; servir la búsqueda desde el índice en memoria.
> 4. **Cambio de diseño:** índice geoespacial **particionado por región/ciudad** (un viaje casi nunca cruza ciudades, así que cada región es independiente y escala sola), ubicación como **estado efímero** (la última gana), y un **stream aparte** si además necesitás el histórico de la ruta para facturación/analytics.

**6. Trade-offs.** Geohash (fijo, simple, nativo en Redis) vs quadtree (se adapta a la densidad, pero más complejo de mantener). Precisión de celda: celda grande = menos celdas pero más candidatos a filtrar por distancia; celda chica = candidatos más precisos pero más celdas vecinas que consultar (elegís la precisión para que el radio de la celda ≥ tu radio de búsqueda). Matching por **distancia en línea recta** (simple) vs por **ETA real** considerando tráfico/calles (mejor experiencia, pero necesita un servicio de routing). El tracking en vivo del viaje reusa el **WebSocket del caso 2** (conexión persistente, el server empuja la posición). Surge pricing, batching de matches y re-matching si el conductor cancela: features aparte — declaralos fuera de scope salvo que los pidan.

> **El cierre de criterio — no sobre-diseñar.** ¿Una app de delivery para una ciudad chica con cientos de repartidores? `GEOADD`/`GEOSEARCH` de Redis (o **PostGIS** con un índice GiST) sobre una sola instancia alcanza y sobra — sin quadtree, sin partición por región, sin fire-hose en memoria. Toda esa maquinaria es para la escala de Uber global, donde el volumen de ubicaciones y la densidad de las ciudades grandes lo justifican.

**Ejercicios — Caso 9**
9.1 🔁 ¿Por qué una query `WHERE lat BETWEEN ... AND lng BETWEEN ...` no escala para "conductores cerca de mí", y qué resuelve mapear el espacio con un geohash?
9.2 🧠 Al buscar conductores cercanos, ¿por qué no alcanza con mirar tu propia celda de geohash, y qué hacés al respecto?
9.3 🧠 El matching podría asignar el mismo conductor a dos pasajeros que piden casi a la vez. ¿Qué patrón de un caso anterior reusás para evitarlo, y cómo se ve?
9.4 ✍️ Tenés los conductores candidatos de tu celda + las vecinas (cada uno con su posición). Escribí `masCercanos(origen, candidatos, k, distanciaKm)` que devuelva los `k` más cercanos por distancia real. Usá los tipos de abajo.
```ts
interface Punto { lat: number; lng: number }
interface Conductor { driverId: string; pos: Punto }
// distanciaKm(a, b): distancia entre dos puntos (haversine); asumila ya implementada.
```
9.5 🧠 El matching por distancia en línea recta puede elegir un conductor que está cruzando un río, en sentido contrario o a 2 minutos en auto pese a estar a 200 m. ¿Qué cambia si rankeás por **ETA real** en vez de distancia recta, y qué dependencia nueva introduce?

---

## Caso 10 — Ad-click aggregator / métricas en tiempo real

**1. Requisitos.**
- Funcionales: cada vez que un usuario ve o clickea un anuncio se genera un evento; el sistema **agrega** esos eventos por `(anuncio, ventana de tiempo)` y los expone de dos formas: un **dashboard** near-real-time para el anunciante ("clicks de tu campaña en el último minuto/hora") y la **facturación** (cobrarle al anunciante por click, *pay-per-click*).
- No funcionales: **escritura-intensivo extremo** (el fire-hose de eventos); el dashboard tolera **segundos** de latencia y una respuesta aproximada; la facturación, en cambio, tiene que ser **correcta y reproducible** (cobrás plata: ni de más ni de menos); tolerar eventos **tardíos y desordenados** (un click ocurrido hace minutos llega ahora); poder **recalcular** si se descubre un bug en la agregación.

**2. Estimación.** Una red de ads grande: ~10×10⁹ clicks/día → ~115K clicks/s promedio, con picos de varios ×. Las **impresiones** (cada vez que se muestra un ad) son ~100× más → del orden de **millones/s**. El volumen manda todo. Guardar cada evento crudo (~200 bytes) son ~2 TB/día solo de clicks; pero los **agregados** (por anuncio × minuto) son chicos. El cuello: **ingerir el fire-hose** y **agregar sin perder ni duplicar**, no la lectura del dashboard (que sobre agregados es trivial).

**3. API y datos.** Ingestión asíncrona de alto volumen; lectura sobre datos ya agregados.
```
POST /events   { adId, tipo: "click"|"impression", eventId, userId, ts }   → 202 Accepted
GET  /stats?adId=...&desde=...&hasta=...&granularidad=minuto                → { ventanas: [{ inicio, clicks, impresiones }] }
```
- Ingestión → un **log (Kafka)** particionado **por `adId`** (todo el historial de un anuncio va en orden a una partición — ver [Streams](streams.md) módulo 4); un **procesador de stream** consume y agrega por ventana de event-time.
- Agregados a un store de lectura **columnar / time-series** (`ad_id | window_start | clicks | impressions`), el mundo **OLAP** (ver [Streams](streams.md) módulo 14): muchas filas, pocas columnas, consultas que suman rangos de tiempo. El log crudo se retiene (Kafka con retención larga / volcado a un *lake*) para poder **reprocesar**.

**4. La decisión clave — agregar en el stream por *event-time*, no contar crudo en la lectura.** Si el dashboard contara los eventos crudos en cada refresh (`COUNT(*) WHERE ad_id=... AND ts BETWEEN ...`), cada carga escanearía miles de millones de filas — inviable. La técnica es **pre-agregar en el stream**: ventanas **tumbling** de 1 minuto por `adId` (ver [Streams](streams.md) módulo 9), sumando clicks a medida que llegan. La pregunta "clicks de este anuncio entre las 10:00 y 10:01" **ya está calculada**; leerla es O(1).

Y el detalle que separa al senior: agrupás por **event-time** (cuándo ocurrió el click, el timestamp embebido) y **no** por processing-time (cuándo lo procesaste). Si agruparas por processing-time, reprocesar el mismo log mañana (otro reloj) pondría los eventos en **otras ventanas** → resultado distinto para los mismos datos → la facturación no sería reproducible (ver [Streams](streams.md) módulo 7). Event-time es la condición para que recalcular dé siempre lo mismo.

**5. Donde duele — el evento tardío y el exactly-once de la facturación.** Dos problemas que definen el diseño:

- **¿Cuándo cierro la ventana?** Un click de las 10:00 puede llegar a las 10:05 (la red móvil bufferea, el cliente reconecta). Si cerrás la ventana "10:00–10:01" apenas pasa el minuto de *processing-time*, ese click queda afuera → subcontás → le cobrás de menos al anunciante. La respuesta es el **watermark** ("ya vi todo hasta T, con un margen") que cierra la ventana cuando pasa su fin, más **allowed lateness** (mantener la ventana viva un rato extra para re-emitir si llega un tardío) y un **side output** para los que llegan *demasiado* tarde → corrección en batch (ver [Streams](streams.md) módulos 8 y 10). El margen del watermark **no es gratis**: cuanto más conservador (margen grande), menos tardíos perdés pero **más tarde** ve el dashboard la ventana cerrada — es el trade-off completitud-vs-latencia, el mismo de la detección de fallas.
- **No contar de más (exactly-once para facturación).** La entrega de Kafka es **at-least-once**: si el procesador se cae y reprocesa, puede sumar el mismo click **dos veces** → le cobrás de más. Tres defensas, cada una en su capa: (a) **dedup por `eventId`** (cada evento —click o impresión— trae una idempotency key única) *antes* de sumar — un reproceso encuentra la key ya vista y la ignora; (b) **checkpointing** del estado del agregador (offset + estado de las ventanas guardados en el mismo punto del stream → restaurar reprocesa sin duplicar lo ya contado, [Streams](streams.md) módulo 11); (c) **sink idempotente**: el store de agregados se escribe con *upsert* por `(adId, window_start)`, así re-emitir una ventana sobrescribe en vez de duplicar.

> **4 capas — el evento tardío que descuadra la facturación:**
> 1. **Qué se rompe:** un click de las 10:00 llega a las 10:05; la ventana "10:00–10:01" ya cerró y emitió su total → ese click no se contó → al anunciante se le facturó de menos (y el dashboard mostró menos de lo real, *para siempre*).
> 2. **Por qué a esta escala/uso:** con miles de millones de eventos por redes móviles que bufferean y reconectan, los eventos tardíos no son la excepción sino **permanentes**; y como esto **factura plata**, cada evento mal ubicado es dinero mal cobrado, no un detalle cosmético.
> 3. **Control de corto plazo:** watermark calibrado al **p99 del lag** observado (no cerrar apenas pasa el minuto); allowed lateness para re-emitir la ventana si entra un tardío dentro del margen.
> 4. **Cambio de diseño:** agregación por **event-time** + watermark + allowed lateness + **side output** a un canal batch que corrige los muy tardíos; y para la **verdad final** de facturación, un reproceso (Lambda/Kappa) que recalcula exacto desde el log crudo. El stream da el número rápido; el batch lo reconcilia.

> **El matiz de fraude.** Una parte de los "clicks" son bots o clicks repetidos para inflar/agotar presupuestos. El **dedup por `eventId`** cubre el reintento *técnico* (el mismo evento entregado dos veces), pero **no** es detección de fraude (mismo usuario clickeando 100 veces, patrones de bot): eso es un sistema aparte (scoring, reglas, ML) que filtra el stream antes de facturar. Declaralo fuera de scope salvo que lo pidan — pero nombralo, porque en ads es donde está la plata.

**6. Trade-offs.**
- **Lambda vs Kappa** (ver [Streams](streams.md) módulo 15): el dashboard quiere "rápido y aproximado ahora", la facturación quiere "exacto" (y tolera horas). **Lambda**: el *speed layer* (stream) da el número en vivo y un *batch layer* nocturno recalcula la verdad para facturar — dos bases de código que deben coincidir. **Kappa**: un solo pipeline por event-time; para recalcular, **reprocesás el log** desde el offset afectado con el job corregido. La tendencia 2026 es Kappa, pero Lambda sobrevive justo acá, donde la facturación exige una pasada batch reconciliada e independiente. Ojo: el reproceso de Kappa es "gratis" en código pero **paga en retención** — exige que el log crudo siga guardado hasta el offset que querés recalcular, y a 2 TB/día (solo clicks) retener semanas o meses es un costo de almacenamiento real; si no podés retener el log lo suficiente, no podés reprocesar.
- **Pre-agregar vs guardar crudo:** pre-agregar por minuto da lecturas O(1), pero si **solo** guardaras los agregados no podrías re-preguntar a otra granularidad ni reprocesar tras un bug. Por eso se guardan **las dos cosas**: el log crudo (para Kappa/auditoría) **y** los agregados (para servir el dashboard).
- **Contar uniques ≠ contar eventos.** "¿Cuántos usuarios **únicos** vieron este ad?" no es sumable: no podés sumar los únicos de dos ventanas (el mismo usuario cuenta en ambas). Para cardinalidad a escala se usa **HyperLogLog** (conteo aproximado con poca memoria y error acotado), no un `COUNT(DISTINCT)` exacto. Es un eje distinto al de sumar clicks.
- **El límite del exactly-once:** vale **dentro** de Kafka (productor idempotente + transacciones); cuando el efecto cruza al store de facturación, esa escritura tiene que ser **idempotente** (upsert por ventana) o se duplica al reprocesar.

> **El cierre de criterio — no sobre-diseñar.** ¿Contás page-views de tu blog o eventos de producto de una app con miles de usuarios? Un `INSERT` por evento y un `GROUP BY date_trunc('minute', ts), ad_id` sobre Postgres con un índice alcanza; o un contador en Redis (`INCR` sobre la clave `ad:{id}:{minuto}` con TTL). Para algo más serio, un Mixpanel/GA o un ClickHouse con `INSERT` directo. El pipeline Kafka + ventanas por event-time + watermarks + Lambda/Kappa es para **volumen de red publicitaria** (miles de millones/día) donde, además, **se factura** sobre el número y un error cuesta plata.

**Ejercicios — Caso 10**
10.1 🔁 ¿Por qué se **pre-agrega** en el stream en vez de contar los eventos crudos en cada lectura del dashboard?
10.2 🧠 ¿Por qué agrupás por **event-time** y no processing-time, y qué le pasa a la facturación si usás processing-time?
10.3 🧠 La entrega es at-least-once y esto **factura plata**. Nombrá los tres mecanismos que evitan contar un click de más (o de menos) y dónde actúa cada uno.
10.4 🧠 El dashboard quiere el número **ya** y la facturación lo quiere **exacto**. ¿Cómo concilia eso Lambda, y cómo lo haría Kappa?
10.5 ✍️ Escribí `agregarPorVentana(eventos)` que agregue clicks por `(adId, ventana tumbling de 1 minuto)` **por event-time**, deduplicando por `eventId` (la entrega es at-least-once). Usá los tipos de abajo.
```ts
interface ClickEvent {
  adId: string;
  eventId: string;     // id único del evento (idempotency key)
  eventTimeMs: number; // event-time en epoch ms (cuándo ocurrió el click)
}
interface VentanaAgg {
  adId: string;
  windowStartMs: number; // inicio de la ventana de 1 min que contiene al evento
  clicks: number;
}
```

---

## Caso 11 — Google Docs / edición colaborativa en tiempo real

**1. Requisitos.**
- Funcionales: varios usuarios editan **el mismo documento a la vez**; cada uno ve los cambios de los demás casi al instante; el documento **converge** (todos terminan viendo exactamente el mismo texto, sin importar el orden en que llegaron las ediciones); cursores y selecciones de los demás (presencia); historial y undo.
- No funcionales: latencia de tipeo **imperceptible** (escribir no puede esperar un round-trip al server); **convergencia garantizada** (*strong eventual consistency*: dos réplicas que recibieron el mismo conjunto de ediciones llegan al mismo estado); funciona **offline** y sincroniza al reconectar; persistencia durable del documento.

**2. Estimación.** Acá el reto **no es el volumen** (un doc tiene pocos editores concurrentes: 2–10 típicos, decenas en el extremo) sino la **corrección bajo concurrencia**. Las operaciones son minúsculas (insertar un carácter, borrar un rango) pero **muy frecuentes** (cada tecla es una op) y **concurrentes sobre la misma región** del texto. El cuello es resolver esas ediciones simultáneas sin perder ni divergir, y mantener una conexión viva por editor — no el throughput.

**3. API y datos.** Transporte: **WebSocket** (el del **caso 2**, chat — conexión persistente bidireccional, el server empuja los cambios de los demás; ver [Tiempo real](tiempo-real.md)).
```
WS  → { tipo: "op", docId, op, baseVersion }   # el cliente manda su edición sobre la versión que tenía
WS  ← { tipo: "op", op, version }              # el server retransmite la op (ya transformada) a los demás
WS  ← { tipo: "ack", version }                 # confirma y devuelve la versión resultante
WS  ← { tipo: "presence", userId, cursor }     # cursores/selecciones de los otros editores
```
- El documento se modela como una **secuencia de operaciones** (un log append-only), no como "el último texto que se guardó". El estado actual = texto base + todas las ops aplicadas en orden. Es **event sourcing** (ver [Event-driven](event-driven.md)): el log de ops es la fuente de verdad.
- Persistencia: **snapshot** periódico del texto + el log de ops desde ese snapshot (para no replay-ear desde cero). Compactás cada N ops.

**4. La decisión clave — cómo convergen las ediciones concurrentes: OT vs CRDT.** El problema central: Ana y Bruno parten de la **misma versión** y editan a la vez. Ana inserta `"X"` en posición 0; Bruno inserta `"Y"` en posición 0. Si cada cliente aplica primero la propia y después la del otro **tal cual**, divergen para siempre: Ana ve `"XY"`, Bruno ve `"YX"`. Hay dos familias de solución, y nombrar las dos (con su trade-off) es lo que se evalúa:

- **OT (Operational Transformation):** las ops se **transforman** contra las concurrentes antes de aplicarse. Cuando llega la inserción de Bruno en pos 0 y el server ya aplicó la de Ana en pos 0, la de Bruno se transforma a "insertar en pos 1" → se preserva la **intención** de ambos y todos convergen. En la práctica se implementa con un **servidor central** que impone un **orden total** de las ops y hace la transformación (el modelo *Jupiter*, el de Google Docs); el OT *peer-to-peer* puro existe en la literatura pero es notoriamente difícil de hacer correcto. El estado es texto plano compacto; el precio es que la lógica de transformación es **notoriamente difícil** de hacer correcta (hay que cubrir todos los pares de tipos de op y garantizar sus propiedades de convergencia — TP1, y sobre todo TP2; ver "donde duele").
- **CRDT (Conflict-free Replicated Data Type):** a cada carácter se le da un **id único e inmutable**, con un **orden total** entre ids. Dos inserciones concurrentes nunca colisionan porque el lugar de cada carácter lo decide su id, no su posición numérica. La convergencia está **garantizada por construcción**, sin necesitar un server central que transforme — los clientes pueden mergear incluso **peer-to-peer**. El precio: **metadata por carácter** (cada uno arrastra su id → el documento "pesa" más) y manejar **tombstones** (los borrados no se eliminan, se marcan, para que un merge tardío no los "reviva"). Es lo que usan **Yjs** y **Automerge** (y los editores *local-first* modernos).

> **La regla de decisión.** **OT** cuando ya tenés un **servidor central autoritativo** y querés un estado compacto (Google Docs). **CRDT** cuando querés convergencia **sin coordinación central**, *offline-first* o P2P (apps *local-first* modernas). Las dos logran *strong eventual consistency*; cambian el *dónde* (server central vs cualquier réplica) y el *costo* (lógica compleja vs metadata por carácter).

> **Una tercera vía (para no idealizar).** No todo editor colaborativo es OT o CRDT puro. **Figma**, por ejemplo, usa un modelo **server-autoritativo con last-write-wins por propiedad**: *inspirado* en ideas de CRDT pero sin ser uno, y sin OT ("no usamos OT", según su propio equipo). Le alcanza porque su documento es un **árbol de objetos con propiedades** (x, color, etc.), donde "el último que escribió esa propiedad gana" es aceptable — no es texto plano que requiera preservar la intención carácter por carácter como Google Docs. La lección de criterio: la técnica la dicta la **estructura del dato** y el modelo de consistencia que tolerás, no la moda.

**5. Donde duele — convergencia bajo ops concurrentes y la reconexión offline.** El happy path (una edición a la vez) es trivial; lo difícil son dos cosas:
- **Ediciones concurrentes en la misma región.** Es el núcleo del problema. El invariante a defender es **strong eventual consistency**: todas las réplicas que vieron el mismo conjunto de ops convergen al mismo estado, *sin importar el orden de llegada*. OT lo logra transformando; CRDT, con los ids. La sutileza fina de OT: cuando dos inserciones caen en la **misma posición**, hace falta un **desempate determinista** (por ejemplo, por id de sitio/usuario) aplicado **igual en los dos clientes** — si cada uno desempata distinto, divergen igual. Eso es lo que **satisface la propiedad TP1**: transformar `a` contra `b` y `b` contra `a` tiene que converger al mismo estado, formalmente `a ∘ T(b,a) ≡ b ∘ T(a,b)` (aplicar `a` y luego `b`-transformada da lo mismo que aplicar `b` y luego `a`-transformada). Con **3 o más** ops concurrentes aparece la propiedad **TP2** (las transformaciones tienen que componer sin importar el orden) — esa es la **verdaderamente difícil**, y es justo la que el modelo centralizado de Google Docs **esquiva** al imponer un orden total desde el server en vez de resolver el caso P2P general.
- **Reconexión offline.** Un cliente editó 10 minutos sin red; al reconectar, su versión base quedó vieja (el server avanzó). OT: el server **transforma** las ops del cliente contra todo lo que pasó desde su `baseVersion`, y el cliente **a su vez transforma sus ops pendientes** contra las que recibió del server — la transformación va en **ambos sentidos**, no solo del server hacia el cliente. CRDT: cada lado le manda al otro **las ops que le faltan** y ambos mergean; los ids garantizan que el orden de aplicación no importa.

> **4 capas — la divergencia por ediciones concurrentes:**
> 1. **Qué se rompe:** dos usuarios insertan en la misma posición a la vez; sin transformación ni ids, cada cliente aplica las ops en distinto orden → los documentos **divergen permanentemente** (uno ve `"XY"`, el otro `"YX"`) y nunca vuelven a coincidir.
> 2. **Por qué a esta escala/uso:** en edición colaborativa las ops concurrentes sobre la **misma región** son la norma (dos personas en el mismo párrafo), y la red entrega los mensajes a cada cliente en **distinto orden**.
> 3. **Control de corto plazo:** un **servidor central** que imponga un **orden total** de las ops (todos las aplican en el mismo orden) — la base del enfoque OT.
> 4. **Cambio de diseño:** **OT** (transformar cada op contra las concurrentes, con desempate determinista para preservar intención) o **CRDT** (id único por carácter con orden total → convergencia sin coordinación). En ambos, *strong eventual consistency* es el invariante; sin uno de los dos, no hay convergencia.

**6. Trade-offs.**
- **OT vs CRDT** (el gran trade-off): OT = estado compacto (texto plano) pero exige server central y la transformación es difícil de implementar bien (Google tardó años en estabilizarla); CRDT = convergencia garantizada y P2P/offline-first, pero overhead de metadata por carácter y tombstones. Tendencia 2026: los editores *local-first* van a **CRDT** (Yjs/Automerge); Google Docs sigue con OT por legado y compacidad.
- **Latencia vs consistencia — UI optimista:** la edición local se aplica **de inmediato** (no esperás el round-trip, o tipear se sentiría laggy) y se **reconcilia** cuando llega la confirmación/transformación del server. Es *optimistic UI*: asumís que tu op va a entrar y corregís si la transformación movió algo.
- **Granularidad de las ops:** por carácter (más fino, más ops/mensajes) vs agrupar el tecleo en *batches* (menos tráfico, menos resolución).
- **Undo colaborativo:** deshacer es no-trivial — ¿deshacés *tu* último cambio o el último de *cualquiera*? Lo correcto es **undo selectivo por usuario** (deshacer la propia op, transformada contra lo que pasó después), no un undo global.
- **Presencia/cursores:** el cursor de otro también debe **transformarse** cuando el texto cambia **antes** de su posición; si no, su cursor "salta".

> **El cierre de criterio — no sobre-diseñar.** ¿Un campo de comentarios, o un doc que edita **una persona a la vez**? No necesitás OT ni CRDT: un **lock** ("Bruno está editando…") o *last-write-wins* sobre el documento entero alcanza. Para colaboración "suave" donde rara vez chocan en el mismo bloque (estilo Notion), podés arrancar con **bloqueo por bloque/párrafo**. OT/CRDT es la respuesta cuando varios editan **el mismo texto, carácter por carácter y en simultáneo**, como Google Docs — no para cualquier "feature colaborativo" (y ojo: edición fluida no implica OT/CRDT puro, como muestra la tercera vía de Figma).

**Ejercicios — Caso 11**
11.1 🔁 ¿Por qué se modela el documento como una **secuencia de operaciones** (un log) en vez de guardar "el texto final"? ¿Qué habilita?
11.2 🧠 Ana inserta `"X"` en pos 0 y Bruno `"Y"` en pos 0 a la vez. ¿Por qué divergen si cada cliente aplica las ops tal cual, y cómo lo resuelve OT (transformación) frente a CRDT (ids)?
11.3 🧠 OT vs CRDT: ¿qué gana y qué paga cada uno, y en qué situación elegís cada uno?
11.4 🧠 ¿Por qué la edición local se aplica **de inmediato** (UI optimista) en vez de esperar la confirmación del server, y qué hay que reconciliar después?
11.5 ✍️ Implementá `transformInsert(op, contra)` (el corazón de OT para dos inserciones): ajustá la posición de `op` asumiendo que `contra` (una inserción concurrente) **ya se aplicó**. Regla: si `contra` quedó **antes** de `op`, corré `op` a la derecha por el largo de `contra`; si caen en la **misma** posición, desempatá por `siteId` (determinista, igual en ambos clientes). Usá los tipos de abajo.
```ts
interface InsertOp {
  pos: number;
  texto: string;
  siteId: number; // id del editor; desempata inserciones en la misma posición
}
```
11.6 🧠 El ✍️ de arriba resuelve **insert-vs-insert**, el caso más limpio. ¿Por qué transformar un **borrado** (contra una inserción concurrente, o contra otro borrado) es más difícil, y qué pasa cuando dos borrados afectan **rangos que se solapan**?

---

## Caso 12 — Caché distribuida (diseñar un Redis/Memcached a escala)

**1. Requisitos.**
- Funcionales: un servicio de **caché en memoria** clave→valor, compartido por muchas instancias de la aplicación; `GET`/`SET`/`DELETE` con **TTL**; escala horizontal (agregás nodos para más capacidad y throughput).
- No funcionales: latencia **sub-milisegundo**; throughput altísimo (es el camino caliente delante de la base de datos); **alta disponibilidad** (que se caiga un nodo no puede tumbar todo); poder **agregar/quitar nodos** moviendo la menor cantidad de datos posible; y la propiedad que lo define: es un **caché**, la fuente de verdad es la DB → perder datos es tolerable, servir datos *incorrectos* mucho menos.

**2. Estimación.** Si el sistema hace 1M req/s y el caché tiene buen *hit rate* (~90%), son ~900K ops/s contra el caché. Datos: 1 KB por entrada × 100M entradas activas = **100 GB** → no entra en un solo nodo con headroom y réplicas (un nodo típico tiene 64–256 GB pero no querés llenarlo) → hay que **particionar** entre varios nodos. El cuello no es el cómputo: es **repartir las claves** entre nodos de forma que (a) la carga quede balanceada y (b) agregar o perder un nodo **no** remapee todo el keyspace.

**3. API y datos.**
```
GET    key            → value | miss
SET    key value TTL  → ok
DELETE key            → ok
```
- Cada nodo guarda una porción del keyspace **en memoria** (un hashmap + una estructura para la evicción).
- Alguien —el cliente, un proxy, o el propio cluster— decide **a qué nodo** va cada key. Esa decisión es el corazón del caso.

**4. La decisión clave — repartir las claves: consistent hashing.** ¿A qué nodo va una key con `N` nodos?
- **Hash módulo N** (`hash(key) % N`): trivial, pero si `N` cambia (agregás un nodo o se cae uno), **casi todas** las claves remapean a otro nodo → el caché entero se invalida de golpe → *todas* las requests pasan a la DB al mismo tiempo (avalancha). Inaceptable.
- **Consistent hashing:** mapeás **claves y nodos al mismo anillo** (un espacio de hash circular, p. ej. 0…2³²−1). Una key pertenece al **primer nodo que encontrás girando** en el anillo desde la posición de la key. Al agregar o quitar un nodo, **solo las claves entre ese nodo y su vecino** se remapean (≈ `1/N` del keyspace), no todo.
- **Virtual nodes (vnodes):** cada nodo físico se coloca en **muchos** puntos del anillo (cientos de "réplicas virtuales"), no en uno solo. Dos motivos: (1) con pocos nodos y un punto cada uno, el reparto queda **desparejo** (un nodo se come un arco gigante); muchos puntos promedian y emparejan la carga. (2) Cuando un nodo cae, su porción se **redistribuye entre varios** nodos (los dueños de cada vnode vecino), no toda hacia un único sucesor que se sobrecargaría. (Ojo: el balance mejora con la **raíz** del número de vnodes, no es exacto — con pocos nodos esperá un spread de ±varios puntos porcentuales, no un reparto perfecto.)

**5. Donde duele — el cache stampede y el nodo caído.**
- **Cache stampede (thundering herd).** Una key **muy popular** expira o se invalida; en ese instante, miles de requests concurrentes ven el *miss* a la vez y **todas** recomputan / pegan a la DB al mismo tiempo → la DB se funde, que es justo lo que el caché protegía. Tres defensas, no excluyentes: (a) **single-flight / lock por key** — solo el primero que ve el miss recomputa; los demás **esperan** ese resultado (un lock por key, o `singleflight`); (b) **early/probabilistic expiration** — recomputás la key *antes* de que venza, con probabilidad creciente al acercarse al TTL (el algoritmo canónico, *XFetch*, **pondera** esa probabilidad por el **costo de recálculo**, para refrescar antes las entradas más caras), de modo que no expire para todos en el mismo instante; (c) **stale-while-revalidate** — servís el valor viejo mientras **uno** lo refresca en background. Y `jitter` en los TTL para **desincronizar** los vencimientos.
- **Nodo caído.** Si un nodo muere, su porción de claves se pierde (es caché, tolerable) pero todas esas requests caen a la DB de golpe (un mini-stampede localizado). El consistent hashing con vnodes **reparte** ese golpe entre varios nodos; si no podés tolerar ni la pérdida, **replicás** cada partición en 2 nodos — con réplica **síncrona** no perdés datos ya confirmados (pero cada write paga la copia); con réplica **asíncrona** (más rápida) queda una **ventana** en la que un fallo pierde los writes más recientes (el eje async/sync de los trade-offs).

> **4 capas — el stampede de la key caliente:**
> 1. **Qué se rompe:** la key más popular expira; miles de requests ven el *miss* en el mismo instante y **todas** recomputan/pegan a la DB → la DB colapsa (justo lo que el caché evitaba).
> 2. **Por qué a esta escala/uso:** a ~900K ops/s una sola key caliente concentra miles de req/s, y un TTL fijo hace que **expire para todos a la vez** (vencimiento sincronizado), no escalonado.
> 3. **Control de corto plazo:** **single-flight** (un solo recompute por key; el resto espera) + **jitter** en los TTL para desincronizar los vencimientos.
> 4. **Cambio de diseño:** **stale-while-revalidate** (servir lo viejo mientras se refresca) + **early probabilistic expiration**; nunca dejar que la masa entera dependa de una key que vence en seco.

**6. Trade-offs.**
- **Dónde se decide el routing:** **cliente inteligente** (conoce el anillo, cero saltos extra, pero hay que propagar la topología a todos los clientes — el modelo de Redis Cluster) vs **proxy** (twemproxy/Envoy: cliente simple, pero un salto más de latencia y un componente más que operar) vs **server-side redirect** (los nodos reenvían al dueño correcto).
- **Invalidación — "the two hard things".** **Cache-aside** (el patrón más común: la app lee del caché, en *miss* va a la DB y **puebla** el caché; en *write* invalida o actualiza la entrada) vs **write-through** (escribís a caché y DB juntos: consistente pero cada write paga la DB) vs **write-back** (escribís al caché y a la DB después: rapidísimo, pero si el nodo muere antes de bajar a la DB, **perdés writes**). El **TTL** es la red de seguridad que acota cuánto puede durar una inconsistencia.
- **Evicción:** **LRU** (lo más usado), **LFU**, FIFO, random; el trade-off es qué patrón de acceso favorece y cuánto cuesta mantener el orden de acceso.
- **Réplicas:** asíncronas (rápidas, podés leer un valor *stale* de una réplica) vs síncronas (consistentes, más lentas) — el mismo eje de [Replicación](replicacion.md).

> **El cierre de criterio — no sobre-diseñar.** ¿Una app con una o pocas instancias y un dataset que **entra en memoria**? Un **Redis/Memcached de un solo nodo** (o el cluster manejado de tu cloud) alcanza y sobra: **cache-aside con TTL** y listo. El consistent hashing, los vnodes y las defensas de stampede son para cuando el caché **no entra en un nodo** y hay que particionar, o cuando una **key caliente** puede tumbar tu DB. Para la enorme mayoría de los casos, "poné un Redis adelante con TTL" **es** la respuesta correcta — no lo conviertas en un sistema distribuido sin necesidad.

**Ejercicios — Caso 12**
12.1 🔁 ¿Por qué `hash(key) % N` se vuelve un problema cuando cambia la cantidad de nodos, y qué resuelve el **consistent hashing**?
12.2 🧠 ¿Qué dos problemas resuelven los **virtual nodes** en el anillo que el consistent hashing "a secas" (un punto por nodo) no resuelve bien?
12.3 🧠 Explicá el **cache stampede**: qué lo dispara a escala y nombrá **dos** defensas distintas, diciendo qué hace cada una.
12.4 🧠 **Cache-aside** vs **write-through** vs **write-back**: ¿qué garantiza y qué arriesga cada uno respecto de la consistencia caché↔DB?
12.5 ✍️ Implementá un *consistent hash ring* mínimo con **vnodes**: `agregarNodo`, `quitarNodo` y `nodoPara(key)` (el nodo responsable de la key, o `undefined` si el anillo está vacío). Usá la interfaz y el hash de abajo.
```ts
// hash determinista de string → entero de 32 bits sin signo (FNV-1a). Asumilo dado.
function hash(s: string): number {
  let h = 0x811c9dc5;
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 0x01000193);
  }
  return h >>> 0;
}
```
12.6 🧠 Un nodo del caché **muere**. ¿Qué les pasa a sus claves, por qué los **vnodes** amortiguan el golpe mejor que un anillo con un punto por nodo, y cuándo justificás **replicar** cada partición (y qué consistencia resignás al hacerlo)?

---

## Caso 13 — YouTube / Netflix: streaming de video

**1. Requisitos.**
- Funcionales: alguien **sube** un video (archivo grande); el sistema lo **procesa** y lo deja disponible para que millones lo **miren** en cualquier dispositivo y calidad; reproducción fluida con *seek*. (YouTube: contenido de usuarios (UGC) infinito; Netflix: catálogo curado y acotado.)
- No funcionales: subida resumible de archivos enormes (como el caso 5); **lectura-intensivo extremo** (un video se ve millones de veces); **arranque rápido** y reproducción **sin buffering** aunque la red del usuario varíe; durabilidad del original; y el dato que manda el diseño: el costo dominante es **almacenamiento + ancho de banda de salida (egress)**, no CPU.

**2. Estimación.** El *write path* es chico comparado con el *read path*: se suben cientos de horas de video por minuto, pero se miran **miles de millones de horas por día**. Un video popular: millones de vistas, cada una transfiriendo cientos de MB a GB → el egress agregado es del orden de **petabytes/día**. Esa sola cifra dicta la arquitectura: **servir todo eso desde tus datacenters es inviable** (ni el ancho de banda ni la geografía dan) → el **CDN** no es un add-on, es la pieza central del read path.

**3. API y datos.** Separá tajantemente el **write path** (upload + processing) del **read path** (playback).
```
POST /videos                  { título, ... }   → { videoId, uploadUrl }   # presigned URL (caso 5)
PUT  (subida del archivo original al blob store)
GET  /videos/{id}/manifest                       → manifiesto HLS/DASH (renditions + listas de segmentos)
GET  (segmentos .ts/.m4s)                         → servidos por el CDN, no por el origin
```
- **Original** a **object storage** (S3/GCS): durable, barato (como el caso 5).
- Un **pipeline de transcoding** produce varias **renditions** (240p/480p/720p/1080p/4K) en uno o más códecs, cada una partida en **segmentos** de pocos segundos.
- **Metadata** (título, estado de procesamiento, manifiestos) en una DB; los **segmentos** viven en el blob store y se sirven **vía CDN**.

**4. La decisión clave — transcoding paralelo + adaptive bitrate (ABR).** Dos piezas que definen el caso:
- **Transcoding como fan-out de jobs.** Subir el original no alcanza: hay que transcodificarlo a muchas resoluciones/códecs/bitrates. Se **parte** el video en trozos en **límites de GOP/keyframe** (cada chunk arranca en un *IDR-frame*, para que se pueda decodificar solo) y cada trozo × rendition se transcodifica **en paralelo** sobre una flota de workers (es el patrón **cola de trabajos** del caso 7), y después se ensambla. (La *scene-cut detection* es otra cosa: ayuda a **colocar keyframes** y mejorar la calidad, no es la unidad de reparto.) Es *embarrassingly parallel* → escala agregando workers. Es **asíncrono**: el video queda "procesando" hasta que termina (avisás por webhook/notificación — caso 3).
- **Adaptive Bitrate (HLS / DASH).** No se sirve un archivo monolítico. Cada rendition se corta en **segmentos** de ~2–10 s y un **manifiesto** (playlist `.m3u8` de HLS / MPD de DASH) lista las renditions disponibles y, para cada una, sus segmentos. El **reproductor del cliente** mide su ancho de banda y, **segmento a segmento, elige la calidad** que la red aguanta: arranca bajo (para empezar rápido), sube si hay banda, baja si se congestiona → reproducción sin cortes. La decisión de calidad vive en el **cliente**, no en el server.

**5. Donde duele — servir el egress (CDN) y el video viral.**
- **El CDN es el read path.** Servir petabytes desde el origin es imposible por ancho de banda y por latencia geográfica. El CDN **cachea los segmentos** en *edge PoPs* cerca del usuario; el origin solo se toca en *cache miss*. El arranque es rápido porque el primer segmento sale del edge más cercano. Los segmentos son **inmutables** (un segmento transcodificado nunca cambia) → cacheables con TTL larguísimo, ideales para CDN.
- **El video viral y la cola larga.** Cuando un video explota, todos piden **los mismos** segmentos: el CDN lo absorbe naturalmente, porque el contenido es **idéntico para todos** y cacheable (a diferencia de un feed personalizado, que no se puede cachear igual). El problema inverso es la **cola larga**: millones de videos que casi nadie ve no conviene pre-transcodificarlos en todas las calidades (storage carísimo) → se transcodifican **bajo demanda**, o se difieren las renditions caras (4K/AV1) hasta que el video **acumule vistas** (las básicas/H.264 primero, las pesadas cuando gana tracción).

> **4 capas — el origin que se funde sin CDN:**
> 1. **Qué se rompe:** un video se vuelve viral y sus segmentos no están en el CDN (nunca se cachearon, o venció el TTL) → millones de requests caen al **origin**, que colapsa por ancho de banda.
> 2. **Por qué a esta escala/uso:** el egress de video es el recurso más caro y escaso; un solo video popular genera **terabytes/hora**, imposible de servir desde el datacenter (banda y geografía).
> 3. **Control de corto plazo:** TTL largo para los segmentos (son inmutables); **pre-warming** del CDN para estrenos esperados (Netflix **empuja** el contenido a los edges *antes* del lanzamiento).
> 4. **Cambio de diseño:** segmentos **inmutables** cacheados en un CDN **multi-tier** (edge → regional → *origin shield*, para que un miss no golpee al origin directo), **ABR** para que el cliente **degrade** la calidad en vez de buffer-ear, y transcoding **diferido/por demanda** para la cola larga (las renditions caras solo cuando el video gana tracción).

**6. Trade-offs.**
- **Pre-transcoding (Netflix) vs on-demand (cola larga).** Pre-transcodificar todo el catálogo da playback instantáneo en toda calidad, pero cuesta storage por cada rendition de cada título; on-demand ahorra storage a costa de latencia la primera vez que alguien pide esa calidad. La decisión la fija la **distribución de la demanda**: catálogo curado y muy visto (Netflix pre-transcodifica todo, e incluso hace **per-title encoding**: elige la *escalera* de bitrates óptima **según el contenido** —animación plana vs acción ruidosa— en vez de una escalera fija, y ese costo extra se amortiza por las millones de vistas) vs UGC con cola larga brutal (YouTube, más *lazy*).
- **Tamaño de segmento:** chico = adaptación más fina y arranque más rápido, pero más requests/overhead; grande = lo inverso.
- **Códecs:** H.264 (lo decodifica todo) vs H.265/VP9/AV1 (mejor compresión → menos banda, pero más CPU de transcoding y no todos los dispositivos lo soportan) → servís varios y el cliente elige el que puede.
- **DRM:** Netflix necesita protección (Widevine/FairPlay → segmentos cifrados + intercambio de licencias); YouTube en general no. Cambia el pipeline.
- **VOD vs live:** todo lo anterior es *video on demand*. El **live** agrega encoding en tiempo real y segmentos muy cortos para bajar la latencia — otro problema; declaralo fuera de scope salvo que lo pidan.

> **El cierre de criterio — no sobre-diseñar.** ¿Tu app tiene unos pocos videos (un tutorial, un demo, clips cortos)? Subilos a S3, servilos con un **CDN administrado** (CloudFront/Cloudflare) y embebé un `<video>` o un player como **Mux / Cloudflare Stream**, que ya te dan transcoding y ABR llave en mano. El pipeline de transcoding propio, el per-title encoding y el CDN multi-tier son para cuando **distribuir video ES el producto** a escala de millones de horas. Para todos los demás, un servicio administrado de video es la respuesta correcta.

**Ejercicios — Caso 13**
13.1 🔁 ¿Por qué no se sirve un único archivo de video sino **varias renditions partidas en segmentos**, y qué es el **manifiesto**?
13.2 🧠 ¿Por qué el **transcoding** se modela como un fan-out de jobs paralelos, y qué patrón de un caso anterior reusa?
13.3 🧠 ¿Cómo funciona el **adaptive bitrate** y por qué la decisión de qué calidad servir vive en el **cliente** y no en el server?
13.4 🧠 ¿Por qué el **CDN** es la pieza central del read path de video, y por qué los segmentos son ideales para cachear (a diferencia de un feed personalizado)?
13.5 🧠 Netflix pre-transcodifica todo su catálogo; YouTube no necesariamente. ¿Por qué difiere la estrategia y qué la decide?
13.6 ✍️ Implementá el selector de ABR del cliente: `elegirRendition(anchoBandaKbps, renditions)` que devuelva la rendition de **mayor** calidad cuyo bitrate quepa en el ancho de banda disponible (con un **factor de seguridad** para no saturar); si **ninguna** entra, devolvé la **más baja** (degradar antes que cortar); si no hay renditions, `undefined`. Usá el tipo de abajo.
```ts
interface Rendition {
  altura: number;     // 240, 480, 720, 1080, ...
  bitrateKbps: number;
}
```

---

## Caso 14 — Payment ledger / billetera (contabilidad de doble entrada)

**1. Requisitos.**
- Funcionales: registrar **movimientos de dinero** entre cuentas (transferencias, pagos, cobros, depósitos); consultar el **saldo** de cualquier cuenta y su **historial**; cada transacción mueve plata de una(s) cuenta(s) a otra(s).
- No funcionales: **correctitud absoluta** (es dinero: ni un centavo de más ni de menos); **auditabilidad** (todo movimiento rastreable e inmutable, para compliance y auditorías años después); **consistencia fuerte** (el saldo nunca queda a medias); **idempotencia** (un reintento no puede duplicar un cargo); durabilidad altísima. No es lectura-masiva como un feed — acá **la corrección manda sobre todo lo demás**.

**2. Estimación.** El reto **no es throughput extremo** (aunque un procesador grande haga miles de tx/s) sino **integridad e invariantes**: que la suma del sistema **cuadre siempre**, que ninguna transacción quede a medias, que se pueda **reconstruir y auditar** el pasado. El dato crece **sin parar** (append-only, nunca se borra) → con el tiempo, archivado en frío y snapshots de saldo.

**3. API y datos — el corazón: doble entrada (*double-entry*).**
```
POST /transfers   { from, to, amount, Idempotency-Key }  → { transferId }
GET  /accounts/{id}/balance                               → { balance }
GET  /accounts/{id}/entries                               → { entries[] }
```
- El modelo **no** es "una columna `balance` que sumás y restás". Es un **ledger de doble entrada**: cada transacción genera **≥2 asientos (entries)** que **suman cero** — un **débito** en una cuenta y un **crédito** igual en otra. Invariante sagrado: **la suma de los asientos de una transacción = 0** (la plata no se crea ni se destruye, solo se mueve).
- `entries`: `entryId | transactionId | accountId | amount (con signo) | createdAt` — **append-only e inmutable**. Nunca se updatea ni se borra; un error se arregla con una **transacción compensatoria** (un asiento que revierte), no editando el pasado.
- El **balance** de una cuenta = **suma de sus asientos**, no una columna mutable. Se puede **materializar** (un saldo cacheado) por performance, pero la fuente de verdad es el log.

**4. La decisión clave — doble entrada + inmutabilidad + atomicidad.** Tres invariantes que se sostienen juntos:
- **Doble entrada:** todo movimiento son ≥2 asientos que suman cero. Esto hace al sistema **auto-verificable**: en cualquier momento, la suma de **todos** los asientos debe ser cero (o igual al dinero externo que entró/salió). Si no cuadra, hay un bug — y lo **detectás**. Es la contabilidad de hace 500 años aplicada a software.
- **Inmutabilidad (append-only):** los asientos nunca cambian. Da **auditabilidad total** (el historial *es* la verdad) y elimina una clase entera de bugs de concurrencia (no hay updates compitiendo sobre una fila de balance). Para corregir, **compensás** (revertís con asientos nuevos), no editás.
- **Atomicidad:** los ≥2 asientos de una transacción se escriben **en una sola transacción de DB** (todos o ninguno). Si el débito entra y el crédito no, se perdió plata → inaceptable. Acá reusás la **consistencia fuerte del caso 8** (una base transaccional como Postgres, no un store eventual). Esto vale cuando ambas cuentas viven en **la misma DB**; si el movimiento **cruza servicios/DBs**, **no** hay transacción distribuida → la atomicidad pasa a ser *lógica*: asiento en `pending` + **saga** con compensación (lo vemos en el paso 5). Son dos regímenes distintos, no una contradicción: misma DB → una tx ACID; cruza límites → saga.

**5. Donde duele — idempotencia financiera, saldo bajo concurrencia y money movement entre servicios.**
- **Idempotencia (no doble cargo).** La red reintenta; un `POST /transfers` reintentado no puede cobrar dos veces. Reusás la **idempotency key store-and-return del caso 8**: la primera vez creás la transacción y **guardás el resultado** contra la key; un reintento con la misma key **devuelve el resultado guardado**, no crea otra. El índice `unique` sobre la key es la red de seguridad.
- **Saldo insuficiente bajo concurrencia.** Si dos débitos concurrentes sobre la misma cuenta podrían dejarla en negativo (*overdraft* no permitido), tenés que verificar el saldo **atómicamente** al asentar — el mismo *check-then-act* del caso 8, ahora sobre "¿hay saldo?". En la práctica eso es un `SELECT ... FOR UPDATE` sobre la **fila de la cuenta** (su **balance materializado**, no re-sumando el log en la ruta caliente — eso sería O(n)), y recién entonces asentás. El append-only evita los updates en conflicto sobre el historial, pero la **regla de negocio** "no sobregirar" igual necesita **serializar por cuenta**. Cuidado: si una transferencia lockea **dos** cuentas, **ordená los locks** por `accountId` (siempre el menor primero) para no caer en *deadlock*.
- **Mover plata entre servicios (saga).** Mover dinero entre tu ledger y un procesador externo (Stripe, un banco) no es una transacción ACID única → **saga con compensación** (caso 8). Importante: el ledger **solo contiene asientos `posted`** (la verdad inmutable); el estado **`pending → posted | failed`** vive en la **capa de orquestación/saga**, no como un asiento que después se edita. Si el paso externo falla, **no** tocás nada: emitís un asiento **compensatorio** que revierte. El estado intermedio es explícito en la saga, nunca implícito.

> **4 capas — el asiento que entra a medias:**
> 1. **Qué se rompe:** una transferencia escribe el **débito** de A pero el **crédito** a B falla (crash/timeout) → A perdió plata que nadie recibió y el sistema **deja de cuadrar**.
> 2. **Por qué a esta escala/uso:** es dinero real y hay concurrencia; cualquier escritura **parcial** viola el invariante de suma-cero y, peor, es plata perdida o duplicada — no un detalle cosmético.
> 3. **Control de corto plazo:** escribir los ≥2 asientos en **una transacción de DB atómica** (todos o ninguno); idempotency key para que el reintento no duplique.
> 4. **Cambio de diseño:** ledger de **doble entrada, append-only e inmutable** (se corrige compensando, no editando); transacciones cross-servicio como **saga** con asientos de compensación y estados explícitos (`pending → posted/failed`); **balance derivado** de la suma de asientos, auditable y **reconciliable**.

**6. Trade-offs.**
- **Balance materializado vs sumar el log.** Sumar todos los asientos cada vez es O(n) y se vuelve lento → se **materializa** un saldo (snapshot + asientos desde el snapshot, el *event sourcing* del caso 11). Trade-off: performance vs una única fuente de verdad. Una **reconciliación periódica** (re-sumar el log y comparar con el saldo materializado) detecta cualquier *drift*.
- **Inmutabilidad vs storage.** Append-only crece para siempre → archivado de asientos viejos en frío + snapshots para no re-sumar todo desde el origen.
- **Consistencia fuerte (CP) elegida a propósito.** Como el caso 8, ante una partición preferís **rechazar** antes que arriesgar un saldo incorrecto. Un ledger **no** es eventual.
- **Precisión numérica.** **Jamás `float` para dinero** (errores de redondeo binario: `0.1 + 0.2 !== 0.3`) → enteros en la unidad mínima (**centavos**) o un decimal de precisión fija. Y ojo con el lenguaje: el `number` de JS **es** un float de 64 bits (enteros exactos solo hasta 2⁵³), así que en producción el monto se **persiste** como `NUMERIC`/decimal de la DB o `bigint`; el `number` del ejemplo de abajo vale como ilustración didáctica, no como tipo de producción. Es el error de junior más clásico del caso.
- **Multi-moneda.** El invariante "suma = 0" es **por moneda**: cada asiento lleva su `currency` y **nunca** mezclás monedas en una misma ecuación de cero. Una **conversión** no es un asiento cross-currency mágico: son asientos en cada cuenta de moneda más una **cuenta puente de FX** que absorbe el diferencial de redondeo de la tasa fraccional (así cada moneda cuadra por separado). Declaralo fuera de scope salvo que lo pidan, pero sabé que ese es el modelo.

> **El cierre de criterio — no sobre-diseñar.** ¿Llevás "créditos", puntos o likes internos de una app, sin requisitos de auditoría financiera? Una tabla con una columna `balance` y updates transaccionales puede alcanzar. El ledger de doble entrada, la inmutabilidad y la reconciliación son para cuando manejás **dinero real** (o algo que se audita como tal): pagos, billeteras, marketplaces que liquidan a terceros. Ahí el *double-entry* **no es over-engineering, es el estándar** — y un auditor te lo va a exigir.

**Ejercicios — Caso 14**
14.1 🔁 ¿Por qué el **balance** de una cuenta se modela como la **suma de asientos inmutables** y no como una columna que sumás y restás?
14.2 🧠 ¿Qué es la **doble entrada** y qué invariante te da "gratis" que vuelve al sistema **auto-verificable**?
14.3 🧠 ¿Por qué los asientos son **append-only/inmutables**, y cómo corregís un error si no podés editar el pasado?
14.4 🧠 ¿Por qué los ≥2 asientos de una transacción deben escribirse en **una sola transacción de DB**, y qué dos mecanismos de casos anteriores reusás para "no doble cargo" y "no sobregirar"?
14.5 ✍️ Implementá `asientosTransferencia(from, to, amountCentavos)`: generá los **dos asientos** de doble entrada (débito en `from`, crédito en `to`), validá que el monto sea un **entero positivo de centavos** (nada de `float`) y que los asientos **sumen 0**. Usá el tipo de abajo.
```ts
interface Entry {
  accountId: string;
  amount: number; // en centavos (entero); negativo = débito, positivo = crédito
}
```

---

## Caso 15 — Order matching engine (motor de matching de órdenes de un exchange)

**1. Requisitos.**
- Funcionales: aceptar **órdenes** (compra/venta, límite o market) sobre un símbolo; mantener un **order book** (libro de órdenes) por símbolo; **casar** (*match*) órdenes cuando se cruzan los precios (un compra a precio ≥ la mejor venta); ejecutar **trades** con **fills parciales** (una orden grande se come varias del otro lado); **cancelar/modificar** órdenes; publicar **market data** (mejor bid/ask, profundidad, trades) a los participantes.
- No funcionales: **latencia ultrabaja y predecible** (p99 en microsegundos–milisegundos; el **jitter** acotado importa tanto como el promedio); **fairness y correctitud** (prioridad **precio-tiempo** estricta: nadie se cuela); **throughput alto** (cientos de miles a millones de órdenes/s en picos, agregado); **recovery exacto** (tras un crash reconstruís el book *idéntico*, sin perder ni duplicar órdenes); **auditabilidad** regulatoria (reconstruir qué pasó y por qué). Acá **la latencia es el producto** y la fairness está regulada.

**2. Estimación.** No es un problema de storage como un feed. El reto es **latencia/throughput por símbolo** y **determinismo**. El order book de un símbolo líquido tiene a lo sumo **decenas de miles de órdenes vivas**: 10 000 órdenes × ~100 bytes ≈ **1 MB** → entra holgado en RAM (incluso 1 M de órdenes ≈ 100 MB). El **match** en memoria son **nanosegundos–microsegundos**; el costo real está en la **red, la serialización y la persistencia**, no en cruzar dos órdenes. Un solo core casa millones de órdenes simples por segundo. Para anclar por qué la DB no puede estar en el camino crítico: un match en RAM son **decenas–cientos de nanosegundos**, contra los **milisegundos** de un round-trip a una DB relacional — un factor de **10⁴–10⁶**. La estimación acá no dimensiona discos: justifica que **el book vive en memoria** y que el cuello de botella no es el matching.

**3. API y datos — el order book + prioridad precio-tiempo.**
```
POST  /orders            { symbol, side, type, price, qty, clientOrderId }  → { orderId, seq }
PATCH /orders/{orderId}  { price?, qty? }                                  → { orderId, seq }
DELETE /orders/{orderId}
(stream) market data: trades + cambios del book   (WebSocket / multicast)
```
- En un exchange real el protocolo es **binario** (FIX/SBE sobre TCP/UDP), no JSON/HTTP — cada microsegundo cuenta. El REST de arriba es la versión didáctica del contrato.
- **Modificar tiene una trampa de fairness:** cambiar el **precio** o **subir** la cantidad de una orden viva la trata como **cancel + nueva orden** → **pierde la prioridad de tiempo** (va al final de la cola FIFO de su nivel). Solo **bajar** la cantidad la conserva. Es un detalle entrevistable: un *modify* no es un update barato, es reordenar en el book.
- El **order book** de un símbolo tiene **dos lados**: *bids* (compras, ordenadas de mayor a menor precio) y *asks* (ventas, de menor a mayor). Cada **nivel de precio** es una **cola FIFO** de órdenes.
- **Prioridad precio-tiempo (*price-time priority*):** se ejecuta primero el **mejor precio** (el bid más alto contra el ask más bajo); a **igual precio**, la que **llegó antes** (FIFO dentro del nivel). Esto se refleja en la estructura: un mapa **ordenado** de precio→nivel (árbol/skip-list, o arrays indexados por precio para baja latencia) y, dentro de cada nivel, una **cola**.

**4. La decisión clave — procesamiento secuencial determinista por símbolo.**
- El matching de un símbolo se hace **single-threaded**: un único hilo procesa el flujo de órdenes **en orden de llegada**. ¿Por qué no paralelizar? Porque el book es **estado compartido mutable** y el resultado depende del **orden exacto** (fairness/time priority). Serializar **elimina los locks**, vuelve el resultado **determinista** (mismo input → mismo output) y es **más rápido** que coordinar threads: sin contención, todo el estado caliente vive en cache de CPU. Es el patrón **LMAX Disruptor** / *mechanical sympathy*.
- **Secuenciador (*sequencer*):** antes del engine, un componente asigna a cada orden un **número de secuencia monótono** y la escribe al **log de entrada** (event sourcing) **antes** de procesarla. El engine consume ese log en orden. Ese log + el determinismo son lo que después da el recovery.
- **Sharding por símbolo:** distintos símbolos corren en **engines/cores distintos en paralelo** (no comparten book). Un trade siempre ocurre **dentro** de un símbolo, así que ningún match cruza shards → escalás horizontalmente por símbolo sin coordinación.

**5. Donde duele — recovery, fills parciales, fan-out de market data y fairness en ráfaga.**
- **Recovery determinista (el corazón).** El engine es una **máquina de estados en memoria**: si el proceso muere, el book desaparece. Solución = **event sourcing** (el *event sourcing* del caso 11/14): el secuenciador persiste **toda** entrada (órdenes, cancelaciones) al log **antes** de procesarla, y se toman **snapshots** periódicos del book. Para recuperar: cargás el último snapshot y **reproducís** el log desde ahí. Como el matching es **determinista**, el replay reconstruye el book **exacto** (mismas secuencias → mismo estado).
- **Alta disponibilidad (hot standby).** Un engine por símbolo es un **SPOF por diseño**. La réplica corre como **hot standby** consumiendo el **mismo log secuenciado** y aplicando las mismas transiciones → mantiene un book **idéntico** (replicación de **máquina de estados**, el terreno de [consenso](consenso.md)). En el failover, el standby toma el control desde la **última secuencia confirmada**. La clave: ambos consumen el log en el **mismo orden**, por eso convergen al mismo estado.
- **Fills parciales.** Una orden grande puede **comerse varios niveles** del otro lado → genera **múltiples trades** y deja un **remanente**: si es límite, entra al book; si es market (o IOC) y no hay más liquidez, se descarta. El matching **itera**: mientras la entrante cruce el mejor precio contrario y le quede cantidad, ejecuta contra la **primera de la cola FIFO**, descontando de ambas.
- **Fan-out de market data.** Cada trade y cambio del book debe llegar a **miles de participantes** con baja latencia → el engine **no** lo hace en el hot path (lo frenaría). El engine **emite** un stream de eventos y un **publisher aparte** hace el fan-out, típicamente por **multicast UDP** (un mensaje, N receptores) en lugar de N envíos. Es el **fan-out de lectura del caso 1**, llevado al límite de latencia.
- **Fairness en ráfaga.** Ante un *burst*, las órdenes se **encolan en el secuenciador** y se procesan **en orden de secuencia**: la latencia sube, pero la **fairness se mantiene** (nadie se cuela). No se descartan órdenes en silencio; si la cola se llena, se aplica **backpressure** o se **rechaza explícitamente** — un silencio acá es una orden perdida, inaceptable.

> **4 capas — el engine que se cae con el book en RAM:**
> 1. **Qué se rompe:** el proceso de matching de un símbolo crashea → el order book (estado en memoria) **desaparece** en pleno trading; las órdenes vivas se pierden.
> 2. **Por qué a esta escala/uso:** el book vive **solo en RAM** por latencia y es **un único hilo por símbolo** (SPOF por diseño); cada microsegundo de caída son trades perdidos y exposición regulatoria.
> 3. **Control de corto plazo:** un **hot standby** que consumió el **mismo log secuenciado** ya tiene el book idéntico → toma el control desde la última secuencia confirmada (failover en milisegundos).
> 4. **Cambio de diseño:** **event sourcing** — secuenciador que persiste cada entrada al log **antes** de procesar + **snapshots** periódicos; recovery = snapshot + **replay determinista**; **replicación de máquina de estados** para HA.

**6. Trade-offs.**
- **Single-thread vs paralelizar (contraintuitivo).** Para *este* problema un solo hilo es **más rápido** que muchos: sin locks, datos calientes en cache, predicción de saltos estable. Paralelizás **entre símbolos**, nunca dentro de uno. El techo de un símbolo es **un core**; si un único símbolo satura un core (rarísimo), es un problema de producto, no de arquitectura.
- **In-memory vs DB transaccional.** A diferencia del caso 8/14 (Postgres, ACID en el hot path), acá una DB relacional en el camino crítico es **demasiado lenta** (importan los microsegundos). El estado vive en RAM; la **durabilidad** la da el **log de eventos** (append secuencial, lo más rápido que hay), no la DB. Es **event sourcing puro**: el log es la fuente de verdad, el book es una **proyección**.
- **El determinismo cuesta disciplina.** Para que el replay reconstruya el book exacto, el engine **no puede** usar nada no determinista en el hot path: ni **reloj de pared** (los timestamps los pone el secuenciador y van en el log), ni random, ni orden de iteración de un hashmap, ni I/O bloqueante. Es el **mismo principio que las *activities* deterministas** de [workflows durables](workflows.md).
- **Latencia vs durabilidad.** Persistir/replicar **antes** de confirmar el match agrega latencia; hacerlo **después** arriesga perder una orden ya ackeada. En finanzas el balance es claro: persistir al log y replicar al standby **antes** de confirmar — perder una orden confirmada es inaceptable.
- **Precios y cantidades: enteros, jamás `float`.** Como el caso 14, se trabaja en **enteros de la unidad mínima** (*ticks*, centavos): un error de redondeo binario en un precio es un **trade mal ejecutado**, no un detalle cosmético.

> **El cierre de criterio — no sobre-diseñar.** ¿Un marketplace que casa compradores y vendedores **sin** urgencia de microsegundos (clasificados, una subasta que cierra en horas, asignar pedidos a repartidores)? Una **DB transaccional** con `SELECT ... FOR UPDATE` y un job que casa órdenes alcanza y sobra — es el **caso 8**. El order book en memoria, el secuenciador, el replay determinista y el hot standby son para cuando la **latencia es el producto** y la **fairness está regulada**: un exchange financiero, un *ad exchange* en tiempo real, un motor de subastas de alta frecuencia. Para casi todo lo demás, **"casá en la DB con una transacción" es la respuesta correcta** — no construyas un Disruptor sin necesidad.

**Ejercicios — Caso 15**
15.1 🔁 ¿Qué es la **prioridad precio-tiempo** y cómo se refleja en la **estructura de datos** del order book?
15.2 🧠 ¿Por qué el matching de un símbolo se procesa **single-threaded** en vez de paralelizarlo con varios hilos, y dónde sí paralelizás?
15.3 🧠 El engine tiene el book **solo en RAM**. ¿Cómo lo recuperás **exacto** tras un crash, y por qué el **determinismo** es justo lo que lo vuelve posible?
15.4 🧠 ¿Por qué el **fan-out de market data** no lo hace el matching engine en el hot path, y qué patrón de qué caso anterior reusás?
15.5 ✍️ Implementá `matchearCompra(entrante, asks)`: la orden de compra entrante se casa contra el lado de **asks** (ya ordenado por precio ascendente; dentro de cada precio, la primera del array es la más vieja → FIFO). Generá los **trades** por prioridad precio-tiempo, soportá **fills parciales**, ejecutá cada trade **al precio de la orden que ya estaba en el book**, y devolvé los trades más el **remanente** sin casar de la entrante. En la solución, explicá además qué pasa con ese remanente según el **tipo** de orden (límite → entra al book; market/IOC → se descarta). Usá los tipos de abajo.
```ts
interface Order {
  id: string;
  qty: number;   // cantidad restante (entero, unidad mínima)
  price: number; // en ticks (entero); nunca float
}
interface Trade {
  buyOrderId: string;
  sellOrderId: string;
  qty: number;
  price: number;
}
```
15.6 🧠 ¿Por qué precios y cantidades se guardan como **enteros** en la unidad mínima (ticks/centavos) y **nunca** como `float`? ¿Qué se rompe si usás coma flotante en un matching engine?

---

## Caso 16 — Sistema RAG / asistente LLM (diseñar un "chatea con tus documentos")

> Este caso es la versión **system design de entrevista** del pipeline que [RAG](rag.md), [Vector DBs](vector-dbs.md) y [Agentes](agentes.md) cubren a nivel implementación. Acá no enseñamos a programar el RAG: aplicamos el método para **dimensionarlo, costearlo y decidir trade-offs** como en una entrevista — porque el system design de IA ya entra en el loop para roles senior.

**1. Requisitos.**
- Funcionales: el usuario pregunta en **lenguaje natural** sobre un corpus (docs internos, base de conocimiento, tickets de soporte); el sistema **recupera** los pasajes relevantes y **genera** una respuesta **fundamentada en ellos, con citas** a la fuente; soporta **conversación multi-turno** (follow-ups); y dice **"no lo sé"** cuando no hay evidencia, en vez de inventar.
- No funcionales: **fundamentación (*groundedness*)** — la respuesta se apoya en fuentes reales y citables, no alucina; es **la métrica que más importa**. **Latencia** tolerable de chat (primer token en <~1 s con streaming; respuesta completa en segundos). **Costo por consulta acotado** (el LLM cobra **por token**, de entrada y de salida — no por request). **Frescura** sin reentrenar (si cambia un doc, la respuesta lo refleja en minutos, no hay que tocar el modelo). **Evaluable en producción** (el LLM es **no determinista** → no hay `assert` clásico).

**2. Estimación — el napkin math de tokens, costo y latencia.** Lo que distingue al system design de IA: la unidad de costo y latencia es el **token**, no el request.
- **Corpus e índice (una sola vez):** 100k documentos × ~1000 tokens ≈ **100M tokens**. Chunking en ~500 tokens → **~200k chunks**. Un embedding por chunk (dim ~1024) → 200k × 1024 × 4 bytes ≈ **~820 MB** de vectores → entra holgado en RAM de un índice HNSW (pgvector o un motor vectorial; HNSW vive **en memoria**, por eso después la búsqueda es de milisegundos). El embedding lo hace un **proveedor de embeddings dedicado** (Voyage, OpenAI, Cohere), no el LLM de generación.
- **Por consulta:** recuperás top-k (k≈6) chunks ≈ 6×500 = 3000 tokens de contexto + system prompt (~1k) + pregunta (~200) ≈ **~4200 tokens de entrada**; salida ≈ **~400 tokens**.
- **Costo por consulta** con un modelo tier-alto a $5/1M entrada y $25/1M salida: 4200 × $5/1M = **$0.021** + 400 × $25/1M = **$0.010** → **~$0.031/consulta**. A 1M consultas/mes ≈ **~$31k/mes** solo en generación. **Tres palancas:** (a) **modelo más barato** para lo simple ($1/$5 → ~5× menos); (b) **prompt caching** del system prompt e instrucciones estables (~0.1× la lectura) si el prefijo se repite; (c) **Batch API** (~50%) para lo no interactivo. El embedding de la query es **despreciable** (1 vector por consulta).
- **Latencia:** embed de la query (~50 ms) + búsqueda vectorial (~10–50 ms, con el índice HNSW **en RAM**) + **el LLM, que domina** (primer token en cientos de ms, luego ~50–100 tokens/s). Con **streaming** el usuario ve el primer token enseguida aunque la respuesta total tarde varios segundos. Conclusión de la estimación: **el LLM domina costo y latencia**; todo lo demás es ruido.

**3. API y datos — el pipeline de recuperación + generación.**
```
POST /chat  { conversationId, message }  → (stream SSE de tokens) + { fuentes[] } al cierre
```
- **Streaming es la decisión de UX clave:** sin él, el usuario espera segundos en blanco; con SSE, ve la respuesta formarse token a token.
- **Datos:** tabla `chunks` (`id | documentId | texto | embedding(vector) | metadata`: tenant, fecha, **permisos**) + el documento original en object storage + el **índice vectorial (HNSW)** sobre el embedding + el historial por `conversationId`.
- **El pipeline RAG, en una línea:** query → **embed** de la query → **búsqueda vectorial** top-k (filtrando por metadata/permisos) → **armar contexto** (chunks + system prompt + historial, dentro del presupuesto de tokens) → **LLM** con instrucción de citar → **stream + fuentes**.

**4. La decisión clave — fundamentar la generación (RAG) en vez de confiar en el modelo.** El LLM solo "sabe" lo de su entrenamiento (con *cutoff*) y **alucina con seguridad**. RAG inyecta **evidencia recuperada** en el prompt para que la respuesta se base en **tus** datos, actuales y citables. Tres piezas:
- **Recuperación:** índice vectorial (semántica) e idealmente **híbrida** con léxica (BM25, ver [Búsqueda a escala](busqueda.md)) fusionada con **RRF** (que combina por **posición de ranking**, no por el score crudo —que no es comparable entre vectorial y BM25) — la vectorial sola falla con nombres propios, códigos o términos exactos.
- **Generación fundamentada:** el system prompt instruye *"respondé SOLO con el contexto; si no está, decí que no lo sabés; citá la fuente"*. El LLM es una **dependencia no determinista** → la misma pregunta puede dar respuestas distintas.
- **Citas:** cada afirmación apunta a un chunk fuente, **verificable** por el usuario. (Matiz de API: el structured output suele ser **incompatible con las citas nativas** del proveedor —el modo JSON ocupa el canal de salida que el proveedor usaría para emitir las citas— → las fuentes van como un **campo del esquema** o se mapean por `documentId`.)

**5. Donde duele — alucinación, presupuesto de contexto, latencia/costo, frescura y eval en producción.**
- **Alucinación / *groundedness* (el modo de falla central).** El modelo puede ignorar el contexto o inventar. Mitigación: prompt estricto + **buena evidencia** (si el retrieval trae basura, la generación es basura — *garbage in, garbage out*); **umbral de score** (si el mejor chunk está por debajo, responder "no tengo información"); **fallback a "no sé"**; verificación opcional con un **LLM-juez** que chequea que la respuesta esté soportada por las fuentes (suma latencia/costo → reservar para alto riesgo o muestrear offline, como en [Evals](evals.md)). Aguas arriba, la **estrategia de chunking** (tamaño + solapamiento, corte fijo vs semántico) es una palanca de groundedness tan decisiva como el prompt: chunks mal cortados parten una idea en dos y el top-k nunca trae la evidencia completa (el detalle vive en [RAG](rag.md)).
- **Presupuesto de contexto.** No metés todo: k muy grande → caro **y** *lost in the middle* (el modelo ignora el medio del contexto). Armás el contexto priorizando los chunks de mayor score + historial reciente comprimido. **Ojo:** la ventana de 1M de los modelos actuales no cambia que **cada token se re-paga en cada turno**.
- **Latencia y costo.** El LLM domina; las palancas del napkin math (streaming, modelo más chico, prompt caching) + **caché de respuestas** para preguntas idénticas o muy similares (*semantic cache*).
- **Frescura sin reentrenar (la gracia de RAG).** Cuando un doc cambia, **re-ingestás solo ese doc** (re-chunk + re-embed + **upsert por `documentId`**, borrando los chunks viejos) y la próxima consulta ya lo ve — **no se toca el modelo**. Es el "día 2" del ingest, **idempotente por `documentId`** (reusa la idempotencia del caso 8).
- **Eval en producción.** No hay `assert` clásico (no determinista). **Offline:** *golden set* de preguntas con respuesta/fuentes esperadas; métricas de **groundedness** (¿soportado por las fuentes?), **recall@k** del retrieval (¿el chunk correcto entró en el top-k?) y **corrección** (LLM-juez calibrado contra humano). **Online:** loggear cada consulta (pregunta, chunks, respuesta, latencia, costo, 👍/👎) para cazar regresiones.

> **4 capas — el modelo que alucina una respuesta con cita inventada:**
> 1. **Qué se rompe:** el usuario pregunta algo fuera del corpus (o el retrieval trae chunks irrelevantes) y el LLM **inventa** una respuesta plausible y falsa, a veces con una cita inexistente.
> 2. **Por qué a esta escala/uso:** el LLM es no determinista y está entrenado para **sonar seguro**; sin evidencia buena, rellena. A escala, una fracción de respuestas falsas erosiona la confianza, y en dominios regulados es responsabilidad legal.
> 3. **Control de corto plazo:** prompt estricto ("solo con el contexto; si no está, decí que no lo sabés"); **umbral de score** del retrieval (por debajo → "no tengo información"); **citar fuentes verificables**.
> 4. **Cambio de diseño:** retrieval **híbrido + reranking** para subir la calidad de la evidencia; **LLM-juez de groundedness** en el camino (alto riesgo) o muestreado (offline); **golden set + logging + feedback** para medir y atajar regresiones; **human-in-the-loop** donde el error sale caro.

**6. Trade-offs.**
- **RAG vs fine-tuning vs contexto largo.** RAG para conocimiento **que cambia y necesita citas** (la mayoría de los casos); **fine-tuning** para **estilo/formato/tarea**, no para hechos frescos; meter **todo el corpus** en la ventana de 1M es **caro** (se re-paga por turno) y sufre *lost in the middle* — RAG recupera solo lo relevante. No son excluyentes; se combinan.
- **Calidad del retrieval vs costo.** Vectorial sola es barata pero floja en exactitud léxica; **híbrida + reranker** mejora groundedness pero suma latencia y un modelo más. Trade-off **recall ↔ latencia/costo** al subir k.
- **Modelo caro vs barato + routing.** Modelo tier-alto para respuestas difíciles/razonamiento; modelo chico para clasificación y respuestas simples; **routing** (un modelo barato evalúa la complejidad y enruta) baja el costo. El **tiering es la palanca de costo #1**.
- **Frescura del índice.** Re-ingest inmediato (caro, muchas escrituras) vs batch nocturno (barato, *stale*) — el mismo dilema que el *freshness* de [Búsqueda](busqueda.md).
- **Determinismo: no hay.** Misma pregunta, respuestas distintas. Para evals reproducibles fijás el golden set y medís **distribuciones**, no igualdad exacta (caso [Evals](evals.md)).

> **El cierre de criterio — no sobre-diseñar.** ¿Pocos documentos que **entran en la ventana de contexto** (un manual, un puñado de PDFs)? Metelos **directo en el prompt** (con prompt caching si la consulta se repite) y salteate el índice vectorial entero — es "RAG" sin vector DB. El pipeline de embeddings, el índice HNSW, el retrieval híbrido y el reranking son para cuando el corpus **no entra en contexto**, **cambia seguido**, o necesitás **citar y filtrar por permisos**. Para casi todo lo demás, **"poné los docs en el prompt" es la respuesta correcta** — no construyas un pipeline de recuperación sin necesidad. La implementación de todo esto vive en [RAG](rag.md), [Vector DBs](vector-dbs.md), [Búsqueda](busqueda.md) y [Evals](evals.md).

**Ejercicios — Caso 16**
16.1 🔁 ¿Qué problema del LLM resuelve **RAG**, y en qué se diferencia de **fine-tuning**?
16.2 🧠 Napkin math: con k=6 chunks de ~500 tokens, system prompt ~1k, pregunta ~200 y salida ~400, a $5/1M (entrada) y $25/1M (salida), ¿cuánto cuesta **una** consulta y qué **tres palancas** la abaratan?
16.3 🧠 El modelo alucina una respuesta con una cita inventada. Recorré las **4 capas** (qué se rompe / por qué / control de corto plazo / cambio de diseño).
16.4 🧠 ¿Por qué meter **todo el corpus** en la ventana de 1M **no** reemplaza al retrieval, y **cuándo sí** saltearías el índice vectorial?
16.5 ✍️ Implementá `armarContexto(pregunta, chunks, presupuestoTokens, estimarTokens)`: los `chunks` vienen **ordenados por score descendente**; incluí los de mayor score hasta **agotar el presupuesto de tokens** (sin pasarte), construí el prompt con la instrucción de fundamentar + citar, y devolvé el prompt junto con las **fuentes únicas** (por `documentId`, en orden de aparición). Usá los tipos de abajo.
```ts
interface Chunk {
  id: string;
  documentId: string;
  texto: string;
  score: number;
}
interface Contexto {
  prompt: string;
  fuentes: string[]; // documentId de cada chunk incluido, para citar
}
```

## Caso 17 — Distributed job scheduler (programar millones de jobs en el tiempo)

> Cierra el banco con el patrón que más se confunde con "una cola". Una **cola** (caso 3/7) responde *"¿qué ejecuto ahora?"*; un **scheduler** responde *"¿qué debe ejecutarse y exactamente cuándo?"*. Acá el reto **no** es ejecutar el job —eso es el worker pool del caso 7— sino **decidir qué disparar y en qué instante, a escala, sin dispararlo de más**. La teoría de durable execution, sagas y "el cron que no debe dispararse N veces" vive en [Workflows durables](workflows.md); este caso es el **diseño del servicio de scheduling** que está debajo.

**1. Requisitos.**
- Funcionales: programar un job **one-shot diferido** ("en 30 min", "el 2026-07-01 a las 09:00") o **recurrente** (cron: "todos los días a las 3am"); **cancelar/reprogramar**; al vencer, **entregar** el job a un ejecutor; **reintentar** si la ejecución falla.
- No funcionales: **escala** (decenas–cientos de millones de jobs programados vivos); **puntualidad acotada** (disparar cerca de la hora prometida — un job "a las 9:00" no puede salir 9:05; el SLA de puntualidad es un número, no "lo antes posible"); **at-least-once + sin duplicados *visibles*** (cada job se dispara al menos una vez; los duplicados los absorbe la idempotencia del ejecutor, no el scheduler); **durabilidad** (un job programado sobrevive a un crash — no se pierde un cobro mensual porque el nodo se reinició); **alta disponibilidad** (el scheduler no puede ser un SPOF). El reto central: **encontrar los jobs vencidos a escala y dispararlos una sola vez**.

**2. Estimación.** No es un problema de storage. 100M jobs × ~300 bytes (payload + schedule + metadata) ≈ **30 GB** → entra en una DB cómodamente. La tasa **promedio** es engañosa: 100M jobs repartidos en un mes ≈ **~38 disparos/s**, trivial. El reto son los **picos**: si apenas el **1%** son cron de medianoche (`0 0 * * *`), **1 millón de jobs con el mismo vencimiento exacto** → un **frente de 10⁶ disparos a drenar de golpe** (no una tasa sostenida de 38/s), cinco órdenes de magnitud sobre el promedio. La estimación acá no dimensiona discos: justifica que **se dimensiona para el pico, no para el promedio**, y que la pregunta cara es *"¿cuáles de los 100M vencen ahora?"* — que tiene que costar un *range scan*, no un *full scan*.

**3. API y datos — el índice por tiempo.**
```
POST   /jobs            { runAt | cron, payload, idempotencyKey }  → { jobId }
PATCH  /jobs/{jobId}    { runAt? | cron? }
DELETE /jobs/{jobId}
```
- **Modelo de datos:** tabla `jobs` (`jobId | next_run_at | cron | payload | status | locked_until | owner`) con un **índice sobre `next_run_at`**. Esa columna indexada es lo que vuelve barata la pregunta del paso 2: *"dame los vencidos"* es `WHERE next_run_at <= now()` → un **range scan** sobre el índice, no un escaneo de 100M filas (encontrarlos es barato; **reclamarlos** sigue acotado a N por tick — ver la tormenta de medianoche).
- **Recurrentes = one-shot que se re-agenda.** Un cron no es una entidad especial: guardás la **expresión**, y al disparar **calculás el próximo `next_run_at`** y lo reprogramás (mismo registro, nuevo vencimiento). Así el motor de disparo solo entiende de "`next_run_at` vencido"; la recurrencia es un detalle del re-agendado.

**4. La decisión clave — cómo encontrar los vencidos y reclamarlos exactamente una vez.**
- **Polling de un índice ordenado por tiempo** (el enfoque por defecto): cada *tick* (p. ej. cada 1 s), `SELECT ... WHERE next_run_at <= now() ORDER BY next_run_at FOR UPDATE SKIP LOCKED LIMIT N`. Simple, **durable** (la DB es la fuente de verdad) y escala a millones. El **intervalo de polling fija la granularidad** de puntualidad: poll cada 1 s → disparás con ~1 s de jitter.
- **Timer wheel en memoria** (para puntualidad sub-segundo): una rueda de *timing wheel* que avanza tick a tick da disparo de milisegundos y programa/cancela/expira en **O(1) amortizado** (lo usan Kafka Purgatory y Netty), frente al O(log n) del índice de la DB; pero su estado vive en RAM → necesita durabilidad y recovery aparte. El **híbrido** habitual: la **DB es la verdad durable**, y un timer wheel en memoria carga solo la **ventana próxima** (los jobs de los próximos N minutos) para dispararlos con precisión.
- **Disparo sin duplicar (el corazón).** Con varios nodos scheduler corriendo para HA, hay que evitar que dos disparen el mismo job. Dos vías:
  - **Claim atómico:** `SELECT ... FOR UPDATE SKIP LOCKED` (Postgres) **+ un `UPDATE locked_until = now()+lease, owner = yo` en la misma transacción**. Ojo: `SKIP LOCKED` por sí solo toma el lock de fila **solo mientras dura la transacción** —al commitear se suelta—; lo que hace que el lease **sobreviva al commit** (y que un crash justo después no deje el job ni perdido ni eternamente bloqueado) es **escribir** `locked_until`/`owner` en la fila. Es el **claim del caso 8** (reservas) y la **cola del caso 7**: cada nodo agarra un lote distinto sin pisarse.
  - **Sharding + leader election:** particionás los jobs por `jobId` (cada nodo dueño de un rango → un solo nodo mira cada job, sin pelear por locks), y al caer un nodo se **reasignan sus shards** vía *leader election* ([consenso](consenso.md), [workflows](workflows.md) M5). Escala mejor pero agrega rebalanceo.
- **Separación scheduler ↔ ejecutor.** El scheduler **decide cuándo** y **encola**; la **cola + worker pool ejecuta y reintenta** (caso 3/7). No mezclar: el scheduler no ejecuta el trabajo en el hot path del disparo, solo lo despacha.

**5. Donde duele — la tormenta de medianoche, duplicados, clock skew, leases huérfanos y catch-up.**
- **La tormenta de medianoche (thundering herd de cron).** Millones de jobs `0 0 * * *` vencen en el mismo segundo → el pico satura el dispatch y los workers. Mitigación: **jitter** (esparcir el disparo en una ventana de ±minutos si el job lo tolera — casi siempre lo tolera); **dispatch a tasa acotada** (el scheduler encola a un ritmo techado y la **cola absorbe el pico** — la frontera acotada del caso 7 + el desacople por colas del caso 3); y **dimensionar los workers para el pico**, no para el promedio. Es **backpressure**: no dispares más rápido de lo que el sistema aguas abajo puede tragar.
- **At-least-once → duplicados (el modo de falla central).** El scheduler reclama un job, lo encola, y **crashea antes de marcarlo como disparado** → al recuperar, lo vuelve a disparar. **Exactly-once de punta a punta es imposible** (el mismo límite del caso 3 y de [workflows](workflows.md)): la garantía real es **at-least-once + idempotencia en el ejecutor** (el `idempotencyKey` del caso 8 deduplica el segundo disparo). Marcar el disparo en la misma transacción que el claim cierra el hueco *claim↔marca*, pero **el encolado vive fuera de la DB** (la cola es otro sistema) → la ventana sigue abierta; el patrón canónico para acercarla a cero es un **transactional outbox** (escribir el "a encolar" en la misma transacción + un relay que publica), y **aun así el ejecutor debe ser idempotente**. El scheduler promete *"se dispara"*, no *"se dispara una sola vez"*.
- **Clock skew.** Cada nodo scheduler tiene su propio reloj; si cada uno decide "vencido" contra su **reloj de pared**, uno dispara temprano y otro tarde, y dos nodos pueden creer que un job todavía no vence (o ya venció) en desacuerdo. Mitigación: una **única fuente de tiempo** para el due-check (el `now()` de la **DB**, o relojes NTP-sincronizados con tolerancia explícita). Es el mismo cuidado que el *clock skew* de los IDs Snowflake ([datos-escala](datos-escala.md) M13).
- **Leases huérfanos.** Un nodo reclama un job (lock) y muere **sin dispararlo ni soltarlo** → el job queda "claimed" para siempre y nunca se ejecuta. Mitigación: el claim es un **lease con expiración** (`locked_until = now + leaseMs`); si el lease vence sin confirmación de disparo, **otro nodo lo re-reclama**. Es el *visibility timeout* de SQS llevado al scheduler.
- **Catch-up de recurrentes (missed runs).** El scheduler estuvo caído 2 horas; al volver, ¿dispara los cron que se perdió o salta al próximo? Es una **decisión de producto explícita**: **catch-up** (dispara los atrasados — peligroso: si son muchos, es una tormenta auto-infligida al arrancar) vs **skip-and-realign** (saltea al próximo vencimiento válido). Quartz lo llama *misfire policy*; lo importante en la entrevista es **nombrar el dilema y elegir**, no que haya una única respuesta.

> **4 capas — el scheduler que dispara un job dos veces:**
> 1. **Qué se rompe:** el scheduler reclama un job, lo encola y **crashea antes de confirmar el disparo**; al recuperar, lo dispara de nuevo → el ejecutor corre el job **dos veces** (un cobro duplicado, dos emails).
> 2. **Por qué a esta escala/uso:** con HA hay **varios nodos** mirando el mismo conjunto de jobs y los crashes son rutina a escala; **exactly-once de disparo es imposible** entre "reclamar" y "marcar disparado" hay una ventana sin atomicidad cross-sistema.
> 3. **Control de corto plazo:** **claim atómico** (`FOR UPDATE SKIP LOCKED`) para que dos nodos no reclamen el mismo job a la vez; **lease con expiración** para no perder un job cuyo dueño murió, sin dispararlo eternamente.
> 4. **Cambio de diseño:** asumir **at-least-once** y empujar la unicidad al **ejecutor idempotente** (`idempotencyKey`, caso 8); marcar el disparo en la **misma transacción** que el claim (o un **transactional outbox** para el encolado, que vive fuera de la DB); **sharding + leader election** para que un solo nodo sea dueño de cada job.

**6. Trade-offs.**
- **Polling (DB) vs timer wheel (memoria).** Polling: simple, durable, escala — pero la granularidad es el intervalo de poll (sub-segundo barato no se consigue). Timer wheel: preciso al ms, pero estado en RAM → durabilidad y recovery aparte. El híbrido (DB durable + ventana próxima en memoria) compra precisión sin perder durabilidad, a costa de complejidad.
- **Centralizado (un leader dispara) vs particionado (cada nodo su shard).** Leader único: simple, cero doble-disparo, pero el techo de throughput es **un nodo** y es un SPOF (mitigado por re-elección). Particionado por `jobId`: escala horizontal, pero hay que **rebalancear shards** al cambiar la membresía del cluster ([consenso](consenso.md)).
- **Puntualidad vs costo.** Poll cada 100 ms = más puntual, más carga sobre la DB; cada 10 s = barato pero ±10 s de jitter. Se elige por el **SLA de puntualidad**, no por gusto — y se mide con el **scheduling lag** (el p99 de `disparo_real − next_run_at`), la métrica que valida que el diseño cumple la puntualidad prometida en los requisitos.
- **At-least-once es el techo (no lo pelees).** Igual que el caso 3 y [workflows](workflows.md): no prometas exactly-once de disparo; prometé **at-least-once + idempotencia**. Pelear por exactly-once de disparo es perseguir una garantía que el sistema no puede dar.

> **El cierre de criterio — no sobre-diseñar.** ¿Unos pocos jobs recurrentes, una sola máquina, y tolerás que un disparo se pierda si esa máquina está caída? El **`cron` de Linux**, un `setTimeout`, o el **`repeat` de BullMQ** ([Redis](redis.md)) alcanzan y sobran. ¿Necesitás cron en un **cluster** sin que dispare N veces? Un **lock distribuido por tick** o **leader election** ([workflows](workflows.md) M5) lo resuelve sin construir un servicio. El escalón intermedio más común en producción —y lo que implementa el ejercicio 17.5— es una **tabla `jobs` + un worker que pollea con `FOR UPDATE SKIP LOCKED`**: resuelve la enorme mayoría de los casos sin un servicio dedicado. El scheduler distribuido completo (índice por tiempo, claim con lease, **sharding + leader election + timer wheel**) es para cuando programás **millones** de jobs con **SLA de puntualidad** y **HA real** — un EventBridge Scheduler / Cloud Scheduler / Quartz a escala. Para casi todo lo demás, **"usá el cron que ya tenés" es la respuesta correcta** — no construyas un scheduler distribuido sin necesidad. La teoría de durable execution, sagas y el cron-que-no-dispara-N-veces vive en [Workflows durables](workflows.md); las colas y el worker pool en [Redis](redis.md) y los casos 3/7; la idempotencia del ejecutor en el caso 8.

**Ejercicios — Caso 17**
17.1 🔁 ¿Por qué la columna `next_run_at` **con índice** es lo que vuelve barata la pregunta "¿qué jobs vencen ahora?", y cómo se modela un job **recurrente** (cron) sin que el motor de disparo tenga que entender de cron?
17.2 🧠 La **tormenta de medianoche**: millones de cron `0 0 * * *` vencen en el mismo segundo. ¿Qué se rompe y con qué tres palancas lo mitigás?
17.3 🧠 Recorré las **4 capas** del modo de falla central: el scheduler reclama un job, lo encola y crashea antes de marcarlo como disparado.
17.4 🧠 **Polling** de un índice por tiempo vs **timer wheel** en memoria: ¿qué trade-off tiene cada uno y qué **híbrido** usarías para puntualidad sub-segundo sin perder durabilidad?
17.5 ✍️ Implementá `reclamarVencidos(jobs, ahora, leaseMs, limite)`: `jobs` viene **ordenado por `nextRunAt` ascendente** (el orden del índice). Reclamá hasta `limite` jobs que estén **vencidos** (`nextRunAt <= ahora`) **y libres** (sin lease, o con el lease **expirado**), marcándolos in-place con un **lease nuevo** (`lockedUntil = ahora + leaseMs`), y devolvé los reclamados. Esto modela el `FOR UPDATE SKIP LOCKED` + lease del dispatcher. Usá los tipos de abajo.
```ts
interface Job {
  id: string;
  nextRunAt: number;          // epoch ms del próximo disparo
  lockedUntil: number | null; // lease: hasta cuándo está reclamado; null = libre
}
```
17.6 🧠 Tu scheduler estuvo caído 2 horas y al volver hay cientos de cron recurrentes que "se perdieron" su disparo. ¿Qué políticas tenés (catch-up vs skip-and-realign) y cuándo elegís cada una?

---

# Soluciones

> Mirá esto solo después de intentar cada caso en voz alta. Casi todas son de **criterio**: si tu razonamiento difiere pero justifica bien el trade-off, probablemente también esté bien. En diseño de sistemas, el *cómo* justificás importa más que el *qué* elegís.

### Caso 1 — Feed / timeline
**1.1** **Fan-out on write (push):** al postear, escribís el post en el timeline precomputado de cada seguidor; leer el feed es O(1) (ya está armado). **Fan-out on read (pull):** no precomputás nada; al leer, juntás los posts recientes de todos los que seguís y los mezclás. El que **abarata las lecturas** es el push (pagás al escribir, leés gratis); el pull abarata las escrituras a costa de lecturas caras.

**1.2** Porque un post de una cuenta con millones de seguidores dispara millones de escrituras de fan-out: satura la cola, atrasa los timelines de todos y desperdicia trabajo (la mayoría de esos seguidores ni abren la app). El **híbrido** lo resuelve no haciendo push para las cuentas por encima de un umbral de seguidores: sus posts se **pullean en el momento de leer** y se mezclan con el timeline precomputado (push) de las cuentas normales. Lectura = (lo precomputado) ⊕ (los pocos famosos que seguís, traídos por pull).

**1.3** El timeline precomputado se llena cuando los seguidos postean; para el arranque conviene **pull**: una query de los posts recientes de las 500 cuentas seguidas, mezclados como timeline inicial. Es un caso puntual (un evento, no por request), así que el costo del pull es aceptable y evita esperar. Clave: **backfilleá** ese resultado en el timeline precomputado (no lo uses solo para la primera pantalla), así el primer scroll profundo no vuelve a pagar pull y las lecturas siguientes son O(1). A partir de ahí, los posts nuevos entran por push como siempre.

### Caso 2 — Chat en tiempo real
**2.1** Porque el chat necesita que el **server empuje** mensajes al cliente apenas llegan (push), y HTTP polling obliga al cliente a preguntar repetidamente (latencia + carga inútil). WebSocket es una conexión **persistente bidireccional**: el server manda sin que el cliente pregunte. El principal cuello de botella no es CPU sino **mantener millones de conexiones abiertas** simultáneas (memoria y file descriptors por conexión) y rutear entre ellas.

**2.2** La red da at-least-once; el efecto exactly-once se logra con **dos idempotency keys**: el **`clientMsgId`** (lo genera el cliente) permite que el **server deduplique** un reenvío tras un ACK perdido; el **`msgId`** (lo asigna el server, ordenable) permite que el **cliente deduplique** si recibe el mismo mensaje dos veces. Cada punta descarta duplicados por su key → el usuario ve cada mensaje una vez.

**2.3** Porque el patrón de acceso es "traeme **los mensajes de esta conversación** en orden" — shardear por `conversationId` mantiene todos los mensajes de una charla **juntos en el mismo shard**, así leer el historial o el delta es una query local, no un scatter-gather entre shards. Shardear por `userId` partiría una conversación entre dos shards (emisor y receptor); por `msgId` esparciría una misma charla por todos lados. La clave de sharding sigue al patrón de lectura.

### Caso 3 — Notificaciones push
**3.1** Porque entregar a APNs/FCM/SMS es lento, falible y con rate limits externos; si el productor del evento esperara la entrega en línea, lo acoplaría a esos proveedores y se caería con ellos. Devolver `202` y encolar **desacopla**: el productor sigue, y la cola **absorbe los picos** (backpressure) drenando al ritmo que los proveedores aguantan.

**3.2** Una **idempotency key** (`dedupKey` derivada del evento): el worker, **antes de despachar**, hace un insert atómico de la key (`INSERT ... ON CONFLICT DO NOTHING` o `SET NX` en Redis); si ya existía, no reenvía. Actúa **antes del envío al proveedor** y el worker ackea la cola **después** de registrar el envío, de modo que un reproceso (worker que murió sin ackear) encuentra la key ya puesta y no duplica.

**3.3** Si comparten cola, el millón de newsletters **bloquea** al OTP detrás de ellos (head-of-line blocking): el OTP, que es urgente y time-sensitive, llega tarde o inútil. Se resuelve con **colas separadas por prioridad** (o un tier dedicado para lo time-sensitive), de modo que los mensajes urgentes se drenan por su propio carril sin esperar al bulk.

### Caso 4 — Typeahead
**4.1** Un **trie** (árbol de prefijos): cada nodo representa un prefijo y guarda sus top-k completaciones. Buscar "lap" navega 3 nodos y devuelve la lista — el costo es O(longitud del prefijo), **no** O(cantidad de términos). Por eso da igual tener mil o mil millones de términos: lo que importa es cuántas letras tiene el prefijo.

**4.2** Porque calcular el ranking por popularidad en cada request (sobre millones de términos, a 500K req/s) sería prohibitivo. Precomputar el top-k en cada nodo hace que la respuesta **ya exista** y leerla sea O(1) por nodo. Lo que se sacrifica es la **frescura**: como el trie se reconstruye en batch (cada minutos/horas), un término que se volvió popular hace 30 segundos todavía no aparece. Latencia instantánea a cambio de sugerencias levemente desactualizadas — un trade-off aceptable para typeahead.

**4.3** Son los más **peligrosos** porque la distribución de prefijos es muy sesgada: un puñado de prefijos cortos ("a", "fa", "lo") concentra un porcentaje enorme del tráfico → son *hot keys* que pueden fundir un nodo. Y son los más **fáciles de mitigar** justamente porque son pocos y muy repetidos: cachearlos en CDN/edge y en memoria con TTL cubre la mayoría del tráfico con un set chiquito de entradas. Pocos prefijos, muchísimo hit rate.

### Caso 5 — Almacenamiento de archivos
**5.1** Porque tienen naturalezas opuestas: la **metadata** es chica, estructurada y transaccional (nombre, versión, qué chunks componen el archivo) → encaja en una base relacional con consultas e integridad; el **contenido** es enorme, opaco e inmutable → encaja en **object storage** (S3/GCS), que da durabilidad altísima, costo bajo y escala infinita sin tocar tu base. Mezclarlos (blobs en la base) la haría lenta y carísima.

**5.2** (1) **Deduplicación:** si dos archivos (o dos usuarios) comparten un chunk, su hash coincide y lo guardás **una sola vez**. (2) **Sync por delta:** al editar un archivo, solo cambian los hashes de algunos chunks; subís **solo esos**, no el archivo entero (ahorra ancho de banda). (3) **Integridad:** recalcular el hash del chunk bajado y compararlo con su id verifica que no se corrompió en tránsito ni en disco.

**5.3** Primero los **blobs** (chunks a S3), verificás que están todos, y **después** commiteás la **metadata** que los referencia. Si lo hicieras al revés (metadata primero), habría una ventana en la que el archivo "existe" según la base pero sus bytes no están subidos → un lector pide el archivo y le falta un chunk = corrupto. Publicar la metadata solo cuando el contenido está confirmado garantiza que toda versión visible es completa.

### Caso 6 — Rate limiter distribuido
**6.1** Porque el límite debe valer **a través de todas las instancias** del API: si cada instancia contara en su propia memoria, con M instancias el cliente podría hacer M×N requests (N por instancia) en vez de N. Redis es un **estado compartido** central, single-thread y atómico, que todas las instancias consultan, así el contador es uno solo para todo el cluster.

**6.2** Porque no es atómico: si el proceso (o Redis en un fallo puntual) se interrumpe **entre** el `INCR` y el `EXPIRE`, la clave queda **sin TTL** y nunca se resetea → el cliente queda limitado para siempre (su contador nunca baja). Un **script Lua** colapsa ambas operaciones en una unidad que Redis ejecuta **atómicamente** (es single-thread y corre el script entero sin intercalar otros comandos): o se hacen las dos o ninguna, nunca queda a medias. (Alternativa para fixed window simple: `SET key 1 EX ttl NX` + `INCR`.)

**6.3** **Fail-open:** si Redis no responde, **dejás pasar** las requests (no aplicás límite) → protegés la disponibilidad de tu API a costa de perder el límite temporalmente. **Fail-closed:** **rechazás** las requests → protegés el recurso limitado a costa de tumbar tu API por un problema del limiter. Elegís según qué protege el límite: si es abuso/costo, normalmente **fail-open** (con un límite local de respaldo por instancia); si protege un recurso frágil que se rompe con sobrecarga, **fail-closed**.

**6.4**
```lua
-- KEYS[1] = clave del cliente (ej. "rl:user:123:minuto")
-- ARGV[1] = TTL en segundos · ARGV[2] = límite de requests por ventana
local c = redis.call('INCR', KEYS[1])
if c == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[1])   -- primera request de la ventana: fija el vencimiento
end
if c > tonumber(ARGV[2]) then
  return 0   -- se pasó del límite → el caller responde 429
end
return 1     -- permitida
-- Atómico porque Redis corre el script entero sin intercalar otros comandos (single-thread).
-- El INCR y el EXPIRE pasan juntos o no pasan: la clave nunca queda sin TTL por un crash entre ambos.
```

### Caso 7 — Web crawler / cola de trabajos
**7.1** Un **Bloom filter** responde "¿ya vi esta URL?" en O(1) con una fracción de la memoria de un set exacto (clave a miles de millones de URLs). Puede cometer **falsos positivos** (decir "probablemente sí la vi" cuando no) → ocasionalmente descarta una URL nueva, lo cual es aceptable (perdés una página). **Nunca** comete falsos negativos: si dice "seguro que no la vi", es cierto — así que jamás recrawlea algo creyendo que es nuevo por error del filtro.

**7.2** Porque la *politeness* es **por dominio**: no podés mandar miles de req/s a un solo host (lo tumbás y te bloquean). Particionar la frontera por host te deja aplicar un **rate independiente por dominio** (respetando `robots.txt` y `crawl-delay`) y desacopla "tengo muchísimo trabajo en total" de "cuánto puedo golpear *este* sitio ahora". Un planificador reparte el trabajo entre hosts sin violar el límite de ninguno.

**7.3** Porque cada página extrae decenas o cientos de links nuevos → la frontera **crece más rápido de lo que se drena** (ramificación alta), y los *spider traps* (URLs infinitas generadas dinámicamente) la inundan. Se mantiene acotada con **backpressure** (limitar el tamaño de la frontera: frenar o despriorizar la extracción cuando se llena) y **dedup agresivo + normalización de URLs + límites de profundidad/por-host** (para no encolar variantes del mismo recurso ni caer en trampas).

### Caso 8 — Reservas / pagos
**8.1** Porque acá un dato desactualizado **vende de más**: si dos lecturas ven el asiento como libre, los dos lo compran y uno se queda sin lugar (oversell), o se cobra dos veces. En el timeline (caso 1) ver un post unos segundos tarde no rompe nada — la consistencia eventual es aceptable. En inventario escaso, no: la corrección del estado "¿está libre?" es absoluta, así que necesitás consistencia **fuerte** (una base transaccional, no un store eventual). Es el trade-off CAP elegido al revés que el feed: acá **CP** (correctitud), allá **AP** (disponibilidad).

**8.2** Porque pagar tarda (cientos de ms a segundos) y puede fallar; si tenés la transacción abierta esperando al proveedor, mantenés los **locks tomados** todo ese tiempo → matás la concurrencia (todos los demás esperan ese asiento bloqueado) y arriesgás timeouts de la base. El patrón de **tres fases**: (1) **hold** atómico y cortísimo (reservás con un TTL, transacción que dura milisegundos), (2) **cobrás fuera de la transacción** con una idempotency key (un reintento no duplica el cargo), (3) **confirmás** (de `held` a `booked`). Si el pago falla o el hold expira, se **libera** el asiento. Coordinado entre inventario y pagos, es una **saga** con compensación.

**8.3** **Lock pesimista** (`SELECT ... FOR UPDATE`): correcto y simple, pero bajo contención extrema los intentos se encolan sobre la fila bloqueada (contención, posibles deadlocks). **Lock optimista** (`WHERE version = leída`): sin bloqueo, pero en alta contención casi todos pierden la carrera y reintentan mucho. **UPDATE condicional** (`SET estado='held' WHERE estado='available'`): un solo statement atómico, el motor serializa, el que gana afecta 1 fila y el resto 0 — sin locks explícitos ni reintentos. Para "tomar un recurso único" elijo el **UPDATE condicional**: es el más simple y el que mejor aguanta la contención (no hay ventana entre leer y escribir). Chequeo las filas afectadas para saber si gané.

**8.4**
```ts
interface SeatRepo {
  marcarHeldSiLibre(seatId: string, userId: string): Promise<number>;
}

async function reservarAsiento(
  seatId: string,
  userId: string,
  repo: SeatRepo,
): Promise<boolean> {
  // El UPDATE condicional (estado='held' WHERE estado='available') es atómico:
  // afecta 1 fila si el asiento seguía libre, 0 si alguien lo tomó antes.
  const filasAfectadas = await repo.marcarHeldSiLibre(seatId, userId);
  return filasAfectadas === 1;   // true = lo reservaste; false = 409 Conflict, ya estaba tomado
}
// No hay check-then-act: no leemos y después escribimos (esa ventana causaría oversell).
// La condición y la escritura son UN solo statement que la base serializa.
```

**8.5** Cuando el **pico de admisión** es tan grande que aun con reservas atómicas correctas la base se satura: cientos de miles de personas golpeando el inventario en el mismo segundo no causan oversell (el UPDATE condicional lo evita), pero **sí saturan la base de datos** con contención sobre las mismas filas y agotan conexiones — el sistema se cae aunque sea "correcto". La sala de espera resuelve un problema que el lock no toca: **el lock garantiza corrección bajo contención, pero no reduce la contención**. La sala admite usuarios al inventario en **tandas controladas** (throttling de admisión), convirtiendo un pico imposible en un caudal manejable. Es control de flujo / backpressure (ver [Concurrencia](concurrencia.md)) en la puerta de entrada, no en la base.

### Caso 9 — Uber / ride-sharing
**9.1** Porque un índice B-tree es 1-D: un índice compuesto `(lat, lng)` filtra por `lat` y después **escanea todas** las longitudes de esa franja (o al revés) — buscás una *banda*, no una caja, y con millones de puntos que se mueven eso es lento. Un **geohash** mapea 2-D → 1-D: puntos cercanos comparten un **prefijo** de string, así "cerca de mí" se convierte en una búsqueda por prefijo/rango sobre **una** dimensión, que sí es indexable (un sorted set de Redis, `GEOSEARCH`, o un índice común). Convertís un problema 2-D en uno 1-D que la infraestructura ya sabe resolver.

**9.2** Porque las celdas tienen **bordes duros**: un conductor a 50 m tuyo pero del otro lado de la línea de la celda cae en un geohash distinto, así que mirar solo tu celda lo deja afuera (y te da la falsa sensación de que no hay nadie cerca). La solución: buscar en **tu celda + las 8 vecinas** (grilla 3×3) y después **refinar por distancia real** sobre ese conjunto. Elegís la precisión de la celda de modo que su radio sea ≥ tu radio de búsqueda, así las 8 vecinas alcanzan para cubrir el círculo.

**9.3** Reusás el **claim atómico del caso 8** (el UPDATE condicional / lock para "tomar un recurso único"): asignar un conductor es exactamente "tomar un asiento". Lo marcás atómicamente — `UPDATE drivers SET estado='asignado' WHERE driverId=$1 AND estado='disponible'` — y mirás las filas afectadas: 1 = lo conseguiste, 0 = otro pasajero te ganó y pasás al siguiente candidato. Mismo patrón de "no doble asignación" que el "no oversell" del inventario: la atomicidad del instante evita que dos requests simultáneas se queden con el mismo recurso.

**9.4**
```ts
interface Punto { lat: number; lng: number }
interface Conductor { driverId: string; pos: Punto }

function masCercanos(
  origen: Punto,
  candidatos: Conductor[],
  k: number,
  distanciaKm: (a: Punto, b: Punto) => number,
): Conductor[] {
  return [...candidatos]
    .sort((a, b) => distanciaKm(origen, a.pos) - distanciaKm(origen, b.pos))
    .slice(0, k);
}
// La celda de geohash hace el trabajo barato (broad-phase): te da los CANDIDATOS por prefijo.
// El refinamiento por distancia real (narrow-phase) corre solo sobre ese conjunto chico,
// nunca sobre los millones de conductores. Es el mismo "filtro grueso → filtro fino" de las
// colisiones en geometría: primero descartás por celda, después medís exacto.
```

**9.5** La distancia en línea recta (haversine) es barata y no depende de nada externo, pero ignora la red de calles: el más cercano "a vuelo de pájaro" puede estar a 10 minutos por una autopista, en sentido contrario, o del otro lado de un río sin puente. Rankear por **ETA real** (tiempo de llegada considerando calles, sentido y tráfico) da mejores matches, pero introduce una **dependencia nueva**: un servicio de **routing/ETA** (mapas + tráfico en vivo) que hay que llamar por candidato → más latencia y costo por request. El patrón que usa la industria es el mismo **dos fases** del 9.4: broad-phase con geohash + distancia recta para sacar un puñado de candidatos baratos, y narrow-phase llamando al servicio de ETA **solo sobre esos pocos**, no sobre todos los conductores. El filtro grueso (geométrico) descarta; el filtro fino (ETA, caro) decide.

### Caso 10 — Ad-click aggregator / métricas en tiempo real
**10.1** Porque contar los eventos crudos en cada lectura (`COUNT(*)` sobre miles de millones de filas por cada refresh del dashboard) es inviable a este volumen. **Pre-agregar** en el stream —sumar clicks por `(adId, ventana de 1 min)` a medida que llegan— hace que la respuesta **ya exista**: leer "clicks entre las 10:00 y 10:01" es O(1) sobre una fila agregada, no un scan. Pagás el cómputo una vez, al ingerir, en vez de repetirlo en cada lectura.

**10.2** Porque **event-time** (cuándo ocurrió el click) es fijo: el mismo evento cae siempre en la misma ventana, así que reprocesar el log da el **mismo resultado** (reproducible). Con **processing-time** (cuándo lo procesaste), reprocesar otro día usa otro reloj → los eventos caen en **otras ventanas** → otro total para los mismos datos. Para la facturación eso es fatal: el número con el que cobrás dependería de *cuándo* corriste el job, no de los clicks reales, y un evento tardío de las 10:00 que llega a las 10:05 se cobraría en la ventana equivocada (o no se cobraría).

**10.3** (1) **Dedup por `eventId`** (idempotency key del evento), **antes de sumar**: un reproceso encuentra la key ya vista y no la vuelve a contar — defiende contra el duplicado en el procesador. (2) **Checkpointing** del estado del agregador (estado de las ventanas + offset guardados en el mismo punto): al reiniciar tras una caída, reprocesa desde el offset sin duplicar lo ya reflejado ni perder lo posterior — defiende el estado interno. (3) **Sink idempotente**: el store de agregados se escribe con *upsert* por `(adId, window_start)`, de modo que re-emitir una ventana (por allowed lateness o reproceso) **sobrescribe** en vez de duplicar — defiende el efecto externo, que el checkpoint no cubre. Y para no contar de **menos**: watermark + allowed lateness para no descartar los tardíos.

**10.4** **Lambda** corre dos caminos: el *speed layer* (stream) da el número en vivo, aproximado y rápido para el dashboard; un *batch layer* recalcula periódicamente la **verdad exacta** sobre todos los datos para facturar; el *serving layer* sirve el speed para lo reciente y el batch para lo consolidado. **Kappa** usa un solo pipeline de stream por event-time: el dashboard lee el resultado en vivo, y para la verdad de facturación (o para corregir un bug) **reprocesa el log** desde el offset afectado con el job corregido y hace el switch. Lambda paga con dos bases de código que deben coincidir; Kappa, con exigir un log reproducible y procesamiento por event-time.

**10.5**
```ts
interface ClickEvent {
  adId: string;
  eventId: string;
  eventTimeMs: number;
}
interface VentanaAgg {
  adId: string;
  windowStartMs: number;
  clicks: number;
}

const VENTANA_MS = 60_000; // 1 minuto

function agregarPorVentana(eventos: ClickEvent[]): VentanaAgg[] {
  const vistos = new Set<string>();             // dedup por eventId (entrega at-least-once)
  const conteo = new Map<string, VentanaAgg>();  // key = `${adId}:${windowStartMs}`

  for (const ev of eventos) {
    if (vistos.has(ev.eventId)) continue;        // ya contado → un reproceso no duplica
    vistos.add(ev.eventId);

    // Ventana tumbling por EVENT-TIME: el inicio del minuto que contiene al evento.
    const windowStartMs = Math.floor(ev.eventTimeMs / VENTANA_MS) * VENTANA_MS;
    const key = `${ev.adId}:${windowStartMs}`;

    const actual = conteo.get(key);              // Map.get → VentanaAgg | undefined en --strict
    if (actual === undefined) {
      conteo.set(key, { adId: ev.adId, windowStartMs, clicks: 1 });
    } else {
      actual.clicks += 1;
    }
  }
  return [...conteo.values()];
}
// La ventana sale del event-time del evento, no del reloj → reprocesar da el mismo resultado.
// OJO en producción: este `vistos` crece sin límite. El dedup real es ACOTADO —
// por ventana (se descarta junto con el estado de la ventana al cerrarla) o con TTL—,
// porque no podés guardar todos los eventId del historial en memoria.
```

### Caso 11 — Google Docs / edición colaborativa
**11.1** Porque guardar solo "el texto final" pierde la información necesaria para **mezclar ediciones concurrentes**: si dos clientes parten de la misma versión y cada uno guarda su texto entero, uno pisa al otro (*last-write-wins* → se pierden cambios). Modelar el doc como una **secuencia de ops** (insertar/borrar) lo convierte en *event sourcing* y habilita tres cosas: (1) **transformar/mergear** ops concurrentes (OT/CRDT operan sobre ops, no sobre el texto plano); (2) **sincronizar por delta** al reconectar (mandás solo las ops que faltan, no el documento entero); (3) **historial y undo** (cada op es un paso reversible). El texto actual se reconstruye aplicando el log; los snapshots evitan replay-earlo desde cero.

**11.2** Divergen porque **insertar no es conmutativo respecto de la posición**: ambos parten de `""`, Ana aplica `ins(0,"X")` y queda `"X"`, después le llega `ins(0,"Y")` y queda `"YX"`; Bruno hace lo simétrico y queda `"XY"`. Mismas dos ops, distinto orden de llegada → estados distintos que **nunca convergen**. **OT** lo resuelve **transformando** la op entrante contra las concurrentes ya aplicadas: cuando a Ana le llega `ins(0,"Y")` y ella ya aplicó `ins(0,"X")`, la transforma según quién gana el empate de posición; ambos clientes, transformando con el **mismo desempate**, llegan al mismo texto. **CRDT** lo resuelve dándole a cada carácter un **id único con orden total**: `"X"` y `"Y"` tienen ids fijos, así que su orden relativo es el mismo en toda réplica sin importar cuándo llegó cada uno → convergen por construcción, sin transformar.

**11.3** **OT** gana **compacidad** (el estado es texto plano, sin metadata extra) y encaja natural con un server central autoritativo; **paga** con la dificultad de implementar la transformación correctamente (cubrir todos los pares de ops y garantizar TP1 y sobre todo **TP2**) y con que en la práctica **necesita** ese punto central que ordene. **CRDT** gana **convergencia garantizada por construcción** (sin coordinación central, sirve P2P y *offline-first*); **paga** con **metadata por carácter** (cada uno arrastra su id → más memoria/almacenamiento) y con los **tombstones** de los borrados. Elegís **OT** si ya tenés server central y querés estado compacto (Google Docs); **CRDT** si querés merge sin coordinación, local-first o P2P (**Yjs, Automerge**). Ambos dan *strong eventual consistency*. (Y existe la tercera vía: server-autoritativo con *last-write-wins* por propiedad, como Figma, cuando el dato no es texto y no hace falta preservar intención carácter por carácter.)

**11.4** Porque si cada tecla esperara el round-trip al server antes de mostrarse, escribir se sentiría **laggy** (cientos de ms de lag por carácter es inusable). Con **UI optimista** aplicás tu edición **localmente al instante** y la mandás en paralelo; asumís que va a entrar. Lo que hay que **reconciliar** después: cuando vuelve la confirmación, el server pudo haber **transformado** tu op contra ediciones concurrentes de otros (o intercalado las de ellos antes que la tuya), así que tu estado local se **reajusta** al estado canónico — y, clave, **tu cursor y los de los demás se transforman** para no saltar cuando entró texto antes de su posición. El usuario tipea fluido; la convergencia se resuelve por debajo.

**11.5**
```ts
interface InsertOp {
  pos: number;
  texto: string;
  siteId: number;
}

function transformInsert(op: InsertOp, contra: InsertOp): InsertOp {
  // `contra` (inserción concurrente) ya se aplicó; ajustamos `op` para preservar su intención.
  // Si `contra` quedó estrictamente antes, o en la MISMA pos pero gana el desempate por siteId,
  // entonces `op` se corre a la derecha por el largo de `contra`.
  const contraVaPrimero =
    contra.pos < op.pos || (contra.pos === op.pos && contra.siteId < op.siteId);

  if (contraVaPrimero) {
    return { ...op, pos: op.pos + contra.texto.length };
  }
  return op; // `contra` va después: la posición de `op` no cambia.
}
// El desempate por siteId es lo que SATISFACE la propiedad TP1 de OT: AMBOS clientes
// resuelven el empate de la misma forma, así que transforman igual y terminan con el
// mismo texto. Un `<=` ingenuo sin desempate divergiría cuando dos inserciones caen
// exactamente en la misma posición.
```

**11.6** Porque `transformInsert` solo mueve **un punto** (una posición), mientras que un borrado afecta un **rango** `[inicio, fin)`, así que transformarlo es ajustar **longitudes**, no correr un índice. Contra una **inserción** concurrente: si el insert cae antes del rango, lo corro; si cae **dentro** del rango, el rango tiene que **crecer** para abarcar el texto recién insertado (o decidir explícitamente no borrarlo). Contra **otro borrado**: si los rangos **se solapan**, no puedo "borrar dos veces" la intersección que el otro ya borró → tengo que restar esa parte, y el resultado puede quedar **vacío** (la op se vuelve un *no-op*) o partirse en dos. Ese manejo de rangos solapados y longitudes variables es lo que multiplica los casos a cubrir (insert-insert, insert-delete, delete-insert, delete-delete) y lo que hace la transformación de OT "notoriamente difícil"; el insert-insert del ejercicio anterior es solo la **esquina fácil**.

### Caso 12 — Caché distribuida
**12.1** Porque `hash(key) % N` ata el destino de **cada** key al valor de `N`: si `N` cambia (agregás un nodo o se cae uno), el resto de la división cambia para **casi todas** las claves → casi todo el keyspace remapea a otro nodo de golpe. En un caché eso significa un *miss* masivo simultáneo y una avalancha de requests a la DB. El **consistent hashing** mapea claves y nodos a un **anillo**: cada key va al primer nodo girando desde su posición, así que agregar/quitar un nodo solo **reasigna las claves de ese arco** (≈ `1/N` del total), no todas. El cambio de topología pasa de "remapear todo" a "remapear una fracción".

**12.2** (1) **Balance de carga:** con un solo punto por nodo y pocos nodos, los arcos del anillo quedan de tamaños muy distintos → un nodo se come una porción enorme y otro casi nada. Muchos puntos por nodo (vnodes) **promedian** esos arcos y emparejan la distribución. (2) **Redistribución al fallar:** con un punto por nodo, cuando un nodo cae **toda** su porción se va a su único sucesor, que se sobrecarga; con vnodes, sus muchas porciones están intercaladas con las de todos, así que la carga del caído se **reparte entre varios** nodos. (El balance real depende de tener **suficientes** vnodes y un hash con buena dispersión — con un hash pobre, ni los vnodes salvan la distribución.)

**12.3** Lo dispara una **key caliente que vence (o se invalida) en seco**: en ese instante miles de requests concurrentes ven el *miss* a la vez y todas recomputan/pegan a la DB en paralelo, fundiéndola. A escala es peor porque el TTL fijo hace que expire **para todos simultáneamente**. Dos defensas (de tres posibles): **(a) single-flight / lock por key** — solo el primero que ve el miss recomputa y los demás **esperan** su resultado, colapsando N recomputos en uno; **(b) stale-while-revalidate** — se sirve el valor **viejo** mientras **uno** lo refresca en background, así nadie ve un miss. (La tercera: *early probabilistic expiration*, recomputar antes del TTL con probabilidad creciente; y `jitter` en los TTL para desincronizar.)

**12.4** **Cache-aside** (lazy): la app lee del caché, en *miss* va a la DB y puebla el caché; en *write* invalida/actualiza la entrada. Garantiza que solo cacheás lo que se usa; arriesga una **ventana de inconsistencia** entre el write a la DB y la invalidación (y un miss inicial por entrada). **Write-through:** cada write va al caché **y** a la DB en la misma operación → caché y DB siempre consistentes; arriesga **latencia** (todo write paga la DB) y cachear cosas que nunca se leen. **Write-back** (write-behind): el write va al caché y se baja a la DB **después**, en batch → rapidísimo en escritura; arriesga **perder writes** si el nodo muere antes de persistir, y deja la DB temporalmente atrasada. En los tres, el **TTL** acota cuánto puede vivir una inconsistencia.

**12.5**
```ts
function hash(s: string): number {
  let h = 0x811c9dc5;
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i);
    h = Math.imul(h, 0x01000193);
  }
  return h >>> 0;
}

const VNODES = 150; // puntos por nodo físico; más vnodes ⇒ distribución más pareja

interface Punto {
  hash: number;
  nodo: string;
}

class ConsistentHashRing {
  private anillo: Punto[] = []; // siempre ordenado por hash ascendente

  agregarNodo(nodo: string): void {
    for (let i = 0; i < VNODES; i++) {
      // La parte variable (i) va PRIMERO: con FNV, anteponer el índice dispersa
      // mucho mejor los vnodes de un mismo nodo que `${nodo}#${i}`.
      this.anillo.push({ hash: hash(`${i}@${nodo}`), nodo });
    }
    this.anillo.sort((a, b) => a.hash - b.hash);
  }

  quitarNodo(nodo: string): void {
    this.anillo = this.anillo.filter((p) => p.nodo !== nodo);
  }

  nodoPara(key: string): string | undefined {
    if (this.anillo.length === 0) return undefined;
    const h = hash(key);
    // Primer punto del anillo con hash >= h; si ninguno, se "envuelve" al primero
    // (el anillo es circular). En producción esto es una búsqueda binaria sobre el array.
    const punto = this.anillo.find((p) => p.hash >= h) ?? this.anillo[0];
    return punto.nodo;
  }
}
// Verificado: con 3 nodos reparte ~30/33/36% de 10.000 claves, y al quitar un nodo
// SOLO se remapean las claves que estaban en él (~1/N), no todo el keyspace.
```

**12.6** Sus claves **se pierden** (es caché: la DB sigue siendo la fuente de verdad), y todas las requests de esas claves pasan a *miss* y caen a la DB de golpe — un **mini-stampede** localizado en la porción del nodo muerto. Con **un punto por nodo**, esa porción entera se la come el **único sucesor** en el anillo, que se sobrecarga (y puede caer en cascada, arrastrando al siguiente). Con **vnodes**, las muchas porciones del nodo muerto están **intercaladas** con las de todos los demás, así que su carga se **reparte entre varios** sucesores y el golpe se amortigua. **Replicar** cada partición en ≥2 nodos se justifica cuando ni ese mini-stampede es tolerable (la DB no aguanta perder esa porción de caché de golpe, o el dato es caro de recomputar): la réplica sigue sirviendo las claves del nodo caído. El costo es **consistencia**: con réplica **síncrona** no perdés writes confirmados pero cada write paga la latencia de copiar; con réplica **asíncrona** (lo común en cachés) ganás velocidad pero queda una ventana donde un fallo pierde los writes más recientes — el mismo trade-off async/sync de [Replicación](replicacion.md).

### Caso 13 — YouTube / Netflix: streaming de video
**13.1** Porque un único archivo monolítico obliga a todos a descargar **una sola calidad**: si es alta, el que tiene mala red no puede reproducir (buffering); si es baja, el que tiene fibra ve un video feo. Servir **varias renditions** (240p…4K) deja que cada cliente use la que su red aguanta, y **partir cada una en segmentos** de pocos segundos permite **cambiar de calidad sobre la marcha** (segmento a segmento) y empezar a reproducir sin bajar el archivo entero (arranque rápido + *seek* barato). El **manifiesto** (`.m3u8` de HLS / MPD de DASH) es el índice que el reproductor lee primero: lista qué renditions hay y, para cada una, la secuencia de segmentos con sus URLs — el mapa que el cliente usa para pedir lo que necesita.

**13.2** Porque transcodificar un video a muchas resoluciones/códecs/bitrates es trabajo **pesado, independiente y paralelizable**: se parte el original en trozos (por GOP/escenas) y cada (trozo × rendition) es un job que no depende de los demás → se reparten en una **flota de workers** que escala horizontalmente, y al final se ensambla. Reusa el patrón **cola de trabajos a escala** del **caso 7** (frontier/workers/reintentos): productores encolan jobs de transcoding, los workers los consumen, los fallidos van a DLQ. Y como tarda, es **asíncrono**: el video queda "procesando" y se avisa al terminar (las colas del caso 3).

**13.3** El **adaptive bitrate** parte cada calidad en segmentos cortos; el reproductor del cliente **mide su ancho de banda real** (cuánto tardó en bajar los últimos segmentos) y, antes de pedir el próximo, **elige la rendition** cuyo bitrate entra en esa banda: arranca bajo (para empezar ya), sube si sobra banda, baja si se congestiona. Vive en el **cliente** porque es **el único que conoce su situación real** momento a momento: su ancho de banda, el tamaño de su buffer, el tipo de pantalla, la CPU para decodificar. El server no puede saber que *a este* usuario se le saturó el WiFi hace 2 segundos; el cliente sí, y reacciona al instante sin un round-trip. El server solo publica las opciones (el manifiesto); el cliente decide.

**13.4** Porque el egress de video es **petabytes/día** y servirlo desde el origin es inviable por **ancho de banda** (no tenés esa capacidad de salida) y por **geografía** (la latencia a un usuario lejano mata el arranque). El CDN **replica los segmentos en edges cercanos** al usuario, absorbe la mayor parte del tráfico y solo va al origin en *miss*. Los segmentos son **ideales para cachear** porque son **inmutables e idénticos para todos**: el segmento 47 de la rendition 720p de un video es el mismo byte a byte para cualquiera que lo mire → un objeto cacheable con TTL larguísimo y altísimo *hit rate*. Un **feed personalizado**, en cambio, es distinto para cada usuario → no se puede cachear igual (cada respuesta es única). Contenido estático y compartido = el caso perfecto para un CDN.

**13.5** Por la **distribución de la demanda**. **Netflix** tiene un catálogo **curado y acotado**, y cada título se ve **millones de veces** → pre-transcodificar todo (y hasta optimizar el bitrate por título, *per-title encoding*) se amortiza enseguida y da playback instantáneo en toda calidad; el costo de storage por rendition vale la pena porque el contenido se reusa muchísimo. **YouTube** recibe **UGC infinito** con una **cola larga** brutal: la enorme mayoría de los videos casi no se ven → pre-transcodificar cada uno en todas las calidades sería tirar storage. Conviene transcodificar **on-demand** (o solo en calidades básicas) y reservar el procesamiento caro para lo que **demuestra** demanda. La regla: pre-computás cuando el resultado se va a reusar mucho; lazy cuando la mayoría no se usa.

**13.6**
```ts
interface Rendition {
  altura: number;
  bitrateKbps: number;
}

function elegirRendition(
  anchoBandaKbps: number,
  renditions: Rendition[],
): Rendition | undefined {
  const presupuesto = anchoBandaKbps * 0.8; // factor de seguridad: no usar el 100% de la banda
  const ordenadas = [...renditions].sort((a, b) => a.bitrateKbps - b.bitrateKbps);
  // Arranco con la más baja (o undefined si la lista viene vacía). Tipar como
  // `Rendition | undefined` deja el código a salvo de noUncheckedIndexedAccess.
  let elegida: Rendition | undefined = ordenadas[0];
  for (const r of ordenadas) {
    if (r.bitrateKbps <= presupuesto) elegida = r; // la mejor que entra en el presupuesto
  }
  return elegida;
}
// El factor 0.8 deja headroom para que un bajón de banda no vacíe el buffer de inmediato.
// `undefined` SOLO si no hay renditions; si hay pero ninguna entra en la banda, se devuelve
// la más baja (degradar a una imagen fea es mejor que cortar la reproducción).
// Un ABR de producción además mira el nivel del buffer, no solo la banda (evita oscilar de calidad).
```

### Caso 14 — Payment ledger / billetera
**14.1** Porque una columna `balance` mutable **pierde la historia** y es frágil: solo te dice el "ahora", no **cómo** llegaste ahí (imposible de auditar), y cada movimiento es un `UPDATE` que compite con otros bajo concurrencia (carreras, y un bug puede dejar el saldo mal **sin rastro**). Modelar el balance como la **suma de asientos inmutables** hace que el historial **sea** la fuente de verdad: el saldo es una **derivación** reproducible (re-sumás y verificás), cada centavo tiene su asiento rastreable, y no hay updates en conflicto sobre una fila caliente. Por performance se **materializa** un saldo (snapshot), pero siempre podés reconstruirlo y reconciliarlo desde el log.

**14.2** La **doble entrada** registra cada movimiento como **≥2 asientos que suman cero**: un **débito** en una cuenta y un **crédito** igual en otra (la plata sale de un lado y entra en otro, nunca aparece ni desaparece). El invariante que te da gratis: **la suma de todos los asientos del sistema es siempre 0** (o igual al dinero externo que entró/salió). Eso vuelve al sistema **auto-verificable** — en cualquier momento sumás todo y, si no da cero, **sabés que hay un bug** y dónde mirar. Es una red de seguridad estructural: los errores de plata se **delatan solos** en vez de esconderse en una columna.

**14.3** Porque la inmutabilidad da dos cosas que el dinero exige: **auditabilidad** (el pasado nunca cambia → un auditor puede confiar en que el historial es la verdad, y podés reconstruir el estado a cualquier fecha) y **ausencia de carreras** (no hay `UPDATE` sobre un balance que dos transacciones se peleen — solo `INSERT` de asientos nuevos). Si cometés un error (un cargo equivocado), **no editás ni borrás** el asiento viejo: emitís una **transacción compensatoria** — asientos nuevos que **revierten** el efecto (un crédito que anula el débito errado) y, si corresponde, los asientos correctos. El error y su corrección **quedan ambos en el registro**, que es justo lo que una auditoría necesita ver.

**14.4** Porque los asientos de una transacción son un **paquete indivisible**: si el débito de A entra pero el crédito a B no (crash, timeout), A perdió plata que nadie recibió y el sistema **deja de cuadrar** (la suma ya no es cero). Escribirlos en **una sola transacción de DB** garantiza "todos o ninguno" (atomicidad — la consistencia fuerte del caso 8). Los dos mecanismos reusados: para **no doble cargo**, la **idempotency key store-and-return del caso 8** (un reintento con la misma key devuelve el resultado ya creado, no cobra de nuevo); para **no sobregirar**, el **check-then-act atómico del caso 8** (verificar el saldo y asentar en una operación serializada por cuenta, para que dos débitos concurrentes no dejen la cuenta en negativo). Concretamente: `SELECT ... FOR UPDATE` sobre la **fila de la cuenta** (su balance materializado, no re-sumando el log en la ruta de escritura), y si una transferencia lockea dos cuentas, **ordenás los locks por `accountId`** para evitar *deadlocks*.

**14.5**
```ts
interface Entry {
  accountId: string;
  amount: number; // en centavos (entero); negativo = débito, positivo = crédito
}

function asientosTransferencia(from: string, to: string, amountCentavos: number): Entry[] {
  // Dinero = enteros en la unidad mínima (centavos). Nunca float: 0.1 + 0.2 !== 0.3.
  if (!Number.isInteger(amountCentavos) || amountCentavos <= 0) {
    throw new Error("El monto debe ser un entero positivo de centavos");
  }
  const asientos: Entry[] = [
    { accountId: from, amount: -amountCentavos }, // débito: sale de `from`
    { accountId: to, amount: amountCentavos },    // crédito: entra a `to`
  ];
  // Invariante de doble entrada: la suma de los asientos es 0 (la plata solo se movió).
  const suma = asientos.reduce((acc, e) => acc + e.amount, 0);
  if (suma !== 0) throw new Error("Los asientos no cuadran (suma ≠ 0)");
  return asientos;
}
// Estos dos asientos se persisten en UNA transacción de DB (atómica, todos o ninguno),
// idealmente junto con la idempotency key, para que un reintento no genere otra transferencia.
```

### Caso 15 — Order matching engine
**15.1** La **prioridad precio-tiempo** es la regla de fairness del matching: se ejecuta primero la orden con **mejor precio** (el bid más alto contra el ask más bajo) y, **a igual precio**, la que **llegó antes**. Se refleja en la estructura del book en **dos niveles**: un mapa **ordenado por precio** (un árbol/skip-list, o arrays indexados por precio para baja latencia) te da el "mejor precio primero" en O(1)/O(log n); y dentro de **cada nivel de precio**, una **cola FIFO** te da el "primero en llegar, primero en ejecutar". Casar = tomar siempre el mejor nivel del lado contrario y, dentro de él, la **cabeza de la cola**. Esa doble estructura *es* la prioridad precio-tiempo materializada en datos.

**15.2** Porque el order book es **estado compartido mutable** y el resultado **depende del orden exacto** de procesamiento (la time priority). Si dos hilos tocaran el mismo book, necesitarías **locks** —que serializan igual pero con contención y latencia impredecible— y perderías el **determinismo** (el orden de ejecución dejaría de ser reproducible, y sin reproducibilidad no hay replay ni auditoría). Un **solo hilo** procesando en orden de secuencia es **más rápido**: cero locks, todo el estado caliente en cache de CPU, predicción de saltos estable (el patrón LMAX Disruptor). Donde **sí** paralelizás es **entre símbolos**: cada símbolo tiene su propio book y su propio engine/core, y como un trade nunca cruza símbolos, no hay nada que coordinar. El techo de **un** símbolo es **un core** — un límite que en la práctica casi nunca se toca.

**15.3** El recovery se apoya en **event sourcing**: el **secuenciador** persiste **toda entrada** (cada orden y cada cancelación, con su número de secuencia monótono) a un **log append-only** *antes* de que el engine la procese, y periódicamente se toma un **snapshot** del book en memoria. Para recuperar tras un crash: cargás el **último snapshot** y **reproducís** el log desde la secuencia de ese snapshot en adelante. El **determinismo** es lo que lo vuelve posible: como el engine no usa nada no determinista (sin reloj de pared —los timestamps viajan en el log—, sin random, sin orden de hashmap, sin I/O bloqueante), **reproducir las mismas secuencias en el mismo orden produce exactamente el mismo book**. Si el replay no fuera determinista, reconstruirías *un* book, no *el* book —y en un exchange eso es inaceptable. El determinismo acá **no es elegancia, es la condición** que hace que el replay (y el hot standby) reconstruyan el book *idéntico*: si se cuela algo no determinista en el hot path (el orden de iteración de un `Map`, un redondeo de `float`, un `Date.now()`), los books **divergen en silencio** —sin ninguna excepción que te avise— y recién lo descubrís cuando una reconciliación no cuadra. Ese silencio es justo lo que lo vuelve peligroso. El mismo determinismo permite además un **hot standby** que consume el mismo log y mantiene un book idéntico para failover instantáneo.

**15.4** Porque publicar cada trade y cada cambio del book a **miles de participantes** es trabajo de fan-out que, hecho **en el hot path**, frenaría el matching —y el matching es justo lo que tiene que ser ultrarrápido y de jitter acotado. El engine se limita a **emitir** un stream de eventos; un **publisher separado** hace la difusión, idealmente por **multicast UDP** (un mensaje, N receptores) en vez de N envíos individuales. Es el **fan-out de lectura del caso 1** (feed): separar la **generación** del evento de su **distribución** a muchos, para que el camino crítico de escritura no pague el costo de los lectores. Acá llevado al extremo de latencia.

**15.5**
```ts
interface Order {
  id: string;
  qty: number;   // cantidad restante (entero, unidad mínima)
  price: number; // en ticks (entero); nunca float
}
interface Trade {
  buyOrderId: string;
  sellOrderId: string;
  qty: number;
  price: number;
}

// `asks` viene ordenado por precio ascendente; dentro de cada precio, la primera del
// array es la más vieja (FIFO). La función MUTA `asks`: descuenta cantidades y saca del
// libro los asks que se llenan del todo. Devuelve los trades y el remanente sin casar.
function matchearCompra(
  entrante: Order,
  asks: Order[],
): { trades: Trade[]; remanente: number } {
  const trades: Trade[] = [];
  let restante = entrante.qty;

  // Mientras me quede cantidad Y exista un mejor ask que CRUCE mi precio de compra.
  while (restante > 0 && asks.length > 0) {
    const mejorAsk = asks[0]; // con noUncheckedIndexedAccess es Order | undefined
    // El precio de compra debe ser >= la mejor venta; si no, no hay cruce → corto.
    if (mejorAsk === undefined || mejorAsk.price > entrante.price) break;

    const qty = Math.min(restante, mejorAsk.qty); // fill parcial: lo que alcance
    trades.push({
      buyOrderId: entrante.id,
      sellOrderId: mejorAsk.id,
      qty,
      price: mejorAsk.price, // se ejecuta al precio de la orden que YA estaba en el book
    });

    restante -= qty;
    mejorAsk.qty -= qty;
    if (mejorAsk.qty === 0) asks.shift(); // se llenó del todo → sale del libro
  }

  return { trades, remanente: restante };
}
// Notas: (1) el trade se ejecuta al precio del ask EN REPOSO (el del maker) — prioridad
// PRECIO; al tomar siempre asks[0] respeto la prioridad TIEMPO (FIFO). Hay *price improvement*
// para el agresor SOLO cuando su límite cruza POR ENCIMA del ask en reposo (compra a 50 contra
// ask a 49 → fill a 49, mejora de 1); si entran al mismo precio, no hay mejora.
// (2) Si queda `remanente > 0`: una orden LÍMITE lo deja en el book como nuevo bid; una IOC
// (Immediate-Or-Cancel) ejecuta lo que pueda y DESCARTA el resto; una FOK (Fill-Or-Kill) es
// todo-o-nada → si no se llena completa, se rechaza entera (ni un fill parcial).
// (3) La función MUTA el book in-place (`mejorAsk.qty -= qty`, `asks.shift()`): refleja que el
// matching CONSUME liquidez del libro real. (4) Enteros en ticks: nunca float para precios.
```

**15.6** Porque el `float` (IEEE-754) **no puede representar exactamente** la mayoría de los decimales: el clásico `0.1 + 0.2 === 0.30000000000000004`, y en JS `number` *es* un float de 64 bits (enteros exactos solo hasta 2⁵³). En un matching engine eso es catastrófico por dos razones encadenadas: **(1) correctitud** — comparar precios para decidir si dos órdenes cruzan (`bid >= ask`) con floats da resultados erráticos (dos precios "iguales" que no lo son por un epsilon), sumar fills acumula error de redondeo, y terminás ejecutando trades a precios mal calculados o casando órdenes que no debían cruzar; **(2) determinismo** — el redondeo de coma flotante puede depender del orden de las operaciones y de la plataforma, así que rompe la reconstrucción exacta del book por replay (el riesgo de divergencia silenciosa del 15.3). La solución es la misma del ledger del caso 14: **enteros en la unidad mínima** (precios en *ticks*, montos en centavos) o un decimal de precisión fija de la DB (`NUMERIC`); toda la aritmética del engine es entera, exacta y reproducible.

### Caso 16 — Sistema RAG / asistente LLM
**16.1** RAG resuelve dos límites del LLM: su conocimiento **está congelado en el *cutoff*** del entrenamiento (no sabe nada posterior ni nada privado tuyo) y **alucina con seguridad** cuando no sabe. Recuperando evidencia de **tu** corpus e inyectándola en el prompt, la respuesta se **fundamenta en datos actuales, propios y citables**, y podés instruir al modelo a decir "no sé" cuando la evidencia no alcanza. La diferencia con **fine-tuning**: fine-tuning ajusta los **pesos** del modelo para enseñarle un **estilo, formato o tarea** (cómo responder), y es caro de re-hacer cuando los hechos cambian; RAG no toca el modelo, le da **hechos frescos en el contexto** (qué responder) y se actualiza re-ingestando un documento. Regla: ¿conocimiento que cambia y hay que citar? → RAG. ¿Comportamiento/estilo consistente? → fine-tuning. Se combinan.

**16.2** Entrada = 6×500 + 1000 + 200 = **4200 tokens**; salida = **400 tokens**. Costo = 4200 × $5/1.000.000 + 400 × $25/1.000.000 = **$0.021 + $0.010 = ~$0.031 por consulta** (a 1M consultas/mes ≈ **~$31k/mes** solo en generación). Tres palancas: **(1) modelo más barato** para lo simple (a $1/$5 la misma consulta sale ~$0.0062, ~5× menos) con **routing** que manda lo difícil al modelo caro; **(2) prompt caching** del system prompt + instrucciones estables (la lectura cacheada cuesta ~0.1× → si el prefijo de ~1k se repite, casi desaparece de la cuenta de entrada); **(3) Batch API** (~50%) para todo lo que no sea interactivo (reindexar respuestas, evals, generación offline). El embedding de la query es ~1 vector → costo despreciable frente a la generación.

**16.3** **(1) Qué se rompe:** la pregunta cae fuera del corpus —o el retrieval trae chunks irrelevantes— y el LLM **inventa** una respuesta plausible y falsa, a veces citando una fuente que no existe. **(2) Por qué a esta escala/uso:** el modelo es no determinista y está entrenado para **sonar seguro**; sin evidencia buena, **rellena** en vez de abstenerse, y a volumen una fracción de respuestas falsas erosiona la confianza (y en dominios regulados es responsabilidad legal). **(3) Control de corto plazo:** prompt estricto ("respondé **solo** con el contexto; si no está, decí que no lo tenés"); **umbral de score** del retrieval (si el mejor chunk no supera el umbral → "no tengo información", no generes); exigir **citas verificables** que el usuario pueda abrir. **(4) Cambio de diseño:** retrieval **híbrido + reranking** para que la evidencia que llega sea buena (si entra basura, sale basura); un **LLM-juez de groundedness** que verifica que cada afirmación esté soportada por las fuentes —en el camino crítico para alto riesgo, o muestreado offline para no pagar latencia siempre—; **golden set + logging + feedback 👍/👎** para medir groundedness y detectar regresiones; **human-in-the-loop** donde el costo del error es alto.

**16.4** Porque **cada token del contexto se re-paga en cada turno** (costo lineal en el tamaño del corpus inyectado, multiplicado por la longitud de la conversación) y porque los modelos sufren *lost in the middle* —ignoran la información sepultada en el medio de un contexto enorme—, así que meter 100M tokens de corpus en la ventana es a la vez **carísimo y menos preciso** que recuperar los 6 chunks que importan. El retrieval es, en el fondo, **traer solo lo relevante** para no pagar ni diluir. **Sí saltearías el índice vectorial** cuando el corpus **entra cómodo en la ventana de contexto** (un manual, unos pocos PDFs, una política): metés los docs directo en el prompt —idealmente con **prompt caching** si la consulta se repite— y te ahorrás todo el pipeline de embeddings + HNSW + reranking. El índice vectorial recién se justifica cuando el corpus **no entra**, **cambia seguido** o necesitás **filtrar por permisos y citar**.

**16.5**
```ts
interface Chunk {
  id: string;
  documentId: string;
  texto: string;
  score: number;
}
interface Contexto {
  prompt: string;
  fuentes: string[]; // documentId de cada chunk incluido, para citar
}

// `chunks` viene ordenado por score descendente. `estimarTokens` cuenta tokens de un string
// (en producción: la API count_tokens del proveedor, NUNCA un tokenizer de otro modelo —
// undercuenta y rompe el presupuesto).
function armarContexto(
  pregunta: string,
  chunks: Chunk[],
  presupuestoTokens: number,
  estimarTokens: (s: string) => number,
): Contexto {
  const incluidos: Chunk[] = [];
  let usados = estimarTokens(pregunta);
  for (const c of chunks) {
    const costo = estimarTokens(c.texto);
    if (usados + costo > presupuestoTokens) break; // no entra: corto (los de mayor score ya entraron)
    incluidos.push(c);
    usados += costo;
  }
  const prompt = [
    "Respondé SOLO con el siguiente contexto. Si la respuesta no está, decí que no la tenés.",
    ...incluidos.map((c) => `[${c.documentId}] ${c.texto}`),
    `Pregunta: ${pregunta}`,
  ].join("\n\n");
  // Fuentes únicas, preservando el orden de aparición (mayor score primero).
  const fuentes = [...new Set(incluidos.map((c) => c.documentId))];
  return { prompt, fuentes };
}
// Notas: (1) asume `chunks` YA ordenado por score desc; corto con `break` en el PRIMER chunk que
// no entra en vez de saltearlo y seguir buscando uno más chico — así preservo la prioridad de
// score y no relleno el contexto con chunks de score bajo "porque caben" (eso mete ruido y
// agrava el "lost in the middle"). (2) En producción el presupuesto se reparte: ANTES del loop
// hay que restar el costo del system prompt de fundamentación + la instrucción de citar +
// el historial; acá modelamos solo los chunks. (3) Las `fuentes` son lo que se cita al usuario.
```

### Caso 17 — Distributed job scheduler
**17.1** Porque la pregunta que el scheduler hace en cada *tick* es siempre la misma —*"¿qué jobs tienen `next_run_at <= ahora`?"*— y un **índice sobre `next_run_at`** la convierte en un **range scan** (recorrer solo el tramo del índice cuyo vencimiento ya pasó) en vez de un **full scan** de los 100M registros. Sin el índice, encontrar los vencidos cuesta O(n) en cada tick y el scheduler colapsa a escala; con el índice, cuesta O(log n + k) donde k son los vencidos. Un job **recurrente** no es una entidad especial: se guarda la **expresión cron** y se modela como un **one-shot que se re-agenda a sí mismo** — al dispararlo, calculás el **próximo** `next_run_at` a partir del cron y actualizás el mismo registro. Así el motor de disparo **solo entiende de "`next_run_at` vencido"**; la recurrencia queda encapsulada en el paso de re-agendado, no en la consulta caliente.

**17.2** Se rompe porque millones de jobs comparten **exactamente** el mismo vencimiento (`0 0 * * *` = medianoche) → en un segundo el scheduler tiene que despachar 10⁶ jobs y los workers tienen que ejecutarlos, un pico de cinco órdenes de magnitud sobre el promedio que **satura el dispatch, la cola y los workers** a la vez. Tres palancas: **(1) jitter** — esparcir el disparo en una ventana de ±minutos (la mayoría de los jobs "de medianoche" toleran salir 00:03 en vez de 00:00; solo no jitterás los que tienen un instante duro real); **(2) dispatch a tasa acotada + cola** — el scheduler **encola a un ritmo techado** y la **cola absorbe el pico** (la frontera acotada del caso 7 + el desacople del caso 3), de modo que el pico se convierte en una **ráfaga que se drena en minutos** en vez de un golpe instantáneo; **(3) dimensionar los workers para el pico**, no para el promedio de 38/s (o autoescalar la flota de workers ante la profundidad de la cola). En una línea: es **backpressure** — nunca dispares más rápido de lo que aguas abajo puede tragar.

**17.3** **(1) Qué se rompe:** el scheduler selecciona un job vencido, lo **reclama** y lo **encola**, pero **crashea antes de marcarlo como disparado** en la DB; al recuperar, lo ve "pendiente" otra vez y lo **dispara de nuevo** → el ejecutor corre el job **dos veces** (un cobro mensual duplicado, dos notificaciones). **(2) Por qué a esta escala/uso:** con HA hay **varios nodos** scheduler mirando el mismo conjunto de jobs, y los crashes son rutina a escala; entre "encolé" y "marqué disparado" hay una **ventana sin atomicidad cross-sistema** (la cola y la tabla de jobs son recursos distintos), así que **exactly-once de disparo es imposible**. **(3) Control de corto plazo:** **claim atómico** (`FOR UPDATE SKIP LOCKED`) para que dos nodos no reclamen el mismo job simultáneamente; **lease con expiración** (`locked_until`) para que un job cuyo dueño murió no quede ni perdido ni disparándose para siempre — otro nodo lo re-reclama cuando el lease vence. **(4) Cambio de diseño:** asumir **at-least-once** y empujar la unicidad al **ejecutor idempotente** (`idempotencyKey` del caso 8, que deduplica el segundo disparo); marcar el disparo en la **misma transacción** que el claim cuando el ejecutor vive en la misma DB; **sharding por `jobId` + leader election** para que **un solo nodo** sea dueño de cada job y la pelea por el lock desaparezca.

**17.4** **Polling de un índice por tiempo** (cada tick, `SELECT ... WHERE next_run_at <= now() FOR UPDATE SKIP LOCKED`): es **simple y durable** —la DB es la fuente de verdad, un crash no pierde nada— y **escala a millones**; el costo es que la **granularidad de puntualidad = el intervalo de poll** (poll cada 1 s → ±1 s de jitter; bajar a 100 ms para más precisión carga más la DB). **Timer wheel en memoria** (una *timing wheel* que avanza tick a tick y dispara los timers de cada slot): da puntualidad de **milisegundos**, pero su estado vive en **RAM** → un crash lo borra, así que necesita **durabilidad y recovery aparte**. El **híbrido** que usaría: la **DB indexada es la verdad durable** y un **timer wheel en memoria carga solo la ventana próxima** (los jobs de los próximos N minutos, recargada periódicamente desde la DB). Así obtenés disparo sub-segundo de la rueda **sin perder durabilidad**: si el nodo muere, la ventana se reconstruye desde la DB; lo único en riesgo son los segundos en vuelo, que el lease + el re-claim recuperan.

**17.5**
```ts
interface Job {
  id: string;
  nextRunAt: number;          // epoch ms del próximo disparo
  lockedUntil: number | null; // lease: hasta cuándo está reclamado; null = libre
}

// `jobs` viene ordenado por nextRunAt ascendente (el orden del índice): los primeros del
// array son los MÁS vencidos. Reclama hasta `limite` jobs vencidos y libres, marcándolos
// in-place con un lease nuevo. Modela el `FOR UPDATE SKIP LOCKED` + lease del dispatcher.
function reclamarVencidos(
  jobs: Job[],
  ahora: number,
  leaseMs: number,
  limite: number,
): Job[] {
  const reclamados: Job[] = [];
  for (const job of jobs) {
    if (reclamados.length >= limite) break;   // dispatch a tasa acotada: paro al llegar al lote
    const vencido = job.nextRunAt <= ahora;
    const libre = job.lockedUntil === null || job.lockedUntil <= ahora; // null = libre; lease expirado = re-reclamable
    if (vencido && libre) {
      job.lockedUntil = ahora + leaseMs;       // claim: tomo el lease (el SKIP LOCKED en memoria)
      reclamados.push(job);
    }
  }
  return reclamados;
}
// Notas: (1) el `lease` (lockedUntil) es lo que da la garantía del 17.3: un job reclamado por un
// nodo que después muere SIN dispararlo vuelve a estar "libre" cuando `lockedUntil <= ahora`, y
// otro nodo lo re-reclama — ni se pierde ni se dispara dos veces a la vez. (2) Reclamar NO es
// disparar: el caller encola los reclamados y, recién tras confirmar el disparo, los marca/los
// re-agenda (si son cron, recalcula `nextRunAt`); si crashea antes, el lease expira y se reintenta
// → at-least-once + idempotencia en el ejecutor (caso 8). (3) `limite` acota el lote por tick
// (backpressure del 17.2): como `jobs` está ordenado, tomar los primeros `limite` libres = atender
// primero a los más vencidos. (4) Muta los jobs in-place igual que un claim real marca la fila en
// la DB. (5) el `for` recorre `jobs` hasta llenar el lote; en SQL el índice sobre `next_run_at` +
// `LIMIT N` evita tocar los no vencidos — el ejercicio en memoria no modela esa poda del índice.
// (6) esta selección+marcado es acá un `for` single-thread, pero en la DB es UNA transacción atómica
// por nodo (`FOR UPDATE SKIP LOCKED`): el array modela la LÓGICA, no la CONCURRENCIA que justifica el SKIP LOCKED.
```

**17.6** Las dos políticas son **catch-up** (al volver, disparás todos los vencimientos que se perdieron mientras estabas caído) y **skip-and-realign** (ignorás los atrasados y saltás directo al **próximo** vencimiento válido). **Catch-up** sirve cuando cada ejecución importa y no es reemplazable —un job de **facturación** o de **cierre contable** que se saltó hay que correrlo igual— pero es **peligroso**: si se acumularon cientos de disparos, al arrancar generás una **tormenta auto-infligida** (el mismo thundering herd del 17.2, ahora provocado por el propio recovery) → conviene combinarlo con jitter/rate-limit, o con un *cap* ("corré a lo sumo el último, no los 120 atrasados"). **Skip-and-realign** sirve cuando el job es **idempotente sobre el estado actual** y solo importa el "ahora" —un job que **recalcula un agregado** o **refresca un caché** no gana nada corriendo 120 veces seguidas, le alcanza con la próxima— y evita la tormenta por construcción. Hay una **tercera vía intermedia**: **coalescer** los disparos perdidos en **una sola corrida** al volver (ni los 120 ni cero: uno), lo que ofrecen varios schedulers modernos — ideal cuando querés que el job **corra** tras el outage pero **no N veces**. Quartz formaliza esto como *misfire policy*; lo entrevistable es **nombrar el dilema y justificar la elección según si cada corrida perdida tiene valor propio o no**, no que exista una respuesta única.

Estos 17 casos cubren los patrones que más caen: fan-out, tiempo real, async con colas, lectura-intensivo con precómputo, separación metadata/contenido, estado compartido atómico, procesamiento masivo, consistencia fuerte transaccional, indexación espacial, agregación en streaming por event-time, convergencia de ediciones concurrentes (OT/CRDT), particionado por consistent hashing, distribución de contenido por CDN, contabilidad de doble entrada, matching secuencial determinista, recuperación aumentada (RAG) y disparo programado a escala. Para seguir:

- **Resolvé en voz alta los que faltan**, reusando los mismos ladrillos: "diseñá un full-text search" (índice invertido + el caso 4 de precómputo — ya hay un módulo dedicado en [Búsqueda a escala](busqueda.md)). Lo que se evalúa es el recorrido, no la respuesta.
- **Conectá cada decisión con su implementación en el hub:** colas e idempotencia en [Event-driven](event-driven.md) y [Redis](redis.md), el contrato de cada API en [API design](api-design.md), tiempo real en [Tiempo real](tiempo-real.md), datos en [PostgreSQL](postgresql.md)/[NoSQL](nosql.md), control de flujo en [Concurrencia](concurrencia.md), y cómo operarlo en [Observabilidad](observabilidad.md).
- **Volvé al método** de [Diseño de sistemas](system-design.md) módulo 1 cada vez: requisitos → estimación → API/datos → alto nivel → profundizar donde duele → trade-offs. El método es el mismo para los 17 casos y para cualquier problema nuevo que te tiren.
