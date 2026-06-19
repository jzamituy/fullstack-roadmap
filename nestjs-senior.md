# NestJS a nivel Senior — los temas a fondo

**No es saber más patrones, es saber elegirlos, combinarlos y justificarlos.**

> Esta página es densa a propósito. La idea no es memorizar, sino entender *por qué* cada decisión y *cuándo* conviene lo contrario. Cada sección tiene teoría clara, pros y contras, y código solo donde ayuda a ver el porqué. Al final hay **casos de decisión** (con soluciones) para entrenar criterio, que es lo que realmente evalúan en una entrevista Senior.

**Contenido**
1. Criterio de arquitectura (el tema central)
2. Domain-Driven Design (DDD)
3. Consistencia y transacciones
4. Resiliencia
5. NestJS por dentro (DI avanzada, módulos dinámicos, microservicios)
6. Observabilidad y performance
7. Estrategia de testing
8. Seguridad en profundidad
9. Comunicar y multiplicar (ADRs, convenciones)
10. Casos de decisión (ejercicios + soluciones)

---

## 1. Criterio de arquitectura

La diferencia entre Mid y Senior no es conocer Clean Architecture o microservicios. Es saber **cuándo aplicarlos y cuándo evitarlos**. Un Mid agrega capas porque "es lo correcto"; un Senior se pregunta *qué problema resuelve esta capa y qué cuesta mantenerla*. Toda decisión de arquitectura es un intercambio: ganás algo (flexibilidad, testabilidad, escalabilidad) y pagás algo (complejidad, indirección, tiempo). Senior es ver ambos lados antes de decidir.

### 1.1 La regla de oro: complejidad accidental vs. esencial

La **complejidad esencial** es la del problema en sí (un sistema de facturación con impuestos por país es complejo, no hay vuelta). La **complejidad accidental** es la que vos agregás con tus decisiones (cinco capas de abstracción para un CRUD de notas). El trabajo del Senior es minimizar la accidental sin esconder la esencial.

Síntoma de que metiste complejidad accidental: para entender un cambio simple tenés que abrir ocho archivos y saltar entre cuatro interfaces que tienen una sola implementación. Una abstracción que nunca va a tener un segundo caso de uso **no es flexibilidad, es ruido**.

### 1.2 El espectro de arquitecturas (de menos a más complejo)

No hay una "mejor" arquitectura; hay una adecuada para tu contexto (tamaño del equipo, madurez del dominio, escala real, no imaginada).

**Monolito en capas simple.** Controller → Service → Repository, todo en un proyecto, una base de datos.

- A favor: rapidísimo de construir, fácil de entender, deploy simple, transacciones triviales (todo en una DB), refactors baratos.
- En contra: si crece sin disciplina, los módulos se acoplan y se vuelve un "big ball of mud".
- Cuándo: casi siempre al empezar. La mayoría de los productos viven felices acá toda su vida.

**Monolito modular.** Un solo deploy y (normalmente) una sola DB, pero internamente dividido en módulos con límites estrictos: cada módulo expone una interfaz pública y oculta su interior. En Nest esto cae natural porque un provider solo se ve fuera de su módulo si está en `exports`.

- A favor: el 80% de los beneficios de microservicios (límites claros, equipos que trabajan en paralelo, posibilidad de extraer un servicio después) con el 20% del costo operativo. Es el *sweet spot* para casi todo producto que crece.
- En contra: requiere disciplina; nada técnico te impide saltarte un límite si no lo cuidás.
- Cuándo: cuando el monolito simple empieza a doler por acoplamiento, antes de pensar en microservicios.

**Microservicios.** Servicios independientes, cada uno con su deploy, su base de datos y su ciclo de vida, comunicándose por red (HTTP, gRPC, mensajería).

- A favor: escalado y deploy independientes, aislamiento de fallos, equipos totalmente autónomos, libertad de stack por servicio.
- En contra: cambiás complejidad de código por **complejidad operativa y distribuida**, que es mucho peor. Aparecen: latencia de red, fallos parciales, consistencia eventual, transacciones distribuidas, observabilidad difícil, versionado de contratos, y un costo de infraestructura y DevOps real.
- Cuándo: cuando tenés un dolor concreto que solo los microservicios resuelven (un módulo necesita escalar 100x más que el resto, equipos que se pisan en el deploy, requisitos de aislamiento). **Nunca** "porque es lo moderno" o "por las dudas".

> El error Senior-killer en entrevistas es proponer microservicios para un MVP. La respuesta que impresiona es: "Empezaría con un monolito modular con límites claros; si más adelante un módulo necesita escalar o un equipo se independiza, ya está aislado para extraerlo". Eso demuestra que entendés el costo.

### 1.3 Por qué "monolito primero" casi siempre gana

El problema de arrancar con microservicios es que **todavía no entendés tu dominio**. Los límites entre servicios son lo más caro de cambiar (mover código entre dos repos con dos DBs y dos deploys es brutal comparado con mover una carpeta). Si trazás esos límites antes de entender el dominio, casi seguro los trazás mal, y quedás con servicios que tienen que llamarse entre sí para cualquier operación — lo peor de los dos mundos (un "monolito distribuido").

El monolito modular te deja **mover los límites baratos** mientras aprendés, y extraer un servicio el día que un límite ya demostró ser estable y tener una razón real para independizarse.

### 1.4 Acoplamiento y cohesión, el verdadero criterio

Por debajo de toda esta discusión hay dos conceptos que valen más que cualquier nombre de patrón:

- **Cohesión alta**: las cosas que cambian juntas viven juntas. Si cada vez que tocás "precios" también tocás archivos en tres módulos distintos, tu cohesión es mala.
- **Acoplamiento bajo**: un módulo no necesita conocer el interior de otro para funcionar; se comunican por contratos estables.

Un Senior diseña límites guiándose por esto, no por capas técnicas. De hecho, organizar por **feature/dominio** (`usuarios/`, `pedidos/`, `pagos/`) suele dar mejor cohesión que organizar por capa técnica (`controllers/`, `services/`, `repositories/`), porque un cambio de negocio toca una carpeta, no se desparrama por todas.

```
// Por capa técnica: un cambio en "pedidos" toca 3 carpetas
src/controllers/pedido.controller.ts
src/services/pedido.service.ts
src/repositories/pedido.repository.ts

// Por feature/dominio: un cambio en "pedidos" vive en una carpeta (mejor cohesión)
src/pedidos/pedido.controller.ts
src/pedidos/pedido.service.ts
src/pedidos/pedido.repository.ts
src/pedidos/pedido.module.ts
```

### 1.5 Cuándo una abstracción vale la pena

Regla práctica para decidir si meter una interfaz/puerto: **¿existe (o existirá pronto y con certeza razonable) una segunda implementación, o necesito mockearla para testear lógica valiosa?** Si la respuesta es sí, la abstracción aporta. Si es "por si algún día", probablemente sea complejidad accidental.

Ejemplo donde **sí** aporta (un puerto de pago: vas a tener Stripe en prod y un fake en tests, y quizás otro proveedor mañana):

```ts
interface PagoPort { cobrar(monto: number, token: string): Promise<boolean>; }
// StripeAdapter en prod, PagoFalsoAdapter en tests → la abstracción se paga sola
```

Ejemplo donde **no** aporta (envolver el ORM en un repositorio genérico que solo delega, cuando solo vas a usar Postgres y testeás con una DB de prueba): agregás indirección sin ganar nada.

---

## 2. Domain-Driven Design (DDD)

DDD es el marco que le da sentido a Clean/Hexagonal y a CQRS. Sin DDD, esos patrones quedan como estructura vacía. La idea central: **el código debe modelar el negocio**, y el modelo debe hablar el mismo lenguaje que la gente del negocio (el *lenguaje ubicuo*). Si los expertos hablan de "cancelar una reserva" y tu código tiene `updateStatus(2)`, hay una brecha que causa bugs y malentendidos.

DDD tiene dos mitades: **estratégica** (cómo dividís el sistema grande) y **táctica** (cómo modelás dentro de cada parte).

### 2.1 DDD estratégico: bounded contexts

Un **bounded context** (contexto delimitado) es una frontera dentro de la cual un término significa una sola cosa. "Cliente" en el contexto de *Ventas* (con historial de compras, descuentos) no es lo mismo que "Cliente" en *Soporte* (con tickets, SLAs) ni en *Facturación* (con datos fiscales). Forzar un único modelo de "Cliente" gigante para toda la empresa es un error clásico: termina en una clase con 60 campos que nadie entiende.

La jugada Senior: **un modelo por contexto**, cada uno con solo lo que ese contexto necesita, y se comunican por IDs y eventos. Esto, además, es la guía natural para dónde podrían ir los límites de futuros módulos o microservicios.

### 2.2 DDD táctico: los bloques de construcción

**Entity (entidad).** Tiene identidad que persiste en el tiempo, más allá de sus atributos. Dos usuarios con el mismo nombre son distintos porque tienen distinto `id`. La igualdad se define por identidad, no por valores.

**Value Object (objeto de valor).** No tiene identidad; se define por sus valores y es inmutable. `Dinero(100, "USD")`, `Email("a@b.com")`, un rango de fechas. Dos value objects con los mismos valores son intercambiables. Modelar con value objects elimina una categoría enorme de bugs, porque la validación vive en el tipo:

```ts
// En vez de pasar `number` suelto para plata (propenso a errores: ¿centavos? ¿qué moneda?)
class Dinero {
  // El constructor privado recibe CENTAVOS (entero); el factory `crear` recibe
  // unidades (ej. 9.99) y las convierte. Por eso `sumar` llama al constructor
  // directo: ya está trabajando en centavos.
  private constructor(
    public readonly montoEnCentavos: number,
    public readonly moneda: string,
  ) {}

  static crear(monto: number, moneda: string): Dinero {
    if (monto < 0) throw new Error("El monto no puede ser negativo");
    return new Dinero(Math.round(monto * 100), moneda);
  }

  sumar(otro: Dinero): Dinero {
    if (otro.moneda !== this.moneda) throw new Error("Monedas distintas");
    return new Dinero(this.montoEnCentavos + otro.montoEnCentavos, this.moneda);
  }
}
// Ahora es IMPOSIBLE sumar USD + EUR por accidente: el tipo no te deja.
```

- A favor de value objects: validación centralizada, imposible crear estados inválidos, código que se lee como el negocio.
- En contra: más clases. No conviertas *todo* en value object; reservalo para conceptos con reglas (dinero, email, cantidades, rangos), no para cada string.

**Aggregate (agregado).** Un grupo de entidades y value objects que se tratan como una unidad de consistencia, con una **raíz** (aggregate root) que es la única puerta de entrada. Ejemplo: un `Pedido` (raíz) contiene `LineasDePedido`. No modificás una línea por fuera; le pedís al `Pedido` que lo haga, y el `Pedido` garantiza sus invariantes (ej. "el total no puede ser negativo", "un pedido confirmado no acepta nuevas líneas").

```ts
class Pedido {
  private lineas: LineaPedido[] = [];
  private estado: "borrador" | "confirmado" = "borrador";
  private eventos: object[] = []; // eventos de dominio acumulados

  agregarLinea(producto: string, cantidad: number) {
    if (this.estado === "confirmado")
      throw new Error("No se puede modificar un pedido confirmado"); // invariante protegida
    this.lineas.push(new LineaPedido(producto, cantidad));
  }

  confirmar() {
    if (this.lineas.length === 0)
      throw new Error("Un pedido vacío no se puede confirmar"); // invariante protegida
    this.estado = "confirmado";
    this.eventos.push(new PedidoConfirmado(/* id */)); // registra el hecho, no lo publica
  }

  // el caso de uso "drena" estos eventos tras guardar y los vuelca al outbox (sección 3.3)
  drenarEventos(): object[] {
    const e = this.eventos;
    this.eventos = [];
    return e;
  }
}
```

El agregado **registra** los eventos de dominio (no los publica): el caso de uso los drena después de guardar y los escribe en el `outbox` (sección 3.3). Esos son los mismos eventos de dominio que consumen los módulos de **Redis** (colas) y **AWS** (EventBridge).

La regla clave del agregado: **es el límite de una transacción**. Modificás un agregado por operación, y se guarda entero o no se guarda. Esto evita estados inconsistentes y te dice naturalmente dónde poner los límites transaccionales (ver sección 3).

**Domain Event (evento de dominio).** Algo relevante que pasó en el negocio, en pasado: `PedidoConfirmado`, `PagoRechazado`. Permite que otras partes reaccionen sin que el agregado las conozca (desacople). Es la base de la consistencia eventual y de integrar bounded contexts.

**Repository (en clave DDD).** No es "una capa sobre el ORM": es la abstracción de *cómo se persiste y recupera un agregado completo*, expresada en el lenguaje del dominio (`pedidoRepo.guardar(pedido)`, no `INSERT`).

### 2.3 Dónde poner la lógica: el anti-patrón del "modelo anémico"

El error más común que separa Mid de Senior: el **modelo anémico**. Es cuando tus entidades son solo bolsas de getters/setters sin comportamiento, y toda la lógica vive en los services. Funciona, pero la lógica de negocio se desparrama, se duplica y nadie sabe dónde están las reglas.

DDD propone lo contrario: la lógica que protege las invariantes de una entidad **vive en la entidad** (como `Pedido.confirmar()` arriba). El service orquesta (trae el agregado, le pide que haga algo, lo guarda), pero no contiene las reglas. Cuándo está bien tener lógica en el service: cuando coordina varios agregados o habla con infraestructura.

- A favor del modelo rico: reglas en un solo lugar, imposible saltárselas, código autoexplicativo.
- En contra / cuándo evitarlo: en un CRUD genuino sin reglas (una ABM de "categorías" con nombre y nada más), un modelo rico es ceremonia inútil. DDD se justifica donde **hay complejidad de dominio real**; no lo apliques a todo el sistema, solo a los contextos con reglas jugosas (pricing, reservas, facturación, permisos).

### 2.4 DDD en NestJS, en concreto

Un módulo Nest mapea muy bien a un bounded context. Adentro: las entidades/value objects/agregados son **clases TS puras** (sin decoradores de Nest ni del ORM, para que el dominio no dependa de infraestructura), los casos de uso son services, y los repositorios son puertos (interfaces) con adaptadores concretos. Esto conecta directo con hexagonal (sección 5 y el módulo de patrones).

---

## 3. Consistencia y transacciones

Acá se separan los seniors de verdad, porque es donde la teoría linda choca con la realidad de que las cosas fallan a la mitad.

### 3.1 Transacciones dentro de una sola base (lo manejable)

Mientras todo viva en una DB, tenés transacciones ACID gratis: varias operaciones se confirman juntas o se revierten juntas. El desafío de diseño es **dónde controlar la transacción** sin ensuciar el dominio. El agregado (sección 2.2) te da la respuesta: una transacción = guardar un agregado. El patrón que formaliza esto es **Unit of Work**: se juntan todos los cambios de una operación y se confirman en un único commit.

En Nest con un ORM, el caso de uso abre la transacción y se la pasa al repositorio; el dominio ni se entera:

```ts
async confirmarPedido(pedidoId: string) {
  await this.uow.runInTransaction(async (tx) => {
    const pedido = await this.pedidoRepo.obtener(pedidoId, tx);
    pedido.confirmar();               // lógica de dominio (pura)
    await this.pedidoRepo.guardar(pedido, tx);
    await this.stockRepo.descontar(pedido.items(), tx);
  }); // commit si todo OK, rollback si algo lanza
}
```

- A favor: simple, fuerte, sin sorpresas. Si podés mantener tus invariantes dentro de una DB, **hacelo**: es muchísimo más fácil que cualquier alternativa distribuida.
- Implica: no repartas a la ligera datos que necesitan cambiar juntos en bases distintas, porque perdés esta garantía.

### 3.2 El problema del "dual write" (por qué la consistencia distribuida es difícil)

Imaginá: confirmás un pedido (guardás en DB) y publicás un evento `PedidoConfirmado` a una cola para que otro servicio (envíos) reaccione. Son **dos sistemas distintos** (DB + cola). ¿Qué pasa si guardás en la DB pero el proceso muere antes de publicar el evento? Envíos nunca se entera. ¿Y si publicás el evento pero falla el commit de la DB? Envíos procesa un pedido que no existe. Esto es el problema del **dual write**, y no se resuelve con "primero uno y después el otro": siempre hay una ventana de fallo.

### 3.3 Patrón Outbox (la solución correcta al dual write)

En vez de escribir en la DB *y* publicar en la cola por separado, escribís **ambas cosas en la misma transacción de la DB**: el cambio de negocio y una fila en una tabla `outbox` con el evento a publicar. Como es una sola transacción, o se guardan las dos o ninguna — no hay ventana de fallo. Después, un proceso aparte lee la tabla `outbox` y publica los eventos a la cola, marcándolos como enviados.

```ts
await this.uow.runInTransaction(async (tx) => {
  const pedido = await this.pedidoRepo.obtener(id, tx);
  pedido.confirmar();
  await this.pedidoRepo.guardar(pedido, tx);
  // El evento se guarda en la MISMA transacción → atomicidad garantizada
  await this.outbox.agregar({ tipo: "PedidoConfirmado", pedidoId: id }, tx);
});
// Un worker separado publica los eventos del outbox a la cola y los marca enviados.
```

- A favor: garantía real de "se guardó el cambio ⟺ se publicará el evento" (al menos una vez).
- En contra: agregás una tabla, un worker y la posibilidad de entregar un evento **más de una vez** (de ahí la importancia de la idempotencia, abajo).
- Cuándo: siempre que un cambio de estado deba propagarse a otro sistema de forma confiable. Es de los patrones que más diferencian a un Senior backend.

El worker que publica el outbox se implementa de dos formas: **polling** de la tabla (simple, con algo de latencia y carga sobre la DB) o **CDC** (Change Data Capture, ej. Debezium leyendo el WAL de Postgres) — más complejo de operar pero sin polling. Si el orden importa, garantizalo **por agregado**, no global. Y del lado del consumidor, el complemento del outbox es el **inbox**: registrar los IDs de eventos ya procesados para deduplicar (la idempotencia de 3.4 aplicada a la recepción).

### 3.4 Idempotencia (imprescindible en sistemas distribuidos)

Una operación es **idempotente** si ejecutarla dos veces produce el mismo resultado que ejecutarla una. En sistemas distribuidos *vas* a recibir mensajes duplicados (reintentos, outbox que entrega "al menos una vez", el usuario que hace doble clic). Si "procesar pago" no es idempotente, cobrás dos veces.

La técnica habitual: una **clave de idempotencia** (un ID único de la operación) que registrás; si llega de nuevo la misma clave, devolvés el resultado anterior en vez de reejecutar.

```ts
async procesarPago(idemKey: string, monto: Dinero) {
  const previo = await this.idem.buscar(idemKey);
  if (previo) return previo;                 // ya se procesó: no cobramos de nuevo
  const resultado = await this.gateway.cobrar(monto);
  await this.idem.guardar(idemKey, resultado);
  return resultado;
}
```

### 3.5 Saga (transacciones distribuidas)

Cuando una operación cruza varios servicios/agregados y no podés tener una transacción única, usás una **Saga**: una secuencia de pasos locales, donde cada paso tiene una **compensación** (un "deshacer") por si un paso posterior falla. Ejemplo: reservar stock → cobrar → agendar envío. Si el cobro falla, ejecutás la compensación del paso anterior (liberar stock).

- A favor: permite consistencia (eventual) sin transacciones distribuidas reales, que son frágiles y lentas.
- En contra: complejidad alta — tenés que diseñar cada compensación, manejar estados intermedios y aceptar que el sistema pasa por estados temporalmente inconsistentes.
- Cuándo: solo en flujos distribuidos genuinos. En un monolito con una DB, **no la necesitás**: usá una transacción y listo. Proponer sagas en un monolito es sobre-ingeniería.

> Hilo conductor de toda esta sección: **la consistencia fuerte (una DB, ACID) es un lujo; conservala todo lo que puedas**. Recién cuando la distribución es inevitable, bajás a consistencia eventual con outbox, idempotencia y sagas — y aceptás su costo conscientemente.

---

## 4. Resiliencia

Un Mid programa el camino feliz; un Senior asume que **todo lo que depende de la red va a fallar** — y diseña para eso. Resiliencia es el conjunto de patrones para que un fallo parcial no se convierta en una caída total.

### 4.1 Timeouts (lo primero, siempre)

Toda llamada a algo externo (otra API, la DB, una cola) **debe tener timeout**. Sin timeout, una dependencia lenta no te da un error: te deja conexiones colgadas hasta agotar el pool, y tu servicio se cae aunque el problema fuera de otro. Un request sin timeout es una bomba de tiempo.

```ts
// Mal: si el proveedor cuelga, este await nunca vuelve y vas acumulando requests trabados
const r = await fetch(url);

// Bien: cortás a los 3s y liberás recursos
const r = await fetch(url, { signal: AbortSignal.timeout(3000) });
```

### 4.2 Retry con backoff exponencial y jitter

Si un fallo es transitorio (un blip de red, un 503 momentáneo), reintentar tiene sentido. Pero reintentar mal empeora todo: si 1000 requests fallan y todos reintentan al mismo tiempo, generás una "tormenta de reintentos" que tumba al servicio que justo se estaba recuperando. La solución: **backoff exponencial** (esperar cada vez más: 1s, 2s, 4s…) con **jitter** (un componente aleatorio para que no reintenten todos sincronizados).

Reglas: reintentá solo errores transitorios (timeouts, 5xx, 429), **nunca** errores de negocio (un 400 o "saldo insuficiente" no mejora reintentando), y solo operaciones idempotentes (sección 3.4), porque si no podés cobrar tres veces.

### 4.3 Circuit breaker

Si un servicio del que dependés está caído, seguir mandándole requests (aunque reintentes con backoff) es desperdiciar recursos y empeorar su agonía. El **circuit breaker** funciona como una térmica eléctrica: si detecta muchos fallos seguidos, "abre el circuito" y durante un tiempo **falla rápido** sin siquiera intentar la llamada (devolviendo un error inmediato o un fallback). Pasado ese tiempo, deja pasar una prueba; si funciona, "cierra" y vuelve a la normalidad.

- A favor: evita el colapso en cascada (que un servicio caído arrastre a los que dependen de él), libera recursos y le da aire al servicio caído para recuperarse.
- En contra: un estado más para entender y monitorear; mal configurado puede abrir de más.
- Cuándo: en dependencias críticas y propensas a fallar, especialmente entre microservicios.

### 4.4 Bulkhead y degradación elegante

**Bulkhead** (mamparo, como los compartimentos estancos de un barco): aislás recursos por dependencia para que un problema en una no hunda todo. Ejemplo: pools de conexiones separados para "pagos" y "recomendaciones", así si recomendaciones se satura, no te quedás sin conexiones para cobrar.

**Degradación elegante**: cuando una dependencia no esencial falla, el sistema sigue funcionando con menos. Si el servicio de recomendaciones está caído, mostrás la página sin recomendaciones, no un error 500. Senior piensa qué es esencial y qué es "lindo tener".

### 4.5 Dead-letter queue (DLQ)

En sistemas con colas: si un mensaje falla repetidamente (no por un blip, sino porque está "envenenado" — malformado o dispara un bug), no puede quedar reintentándose para siempre bloqueando la cola. Tras N intentos se manda a una **dead-letter queue**: un apartado donde queda para inspección manual sin frenar el resto. Es la red de seguridad del procesamiento asíncrono.

---

## 5. NestJS por dentro (DI avanzada, módulos dinámicos, microservicios)

Acá es donde dejás de "usar Nest" y empezás a entender *cómo funciona*, que es lo que te permite resolver los problemas raros.

### 5.1 Custom providers a fondo

Ya viste `useClass`/`useValue`/`useFactory` en el módulo de patrones. A nivel Senior, lo que importa es usarlos para **inyectar contra abstracciones** (puertos) con tokens, que es lo que habilita hexagonal:

```ts
// Token para el puerto
export const PAGO_PORT = Symbol("PagoPort");

@Module({
  providers: [
    ProcesarPagoUseCase,
    {
      provide: PAGO_PORT,                       // el caso de uso pide PAGO_PORT
      useClass: process.env.NODE_ENV === "test"
        ? PagoFalsoAdapter
        : StripeAdapter,                        // y recibe el adaptador según entorno
    },
  ],
})
export class PagosModule {}
```

Usar un `Symbol` como token (en vez de un string) evita colisiones de nombres entre módulos: es la práctica prolija.

> Nota: el ternario por `NODE_ENV` de arriba es un ejemplo simplificado. En producción el binding debería ser **fijo** (`useClass: StripeAdapter`); en los tests se sustituye con `Test.createTestingModule(...).overrideProvider(PAGO_PORT).useClass(PagoFalsoAdapter)`, sin ramificar el módulo de producción por entorno.

### 5.2 Scopes de DI y por qué importan para performance

Los providers son **singleton por defecto** (una instancia para toda la app), y eso es lo que querés el 99% del tiempo. Nest ofrece otros scopes:

- `REQUEST`: una instancia nueva por cada request. Útil si necesitás estado por request (ej. el usuario actual), pero **tiene un costo de performance real**: Nest tiene que recrear ese provider —y toda su cadena de dependencias— en cada request. Un provider request-scoped "contamina" hacia arriba: todo lo que dependa de él también pasa a recrearse por request.
- `TRANSIENT`: una instancia nueva cada vez que se inyecta.

El error caro de Mid: marcar algo como request-scoped por comodidad (para acceder al request) sin entender que degrada el rendimiento de toda la rama de dependencias. Alternativas Senior: pasar el contexto explícitamente, o usar `AsyncLocalStorage` para datos por request sin tocar los scopes.

### 5.3 Módulos dinámicos (`forRoot` / `forRootAsync`)

Cuando un módulo necesita configurarse al importarlo (un módulo de base de datos que recibe la connection string, uno de config que lee variables), se usa el patrón de módulo dinámico. La versión `Async` permite que esa config dependa de otros providers (ej. esperar a que cargue el `ConfigService`). Entender esto es lo que te deja escribir módulos reutilizables de calidad librería, no solo consumirlos.

### 5.4 Lifecycle hooks

Nest expone ganchos del ciclo de vida: `OnModuleInit` (inicializar conexiones al arrancar), `OnApplicationShutdown` (cerrarlas ordenadamente). El **graceful shutdown** es marca de Senior: cuando el orquestador (Kubernetes) manda a apagar tu pod, querés terminar los requests en curso, cerrar conexiones a la DB y dejar de tomar trabajo nuevo — no cortar de golpe y dejar transacciones a medias.

### 5.5 Nest como microservicio y comunicación

Nest tiene un módulo de microservicios con varios *transports*: TCP, Redis, NATS, RabbitMQ, Kafka, gRPC. Lo Senior no es la sintaxis, es **elegir el transport y el estilo de comunicación**:

- **Síncrono (request/response, ej. gRPC, HTTP)**: simple de razonar, pero acopla en el tiempo (si el otro servicio está caído, vos fallás). Usalo cuando necesitás la respuesta para continuar.
- **Asíncrono (mensajería/eventos, ej. Kafka, RabbitMQ)**: desacopla (publicás un evento y seguís; el consumidor lo procesa cuando puede), tolera caídas, pero introduce consistencia eventual y complejidad de razonamiento. Usalo para propagar hechos y procesos que no necesitan respuesta inmediata.
- **API Gateway / BFF**: un punto de entrada que agrega y enruta hacia los servicios internos; el patrón **BFF** (Backend For Frontend) arma respuestas a medida de cada cliente (web, mobile) para no obligar al frontend a hacer diez llamadas.

---

## 6. Observabilidad y performance

> Regla de oro: **medí antes de optimizar**. Optimizar sin datos es adivinar, y casi siempre optimizás lo que no era el cuello de botella. Senior instrumenta primero.

### 6.1 Los tres pilares de la observabilidad

- **Logs**: eventos discretos. A nivel Senior, **estructurados** (JSON, no texto suelto), con un `correlationId` que atraviese todo un request para poder seguirlo. Usá niveles (`error`/`warn`/`info`/`debug`) con criterio: loguear todo es tan inútil como no loguear nada.
- **Métricas**: números agregados en el tiempo (requests/segundo, latencia p95/p99, tasa de errores, uso de pool). Mirá **percentiles, no promedios**: un promedio de 100ms puede esconder que el 1% de usuarios espera 5s. El p99 cuenta la verdad.
- **Traces (trazas distribuidas)**: siguen un request a través de varios servicios, mostrando cuánto tardó cada salto. Imprescindible en microservicios para responder "¿dónde se fue el tiempo?". El estándar es **OpenTelemetry**.

En Nest, los **interceptors** son el lugar natural para instrumentar (medir tiempos, contar errores) sin ensuciar la lógica.

### 6.2 Performance: los sospechosos de siempre

- **Problema N+1**: hacés 1 query para traer N pedidos y luego 1 query por cada uno para sus líneas → N+1 queries. Es la causa #1 de lentitud con ORMs. Se resuelve con eager loading / joins / dataloader. Saber detectarlo (mirando los logs de SQL) es básico Senior.
- **Caché**: la estrategia más común es **cache-aside** (buscás en caché; si no está, vas a la DB y guardás en caché). Con Redis. Lo difícil no es cachear: es **invalidar** (mantener el caché coherente cuando el dato cambia). Estrategias concretas: **TTL** (expiración como red de seguridad, siempre), **write-through** (escribís en caché y DB juntas) e **invalidación dirigida por eventos de dominio** (cuando el dato cambia, un evento invalida la clave) — esto último engancha con el outbox y se detalla en el módulo de Redis. Relacionado: las **claves de idempotencia** (3.4) también necesitan TTL o limpieza periódica, o la tabla crece sin control. "Hay solo dos problemas difíciles en computación: invalidación de caché y nombrar cosas".
- **Connection pooling**: la DB tiene un límite de conexiones; reusás un pool en vez de abrir/cerrar por request. Dimensionar el pool (ni muy chico = cuello de botella, ni muy grande = saturás la DB) es ajuste fino Senior.
- **Índices**: la diferencia entre una query de 5ms y una de 5s suele ser un índice. Pertenece tanto a backend como a DB.

---

## 7. Estrategia de testing

A nivel Senior no se trata de "más tests" ni de perseguir 100% de cobertura (una métrica engañosa). Se trata de **invertir el esfuerzo donde da confianza**.

### 7.1 La pirámide (y por qué)

Muchos tests **unitarios** (rápidos, aislados, prueban lógica de dominio), una capa media de tests de **integración** (prueban que tu código habla bien con la DB, la cola, etc., usando dependencias reales en contenedores con **testcontainers**), y pocos tests **end-to-end** (lentos y frágiles, pero validan flujos completos). La pirámide es así porque los unitarios son baratos y estables y los e2e son caros y se rompen por cualquier cosa.

### 7.2 El regalo de la arquitectura: testear el puerto, no el adaptador

Acá se ve por qué hexagonal "se paga sola". Si tu caso de uso depende de `PagoPort` (interfaz), en los tests le inyectás un `PagoFalsoAdapter` y probás **toda la lógica de negocio sin red, sin claves, sin Stripe, en milisegundos**. La arquitectura limpia no es estética: es lo que hace los tests rápidos y estables.

Qué mockear y qué no (criterio Senior): mockeá lo que está **fuera de tu control** (APIs de terceros, el reloj, lo aleatorio) y lo lento. **No** mockees tu propia DB en los tests de integración — ahí justamente querés probar contra una real (en contenedor), porque los bugs de persistencia (mapeos, transacciones, constraints) no aparecen contra un mock.

### 7.3 Contract testing

Entre microservicios, los tests e2e de todo junto son frágiles. El **contract testing** verifica que cada servicio respeta el "contrato" (el formato de mensajes) que los otros esperan, probando cada servicio por separado contra ese contrato. Te da confianza de que la integración funciona sin levantar todo el sistema.

### 7.4 Más allá de la cobertura: mutation y carga

La cobertura dice qué líneas se ejecutan, no si tus aserciones valen. El **mutation testing** (ej. Stryker) introduce cambios pequeños en el código ("mutantes") y verifica que algún test falle; si ninguno se rompe, esa aserción no estaba probando nada. Es la forma concreta de medir la *calidad* de la suite, no su tamaño. Aparte, los **tests de carga/performance** (k6, Artillery) validan latencia p95/p99 y throughput bajo presión — criterio Senior que la pirámide funcional no cubre.

---

## 8. Seguridad en profundidad

Seguridad no es un módulo que agregás al final; es transversal. Lo mínimo que un Senior tiene internalizado:

- **Autenticación vs. autorización**: autenticación es *quién sos* (login, JWT), autorización es *qué podés hacer*. En Nest, autenticación con guards/estrategias (Passport) y autorización con guards de roles/permisos.
- **RBAC vs. ABAC**: control de acceso por **roles** (admin/user) es simple y alcanza para la mayoría; por **atributos** (ej. "podés editar este documento solo si sos el dueño y está en estado borrador") es más granular y necesario en dominios complejos. Saber cuál usar es criterio.
- **Validar toda entrada**: nunca confíes en el cliente. DTOs con validación, y para datos externos en runtime, algo como Zod. La inyección (SQL, etc.) nace de confiar en input no validado.
- **Rate limiting**: limitar requests por cliente para frenar abuso y ataques de fuerza bruta (en Nest, `ThrottlerModule`).
- **Secretos**: nunca en el código ni en git; en variables de entorno o un gestor de secretos. Rotables.
- **Multi-tenancy**: si servís a varios clientes, aislar sus datos correctamente (a nivel fila, esquema o base) es una decisión de arquitectura *y* de seguridad crítica.
- **OWASP Top 10**: conocer las vulnerabilidades web más comunes y cómo prevenirlas es base no negociable.

El principio que ordena todo esto: **mínimo privilegio** (cada parte tiene solo los permisos que necesita) y **defensa en profundidad** (varias capas, porque cualquiera puede fallar).

---

## 9. Comunicar y multiplicar

Esto no es "soft skill opcional": es **parte del trabajo técnico Senior**, y muchas veces lo que define el ascenso. Tu impacto deja de medirse por tu código y pasa a medirse por el del equipo.

- **ADRs (Architecture Decision Records)**: documentos cortos que registran una decisión de arquitectura, el contexto, las alternativas consideradas y las consecuencias. Valen oro: dentro de un año nadie recuerda *por qué* se eligió X, y un ADR responde sin arqueología. Escribir un buen ADR demuestra que pensaste los trade-offs.
- **Convenciones del equipo**: definir y documentar cómo se estructura un módulo, cómo se nombran las cosas, qué se testea. Reduce la fricción y las discusiones repetidas.
- **Code reviews que enseñan**: una review Senior no es "está mal"; explica *por qué* y propone el camino, dejando al otro mejor que antes.
- **Saber explicar el porqué, no solo el qué**: poder bajar una decisión compleja a lenguaje simple para un junior (o para negocio) es, justamente, la habilidad que esta misma página intenta ejercitar.

---

## 10. Casos de decisión (ejercicios + soluciones)

A nivel Senior lo que se evalúa es **criterio**. Estos casos no tienen una única respuesta "de libro": lo que importa es identificar los trade-offs. Pensá tu respuesta antes de leer la sugerida.

**Caso 1.** Un equipo de 4 personas arranca un MVP de un marketplace. El CTO propone microservicios "para que escale desde el día uno". ¿Qué recomendás?

**Caso 2.** Tenés un CRUD de "categorías de producto" (nombre, descripción). Un compañero quiere aplicarle entidades ricas de DDD, repositorios con puertos e interfaces. ¿Estás de acuerdo?

**Caso 3.** Al confirmar un pedido, tu servicio guarda en Postgres y publica un evento a Kafka para que Envíos reaccione. A veces Envíos no se entera de pedidos confirmados. ¿Qué está pasando y cómo lo resolvés?

**Caso 4.** Un endpoint llama a una API externa de un proveedor. Cuando el proveedor se pone lento, tu servicio entero deja de responder, no solo ese endpoint. ¿Qué pasó y qué patrones aplicás?

**Caso 5.** Para acceder al usuario actual en muchos services, un dev marcó varios providers como `REQUEST`-scoped. Notan que la app está más lenta. ¿Por qué y qué alternativas hay?

**Caso 6.** El equipo quiere subir la cobertura de tests al 100% y propone mockear la base de datos en todos los tests para que sean rápidos. ¿Qué objetás?

### Soluciones sugeridas

```
Caso 1. Recomendar monolito modular, NO microservicios. Con 4 personas y un
  dominio que todavía no entienden, los límites entre servicios casi seguro van
  a estar mal trazados (lo más caro de corregir), y el costo operativo de
  microservicios (DevOps, observabilidad, consistencia distribuida) los frena en
  vez de ayudarlos. Modular monolith: límites internos claros para poder extraer
  servicios después, cuando un límite demuestre ser estable y haya un motivo real
  (escala, equipo independiente). Microservicios resuelven problemas
  organizativos y de escala que un MVP de 4 personas no tiene.

Caso 2. No. DDD táctico (entidades ricas, puertos) se justifica donde hay
  COMPLEJIDAD DE DOMINIO: reglas, invariantes, lenguaje del negocio. Una ABM de
  categorías sin reglas es complejidad accidental: agregás capas e indirección
  para algo que un service + ORM resuelve en 20 líneas. DDD se aplica a los
  contextos jugosos (pricing, reservas, pagos), no a todo el sistema por igual.

Caso 3. Es el problema del dual write: guardás en la DB y publicás en Kafka como
  dos operaciones separadas; si el proceso muere entre una y otra (o falla la
  publicación), el evento se pierde. Solución: patrón Outbox. Guardás el cambio
  del pedido y una fila de evento en la MISMA transacción de Postgres (atómico);
  un worker aparte lee el outbox y publica a Kafka marcando lo enviado. Como
  puede entregar el evento más de una vez, Envíos debe ser idempotente.

Caso 4. La llamada externa no tiene timeout: cuando el proveedor se pone lento,
  los requests quedan colgados acumulando conexiones hasta agotar el pool, y cae
  todo el servicio. Aplicar: (1) timeout en la llamada; (2) circuit breaker para
  fallar rápido mientras el proveedor está mal; (3) bulkhead para aislar ese
  recurso del resto; (4) degradación elegante si esa funcionalidad no es esencial.

Caso 5. REQUEST scope obliga a Nest a recrear el provider y TODA su cadena de
  dependencias en cada request, lo que degrada el rendimiento y se propaga hacia
  arriba. Alternativas: pasar el usuario/contexto explícitamente como parámetro,
  o usar AsyncLocalStorage para tener datos por request sin volver request-scoped
  toda la rama de DI. Mantener singletons donde se pueda.

Caso 6. Dos objeciones. (1) 100% de cobertura es una métrica engañosa: mide
  líneas ejecutadas, no que las aserciones sean valiosas; perseguirla lleva a
  tests inútiles. (2) Mockear la DB en TODOS los tests elimina justo los tests de
  integración que atrapan bugs reales de persistencia (mapeos, transacciones,
  constraints, queries). Lo correcto: unitarios con mocks para la lógica de
  dominio + tests de integración contra una DB real en contenedor
  (testcontainers). La confianza importa más que el número de cobertura.
```

---

## Cómo estudiar esto

No intentes todo de una. El orden de mayor impacto para llegar a Senior: **(1) criterio de arquitectura** (sección 1) hasta que puedas defender un "depende" con trade-offs concretos; **(2) DDD + consistencia/transacciones** (2 y 3), que es probablemente tu mayor salto pendiente; **(3) resiliencia y observabilidad** (4 y 6), que es lo que distingue a quien operó sistemas reales. Lo demás se apoya en esa base. Y practicá explicar cada tema en voz alta en lenguaje simple: si no lo podés explicar claro, todavía no lo dominás.
