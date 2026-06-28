# Go para backend (desde TS/Node)

**El lenguaje del cloud-native · concurrencia simple · stdlib que alcanza · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo está pensado para vos que venís de **TypeScript / Node**: cada concepto se explica *en contraste* con lo que ya sabés (async/await, `try/catch`, clases, npm), para que el salto sea una traducción y no un examen. Go es el lenguaje en el que están escritos Docker, Kubernetes, Prometheus y media infraestructura que ya usás — aprenderlo te abre los roles de **backend e infra** sin pedirte que tires lo que sabés de sistemas.

**Lo que asumimos.** TypeScript/JavaScript con comodidad, `async/await`, HTTP y APIs REST, qué es una base de datos relacional, y haber tocado una terminal. **No** asumimos que sepas C, punteros ni concurrencia con threads — eso lo construimos acá.

> ⚠️ **Nota sobre versiones.** Go es un lenguaje **muy estable** (la promesa de compatibilidad de Go 1.x es casi sagrada), así que el riesgo de desactualización es bajo. Aun así, este módulo usa features que llegaron en releases recientes: **routing por método en `net/http` (Go 1.22)**, **`log/slog` (1.21)**, `errors.Join` (1.20). Si usás una versión vieja, algunos snippets no compilan — verificá `go version` (asumimos **Go 1.22+**).

**Índice de módulos**
1. Por qué Go y el modelo mental viniendo de TS/Node
2. Sintaxis esencial y el sistema de tipos (sin clases, sin `null`… pero con `nil`)
3. Errores como valores: el adiós a `try/catch`
4. Concurrencia: goroutines, channels y `select`
5. `context` y sincronización: cancelar, timeouts y compartir estado
6. Servidor HTTP con la librería estándar
7. Estructura, interfaces implícitas e inyección de dependencias
8. Acceso a datos: `database/sql`, `pgx` y el patrón repositorio
9. Testing idiomático: *table-driven tests*
10. Producción: build, *graceful shutdown*, logging estructurado
11. El criterio: cuándo Go, cuándo Node, cuándo Rust
12. Camino de aprendizaje (de cero a backend en producción)

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere `go` instalado).

---

## Módulo 1 — Por qué Go y el modelo mental viniendo de TS/Node

**Teoría.** Go es un lenguaje **compilado, con tipado estático, recolección de basura (GC) y concurrencia de primera clase**, diseñado en Google para que **equipos grandes entreguen software de red rápido y mantenible**. Su rasgo de identidad: es **chico a propósito**. Hay una sola forma de hacer la mayoría de las cosas; no vas a perder una tarde eligiendo entre cinco maneras de iterar.

El cambio mental más grande viniendo de Node es **la concurrencia**:

- En Node tenés **un solo hilo** y un *event loop*: para no bloquear, todo I/O es `async` y escribís `await`. El modelo te obliga a pensar en callbacks/promesas todo el tiempo.
- En Go escribís código que **parece secuencial y bloqueante** (`resp, err := http.Get(url)` — sin `await`), pero el *runtime* corre tu función en una **goroutine** sobre un *scheduler* M:N (multiplexa miles de goroutines sobre unos pocos threads del SO). Cuando una goroutine hace I/O, el scheduler **la cede sola** y corre otra. Vos no manejás el loop: el runtime lo hace por vos.

La frase mental: **en Node la asincronía es explícita y vos la orquestás; en Go la concurrencia es del runtime y vos solo decís "esto corre en paralelo" con la palabra `go`.**

El otro cambio: **no hay clases ni herencia.** Go tiene `struct` (datos), métodos colgados de structs, **interfaces que se cumplen de forma implícita**, y **composición** en vez de herencia. Si venís de jerarquías de clases en NestJS, esto se siente raro al principio y liberador después.

```go
package main

import "fmt"

func main() {
	fmt.Println("Hola, backend en Go")
}
```

**Ejercicios 1**
1.1 🔁 ¿Por qué en Go no escribís `await` antes de una llamada HTTP, si igual no querés bloquear todo el servidor?
1.2 🧠 Un compañero dice "Go es como Node pero compilado". ¿En qué se equivoca respecto al modelo de concurrencia?
1.3 🧠 ¿Qué reemplaza a la herencia de clases en Go, y por qué eso simplifica el diseño?

---

## Módulo 2 — Sintaxis esencial y el sistema de tipos

**Teoría.** Lo mínimo para leer y escribir Go:

- **Declaración:** `var x int = 3` o, dentro de funciones, la forma corta `x := 3` (infiere el tipo). El tipo va **después** del nombre (`nombre string`), al revés que en TS.
- **`struct`** = el equivalente a una `interface`/clase de datos en TS. Los **métodos** se declaran aparte, con un *receiver*:

```go
type Usuario struct {
	ID     string
	Nombre string
	Edad   int
}

// método con receiver de puntero (puede mutar el struct)
func (u *Usuario) Cumpleaños() {
	u.Edad++
}
```

> 🧭 **Detalle que no existe en TS/Node:** `Cumpleaños` tiene *receiver de puntero* (`*Usuario`), pero podés llamarlo sobre una variable de valor — `u.Cumpleaños()` — porque `u` es **direccionable** y Go inserta el `(&u)` por vos. La distinción **valor vs puntero** define si el método muta el original (`*Usuario`) o una copia (`Usuario`); es una de las primeras cosas que confunde viniendo de JS, donde todo objeto es una referencia.

- **No existe `null`.** Cada tipo tiene un **zero value**: `0` para números, `""` para strings, `false` para bool, y `nil` para punteros, slices, maps, channels y funciones. Un `struct` recién creado tiene todos sus campos en su zero value — no hay `undefined`.
- **`nil` sí existe** y es la trampa #1: desreferenciar un puntero `nil` produce un *panic* (como el `Cannot read properties of null` de JS, pero rompe el programa si no lo manejás).
- **Colecciones:** `[]T` es un *slice* (array dinámico, como `T[]`), `map[K]V` es un diccionario (como `Record<K,V>`/`Map`). Mayúscula inicial = **exportado** (público); minúscula = privado al paquete. No hay `public`/`private`: lo decide la capitalización.

```go
u := Usuario{ID: "u1", Nombre: "Ana", Edad: 30}
ids := []string{"u1", "u2"}        // slice
porID := map[string]Usuario{}      // map vacío
porID[u.ID] = u
valor, existe := porID["u1"]        // el segundo retorno dice si la clave estaba
```

**Ejercicios 2**
2.1 🔁 ¿Cuál es el zero value de `string`, de `int`, y de un puntero `*Usuario`?
2.2 🧠 En TS escribirías `user?.name ?? "anónimo"`. ¿Por qué en Go ese patrón casi no aparece, y qué peligro reemplaza al `undefined`?
2.3 ✍️ Definí un struct `Producto` con `ID string`, `Nombre string` y `PrecioCentavos int`, y un método `Precio() float64` que devuelva el precio en unidades (centavos / 100).

---

## Módulo 3 — Errores como valores: el adiós a `try/catch`

**Teoría.** Go **no tiene excepciones** para el control de flujo normal. Una función que puede fallar **devuelve el error como último valor de retorno**, y vos lo chequeás explícitamente:

```go
func leerConfig(ruta string) (Config, error) {
	data, err := os.ReadFile(ruta)
	if err != nil {
		return Config{}, fmt.Errorf("leer config %q: %w", ruta, err) // %w "envuelve" el error
	}
	var cfg Config
	if err := json.Unmarshal(data, &cfg); err != nil {
		return Config{}, fmt.Errorf("parsear config %q: %w", ruta, err)
	}
	return cfg, nil
}
```

El patrón `if err != nil { return ..., err }` se repite **mucho**. Es verboso, sí — pero a cambio **cada punto de falla es visible** en el código; no hay un `throw` invisible tres capas abajo que explote sin que lo veas.

Claves:
- **Envolver con `%w`** preserva la cadena de errores. Después podés preguntar por el error original:
  - `errors.Is(err, ErrNoEncontrado)` — ¿este error (o algún envuelto) es ese centinela?
  - `errors.As(err, &miError)` — ¿alguno en la cadena es de este tipo? (lo extrae).
- **Errores centinela:** `var ErrNoEncontrado = errors.New("no encontrado")` para comparar con `errors.Is`.
- **`panic`/`recover`** existen, pero son para fallos **irrecuperables** (un bug, un estado imposible), **no** para errores esperables como "usuario no existe". Tratá `panic` como el `process.exit` del desastre, no como tu `throw` de todos los días.

```go
var ErrNoEncontrado = errors.New("usuario no encontrado")

func (s *Servicio) Obtener(id string) (*Usuario, error) {
	u, ok := s.cache[id]
	if !ok {
		return nil, ErrNoEncontrado
	}
	return &u, nil
}

// quien llama:
u, err := s.Obtener("u1")
if errors.Is(err, ErrNoEncontrado) {
	// responder 404
}
```

**Ejercicios 3**
3.1 🔁 ¿Qué hace el verbo `%w` en `fmt.Errorf`, y qué te permite hacer después?
3.2 🧠 ¿Por qué usar `panic` para "el usuario no existe" es un antipatrón en Go? ¿Cuál es la alternativa correcta?
3.3 ✍️ Escribí una función `dividir(a, b int) (int, error)` que devuelva un error centinela `ErrDivisionPorCero` cuando `b == 0`, y el cociente en caso contrario.

---

## Módulo 4 — Concurrencia: goroutines, channels y `select`

**Teoría.** Esta es la joya de Go. Tres piezas:

- **Goroutine:** poné `go` delante de una llamada y esa función corre **concurrentemente**. Son baratísimas (arrancan con ~2 KB de stack); podés tener cientos de miles.

```go
go enviarEmail(u)   // no esperamos; sigue la ejecución
```

- **Channel:** el caño tipado por el que las goroutines se **comunican y sincronizan**. `ch <- v` envía, `v := <-ch` recibe. Un channel **sin buffer** sincroniza: el envío bloquea hasta que alguien recibe. La filosofía de Go: *"no comuniques compartiendo memoria; compartí memoria comunicando"*.

```go
resultados := make(chan int)        // sin buffer
go func() { resultados <- calcular() }()
r := <-resultados                   // espera el resultado
```

- **`select`:** espera sobre **varios** channels a la vez (el `Promise.race` de Go, pero de primera clase). Es como elegís entre "llegó dato" / "se canceló" / "timeout".

```go
select {
case r := <-resultados:
	usar(r)
case <-time.After(2 * time.Second):
	// timeout
}
```

Para esperar a que **un grupo** de goroutines termine, usás `sync.WaitGroup`. Patrón típico de backend, el **worker pool** (procesar N trabajos con M workers):

```go
func procesar(trabajos []int) []int {
	jobs := make(chan int, len(trabajos))
	out := make(chan int, len(trabajos))
	var wg sync.WaitGroup

	for w := 0; w < 3; w++ { // 3 workers
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs { // lee hasta que cierren el channel
				out <- j * 2
			}
		}()
	}

	for _, t := range trabajos {
		jobs <- t
	}
	close(jobs)   // avisa a los workers que no hay más
	wg.Wait()     // espera a que todos terminen
	close(out)

	var res []int
	for r := range out {
		res = append(res, r)
	}
	return res
}
```

> ⚠️ **La trampa clásica:** los **data races** existen en Go (a diferencia de Rust, el compilador no los previene). Si dos goroutines tocan la misma variable y al menos una escribe, tenés un bug. Corré tus tests con `go test -race` para detectarlos.

> 💡 **El patrón de producción:** `WaitGroup` es la base para entender el fan-out, pero en código real se usa **`golang.org/x/sync/errgroup`**, que combina `WaitGroup` + **propagación del primer error** + **cancelación por `context`** del resto de las goroutines cuando una falla. Cuando varias goroutines pueden devolver error, `errgroup.Group` es lo idiomático; el `WaitGroup` crudo se queda para cuando no hay errores que coordinar.

**Ejercicios 4**
4.1 🔁 ¿Qué diferencia hay entre un channel con buffer y uno sin buffer respecto al bloqueo?
4.2 🧠 ¿Por qué `select` es el mecanismo natural para implementar un timeout sobre una operación concurrente?
4.3 ✍️ Lanzá 3 goroutines que cada una envíe su número (`0,1,2`) a un channel; recogé los 3 valores y sumalos. (Pista: channel con buffer 3 + `WaitGroup`, o leer 3 veces.)

---

## Módulo 5 — `context`: cancelar, timeouts y propagar

**Teoría.** En un backend, una request puede **cancelarse** (el cliente cortó), **expirar** (timeout), o necesitar propagar valores (request ID, usuario) por toda la cadena de llamadas. Go resuelve esto con **`context.Context`**, que por convención es **el primer parámetro** de toda función que hace I/O o puede tardar:

```go
func (r *Repo) Buscar(ctx context.Context, id string) (*Usuario, error) {
	// si ctx se cancela, la query se aborta
	return r.db.QueryRowContext(ctx, "SELECT ... WHERE id=$1", id) // ...
}
```

- **`context.WithTimeout`** deriva un contexto que se cancela solo a los N segundos. **Siempre** llamás el `cancel` con `defer` para liberar recursos.
- En un handler HTTP, `r.Context()` ya viene atado al ciclo de vida de la request: si el cliente se va, ese contexto se cancela y vos podés abortar la query.

```go
func llamarServicio(ctx context.Context, url string) ([]byte, error) {
	ctx, cancel := context.WithTimeout(ctx, 2*time.Second)
	defer cancel()

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, fmt.Errorf("armar request %s: %w", url, err)
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, fmt.Errorf("llamar %s: %w", url, err)
	}
	defer resp.Body.Close()
	return io.ReadAll(resp.Body)
}
```

**Sincronización con memoria compartida.** Cuando *sí* necesitás compartir estado (un contador, un cache en memoria), usás los primitivos de `sync`:
- `sync.Mutex` / `sync.RWMutex` — exclusión mutua (RW permite muchos lectores o un escritor).
- `sync.Once` — ejecutar algo exactamente una vez (inicialización lazy).

```go
type Contador struct {
	mu sync.Mutex
	n  int
}

func (c *Contador) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}
```

**Ejercicios 5**
5.1 🔁 ¿Por qué `context.Context` se pasa como primer parámetro y siempre se acompaña de `defer cancel()`?
5.2 🧠 Un endpoint hace una query lenta. El cliente cancela la request. ¿Cómo evita Go seguir gastando esa query, y qué tenías que haber pasado a la DB para que funcione?
5.3 ✍️ Implementá un `Contador` *thread-safe* con `sync.Mutex` y un método `Valor() int` que lea el conteo de forma segura.

---

## Módulo 6 — Servidor HTTP con la librería estándar

**Teoría.** En Go **no necesitás un framework** para empezar: `net/http` de la stdlib es production-grade. Desde **Go 1.22**, el router (`http.ServeMux`) entiende **método + patrón con variables**, así que el caso que antes pedía un framework hoy lo cubre la stdlib:

```go
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("GET /usuarios/{id}", obtenerUsuario)
	mux.HandleFunc("POST /usuarios", crearUsuario)

	log.Fatal(http.ListenAndServe(":8080", mux))
}

func obtenerUsuario(w http.ResponseWriter, r *http.Request) {
	id := r.PathValue("id") // variable de ruta (1.22+)
	u := Usuario{ID: id, Nombre: "Ana"}

	w.Header().Set("Content-Type", "application/json")
	if err := json.NewEncoder(w).Encode(u); err != nil {
		http.Error(w, "error interno", http.StatusInternalServerError)
	}
}

func crearUsuario(w http.ResponseWriter, r *http.Request) {
	var dto Usuario
	if err := json.NewDecoder(r.Body).Decode(&dto); err != nil {
		http.Error(w, "json inválido", http.StatusBadRequest)
		return
	}
	// ... guardar ...
	w.WriteHeader(http.StatusCreated)
	if err := json.NewEncoder(w).Encode(dto); err != nil {
		slog.Error("encode respuesta", "err", err) // el status ya se envió; solo logueamos
	}
}
```

> ⚠️ **Trampa del `ResponseWriter`:** el código de estado se fija con el **primer** `Write` o `WriteHeader`; después ya no lo podés cambiar. Por eso, si serializás algo que puede fallar (como en `obtenerUsuario`), el `http.Error` con `500` llega **tarde** si `Encode` ya escribió bytes. En respuestas que pueden fallar, lo robusto es **serializar a un buffer primero** y recién entonces escribir status + cuerpo.

**Middleware** en Go no es magia: es una función que **envuelve** un handler y devuelve otro. Lo encadenás vos:

```go
func conLog(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		inicio := time.Now()
		next.ServeHTTP(w, r)
		slog.Info("request", "metodo", r.Method, "ruta", r.URL.Path, "dur", time.Since(inicio))
	})
}

// uso: http.ListenAndServe(":8080", conLog(mux))
```

Cuándo sí mirar un framework (`chi`, `echo`, `gin`): si querés grupos de rutas, binding/validación automáticos o un ecosistema de middleware listo. Pero **empezá con la stdlib**: entendés qué pasa por debajo y casi siempre alcanza.

**Ejercicios 6**
6.1 🔁 ¿Qué te dio Go 1.22 en `net/http` que antes empujaba a usar un framework de routing?
6.2 🧠 ¿Por qué un middleware en Go es "solo una función que envuelve un handler"? ¿Qué ventaja tiene sobre un sistema de middleware mágico?
6.3 ✍️ Escribí un handler `GET /salud` que responda `200` con el JSON `{"status":"ok"}`.

---

## Módulo 7 — Estructura, interfaces implícitas e inyección de dependencias

**Teoría.** Acá Go se separa más de NestJS. No hay decoradores ni un contenedor de DI: **inyectás dependencias a mano, por el constructor**. Y las **interfaces se cumplen solas**: un tipo implementa una interface con solo tener sus métodos — no escribís `implements`.

```go
// La interface define QUÉ necesito, no quién lo provee.
type UsuarioRepo interface {
	Buscar(ctx context.Context, id string) (*Usuario, error)
	Guardar(ctx context.Context, u *Usuario) error
}

// El servicio depende de la interface, no de una implementación concreta.
type Servicio struct {
	repo UsuarioRepo
}

func NuevoServicio(repo UsuarioRepo) *Servicio {
	return &Servicio{repo: repo}
}

// PostgresRepo "implementa" UsuarioRepo solo por tener los métodos. Sin "implements".
type PostgresRepo struct{ db *sql.DB }

func (r *PostgresRepo) Buscar(ctx context.Context, id string) (*Usuario, error) { /* ... */ return nil, nil }
func (r *PostgresRepo) Guardar(ctx context.Context, u *Usuario) error           { /* ... */ return nil }
```

Dos máximas idiomáticas:
- **"Accept interfaces, return structs."** Tus funciones piden interfaces (flexible para testear) y devuelven tipos concretos.
- **Definí la interface en quien la consume**, no en quien la implementa. La interface `UsuarioRepo` vive en el paquete del *servicio*, no en el del Postgres. Esto invierte la dependencia (es el "D" de SOLID) sin ningún framework.

**Layout de proyecto** (convención de facto):
- `cmd/api/main.go` — el ejecutable: arma todo y arranca el servidor.
- `internal/` — código privado de tu app (Go **prohíbe** importarlo desde otros módulos). Acá van tus paquetes de dominio, servicios, repos.
- `pkg/` — código reutilizable que sí querés exponer (opcional, muchos equipos no lo usan).

**Ejercicios 7**
7.1 🔁 ¿Qué significa que en Go las interfaces son "implícitas"? ¿Qué palabra clave NO escribís?
7.2 🧠 ¿Por qué conviene definir la interface `UsuarioRepo` en el paquete del servicio y no en el del repositorio Postgres? (Pensá en testeo y en quién depende de quién.)
7.3 ✍️ Definí una interface `Notificador` con un método `Enviar(ctx context.Context, a, msg string) error`, y un struct `EmailNotificador` que la cumpla (la implementación puede ser un `return nil`).

---

## Módulo 8 — Acceso a datos: `database/sql`, `pgx` y repositorio

**Teoría.** La stdlib trae `database/sql`, una API genérica sobre la que enchufás un *driver*. Para Postgres, el driver estándar de facto es **`pgx`** (usable como driver de `database/sql` o con su propia API más rica). Claves de producción:

- **`*sql.DB` es un pool de conexiones**, no una conexión. Crealo una vez y compartilo; configurá `SetMaxOpenConns`, `SetMaxIdleConns` y `SetConnMaxLifetime` (este último es clave detrás de **PgBouncer** para no acumular conexiones muertas — ver [PostgreSQL](postgresql.md) para dimensionar el pool).
- Usá siempre las variantes **`...Context`** (`QueryRowContext`, `ExecContext`) para que la cancelación/timeout fluya hasta la DB.
- **Parámetros con `$1, $2`** (Postgres) — **nunca** concatenes strings (inyección SQL).
- `Scan` mapea columnas a variables; `sql.ErrNoRows` es el "no encontrado".

```go
func (r *PostgresRepo) Buscar(ctx context.Context, id string) (*Usuario, error) {
	row := r.db.QueryRowContext(ctx,
		`SELECT id, nombre, edad FROM usuarios WHERE id = $1`, id)

	var u Usuario
	if err := row.Scan(&u.ID, &u.Nombre, &u.Edad); err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, ErrNoEncontrado
		}
		return nil, fmt.Errorf("scan usuario %s: %w", id, err)
	}
	return &u, nil
}
```

> 💡 **Type-safety sin escribir mapeo a mano:** **`sqlc`** genera código Go tipado a partir de tu SQL (vos escribís la query, él te da la función y los structs). Es lo más parecido a la seguridad de un ORM **sin** el ORM, y muy querido en la comunidad Go. Para casos con relaciones dinámicas, `pgx` directo o un query-builder como `squirrel`.

> 💡 **Migraciones de esquema:** Go no trae una en la stdlib; el estándar de facto es **`golang-migrate`** o **`goose`** (archivos `.sql` versionados que aplicás en el deploy, no desde el código de la app). Las estrategias (expand/contract, etc.) las cubre [PostgreSQL](postgresql.md); acá solo recordá que el esquema se versiona **aparte** del binario.

**Ejercicios 8**
8.1 🔁 ¿Por qué `*sql.DB` se crea una sola vez y se comparte, en vez de abrir una conexión por request?
8.2 🧠 ¿Qué dos cosas conseguís usando `QueryRowContext(ctx, ...)` con `$1` en vez de `Query("...WHERE id="+id)`?
8.3 ✍️ Escribí la firma de un método `Guardar(ctx context.Context, u *Usuario) error` y el `ExecContext` con un `INSERT` parametrizado (no hace falta manejar el resultado más allá del error).

---

## Módulo 9 — Testing idiomático: *table-driven tests*

**Teoría.** El testing viene **en la stdlib** (`testing`): no necesitás Jest ni un runner aparte. Convenciones:
- Archivo `xxx_test.go`, función `func TestAlgo(t *testing.T)`.
- Corrés con `go test ./...`. Con `-race` activás el detector de data races. Con `-cover`, cobertura.
- El patrón idiomático estrella es el **table-driven test**: una tabla de casos + un loop con subtests (`t.Run`). Agregar un caso es agregar una fila.

```go
func TestPrecio(t *testing.T) {
	casos := []struct {
		nombre   string
		centavos int
		quiero   float64
	}{
		{"entero", 1000, 10.0},
		{"con decimales", 1599, 15.99},
		{"cero", 0, 0.0},
	}

	for _, c := range casos {
		t.Run(c.nombre, func(t *testing.T) {
			p := Producto{PrecioCentavos: c.centavos}
			if got := p.Precio(); got != c.quiero {
				t.Errorf("Precio() = %v; quiero %v", got, c.quiero)
			}
		})
	}
}
```

Para **handlers HTTP**, la stdlib trae `net/http/httptest` (un `ResponseRecorder` que captura la respuesta sin levantar un servidor real). Para testear servicios, **inyectás un repo falso** que cumpla la interface (volvé al Módulo 7) — sin mocks mágicos, una struct con los métodos que devuelven lo que el test necesita.

```go
func TestSalud(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/salud", nil)
	rec := httptest.NewRecorder()

	salud(rec, req) // tu handler

	if rec.Code != http.StatusOK {
		t.Errorf("código = %d; quiero 200", rec.Code)
	}
}
```

`testify` (asserts + mocks) es popular y válido, pero la stdlib alcanza para casi todo y es la convención de partida.

**Ejercicios 9**
9.1 🔁 ¿Qué es un *table-driven test* y por qué es el patrón por defecto en Go?
9.2 🧠 Para testear un `Servicio` que depende de `UsuarioRepo`, ¿qué le inyectás en el test y por qué no necesitás una librería de mocking?
9.3 ✍️ Escribí un table-driven test para la función `dividir` del Módulo 3, cubriendo un caso normal y el caso `b == 0` (verificá con `errors.Is` que devuelve `ErrDivisionPorCero`).

---

## Módulo 10 — Producción: build, *graceful shutdown*, logging

**Teoría.** Lo que separa un "anda en mi máquina" de un servicio de producción:

- **Build:** `go build` produce **un binario estático único** sin dependencias de runtime. Eso hace que el Docker sea trivial y minúsculo (imagen *multi-stage* → `scratch` o `distroless`, MBs en vez de cientos):

```dockerfile
FROM golang:1.22 AS build
WORKDIR /src
COPY . .
RUN CGO_ENABLED=0 go build -o /app ./cmd/api

FROM gcr.io/distroless/static-debian12
COPY --from=build /app /app
ENTRYPOINT ["/app"]
```

- **Graceful shutdown:** cuando el orquestador manda `SIGTERM` (deploy, scale-down), no podés cortar requests en vuelo. Escuchás la señal y le das al servidor un plazo para terminar lo que está sirviendo:

```go
func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, syscall.SIGTERM)
	defer stop()

	srv := &http.Server{Addr: ":8080", Handler: mux}

	go func() {
		if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			slog.Error("listen", "err", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done() // espera SIGTERM/Ctrl-C
	slog.Info("apagando…")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	if err := srv.Shutdown(shutdownCtx); err != nil {
		slog.Error("shutdown forzado", "err", err)
	}
}
```

- **Configuración por entorno (12-factor):** nada de hardcodear puerto, DSN de la DB ni secrets. Se leen de variables de entorno con `os.Getenv` (o flags con el paquete `flag`), y **se validan al arrancar** — si falta `DATABASE_URL`, el proceso debe fallar de entrada con un error claro, no a la primera request. El binario es el mismo en dev y prod; lo que cambia es el entorno.

```go
func cargarConfig() (string, error) {
	dsn := os.Getenv("DATABASE_URL")
	if dsn == "" {
		return "", errors.New("falta DATABASE_URL")
	}
	return dsn, nil
}
```

- **Logging estructurado** con **`log/slog`** (stdlib desde 1.21): logs en JSON con campos clave-valor, listos para tu stack de observabilidad. Adiós a `fmt.Println` en producción.

```go
logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)
slog.Info("servidor iniciado", "puerto", 8080, "env", "prod")
```

Esto conecta con tus módulos de [Docker y deploy](docker-deploy.md) y [Observabilidad](observabilidad.md): el binario estático + logs JSON + healthcheck es exactamente lo que esos módulos esperan.

**Ejercicios 10**
10.1 🔁 ¿Por qué el Docker de un servicio Go puede basarse en `scratch`/`distroless` y pesar pocos MB?
10.2 🧠 ¿Qué pasa con las requests en vuelo si en un deploy matás el proceso sin *graceful shutdown*, y cómo lo evita `srv.Shutdown(ctx)`?
10.3 ✍️ Mostrá las dos líneas que configuran `slog` para emitir JSON a stdout y loguear un evento `"db conectada"` con un campo `"pool": 10`.

---

## Módulo 11 — El criterio: cuándo Go, cuándo Node, cuándo Rust

**Teoría.** Go no reemplaza a Node ni a Rust: ocupa un lugar específico. El criterio honesto:

| Elegí… | Cuando… |
|---|---|
| **Go** | Microservicios y APIs, *glue* entre DB/colas/servicios, herramientas de CLI/infra, workers concurrentes. Querés **performance buena + concurrencia simple + onboarding rápido + binario para deploy trivial**. El caso backend más común. |
| **Node/TS** | Compartís tipos/código con un frontend React, el equipo ya es TS, necesitás el ecosistema npm (SDKs, integraciones), o el producto es I/O-bound con mucha lógica de negocio que cambia rápido. |
| **Rust** | Latencia **sin pausas de GC**, servicio **CPU-bound** de verdad (parsing, cripto, motores), footprint de memoria crítico, o un componente de larguísima vida donde el costo extra inicial se amortiza. |

Lo que Go te da y conviene no romantizar:
- ✅ Concurrencia I/O **simple y potente**, compilación rápida, binario único, stdlib fuerte, GC sin tuning para el 99%.
- ⚠️ **Data races posibles** (corré `-race`), genéricos llegaron tarde y son limitados, manejo de errores verboso, GC = pausas chiquitas pero existen (mal para *low-latency* extremo).

La regla mental, alineada con tu transición desde Node: **Go es el salto natural para sumar un backend compilado y concurrente sin tirar lo que sabés.** Rust queda como segunda especialización para cuando un número concreto (p95, RAM, throughput/core) lo exija. Esto extiende el criterio del módulo [Concurrencia en system design](concurrencia.md), donde ya viste código en Node/TS **y** Go lado a lado.

**Ejercicios 11**
11.1 🧠 Tenés que escribir un servicio que recibe webhooks, los valida y los encola en Redis, con picos de 10k req/s. ¿Go, Node o Rust? Justificá en una o dos dimensiones concretas.
11.2 🧠 ¿En qué caso elegir Rust **por encima** de Go para un backend está justificado, y en qué caso sería sobre-ingeniería?

---

## Módulo 12 — Camino de aprendizaje (de cero a backend en producción)

**Teoría.** No estudies Go "entero": seguí un camino orientado a backend. Cuatro fases, cada una con un **objetivo verificable** y los módulos de esta guía que la cubren. Ritmo realista viniendo de Node: **4-6 semanas** a ~8-10 h/semana.

**Fase 0 — Fundamentos del lenguaje (3-5 días).**
Hacé **[A Tour of Go](https://go.dev/tour/)** entero (es interactivo, en el browser) y leé **[Effective Go](https://go.dev/doc/effective_go)**. Objetivo: leer código Go sin trabarte con la sintaxis. Cubre los **módulos 1-3** de esta guía. Hito: reescribís en Go un par de funciones que ya tenías en TS.

**Fase 1 — Concurrencia (4-6 días).**
El diferencial de Go. Goroutines, channels, `select`, `context`, `sync`. Practicá con **[Go by Example](https://gobyexample.com/)** (goroutines, channels, worker pools, timeouts). Cubre **módulos 4-5**. Hito: implementás un *worker pool* con cancelación por `context` y lo corrés con `-race` limpio.

**Fase 2 — Un servicio HTTP de verdad (1-2 semanas).**
Acá se vuelve backend. `net/http` (1.22 routing), middleware, JSON, estructura `cmd/`+`internal/`, interfaces + DI por constructor, `database/sql`+`pgx` contra un Postgres local (Docker). Cubre **módulos 6-8**. Hito: una **API REST CRUD** sobre Postgres, con capas servicio/repo separadas por interfaces.

**Fase 3 — Calidad y producción (1 semana).**
Table-driven tests + `httptest`, `-race`, graceful shutdown, `slog` JSON, Dockerfile multi-stage. Cubre **módulos 9-10**. Hito: la misma API con tests, apagado limpio y un binario en `distroless`.

**Capstone (el proyecto que te hace contratable):** una **API de tareas** (o lo que sea de tu dominio) con: rutas CRUD en `net/http` 1.22, Postgres vía repositorio detrás de interface, validación y errores con `%w`+`errors.Is`, un endpoint que llame a un servicio externo con timeout por `context`, *table-driven tests* + un test de handler con `httptest`, *graceful shutdown*, logs `slog` en JSON, y Docker multi-stage. Eso toca **todos** los módulos y es exactamente lo que se ve en un code-challenge de backend Go.

**Recursos de cabecera** (cuando quieras profundidad, no para arrancar):
- Libro: **"Learning Go"** (Jon Bodner) — el mejor para venir de otro lenguaje.
- **"100 Go Mistakes and How to Avoid Them"** (Teiva Harsanyi) — te ahorra los errores típicos de recién llegados.
- Oficial: **[Go by Example](https://gobyexample.com/)** como referencia rápida, **[pkg.go.dev](https://pkg.go.dev/)** para la stdlib.

> ⚠️ **Trampa del que viene de otro lenguaje:** no intentes escribir Go "como si fuera TS/Java". Nada de jerarquías de clases, nada de frameworks pesados el primer día, no abuses de goroutines "porque sí". Go premia lo **simple y explícito**; pelear contra eso es la causa #1 de frustración.

**Ejercicios 12**
12.1 🧠 ¿Por qué la Fase 1 (concurrencia) va **antes** de levantar un servidor HTTP, si el servidor "ya maneja concurrencia solo"?
12.2 🧠 Mirá el capstone: nombrá tres piezas que un revisor técnico buscaría para distinguir un Go "de juguete" de uno "de producción".

---

## Soluciones

### Módulo 1
```
1.1 Porque el modelo de concurrencia es distinto: en Go el runtime corre tu función en una
    goroutine sobre un scheduler M:N, y cuando hacés I/O lo cede solo a otra goroutine. El
    código se ve bloqueante/secuencial, pero no bloquea al resto del servidor — el await es
    implícito, lo maneja el runtime, no vos.
1.2 En que iguala "compilado" con "mismo modelo". Node es single-thread + event loop con
    async/await explícito; Go corre miles de goroutines sobre varios threads del SO con un
    scheduler propio, y la concurrencia es del runtime (vos solo marcás `go`). Además Go usa
    todos los cores sin "cluster"; Node necesita procesos/workers para eso.
1.3 La composición (embeber structs) y las interfaces implícitas. En vez de heredar de una
    clase base, componés tipos y dependés de interfaces pequeñas; eso evita jerarquías
    frágiles y acopla por comportamiento, no por linaje.
```

### Módulo 2
```
2.1 string → "" (cadena vacía); int → 0; *Usuario → nil.
2.2 Porque Go no tiene null/undefined: un valor faltante es el zero value (""/0/struct vacío),
    no "ausencia". El peligro que reemplaza al undefined es el puntero nil: desreferenciar un
    *T que es nil produce un panic en runtime. El optional chaining no existe; chequeás nil
    explícito cuando trabajás con punteros.
```
```go
// 2.3
type Producto struct {
	ID            string
	Nombre        string
	PrecioCentavos int
}

func (p Producto) Precio() float64 {
	return float64(p.PrecioCentavos) / 100
}
```

### Módulo 3
```
3.1 %w "envuelve" el error original dentro del nuevo, preservando la cadena. Después podés
    usar errors.Is (comparar con un centinela) o errors.As (extraer un tipo concreto) para
    inspeccionar el error de más abajo sin perder el contexto que fuiste agregando.
3.2 Porque panic corta el flujo como una excepción no manejada y, si nadie hace recover,
    tumba el proceso (o la goroutine). "El usuario no existe" es un caso esperable de
    negocio, no un bug: devolvelo como error (un centinela ErrNoEncontrado) y que quien
    llama decida (responder 404). panic se reserva para estados imposibles/irrecuperables.
```
```go
// 3.3
var ErrDivisionPorCero = errors.New("división por cero")

func dividir(a, b int) (int, error) {
	if b == 0 {
		return 0, ErrDivisionPorCero
	}
	return a / b, nil
}
```

### Módulo 4
```
4.1 Sin buffer: el envío (ch <- v) bloquea hasta que otra goroutine recibe — sincroniza a las
    dos. Con buffer de N: el envío no bloquea mientras haya lugar en el buffer (hasta N
    pendientes); recién bloquea cuando está lleno. El recibir bloquea si está vacío en ambos
    casos.
4.2 Porque select espera sobre varios channels a la vez y avanza con el primero que esté
    listo. Ponés un case con el resultado de la operación y otro con <-time.After(d): si el
    resultado no llega antes de d, dispara el case del timeout. Es elegir "lo que ocurra
    primero" sin polling.
```
```go
// 4.3
func main() {
	ch := make(chan int, 3)
	var wg sync.WaitGroup
	for i := 0; i < 3; i++ {
		wg.Add(1)
		go func(n int) {
			defer wg.Done()
			ch <- n
		}(i) // explícito y claro; nota: desde Go 1.22 la variable del loop ya no se comparte entre iteraciones (antes, pasarla por argumento era OBLIGATORIO para evitar el bug de captura)
	}
	wg.Wait()
	close(ch)

	suma := 0
	for v := range ch {
		suma += v
	}
	fmt.Println(suma) // 3
}
```

### Módulo 5
```
5.1 Por convención, para que la cancelación/deadline fluya por toda la cadena de llamadas de
    forma uniforme (el primer parámetro = "el contexto de esta operación"). El defer cancel()
    libera los recursos asociados al contexto derivado (timers, goroutines internas) aunque la
    función salga por un camino temprano; no llamarlo es una fuga de recursos.
5.2 r.Context() está atado a la request: si el cliente corta, ese contexto se cancela. Si
    pasaste ese ctx a la query con QueryRowContext(ctx, ...), el driver aborta la consulta en
    la DB. La clave era propagar el contexto hasta la capa de datos; si hubieras usado la
    variante sin Context, la query seguiría corriendo igual.
```
```go
// 5.3
type Contador struct {
	mu sync.Mutex
	n  int
}

func (c *Contador) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *Contador) Valor() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n
}
```

### Módulo 6
```
6.1 Routing por método + patrón con variables de ruta: mux.HandleFunc("GET /usuarios/{id}",
    ...) y r.PathValue("id"). Antes el ServeMux no distinguía método ni capturaba segmentos,
    así que para eso casi siempre metías chi/gin/echo.
6.2 Porque un http.Handler es una interface con un método (ServeHTTP); un middleware es una
    función que recibe un Handler y devuelve otro que hace algo antes/después y llama a
    next.ServeHTTP. La ventaja: es explícito y componible (los encadenás vos, ves el orden),
    sin un registro mágico ni un orden implícito difícil de depurar.
```
```go
// 6.3
func salud(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(map[string]string{"status": "ok"})
}
// registro: mux.HandleFunc("GET /salud", salud)
```

### Módulo 7
```
7.1 Que un tipo implementa una interface con solo tener los métodos que ésta declara — el
    compilador lo verifica solo. NO escribís "implements" (ni nada): no hay declaración
    explícita de conformidad.
7.2 Porque así el servicio define el contrato que necesita y no depende del paquete de
    Postgres (inversión de dependencias). En los tests le inyectás un repo falso que cumple la
    misma interface, sin tocar la DB. Si la interface viviera en el paquete Postgres, el
    servicio dependería de ese paquete concreto y testear sin DB sería incómodo.
```
```go
// 7.3
type Notificador interface {
	Enviar(ctx context.Context, a, msg string) error
}

type EmailNotificador struct{}

func (EmailNotificador) Enviar(ctx context.Context, a, msg string) error {
	// ... enviar email real ...
	return nil
}
```

### Módulo 8
```
8.1 Porque *sql.DB no es una conexión sino un pool concurrente y seguro para usar desde
    muchas goroutines: administra, reutiliza y limita conexiones. Abrir una por request
    desperdicia recursos, agota la DB y pierde el pooling. Lo creás una vez al arrancar y lo
    compartís.
8.2 (a) La cancelación/timeout del contexto llega hasta la DB (si la request se cancela, la
    query se aborta). (b) Seguridad: los parámetros $1 van separados del SQL, así que evitás
    inyección SQL (y además la DB puede cachear el plan). Concatenar el id en el string abre
    inyección.
```
```go
// 8.3
func (r *PostgresRepo) Guardar(ctx context.Context, u *Usuario) error {
	_, err := r.db.ExecContext(ctx,
		`INSERT INTO usuarios (id, nombre, edad) VALUES ($1, $2, $3)`,
		u.ID, u.Nombre, u.Edad)
	if err != nil {
		return fmt.Errorf("guardar usuario %s: %w", u.ID, err)
	}
	return nil
}
```

### Módulo 9
```
9.1 Es un test donde los casos son filas de una tabla (un slice de structs con entradas y
    salida esperada) y un loop los recorre con t.Run para subtests nombrados. Es el patrón por
    defecto porque agregar un caso = agregar una fila, los subtests se reportan por nombre, y
    evita copiar/pegar la misma lógica de aserción N veces.
9.2 Le inyectás un repo falso: una struct que cumple la interface UsuarioRepo y devuelve los
    valores que el test necesita (incluido un error). No necesitás librería de mocking porque
    las interfaces implícitas + la DI por constructor hacen trivial pasar una implementación
    de prueba.
```
```go
// 9.3
func TestDividir(t *testing.T) {
	casos := []struct {
		nombre   string
		a, b     int
		quiero   int
		quieroErr error
	}{
		{"normal", 10, 2, 5, nil},
		{"por cero", 1, 0, 0, ErrDivisionPorCero},
	}
	for _, c := range casos {
		t.Run(c.nombre, func(t *testing.T) {
			got, err := dividir(c.a, c.b)
			if !errors.Is(err, c.quieroErr) {
				t.Fatalf("err = %v; quiero %v", err, c.quieroErr)
			}
			if err == nil && got != c.quiero {
				t.Errorf("dividir(%d,%d) = %d; quiero %d", c.a, c.b, got, c.quiero)
			}
		})
	}
}
```

### Módulo 10
```
10.1 Porque go build genera un binario estático (con CGO_ENABLED=0 no depende ni de libc): no
     necesita intérprete, runtime ni librerías del SO en la imagen. Por eso la etapa final
     puede ser scratch (vacía) o distroless (solo lo mínimo), y la imagen pesa unos pocos MB.
10.2 Sin graceful shutdown, matar el proceso corta las requests en vuelo a la mitad: el
     cliente recibe errores/conexiones cortadas y podés dejar trabajo a medio hacer.
     srv.Shutdown(ctx) deja de aceptar conexiones nuevas pero espera (hasta el timeout del
     ctx) a que terminen las que están en curso, y recién ahí cierra.
```
```go
// 10.3
slog.SetDefault(slog.New(slog.NewJSONHandler(os.Stdout, nil)))
slog.Info("db conectada", "pool", 10)
```

### Módulo 11
```
11.1 Go. Es I/O-bound con alta concurrencia (validar + encolar): las goroutines manejan 10k
     req/s con código simple y predecible, el binario despliega trivial y el footprint es
     bajo. Node podría, pero un solo hilo + necesitar cluster para usar los cores lo complica;
     Rust sería más rápido pero el cuello acá es I/O (Redis/red), no CPU, así que su ventaja
     no se nota y pagás la curva.
11.2 Justificado: cuando el servicio es CPU-bound de verdad (cripto, parsing/compresión
     pesada, un motor de matching) o exige latencia p99 sin pausas de GC / RAM muy acotada —
     ahí el "sin GC" y el control fino de Rust pagan. Sobre-ingeniería: para una API CRUD
     I/O-bound normal, donde Rust solo te suma curva de aprendizaje y tiempo de desarrollo sin
     una ganancia que el negocio note.
```

### Módulo 12
```
12.1 Porque el servidor maneja concurrencia entre requests, pero apenas tu handler hace algo
     no trivial (llamar a otro servicio con timeout, paralelizar trabajo, cancelar al cortar
     el cliente, compartir un cache) necesitás goroutines, channels y context. Sin esa base,
     escribís handlers que bloquean, fugan goroutines o tienen data races. La concurrencia es
     el cimiento, no un adorno posterior.
12.2 Por ejemplo: (a) context propagado hasta la DB/servicios externos con timeouts (no
     queries que cuelgan); (b) graceful shutdown ante SIGTERM (no cortar requests en deploy);
     (c) tests reales (table-driven + httptest) y -race limpio; (d) errores envueltos con %w y
     manejados, logs estructurados con slog, capas separadas por interfaces. Cualquiera de
     estos separa el juguete del servicio serio.
```

---

## Siguientes pasos

Con este módulo tenés **Go orientado a backend** y un camino para llevarlo a producción. El recorrido natural desde acá: **(1)** profundizá la concurrencia con criterio de sistemas en [Concurrencia en system design](concurrencia.md) y su [laboratorio práctico](concurrencia-practica.md) — ahí ves los mismos patrones (worker pool, locks, backpressure) en Node/TS **y** Go, con el marco de fallas de producción; **(2)** llevá tu servicio Go a contenedores y pipeline con [Docker, deploy y CI/CD](docker-deploy.md), donde el binario estático brilla; **(3)** sumá [Observabilidad práctica](observabilidad.md) para que tus logs `slog` y tus métricas cuenten una historia. La constante: Go no te pide tirar lo que sabés de Node — te da una segunda herramienta, compilada y concurrente, para cuando el problema la pida.
