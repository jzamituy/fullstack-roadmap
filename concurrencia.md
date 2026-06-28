# Concurrencia en system design a fondo

**De las primitivas a los patrones que se rompen a escala · Node + Go · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo te da los **20 conceptos** que tenés que dominar para hablar de concurrencia en una entrevista de system design. Su complemento es el [laboratorio práctico](concurrencia-practica.md): acá entendés, allá **construís** (una race condition, un producer-consumer, un pool con semáforo, un deadlock a propósito y un thread pool con cola acotada).

**Lo que asumimos.** JavaScript/TypeScript moderno (`Promise`, `async/await`), una idea de qué es un proceso del sistema operativo, y haber visto el event loop de Node aunque sea de pasada (lo recordamos abajo). No asumimos que sepas Go: los ejemplos en Go se explican línea por línea.

> **¿Te falta alguna base?** Si el **event loop de Node** te suena nuevo, leé primero [Node.js por dentro](nodejs.md) — acá lo usamos como ancla para el modelo mental. Si nunca pensaste la concurrencia "desde abajo", no importa: el módulo 1 arranca de cero.

> ⚠️ **Sobre los lenguajes.** Node es **single-thread con event loop**: te da concurrencia (muchas tareas en progreso) pero **no paralelismo de memoria compartida** por defecto. Por eso los ejemplos van en **dos planos**: en **Node/TS** (tu stack: `async`, `worker_threads`, `Atomics`, `SharedArrayBuffer`) y en **Go** (goroutines, channels, `sync.Mutex` — concurrencia *real* con hilos), con alguna nota en **Java** donde es el ejemplo canónico. Ver el mismo concepto en los dos planos es lo que te hace sonar senior: entendés la idea, no la sintaxis de un runtime.

### El marco que usamos en todo el módulo

Una entrevista de system design no se gana recitando definiciones. Se gana **razonando fallas**. Por eso, para cada concepto que puede romperse a escala, lo atacamos en **cuatro capas**:

1. **Qué se rompe** exactamente.
2. **Por qué se rompe** a *esta* escala o con *esta* forma de tráfico (no en abstracto).
3. **Control de corto plazo** que frena el *blast radius* (el daño inmediato).
4. **Cambio de diseño de largo plazo** que baja la probabilidad de que se repita.

> **El ejemplo que fija el tono — el *retry storm*.** Respuesta débil: *"usá exponential backoff"*. Respuesta profunda, en cascada: **la latencia sube → los timeouts disparan → los clientes reintentan en paralelo → la carga sobre la dependencia crece → la latencia sube más → y ahora un mecanismo de recuperación (el reintento) se convirtió en un amplificador de carga**. Eso es ver el sistema, no la línea de código. Volveremos a este caso en el módulo 19.

Y cuando tengas que **diseñar** algo concurrente, una buena respuesta tiene **tres cosas**: una **forma para el happy path**, un **modelo de falla**, y un **proceso de recuperación**. Lo cerramos en el módulo 21.

**Tipos de ejercicio** (para saber cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (el grueso de la implementación está en el [laboratorio](concurrencia-practica.md)).

**Índice de módulos**

*Etapa 1 — Qué es concurrencia (el modelo mental)*
1. Concurrencia
2. Concurrencia vs Paralelismo
3. Procesos vs Hilos
4. Ciclo de vida de un hilo

*Etapa 2 — Por qué duele: el estado compartido*
5. Race condition (el corazón del tema)
6. Operaciones atómicas
7. Inmutabilidad

*Etapa 3 — Primitivas de sincronización*
8. Mutex
9. Semáforo
10. Variable de condición
11. Lock reentrante
12. Locking grueso vs fino

*Etapa 4 — Patologías (lo que sale mal)*
13. Deadlock
14. Livelock
15. Starvation

*Etapa 5 — Patrones a escala (system design)*
16. Thread pool
17. Producer-Consumer
18. Blocking vs Non-blocking
19. Backpressure (y el retry storm)

*Etapa 6 — Correctitud en distribuido*
20. Idempotencia bajo concurrencia

*Cierre*
21. Cómo hablar de concurrencia en una entrevista

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Etapa 1 — Qué es concurrencia (el modelo mental)

## Módulo 1 — Concurrencia

**Teoría.** **Concurrencia es que muchas tareas progresan en la misma ventana de tiempo**, aunque no se ejecuten en el mismísimo instante. La imagen clásica: un cocinero que pone la pasta a hervir, y *mientras* hierve corta la cebolla y revuelve la salsa. No hace tres cosas en el mismo microsegundo — **intercala**: avanza una, la deja esperando, avanza otra. Hay un solo cocinero (un core) y aun así tres platos progresan.

Eso es exactamente lo que hace Node con un solo hilo: mientras una request espera la respuesta de la base de datos (I/O), el event loop atiende otra. La espera de una tarea es la oportunidad de otra.

La distinción que importa: concurrencia es una propiedad de **cómo estructurás** el trabajo (tareas que se solapan en el tiempo), no de cuántos cores tenés. Podés tener concurrencia con un solo core.

La frase mental: **concurrencia = varias tareas en vuelo a la vez; no necesariamente ejecutándose en el mismo instante, sino progresando en la misma ventana de tiempo.**

**Ejercicios 1**
1.1 🔁 Definí concurrencia en una frase, sin usar la palabra "paralelo".
1.2 🧠 Node tiene un solo hilo para tu código. ¿Cómo logra que mil requests "progresen a la vez"?
1.3 🧠 ¿Por qué la concurrencia no requiere varios cores?

---

## Módulo 2 — Concurrencia vs Paralelismo

**Teoría.** Se confunden todo el tiempo y en una entrevista distinguirlos te posiciona enseguida:

- **Concurrencia = intercalado (interleaving).** Un core alterna entre tareas; ninguna corre en el mismo instante que otra, pero todas avanzan.
- **Paralelismo = ejecución simultánea de verdad**, en varios cores, en el mismo instante.

| | Concurrencia | Paralelismo |
|---|---|---|
| Qué es | Estructurar tareas que se solapan en el tiempo | Ejecutarlas literalmente a la vez |
| Hardware | Funciona con **1 core** | Requiere **varios cores** |
| Pregunta que responde | *¿Cómo manejo muchas cosas a la vez?* | *¿Cómo hago más cosas por segundo?* |
| En Node | El **event loop** (default) | **`worker_threads`** / **`cluster`** |
| En Go | Goroutines en 1 thread | El runtime las reparte en varios cores (`GOMAXPROCS`) |

La frase de Rob Pike (creador de Go) que conviene tener lista: **"la concurrencia no es paralelismo"**. La concurrencia es *diseño* (cómo descomponés el problema en tareas independientes); el paralelismo es *ejecución* (correrlas a la vez si hay cores). Una app bien diseñada de forma concurrente **puede** volverse paralela si le das más cores — pero son cosas distintas.

```ts
// Node: concurrencia SIN paralelismo — un solo hilo, pero las dos esperas se solapan
const [usuario, pedidos] = await Promise.all([
  fetchUsuario(id),   // mientras esta espera I/O...
  fetchPedidos(id),   // ...esta ya arrancó. Se solapan, no corren "a la vez" en CPU.
])
```

```go
// Go: paralelismo real — el runtime puede correr estas goroutines en cores distintos
var wg sync.WaitGroup
for _, tarea := range tareas {
    wg.Add(1)
    go func(t Tarea) {        // 'go' lanza una goroutine
        defer wg.Done()
        procesar(t)            // si hay cores libres, corren simultáneamente
    }(tarea)
}
wg.Wait()                      // espera a que todas terminen
```

La frase mental: **concurrencia es cómo lo diseñás (tareas solapadas); paralelismo es cómo lo ejecutás (a la vez en varios cores). Node te da lo primero gratis; lo segundo, con worker_threads.**

**Ejercicios 2**
2.1 🔁 Diferencia en una línea entre concurrencia y paralelismo.
2.2 🧠 ¿Por qué se dice que una app puede ser concurrente sin ser paralela, pero el paralelismo "útil" casi siempre nace de un buen diseño concurrente?
2.3 🧠 Tenés una tarea CPU-bound pesada (comprimir imágenes) que bloquea el event loop. ¿Concurrencia o paralelismo te saca del problema en Node, y con qué herramienta?

---

## Módulo 3 — Procesos vs Hilos

**Teoría.** Son las dos unidades de ejecución que te da el sistema operativo, y la diferencia define casi todos los riesgos del tema:

| | Proceso | Hilo (thread) |
|---|---|---|
| Memoria | **Aislada**: cada uno su espacio | **Compartida** entre hilos del mismo proceso |
| Peso | Pesado (crear/cambiar cuesta) | Liviano |
| Comunicación | IPC (mensajes, sockets, pipes) | Variables compartidas (¡peligro!) |
| Si uno crashea | Los demás siguen vivos | Puede llevarse todo el proceso |
| Riesgo de races | Bajo (no comparten memoria) | **Alto** (comparten memoria) |

La idea central: **los hilos son livianos y rápidos porque comparten memoria — y son peligrosos por exactamente lo mismo.** Dos hilos tocando la misma variable sin coordinación es la fuente N°1 de bugs de concurrencia (módulo 5). Los procesos evitan ese problema a costa de aislamiento y de comunicación más cara.

En Node esto se mapea directo:
- **`cluster`** = varios **procesos** Node (cada uno su memoria, su event loop). Bueno para escalar un servidor HTTP en varios cores sin compartir estado.
- **`worker_threads`** = varios **hilos** dentro de un proceso. Comparten memoria solo si vos lo pedís explícitamente con `SharedArrayBuffer`; por defecto se comunican por mensajes (más seguro).

> El diseño de Node es deliberado: hace que **compartir memoria sea opt-in** (`SharedArrayBuffer`), no el default. Así te empuja al modelo de "comunicar por mensajes" (como procesos), que es mucho más difícil de romper. En Go, en cambio, la memoria compartida entre goroutines es el default — y por eso Go te da `go run -race` para detectar las races (módulo 5).

La frase mental: **procesos = aislados y seguros pero caros; hilos = livianos y veloces pero comparten memoria, y ahí nacen casi todos los bugs. Node te deja elegir: `cluster` (procesos) o `worker_threads` (hilos).**

**Ejercicios 3**
3.1 🔁 ¿Qué comparten los hilos de un proceso que los procesos entre sí no comparten?
3.2 🧠 "Los hilos son más riesgosos que los procesos." ¿Por qué — y qué exacta característica los hace a la vez más rápidos y más peligrosos?
3.3 🧠 Querés escalar un servidor Express stateless a 8 cores. ¿`cluster` o `worker_threads`? ¿Y si en cambio querés acelerar un cálculo numérico pesado compartiendo un buffer grande?

---

## Módulo 4 — Ciclo de vida de un hilo

**Teoría.** Un hilo no está "corriendo o no": pasa por **estados**. Conocerlos te deja leer un volcado de hilos (thread dump) y entender dónde se traba un sistema:

```
NEW → RUNNABLE → RUNNING → (BLOCKED | WAITING) → RUNNABLE → ... → TERMINATED
```

- **New**: creado, todavía no arrancó.
- **Runnable**: listo para correr, esperando que el scheduler le dé CPU.
- **Running**: ejecutándose en un core ahora.
- **Blocked**: esperando un **lock** que tiene otro hilo (módulo 8).
- **Waiting**: dormido hasta que algo lo despierte (una señal, I/O, un timeout) (módulo 10).
- **Terminated**: terminó (normal o por excepción).

> ⚠️ **Es un modelo simplificado.** Distintos runtimes nombran los estados distinto. La **JVM**, por ejemplo, **colapsa runnable+running** en un solo estado `RUNNABLE` (no distingue "listo" de "ejecutándose") y agrega `TIMED_WAITING` (un *waiting* con timeout, como `sleep(100ms)`). Si en una entrevista de Java te preguntan los estados de `Thread.State`, son seis: `NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED`. La idea de fondo (corriendo / esperando un lock / esperando una señal / terminado) es la misma.

Por qué importa en una entrevista: cuando un sistema "se cuelga", el thread dump te dice **en qué estado están los hilos**. Muchos en `BLOCKED` sobre el mismo lock → contención o deadlock (módulo 13). Muchos en `WAITING` sin nadie que los despierte → un señalador que nunca llegó. Muchos en `RUNNABLE` sin avanzar → falta CPU o starvation (módulo 15).

> **El paralelo en Node.** Node no expone "estados de hilo" porque tu código corre en un solo hilo, pero el event loop tiene su propia versión: tareas en la **cola de callbacks** (runnable), la que se ejecuta ahora (running), y las **microtasks** (`Promise`) que corren antes de la próxima vuelta. Una `Promise` pendiente es el equivalente a *waiting*; el event loop bloqueado por un `while(true)` síncrono es el equivalente a un hilo que nunca suelta el core (y por eso congela todo: módulo 18).

La frase mental: **un hilo vive entre runnable, running, blocked y waiting. Dónde se acumulan los hilos en un thread dump es el primer diagnóstico de un sistema trabado: BLOCKED → lock; WAITING → señal que no llegó; RUNNABLE sin avanzar → CPU/starvation.**

**Ejercicios 4**
4.1 🔁 Nombrá los estados típicos del ciclo de vida de un hilo.
4.2 🧠 Diferenciá **blocked** de **waiting**: ¿qué espera cada uno?
4.3 🧠 En un thread dump ves 200 hilos en `BLOCKED` sobre el mismo lock. ¿Qué dos patologías sospechás y cómo las distinguirías?

---

## Etapa 2 — Por qué duele: el estado compartido

## Módulo 5 — Race condition (el corazón del tema)

**Teoría.** Una **race condition** ocurre cuando **dos tareas tocan estado compartido sin coordinación y el resultado depende de quién llega primero** — así que a veces sale bien y a veces sale mal, de forma no determinista. Es **el** problema que todas las primitivas de los próximos módulos vienen a resolver.

El ejemplo canónico: dos hilos hacen `contador++`. Parece atómico, pero son **tres pasos** (leer, sumar, escribir):

```
Hilo A: lee contador (=5)
Hilo B: lee contador (=5)   ← ambos leyeron 5
Hilo A: suma → 6, escribe 6
Hilo B: suma → 6, escribe 6  ← perdimos una suma: debería ser 7
```

**Hay dos sabores, y mezclarlos es lo que confunde a la gente.**

**a) Data race (memoria compartida).** Dos hilos escriben la misma posición de memoria a la vez. Es el clásico de Go/Java/C++:

```go
// Go: DATA RACE — corré con `go run -race` y lo detecta
var contador int
var wg sync.WaitGroup
for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); contador++ }()  // 1000 goroutines pisándose
}
wg.Wait()
fmt.Println(contador)  // casi nunca da 1000
```

**b) Race lógica / check-then-act (¡también pasa en Node!).** Acá está la sorpresa para quien cree que "Node es single-thread, no tengo races". El event loop **no interrumpe** una función síncrona a la mitad — pero **sí** puede intercalar en cada `await`. Si hacés *chequeá-y-actuá* con un `await` en el medio, dos requests se cuelan:

```ts
// Node: RACE LÓGICA — dos requests concurrentes pueden reservar el MISMO asiento
async function reservar(asiento: string, usuario: string) {
  const libre = await asientoEstaLibre(asiento)   // (1) chequea  ← cede el control acá
  if (libre) {
    await marcarOcupado(asiento, usuario)          // (2) actúa
  }
}
// Request A en (1) ve libre → cede en el await.
// Request B en (1) TAMBIÉN ve libre (A todavía no marcó).
// Las dos marcan ocupado. Doble reserva.
```

Esto es una race real **sin un solo hilo extra**: el `await` es el punto donde el event loop le da el turno a otra tarea. La lección de entrevista: *la concurrencia, no el paralelismo, es lo que causa races; y Node tiene concurrencia.*

> **El razonamiento de 4 capas — la doble reserva de asientos:**
> 1. **Qué se rompe:** dos usuarios reservan el mismo asiento; se viola una invariante de negocio (un asiento, un dueño).
> 2. **Por qué a esta escala/tráfico:** es invisible con tráfico bajo (las requests no se solapan). Aparece bajo **concurrencia alta sobre el mismo recurso** — un lanzamiento, un asiento popular, dos clicks rápidos. La ventana es el tiempo entre el *check* y el *act*.
> 3. **Control de corto plazo:** mover el check-then-act a una **operación atómica** — un `UPDATE ... WHERE estado='libre'` condicional en la DB (la DB serializa), o un lock. Frena el blast radius ya mismo.
> 4. **Diseño de largo plazo:** modelar la reserva como una transición de estado con **garantía de unicidad** (constraint único, optimistic locking con versión, o una cola que serializa por asiento). Que la invariante la imponga el almacén, no el código de aplicación.

La frase mental: **una race condition es un resultado que depende del orden de llegada sobre estado compartido. No necesitás hilos: te alcanza con un `await` en medio de un "chequeá y actuá". El arreglo es siempre el mismo: volver atómica la operación o coordinar el acceso.**

**Ejercicios 5**
5.1 🔁 ¿Qué dos ingredientes necesita una race condition para existir?
5.2 🧠 "En Node no hay races porque es single-thread." Refutalo con un ejemplo concreto.
5.3 🧠 ¿Por qué `contador++` no es atómico aunque sea una sola línea de código?
5.4 🧠 Aplicá las 4 capas a un "like" que a veces cuenta de menos cuando un posteo se vuelve viral.

---

## Módulo 6 — Operaciones atómicas

**Teoría.** Una **operación atómica** ocurre **entera o no ocurre** — nadie la puede observar a la mitad ni intercalarse en el medio. Es el ladrillo con el que se construye todo lo demás: si `contador++` fuera atómico, no habría race en el módulo 5.

La primitiva de hardware que las hace posibles se llama **CAS (Compare-And-Swap)**: *"escribí este valor nuevo **solo si** el valor actual sigue siendo el que leí"*. Si otro lo cambió en el medio, CAS falla y reintentás. Sobre CAS se construyen los contadores atómicos, los locks, y las estructuras *lock-free*.

```go
// Go: contador atómico — NO hay data race, sin lock explícito
var contador atomic.Int64
for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() { defer wg.Done(); contador.Add(1) }()  // Add es atómico (CAS por debajo)
}
wg.Wait()
fmt.Println(contador.Load())  // siempre 1000
```

```ts
// Node: Atomics sobre memoria compartida entre worker_threads
const sab = new SharedArrayBuffer(8)
const contador = new Int32Array(sab)
// en cada worker:
Atomics.add(contador, 0, 1)        // suma atómica, visible entre hilos
Atomics.load(contador, 0)          // lectura atómica
// Atomics.compareExchange(...)     // el CAS crudo, por si lo necesitás
```

**El matiz clave para entrevistas: atómico ≠ libre de races a nivel lógico.** Una operación *individual* puede ser atómica y aun así tu lógica tener una race, si la **secuencia** de operaciones no lo es. El check-then-act del módulo 5 puede usar dos operaciones atómicas (un `load` atómico y un `store` atómico) y seguir roto, porque **el conjunto** no es atómico. Por eso existen los locks (módulo 8): para volver atómica una *secuencia*, no una sola instrucción.

La frase mental: **atómico = todo-o-nada, indivisible. CAS es la primitiva de hardware que lo hace posible. Pero atomizar una instrucción no atomiza tu lógica: si la secuencia de pasos no es atómica, seguís teniendo una race.**

### Atomicidad ≠ visibilidad (la otra mitad del problema)

Hay un segundo problema, distinto de la atomicidad, que en las entrevistas mid/senior se pregunta muchísimo y casi nadie lo separa bien: **la visibilidad**. No alcanza con que una escritura sea atómica; el otro hilo tiene que **verla**.

¿Por qué no la vería? Porque, por performance, cada core tiene **cachés** y tanto el compilador como la CPU pueden **reordenar** instrucciones. Sin una instrucción especial que lo impida, un hilo puede escribir una variable y otro seguir leyendo un valor **viejo** de su caché, indefinidamente:

```java
// Java: SIN 'volatile', este loop puede no terminar NUNCA aunque otro hilo ponga listo=true
boolean listo = false;           // el hilo lector puede cachear 'false' para siempre
// hilo A: listo = true;
// hilo B: while (!listo) { }     // quizá nunca ve el cambio → loop infinito
```

La solución son las **barreras de memoria** (memory barriers), que el lenguaje te da empaquetadas:
- En **Java**, la palabra clave **`volatile`** garantiza que cada lectura/escritura va a memoria principal (no a la caché) y prohíbe el reordenamiento alrededor. Es la base del modelo **happens-before**: "si A escribió antes de B en el orden del programa y hay una barrera, B ve lo de A".
- En **Go**, no hay `volatile`: el modelo de memoria dice que la visibilidad se garantiza usando los canales o `sync` (un `Mutex`/`atomic` **también** actúa de barrera, no solo de exclusión).
- En **Node**, las operaciones de **`Atomics`** sobre un `SharedArrayBuffer` no son solo "atómicas": también son la **barrera de visibilidad** entre `worker_threads`. Por eso para coordinar workers usás `Atomics`, no un `Int32Array` pelado.

El insight que cierra el módulo: **un mutex o un atómico hacen dos trabajos a la vez** — garantizan exclusión/atomicidad **y** publican los cambios para que otros los vean. La gente recuerda lo primero y olvida lo segundo.

La frase mental: **atomicidad es "todo o nada"; visibilidad es "que el otro hilo lo vea". Son problemas distintos: sin una barrera de memoria (`volatile` en Java, `Atomics`/`sync` en Node/Go) una escritura atómica puede quedar invisible en la caché de otro core. Los locks y los atómicos resuelven los dos a la vez.**

**Ejercicios 6**
6.1 🔁 ¿Qué significa que una operación sea atómica?
6.2 🧠 ¿Qué hace CAS y por qué es la base de los contadores atómicos y los locks?
6.3 🧠 Tenés `load` atómico y `store` atómico, pero tu check-then-act sigue dando resultados incorrectos. ¿Por qué? ¿Qué necesitás?
6.4 🧠 Un hilo hace `while(!listo){}` y otro setea `listo=true`, pero el primero nunca termina aunque la escritura sea atómica. ¿Qué problema es —distinto de la atomicidad— y cómo se arregla?

---

## Módulo 7 — Inmutabilidad

**Teoría.** La forma más fácil de proteger estado compartido es **que no se pueda mutar**. Si un dato nunca cambia después de creado, **mil hilos pueden leerlo a la vez sin ningún lock**: no hay race posible, porque no hay escritura que competir.

Esta es una de las ideas más poderosas y más subestimadas del tema. En vez de coordinar el acceso a estado mutable (locks, semáforos, toda la maquinaria de la Etapa 3), **eliminás el problema de raíz**: no hay estado mutable que coordinar.

```ts
// En vez de mutar un objeto compartido (race-prone)...
config.timeout = 5000   // ¿quién más lo está leyendo justo ahora?

// ...reemplazás por una versión nueva inmutable (sin race)
const nuevaConfig = { ...config, timeout: 5000 }   // los lectores viejos siguen con la suya
publicar(nuevaConfig)   // un único swap de referencia
```

Dónde se apoya esto en lo que ya sabés:
- **React** ya te entrenó: el estado es inmutable, creás copias nuevas en vez de mutar. Misma idea, otro contexto.
- **Value Objects de DDD** ([ddd.md]) son inmutables por diseño, justo por esto.
- **Event sourcing** ([event-driven.md]): los eventos son hechos inmutables del pasado; nunca se editan, solo se agregan.
- **Estructuras persistentes** (Immutable.js, los `record`/`readonly` de TS) te dan "copias" baratas que comparten estructura.

El trade-off honesto (siempre hay uno): inmutabilidad cambia **CPU/memoria por simplicidad**. Crear copias en vez de mutar in-place tiene costo. Para datos chicos o que cambian poco, es una ganga. Para un buffer de millones de elementos que muta en un loop caliente, mutar con un lock puede ser más barato. Como todo en concurrencia: es un trade-off, no un dogma.

La frase mental: **el estado más fácil de compartir es el que nadie puede mutar. La inmutabilidad no coordina el acceso al problema: lo elimina. Pagás copias a cambio de borrar una clase entera de bugs.**

**Ejercicios 7**
7.1 🔁 ¿Por qué un dato inmutable no necesita locks para ser compartido entre hilos?
7.2 🧠 ¿Qué trade-off concreto aceptás al elegir inmutabilidad sobre mutar con un lock?
7.3 🧠 Conectá la inmutabilidad con algo que ya usás en React y con los Value Objects de DDD: ¿qué problema de concurrencia evitan los tres por la misma razón?

---

## Etapa 3 — Primitivas de sincronización

## Módulo 8 — Mutex

**Teoría.** Un **mutex** (de *mutual exclusion*) garantiza que **un solo hilo a la vez** entre en una **sección crítica** (el pedacito de código que toca el estado compartido). Quien llega primero entra; el resto **espera** (estado *blocked*, módulo 4) hasta que el dueño lo libere.

Es la herramienta directa contra la race del módulo 5: si el check-then-act está dentro de un mutex, ningún otro hilo puede colarse entre el *check* y el *act*.

```go
// Go: mutex protege el contador — sin data race, sin Atomics
var mu sync.Mutex
var contador int
for i := 0; i < 1000; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        mu.Lock()         // entra a la sección crítica (o espera su turno)
        contador++        // nadie más puede estar acá adentro
        mu.Unlock()       // libera; despierta a uno de los que esperan
    }()
}
wg.Wait()  // contador == 1000, garantizado
```

```ts
// Node: NO hay mutex nativo entre tareas async, pero se modela con una promesa-cadena.
// (En el laboratorio lo construimos completo.)
class Mutex {
  private cola: Promise<void> = Promise.resolve()
  async runExclusive<T>(fn: () => Promise<T>): Promise<T> {
    const anterior = this.cola
    let liberar!: () => void
    this.cola = new Promise(res => (liberar = res))   // el próximo espera a este
    await anterior                                     // espero mi turno
    try { return await fn() } finally { liberar() }    // libero al siguiente
  }
}
```

**El costo que tenés que nombrar en una entrevista: contención.** Un mutex **serializa** el acceso. Si muchos hilos pelean por el mismo mutex, se forman colas y el throughput cae — en el extremo, tu sistema de 8 cores se comporta como si tuviera uno solo, porque todos esperan el mismo lock. Eso es lo que arregla el locking fino (módulo 12). Y dos mutex mal ordenados son la receta del deadlock (módulo 13).

La frase mental: **un mutex deja entrar a uno y hace esperar al resto. Convierte una sección crítica en atómica. Su precio es la contención: cuanto más peleado el lock, más serializás y menos escalás.**

**Ejercicios 8**
8.1 🔁 ¿Qué garantiza un mutex sobre una sección crítica?
8.2 🧠 ¿Por qué un mutex muy peleado puede hacer que 8 cores rindan como 1?
8.3 🧠 ¿Cómo resuelve el mutex exactamente la race de la doble reserva del módulo 5?

---

## Módulo 9 — Semáforo

**Teoría.** Un **semáforo** es como un mutex **pero con capacidad**: permite que hasta **N hilos** usen un recurso a la vez. Mantiene un contador de "permisos": cada hilo que entra toma uno (`acquire`), cada uno que sale lo devuelve (`release`); cuando no quedan permisos, los siguientes esperan.

- Un semáforo de **N=1** es esencialmente un mutex (mutual exclusion).
- Un semáforo de **N>1** es para **limitar concurrencia**: "como mucho N a la vez".

El caso de uso estrella —y uno de los 5 builds del [laboratorio](concurrencia-practica.md)— es el **connection pool**: tenés 10 conexiones a la base de datos y 500 requests que las quieren. Un semáforo de 10 deja pasar 10 a la vez y encola al resto, en vez de abrir 500 conexiones y tumbar la base.

```go
// Go: un channel con buffer ES un semáforo idiomático
sem := make(chan struct{}, 10)        // 10 permisos
for _, req := range requests {
    sem <- struct{}{}                  // acquire: bloquea si ya hay 10 adentro
    go func(r Req) {
        defer func() { <-sem }()       // release al terminar
        consultarDB(r)                 // como mucho 10 de estas a la vez
    }(req)
}
```

```ts
// Node: semáforo para limitar llamadas concurrentes a una API (evitar 429)
class Semaforo {
  private permisos: number
  private cola: Array<() => void> = []
  constructor(n: number) { this.permisos = n }
  async acquire(): Promise<void> {
    if (this.permisos > 0) { this.permisos--; return }
    await new Promise<void>(res => this.cola.push(res))   // espero un permiso
  }
  release(): void {
    this.permisos++
    const siguiente = this.cola.shift()
    if (siguiente) { this.permisos--; siguiente() }
  }
}
```

> **4 capas — el connection pool sin límite (semáforo ausente):**
> 1. **Qué se rompe:** la base de datos se queda sin conexiones y empieza a rechazar; o la memoria del servidor explota con miles de conexiones abiertas.
> 2. **Por qué a esta escala/tráfico:** invisible con poco tráfico (nunca pasás del límite). Aparece en un **pico** — una promoción, un reintento masivo — cuando las requests concurrentes superan el cupo del recurso.
> 3. **Control de corto plazo:** un semáforo (pool) que acota a N conexiones y encola el resto; un timeout en el `acquire` para no esperar para siempre.
> 4. **Diseño de largo plazo:** pool con tamaño derivado de la capacidad real de la DB (no "lo más grande posible"), **backpressure** hacia el cliente cuando la cola se llena (módulo 19), y un *bulkhead* para aislar pools por tipo de carga.

La frase mental: **un semáforo es un mutex con cupo N: deja pasar a N y encola al resto. Es la herramienta para limitar concurrencia sobre un recurso finito — y un connection pool es exactamente eso.**

### Read-Write lock: el caso "muchos lectores, pocos escritores"

Un mutex es demasiado estricto para un patrón muy común: un dato que **se lee muchísimo y se escribe poco** (una config, una cache en memoria, una tabla de rutas). Con un mutex, dos *lecturas* simultáneas se serializan sin razón — leer no rompe nada, ¿por qué esperar?

El **read-write lock** (RWMutex) lo resuelve con **dos modos**:
- **Read lock (compartido):** **muchos** lectores a la vez. Mientras solo se lee, nadie espera.
- **Write lock (exclusivo):** **un** escritor solo, y mientras escribe, **ningún** lector entra.

```go
// Go: sync.RWMutex — N lectores en paralelo, escritor exclusivo
var rw sync.RWMutex
var config map[string]string

func leer(k string) string {        // camino caliente: muchos en paralelo
    rw.RLock(); defer rw.RUnlock()
    return config[k]
}
func actualizar(c map[string]string) {  // raro: exclusivo
    rw.Lock(); defer rw.Unlock()
    config = c
}
```

Conecta con dos cosas que ya viste: es el complemento del módulo 7 (si el dato fuera 100% inmutable no necesitarías ni RLock; el RW lock es para cuando **a veces** se escribe), y trae su propia patología — **writer starvation**: si entran lectores sin parar, el escritor puede no conseguir nunca el lock exclusivo (módulo 15). Por eso muchos RW locks priorizan al escritor que está esperando.

> En Java es `ReentrantReadWriteLock` (o `StampedLock`, más moderno, con *optimistic reads*). En Node, al ser single-thread, rara vez lo necesitás; el equivalente conceptual aparece a nivel de **base de datos** (locks de lectura/escritura, y el MVCC del recuadro siguiente).

La frase mental: **un read-write lock deja leer a muchos a la vez pero escribir a uno solo (exclusivo). Es la primitiva para "muchas lecturas, pocas escrituras". Cuidado con el writer starvation: lectores infinitos pueden dejar al escritor afuera.**

**Ejercicios 9**
9.1 🔁 ¿En qué se diferencia un semáforo de un mutex?
9.2 🧠 ¿Por qué un semáforo de N=1 equivale a un mutex?
9.3 🧠 Tenés un pool de 10 conexiones y llegan 500 requests. ¿Qué hace el semáforo y por qué es mejor que abrir 500 conexiones?
9.4 🧠 Tenés una config que se lee miles de veces por segundo y se actualiza una vez por hora. ¿Por qué un mutex común desperdicia rendimiento y qué primitiva lo arregla? ¿Qué patología nueva introduce?

---

## Módulo 10 — Variable de condición

**Teoría.** Una **variable de condición** (condition variable) deja que un hilo **duerma hasta que otro le avise** que cierta condición ya es verdadera. Resuelve un problema concreto: *esperar a que pase algo* sin quemar CPU.

La alternativa ingenua es **busy-waiting** (espera activa): `while (!hayDatos) {}` — un loop que pregunta sin parar y desperdicia un core entero girando en falso. La variable de condición es la versión eficiente: el hilo se va a *waiting* (módulo 4), libera el core, y el sistema lo **despierta** solo cuando otro hilo señala (`signal`/`notify`).

El patrón siempre es el mismo (y tiene una trampa famosa):

```go
// Go: consumidor espera a que haya un item; productor lo despierta
var mu sync.Mutex
cond := sync.NewCond(&mu)
cola := []int{}

// consumidor
mu.Lock()
for len(cola) == 0 {        // ⚠️ 'for', NO 'if' — ver abajo
    cond.Wait()             // libera mu y duerme; al despertar, re-toma mu
}
item := cola[0]; cola = cola[1:]
mu.Unlock()

// productor
mu.Lock()
cola = append(cola, 42)
cond.Signal()               // despierta a UN consumidor dormido
mu.Unlock()
```

⚠️ **La trampa del `for` en vez de `if` (pregunta de entrevista clásica).** Tenés que re-chequear la condición en un **bucle**, no una sola vez, por dos razones: (a) los **spurious wakeups** (el SO puede despertar el hilo sin que nadie señaló) y (b) que entre que te despertaron y que re-tomaste el lock, **otro consumidor ya se llevó el item**. Si usás `if`, asumís que la condición sigue siendo verdadera y leés de una cola vacía. Con `for`, re-verificás y, si no, volvés a dormir.

> **En Node no hay condition variables** porque el modelo async las reemplaza: en vez de "dormir hasta una señal", tenés una `Promise` que **resolvés** cuando la condición se cumple. El productor-consumidor del [laboratorio](concurrencia-practica.md) usa justo eso: el consumidor hace `await` de una promesa que el productor resuelve al encolar.

La frase mental: **una variable de condición es "dormí hasta que te avise", sin quemar CPU en un busy-wait. Siempre re-chequeás la condición en un `while`/`for` (no `if`) por los spurious wakeups y por las carreras al re-tomar el lock.**

**Ejercicios 10**
10.1 🔁 ¿Qué problema resuelve una variable de condición frente al busy-waiting?
10.2 🧠 ¿Por qué la condición se re-chequea en un `while`/`for` y no en un `if`? Dá las dos razones.
10.3 🧠 ¿Cuál es el equivalente del "dormir hasta una señal" en el modelo async de Node?

---

## Módulo 11 — Lock reentrante

**Teoría.** Un **lock reentrante** (o recursivo) permite que **el mismo hilo que ya tiene el lock lo vuelva a tomar** sin trabarse a sí mismo. Lleva una cuenta: incrementa en cada `lock()` del mismo dueño y solo libera de verdad cuando la cuenta vuelve a cero.

¿Por qué existe? Por un auto-deadlock sutil. Imaginá un método sincronizado que llama a **otro** método sincronizado del mismo objeto:

```java
// Java: ReentrantLock — sin reentrancia, esto se auto-trabaría
class Cuenta {
  private final ReentrantLock lock = new ReentrantLock();
  void transferir() {
    lock.lock();
    try { depositar(); }       // depositar() también toma el MISMO lock...
    finally { lock.unlock(); }
  }
  void depositar() {
    lock.lock();               // ...si NO fuera reentrante, acá el hilo se esperaría
    try { /* ... */ }          // a sí mismo para siempre: auto-deadlock
    finally { lock.unlock(); }
  }
}
```

Sin reentrancia, `transferir()` tiene el lock, llama a `depositar()`, que intenta tomar el mismo lock, que **nunca se va a liberar porque el que lo tiene es el mismo hilo que ahora espera**. Deadlock de un solo hilo consigo mismo. El lock reentrante lo evita: reconoce al dueño y lo deja reentrar.

> **Mutex de Go NO es reentrante** — a propósito. Si un goroutine hace `mu.Lock()` dos veces, se traba. La filosofía de Go es que necesitar reentrancia suele ser una **señal de diseño confuso** (secciones críticas que se llaman entre sí); preferí reestructurar para que la sección crítica no llame a otra que toma el mismo lock. Java y C# sí ofrecen reentrancia. Saber **que es una decisión de diseño del lenguaje, con trade-offs**, es lo que suena senior.

La frase mental: **un lock reentrante deja que el dueño actual lo vuelva a tomar (cuenta interna), evitando el auto-deadlock cuando una sección crítica llama a otra. Que un lock sea o no reentrante es una decisión de diseño: Go dice "no, reestructurá"; Java dice "sí".**

**Ejercicios 11**
11.1 🔁 ¿Qué permite un lock reentrante que un lock común no?
11.2 🧠 Describí el auto-deadlock que la reentrancia previene.
11.3 🧠 El mutex de Go no es reentrante. ¿Qué te está diciendo el lenguaje sobre cómo deberías diseñar tus secciones críticas?

---

## Módulo 12 — Locking grueso vs fino

**Teoría.** Cuando protegés estado con locks, hay una decisión de diseño con un trade-off directo entre **simplicidad** y **escalabilidad**:

- **Locking grueso (coarse-grained):** un lock grande protege mucho estado. **Simple** de razonar (un solo lock, imposible olvidarse cuál tomar) pero **serializa todo**: aunque dos hilos toquen datos no relacionados, esperan el mismo lock.
- **Locking fino (fine-grained):** muchos locks chicos, cada uno protege un pedacito. **Escala** mejor (hilos que tocan datos distintos avanzan en paralelo) pero es **mucho más difícil de razonar** y abre la puerta a deadlocks (tomar dos locks en orden distinto).

El ejemplo clásico es una tabla hash:

```go
// GRUESO: un lock para todo el mapa. Dos escrituras a claves distintas se serializan igual.
type MapaGrueso struct { mu sync.Mutex; m map[string]int }

// FINO: el mapa se parte en N "shards", cada uno con su lock.
// Escrituras a claves que caen en shards distintos NO se pisan → más throughput.
type MapaFino struct {
    shards [16]struct {
        mu sync.Mutex
        m  map[string]int
    }
}
func (mf *MapaFino) shard(k string) int { return int(hash(k)) % 16 }  // elige el shard
```

Esta idea —partir el estado para repartir la contención— es **sharding**, y es el mismo principio que vas a ver en bases de datos ([postgresql.md], [nosql.md]) y en `ConcurrentHashMap` de Java. La progresión mental: empezá **grueso** (correcto y simple); medí; si el lock es el cuello de botella, **refiná** solo esa parte caliente. No arranques fino "por las dudas" — pagás complejidad y riesgo de deadlock sin saber si los necesitás.

La frase mental: **grueso = un lock grande, simple pero serializa todo; fino = muchos locks chicos, escala pero es difícil de razonar y arriesga deadlocks. Empezá grueso, medí, y refiná solo el punto caliente.**

### Optimistic vs pessimistic locking (el puente a tu base de datos)

Hay otro eje de decisión, distinto de grueso/fino, que vas a usar **todos los días** como backend con Postgres ([postgresql.md]) y que cae seguro en entrevistas: **¿asumís que va a haber conflicto, o asumís que no?**

- **Pessimistic locking (pesimista):** *"seguro alguien más lo va a tocar, lo bloqueo antes"*. Tomás un lock **antes** de leer/modificar (`SELECT ... FOR UPDATE`). Nadie más toca esa fila hasta que terminás. Correcto y simple, pero **serializa** y puede causar deadlocks y contención (todo lo de este módulo). Conviene cuando el conflicto es **frecuente**.
- **Optimistic locking (optimista):** *"probablemente nadie más lo toque, lo verifico al final"*. No bloqueás: leés el dato con una **versión** (o timestamp), hacés tu cambio, y al guardar verificás que la versión **no cambió** (`UPDATE ... SET ..., version = version+1 WHERE id = $1 AND version = $2`). Si otro lo modificó en el medio, afectás 0 filas → **detectás el conflicto y reintentás**. Sin locks, escala muchísimo mejor. Conviene cuando el conflicto es **raro**.

Fijate que el optimistic locking **es CAS aplicado a una fila** (módulo 6): "escribí solo si la versión sigue siendo la que leí". Y que esto es lo que estaba detrás del "control de corto plazo" de la doble reserva (módulo 5, capa 3): el `WHERE dueno IS NULL` es exactamente un check optimista que la DB resuelve atómicamente.

> **MVCC (Multi-Version Concurrency Control)**, el motor de Postgres: en vez de bloquear lectores contra escritores, mantiene **varias versiones** de cada fila, así los lectores ven un snapshot consistente **sin esperar** a los escritores (y viceversa). Es la razón por la que en Postgres "los lectores no bloquean a los escritores ni al revés". Es el read-write lock del módulo 9 llevado al extremo: nadie espera a nadie para leer. Detalle en [postgresql.md].

La frase mental: **pessimistic = bloqueo antes, asumiendo conflicto (bueno si choca seguido); optimistic = verifico la versión al final y reintento si cambió, asumiendo que no choca (escala mejor si el conflicto es raro). Optimistic locking es CAS sobre una fila; MVCC es cómo Postgres evita que lectores y escritores se bloqueen.**

**Ejercicios 12**
12.1 🔁 ¿Qué ganás y qué perdés al pasar de locking grueso a fino?
12.2 🧠 ¿Por qué partir un mapa en shards con un lock cada uno mejora el throughput? ¿Qué patrón de bases de datos es el mismo?
12.3 🧠 ¿Por qué la recomendación es "empezá grueso y refiná", en vez de arrancar fino?
12.4 🧠 Dos usuarios editan el mismo documento. ¿Cuándo elegís optimistic locking y cuándo pessimistic? ¿Por qué el optimistic es "CAS sobre una fila"?

---

## Etapa 4 — Patologías (lo que sale mal)

## Módulo 13 — Deadlock

**Teoría.** Un **deadlock** es una **espera circular**: el hilo A tiene el lock 1 y quiere el 2; el hilo B tiene el 2 y quiere el 1. Ninguno suelta lo que tiene, ninguno consigue lo que falta, **nadie avanza nunca**.

La herramienta de entrevista que tenés que tener memorizada son las **4 condiciones de Coffman** — un deadlock necesita las **cuatro** a la vez, así que **romper una sola lo previene**:

1. **Exclusión mutua:** el recurso no se puede compartir (un lock es de uno).
2. **Hold and wait:** un hilo retiene un recurso mientras espera otro.
3. **No preemption:** no se le puede quitar el recurso a la fuerza; lo suelta voluntariamente.
4. **Espera circular:** existe un ciclo A→B→A en quién espera a quién.

```go
// Go: deadlock a propósito — dos goroutines, dos mutex, orden opuesto
var m1, m2 sync.Mutex
go func() { m1.Lock(); time.Sleep(time.Millisecond); m2.Lock(); /* ... */ }()  // 1 luego 2
go func() { m2.Lock(); time.Sleep(time.Millisecond); m1.Lock(); /* ... */ }()  // 2 luego 1
// A tiene m1 y espera m2; B tiene m2 y espera m1. Colgado. (Go lo detecta y panic-ea
// si TODO el programa se traba; en un servidor real, solo se cuelgan esas requests.)
```

> **4 capas — un deadlock en producción:**
> 1. **Qué se rompe:** un subconjunto de requests se cuelga para siempre; los hilos quedan en `BLOCKED` (módulo 4) y se van agotando del pool hasta que el servicio deja de responder.
> 2. **Por qué a esta escala/tráfico:** suele ser **latente** y aparecer bajo concurrencia alta o con cierto **orden de operaciones** poco frecuente (transferir de la cuenta A a la B mientras otra transfiere de B a A — el orden de locks se invierte). Por eso pasa los tests y explota en prod.
> 3. **Control de corto plazo:** **timeouts** en la adquisición de locks (rompe *no preemption*: si no lo conseguís en X ms, soltás todo y reintentás) + reiniciar/recortar el pool afectado para liberar el blast radius.
> 4. **Diseño de largo plazo:** imponer un **orden total de adquisición de locks** (siempre tomar el de menor id primero — rompe *espera circular*); o evitar *hold-and-wait* tomando todos los locks de una; o rediseñar para no necesitar dos locks (una cola que serializa por cuenta).

La prevención más usada en la práctica es el **orden total**: si todos los hilos siempre toman los locks en el mismo orden global, no puede haber ciclo. Es exactamente lo que vas a arreglar en el [laboratorio](concurrencia-practica.md).

La frase mental: **deadlock = espera circular; necesita las 4 condiciones de Coffman a la vez, así que romper una lo previene. En la práctica: orden total de locks (rompe la espera circular) y timeouts (rompen el hold-and-wait).**

**Ejercicios 13**
13.1 🔁 Enumerá las 4 condiciones de Coffman.
13.2 🧠 ¿Por qué imponer un orden total de adquisición de locks elimina los deadlocks? ¿Qué condición rompe?
13.3 🧠 Un deadlock pasó todos los tests y apareció en prod en hora pico. Explicá por qué, con las 4 capas.

---

## Módulo 14 — Livelock

**Teoría.** Un **livelock** es más sutil y más raro que un deadlock: los hilos **no están bloqueados** —siguen ejecutándose, consumiendo CPU— pero **reaccionan tanto unos a otros que ninguno progresa**. Están ocupadísimos sin hacer trabajo real.

La analogía perfecta: dos personas en un pasillo angosto que se cruzan; uno se corre a la izquierda, el otro también; los dos a la derecha; los dos a la izquierda… se mueven sin parar pero nunca pasan. En código pasa cuando los hilos detectan un conflicto y **todos ceden a la vez**, vuelven a intentar a la vez, vuelven a chocar:

```
Hilo A: detecta que B necesita el recurso → lo suelta y reintenta
Hilo B: detecta que A necesita el recurso → lo suelta y reintenta
(los dos sueltan, los dos reintentan, los dos vuelven a detectar el conflicto... infinito)
```

La diferencia con deadlock, en una línea: en un **deadlock** los hilos están **quietos** (blocked, CPU al 0%); en un **livelock** están **frenéticos** (running, CPU al 100%) pero igual no avanzan. Un livelock puede ser *peor* de diagnosticar porque el sistema "parece vivo" (CPU alta, no hay hilos colgados) pero el throughput es cero.

**El arreglo** es el mismo truco que los protocolos de red: **aleatoriedad / backoff**. Si cada hilo espera un tiempo **aleatorio** antes de reintentar (en vez de todos a la vez), rompés la sincronía y alguno gana. Es la misma razón por la que el reintento con *jitter* evita el retry storm (módulo 19).

La frase mental: **livelock = hilos activos al 100% de CPU que reaccionan entre sí y nunca terminan, como dos personas trabadas en un pasillo. Se rompe con backoff aleatorio: que no todos reintenten en el mismo instante.**

**Ejercicios 14**
14.1 🔁 ¿En qué se diferencia un livelock de un deadlock respecto del uso de CPU?
14.2 🧠 ¿Por qué un livelock puede ser más difícil de diagnosticar que un deadlock?
14.3 🧠 ¿Por qué la aleatoriedad (backoff con jitter) rompe un livelock?

---

## Módulo 15 — Starvation

**Teoría.** La **starvation** (inanición) es cuando **un hilo nunca consigue el recurso o la CPU que necesita** porque otros se lo quedan sistemáticamente. El hilo no está roto ni bloqueado por un ciclo: simplemente **siempre pierde**.

Causas típicas:
- **Prioridades:** si el scheduler siempre favorece a los hilos de alta prioridad, los de baja prioridad pueden no correr nunca. Su primo famoso es la **priority inversion**: un hilo de **baja** prioridad tiene un lock que necesita uno de **alta** prioridad, así que el de alta queda bloqueado esperando al de baja — que a su vez ni corre porque hilos de prioridad media le ganan la CPU. Resultado: el de alta prioridad, invertido, espera a todos. El fix clásico es **priority inheritance**: mientras el hilo de baja prioridad tenga un lock que bloquea a uno de alta, **hereda temporalmente** la prioridad alta para soltarlo rápido. (Es el bug que casi hunde la misión Mars Pathfinder en 1997 — buena anécdota para una entrevista.)
- **Locks injustos (unfair):** un mutex que no garantiza orden FIFO puede dejar que los hilos "rápidos" se queden siempre con el lock y uno "lento" espere para siempre.
- **Diseño:** una cola LIFO donde, bajo carga, los ítems del fondo nunca salen.

La solución es la **fairness** (justicia): garantizar que todos progresen eventualmente. Muchos locks ofrecen un modo *fair* (FIFO: el que llegó primero, entra primero) a costa de algo de throughput. Las colas justas, el *aging* (subirle la prioridad a quien lleva mucho esperando) y el round-robin son las herramientas.

> **Starvation vs deadlock vs livelock (el cuadro que cierra la Etapa 4):**
> - **Deadlock:** nadie avanza, todos quietos (espera circular).
> - **Livelock:** nadie avanza, todos frenéticos (reaccionan entre sí).
> - **Starvation:** *algunos* avanzan, *otros* nunca (reparto injusto).

La frase mental: **starvation = un hilo que siempre pierde el reparto y nunca progresa, mientras otros sí. Es un problema de fairness: se arregla con colas FIFO, locks fair o aging del que más espera.**

**Ejercicios 15**
15.1 🔁 ¿Qué es starvation y en qué se diferencia de un deadlock?
15.2 🧠 Dá dos causas concretas de starvation.
15.3 🧠 Completá el cuadro deadlock / livelock / starvation en términos de "quién avanza" y "qué hace la CPU".

---

## Etapa 5 — Patrones a escala (system design)

## Módulo 16 — Thread pool

**Teoría.** Un **thread pool** es un conjunto **fijo** de hilos trabajadores que se **reutilizan** para ejecutar muchas tareas, en vez de crear un hilo nuevo por cada tarea. Las tareas entran a una **cola**; los workers libres las van tomando.

¿Por qué? Crear un hilo es **caro** (memoria de stack, llamada al SO, presión sobre el scheduler). Si abrís un hilo por request, bajo carga creás miles, el SO se ahoga haciendo *context switching* y te quedás sin memoria. El pool acota eso: **N workers fijos, costo de creación pagado una vez**, y un límite natural a cuánto trabajo corre en paralelo.

El parámetro que define todo es **el tamaño del pool**, y la respuesta de entrevista depende del tipo de carga:

- **CPU-bound** (cálculo puro): el tamaño óptimo ≈ **número de cores**. Más hilos que cores no hacen más trabajo, solo agregan context switching.
- **I/O-bound** (esperan red/disco): conviene **más hilos que cores**, porque mientras unos esperan I/O, otros usan la CPU. Cuántos más sale de la **Ley de Little**: para sostener X tareas concurrentes con latencia L, necesitás ≈ `throughput × latencia` en vuelo.

```go
// Go: worker pool idiomático con channels — N workers consumen de una cola
tareas := make(chan Tarea, 100)   // la cola (acotada: 100)
var wg sync.WaitGroup
for i := 0; i < runtime.NumCPU(); i++ {   // N workers ≈ cores (carga CPU-bound)
    wg.Add(1)
    go func() {
        defer wg.Done()
        for t := range tareas {    // cada worker toma tareas hasta que se cierre el channel
            procesar(t)
        }
    }()
}
for _, t := range listaDeTareas { tareas <- t }   // encolar (bloquea si está llena → backpressure)
close(tareas)
wg.Wait()
```

> **El paralelo en Node.** Node ya usa un thread pool por debajo: **libuv** tiene un pool (por defecto 4 hilos, configurable con `UV_THREADPOOL_SIZE`) para I/O de disco, DNS y crypto. Y para CPU-bound, vos armás un pool de **`worker_threads`** — exactamente el quinto build del [laboratorio](concurrencia-practica.md). Ver [nodejs.md] para el detalle del thread pool de libuv.

La frase mental: **un thread pool reusa N workers fijos sobre una cola de tareas, en vez de crear un hilo por tarea. El tamaño es la decisión clave: ≈ cores para CPU-bound, más para I/O-bound (Ley de Little). Crear hilos sin límite tumba el sistema.**

**Ejercicios 16**
16.1 🔁 ¿Qué problema concreto resuelve un thread pool frente a "un hilo por tarea"?
16.2 🧠 ¿Por qué para carga CPU-bound el tamaño óptimo ≈ cores, pero para I/O-bound conviene más?
16.3 🧠 ¿Qué thread pool ya usa Node por debajo y para qué tipo de operaciones?

---

## Módulo 17 — Producer-Consumer

**Teoría.** El patrón **productor-consumidor** separa **quién genera trabajo** (productores) de **quién lo procesa** (consumidores), con una **cola en el medio**. Es uno de los patrones más usados de todo el system design: desacopla la velocidad de producción de la de consumo.

La clave —y la pregunta de entrevista— está en **la cola del medio**:

- **Cola no acotada (unbounded):** los productores nunca esperan. Suena bien hasta que los consumidores no dan abasto: la cola **crece sin límite** y te quedás sin memoria (OOM). Es una bomba de tiempo.
- **Cola acotada (bounded):** tiene capacidad máxima. Cuando se llena, los productores **esperan** (o se les rechaza). Eso es **backpressure** (módulo 19): la cola le comunica al productor "frená". Casi siempre es lo que querés.

```go
// Go: productor-consumidor con channel ACOTADO (capacidad 10)
cola := make(chan Pedido, 10)

// productor: si la cola está llena, este 'cola <- p' BLOQUEA → backpressure natural
go func() {
    for _, p := range pedidos { cola <- p }
    close(cola)
}()

// consumidores: N goroutines drenando la misma cola
for i := 0; i < 3; i++ {
    go func() { for p := range cola { procesar(p) } }()
}
```

> Esta es la base de toda la arquitectura de **colas y mensajería**: una cola de mensajes (RabbitMQ, SQS, BullMQ sobre Redis) es un productor-consumidor distribuido y durable. Lo viste en [redis.md] (colas con BullMQ) y [event-driven.md] (eventos). Lo que cambia es que la cola vive fuera del proceso y sobrevive a caídas — pero el modelo mental es **este**.

> **4 capas — la cola no acotada que se come la memoria:**
> 1. **Qué se rompe:** el servicio muere por OOM (out of memory) porque la cola creció sin tope.
> 2. **Por qué a esta escala/tráfico:** funciona perfecto mientras consumo ≥ producción. Explota cuando los consumidores se vuelven más lentos (una dependencia degradada) o la producción pica: el desbalance se **acumula** en la cola.
> 3. **Control de corto plazo:** acotar la cola y aplicar una política al llenarse (bloquear el productor, o *drop* de los ítems menos importantes); alertar sobre la profundidad de la cola.
> 4. **Diseño de largo plazo:** backpressure de punta a punta (módulo 19), autoescalado de consumidores según la profundidad de la cola, y una DLQ (dead-letter queue) para lo que no se puede procesar.

La frase mental: **productor-consumidor desacopla generar de procesar con una cola en el medio. La cola lo es todo: acotada (con backpressure) protege la memoria; no acotada es una bomba de OOM esperando un consumidor lento.**

**Ejercicios 17**
17.1 🔁 ¿Qué desacopla el patrón productor-consumidor y qué hay siempre en el medio?
17.2 🧠 ¿Por qué una cola no acotada es peligrosa? ¿Qué falla concreta provoca?
17.3 🧠 ¿Qué relación hay entre una cola acotada y el backpressure?

---

## Módulo 18 — Blocking vs Non-blocking

**Teoría.** Es una distinción sobre **qué hace un hilo cuando una operación no puede completarse ya**:

- **Blocking:** el hilo **espera ahí parado** hasta que la operación termine. Simple de escribir y leer, pero **ese hilo no hace nada más** mientras espera. Si tenés un hilo por request y todos bloquean en I/O, agotás el pool con todos los hilos dormidos.
- **Non-blocking:** la operación **vuelve enseguida** (aunque no haya terminado) y el hilo **sigue con otra cosa**, chequeando después si ya está lista (vía callbacks, eventos, polling). Más eficiente con recursos, más difícil de razonar.

Este es **el corazón de por qué Node escala** con un solo hilo. Un servidor blocking tradicional dedica **un hilo por conexión**: 10.000 conexiones = 10.000 hilos, casi todos dormidos esperando I/O, quemando memoria. Node usa I/O **non-blocking**: un solo hilo dispara miles de operaciones de I/O y el event loop le avisa a cada una cuando su dato llegó. El hilo nunca se queda parado esperando — siempre tiene algo que hacer.

```ts
// Node: non-blocking — el hilo NO espera el archivo; sigue, y el callback corre cuando llega
import { readFile } from 'node:fs/promises'
const p = readFile('grande.txt')   // vuelve enseguida (una Promise pendiente)
hacerOtraCosa()                     // el hilo sigue trabajando mientras el disco lee
const datos = await p               // recién acá "esperamos" — pero cediendo a otras tareas
```

```ts
// El ANTI-patrón en Node: bloquear el único hilo congela TODO el servidor
import { readFileSync } from 'node:fs'
const datos = readFileSync('grande.txt')   // ⛔ el event loop se detiene acá: ninguna otra
                                            // request avanza hasta que el disco termine
```

⚠️ **La trampa específica de Node:** como hay **un solo hilo** para tu código, cualquier operación **síncrona pesada** (un `readFileSync`, un `JSON.parse` de 50MB, un loop de CPU) **bloquea a todos**. No es solo "esta request espera": es "**ninguna** request avanza". Por eso el CPU-bound va a `worker_threads` (módulo 16) y el I/O siempre a las APIs async.

La frase mental: **blocking = el hilo espera parado; non-blocking = el hilo sigue y chequea después. Node escala porque su I/O es non-blocking sobre un hilo. El reverso: cualquier código síncrono pesado en ese hilo congela el servidor entero.**

**Ejercicios 18**
18.1 🔁 ¿Qué hace un hilo en una operación blocking vs una non-blocking?
18.2 🧠 ¿Por qué un servidor non-blocking de un hilo puede manejar más conexiones que uno blocking de mil hilos?
18.3 🧠 ¿Por qué `JSON.parse` de un payload gigante es especialmente peligroso en Node y no tanto en un servidor multihilo?

---

## Módulo 19 — Backpressure (y el retry storm)

**Teoría.** **Backpressure** es el mecanismo por el cual un sistema **le comunica río arriba que vaya más despacio** cuando los consumidores no dan abasto. Sin él, un productor rápido + un consumidor lento = colas que crecen sin fin, memoria que explota, latencia que se dispara (módulo 17).

Las estrategias, de menos a más agresiva:
- **Bloquear / esperar:** el productor se frena hasta que haya lugar (la cola acotada del módulo 17). Preserva todo el trabajo pero propaga la lentitud hacia atrás.
- **Buffer + límite:** acumulás hasta un tope, después aplicás una de las siguientes.
- **Drop (descartar):** tirás ítems (los más viejos, los menos importantes). Pierde trabajo pero protege el sistema. Válido para métricas, logs, telemetría.
- **Sampling / shedding:** bajo carga, procesás solo una fracción (*load shedding*: rechazás requests con 503 antes de caer del todo).

```ts
// Node: los streams traen backpressure incorporado. .pipe() respeta el ritmo del destino:
origen.pipe(destino)   // si 'destino' es lento, 'origen' se pausa automáticamente

// A mano, .write() devuelve false cuando el buffer está lleno → señal de "frená":
if (!destino.write(chunk)) {
  origen.pause()                          // dejá de producir
  destino.once('drain', () => origen.resume())  // resumí cuando se vació
}
```

> **El retry storm — el caso de 4 capas que define la madurez de tu respuesta.** Es el ejemplo del marco al inicio del módulo, ahora completo:
> 1. **Qué se rompe:** una dependencia (una DB, un servicio) se pone lenta y, en vez de recuperarse, **colapsa del todo** — y arrastra a los que dependen de ella.
> 2. **Por qué a esta escala/tráfico:** la latencia sube → los timeouts de los clientes disparan → **cada cliente reintenta** → el número de requests en vuelo se **multiplica** justo cuando la dependencia está más débil → más carga → más latencia → más timeouts. **El reintento, que era un mecanismo de recuperación, se convirtió en un amplificador de carga.** Esto solo emerge a escala: con pocos clientes, los reintentos no mueven la aguja.
> 3. **Control de corto plazo:** **circuit breaker** (cortar las llamadas a la dependencia caída y fallar rápido, dándole aire a recuperarse) + **load shedding** (rechazar parte del tráfico con 503).
> 4. **Diseño de largo plazo:** reintentos con **exponential backoff + jitter** (que no todos reintenten al mismo tiempo — la misma aleatoriedad que rompe el livelock del módulo 14), **límite de reintentos**, **token bucket** para acotar el total de reintentos, e **idempotencia** (módulo 20) para que reintentar sea seguro.
>
> Respuesta débil: *"exponential backoff"*. Respuesta senior: **toda la cascada de las 4 capas.** Esa es la diferencia que busca un entrevistador.

La frase mental: **backpressure es el sistema diciendo "frená" cuando el consumidor no da abasto; sin él, las colas crecen hasta el OOM. Y un reintento sin backpressure (backoff + jitter + circuit breaker) deja de ser recuperación y se vuelve un amplificador de carga: el retry storm.**

**Ejercicios 19**
19.1 🔁 ¿Qué comunica el backpressure y en qué dirección?
19.2 🧠 Nombrá tres estrategias de backpressure y cuándo es aceptable hacer *drop*.
19.3 🧠 Contá el retry storm en las 4 capas, terminando en el cambio de diseño de largo plazo.
19.4 🧠 ¿Por qué el *jitter* aparece tanto en el retry storm como en el livelock? ¿Qué problema común resuelve?

---

## Etapa 6 — Correctitud en distribuido

## Módulo 20 — Idempotencia bajo concurrencia

**Teoría.** Una operación es **idempotente** si **ejecutarla varias veces produce el mismo resultado que ejecutarla una vez**. Bajo concurrencia y en sistemas distribuidos esto deja de ser un lujo y se vuelve **obligatorio**, porque los reintentos y los duplicados son **inevitables**.

¿Por qué inevitables? Porque en una red **no podés distinguir** "la operación falló" de "la operación funcionó pero se perdió la respuesta". Si tu pago devolvió timeout, ¿se cobró o no? El cliente, por las dudas, **reintenta**. Si la operación no es idempotente, cobrás dos veces. Esto se cruza directo con el módulo 19: los reintentos que evitan el retry storm **solo son seguros si la operación es idempotente**.

```ts
// NO idempotente: cada llamada suma. Un reintento cobra de más.
async function cobrar(monto: number) { await db.saldo.decrement(monto) }

// Idempotente con CLAVE DE IDEMPOTENCIA: la segunda vez con la misma clave no hace nada.
async function cobrar(monto: number, idemKey: string) {
  const insertado = await db.insertIfNotExists('cobros', { idemKey, monto })  // atómico
  if (!insertado) return                       // ya se procesó esta clave → no-op
  await db.saldo.decrement(monto)
}
```

La clave (de entrevista y de implementación) es la **idempotency key**: un identificador único de la *intención* (no del intento). El servidor registra qué claves ya procesó —de forma **atómica**, con un constraint único o un `INSERT ... ON CONFLICT`— y descarta los duplicados. Que el registro sea atómico es lo que lo hace seguro **bajo concurrencia**: dos reintentos en paralelo con la misma clave no pueden ambos "ganar" el insert.

> **"Exactly-once" es un mito; lo real es "at-least-once + idempotencia".** En un sistema distribuido no podés garantizar que un mensaje se procese exactamente una vez (entrega exactamente-una-vez es imposible en el caso general). Lo que hacés es entregar **al menos una vez** (con reintentos) y volver el procesamiento **idempotente**, de modo que los duplicados no cambien el resultado. El efecto observable es "como si hubiera sido una vez". Esto lo viste en [event-driven.md] (inbox/outbox, idempotency key en el sink) y en [redis.md] (rate limiting atómico). Decirlo así en una entrevista te marca como senior.

> **Conexión con todo lo anterior:** idempotencia es la que hace que las herramientas de los módulos previos sean **seguras de combinar**. Reintentos (módulo 19) + idempotencia = recuperación sin corrupción. Operaciones atómicas (módulo 6) = el mecanismo que hace atómico el registro de la clave. Es el cierre natural del recorrido: de proteger una variable en un proceso, a proteger una operación a través de la red.

La frase mental: **idempotente = ejecutarla N veces da el mismo resultado que ejecutarla una. En distribuido es obligatoria porque los reintentos son inevitables (no distinguís "falló" de "se perdió la respuesta"). Se logra con una idempotency key registrada atómicamente. "Exactly-once" no existe: existe "at-least-once + idempotencia".**

**Ejercicios 20**
20.1 🔁 Definí idempotencia y dá un ejemplo de operación que ya es idempotente por naturaleza.
20.2 🧠 ¿Por qué en un sistema distribuido los reintentos son inevitables? Conectalo con por qué la idempotencia se vuelve obligatoria.
20.3 🧠 ¿Por qué la idempotency key tiene que registrarse de forma **atómica**? ¿Qué pasa si no lo es, bajo dos reintentos concurrentes?
20.4 🧠 Explicá por qué "exactly-once" se reemplaza por "at-least-once + idempotencia".

---

## Cierre

## Módulo 21 — Cómo hablar de concurrencia en una entrevista

**Teoría.** Tenés los 20 conceptos. Esto es cómo **convertirlos en una respuesta que suene senior**. Una buena respuesta de system design sobre concurrencia tiene **tres cosas**, siempre:

1. **Una forma para el happy path.** Cómo fluye el sistema cuando todo anda: productores, una cola acotada, un pool de N workers, el camino de los datos. Concreto, no vago.
2. **Un modelo de falla.** Qué se rompe cuando sube la carga o cambia la forma del tráfico. Acá usás el **marco de 4 capas**: qué se rompe, por qué a esta escala, control de corto plazo, diseño de largo plazo. Nombrás races, deadlocks, colas que crecen, retry storms — y cómo los evitás.
3. **Un proceso de recuperación.** Qué pasa cuando *igual* falla: circuit breakers, backpressure, reintentos con backoff+jitter, idempotencia para que reintentar sea seguro, DLQ para lo irrecuperable.

> **El patrón de la respuesta débil vs la senior** (vale para casi cualquier tema de este módulo):
> - **Débil:** una solución de una línea. *"Usá un lock."* / *"Usá backoff."* / *"Agregá más workers."*
> - **Senior:** la cascada. *Qué se rompe → por qué a esta escala → cómo lo contengo ya → cómo lo rediseño para que no vuelva.* Y nombrar el **trade-off** de tu solución (un lock serializa; más workers no ayudan si la carga es I/O-bound; el drop pierde datos).

**El vocabulario que tenés que usar con soltura** (te posiciona en segundos): race condition, sección crítica, contención, atómico/CAS, deadlock (y Coffman), backpressure, idempotencia, at-least-once, blast radius, load shedding, circuit breaker, exponential backoff + jitter, thread pool sizing, CPU-bound vs I/O-bound.

**Y el reflejo más importante:** ante cualquier diseño concurrente, preguntate (y decí en voz alta) **"¿qué estado se comparte, y qué pasa si dos cosas lo tocan a la vez?"**. Esa pregunta sola te lleva a races, locks, atómicos e idempotencia — es decir, a todo este módulo.

La frase mental: **una respuesta senior tiene happy path + modelo de falla + recuperación. No des soluciones de una línea: contá la cascada (qué se rompe → por qué a esta escala → control inmediato → rediseño) y nombrá el trade-off. Y siempre arrancá por "¿qué estado se comparte?".**

**Ejercicios 21**
21.1 🔁 ¿Cuáles son las "tres cosas" de una buena respuesta de system design?
21.2 🧠 Tomá cualquier concepto del módulo (deadlock, backpressure, race) y armá una respuesta senior de 4 capas en voz alta.
21.3 🧠 ¿Por qué nombrar el trade-off de tu solución es parte de sonar senior, y no de "complicar la respuesta"?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Concurrencia es que varias tareas **progresan en la misma ventana de tiempo**, intercalándose, aunque no se ejecuten en el mismo instante.

**1.2** El único hilo atiende una request hasta que esta tiene que **esperar I/O** (DB, red); en esa espera, el event loop avanza otra. Como casi todo el tiempo de una request web es espera, miles "progresan a la vez" sin paralelismo.

**1.3** Porque concurrencia es **intercalado**: un solo core alterna entre tareas. Necesitás varios cores para *paralelismo* (correr a la vez), no para concurrencia (progresar solapadas).

**2.1** Concurrencia = intercalar tareas (diseño, sirve con 1 core); paralelismo = ejecutarlas literalmente a la vez (ejecución, requiere varios cores).

**2.2** Una app puede intercalar tareas en un core (concurrente, no paralela). Pero el paralelismo "útil" nace de haber **descompuesto el problema en tareas independientes** (diseño concurrente): si lo hiciste, agregar cores las corre a la vez; si no, más cores no ayudan.

**2.3** **Paralelismo**, porque es trabajo de CPU (no espera I/O que se pueda solapar). En Node lo sacás del event loop con **`worker_threads`** (o un proceso aparte), para no congelar el hilo principal.

**3.1** La **memoria** (heap del proceso): los hilos del mismo proceso comparten variables; los procesos tienen memoria aislada y se comunican por IPC.

**3.2** Porque **comparten memoria**: eso los hace livianos (no hay que copiar/aislar espacios) y rápidos de crear/cambiar, pero también significa que dos hilos pueden tocar la misma variable a la vez → races. La misma característica (memoria compartida) da la velocidad y el peligro.

**3.3** Para el servidor stateless: **`cluster`** (varios procesos, cada uno su event loop, sin compartir estado — escala HTTP a 8 cores). Para el cálculo con buffer compartido: **`worker_threads`** + `SharedArrayBuffer` (hilos que comparten memoria).

**4.1** New, Runnable, Running, Blocked, Waiting, Terminated.

**4.2** **Blocked**: espera un **lock** que tiene otro hilo. **Waiting**: duerme hasta que algo lo **despierte** (una señal, I/O, un timeout). Blocked es por contención de un recurso; waiting es por una condición/evento.

**4.3** **Deadlock** o **contención severa**. Los distinguís mirando si el lock **se libera alguna vez**: en contención, los hilos avanzan de a poco (el lock rota, aunque lento); en deadlock, **nadie** progresa nunca y, si mirás quién tiene qué, encontrás un **ciclo** de espera.

**5.1** (1) **Estado compartido** y (2) **acceso concurrente sin coordinación** (el resultado depende del orden de llegada).

**5.2** El event loop intercala en cada `await`. Un check-then-act como "¿asiento libre? → marcarlo ocupado" con un `await` entre el check y el act deja que dos requests vean "libre" antes de que cualquiera marque → doble reserva. Es una race sin hilos.

**5.3** Porque son **tres operaciones**: leer el valor, sumarle 1, escribir el resultado. Otro hilo puede leer el mismo valor viejo entre el read y el write, y una de las sumas se pierde.

**5.4** (1) **Qué se rompe:** el contador de likes cuenta de menos (se pierden incrementos). (2) **Por qué a esta escala:** con el posteo viral, miles de `like++` concurrentes leen el mismo valor y se pisan (la ventana read-modify-write se solapa). (3) **Corto plazo:** incremento atómico (`INCR` en Redis / `UPDATE ... SET likes = likes + 1` en la DB, que serializa). (4) **Largo plazo:** contadores distribuidos/sharded que se agregan (cada shard cuenta lo suyo, se suman), evitando el punto único de contención.

**6.1** Que ocurre **entera o no ocurre**: es indivisible, nadie la observa a medias ni se intercala en el medio.

**6.2** CAS (compare-and-swap) escribe un valor nuevo **solo si** el actual sigue siendo el que leíste; si cambió, falla y reintentás. Es la primitiva de hardware sobre la que se construyen contadores atómicos, locks y estructuras lock-free.

**6.3** Porque **atomizar cada operación no atomiza la secuencia**: entre tu `load` atómico y tu `store` atómico, otro hilo puede colarse y cambiar el valor. Necesitás que **el conjunto** sea atómico → un lock (o un único CAS sobre toda la transición).

**6.4** Es un problema de **visibilidad** (memory visibility), distinto de la atomicidad: por las **cachés** de cada core y el **reordenamiento** de compilador/CPU, el hilo lector puede no ver nunca la escritura del otro. Se arregla con una **barrera de memoria**: `volatile` en Java, `Atomics` sobre `SharedArrayBuffer` en Node, o usar `sync`/canales en Go (un mutex/atómico además de exclusión también publica el cambio).

**7.1** Porque no hay escrituras que competir: mil lecturas concurrentes de un dato que nunca cambia no pueden producir una race (las races necesitan al menos una escritura sobre estado compartido).

**7.2** Pagás **CPU y memoria** (crear copias nuevas en vez de mutar in-place) a cambio de **eliminar una clase entera de bugs** (no hay estado mutable que coordinar). Buen trato para datos chicos/que cambian poco; malo para buffers enormes en loops calientes.

**7.3** React (estado inmutable, copiás en vez de mutar) y los Value Objects de DDD (inmutables por diseño) evitan, igual que acá, que **un estado compartido sea mutado mientras otro lo lee/usa**. Los tres eliminan el problema en vez de coordinarlo.

**8.1** Que **un solo hilo a la vez** esté adentro: vuelve la sección crítica efectivamente atómica respecto de los otros hilos.

**8.2** Porque un mutex **serializa** el acceso: si todos los hilos pelean por el mismo lock, esperan en fila y solo uno trabaja a la vez. El paralelismo de los 8 cores se pierde en la cola del lock → rinde como 1.

**8.3** Metés el check-then-act (¿libre? → marcar) **dentro** del mutex. Como solo un hilo entra a la vez, ninguno puede colarse entre el check y el act: el segundo encuentra el asiento ya ocupado.

**9.1** El mutex deja pasar a **uno**; el semáforo deja pasar a **N** (tiene capacidad/permisos). El mutex es exclusión mutua; el semáforo es límite de concurrencia.

**9.2** Porque con un solo permiso, solo un hilo puede tener el recurso a la vez — exactamente la exclusión mutua de un mutex.

**9.3** El semáforo (de 10) deja pasar 10 requests a la vez y **encola las otras 490** hasta que se libere una conexión. Mejor que abrir 500 porque la base tiene un límite real de conexiones; pasarlo la tumba (o la hace rechazar todo). Acotás la concurrencia al recurso disponible.

**9.4** Un mutex común serializa **también las lecturas** entre sí, aunque leer no rompe nada: miles de lectores esperando en fila sin necesidad. Lo arregla un **read-write lock** (RWMutex): permite N lectores en paralelo y solo exige exclusividad al escribir (que es raro). La patología nueva es **writer starvation**: si los lectores no paran de entrar, el escritor puede no conseguir nunca el lock exclusivo (se mitiga priorizando al escritor que espera).

**10.1** Evita el **busy-waiting** (un loop que pregunta sin parar quemando CPU): el hilo se va a *waiting*, libera el core, y lo despiertan solo cuando la condición se cumple.

**10.2** (1) **Spurious wakeups**: el SO puede despertar el hilo sin que nadie haya señalado. (2) **Carrera al re-tomar el lock**: entre que te despiertan y re-adquirís el lock, otro hilo puede haber consumido lo que esperabas. Con `while`/`for` re-verificás y volvés a dormir si todavía no se cumple.

**10.3** Una **`Promise`**: el consumidor hace `await` de una promesa que el productor **resuelve** cuando la condición se cumple (p. ej., al encolar un item). No "duerme un hilo" — suspende una tarea async.

**11.1** Que **el mismo hilo** que ya tiene el lock lo **vuelva a tomar** sin trabarse (lleva una cuenta y solo libera de verdad al volver a cero).

**11.2** Una sección crítica que llama a otra que toma el **mismo** lock: sin reentrancia, el hilo —que ya tiene el lock— se quedaría esperando un lock que solo él podría liberar, pero está bloqueado esperándolo. Se traba consigo mismo para siempre.

**11.3** Que preferís **reestructurar** para que una sección crítica no llame a otra que toma el mismo lock. Necesitar reentrancia suele señalar secciones críticas anidadas/confusas; Go te empuja a un diseño más plano.

**12.1** **Ganás** escalabilidad (hilos que tocan datos distintos avanzan en paralelo en vez de pelear un lock único). **Perdés** simplicidad: más locks, más difícil de razonar, y riesgo de deadlock por orden de adquisición.

**12.2** Porque escrituras a claves de **shards distintos** toman **locks distintos** y no se serializan entre sí → más throughput. Es el mismo principio que el **sharding/particionado** de bases de datos (repartir el dato para repartir la contención).

**12.3** Porque el locking fino paga complejidad y riesgo de deadlock **siempre**, pero solo rinde si ese lock era realmente el cuello de botella. Empezás grueso (correcto y simple), **medís**, y refinás solo el punto caliente comprobado.

**12.4** **Optimistic** si el conflicto es **raro** (dos personas rara vez editan el mismo doc a la vez): leés con versión, guardás con `WHERE version=$leída`, y si afectó 0 filas detectás el choque y reintentás/avisás — sin bloquear, escala mejor. **Pessimistic** si el conflicto es **frecuente** o el reintento es caro: tomás `SELECT ... FOR UPDATE` y nadie más toca la fila mientras editás. Es "CAS sobre una fila" porque hace exactamente lo de compare-and-swap (módulo 6): escribe **solo si** el valor (la versión) sigue siendo el que leíste.

**13.1** (1) Exclusión mutua, (2) hold-and-wait, (3) no preemption, (4) espera circular.

**13.2** Porque elimina la **espera circular** (condición 4): si todos toman los locks en el mismo orden global, no puede formarse un ciclo A→B→A; alguien siempre puede avanzar.

**13.3** (1) **Qué se rompe:** un subconjunto de requests se cuelga; los hilos quedan BLOCKED y se agota el pool. (2) **Por qué a esta escala:** el deadlock necesita un **orden de operaciones poco frecuente** (A→B mientras otro hace B→A) + concurrencia alta; en tests (poca concurrencia, casos felices) no se da. (3) **Corto plazo:** timeouts en los locks + recortar/reiniciar el pool. (4) **Largo plazo:** orden total de adquisición de locks, o no necesitar dos locks.

**14.1** En deadlock los hilos están **quietos** (blocked, CPU ~0%); en livelock están **activos** (running, CPU ~100%) reaccionando entre sí, pero sin progresar.

**14.2** Porque el sistema "parece vivo": CPU alta, ningún hilo colgado en un lock, nada obviamente trabado. Pero el throughput es cero. Un deadlock es más fácil de ver (hilos parados en un ciclo claro).

**14.3** Porque el livelock nace de que **todos reaccionan en sincronía** (ceden y reintentan a la vez). El backoff aleatorio rompe la sincronía: con esperas distintas, alguno reintenta cuando el otro no, y avanza.

**15.1** Starvation es un hilo que **nunca consigue** el recurso/CPU que necesita porque otros se lo quedan siempre. A diferencia del deadlock (donde **nadie** avanza por un ciclo), acá **otros sí avanzan**; el problema es reparto injusto, no espera circular.

**15.2** Dos de: prioridades que siempre favorecen a otros hilos; locks *unfair* (sin orden FIFO) que dejan a uno esperando indefinidamente; estructuras LIFO donde el fondo nunca sale bajo carga.

**15.3** **Deadlock:** nadie avanza, CPU ~0% (quietos). **Livelock:** nadie avanza, CPU ~100% (frenéticos). **Starvation:** algunos avanzan, otros nunca; CPU ocupada por los que sí progresan.

**16.1** Evita el costo de **crear un hilo por tarea** (memoria de stack, llamadas al SO, context switching). Con N workers fijos reusados, pagás la creación una vez y ponés un techo a la concurrencia, evitando agotar memoria bajo carga.

**16.2** CPU-bound: más hilos que cores no hace más trabajo (no hay más CPU), solo agrega context switching → óptimo ≈ cores. I/O-bound: mientras unos esperan I/O, otros pueden usar la CPU, así que conviene más hilos que cores (cuántos, por la Ley de Little: throughput × latencia).

**16.3** El thread pool de **libuv** (por defecto 4, `UV_THREADPOOL_SIZE`), que Node usa para I/O de disco, DNS y operaciones de crypto.

**17.1** Desacopla **generar** trabajo (productores) de **procesarlo** (consumidores). En el medio hay siempre una **cola**.

**17.2** Porque si los consumidores no dan abasto, la cola **crece sin límite** y agota la memoria → **OOM** (el proceso muere). No tiene mecanismo para frenar al productor.

**17.3** Una cola acotada, al llenarse, **frena al productor** (o lo rechaza): eso **es** el backpressure — la cola comunicando "no hay lugar, esperá".

**18.1** Blocking: el hilo **espera parado** hasta que la operación termina (no hace nada más). Non-blocking: la operación vuelve enseguida y el hilo **sigue con otra cosa**, chequeando después.

**18.2** Porque el non-blocking no dedica un hilo por conexión esperando: un solo hilo dispara miles de I/O y atiende cada una cuando su dato llega. El blocking con mil hilos gasta memoria en mil stacks casi todos dormidos y satura el scheduler con context switches.

**18.3** Porque Node tiene **un solo hilo** para tu código: un `JSON.parse` gigante es trabajo **síncrono** que lo monopoliza, y **ninguna** otra request avanza mientras dura. Un servidor multihilo bloquea solo el hilo de esa request; los demás siguen.

**19.1** Comunica "**vayan más despacio**" **río arriba** (hacia los productores/clientes), cuando los consumidores no dan abasto.

**19.2** Tres de: bloquear/esperar (cola acotada), buffer con límite, **drop** (descartar ítems), sampling/load shedding (rechazar parte con 503). El *drop* es aceptable cuando perder datos no es crítico: métricas, logs, telemetría, frames de video.

**19.3** (1) **Qué se rompe:** una dependencia lenta colapsa del todo. (2) **Por qué a esta escala:** latencia ↑ → timeouts → todos reintentan → requests en vuelo se multiplican → más carga → más latencia; el reintento se volvió amplificador de carga (solo emerge a escala). (3) **Corto plazo:** circuit breaker + load shedding. (4) **Largo plazo:** backoff exponencial + jitter, límite y token bucket de reintentos, e idempotencia para que reintentar sea seguro.

**19.4** Porque ambos nacen de **acciones sincronizadas en el tiempo** (todos reintentan / todos ceden en el mismo instante). El jitter (aleatoriedad) **desincroniza**: rompe el patrón colectivo para que no todos golpeen a la vez.

**20.1** Idempotente = ejecutarla N veces da el mismo resultado que una vez. Ejemplo natural: `SET x = 5` (asignación absoluta), o `DELETE` de un id (borrar lo ya borrado no cambia nada). Contraejemplo: `x += 5`.

**20.2** Porque en una red no podés distinguir "**la operación falló**" de "**funcionó pero se perdió la respuesta**" (un timeout no te dice cuál). Ante la duda, el cliente reintenta. Como los reintentos van a pasar sí o sí, la operación **debe** ser idempotente o vas a duplicar efectos (cobrar dos veces).

**20.3** Porque dos reintentos concurrentes con la misma clave podrían **ambos** pasar el "¿ya existe?" antes de que cualquiera la registre, y los dos ejecutar el efecto. El registro atómico (constraint único / `INSERT ... ON CONFLICT`) garantiza que **solo uno gane**; el otro ve el conflicto y se vuelve no-op.

**20.4** Porque la entrega exactamente-una-vez es imposible en el caso general en sistemas distribuidos. Lo alcanzable es entregar **al menos una vez** (con reintentos) + procesar de forma **idempotente**, de modo que los duplicados no cambien el resultado: el efecto observable es "como si hubiera sido una sola vez".

**21.1** (1) Una **forma para el happy path**, (2) un **modelo de falla**, (3) un **proceso de recuperación**.

**21.2** Ejemplo (deadlock): (happy path) transferencias toman dos locks y completan; (modelo de falla, 4 capas) qué se rompe = requests colgadas; por qué a esta escala = orden de locks invertido bajo concurrencia; corto plazo = timeouts; largo plazo = orden total de locks; (recuperación) recortar el pool, reintentar las requests fallidas de forma idempotente.

**21.3** Porque toda solución de concurrencia **tiene** un costo (un lock serializa, más workers no ayudan en I/O-bound, el drop pierde datos). Nombrarlo muestra que entendés que estás **eligiendo** entre opciones con consecuencias, no aplicando una receta — que es justo lo que hace un senior.

---

> **Cierre.** La concurrencia es, en el fondo, **una sola pregunta repetida**: *¿qué estado se comparte, y qué pasa si dos cosas lo tocan a la vez?*. Todo lo demás —mutex, semáforos, deadlocks, backpressure, idempotencia— son respuestas a esa pregunta en distintos niveles: dentro de un proceso, entre hilos, a través de la red. Si en una entrevista arrancás por ahí y contás la **cascada de 4 capas** (qué se rompe → por qué a esta escala → control inmediato → rediseño) en vez de soluciones de una línea, ya sonás senior. Ahora pasá al [laboratorio práctico](concurrencia-practica.md) y **construí** los cinco: una race, un producer-consumer, un pool con semáforo, un deadlock a propósito y un thread pool con cola acotada. Entender es la mitad; romper y arreglar con tus manos es la otra.
