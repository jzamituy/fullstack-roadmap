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
npm i bullmq @nestjs/bullmq ioredis @nestjs/cache-manager cache-manager
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

**Ejercicios 1**
1.1 ¿Por qué Redis es mucho más rápido que Postgres para leer un valor? ¿Qué resignás a cambio?
1.2 ¿Redis reemplaza a tu base de datos relacional? Explicá la relación entre ambos.
1.3 Dá un ejemplo de dato que pondrías en Redis y uno que jamás pondrías solo en Redis. Justificá.

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

**Ejercicios 2**
2.1 ¿Qué hace `SET clave valor EX 60` que un `SET clave valor` no hace? ¿Por qué es la base del caché?
2.2 ¿Para qué sirve la convención de prefijos con `:` en las claves (ej. `producto:7`)?
2.3 ¿Por qué `INCR` es mejor que leer un contador, sumarle 1 en tu código y volver a escribirlo?

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

**Ejercicios 3**
3.1 Describí los tres pasos del patrón cache-aside ante un cache miss.
3.2 ¿Qué tipo de dato es buen candidato para cachear y cuál no? Dá un ejemplo de cada uno.
3.3 ¿Qué pasaría en la primera petición a un producto que no está cacheado vs. en la segunda (dentro del TTL)?

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

**Ejercicios 4**
4.1 ¿Qué es "stale data" y por qué el caché lo provoca?
4.2 ¿Cuándo alcanza con TTL y cuándo necesitás invalidación explícita? Dá un ejemplo de cada uno.
4.3 Escribí el patrón: al actualizar un producto en la base, ¿qué hacés con su clave en el caché y por qué?

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

**Ejercicios 5**
5.1 ¿Por qué guardar sesiones en la memoria del proceso rompe el escalado horizontal?
5.2 ¿Por qué Redis (y no Postgres) es el lugar típico para sesiones y refresh tokens? Mencioná el TTL.
5.3 Conectá con auth: ¿qué ventaja da que el store de refresh tokens esté en Redis y no en memoria, para el logout y para escalar?

---

## Módulo 6 — Rate limiting con Redis

**Teoría.** El **rate limiting** (limitar cuántos requests puede hacer un cliente en una ventana de tiempo) protege de abuso y de fuerza bruta — lo viste como medida de seguridad en el módulo de auth. La implementación natural usa Redis porque necesita un **contador compartido entre instancias** (si cada instancia cuenta por su lado, el límite real es N veces mayor) y con **expiración automática**.

La idea base, con `INCR` + `EXPIRE`:

```ts
async permitir(ip: string, limite = 10, ventanaSeg = 60): Promise<boolean> {
  const clave = `rate:${ip}`;
  const actual = await this.redis.incr(clave);  // suma 1 atómicamente
  if (actual === 1) {
    await this.redis.expire(clave, ventanaSeg);  // primera vez: arranca la ventana
  }
  return actual <= limite;                        // ¿superó el límite?
}
```

La primera petición de la ventana crea el contador y le pone un TTL; las siguientes lo incrementan; cuando pasa la ventana, la clave expira y el contador se reinicia solo. Como `INCR` es atómico y la clave es compartida, el límite se respeta aunque tengas muchas instancias.

En NestJS no hace falta escribir esto a mano: el **`ThrottlerModule`** (del módulo de auth) lo hace, y se le puede configurar Redis como almacén para que funcione bien en entornos multi-instancia. Pero entender la mecánica con `INCR`/`EXPIRE` es lo que te deja explicarlo y diagnosticarlo.

**Ejercicios 6**
6.1 ¿Por qué el rate limiting necesita un contador en Redis y no en la memoria de cada instancia?
6.2 ¿Qué rol cumple el `EXPIRE` en la implementación con `INCR`? ¿Qué pasaría sin él?
6.3 ¿Qué ataque (visto en el módulo de auth) ayuda a frenar el rate limiting en el endpoint de login?

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

const connection = { host: "localhost", port: 6379 };

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

Pero, ¿qué pasa si un trabajo falla **siempre** (un email malformado, un bug)? No puede reintentarse para siempre bloqueando recursos. Tras agotar los `attempts`, el job queda marcado como **failed**, y el patrón es enviarlo a una **dead-letter queue (DLQ)**: un apartado donde queda para inspección manual sin frenar el resto de la cola. Es la "red de seguridad del procesamiento asíncrono" del senior.

```ts
new Worker("emails", procesar, { connection })
  .on("failed", async (job, err) => {
    if (job && job.attemptsMade >= (job.opts.attempts ?? 1)) {
      await deadLetterQueue.add("email-fallido", { original: job.data, error: err.message });
      // queda para revisión; la cola principal sigue fluida
    }
  });
```

La regla del senior aplicada: reintentá solo fallos **transitorios** (timeouts, 5xx), nunca errores de negocio (un email inválido no mejora reintentando — ese va directo a la DLQ); y asegurá que el trabajo sea **idempotente** (módulo 10), porque los reintentos lo van a ejecutar más de una vez.

**Ejercicios 9**
9.1 ¿Por qué los reintentos usan backoff exponencial y no reintentan inmediato? (Recordá la "tormenta de reintentos" del senior.)
9.2 ¿Qué es una dead-letter queue y qué problema resuelve?
9.3 ¿Por qué tiene sentido reintentar un timeout pero NO un "email con formato inválido"?

---

## Módulo 10 — Idempotencia en los workers

**Teoría.** Como los reintentos (módulo 9) y la entrega "al menos una vez" de las colas hacen que un trabajo se ejecute **más de una vez**, los workers **deben ser idempotentes**: ejecutar el mismo trabajo dos veces produce el mismo resultado que ejecutarlo una. Es el concepto del archivo senior, ahora en el lugar donde más importa. Si "procesar pago" no es idempotente y el job se reintenta, **cobrás dos veces**.

El caso peligroso: el worker hace el trabajo, pero **muere antes de marcar el job como completado** → la cola cree que falló → lo reintenta → se ejecuta de nuevo. No podés evitar el reintento; tenés que hacer que **reejecutar sea inofensivo**.

La técnica (la misma del senior): una **clave de idempotencia** que registrás; si el trabajo ya se procesó, no lo repetís.

```ts
async process(job: Job<{ pedidoId: string }>): Promise<void> {
  const clave = `procesado:pago:${job.data.pedidoId}`;

  // SET NX = "set if not exists": pone la clave solo si no estaba. Atómico.
  const esNuevo = await this.redis.set(clave, "1", "EX", 86400, "NX");
  if (!esNuevo) return; // ya se procesó este pago: no cobramos de nuevo

  await this.gateway.cobrar(job.data.pedidoId);
}
```

El `SET ... NX` de Redis es atómico: garantiza que solo **un** intento "gana" y procesa; los reintentos ven la clave ya puesta y salen sin reejecutar. Para operaciones que ya son naturalmente idempotentes (marcar una tarea como completada: ponerla en `true` dos veces da lo mismo) no hace falta nada extra; el cuidado es para las que tienen efectos acumulativos (cobrar, enviar, incrementar).

**Ejercicios 10**
10.1 ¿Por qué un worker de cola DEBE ser idempotente? ¿Qué propiedad de las colas lo obliga?
10.2 Dá un ejemplo de operación que se rompe si no es idempotente y una que ya lo es naturalmente.
10.3 ¿Cómo usás `SET NX` con TTL en Redis para evitar procesar dos veces el mismo trabajo?

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

Pero **cuidado con el cron en multi-instancia**: si corrés 3 instancias de tu API, las 3 dispararían el mismo cron a las 3 AM → el trabajo se ejecuta 3 veces. Soluciones: usar **BullMQ con jobs repetibles** (`repeat`), que se coordinan vía Redis y garantizan que solo un worker lo tome; o un **lock distribuido** en Redis (`SET NX`, como la idempotencia del módulo 10) para que solo una instancia ejecute. Este es un error clásico que separa a quien probó algo en local de quien lo operó escalado.

Para el caché en Nest, el **`CacheModule`** integra Redis de forma declarativa, e incluso podés cachear endpoints enteros con un interceptor (`@UseInterceptors(CacheInterceptor)`), aunque para invalidación fina (módulo 4) conviene el control manual con el patrón cache-aside.

**Ejercicios 11**
11.1 ¿Qué problema aparece si tenés un `@Cron` y escalás tu API a 3 instancias? ¿Cómo lo resolvés?
11.2 ¿Qué herramienta de Nest usarías para una tarea programada simple, y qué para coordinarla en multi-instancia?
11.3 Conectá con el módulo 10: ¿qué mecanismo de Redis usarías como "lock" para que solo una instancia ejecute el cron?

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
```

### Módulo 9
```
9.1 Para no generar una "tormenta de reintentos": si mil trabajos fallan y todos
    reintentan a la vez e inmediato, saturan al servicio que justo se estaba recuperando.
    El backoff (1s, 2s, 4s...) con jitter espacia los reintentos y le da aire.
9.2 Es una cola aparte adonde van los trabajos que fallaron tras agotar los reintentos.
    Resuelve que un trabajo "envenenado" (que falla siempre) no se reintente para siempre
    bloqueando la cola: queda apartado para inspección manual sin frenar el resto.
9.3 Un timeout suele ser transitorio (el servicio estaba lento un instante): reintentar
    puede funcionar. Un email con formato inválido es un error determinista de negocio:
    va a fallar igual en cada intento, así que reintentar solo gasta recursos; ese va a
    la DLQ.
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
     puesta y salen sin reejecutar.
```

### Módulo 11
```
11.1 Las 3 instancias dispararían el mismo cron a la vez → el trabajo se ejecuta 3 veces
     (emails triplicados, etc.). Se resuelve con BullMQ jobs repetibles (se coordinan vía
     Redis y solo un worker toma cada ejecución) o un lock distribuido en Redis.
11.2 Para una tarea simple, @nestjs/schedule con @Cron. Para coordinarla en
     multi-instancia, BullMQ con repeat (o un lock en Redis), de modo que solo una
     instancia ejecute cada disparo.
11.3 SET NX con TTL: la instancia que logra poner la clave (gana el lock) ejecuta; las
     demás ven que ya existe y se abstienen. El TTL evita que el lock quede tomado para
     siempre si la instancia que lo tiene muere.
```

---

## Siguientes pasos

Con este módulo sumaste la capa que acelera lecturas, comparte estado entre instancias y procesa trabajo pesado fuera del request: las tres caras de Redis. Y cerraste varios hilos que venías tirando — el "stateless" del escalado, el "no bloquees el event loop", el store de refresh tokens, y la resiliencia (retry/backoff, DLQ, idempotencia) del archivo senior, ahora con código concreto. El recorrido desde acá: **(1)** implementá el **"Notification/Jobs"** del roadmap: encolá emails con reintentos y cacheá tus endpoints más pesados; **(2)** mové el store de refresh tokens de tu auth a Redis; **(3)** el último gran tema del temario es **tiempo real (WebSockets)** + documentación de API con **Swagger** — el cierre del círculo full stack conectando tu frontend React/React Native a un backend con features en vivo. Con DB, auth, tests, deploy y Redis, ya tenés un backend que sostiene un producto real.
