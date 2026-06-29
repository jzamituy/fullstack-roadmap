# Workflows durables y orquestación de jobs: procesos que sobreviven al crash

**Durable execution (Temporal/Step Functions), workflow vs activity, saga durable y TCC, scheduling distribuido, tipos de jobs y los modos de falla a nivel diseño · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo sube un escalón sobre lo que ya sabés de colas: en [Redis](redis.md) M7-10 aprendiste a sacar trabajo del request con una cola + worker + reintentos + DLQ (la mecánica de **un paso asíncrono**); en [Event-driven](event-driven.md) M9 viste la **saga** (coreografía vs orquestación) a nivel concepto. Acá respondemos la pregunta que queda: **¿cómo coordinás un proceso de varios pasos, que tarda minutos/días, y que tiene que sobrevivir a que el server se reinicie a la mitad?** Eso es **durable execution**, y es lo que distingue "tengo una cola" de "tengo un motor de workflows".

**Lo que asumimos.** Que conocés colas de trabajos y reintentos/idempotencia ([Redis](redis.md) M7-10), la saga y la compensación ([Event-driven](event-driven.md) M9), idempotency keys ([api-design](api-design.md) M4, [Caso 8](system-design-casos.md)) y elección de líder/locks distribuidos ([Consenso](consenso.md)). No re-explicamos eso: lo **reusamos**.

> ⚠️ **Nota sobre datos volátiles.** Las APIs y los límites de Temporal, AWS Step Functions, Azure Durable Functions y Airflow cambian de versión a versión. Todo lo marcado con ⚠️ verificalo en la doc oficial (`docs.temporal.io`, `docs.aws.amazon.com/step-functions`, etc.). Los **patrones** (durable execution, replay determinista, fan-out/fan-in, TCC) no cambian; los límites concretos sí.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Índice de módulos**
1. El problema: durable execution (sobrevivir al crash a mitad de camino)
2. Workflow vs activity y la restricción de determinismo
3. El panorama de motores: Temporal, Step Functions, Airflow
4. Saga durable y compensación: TCC y el orquestador que no se olvida
5. Tipos de jobs y scheduling distribuido (el cron que no debe dispararse dos veces)
6. Modos de falla a nivel diseño: poison messages, DLQ y visibilidad
7. El criterio: ¿cola simple, motor de workflows o scheduler gestionado?

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El problema: durable execution

**Teoría.** Tenés un proceso de varios pasos: confirmar un pedido = **reservar stock → cobrar la tarjeta → agendar el envío → mandar el mail**. Lo escribís como una función con una cola normal ([Redis](redis.md)):

```
async function confirmarPedido(pedido) {
  await reservarStock(pedido);   // paso 1
  await cobrar(pedido);          // paso 2  ← el server se reinicia ACÁ
  await agendarEnvio(pedido);    // paso 3
  await mandarMail(pedido);      // paso 4
}
```

¿Qué pasa si el proceso **se cae justo después del paso 2** (un deploy, un OOM, un nodo que muere)? Reservaste stock y cobraste, pero no agendaste ni avisaste. Al reintentar el job desde cero, **volvés a reservar y volvés a cobrar** → doble cargo. La cola te da reintentos, pero el reintento **re-ejecuta todo desde el principio**: no sabe en qué paso estabas. La idempotencia ([Caso 8](system-design-casos.md)) evita el doble efecto, pero tenés que cablearla a mano en cada paso y seguís reejecutando trabajo ya hecho.

**Durable execution** es la solución de raíz: un motor que **persiste el resultado de cada paso a medida que se completan**, de modo que si el proceso se cae y reanuda, **salta los pasos ya hechos** y sigue desde donde quedó. El estado del workflow (qué pasos corrieron, con qué resultado) vive en un **store durable**, no en la memoria del proceso. Es como tener un proceso con *checkpoints* automáticos en cada línea: el crash deja de ser una catástrofe y pasa a ser una pausa.

La idea en una imagen: el workflow es una **receta**, y el motor lleva un **diario** de qué pasos ya hiciste con qué resultado. Si te interrumpen en la mitad, al volver **leés el diario** (no rehacés lo hecho) y seguís en el paso siguiente. Eso es lo que dan **Temporal**, **AWS Step Functions**, **Azure Durable Functions**, **Cadence** (módulo 3) — y lo que una cola pelada **no** da.

> Puente con [Redis](redis.md): una cola + worker es la **base** (sacar trabajo del request, reintentar). Un motor de workflows es la **capa de arriba**: coordina *muchos* de esos pasos en un proceso durable y resumible. No reemplaza a la cola — por debajo, los motores usan colas/persistencia; te dan la **durabilidad del proceso completo**.

La frase mental: **una cola reintenta un job desde cero (re-ejecuta todo, no sabe en qué paso ibas). Durable execution persiste el resultado de cada paso a medida que se completa, así un crash a la mitad reanuda desde donde quedó —sin repetir lo hecho—. El estado del proceso vive en un store durable, no en la RAM. Es lo que da un motor de workflows (Temporal/Step Functions), no una cola pelada.**

**Ejercicios 1**
1.1 🔁 ¿Qué problema tiene reintentar desde una cola normal un proceso de 4 pasos que se cae después del paso 2? ¿Qué hace distinto la durable execution?
1.2 🧠 La idempotencia ([Caso 8](system-design-casos.md)) ya evita el doble cobro al reintentar. Entonces, ¿qué te sigue dando un motor de workflows que la idempotencia sola no? (pista: trabajo repetido y estado del proceso)
1.3 🧠 ¿Por qué se dice que un motor de workflows "no reemplaza a la cola" sino que se apoya en ella?

---

## Módulo 2 — Workflow vs activity y la restricción de determinismo

**Teoría.** ¿Cómo hace el motor para "reanudar desde donde quedó"? Con **replay desde un event history**. Cada vez que el workflow avanza, el motor anota en un log durable los eventos ("paso 1 completado con resultado X"). Cuando el proceso se reinicia, el motor **vuelve a ejecutar tu código de workflow desde el principio**, pero **no re-ejecuta los pasos ya hechos**: cuando tu código pide el paso 1, el motor **devuelve el resultado grabado** en vez de correrlo de nuevo. Así reconstruye el estado en memoria hasta el punto donde se cayó, y de ahí sigue en vivo. Esto parte el mundo en dos tipos de código:

- **Activity (actividad):** el trabajo que tiene **efectos secundarios** — llamar a una API, escribir en la base, cobrar, mandar un mail. Es lo que **puede fallar** y lo que el motor **reintenta** (con su propia política de backoff). Su resultado se **persiste** en el history.
- **Workflow:** el código que **orquesta** las activities (el orden, los if, los loops, las compensaciones). Es lo que se **replaya**.

Un matiz que distingue al senior: el motor garantiza que una activity se ejecuta **al menos una vez (at-least-once)**, **no exactamente una**. Entre que la activity termina su efecto (cobró) y que su resultado se persiste, un crash puede hacer que el motor la **reintente** → corre dos veces. Por eso **la durabilidad del workflow NO te exime de idempotencia**: el efecto de cada activity debe ser idempotente ([Caso 8](system-design-casos.md)) para que reejecutarla no duplique nada. Durable execution resuelve "no repetir el proceso entero ni perder el progreso"; la idempotencia resuelve "que reintentar un paso sea seguro". Las dos, juntas.

De ahí sale **la restricción que más confunde** y que cae en entrevistas: **el código del workflow debe ser determinista.** Como se replaya, si en el replay tomara un camino distinto al de la ejecución original, el motor no podría casar tu código con el history grabado y todo se rompe. Concretamente, **dentro del workflow no podés**: leer la hora directamente (`Date.now()`), generar aleatorios (`Math.random()`), generar UUIDs al voleo, llamar a una API o leer de la base **directo**. Todo eso —lo no determinista y lo que tiene efectos— va **adentro de una activity**, cuyo resultado queda grabado y se devuelve igual en el replay. El motor te da versiones deterministas de lo que necesites (un "ahora" del workflow, timers durables, etc.).

> Es el mismo principio que el **event sourcing** ([Event-driven](event-driven.md) M10) o el replay de un log: el estado es una **función del log de eventos**. Acá el "estado" es *en qué punto del proceso estás*, y reconstruirlo es replayar el código contra el history. Por eso el código debe ser una función pura de sus inputs + los resultados grabados.

> ⚠️ El detalle de qué exactamente es no determinista depende del motor. Temporal lo impone fuerte en el SDK (vos escribís el código, vos podés meter un `Date.now()` que rompa el replay). Step Functions **igual reanuda desde su propio historial de ejecución** —el replay no es exclusivo de Temporal—, pero como el flujo es una **state machine declarativa** en JSON/ASL (no código imperativo tuyo — módulo 3), **no te deja meter no-determinismo en el control de flujo**: la trampa no se te expone. El concepto es transversal: **lo que se replaya, determinista; lo que tiene efectos, en una activity reintentable y grabada.**

La frase mental: **el motor reanuda replayando tu código de workflow contra un log de eventos: los pasos ya hechos no se re-ejecutan, devuelven su resultado grabado. Por eso el WORKFLOW debe ser determinista (nada de `Date.now()`/random/IO directo) y todo lo que tiene efectos o puede fallar va en una ACTIVITY (reintentable y persistida). Mismo principio que event sourcing: el estado es función del log.**

**Ejercicios 2**
2.1 🔁 ¿Qué es una activity y qué es un workflow? ¿Cuál se reintenta y cuál se replaya?
2.2 🧠 ¿Por qué el código de un workflow no puede llamar a `Date.now()` ni a `Math.random()` directamente? ¿Dónde va eso?
2.3 🧠 Relacioná el replay de un motor de workflows con el event sourcing: ¿en qué sentido "el estado es función del log" en ambos?
2.4 ✍️ Implementá en TypeScript el **corazón** de la durable execution: un `DurableExecutor` con `step<T>(stepId, fn): Promise<T>` que, si el paso **ya corrió** (está en un `StepStore` persistido), devuelva el resultado grabado **sin re-ejecutar** `fn`; y si no, corra `fn`, persista el resultado y lo devuelva. Con eso, un workflow que se cae y reanuda salta los pasos ya hechos. (La solución está al final; tiene que compilar en `--strict`.)

---

## Módulo 3 — El panorama de motores: Temporal, Step Functions, Airflow

**Teoría.** No todos los "orquestadores" son lo mismo; se confunden y se eligen mal. Tres familias:

- **Temporal** (y su ancestro **Cadence**, y Azure Durable Functions en la misma idea): **durable execution como código**. Escribís el workflow en tu lenguaje (TS, Go, Java…) como una función casi normal, y el SDK le da durabilidad por replay (módulo 2). Brilla cuando el flujo es **lógica compleja** (muchos if, loops, esperas largas de días, señales externas) y querés expresarlo en código testeable. Costo: corrés/operás un cluster de Temporal (o pagás Temporal Cloud ⚠️), y tenés que respetar la restricción de determinismo.
- **AWS Step Functions**: durable execution **declarativa y gestionada**. El workflow es una **state machine** definida en JSON (ASL — Amazon States Language), no código tuyo; AWS la ejecuta, persiste el estado y reintenta. Brilla cuando estás **en AWS**, querés **cero operación** y el flujo se expresa bien como estados/transiciones (incluye `Map` para fan-out, `Parallel`, `Wait`, catch/retry declarativos). Costo: el modo **Standard** (el durable, de larga vida, el que interesa acá) **se paga por transición de estado** ⚠️ —y cada reintento cuenta como transición extra—, lo que se vuelve caro para flujos con muchísimos pasos chicos; la lógica compleja en JSON también incomoda. (Existe **Express**, que se factura por requests+duración y es mucho más barato, pero está pensado para flujos cortos de alto volumen y **no da el mismo historial durable de larga vida** — para durable execution el relevante es Standard ⚠️.)
- **Airflow / Dagster / Prefect**: orquestadores de **data pipelines** (ETL, batch, ML). El workflow es un **DAG** (grafo de tareas) que corre **en un schedule** (todas las noches procesá estos datos). Brilla para **pipelines de datos programados**, no para procesos de negocio por-request de larga vida. Confundir Airflow con Temporal es un error clásico: Airflow piensa en "corré este DAG de batch cada noche"; Temporal/Step Functions piensan en "este pedido concreto está en el paso 3 desde hace 2 días".

El criterio de elección, en una línea: **¿lógica de negocio compleja y por-instancia de larga vida? → Temporal (código) o Step Functions (si querés gestionado en AWS y el flujo es declarativo). ¿Pipeline de datos programado (batch/ETL/ML)? → Airflow/Dagster.** Y conecta con [Event-driven](event-driven.md) M9: el **orquestador** de la saga de ahí, llevado a producción, **es** uno de estos motores; la **coreografía** (sin orquestador, puro eventos) es la alternativa cuando el flujo es simple.

La frase mental: **Temporal = durable execution como CÓDIGO (flujo complejo, lógica rica, lo operás vos/Cloud). Step Functions = durable execution DECLARATIVA y gestionada en AWS (state machine JSON, cero ops, pagás por transición). Airflow/Dagster = DAGs de DATOS programados (ETL/batch/ML), no procesos de negocio por-request. Elegís por la forma del flujo, no por moda.**

**Ejercicios 3**
3.1 🔁 ¿Cuál es la diferencia central entre Temporal y Step Functions en *cómo* definís el workflow?
3.2 🧠 ¿Por qué Airflow no es la herramienta para orquestar "el proceso de confirmación de un pedido concreto que puede tardar días"? ¿Para qué sí es?
3.3 🧠 Conectá con la saga de [Event-driven](event-driven.md) M9: ¿qué papel cumple un motor como Step Functions o Temporal en una saga **orquestada**?

---

## Módulo 4 — Saga durable y compensación: TCC y el orquestador que no se olvida

**Teoría.** [Event-driven](event-driven.md) M9 te dio la saga: pasos locales, cada uno con su **compensación** (deshacer), porque entre servicios **no hay rollback automático**. El problema que dejó abierto: el **orquestador** de la saga **también puede caerse a la mitad** — ¿quién garantiza que las compensaciones realmente corran si el coordinador muere justo después de cobrar? Acá cierra: un **motor de durable execution es el orquestador de saga ideal**, porque la durabilidad (módulos 1-2) garantiza que el flujo —incluidas las compensaciones— **se completa aunque el proceso se reinicie**. La saga orquestada deja de ser frágil:

```
workflow ConfirmarPedido:
  try:
    reserva = activity reservarStock()      // cada paso es una activity (reintentable, grabada)
    cobro   = activity cobrar()
    envio   = activity agendarEnvio()
  on error:
    // el motor garantiza que esto corre aunque el server se haya caído:
    if cobro:   activity reembolsar(cobro)        // compensaciones en orden inverso
    if reserva: activity liberarStock(reserva)
```

La compensación es **código con try/catch**, y el motor se encarga de que se ejecute. Nada de "armar a mano la máquina de estados y rezar que el coordinador no muera".

Un patrón formal que vale nombrar es **TCC (Try-Confirm-Cancel)**, la versión "reserva en dos fases" de la saga:

- **Try:** reservás el recurso **tentativamente** (apartás el stock, hacés un *hold* en la tarjeta — autorizás sin capturar). No es definitivo, pero queda apartado.
- **Confirm:** si todos los Try salieron bien, confirmás (capturás el cobro, bajás el stock de verdad).
- **Cancel:** si algo falló, cancelás los Try (liberás el *hold*, devolvés el stock).

TCC evita el problema de "compensar algo que ya tuvo efecto visible": en vez de cobrar y después reembolsar (el cliente ve dos movimientos), hacés un *hold* y solo capturás al final. Es más limpio cuando el recurso **soporta reserva tentativa** (tarjetas, inventario); cuando no, caés en la saga clásica con compensación posterior. En ambos, los pasos deben ser **idempotentes** ([Caso 8](system-design-casos.md)): el motor puede reintentar un Try o un Cancel, y reejecutarlo no debe duplicar el efecto.

> El recordatorio del senior, intacto desde [Event-driven](event-driven.md) M9: **en un monolito con una sola base, no hay saga ni TCC — hay una transacción ACID.** Todo esto es para cuando el flujo **cruza límites de servicio/DB** y la transacción única no existe. Meter Temporal para coordinar dos tablas de la misma Postgres es sobre-ingeniería.

La frase mental: **el motor durable es el orquestador de saga ideal: garantiza que el flujo Y sus compensaciones corran aunque el coordinador se caiga (lo que la saga "a mano" no aseguraba). TCC = saga en dos fases (Try reserva tentativo → Confirm captura / Cancel libera), más limpio que cobrar-y-reembolsar cuando el recurso soporta hold. Pasos idempotentes siempre. Y en una sola DB: transacción, no saga.**

**Ejercicios 4**
4.1 🔁 ¿Qué problema de la saga orquestada "a mano" resuelve apoyarla en un motor de durable execution?
4.2 🧠 Explicá TCC (Try-Confirm-Cancel) y por qué un *hold* + captura es más limpio que cobrar y después reembolsar.
4.3 🧠 ¿Por qué los pasos de una saga durable (Try, Confirm, Cancel, compensaciones) tienen que ser idempotentes? (conectá con los reintentos del motor)

---

## Módulo 5 — Tipos de jobs y scheduling distribuido

**Teoría.** No todos los trabajos en background son iguales. Los cuatro tipos que conviene tener nombrados:

- **Fire-and-forget:** encolás y te olvidás (mandar un mail, invalidar una caché). Una cola simple ([Redis](redis.md)) alcanza.
- **Delayed / scheduled:** correr **una vez en un momento futuro** ("recordatorio a las 24h", "cobrar al terminar el trial"). La cola lo agenda (`delay`), o un timer durable del motor.
- **Recurring (cron):** correr **periódicamente** ("todas las noches a las 2am reconciliá pagos"). Acá aparece el problema distribuido de abajo.
- **Fan-out / fan-in:** **abrir N sub-tareas en paralelo, esperar a todas y agregar** el resultado (procesar las 10.000 imágenes de un upload y al final marcar "listo"; el *scatter-gather* de [Búsqueda](busqueda.md), el transcoding del [Caso 13](system-design-casos.md), o la cola de trabajos a escala del [Caso 7](system-design-casos.md) — el crawler que reparte URLs entre workers). El motor te da esto nativo (el `Map` de Step Functions, `Promise.all` de child-workflows en Temporal): la parte difícil es el **fan-in** — saber cuándo terminaron *todas*, manejar las que fallan, y agregar sin perder ninguna.

El problema estrella del scheduling es el **cron distribuido**: si corrés **N instancias** de tu app y cada una tiene el cron "2am: reconciliar", **las N lo disparan** → el job corre N veces (N cobros, N reportes). La solución es garantizar que **exactamente una** instancia lo ejecute:

- **Lock por-tick o elección de líder** ([Consenso](consenso.md)) — dos variantes que conviene no confundir: (a) **lock por-tick**: cada instancia intenta tomar un **lock con TTL** justo antes de ejecutar, con la **fecha del tick como clave** (un `SET NX` en [Redis](redis.md)), y solo la que lo gana corre **esa** ventana; (b) **elección de líder**: un líder **estable** (con un *lease* renovado) que vive entre ticks y es el único que corre los crons. Ambas resuelven el cron distribuido. Ojo con el lock que **expira a mitad** del job (el clásico problema del lock distribuido — fencing tokens de [Consenso](consenso.md)).
- **Un scheduler dedicado:** un único componente que dispara (el *scheduler* de Temporal, EventBridge Scheduler en AWS, un Kubernetes `CronJob`) y **encola** el trabajo; los workers lo toman. El disparo está centralizado en una pieza, los N workers solo consumen.

La regla: **la ejecución del trabajo puede (y debe) escalar a N workers; el disparo del tick recurrente tiene que ser uno solo.** Separá "quién dispara" de "quién ejecuta".

La frase mental: **cuatro tipos de job: fire-and-forget, delayed (una vez en el futuro), recurring (cron) y fan-out/fan-in (N en paralelo + esperar a todas + agregar; lo difícil es el fan-in). El problema del cron distribuido: N instancias disparan N veces → garantizá UN disparo con líder/lock (fencing) o un scheduler dedicado. Escalá la ejecución a N workers, pero el tick lo dispara uno solo.**

**Ejercicios 5**
5.1 🔁 Nombrá los cuatro tipos de job y dá un ejemplo de cada uno. ¿Cuál es la parte difícil del fan-out/fan-in?
5.2 🧠 Corrés 5 instancias de tu API, cada una con un cron "2am: enviar resumen diario". ¿Qué pasa y cómo garantizás que el resumen salga una sola vez?
5.3 🧠 ¿Por qué conviene separar "quién dispara el tick recurrente" de "quién ejecuta el trabajo"? ¿Qué escala a N y qué no?

---

## Módulo 6 — Modos de falla a nivel diseño: poison messages, DLQ y visibilidad

**Teoría.** [Redis](redis.md) M9 te dio la mecánica de reintentos/backoff/DLQ en BullMQ. Acá subimos a la altura de diseño: los modos de falla que tenés que **nombrar y mitigar** en una entrevista de sistemas, con el marco de las 4 capas.

Un **poison message** (mensaje veneno) es un job que **falla siempre** —datos malformados, un bug, un caso no contemplado— y que, sin defensa, se reintenta para siempre consumiendo recursos o bloqueando la cola.

> 🔥 **Falla en 4 capas — el poison message que atasca la cola.**
> (1) **Qué se rompe:** un job que siempre falla se reintenta sin fin; si la cola procesa **en orden** (o el job reencola al frente), **bloquea a los de atrás** (head-of-line blocking) y la cola entera se detiene.
> (2) **Por qué a esta escala:** con alto volumen, un solo veneno + reintentos infinitos drena workers y tapa el pipeline.
> (3) **Corto plazo:** límite de reintentos (`attempts`) → tras agotarlos, el job va a la **DLQ** (o al estado `failed` de BullMQ) y **sale del camino**; alertar sobre la DLQ.
> (4) **Largo plazo:** **distinguir fallos transitorios de permanentes** (reintentá timeouts/5xx; los errores de negocio fallan rápido y van directo a DLQ, no se reintentan — [Redis](redis.md) M9); y **idempotencia** ([Caso 8](system-design-casos.md)) para que los reintentos no dupliquen efecto.

> 🔥 **Falla en 4 capas — la tormenta de reintentos (retry storm).**
> (1) **Qué se rompe:** un servicio downstream se degrada, **todos** los jobs empiezan a fallar y reintentar a la vez → un pico de carga que **impide que el downstream se recupere** (lo termina de tumbar).
> (2) **Por qué a esta escala:** miles de reintentos sincronizados son un auto-DDoS.
> (3) **Corto plazo:** **backoff exponencial + jitter** ([Redis](redis.md) M9) para desincronizar; **circuit breaker** ([Go resiliencia](go-resiliencia.md)) que corta los intentos mientras el downstream esté caído.
> (4) **Largo plazo:** límites de concurrencia por cola/worker, *rate limiting* hacia el downstream, y *dead-lettering* para drenar lo irrecuperable.

Y una propiedad que un motor de workflows te da casi gratis y que en una entrevista vale oro: **visibilidad**. Como el estado de cada workflow está persistido (módulos 1-2), podés **ver exactamente en qué paso está cada instancia, qué falló y por qué** — la "arqueología" que [Event-driven](event-driven.md) M9 marcaba como el dolor de la coreografía. Sumá un **correlation ID** que viaje por todos los pasos/eventos para poder rastrear un proceso de punta a punta ([Observabilidad](observabilidad.md)).

Estructurá una respuesta de workflows con **las tres cosas**: **happy path** (workflow con sus activities, durable), **modelo de falla** (poison message, retry storm, orquestador que muere, paso no idempotente), **recuperación** (DLQ + alertas, backoff+jitter+circuit breaker, durabilidad que reanuda, idempotencia).

La frase mental: **poison message = job que falla siempre → límite de reintentos + DLQ para sacarlo del camino (y distinguir transitorio de permanente). Retry storm = todos reintentan a la vez y tumban al downstream → backoff+jitter + circuit breaker. Y un motor durable te da VISIBILIDAD: ves en qué paso está cada instancia (más un correlation ID para rastrear de punta a punta).**

**Ejercicios 6**
6.1 🔁 ¿Qué es un poison message y por qué reintentarlo infinitamente es peligroso? ¿Qué lo saca del camino?
6.2 🧠 Un servicio downstream se cae y mil jobs empiezan a fallar y reintentar. ¿Qué dos mecanismos evitan que los reintentos impidan que el downstream se recupere?
6.3 🧠 ¿Por qué la "visibilidad" (ver en qué paso está cada proceso) es más fácil con un motor de workflows que con una saga por coreografía? (conectá con [Event-driven](event-driven.md) M9)

---

## Módulo 7 — El criterio: ¿cola simple, motor de workflows o scheduler gestionado?

**Teoría.** El cierre, igual que en cada módulo del hub: **¿de verdad necesitás un motor de workflows?** La mayoría de las veces, **no** — y meter Temporal para mandar un mail async es la sobre-ingeniería de manual.

**Te alcanza con una cola simple ([Redis](redis.md)/BullMQ) cuando:**
- El trabajo es **un solo paso asíncrono** (mandar mail, generar un thumbnail, invalidar caché): fire-and-forget con reintentos. No hay "proceso multi-paso" que coordinar.
- Incluso 2-3 pasos **cortos** que podés rehacer idempotentemente sin drama y que no tardan días.

**Te alcanza con un scheduler gestionado (EventBridge Scheduler, K8s CronJob, cron de la plataforma) cuando:**
- Necesitás **disparar trabajo en un horario** y poco más; el scheduler encola y los workers consumen (módulo 5).

**Necesitás un motor de durable execution (Temporal/Step Functions) cuando:**
- El proceso es **multi-paso, de larga vida** (minutos a días), **debe sobrevivir a reinicios**, y la **corrección importa** (pagos, onboarding, provisioning, sagas con compensación).
- Querés **visibilidad** de en qué paso está cada instancia, **esperas largas** (timers durables de días), **señales externas** (esperar la aprobación de un humano), o **fan-out/fan-in** con manejo de fallos serio.
- El costo de cablear a mano la durabilidad + idempotencia + compensación + visibilidad **supera** el costo de operar el motor.

El costo que se subestima de los motores: es **otra pieza de infra que operar** (un cluster de Temporal, o el lock-in y el costo-por-transición de Step Functions ⚠️), la **restricción de determinismo** que hay que respetar, y una curva de aprendizaje real. No es gratis; lo justificás cuando el proceso lo pide.

> **El cierre de criterio — no sobre-diseñar.** ¿Mandar un mail de bienvenida al registrarse? Cola + worker, listo. ¿Un cron nocturno? Scheduler gestionado. ¿El onboarding de un cliente que toca 6 servicios, espera la firma de un contrato 3 días, y debe poder reanudar y compensar si algo falla en el medio? **Ahí** un motor de workflows gana lo que cuesta. La pregunta senior: *"¿esto es un job, o es un proceso?"*. Un job lo hace una cola; un proceso durable de larga vida pide un motor.

La frase mental de cierre: **un paso async → cola (BullMQ). Un disparo por horario → scheduler gestionado. Un PROCESO multi-paso de larga vida que debe sobrevivir crashes, compensar y ser visible → motor de durable execution (Temporal/Step Functions). El motor cuesta (infra, determinismo, aprendizaje): justificalo cuando cablear durabilidad+compensación+visibilidad a mano cueste más. Pregunta clave: ¿es un job o un proceso?**

**Ejercicios 7**
7.1 🔁 Dá un caso donde alcanza una cola simple, uno donde alcanza un scheduler, y uno donde necesitás un motor de workflows.
7.2 🧠 ¿Cuál es el costo "oculto" de adoptar Temporal/Step Functions que hay que poner en la balanza?
7.3 🧠 ¿Qué pregunta resume el criterio entre "una cola me alcanza" y "necesito un motor de workflows"?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** El problema: la cola **reintenta el job desde el principio** — no sabe que los pasos 1 y 2 ya se hicieron, así que **los re-ejecuta** (re-reserva stock, **vuelve a cobrar** → doble cargo). La **durable execution** persiste el resultado de cada paso a medida que se completa; al reanudar tras el crash, **salta los pasos ya hechos** (devuelve su resultado grabado) y sigue desde donde quedó (paso 3), sin repetir efectos ni trabajo.

**1.2** La idempotencia evita el **doble efecto** (no cobrás dos veces), pero por sí sola **seguís re-ejecutando** los pasos ya hechos en cada reintento (trabajo desperdiciado, latencia, llamadas de más al downstream) y **vos** tenés que cablear la clave de idempotencia en cada paso y mantener el estado de "por dónde iba". Un motor de workflows te da, además: **no repetir el trabajo** (salta lo hecho), el **estado del proceso persistido** (sabe en qué paso está sin que lo manejes vos), reintentos por activity, y visibilidad. La idempotencia es **necesaria igual** (el motor reintenta), pero resuelve menos.

**1.3** Porque por debajo un motor de workflows **usa colas y persistencia** para mover y durar el trabajo; no inventa un transporte nuevo. Lo que agrega es la **capa de coordinación durable** encima: orquestar muchos pasos como un proceso resumible. La cola sigue siendo la base (sacar trabajo del request, reintentar un paso); el motor coordina el **proceso completo** sobre ella.

**2.1** Una **activity** es el código con **efectos secundarios** (llamar APIs, escribir en la base, cobrar) — lo que **puede fallar** y el motor **reintenta**, y cuyo resultado se **persiste**. Un **workflow** es el código que **orquesta** las activities (orden, ifs, loops, compensaciones). Se **reintenta** la activity; se **replaya** el workflow.

**2.2** Porque el workflow **se replaya** desde el event history al reanudar: el motor re-ejecuta tu código y, para reconstruir el estado, **casa cada operación con lo grabado**. Si el código llamara a `Date.now()` o `Math.random()`, en el replay daría **un valor distinto** al de la ejecución original → el código tomaría otro camino y no casaría con el history (ejecución no determinista) → se rompe. Todo lo no determinista o con efectos va **adentro de una activity**, cuyo resultado se graba una vez y se **devuelve igual** en cada replay (el motor también ofrece timers/`now` deterministas propios).

**2.3** En el **event sourcing** ([Event-driven](event-driven.md) M10) el estado de una entidad es el **fold** de su log de eventos: reproducís los eventos y obtenés el estado actual. En un motor de workflows, el "estado" es **en qué punto del proceso estás (y con qué resultados parciales)**, y reconstruirlo es **replayar el código del workflow contra el history** de pasos completados. En ambos, **no guardás el estado mutable: guardás el log y derivás el estado** — por eso ambos exigen que la derivación sea determinista/reproducible.

**2.4**
```ts
// Cada paso completado se graba envuelto en { done, result } para que un
// resultado legítimamente `undefined` no se confunda con "el paso no corrió".
interface StepRecord {
  done: true;
  result: unknown;
}

interface StepStore {
  get(stepId: string): Promise<StepRecord | undefined>;
  set(stepId: string, record: StepRecord): Promise<void>;
}

class DurableExecutor {
  constructor(private readonly store: StepStore) {}

  async step<T>(stepId: string, fn: () => Promise<T>): Promise<T> {
    const recorded = await this.store.get(stepId);
    if (recorded !== undefined) {
      // Ya corrió: devolvemos el resultado grabado SIN re-ejecutar fn (el "salto" del replay).
      return recorded.result as T;
    }
    const result = await fn();
    await this.store.set(stepId, { done: true, result });
    return result;
  }
}

// Un workflow lo usa así; si el proceso muere tras "cobrar" y se reanuda,
// "reservar-stock" y "cobrar" devuelven lo grabado y solo corre "agendar-envio".
async function confirmarPedido(ex: DurableExecutor, pedidoId: string): Promise<void> {
  const reserva = await ex.step(`reservar-stock:${pedidoId}`, () => reservarStock(pedidoId));
  const cobro = await ex.step(`cobrar:${pedidoId}`, () => cobrar(pedidoId));
  await ex.step(`agendar-envio:${pedidoId}`, () => agendarEnvio(pedidoId, reserva, cobro));
}

declare function reservarStock(id: string): Promise<{ reservaId: string }>;
declare function cobrar(id: string): Promise<{ cobroId: string }>;
declare function agendarEnvio(
  id: string,
  reserva: { reservaId: string },
  cobro: { cobroId: string },
): Promise<void>;
```
Idea: `step` consulta el store por `stepId`; si hay un registro, **devuelve el resultado grabado sin correr `fn`** (ese es el "salto" de pasos ya hechos del replay del módulo 2); si no, ejecuta, persiste y devuelve. El `stepId` debe ser **estable y único** por paso (acá lo derivamos del id del pedido) para que el replay lo reconozca. El wrapper `{ done, result }` evita la ambigüedad de un resultado `undefined`. Compila en `--strict`. Esto es una **maqueta didáctica**: un motor real persiste el `StepStore` en una base durable, reintenta cada `fn` como activity, serializa los resultados, y —clave— hace dos cosas que esta maqueta no: (1) **detecta `stepId` duplicados/colisiones** (acá dos `step` con el mismo id devolverían en silencio el resultado del primero — un bug fácil), y (2) **valida que la secuencia de pasos del replay case con el history** grabado; si no casa, tira un *non-determinism error* (la restricción del módulo 2 en acción). El principio "graba cada paso, saltá los hechos al reanudar" sí es exactamente este.

**3.1** En **Temporal** definís el workflow como **código** en tu lenguaje (una función con lógica normal), y el SDK le da durabilidad por replay — la lógica compleja (ifs, loops, esperas) se expresa y testea como código. En **Step Functions** definís el workflow como una **state machine declarativa en JSON (ASL)**: estados y transiciones que AWS ejecuta; no escribís el flujo como código tuyo, lo describís. Code-first (flexible, lo operás) vs declarativo-gestionado (cero ops, pero la lógica compleja en JSON incomoda).

**3.2** Porque Airflow está pensado para **DAGs de datos que corren en un schedule** (procesá este batch todas las noches): piensa en *tareas de datos programadas*, no en *una instancia de proceso de negocio por-request* que vive días esperando señales externas. Orquestar "este pedido concreto está en el paso 3 desde hace 2 días, esperando la firma" es el terreno de **Temporal/Step Functions** (durable execution por-instancia). Airflow **sí** es la herramienta para **ETL/batch/ML programado**: "cada noche, extraé-transformá-cargá estos datos".

**3.3** En una saga **orquestada**, el orquestador es el componente central que dirige los pasos y dispara las compensaciones. Llevado a producción, **ese orquestador es un motor de durable execution**: Step Functions (state machine) o Temporal (workflow en código) ejecutan la secuencia, reintentan cada paso, y —clave— **garantizan que el flujo y las compensaciones se completen aunque el coordinador se reinicie**. El motor le da a la saga orquestada la **durabilidad y la visibilidad** que la versión "a mano" no aseguraba.

**4.1** Que el **orquestador mismo puede caerse a la mitad**: en una saga orquestada armada a mano, si el coordinador muere justo después de cobrar, **nadie garantiza que las compensaciones corran** (el "deshacer" queda colgado) y el proceso queda inconsistente. Un motor de durable execution **persiste el estado del flujo** y lo **reanuda** tras el crash, así que el camino de compensación **se ejecuta igual** aunque el proceso se haya reiniciado. La durabilidad convierte la saga frágil en una confiable.

**4.2** **TCC = Try-Confirm-Cancel:** **Try** reserva el recurso tentativamente (hold en la tarjeta — autorizar sin capturar; apartar stock); **Confirm** lo hace definitivo si todos los Try salieron bien (capturar el cobro); **Cancel** deshace los Try si algo falló (liberar el hold). Un **hold + captura** es más limpio que **cobrar y después reembolsar** porque evita el **efecto visible intermedio**: el cliente no ve un cargo y luego una devolución (dos movimientos, soporte, confusión), solo ve el cargo final si la operación se confirma, o nada si se cancela. Sirve cuando el recurso **soporta reserva tentativa**; si no, caés en compensación posterior.

**4.3** Porque el motor **reintenta** las activities (un Try, un Cancel, una compensación pueden ejecutarse más de una vez ante un timeout/crash). Si no fueran idempotentes, un reintento de "capturar cobro" cobraría dos veces, o un "liberar stock" repetido liberaría de más. La idempotencia ([Caso 8](system-design-casos.md): idempotency key + check-then-act) garantiza que **reejecutar el paso no cambia el resultado** — condición para que los reintentos del motor sean seguros.

**5.1** **Fire-and-forget** (mandar un mail), **delayed/scheduled** (recordatorio a las 24h), **recurring/cron** (reconciliar pagos cada noche), **fan-out/fan-in** (procesar 10.000 imágenes en paralelo y al final marcar "listo"). La parte difícil del **fan-out/fan-in** es el **fan-in**: detectar cuándo terminaron **todas** las sub-tareas, manejar las que fallan (¿reintentás solo esa?, ¿abortás todo?) y **agregar** el resultado sin perder ninguna.

**5.2** Las **5 instancias disparan el cron**, así que el resumen se envía **5 veces**. Para garantizar **una sola** ejecución: (a) **lock distribuido** — cada instancia intenta tomar un lock (ej. `SET NX` en [Redis](redis.md) con la fecha del tick como clave) antes de ejecutar; solo la que lo gana corre, y el TTL evita que quede tomado para siempre (cuidado con el lock que **expira a mitad** del job → fencing tokens de [Consenso](consenso.md)); (b) **elección de líder** — solo el líder corre los crons; (c) **un scheduler dedicado** (EventBridge Scheduler, K8s CronJob, scheduler de Temporal) que **dispara una vez** y encola; los workers consumen.

**5.3** Porque la **ejecución** del trabajo es lo que querés **escalar a N workers** (más throughput, paralelismo, resiliencia), pero el **disparo del tick** recurrente debe ocurrir **exactamente una vez** — si cada uno de los N dispara, el job corre N veces. Separando "quién dispara" (uno: líder/lock/scheduler) de "quién ejecuta" (N: los workers que consumen de la cola), conseguís **un disparo, ejecución escalable**. Escala a N: los workers/ejecución. No escala (es uno): el disparo del tick.

**6.1** Un **poison message** es un job que **falla siempre** (datos malformados, bug, caso no contemplado). Reintentarlo infinitamente es peligroso porque **consume workers/recursos sin fin** y, si la cola procesa en orden o reencola al frente, **bloquea a los jobs de atrás** (head-of-line blocking) y puede frenar el pipeline entero. Lo saca del camino un **límite de reintentos** (`attempts`): al agotarse, el job va a la **DLQ** / estado `failed` ([Redis](redis.md) M9) y deja de reintentarse, idealmente con una **alerta** para investigarlo.

**6.2** (1) **Backoff exponencial + jitter** ([Redis](redis.md) M9): reintentar esperando cada vez más y **desincronizado** (jitter), para no martillar al downstream todos a la vez. (2) **Circuit breaker** ([Go resiliencia](go-resiliencia.md)): tras detectar que el downstream está caído, **corta los intentos** por un tiempo (se "abre"), dándole aire para recuperarse en vez de seguir golpeándolo; reintenta de a poco (half-open) hasta cerrarse. Juntos evitan el **auto-DDoS** de la tormenta de reintentos.

**6.3** Porque en un motor de workflows el **estado de cada instancia está persistido** (event history): podés consultar **exactamente en qué paso está cada proceso, qué activity falló y por qué**. En una saga por **coreografía** ([Event-driven](event-driven.md) M9) no hay un lugar central con el estado del flujo — está **repartido** en "quién emitió qué evento y quién lo escuchó", así que reconstruir "por qué este pedido quedó a medias" es **arqueología** entre logs de varios servicios. La durabilidad centralizada del motor **es** la visibilidad (sumale un correlation ID para rastrear de punta a punta — [Observabilidad](observabilidad.md)).

**7.1** **Cola simple:** mandar un mail de bienvenida al registrarse (un paso async, fire-and-forget con reintentos). **Scheduler gestionado:** un reporte nocturno a las 2am (disparo por horario + workers que consumen). **Motor de workflows:** el onboarding de un cliente que toca 6 servicios, espera la firma de un contrato durante días y debe poder reanudar y **compensar** si un paso falla (proceso multi-paso, de larga vida, que debe sobrevivir crashes y ser visible).

**7.2** Que es **otra pieza de infraestructura que operar y entender**: un **cluster de Temporal** para mantener (o el **costo-por-transición y el lock-in** de Step Functions ⚠️), la **restricción de determinismo** del código de workflow que el equipo tiene que respetar (un bug de no-determinismo es sutil), y una **curva de aprendizaje** real. No es "gratis y mejor"; sumás complejidad operativa que solo se justifica cuando el proceso de verdad necesita durabilidad/compensación/visibilidad.

**7.3** **"¿Esto es un job, o es un proceso?"** Un **job** —un paso (o pocos) async, corto, que podés rehacer idempotentemente— lo resuelve una **cola**. Un **proceso** —multi-paso, de larga vida (minutos a días), que debe sobrevivir reinicios, compensar fallos y ser auditable/visible— pide un **motor de durable execution**. Si dudás y el flujo es simple, empezá por la cola; subí al motor cuando cablear durabilidad + compensación + visibilidad a mano cueste más que operarlo.

---

> **Para seguir.** Las fuentes de verdad: la doc de **Temporal** (`docs.temporal.io` — workflows, activities, determinismo, signals, child workflows), **AWS Step Functions** (`docs.aws.amazon.com/step-functions` — ASL, `Map`, `Parallel`, retry/catch), **Azure Durable Functions** y, para data pipelines, **Airflow/Dagster/Prefect**. Re-verificá lo marcado con ⚠️ (límites y precios de Step Functions, APIs de los SDKs, Temporal Cloud): se mueven. Los puentes en el hub: [Redis](redis.md) M7-10 es la cola/worker/DLQ sobre la que esto se apoya; [Event-driven](event-driven.md) M9-10 es la saga (coreografía vs orquestación) y el event sourcing que este módulo lleva a durable execution; [Consenso](consenso.md) es la elección de líder/locks/fencing del cron distribuido; [api-design](api-design.md) M4 y el [Caso 8](system-design-casos.md) son la idempotencia que todo paso reintentable necesita; [Go resiliencia](go-resiliencia.md) es el circuit breaker contra la retry storm; y [Observabilidad](observabilidad.md) es el correlation ID que hace rastreable un proceso de punta a punta.
