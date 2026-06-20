# Arquitectura event-driven aplicada: eventos, CQRS y sagas

**De la teoría del senior a patrones de producción · Outbox, event sourcing, sagas · nivel Tech Lead · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. El archivo senior te dio los conceptos (dual write, outbox, idempotencia, saga) y el de AWS los servicios (EventBridge, SQS, SNS, Step Functions). Este módulo los une en **patrones aplicados** que se implementan y se defienden en una entrevista de Tech Lead: cómo publicar eventos de forma confiable (Outbox), cómo separar lecturas de escrituras (CQRS), cómo guardar el historial en vez del estado (Event Sourcing), y cómo coordinar transacciones que cruzan servicios (Sagas). El hilo que recorre todo: la **consistencia fuerte es un lujo; cuando la perdés, hay que diseñar para la consistencia eventual conscientemente**. Y, sobre todo, el criterio de **cuándo NO** aplicar nada de esto.

**Lo que asumimos.** El archivo senior (consistencia/transacciones, resiliencia, DDD), los módulos de PostgreSQL (transacciones), Redis (colas, idempotencia) y AWS (EventBridge/SQS/Step Functions), y el módulo de patrones (CQRS+Mediator).

**Advertencia honesta.** Casi todo lo de esta página es **complejidad que se justifica solo en sistemas distribuidos o dominios complejos**. En un monolito con una sola base, una transacción ACID resuelve el 90% de esto. Lo aprendés para saber **cuándo** te hace falta y para no aplicarlo donde sobra — que es la marca de un Tech Lead, no coleccionar patrones.

**Índice de módulos**
1. EDA: eventos, comandos y queries
2. Consistencia eventual: el costo real de desacoplar
3. El dual write y el patrón Outbox
4. Idempotencia y el patrón Inbox
5. CQRS aplicado (y cuándo no)
6. CQRS en NestJS (`@nestjs/cqrs`)
7. Event Sourcing: guardar el historial, no el estado
8. Projections y read models
9. Sagas: coreografía vs. orquestación
10. Versionado de eventos (schema evolution)
11. Resiliencia y observabilidad de EDA
12. El criterio: cuándo NO

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — EDA: eventos, comandos y queries

**Teoría.** En una arquitectura **event-driven (EDA)**, los componentes se comunican por **eventos** en vez de llamarse directo (RPC síncrono). La distinción de vocabulario es fundamental y se confunde todo el tiempo:

- **Comando (Command)**: una **intención** de cambiar el estado. Imperativo, dirigido a alguien, puede ser rechazado. `ConfirmarPedido`. Espera que algo pase.
- **Evento (Event)**: un **hecho** que **ya ocurrió**, inmutable, en pasado. `PedidoConfirmado`. No se rechaza (ya pasó); se notifica. Quien lo emite no sabe (ni le importa) quién reacciona.
- **Query**: una **pregunta** que no cambia nada. `ObtenerPedido`. Devuelve datos.

La diferencia entre llamar directo y emitir un evento es el **acoplamiento temporal**. Si tu servicio de Pedidos **llama** al de Envíos (`envios.crear(...)`), depende de que Envíos esté vivo **ahora**: si está caído, Pedidos falla. Si Pedidos **emite** `PedidoConfirmado` y Envíos reacciona cuando puede, los dos están **desacoplados en el tiempo**: Envíos puede estar caído y procesar después, sin afectar la confirmación del pedido.

```
Acoplado (RPC):   Pedidos ──llama──▶ Envíos   (si Envíos está caído, Pedidos falla)
Desacoplado (EDA): Pedidos ──emite──▶ [evento] ◀──reacciona── Envíos
                                                  ◀──reacciona── Facturación
                                                  ◀──reacciona── Analytics
```

Ese desacople es el superpoder de EDA (agregar consumidores sin tocar al emisor, tolerar caídas) **y** su maldición (consistencia eventual, difícil de razonar — los próximos módulos).

**Ejercicios 1**
1.1 Clasificá cada uno como comando, evento o query: (a) `RegistrarUsuario`; (b) `UsuarioRegistrado`; (c) `ListarUsuarios`. ¿Cuál está en pasado y por qué?
1.2 ¿Qué es el "acoplamiento temporal" y cómo lo elimina emitir un evento en vez de llamar directo a otro servicio?
1.3 Nombrá una ventaja y una desventaja concretas de que el emisor de un evento no conozca a sus consumidores.

---

## Módulo 2 — Consistencia eventual: el costo real de desacoplar

**Teoría.** Cuando dejás de tener todo en una transacción ACID y empezás a propagar cambios por eventos, ganás desacople pero pagás un precio: la **consistencia eventual**. El sistema no está consistente *al instante*, sino *con el tiempo* (típicamente milisegundos a segundos, a veces más). Durante esa ventana, distintas partes ven estados distintos.

Ejemplo concreto: confirmás un pedido → se emite `PedidoConfirmado` → la vista de "mis pedidos" se actualiza vía un consumidor. Entre la confirmación y la actualización de la vista hay una **ventana** donde el usuario podría no ver su pedido todavía. No es un bug: es la naturaleza del sistema, y hay que **diseñar la UX y el negocio para tolerarla** (mostrar "procesando", optimistic UI, etc.).

El contraste con lo que ya sabés: en una sola base (módulo de PostgreSQL), una transacción te da **consistencia fuerte** gratis — confirmás y descontás stock **atómicamente**, nadie ve un estado intermedio. La regla de oro del archivo senior sigue valiendo: **mientras puedas mantener tus invariantes dentro de una DB, hacelo**; la consistencia fuerte es muchísimo más fácil que cualquier alternativa distribuida. Bajás a consistencia eventual solo cuando la distribución es **inevitable** (servicios separados, escalas distintas), y lo aceptás conscientemente.

Conceptos clave que vienen de la mano:

- **At-least-once vs at-most-once vs exactly-once**: en sistemas distribuidos, la entrega "exactly-once" **no existe** a nivel de transporte. Lo realista es **at-least-once** (puede duplicar) + **idempotencia** del lado del consumidor, lo que logra *exactly-once processing* (el efecto neto de procesar una sola vez). Esto es el módulo 4.

**Ejercicios 2**
2.1 ¿Qué es la consistencia eventual y qué se gana y qué se paga respecto de una transacción ACID en una sola base?
2.2 Según la regla del archivo senior, ¿cuándo conviene quedarse con consistencia fuerte y cuándo bajar a eventual?
2.3 ¿Por qué se dice que "exactly-once delivery" no existe y cómo se logra en la práctica el efecto de procesar una sola vez?

---

## Módulo 3 — El dual write y el patrón Outbox

**Teoría.** El problema central de EDA (visto en el senior, ahora lo implementamos). Querés hacer dos cosas al confirmar un pedido: **guardar en la base** (Postgres) **y publicar un evento** (a una cola/bus). Son **dos sistemas distintos**, y no hay transacción que los abarque. Esto es el **dual write**:

```ts
// MAL: dual write. Dos sistemas, sin atomicidad → ventana de fallo.
await db.insert(pedidos).values({ ... });        // (1) guarda en Postgres
await eventBus.publish("PedidoConfirmado", { ... }); // (2) publica el evento
// Si el proceso muere entre (1) y (2): el pedido existe pero NADIE se entera.
// Si (1) falla tras publicar (2): los consumidores procesan un pedido que no existe.
```

La solución es el **patrón Outbox**: en vez de publicar a un sistema externo, escribís el evento en una **tabla `outbox` de la misma base**, **dentro de la misma transacción** que el cambio de negocio. Como es una sola transacción, o se guardan **las dos cosas** o **ninguna** — atomicidad garantizada. Después, un proceso aparte lee la tabla y publica los eventos.

```ts
// BIEN: outbox. Cambio + evento en la MISMA transacción de Postgres.
await db.transaction(async (tx) => {
  await tx.insert(pedidos).values({ id, estado: "confirmado" });
  await tx.insert(outbox).values({
    tipo: "PedidoConfirmado",
    payload: { pedidoId: id },
    enviado: false,
  });
}); // atómico: o se guardan ambos o ninguno
```

Y un **publisher** (worker o cron) lee lo no enviado y lo publica:

```ts
// proceso aparte: lee el outbox y publica (a EventBridge, SQS, Kafka...)
const pendientes = await db.select().from(outbox).where(eq(outbox.enviado, false)).limit(100);
for (const ev of pendientes) {
  await bus.publish(ev.tipo, ev.payload);      // si falla, se reintenta en la próxima vuelta
  await db.update(outbox).set({ enviado: true }).where(eq(outbox.id, ev.id));
}
```

Dos formas de leer el outbox:

- **Polling** (lo de arriba): un cron consulta la tabla cada X. Simple, suficiente para la mayoría.
- **CDC (Change Data Capture)** con **Debezium**: lee el **WAL** (log de transacciones) de Postgres y publica los cambios del outbox automáticamente, sin polling. Más eficiente y de menor latencia, a cambio de más infraestructura.

Costo del patrón: el evento puede publicarse **más de una vez** (si el publisher muere tras publicar pero antes de marcar `enviado`). De ahí que los consumidores **deban ser idempotentes** (módulo 4). Es el patrón que más distingue a un senior backend: garantiza "se guardó el cambio ⟺ se publicará el evento".

**Ejercicios 3**
3.1 Explicá el problema del dual write con el ejemplo de guardar un pedido y publicar su evento. ¿Cuáles son los dos modos de fallo?
3.2 ¿Cómo logra el patrón Outbox la atomicidad entre el cambio de negocio y el evento?
3.3 ¿Cuál es la diferencia entre leer el outbox por polling y por CDC (Debezium)? ¿Qué costo introduce el patrón que obliga a consumidores idempotentes?

---

## Módulo 4 — Idempotencia y el patrón Inbox

**Teoría.** Como el Outbox (y las colas at-least-once) entregan eventos **más de una vez**, los consumidores **deben ser idempotentes**: procesar el mismo evento dos veces produce el mismo resultado que una. Ya viste el concepto en el senior y en Redis; acá el patrón formal del lado consumidor: el **Inbox** (o tabla de mensajes procesados / idempotency keys).

La idea: cada evento tiene un **id único**. Antes de procesarlo, el consumidor registra ese id; si ya está registrado, **descarta** el evento (ya lo procesó).

```ts
async function manejar(evento: { id: string; tipo: string; payload: unknown }) {
  await db.transaction(async (tx) => {
    // intentamos registrar el id; si ya existe, falla el insert (unique) → ya procesado
    const yaProcesado = await tx
      .insert(inbox)
      .values({ eventoId: evento.id })
      .onConflictDoNothing()
      .returning();

    if (yaProcesado.length === 0) return; // duplicado: no reprocesamos

    await procesarLogica(evento, tx);     // el efecto y el registro, en la MISMA transacción
  });
}
```

La clave: registrar el id **y** aplicar el efecto en la **misma transacción**, para que no quede un id registrado sin efecto aplicado (ni al revés). Variantes según el caso:

- **Inbox en la base** (lo de arriba): durable, transaccional con el efecto.
- **Redis `SET NX` con TTL** (del módulo de Redis): más rápido, para efectos no transaccionales con la base.
- **Operaciones naturalmente idempotentes**: marcar `estado = "confirmado"` dos veces da lo mismo; ahí no hace falta nada extra. El cuidado es para efectos **acumulativos** (cobrar, sumar, enviar).

Junto con el Outbox (módulo 3), el Inbox cierra el círculo de **exactly-once processing** sobre un transporte at-least-once: el productor garantiza "al menos una vez", el consumidor deduplica → efecto neto de "exactamente una vez".

**Ejercicios 4**
4.1 ¿Por qué un consumidor de eventos debe ser idempotente? ¿Qué patrones/transportes lo obligan?
4.2 ¿Por qué hay que registrar el id del evento y aplicar el efecto en la misma transacción, y no en dos pasos separados?
4.3 ¿Cómo colaboran Outbox (productor) e Inbox (consumidor) para lograr "exactly-once processing" sobre un transporte at-least-once?

---

## Módulo 5 — CQRS aplicado (y cuándo no)

**Teoría.** **CQRS** (Command Query Responsibility Segregation) separa el modelo de **escritura** (commands, que cambian estado) del de **lectura** (queries, que leen). Lo viste como concepto en el módulo de patrones; acá el criterio aplicado.

La motivación: las escrituras y las lecturas tienen necesidades **opuestas**. Las escrituras necesitan validar invariantes, proteger consistencia, normalización. Las lecturas necesitan ser rápidas, a menudo combinando datos de varias entidades, y se hacen muchas más veces. Forzar un único modelo para ambas obliga a compromisos. CQRS los separa: un **modelo de escritura** (normalizado, con reglas de dominio) y uno o varios **modelos de lectura (read models)** desnormalizados, optimizados para cada vista.

```
Commands ──▶ Modelo de escritura (dominio, invariantes) ──▶ [base de escritura]
                                                                  │ (evento/proyección)
                                                                  ▼
Queries  ◀── Read models (desnormalizados, por vista) ◀── [base de lectura]
```

Puntos críticos de criterio:

- **CQRS NO implica dos bases ni event sourcing.** En su forma más simple, es solo separar los objetos/handlers de comando y de query en el mismo proyecto y la misma base. Las versiones avanzadas (read models en otra base, actualizados por eventos) son **opcionales** y traen consistencia eventual entre escritura y lectura.
- **Cuándo aporta**: dominios donde lecturas y escrituras divergen mucho, lecturas que necesitan agregaciones costosas o múltiples vistas distintas del mismo dato, sistemas con mucha más lectura que escritura.
- **Cuándo es sobre-ingeniería**: un CRUD donde el modelo de lectura y el de escritura son el mismo. Ahí CQRS solo **duplica código** y, si separás bases, agrega consistencia eventual sin ningún beneficio. La mayoría de las features no necesitan CQRS.

El error de Mid que separa de Senior: aplicar CQRS "porque es arquitectura limpia" a todo el sistema. Se aplica a los **subdominios que lo justifican**, no por defecto.

**Ejercicios 5**
5.1 ¿Por qué separar el modelo de lectura del de escritura puede tener sentido? ¿Qué necesidad opuesta tiene cada uno?
5.2 ¿Es cierto que CQRS implica dos bases de datos y event sourcing? Explicá.
5.3 Dá un caso donde CQRS aporta y uno donde es sobre-ingeniería.

---

## Módulo 6 — CQRS en NestJS (`@nestjs/cqrs`)

**Teoría.** NestJS trae `@nestjs/cqrs`, que implementa CQRS con buses. Tres piezas:

- **`CommandBus`**: recibe un comando y lo despacha a su `@CommandHandler`.
- **`QueryBus`**: recibe una query y la despacha a su `@QueryHandler`.
- **`EventBus`**: publica eventos a sus `@EventsHandler`.

```ts
// el comando: una intención inmutable con sus datos
export class ConfirmarPedidoCommand {
  constructor(public readonly pedidoId: string) {}
}

// el handler: la lógica de ese comando
@CommandHandler(ConfirmarPedidoCommand)
export class ConfirmarPedidoHandler implements ICommandHandler<ConfirmarPedidoCommand> {
  constructor(@Inject("PEDIDO_REPO") private readonly repo: PedidoRepository) {}

  async execute(cmd: ConfirmarPedidoCommand): Promise<void> {
    const pedido = await this.repo.obtener(cmd.pedidoId);
    pedido.confirmar();                 // lógica de dominio (invariantes)
    await this.repo.guardar(pedido);
  }
}

// en el controller: el endpoint solo despacha, no tiene lógica
@Post(":id/confirmar")
confirmar(@Param("id") id: string) {
  return this.commandBus.execute(new ConfirmarPedidoCommand(id));
}
```

El `CommandBus` es un **Mediator** (del módulo de patrones): desacopla a quien dispara la acción (el controller) de quien la ejecuta (el handler).

**El malentendido #1, crítico para una entrevista:** `@nestjs/cqrs` es un **bus en proceso (in-process)**, dentro de **una sola instancia** de tu app. **No** distribuye eventos entre microservicios ni los persiste. Sirve para **organizar** tu código (separar comandos/queries/eventos limpiamente) y para eventos de dominio **dentro** de un servicio. Para EDA **distribuida** (entre servicios), necesitás un transporte real: **EventBridge/SNS/SQS** (módulo de AWS), **Kafka**, **RabbitMQ** o **NATS** — que NestJS integra con sus *microservice transports*. Confundir el `EventBus` in-process con un broker distribuido es un error clásico.

**Ejercicios 6**
6.1 ¿Qué hacen el `CommandBus`, el `QueryBus` y el `EventBus` de `@nestjs/cqrs`? ¿Qué patrón (del módulo de patrones) implementa el bus?
6.2 ¿`@nestjs/cqrs` distribuye eventos entre microservicios? Explicá qué es y qué no es.
6.3 Si necesitás propagar un evento a otro servicio, ¿qué usarías en lugar (o además) del `EventBus` in-process?

---

## Módulo 7 — Event Sourcing: guardar el historial, no el estado

**Teoría.** **Event Sourcing (ES)** invierte cómo persistís: en vez de guardar el **estado actual** (una fila que actualizás), guardás la **secuencia inmutable de eventos** que llevaron a ese estado. El estado se **reconstruye** reproduciendo los eventos. Es como un libro contable: no borrás, agregás asientos.

```
CRUD tradicional:        cuenta { id, saldo: 150 }   ← se sobreescribe en cada cambio

Event Sourcing:          [DineroDepositado(100), DineroDepositado(80), DineroRetirado(30)]
                         saldo actual = reproducir los eventos = 150
                         (el historial completo queda para siempre)
```

```ts
class CuentaBancaria {
  private saldo = 0;
  private nuevosCambios: DomainEvent[] = [];

  depositar(monto: number): void {
    if (monto <= 0) throw new Error("Monto inválido");      // invariante
    this.aplicar({ tipo: "DineroDepositado", monto });
  }

  retirar(monto: number): void {
    if (monto > this.saldo) throw new Error("Saldo insuficiente"); // invariante
    this.aplicar({ tipo: "DineroRetirado", monto });
  }

  private aplicar(evento: DomainEvent): void {
    this.mutar(evento);                 // cambia el estado en memoria
    this.nuevosCambios.push(evento);    // y registra el evento para persistir
  }

  private mutar(evento: DomainEvent): void {
    if (evento.tipo === "DineroDepositado") this.saldo += evento.monto;
    if (evento.tipo === "DineroRetirado") this.saldo -= evento.monto;
  }

  // reconstruir desde el historial (rehidratación)
  static rehidratar(eventos: DomainEvent[]): CuentaBancaria {
    const cuenta = new CuentaBancaria();
    for (const e of eventos) cuenta.mutar(e); // reproduce sin re-registrar
    return cuenta;
  }
}
```

Conceptos:

- **Event store**: la base de los eventos. Una tabla `events(stream_id, version, tipo, payload)` en Postgres alcanza para empezar; o un store dedicado (**EventStoreDB**, rebrandeado a **KurrentDB**).
- **Optimistic concurrency**: una restricción `UNIQUE(stream_id, version)` evita que dos escrituras concurrentes pisen el mismo aggregate (si las dos intentan la versión N, una falla y reintenta).
- **Snapshots**: reproducir miles de eventos en cada lectura es lento; cada N eventos guardás un "snapshot" del estado para no reconstruir desde cero.

Qué ganás: **auditoría perfecta** (sabés *exactamente* qué pasó y cuándo, gratis), poder reconstruir estados pasados, y derivar nuevas vistas reproduciendo el historial. Qué cuesta: complejidad alta, te **obliga a CQRS** (no podés hacer queries ad-hoc sobre eventos), schema evolution para siempre (módulo 10), y problemas como GDPR/"derecho al olvido" sobre un store inmutable (se resuelve con *crypto-shredding*).

**Ejercicios 7**
7.1 ¿Cuál es la diferencia fundamental entre el CRUD tradicional y Event Sourcing en cómo se persiste el estado?
7.2 ¿Para qué sirve la restricción `UNIQUE(stream_id, version)` en el event store?
7.3 ¿Por qué se usan snapshots? Nombrá una ventaja y un costo de Event Sourcing.

---

## Módulo 8 — Projections y read models

**Teoría.** Si guardás eventos (Event Sourcing), no podés consultarlos como una tabla normal ("dame todas las cuentas con saldo > 1000" no se responde sobre un log de eventos). Por eso ES **obliga a CQRS**: construís **projections** (proyecciones) — vistas de lectura desnormalizadas, derivadas de los eventos, optimizadas para tus queries.

Una projection es un consumidor de eventos que mantiene actualizada una tabla de lectura:

```ts
// proyección que mantiene una tabla "saldos" lista para consultar
@EventsHandler(DineroDepositado, DineroRetirado)
export class SaldosProjection {
  async handle(evento: DineroDepositado | DineroRetirado): Promise<void> {
    const delta = evento.tipo === "DineroDepositado" ? evento.monto : -evento.monto;
    await db.update(saldos)
      .set({ saldo: sql`saldo + ${delta}` })
      .where(eq(saldos.cuentaId, evento.cuentaId));
    // ahora "SELECT saldo FROM saldos WHERE ..." es instantáneo
  }
}
```

Propiedades clave:

- **Reconstruibles**: como las projections derivan de los eventos, podés **borrarlas y reconstruirlas** reproduciendo el historial desde cero. Esto es superpoderoso: ¿necesitás una vista nueva? La generás reproduciendo los eventos viejos. ¿Una projection tiene un bug? La borrás y la reconstruís corregida.
- **Eventualmente consistentes**: la projection se actualiza **después** del evento, así que hay un **lag** entre la escritura y que la lectura lo refleje (la consistencia eventual del módulo 2). Hay que diseñar la UX para tolerarlo.
- **Múltiples projections del mismo evento**: del mismo `DineroDepositado` podés alimentar una vista de saldos, un reporte mensual y un feed de actividad — cada una optimizada para su query.

El ejercicio mental que demuestra que entendiste: **borrar una projection y reconstruirla desde los eventos**. Si podés hacerlo, entendiste por qué los eventos son la fuente de verdad y las vistas son derivadas y descartables.

**Ejercicios 8**
8.1 ¿Por qué Event Sourcing obliga a usar projections / CQRS para las lecturas?
8.2 ¿Qué significa que una projection sea "reconstruible" y por qué es tan valioso?
8.3 ¿Por qué hay lag entre una escritura y que una projection la refleje, y qué implica para la UX?

---

## Módulo 9 — Sagas: coreografía vs. orquestación

**Teoría.** Cuando una operación cruza **varios servicios o aggregates** y no podés tener una transacción única (módulo 2), usás una **Saga**: una secuencia de pasos locales, cada uno con su **compensación** (un "deshacer") por si un paso posterior falla. Lo viste en el senior; acá los dos estilos de implementarla.

Ejemplo: confirmar un pedido = reservar stock → cobrar → agendar envío. Si el cobro falla, hay que **compensar** el paso previo (liberar el stock reservado). No hay rollback automático entre servicios; lo programás vos.

**Coreografía (choreography)**: cada servicio **reacciona a eventos** y emite el suyo. No hay coordinador central; el flujo "emerge" de quién escucha a quién.

```
StockReservado ──▶ (Pagos escucha) ──▶ PagoCobrado ──▶ (Envíos escucha) ──▶ EnvíoAgendado
                                          │
                                       PagoRechazado ──▶ (Stock escucha) ──▶ StockLiberado (compensación)
```

- A favor: máximo desacople, sin punto central, fácil de extender.
- En contra: con muchos servicios, **nadie ve el flujo completo** — está repartido en quién-escucha-qué. Debuggear "¿por qué este pedido quedó a medias?" se vuelve arqueología.

**Orquestación (orchestration)**: un **orquestador** central dirige el flujo paso a paso, llama a cada servicio y maneja las compensaciones. En AWS, **Step Functions** (módulo de AWS); en código, una state machine o el patrón Saga de `@nestjs/cqrs`.

- A favor: el flujo es **explícito y visible** en un solo lugar; fácil de razonar, debuggear y modificar.
- En contra: acopla el flujo al orquestador (menos "puro"); un componente más.

El criterio: **coreografía para flujos simples y pocos servicios** (desacople máximo); **orquestación cuando el flujo es complejo, con muchos pasos, condiciones y compensaciones** (la visibilidad gana). Y el recordatorio del senior: en un **monolito con una sola base, no necesitás saga** — usás una transacción ACID. Proponer una saga ahí es sobre-ingeniería.

**Ejercicios 9**
9.1 ¿Qué es una compensación en una saga y por qué hace falta (por qué no hay rollback automático)?
9.2 Contrastá coreografía y orquestación: ¿cuál da más desacople y cuál más visibilidad del flujo?
9.3 ¿Cuándo NO necesitás una saga? ¿Qué usás en su lugar?

---

## Módulo 10 — Versionado de eventos (schema evolution)

**Teoría.** Los eventos son **inmutables y para siempre** (sobre todo en Event Sourcing: los eventos viejos quedan guardados eternamente). Pero el negocio cambia: un día `PedidoConfirmado` necesita un campo nuevo, o cambia el significado de uno. ¿Cómo evolucionás el esquema sin romper los eventos ya guardados ni a los consumidores existentes? Esto es **schema evolution**, y es un costo permanente que mucha gente subestima.

Estrategias:

- **Cambios aditivos (preferido)**: agregar campos **opcionales**. Los consumidores viejos los ignoran; los nuevos los usan. Nunca renombres ni borres campos de un evento ya publicado.
- **Versionado explícito**: cuando un cambio no es aditivo, creás una versión nueva del evento (`PedidoConfirmado.v2`) y mantenés ambas.
- **Upcasting**: una función que transforma eventos viejos al formato nuevo **al leerlos**, así el resto del código solo conoce la versión actual. Los eventos guardados no se tocan; se "actualizan" en el momento de leerlos.

```ts
// upcaster: convierte un evento v1 al formato v2 al leerlo del store
function upcast(evento: EventoGuardado): EventoActual {
  if (evento.tipo === "PedidoConfirmado" && evento.version === 1) {
    return { ...evento, version: 2, moneda: "USD" }; // campo nuevo con default razonable
  }
  return evento;
}
```

- **Schema registry**: en sistemas con Kafka/EventBridge, un registro central (Confluent Schema Registry, **Apicurio**) valida que los productores y consumidores respeten esquemas compatibles, detectando rupturas antes de desplegar.

La regla de oro: **un evento publicado es un contrato público; tratalo como una API que no podés romper**. Los consumidores (que no controlás) dependen de su forma. Por eso los cambios aditivos y el upcasting son la norma, no renombrar/borrar.

**Ejercicios 10**
10.1 ¿Por qué evolucionar el esquema de los eventos es un problema, especialmente con Event Sourcing?
10.2 ¿Qué es un "cambio aditivo" y por qué es la estrategia preferida?
10.3 ¿Qué hace un upcaster y por qué permite cambiar el formato sin tocar los eventos ya guardados?

---

## Módulo 11 — Resiliencia y observabilidad de EDA

**Teoría.** Un sistema event-driven distribuido falla de formas que un monolito no, y es **más difícil de observar** (un flujo cruza varios servicios async). Acá se reúne lo de resiliencia (senior), colas (Redis/AWS) y observabilidad (senior/AWS).

Resiliencia:

- **Reintentos con backoff exponencial + jitter**: ante un fallo transitorio, reintentar esperando cada vez más (1s, 2s, 4s…) con un componente aleatorio, para no generar una "tormenta de reintentos". Reintentá solo errores **transitorios** (timeouts, 5xx), nunca de negocio.
- **Dead-letter queue (DLQ)**: tras N intentos, el mensaje "envenenado" va a una DLQ para inspección, sin bloquear la cola. **Una DLQ no es un cementerio**: hay que **alertarla** (un mensaje en la DLQ es un problema que alguien debe ver).
- **Idempotencia** (módulo 4): obligatoria, porque vas a reprocesar.

Observabilidad (el desafío central de EDA):

- **Correlation ID / trace ID**: un id único que viaja **en cada evento** y atraviesa todos los servicios de un flujo. Sin él, reconstruir "qué le pasó a este pedido" entre 5 servicios async es imposible. Es **no negociable** en EDA.
- **Tracing distribuido**: OpenTelemetry / X-Ray (módulo de AWS) para ver el flujo completo de un request a través de los servicios y los eventos.
- **Métricas de la cola/bus**: profundidad de cola (¿se acumulan mensajes?), edad del mensaje más viejo, tasa de mensajes en DLQ, lag de las projections.

El principio: en EDA, la observabilidad **no es opcional**. En un monolito podés debuggear con un stack trace; en un sistema distribuido async, sin correlation IDs y tracing estás ciego.

**Ejercicios 11**
11.1 ¿Por qué un `correlationId` que viaja en cada evento es imprescindible en una arquitectura event-driven?
11.2 ¿Por qué "una DLQ no es un cementerio"? ¿Qué hay que hacer con ella?
11.3 Nombrá dos métricas de una cola/bus que conviene monitorear y qué problema detecta cada una.

---

## Módulo 12 — El criterio: cuándo NO

**Teoría.** El módulo más importante, y el que define a un Tech Lead. Todo lo anterior es **poderoso y caro**. Aplicarlo donde no corresponde es el error más común y costoso. El criterio:

- **Empezá con un monolito modular + transacciones ACID.** Mientras tus invariantes vivan en una base, una transacción resuelve la consistencia sin nada de esto. Es la opción correcta para la enorme mayoría de los productos.
- **Eventos de dominio in-process antes que EDA distribuida.** Podés tener el desacople de eventos **dentro** de un monolito (`@nestjs/cqrs` EventBus) sin la complejidad de brokers, outbox y consistencia eventual entre servicios. Da el 80% del beneficio (límites claros) con el 20% del costo.
- **Event Sourcing solo en subdominios con auditoría fuerte, temporalidad o lógica rica** (finanzas, ledgers, inventario, reservas). Para un CRUD, una tabla `audit_log` cubre el 80% de la necesidad sin el costo permanente de ES (schema evolution para siempre, CQRS obligatorio, GDPR sobre datos inmutables).
- **CQRS solo donde lectura y escritura divergen de verdad.** Si el modelo es el mismo, duplica código sin beneficio.
- **Saga solo cuando el flujo cruza límites de servicio/DB reales.** En una sola base, transacción.
- **EDA distribuida cuando la distribución es inevitable** (servicios separados por escala/equipo/aislamiento), no "para escalar" sin tener el problema de escala.

> El meta-error (del archivo senior): adoptar estos patrones "porque es lo moderno" o "por las dudas". Cada uno cambia complejidad de código por **complejidad operativa y cognitiva**, que es peor. La pregunta del Tech Lead no es "¿cómo aplico event sourcing?", sino "¿este dominio **necesita** event sourcing, o una tabla con un `audit_log` alcanza?". Saber decir "esto todavía no lo necesitamos" y defenderlo con trade-offs es la habilidad que más vale.

**Ejercicios 12**
12.1 Un equipo quiere arrancar un MVP con Event Sourcing y EDA distribuida "para que escale". ¿Qué recomendás y por qué?
12.2 ¿Cómo obtenés el desacople de eventos sin pagar el costo de una EDA distribuida, en un monolito?
12.3 ¿Para qué tipo de subdominio se justifica Event Sourcing y para cuál una simple tabla `audit_log`?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 (a) comando (intención de cambiar estado). (b) evento (hecho ya ocurrido, en pasado).
    (c) query (pregunta que no cambia nada). El evento está en pasado porque describe algo
    que YA pasó: no se rechaza, se notifica.
1.2 Es la dependencia de que el otro servicio esté disponible JUSTO cuando lo llamás. Al
    emitir un evento, el emisor sigue sin esperar respuesta; el consumidor reacciona cuando
    puede (incluso si estuvo caído un rato), así que ya no dependen de estar vivos a la vez.
1.3 Ventaja: agregás consumidores nuevos (analytics, notificaciones) sin tocar al emisor.
    Desventaja: nadie tiene la vista completa de qué pasa con un evento (consistencia
    eventual, flujo difícil de rastrear sin observabilidad).
```

### Módulo 2
```
2.1 Es que el sistema se vuelve consistente CON EL TIEMPO, no al instante: tras un cambio,
    distintas partes pueden ver estados distintos por una ventana. Ganás desacople y
    tolerancia a fallos; pagás que hay un lag y hay que diseñar para estados intermedios.
2.2 Consistencia fuerte mientras puedas mantener las invariantes dentro de una sola base
    (una transacción ACID lo resuelve y es mucho más simple). Bajás a eventual solo cuando
    la distribución es inevitable (servicios separados), aceptando su costo conscientemente.
2.3 Porque en sistemas distribuidos siempre puede haber reintentos/duplicados (no hay
    garantía de entregar exactamente una vez). Se logra el efecto con at-least-once (puede
    duplicar) + consumidores idempotentes: el efecto neto es procesar una sola vez.
```

### Módulo 3
```
3.1 Querés guardar el pedido (Postgres) Y publicar el evento (cola/bus), dos sistemas sin
    transacción común. Modos de fallo: (1) el proceso muere tras guardar y antes de
    publicar → el pedido existe pero nadie se entera; (2) publicás y luego falla el guardado
    → los consumidores procesan un pedido que no existe.
3.2 Escribiendo el evento en una tabla `outbox` de la MISMA base, dentro de la MISMA
    transacción que el cambio de negocio: como es una sola transacción, o se guardan ambos
    (cambio + evento) o ninguno. No hay ventana de fallo.
3.3 Polling: un cron consulta la tabla outbox cada X (simple). CDC/Debezium: lee el WAL de
    Postgres y publica los cambios automáticamente (menor latencia, más infra). Costo del
    patrón: el evento puede publicarse más de una vez → los consumidores deben ser
    idempotentes.
```

### Módulo 4
```
4.1 Porque va a recibir el mismo evento más de una vez: el patrón Outbox puede republicar,
    y las colas/brokers entregan "al menos una vez" (at-least-once). Procesar dos veces no
    idempotentemente duplica efectos (cobros, emails).
4.2 Para que no quede inconsistencia: si registraras el id en una transacción y aplicaras
    el efecto en otra, un fallo en el medio dejaría el id marcado como procesado sin efecto
    aplicado (o el efecto sin registrar, reprocesándose). En la misma transacción, ambos
    pasan o ninguno.
4.3 El productor (Outbox) garantiza que el evento se publica al menos una vez; el consumidor
    (Inbox) deduplica por id y descarta repetidos. El efecto neto es que cada evento se
    procesa exactamente una vez (exactly-once processing), sobre un transporte at-least-once.
```

### Módulo 5
```
5.1 Porque escritura y lectura tienen necesidades opuestas: la escritura valida invariantes
    y protege consistencia (normalizado); la lectura necesita ser rápida y suele combinar
    datos de varias entidades, y se hace mucho más seguido. Separarlas deja optimizar cada
    lado sin comprometer al otro.
5.2 No. En su forma simple, CQRS es solo separar handlers/objetos de comando y de query en
    el mismo proyecto y la misma base. Read models en otra base y event sourcing son
    versiones AVANZADAS y opcionales (que además agregan consistencia eventual).
5.3 Aporta: un dashboard muy consultado con vistas agregadas distintas del modelo de
    escritura, o un sistema con muchísima más lectura que escritura. Sobre-ingeniería: un
    CRUD donde el modelo de lectura y el de escritura son idénticos.
```

### Módulo 6
```
6.1 CommandBus despacha comandos a su CommandHandler; QueryBus despacha queries a su
    QueryHandler; EventBus publica eventos a sus EventsHandler. El bus implementa el patrón
    Mediator: desacopla a quien dispara (controller) de quien ejecuta (handler).
6.2 No distribuye entre microservicios. Es un bus IN-PROCESS, dentro de una sola instancia:
    organiza el código (separa comandos/queries/eventos) y maneja eventos de dominio dentro
    de un servicio. No persiste eventos ni los manda por red.
6.3 Un transporte real de mensajería: EventBridge/SNS/SQS (AWS), Kafka, RabbitMQ o NATS,
    integrados con los microservice transports de NestJS. El EventBus in-process no cruza
    los límites del proceso.
```

### Módulo 7
```
7.1 El CRUD guarda el estado actual y lo sobreescribe en cada cambio (perdés el historial).
    Event Sourcing guarda la secuencia inmutable de eventos; el estado actual se reconstruye
    reproduciéndolos. El historial completo queda para siempre.
7.2 Para optimistic concurrency: garantiza que no haya dos eventos con la misma versión en
    el mismo stream. Si dos escrituras concurrentes intentan la versión N, una falla (viola
    el unique) y debe reintentar, evitando que se pisen.
7.3 Porque reproducir miles de eventos en cada lectura es lento; un snapshot guarda el
    estado cada N eventos para no reconstruir desde cero. Ventaja: auditoría perfecta /
    historial completo. Costo: complejidad alta (obliga a CQRS, schema evolution eterna,
    GDPR sobre datos inmutables).
```

### Módulo 8
```
8.1 Porque sobre un log de eventos no podés hacer queries ad-hoc ("cuentas con saldo >
    1000"). Necesitás derivar vistas de lectura (projections) desnormalizadas y optimizadas
    para tus queries: eso es la "Q" de CQRS aplicada a Event Sourcing.
8.2 Que se puede borrar y volver a generar reproduciendo los eventos desde el principio,
    porque deriva de ellos. Es valioso porque podés crear vistas nuevas a partir del
    historial viejo y corregir una projection con bug reconstruyéndola.
8.3 Porque la projection se actualiza DESPUÉS de que ocurre el evento (es un consumidor), así
    que hay un retraso entre la escritura y que la lectura lo refleje. Implica diseñar la UX
    para tolerarlo (mostrar "procesando", optimistic UI, etc.).
```

### Módulo 9
```
9.1 Una compensación es la acción que deshace un paso previo cuando un paso posterior falla
    (ej. liberar el stock si el cobro falla). Hace falta porque entre servicios/bases
    distintas no hay rollback transaccional automático: el "deshacer" lo programás vos.
9.2 Coreografía: cada servicio reacciona a eventos y emite el suyo, sin coordinador → MÁS
    desacople, pero nadie ve el flujo completo. Orquestación: un coordinador central dirige
    los pasos → MÁS visibilidad y fácil de debuggear, pero acopla el flujo al orquestador.
9.3 Cuando todo vive en una sola base: ahí usás una transacción ACID, que te da atomicidad
    real sin compensaciones. La saga es para flujos que cruzan límites de servicio/DB donde
    no hay transacción única.
```

### Módulo 10
```
10.1 Porque los eventos son inmutables y permanentes (en ES quedan para siempre), pero el
     negocio cambia y el esquema necesita evolucionar. Hay que cambiar el formato sin romper
     los eventos ya guardados ni a los consumidores que dependen de la forma anterior.
10.2 Agregar campos opcionales nuevos sin renombrar ni borrar los existentes. Es preferido
     porque los consumidores viejos siguen funcionando (ignoran lo nuevo) y los nuevos usan
     los campos agregados: no rompe nada.
10.3 Un upcaster transforma un evento viejo al formato nuevo AL LEERLO del store, así el
     resto del código solo trabaja con la versión actual. Permite cambiar el formato sin
     tocar los eventos persistidos (se "actualizan" en memoria al leerlos, no en disco).
```

### Módulo 11
```
11.1 Porque un flujo cruza varios servicios de forma asíncrona; sin un id común que viaje en
     cada evento, no podés correlacionar los logs/trazas de los distintos servicios para
     reconstruir qué le pasó a una operación. Es la única forma de seguir el flujo completo.
11.2 Porque los mensajes que caen en la DLQ son problemas reales (mensajes que fallaron N
     veces) que alguien debe ver y resolver. Hay que alertar sobre la DLQ y tener un proceso
     para inspeccionar y reprocesar (redrive), no dejarla acumular ignorada.
11.3 Profundidad de cola (mensajes acumulándose → los consumidores no dan abasto) y tasa de
     mensajes en DLQ (algo está fallando sistemáticamente). También: edad del mensaje más
     viejo y lag de las projections.
```

### Módulo 12
```
12.1 No. Recomendar monolito modular + transacciones ACID. Con un MVP y un dominio aún no
     entendido, ES y EDA distribuida son complejidad enorme (operación, consistencia
     eventual, schema evolution) que frena al equipo sin resolver un problema de escala que
     todavía no tienen. Se introducen cuando un dolor concreto lo justifique.
12.2 Usando eventos de dominio IN-PROCESS (ej. el EventBus de @nestjs/cqrs) dentro del
     monolito: tenés límites claros y desacople entre módulos sin brokers, outbox ni
     consistencia eventual entre servicios. El 80% del beneficio con el 20% del costo.
12.3 Event Sourcing se justifica en subdominios con auditoría fuerte, temporalidad o lógica
     de dominio rica (finanzas, ledgers, inventario, reservas). Para un CRUD sin esas
     necesidades, una tabla audit_log cubre el 80% sin el costo permanente de ES.
```

---

## Siguientes pasos

Con este módulo tenés los patrones de EDA que se implementan y se defienden en una entrevista de Tech Lead: Outbox e Inbox para mensajería confiable, CQRS y Event Sourcing para separar y auditar, sagas para transacciones distribuidas, y el versionado y la observabilidad que sostienen todo — más el criterio de cuándo NO. El recorrido desde acá: **(1)** implementá el Outbox en tu "Task API" (tabla + publisher) y consumí los eventos con idempotencia; **(2)** modelá un subdominio con `@nestjs/cqrs` (eventos in-process) antes de saltar a distribuido; **(3)** si un flujo lo pide, llevá una saga a **Step Functions** (módulo de AWS); **(4)** seguí con **NoSQL hands-on** (DynamoDB/MongoDB, donde el estado de proyecciones y agentes suele vivir) y **observabilidad práctica** (OpenTelemetry), que cierran el stack distribuido. Y siempre, el hilo del senior: la mejor arquitectura es la más simple que resuelve el problema real.
