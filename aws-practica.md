# AWS hands-on con CDK: infraestructura, servicios y costos

**De los conceptos a infra real en TypeScript · S3, RDS, networking, FinOps · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. El módulo anterior (**AWS para backend**) te dio el *criterio*: qué servicio elegir y cuándo. Este lo baja a **práctica con código**: definís tu infraestructura con **AWS CDK en TypeScript** (aprovechando lo que ya sabés), sumás los servicios fundamentales que faltaban (**S3, RDS/Aurora, API Gateway, SNS, ElastiCache, Step Functions**), entendés el **networking (VPC)** que sostiene todo, y aprendés a **controlar costos (FinOps)** y a evaluar una arquitectura con el **Well-Architected Framework**. El código CDK de acá compila en TypeScript estricto, igual que el resto del temario.

**Lo que asumimos.** El módulo de **AWS para backend** (criterio de Lambda/Fargate/SQS/EventBridge/Kinesis/DynamoDB/IAM), tu "Task API" dockerizada, y tus módulos de PostgreSQL y Redis (RDS y ElastiCache son sus versiones gestionadas).

**Para practicar.** AWS CLI configurado (`aws configure`), Node, y CDK:

```bash
npm i -g aws-cdk          # la CLI de CDK
cdk init app --language typescript   # nuevo proyecto de infra
npm i aws-cdk-lib constructs         # la librería de constructs (v2)
```

**Índice de módulos**
1. Infraestructura como código con CDK
2. Networking: VPC, subnets y security groups
3. S3: object storage (el servicio más fundamental)
4. RDS / Aurora: PostgreSQL gestionado
5. ElastiCache: Redis gestionado
6. API Gateway + Lambda: una API serverless
7. SNS y el patrón fan-out (SNS → SQS)
8. Step Functions: orquestar sagas
9. Costos y FinOps: pensar en plata
10. AWS Well-Architected Framework (los 6 pilares)
11. Caso integrador: la "Task API" entera en CDK

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Infraestructura como código con CDK

**Teoría.** Crear recursos a mano en la consola de AWS es irreproducible, irrevisable y propenso a error — el mismo problema que tocar la base de producción sin migraciones. La solución es **infraestructura como código (IaC)**: tu infra vive en el repo, versionada, revisada en un PR y aplicable idéntica en cada entorno.

**AWS CDK** (Cloud Development Kit) te deja definir la infra en **TypeScript** (no en YAML), con tipos, autocompletado y abstracciones reutilizables. Por debajo, CDK genera **CloudFormation** (el motor de IaC nativo de AWS) y lo despliega. La alternativa es **Terraform** (multi-cloud, lenguaje propio HCL); CDK gana cuando estás en AWS y querés usar tu lenguaje.

Conceptos:

- **App**: la raíz de tu proyecto CDK.
- **Stack**: una unidad de despliegue (≈ un CloudFormation stack). Agrupás recursos relacionados.
- **Construct**: la pieza reutilizable; representa uno o varios recursos. Vienen en niveles (L1 = mapeo directo de CloudFormation; L2 = abstracciones con defaults sensatos; L3/patterns = arquitecturas completas).

```ts
import * as cdk from "aws-cdk-lib";
import { Construct } from "constructs";
import * as s3 from "aws-cdk-lib/aws-s3";

export class StorageStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // un construct L2: un bucket S3 con buenos defaults
    new s3.Bucket(this, "Uploads", {
      versioned: true,
      encryption: s3.BucketEncryption.S3_MANAGED,
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    });
  }
}
```

El flujo de trabajo:

```bash
cdk synth     # genera el CloudFormation (ves qué se va a crear)
cdk diff      # muestra el cambio respecto de lo desplegado (como un git diff de infra)
cdk deploy    # aplica los cambios a AWS
cdk destroy   # borra todo lo del stack (clave para no dejar recursos costando)
```

`cdk diff` antes de `deploy` es tu red de seguridad: ves exactamente qué cambia antes de tocar producción. Esto se integra al **CI/CD** (módulo de Docker): el pipeline corre `cdk deploy` tras los tests.

**Ejercicios 1**
1.1 ¿Qué ventajas da definir la infraestructura con CDK en vez de crear los recursos a mano en la consola? Mencioná dos.
1.2 ¿Qué hacen `cdk synth`, `cdk diff` y `cdk deploy`? ¿Por qué `cdk diff` es importante antes de un deploy a producción?
1.3 ¿Cuál es la diferencia entre un Stack y un Construct en CDK?

---

## Módulo 2 — Networking: VPC, subnets y security groups

**Teoría.** Casi todo en AWS vive dentro de una **VPC (Virtual Private Cloud)**: tu red privada y aislada. Entenderla importa porque define **qué puede hablar con qué** (seguridad) y es fuente de **costos sorpresa** (módulo 9).

Las piezas:

- **Subnets**: subdivisiones de la VPC, repartidas en varias AZ. **Públicas** (con ruta a internet, para el balanceador) y **privadas** (sin acceso directo desde afuera, para tu app y tu base). La regla de oro: **tu base de datos va en subnet privada**, jamás expuesta a internet.
- **Internet Gateway**: la puerta de la VPC hacia internet (para las subnets públicas).
- **NAT Gateway**: deja que los recursos en subnets privadas **salgan** a internet (ej. tu app llamando a una API externa) sin ser accesibles **desde** internet. ⚠️ **Cuesta caro** (tarifa por hora + por GB procesado): es uno de los costos sorpresa más comunes.
- **Security Groups**: firewalls a nivel de recurso. Definen qué tráfico entra/sale. Por defecto, **todo bloqueado**: abrís solo lo necesario (mínimo privilegio del archivo senior, aplicado a la red). Ej: el security group de la base solo acepta conexiones **desde** el security group de la app, en el puerto 5432.

```ts
import * as ec2 from "aws-cdk-lib/aws-ec2";

// CDK arma subnets públicas y privadas en 2 AZ automáticamente
const vpc = new ec2.Vpc(this, "Vpc", {
  maxAzs: 2,
  natGateways: 1, // OJO: cada NAT Gateway cuesta ~$32/mes + tráfico. 1 alcanza para dev.
});
```

La arquitectura típica de un backend: el **ALB** en subnets públicas (recibe el tráfico de internet), tu **app (Fargate)** y tu **base (RDS)** en subnets privadas, y un security group que solo deja a la app hablar con la base. Así, aunque alguien conozca la dirección de tu base, no le llega desde afuera.

**Ejercicios 2**
2.1 ¿Por qué tu base de datos debe ir en una subnet privada y no en una pública?
2.2 ¿Qué hace un NAT Gateway y por qué es un costo a vigilar?
2.3 ¿Cómo aplicarías mínimo privilegio con security groups entre tu app y tu base de datos?

---

## Módulo 3 — S3: object storage (el servicio más fundamental)

**Teoría.** **S3 (Simple Storage Service)** es almacenamiento de **objetos** (archivos) prácticamente infinito, durable y barato. Es uno de los servicios más usados de AWS y aparece en casi toda arquitectura. No es un filesystem ni una base: guardás **objetos** (archivo + metadata) en **buckets** (contenedores con nombre único global), identificados por una **key** (la "ruta").

Casos de uso típicos en backend:

- **Archivos de usuarios**: avatares, adjuntos, documentos (en vez de guardarlos en tu servidor, que no escala ni persiste en contenedores efímeros).
- **Hosting estático**: servir un frontend (HTML/JS/CSS) o assets, normalmente detrás de **CloudFront** (la CDN de AWS).
- **Backups, logs, data lake**: destino de datos para analytics/ML.
- **Disparar eventos**: subir un archivo puede disparar una **Lambda** (procesar imagen, parsear CSV) — el trigger que viste en el módulo de cómputo.

El patrón clave para backend: **presigned URLs**. En vez de que el archivo del usuario pase por tu servidor (consumiendo ancho de banda y memoria), tu backend genera una **URL firmada temporal** y el cliente sube/baja **directo a S3**. Tu app solo autoriza; no toca el archivo.

```ts
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3 = new S3Client({});
// tu endpoint devuelve esta URL; el cliente sube el archivo directo a S3 con ella
const url = await getSignedUrl(
  s3,
  new PutObjectCommand({ Bucket: "uploads", Key: `avatares/${usuarioId}.jpg` }),
  { expiresIn: 3600 }, // válida por 1 hora
);
```

Buenas prácticas: **bloquear acceso público** por defecto (un bucket público mal configurado es una de las filtraciones de datos más comunes de la historia de AWS), **cifrado** (S3 cifra por defecto), **versionado** para recuperar archivos borrados, y **lifecycle policies** para mover datos viejos a clases más baratas (S3 Glacier) o borrarlos automáticamente.

**Ejercicios 3**
3.1 ¿Por qué conviene que el cliente suba archivos directo a S3 con una presigned URL en vez de pasarlos por tu backend?
3.2 Nombrá tres casos de uso de S3 en un backend.
3.3 ¿Por qué "bloquear acceso público" es una buena práctica por defecto en un bucket?

---

## Módulo 4 — RDS / Aurora: PostgreSQL gestionado

**Teoría.** En el módulo de Docker corrías Postgres en un contenedor; el de AWS te recordó que la base de **producción** no va en un contenedor efímero. **RDS (Relational Database Service)** es Postgres (o MySQL, etc.) **gestionado**: AWS se encarga de backups, parches, alta disponibilidad y réplicas. **Aurora** es la versión de AWS compatible con Postgres/MySQL, más performante y escalable (y con **Aurora Serverless v2**, que escala la capacidad automáticamente).

Lo que te da RDS/Aurora sobre "Postgres en un contenedor":

- **Multi-AZ**: una réplica en standby en otra AZ; si la primaria falla, failover automático. Alta disponibilidad sin que hagas nada.
- **Backups automáticos** y **point-in-time recovery**: restaurar la base a cualquier momento de los últimos N días.
- **Read replicas**: réplicas de solo lectura para repartir la carga de lecturas (conecta con el N+1 y la performance del archivo senior: las lecturas pesadas van a réplicas).
- **Parches y mantenimiento** gestionados por AWS.

```ts
import * as rds from "aws-cdk-lib/aws-rds";

const db = new rds.DatabaseInstance(this, "Postgres", {
  engine: rds.DatabaseInstanceEngine.postgres({
    version: rds.PostgresEngineVersion.VER_16,
  }),
  vpc,
  vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS }, // privada (módulo 2)
  instanceType: ec2.InstanceType.of(ec2.InstanceClass.T3, ec2.InstanceSize.MICRO),
  multiAz: false, // true en producción para alta disponibilidad
  credentials: rds.Credentials.fromGeneratedSecret("postgres"), // genera y guarda en Secrets Manager
});
```

Fijate la sinergia: `fromGeneratedSecret` crea la contraseña y la guarda en **Secrets Manager** (módulo de IAM/secretos del módulo anterior), y la base va en **subnet privada** (módulo 2). Tu app (Fargate) lee la `DATABASE_URL` desde ese secreto; nada de credenciales en el código. Tus migraciones de Drizzle (módulo de Postgres) corren contra esta base en el paso de release del pipeline.

**Cuándo RDS y cuándo Aurora**: RDS Postgres para la mayoría de los casos (más barato, suficiente); Aurora cuando necesitás más performance, escala de lecturas, o el autoscaling de Serverless v2. Lo que **no** cambia: tu código sigue hablando Postgres; cambia solo la infraestructura que lo opera.

**Ejercicios 4**
4.1 Nombrá tres cosas que RDS/Aurora te da sobre correr Postgres vos mismo en un contenedor.
4.2 ¿Qué problema de performance (del archivo senior) ayudan a resolver las read replicas?
4.3 ¿Cómo llega la contraseña de la base a tu app sin hardcodearla? Conectá con Secrets Manager (módulo anterior).

---

## Módulo 5 — ElastiCache: Redis gestionado

**Teoría.** En el módulo de Redis corrías Redis en tu `docker-compose`. **ElastiCache** es Redis (o Memcached) **gestionado** por AWS: la misma herramienta, sin que operes el servidor. Todo lo que aprendiste —caché cache-aside, sesiones, rate limiting, estado compartido para apps stateless— aplica igual; cambia solo quién opera la infra.

Lo que ElastiCache agrega sobre "Redis en un contenedor":

- **Alta disponibilidad**: réplicas con failover automático (modo cluster o réplica primaria/secundaria).
- **Escalado**: agregar réplicas de lectura o shards sin downtime.
- **Backups, parches y monitoreo** gestionados.

El caso de uso que cierra el círculo del escalado horizontal (módulos de Node y Redis): cuando tu API corre en **varias tasks de Fargate**, el caché, las sesiones y el store de refresh tokens **no pueden vivir en la memoria de cada task** (cada request cae en una distinta). Van en **ElastiCache**, compartido por todas. Es la pieza de "estado compartido externo" que hace posible escalar la app a N instancias.

```ts
// ElastiCache se define con constructs L1 (CfnReplicationGroup) o L2 según versión de CDK.
// La idea: un cluster Redis en subnets privadas, accesible solo desde el SG de la app.
// Tu app lee REDIS_URL del entorno y usa ioredis/BullMQ igual que en local.
```

El criterio (igual que con RDS): en local/dev, Redis en Docker; en producción AWS, ElastiCache. Tu código (ioredis, BullMQ, cache-manager) no cambia: cambia el endpoint. Y como con la base, va en **subnet privada** con un security group que solo deja entrar a tu app.

**Ejercicios 5**
5.1 ¿Qué te da ElastiCache sobre correr Redis vos mismo en un contenedor?
5.2 Conectá con el escalado horizontal: ¿por qué el caché y las sesiones deben ir en ElastiCache y no en la memoria de cada task de Fargate?
5.3 ¿Cambia tu código de aplicación (ioredis/BullMQ) al pasar de Redis en Docker a ElastiCache? ¿Qué cambia?

---

## Módulo 6 — API Gateway + Lambda: una API serverless

**Teoría.** En el módulo anterior viste que una API de tráfico constante vive cómoda en Fargate. Pero para APIs **event-driven, esporádicas o de bajo tráfico**, existe la opción **100% serverless**: **API Gateway** (recibe las requests HTTP) + **Lambda** (las procesa). Sin servidores, escala de 0 a miles, pagás por request.

- **API Gateway** expone endpoints HTTP/REST, maneja routing, throttling, autenticación (puede validar JWT con un *authorizer*), CORS y rate limiting — varias cosas que en NestJS hacías con guards/throttler, acá a nivel de infraestructura.
- Cada ruta dispara una **Lambda** que responde.

Dos sabores: **REST API** (más features: API keys, modelos, etc.) y **HTTP API** (más simple, barata y rápida — la opción por defecto en 2026 salvo que necesites algo de REST API).

¿Y tu NestJS? Podés correr una **app NestJS completa en Lambda** con el **Lambda Web Adapter** (o adaptadores tipo `@codegenie/serverless-express`): tu mismo código de Nest corre detrás de API Gateway, sin reescribir. Mitigás los cold starts con **provisioned concurrency** o **SnapStart**.

```ts
import * as lambda from "aws-cdk-lib/aws-lambda";
import * as apigw from "aws-cdk-lib/aws-apigatewayv2";
import { HttpLambdaIntegration } from "aws-cdk-lib/aws-apigatewayv2-integrations";

const fn = new lambda.Function(this, "ApiFn", {
  runtime: lambda.Runtime.NODEJS_22_X,
  handler: "main.handler",
  code: lambda.Code.fromAsset("dist"),
});

const api = new apigw.HttpApi(this, "HttpApi");
api.addRoutes({
  path: "/tareas",
  methods: [apigw.HttpMethod.GET],
  integration: new HttpLambdaIntegration("TareasInt", fn),
});
```

El criterio de elección (del módulo anterior, ahora concreto): **API Gateway + Lambda** para APIs internas, de bajo/intermitente tráfico, MVPs, o partes event-driven; **ALB + Fargate** para tu API principal de tráfico sostenido donde los cold starts y el costo por request a escala no convienen. Muchas arquitecturas combinan ambos.

**Ejercicios 6**
6.1 ¿Qué responsabilidades (que en NestJS hacías con guards/throttler) puede manejar API Gateway a nivel de infraestructura?
6.2 ¿Cómo correrías tu app NestJS completa detrás de API Gateway sin reescribirla?
6.3 ¿Cuándo elegirías API Gateway + Lambda y cuándo ALB + Fargate para una API?

---

## Módulo 7 — SNS y el patrón fan-out (SNS → SQS)

**Teoría.** **SNS (Simple Notification Service)** es pub/sub: publicás un mensaje en un **topic** y se entrega a **todos** los suscriptores a la vez (fan-out). Suscriptores: colas SQS, Lambdas, emails, SMS, HTTP endpoints.

¿En qué se diferencia de EventBridge (módulo anterior)? Ambos hacen pub/sub, pero:

- **SNS**: fan-out simple y de **altísimo throughput**, sin reglas de filtrado complejas (tiene filtros básicos por atributos). Pensado para "mandá esto a estos N destinos, rápido".
- **EventBridge**: enrutamiento **rico por contenido**, integración con SaaS/servicios AWS, schema registry, archive & replay. Pensado para EDA de negocio.

El patrón clásico es **SNS → SQS fan-out**: un topic SNS con **varias colas SQS** suscriptas. Cada servicio consumidor tiene **su propia cola** (con su propio ritmo, reintentos y DLQ), y el evento llega a todas. Combina lo mejor: el fan-out de SNS + la durabilidad/amortiguación/reintentos de SQS.

```ts
import * as sns from "aws-cdk-lib/aws-sns";
import * as sqs from "aws-cdk-lib/aws-sqs";
import * as subs from "aws-cdk-lib/aws-sns-subscriptions";

const topic = new sns.Topic(this, "PedidoConfirmado");

// cada servicio consumidor tiene su cola, suscrita al mismo topic
const colaFacturacion = new sqs.Queue(this, "Facturacion");
const colaEnvios = new sqs.Queue(this, "Envios");

topic.addSubscription(new subs.SqsSubscription(colaFacturacion));
topic.addSubscription(new subs.SqsSubscription(colaEnvios));
// publicar en el topic → llega a AMBAS colas; cada consumidor procesa a su ritmo
```

El criterio: **SNS→SQS** cuando querés fan-out durable y de alto volumen con consumidores desacoplados (cada uno con su buffer y reintentos); **EventBridge** cuando necesitás enrutamiento por contenido, integración con muchos servicios y replay. Para eventos de dominio de tu app, EventBridge suele ganar por features; SNS→SQS gana en throughput puro y simplicidad de fan-out.

**Ejercicios 7**
7.1 ¿Qué es el patrón fan-out SNS → SQS y qué ventaja combina?
7.2 ¿Por qué darle a cada servicio consumidor su propia cola SQS en vez de que todos lean del topic directamente?
7.3 ¿Cuándo elegirías SNS y cuándo EventBridge para distribuir un evento?

---

## Módulo 8 — Step Functions: orquestar sagas

**Teoría.** En el archivo senior viste las **sagas** (transacciones distribuidas con compensaciones) y la distinción **coreografía** (cada servicio reacciona a eventos, sin coordinador) vs **orquestación** (un coordinador dirige el flujo paso a paso). **AWS Step Functions** es el servicio de orquestación: definís una **máquina de estados** (los pasos, las ramas, los reintentos, las compensaciones) y AWS la ejecuta de forma confiable, guardando el estado de cada ejecución.

Por qué importa: la coreografía con eventos (EventBridge) es genial para desacoplar, pero cuando un flujo tiene **muchos pasos con orden, condiciones y compensaciones** (reservar stock → cobrar → agendar envío, y deshacer si algo falla), seguir el flujo entre N servicios que reaccionan a eventos se vuelve imposible de razonar y debuggear. Step Functions hace el flujo **explícito y visible**: ves el diagrama, en qué paso está cada ejecución, qué falló y qué se compensó.

```
Reservar stock ──ok──▶ Cobrar pago ──ok──▶ Agendar envío ──▶ ✅ Fin
      │                     │
   falla                 falla ──▶ Compensar: liberar stock ──▶ ❌ Fin
```

- A favor: flujo explícito, reintentos y manejo de errores declarativos, visibilidad total, estado durable (ejecuciones de hasta 1 año).
- En contra: acopla el flujo en un lugar central (menos "puro" que la coreografía), y agrega un servicio más.
- Cuándo: flujos distribuidos **complejos** con compensaciones y necesidad de trazabilidad (justo el caso de saga del senior). Para reaccionar a un evento simple, EventBridge alcanza; no orquestes lo que no lo necesita.

El criterio que cierra el tema del senior: **coreografía** (EventBridge) para desacople entre pocos servicios y flujos simples; **orquestación** (Step Functions) cuando el flujo es complejo, tiene compensaciones y necesitás verlo y debuggearlo. AWS mismo recomienda Step Functions para implementar el patrón Saga.

**Ejercicios 8**
8.1 ¿Qué problema de la coreografía (eventos sueltos) resuelve la orquestación con Step Functions?
8.2 Conectá con el archivo senior: ¿cómo implementarías el "deshacer" de una saga (compensación) en una máquina de estados?
8.3 ¿Cuándo NO usarías Step Functions y te quedarías con coreografía vía EventBridge?

---

## Módulo 9 — Costos y FinOps: pensar en plata

**Teoría.** Un Tech Lead no decide solo por lo técnico: decide por **costo**. AWS cobra por casi todo, y las arquitecturas "elegantes" pueden salir carísimas si no medís. **FinOps** es la disciplina de hacer el costo visible y optimizarlo como parte del diseño. Los **sospechosos de siempre** (fugas de costo clásicas):

- **NAT Gateways**: ~$32/mes cada uno + tarifa por GB procesado. Tener uno por AZ "por las dudas" en dev es plata tirada. Alternativas: VPC endpoints para hablar con servicios AWS sin pasar por NAT.
- **Egress (transferencia de salida)**: mandar datos **fuera** de AWS (o entre regiones/AZ) se paga. Arquitecturas "chatty" (servicios que se mandan mucho tráfico entre sí cruzando AZ) acumulan costo invisible.
- **On-demand mal dimensionado**: instancias RDS/ECS sobredimensionadas corriendo 24/7. Right-sizing + **Savings Plans / Reserved Instances** para cargas estables bajan el costo fuerte.
- **Recursos olvidados**: un entorno de pruebas que quedó prendido, volúmenes EBS huérfanos, snapshots viejos. El `cdk destroy` es tu amigo.
- **Lambda mal usada**: a tráfico altísimo y constante, Lambda puede salir más cara que Fargate (y al revés en tráfico esporádico). El costo es parte del criterio Lambda-vs-Fargate del módulo anterior.
- **Logs y datos sin lifecycle**: CloudWatch Logs y S3 sin políticas de retención acumulan costo; configurá expiración y clases de almacenamiento.

Herramientas: **Cost Explorer** (analizar el gasto), **Budgets** (alertas cuando proyectás pasarte), **tags** de costo (etiquetar recursos por proyecto/equipo para saber quién gasta qué), y el **pricing calculator** para estimar **antes** de construir.

El criterio senior: **estimá el costo en la fase de diseño**, no cuando llega la factura; elegí lo gestionado pero dimensionalo; y aplicá el mismo "medí antes de optimizar" del senior, también a la plata. Un Tech Lead que propone una arquitectura sabe **cuánto va a costar** y lo justifica.

**Ejercicios 9**
9.1 Nombrá tres fugas de costo clásicas en AWS y cómo las mitigarías.
9.2 ¿Por qué una arquitectura "chatty" entre servicios en distintas AZ puede generar costo invisible?
9.3 ¿Qué herramienta usarías para que te avise antes de pasarte del presupuesto, y cómo sabés qué proyecto gasta qué?

---

## Módulo 10 — AWS Well-Architected Framework (los 6 pilares)

**Teoría.** AWS publica el **Well-Architected Framework**: seis pilares para evaluar y diseñar arquitecturas. Es una **checklist de criterio** que aparece seguido en entrevistas senior/Tech Lead, y un buen marco para ordenar todo lo que estudiaste:

1. **Excelencia operacional**: poder operar y mejorar el sistema. IaC (CDK), CI/CD, observabilidad, runbooks, automatización. *(Tus módulos de Docker, observabilidad.)*
2. **Seguridad**: mínimo privilegio (IAM), datos cifrados, secretos gestionados, defensa en profundidad, todo en subnets privadas. *(Tus módulos de auth, IAM, networking.)*
3. **Confiabilidad (Reliability)**: que el sistema resista fallos. Multi-AZ, healthchecks, auto-recovery, reintentos con backoff, DLQ, graceful shutdown. *(Resiliencia del senior, Multi-AZ de RDS.)*
4. **Eficiencia de performance**: usar los recursos adecuados. Elegir el servicio correcto (Lambda vs Fargate), caché, índices, read replicas, autoscaling. *(Performance del senior, ElastiCache.)*
5. **Optimización de costos**: no gastar de más. Right-sizing, serverless donde conviene, Savings Plans, eliminar lo ocioso. *(El módulo 9.)*
6. **Sostenibilidad**: minimizar el impacto ambiental (eficiencia de recursos, regiones, serverless que escala a cero). El pilar más nuevo.

Lo valioso no es memorizar los nombres, sino **usarlos como lente**: ante cualquier diseño, preguntarte por cada pilar —"¿qué pasa si se cae una AZ? (confiabilidad), ¿quién puede acceder a esto? (seguridad), ¿cuánto cuesta? (costos), ¿cómo lo opero y observo? (operacional)"—. Eso es, literalmente, pensar como arquitecto. Notá cómo cada pilar mapea a temas que ya estudiaste: el framework no te enseña algo nuevo, **te ordena el criterio** que venís construyendo.

**Ejercicios 10**
10.1 Nombrá los 6 pilares del Well-Architected Framework y una práctica concreta de cada uno.
10.2 Para el pilar de **confiabilidad**, ¿qué tres mecanismos (vistos en el temario) aplicarías a tu "Task API" en AWS?
10.3 ¿Por qué el framework es útil como "lente" al revisar un diseño, más allá de memorizar los pilares?

---

## Módulo 11 — Caso integrador: la "Task API" entera en CDK

**Teoría.** Cerramos juntando todo: definir tu "Task API" de producción en un stack CDK, aplicando los dos módulos de AWS. La arquitectura:

```
Internet
   │
[API Gateway / ALB]
   │
[Fargate: NestJS]──┬── RDS Postgres (Multi-AZ, subnet privada)
   │               ├── ElastiCache Redis (caché, sesiones, BullMQ)
   │               └── S3 (archivos, presigned URLs)
   │ publica eventos
   ▼
[EventBridge / SNS] ──▶ SQS ──▶ Lambda (procesamiento async)
                                   │
                            [Step Functions] (sagas complejas)

Transversal: CloudWatch + X-Ray/OTel (observabilidad) · IAM + Secrets Manager (seguridad)
Todo en una VPC con subnets privadas · definido en CDK · desplegado por CI/CD
```

El esqueleto en CDK (constructs L2/patterns hacen el trabajo pesado):

```ts
export class TaskApiStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, "Vpc", { maxAzs: 2, natGateways: 1 });

    const db = new rds.DatabaseInstance(this, "Db", {
      engine: rds.DatabaseInstanceEngine.postgres({ version: rds.PostgresEngineVersion.VER_16 }),
      vpc, vpcSubnets: { subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS },
      credentials: rds.Credentials.fromGeneratedSecret("postgres"),
    });

    const bucket = new s3.Bucket(this, "Uploads", {
      blockPublicAccess: s3.BlockPublicAccess.BLOCK_ALL,
    });

    // API NestJS en Fargate detrás de un ALB, con la imagen Docker del repo
    const api = new ecsPatterns.ApplicationLoadBalancedFargateService(this, "Api", {
      vpc, cpu: 256, memoryLimitMiB: 512, desiredCount: 2,
      taskImageOptions: {
        image: ecs.ContainerImage.fromAsset("."), // tu Dockerfile multi-stage
        environment: { DATABASE_HOST: db.dbInstanceEndpointAddress },
        secrets: { DB_PASSWORD: ecs.Secret.fromSecretsManager(db.secret!) },
      },
    });

    db.connections.allowDefaultPortFrom(api.service); // SG: solo la app habla con la base
    bucket.grantReadWrite(api.taskDefinition.taskRole); // IAM: la app accede al bucket
    api.service.autoScaleTaskCount({ maxCapacity: 6 }).scaleOnCpuUtilization("Cpu", {
      targetUtilizationPercent: 70,
    });
  }
}
```

Mirá cuánto del temario se condensa acá: tu **imagen Docker** (`fromAsset`), la **base privada** con contraseña en **Secrets Manager**, el **security group** de mínimo privilegio (`allowDefaultPortFrom`), el **IAM** acotado (`grantReadWrite`), el **autoscaling** (escalado horizontal del módulo de Node), todo como **IaC** versionado. Y, fiel al criterio del senior: **este stack es para cuando lo necesitás**. Un MVP arranca con tu monolito en Fargate + RDS; sumás EventBridge/SQS/Lambda/Step Functions cuando el dominio lo pida. Saber escribir esto **y** saber cuándo todavía no hace falta es lo que define a un Tech Lead.

**Ejercicios 11**
11.1 En el stack de ejemplo, ¿qué hacen `db.connections.allowDefaultPortFrom(api.service)` y `bucket.grantReadWrite(...)`, y qué principio aplican?
11.2 ¿Cómo llega tu imagen Docker (la del módulo de Docker) a correr en Fargate dentro de este stack?
11.3 Sos Tech Lead de un MVP. ¿Desplegarías este stack completo desde el día uno? Justificá con el criterio del archivo senior.

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 (1) Reproducibilidad/versionado: la infra vive en el repo, se revisa en PR y se
    aplica idéntica en cada entorno. (2) Tipos y abstracciones de CDK (autocompletado,
    constructs reutilizables) y un historial de cambios, vs clicks irreproducibles en la
    consola.
1.2 synth genera el CloudFormation (qué se va a crear); diff muestra el cambio respecto de
    lo desplegado; deploy lo aplica. diff es clave antes de prod porque ves exactamente
    qué va a cambiar (recursos creados/modificados/borrados) antes de tocar nada.
1.3 Un Stack es una unidad de despliegue que agrupa recursos relacionados (≈ un
    CloudFormation stack). Un Construct es la pieza reutilizable que representa uno o
    varios recursos; los componés dentro de los stacks.
```

### Módulo 2
```
2.1 Para que no sea accesible desde internet: una base en subnet pública queda expuesta a
    ataques. En subnet privada solo le llega tráfico desde dentro de la VPC (tu app), no
    desde afuera.
2.2 Permite que recursos en subnets privadas salgan a internet (ej. llamar a una API
    externa) sin ser accesibles desde afuera. Es un costo a vigilar porque cobra por hora
    (~$32/mes c/u) más por GB procesado; varios NAT "por las dudas" inflan la factura.
2.3 Dándole a la base un security group que solo acepte conexiones DESDE el security group
    de la app, en el puerto de Postgres (5432), y nada más. Así, aunque se conozca la
    dirección de la base, solo la app puede conectarse.
```

### Módulo 3
```
3.1 Porque el archivo no pasa por tu servidor: ahorrás ancho de banda, memoria y CPU, y
    escala mejor (S3 maneja la subida directo). Tu backend solo genera la URL firmada
    (autoriza), no procesa el archivo.
3.2 Archivos de usuarios (avatares, adjuntos), hosting estático de un frontend/assets
    (con CloudFront), backups/logs/data lake, o disparar Lambdas al subir un archivo.
    (Cualquier tres.)
3.3 Porque un bucket público mal configurado es una de las causas más comunes de
    filtración de datos: cualquiera en internet podría listar/leer los objetos. Por
    defecto se bloquea el acceso público y se expone solo lo necesario (vía presigned URLs
    o CloudFront).
```

### Módulo 4
```
4.1 Backups automáticos y point-in-time recovery; alta disponibilidad con Multi-AZ
    (failover automático); read replicas para escalar lecturas; y parches/mantenimiento
    gestionados por AWS. (Cualquier tres.)
4.2 El problema de las lecturas pesadas que sobrecargan la base: las read replicas
    reparten esa carga (las queries de solo lectura van a réplicas), aliviando la primaria
    que atiende las escrituras.
4.3 RDS genera la contraseña y la guarda en Secrets Manager (fromGeneratedSecret); la
    Task Definition de Fargate la inyecta a la app como variable/secret en runtime. La app
    nunca tiene la contraseña en el código ni en la imagen.
```

### Módulo 5
```
5.1 Alta disponibilidad (réplicas con failover automático), escalado sin downtime (más
    réplicas o shards), y backups/parches/monitoreo gestionados por AWS, sin que operes el
    servidor Redis.
5.2 Porque con varias tasks de Fargate cada request puede caer en una distinta; si el
    caché/sesiones vivieran en la memoria de una task, las demás no los tendrían (el
    usuario se "desloguearía" al azar). En ElastiCache, compartido, todas las tasks ven el
    mismo estado: eso hace posible escalar a N instancias.
5.3 No cambia el código de aplicación: ioredis/BullMQ/cache-manager funcionan igual.
    Cambia solo el endpoint (REDIS_URL apunta a ElastiCache en vez de al contenedor local).
```

### Módulo 6
```
6.1 Routing, throttling/rate limiting, CORS y autenticación (validar JWT con un
    authorizer): cosas que en NestJS hacías con guards y el ThrottlerModule, acá API
    Gateway las maneja a nivel de infraestructura, antes de llegar a tu código.
6.2 Con el Lambda Web Adapter (o un adaptador serverless-express): tu app NestJS completa
    corre dentro de una Lambda detrás de API Gateway, sin reescribir el código; mitigás
    cold starts con provisioned concurrency o SnapStart.
6.3 API Gateway + Lambda para APIs internas, de tráfico bajo/intermitente, MVPs o partes
    event-driven (pagás por uso, escala a cero). ALB + Fargate para la API principal de
    tráfico sostenido, donde cold starts y costo por request a escala no convienen.
```

### Módulo 7
```
7.1 Un topic SNS con varias colas SQS suscriptas: el mensaje se entrega (fan-out) a todas
    las colas a la vez. Combina el fan-out de SNS con la durabilidad, amortiguación y
    reintentos/DLQ de SQS.
7.2 Para que cada consumidor procese a su propio ritmo, con su buffer, sus reintentos y su
    DLQ independientes. Si un consumidor se atrasa o falla, no afecta a los demás; cada
    cola aísla a su servicio.
7.3 SNS (o SNS→SQS) para fan-out durable de alto throughput con consumidores desacoplados.
    EventBridge cuando necesitás enrutamiento por contenido, integración con servicios
    AWS/SaaS, schema registry y replay (EDA de negocio).
```

### Módulo 8
```
8.1 La falta de visibilidad: con eventos sueltos entre muchos servicios, nadie ve el flujo
    completo ni en qué paso está cada ejecución, y debuggear se vuelve muy difícil. Step
    Functions hace el flujo explícito (diagrama, estado de cada ejecución, qué falló).
8.2 Como pasos de compensación en la máquina de estados: ante un error en un paso, la
    máquina transiciona a estados que deshacen los pasos previos (ej. liberar el stock
    reservado si el cobro falla), en orden inverso.
8.3 Cuando el flujo es simple (reaccionar a un evento sin orden complejo ni
    compensaciones): ahí la coreografía con EventBridge alcanza y es más desacoplada.
    Orquestar algo simple agrega un servicio y acoplamiento central sin necesidad.
```

### Módulo 9
```
9.1 (1) NAT Gateways de más → usar los justos y VPC endpoints para servicios AWS. (2)
    Egress/tráfico entre AZ → minimizar datos que salen y la comunicación cross-AZ. (3)
    Recursos sobredimensionados u olvidados → right-sizing, Savings Plans y cdk destroy /
    lifecycle policies. (Cualquier tres.)
9.2 Porque la transferencia de datos entre AZ (y fuera de AWS) se cobra por GB: servicios
    que se mandan mucho tráfico entre sí cruzando AZ acumulan un costo que no se ve en el
    diseño pero aparece en la factura.
9.3 AWS Budgets, que alerta cuando el gasto proyectado supera un umbral. Para saber qué
    proyecto/equipo gasta qué, se etiquetan los recursos con tags de costo y se analiza en
    Cost Explorer por tag.
```

### Módulo 10
```
10.1 (1) Excelencia operacional — IaC/CI-CD/observabilidad. (2) Seguridad — IAM mínimo
     privilegio, cifrado, secretos. (3) Confiabilidad — Multi-AZ, reintentos, DLQ. (4)
     Eficiencia de performance — elegir el servicio correcto, caché, autoscaling. (5)
     Optimización de costos — right-sizing, serverless, Savings Plans. (6) Sostenibilidad
     — eficiencia de recursos, escalar a cero.
10.2 Por ejemplo: RDS Multi-AZ (failover), healthchecks + auto-recovery del ECS service +
     graceful shutdown, y colas con reintentos/backoff + DLQ para el procesamiento async.
10.3 Porque te da una checklist de criterio para interrogar cualquier diseño desde varios
     ángulos (qué pasa si se cae una AZ, quién accede, cuánto cuesta, cómo lo operás), que
     es justo pensar como arquitecto. No agrega temas nuevos: ordena el criterio que ya
     tenés.
```

### Módulo 11
```
11.1 allowDefaultPortFrom abre el security group de la base SOLO para el security group de
     la app (mínimo privilegio de red); grantReadWrite le da al rol IAM de la task permiso
     de leer/escribir SOLO ese bucket (mínimo privilegio de IAM). Ambos aplican defensa en
     profundidad: red + permisos acotados.
11.2 ContainerImage.fromAsset(".") construye la imagen desde tu Dockerfile (el multi-stage
     del módulo de Docker), la sube a ECR y la usa en la Task Definition de Fargate. La
     misma imagen que probaste en local corre en producción.
11.3 No. Para un MVP, este stack completo es complejidad accidental (operación, costo,
     consistencia distribuida). Arrancaría con Fargate + RDS (+ ElastiCache si hace falta
     caché) y sumaría EventBridge/SQS/Lambda/Step Functions cuando un dolor concreto lo
     justifique. Saber cuándo NO desplegar todo es criterio de Tech Lead.
```

---

## Siguientes pasos

Con los dos módulos de AWS (criterio + práctica) tenés lo que pide un rol senior/Tech Lead: elegir bien los servicios **y** construir la infra de verdad, con código, costos y el lente del Well-Architected. El recorrido desde acá: **(1)** desplegá tu "Task API" real con este stack CDK y dejala corriendo con CI/CD; **(2)** profundizá la **arquitectura event-driven aplicada** (event sourcing, CQRS, saga con Step Functions, outbox con CDC) como módulo propio; **(3)** sumá **Kubernetes** para tener el criterio K8s-vs-Fargate completo, y **observabilidad práctica** con OpenTelemetry; **(4)** practicá **estimar costo y diseñar en voz alta** con trade-offs — diseño + plata + operación es el combo que se evalúa en una entrevista de Tech Lead. Y siempre, el hilo del senior: la mejor arquitectura es la más simple que resuelve el problema real.
