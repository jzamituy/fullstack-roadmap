# Redis: caché, colas y tareas en background

**Acelerar lecturas, sacar trabajo del request y escalar sin estado · BullMQ · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Redis aparece en casi toda oferta de backend, y por una razón: resuelve tres problemas distintos con la misma herramienta. **Caché** (que tus endpoints pesados no peguen a Postgres cada vez), **estado compartido** (sesiones/rate limiting que sobreviven al escalado horizontal del módulo de Node) y **colas de trabajos** (sacar del request lo lento o lo CPU-bound, también del módulo de Node). Este módulo conecta varios hilos que venís tirando: el N+1 y la performance del senior, el "no bloquees el event loop", el "stateless" del escalado, y el store de refresh tokens del módulo de auth.

**Lo que asumimos.** Tu "Task API" con Postgres, auth y deploy. El `docker-compose` del módulo anterior ya levanta Redis; lo usamos.

**Para practicar.** Redis ya está en tu Compose (`redis:7`). Para conectarte y probar comandos a mano:

```bash
docker compose exec redis redis-cli
```

Y las librerías para Node/Nest:

```bash
npm i bullmq @nestjs/bullmq ioredis @nestjs/cache-manager cache-manager @keyv/redis
```

**Índice de módulos**
1. Qué es Redis y por qué es tan rápido
2. Claves, valores y TTL (expiración)
3. Caché de lecturas: el patrón cache-aside
4. Invalidación: el problema difícil
5. Estado compartido: sesiones y apps stateless
6. Rate limiting con Redis
7. Colas de trabajos: por qué sacar trabajo del request
8. BullMQ: productor y consumidor
9. Reintentos, backoff y dead-letter queue
10. Idempotencia en los workers
11. Tareas programadas (cron) y Redis en NestJS

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es Redis y por qué es tan rápido

**Teoría.** **Redis** es un almacén de datos **clave-valor en memoria**. La diferencia clave con Postgres: Postgres guarda en disco (durable, relacional, transaccional); Redis guarda principalmente en **RAM**, lo que lo hace **órdenes de magnitud más rápido** para leer/escribir, a cambio de menos garantías de durabilidad y un modelo de datos mucho más simple.

No reemplaza a tu base de datos: la **complementa**. Postgres es la **fuente de verdad** (donde viven tus datos para siempre); Redis es la capa veloz para lo que necesitás **rápido y/o efímero**:

- **Caché**: copias temporales de datos caros de calcular o traer.
- **Sesiones / tokens**: estado de usuario que querés compartir entre instancias.
- **Rate limiting**: contadores que expiran solos.
- **Colas**: lista de trabajos a procesar en background.

Redis guarda valores bajo **claves** (strings) y soporta varias estructuras: strings, hashes (objetos), lists (colas), sets, sorted sets. Pero el 80% del uso cotidiano es: guardar un valor bajo una clave, leerlo, y que **expire solo** tras un tiempo (TTL).

La regla mental: **si lo perdés y no pasa nada grave, puede ir en Redis** (un caché se reconstruye desde Postgres). Si perderlo es perder datos del negocio, va en Postgres. Redis, por defecto, no es para tu fuente de verdad.

**Por dentro: single-thread.** Acá viene la pregunta de entrevista que separa al que "usó Redis" del que lo entiende: Redis procesa los comandos en **un solo hilo**. Por eso un `INCR` no necesita locks (no hay dos comandos ejecutándose a la vez) y la atomicidad por comando sale gratis. Pero tiene una contracara filosa: **un comando lento bloquea a todos los demás**. Si corrés `KEYS *` (que recorre todo el keyspace, O(n)) en una base con millones de claves, congelás Redis para todos los clientes durante ese tiempo. Por eso en producción se usa `SCAN` (itera de a cursores, no bloquea) y se evitan comandos O(n) sobre estructuras grandes. Regla: en Redis, "lento para uno" es "lento para todos".

**Persistencia: ¿qué pasa si Redis se reinicia?** "En memoria" no significa "se pierde todo siempre": Redis puede persistir a disco de dos formas.

- **RDB** (snapshots): cada cierto tiempo vuelca toda la memoria a un archivo. Rápido y compacto, pero si crashea entre snapshots **perdés los últimos segundos/minutos** de escrituras.
- **AOF** (append-only file): registra cada escritura en un log. Más durable (con `appendfsync everysec` perdés como mucho ~1s), a costa de archivos más grandes y algo más lento.

Para un caché esto casi no importa (se reconstruye desde Postgres). Pero el punto operativo es: tras un restart o un failover, **el caché puede aparecer vacío de golpe** — y eso conecta con un problema serio que vemos en el módulo 4 (cache stampede).

**Evicción: ¿qué pasa cuando Redis se llena?** La RAM es finita. Con `maxmemory` configurás un tope; con `maxmemory-policy` decidís qué hace al alcanzarlo. Para un caché, la política típica es **`allkeys-lru`**: descarta las claves menos usadas recientemente para hacer lugar. Sin una política de evicción adecuada, Redis empieza a rechazar escrituras (o mata el proceso por OOM) cuando se llena. Es la respuesta a la pregunta clásica: *"¿qué pasa cuando tu Redis de caché se queda sin memoria?"* → evicta las claves frías, no explota.

**Ejercicios 1**
1.1 ¿Por qué Redis es mucho más rápido que Postgres para leer un valor? ¿Qué resignás a cambio?
1.2 ¿Redis reemplaza a tu base de datos relacional? Explicá la relación entre ambos.
1.3 Dá un ejemplo de dato que pondrías en Redis y uno que jamás pondrías solo en Redis. Justificá.
1.4 Redis es single-thread. Explicá por qué eso hace peligroso correr `KEYS *` en producción y qué usarías en su lugar.
1.5 Diferenciá RDB de AOF: ¿qué se pierde con cada uno tras un crash? Para un caché, ¿importa mucho? ¿Por qué?
1.6 ¿Qué hace `maxmemory-policy allkeys-lru` y por qué es la política típica cuando usás Redis como caché?

---

## Módulo 2 — Claves, valores y TTL (expiración)

**Teoría.** Las operaciones básicas son `SET` (guardar), `GET` (leer) y `DEL` (borrar). Lo que hace a Redis especialmente útil para caché es el **TTL** (time to live): podés decirle que una clave **se borre sola** después de N segundos.

```bash
SET usuario:42:nombre "Ana"        # guardar
GET usuario:42:nombre              # → "Ana"
SET sesion:abc "datos" EX 3600     # guardar con expiración de 3600s (1 hora)
TTL sesion:abc                     # → segundos que le quedan de vida
DEL usuario:42:nombre              # borrar
INCR visitas                       # incrementa un contador atómicamente (útil para rate limiting)
```

Dos conceptos importantes:

- **Convención de claves**: como todo es un espacio plano de claves, se usan **prefijos con `:`** para organizar (`usuario:42`, `producto:7`, `sesion:abc`). No es una regla del motor, es una convención para no pisar claves entre features.
- **El TTL es la base del caché**: una clave con `EX 60` vive 60 segundos y desaparece. Eso te da expiración automática sin que tengas que limpiar nada — exactamente lo que querés para datos cacheados que pueden quedar desactualizados.

`INCR` merece una mención: incrementa un número de forma **atómica** (sin condiciones de carrera aunque mil requests lo toquen a la vez). Es lo que hace que Redis sea perfecto para contadores y rate limiting (módulo 6).

**Ojo con el alcance de la atomicidad:** es **por comando individual**, no por secuencias. Un `INCR` es atómico; pero `INCR` *seguido de* `EXPIRE` son **dos comandos separados** y entre uno y otro puede pasar cualquier cosa (incluso que el proceso muera). Esa distinción es la que rompe el rate limiting ingenuo del módulo 6 — y la solución (scripts Lua / `MULTI`) es justamente cómo volver atómica una secuencia. Tenelo presente desde ya.

**Ejercicios 2**
2.1 ¿Qué hace `SET clave valor EX 60` que un `SET clave valor` no hace? ¿Por qué es la base del caché?
2.2 ¿Para qué sirve la convención de prefijos con `:` en las claves (ej. `producto:7`)?
2.3 ¿Por qué `INCR` es mejor que leer un contador, sumarle 1 en tu código y volver a escribirlo?
2.4 (teclado) Abrí `redis-cli` y probá: `SET demo:contador 0`, `INCR demo:contador` (varias veces), `EXPIRE demo:contador 10`, `TTL demo:contador`, y observá cómo desaparece la clave al pasar los 10s (`GET demo:contador` → `(nil)`). ¿Qué devuelve `TTL` de una clave sin expiración?

---

## Módulo 3 — Caché de lecturas: el patrón cache-aside

**Teoría.** El uso más común de Redis. Tenés un endpoint que trae datos caros (una query pesada, un cálculo, una API externa) y se piden seguido pero cambian poco. En vez de pegarle a Postgres cada vez, **cacheás** el resultado.

El patrón estándar es **cache-aside** (el del archivo senior): buscás en el caché; si está (**hit**), lo devolvés; si no está (**miss**), vas a la fuente, guardás en el caché con un TTL, y devolvés.

```ts
async obtenerProducto(id: number): Promise<Producto> {
  const clave = `producto:${id}`;

  // 1. ¿está en caché?
  const cacheado = await this.cache.get<Producto>(clave);
  if (cacheado) return cacheado;                 // HIT: no tocamos Postgres

  // 2. MISS: vamos a la fuente de verdad
  const producto = await this.repo.findById(id);
  if (!producto) throw new NotFoundException();

  // 3. guardamos en caché con TTL para próximas lecturas
  await this.cache.set(clave, producto, 60_000); // 60s
  return producto;
}
```

Lo que ganás: las primeras peticiones pagan el costo de Postgres; las siguientes (durante 60s) responden desde memoria, en microsegundos, y le sacan carga a la base. Es una de las optimizaciones de mayor impacto cuando un endpoint se consulta mucho.

El criterio (que conecta con "medí antes de optimizar" del senior): cacheá lo que es **caro de obtener** y se lee **mucho más de lo que cambia** (un catálogo de productos, la config). No caches lo que cambia constantemente o se lee una vez: ahí el caché agrega complejidad (e inconsistencia, módulo 4) sin beneficio.

> **Detalle que rompe en producción: ¿dónde guarda el `CacheModule`?** Por defecto, `@nestjs/cache-manager` usa un store **en memoria del proceso**. Eso significa que cada instancia de tu API tiene **su propio caché** — y tirás abajo todo lo que vas a ver en el módulo 5 (stateless): 10 instancias = 10 cachés distintos, hits inconsistentes, e invalidar en una no invalida en las otras. Para que el caché sea **compartido** tenés que registrar un **store de Redis**:
>
> ```ts
> import { CacheModule } from "@nestjs/cache-manager";
> import { createKeyv } from "@keyv/redis";
>
> CacheModule.registerAsync({
>   isGlobal: true,
>   useFactory: () => ({
>     stores: [createKeyv("redis://redis:6379")], // store compartido, no memoria del proceso
>   }),
> });
> ```
>
> Además, **cuidado con la versión y la unidad del TTL**: en `cache-manager` v5/v6 (el moderno) la firma es `set(key, value, ttl)` con el **TTL en milisegundos** (`60_000` = 60s, como arriba). En v4 el TTL iba en **segundos**. Pineá la versión en tu `package.json` para no comerte un caché que dura 1000× más o menos de lo que creés.

**Ejercicios 3**
3.1 Describí los tres pasos del patrón cache-aside ante un cache miss.
3.2 ¿Qué tipo de dato es buen candidato para cachear y cuál no? Dá un ejemplo de cada uno.
3.3 ¿Qué pasaría en la primera petición a un producto que no está cacheado vs. en la segunda (dentro del TTL)?
3.4 Si registrás el `CacheModule` sin un store de Redis y corrés 3 instancias de tu API, ¿qué problema aparece? ¿Por qué contradice el "stateless" del módulo 5?
3.5 (teclado) Agregá un `console.time`/`console.timeEnd` (o un log de duración) alrededor de la query a Postgres y de la lectura de caché en `obtenerProducto`. Pegale dos veces al endpoint y compará la latencia del miss (1ra) contra el hit (2da).

---

## Módulo 4 — Invalidación: el problema difícil

**Teoría.** "Solo hay dos problemas difíciles en computación: la invalidación de caché y nombrar cosas" (la cita del archivo senior). El caché es fácil de poner; lo difícil es mantenerlo **coherente** cuando el dato cambia. Si cacheaste un producto por 60s y alguien le edita el precio, durante hasta 60s estás sirviendo el **precio viejo**. Eso es **stale data** (dato desactualizado).

Dos estrategias para manejarlo:

- **Expiración por TTL (la simple)**: aceptás que el dato puede estar desactualizado hasta `TTL` segundos. Para muchos casos (un catálogo, un perfil) un TTL corto es suficiente y no requiere nada más. El precio de la simplicidad es una ventana de inconsistencia acotada.
- **Invalidación explícita (la precisa)**: cuando **escribís** el dato, **borrás (o actualizás)** su clave en el caché, para que la próxima lectura sea un miss y traiga lo nuevo.

```ts
async actualizarProducto(id: number, datos: Partial<Producto>): Promise<Producto> {
  const actualizado = await this.repo.update(id, datos);
  await this.cache.del(`producto:${id}`); // invalidamos: la próxima lectura recachea lo nuevo
  return actualizado;
}
```

El criterio: **TTL para tolerancia a desactualización** (la mayoría de los casos); **invalidación explícita cuando la frescura importa** (un saldo, un stock). Lo difícil aparece con datos cacheados en varios lugares o de forma derivada (cacheaste "lista de productos" y editás uno: ¿invalidás la lista entera?). No hay bala de plata; por eso es "difícil". Para empezar, **TTL corto + invalidar la clave puntual al escribir** cubre casi todo sin meterte en complejidad accidental.

**El problema senior: cache stampede (thundering herd).** Hay un segundo problema, menos obvio que la frescura, y es la pregunta de entrevista que casi nadie ve venir. Imaginá una clave **caliente** (un producto popular, la home) con TTL de 60s, recibiendo 5.000 req/s. En el instante exacto en que esa clave **expira**, las 5.000 peticiones del siguiente segundo encuentran **miss a la vez** y **todas pegan a Postgres simultáneamente** para reconstruir el mismo valor. Postgres recibe un pico brutal de queries idénticas y puede caerse — justo lo que el caché venía evitando. El mismo efecto, multiplicado, ocurre tras un restart o failover de Redis: el caché aparece vacío de golpe (recordá la persistencia del módulo 1) y *todo* es miss al mismo tiempo.

Mitigaciones (de la más simple a la más robusta):

- **TTL con jitter**: en vez de `EX 60` fijo, usá `60 + random(0..15)`. Así las claves no expiran todas en el mismo segundo y los misses se reparten en el tiempo.
- **Single-flight / lock de recálculo**: ante un miss, solo **un** request toma un lock (`SET NX`, módulo 10) y recalcula; los demás esperan o sirven el valor viejo un instante. Evita que mil hilos hagan la misma query.
- **Early recompute**: refrescás la clave *antes* de que expire (cuando le queda poco TTL), de fondo, sin esperar al miss.

Para tu Task API, con empezar por **jitter en el TTL** ya cubrís el 90% del riesgo sin complejidad. Pero saber nombrar "cache stampede" y proponer single-flight es lo que se espera en una entrevista mid/senior.

**Ejercicios 4**
4.1 ¿Qué es "stale data" y por qué el caché lo provoca?
4.2 ¿Cuándo alcanza con TTL y cuándo necesitás invalidación explícita? Dá un ejemplo de cada uno.
4.3 Escribí el patrón: al actualizar un producto en la base, ¿qué hacés con su clave en el caché y por qué?
4.4 ¿Qué es un "cache stampede" y en qué dos momentos típicos ocurre? Nombrá al menos dos mitigaciones.
4.5 ¿Por qué agregarle un jitter aleatorio al TTL ayuda contra el stampede? ¿Qué problema NO resuelve (pista: un restart de Redis que vacía todo)?

---

## Módulo 5 — Estado compartido: sesiones y apps stateless

**Teoría.** Acá Redis cierra un hilo del módulo de Node. Para escalar horizontalmente, tu app debe ser **stateless**: no guardar en la memoria del proceso estado que deba sobrevivir entre requests, porque cada request puede caer en una instancia distinta detrás del balanceador.

¿Dónde va entonces ese estado compartido? En un **store externo** que todas las instancias ven: Postgres para lo durable, **Redis para lo efímero y veloz**. Casos típicos:

- **Sesiones**: si guardás la sesión en memoria del proceso, un usuario que cae en otra instancia "se desloguea". En Redis, todas las instancias leen la misma sesión.
- **Refresh tokens**: justo lo que viste en el módulo de auth. El store de refresh tokens (para poder revocarlos) va en Redis: rápido, con TTL automático (el token expira solo), y compartido entre instancias.

```ts
// guardar el hash del refresh token con TTL = su tiempo de vida
await this.redis.set(`refresh:${usuarioId}:${jti}`, hash, "EX", 7 * 24 * 3600);

// al renovar: verificar que sigue vigente (no revocado, no expirado)
const guardado = await this.redis.get(`refresh:${usuarioId}:${jti}`);
if (!guardado) throw new UnauthorizedException("Refresh token inválido");

// logout: revocar = borrar la clave
await this.redis.del(`refresh:${usuarioId}:${jti}`);
```

El TTL de Redis hace el trabajo sucio: el refresh token se borra solo cuando expira, sin un cron de limpieza. Y como vive afuera del proceso, podés correr 10 instancias de tu API y el login funciona igual caigas donde caigas. Esta es la pieza concreta que faltaba para que el "escalado horizontal + stateless" del módulo de Node sea real.

**El matiz que diferencia esto de un JWT stateless: la revocación.** Un access token JWT puro es **stateless** y no se puede invalidar antes de que expire — por eso se le pone una vida corta (minutos). El refresh token, en cambio, vive en Redis: para **revocarlo al instante** (logout, sospecha de robo) basta con `DEL` de su clave, y la próxima renovación falla. Eso es exactamente lo que el JWT stateless no te da. Fijate además que guardamos el **hash** del token, no el token: si en la rotación de refresh tokens detectás que llega un token cuyo hash no coincide con el guardado (o que ya fue usado), tenés señal de **reuso/robo** y podés revocar toda la familia de tokens del usuario. La presencia de la clave (con `jti` único) es la validación; el hash guardado habilita la detección de reuso.

**Ejercicios 5**
5.1 ¿Por qué guardar sesiones en la memoria del proceso rompe el escalado horizontal?
5.2 ¿Por qué Redis (y no Postgres) es el lugar típico para sesiones y refresh tokens? Mencioná el TTL.
5.3 Conectá con auth: ¿qué ventaja da que el store de refresh tokens esté en Redis y no en memoria, para el logout y para escalar?
5.4 Un access token JWT stateless no se puede revocar antes de expirar. ¿Cómo logra Redis la revocación inmediata del refresh token, y para qué sirve guardar su hash y no el token en claro?

---

## Módulo 6 — Rate limiting con Redis

**Teoría.** El **rate limiting** (limitar cuántos requests puede hacer un cliente en una ventana de tiempo) protege de abuso y de fuerza bruta — lo viste como medida de seguridad en el módulo de auth. La implementación natural usa Redis porque necesita un **contador compartido entre instancias** (si cada instancia cuenta por su lado, el límite real es N veces mayor) y con **expiración automática**.

La idea base es un contador con `INCR` y una ventana con `EXPIRE`. **Pero acá hay una trampa de producción** que es pregunta clásica de entrevista senior. La versión ingenua:

```ts
// ❌ NO en producción: tiene una race condition
async permitir(ip: string, limite = 10, ventanaSeg = 60): Promise<boolean> {
  const clave = `rate:${ip}`;
  const actual = await this.redis.incr(clave);   // comando 1
  if (actual === 1) {
    await this.redis.expire(clave, ventanaSeg);   // comando 2 (¡separado!)
  }
  return actual <= limite;
}
```

El problema: `INCR` y `EXPIRE` son **dos comandos distintos, no atómicos** (recordá el matiz del módulo 2). Si entre el `INCR` que crea la clave y el `EXPIRE` el proceso muere, la conexión se corta o Redis tiene un hipo, la clave queda **sin TTL para siempre**. Resultado: el contador nunca se reinicia y ese cliente queda **bloqueado permanentemente** — el mismo desastre que describe la solución 6.2 para el caso "sin EXPIRE", solo que ahora pasa por accidente. No es teórico: es *el* ejemplo canónico de por qué "atómico por comando" no es lo mismo que "atómico por secuencia".

**La solución real: un script Lua**, que Redis ejecuta de forma **atómica** (un solo paso del hilo único, sin nada en el medio):

```ts
// ✅ INCR + EXPIRE en una sola operación atómica
const LUA_RATE_LIMIT = `
  local actual = redis.call('INCR', KEYS[1])
  if actual == 1 then
    redis.call('EXPIRE', KEYS[1], ARGV[1])
  end
  return actual
`;

async permitir(ip: string, limite = 10, ventanaSeg = 60): Promise<boolean> {
  const actual = (await this.redis.eval(
    LUA_RATE_LIMIT, 1, `rate:${ip}`, String(ventanaSeg),
  )) as number;
  return actual <= limite;
}
```

Como el script corre entero o no corre, nunca queda una clave sin TTL. (Alternativas equivalentes: `MULTI`/pipeline, o `SET clave 0 EX n NX` para crear la clave con TTL antes de incrementar.)

**Ventana fija vs sliding window.** Lo de arriba es un **rate limiter de ventana fija** (fixed window): cuenta requests en bloques de 60s alineados al reloj. Tiene un defecto conocido: en el **borde de la ventana** un cliente puede hacer el límite completo al final de una ventana y otro límite completo al principio de la siguiente → hasta **2× el límite** en un instante (10 req en el segundo 59 + 10 en el 61 = 20 req en 2 segundos). Para límites estrictos esto importa. Alternativas que se esperan a nivel senior:

- **Sliding window** (log o counter): cuenta los requests de los *últimos* 60s reales, no del bloque alineado. Se implementa con un **sorted set** (`ZADD` con timestamp, `ZREMRANGEBYSCORE` para purgar lo viejo, `ZCARD` para contar).
- **Token bucket**: un balde que se rellena a ritmo constante; cada request consume un token. Permite ráfagas controladas. Es lo que usan muchos rate limiters reales (y APIs como las de los cloud providers).

En NestJS no hace falta escribir nada de esto a mano: el **`ThrottlerModule`** (del módulo de auth) lo hace, y con `ThrottlerStorageRedis` usa Redis como almacén para multi-instancia. Y no es casualidad que **ese storage use scripts Lua por dentro**: justamente para evitar la race de INCR+EXPIRE. Entender la mecánica es lo que te deja explicarlo, elegir el algoritmo y diagnosticarlo.

**Ejercicios 6**
6.1 ¿Por qué el rate limiting necesita un contador en Redis y no en la memoria de cada instancia?
6.2 ¿Qué rol cumple el `EXPIRE` en la implementación con `INCR`? ¿Qué pasaría sin él?
6.3 ¿Qué ataque (visto en el módulo de auth) ayuda a frenar el rate limiting en el endpoint de login?
6.4 Explicá la race condition de `INCR` + `EXPIRE` como dos comandos separados: ¿qué pasa si el proceso muere en el medio, y cómo lo arregla un script Lua?
6.5 ¿Qué defecto tiene la "ventana fija" en el borde entre dos ventanas? Nombrá un algoritmo alternativo y con qué estructura de Redis lo implementarías.

---

## Módulo 7 — Colas de trabajos: por qué sacar trabajo del request

**Teoría.** Esta es la otra gran función de Redis y conecta directo con el módulo de Node ("no bloquees el event loop"). Imaginá que al registrarse un usuario querés enviarle un email de bienvenida. Enviar el email tarda 1-2 segundos (red, servidor SMTP) y **puede fallar**. Si lo hacés dentro del request:

- El usuario **espera** esos 2 segundos para recibir su respuesta.
- Si el envío falla, ¿le devolvés error al registro, aunque el usuario sí se creó?
- Trabajo pesado (generar un PDF, procesar un video) **bloquearía** o colgaría el request.

La solución es una **cola de trabajos** (job queue): en vez de hacer el trabajo lento en el request, **encolás** un "trabajo" (un mensaje con los datos) y respondés al instante. Un **worker** (proceso aparte) toma los trabajos de la cola y los procesa en **background**, a su ritmo, con reintentos si fallan.

```
Request → crea usuario → encola "enviar email" → responde 201 (rápido)
                                  │
                         [ cola en Redis ]
                                  │
                     Worker (proceso aparte) → toma el trabajo → envía el email
                                                                 (reintenta si falla)
```

Lo que ganás: respuestas rápidas (el usuario no espera el email), **resiliencia** (si el email falla, se reintenta sin afectar el registro), y la posibilidad de sacar trabajo **CPU-bound** del hilo principal (el "mandalo a una cola" del módulo de Node). Redis es el **broker**: guarda la cola de trabajos pendientes que el worker consume.

**Ejercicios 7**
7.1 Nombrá tres problemas de enviar el email de bienvenida **dentro** del request de registro.
7.2 ¿Qué hace el request y qué hace el worker en un esquema de cola? ¿Quién espera y quién no?
7.3 Conectá con el módulo de Node: ¿por qué una cola es la respuesta a "no bloquees el event loop con trabajo CPU-bound"?

---

## Módulo 8 — BullMQ: productor y consumidor

**Teoría.** **BullMQ** es la librería estándar de colas sobre Redis en Node. Tiene dos lados: el **productor** encola trabajos (`Queue.add`), el **consumidor** (`Worker`) los procesa.

```ts
import { Queue, Worker } from "bullmq";

// En Docker Compose el host es el NOMBRE DEL SERVICIO (redis), no localhost.
// localhost solo sirve si corrés Node fuera de Compose contra un Redis local.
const connection = {
  host: process.env.REDIS_HOST ?? "redis",
  port: Number(process.env.REDIS_PORT ?? 6379),
};

// PRODUCTOR (en tu service, tras crear el usuario)
const emailQueue = new Queue("emails", { connection });
await emailQueue.add("bienvenida", { to: "ana@test.com", nombre: "Ana" });

// CONSUMIDOR (proceso worker, separado)
new Worker(
  "emails",
  async (job) => {
    // job.data = { to, nombre } ; acá hacés el trabajo lento/que puede fallar
    await enviarEmail(job.data.to, job.data.nombre);
  },
  { connection },
);
```

En NestJS se integra con `@nestjs/bullmq`: registrás la cola con `BullModule.registerQueue`, inyectás la `Queue` para encolar, y definís el procesador con `@Processor`:

```ts
@Processor("emails")
export class EmailProcessor extends WorkerHost {
  constructor(@Inject("MAILER") private readonly mailer: MailerPort) {}

  async process(job: Job<{ to: string; nombre: string }>): Promise<void> {
    await this.mailer.enviar(job.data.to, job.data.nombre);
    // si esto lanza, BullMQ marca el job como fallido y lo reintenta (módulo 9)
  }
}
```

Puntos clave: el productor y el consumidor pueden vivir en **procesos distintos** (incluso escalar el worker por separado de la API — otra vez el stateless: el estado está en Redis); y si el `process` lanza una excepción, BullMQ trata el job como fallido y aplica la política de reintentos. Fijate que el procesador inyecta sus dependencias por DI igual que cualquier service — el `MailerPort` que conociste en hexagonal.

**Ejercicios 8**
8.1 ¿Cuál es el rol del productor (`Queue.add`) y cuál el del consumidor (`Worker`/`@Processor`)?
8.2 ¿Qué pasa con un job si el método `process` lanza una excepción?
8.3 ¿Por qué es una ventaja que el worker pueda correr en un proceso separado de la API? Conectá con "stateless" y escalado.
8.4 (teclado) Levantá una `Queue` y un `Worker` sobre tu Redis del Compose. Encolá un job cuyo `process` haga `console.log(job.data)` tras un `await` de 2s. Verificá que el productor responde al instante y que el log del worker aparece 2s después, en otro proceso/terminal.

---

## Módulo 9 — Reintentos, backoff y dead-letter queue

**Teoría.** Las colas brillan en el manejo de fallos, y acá se reúne todo lo de **resiliencia** del archivo senior. Un trabajo (enviar email, llamar a una API) puede fallar por un blip transitorio. BullMQ permite configurar **reintentos automáticos con backoff exponencial**:

```ts
await emailQueue.add(
  "bienvenida",
  { to: "ana@test.com" },
  {
    attempts: 4,                                   // hasta 4 intentos
    backoff: { type: "exponential", delay: 1000 }, // 1s, 2s, 4s... (con jitter interno)
  },
);
```

Esto es exactamente el **retry con backoff exponencial** del senior: reintentar esperando cada vez más, para no generar una "tormenta de reintentos" contra un servicio que se está recuperando. La cola te lo da configurado, no lo escribís a mano.

Pero, ¿qué pasa si un trabajo falla **siempre** (un email malformado, un bug)? No puede reintentarse para siempre bloqueando recursos. Tras agotar los `attempts`, el job queda marcado como **failed**.

**Lo primero que hay que saber: BullMQ ya tiene una "DLQ" incorporada.** Los jobs que agotan sus intentos **no se pierden**: BullMQ los retiene en el estado `failed`, y los inspeccionás con `queue.getFailed()`. Podés controlar cuántos retiene con `removeOnFail` (ej. `removeOnFail: 1000` o `{ age: 7 * 24 * 3600 }`). Para la mayoría de los casos, **eso ya es tu dead-letter**: revisás los failed, arreglás la causa y los reintentás con `job.retry()`.

```ts
// Inspeccionar y reintentar trabajos fallidos (en un endpoint de admin o un script)
const fallidos = await emailQueue.getFailed();
for (const job of fallidos) {
  console.log(job.id, job.failedReason, job.data);
  // await job.retry(); // si arreglaste la causa
}
```

Una **DLQ separada** (una cola aparte adonde mover los muertos) tiene sentido cuando querés un flujo distinto para ellos (alertas, otro worker, retención larga, métricas). Si la implementás con un listener, hacelo con cuidado, porque el patrón ingenuo es frágil:

```ts
// DLQ explícita opcional. Ojo: el evento 'failed' se emite en cada intento fallido,
// no solo en el último; y el job puede fallar por 'stalled', no solo porque process() lanzó.
new Worker("emails", procesar, { connection }).on(
  "failed",
  async (job, err) => {
    if (!job) return; // en algunos fallos (stalled) job puede venir undefined
    const agotado = job.attemptsMade >= (job.opts.attempts ?? 1);
    if (!agotado) return; // todavía le quedan reintentos: no es la DLQ aún
    await deadLetterQueue.add("email-fallido", { original: job.data, error: err.message });
  },
);
```

Aun así, mover a mano desde un listener puede **duplicar o perder** bajo un crash (el listener corre después del fallo, sin garantía transaccional). Para algo serio, preferí apoyarte en la cola `failed` nativa o usar `QueueEvents` para observabilidad. La regla: **empezá usando el `failed` que BullMQ ya te da**; una DLQ separada es una decisión deliberada, no el default.

La regla del senior aplicada: reintentá solo fallos **transitorios** (timeouts, 5xx), nunca errores de negocio (un email inválido no mejora reintentando — ese debería fallar rápido y quedar en `failed`); y asegurá que el trabajo sea **idempotente** (módulo 10), porque los reintentos lo van a ejecutar más de una vez.

**Ejercicios 9**
9.1 ¿Por qué los reintentos usan backoff exponencial y no reintentan inmediato? (Recordá la "tormenta de reintentos" del senior.)
9.2 ¿Qué es una dead-letter queue y qué problema resuelve? ¿Por qué con BullMQ muchas veces no necesitás una cola separada?
9.3 ¿Por qué tiene sentido reintentar un timeout pero NO un "email con formato inválido"?
9.4 (teclado) Encolá un job cuyo `process` lance siempre una excepción, con `attempts: 3` y `backoff` exponencial. Observá en los logs cómo reintenta espaciando los intentos, y al final listá los trabajos muertos con `await queue.getFailed()`.

---

## Módulo 10 — Idempotencia en los workers

**Teoría.** Las colas garantizan entrega **"al menos una vez" (at-least-once)**, no "exactamente una vez". ¿Por qué no exactly-once? Porque entre *hacer el efecto* y *avisar que lo hiciste* (el ack) siempre hay un instante en que el proceso puede morir: si el worker cobra y muere **antes** de marcar el job completado, la cola cree que falló y lo reintenta. Garantizar "exactamente una vez" de punta a punta es esencialmente **imposible** en un sistema distribuido (es la forma práctica del problema de los dos generales): no hay forma de que dos máquinas se pongan de acuerdo con certeza sobre si un mensaje llegó cuando la red y los procesos pueden fallar en cualquier momento. Lo que se hace en la realidad es **at-least-once + idempotencia**, que entrega el efecto observable de "exactamente una vez".

Por eso los workers **deben ser idempotentes**: ejecutar el mismo trabajo dos veces produce el mismo resultado que ejecutarlo una. Es el concepto del archivo senior, ahora donde más importa. Si "procesar pago" no es idempotente y el job se reintenta, **cobrás dos veces**.

La técnica: una **clave de idempotencia** que registrás; si el trabajo ya se procesó, no lo repetís.

```ts
async process(job: Job<{ pedidoId: string }>): Promise<void> {
  const clave = `procesado:pago:${job.data.pedidoId}`;

  // SET ... NX = "set if not exists": pone la clave solo si no estaba, de forma atómica.
  // OJO con ioredis: NO devuelve un booleano. Devuelve "OK" si seteó, o null si ya existía.
  const resultado = await this.redis.set(clave, "1", "EX", 86400, "NX");
  if (resultado === null) return; // la clave ya estaba: este pago ya se procesó

  await this.gateway.cobrar(job.data.pedidoId);
}
```

**Cuidado con el orden, que es sutil.** En el código de arriba seteamos la clave *antes* de cobrar. Eso evita el doble cobro, pero abre otra ventana: si el worker setea la clave y **muere justo antes de `cobrar`**, el reintento ve la clave puesta y **no cobra nunca** (pérdida del efecto). Al revés (cobrar primero, setear después) corrés el riesgo opuesto: morir entre el cobro y el set → doble cobro. **No hay orden perfecto con un flag en Redis.** Para efectos críticos como un pago, lo robusto es **empujar la idempotencia al destino (el sink)**: mandar una `idempotency key` al gateway de pago, que es quien garantiza no cobrar dos veces la misma key. El flag en Redis es una buena primera línea para emails/notificaciones; para dinero, la garantía la tiene que dar el sistema que mueve el dinero.

Para operaciones **naturalmente idempotentes** (marcar una tarea como completada: ponerla en `true` dos veces da lo mismo) no hace falta nada extra; el cuidado es para las que tienen efectos acumulativos (cobrar, enviar, incrementar).

**Ejercicios 10**
10.1 ¿Por qué un worker de cola DEBE ser idempotente? ¿Qué propiedad de las colas lo obliga?
10.2 Dá un ejemplo de operación que se rompe si no es idempotente y una que ya lo es naturalmente.
10.3 ¿Cómo usás `SET NX` con TTL en Redis para evitar procesar dos veces el mismo trabajo? ¿Qué devuelve `SET ... NX` en ioredis cuando la clave ya existía?
10.4 ¿Por qué "exactamente una vez" es esencialmente imposible end-to-end, y cómo se logra el efecto equivalente en la práctica?
10.5 Mostrá la ventana de fallo según el orden: ¿qué pasa si seteás la clave de idempotencia antes de cobrar y morís en el medio? ¿Y si cobrás primero? ¿Por qué para un pago conviene una idempotency key en el gateway?

---

## Módulo 11 — Tareas programadas (cron) y Redis en NestJS

**Teoría.** Además de trabajos disparados por eventos, a veces necesitás trabajo **programado**: limpiar registros viejos cada noche, enviar un resumen semanal, recalcular estadísticas cada hora. Son los **cron jobs**.

En NestJS, para tareas programadas simples está `@nestjs/schedule` con el decorador `@Cron`:

```ts
@Injectable()
export class LimpiezaService {
  @Cron("0 3 * * *") // todos los días a las 3:00 AM (sintaxis cron)
  async limpiarTokensExpirados(): Promise<void> {
    await this.repo.borrarTokensVencidos();
  }
}
```

Pero **cuidado con el cron en multi-instancia**: si corrés 3 instancias de tu API, las 3 dispararían el mismo cron a las 3 AM → el trabajo se ejecuta 3 veces (emails triplicados, estadísticas recalculadas 3 veces). Este es un error clásico que separa a quien probó algo en local de quien lo operó escalado.

**La opción recomendada: BullMQ con jobs repetibles** (`repeat`). En vez de un `@Cron` en cada instancia, encolás un job repetible y BullMQ se coordina vía Redis para que **solo un worker tome cada ejecución**:

```ts
await tareasQueue.add(
  "limpieza-nocturna",
  {},
  { repeat: { pattern: "0 3 * * *" }, jobId: "limpieza-nocturna" }, // jobId fijo evita duplicar el repeatable
);
```

**La otra opción, el lock distribuido, viene con letra chica que se espera que conozcas.** Sí, podés usar `SET NX EX` para que solo la instancia que "gana" el lock ejecute. Pero un lock sobre Redis es **best-effort, no una garantía dura de exclusión mutua**, y tiene dos trampas:

1. **Expiración prematura**: si le ponés `EX 30` al lock pero el trabajo tarda 40s, el lock expira mientras seguís trabajando → otra instancia agarra el lock y ejecuta **en paralelo** (doble ejecución, justo lo que querías evitar).
2. **Borrado inseguro**: si al terminar hacés `DEL` del lock a ciegas, podés estar **borrando el lock de otra instancia** (la tuya expiró, otra lo tomó, y vos borrás el de ella). Por eso el lock se guarda con un **valor único (token)** y se libera comparando ese token, idealmente con un script Lua atómico:

```ts
const token = randomUUID();
const tomado = await this.redis.set("lock:cron", token, "EX", 30, "NX"); // "OK" | null
if (tomado === null) return; // otra instancia tiene el lock
try {
  await this.tarea();
} finally {
  // liberar solo si el token sigue siendo el nuestro (atómico, evita borrar el de otro)
  await this.redis.eval(
    `if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) else return 0 end`,
    1, "lock:cron", token,
  );
}
```

Aun así, esto **no garantiza** exclusión bajo pausas de GC o relojes desincronizados: es la crítica de Martin Kleppmann a **Redlock** (el algoritmo de lock distribuido multi-nodo de Redis). Para exclusión *correcta* hacen falta **fencing tokens** (un número creciente que el recurso protegido valida), algo que Redis solo no te da. Conclusión práctica: para un cron, **usá `repeat` de BullMQ**; el lock con token es aceptable para tareas idempotentes donde una doble ejecución ocasional no es catastrófica, pero conocé sus límites.

**Caché declarativo en Nest.** El **`CacheModule`** (con el store Redis del módulo 3) integra Redis de forma declarativa, e incluso podés cachear endpoints enteros con un interceptor (`@UseInterceptors(CacheInterceptor)`), aunque para invalidación fina (módulo 4) conviene el control manual con el patrón cache-aside.

**Lo que queda para el próximo módulo: Pub/Sub.** Redis también tiene **Pub/Sub** (`PUBLISH`/`SUBSCRIBE`): publicás un mensaje en un canal y todos los suscriptores lo reciben. Es **fire-and-forget** — no persiste, no reintenta, si nadie escucha el mensaje se pierde (por eso no sirve como cola de trabajos: para eso está BullMQ). Pero es ideal para propagar eventos en vivo entre instancias, y es exactamente el mecanismo que necesitás para escalar **WebSockets** a varias instancias — el tema del próximo módulo.

**Ejercicios 11**
11.1 ¿Qué problema aparece si tenés un `@Cron` y escalás tu API a 3 instancias? ¿Cuál es la opción recomendada para resolverlo?
11.2 ¿Qué herramienta de Nest usarías para una tarea programada simple, y qué para coordinarla en multi-instancia?
11.3 Conectá con el módulo 10: ¿qué mecanismo de Redis usarías como "lock" para que solo una instancia ejecute el cron? Nombrá dos peligros del lock (expiración prematura y borrado inseguro) y cómo se mitigan.
11.4 ¿En qué se diferencia Pub/Sub de una cola como BullMQ, y por qué Pub/Sub NO sirve para procesar trabajos pero sí para propagar eventos en vivo?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Porque guarda los datos en RAM (memoria), no en disco como Postgres: el acceso a
    memoria es órdenes de magnitud más rápido. A cambio resignás durabilidad (es
    volátil por naturaleza), capacidad (la RAM es más cara/limitada) y el modelo
    relacional con transacciones y queries complejas.
1.2 No lo reemplaza, lo complementa. Postgres es la fuente de verdad (datos durables,
    relacionales); Redis es una capa veloz para lo efímero/cacheable (caché, sesiones,
    colas). Si Redis se pierde, se reconstruye desde Postgres.
1.3 En Redis: un caché del catálogo de productos o un refresh token (si se pierde, se
    reconstruye o se re-loguea). Jamás solo en Redis: los pedidos o los pagos de un
    usuario, porque perderlos es perder datos del negocio sin respaldo.
1.4 Redis procesa los comandos en un solo hilo, así que un comando lento bloquea a TODOS
    los clientes mientras corre. KEYS * recorre todo el keyspace (O(n)): en una base
    grande congela Redis por completo. En su lugar se usa SCAN, que itera de a cursores
    sin bloquear.
1.5 RDB son snapshots periódicos: tras un crash perdés lo escrito desde el último
    snapshot (segundos/minutos). AOF registra cada escritura en un log: con
    appendfsync everysec perdés como mucho ~1s, a costa de archivos más grandes y algo
    más lento. Para un caché casi no importa: si Redis se reinicia vacío, se reconstruye
    desde Postgres (el riesgo real es el cache stampede del módulo 4, no la pérdida).
1.6 Define qué hace Redis al llegar a maxmemory: allkeys-lru descarta las claves menos
    usadas recientemente para hacer lugar. Es la típica para caché porque mantiene lo
    caliente y tira lo frío, en vez de rechazar escrituras o morir por OOM cuando se llena.
```

### Módulo 2
```
2.1 EX 60 le pone un TTL: la clave se borra sola tras 60 segundos. Es la base del caché
    porque te da expiración automática (el dato cacheado caduca solo) sin tener que
    limpiarlo manualmente.
2.2 Para organizar el espacio plano de claves por feature/entidad y evitar colisiones
    (producto:7, usuario:42, sesion:abc). Es una convención de nombres, no una regla del
    motor, pero hace el conjunto de claves legible y predecible.
2.3 Porque INCR es atómico: incrementa sin condiciones de carrera. Leer-sumar-escribir
    desde tu código tiene una ventana donde dos requests leen el mismo valor y ambos
    escriben +1, perdiéndose un incremento (race condition).
2.4 TTL de una clave sin expiración devuelve -1 (y -2 si la clave no existe). El ejercicio
    muestra que INCR crea/incrementa la clave, EXPIRE le pone la ventana, y al pasar el
    TTL la clave desaparece sola (GET → (nil)).
```

### Módulo 3
```
3.1 (1) Miss: la clave no está en caché. (2) Vas a la fuente de verdad (Postgres) y
    traés el dato. (3) Lo guardás en el caché con un TTL y lo devolvés, para que las
    próximas lecturas sean hits.
3.2 Buen candidato: datos caros de obtener y que se leen mucho más de lo que cambian
    (catálogo, config, perfil). Mal candidato: datos que cambian a cada momento o que se
    leen una sola vez (ahí el caché agrega inconsistencia y complejidad sin beneficio).
3.3 La primera es un miss: paga el costo de la query a Postgres y guarda el resultado en
    caché. La segunda (dentro del TTL) es un hit: responde desde memoria en microsegundos
    sin tocar Postgres.
3.4 Sin store de Redis, el CacheModule usa la memoria de cada proceso: con 3 instancias
    tenés 3 cachés distintos. Los hits son inconsistentes (depende de a qué instancia
    caés) e invalidar en una no invalida en las otras. Contradice el stateless del
    módulo 5: el estado compartido (el caché) debe vivir afuera del proceso, en Redis.
3.5 (teclado) La 1ra petición (miss) muestra la latencia de Postgres (varios ms); la 2da
    (hit) responde en microsegundos sin tocar la base. Es la evidencia concreta del valor
    del caché — y conecta con "medí antes de optimizar".
```

### Módulo 4
```
4.1 Stale data es un dato desactualizado: el caché sirve una copia vieja después de que
    la fuente cambió. El caché lo provoca porque guarda una copia temporal que no se
    entera automáticamente de los cambios en Postgres.
4.2 TTL alcanza cuando tolerás cierta desactualización (un catálogo, un perfil: estar
    unos segundos viejo no es grave). Necesitás invalidación explícita cuando la frescura
    es crítica (un stock, un saldo: servir un valor viejo causa errores reales).
4.3 Al actualizar en Postgres, borrás (DEL) o actualizás su clave en el caché, para que
    la próxima lectura sea un miss y recachee el valor nuevo. Si no, seguirías sirviendo
    el valor viejo hasta que expire el TTL.
4.4 Cache stampede: cuando una clave caliente expira (o tras un restart/failover que
    vacía Redis), muchísimos requests sufren miss A LA VEZ y todos pegan a Postgres
    simultáneamente para reconstruir el mismo valor, pudiendo tumbarlo. Ocurre típicamente
    (a) al expirar una clave muy consultada y (b) tras un flush/restart de Redis.
    Mitigaciones: TTL con jitter, single-flight (un solo request recalcula con lock),
    early recompute (refrescar antes de que expire).
4.5 El jitter (TTL = base + random) hace que las claves no expiren todas en el mismo
    instante, así los misses se reparten en el tiempo en vez de concentrarse en un pico.
    NO resuelve el caso del restart de Redis que vacía TODO de golpe: ahí todas las claves
    nacen ausentes a la vez, sin jitter que valga; para eso sirve el single-flight/lock o
    el warming del caché.
```

### Módulo 5
```
5.1 Porque con varias instancias detrás de un balanceador, cada request puede caer en
    una distinta; si la sesión vive en la memoria de UNA, las demás no la tienen y el
    usuario "se desloguea" al azar. El estado en memoria del proceso no se comparte.
5.2 Porque es rápido (sesiones/tokens se chequean en cada request), tiene TTL automático
    (la sesión/token expira solo sin un cron de limpieza) y es un store externo
    compartido por todas las instancias. Postgres serviría pero es más lento y durable de
    lo necesario para algo efímero.
5.3 Para el logout, permite revocar el token borrándolo del store (un JWT stateless puro
    no se puede invalidar antes de expirar). Para escalar, al estar afuera del proceso,
    todas las instancias ven el mismo store y el login funciona caigas donde caigas.
5.4 Como el refresh token vive en Redis (no en el JWT), revocarlo es DEL de su clave: la
    próxima renovación falla al instante, algo que un access token stateless no permite
    hasta que expira. Se guarda el hash y no el token en claro para no exponer un secreto
    si Redis se filtra, y para detectar reuso/robo: si llega un refresh cuyo hash no
    coincide (o ya fue usado en la rotación), se revoca toda la familia de tokens.
```

### Módulo 6
```
6.1 Porque el límite debe ser global por cliente: si cada instancia cuenta en su propia
    memoria, con N instancias el cliente podría hacer N veces el límite. Un contador
    compartido en Redis hace que el límite se respete sin importar a qué instancia llegue.
6.2 EXPIRE define la ventana de tiempo: cuando expira la clave, el contador se reinicia
    solo. Sin él, el contador crecería para siempre y, tras llegar al límite una vez, el
    cliente quedaría bloqueado permanentemente.
6.3 La fuerza bruta / credential stuffing: limitar los intentos de login por IP/usuario
    impide probar miles de contraseñas. (Visto en el módulo de auth.)
6.4 INCR y EXPIRE son dos comandos separados; no son atómicos juntos. Si el proceso muere
    (o la conexión se corta) justo después del INCR que crea la clave y antes del EXPIRE,
    la clave queda SIN TTL para siempre: el contador no se reinicia nunca y ese cliente
    queda bloqueado de forma permanente. Un script Lua ejecuta INCR+EXPIRE como una sola
    operación atómica (el hilo único de Redis no intercala nada en el medio), así nunca
    queda una clave sin expiración.
6.5 La ventana fija permite hasta el doble del límite en el borde: un cliente puede gastar
    el límite completo al final de una ventana y otro completo al inicio de la siguiente
    (ej. 10 en el seg 59 + 10 en el 61 = 20 en ~2s). Alternativas: sliding window con un
    sorted set (ZADD por timestamp, ZREMRANGEBYSCORE para purgar, ZCARD para contar) o
    token bucket (un balde que se rellena a ritmo fijo y permite ráfagas controladas).
```

### Módulo 7
```
7.1 (1) El usuario espera 1-2s a que se envíe el email para recibir su respuesta. (2) Si
    el envío falla, no sabés si fallar el registro (que sí funcionó) o no. (3) Trabajo
    pesado/CPU-bound bloquearía o colgaría el request (y el event loop).
7.2 El request solo encola el trabajo y responde al instante (no espera). El worker, en
    un proceso aparte, toma el trabajo de la cola y lo procesa en background a su ritmo,
    con reintentos. El usuario no espera; el worker hace el trabajo lento.
7.3 Porque mover el trabajo CPU-bound a una cola lo saca del hilo que atiende requests:
    el worker (otro proceso) lo procesa sin bloquear el event loop de la API, que sigue
    respondiendo. Es el "mandalo a una cola" del módulo de Node.
```

### Módulo 8
```
8.1 El productor (Queue.add) encola un trabajo con sus datos y sigue. El consumidor
    (Worker / @Processor) toma los trabajos de la cola y los ejecuta en background.
8.2 BullMQ marca el job como fallido y aplica la política de reintentos configurada
    (attempts + backoff); si agota los intentos, queda como failed (y puede ir a una DLQ).
8.3 Porque podés escalar el worker independientemente de la API (más workers si hay
    mucho trabajo en background, sin tocar la API) y aislar fallos. Es posible porque el
    estado (la cola) vive en Redis, no en el proceso: API y worker son stateless y se
    coordinan vía Redis.
8.4 (teclado) El productor (Queue.add) retorna casi al instante: no espera a que el job
    se procese. El log del worker aparece ~2s después y en otro proceso, demostrando que
    el trabajo corre en background, desacoplado del request y escalable por separado.
```

### Módulo 9
```
9.1 Para no generar una "tormenta de reintentos": si mil trabajos fallan y todos
    reintentan a la vez e inmediato, saturan al servicio que justo se estaba recuperando.
    El backoff (1s, 2s, 4s...) con jitter espacia los reintentos y le da aire.
9.2 Es una cola aparte adonde van los trabajos que fallaron tras agotar los reintentos.
    Resuelve que un trabajo "envenenado" (que falla siempre) no se reintente para siempre
    bloqueando la cola: queda apartado para inspección manual sin frenar el resto. Con
    BullMQ muchas veces no necesitás una cola separada: los jobs que agotan los intentos
    quedan retenidos en el estado `failed` (se inspeccionan con queue.getFailed() y se
    reintentan con job.retry()). Una DLQ separada se justifica si querés un flujo distinto
    para los muertos (alertas, otro worker, retención larga).
9.3 Un timeout suele ser transitorio (el servicio estaba lento un instante): reintentar
    puede funcionar. Un email con formato inválido es un error determinista de negocio:
    va a fallar igual en cada intento, así que reintentar solo gasta recursos; ese debe
    fallar rápido y quedar en `failed`.
9.4 (teclado) En los logs ves los 3 intentos espaciándose por el backoff exponencial
    (~1s, 2s, 4s); tras agotarlos el job no se pierde: queda en estado failed y aparece
    en await queue.getFailed() con su failedReason.
```

### Módulo 10
```
10.1 Porque las colas entregan "al menos una vez" y los reintentos reejecutan trabajos:
     un job se puede procesar más de una vez (ej. si el worker muere tras hacer el trabajo
     pero antes de marcarlo completado). Idempotente = reejecutar no causa daño.
10.2 Se rompe si no es idempotente: cobrar un pago o enviar un email (se duplicaría).
     Naturalmente idempotente: marcar una tarea como completada (ponerla en true dos veces
     da el mismo resultado).
10.3 Con SET clave "1" EX <ttl> NX: el NX hace que la clave se ponga solo si no existía,
     de forma atómica. El primer intento la crea y procesa; los reintentos ven la clave ya
     puesta y salen sin reejecutar. En ioredis ese SET ... NX NO devuelve un booleano:
     devuelve "OK" si seteó la clave, o null si ya existía. Por eso se chequea
     `if (resultado === null) return;`.
10.4 Porque entre hacer el efecto y confirmar (ack) que se hizo siempre hay un instante
     donde el proceso puede morir, y dos máquinas no pueden acordar con certeza si un
     mensaje llegó cuando la red/los procesos fallan (problema de los dos generales). En
     la práctica se usa at-least-once + workers idempotentes, que produce el efecto
     observable de "exactamente una vez".
10.5 Si seteás la clave de idempotencia ANTES de cobrar y morís en el medio, el reintento
     ve la clave puesta y NO cobra nunca (pérdida del efecto). Si cobrás PRIMERO y morís
     antes de setear, el reintento cobra de nuevo (doble cobro). No hay orden perfecto con
     un flag en Redis; para un pago lo robusto es mandar una idempotency key al gateway,
     que garantiza no cobrar dos veces la misma key en el sistema que mueve el dinero.
```

### Módulo 11
```
11.1 Las 3 instancias dispararían el mismo cron a la vez → el trabajo se ejecuta 3 veces
     (emails triplicados, etc.). La opción recomendada es BullMQ con jobs repetibles
     (repeat): se coordinan vía Redis y solo un worker toma cada ejecución.
11.2 Para una tarea simple, @nestjs/schedule con @Cron. Para coordinarla en
     multi-instancia, BullMQ con repeat (recomendado) o, en su defecto, un lock en Redis,
     de modo que solo una instancia ejecute cada disparo.
11.3 SET NX con TTL: la instancia que logra poner la clave (gana el lock) ejecuta; las
     demás ven que ya existe y se abstienen. Dos peligros: (a) expiración prematura — si
     el trabajo dura más que el TTL del lock, este expira y otra instancia ejecuta en
     paralelo (doble ejecución); se mitiga dando margen al TTL o renovándolo. (b) borrado
     inseguro — un DEL a ciegas puede borrar el lock de otra instancia; se mitiga
     guardando un token único y liberando solo si el token sigue siendo el nuestro, con un
     script Lua atómico. Aun así no garantiza exclusión bajo pausas de GC/relojes
     desfasados (crítica de Kleppmann a Redlock; la exclusión correcta exige fencing
     tokens). Por eso, para un cron, mejor BullMQ repeat.
11.4 Pub/Sub es fire-and-forget: publicás en un canal y los suscriptores conectados EN ESE
     momento lo reciben; no persiste ni reintenta, si nadie escucha se pierde. Una cola
     (BullMQ) persiste el trabajo, lo entrega al menos una vez, reintenta y un solo worker
     lo procesa. Por eso Pub/Sub no sirve para trabajos (perderías jobs) pero sí para
     propagar eventos en vivo entre instancias (ej. broadcast de WebSockets).
```

---

## Siguientes pasos

Con este módulo sumaste la capa que acelera lecturas, comparte estado entre instancias y procesa trabajo pesado fuera del request: las tres caras de Redis. Y cerraste varios hilos que venías tirando — el "stateless" del escalado, el "no bloquees el event loop", el store de refresh tokens, y la resiliencia (retry/backoff, DLQ, idempotencia) del archivo senior, ahora con código concreto. El recorrido desde acá: **(1)** implementá el **"Notification/Jobs"** del roadmap: encolá emails con reintentos y cacheá tus endpoints más pesados; **(2)** mové el store de refresh tokens de tu auth a Redis; **(3)** el último gran tema del temario es **tiempo real (WebSockets)** + documentación de API con **Swagger** — el cierre del círculo full stack conectando tu frontend React/React Native a un backend con features en vivo. Con DB, auth, tests, deploy y Redis, ya tenés un backend que sostiene un producto real.
