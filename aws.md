# AWS para backend: cómputo, mensajería y datos en la nube

**De Docker a una arquitectura distribuida y event-driven · nivel Tech Lead · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Hasta acá desplegaste en una PaaS (Railway/Render). Este módulo sube el nivel a lo que piden las búsquedas **senior y Tech Lead**: desplegar y operar sobre **AWS**, el cloud dominante, con sus servicios de **cómputo** (Lambda, ECS/Fargate), **mensajería y eventos** (SQS, EventBridge, Kinesis), **datos** (DynamoDB) y **observabilidad** (CloudWatch, X-Ray). El objetivo no es memorizar cada servicio, sino entender **qué problema resuelve cada uno y cuándo elegirlo** — que es exactamente lo que se evalúa en una entrevista de arquitectura. Conecta directo con varios módulos: ECS reusa tu imagen Docker, SQS es el primo gestionado de tu cola BullMQ/Redis, DynamoDB lleva tus nociones de NoSQL a producción, y EventBridge/Kinesis materializan los eventos de dominio y el patrón Outbox del archivo senior.

**Lo que asumimos.** Tu "Task API" dockerizada, con la teoría de colas (Redis/BullMQ), eventos de dominio, outbox/idempotencia y observabilidad (3 pilares) del archivo senior.

**Para practicar.** Una cuenta AWS (capa gratuita), el **AWS CLI** (`aws configure`) y el **AWS SDK v3** para Node (`@aws-sdk/client-*`). Para infraestructura como código, **AWS CDK** (en TypeScript, ideal para vos) o Terraform. Cuidado con los costos: apagá lo que levantes.

> Este módulo cubre los **conceptos y el criterio** (qué servicio elegir y cuándo). La **práctica con código** —infraestructura como código con CDK, más S3, RDS/Aurora, API Gateway, SNS, ElastiCache, Step Functions, networking, costos y el Well-Architected Framework— vive en el módulo siguiente, **[AWS hands-on con CDK](aws-practica.md)**.

**Índice de módulos**
1. El modelo mental de AWS (regiones, IAM, "todo es un servicio")
2. Cómputo: Lambda vs. ECS/Fargate (cuándo cada uno)
3. ECS/Fargate: tu NestJS en contenedores gestionados
4. Lambda: funciones serverless
5. SQS: colas gestionadas (vs. BullMQ/Redis)
6. EventBridge: el event bus y la arquitectura event-driven
7. Kinesis: streaming de datos (vs. EventBridge)
8. DynamoDB: NoSQL gestionado a escala
9. Observabilidad: CloudWatch, X-Ray y OpenTelemetry
10. IAM y secretos: mínimo privilegio en la nube
11. Juntando todo: una arquitectura event-driven en AWS

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El modelo mental de AWS (regiones, IAM, "todo es un servicio")

**Teoría.** AWS no es "un servidor en la nube": es un catálogo de **cientos de servicios gestionados** que componés como piezas. El cambio mental desde "tengo una máquina con mi app" es: en AWS describís **qué querés** (una cola, una base, un contenedor corriendo) y AWS lo opera por vos. Tres conceptos transversales que necesitás antes de cualquier servicio:

- **Regiones y zonas de disponibilidad (AZ)**: AWS está dividido en regiones geográficas (`us-east-1`, `sa-east-1` São Paulo), y cada región tiene varias AZ (data centers independientes). Distribuir en varias AZ es la base de la **alta disponibilidad**: si una se cae, las otras siguen.
- **IAM (Identity and Access Management)**: el sistema de permisos. **Todo** acceso en AWS pasa por IAM. Nada habla con nada sin permiso explícito. Lo profundizamos en el módulo 10, pero internalizalo desde ya: en AWS, por defecto, **nada está permitido**.
- **Gestionado vs. autogestionado**: casi todo servicio tiene una versión "serverless/gestionada" (AWS opera la infra) y a veces una autogestionada (vos sobre EC2). La regla senior: **preferí lo gestionado** salvo que tengas una razón concreta (costo a gran escala, control fino). Pagás un poco más por dejar de operar servidores; casi siempre vale la pena.

La forma de crear recursos: la consola web (para aprender), el CLI (`aws ...`), o —lo profesional— **infraestructura como código (IaC)** con CDK o Terraform, que versiona tu infra en el repo igual que las migraciones versionan tu DB. Hacer clicks en la consola para producción es el equivalente a tocar la base de prod a mano.

**Ejercicios 1**
1.1 ¿Qué significa que un servicio sea "gestionado" en AWS y cuál es la regla senior por defecto al elegir entre gestionado y autogestionado?
1.2 ¿Por qué distribuir tu app en varias zonas de disponibilidad (AZ) mejora la disponibilidad?
1.3 ¿Por qué crear la infraestructura con IaC (CDK/Terraform) es preferible a hacer clicks en la consola, para producción? Conectá con un concepto del módulo de Docker/deploy.

---

## Módulo 2 — Cómputo: Lambda vs. ECS/Fargate (cuándo cada uno)

**Teoría.** La primera gran decisión: ¿dónde corre tu código? AWS tiene dos caminos principales para backend moderno, y elegir bien es criterio de arquitecto.

- **AWS Lambda (Function as a Service)**: subís una **función**, AWS la ejecuta cuando llega un evento (un request, un mensaje en una cola, un archivo subido). No hay servidor que mantener: escala de 0 a miles de ejecuciones automáticamente y **pagás solo por invocación y tiempo de ejecución**. Pero: cada ejecución es efímera y sin estado, tiene un **límite de 15 minutos**, y sufre **cold starts** (latencia extra la primera vez que arranca tras estar inactiva).

- **ECS/Fargate (contenedores)**: corrés tu **contenedor** (la imagen Docker que ya sabés construir) de forma continua. ECS es el orquestador; **Fargate** es el modo serverless donde AWS gestiona las máquinas por vos (no administrás EC2). Tu app corre permanentemente, sin límite de tiempo, sin cold starts, ideal para un servidor web tradicional como tu API NestJS.

El criterio de elección:

| Usás… | Cuándo |
|---|---|
| **Lambda** | Cargas esporádicas o event-driven (procesar un mensaje, un cron, un webhook), picos impredecibles, "pegamento" entre servicios. Pagás casi nada si no se usa. |
| **ECS/Fargate** | Una API que recibe tráfico constante, procesos largos, conexiones persistentes (WebSockets), cuando los cold starts no son aceptables. |

La regla práctica: tu **API NestJS** de tráfico sostenido vive cómoda en **Fargate** (es tu imagen Docker corriendo gestionada); los **trabajos event-driven** (procesar lo que cae en una cola, reaccionar a un evento) son el caso ideal de **Lambda**. Muchas arquitecturas usan **ambos**: Fargate para la API, Lambda para el procesamiento asíncrono.

Hay opciones intermedias que conviene tener en el radar: **AWS App Runner** es un "Fargate simplificado" — le das tu imagen (o tu repo) y AWS levanta la API con balanceo, HTTPS y autoescalado sin que configures ALB, cluster ni red a mano; ideal para una API contenedorizada cuando no necesitás el control fino de ECS. También existen **Lambda con contenedores** (empaquetar la función como imagen, hasta 10 GB) y **ECS sobre EC2** (vos administrás las máquinas, para cargas grandes y estables donde el costo importa). Para el criterio de este módulo, el eje sigue siendo **Lambda (event-driven/esporádico) vs. Fargate (siempre encendido)**; App Runner es el atajo cuando solo querés "subí mi contenedor y andá".

**Ejercicios 2**
2.1 Nombrá dos limitaciones de Lambda que lo hacen inadecuado para una API web de tráfico constante.
2.2 ¿Qué es Fargate y qué te ahorra respecto de correr contenedores sobre EC2 vos mismo?
2.3 Para cada caso, elegí Lambda o Fargate: (a) tu API REST NestJS con tráfico constante; (b) procesar imágenes que los usuarios suben de a ratos; (c) un cron que corre una vez por día.

---

## Módulo 3 — ECS/Fargate: tu NestJS en contenedores gestionados

**Teoría.** Acá tu módulo de Docker se vuelve producción real. La **misma imagen** multi-stage que construiste se despliega en Fargate. Los conceptos de ECS:

- **Task Definition**: la "receta" de tu contenedor — qué imagen usar (desde **ECR**, el registry de Docker de AWS), cuánta CPU/memoria, qué variables de entorno y secretos, qué puerto. Es el equivalente en AWS a tu `docker run` con todos sus flags.
- **Task**: una instancia en ejecución de esa definición (un contenedor corriendo). El equivalente a un contenedor levantado.
- **Service**: mantiene N tasks corriendo (ej. "siempre 3 réplicas"), las reemplaza si mueren, y hace **deploys sin downtime** (levanta las nuevas, espera que estén sanas, baja las viejas). Es el **escalado horizontal** del módulo de Node, operado por AWS.
- **Application Load Balancer (ALB)**: reparte el tráfico entre las tasks y consulta su **healthcheck** (el `/health` del módulo de Docker). Si una task falla el health, deja de mandarle tráfico.

El flujo de deploy: construís la imagen → la subís a **ECR** → actualizás la Task Definition con la nueva imagen → el Service hace rolling deployment. Todo esto se automatiza en tu **CI/CD** (GitHub Actions del módulo de Docker): el job de deploy empuja a ECR y dispara el update del servicio.

```bash
# subir la imagen a ECR (el registry de AWS)
aws ecr get-login-password | docker login --username AWS --password-stdin <cuenta>.dkr.ecr.<region>.amazonaws.com
docker build -t task-api .
docker tag task-api <cuenta>.dkr.ecr.<region>.amazonaws.com/task-api:latest
docker push <cuenta>.dkr.ecr.<region>.amazonaws.com/task-api:latest

# forzar un nuevo deployment del servicio con la imagen actualizada
aws ecs update-service --cluster prod --service task-api --force-new-deployment
```

Acá todo lo del módulo de Docker rinde: el **graceful shutdown** (SIGTERM) permite que el rolling deploy no corte requests; el **healthcheck** le dice al ALB cuándo la task está lista; las variables de entorno y secretos se inyectan desde la Task Definition (módulo 10). Auto Scaling ajusta el número de tasks según CPU/memoria o requests.

**Ejercicios 3**
3.1 Relacioná cada pieza de ECS con su equivalente "manual": Task Definition, Task, Service.
3.2 ¿Qué rol cumple el ALB y cómo usa el endpoint `/health` de tu app (del módulo de Docker)?
3.3 ¿Por qué el graceful shutdown (SIGTERM) de tu app es importante para que el rolling deployment de un Service no corte requests?

---

## Módulo 4 — Lambda: funciones serverless

**Teoría.** Una Lambda es una función que AWS ejecuta en respuesta a un **trigger** (evento). No la "arrancás": se invoca sola cuando pasa algo. Los triggers típicos en backend:

- Un mensaje llega a **SQS** → Lambda lo procesa (el worker del módulo de Redis, pero serverless).
- Un evento en **EventBridge** → Lambda reacciona.
- Un archivo sube a **S3** → Lambda lo procesa (redimensionar imagen, parsear CSV).
- Un **schedule** (cron) → Lambda corre la tarea programada.
- Una request HTTP vía **API Gateway** → Lambda responde (API serverless).

```ts
// una Lambda que procesa mensajes de SQS (handler estándar)
import { SQSEvent } from "aws-lambda";

export const handler = async (event: SQSEvent): Promise<void> => {
  for (const record of event.Records) {
    const trabajo = JSON.parse(record.body);
    await enviarEmail(trabajo.to); // si lanza, AWS reintenta / manda a DLQ
  }
};
```

Las características que definen cuándo conviene: **escala automática** (de 0 a miles sin configurar nada), **pago por uso** (ideal para cargas intermitentes — si no se usa, no pagás), y **cero operación de servidores**. Las restricciones que hay que respetar: **stateless** (no guardes estado entre invocaciones — va en DynamoDB/Redis), **15 minutos máximo** (trabajo más largo → Fargate o partir en pasos), y **cold start** (la primera invocación tras inactividad tarda más; mitigable con *provisioned concurrency* si la latencia importa).

El error de criterio: meter una API web entera de alto tráfico constante en Lambda. Funciona, pero a tráfico sostenido suele salir **más caro y con peor latencia** (cold starts) que un Fargate. Lambda brilla en lo **event-driven y esporádico**, no en lo "siempre encendido".

**Ejercicios 4**
4.1 Nombrá tres triggers típicos de una Lambda en un backend.
4.2 ¿Por qué una Lambda debe ser stateless? ¿Dónde guardarías el estado que necesite persistir entre invocaciones?
4.3 ¿Qué es un cold start y en qué tipo de carga es más molesto?

---

## Módulo 5 — SQS: colas gestionadas (vs. BullMQ/Redis)

**Teoría.** **SQS (Simple Queue Service)** es la cola de mensajes gestionada de AWS. Conceptualmente es lo mismo que aprendiste con BullMQ/Redis (un productor encola, un consumidor procesa en background), pero **operado por AWS**: sin servidores que mantener, escala infinita, durable. Es el desacople asíncrono por excelencia en AWS.

Dos tipos:

- **Standard**: throughput casi ilimitado, entrega **al menos una vez** (puede duplicar) y orden **best-effort** (no garantizado). Para la mayoría de los casos.
- **FIFO**: orden estricto y **deduplicación en la entrega** (exactly-once *delivery* dentro de una ventana de 5 minutos), a cambio de menor throughput. Cuando el orden importa (procesar eventos de una misma cuenta en secuencia). Ojo: ese "exactly-once" es *de entrega y acotado a 5 minutos*, no una garantía absoluta de procesamiento único en tu lógica — ante reprocesos o ventanas más largas, la **idempotencia del consumidor sigue siendo la red de seguridad**.

Conceptos clave (varios ya los conocés del módulo de Redis):

- **Visibility timeout**: cuando un consumidor toma un mensaje, queda "invisible" un tiempo para que otros no lo procesen a la vez. Si el consumidor termina, lo borra; si falla (no lo borra a tiempo), vuelve a la cola y se reintenta.
- **Dead-letter queue (DLQ)**: tras N intentos fallidos, el mensaje va a una DLQ para inspección — exactamente el patrón del módulo de Redis y del archivo senior, acá nativo.
- **Idempotencia obligatoria**: como Standard entrega "al menos una vez", tus consumidores **deben** ser idempotentes (el concepto del senior y del módulo de Redis). Vas a recibir duplicados.

```ts
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";

const sqs = new SQSClient({});
// validá la env var al arrancar: process.env.X es string | undefined,
// y QueueUrl exige string (en --strict, pasarla directo no compila)
const QueueUrl = process.env.EMAIL_QUEUE_URL;
if (!QueueUrl) throw new Error("Falta EMAIL_QUEUE_URL");
// productor: encolar un trabajo (tu API lo hace y responde rápido)
await sqs.send(new SendMessageCommand({
  QueueUrl,
  MessageBody: JSON.stringify({ to: "ana@test.com", tipo: "bienvenida" }),
}));
// el consumidor suele ser una Lambda (módulo 4) disparada por la cola
```

**SQS vs. BullMQ/Redis** (la decisión de arquitecto): BullMQ te da más features de "jobs" (reintentos con backoff configurables, prioridades, jobs programados, scheduling fino) y lo controlás vos; SQS te da **cero operación, durabilidad y escala** gestionadas por AWS, integrado con Lambda. En un stack AWS, SQS + Lambda es el camino estándar para el procesamiento asíncrono; BullMQ tiene sentido si ya tenés Redis y querés control fino o no estás casado con AWS.

**Ejercicios 5**
5.1 ¿Cuál es la diferencia entre una cola SQS Standard y una FIFO? ¿Cuándo necesitás FIFO?
5.2 Como SQS Standard entrega "al menos una vez", ¿qué propiedad deben cumplir tus consumidores y por qué? (Conectá con el módulo de Redis/senior.)
5.3 ¿Qué hace el "visibility timeout" y qué pasa con un mensaje si el consumidor falla antes de borrarlo?

---

## Módulo 6 — EventBridge: el event bus y la arquitectura event-driven

**Teoría.** Acá los **eventos de dominio** del archivo senior se vuelven infraestructura. **EventBridge** es un **event bus**: un servicio donde tus servicios **publican eventos** ("PedidoConfirmado", "UsuarioRegistrado") y otros se **suscriben** para reaccionar — sin conocerse entre sí. Es el corazón de la **arquitectura event-driven (EDA)** en AWS.

La diferencia conceptual con SQS: SQS es **punto a punto** (un productor → una cola → un consumidor que la vacía). EventBridge es **publish/subscribe con enrutamiento**: un evento puede dispararse y llegar a **muchos** destinos distintos según **reglas** que filtran por el contenido del evento.

```ts
import { EventBridgeClient, PutEventsCommand } from "@aws-sdk/client-eventbridge";

const eb = new EventBridgeClient({});
// publicar un evento de dominio (tu servicio de pedidos, tras confirmar)
await eb.send(new PutEventsCommand({
  Entries: [{
    Source: "app.pedidos",
    DetailType: "PedidoConfirmado",
    Detail: JSON.stringify({ pedidoId: "123", usuarioId: 42, total: 9900 }),
    EventBusName: "app-bus",
  }],
}));
```

Sobre ese evento definís **reglas** que enrutan a destinos (*targets*): una regla manda `PedidoConfirmado` a una Lambda de Envíos, otra lo manda a una cola SQS de Facturación, otra a Analytics. El emisor **no sabe** quiénes escuchan — ese desacople es justo el de los eventos de dominio del senior, ahora entre servicios.

Esto habilita **choreography** (coreografía): cada servicio reacciona a eventos por su cuenta, sin un coordinador central — opuesto a **orchestration** (un servicio dirige el flujo paso a paso, ej. una Step Function). Y conecta con el patrón **Outbox** del senior: tu servicio guarda el cambio + un evento en la misma transacción de Postgres, y un proceso publica ese evento a EventBridge de forma confiable.

Features útiles: **schema registry** (documenta la forma de tus eventos), **archive & replay** (reprocesar eventos pasados), y **EventBridge Scheduler** (cron gestionado, alternativa a una Lambda programada).

**Ejercicios 6**
6.1 ¿Cuál es la diferencia fundamental entre SQS (punto a punto) y EventBridge (pub/sub con enrutamiento)?
6.2 Un evento `PedidoConfirmado` debe disparar: envío, facturación y analytics, cada uno en su servicio. ¿Por qué EventBridge encaja mejor que SQS acá?
6.3 Conectá con el archivo senior: ¿qué patrón usarías para publicar el evento a EventBridge de forma confiable (sin perderlo si falla el proceso) y por qué?

---

## Módulo 7 — Kinesis: streaming de datos (vs. EventBridge)

**Teoría.** **Kinesis Data Streams** es para **streams de datos de alto volumen**: flujos continuos de muchos eventos por segundo donde importan el **orden**, la **reproducibilidad** y que **varios consumidores** lean el mismo flujo de forma independiente. Casos: telemetría/IoT, clickstream (cada click de los usuarios), logs de aplicación, métricas en tiempo real, pipelines de datos para analytics o ML.

La diferencia con EventBridge —una pregunta clásica de arquitectura— está en el modelo:

| | **EventBridge** | **Kinesis** |
|---|---|---|
| Modelo | Enrutar/filtrar eventos a destinos | Stream ordenado de registros |
| Volumen | Moderado, eventos de negocio discretos | Altísimo (miles/seg), telemetría/analytics |
| Tamaño máx | 256 KB por evento | 1 MB por registro |
| Orden | No garantizado | Ordenado **por shard** (partition key) |
| Reproceso | Archive & replay | Retención (hasta 365 días), re-leer desde un punto |
| Consumidores | Reglas → targets | Múltiples consumidores leen el mismo stream a su ritmo |
| Pensado para | EDA de negocio, integración de servicios | Streaming de datos, analytics en tiempo real |

Kinesis se organiza en **shards** (particiones): el orden se garantiza **dentro** de un shard, y los registros se reparten por una **partition key** (ej. `usuarioId`, para que todos los eventos de un usuario vayan ordenados al mismo shard). Los consumidores llevan su propia posición de lectura (*checkpoint*), así que pueden re-procesar desde atrás.

El criterio: **EventBridge para eventos de negocio** que enrutás a servicios ("cuando pasa X, que reaccionen Y y Z"); **Kinesis para torrentes de datos** que necesitás ordenados, reproducibles y consumidos por varios pipelines. Si dudás entre los dos para eventos de aplicación normales, casi siempre la respuesta es EventBridge (menos operación). Kinesis aparece cuando hablás de **volumen y analytics**, no de "reaccioná a este evento". (Alternativa que verás: **MSK**, Kafka gestionado, cuando el equipo ya vive en Kafka.)

**Ejercicios 7**
7.1 Nombrá dos casos de uso donde Kinesis encaja mejor que EventBridge.
7.2 ¿Qué es un "shard" y cómo se relaciona con el orden de los registros y la partition key?
7.3 Tenés que enrutar el evento de negocio "UsuarioRegistrado" a tres servicios. ¿Kinesis o EventBridge? Justificá.

---

## Módulo 8 — DynamoDB: NoSQL gestionado a escala

**Teoría.** **DynamoDB** es la base **NoSQL** clave-valor / documento totalmente gestionada de AWS, diseñada para **escala masiva con latencia de milisegundos predecible**. El cambio mental respecto de Postgres es total: **modelás según tus patrones de acceso (las queries que vas a hacer), no normalizando**. En DynamoDB la desnormalización y la duplicación de datos son normales y deseables.

Conceptos núcleo:

- **Partition key (PK)**: determina en qué partición física vive el ítem; las lecturas/escrituras se distribuyen por ella. Elegir una PK de **alta cardinalidad** evita "hot partitions" (una partición saturada).
- **Sort key (SK)**: ordena los ítems dentro de una misma PK y habilita queries por rango. PK + SK forman la clave compuesta.
- **Single-table design**: el patrón avanzado (y contraintuitivo) de DynamoDB: guardar **múltiples tipos de entidad en una sola tabla**, diseñando PK/SK para resolver todos tus patrones de acceso con `Query`. Distinto de relacional, donde cada entidad es una tabla.
- **GSI (Global Secondary Index)**: un índice con **otra** PK/SK, para consultar por atributos distintos de la clave principal. **LSI**: misma PK, otra SK.
- **`Query` vs. `Scan`**: `Query` usa la clave y es eficiente; **`Scan` lee toda la tabla** y es carísimo/lento — evitalo en producción. Si necesitás `Scan` seguido, tu modelo de datos está mal diseñado.
- **Consistencia**: lecturas **eventualmente consistentes** (default, más baratas/rápidas) o **fuertemente consistentes** (ven la última escritura, cuestan más).
- **DynamoDB Streams**: un flujo de los cambios de la tabla (ideal para disparar Lambdas ante inserts/updates — y para implementar el patrón Outbox).

```ts
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";
import { DynamoDBDocumentClient, QueryCommand } from "@aws-sdk/lib-dynamodb";

// removeUndefinedValues evita el error típico al guardar ítems con campos undefined
const ddb = DynamoDBDocumentClient.from(new DynamoDBClient({}), {
  marshallOptions: { removeUndefinedValues: true },
});
// traer todas las tareas de un proyecto: PK = PROYECTO#1, SK empieza con TAREA#
const res = await ddb.send(new QueryCommand({
  TableName: "app",
  KeyConditionExpression: "PK = :pk AND begins_with(SK, :sk)",
  ExpressionAttributeValues: { ":pk": "PROYECTO#1", ":sk": "TAREA#" },
}));
```

**Cuándo DynamoDB y cuándo Postgres** (criterio de arquitecto): DynamoDB cuando los patrones de acceso son **conocidos y estables**, necesitás escala enorme y latencia constante, y estás en AWS (sesiones, carritos, feeds, IoT, estado de agentes). **Postgres** cuando tenés relaciones fuertes, queries ad-hoc/analíticas, joins, o integridad referencial. No es "uno mejor": es elegir por el problema. El error clásico es usar DynamoDB como si fuera relacional (muchas tablas, joins emulados, `Scan` por todos lados) — ahí Postgres era la respuesta.

**Ejercicios 8**
8.1 ¿Cuál es el principio de modelado de DynamoDB y en qué se opone al de una base relacional?
8.2 ¿Por qué hay que evitar `Scan` en producción y preferir `Query`? ¿Qué indica que necesitás `Scan` seguido?
8.3 Para cada caso, ¿DynamoDB o Postgres?: (a) sesiones de usuario a escala con acceso por clave; (b) reportes con joins y filtros ad-hoc; (c) integridad referencial entre pedidos, líneas y pagos.

---

## Módulo 9 — Observabilidad: CloudWatch, X-Ray y OpenTelemetry

**Teoría.** La teoría de los **tres pilares** (logs, métricas, traces) del archivo senior se vuelve concreta con las herramientas de AWS. En un sistema distribuido (varias Lambdas, Fargate, colas), sin observabilidad estás ciego.

- **CloudWatch Logs**: adonde van todos los logs de Lambda, ECS, etc. Acá rinden los **logs estructurados (JSON)** del módulo de Docker: CloudWatch Logs Insights los consulta y filtra ("todos los errores 500 de la última hora"). Sin JSON estructurado, esto es imposible.
- **CloudWatch Metrics & Alarms**: métricas (latencia, invocaciones, errores, profundidad de cola, CPU) y **alarmas** que disparan acciones (notificar, escalar) cuando se cruza un umbral. Acá aplicás lo del senior: mirá **percentiles (p95/p99), no promedios** — un promedio bajo esconde la cola lenta.
- **CloudWatch Dashboards**: tableros para ver el estado del sistema de un vistazo.
- **AWS X-Ray (tracing distribuido)**: sigue un request **a través de varios servicios** (API → cola → Lambda → DynamoDB) y te muestra cuánto tardó cada salto en un *service map*. Es la respuesta a "¿dónde se fue el tiempo?" en una arquitectura distribuida — imposible de responder solo con logs.
- **OpenTelemetry (OTel)**: el **estándar abierto y vendor-neutral** de instrumentación (el que menciona el senior). AWS lo soporta vía **ADOT (AWS Distro for OpenTelemetry)**. Ventaja estratégica: instrumentás una vez con OTel y podés mandar los datos a X-Ray, Datadog, Grafana o donde sea, sin atarte a AWS. Es la elección senior para no quedar *vendor-locked*.

El concepto que une todo (del senior): un **correlationId / trace ID** que atraviesa todos los servicios de un request, para reconstruir su recorrido completo. En distribuido, esto no es opcional.

**Ejercicios 9**
9.1 ¿Por qué los logs estructurados en JSON (del módulo de Docker) son clave para que CloudWatch Logs sea útil?
9.2 ¿Qué problema resuelve X-Ray que los logs por sí solos no pueden, en una arquitectura distribuida?
9.3 ¿Qué ventaja estratégica da instrumentar con OpenTelemetry en vez de atarte solo a las herramientas nativas de AWS?

---

## Módulo 10 — IAM y secretos: mínimo privilegio en la nube

**Teoría.** **IAM** es el sistema nervioso de la seguridad en AWS, y aplica el principio de **mínimo privilegio** del archivo senior de forma literal: cada componente recibe **solo** los permisos que necesita, nada más. Piezas:

- **Políticas (policies)**: documentos JSON que declaran qué acciones se permiten sobre qué recursos. Ej: "esta función puede leer de *esta* cola SQS y escribir en *esta* tabla DynamoDB" — y nada más.
- **Roles**: en vez de poner credenciales en el código, los servicios **asumen un rol** con sus permisos. Tu Lambda o tu task de Fargate tiene un **execution role / task role**: AWS le inyecta credenciales temporales automáticamente. **Nunca** hardcodees access keys en tu app — ese es el error de seguridad #1 en AWS (claves filtradas en git = cuenta comprometida y factura gigante).
- **Mínimo privilegio en la práctica**: no le des `AdministratorAccess` a todo "para que funcione". Empezás restrictivo y agregás permisos cuando hace falta. Un rol con permisos de más es una brecha esperando a pasar.

Para **secretos** (el `JWT_SECRET`, la `DATABASE_URL`):

- **AWS Secrets Manager**: almacena secretos cifrados, con rotación automática (ideal para credenciales de DB).
- **SSM Parameter Store**: similar, más simple y económico para config y secretos.

Tu app los lee en runtime (la Task Definition de Fargate o la Lambda los inyecta como variables de entorno o los pide al arrancar) — exactamente el principio del módulo de Docker: **secretos fuera del código y de la imagen, inyectados desde afuera**, ahora con el gestor de AWS. Combinado con el rol que da acceso solo a *esos* secretos, tenés defensa en profundidad.

**Mínimo privilegio también a nivel de red.** El mismo criterio de "solo lo necesario" aplica a la **VPC**: el **ALB** va en **subnets públicas** (es lo único expuesto a internet), mientras tu **cómputo (Fargate/Lambda) y tu base (RDS) viven en subnets privadas**, sin IP pública. El acceso entre piezas se abre con **security groups** puntuales: el SG de la DB solo acepta tráfico desde el SG de la app en el puerto de Postgres, no desde cualquier lado. Así, aunque alguien llegue al ALB, no toca la base directo. La práctica detallada (cómo se arma esa red con CDK) vive en el módulo siguiente, pero el **criterio** —exponer lo mínimo, todo lo demás privado— es parte del diseño senior.

**Ejercicios 10**
10.1 ¿Por qué tu Lambda/Fargate debe usar un **rol IAM** en vez de access keys hardcodeadas? ¿Qué riesgo evita?
10.2 ¿Cómo se aplica el principio de mínimo privilegio (del senior) al definir los permisos de un servicio en AWS?
10.3 ¿Dónde guardás la `DATABASE_URL` y el `JWT_SECRET` en AWS, y cómo llegan a tu app? Conectá con el módulo de Docker.

---

## Módulo 11 — Juntando todo: una arquitectura event-driven en AWS

**Teoría.** Cerramos integrando todo en una arquitectura realista, la que tendrías que diseñar (y defender) en una entrevista de Tech Lead. Tomemos el flujo "un usuario confirma un pedido":

```
Cliente
  │  HTTPS
  ▼
[ALB] → [Fargate: API NestJS]  ── escribe el pedido + evento (Outbox) → [PostgreSQL/RDS]
                │
                │ publica "PedidoConfirmado"
                ▼
         [EventBridge: app-bus]
            │           │             │
        regla 1     regla 2       regla 3
            ▼           ▼             ▼
      (1) Envíos   (2) Facturación  (3) Analytics

Leyenda de targets (cada regla enruta el evento a un destino independiente):
  (1) Envíos      → [Lambda Envíos] → [DynamoDB: estado de envíos]
  (2) Facturación → [SQS (buffer + DLQ)] → [Lambda Facturación]
  (3) Analytics   → [Kinesis] → pipeline de datos / ML

Observabilidad transversal: CloudWatch (logs/métricas/alarmas) + X-Ray/OTel (trazas)
Seguridad: cada componente con su rol IAM (mínimo privilegio), secretos en Secrets Manager
Red: ALB en subnet pública; Fargate, Lambda y RDS en subnets privadas (security groups)
```

Cómo se conecta con todo lo que estudiaste:

- La **API** (Fargate) es tu NestJS dockerizado, escalado horizontalmente, con healthcheck y graceful shutdown.
- Escribe en **Postgres (RDS)** usando el patrón **Outbox** (senior) para publicar el evento de forma confiable.
- **EventBridge** desacopla: Envíos, Facturación y Analytics reaccionan sin conocerse (choreography, eventos de dominio).
- El procesamiento asíncrono va en **Lambda** (event-driven, pago por uso); lo que necesita amortiguación o reintentos pasa por **SQS** (con DLQ e idempotencia).
- **Kinesis** alimenta el pipeline de datos de alto volumen (analytics).
- **DynamoDB** guarda estado de acceso rápido y conocido; **Postgres** la data relacional/transaccional.
- **CloudWatch + X-Ray/OTel** dan visibilidad de punta a punta; **IAM** asegura cada pieza.

El criterio que demuestra seniority (el hilo de todo el archivo senior): **no metas todo esto en un MVP**. Empezás con tu monolito modular en Fargate + Postgres; introducís EventBridge/SQS/Lambda/DynamoDB **cuando el dominio lo pida** (un módulo que necesita escalar distinto, un flujo asíncrono real, un volumen que justifica streaming). Saber **diseñar** esta arquitectura y, sobre todo, saber **cuándo NO usarla todavía**, es lo que separa a un Tech Lead de alguien que colecciona servicios de AWS.

**Ejercicios 11**
11.1 En la arquitectura de ejemplo, ¿por qué la API publica el evento vía Outbox + EventBridge en vez de llamar directo a los servicios de Envíos y Facturación?
11.2 ¿Por qué el procesamiento de facturación pasa por SQS antes de la Lambda, en vez de que EventBridge invoque la Lambda directo? (Pensá en amortiguación y reintentos.)
11.3 Sos Tech Lead de un MVP nuevo con un equipo chico. ¿Arrancarías con toda esta arquitectura distribuida? Justificá con el criterio del archivo senior.

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Que AWS opera la infraestructura por vos (provisión, parches, escalado, alta
    disponibilidad); vos solo describís qué querés. Regla senior: preferir lo gestionado
    por defecto, y solo ir a autogestionado si hay una razón concreta (costo a gran
    escala, control fino que el gestionado no da).
1.2 Porque las AZ son data centers físicamente independientes: si una falla (corte de
    luz, red), las réplicas en otras AZ siguen atendiendo. Concentrar todo en una sola AZ
    es un único punto de fallo.
1.3 Porque IaC versiona la infraestructura en el repo: es reproducible, revisable en un
    PR y aplicable idéntica en cada entorno. Hacer clicks en la consola es irreproducible
    e irrevisable, el mismo problema que tocar la base de prod a mano en vez de usar
    migraciones (módulo de Postgres/Docker).
```

### Módulo 2
```
2.1 (1) Límite de 15 minutos de ejecución. (2) Cold starts: latencia extra al arrancar
    tras inactividad, mala para una API que debe responder rápido y constante. (También:
    es stateless y a tráfico sostenido suele salir más caro.)
2.2 Fargate es el modo serverless de ECS: corrés contenedores sin administrar las
    máquinas (EC2) por debajo. Te ahorra aprovisionar, parchear, escalar y mantener
    servidores: solo declarás CPU/memoria y AWS pone la infra.
2.3 (a) Fargate (tráfico constante, sin cold starts). (b) Lambda (carga esporádica,
    event-driven al subir la imagen). (c) Lambda (tarea programada puntual, pago por uso).
```

### Módulo 3
```
3.1 Task Definition = la receta del contenedor (imagen, CPU/mem, env, puerto), como tu
    `docker run` con sus flags. Task = una instancia corriendo de esa definición, como un
    contenedor levantado. Service = mantiene N tasks vivas, las reemplaza y hace deploys
    sin downtime (el escalado horizontal operado por AWS).
3.2 El ALB reparte el tráfico entre las tasks y consulta su /health; si una task empieza
    a fallar el healthcheck, el ALB deja de enviarle tráfico (y el Service la reemplaza),
    evitando mandar requests a una instancia no sana.
3.3 Porque en un rolling deployment el Service baja las tasks viejas; con graceful
    shutdown (SIGTERM) esas tasks terminan los requests en curso y cierran conexiones
    antes de morir, en vez de cortarlos a la mitad. Así el deploy no genera errores.
```

### Módulo 4
```
4.1 Mensajes de SQS, eventos de EventBridge, archivos subidos a S3, un schedule (cron),
    o requests HTTP vía API Gateway. (Cualquier tres.)
4.2 Porque cada invocación es efímera e independiente: no hay memoria garantizada entre
    una y otra (pueden correr en entornos distintos). El estado que deba persistir va en
    DynamoDB, Redis o Postgres, no en variables del proceso.
4.3 Es la latencia extra de la primera invocación tras un período inactivo (AWS tiene que
    inicializar el entorno de ejecución). Es más molesto en cargas con latencia crítica y
    tráfico intermitente (una API interactiva), menos en procesamiento asíncrono.
```

### Módulo 5
```
5.1 Standard: throughput casi ilimitado, entrega al menos una vez (puede duplicar) y
    orden no garantizado. FIFO: orden estricto y deduplicación en la entrega (exactly-once
    delivery dentro de una ventana de 5 min), con menor throughput. Necesitás FIFO cuando
    el orden de los mensajes importa (ej. procesar en secuencia eventos de una misma
    entidad). Aun con FIFO, el consumidor debería ser idempotente para reprocesos o
    ventanas mayores a 5 min.
5.2 Deben ser idempotentes: como un mensaje puede entregarse más de una vez, procesarlo
    dos veces tiene que dar el mismo resultado que una (si no, duplicás cobros/emails).
    Es el mismo requisito del módulo de Redis y del archivo senior.
5.3 El mensaje queda invisible para otros consumidores durante ese tiempo, para que no se
    procese en paralelo. Si el consumidor falla y no lo borra antes de que expire el
    timeout, el mensaje vuelve a hacerse visible y se reintenta (y tras N intentos va a la
    DLQ).
```

### Módulo 6
```
6.1 SQS es punto a punto: un productor pone mensajes en una cola y un consumidor la
    vacía (cada mensaje lo procesa uno). EventBridge es pub/sub con enrutamiento: un
    evento puede llegar a MUCHOS destinos distintos según reglas que filtran por su
    contenido, y el emisor no conoce a los consumidores.
6.2 Porque un mismo evento (PedidoConfirmado) debe llegar a tres servicios independientes
    a la vez; con EventBridge definís tres reglas/targets y los tres reaccionan, sin que
    el emisor los conozca. Con SQS tendrías que publicar a tres colas a mano y acoplar el
    emisor a quiénes escuchan.
6.3 El patrón Outbox: guardás el cambio de negocio y el evento en la MISMA transacción de
    Postgres (atómico), y un proceso aparte publica los eventos del outbox a EventBridge.
    Así no se pierde el evento si el proceso falla entre guardar y publicar (dual write).
```

### Módulo 7
```
7.1 Telemetría/IoT, clickstream (clicks de usuarios), logs/métricas en tiempo real,
    pipelines de datos para analytics o ML: flujos de altísimo volumen, ordenados y
    reproducibles. (Cualquier dos.)
7.2 Un shard es una partición del stream; el orden se garantiza DENTRO de un shard. La
    partition key decide a qué shard va cada registro, así que usando una buena key (ej.
    usuarioId) todos los eventos de esa entidad caen ordenados en el mismo shard.
7.3 EventBridge: es un evento de negocio discreto que hay que enrutar a tres servicios.
    Kinesis está pensado para torrentes de datos de alto volumen con orden/replay, no para
    "reaccioná a este evento"; sería sobredimensionado y más operación.
```

### Módulo 8
```
8.1 Modelás según los patrones de acceso (las queries que harás), desnormalizando y
    duplicando datos para que cada query sea eficiente. Se opone al relacional, que
    normaliza (cada dato una vez) y arma las vistas con joins en tiempo de query.
8.2 Porque Scan lee TODA la tabla (costoso y lento, escala pésimo), mientras Query usa la
    clave y va directo a los ítems. Necesitar Scan seguido indica que el modelo de datos
    (PK/SK/GSI) no está diseñado para tus patrones de acceso.
8.3 (a) DynamoDB (acceso por clave a escala, latencia constante). (b) Postgres (joins y
    filtros ad-hoc). (c) Postgres (integridad referencial entre entidades relacionadas).
```

### Módulo 9
```
9.1 Porque CloudWatch Logs Insights puede parsear, filtrar y agregar JSON (buscar por
    campos: usuario, status, latencia). Con texto plano no podés consultar de forma
    estructurada; el log queda como una pared de texto poco útil para diagnosticar.
9.2 X-Ray sigue un request A TRAVÉS de varios servicios (API → cola → Lambda → DynamoDB)
    y muestra cuánto tardó cada salto en un service map. Los logs, dispersos por servicio,
    no reconstruyen ese recorrido ni dicen dónde se fue el tiempo en el flujo completo.
9.3 Que no quedás atado a un proveedor (vendor lock-in): instrumentás una vez con el
    estándar OTel y podés enviar los datos a X-Ray, Datadog, Grafana, etc., o cambiar de
    backend de observabilidad sin reinstrumentar la app.
```

### Módulo 10
```
10.1 Porque el rol le da credenciales temporales y rotadas automáticamente, sin secretos
     en el código. Hardcodear access keys arriesga que se filtren (ej. en git): con ellas,
     un atacante accede a tu cuenta AWS (datos y una factura potencialmente enorme).
10.2 Dándole a cada servicio una policy con SOLO los permisos que necesita sobre los
     recursos puntuales que usa (ej. leer esta cola y escribir esta tabla), en vez de
     permisos amplios tipo AdministratorAccess "para que ande". Menos permisos = menos
     superficie de ataque.
10.3 En AWS Secrets Manager o SSM Parameter Store (cifrados, fuera del código y de la
     imagen). Llegan a la app inyectados por la Task Definition (Fargate) o la config de
     la Lambda como variables de entorno, o pedidos al arrancar vía el rol IAM. Es el
     mismo principio del módulo de Docker: secretos inyectados desde afuera.
```

### Módulo 11
```
11.1 Para desacoplar: con Outbox + EventBridge, la API publica el hecho una vez de forma
     confiable y los servicios reaccionan sin que ella los conozca. Llamarlos directo la
     acoplaría a Envíos/Facturación (si uno está caído, falla el pedido) y la obligaría a
     conocer a todos los consumidores.
11.2 Porque SQS amortigua (absorbe picos sin saturar a Facturación) y da reintentos con
     DLQ: si el procesamiento falla, el mensaje vuelve y se reintenta, y tras N intentos
     va a la DLQ. Invocar la Lambda directo desde EventBridge da menos control de
     buffering y reintentos para ese trabajo.
11.3 No. Con un equipo chico y un dominio aún no entendido, esa arquitectura distribuida
     es complejidad accidental enorme (operación, consistencia eventual, observabilidad).
     Arrancaría con un monolito modular en Fargate + Postgres y extraería piezas
     (EventBridge/SQS/Lambda/DynamoDB) cuando un dolor concreto lo justifique. Saber
     cuándo NO usarla es criterio de Tech Lead.
```

---

## Siguientes pasos

Con este módulo tenés el mapa de AWS que pide un rol senior/Tech Lead: cuándo usar Lambda vs. Fargate, cómo desacoplar con SQS y EventBridge, cuándo Kinesis, cómo modelar en DynamoDB, y cómo observar y asegurar todo. Y, sobre todo, el criterio de **cuándo NO** distribuir todavía. El recorrido desde acá: **(1)** llevá tu "Task API" a **Fargate** (reusá tu imagen Docker + ECR) con CI/CD; **(2)** profundizá la **arquitectura event-driven aplicada** (event sourcing, CQRS, choreography vs. orchestration) como módulo propio; **(3)** sumá **NoSQL hands-on** (DynamoDB + MongoDB) y **Kubernetes** (cuándo K8s vs. Fargate) para completar el stack distribuido; **(4)** practicá **diseñar arquitecturas en voz alta** con trade-offs — es el corazón de la entrevista de Tech Lead. Todo se apoya en el criterio del archivo senior: la mejor arquitectura es la más simple que resuelve el problema real, no la que más servicios usa.
