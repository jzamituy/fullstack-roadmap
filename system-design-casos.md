# Banco de casos de diseño de sistemas: 9 problemas resueltos de punta a punta

**Los casos clásicos de entrevista senior, resueltos con el método · feed, chat, notificaciones, typeahead, archivos, rate limiter, crawler, reservas/pagos, proximidad · el recorrido importa más que la respuesta · 2026**

> Cómo usar este banco: este módulo es la **práctica** de [Diseño de sistemas backend](system-design.md). Ahí aprendiste el método y el criterio; acá los aplicamos a los 9 problemas que más caen en entrevistas. Para cada caso, **tapá la solución y resolvelo vos primero en voz alta** siguiendo los 6 pasos; después contrastá. Lo que se evalúa no es si llegás a *mi* diseño, sino si recorrés bien: requisitos → estimación → API/datos → alto nivel → profundizar donde duele → trade-offs.

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

> **El patrón que se repite.** Vas a ver que los mismos ladrillos reaparecen en casi todos los casos: **fan-out** (¿escribo a muchos en la escritura o leo de muchos en la lectura?), **caché del camino caliente**, **idempotencia ante reintentos**, **colas para desacoplar y absorber picos**, **sharding por la clave de acceso**, y **el caso patológico** (la celebridad, el prefijo popular, el archivo gigante). Aprender a *reconocer* qué ladrillo aplica es la mitad de la entrevista.

---

## Caso 1 — Feed / timeline de red social (Twitter/Instagram)

**1. Requisitos.**
- Funcionales: un usuario postea; sus seguidores ven ese post en su **home timeline** (los posts de la gente que siguen, ordenados por recencia).
- No funcionales: **lectura-intensivo** (mirás el feed mucho más de lo que posteás); el timeline tolera **consistencia eventual** (que un post tarde segundos en aparecer está bien); latencia de lectura baja (abrir la app tiene que ser instantáneo); grafo social muy desbalanceado (un famoso tiene 100M seguidores, vos 200).

**2. Estimación.** 300M usuarios activos, cada uno abre el feed ~10 veces/día → ~35K lecturas/s promedio, picos de varios cientos de miles. Escrituras (posts): mucho menos, ~5K/s. Ratio lectura:escritura ~100:1 o más. Cada timeline muestra ~20 posts por pantalla.

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

**2. Estimación.** 500M usuarios, 50M conexiones WebSocket vivas en pico, ~40 mensajes/usuario/día → ~230K mensajes/s promedio, picos mayores. El cuello no es CPU: es **mantener millones de conexiones abiertas** (memoria/FD por conexión) y rutear entre ellas.

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

**2. Estimación.** Decenas de millones de notificaciones/día, con picos brutales (un partido, una caída que dispara alertas a todos). El sistema tiene que **absorber el pico** sin tumbar a los proveedores externos (APNs/FCM/el proveedor de SMS tienen sus propios rate limits).

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

**2. Estimación.** Cientos de millones de usuarios, archivos de KB a GB. El volumen total son **petabytes**; el costo dominante es **almacenamiento y ancho de banda**, no CPU. Muchos archivos son idénticos entre usuarios (el mismo PDF, el mismo instalador) → la **deduplicación** ahorra una fortuna.

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

Estos 9 casos cubren los patrones que más caen: fan-out, tiempo real, async con colas, lectura-intensivo con precómputo, separación metadata/contenido, estado compartido atómico, procesamiento masivo, consistencia fuerte transaccional e indexación espacial. Para seguir:

- **Resolvé en voz alta los que faltan**, reusando los mismos ladrillos: "diseñá YouTube/Netflix" (transcoding + CDN + el caso 5 de almacenamiento), "diseñá un ad-click aggregator / sistema de métricas" (streaming + ventanas + el caso 3 de colas), "diseñá Google Docs" (colaboración en vivo + el caso 2 de tiempo real). Lo que se evalúa es el recorrido, no la respuesta.
- **Conectá cada decisión con su implementación en el hub:** colas e idempotencia en [Event-driven](event-driven.md) y [Redis](redis.md), el contrato de cada API en [API design](api-design.md), tiempo real en [Tiempo real](tiempo-real.md), datos en [PostgreSQL](postgresql.md)/[NoSQL](nosql.md), control de flujo en [Concurrencia](concurrencia.md), y cómo operarlo en [Observabilidad](observabilidad.md).
- **Volvé al método** de [Diseño de sistemas](system-design.md) módulo 1 cada vez: requisitos → estimación → API/datos → alto nivel → profundizar donde duele → trade-offs. El método es el mismo para los 9 casos y para cualquier problema nuevo que te tiren.
