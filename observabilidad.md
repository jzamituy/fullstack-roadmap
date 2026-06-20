# Observabilidad práctica: ver qué hace tu sistema en producción

**Logs, métricas y traces con OpenTelemetry · viniendo de backend Node/NestJS · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya tenés una API en producción —la **Task API** del roadmap (usuarios ↔ proyectos ↔ tareas en NestJS)—, que conocés [Docker y deploy](docker-deploy.md), [Kubernetes](kubernetes.md), [AWS](aws.md) y la [arquitectura event-driven](event-driven.md) (varios servicios hablando entre sí). También conecta de forma directa con el [módulo 8 de Evaluations](evals.md): el **tracing de un sistema de IA es exactamente el mismo concepto** que vas a ver acá, aplicado a microservicios. La observabilidad es lo que te deja **entender qué está pasando dentro de un sistema que no podés pausar ni adjuntarle un debugger**: producción. El objetivo es doble: **los tres pilares** (logs, métricas, traces) y **el criterio** —porque el error caro acá no es no tener dashboards, es ahogarte en datos que no responden ninguna pregunta, o pagar una fortuna por telemetría que nadie mira.

**Lo que asumimos.** Una API Node/NestJS, HTTP, async (event loop, del módulo de Node), nociones de microservicios (event-driven), Docker/K8s, y la idea de "medí antes de optimizar" que apareció en [NestJS senior](nestjs-senior.md) y [Redis](redis.md). No asumimos experiencia previa con Prometheus, Grafana ni OpenTelemetry.

**Para practicar.** El stack open source típico: un collector y backends locales con Docker.

```bash
# OpenTelemetry: el SDK vendor-neutral para instrumentar tu app
npm i @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node \
      @opentelemetry/exporter-trace-otlp-http @opentelemetry/exporter-prometheus

# Backends locales para ver la telemetría (uno de muchos stacks posibles):
#   - Prometheus  → métricas
#   - Grafana     → dashboards
#   - Jaeger/Tempo → traces
# Se levantan con docker compose; ver módulo 5.
```

> Nota sobre datos volátiles: los nombres de paquetes OTel, las versiones del SDK y qué backends están vigentes (Jaeger, Tempo, Grafana, Datadog, CloudWatch/X-Ray) cambian rápido. Verificá contra `opentelemetry.io` al implementar.

**Índice de módulos**
1. Qué es observabilidad (y por qué no es lo mismo que monitoreo)
2. Logs estructurados: de `console.log` a JSON con correlación
3. Métricas: RED, USE y los golden signals
4. Tracing distribuido: el trace, los spans y la propagación de contexto
5. OpenTelemetry: el estándar vendor-neutral
6. Instrumentar tu API NestJS con OTel (hands-on)
7. Correlacionar los tres pilares
8. SLI, SLO y error budgets
9. Alerting: alertá sobre síntomas, no sobre causas
10. Cardinalidad y el costo de la observabilidad
11. El criterio: cuánta observabilidad, cuándo y para qué

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es observabilidad (y por qué no es lo mismo que monitoreo)

**Teoría.** Tu Task API funciona perfecto en tu máquina y en los tests. La deployás. A las tres semanas, un usuario reporta que "crear una tarea a veces tarda 8 segundos". No siempre. No para todos. No podés reproducirlo. No podés adjuntar un debugger a producción ni meter un `breakpoint`. ¿Cómo averiguás qué pasa? Esa pregunta —**entender el comportamiento interno de un sistema desde afuera, a partir de las señales que emite**— es la observabilidad.

La distinción clave, y pregunta de entrevista segura: **monitoreo vs. observabilidad.**

- **Monitoreo** responde preguntas que ya sabías hacer de antemano: "¿el CPU pasó del 80%?", "¿la API está caída?". Vigilás un conjunto **conocido y finito** de métricas con umbrales. Esto cubre los **known-unknowns**: sabés qué cosas pueden fallar, solo no sabés cuándo.
- **Observabilidad** te deja responder preguntas que **no anticipaste**, sin tener que deployar código nuevo para investigar: "¿por qué las requests del usuario 4521, al proyecto X, entre las 3 y las 4 PM, desde la región de Brasil, tardan 8 segundos?". Eso es un **unknown-unknown**: ni sabías que esa combinación existía como problema. Un sistema es observable si, con la telemetría que ya emite, podés contestar ese tipo de pregunta nueva.

La forma más citada de decirlo: **monitoreo es para los known-unknowns; observabilidad es para los unknown-unknowns.** El monitoreo es un subconjunto de la observabilidad —seguís necesitando tus alarmas de "API caída"—, pero un sistema distribuido moderno (microservicios, colas, K8s) tiene tantos estados posibles que es imposible anticiparlos todos con dashboards fijos. Necesitás poder **explorar**.

Esa capacidad se construye sobre **tres pilares** —tres tipos de señal que tu sistema emite—:

- **Logs**: eventos discretos con detalle ("se creó la tarea 99 para el usuario 4521", "falló la conexión a Postgres"). El *qué pasó*, con contexto.
- **Métricas**: números agregados en el tiempo ("450 requests/min", "p99 de latencia = 1.2s", "85% de uso de memoria"). El *cuánto / qué tan rápido*, baratos de almacenar y agregar.
- **Traces**: el recorrido completo de **una** request a través de todos los servicios que tocó. El *por dónde pasó y dónde se fue el tiempo*.

Cada pilar responde una pregunta distinta y los tres se potencian (módulo 7). La frase mental: **no podés arreglar —ni siquiera entender— lo que no podés ver. La observabilidad es la parte del sistema que te deja verlo.** Es exactamente el mismo principio que en [Evaluations módulo 8](evals.md): un sistema de IA sin tracing es indebuggeable; un sistema distribuido sin observabilidad, también.

**Ejercicios 1**
1.1 Definí con tus palabras la diferencia entre monitoreo y observabilidad usando los términos *known-unknowns* y *unknown-unknowns*.
1.2 Para cada pregunta, decí qué pilar la responde mejor: (a) "¿cuántas requests por segundo recibimos?"; (b) "¿qué le pasó exactamente a la request que falló del usuario 4521?"; (c) "¿qué mensaje de error tiró la conexión a la DB a las 15:42?".
1.3 Tu jefe dice "ya tenemos monitoreo: una alarma si el CPU pasa el 90%. Con eso alcanza". ¿Qué tipo de problema te va a quedar invisible y por qué?

---

## Módulo 2 — Logs estructurados: de `console.log` a JSON con correlación

**Teoría.** El log es el pilar más viejo y el más fácil de hacer mal. El error universal es el **log no estructurado**: texto libre.

```ts
// ❌ Log no estructurado: legible para un humano, inútil para una máquina
console.log(`Usuario ${userId} creó la tarea ${taskId} en ${ms}ms`);
// → "Usuario 4521 creó la tarea 99 en 8200ms"
```

En tu máquina, leyendo una línea, está bien. En producción tenés **millones de líneas** de docenas de instancias mezcladas. Querés preguntar: "mostrame todas las creaciones de tarea que tardaron más de 5s, del usuario 4521". Con texto libre, eso es parsing frágil con regex. La solución es el **log estructurado**: cada log es un objeto (típicamente **JSON**) con campos consultables.

```ts
// ✅ Log estructurado: una línea de JSON por evento
logger.info({
  event: 'task.created',
  userId: 4521,
  taskId: 99,
  projectId: 7,
  durationMs: 8200,
});
// → {"level":"info","event":"task.created","userId":4521,"taskId":99,...,"ts":"..."}
```

Ahora un sistema de logs (Loki, CloudWatch Logs, Elasticsearch) puede indexar esos campos y vos preguntás `event="task.created" AND durationMs > 5000 AND userId=4521`. El log dejó de ser prosa y pasó a ser **datos**.

En Node, **no uses `console.log`** para esto: es síncrono y bloquea el event loop (acordate de las fases del event loop del módulo de Node). Usá un logger pensado para producción —**Pino** es el estándar por rápido y JSON-first; NestJS lo integra vía `nestjs-pino`—.

> **Matiz de producción que casi nadie cuenta.** Pino es rápido porque sus **transports** (el formateo/escritura pesada) corren en un **worker thread** aparte, en vez de bloquear el hilo principal. La contracara: si el proceso muere de golpe (`process.exit`, un crash, un `SIGKILL`), **los últimos logs que quedaron en el buffer se pierden** — justo los que más querés cuando algo explotó. Por eso en producción se hace *flush* en el shutdown (`pino.final` / cerrar el logger en los handlers de `SIGTERM`/`SIGINT`, que conecta con el graceful shutdown del módulo de Docker). Y en contenedores, el patrón es **loguear a `stdout`** y dejar que el runtime (Docker/K8s) recolecte, en vez de escribir a archivos: el contenedor es efímero.

```ts
// main.ts — Pino como logger de la app, JSON en prod, lindo en dev
import { Logger } from 'nestjs-pino';
const app = await NestFactory.create(AppModule, { bufferLogs: true });
app.useLogger(app.get(Logger));
```

Dos prácticas que separan un log amateur de uno profesional:

1. **Niveles bien usados.** `error` (algo falló y requiere atención), `warn` (algo raro pero recuperable), `info` (eventos de negocio: se creó una tarea), `debug` (detalle para diagnóstico, apagado en prod). En prod corrés con nivel `info`; podés subir a `debug` temporalmente sin redeploy si tu logger lo permite.

2. **Correlación: el `trace_id` / `request_id`.** Una sola request HTTP genera muchos logs (entró, validó, consultó la DB, publicó un evento, respondió). Si cada log lleva el **mismo identificador de correlación**, podés reconstruir la historia completa de esa request filtrando por ese ID. Sin él, los logs de requests concurrentes están entrelazados y no sabés cuál es cuál. Ese ID es el puente con el tracing (módulo 4): el `trace_id` es lo que une los tres pilares.

```ts
// Middleware: genera/propaga un request-id y lo mete en cada log de la request
@Injectable()
export class CorrelationMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const id = req.headers['x-request-id'] ?? randomUUID();
    res.setHeader('x-request-id', id);
    // El logger toma este id del contexto (AsyncLocalStorage) y lo agrega a cada línea
    next();
  }
}
```

**Regla de oro: nunca loguees secretos ni PII en claro** (passwords, tokens JWT, números de tarjeta, emails según tu marco regulatorio). Los logs se van a un sistema central que mucha gente puede leer y que retenés meses. Conecta con [Autenticación](autenticacion.md): un token en un log es un token filtrado.

**Ejercicios 2**
2.1 Reescribí este log no estructurado como log estructurado con campos: `console.log("Error al crear tarea para user " + userId + ": " + err.message)`.
2.2 ¿Por qué `console.log` es mala idea para logs de alto volumen en Node? (pensá en el event loop)
2.3 ¿Qué problema concreto resuelve el `request_id` / `trace_id` en los logs, y por qué es imposible de resolver sin él cuando hay tráfico concurrente?
2.4 Nombrá dos cosas que **nunca** deberían aparecer en un log.
2.5 Pino usa un worker thread para sus transports. ¿Qué riesgo aparece si el proceso muere de golpe, y qué hacés en el shutdown para mitigarlo? ¿Por qué en un contenedor logueás a stdout y no a un archivo?

---

## Módulo 3 — Métricas: RED, USE y los golden signals

**Teoría.** Un log es un evento; una **métrica** es un número agregado a lo largo del tiempo. Loguear cada request es caro y verboso; contar "requests por segundo" es barato y te da la foto de salud al instante. Las métricas son baratas de almacenar (un número cada N segundos, no un objeto por evento) y la base de dashboards y alertas.

El problema del principiante es medir **todo** y no saber mirar nada. Por eso hay **marcos** que te dicen *qué* medir:

- **RED** (para servicios que atienden requests, como tu API): **R**ate (requests/seg), **E**rrors (cuántas fallan), **D**uration (cuánto tardan, mirando percentiles). Es la vista del **usuario**: ¿llegan, fallan, son lentas?
- **USE** (para recursos: CPU, disco, memoria, pool de conexiones): **U**tilization (% en uso), **S**aturation (cuánto trabajo encolado/esperando), **E**rrors. Es la vista de la **máquina**.
- **Golden signals** (de Google SRE): **latencia, tráfico, errores y saturación**. Es prácticamente RED + saturación; el set mínimo para cualquier servicio.

**Percentiles, no promedios.** El error más caro en latencia: mirar el **promedio**. Si 99 requests tardan 50ms y 1 tarda 10s, el promedio (≈150ms) miente: oculta que tenés usuarios sufriendo. Por eso se miran **percentiles**: el **p99** ("el 99% de las requests fue más rápido que esto") expone justo esa cola lenta. La regla: **alertá y mirá p95/p99, no el promedio.** Esto es el "medí antes de optimizar" de [NestJS senior](nestjs-senior.md) hecho operacional.

Tipos de métrica que vas a usar:

- **Counter**: solo sube (total de requests, total de errores). Mirás su *tasa de cambio*.
- **Gauge**: sube y baja (conexiones activas, uso de memoria, tamaño de una cola — acordate de las colas de [Redis/BullMQ](redis.md)).
- **Histogram**: distribución de valores en buckets (latencias). De acá salen los percentiles.

**Prometheus** es el estándar de facto open source para métricas. Su modelo es **pull**: Prometheus *scrapea* (consulta periódicamente) un endpoint `/metrics` que tu app expone. Las métricas llevan **labels** (dimensiones) para poder filtrar/agrupar:

```ts
// prom-client: exponer métricas Prometheus desde Node
import { Counter, Histogram } from 'prom-client';

const httpRequests = new Counter({
  name: 'http_requests_total',
  help: 'Total de requests HTTP',
  labelNames: ['method', 'route', 'status'], // dimensiones
});

const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Latencia de requests HTTP',
  labelNames: ['method', 'route'],
  buckets: [0.05, 0.1, 0.3, 0.5, 1, 3, 8], // los buckets definen tus percentiles
});

// En un interceptor, por cada request:
httpRequests.inc({ method: 'POST', route: '/tasks', status: '201' });
httpDuration.observe({ method: 'POST', route: '/tasks' }, durationSeconds);
```

Y se consulta con **PromQL**:

```promql
# Tasa de errores 5xx sobre el total, últimos 5 min (la "E" de RED)
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))

# p99 de latencia de POST /tasks (la "D" de RED)
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{route="/tasks"}[5m])) by (le))
```

**Por qué esa query suma los buckets *antes* de calcular el cuantil.** Esto no es un detalle de sintaxis: es *la* razón por la que los percentiles se hacen así. **Los percentiles no se promedian** — el p99 de 3 instancias no es el promedio de sus p99. Por eso el histograma de Prometheus guarda **buckets** (conteos acumulados por umbral de latencia), y la query primero `sum(...) by (le)` **suma los buckets de todas las instancias** y *recién después* `histogram_quantile` estima el percentil sobre esa distribución agregada. Sumás conteos (que sí son agregables), no percentiles (que no lo son).

La contracara, que un senior conoce: los **buckets son fijos y se definen a priori** (los `[0.05, ..., 8]` de arriba). Eso trae dos límites: (1) si tu p99 real cae **fuera del último bucket** (>8s), `histogram_quantile` no puede estimarlo bien y lo topa; (2) dentro de un bucket interpola linealmente, así que la precisión depende de qué tan finos sean los buckets donde está tu cola. Alternativas: **native histograms** (Prometheus 2.40+/3.x) usan buckets exponenciales automáticos de alta resolución, sin elegirlos a mano; y los **summaries** calculan cuantiles exactos *en el cliente* pero **no son agregables entre instancias** (no podés combinar el p99 de varias) — por eso para algo distribuido se prefieren histogramas.

Cuidado con los `labels`: cada combinación de valores crea una **serie temporal** nueva. Un label con muchos valores posibles (ej. `userId`) hace explotar el costo —es el problema de **cardinalidad** del módulo 10—. Por eso `route` (decenas de valores) sí es label, `userId` (millones) **no**.

**Ejercicios 3**
3.1 Asociá cada marco con su pregunta: RED, USE, golden signals. ¿Cuál mirás para tu API y cuál para el uso de CPU del nodo?
3.2 Explicá con un ejemplo numérico por qué el promedio de latencia engaña y el p99 no.
3.3 Clasificá como counter, gauge o histogram: (a) total de tareas creadas; (b) cantidad de jobs esperando en la cola de BullMQ; (c) latencias de la consulta a Postgres.
3.4 ¿Por qué `route` puede ser un label pero `userId` no? (adelantás el módulo 10)
3.5 ¿Por qué la query del p99 hace `sum(...) by (le)` *antes* de `histogram_quantile`, y no al revés? (pista: qué se puede agregar y qué no)
3.6 (PromQL) Escribí la query del **p95** de latencia de `GET /tasks` agrupado por ruta. Si el resultado da 0.42, ¿se cumple un SLO de "p95 < 300ms"?

---

## Módulo 4 — Tracing distribuido: el trace, los spans y la propagación de contexto

**Teoría.** Logs y métricas te dicen *qué* y *cuánto*, pero no *por dónde pasó una request ni dónde se fue el tiempo*. En un monolito chico no importa tanto. Pero tu sistema event-driven tiene varios servicios: el request entra a la API, que llama a un servicio de proyectos, que consulta Postgres, que publica un evento a una cola, que dispara un worker. Cuando "crear tarea tarda 8s", **¿en cuál de esos saltos se fue el tiempo?** El **tracing distribuido** lo responde.

Las unidades —idénticas a las del [módulo 8 de Evaluations](evals.md), donde un trace de IA descompone llamadas al LLM, retrievals y tool calls—:

- **Trace** (traza): el registro completo de **una** request a través de **todo** el sistema. Tiene un **`trace_id`** único.
- **Span**: cada operación individual dentro del trace (la llamada HTTP entrante, la query a la DB, la publicación a la cola, el trabajo del worker). Cada span tiene nombre, **timestamp de inicio y fin** (de ahí su duración), atributos (`db.statement`, `http.status_code`) y un **`span_id`**.
- Los spans forman un **árbol**: cada uno apunta a su **padre** (`parent_span_id`). El span raíz es la request entrante; sus hijos son las operaciones que disparó. Visualizado, es un diagrama de cascada (waterfall) donde *ves* qué corrió en serie, qué en paralelo y dónde está la barra larga que se comió los 8 segundos.

```
trace_id: abc123
└─ POST /tasks                         [████████████████████] 8200ms  ← span raíz
   ├─ validate request                 [█] 12ms
   ├─ INSERT INTO tasks                 [██] 45ms
   ├─ check project permissions (HTTP)  [███████████████] 7900ms ← ¡acá está!
   └─ publish task.created (queue)      [█] 30ms
```

Con ese waterfall, el "tarda 8s" deja de ser un misterio: la llamada HTTP al servicio de permisos está lenta. Sin el trace, lo adivinabas.

**La pieza que hace que funcione entre servicios: la propagación de contexto (context propagation).** Para que el span de Postgres y el span del worker pertenezcan al *mismo* trace que la request entrante, el `trace_id` (y el `span_id` del padre) tiene que **viajar con la llamada** de un servicio a otro. En HTTP, viaja en un **header estándar**: `traceparent` (del estándar **W3C Trace Context**). En una cola, viaja en los **metadatos del mensaje**. Cada servicio lee el contexto entrante, crea sus spans como hijos, y lo propaga al siguiente salto.

```
Servicio A ──HTTP──> Servicio B ──cola──> Worker C
  traceparent: 00-abc123-span1-01  →  lee abc123, crea span2 hijo de span1
                                       y al publicar mete abc123 en el mensaje  →  Worker C continúa el MISMO trace
```

Ese mismo `trace_id` es el que pusiste en los logs (módulo 2). Así, desde un span lento del trace saltás directo a los logs de esa request exacta. Eso es la correlación de los tres pilares (módulo 7).

El concepto que cierra esto en Node: la propagación **dentro** de un proceso (que el span de la query sepa quién es su padre, aunque pasen por callbacks async) se apoya en **`AsyncLocalStorage`** —el mecanismo de Node para mantener contexto a través de la cadena async sin pasarlo a mano por cada función—. OpenTelemetry lo usa por debajo; vos casi nunca lo tocás.

> **Ojo con las colas: la propagación NO siempre es automática.** En HTTP, la auto-instrumentación (módulo 5) inyecta y lee el `traceparent` por vos. Pero al pasar por una **cola** (BullMQ, SQS, RabbitMQ) muchas veces **no hay auto-instrumentación que lo haga**, y si no inyectás el contexto a mano, el trace se **parte**: el productor termina su trace y el worker arranca uno nuevo, desconectado. El patrón es usar el **propagador de OTel** para `inject` el `traceparent` en los **metadatos del mensaje** al encolar, y `extract`erlo en el worker al consumir, recreando el contexto. Es el caso donde más se rompen los traces en la práctica.

**Ejercicios 4**
4.1 Definí trace, span y `trace_id`. ¿Qué relación tienen los spans entre sí (qué estructura forman)?
4.2 Mirando el waterfall del ejemplo, ¿qué span es el culpable de los 8s y cómo lo dedujiste?
4.3 ¿Qué es la "propagación de contexto" y por qué sin ella tendrías traces rotos (un trace por servicio en vez de uno solo)? ¿Por dónde viaja el contexto en HTTP y en una cola?
4.4 ¿Con qué concepto del [módulo 8 de Evaluations](evals.md) es idéntico esto, y qué mecanismo de Node lo sostiene dentro de un proceso?
4.5 La propagación por HTTP suele ser automática, pero por una cola no. ¿Qué pasa con el trace si no propagás el contexto al encolar un job, y cómo lo arreglás (qué hacés con el `traceparent` en el mensaje)?

---

## Módulo 5 — OpenTelemetry: el estándar vendor-neutral

**Teoría.** Hasta acá viste tres pilares y herramientas sueltas (Pino para logs, prom-client para métricas, "algo" para traces). El problema histórico fue el **lock-in**: cada vendor (Datadog, New Relic, etc.) traía su propio agente y su propio SDK incrustado en tu código. Cambiar de proveedor significaba **reinstrumentar toda la app**. **OpenTelemetry (OTel)** resuelve eso: es el **estándar abierto y vendor-neutral** —proyecto de la CNCF, la misma fundación que Kubernetes— para **generar y exportar** telemetría (las tres señales). Instrumentás **una vez** con la API de OTel, y elegís el backend (Jaeger, Grafana, Datadog, CloudWatch) por **configuración**, sin tocar código.

La frase mental: **OTel es a la observabilidad lo que SQL es a las bases —una interfaz común que te despega del proveedor concreto.**

Las piezas de OTel que tenés que conocer:

- **API / SDK**: lo que usás en tu código para crear spans, métricas y logs (o lo que la **instrumentación automática** usa por vos).
- **Instrumentación automática (auto-instrumentation)**: librerías que parchean frameworks comunes (HTTP, Express/Nest, `pg`, `ioredis`, `amqplib`) para emitir spans **sin que escribas nada**. Levantás tu app con el SDK y ya tenés traces de cada request HTTP y cada query a la DB gratis. La instrumentación **manual** la agregás solo para spans de tu lógica de negocio.
- **Exporters**: traducen la telemetría al formato del destino. El formato nativo es **OTLP** (OpenTelemetry Protocol).
- **El Collector**: un proceso aparte que **recibe** telemetría de tus apps (vía OTLP), la **procesa** (filtra, samplea, agrega atributos, quita PII) y la **exporta** a uno o varios backends. Es la pieza que te da el desacople real: tus apps mandan todo al Collector; cambiar de backend es cambiar la config del Collector, no de las 20 apps.

```
[tu API NestJS]  ─┐
[servicio B]     ─┤── OTLP ──> [OTel Collector] ──> Jaeger/Tempo (traces)
[worker C]       ─┘              (procesa, samplea)──> Prometheus (métricas)
                                                   └─> Loki / CloudWatch (logs)
```

Los **backends** son donde la telemetría vive y se consulta. Un stack open source común: **Prometheus** (métricas) + **Tempo o Jaeger** (traces) + **Loki** (logs), todo visualizado en **Grafana**. En AWS, el equivalente gestionado es **CloudWatch** (logs/métricas) + **X-Ray** (traces) —del módulo de [AWS](aws.md)—. OTel exporta a cualquiera de los dos sin cambiar tu código.

**El sampling (muestreo).** Guardar **todos** los traces de un sistema de alto tráfico es carísimo y casi nadie lo necesita. El **sampling** decide qué fracción de traces conservás. Dos estrategias: **head-based** (decidís al inicio del trace, ej. "guardo 10%" — simple, barato) y **tail-based** (decidís al final, viendo el trace completo, ej. "guardo el 100% de los que tuvieron error o fueron lentos, y 1% del resto" — más útil, más complejo, lo hace el Collector). La regla: querés **todos los traces interesantes** (errores, lentos) y una **muestra** de los normales.

> **El costo escondido del tail-sampling** (lo que distingue el diseño naíf del real). Para decidir "este trace tuvo un error" al final, el Collector necesita **ver TODOS los spans del trace** — pero esos spans llegan en momentos distintos y, peor, **desde servicios distintos**. Eso obliga a dos cosas: (1) **bufferear los spans en memoria** hasta que el trace "cierra" (con su latencia y su costo de RAM), y (2) garantizar que **todos los spans del mismo `trace_id` lleguen al mismo Collector** — con varias réplicas y balanceo trivial, los spans de un trace se reparten y la decisión sale mal. La arquitectura real es de **dos capas**: Collectors *agent* (DaemonSet, uno por nodo) que reenvían a un tier de Collectors *gateway* con un **`loadbalancing exporter` por `trace_id`**, para que cada trace entero aterrice en un solo gateway que pueda decidir. No es "prendé tail-sampling y listo".

Una nota de madurez: de las tres señales de OTel, **logs es la menos madura**; por eso es común seguir usando **Pino + inyección del `trace_id`** (módulo 7) en vez del logging SDK de OTel, mientras traces y métricas sí se llevan con OTel de punta a punta.

**Ejercicios 5**
5.1 ¿Qué problema concreto resuelve OpenTelemetry y por qué se lo llama "vendor-neutral"? Usá la analogía con SQL.
5.2 ¿Qué te da la instrumentación automática "gratis" y cuándo necesitás instrumentación manual?
5.3 ¿Qué hace el OTel Collector y qué desacople te da frente a meter el exporter de cada vendor en tu app?
5.4 Explicá head-based vs tail-based sampling. Si querés capturar siempre los errores y los traces lentos, ¿cuál usás y dónde corre?
5.5 ¿Por qué el tail-sampling obliga a bufferear en memoria y a rutear todos los spans de un mismo `trace_id` al mismo Collector? ¿Qué arquitectura de dos capas lo resuelve?

---

## Módulo 6 — Instrumentar tu API NestJS con OTel (hands-on)

**Teoría.** Pongamos OTel a correr en la Task API. El paso clave de Node: el SDK de OTel tiene que arrancar **antes** que cualquier otra cosa, para poder parchear los módulos (`http`, `pg`, etc.) en el momento del `require`/`import`. Por eso va en un archivo aparte que se carga primero (`--import` o `-r`).

```ts
// tracing.ts — se carga ANTES que la app (node --import ./tracing.js dist/main)
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { resourceFromAttributes } from '@opentelemetry/resources';
import { ATTR_SERVICE_NAME } from '@opentelemetry/semantic-conventions';

const sdk = new NodeSDK({
  resource: resourceFromAttributes({
    [ATTR_SERVICE_NAME]: 'task-api', // así distinguís este servicio en los traces
  }),
  traceExporter: new OTLPTraceExporter({
    url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT, // → el Collector
  }),
  instrumentations: [getNodeAutoInstrumentations()], // HTTP, pg, ioredis, nest... gratis
});

sdk.start(); // a partir de acá, cada request HTTP y cada query genera spans solos
```

```bash
# Arrancás cargando el tracing primero:
node --import ./dist/tracing.js dist/main.js
```

Con solo eso ya tenés un trace por cada request entrante, con spans hijos de cada query a Postgres y cada llamada a Redis. Para tu **lógica de negocio** agregás spans manuales:

```ts
// Un span manual alrededor de una operación de dominio que querés ver en el waterfall
import { trace } from '@opentelemetry/api';
const tracer = trace.getTracer('task-service');

async createTask(dto: CreateTaskDto) {
  return tracer.startActiveSpan('checkProjectPermissions', async (span) => {
    try {
      span.setAttribute('project.id', dto.projectId); // atributo consultable (¡no PII!)
      const allowed = await this.permissions.check(dto.projectId, dto.userId);
      span.setStatus({ code: allowed ? SpanStatusCode.OK : SpanStatusCode.ERROR });
      return allowed;
    } catch (err) {
      span.recordException(err); // el error queda pegado al span exacto que falló
      throw err;
    } finally {
      span.end(); // SIEMPRE cerrar el span, o el trace queda abierto/incompleto
    }
  });
}
```

Tres reglas que evitan los errores típicos:

1. **Cerrá siempre el span** (`span.end()` en `finally`). Un span sin cerrar es un trace roto.
2. **Atributos sí, PII no.** `project.id` es útil y de baja cardinalidad; `user.email` es PII y además dispara cardinalidad. Lo mismo que en logs (módulo 2).
3. **No reinstrumentes lo que la auto-instrumentación ya cubre.** No envuelvas tus queries a mano: el instrumentation de `pg` ya las traza. Los spans manuales son para *tu* lógica (permisos, reglas de negocio, llamadas a APIs externas que no tengan auto-instrumentación).

En **Kubernetes**, el patrón es el mismo pero el Collector suele correr como **DaemonSet** (uno por nodo) o como **sidecar**, y tus pods le exportan vía OTLP —conecta con el [módulo de K8s](kubernetes.md): el Collector es una pieza más de infra declarativa—.

**Para verlo de verdad (laboratorio local).** Leer el snippet no es lo mismo que ver un trace. Armá un `docker-compose.yml` con el stack mínimo y corré la Task API instrumentada apuntándole:

```yaml
# docker-compose.observabilidad.yml (esqueleto)
services:
  otel-collector:
    image: otel/opentelemetry-collector-contrib
    command: ["--config=/etc/otel.yaml"]
    volumes: ["./otel.yaml:/etc/otel.yaml"]
    ports: ["4318:4318"]          # OTLP/HTTP (a donde exporta tu app)
  jaeger:
    image: jaegertracing/all-in-one  # UI de traces en :16686
    ports: ["16686:16686"]
  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
  grafana:
    image: grafana/grafana
    ports: ["3001:3000"]
```

Levantás el stack, corrés `node --import ./dist/tracing.js dist/main.js` con `OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318`, generás tráfico contra la API, y abrís Jaeger en `:16686` para *ver* los waterfalls reales. Eso es el "hands-on" del título.

**Ejercicios 6**
6.1 ¿Por qué el archivo `tracing.ts` se tiene que cargar **antes** que `main.ts`? ¿Qué se rompería si lo importaras desde dentro de la app?
6.2 Listá tres cosas que la auto-instrumentación traza sin que escribas código, y una que necesite span manual.
6.3 ¿Por qué `span.end()` va en `finally` y no después del `await`?
6.4 ¿Cuál es el error de meter `user.email` como atributo de un span? (dos razones)
6.5 (laboratorio) Levantá el stack del `docker-compose` con Collector + Jaeger + Prometheus + Grafana, instrumentá la Task API y generá tráfico. Meté un `await sleep(3000)` dentro de un handler y **encontrá en el waterfall de Jaeger** qué span se comió el tiempo.
6.6 (laboratorio) Rompé la propagación: en una llamada saliente (fetch o al encolar un job) **quitá/no propagues el `traceparent`** y observá cómo el trace se parte en dos traces desconectados. Después arreglalo y verificá que vuelve a ser uno solo.

---

## Módulo 7 — Correlacionar los tres pilares

**Teoría.** Tener logs, métricas y traces por separado vale, pero el salto de nivel es **conectarlos**, para poder navegar de uno a otro durante un incidente. La pieza que los une es el identificador que ya viste tres veces: el **`trace_id`**.

El flujo de un debugging real, conectando los pilares:

1. **Métrica** (la alarma): un dashboard muestra que el **p99 de `POST /tasks` saltó a 8s**. Sabés *que* hay un problema y *dónde* (qué endpoint), pero no *por qué*. Las métricas son agregadas: no tienen el detalle de una request individual.
2. **Trace** (el diagnóstico): la métrica de latencia lleva **exemplars** —punteros a `trace_id`s de ejemplo que cayeron en ese bucket lento—. Hacés clic y abrís el **trace de una request real de 8s**. El waterfall te muestra que el span `checkProjectPermissions` se comió 7.9s. Ahora sabés *en qué paso*.
3. **Log** (la causa raíz): desde ese span, como comparte el `trace_id`, saltás a **los logs exactos de esa request**. Ahí ves: `{"level":"warn","event":"permissions.retry","trace_id":"abc123","attempt":3,"reason":"upstream timeout"}`. El servicio de permisos estaba reintentando contra un upstream caído. Causa raíz encontrada.

Métrica → trace → log: de *qué tan mal* a *dónde* a *por qué*. Sin correlación, cada pilar es una isla y el debugging es a ciegas.

Para que esto funcione, dos requisitos concretos:

- **El `trace_id` tiene que estar en los logs.** Cuando un log se emite dentro de un span, el SDK de OTel puede inyectar el `trace_id` y `span_id` activos en el log automáticamente (vía el `AsyncLocalStorage` del módulo 4). Así filtrás logs por `trace_id`.
- **Exemplars en las métricas.** Un histogram de Prometheus puede adjuntar, a cada bucket, ejemplos de `trace_id`s que cayeron ahí. Es el puente directo métrica → trace.

La frase mental: **los tres pilares no son tres herramientas, son tres vistas del mismo evento, cosidas por el `trace_id`.** Una request mala dejó un punto en una métrica, un waterfall en un trace y varias líneas en los logs; el `trace_id` es el hilo que las une.

**Ejercicios 7**
7.1 Ordená los tres pilares en el flujo "de la alarma a la causa raíz" y decí qué pregunta responde cada uno.
7.2 ¿Qué son los *exemplars* y qué dos pilares conectan?
7.3 ¿Qué tiene que estar presente en una línea de log para poder saltar de un trace a sus logs? ¿Cómo llega ahí en una app con OTel?
7.4 Sin correlación entre pilares, ¿por qué un dashboard de "p99 = 8s" no alcanza para arreglar nada?

---

## Módulo 8 — SLI, SLO y error budgets

**Teoría.** Ya sabés medir. ¿Pero cuándo está "bien" tu sistema? "Lo más rápido posible" y "cero errores" no son objetivos: son imposibles y no te dejan decidir. Acá entra el vocabulario de **SRE** (Site Reliability Engineering, la disciplina de Google), que convierte la confiabilidad en algo **medible y negociable**.

- **SLI** (*Service Level Indicator*): la **métrica** concreta de salud, vista desde el usuario. Ej.: "% de requests a `/tasks` que responden con éxito en menos de 300ms". Es un número, sale de tus métricas (módulo 3).
- **SLO** (*Service Level Objective*): el **objetivo interno** sobre ese SLI. Ej.: "el 99.9% de las requests cumplen el SLI, medido sobre 30 días". Es la meta que tu equipo se compromete a sostener.
- **SLA** (*Service Level Agreement*): el **contrato con el cliente**, con consecuencias (reembolsos). Suele ser **más laxo** que el SLO interno (ej. SLA 99.5%, SLO 99.9%), para tener colchón antes de incumplir el contrato.

La idea más potente y la que más cambia cómo trabajás: el **error budget (presupuesto de error)**. Si tu SLO es 99.9% de éxito, estás diciendo que aceptás **0.1% de fallos**. Ese 0.1% es un **presupuesto** que podés gastar. Sobre 30 días, 99.9% de disponibilidad ≈ **43 minutos** de "fuera de SLO" permitidos al mes.

¿Para qué sirve un presupuesto de fallos? Para **resolver la tensión entre features y estabilidad con un dato, no con opiniones**:

- Si te **sobra** error budget (vas en 99.97% y el SLO es 99.9%), podés **arriesgar**: deployar más rápido, probar cosas nuevas. La confiabilidad está de sobra.
- Si te **quedaste sin** error budget (ya gastaste los 43 min este mes), **frenás las features** y dedicás el equipo a estabilidad hasta recuperarlo. El presupuesto se agotó.

Esto saca la discusión "¿deployamos esto arriesgado o no?" del terreno del más gritón y la pone en un número objetivo. Es **criterio de Tech Lead**: el error budget es la herramienta para negociar entre producto (quiere features) e ingeniería (quiere estabilidad). Y conecta con el módulo de [Evaluations](evals.md): el SLO es el umbral de "suficientemente bueno" del sistema, igual que el threshold de una eval decide si un cambio es seguro de deployar.

La clave de un buen SLI: **medilo desde la perspectiva del usuario, no de la máquina.** "CPU al 50%" no es un SLI (al usuario no le importa el CPU); "las requests responden rápido y sin error" sí lo es. **100% nunca es el objetivo**: cuesta infinito y el usuario no nota la diferencia entre 99.9% y 100% —lo nota su factura—.

**Ejercicios 8**
8.1 Definí SLI, SLO y SLA con un ejemplo de cada uno para la Task API. ¿Por qué el SLA suele ser más laxo que el SLO?
8.2 Tu SLO de disponibilidad es **99.95%** mensual. ¿Aproximadamente cuántos minutos de error budget tenés al mes (calculalo vos)? Y si para el día 10 del mes ya gastaste casi todo, ¿qué te dice eso del *burn rate* (módulo 9) y qué decidís?
8.3 ¿Por qué "CPU al 80%" es un mal SLI y "p99 de latencia < 300ms" es un buen SLI?
8.4 ¿Por qué un SLO de 100% es una mala idea?

---

## Módulo 9 — Alerting: alertá sobre síntomas, no sobre causas

**Teoría.** Tenés métricas y SLOs. El paso final operativo: que te **avisen** cuando algo está mal, sin tener que mirar dashboards 24/7. Pero alertar mal es peor que no alertar: genera la **fatiga de alertas (alert fatigue)** —tantas alertas (muchas falsas o irrelevantes) que el equipo las empieza a ignorar, y el día que llega la importante, nadie la mira—. Una alerta a las 3 AM que no requería acción humana es un bug.

Las dos reglas que separan un buen alerting de un infierno de ruido:

1. **Alertá sobre síntomas, no sobre causas.** El síntoma es lo que el **usuario sufre**: "el p99 de `/tasks` superó 1s", "la tasa de errores 5xx pasó del 1%". La causa es un detalle interno: "el CPU está al 90%", "hay 80 conexiones a la DB". El problema de alertar sobre causas: el CPU al 90% **puede que no afecte a nadie** (falsa alarma → fatiga), y además hay mil causas posibles que no podés anticipar (volvé al módulo 1: unknown-unknowns). Si alertás sobre el **síntoma**, capturás *cualquier* causa que degrade al usuario, conocida o no —y solo te despierta si realmente hay impacto—. Las causas las investigás *después* de que sonó la alerta, con tus traces y logs.

2. **Toda alerta tiene que ser accionable.** Si suena y la respuesta es "ah, sí, eso pasa siempre, ignorala", esa alerta no debería existir. Cada alerta debe significar "un humano tiene que hacer algo ahora". Lo que no es accionable va a un dashboard, no a un pager.

**Alertas basadas en SLO (burn rate).** La forma moderna de alertar conecta con el error budget (módulo 8): en vez de "errores > 1%" (umbral arbitrario), alertás según **qué tan rápido estás quemando el error budget**. El *burn rate* es un **múltiplo del ritmo sostenible**: burn rate = 1 significa que vas justo para gastar exactamente el budget del mes (ni más ni menos); burn rate = 14.4 significa que vas **14.4× más rápido**.

**¿De dónde sale el 14.4?** (número que no hay que tragarse de memoria). Un mes tiene ~720 horas. El budget es 100% de los fallos permitidos. Si quisieras consumir el **5% del budget mensual en 1 hora**, vas a un ritmo de `0.05 × 720 = 14.4×` el sostenible — a ese ritmo, agotarías **todo** el presupuesto del mes en ~2 días. Esa es la combinación canónica para un **page urgente**. La query, con SLO 99.9% (budget = 0.001):

```promql
# Burn rate alto: error 5xx quemando el budget ~14.4x → page urgente
(
  sum(rate(http_requests_total{status=~"5.."}[1h]))
    / sum(rate(http_requests_total[1h]))
) > (14.4 * 0.001)   # 0.001 = 1 - SLO(99.9%)
```

**Pero una sola ventana no alcanza: multiwindow multi-burn-rate.** Una alerta con una sola ventana `[1h]` tiene dos defectos: salta por un **pico transitorio** (un blip de 2 minutos no debería despertarte) y **tarda en apagarse** cuando el problema ya pasó. La práctica de Google SRE (*SRE Workbook*) es combinar una **ventana larga y una corta** con `AND`: la larga confirma que el quemado es sostenido, la corta confirma que **sigue activo ahora** (así la alerta se resetea rápido al recuperarte). Y se usan **varios niveles de severidad** según el burn rate:

| Burn rate | Ventana larga + corta | % budget consumido | Acción |
|-----------|----------------------|--------------------|--------|
| **14.4×** | 1h y 5m | ~2% en 1h | **page** (urgente) |
| **6×** | 6h y 30m | ~5% en 6h | **page** |
| **1×** | 3d y 6h | ~10% en 3d | **ticket** |

```promql
# Page rápido: 14.4x sostenido en la ventana larga Y todavía activo en la corta
(
  (sum(rate(http_requests_total{status=~"5.."}[1h])) / sum(rate(http_requests_total[1h]))) > (14.4 * 0.001)
  and
  (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))) > (14.4 * 0.001)
)
```

Distinguí siempre **page** (urgente, despierta a alguien, hay impacto al usuario *ahora*) de **ticket** (puede esperar a horario laboral). Mandar todo a page es la receta de la fatiga.

> **Toda alerta de page debería linkear a un runbook.** Un *runbook* (o playbook) es el documento corto que dice "si suena esta alerta, mirá X, chequeá Y, mitigá con Z". Despertar a alguien a las 3 AM con una alerta y sin runbook es la mitad del trabajo: el runbook es lo que convierte "algo está mal" en "esto es lo que hago ahora", y baja el MTTR (módulo 11).

**Ejercicios 9**
9.1 ¿Qué es la fatiga de alertas y por qué una alerta no accionable es activamente dañina?
9.2 Te dan dos alertas candidatas: (a) "CPU del nodo > 85%"; (b) "tasa de errores 5xx de la API > 2% por 5 min". ¿Cuál es síntoma y cuál causa? ¿Cuál pondrías como page y por qué?
9.3 ¿Qué ventaja tiene alertar sobre síntomas frente a alertar sobre causas, en términos de los unknown-unknowns del módulo 1?
9.4 ¿Qué es alertar por *burn rate* y cómo conecta con el error budget del módulo 8? Derivá de dónde sale el factor 14.4.
9.5 ¿Qué problema resuelve combinar una ventana larga y una corta (multiwindow) frente a una sola ventana, y para qué sirve el runbook que acompaña a una alerta de page?

---

## Módulo 10 — Cardinalidad y el costo de la observabilidad

**Teoría.** La observabilidad no es gratis, y el que no entiende su costo termina con una factura de telemetría más cara que la infraestructura que observa, o con un sistema de métricas que colapsa. El concepto que tenés que dominar es la **cardinalidad**.

**Cardinalidad = cantidad de combinaciones únicas de labels.** En métricas (Prometheus), cada combinación distinta de valores de labels es una **serie temporal** separada que hay que almacenar y consultar. Si `http_requests_total` tiene los labels `method` (5 valores) × `route` (40 valores) × `status` (15 valores), eso son hasta 5 × 40 × 15 = **3000 series**. Manejable. Pero si agregás `userId` con un millón de usuarios, multiplicás por un millón: **explosión de cardinalidad**. Tu Prometheus se queda sin memoria, las queries se arrastran y la factura se dispara.

La regla: **los labels de métricas tienen que ser de cardinalidad acotada y conocida.** `route`, `method`, `status`, `region` (pocos valores) → sí. `userId`, `taskId`, `trace_id`, `email`, timestamps (ilimitados) → **nunca como label de métrica**.

¿Y si necesitás el detalle por usuario? Ahí está la **división del trabajo entre pilares** —la respuesta al "¿dónde meto la info de alta cardinalidad?"—:

- **Métricas**: baratas, agregadas, **baja cardinalidad**. Para "¿qué tan sano está el sistema?".
- **Logs y traces**: alta cardinalidad permitida (cada log/trace *es* un evento individual con `userId`, `taskId`, etc.). Para "¿qué le pasó a *esta* request/usuario?".

Entonces: "creación de tareas por usuario" **no** es una métrica con label `userId`; es una **consulta sobre logs/traces** filtrando por `userId`. La alta cardinalidad vive donde es *más* barata (eventos), no donde explota (series temporales).

> **Pero "barata" no es "gratis".** Que en logs/traces puedas meter `userId`/`taskId` no significa que la alta cardinalidad salga sin costo: los atributos de alta cardinalidad inflan el **almacenamiento, el indexado y la retención** del backend de traces/logs. La diferencia con las métricas es **cómo se controla el costo**: en métricas, manteniendo los **labels acotados**; en logs/traces, con **sampling y retención** (no guardás todo, no lo guardás para siempre). El error es creer que logs/traces son un volcadero infinito gratis.

Las otras palancas de costo, y cómo controlarlas:

- **Sampling de traces** (módulo 5): no guardes el 100%; quedate con los interesantes (errores, lentos) y una muestra del resto. Es la palanca #1 de costo de traces.
- **Volumen y retención de logs**: logueá a nivel `info` en prod (no `debug`), y definí cuánto tiempo retenés (¿30 días? ¿90?). Logs `debug` a full en prod = factura enorme y ruido.
- **Resolución y retención de métricas**: muestrear cada 15s vs cada 1s, y agregar (downsampling) los datos viejos.

La frase de criterio: **la observabilidad es una inversión con rendimientos decrecientes —instrumentá lo que te ayuda a decidir y a debuggear, no "todo por las dudas"—.** Cada serie, cada log y cada trace cuesta; el objetivo no es máxima telemetría, es máxima *capacidad de responder preguntas* por dólar gastado.

**Ejercicios 10**
10.1 Definí cardinalidad en el contexto de métricas. ¿Por qué agregar `userId` como label puede tumbar tu Prometheus?
10.2 Querés saber "cuántas tareas creó el usuario 4521 esta semana". ¿Lo resolvés con una métrica labelada por `userId` o con una consulta sobre logs/traces? ¿Por qué?
10.3 Nombrá tres palancas concretas para controlar el costo de la observabilidad.
10.4 Repartí estos datos en el pilar correcto según su cardinalidad: (a) total de requests por endpoint; (b) el `taskId` y `userId` de cada creación; (c) el p99 de latencia por ruta.

---

## Módulo 11 — El criterio: cuánta observabilidad, cuándo y para qué

**Teoría.** El módulo que cierra y el más de Tech Lead. Tenés todo el arsenal —tres pilares, OTel, SLIs/SLOs, error budgets, alerting por síntomas, control de cardinalidad—. El error ahora es el opuesto al de no observar nada: **sobre-instrumentar**, construir un muro de 40 dashboards que nadie mira, alertar de todo hasta la fatiga, o gastar en telemetría más de lo que vale el sistema. La observabilidad madura es la que **responde preguntas y dirige acciones**, no la que acumula gráficos.

Los principios de criterio que tenés que poder defender en una entrevista de senior/lead:

1. **Observabilidad ≠ dashboards bonitos.** Un dashboard que nadie abre y que no dispara ninguna decisión es deuda, no observabilidad. La pregunta correcta no es "¿qué puedo graficar?" sino "**¿qué pregunta voy a necesitar responder a las 3 AM durante un incidente?**" —e instrumentás para *esa* pregunta—.

2. **Instrumentá desde el día uno, no después del incidente.** No podés agregar tracing *durante* el incidente para entenderlo: necesitás los datos de *antes*. La observabilidad es parte del sistema, no un extra que se agrega cuando ya duele —el mismo principio del [módulo 8 de Evaluations](evals.md) ("lo instrumentás desde el inicio") y de [NestJS senior](nestjs-senior.md) ("medí antes de optimizar")—.

3. **Escalá la observabilidad al sistema.** No todo merece lo mismo:
   - Un **script o un side project**: logs estructurados y poco más. Tracing distribuido es overkill (no hay nada distribuido).
   - Una **API monolítica con tráfico real**: los tres pilares básicos (logs JSON, métricas RED, traces con auto-instrumentación), un par de SLOs y alertas de síntoma.
   - Un **sistema distribuido crítico** (microservicios, event-driven, dinero o salud en juego): tracing completo con propagación de contexto, SLOs con error budgets, alerting por burn rate, tail-sampling, y on-call serio.

   Es el mismo eje "escalá la solución al problema" que la escalera llamada → workflow → agente del [módulo de Agentes](agentes.md) y el "¿K8s o Fargate?" del [módulo de Kubernetes](kubernetes.md): meter observabilidad de sistema crítico en un side project es la misma sobre-ingeniería que meter K8s donde alcanzaba un Fargate.

4. **El objetivo final es reducir el MTTR** (Mean Time To Recovery: cuánto tardás en resolver un incidente). La observabilidad no evita que las cosas fallen —fallan igual—; lo que hace es que cuando fallan, pases de "no tengo idea qué pasa" a "causa raíz en 5 minutos". Si tu telemetría no reduce el MTTR, no está haciendo su trabajo. Y lo que cierra el ciclo del MTTR es el **postmortem blameless** (sin culpar personas, enfocado en el sistema): cada incidente deja aprendizajes, runbooks nuevos y alertas mejor calibradas. La observabilidad alimenta el postmortem; el postmortem mejora la observabilidad.

5. **Cuidado con las vanity metrics.** Un número que sube y se ve lindo en una pantalla pero que no cambia ninguna decisión es ruido. Toda métrica que mirás debería poder terminar la frase "si esto cambia, yo haría ___".

> **Dos distinciones que caen en entrevista.** (1) **Healthcheck ≠ SLO**: el liveness/readiness de K8s (módulo de Kubernetes) es **binario** ("¿el proceso está vivo / listo para recibir tráfico?") y solo dispara reinicios/ruteo; un pod perfectamente "healthy" puede estar **violando el SLO** de latencia. Son señales distintas: salud del proceso vs calidad de servicio percibida. (2) **La instrumentación tiene overhead**: la auto-instrumentación y cada span cuestan CPU y un poco de latencia. Por eso el sampling no solo abarata el *almacenamiento* — también reduce el *overhead* de generar y exportar telemetría.

El cierre del módulo y el puente con el rol: la observabilidad es la mitad operativa del perfil senior/lead. La otra mitad —entregar bien— la viste en testing, CI/CD y arquitectura; **esta** es la que te deja **operar** lo que entregaste: saber qué hace en producción, detectar cuándo se degrada, y arreglarlo rápido cuando se rompe. Junto con SLOs y error budgets, es además la herramienta con la que un Tech Lead **negocia entre velocidad y estabilidad con datos**, no con opiniones.

**Ejercicios 11**
11.1 ¿Por qué "tenemos 40 dashboards" no es señal de buena observabilidad? ¿Cuál es la pregunta que debería guiar qué instrumentás?
11.2 ¿Por qué la observabilidad se instrumenta desde el inicio y no durante el incidente?
11.3 Para cada caso, decí qué nivel de observabilidad es razonable y por qué: (a) un CLI personal; (b) un monolito con tráfico real; (c) un sistema event-driven que mueve pagos.
11.4 ¿Qué es el MTTR y por qué es la métrica que justifica toda la inversión en observabilidad? ¿Cómo cierra el ciclo el postmortem blameless?
11.5 ¿Qué es una vanity metric y qué pregunta te dice si una métrica vale la pena mirarla?
11.6 ¿Por qué un pod "healthy" en su healthcheck de K8s puede igual estar violando el SLO? ¿En qué se diferencian las dos señales?

---

## Soluciones

### Módulo 1
1.1 **Monitoreo** vigila un set conocido y finito de señales con umbrales: cubre los *known-unknowns* (sabés qué puede fallar, no cuándo —ej. "¿el CPU pasó el 80%?"). **Observabilidad** te deja responder preguntas que no anticipaste, sin deployar código para investigar: cubre los *unknown-unknowns* (ni sabías que esa combinación de condiciones era un problema —ej. "¿por qué las requests de este usuario, a este proyecto, en esta franja horaria, tardan 8s?"). El monitoreo es un subconjunto de la observabilidad.
1.2 (a) **Métricas** (número agregado en el tiempo). (b) **Traces** (el recorrido completo de esa request individual). (c) **Logs** (evento discreto con el mensaje de error y su contexto).
1.3 Te quedan invisibles los **unknown-unknowns**: una alarma de CPU solo dispara ante una causa que ya anticipaste, y solo si esa causa mueve el CPU. Un endpoint lento por reintentos a un upstream caído, un grupo de usuarios afectados por un bug específico, una query lenta solo para ciertos datos —nada de eso necesariamente sube el CPU, y aunque lo hiciera, la alarma no te dice *qué* request ni *por qué*. Sin los tres pilares no podés explorar.

### Módulo 2
2.1 `logger.error({ event: 'task.create.failed', userId, error: err.message })` (campos consultables en vez de string concatenado; idealmente con `trace_id` por contexto).
2.2 `console.log` escribe de forma **síncrona** a stdout: bloquea el event loop (módulo de Node) mientras escribe. Con alto volumen, eso roba tiempo de CPU a procesar requests y degrada la latencia. Un logger como Pino serializa rápido y escribe de forma no bloqueante.
2.3 Resuelve **reconstruir la historia completa de una sola request** cuando muchas corren en paralelo. Con tráfico concurrente, los logs de distintas requests se entrelazan en el mismo stream; sin un identificador común que todos los logs de una request comparten, no hay forma de saber qué línea pertenece a qué request. Filtrando por `trace_id`/`request_id` aislás una sola.
2.4 Secretos (passwords, tokens JWT, API keys) y PII/datos sensibles (números de tarjeta, según el marco regulatorio: emails, etc.). Los logs van a un sistema central legible por muchos y se retienen meses.
2.5 Como los transports de Pino corren en un worker thread, si el proceso muere de golpe (crash, process.exit, SIGKILL) los logs que quedaron en el buffer **se pierden** —justo los del momento del fallo—. Mitigación: hacer flush en el shutdown (pino.final / cerrar el logger en los handlers de SIGTERM/SIGINT, junto al graceful shutdown). En contenedores logueás a stdout (no a archivo) porque el contenedor es efímero y el runtime (Docker/K8s) recolecta stdout.

### Módulo 3
3.1 **RED** (Rate, Errors, Duration) → vista del usuario sobre un servicio que atiende requests; lo mirás para tu **API**. **USE** (Utilization, Saturation, Errors) → vista de un recurso; lo mirás para el **CPU/memoria/disco del nodo**. **Golden signals** (latencia, tráfico, errores, saturación) → set mínimo universal, prácticamente RED + saturación.
3.2 Ej.: 99 requests de 50ms y 1 de 10000ms → promedio ≈ (99×50 + 10000)/100 ≈ **150ms**, que parece sano y oculta que un usuario esperó 10s. El **p99 = 10000ms** expone exactamente esa cola lenta. El promedio diluye los outliers; el percentil los muestra.
3.3 (a) **counter** (solo sube). (b) **gauge** (sube y baja). (c) **histogram** (distribución → percentiles).
3.4 Porque cada combinación de labels crea una serie temporal: `route` tiene decenas de valores acotados (manejable), `userId` tiene millones e ilimitado (explosión de cardinalidad → tumba Prometheus). El detalle por usuario va en logs/traces (módulo 10).
3.5 Porque los **conteos** de los buckets SÍ se pueden agregar (sumar) entre instancias, pero los **percentiles NO** (el p99 de 3 instancias no es el promedio de sus p99). Por eso se suman primero los buckets de todas las instancias (`sum(...) by (le)`) y recién sobre esa distribución agregada se estima el cuantil. Hacerlo al revés (promediar percentiles) da un número sin sentido.
3.6 `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{route="/tasks"}[5m])) by (le))`. Si da 0.42 (segundos = 420ms), el SLO de "p95 < 300ms" **no se cumple** (420 > 300).

### Módulo 4
4.1 **Trace**: el registro completo de una request a través de todo el sistema, con un `trace_id` único. **Span**: cada operación individual dentro del trace (con inicio/fin → duración, atributos, `span_id`). **`trace_id`**: el identificador único del trace entero. Los spans forman un **árbol**: cada uno apunta a su padre (`parent_span_id`); la raíz es la request entrante.
4.2 El span `check project permissions (HTTP)` con 7900ms: en el waterfall su barra ocupa casi todo el tiempo del span raíz (8200ms), mientras los demás (validate 12ms, INSERT 45ms, publish 30ms) son insignificantes. La barra larga es la culpable.
4.3 Es hacer **viajar el contexto del trace** (`trace_id` + `span_id` del padre) de un servicio al siguiente, para que sus spans se cuelguen del mismo trace. Sin ella, cada servicio generaría un trace propio y desconectado → en vez de un waterfall único de toda la request, tendrías N traces sueltos sin relación. En **HTTP** viaja en el header `traceparent` (W3C Trace Context); en una **cola**, en los metadatos del mensaje.
4.4 Es idéntico al tracing del **módulo 8 de Evaluations** (trace de un sistema de IA descompuesto en spans: llamadas al LLM, retrievals, tool calls). Dentro de un proceso Node, lo sostiene **`AsyncLocalStorage`**, que mantiene el contexto activo a través de la cadena async sin pasarlo a mano.
4.5 (colas) En HTTP la auto-instrumentación inyecta/lee el `traceparent` sola, pero en una cola (BullMQ/SQS/RabbitMQ) muchas veces NO hay auto-instrumentación que lo haga: si no inyectás el contexto a mano en los metadatos del mensaje (con el propagador de OTel: `inject` al encolar, `extract` al consumir), el trace se parte —el productor cierra su trace y el worker arranca uno nuevo desconectado—.

### Módulo 5
5.1 Resuelve el **vendor lock-in**: instrumentás una vez con la API de OTel y elegís/cambiás el backend (Jaeger, Datadog, CloudWatch) por configuración, sin reinstrumentar la app. "Vendor-neutral" = no atado a ningún proveedor. Como **SQL**: una interfaz común que te despega del motor concreto.
5.2 **Gratis**: spans automáticos de frameworks comunes (HTTP entrante/saliente, queries `pg`, `ioredis`, Nest, etc.) sin escribir código. **Manual**: para spans de **tu lógica de negocio** (chequeo de permisos, reglas de dominio, llamadas a APIs externas sin auto-instrumentación).
5.3 El **Collector** recibe telemetría de tus apps vía OTLP, la procesa (filtra, samplea, quita PII, agrega atributos) y la exporta a uno o varios backends. Desacople: tus apps solo conocen el Collector; cambiar de backend es cambiar la config del Collector, no las N apps (frente a meter el SDK/agente de cada vendor incrustado en cada app).
5.4 **Head-based**: decidís al inicio del trace (ej. 10% fijo) —simple y barato, pero podés perder errores—. **Tail-based**: decidís al final, viendo el trace completo (ej. 100% de errores/lentos + 1% del resto) —más útil, más complejo—. Para capturar siempre errores y lentos usás **tail-based**, y corre en el **Collector** (que es quien ve el trace completo antes de decidir).
5.5 Porque para decidir mirando el trace completo, el Collector tiene que esperar y **bufferear en memoria todos los spans** del trace hasta que cierra; y como los spans de un trace llegan desde varios servicios, **todos los del mismo `trace_id` deben llegar al mismo Collector** (con balanceo trivial se reparten y la decisión sale mal). Se resuelve con dos capas: Collectors *agent* (DaemonSet) que reenvían a un tier *gateway* con un `loadbalancing exporter` por `trace_id`, de modo que cada trace entero aterrice en un solo gateway.

### Módulo 6
6.1 Porque OTel **parchea** los módulos (`http`, `pg`, etc.) en el momento en que se importan, para inyectar la instrumentación. Si la app (y por ende esos módulos) ya se importaron antes de que arranque el SDK, el parcheo llega tarde y la auto-instrumentación **no captura nada**. Por eso se carga con `--import`/`-r` antes que `main`.
6.2 Traza solas (ej.): requests HTTP entrantes, queries a Postgres (`pg`), llamadas a Redis (`ioredis`). Necesita **span manual**: tu lógica de negocio, p. ej. `checkProjectPermissions` o una regla de dominio.
6.3 Para garantizar que el span **siempre** se cierra, falle o no la operación. Si solo lo cerraras después del `await` y este lanza una excepción, el `span.end()` no se ejecutaría → span/trace abierto e incompleto. El `finally` corre en ambos caminos.
6.4 Dos razones: (1) **PII** —el email es dato personal en una telemetría que mucha gente lee (igual que en logs)—; (2) **cardinalidad** —el email es de cardinalidad ilimitada y dispara el costo/explosión de series y de índices de traces—. El dato por usuario se filtra desde el contexto del log/trace, no como atributo de alta cardinalidad indiscriminado.
6.5 (laboratorio) Criterio: en la UI de Jaeger abrís el trace del endpoint con el `sleep`, y en el waterfall el span correspondiente (o el span raíz si el sleep está en el handler) muestra una barra de ~3000ms que domina el trace; el resto de los spans (queries, etc.) son insignificantes al lado. Ese contraste visual es el diagnóstico.
6.6 (laboratorio) Criterio: al no propagar el `traceparent`, en Jaeger ves **dos traces distintos** (uno del productor/servicio A, otro del worker/servicio B) en vez de un waterfall único; el `trace_id` no coincide. Al reinyectar el contexto (inject/extract del propagador), vuelven a ser un solo trace con el worker colgando como hijo.

### Módulo 7
7.1 **Métrica** → "¿qué tan mal y dónde?" (el p99 saltó en `/tasks`); **Trace** → "¿en qué paso?" (el span de permisos se comió 7.9s); **Log** → "¿por qué?" (reintentos contra un upstream caído). De agregado a request individual a causa raíz.
7.2 Los **exemplars** son punteros a `trace_id`s de ejemplo adjuntos a los buckets de un histograma de métrica. Conectan **métricas → traces** (desde un punto lento de la métrica saltás a un trace real de ese caso).
7.3 El **`trace_id`** (y normalmente `span_id`) tiene que estar en la línea de log. En una app con OTel, el SDK lo inyecta automáticamente tomando el contexto activo (`AsyncLocalStorage`) cuando el log se emite dentro de un span.
7.4 Porque la métrica es **agregada**: te dice que hay un problema y en qué endpoint, pero no tiene el detalle de ninguna request individual ni la causa. Sin poder saltar al trace de una request lenta real y de ahí a sus logs, sabés *que* duele pero no *por qué* —y no podés arreglar lo que no podés diagnosticar—.

### Módulo 8
8.1 **SLI** = la métrica de salud vista por el usuario (ej. "% de requests a `/tasks` exitosas en <300ms"). **SLO** = el objetivo interno sobre ese SLI (ej. "99.9% sobre 30 días"). **SLA** = el contrato con el cliente con consecuencias (ej. "99.5% o hay reembolso"). El SLA es más laxo que el SLO para tener **colchón**: querés enterarte y reaccionar (rompiste el SLO) *antes* de incumplir el contrato (SLA) y pagar.
8.2 99.95% mensual ≈ **22 minutos** de error budget al mes (0.05% de ~30 días ≈ 0.0005 × 43200 min ≈ 21.6). Si para el día 10 ya casi lo gastaste, vas a un **burn rate alto** (consumiste ~1 mes de budget en ~1/3 del mes → ~3× o más): señal de page/atención y de **frenar features** para estabilizar, no de seguir arriesgando. (Para comparar: 99.9% daría ~43 min/mes.)
8.3 "CPU al 80%" es interno y al usuario no le importa: el CPU alto puede no afectar a nadie (y el bajo no garantiza que todo ande). "p99 < 300ms" mide lo que el **usuario realmente experimenta** (que las requests respondan rápido), que es lo que la confiabilidad debería capturar.
8.4 Porque 100% cuesta (tendiendo a) infinito y el usuario no nota la diferencia entre 99.9% y 100% —pero la factura y la velocidad de entrega sí—. Además no te deja error budget: sin presupuesto de fallo no podés desplegar ni experimentar nunca.

### Módulo 9
9.1 La **fatiga de alertas** es cuando hay tantas alertas (muchas falsas o no accionables) que el equipo las empieza a ignorar; el día de la alerta crítica, nadie la mira. Una alerta no accionable es dañina porque entrena al equipo a ignorar el pager y diluye la señal de las que sí importan.
9.2 (a) "CPU > 85%" es **causa** (detalle interno). (b) "errores 5xx > 2%" es **síntoma** (impacto al usuario). Pondrías **(b) como page**: significa que usuarios están sufriendo errores *ahora* → requiere acción. El CPU alto puede no afectar a nadie → a lo sumo un ticket/dashboard.
9.3 Alertando sobre **síntomas** capturás *cualquier* causa que degrade al usuario, incluidas las que no anticipaste (unknown-unknowns): no necesitás una alerta por cada causa posible (imposible de enumerar). Una sola alerta de síntoma ("la latencia/errores empeoraron") dispara sin importar qué la provocó, y solo cuando hay impacto real.
9.4 Alertar por **burn rate** es disparar según **qué tan rápido estás consumiendo el error budget** (módulo 8) en vez de un umbral fijo arbitrario. Si lo quemás lento → ticket; si lo quemás tan rápido que agotarías el presupuesto del mes en poco tiempo → page urgente. Gradúa la urgencia por el impacto real sobre el SLO. El **14.4** sale de: querer consumir el 5% del budget mensual en 1h equivale a ir a `0.05 × 720h = 14.4×` el ritmo sostenible (agotaría todo el budget en ~2 días).
9.5 Una sola ventana salta por picos transitorios (un blip de minutos te despierta de gusto) y tarda en apagarse cuando ya te recuperaste. Combinar **ventana larga (confirma que es sostenido) + corta (confirma que sigue activo ahora)** con AND evita ambos: solo alerta si el quemado es real y persistente, y se resetea rápido. El **runbook** que acompaña a la page convierte "algo está mal" en "estos son los pasos para diagnosticar/mitigar", bajando el MTTR.

### Módulo 10
10.1 **Cardinalidad** = cantidad de combinaciones únicas de valores de labels, y cada combinación es una serie temporal a almacenar/consultar. `userId` tiene millones de valores: como label, multiplica las series por millones → explosión de memoria/costo y queries lentísimas, hasta tumbar Prometheus.
10.2 Con una **consulta sobre logs/traces** filtrando por `userId`, **no** con una métrica labelada por `userId`. La alta cardinalidad (por usuario) vive barata en logs/traces (cada uno es un evento individual); como label de métrica explotaría las series.
10.3 (Tres de) **sampling de traces** (no guardar el 100%, quedarse con errores/lentos + muestra), **volumen/retención de logs** (nivel `info` no `debug` en prod, definir días de retención), **resolución/retención de métricas** (intervalo de scrape, downsampling de datos viejos).
10.4 (a) **métrica** (counter por endpoint, baja cardinalidad). (b) **logs/traces** (`taskId`/`userId` son alta cardinalidad → evento individual). (c) **métrica** (histogram por ruta → p99, baja cardinalidad). Aclaración: meter alta cardinalidad en logs/traces es aceptable pero NO gratis (cuesta almacenamiento/retención); ahí el costo se controla con sampling y retención, no con labels acotados como en métricas.

### Módulo 11
11.1 Porque la cantidad de dashboards no mide nada: un dashboard que nadie abre y que no dispara ninguna decisión es deuda, no observabilidad. La pregunta que guía qué instrumentar es **"¿qué voy a necesitar responder a las 3 AM durante un incidente?"** —instrumentás para responder *esa* pregunta, no para acumular gráficos—.
11.2 Porque para entender un incidente necesitás los datos de **antes** de que ocurriera; no podés agregar tracing *durante* el incidente para ver qué pasó (ya pasó, sin instrumentar). La observabilidad es parte del sistema desde el día uno, no un parche post-dolor (mismo principio que "instrumentá desde el inicio" en Evaluations y "medí antes de optimizar" en NestJS senior).
11.3 (a) **CLI personal**: logs estructurados y poco más; tracing distribuido es overkill (no hay nada distribuido). (b) **Monolito con tráfico real**: los tres pilares básicos (logs JSON, métricas RED, traces con auto-instrumentación), un par de SLOs, alertas de síntoma. (c) **Sistema de pagos event-driven**: tracing completo con propagación de contexto, SLOs con error budgets, alerting por burn rate, tail-sampling y on-call serio —el costo del error es alto, la inversión se justifica—.
11.4 **MTTR** (Mean Time To Recovery) = tiempo medio en resolver un incidente. Es la métrica que justifica la inversión porque la observabilidad no evita las fallas, las **acorta**: te lleva de "no sé qué pasa" a "causa raíz en minutos". Si tu telemetría no baja el MTTR, no está cumpliendo su función. El **postmortem blameless** cierra el ciclo: tras cada incidente, sin buscar culpables, se extraen aprendizajes, runbooks y alertas mejor calibradas que bajan el MTTR del próximo.
11.5 Una **vanity metric** es un número que se ve lindo en pantalla pero que no cambia ninguna decisión. La pregunta que la valida: "**si esto cambia, ¿qué haría yo distinto?**" —si la respuesta es "nada", la métrica es ruido—.
11.6 Porque el healthcheck (liveness/readiness de K8s) es **binario**: solo dice si el proceso está vivo / listo para recibir tráfico, y dispara reinicios o ruteo. Un pod puede pasar el healthcheck (está "vivo") y a la vez responder lento, violando el SLO de latencia. Son señales distintas: salud del proceso (binaria, para el orquestador) vs calidad de servicio percibida por el usuario (el SLO).

---

Con este módulo sumás la **mitad operativa** del perfil senior/Tech Lead: ya sabés entregar (testing, CI/CD, arquitectura) y ahora sabés **operar** lo entregado —ver qué hace en producción, detectar cuándo se degrada y arreglarlo rápido—. Recorriste los tres pilares (logs estructurados, métricas RED/USE, tracing distribuido), el estándar **OpenTelemetry** que los unifica y te despega del vendor, cómo **correlacionarlos** por `trace_id`, y el vocabulario de SRE para volver la confiabilidad medible y negociable (**SLI/SLO/error budgets**), además del alerting por síntomas y el control de costo vía cardinalidad y sampling. El puente con el resto del temario queda explícito: es el **mismo tracing** del [módulo 8 de Evaluations](evals.md) aplicado a microservicios, la pieza que opera lo que corre en [Kubernetes](kubernetes.md) y [AWS](aws.md) (CloudWatch/X-Ray), y la herramienta con la que un Tech Lead negocia velocidad vs. estabilidad con datos. Próximos pasos del track: **TDD como disciplina** y **liderazgo técnico** (ADRs, DORA, Team Topologies).
