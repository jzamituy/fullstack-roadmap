# Concurrencia: laboratorio práctico

**Construí, rompé y arreglá: race condition, producer-consumer, pool con semáforo, deadlock y thread pool · Node + Go · 2026**

> Cómo usar esta guía: este es el complemento de mano del módulo de teoría [Concurrencia en system design](concurrencia.md). Ahí entendiste los 20 conceptos; acá **construís cinco cosas**, cada una con el mismo método: **(1)** el problema, **(2)** código que **falla a propósito**, **(3)** qué observás cuando lo corrés, **(4)** el fix, **(5)** qué demuestra. Romper el código con tus manos y verlo fallar fija el concepto como ninguna definición.

**Lo que asumimos.** Haber leído (o tener a mano) el [módulo de teoría](concurrencia.md). Node 20+ instalado. Para los ejemplos de Go, tener Go instalado es ideal —especialmente para correr el detector de races con `go run -race`—, pero podés seguirlos leyendo: están explicados línea por línea.

> ⚠️ **Por qué dos lenguajes.** Node es single-thread: te deja demostrar las **races lógicas** (las que pasan por el `await`) y todos los patrones async, pero para una **data race de memoria compartida de verdad** y para ver un **detector de races** en acción, Go es mucho más claro (`go run -race`). Cada build muestra el plano que mejor lo enseña; varios muestran los dos.

**Los cinco builds**
1. Una **race condition** (lógica en Node + data race real en Go, y sus fixes)
2. Una **cola producer-consumer** acotada (con backpressure)
3. Un **connection pool con semáforo** (con timeout)
4. Un **deadlock a propósito** (y el fix por orden de locks)
5. Un **thread pool con cola acotada** (worker_threads + backpressure)

> Convención: el código está pensado para **correr**. El TypeScript compila en `--strict`. Las **extensiones** (ejercicios para llevar cada build más lejos) están al final de cada sección, y sus pistas/soluciones, agrupadas en la última sección.

---

## Build 1 — Una race condition (y cómo se arregla)

**El problema.** Queremos reservar asientos. La invariante: **un asiento, un dueño**. Vamos a escribir código que la viola y verlo fallar.

### 1a) La race lógica en Node (sin un solo hilo extra)

El clásico check-then-act con un `await` en el medio (módulo 5 de la teoría):

```ts
// reservas-rotas.ts — FALLA a propósito
const asientos = new Map<string, string | null>([['A1', null]])  // null = libre

async function leerEstado(asiento: string): Promise<string | null> {
  await new Promise(r => setTimeout(r, 10))   // simula la latencia de una DB (cede el control)
  return asientos.get(asiento) ?? null
}
async function escribirDueno(asiento: string, usuario: string): Promise<void> {
  await new Promise(r => setTimeout(r, 10))
  asientos.set(asiento, usuario)
}

async function reservar(asiento: string, usuario: string): Promise<void> {
  const dueno = await leerEstado(asiento)      // (1) CHECK  ← acá el event loop cede el turno
  if (dueno === null) {
    await escribirDueno(asiento, usuario)        // (2) ACT
    console.log(`${usuario} reservó ${asiento}`)
  } else {
    console.log(`${usuario}: ${asiento} ya era de ${dueno}`)
  }
}

// dos personas reservan el MISMO asiento al mismo tiempo
await Promise.all([reservar('A1', 'Ana'), reservar('A1', 'Beto')])
console.log('dueño final:', asientos.get('A1'))
```

**Qué observás.** Las dos imprimen "reservó A1". El dueño final es uno solo (el último `set` gana), pero **a las dos se les confirmó la reserva**. Doble reserva. La ventana es el `await` del check: Ana lee `null` y cede; Beto lee `null` (Ana todavía no escribió); las dos creen que ganaron.

**El fix: serializar por recurso con un mutex async.** No alcanza con "hacelo más rápido": hay que volver **atómica la secuencia** check→act. Un mutex por asiento garantiza que solo una reserva del mismo asiento corra a la vez:

```ts
// mutex.ts — un mutex async reutilizable (lo usamos en varios builds)
export class Mutex {
  private cola: Promise<void> = Promise.resolve()
  async runExclusive<T>(fn: () => Promise<T>): Promise<T> {
    const anterior = this.cola
    let liberar!: () => void
    this.cola = new Promise<void>(res => (liberar = res))  // el siguiente esperará a este
    await anterior                                          // espero mi turno
    try {
      return await fn()
    } finally {
      liberar()                                             // dejo pasar al siguiente
    }
  }
}
```

```ts
// reservas-ok.ts — el check-then-act ahora es atómico por asiento
// (reusamos `leerEstado` / `escribirDueno` / `asientos` definidos en 1a)
import { Mutex } from './mutex.js'
const locks = new Map<string, Mutex>()
function lockDe(asiento: string): Mutex {
  let m = locks.get(asiento)
  if (!m) { m = new Mutex(); locks.set(asiento, m) }
  return m
}

async function reservar(asiento: string, usuario: string): Promise<void> {
  await lockDe(asiento).runExclusive(async () => {   // ← nadie más con este asiento entra acá
    const dueno = await leerEstado(asiento)
    if (dueno === null) {
      await escribirDueno(asiento, usuario)
      console.log(`${usuario} reservó ${asiento}`)
    } else {
      console.log(`${usuario}: ${asiento} ya era de ${dueno}`)
    }
  })
}
```

Ahora Beto entra al check **después** de que Ana terminó el act, ve el asiento ocupado, y se le rechaza. Invariante intacta.

> **El fix de producción** no es un mutex en memoria (no sirve si tenés varias instancias del servidor): es empujar la atomicidad **al almacén**. Un `UPDATE asientos SET dueno=$1 WHERE id=$2 AND dueno IS NULL` devuelve "1 fila afectada" para el ganador y "0" para el perdedor — la DB serializa por vos. O un constraint único. El mutex en memoria es la versión didáctica del mismo principio.

### 1b) La data race real en Go (memoria compartida)

Acá sí hay hilos pisándose la misma memoria. Corré con el **detector de races**:

```go
// race.go — corré:  go run -race race.go
package main

import ("fmt"; "sync")

func main() {
    contador := 0
    var wg sync.WaitGroup
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() { defer wg.Done(); contador++ }()  // 1000 goroutines, sin coordinación
    }
    wg.Wait()
    fmt.Println(contador)  // casi nunca 1000; el -race te grita "DATA RACE"
}
```

**Qué observás.** El número final varía entre corridas y casi nunca es 1000. Con `-race`, Go te imprime exactamente **qué dos goroutines** tocaron la misma dirección sin sincronizar.

**Dos fixes, según el caso:**

```go
// fix A — mutex: cuando protegés una sección (varias operaciones)
var mu sync.Mutex
go func() { defer wg.Done(); mu.Lock(); contador++; mu.Unlock() }()

// fix B — atómico: cuando es UNA operación simple (más rápido, sin lock)
var contador atomic.Int64
go func() { defer wg.Done(); contador.Add(1) }()   // siempre 1000
```

**Qué demuestra este build.** Que **la race nace de la concurrencia, no de los hilos**: la viste en Node (un hilo, un `await`) y en Go (mil hilos, memoria compartida). Y que el fix siempre es el mismo principio —**atomizar la secuencia**— con dos sabores: lock (para secuencias) o atómico (para una operación). Conecta con los módulos 5, 6 y 8 de la teoría.

**Extensiones**
1.1 ✍️ En la versión Node rota, agregá un log con `Date.now()` antes del check y después del act de cada reserva. Observá cómo se intercalan.
1.2 🧠 ¿Por qué un mutex **por asiento** (no uno global) es mejor? ¿Qué pasaría con un único mutex para todas las reservas?
1.3 ✍️ Reescribí el fix usando el patrón de producción (`UPDATE ... WHERE dueno IS NULL`) en pseudo-SQL y explicá por qué funciona con varias instancias del servidor.

---

## Build 2 — Una cola producer-consumer acotada

**El problema.** Un productor genera trabajo más rápido de lo que el consumidor procesa. Si la cola del medio no tiene límite, crece hasta el OOM (módulo 17). Queremos una cola **acotada** que **frene al productor** cuando se llena: backpressure.

### En Node (async, con promesas)

```ts
// cola-acotada.ts — cola con capacidad máxima; push() espera si está llena
export class ColaAcotada<T> {
  private items: T[] = []
  private esperandoEspacio: Array<() => void> = []   // productores frenados
  private esperandoItem: Array<(v: T) => void> = []  // consumidores esperando

  constructor(private readonly capacidad: number) {}

  // productor: si no hay lugar, AWAITea hasta que se libere (backpressure)
  async push(item: T): Promise<void> {
    // ⚠️ 'while', NO 'if': al despertar hay que RE-CHEQUEAR la capacidad (módulo 10 de
    // la teoría). Con varios productores, otro push pudo robarse el lugar libre entre el
    // shift() del pop y el momento en que esta tarea corre → re-verificamos y, si no hay
    // lugar, volvemos a esperar. Con 'if' (un solo await) la cola podría superar la capacidad.
    while (this.items.length >= this.capacidad) {
      await new Promise<void>(res => this.esperandoEspacio.push(res))
    }
    const consumidor = this.esperandoItem.shift()
    if (consumidor) { consumidor(item); return }      // entrega directa si alguien esperaba
    this.items.push(item)
  }

  // consumidor: si está vacía, AWAITea hasta que llegue un item
  async pop(): Promise<T> {
    const item = this.items.shift()
    if (item !== undefined) {
      const productor = this.esperandoEspacio.shift()
      if (productor) productor()                       // liberé un lugar → despierto un productor
      return item
    }
    return new Promise<T>(res => this.esperandoItem.push(res))
  }
}
```

```ts
// demo.ts — productor rápido, consumidor lento: la cola NO crece sin límite
const cola = new ColaAcotada<number>(5)   // tope: 5

async function productor() {
  for (let i = 0; i < 20; i++) {
    await cola.push(i)                      // se FRENA solo cuando hay 5 sin consumir
    console.log('produje', i, '| en cola:', /* nunca pasa de 5 */ i)
  }
}
async function consumidor() {
  for (let i = 0; i < 20; i++) {
    const v = await cola.pop()
    await new Promise(r => setTimeout(r, 50))  // lento a propósito
    console.log('   consumí', v)
  }
}
await Promise.all([productor(), consumidor()])
```

**Qué observás.** El productor avanza a los saltos: mete 5, se frena, y solo continúa cuando el consumidor saca uno. La memoria queda acotada por más desbalanceadas que estén las velocidades. **Eso es backpressure**: la cola, al estar llena, hace esperar al `push`.

### En Go (un channel acotado YA es esto)

Go te lo da gratis: un channel con buffer **es** una cola acotada con backpressure incorporado.

```go
// pc.go
cola := make(chan int, 5)   // capacidad 5

go func() {                 // productor
    for i := 0; i < 20; i++ { cola <- i }   // 'cola <- i' BLOQUEA si está llena → backpressure
    close(cola)
}()

for v := range cola {       // consumidor: drena hasta que se cierre el channel
    time.Sleep(50 * time.Millisecond)
    fmt.Println("consumí", v)
}
```

**Qué demuestra este build.** Que la **cola del medio es la pieza de diseño** del patrón productor-consumidor (módulo 17), y que **acotarla = backpressure** (módulo 19). En Node lo construiste a mano para *ver* el mecanismo; en Go es una primitiva del lenguaje. El salto a producción es una cola de mensajes (BullMQ, SQS) — mismo modelo, durable y fuera del proceso (ver [redis.md], [event-driven.md]).

**Extensiones**
2.1 ✍️ Agregá a `ColaAcotada` un método `size()` y logueá el tamaño en cada push: comprobá que nunca supera la capacidad.
2.2 🧠 ¿Qué cambiarías para implementar una política de **drop** (descartar el item más viejo) en vez de bloquear al productor? ¿Cuándo elegirías cada una?
2.3 ✍️ Hacé que `push` acepte un timeout: si no consigue lugar en X ms, rechaza con error (útil para no esperar para siempre).

---

## Build 3 — Un connection pool con semáforo

**El problema.** Tenés un recurso finito (10 conexiones a la DB) y muchos clientes que lo quieren (500 requests). Abrir 500 conexiones tumba la base. Queremos un **pool**: como mucho N en uso a la vez, el resto encolado. Es un **semáforo de capacidad N** (módulo 9) envolviendo conexiones reales.

### En Node

```ts
// semaforo.ts — semáforo con acquire/release y timeout opcional
export class Semaforo {
  private permisos: number
  private cola: Array<() => void> = []
  constructor(n: number) { this.permisos = n }

  async acquire(timeoutMs?: number): Promise<void> {
    if (this.permisos > 0) { this.permisos--; return }
    return new Promise<void>((resolve, reject) => {
      const liberar = () => { this.permisos--; resolve() }
      this.cola.push(liberar)
      if (timeoutMs !== undefined) {
        setTimeout(() => {
          const i = this.cola.indexOf(liberar)
          if (i >= 0) { this.cola.splice(i, 1); reject(new Error('timeout esperando permiso')) }
        }, timeoutMs)
      }
    })
  }

  release(): void {
    this.permisos++
    const siguiente = this.cola.shift()
    if (siguiente) siguiente()   // despierto a uno; él hace permisos-- de nuevo
  }
}
```

```ts
// pool.ts — pool de conexiones simuladas sobre el semáforo
import { Semaforo } from './semaforo.js'

interface Conexion { id: number; query(_sql: string): Promise<unknown> }

export class Pool {
  private sem: Semaforo
  private libres: Conexion[]
  constructor(conexiones: Conexion[]) {
    this.libres = [...conexiones]
    this.sem = new Semaforo(conexiones.length)   // tantos permisos como conexiones
  }

  // tomá una conexión, usala, y se devuelve sola (patrón "withConnection")
  async withConnection<T>(fn: (c: Conexion) => Promise<T>, timeoutMs?: number): Promise<T> {
    await this.sem.acquire(timeoutMs)            // espero un permiso (o timeout)
    const conn = this.libres.pop()!              // garantizado por el semáforo
    try {
      return await fn(conn)
    } finally {
      this.libres.push(conn)                     // devuelvo la conexión...
      this.sem.release()                         // ...y el permiso
    }
  }
}
```

```ts
// demo.ts — 500 tareas, 10 conexiones: nunca hay más de 10 queries a la vez
const conexiones: Conexion[] = Array.from({ length: 10 }, (_, id) => ({
  id,
  async query(_sql) { await new Promise(r => setTimeout(r, 100)); return [] },
}))
const pool = new Pool(conexiones)
let enVuelo = 0, maxEnVuelo = 0

await Promise.all(Array.from({ length: 500 }, (_, i) =>
  pool.withConnection(async (c) => {
    enVuelo++; maxEnVuelo = Math.max(maxEnVuelo, enVuelo)
    await c.query(`SELECT ${i}`)
    enVuelo--
  })
))
console.log('máximo de queries simultáneas:', maxEnVuelo)  // === 10, nunca más
```

**Qué observás.** Las 500 tareas completan, pero `maxEnVuelo` **nunca supera 10**. El semáforo encola el excedente y lo deja pasar a medida que se liberan conexiones. Si agregás un `timeoutMs`, las que esperan demasiado fallan rápido en vez de colgarse — control de blast radius (módulo 9, capa 3).

### En Go (channel como pool de conexiones)

```go
// pool.go — el channel ES el pool: contiene las conexiones disponibles
type Conn struct{ id int }
pool := make(chan *Conn, 10)
for i := 0; i < 10; i++ { pool <- &Conn{id: i} }   // lleno el pool

usar := func(i int) {
    conn := <-pool                  // acquire: bloquea si no hay conexiones libres
    defer func() { pool <- conn }() // release: la devuelvo al terminar
    time.Sleep(100 * time.Millisecond)  // query
}
// lanzar 500 usos: como mucho 10 corren a la vez
```

**Qué demuestra este build.** Que un connection pool **es** un semáforo (módulo 9) y que limitar concurrencia sobre un recurso finito es backpressure aplicado a la base de datos. Es exactamente lo que hace `pg.Pool` por debajo (ver [postgresql.md]): vos pedís una conexión, esperás si no hay, la devolvés al `finally`. Construirlo te saca el misterio de encima.

**Extensiones**
3.1 ✍️ Agregá al pool un contador de "esperando" y logueá la profundidad de la cola: es la métrica que dispararía autoescalado o alertas.
3.2 🧠 ¿Por qué el tamaño del pool debería salir de la capacidad real de la DB y no ser "lo más grande posible"? Conectá con el módulo 9 (capa 4).
3.3 ✍️ Hacé que el `acquire` con timeout, al expirar, incremente una métrica `poolTimeouts`: es la señal temprana de que el pool quedó chico para la carga.

---

## Build 4 — Un deadlock a propósito (y el fix)

**El problema.** Vamos a **provocar** un deadlock para verlo, y después arreglarlo con la técnica estándar: **orden total de adquisición de locks** (módulo 13). El escenario clásico: transferencias bancarias que toman dos locks (cuenta origen y destino) en orden distinto.

### En Go (el detector de deadlock del runtime lo hace evidente)

```go
// deadlock.go — corré:  go run deadlock.go   (Go panic-ea si TODO se traba)
package main

import ("sync"; "time")

type Cuenta struct { mu sync.Mutex; saldo int }

func transferir(de, a *Cuenta, monto int) {
    de.mu.Lock()                       // toma el lock del origen
    time.Sleep(time.Millisecond)        // ← da tiempo a que la otra goroutine tome SU primer lock
    a.mu.Lock()                        // toma el lock del destino  ← acá se traba
    de.saldo -= monto; a.saldo += monto
    a.mu.Unlock(); de.mu.Unlock()
}

func main() {
    x, y := &Cuenta{saldo: 100}, &Cuenta{saldo: 100}
    var wg sync.WaitGroup; wg.Add(2)
    go func() { defer wg.Done(); transferir(x, y, 10) }()  // bloquea x, quiere y
    go func() { defer wg.Done(); transferir(y, x, 10) }()  // bloquea y, quiere x
    wg.Wait()  // nunca retorna: "fatal error: all goroutines are asleep - deadlock!"
}
```

**Qué observás.** El programa **se cuelga**. La goroutine 1 tiene el lock de `x` y espera el de `y`; la 2 tiene el de `y` y espera el de `x`. Espera circular (módulo 13). Go detecta que **todas** las goroutines están dormidas y aborta con `deadlock!`. En un servidor real no abortaría: solo se colgarían esas requests, los hilos quedarían en BLOCKED, y el pool se iría agotando — más insidioso.

**El fix: orden total.** Hacemos que **todas** las transferencias tomen los locks en el mismo orden global (por ejemplo, por id de cuenta ascendente). Así nunca puede formarse un ciclo — se rompe la condición 4 de Coffman:

```go
// fix: tomar SIEMPRE primero el lock de la cuenta con id menor
type Cuenta struct { id int; mu sync.Mutex; saldo int }

func transferir(de, a *Cuenta, monto int) {
    primero, segundo := de, a
    if a.id < de.id { primero, segundo = a, de }   // orden total por id
    primero.mu.Lock()
    segundo.mu.Lock()                               // ya nadie puede tomarlos en orden inverso
    de.saldo -= monto; a.saldo += monto
    segundo.mu.Unlock(); primero.mu.Unlock()
}
```

Ahora, transfiera quien transfiera, **ambas** goroutines intentan tomar primero el lock de la cuenta de menor id. Una gana los dos, la otra espera ordenadamente. Sin ciclo, sin deadlock.

### En Node (deadlock async con dos mutex)

Aunque Node sea single-thread, **podés deadlockear con locks async** si dos tareas los adquieren en orden opuesto (usando el `Mutex` del Build 1):

```ts
// deadlock-async.ts — se cuelga: A toma m1 y espera m2; B toma m2 y espera m1
import { Mutex } from './mutex.js'
const m1 = new Mutex(), m2 = new Mutex()

async function tareaA() {
  await m1.runExclusive(async () => {
    await new Promise(r => setTimeout(r, 10))
    await m2.runExclusive(async () => { /* nunca llega */ })  // m2 lo tiene B
  })
}
async function tareaB() {
  await m2.runExclusive(async () => {
    await new Promise(r => setTimeout(r, 10))
    await m1.runExclusive(async () => { /* nunca llega */ })  // m1 lo tiene A
  })
}
await Promise.all([tareaA(), tareaB()])  // nunca resuelve
// Fix: que ambas tomen SIEMPRE m1 antes que m2 (orden total).
```

**Qué demuestra este build.** Que el deadlock es **estructural** (espera circular), no específico de los hilos del SO: aparece en Go con mutex reales y en Node con mutex async. Y que el fix canónico —**orden total de adquisición**— rompe la condición 4 de Coffman en los dos. El control de corto plazo (módulo 13, capa 3) sería un **timeout** en `Lock`; el de largo plazo es este orden total.

**Extensiones**
4.1 ✍️ En la versión Go con bug, agregá `runtime.Stack` o corré con `GOTRACEBACK=all` para ver las dos goroutines trabadas.
4.2 ✍️ Implementá la variante "control de corto plazo": un `TryLock` con timeout que, si no consigue el segundo lock en X ms, suelta el primero y reintenta. ¿Qué condición de Coffman rompe?
4.3 🧠 ¿Por qué el orden total funciona aunque haya 100 cuentas y miles de transferencias simultáneas?

---

## Build 5 — Un thread pool con cola acotada

**El problema.** Querés procesar trabajo CPU-bound en paralelo de verdad (no en el event loop, que se congelaría: módulo 18) pero **sin crear un worker por tarea** (módulo 16) y **sin que la cola de pendientes crezca sin límite** (módulos 17 y 19). La pieza final que junta casi todo: un pool de `worker_threads` + una cola **acotada** con backpressure.

### El worker (el código que corre en cada hilo)

```ts
// worker.ts — corre en un hilo aparte; recibe tareas y devuelve resultados
import { parentPort } from 'node:worker_threads'

function trabajoPesado(n: number): number {       // CPU-bound de ejemplo
  let s = 0
  for (let i = 0; i < n * 1e6; i++) s += Math.sqrt(i)
  return s
}

parentPort!.on('message', (msg: { id: number; n: number }) => {
  const resultado = trabajoPesado(msg.n)
  parentPort!.postMessage({ id: msg.id, resultado })
})
```

### El pool (N workers fijos + cola acotada)

```ts
// pool-workers.ts
import { Worker } from 'node:worker_threads'

interface Tarea { id: number; n: number; resolve: (v: number) => void; reject: (e: Error) => void }

export class ThreadPool {
  private workers: Worker[] = []
  private libres: Worker[] = []
  private cola: Tarea[] = []
  private pendientesPorWorker = new Map<Worker, Tarea>()
  private esperandoEspacio: Array<() => void> = []
  private nextId = 0

  constructor(tamano: number, private readonly capacidadCola: number) {
    for (let i = 0; i < tamano; i++) {
      const w = new Worker(new URL('./worker.js', import.meta.url))
      w.on('message', (msg: { id: number; resultado: number }) => {
        const tarea = this.pendientesPorWorker.get(w)!
        this.pendientesPorWorker.delete(w)
        tarea.resolve(msg.resultado)
        this.libres.push(w)
        this.despachar()                       // este worker quedó libre → tomar de la cola
        const prod = this.esperandoEspacio.shift()
        if (prod) prod()                        // liberé un lugar en la cola → despierto un submit
      })
      this.workers.push(w)
      this.libres.push(w)
    }
  }

  // submit aplica BACKPRESSURE: si la cola está llena, espera (no la deja crecer infinita)
  async submit(n: number): Promise<number> {
    if (this.cola.length >= this.capacidadCola) {
      await new Promise<void>(res => this.esperandoEspacio.push(res))
    }
    return new Promise<number>((resolve, reject) => {
      this.cola.push({ id: this.nextId++, n, resolve, reject })
      this.despachar()
    })
  }

  private despachar(): void {
    while (this.libres.length > 0 && this.cola.length > 0) {
      const w = this.libres.pop()!
      const tarea = this.cola.shift()!
      this.pendientesPorWorker.set(w, tarea)
      w.postMessage({ id: tarea.id, n: tarea.n })
    }
  }

  async destroy(): Promise<void> { await Promise.all(this.workers.map(w => w.terminate())) }
}
```

```ts
// demo.ts — 4 workers, cola tope 8: 100 tareas pesadas sin congelar ni reventar memoria
const pool = new ThreadPool(4, 8)
const resultados = await Promise.all(
  Array.from({ length: 100 }, (_, i) => pool.submit(20))
)
console.log('listas:', resultados.length)
await pool.destroy()
```

**Qué observás.** Las 100 tareas pesadas corren **en 4 hilos reales** (el event loop principal queda libre — podrías seguir atendiendo HTTP mientras tanto). En todo momento hay **a lo sumo 4 ejecutándose** y **a lo sumo 8 en cola**; los `submit` de más **esperan** hasta que se libere lugar. Pool fijo (módulo 16) + cola acotada (módulo 17) + backpressure (módulo 19), todo junto.

### En Go (worker pool con channels — el patrón idiomático)

```go
// pool.go — N workers consumen de un channel acotado; enviar bloquea si está lleno
tareas := make(chan int, 8)       // cola acotada (capacidad 8) → backpressure
resultados := make(chan int, 8)
var wg sync.WaitGroup

for i := 0; i < 4; i++ {           // 4 workers fijos
    wg.Add(1)
    go func() {
        defer wg.Done()
        for n := range tareas { resultados <- trabajoPesado(n) }
    }()
}
go func() {                        // productor
    for i := 0; i < 100; i++ { tareas <- 20 }   // bloquea si hay 8 sin tomar
    close(tareas)
}()
go func() { wg.Wait(); close(resultados) }()
for r := range resultados { _ = r }
```

**Qué demuestra este build.** Es el **capstone** del laboratorio: combina paralelismo real (módulo 2/3 — `worker_threads`/goroutines), thread pool (módulo 16), producer-consumer (módulo 17), blocking vs non-blocking (módulo 18 — sacamos el CPU del event loop) y backpressure (módulo 19). Si entendés este build, entendés cómo se construye un procesador de trabajo serio.

**Extensiones**
5.1 ✍️ Agregá manejo de errores: si un worker emite `error`, rechazá la tarea pendiente y reemplazá el worker muerto por uno nuevo (resiliencia del pool).
5.2 🧠 ¿Por qué el tamaño del pool acá debería ser ≈ número de cores y no 100? Conectá con el módulo 16 (CPU-bound).
5.3 ✍️ Exponé una métrica `cola.length` y `libres.length` cada 100ms: es el tablero mínimo para saber si el pool está saturado (la profundidad de cola creciente = consumidores no dan abasto).

---

## Soluciones y pistas de las extensiones

> Las extensiones son abiertas; acá van las pistas y los puntos clave que tu solución debería tocar.

**1.1** Vas a ver el patrón `check(Ana) → check(Beto) → act(Ana) → act(Beto)`: los dos checks ocurren antes de cualquier act, que es exactamente la ventana de la race.

**1.2** Un mutex **por asiento** permite que reservas de asientos **distintos** corran en paralelo (no se serializan entre sí) — es locking fino (módulo 12). Un único mutex global serializaría **todas** las reservas del sistema aunque sean de asientos sin relación: correcto pero un cuello de botella innecesario.

**1.3** `UPDATE asientos SET dueno=$user WHERE id=$seat AND dueno IS NULL`. La DB evalúa la condición y escribe **en una sola operación atómica**; devuelve filas afectadas = 1 al ganador, 0 a los demás. Funciona con N instancias del servidor porque la atomicidad vive en la **DB compartida**, no en la memoria de un proceso (que cada instancia tendría por separado).

**2.1** `size()` devuelve `this.items.length`. Vas a comprobar que jamás supera `capacidad`: ese es el invariante que da el backpressure.

**2.2** Drop: si `items.length >= capacidad`, en vez de `await`, hacés `this.items.shift()` (descartás el más viejo) antes del `push`, y **no** frenás al productor. Elegís **bloquear** cuando cada ítem importa (pedidos, pagos) y **drop** cuando la frescura importa más que la completitud (métricas, telemetría, posiciones en tiempo real).

**2.3** En `push`, si tenés que esperar espacio, corré un `setTimeout` que haga `reject(new Error('push timeout'))` y remové al productor de `esperandoEspacio` si expira (mismo patrón que el `acquire` con timeout del Build 3).

**3.1** Un contador `esperando` que incrementás al entrar a la cola del semáforo y decrementás al salir. Logueado, es la **profundidad de cola** — la métrica que en producción dispara alertas o autoescalado.

**3.2** Porque el recurso real (la DB) tiene un máximo de conexiones que puede atender bien; un pool más grande que eso solo **traslada** la saturación a la DB (que empieza a rechazar o a degradarse). El tamaño correcto ≈ capacidad de la DB; el exceso se maneja con cola + backpressure, no abriendo más conexiones (módulo 9, capa 4).

**3.3** Incrementás `poolTimeouts` en el `reject` del `acquire`. Un valor que sube sostenidamente significa que la demanda supera al pool de forma crónica: o agrandás el recurso, o aplicás backpressure/shedding río arriba.

**4.1** `GOTRACEBACK=all go run deadlock.go` te muestra ambas goroutines con su stack: una en `a.mu.Lock()` esperando, la otra también. Ver las dos esperando el lock que tiene la otra **es** la espera circular.

**4.2** `TryLock` con timeout: tomás el primer lock; intentás el segundo con un límite de tiempo; si no lo conseguís, **soltás el primero**, esperás un tiempo aleatorio (jitter, para no recaer en livelock — módulo 14) y reintentás todo. Rompe **hold-and-wait**: al soltar voluntariamente el lock que ya tenías, dejás de "retener mientras esperás". (Ojo con el matiz: *no preemption* sería que **otro** te quite el lock a la fuerza; acá lo soltás vos, así que lo correcto es decir que atacás hold-and-wait.)

**4.3** Porque el orden total define un **orden global único** sobre todos los locks: para que haya ciclo, alguien tendría que tomar B antes que A mientras otro toma A antes que B — y el orden total lo prohíbe por construcción, sin importar cuántas cuentas o transferencias haya.

**5.1** Escuchá `w.on('error', ...)`: rechazá la tarea que tenía asignada (`pendientesPorWorker.get(w)`), remové el worker muerto de las listas, creá uno nuevo con el mismo handler y agregalo a `libres`. Un pool de producción **se auto-repara**.

**5.2** Porque el trabajo es **CPU-bound**: con más workers que cores no hacés más cálculo (no hay más CPU), solo agregás context switching y memoria. 100 workers para 8 cores rinden peor que 8. El exceso de *trabajo* se maneja con la **cola acotada**, no con más hilos (módulo 16).

**5.3** Logueá `this.cola.length` (pendientes) y `this.libres.length` (workers ociosos). Si la cola crece y `libres` está en 0 de forma sostenida, el pool está saturado: los productores generan más rápido de lo que 4 workers procesan → o escalás workers/cores, o aplicás más backpressure río arriba.

---

> **Cierre.** Construiste los cinco: viste una race fallar con tus ojos (y la arreglaste con un lock y con un atómico), frenaste un productor con una cola acotada, limitaste un recurso con un semáforo, colgaste un programa con un deadlock y lo destrabaste con orden total, y armaste un procesador paralelo con pool + cola + backpressure. Eso es concurrencia **de verdad**, no de definiciones. Volvé al [módulo de teoría](concurrencia.md) y releé el cierre (módulo 21): ahora cada concepto tiene un recuerdo físico de haberlo roto y arreglado — que es exactamente lo que te va a salir natural en una entrevista.
