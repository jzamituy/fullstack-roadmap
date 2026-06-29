# Go backend: laboratorio práctico

**Un servicio de pedidos de punta a punta · método "código que falla → fix" · 2026**

> Cómo usar esta guía: esto **no es teoría, es taller**. Vas a construir **un solo servicio** (un "Pedidos service") en cinco *builds* que se encadenan, y en cada uno arrancás con **código que falla o le falta algo**, diagnosticás *por qué*, y aplicás el fix. Es la contraparte práctica de los módulos [Go para backend](go-backend.md), [gRPC](go-grpc.md), [Performance](go-performance.md) y [Resiliencia](go-resiliencia.md): acá los juntás en algo que corre. Hacé cada build **en tu editor**, no leas y sigas — el aprendizaje está en ver el fallo y arreglarlo.

**Lo que asumimos.** Haber hecho (o tener a mano) los cuatro módulos de Go de arriba. Vas a necesitar instalado: **Go 1.23+**, **Docker** (para Postgres), **`buf`** (codegen de protobuf), y opcionalmente **`grpcurl`** (probar RPCs) y **`graphviz`** (`pprof -web`). Si te falta alguno, cada build te dice dónde se usa.

> ⚠️ **Datos volátiles.** APIs que cambiaron y se usan acá: `grpc.NewClient` (no `Dial`), `gobreaker/v2` con genéricos, `for b.Loop()` (Go 1.24), `math/rand/v2`. Verificá contra los repos oficiales si algo no compila en tu versión.

**El proyecto.** Un servicio de **Pedidos** que: recibe pedidos (gRPC), los persiste de forma **idempotente** (Postgres), **cobra** llamando a un servicio externo de Pagos **poco confiable** (resiliencia), procesa trabajo de fondo con un **pool de workers**, y al final lo **perfilamos** para matar un hotspot. Layout:

```
pedidos/
  cmd/api/main.go        # arma todo y arranca
  internal/pedido/       # dominio + servicio
  internal/grpcserver/   # adaptador gRPC
  internal/pagos/        # cliente resiliente
  internal/store/        # repositorio Postgres
  gen/pedidos/v1/        # código generado del .proto
  proto/pedidos/v1/pedidos.proto
```

**Los cinco builds**
1. Esqueleto gRPC con *graceful shutdown*
2. Capa de datos **idempotente** (la carrera que duplica pedidos)
3. Llamada **resiliente** a Pagos (timeout + reintento + circuit breaker)
4. Workers de fondo: matar un **goroutine leak** y un **data race**
5. Profiling: encontrar y matar el **hotspot de asignaciones**

Cada build cierra con **🚩 la red flag** (qué mirar en un PR/entrevista) y **✅ cómo verificar**. Al final, **Retos** para extender el lab, con soluciones agrupadas.

---

## Build 1 — Esqueleto gRPC con *graceful shutdown*

**Objetivo.** Levantar el servicio de Pedidos por gRPC y que **no corte RPCs en vuelo** cuando llega el `SIGTERM` de un deploy.

El contrato (`proto/pedidos/v1/pedidos.proto`):

```protobuf
syntax = "proto3";
package pedidos.v1;
option go_package = "pedidos/gen/pedidos/v1;pedidosv1";

message Pedido {
  string id = 1;
  string estado = 2;
  int64 total_centavos = 3;
}
message CrearPedidoRequest {
  string idempotency_key = 1;
  repeated string producto_ids = 2;
}

service PedidoService {
  rpc CrearPedido(CrearPedidoRequest) returns (Pedido);
}
```

Generás con `buf generate` (ver módulo [gRPC](go-grpc.md)). Ahora el `main` — **acá está el código que falla**:

```go
// cmd/api/main.go — VERSIÓN CON BUG
func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterPedidoServiceServer(s, grpcserver.Nuevo())

	slog.Info("gRPC en :50051")
	if err := s.Serve(lis); err != nil { // ⛔ bloquea para siempre; SIGTERM mata el proceso de una
		log.Fatalf("serve: %v", err)
	}
}
```

**🔎 Diagnóstico.** `s.Serve` bloquea hasta que el proceso muere. Cuando Kubernetes manda `SIGTERM` en un deploy, el proceso se termina **de golpe**: cualquier `CrearPedido` a mitad de camino se corta, el cliente ve una conexión rota, y podés dejar un pedido a medio escribir. Falta **escuchar la señal** y hacer un apagado ordenado.

**El fix:** serví en una goroutine, esperá la señal con `signal.NotifyContext`, y usá **`GracefulStop`** (deja de aceptar conexiones nuevas y **espera** a que terminen los RPCs en curso):

```go
// cmd/api/main.go — FIX
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterPedidoServiceServer(s, grpcserver.Nuevo())

	go func() {
		// Este err es un fallo de arranque (listener roto, puerto ocupado): hay que verlo.
		// En el apagado normal, GracefulStop hace que Serve devuelva nil, así que esta rama
		// NO se dispara en el shutdown — es solo para fallos reales del servidor.
		if err := s.Serve(lis); err != nil {
			slog.Error("serve", "err", err)
		}
	}()
	slog.Info("gRPC escuchando", "addr", ":50051")

	<-ctx.Done() // espera SIGTERM / Ctrl-C
	slog.Info("apagando, drenando RPCs en curso…")
	s.GracefulStop() // <- la pieza clave
}
```

Y el servidor (recordá embeber `Unimplemented...` para *forward compatibility*):

```go
// internal/grpcserver/server.go
type Servidor struct {
	pb.UnimplementedPedidoServiceServer
	svc *pedido.Servicio
}

func Nuevo() *Servidor { return &Servidor{svc: pedido.NuevoServicio()} }

// ⚠️ Ojo: este Nuevo() se fabrica el servicio adentro solo para que el Build 1 corra solo.
// En el Build 2 cambia a recibir el servicio ya construido por parámetro (DI por constructor).
// No lo tomes como definitivo.

func (s *Servidor) CrearPedido(ctx context.Context, req *pb.CrearPedidoRequest) (*pb.Pedido, error) {
	if len(req.GetProductoIds()) == 0 {
		return nil, status.Error(codes.InvalidArgument, "el pedido no tiene productos")
	}
	p, err := s.svc.Crear(ctx, req.GetIdempotencyKey(), req.GetProductoIds())
	if err != nil {
		return nil, status.Error(codes.Internal, "no se pudo crear el pedido")
	}
	return &pb.Pedido{Id: p.ID, Estado: p.Estado, TotalCentavos: p.TotalCentavos}, nil
}
```

**🚩 Red flag.** Un servicio de red sin manejo de `SIGTERM` es deuda garantizada: en cada deploy corta requests. Si ves `s.Serve(...)` como última línea del `main` sin una señal escuchada, falta el graceful shutdown.

**✅ Verificá.** Levantá el server y, con `grpcurl`, hacé una llamada:
```bash
grpcurl -plaintext -d '{"idempotency_key":"k1","producto_ids":["p1"]}' \
  localhost:50051 pedidos.v1.PedidoService/CrearPedido
```
Después mandale `Ctrl-C` mientras hay una llamada lenta en curso: debe **terminarla** antes de cerrar, no cortarla.

---

## Build 2 — Capa de datos idempotente (la carrera que duplica pedidos)

**Objetivo.** Persistir el pedido en Postgres de forma que **reintentar `CrearPedido` con la misma `idempotency_key` no cree dos pedidos** — ni siquiera con dos requests concurrentes.

Esquema (aplicalo con `golang-migrate`/`goose`):

```sql
CREATE TABLE pedidos (
  id             TEXT PRIMARY KEY,
  idem_key       TEXT NOT NULL UNIQUE,   -- la clave de la idempotencia
  estado         TEXT NOT NULL,
  total_centavos BIGINT NOT NULL
);
```

**El código que falla** — el patrón "chequear y después insertar":

```go
// VERSIÓN CON BUG — race window
func (r *Repo) Crear(ctx context.Context, idemKey string, p *Pedido) (*Pedido, error) {
	if existente, ok, err := r.porIdemKey(ctx, idemKey); err != nil {
		return nil, err
	} else if ok {
		return existente, nil // ya existía
	}
	// ⛔ ventana de carrera: dos requests llegan acá a la vez con la misma idemKey
	if err := r.insertar(ctx, idemKey, p); err != nil {
		return nil, err
	}
	return p, nil
}
```

**🔎 Diagnóstico.** Entre el `porIdemKey` (no existe) y el `insertar`, **dos requests concurrentes** con la misma `idemKey` pasan ambas el chequeo y ambas insertan → o se duplica el pedido, o la segunda explota con violación de `UNIQUE`. El `if` en Go **no es atómico** respecto a la base. La atomicidad la tiene que dar **Postgres**, no tu código.

**El fix:** un único `INSERT ... ON CONFLICT DO NOTHING` y, si no insertó (la clave ya estaba), **leer y devolver el pedido previo** (*read-after-conflict*):

```go
// FIX — atómico, sin ventana de carrera
func (r *Repo) Crear(ctx context.Context, idemKey string, p *Pedido) (*Pedido, error) {
	const q = `
		INSERT INTO pedidos (id, idem_key, estado, total_centavos)
		VALUES ($1, $2, $3, $4)
		ON CONFLICT (idem_key) DO NOTHING`

	res, err := r.db.ExecContext(ctx, q, p.ID, idemKey, p.Estado, p.TotalCentavos)
	if err != nil {
		return nil, fmt.Errorf("insert pedido: %w", err)
	}
	if n, _ := res.RowsAffected(); n == 0 {
		// no insertó → la idemKey ya existía: devolvemos el pedido que ya estaba
		existente, _, err := r.porIdemKey(ctx, idemKey)
		return existente, err
	}
	return p, nil
}
```

> 💡 **El cableado entre builds.** Ahora `pedido.NuevoServicio` recibe el repo, así que el `main` arma la cadena de dependencias **por constructor** (la DI manual del módulo [Go para backend](go-backend.md)): `db → store.Nuevo(db) → pedido.NuevoServicio(repo) → grpcserver.Nuevo(svc)`. El `grpcserver.Nuevo()` del Build 1 pasa a recibir el servicio ya construido, en vez de fabricárselo adentro.

**🚩 Red flag.** Cualquier "buscar-y-si-no-está-crear" en una operación concurrente es una *race condition* esperando a pasar. La idempotencia real se apoya en una **restricción de la base** (`UNIQUE` + `ON CONFLICT`), no en un `if` de Go. Esto conecta con [PostgreSQL](postgresql.md) y el módulo de [Resiliencia](go-resiliencia.md) (idempotency key).

**✅ Verificá.** Disparale 50 `CrearPedido` concurrentes con la **misma** `idemKey`, comprobá que la tabla tiene **una sola** fila **y** que las 50 respuestas devuelven el **mismo `id`** (esto último prueba que el camino de conflicto realmente *lee y devuelve* el pedido previo a los 49 perdedores, no solo que no duplica):
```bash
# en un test Go, o con un script que lance 50 goroutines a la vez
psql -c "SELECT count(*) FROM pedidos WHERE idem_key = 'k1';"  # debe dar 1
# en el test Go: junta los 50 ids devueltos y verificá que todos son iguales
```

---

## Build 3 — Llamada resiliente a Pagos (timeout + reintento + circuit breaker)

**Objetivo.** Cobrar llamando al servicio de **Pagos**, que es **lento e intermitente**. No podemos colgarnos esperándolo, ni martillarlo cuando falla, ni cobrar dos veces.

**El código que falla** — la llamada ingenua:

```go
// VERSIÓN CON BUG
func (c *PagosCliente) Cobrar(ctx context.Context, idemKey string, monto int64) error {
	return c.http.cobrar(ctx, idemKey, monto) // ⛔ sin timeout propio, sin reintento, sin breaker
}
```

**🔎 Diagnóstico.** Tres problemas: (1) si Pagos se cuelga, esta llamada espera lo que herede del `ctx` (o para siempre si el `ctx` no tiene deadline) y **acumula goroutines** bajo carga; (2) un fallo **transitorio** (un `503` puntual) tira el pedido cuando un reintento lo habría salvado; (3) si Pagos sigue caído, seguimos pegándole en vano. Necesitamos las **tres capas** del módulo de [Resiliencia](go-resiliencia.md): timeout por llamada → reintento con backoff+jitter (solo transitorios, y **seguro** porque mandamos la `idemKey`) → circuit breaker.

**El fix:**

```go
// internal/pagos/cliente.go — FIX
import (
	"github.com/sony/gobreaker/v2"
	"math/rand/v2"
)

type PagosCliente struct {
	http httpPagos
	cb   *gobreaker.CircuitBreaker[any]
}

func NuevoCliente(h httpPagos) *PagosCliente {
	cb := gobreaker.NewCircuitBreaker[any](gobreaker.Settings{
		Name:    "pagos",
		Timeout: 30 * time.Second, // cuánto queda abierto antes de probar
		ReadyToTrip: func(c gobreaker.Counts) bool {
			// abrir por PROPORCIÓN de fallos, no por consecutivos (más robusto bajo tráfico)
			return c.Requests >= 10 && float64(c.TotalFailures)/float64(c.Requests) > 0.6
		},
		OnStateChange: func(name string, from, to gobreaker.State) {
			// hacer VISIBLE la transición: un breaker que abre/cierra en silencio es
			// un incidente que no ves. Esto es el puente con Observabilidad.
			slog.Warn("circuit breaker cambió de estado", "name", name, "de", from, "a", to)
		},
	})
	return &PagosCliente{http: h, cb: cb}
}

func (c *PagosCliente) Cobrar(ctx context.Context, idemKey string, monto int64) error {
	_, err := c.cb.Execute(func() (any, error) {
		return nil, c.cobrarConReintentos(ctx, idemKey, monto)
	})
	return err // puede ser el error real o gobreaker.ErrOpenState (falló rápido)
}

func (c *PagosCliente) cobrarConReintentos(ctx context.Context, idemKey string, monto int64) error {
	const maxIntentos = 3
	base := 100 * time.Millisecond
	var err error
	for i := 0; i < maxIntentos; i++ {
		llamada, cancel := context.WithTimeout(ctx, 2*time.Second) // timeout POR intento
		err = c.http.cobrar(llamada, idemKey, monto)               // idemKey ⇒ reintentar es seguro
		cancel()

		if err == nil {
			return nil
		}
		if !esTransitorio(err) {
			return err // 4xx/validación: NO reintentar
		}
		espera := min(base*(1<<i), 5*time.Second)        // backoff con tope
		espera += time.Duration(rand.Int64N(int64(espera))) // jitter (math/rand/v2)
		select {
		case <-time.After(espera):
		case <-ctx.Done():
			return ctx.Err()
		}
	}
	return err
}
```

> 💡 **Dos sutilezas del orden.** (1) El circuit breaker **envuelve** los reintentos, así que cada `Cobrar` cuenta como **un** request ante el breaker (mide *operaciones*, no intentos individuales) — que es justo lo que querés: el breaker reacciona al resultado neto de la operación, no a cada reintento. (2) Usamos `CircuitBreaker[any]` porque el cobro no devuelve payload; si la operación retornara un dato (`*Recibo`), tiparías `[*Recibo]` y `Execute` te lo devuelve **ya tipado** — ahí brillan los genéricos de `gobreaker/v2`. (El jitter asume `espera > 0`, garantizado acá porque el mínimo es `base`=100 ms.)

**🚩 Red flag.** Un reintento sobre una operación **no idempotente** duplica efectos: el reintento de "cobrar" es seguro **solo** porque mandamos la `idemKey` y el servicio de Pagos la respeta. Si ves un retry-loop sobre un cobro/creación **sin** idempotency key, es un bug de doble cobro esperando.

**✅ Verificá.** Levantá un Pagos falso que falle el 50% de las veces con `503` y el resto OK; corré 200 cobros y comprobá que casi todos terminan bien (los reintentos absorben los `503`) y que **ningún** cobro se duplica (la `idemKey` lo garantiza). Después hacé que Pagos falle el 100%: el circuit breaker debe **abrirse** y empezar a fallar rápido (sin esperar el timeout).

---

## Build 4 — Workers de fondo: matar un goroutine leak y un data race

**Objetivo.** Procesar los pedidos cobrados en un **pool de workers** (mandar email de confirmación, actualizar métricas), con apagado limpio. El código de partida tiene **dos bugs clásicos** a la vez.

**El código que falla:**

```go
// VERSIÓN CON BUG
type Procesador struct {
	procesados int // ⛔ tocado por varias goroutines sin protección → data race
}

func (p *Procesador) Correr(jobs <-chan Job) {
	for i := 0; i < 4; i++ {
		go func() {
			for job := range jobs { // ⛔ si jobs nunca se cierra, estas goroutines no salen nunca → leak
				p.procesar(job)
				p.procesados++ // ⛔ escritura concurrente sin lock
			}
		}()
	}
}
```

**🔎 Diagnóstico.** Dos problemas independientes:
1. **Goroutine leak:** los workers viven mientras `for range jobs` tenga de dónde leer. Si en el shutdown nadie **cierra** `jobs` (o no hay una vía de salida por `ctx`), las 4 goroutines quedan colgadas para siempre. Por cada arranque/parada, se acumulan.
2. **Data race:** `p.procesados++` lo ejecutan las 4 goroutines a la vez sobre la misma variable. `++` no es atómico (leer-incrementar-escribir): se pierden incrementos, y `go test -race` lo grita.

**El fix:** vía de salida por `ctx` con `select`, **drenado** con `WaitGroup`, y el contador protegido (`sync.Mutex` o `atomic`):

```go
// FIX
type Procesador struct {
	mu         sync.Mutex
	procesados int
}

func (p *Procesador) incr() {
	p.mu.Lock()
	p.procesados++
	p.mu.Unlock()
}

func (p *Procesador) Correr(ctx context.Context, jobs <-chan Job) {
	var wg sync.WaitGroup
	for i := 0; i < 4; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for {
				select {
				case <-ctx.Done(): // vía de salida en el shutdown
					return
				case job, ok := <-jobs:
					if !ok { // channel cerrado: no hay más trabajo
						return
					}
					p.procesar(job)
					p.incr() // seguro: protegido por el mutex
				}
			}
		}()
	}
	wg.Wait() // espera a que cada worker termine el job EN VUELO y salga
}
```

> 💡 **Dos vías de salida, y qué drena cada una.** Los workers salen por dos caminos: `ctx.Done()` (apagado por señal) o `jobs` cerrado (no hay más trabajo). Ojo con el matiz: al cancelar el `ctx`, cada worker termina **el job que tiene en la mano** y sale — los jobs que quedaron **encolados en el channel se descartan**. `wg.Wait()` drena lo *en vuelo*, no la cola pendiente. Si querés vaciar la cola antes de cerrar, el patrón es el otro: el productor **cierra `jobs`** y los workers hacen `for job := range jobs` (sin el `case <-ctx.Done()`), así procesan todo lo pendiente y recién ahí salen.

> 💡 **El cableado con el Build 1.** El círculo del shutdown limpio se cierra así: llega `SIGTERM` → el productor **deja de encolar y cierra `jobs`** → los workers drenan y `wg.Wait()` retorna → recién entonces el `main` llama a `GracefulStop()`. Es la misma idea del Build 1, extendida al pool: nadie corta trabajo a la mitad.

> 💡 Para `procesados`, un `atomic.Int64` (con `.Add(1)`/`.Load()`) sería aún más simple que el mutex para un solo contador. El mutex se vuelve necesario apenas tengas **dos** campos que deban cambiar juntos.

**🚩 Red flag.** Toda goroutine necesita una **vía de salida garantizada** (`ctx` o channel cerrado); un `for range ch` sin quién lo cierre es un leak. Y cualquier variable tocada por más de una goroutine, con al menos una escritura, **necesita** sincronización — no hay "es solo un contador, no pasa nada".

**✅ Verificá.** Corré los tests con el **detector de carreras**: `go test -race ./...` debe quedar **limpio**. Y mirá el goroutine profile (`go tool pprof http://localhost:6060/debug/pprof/goroutine`) antes y después de un ciclo de carga: el número de goroutines debe **volver a la línea base**, no crecer.

---

## Build 5 — Profiling: encontrar y matar el hotspot de asignaciones

**Objetivo.** Un endpoint que arma un "resumen" de pedidos está lento y la RAM sube bajo carga. **Medí** (no adivines), encontrá el hotspot, arreglalo, **re-medí**.

**El código sospechoso:**

```go
// internal/pedido/resumen.go
func Resumen(items []Item) string {
	s := ""
	for _, it := range items {
		s += it.Nombre + ":" + strconv.Itoa(it.Cant) + ";" // ⛔ concatena strings en loop
	}
	return s
}
```

**Paso 1 — medí con un benchmark** (Go 1.24, `for b.Loop()`):

```go
// internal/pedido/resumen_test.go
func BenchmarkResumen(b *testing.B) {
	items := make([]Item, 1000)
	for i := range items {
		items[i] = Item{Nombre: "prod", Cant: i}
	}
	for b.Loop() {
		Resumen(items)
	}
}
```

```bash
go test -bench=BenchmarkResumen -benchmem -run=^$
# BenchmarkResumen-8   4000   280000 ns/op   ... B/op   ~3000 allocs/op   <- ~3 allocs por item
```

**🔎 Diagnóstico.** `s += ...` arma **strings nuevos en cada vuelta** (los strings son inmutables). Y no es una sola asignación por item: hay **~3** — la conversión `strconv.Itoa` crea un string, la concatenación del lado derecho (`nombre + ":" + … + ";"`) crea otro, y el `s += <rhs>` otro más. Para 1000 items eso es el `~3000 allocs/op`. Confirmás con el heap profile que el costo está en `Resumen`:
```bash
go test -bench=BenchmarkResumen -memprofile mem.out -run=^$
go tool pprof -sample_index=alloc_space mem.out   # (pprof) top → list Resumen
```

**El fix — `strings.Builder`** (un solo buffer que crece, no un string nuevo por vuelta):

```go
func Resumen(items []Item) string {
	var b strings.Builder
	b.Grow(len(items) * 16) // preasignación aproximada; para 1 sola alloc, dimensioná midiendo el tamaño real
	for _, it := range items {
		b.WriteString(it.Nombre)
		b.WriteByte(':')
		b.WriteString(strconv.Itoa(it.Cant))
		b.WriteByte(';')
	}
	return b.String()
}
```

**Paso 2 — re-medí y compará con `benchstat`** (no a ojo: confirmá que la mejora es real):
```bash
go test -bench=BenchmarkResumen -benchmem -count=10 -run=^$ > viejo.txt
# ...aplicás el fix...
go test -bench=BenchmarkResumen -benchmem -count=10 -run=^$ > nuevo.txt
benchstat viejo.txt nuevo.txt   # debe mostrar allocs/op cayendo de ~3000 a unas pocas
```

**🚩 Red flag.** "Lo optimicé porque se veía lento" sin un benchmark/profile antes y después es trabajo a ciegas. Y `s += x` dentro de un loop es el antipatrón de asignaciones #1 en Go — `strings.Builder` casi siempre. Pero **medí primero**: si `Resumen` no fuera un hotspot real, este cambio sería complejidad sin retorno (módulo [Performance](go-performance.md)).

**✅ Verificá.** `benchstat` debe reportar una caída **estadísticamente significativa** de `allocs/op` y `ns/op`. Si la diferencia es ruido (alta varianza, sin significancia), no cantes victoria.

---

## Retos (extendé el lab)

Para practicar de verdad, extendé el servicio. Las soluciones están al final — **no espíes**.

R1. 🧠 En el Build 1, ¿qué diferencia hay entre `s.GracefulStop()` y `s.Stop()` de gRPC, y cuál usás en el shutdown y por qué?
R2. ✍️ Agregá al `PedidoService` un RPC **server-streaming** `SeguirPedido(SeguirRequest) returns (stream Pedido)` que emita los cambios de estado del pedido. Escribí la firma del handler del servidor y el loop de envío respetando `stream.Context()`.
R3. 🧠 En el Build 3, el circuit breaker abre por *proporción* de fallos (`> 0.6` sobre ≥10 requests) en vez de por fallos consecutivos. ¿Por qué es más robusto bajo tráfico real?
R4. ✍️ En el Build 4, reemplazá el `sync.Mutex` + `int` del contador por un `atomic.Int64`. Mostrá el campo, el incremento y la lectura.
R5. 🧠 En el Build 5, después del fix con `strings.Builder`, ¿qué te diría que todavía tenés asignaciones de más, y qué perfil mirarías para confirmarlo?
R6. ✍️ Escribí el **test** que prueba la idempotencia del Build 2: lanzá `N` goroutines que llamen a `Crear` con la **misma** `idemKey` a la vez, y asegurá (a) que termina con **una sola** fila y (b) que **todas** las llamadas devuelven el **mismo `id`**. Pensalo para correr con `go test -race`.

---

## Soluciones

### Reto R1
```
GracefulStop() deja de aceptar conexiones/RPCs nuevos pero ESPERA a que terminen los RPCs en
curso antes de cerrar; Stop() corta todo de inmediato (cancela los RPCs en vuelo). En el
shutdown usás GracefulStop para no cortar pedidos a medio procesar en un deploy. Stop se reserva
para un apagado forzado si GracefulStop tarda demasiado (se suele combinar: GracefulStop con un
timeout y, si no terminó, Stop).
```

### Reto R2
```go
// proto: rpc SeguirPedido(SeguirRequest) returns (stream Pedido);
func (s *Servidor) SeguirPedido(req *pb.SeguirRequest, stream pb.PedidoService_SeguirPedidoServer) error {
	cambios, err := s.svc.SuscribirEstado(stream.Context(), req.GetId())
	if err != nil {
		return status.Error(codes.NotFound, "pedido no encontrado")
	}
	for {
		select {
		case <-stream.Context().Done(): // el cliente cortó
			return stream.Context().Err()
		case p, ok := <-cambios:
			if !ok {
				return nil // no hay más cambios: cerrar el stream
			}
			if err := stream.Send(&pb.Pedido{Id: p.ID, Estado: p.Estado, TotalCentavos: p.TotalCentavos}); err != nil {
				return err // el cliente se fue
			}
		}
	}
}
```

### Reto R3
```
Porque "consecutivos" nunca se cumple bajo tráfico intermitente: si Pagos falla 1 de cada 2
llamadas, casi nunca acumulás N fallos seguidos, así que el breaker no abre y seguís pegándole a
un servicio claramente degradado. La proporción de fallos sobre una ventana (>60% de ≥10
requests) captura justamente ese caso: mide salud real, no una racha. El umbral mínimo de
requests evita abrir por un par de fallos al arranque (poca muestra).
```

### Reto R4
```go
type Procesador struct {
	procesados atomic.Int64
}

func (p *Procesador) incr()      { p.procesados.Add(1) }
func (p *Procesador) Total() int64 { return p.procesados.Load() }
// En el worker: p.incr() en vez de p.mu.Lock()/procesados++/Unlock().
// atomic alcanza para UN contador independiente; si hubiera que mover dos campos juntos
// de forma consistente, volvés al mutex.
```

### Reto R5
```
Señales de allocs de más aún con Builder: (a) si no preasignaste con b.Grow y el buffer se
reasigna varias veces al crecer (cada duplicación es una alloc + copia); (b) strconv.Itoa crea
un string por item — en un hot path extremo se evita con strconv.AppendInt sobre un []byte
reutilizado. Lo confirmás con el heap profile en alloc_space (go tool pprof
-sample_index=alloc_space) y un benchmark con -benchmem: mirás si allocs/op sigue alto y
list Resumen para ver qué línea asigna. Regla: solo seguís optimizando si el profile dice que
Resumen todavía pesa; si no, pará.
```

### Reto R6
```go
func TestCrear_Idempotente_Concurrente(t *testing.T) {
	repo := nuevoRepoDeTest(t) // helper: DB limpia (testcontainers o DB de test) — ver testing.md
	const N = 50
	const idemKey = "k1"

	var wg sync.WaitGroup
	ids := make([]string, N) // cada goroutine escribe SU índice ⇒ sin data race compartido
	errs := make([]error, N)

	wg.Add(N)
	for i := 0; i < N; i++ {
		go func(i int) {
			defer wg.Done()
			// id distinto por intento: solo el ganador del ON CONFLICT debería quedar
			p := &Pedido{ID: fmt.Sprintf("p-%d", i), Estado: "creado", TotalCentavos: 100}
			got, err := repo.Crear(context.Background(), idemKey, p)
			errs[i] = err
			if got != nil {
				ids[i] = got.ID
			}
		}(i)
	}
	wg.Wait()

	// (a) ningún error y una sola fila
	for i, err := range errs {
		if err != nil {
			t.Fatalf("goroutine %d falló: %v", i, err)
		}
	}
	var n int
	if err := repo.db.QueryRow(`SELECT count(*) FROM pedidos WHERE idem_key = $1`, idemKey).Scan(&n); err != nil {
		t.Fatal(err)
	}
	if n != 1 {
		t.Fatalf("se esperaba 1 fila, hay %d (la carrera duplicó)", n)
	}
	// (b) las N respuestas devuelven el MISMO id (el camino de conflicto leyó y devolvió)
	for i, id := range ids {
		if id != ids[0] {
			t.Fatalf("respuesta %d devolvió id %q, distinto del primero %q", i, id, ids[0])
		}
	}
}
// Corré con: go test -race -run TestCrear_Idempotente_Concurrente ./internal/store
// El -race confirma además que el Repo no tiene escrituras concurrentes sin sincronizar.
```

---

## Siguientes pasos

Terminaste un servicio Go que **se parece a uno de producción**: gRPC con apagado limpio, persistencia idempotente, llamadas resilientes, workers concurrentes sin leaks ni races, y un hotspot cazado con `pprof`. Eso es, casi tal cual, un *code-challenge* de backend Go senior. El recorrido desde acá: **(1)** ponele **observabilidad** real — interceptors de OpenTelemetry en el gRPC y métricas de los reintentos/circuitos, atándolo a [Observabilidad](observabilidad.md); **(2)** containerizá con el Dockerfile multi-stage de [Go para backend](go-backend.md) y armá el pipeline de [Docker y CI/CD](docker-deploy.md); **(3)** llevá el caso a diseño de sistemas: ¿cómo escala este Pedidos service?, ¿dónde ponés una cola?, ¿qué pasa si Pagos se cae una hora? — eso es [Diseño de sistemas backend](system-design.md) y [Event-driven](event-driven.md). La constante de todo el lab: **el happy path lo hace cualquiera; lo que te hace contratable es el camino de falla, medido y arreglado.**
