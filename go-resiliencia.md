# Resiliencia y workers de fondo en Go

**Sobrevivir a fallos parciales · timeouts, reintentos, circuit breakers y jobs · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya hiciste [Go para backend](go-backend.md) (goroutines, `context`, errores). Acá vemos lo que separa un servicio que "anda en la demo" de uno que **aguanta producción**: qué hacés cuando la base se pone lenta, el servicio de pagos no responde, o tu worker explota a las 3 AM. En sistemas distribuidos los fallos parciales **no son la excepción, son la norma** — y Go te da las herramientas para diseñarlos en vez de sufrirlos.

**Lo que asumimos.** Go a nivel del módulo [Go para backend](go-backend.md): `context`, goroutines/channels, errores como valores, un servidor HTTP. Si tocaste [Redis](redis.md) (colas) o [Concurrencia](concurrencia.md) (semáforos), vas a reconocer patrones; no es requisito.

> ⚠️ **Nota.** Este módulo usa librerías del ecosistema estándar-de-facto (no stdlib pura): **`golang.org/x/time/rate`** (rate limiter oficial de la extended stdlib), **`golang.org/x/sync`** (`errgroup`, `semaphore`), y para circuit breaker **`github.com/sony/gobreaker`**. Verificá versiones/APIs contra sus repos antes de citarlas como definitivas.

**Índice de módulos**
1. Por qué resiliencia: el fallo parcial es la norma
2. Timeouts: un deadline en cada borde
3. Reintentos con backoff exponencial + jitter
4. Idempotencia: el prerrequisito para reintentar
5. Circuit breaker: dejar de pegarle a lo que está caído
6. Rate limiting y bulkheads: limitar lo que mandás
7. Workers de fondo: goroutines supervisadas y drenado limpio
8. Jobs programados y colas
9. Recuperación de panics en workers
10. El criterio: componer, no reinventar

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere `go`).

---

## Módulo 1 — Por qué resiliencia: el fallo parcial es la norma

**Teoría.** En un monolito, si algo falla, todo falla y te enterás. En un sistema distribuido (tu servicio + DB + cache + servicio de pagos + cola), lo normal es el **fallo parcial**: una dependencia está **lenta**, **caída** o **intermitente**, mientras el resto anda. Diseñar para eso es lo que distingue un backend serio.

Los dos pecados capitales:
- **Esperar para siempre.** Una llamada sin timeout a un servicio colgado deja tu goroutine (y la request del cliente) esperando indefinidamente. Peor: bajo carga, esas esperas **se acumulan** y agotan tus recursos → tu servicio cae **arrastrado** por la dependencia. Es el origen de las **fallas en cascada**.
- **Reintentar a ciegas.** Si una dependencia está sobrecargada y todos tus clientes reintentan de inmediato, le tirás **más** carga justo cuando menos puede — el **retry storm** que amplifica el problema en vez de aliviarlo.

Las dos respuestas de diseño, que vamos a desarrollar:
- **Fail fast:** cortá rápido (timeout) en vez de esperar, para liberar recursos y dar una respuesta (aunque sea un error) a tiempo.
- **Degradar con gracia:** cuando una dependencia no responde, dar una respuesta parcial o de respaldo (un dato cacheado, un default) en vez de un error duro, **si el dominio lo permite**.

La frase mental: **asumí que toda llamada a la red puede ser lenta, fallar o colgarse — y decidí de antemano qué hacés en cada caso.** La resiliencia no se agrega después; se diseña.

**Ejercicios 1**
1.1 🔁 ¿Qué es un "fallo parcial" y por qué es la norma en sistemas distribuidos?
1.2 🧠 ¿Cómo puede una dependencia lenta (no caída) tumbar tu servicio entero? Nombrá el fenómeno.
1.3 🧠 ¿Por qué reintentar de inmediato y sin límite empeora una dependencia sobrecargada?

---

## Módulo 2 — Timeouts: un deadline en cada borde

**Teoría.** El timeout es la **defensa #1** y la más barata. La regla: **toda operación que cruza un borde** (red, DB, disco) **debe tener un deadline**. En Go esto se hace con `context` (lo viste en [Go para backend](go-backend.md)): el deadline **se propaga** por toda la cadena de llamadas.

```go
func (s *Servicio) Procesar(ctx context.Context, id string) error {
	// presupuesto total para esta operación
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	u, err := s.repo.Buscar(ctx, id)        // si tarda, se aborta al vencer el deadline
	if err != nil {
		return fmt.Errorf("buscar %s: %w", id, err)
	}
	return s.notificador.Enviar(ctx, u.Email) // comparte el MISMO presupuesto restante
}
```

Claves:
- **El `http.Client` por defecto NO tiene timeout.** Un cliente HTTP sin `Timeout` esperará para siempre. Siempre configurá uno (o pasá un `ctx` con deadline a cada request con `http.NewRequestWithContext`).
- **Presupuesto de tiempo, no timeouts sueltos.** El deadline del cliente original viaja por toda la cadena: si la request HTTP tiene 3 s y ya gastaste 2 en la DB, a la siguiente llamada le queda 1 s. No le pongas a cada paso su propio timeout independiente generoso: respetá el presupuesto que baja.
- **`DeadlineExceeded`** es el error que vas a ver; manejalo distinto de un error de negocio (suele merecer un `503`/reintento, no un `400`).

**Ejercicios 2**
2.1 🔁 ¿Por qué se dice que el deadline "se propaga" y qué ganás con eso en una cadena de servicios?
2.2 🧠 ¿Cuál es la trampa del `http.Client` por defecto en Go y cómo la evitás?
2.3 ✍️ Escribí una función que llame a `http.Get`-equivalente con un deadline de 2 s usando `context` y `http.NewRequestWithContext`.

---

## Módulo 3 — Reintentos con backoff exponencial + jitter

**Teoría.** Reintentar está bien — **si lo hacés con disciplina**. Tres reglas:

1. **Backoff exponencial con tope:** esperá cada vez más entre intentos (ej. 100 ms, 200 ms, 400 ms, 800 ms…), pero **con un techo** (`min(base·2^i, maxEspera)`): sin cap, el shift desborda y las esperas se vuelven absurdas. No martilles, pero tampoco esperes una eternidad.
2. **Jitter (aleatoriedad):** sumá una porción al azar a cada espera. Sin jitter, mil clientes que fallaron al mismo tiempo reintentan **sincronizados** y crean picos coordinados (el *thundering herd*). El jitter los **desparrama**. La variante canónica de AWS es el **full jitter** (`espera = rand(0, tope)`); el ejemplo de abajo usa una variante aditiva, igual de válida para el objetivo.
3. **Tope de intentos y respeto del `context`:** límite de reintentos (ej. 3-5) y abortar si el `ctx` se cancela. Reintentar para siempre es otra forma de colgarse.

> 📝 El ejemplo usa **`math/rand`** (v1). En Go 1.22+ lo idiomático es **`math/rand/v2`**, donde la función es `rand.Int64N(int64(espera))` (no `Int63n`) y no necesita *seed* global manual. No mezcles los dos paquetes.

```go
func conReintentos(ctx context.Context, intentos int, op func() error) error {
	var err error
	base := 100 * time.Millisecond
	for i := 0; i < intentos; i++ {
		if err = op(); err == nil {
			return nil // éxito
		}
		if !esTransitorio(err) {
			return err // 4xx, validación: NO reintentar
		}
		// backoff exponencial CON TOPE (cap), más jitter para desincronizar
		espera := min(base*(1<<i), 30*time.Second)          // cap: no crece sin techo (evita overflow y esperas eternas)
		espera += time.Duration(rand.Int63n(int64(espera))) // jitter +0–100% (math/rand v1)
		select {
		case <-time.After(espera):
		case <-ctx.Done():
			return ctx.Err() // el cliente se cansó; no insistas
		}
	}
	return fmt.Errorf("agotados %d intentos: %w", intentos, err)
}
```

- **Qué reintentar y qué no:** reintentá errores **transitorios** (timeout, `503 Unavailable`, conexión cortada, `429`). **Nunca** reintentes errores **determinísticos** (`400 BadRequest`, `404`, validación, `401`): van a fallar igual y desperdician recursos.
- **Respetá `Retry-After`:** si el servidor te dice cuánto esperar (header `Retry-After` en un `429`/`503`), usá ese valor en vez de tu backoff.
- **El SDK puede reintentar solo:** muchos clientes (incluido el de Anthropic, AWS, etc.) ya reintentan internamente con backoff. No apiles tu reintento encima del suyo sin saberlo — verificá antes de envolver.

**Ejercicios 3**
3.1 🔁 ¿Qué problema resuelve el *jitter* que el backoff exponencial solo no resuelve?
3.2 🧠 ¿Por qué reintentar un `400 Bad Request` es siempre un error, y qué tipo de errores sí conviene reintentar?
3.3 ✍️ Al loop de `conReintentos`, agregale que respete un `Retry-After` (en segundos) cuando el error lo traiga (asumí una función `retryAfter(err) (time.Duration, bool)`).

---

## Módulo 4 — Idempotencia: el prerrequisito para reintentar

**Teoría.** Acá está la trampa que rompe sistemas en producción: **reintentar una operación que NO es idempotente puede duplicar efectos.** Si mandás "cobrá $100", no recibís respuesta (timeout) y reintentás… ¿cobraste una vez o dos? Quizá la primera **sí** llegó y solo se perdió la respuesta.

- **Operación idempotente:** ejecutarla N veces tiene el **mismo efecto** que ejecutarla una. `GET`, `PUT` (reemplazo total), `DELETE` lo son por naturaleza. **`POST` "crear" y "cobrar" NO lo son.**
- **La solución: idempotency key.** El cliente genera una clave única por operación (un UUID) y la manda en cada intento. El servidor **registra la clave**; si llega una segunda vez, **devuelve el resultado guardado en vez de re-ejecutar**. Así el reintento es seguro aunque el efecto ya haya ocurrido.

```go
func (s *Servicio) Cobrar(ctx context.Context, idemKey string, monto int) (*Recibo, error) {
	// ¿ya procesamos esta clave?
	if r, ok, err := s.store.BuscarRecibo(ctx, idemKey); err != nil {
		return nil, err
	} else if ok {
		return r, nil // idempotente: devolvemos el resultado previo, NO re-cobramos
	}
	recibo, err := s.cobrarDeVerdad(ctx, monto)
	if err != nil {
		return nil, err
	}
	if err := s.store.GuardarRecibo(ctx, idemKey, recibo); err != nil {
		return nil, err
	}
	return recibo, nil
}
```

- ⚠️ **Ojo con la ventana de carrera del snippet:** tal cual está, entre `BuscarRecibo` y `GuardarRecibo` dos reintentos concurrentes de la misma `idemKey` podrían pasar ambos el chequeo y cobrar dos veces. La versión robusta hace `GuardarRecibo` como un **insert único atómico** (`INSERT ... ON CONFLICT` con `UNIQUE(idemKey)` en Postgres, o `SET NX` en Redis): si el insert choca, significa que otro ya cobró → leés y devolvés ese recibo (*read-after-conflict*). La atomicidad la da la DB, no el `if`.
- Regla mental: **antes de poner un reintento, preguntá "¿esta operación es idempotente?". Si no lo es, hacela idempotente primero** (con una clave), o no la reintentes.

**Ejercicios 4**
4.1 🔁 ¿Qué significa que una operación sea idempotente? Dá un ejemplo que lo sea y uno que no.
4.2 🧠 Un timeout en un "cobrar" no garantiza que el cobro no haya ocurrido. ¿Por qué, y cómo lo resolvés para poder reintentar seguro?
4.3 🧠 ¿Por qué el chequeo "¿ya existe esta idempotency key?" debe ser atómico, y qué usás para garantizarlo?

---

## Módulo 5 — Circuit breaker: dejar de pegarle a lo que está caído

**Teoría.** Si una dependencia está caída, seguir llamándola (aun con reintentos) **desperdicia tus recursos** (goroutines, conexiones, tiempo) y **le impide recuperarse**. El **circuit breaker** ("disyuntor") corta el circuito: tras detectar muchos fallos, **deja de llamar** por un rato y falla rápido, dándole aire a la dependencia.

Tiene tres estados:
- **Closed (cerrado):** todo pasa normal. Cuenta los fallos.
- **Open (abierto):** se superó el umbral de fallos → **rechaza las llamadas de inmediato** (sin siquiera intentar) durante un tiempo de espera. Acá es donde *degradás* (devolvés cache/default) o fallás rápido.
- **Half-open (semiabierto):** pasado el tiempo, deja pasar **una llamada de prueba**. Si sale bien, vuelve a *closed*; si falla, vuelve a *open*.

No lo implementes a mano: usá una librería probada como **`sony/gobreaker`**:

```go
import "github.com/sony/gobreaker/v2"

cb := gobreaker.NewCircuitBreaker[*Respuesta](gobreaker.Settings{
	Name:        "servicio-pagos",
	MaxRequests: 1,                // llamadas de prueba en half-open
	Timeout:     30 * time.Second, // cuánto queda abierto antes de probar
	ReadyToTrip: func(c gobreaker.Counts) bool {
		return c.ConsecutiveFailures > 5 // cuándo abrir
	},
})

resp, err := cb.Execute(func() (*Respuesta, error) {
	return llamarPagos(ctx, req)
})
if err != nil {
	// puede ser el error real O gobreaker.ErrOpenState (falló rápido): degradá acá
}
```

- **Combina con timeout y reintento, en orden:** timeout en cada llamada → reintento con backoff para fallos transitorios → circuit breaker por encima para cuando "transitorio" se volvió "está caído". Los tres juntos, no uno solo.
- **Por dependencia, no global:** un breaker por servicio downstream. Que pagos esté caído no debe abrir el circuito hacia la base de datos.
- **Consecutivos vs. tasa de fallos:** `ConsecutiveFailures > 5` es simple pero frágil bajo tráfico real — un fallo intermitente (1 de cada 3) nunca acumula consecutivos y el circuito no abre nunca. En alta carga se prefiere abrir por **proporción de fallos** sobre una ventana (`gobreaker.Counts` te da `Requests` y `TotalFailures`, así que `ReadyToTrip` puede calcular un *failure ratio*).

**Ejercicios 5**
5.1 🔁 Nombrá los tres estados de un circuit breaker y qué hace en cada uno.
5.2 🧠 ¿Qué problema resuelve el circuit breaker que el reintento con backoff **no** resuelve?
5.3 🧠 ¿Por qué conviene un breaker por dependencia y no uno solo para todo el servicio?

---

## Módulo 6 — Rate limiting y bulkheads: limitar lo que mandás

**Teoría.** Dos formas de **protegerte a vos y a los demás** poniendo límites:

- **Rate limiting (cuántas operaciones por unidad de tiempo).** Del lado cliente, para no exceder la cuota de una dependencia; del lado servidor, para no dejar que un cliente te sature. El estándar en Go es **`golang.org/x/time/rate`** (token bucket):

```go
import "golang.org/x/time/rate"

// 10 operaciones por segundo, ráfagas de hasta 20
lim := rate.NewLimiter(rate.Limit(10), 20)

func (s *Servicio) Llamar(ctx context.Context) error {
	if err := lim.Wait(ctx); err != nil { // espera un token (o aborta si ctx se cancela)
		return err
	}
	return hacerLlamada(ctx)
}
```

`lim.Allow()` (no bloqueante: ¿hay token ahora?) sirve para rechazar al toque (`429`); `lim.Wait(ctx)` (bloqueante hasta tener token o cancelar) sirve para *suavizar* tu propio ritmo de salida.

- **Bulkhead (cuántas operaciones concurrentes).** Aislás recursos para que una parte lenta no consuma todo. El patrón en Go es un **semáforo** — un channel con buffer o `golang.org/x/sync/semaphore` — que limita cuántas goroutines hacen X a la vez:

```go
sem := make(chan struct{}, 5) // máximo 5 llamadas concurrentes a esta dependencia

func (s *Servicio) Llamar(ctx context.Context) error {
	select {
	case sem <- struct{}{}: // tomar un "permiso"
		defer func() { <-sem }()
	case <-ctx.Done():
		return ctx.Err()
	}
	return hacerLlamada(ctx)
}
```

El nombre viene de los **mamparos** de un barco: si un compartimento se inunda, los mamparos evitan que se hunda todo. Acá, si una dependencia se pone lenta, el bulkhead limita cuántas goroutines quedan atascadas en ella — el resto del servicio sigue funcionando.

**Ejercicios 6**
6.1 🔁 ¿Qué limita un *rate limiter* y qué limita un *bulkhead*? No es lo mismo.
6.2 🧠 ¿Cuándo usás `lim.Allow()` y cuándo `lim.Wait(ctx)`?
6.3 ✍️ Implementá un bulkhead con un channel-semáforo que permita como máximo 3 ejecuciones concurrentes de una función `trabajo(ctx)`, respetando la cancelación del `ctx`.

---

## Módulo 7 — Workers de fondo: goroutines supervisadas y drenado limpio

**Teoría.** Un **worker de fondo** es una goroutine de larga vida que procesa trabajo fuera del ciclo request/response: consumir una cola, mandar emails, recalcular agregados. Tres cosas que tiene que tener bien, todas atadas al `context` y al *graceful shutdown* de [Go para backend](go-backend.md):

1. **Vía de salida por `context`.** El worker corre hasta que el `ctx` se cancela (en el shutdown). Nada de loops infinitos sin escape — eso es una goroutine leak garantizada.
2. **Drenado (drain) en el apagado.** Cuando llega `SIGTERM`, el worker debe **terminar lo que está procesando** antes de morir, no cortar a la mitad. El `main` espera con un `WaitGroup`.
3. **Supervisión.** Si el worker muere por un panic, hay que recuperarlo (Módulo 9), no perderlo en silencio.

```go
func worker(ctx context.Context, jobs <-chan Job, wg *sync.WaitGroup) {
	defer wg.Done()
	for {
		select {
		case <-ctx.Done():
			return // shutdown: salida limpia
		case job, ok := <-jobs:
			if !ok {
				return // el channel se cerró: no hay más trabajo
			}
			procesar(job) // terminamos este job aunque ya haya llegado el shutdown
		}
	}
}

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	jobs := make(chan Job)
	var wg sync.WaitGroup
	for i := 0; i < 4; i++ { // pool de 4 workers
		wg.Add(1)
		go worker(ctx, jobs, &wg)
	}

	// ... alimentar jobs ...

	<-ctx.Done()  // esperar SIGTERM
	wg.Wait()     // drenar: esperar a que los workers terminen lo en curso
}
```

> 💡 **`errgroup` para workers que pueden fallar:** si tus workers devuelven error y querés que el primero que falla cancele a los demás, usá `golang.org/x/sync/errgroup` con `errgroup.WithContext` (lo nombramos en [Go para backend](go-backend.md)): combina el `WaitGroup`, la propagación del error y la cancelación por `context` en una sola pieza.

**Ejercicios 7**
7.1 🔁 ¿Cuáles son las tres propiedades que un worker de fondo bien hecho debe tener?
7.2 🧠 ¿Qué diferencia hay entre "parar de aceptar trabajo nuevo" y "drenar", y por qué el `wg.Wait()` va *después* del `<-ctx.Done()`?
7.3 ✍️ Escribí el `select` del loop de un worker que sale por `ctx.Done()` o procesa de un channel `tareas`, manejando también el cierre del channel.

---

## Módulo 8 — Jobs programados y colas

**Teoría.** Dos sabores de trabajo de fondo:

- **Jobs programados (scheduled / cron):** "cada 5 minutos, limpiar sesiones vencidas". Lo simple en Go es un `time.Ticker` dentro de un worker; para expresiones cron, librerías como `robfig/cron`.

```go
func limpiador(ctx context.Context) {
	t := time.NewTicker(5 * time.Minute)
	defer t.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-t.C:
			if err := limpiarSesiones(ctx); err != nil {
				slog.Error("limpieza", "err", err) // logueá pero NO mueras: el próximo tick reintenta
			}
		}
	}
}
```

- **Colas de trabajo (job queues):** desacoplás el productor del consumidor con una cola persistente (Redis/`asynq`, una tabla en Postgres, SQS, NATS). Esto da reintentos, *backoff*, *dead-letter queue* y trabajo distribuido entre varias instancias — y es lo correcto cuando el trabajo no puede perderse. Lo cubre [Redis](redis.md) (BullMQ/asynq) y [Event-driven](event-driven.md).

Dos verdades que se arrastran de esos módulos:
- **Casi todas las colas son *at-least-once*:** un mensaje puede entregarse **más de una vez** (reintento tras un fallo). Por eso el consumidor debe ser **idempotente** (Módulo 4): procesar el mismo job dos veces no debe duplicar el efecto. En el caso de una cola, la *idempotency key* suele ser el **ID del job** o un **hash del payload** (no hay un "cliente" que la genere como en una API).
- **Cron ingenuo en multi-instancia se ejecuta N veces:** si corrés 3 réplicas, cada una dispara el ticker → la limpieza corre 3 veces. Necesitás un **lock distribuido** (Redis `SET NX`, advisory lock de Postgres) o un scheduler que garantice una sola ejecución.

**Ejercicios 8**
8.1 🔁 ¿Cuándo usás un `time.Ticker` simple y cuándo una cola de trabajo persistente?
8.2 🧠 ¿Por qué un consumidor de una cola *at-least-once* debe ser idempotente?
8.3 🧠 Corrés un cron con `time.Ticker` en 3 réplicas y la tarea se ejecuta 3 veces. ¿Por qué, y cómo lo arreglás?

---

## Módulo 9 — Recuperación de panics en workers

**Teoría.** Recordá de [Go para backend](go-backend.md): **un `panic` no recuperado en cualquier goroutine tumba TODO el proceso.** En un worker de fondo esto es letal: un job con un dato inesperado paniquea, y se te cae el servidor entero (y todos los demás workers con él).

La defensa: un **`recover` por unidad de trabajo**, de modo que un job envenenado mate *ese job*, no el proceso.

```go
func procesarSeguro(job Job) {
	defer func() {
		if r := recover(); r != nil {
			slog.Error("panic en worker", "job", job.ID, "panic", r, "stack", string(debug.Stack()))
			// opcional: mandar el job a una dead-letter queue para inspección
		}
	}()
	procesar(job) // si paniquea, lo atrapamos arriba y el worker sigue vivo
}
```

- **Dónde poner el `recover`:** envolviendo **cada job**, no el loop entero. Si lo ponés en el loop, un panic igual sale del `for` y mata al worker; envolviendo el job, el worker procesa el siguiente.
- **Logueá el stack** (`debug.Stack()`) para poder diagnosticar; un panic silencioso es lo peor.
- **No lo uses para control de flujo.** `recover` es la red de seguridad para lo *inesperado*; los errores esperables siguen siendo `error` que devolvés y manejás (Módulo 3 de Go para backend). Un worker que paniquea seguido esconde un bug que hay que arreglar, no tapar.

> 💡 En un servidor **HTTP o gRPC**, el framework suele traer un *middleware/interceptor de recovery* (o lo agregás vos, como vimos en [gRPC](go-grpc.md)). En **workers propios** no hay framework: el `recover` es tu responsabilidad.

**Ejercicios 9**
9.1 🔁 ¿Qué le pasa a tu servicio si una goroutine worker hace `panic` y nadie la recupera?
9.2 🧠 ¿Por qué el `recover` va envolviendo cada job y no el loop completo del worker?
9.3 ✍️ Escribí una función `procesarSeguro(job)` que ejecute `procesar(job)` recuperando cualquier panic, logueando el stack con `slog` y dejando al worker vivo.

---

## Módulo 10 — El criterio: componer, no reinventar

**Teoría.** La resiliencia no es una técnica, es **una combinación** aplicada con criterio. El stack típico para una llamada a una dependencia crítica, de adentro hacia afuera:

1. **Timeout** (`context`) en cada llamada — siempre.
2. **Reintento** con backoff + jitter — solo para errores transitorios y operaciones idempotentes.
3. **Circuit breaker** por dependencia — para cuando "transitorio" pasó a "está caído".
4. **Rate limit / bulkhead** — para no saturar a la dependencia ni a vos.
5. **Degradación** (cache, default, respuesta parcial) cuando todo lo anterior falla y el dominio lo permite.

El criterio que importa:
- **No reinventes la rueda.** Timeout y `context` son stdlib; para reintentos, rate limit, circuit breaker y colas hay librerías probadas (`sony/gobreaker`, `golang.org/x/time/rate`, `asynq`). Escribir tu propio circuit breaker es un clásico error de "lo hago en una tarde" que termina con bugs sutiles.
- **No apliques todo a todo.** Una llamada a una cache local no necesita circuit breaker; una al servicio de pagos sí. Resiliencia proporcional al riesgo y al costo del fallo.
- **Hacelo observable.** Reintentos, circuitos abiertos y rechazos por rate limit tienen que ser **métricas** (ver [Observabilidad](observabilidad.md)): un circuit breaker que se abre seguido es una alerta, no un detalle. Si no lo medís, no sabés que tu sistema está degradado.
- **Probá los caminos de falla.** El happy path es fácil; lo que rompe en producción es el camino de error. Testeá timeouts, reintentos y degradación inyectando fallos (ver [Concurrencia: laboratorio](concurrencia-practica.md) y [Testing](testing.md)).

> 🎯 **Dos patrones avanzados para tener en el radar** (aparecen en entrevistas de resiliencia senior): **load shedding** — el complemento *server-side* del rate limit: bajo presión extrema, **rechazar** proactivamente requests de baja prioridad (o las más nuevas) para proteger las que ya están en curso, en vez de degradarte para todos. Y **hedged requests** — si la primera copia de una request tarda más que el p95, mandar una **segunda** en paralelo y quedarte con la que responda antes; baja la latencia de cola a costa de más carga (úsalo solo en operaciones idempotentes y con cuidado).

Esto cierra el arco de Go para backend: del [servicio base](go-backend.md) → comunicación con [gRPC](go-grpc.md) → medición con [profiling](go-performance.md) → y acá, **sobrevivir cuando las piezas fallan**, que es lo que de verdad evalúan a nivel senior.

**Ejercicios 10**
10.1 🧠 Ordená las cinco capas de resiliencia para una llamada a un servicio de pagos, e indicá qué hace cada una.
10.2 🧠 ¿Por qué "escribo mi propio circuit breaker" suele ser mala idea, y qué hacés en su lugar?
10.3 🧠 ¿Por qué la resiliencia tiene que ser observable, y qué métrica te diría que tu sistema está degradado aunque "responda"?

---

## Soluciones

### Módulo 1
```
1.1 Es cuando una parte del sistema falla (lenta/caída/intermitente) mientras el resto anda.
    Es la norma en distribuido porque hay muchas piezas independientes (DB, cache, otros
    servicios, red) y la probabilidad de que TODAS estén sanas todo el tiempo es baja: siempre
    hay algo degradado.
1.2 Una dependencia lenta hace que tus llamadas esperen; sin timeout, esas goroutines/requests
    se acumulan y agotan tus recursos (conexiones, memoria, goroutines), hasta que tu servicio
    deja de responder: caés arrastrado. El fenómeno es la falla en cascada.
1.3 Porque si la dependencia está sobrecargada, el reintento inmediato le suma MÁS carga justo
    cuando menos puede manejarla, profundizando la sobrecarga en vez de darle aire: es el retry
    storm, que amplifica el problema.
```

### Módulo 2
```
2.1 Porque el ctx con deadline se pasa a cada llamada de la cadena: todas comparten el mismo
    presupuesto de tiempo y, si vence (o el cliente cancela), TODAS se abortan. Ganás que no
    quede trabajo huérfano corriendo para una request que ya nadie espera, y cortás la cascada.
2.2 El http.Client por defecto NO tiene Timeout: una llamada a un server colgado espera para
    siempre. Lo evitás seteando client.Timeout, o (mejor) pasando un ctx con deadline a cada
    request con http.NewRequestWithContext.
```
```go
// 2.3
func obtener(ctx context.Context, url string) ([]byte, error) {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, fmt.Errorf("armar request: %w", err)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("GET %s: %w", url, err) // incluye DeadlineExceeded si venció
	}
	defer resp.Body.Close()
	return io.ReadAll(resp.Body)
}
```

### Módulo 3
```
3.1 El thundering herd: sin jitter, muchos clientes que fallaron al mismo tiempo reintentan en
    los MISMOS instantes (100ms, 200ms...), creando picos coordinados que vuelven a tumbar a la
    dependencia. El jitter agrega aleatoriedad y desparrama los reintentos en el tiempo.
3.2 Porque un 400 es un error determinístico (la request está mal): reintentarla va a fallar
    igual, gastando recursos. Conviene reintentar solo errores transitorios: timeouts, 503
    Unavailable, 429 Too Many Requests, conexiones cortadas — cosas que pueden andar al
    reintentar.
```
```go
// 3.3 (dentro del loop, antes de calcular el backoff propio)
espera := min(base*(1<<i), 30*time.Second)          // backoff con cap
espera += time.Duration(rand.Int63n(int64(espera))) // jitter
if d, ok := retryAfter(err); ok {
	espera = d // el servidor manda: respetamos Retry-After en vez de nuestro backoff
}
select {
case <-time.After(espera):
case <-ctx.Done():
	return ctx.Err()
}
```

### Módulo 4
```
4.1 Que ejecutarla N veces deja el mismo estado que ejecutarla una sola. Idempotente: GET,
    PUT (reemplazo total), DELETE de un id. No idempotente: POST "crear pedido" o "cobrar"
    (cada ejecución crea/cobra de nuevo).
4.2 Porque el timeout solo dice que no llegó la RESPUESTA, no que no se ejecutó: el cobro pudo
    completarse y perderse solo el ack. Lo resolvés con una idempotency key: el cliente manda
    una clave única por intento, el server registra la clave y, si ya existe, devuelve el
    resultado guardado en vez de re-cobrar.
4.3 Porque dos reintentos concurrentes podrían chequear "no existe" a la vez y ambos cobrar.
    Lo garantizás con una operación atómica: una restricción UNIQUE sobre la idempotency key en
    Postgres (insert que falla si ya está) o SET NX en Redis, de modo que solo uno gane.
```

### Módulo 5
```
5.1 Closed: pasa todo y cuenta fallos (normal). Open: superado el umbral, rechaza las llamadas
    de inmediato sin intentar, por un tiempo (acá degradás/fallás rápido). Half-open: tras ese
    tiempo, deja pasar una llamada de prueba; si anda vuelve a closed, si falla vuelve a open.
5.2 El reintento sigue PEGÁNDOLE a una dependencia caída (gastando tus recursos e impidiéndole
    recuperarse). El circuit breaker DEJA de llamar cuando detecta que está caída: falla rápido,
    libera tus recursos y le da aire a la dependencia para recuperarse.
5.3 Porque los fallos de una dependencia no deben afectar las llamadas a otras: si pagos está
    caído y tenés un breaker global, abrirías el circuito también hacia la DB (que está sana),
    rompiendo cosas que andaban. Un breaker por dependencia aísla el fallo.
```

### Módulo 6
```
6.1 El rate limiter limita la TASA: cuántas operaciones por unidad de tiempo (ej. 10/seg). El
    bulkhead limita la CONCURRENCIA: cuántas operaciones simultáneas (ej. 5 a la vez). Podés
    tener pocas concurrentes pero muchas por segundo, o viceversa: son ejes distintos.
6.2 Allow() (no bloqueante) cuando querés rechazar al instante si no hay token —ej. responder
    429 en un endpoint. Wait(ctx) (bloqueante hasta tener token o cancelar) cuando querés
    SUAVIZAR tu propio ritmo de salida hacia una dependencia, esperando en vez de rechazar.
```
```go
// 6.3
func correrConLimite(ctx context.Context, tareas []func(context.Context) error) {
	sem := make(chan struct{}, 3) // máximo 3 concurrentes
	var wg sync.WaitGroup
	for _, t := range tareas {
		select {
		case sem <- struct{}{}: // tomar permiso
		case <-ctx.Done():
			return
		}
		wg.Add(1)
		go func(t func(context.Context) error) {
			defer wg.Done()
			defer func() { <-sem }() // liberar permiso
			_ = t(ctx)
		}(t)
	}
	wg.Wait()
}
```

### Módulo 7
```
7.1 (1) Una vía de salida por context (termina cuando el ctx se cancela, sin loops infinitos);
    (2) drenado en el apagado (termina el trabajo en curso antes de morir); (3) supervisión/
    recuperación de panics para no perderse en silencio.
7.2 "Parar de aceptar" es dejar de tomar trabajo nuevo (cancelar el ctx / cerrar el channel de
    entrada); "drenar" es terminar lo que ya está en proceso. wg.Wait() va DESPUÉS de
    <-ctx.Done() porque primero señalás el apagado (ctx cancelado) y recién entonces esperás a
    que los workers terminen lo pendiente; si no esperaras, cortarías jobs a la mitad.
```
```go
// 7.3
for {
	select {
	case <-ctx.Done():
		return
	case tarea, ok := <-tareas:
		if !ok {
			return // channel cerrado: no hay más trabajo
		}
		procesar(tarea)
	}
}
```

### Módulo 8
```
8.1 time.Ticker simple cuando la tarea es periódica, in-process y se puede perder/saltear un
    tick sin drama (limpiezas, refrescos de cache). Cola persistente cuando el trabajo NO puede
    perderse, necesita reintentos/DLQ, o se reparte entre varias instancias (procesar pagos,
    enviar emails críticos).
8.2 Porque at-least-once significa que el mismo mensaje puede entregarse más de una vez (tras un
    fallo/reintento). Si el consumidor no es idempotente, procesarlo dos veces duplica el efecto
    (doble cobro, doble email). Idempotente = el reproceso no cambia el resultado.
8.3 Porque cada réplica corre su propio ticker en proceso, así que las 3 disparan la tarea de
    forma independiente. Lo arreglás con un lock distribuido (Redis SET NX con TTL, advisory
    lock de Postgres) para que solo una réplica ejecute cada disparo, o usando un scheduler
    central / cola que garantice ejecución única.
```

### Módulo 9
```
9.1 Tumba TODO el proceso: un panic no recuperado se propaga hasta el tope de la goroutine y
    termina el programa entero, matando ese worker y todos los demás (y el servidor). Un solo
    job envenenado te tira el servicio.
9.2 Porque si el recover está en el loop, el panic igual rompe la iteración y sale del for →
    el worker muere. Envolviendo cada job (con su propio defer/recover), el panic se contiene en
    ESE job: se loguea, y el worker sigue con el siguiente.
```
```go
// 9.3
func procesarSeguro(job Job) {
	defer func() {
		if r := recover(); r != nil {
			slog.Error("panic en worker", "job", job.ID, "panic", r, "stack", string(debug.Stack()))
		}
	}()
	procesar(job)
}
```

### Módulo 10
```
10.1 (1) Timeout por llamada (context): no esperar para siempre. (2) Reintento backoff+jitter:
     reabsorber fallos transitorios (si es idempotente). (3) Circuit breaker por dependencia:
     dejar de llamar cuando pagos está caído y fallar rápido. (4) Rate limit/bulkhead: no
     saturar pagos ni agotar tus recursos. (5) Degradación: si todo falla, respuesta de
     respaldo (encolar el cobro, avisar "procesando") si el dominio lo permite.
10.2 Porque un circuit breaker correcto tiene sutilezas (estados, ventanas de conteo,
     concurrencia, half-open) fáciles de equivocar, y un bug ahí falla justo cuando más lo
     necesitás. En su lugar usás una librería probada (sony/gobreaker) y te concentrás en
     configurarla bien.
10.3 Porque un sistema puede "responder" mientras está degradado (sirviendo de cache, con
     circuitos abiertos, rechazando por rate limit) y no te enterás sin métricas. Señales:
     tasa de reintentos, circuitos en estado open, rechazos 429, y latencia p99 — cualquiera
     subiendo te dice que estás degradado aunque no haya errores 500 visibles.
```

---

## Siguientes pasos

Con este módulo cerrás el arco de **Go para backend de producción**: ya no solo escribís el servicio, sino que lo diseñás para **sobrevivir cuando las dependencias fallan**. El recorrido natural desde acá: **(1)** llevá la teoría a la práctica inyectando fallos en [Concurrencia: laboratorio](concurrencia-practica.md) y testeando los caminos de error con [Testing](testing.md); **(2)** conectá las colas y la idempotencia con [Redis](redis.md) y [Event-driven](event-driven.md), donde el trabajo asíncrono y el at-least-once se tratan a fondo; **(3)** hacé la resiliencia **observable** con [Observabilidad](observabilidad.md): reintentos, circuitos abiertos y rechazos como métricas y alertas. La constante, y lo que se evalúa a nivel senior: **el happy path lo hace cualquiera; el oficio está en el camino de falla.**
