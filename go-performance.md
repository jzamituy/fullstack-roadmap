# Performance y profiling en Go: pprof, benchmarks y memoria

**Medir antes de optimizar · el tooling que distingue a un senior · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya hiciste [Go para backend](go-backend.md). Acá vemos lo que pocos lenguajes te dan tan integrado: **medir** dónde se va el tiempo y la memoria, con `pprof` y benchmarks de la stdlib. Es lo que separa "optimicé porque me pareció lento" de "medí, encontré el 3% del código que costaba el 80%, y lo arreglé". En una entrevista de Go, saber usar `pprof` es una señal fuerte de seniority.

**Lo que asumimos.** Go a nivel del módulo [Go para backend](go-backend.md): structs, slices/maps, goroutines, `context`, testing básico. **No** asumimos que sepas de GC, stack/heap ni profiling — eso lo construimos.

> ⚠️ **Nota sobre versiones.** Features recientes que usa este módulo: **`GOMEMLIMIT`** (límite de memoria del GC, Go 1.19), **`testing.B.Loop`** (la forma nueva y recomendada de escribir benchmarks, Go 1.24, feb 2025). Asumimos **Go 1.24+**; en versiones viejas, usá la forma clásica `for i := 0; i < b.N; i++`. Verificá contra la doc oficial (`pkg.go.dev`, `go.dev/blog`).

**Índice de módulos**
1. La mentalidad: medir antes de optimizar
2. Benchmarks con `testing.B`
3. `pprof`: qué es y qué perfiles hay
4. CPU profiling en la práctica
5. Memory profiling y *allocations*
6. Escape analysis: stack vs heap
7. El GC y cómo afinarlo (`GOGC`, `GOMEMLIMIT`)
8. Goroutine leaks: detectarlas
9. Optimizaciones idiomáticas (sin romper la legibilidad)
10. El flujo completo y el criterio

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere `go`; algunos, `graphviz` para `pprof -web`).

---

## Módulo 1 — La mentalidad: medir antes de optimizar

**Teoría.** La regla de oro de la performance, y la que más se viola: **no optimices por intuición, medí.** El humano es pésimo adivinando dónde está el cuello de botella; casi siempre apuesta al lugar equivocado y "optimiza" código que no importa, agregando complejidad a cambio de nada.

El método correcto, en orden:

1. **Definí qué te importa.** ¿Latencia (p95/p99)? ¿Throughput? ¿Uso de memoria? ¿Costo en la nube? Optimizar sin un objetivo es perder el tiempo.
2. **Medí el estado actual** con un benchmark o un *profile* en condiciones realistas.
3. **Encontrá el hotspot real** (el 3% del código que cuesta el 80% del tiempo — la ley de Amdahl en acción).
4. **Optimizá solo eso**, y **volvé a medir** para confirmar que mejoró (y que no rompiste otra cosa).

Dos verdades que enmarcan todo lo demás:
- **En un backend, el cuello suele ser I/O** (DB, red, otros servicios), no la CPU de tu Go. Antes de micro-optimizar un loop, preguntate si el problema no es una query sin índice o una llamada de red en serie que debería ser concurrente.
- **La legibilidad es un activo.** Una optimización que vuelve el código incomprensible para ganar un 2% casi nunca vale. Optimizá donde el profile dice, no donde tu ego dice.

Go es excepcional acá porque **el profiler viene en la caja** (`pprof`) y es production-grade: podés perfilar un servicio **en producción** con costo despreciable.

**Ejercicios 1**
1.1 🔁 ¿Cuáles son los cuatro pasos del método de optimización?
1.2 🧠 Un compañero dice "este loop se ve lento, lo voy a reescribir con punteros". ¿Qué le pedís antes de tocar nada?
1.3 🧠 En un servicio HTTP que consulta Postgres, ¿por qué micro-optimizar el parseo de JSON suele ser el lugar equivocado para empezar?

---

## Módulo 2 — Benchmarks con `testing.B`

**Teoría.** Los benchmarks viven en la stdlib, junto a los tests. Una función `BenchmarkXxx(b *testing.B)` en un archivo `_test.go`, y el runner la corre las veces necesarias para una medición estable.

La forma **clásica** (todas las versiones): un loop hasta `b.N`.

```go
func BenchmarkConcatenar(b *testing.B) {
	for i := 0; i < b.N; i++ {
		concatenar([]string{"a", "b", "c", "d"})
	}
}
```

La forma **nueva y recomendada (Go 1.24+)**: `for b.Loop()`. Evita una trampa clásica — que el compilador "optimice" y elimine el código medido por considerarlo inútil — y maneja el setup fuera de la medición:

```go
func BenchmarkConcatenar(b *testing.B) {
	datos := []string{"a", "b", "c", "d"} // setup: con b.Loop() ya queda FUERA de la medición
	for b.Loop() {
		concatenar(datos)
	}
}
```

> 📝 Con `for b.Loop()` el setup previo al loop **no se mide** por diseño, así que **no necesitás `b.ResetTimer()`** acá. Ese `b.ResetTimer()` quedaba para el patrón clásico `for i := 0; i < b.N; i++`, cuando el setup costoso iba dentro de la función y había que descartar su tiempo.

Lo corrés y leés:

```bash
go test -bench=. -benchmem -run=^$    # -benchmem agrega memoria; -run=^$ saltea los tests normales
```

```
BenchmarkConcatenar-8    5000000    250 ns/op    48 B/op    2 allocs/op
```

Cómo leerlo:
- **`-8`** = `GOMAXPROCS` (cores). **`5000000`** = iteraciones que corrió — **lo decide el framework**, no vos (con `b.Loop()` ni siquiera lo gobernás); no lo interpretes como "vueltas que elegí". Las lecturas útiles son las tres que siguen. **`250 ns/op`** = tiempo por operación.
- **`48 B/op`** = bytes asignados por operación. **`2 allocs/op`** = **cantidad de asignaciones** al heap por operación — la métrica más accionable: bajar `allocs/op` suele ser la optimización de mayor impacto, porque cada asignación es presión sobre el GC.

> 💡 Para comparar "antes vs después" sin engañarte con el ruido, guardá las salidas y usá **`benchstat`** (`golang.org/x/perf/cmd/benchstat`): te dice si la diferencia es **estadísticamente significativa** o ruido de medición.

**Ejercicios 2**
2.1 🔁 ¿Qué significan `ns/op`, `B/op` y `allocs/op` en la salida de un benchmark?
2.2 🧠 ¿Por qué `for b.Loop()` (1.24+) es más seguro que un `for i := 0; i < b.N; i++` con la lógica inline?
2.3 ✍️ Escribí un benchmark para `sumar(nums []int) int` usando el patrón **clásico** `for i := 0; i < b.N; i++`, con el slice creado en el setup y `b.ResetTimer()` para no medir ese setup (es el caso donde `ResetTimer` **sí** hace falta).
2.4 🧠 Comparás dos versiones: la nueva da 4% menos `ns/op` pero con alta varianza entre corridas. ¿Concluís que mejoró? ¿Qué herramienta usás para decidir?

---

## Módulo 3 — `pprof`: qué es y qué perfiles hay

**Teoría.** **`pprof`** es el profiler de Go: muestrea tu programa y te dice **dónde se gastan los recursos**. Hay varios tipos de *profile*, cada uno responde una pregunta distinta:

- **CPU profile** — ¿en qué funciones se va el **tiempo de CPU**? (Muestrea el stack ~100 veces/seg.) El que más usás para "está lento".
- **Heap profile** — ¿qué está asignando **memoria** y cuánta sigue **viva**? Para fugas de memoria y presión de GC.
- **Goroutine profile** — ¿cuántas goroutines hay y **dónde están bloqueadas**? El detector de **goroutine leaks** (Módulo 8).
- **Block profile** — ¿dónde se **bloquean** las goroutines esperando (channels, mutexes)?
- **Mutex profile** — ¿qué locks tienen **contención**?

Dos formas de obtenerlos:

1. **Desde un test/benchmark:** `go test -cpuprofile cpu.out -memprofile mem.out -bench=.`
2. **Desde un servicio vivo (lo más potente):** importás `net/http/pprof` solo por su efecto de registro y exponés un endpoint:

```go
import (
	"net/http"
	_ "net/http/pprof" // registra /debug/pprof/* en el DefaultServeMux
)

func main() {
	// ⚠️ NO expongas /debug/pprof públicamente: ponelo en un puerto/red interna
	go func() { http.ListenAndServe("localhost:6060", nil) }()
	// ... tu servidor real en otro puerto ...
}
```

Después analizás con `go tool pprof`:

```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30   # CPU, 30s
go tool pprof http://localhost:6060/debug/pprof/heap                 # memoria
```

> ⚠️ **Seguridad:** `net/http/pprof` registra rutas en el `DefaultServeMux`. Si tu servidor real usa ese mux, estarías exponiendo `/debug/pprof` con info sensible. El patrón de producción es un `http.ServeMux` **dedicado** en un **puerto/interfaz interna aparte** (no el `DefaultServeMux`, no el listener público), y protegido por la red del cluster; no confíes solo en `localhost:6060`, que con un `port-forward` o dentro de un contenedor puede quedar accesible.

**Ejercicios 3**
3.1 🔁 ¿Qué pregunta responde el CPU profile y cuál el heap profile?
3.2 🧠 ¿Por qué importar `net/http/pprof` con `_` (blank import) y no usarlo explícitamente?
3.3 🧠 ¿Cuál es el riesgo de seguridad de `net/http/pprof` y cómo lo mitigás?

---

## Módulo 4 — CPU profiling en la práctica

**Teoría.** Una vez que tenés un profile (de un benchmark o del endpoint), lo explorás con `go tool pprof`. Los comandos que más usás dentro de la consola interactiva:

```bash
go tool pprof cpu.out
```

- **`top`** — las funciones que más CPU consumen, ordenadas. Mirás `flat` (tiempo en *esa* función) vs `cum` (acumulado, incluyendo lo que llama).
- **`top -cum`** — ordena por acumulado: útil para ver qué *camino* domina aunque ninguna función sola se destaque.
- **`list nombreFuncion`** — te muestra el código fuente de esa función con el tiempo **anotado línea por línea**. Acá ves *exactamente* qué línea cuesta.
- **`web`** — abre un **grafo de llamadas** visual; el grosor de las flechas = tiempo. La mejor vista para entender de un vistazo dónde está el peso. ⚠️ Requiere **Graphviz** (`dot` en el `PATH`) — sin él, `web` falla; alternativa sin instalar nada: `go tool pprof -http=:8080 cpu.out`, que sirve la UI web completa (grafo + flame graph) en el navegador.

El flujo mental: `top` para ver los sospechosos → `list` sobre el sospechoso para ver la línea culpable → entender *por qué* esa línea cuesta (¿una asignación en un loop? ¿una conversión? ¿una llamada cara?).

> 💡 **Flame graphs:** la vista `web` (y la UI `go tool pprof -http=:8080 cpu.out`) incluye un *flame graph*: cada barra es una función, el ancho su tiempo. Es la forma más rápida de leer un profile de CPU — buscás las barras anchas.

**Ejercicios 4**
4.1 🔁 ¿Qué diferencia hay entre `flat` y `cum` en la salida de `top`?
4.2 🧠 `top` te muestra que `json.Marshal` está alto en `cum` pero bajo en `flat`. ¿Qué te dice eso sobre dónde está el costo real?
4.3 ✍️ Escribí la secuencia de comandos (shell + dentro de pprof) para perfilar 20 segundos de CPU de un servicio en `localhost:6060` y ver el código anotado de la función `procesarPedido`.

---

## Módulo 5 — Memory profiling y *allocations*

**Teoría.** El heap profile te dice **quién asigna memoria**. Es clave porque en Go **cada asignación al heap es trabajo para el GC**: menos asignaciones = menos presión de GC = menos pausas y menos CPU gastada recolectando.

Hay dos vistas del heap profile:
- **`inuse_space` / `inuse_objects`** (default) — memoria **viva ahora**. Para encontrar **fugas** ("¿por qué este servicio usa 4 GB?").
- **`alloc_space` / `alloc_objects`** — **total asignado** desde el arranque (aunque ya se haya liberado). Para encontrar **presión de asignación** ("¿qué función crea basura sin parar?").

```bash
go tool pprof -sample_index=alloc_space mem.out   # qué asignó más en total
go tool pprof -sample_index=inuse_space mem.out   # qué sigue vivo
```

Causas típicas de muchas asignaciones (y su arreglo, Módulo 9):
- **Slices/maps que crecen sin preasignar** → `make([]T, 0, n)` con capacidad conocida.
- **Concatenar strings en loop** (`s += x`) → `strings.Builder`.
- **Interfaces / `any` que fuerzan *boxing***, conversiones `[]byte`↔`string` innecesarias.
- **Punteros que escapan al heap** cuando podrían vivir en el stack (Módulo 6).

El profile + `allocs/op` del benchmark son las dos caras de lo mismo: el benchmark te dice *cuánto* asignás por operación; el heap profile te dice *quién* lo hace.

**Ejercicios 5**
5.1 🔁 ¿Cuál es la diferencia entre `inuse_space` y `alloc_space`, y para qué problema sirve cada uno?
5.2 🧠 ¿Por qué reducir `allocs/op` mejora la performance aunque tengas RAM de sobra?
5.3 🧠 Un servicio tiene la memoria viva (`inuse_space`) estable pero `alloc_space` crece muchísimo. ¿Hay fuga? ¿Qué problema sí tenés?

---

## Módulo 6 — Escape analysis: stack vs heap

**Teoría.** Go decide **en tiempo de compilación** si una variable vive en el **stack** (barata: se libera sola al volver la función, no toca el GC) o **escapa al heap** (cara: la administra el GC). A esto se le llama **escape analysis**.

Una variable escapa cuando el compilador **no puede probar** que deja de usarse al terminar la función. Casos típicos:
- Devolvés un **puntero** a una variable local (sigue viva después del `return`).
- La guardás en una estructura de vida más larga (un campo, un slice global).
- La pasás a algo cuyo tipo es una **interface** (`any`), o a una función que el compilador no puede analizar.

Lo ves con el flag `-m`:

```bash
go build -gcflags='-m' ./...
# salida: "./x.go:12:6: moved to heap: u"  → esa variable escapó
```

Por qué importa: en un *hot path* (código que corre millones de veces), que una variable escape al heap multiplica las asignaciones y la presión de GC. **No reescribas por esto sin medir** — pero cuando el profile apunta a un hot path con muchas allocs, `-m` te dice *por qué* escapan y cómo evitarlo (devolver el valor en vez del puntero, reusar buffers, etc.).

> 📝 Matiz importante: "stack = bueno, heap = malo" es una simplificación. El heap no es malo *per se*; el problema es la **frecuencia** de asignación en código caliente. En código frío, que algo escape no importa.

**Ejercicios 6**
6.1 🔁 ¿Qué diferencia de costo hay entre una variable en el stack y una en el heap, respecto del GC?
6.2 🧠 Nombrá dos situaciones que hacen que una variable local "escape" al heap.
6.3 ✍️ Escribí el comando que muestra las decisiones de escape analysis de un paquete, y explicá en una línea qué buscás en la salida.

---

## Módulo 7 — El GC y cómo afinarlo (`GOGC`, `GOMEMLIMIT`)

**Teoría.** Go tiene un **garbage collector concurrente** de pausas muy bajas (sub-milisegundo en la mayoría de los casos). Para el 99% de los servicios **no tenés que tocarlo** — y la primera optimización de GC siempre es **asignar menos** (Módulos 5-6), no afinar perillas. Pero conviene entender las dos que existen:

- **`GOGC`** (default `100`) — controla *cada cuánto* corre el GC. `GOGC=100` significa "corré el GC cuando el heap creció 100% desde la última recolección". Subirlo (`GOGC=200`) → GC menos frecuente, **menos CPU en GC pero más RAM**; bajarlo → al revés. Es un trade-off CPU↔memoria.
- **`GOMEMLIMIT`** (Go 1.19+) — un **límite suave de memoria** total. El GC se vuelve más agresivo a medida que te acercás al límite, para no superarlo. Es la perilla que **evita los OOM kills** en contenedores: la configurás un poco por debajo del límite de memoria del pod.

```bash
GOGC=200 GOMEMLIMIT=900MiB ./miapp    # menos GC, pero sin pasar de ~900 MiB
```

El patrón recomendado en Kubernetes (2026): **`GOMEMLIMIT` ≈ 90% del límite de memoria del contenedor**, y `GOGC` en default salvo que un profile diga lo contrario. Así el GC se autorregula contra el techo real del pod en vez de que el kernel te mate el proceso.

> ⚠️ **`GOMEMLIMIT` es un límite *suave*:** el GC hará lo posible por respetarlo, pero si tu programa genuinamente necesita más memoria viva, la pedirá igual (mejor eso que petar). No es un reemplazo de arreglar una fuga real.

**Ejercicios 7**
7.1 🔁 ¿Qué controla `GOGC` y qué trade-off representa subirlo?
7.2 🧠 ¿Por qué `GOMEMLIMIT` es especialmente útil corriendo en un contenedor con límite de RAM?
7.3 🧠 Antes de tocar `GOGC`/`GOMEMLIMIT`, ¿cuál es la optimización de GC que siempre va primero, y por qué?

---

## Módulo 8 — Goroutine leaks: detectarlas

**Teoría.** Una **goroutine leak** es una goroutine que **nunca termina**: queda bloqueada para siempre esperando en un channel que nadie cierra, un lock que no se libera, o un `context` que no se cancela. No es como una fuga de memoria clásica — la goroutine ocupa stack y mantiene vivas sus variables (impidiendo que el GC las libere), y si el patrón se repite por request, las goroutines se **acumulan** hasta tumbar el servicio.

Causas típicas (todas vistas en [Go para backend](go-backend.md)):
- Mandar a un channel **sin buffer** que nadie lee (o leer de uno que nadie llena).
- Lanzar una goroutine con `go` y no darle forma de salir cuando el `context` se cancela.
- Un *worker* que hace `for range ch` sobre un channel que **nunca se cierra**.

Cómo las detectás:
- **Goroutine profile:** `go tool pprof http://localhost:6060/debug/pprof/goroutine` — te muestra cuántas hay y **en qué línea están bloqueadas**. Si el número crece sin parar con el tráfico, tenés un leak.
- **En tests:** `go.uber.org/goleak` falla el test si quedaron goroutines vivas al terminar — un *guardrail* excelente para CI.

```go
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m) // falla si algún test deja goroutines colgadas
}
```

La cura es de diseño: toda goroutine debe tener una **vía de salida** — un `context` que se cancela, un channel que se cierra, un `select` con `<-ctx.Done()`.

**Bonus: block y mutex profiles (contención).** Los perfiles de bloqueo y de mutex (que nombramos en el Módulo 3) atacan un problema relacionado: goroutines que *sí* avanzan pero **esperan demasiado** por un lock o un channel. No están activos por defecto — los habilitás explícitamente y después los leés como cualquier otro profile:

```go
runtime.SetBlockProfileRate(1)     // muestrea eventos de bloqueo (channels, etc.)
runtime.SetMutexProfileFraction(1) // muestrea contención de mutex
// luego: go tool pprof http://localhost:6060/debug/pprof/mutex   (o .../block)
```

Cuándo sospechar contención: throughput que **no escala** al agregar cores/goroutines, o un CPU profile que se ve "ocioso" pese a la carga — señal de que un **lock global** (un cache con `sync.Mutex`, un contador compartido) está serializando todo. La cura suele ser *sharding* del lock, `sync.RWMutex` si hay muchos lectores, o estructuras lock-free (ver [Concurrencia](concurrencia.md)).

**Ejercicios 8**
8.1 🔁 ¿Qué es una goroutine leak y por qué se acumula con el tráfico?
8.2 🧠 ¿Cómo usás el goroutine profile para confirmar que tenés un leak (no solo muchas goroutines momentáneas)?
8.3 ✍️ Mostrá el `TestMain` con `goleak` que falla la suite si quedan goroutines colgadas.
8.4 🧠 Agregás más workers a un pool y el throughput no sube (incluso baja). ¿Qué profile activás y qué sospechás?

---

## Módulo 9 — Optimizaciones idiomáticas (sin romper la legibilidad)

**Teoría.** Cuando el profile ya te señaló el hot path, estas son las palancas idiomáticas — todas apuntan a **asignar menos**:

- **Preasignar slices con capacidad conocida.** `make([]T, 0, n)` evita que el slice se reasigne y copie mientras crece dentro de un loop.

```go
res := make([]int, 0, len(entrada)) // una sola asignación
for _, x := range entrada {
	res = append(res, x*2)
}
```

- **`strings.Builder` para concatenar en loop**, en vez de `s += x` (que asigna un string nuevo cada vez):

```go
var b strings.Builder
for _, parte := range partes {
	b.WriteString(parte)
}
return b.String()
```

- **`sync.Pool`** para **reusar objetos caros** que se crean y descartan muy seguido (buffers, structs grandes) en código de alta frecuencia. Reduce allocs y presión de GC — pero es una optimización avanzada, **solo cuando el profile lo justifica** (mal usado, complica sin ganar). ⚠️ Gotcha clásico: `pool.Get()` devuelve un objeto **potencialmente sucio** (reusado de antes), así que **reseteálo antes de usarlo** (ej. `buf.Reset()`); olvidarse es una fuente típica de bugs de datos cruzados entre requests.
- **Evitar conversiones `[]byte`↔`string` innecesarias** en hot paths (cada una puede copiar).
- **Pasar por valor lo chico, por puntero lo grande/mutable** — pero medí: copiar un struct chico suele ser más barato que la indirección + escape al heap del puntero.

> 📝 **La regla que ata todo:** estas técnicas valen **en el hot path que el profile señaló**, no en todo el código. Preasignar un slice de 3 elementos o meter `sync.Pool` en código frío es complejidad sin retorno. Legibilidad por default; optimización donde la medición manda.

**Ejercicios 9**
9.1 🔁 ¿Por qué `make([]T, 0, n)` con capacidad puede ser más rápido que `var s []T` cuando vas a hacer `append` en un loop?
9.2 🧠 ¿Cuándo tiene sentido `sync.Pool` y cuándo es sobre-ingeniería?
9.3 ✍️ Reescribí esta concatenación para minimizar asignaciones: `func unir(ps []string) string { s := ""; for _, p := range ps { s += p }; return s }`.

---

## Módulo 10 — El flujo completo y el criterio

**Teoría.** Cerramos atando todo en el flujo que usarías en un caso real, y el criterio para no caer en la trampa de optimizar a ciegas.

**El flujo completo:**
1. Tenés un síntoma con número: "el endpoint `/pedidos` tarda p95 = 800 ms" o "el pod usa 3 GB".
2. **Reproducís con un benchmark** o capturás un **profile en producción** (`/debug/pprof`) bajo carga real.
3. **CPU profile** (`top` → `list` → flame graph) si es lentitud; **heap profile** (`alloc_space`/`inuse_space`) si es memoria; **goroutine profile** si sospechás leak.
4. Encontrás el hotspot, entendés *por qué* cuesta (allocs en loop, escape al heap, I/O en serie, lock con contención).
5. **Optimizás solo eso** con la palanca que corresponda (Módulo 9, o concurrencia, o un índice en la DB).
6. **Volvés a medir** con `benchstat` para confirmar mejora real, no ruido.

**El criterio (lo que más importa):**
- **El cuello casi siempre es I/O, no la CPU de Go.** Antes de micro-optimizar, mirá si el problema no es una query sin índice ([PostgreSQL](postgresql.md)), llamadas de red en serie que deberían ser concurrentes ([Go para backend](go-backend.md), Módulo 4), o falta de caché ([Redis](redis.md)).
- **Optimización prematura = deuda.** Complejizar sin un profile que lo respalde casi siempre es una mala inversión.
- **Esto conecta con observabilidad:** los profiles te dicen el *qué* a nivel de código; las métricas y trazas de [Observabilidad](observabilidad.md) te dicen el *dónde* a nivel de sistema. Profiling continuo (Pyroscope/Grafana) lleva `pprof` a producción permanente.

**Ejercicios 10**
10.1 🔁 En el flujo de optimización, ¿qué hacés *después* de aplicar el cambio, y con qué herramienta?
10.2 🧠 Un servicio Go tiene p99 alto. El CPU profile muestra que tu código casi no aparece arriba. ¿Dónde mirás entonces?
10.3 🧠 ¿Por qué "optimizar a ciegas" (sin profile) suele dejar el sistema peor, no mejor?

---

## Soluciones

### Módulo 1
```
1.1 (1) Definir la métrica que importa (latencia/throughput/memoria/costo); (2) medir el
    estado actual; (3) encontrar el hotspot real; (4) optimizar solo eso y volver a medir.
1.2 Que mida primero: un benchmark del estado actual y/o un CPU profile que demuestre que ese
    loop es de verdad un hotspot. Sin medición, probablemente esté por optimizar código que no
    pesa, agregando complejidad a cambio de nada.
1.3 Porque en un servicio que pega a Postgres el tiempo casi seguro se va en la query (I/O):
    red + ejecución en la DB. El parseo de JSON suele ser una fracción mínima; optimizarlo no
    mueve la aguja si el cuello es una query sin índice o una llamada en serie.
```

### Módulo 2
```
2.1 ns/op = nanosegundos por operación (tiempo). B/op = bytes asignados al heap por operación.
    allocs/op = cantidad de asignaciones al heap por operación (la más accionable: cada alloc
    es presión de GC).
2.2 Porque b.Loop() impide que el compilador elimine ("optimice fuera") el código medido por
    considerarlo sin efecto, y separa el setup de la medición. Con el for clásico inline, si
    no usás el resultado, el compilador puede borrar la llamada y medís cero/nada real.
```
```go
// 2.3 (patrón clásico, donde ResetTimer sí hace falta)
func BenchmarkSumar(b *testing.B) {
	nums := make([]int, 1000)
	for i := range nums {
		nums[i] = i
	}
	b.ResetTimer() // descarta el tiempo del setup de arriba
	for i := 0; i < b.N; i++ {
		sumar(nums)
	}
}
```
```
2.4 No: un 4% con alta varianza puede ser puro ruido de medición (scheduling, CPU térmica,
    GC). Corrés cada versión varias veces (go test -bench=. -count=10), guardás las salidas y
    las pasás por benchstat, que reporta la media, la variación (±%) y si la diferencia es
    estadísticamente significativa. Solo si benchstat lo confirma, concluís que mejoró.
```

### Módulo 3
```
3.1 El CPU profile responde "¿en qué funciones se va el tiempo de CPU?" (muestreando el stack).
    El heap profile responde "¿qué asigna memoria y cuánta sigue viva?" (para fugas y presión
    de GC).
3.2 Porque net/http/pprof no se usa por sus símbolos: su init() registra las rutas
    /debug/pprof/* en el DefaultServeMux como efecto secundario. El blank import (_) lo incluye
    solo para ejecutar ese init sin que el compilador se queje de "import no usado".
3.3 Registra rutas con info sensible (perfiles, memoria, stacks) en el DefaultServeMux; si tu
    servidor público usa ese mux, las exponés. Se mitiga sirviéndolas en un puerto/interfaz
    interna aparte (ej. localhost:6060), nunca en el listener público.
```

### Módulo 4
```
4.1 flat = tiempo gastado DENTRO de esa función (sin contar lo que llama). cum = acumulado,
    incluye el tiempo de las funciones que esa llama. Una función con cum alto y flat bajo solo
    "pasa el tiempo" delegando; el costo real está más abajo.
4.2 Que json.Marshal en sí no es el costo: el tiempo está en lo que llama por debajo (reflexión,
    asignaciones, los Marshal de los campos). flat bajo = la función no quema CPU por sí misma;
    cum alto = el camino que arranca ahí sí. Hay que bajar con list/top -cum para hallar la línea.
```
```bash
# 4.3
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=20
# dentro de la consola interactiva de pprof:
(pprof) top
(pprof) list procesarPedido
```

### Módulo 5
```
5.1 inuse_space = memoria viva en este momento (sirve para encontrar fugas: qué sigue ocupado).
    alloc_space = total asignado desde el arranque, aunque ya se haya liberado (sirve para
    encontrar presión de asignación: qué función genera basura sin parar).
5.2 Porque cada asignación al heap es trabajo para el GC: más allocs = el GC corre más seguido
    y/o más caro = más CPU gastada recolectando y más pausas. Bajar allocs/op mejora latencia y
    throughput aunque sobre RAM.
5.3 No hay fuga (la memoria viva es estable, no crece). El problema es presión de asignación:
    la app crea y descarta muchísima memoria, lo que hace trabajar al GC de más (CPU/pausas).
    Se ataca reduciendo allocs (preasignar, Builder, Pool), no subiendo RAM.
```

### Módulo 6
```
6.1 El stack se libera solo al retornar la función (costo casi nulo, el GC ni lo mira). El heap
    lo administra el GC: cada objeto ahí es trabajo de recolección. Por eso, en código caliente,
    que algo escape al heap multiplica la presión de GC.
6.2 Dos de: devolver un puntero a una variable local; guardarla en una estructura/slice de vida
    más larga o global; pasarla por una interface (any) o a una función que el compilador no
    puede analizar (la fuerza a vivir más allá del stack frame).
```
```bash
# 6.3
go build -gcflags='-m' ./...
# Busco líneas como "moved to heap: x" o "x escapes to heap": señalan variables que escapan
# al heap; en un hot path con muchas allocs, son candidatas a evitar (devolver valor, reusar buffer).
```

### Módulo 7
```
7.1 GOGC controla cada cuánto corre el GC (default 100 = recolectar cuando el heap creció 100%
    desde la última pasada). Subirlo (200) hace el GC menos frecuente: gastás MENOS CPU en GC
    pero MÁS RAM. Es un trade-off CPU↔memoria.
7.2 Porque pone un techo suave de memoria total: el GC se vuelve más agresivo al acercarse al
    límite, evitando que el proceso supere la RAM del contenedor y el kernel lo mate (OOM kill).
    Se setea ~90% del límite del pod.
7.3 Asignar menos (Módulos 5-6): menos allocs = menos trabajo de GC, sin trade-offs. Tocar GOGC
    o GOMEMLIMIT solo mueve el balance CPU↔RAM; reducir basura mejora ambos a la vez.
```

### Módulo 8
```
8.1 Una goroutine que nunca termina (bloqueada para siempre en un channel/lock/context que no se
    resuelve). Si el patrón ocurre por request, cada request deja una goroutine colgada: se
    acumulan (stack + variables vivas) hasta degradar o tumbar el servicio.
8.2 Capturás el goroutine profile bajo carga y mirás el conteo y DÓNDE están bloqueadas. Si el
    número crece de forma monótona con el tráfico y no baja cuando la carga cede (y ves muchas
    paradas en la misma línea de bloqueo), es un leak, no picos momentáneos.
```
```go
// 8.3
import "go.uber.org/goleak"

func TestMain(m *testing.M) {
	goleak.VerifyTestMain(m)
}
```
```
8.4 Sospechás contención de lock: hay un mutex/lock global (un cache, un contador compartido)
    que serializa a todos los workers, así que sumar workers solo agrega competencia por ese
    lock (y context switching), no paralelismo real. Activás el mutex profile
    (runtime.SetMutexProfileFraction) y el block profile (runtime.SetBlockProfileRate), y los
    leés con go tool pprof .../mutex y .../block. La cura: shardear el lock, usar RWMutex si
    dominan los lectores, o rediseñar para no compartir ese estado.
```

### Módulo 9
```
9.1 Porque sin capacidad, append va reasignando un array más grande y COPIANDO los elementos
    cada vez que se llena (varias asignaciones). Con make([]T, 0, n) reservás de una vez el
    espacio para n: una sola asignación y cero copias de crecimiento.
9.2 Tiene sentido cuando creás y descartás MUCHOS objetos caros del mismo tipo en código de alta
    frecuencia (buffers, structs grandes) y el profile muestra esas allocs como hotspot. Es
    sobre-ingeniería en código frío o para objetos baratos: agrega complejidad (y bugs de reuso)
    sin ganancia medible.
```
```go
// 9.3
func unir(ps []string) string {
	var b strings.Builder
	for _, p := range ps {
		b.WriteString(p)
	}
	return b.String()
}
```

### Módulo 10
```
10.1 Volvés a medir (re-correr el benchmark / capturar el profile de nuevo) y comparás con
     benchstat, para confirmar que la mejora es estadísticamente significativa y no ruido —y que
     no empeoraste otra métrica.
10.2 En I/O y en el sistema, no en tu CPU: queries a la DB (índices, N+1), llamadas de red en
     serie que deberían ir concurrentes, locks con contención (mutex/block profile), o esperas
     externas. El CPU profile bajo en tu código es justamente la señal de que el tiempo se va
     esperando, no computando.
10.3 Porque sin profile apostás al lugar equivocado (la intuición casi siempre falla): agregás
     complejidad y posibles bugs en código que no pesaba, mientras el verdadero cuello sigue
     intacto. Resultado: más difícil de leer e igual de lento (o peor).
```

---

## Siguientes pasos

Con este módulo tenés el **tooling de performance** que distingue a un backend Go senior: medir con benchmarks, perfilar con `pprof`, entender el GC y cazar goroutine leaks. El recorrido natural desde acá: **(1)** llevá el profiling a **producción continua** con Pyroscope/Grafana, atándolo a [Observabilidad](observabilidad.md) — métricas y trazas para el *dónde* del sistema, profiles para el *qué* del código; **(2)** recordá que el cuello suele estar en datos: cruzá esto con [PostgreSQL](postgresql.md) (índices, `EXPLAIN`, N+1) y [Redis](redis.md) (caché), porque ninguna micro-optimización de CPU compite con arreglar una query lenta; **(3)** repasá la [Concurrencia](concurrencia.md) con esta lente: paralelizar I/O en serie suele ser la mejora de mayor impacto, y `pprof` te muestra cuándo la contención de locks la está frenando. La constante: **medí, no adivines.**
