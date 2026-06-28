# Consenso y coordinación: laboratorio práctico

**Construí, rompé y arreglá: relojes lógicos, quórum R+W>N, elección de líder estilo Raft, fencing token y consistent hashing · Node + Go · 2026**

> Cómo usar esta guía: este es el complemento de mano del módulo de teoría [Consenso y coordinación distribuida](consenso.md). Ahí entendiste los conceptos; acá **construís cinco cosas**, cada una con el mismo método: **(1)** el problema, **(2)** código que **falla a propósito**, **(3)** qué observás cuando lo corrés, **(4)** el fix, **(5)** qué demuestra. Romper el código con tus manos y verlo fallar fija el concepto como ninguna definición.

**Lo que asumimos.** Haber leído (o tener a mano) el [módulo de teoría](consenso.md). Node 20+ y TypeScript. Para los ejemplos de Go, tener Go instalado es ideal, pero podés seguirlos leyendo: están explicados línea por línea.

> ⚠️ **Por qué dos lenguajes.** Los relojes lógicos, el quórum, el fencing y el hash ring son **lógica pura**: van en Node/TS (tu stack). La **elección de líder** se trata de muchos timers concurrentes compitiendo, y eso se ve mucho más claro con goroutines + `select` + timeout en Go — así que el build 3 muestra los dos planos. Cada build usa el lenguaje que mejor lo enseña.

**Los cinco builds**
1. **Relojes lógicos** (Lamport que no detecta concurrencia + vector clock que sí)
2. Un **quórum de lectura/escritura** (lectura stale con `R+W≤N`, fresca con `R+W>N`)
3. Una **elección de líder estilo Raft** (split vote con timeout fijo, fix con timeout aleatorio)
4. Un **lock con fencing token** (dos dueños corrompen, el token los corta)
5. Un **hash ring** (rehash masivo del `% N`, fix con ring + virtual nodes)

> Convención: el código está pensado para **correr**. El TypeScript compila en `--strict`. Las **extensiones** (ejercicios para llevar cada build más lejos) están al final de cada sección, y sus pistas/soluciones, agrupadas en la última sección.

---

## Build 1 — Relojes lógicos (Lamport y vectoriales)

**El problema.** Tres nodos generan eventos y se mandan mensajes. Queremos responder dos preguntas sobre un par de eventos: *¿uno causó al otro?* y *¿fueron concurrentes?*. Vamos a ver que **Lamport responde la primera pero miente en la segunda**, y que el vector clock responde las dos.

### 1a) Reloj de Lamport — y por qué no alcanza

```ts
// lamport.ts
class NodoLamport {
  private l = 0
  constructor(public readonly id: string) {}

  // evento local: incremento antes de actuar (módulo 5, regla 1)
  local(): number {
    this.l += 1
    return this.l
  }
  // envío: adjunto mi reloj actual
  enviar(): number {
    this.l += 1
    return this.l
  }
  // recepción: L = max(L, Lm) + 1 (regla 2)
  recibir(lm: number): number {
    this.l = Math.max(this.l, lm) + 1
    return this.l
  }
}

const a = new NodoLamport('A')
const b = new NodoLamport('B')
const c = new NodoLamport('C')

const a1 = a.local()                 // A1: evento local de A     -> l=1
const msgDeA = a.enviar()            // A envía un mensaje         -> l=2 (adjunta 2)
const b1 = b.recibir(msgDeA)         // B recibe con Lm=2: max(0,2)+1 = 3  -> causalidad A→B
const c1 = c.local()                 // C hace lo suyo, SIN hablar con nadie -> concurrente

console.log({ a1, b1, c1 })          // { a1: 1, b1: 3, c1: 1 }
console.log('¿b1 > a1?', b1 > a1)    // true  -> respeta A→B (causa < efecto): 3 > 1. BIEN.
console.log('¿c1 < b1?', c1 < b1)    // true (1 < 3) ... ¿y entonces?
```

> Nota de convención: tratamos el **envío como un evento** que incrementa el reloj (la convención de Lamport; algunos textos adjuntan `l` sin incrementar). Por eso A llega a `2` antes de mandar, y el Build 1b del vector clock usa **la misma** secuencia (A hace `local()` y después `enviar()`), para que la comparación Lamport-vs-vector sea sobre los mismos eventos.

**Qué observás.** `a1=1`, `b1=3`, `c1=1`. La parte buena: como `A1 → B1` (B recibió el mensaje de A, que ya traía el reloj de A en 2), su timestamp es mayor (`3 > 1`). La causa quedó antes del efecto. **Pero** mirá `c1=1` y `b1=3`: un observador ingenuo vería `1 < 3` y concluiría que `c1` **ocurrió-antes** de `b1` — y se equivocaría, porque C nunca habló con B: son **concurrentes**. (Y `c1` empata con `a1` en `1`, otra vez sin decirte nada de su relación.) Lamport te da un orden total (rompiendo empates por id de nodo) consistente con la causalidad, pero **un timestamp menor NO implica ocurrió-antes** (el "la vuelta no vale" del módulo 5): no podés preguntarle "¿esto fue concurrente?" — la respuesta no está en el número.

### 1b) El fix: vector clock — que SÍ distingue concurrente de causal

```ts
// vector-clock.ts
type Vector = Record<string, number>

class NodoVector {
  private v: Vector
  constructor(public readonly id: string, nodos: readonly string[]) {
    this.v = Object.fromEntries(nodos.map(n => [n, 0]))
  }
  private snapshot(): Vector { return { ...this.v } }

  local(): Vector {
    this.v[this.id] += 1
    return this.snapshot()
  }
  enviar(): Vector {
    this.v[this.id] += 1
    return this.snapshot()
  }
  recibir(vm: Vector): Vector {
    for (const n of Object.keys(this.v)) {
      this.v[n] = Math.max(this.v[n], vm[n] ?? 0)   // tomo el máximo posición a posición
    }
    this.v[this.id] += 1                            // y después incremento la mía
    return this.snapshot()
  }
}

// comparación: 'antes' | 'despues' | 'concurrente' | 'igual'
function comparar(x: Vector, y: Vector): 'antes' | 'despues' | 'concurrente' | 'igual' {
  const claves = new Set([...Object.keys(x), ...Object.keys(y)])
  let xMenor = false, xMayor = false
  for (const k of claves) {
    const xv = x[k] ?? 0, yv = y[k] ?? 0
    if (xv < yv) xMenor = true
    if (xv > yv) xMayor = true
  }
  if (xMenor && xMayor) return 'concurrente'   // cada uno tiene algo que el otro no vio
  if (xMenor) return 'antes'
  if (xMayor) return 'despues'
  return 'igual'
}

const nodos = ['A', 'B', 'C'] as const
const va = new NodoVector('A', nodos)
const vb = new NodoVector('B', nodos)
const vc = new NodoVector('C', nodos)

const eA1 = va.local()          // {A:1, B:0, C:0}
const msg = va.enviar()         // {A:2, B:0, C:0}
const eB1 = vb.recibir(msg)     // {A:2, B:1, C:0}  -> vio a A
const eC1 = vc.local()          // {A:0, B:0, C:1}  -> NO vio a nadie

console.log('A1 vs B1:', comparar(eA1, eB1))   // 'antes'        -> A1 causó (precede a) B1  ✅
console.log('A1 vs C1:', comparar(eA1, eC1))   // 'concurrente'  -> ¡lo que Lamport no podía! ✅
console.log('B1 vs C1:', comparar(eB1, eC1))   // 'concurrente'
```

**Qué observás.** Ahora `comparar(A1, C1)` devuelve `'concurrente'`, no un empate ambiguo: el sistema **sabe** que esos dos eventos chocaron (cada vector tiene algo que el otro no vio). Eso es exactamente lo que una base leaderless necesita para decidir entre "esta versión es más vieja, la piso" y "estas dos chocaron, guardo ambas (siblings)".

**Qué demuestra.** El salto del módulo 5 al 6: Lamport ordena pero **colapsa** la concurrencia en un solo número; el vector clock conserva, por nodo, *qué vio cada quién*, y por eso puede decir "ninguno vio al otro → concurrentes". El precio es el tamaño `O(nodos)`. Es la base de la resolución de conflictos en Dynamo/Riak.

**Extensiones**
- 1.1 ✍️ Agregá un cuarto nodo D que reciba mensajes de B y C; verificá que `comparar` detecta correctamente cuándo un evento de D es posterior a ambos.
- 1.2 🧠 ¿Por qué `recibir` toma el `max` posición a posición **antes** de incrementar la propia, y no al revés?
- 1.3 ✍️ Hacé que `comparar` devuelva, además, el conjunto de nodos en los que `x` "va adelante" — útil para entender por qué dos vectores son concurrentes.

---

## Build 2 — Un quórum de lectura/escritura

**El problema.** Tenemos `N=3` réplicas de una clave. Vamos a escribir con `W` réplicas y leer con `R`, y ver con nuestros ojos que **`R+W≤N` puede devolver un valor stale** y que **`R+W>N` no**.

### 2a) `R+W ≤ N` — la lectura stale

```ts
// quorum.ts
type Version = { valor: string; version: number }

class Replica {
  private estado: Version = { valor: '∅', version: 0 }
  constructor(public readonly id: number, public viva = true) {}
  escribir(v: Version): void {
    if (!this.viva) throw new Error(`réplica ${this.id} caída`)
    if (v.version > this.estado.version) this.estado = v
  }
  leer(): Version {
    if (!this.viva) throw new Error(`réplica ${this.id} caída`)
    return this.estado
  }
}

class Cluster {
  constructor(private replicas: Replica[]) {}

  // escribe en las primeras W réplicas vivas
  escribir(v: Version, W: number): number {
    let ok = 0
    for (const r of this.replicas) {
      if (ok >= W) break
      try { r.escribir(v); ok++ } catch { /* caída: salteo */ }
    }
    if (ok < W) throw new Error(`no se alcanzó W=${W} (solo ${ok})`)
    return ok
  }
  // lee de las primeras R réplicas vivas y se queda con la versión más alta
  leer(R: number): Version {
    const leidas: Version[] = []
    for (const r of this.replicas) {
      if (leidas.length >= R) break
      try { leidas.push(r.leer()) } catch { /* caída */ }
    }
    if (leidas.length < R) throw new Error(`no se alcanzó R=${R}`)
    return leidas.reduce((max, v) => (v.version > max.version ? v : max))
  }
}

const reps = [new Replica(1), new Replica(2), new Replica(3)]
const cl = new Cluster(reps)

// escribo con W=1 (solo réplica 1 recibe v1)
cl.escribir({ valor: 'v1', version: 1 }, 1)

// Cluster.leer(R) arranca por la réplica 1 (que SÍ tiene v1). Para exhibir el peor caso
// de R+W≤N forzamos una lectura sobre la réplica 3, que quedó fuera de la escritura con W=1:
const stale = reps[2].leer()                 // réplica 3 NUNCA recibió v1
console.log('R=1,W=1 (R+W=2 ≤ 3):', stale)   // { valor: '∅', version: 0 }  -> STALE
```

**Qué observás.** Escribiste `v1` con `W=1` (solo la réplica 1 lo tiene). Una lectura con `R=1` que toque la réplica 3 devuelve `{ valor: '∅', version: 0 }` — el valor **viejo**. `R+W = 2 ≤ 3 = N`: no hay garantía de solape, así que el lector puede caer entero en réplicas que no vieron la escritura.

### 2b) El fix: `R+W > N`

```ts
// (mismo Cluster) — ahora W=2, R=2 sobre N=3
const reps2 = [new Replica(1), new Replica(2), new Replica(3)]
const cl2 = new Cluster(reps2)

cl2.escribir({ valor: 'v1', version: 1 }, 2)   // réplicas 1 y 2 tienen v1
const fresca = cl2.leer(2)                       // lee de 2 réplicas cualesquiera
console.log('R=2,W=2 (R+W=4 > 3):', fresca)      // { valor: 'v1', version: 1 }  -> FRESCA
```

**Qué observás.** Cualquier lectura de 2 réplicas **se solapa** con las 2 que recibieron la escritura (por palomar: 2+2 > 3, comparten al menos una). El `reduce` que se queda con la versión más alta encuentra `v1` sí o sí. Lectura fresca garantizada.

**Qué demuestra.** El quórum del módulo 8 en acción: la garantía no es magia, es **solapamiento de conjuntos**. `R+W>N` ⟺ todo par lectura/escritura comparte una réplica ⟺ la lectura ve la última escritura. Y deja claro el costo: subir `W` te da frescura pero hace la escritura más cara y frágil (más réplicas tienen que estar vivas).

**Extensiones**
- 2.1 ✍️ Marcá una réplica como `viva=false` y mostrá que con `W=2` la escritura todavía sucede (usa otra réplica), pero con `W=3` falla.
- 2.2 🧠 Con `N=3, W=3, R=1`: ¿qué garantía de lectura tenés y qué pasa si una réplica se cae?
- 2.3 ✍️ Agregá dos escrituras concurrentes con la **misma** versión pero distinto valor (simulando ausencia de líder) y mostrá por qué `R+W>N` **no** alcanza para resolver ese conflicto (necesitarías vector clocks del build 1).

---

## Build 3 — Elección de líder estilo Raft

**El problema.** Varios nodos sin líder tienen que elegir uno. Cada uno espera un *election timeout*; si no oye un líder, se postula y pide votos. Vamos a ver el **split vote** (nadie junta mayoría y se reintenta para siempre) cuando los timeouts son **iguales**, y cómo el timeout **aleatorio** lo corta.

### 3a) Timeout fijo → split vote (Go, donde los timers concurrentes se ven claros)

```go
// split_vote.go — todos los nodos tienen el MISMO timeout -> se postulan a la vez
package main

import (
	"fmt"
	"sync"
	"time"
)

const N = 5
const mayoria = N/2 + 1 // 3

// cada nodo, al expirar su timeout, "se postula" mandando su candidatura al canal
func nodo(id int, timeout time.Duration, postulaciones chan<- int, wg *sync.WaitGroup) {
	defer wg.Done()
	time.Sleep(timeout) // espera su election timeout
	postulaciones <- id // se postula (en la vida real: pide votos)
}

func main() {
	for ronda := 1; ronda <= 3; ronda++ {
		postulaciones := make(chan int, N)
		var wg sync.WaitGroup
		// TODOS con el MISMO timeout: se postulan prácticamente juntos
		for id := 1; id <= N; id++ {
			wg.Add(1)
			go nodo(id, 150*time.Millisecond, postulaciones, &wg)
		}
		wg.Wait()
		close(postulaciones)

		candidatos := 0
		for range postulaciones {
			candidatos++
		}
		// con 5 candidatos simultáneos, el voto se reparte: nadie junta mayoría
		fmt.Printf("ronda %d: %d candidatos a la vez, mayoría=%d -> %s\n",
			ronda, candidatos, mayoria,
			map[bool]string{true: "SPLIT VOTE, se reintenta", false: "líder"}[candidatos >= mayoria])
	}
}
```

**Qué observás.** En cada ronda, los 5 nodos se postulan casi simultáneamente. Como cada uno vota por sí mismo, el voto se reparte 5 maneras y **nadie** junta 3 → split vote. Y como todos reintentan con el **mismo** timeout otra vez, vuelve a pasar: un **livelock** (módulo 14 de concurrencia) — el sistema gasta CPU pero no progresa.

### 3b) El fix: election timeout aleatorio (Go)

```go
// election_ok.go — cada nodo sortea su timeout en un rango -> uno arranca primero
package main

import (
	"fmt"
	"math/rand"
	"sync"
	"time"
)

const N = 5

func main() {
	primero := make(chan int, 1) // solo el primero en postularse "gana" la ronda
	var once sync.Once
	var wg sync.WaitGroup

	for id := 1; id <= N; id++ {
		wg.Add(1)
		// timeout ALEATORIO en [150, 300) ms: casi nunca empatan
		timeout := time.Duration(150+rand.Intn(150)) * time.Millisecond
		go func(id int, t time.Duration) {
			defer wg.Done()
			time.Sleep(t)
			once.Do(func() { // el PRIMERO que despierta se vuelve líder y corta a los demás
				primero <- id
			})
		}(id, timeout)
	}
	wg.Wait()
	close(primero)
	fmt.Printf("líder electo: nodo %d (los demás recibieron su heartbeat antes de su propio timeout)\n", <-primero)
}
```

**Qué observás.** Un nodo sortea el timeout más corto, despierta primero, se postula y (en Raft real) manda heartbeats; los demás reciben ese heartbeat **antes** de que venza su propio timeout (más largo) y vuelven a follower sin postularse. Una sola ronda, un líder. **El azar desincronizó las postulaciones** — la misma cura que el jitter contra el retry storm.

### 3c) El mismo patrón en Node/TS (simulación determinística)

Si no tenés Go, esta versión en TS modela el mismo fenómeno con timers reales:

```ts
// election.ts
function elegir(timeoutsMs: number[]): { lider: number | null; motivo: string } {
  // el "líder" es el índice del timeout mínimo ÚNICO; si hay empate en el mínimo -> split vote
  const min = Math.min(...timeoutsMs)
  const conMin = timeoutsMs.filter(t => t === min).length
  if (conMin > 1) return { lider: null, motivo: `${conMin} nodos empataron en ${min}ms -> SPLIT VOTE` }
  return { lider: timeoutsMs.indexOf(min), motivo: `nodo ${timeoutsMs.indexOf(min)} despertó primero` }
}

console.log(elegir([150, 150, 150, 150, 150]))                 // empate -> split vote
console.log(elegir([212, 177, 263, 158, 291]))                 // aleatorio -> nodo 3 (158ms)
```

**Qué demuestra.** El corazón de la elección de Raft (módulo 9) no es la cuenta de votos: es **romper la simetría**. Con timeouts iguales el sistema es simétrico y se traba; el timeout aleatorio introduce asimetría y casi siempre produce un único "primero". Liveness garantizada con probabilidad altísima por ronda, sin sacrificar safety (un nodo vota una sola vez por término, así que aunque haya dos candidatos, no salen dos líderes).

**Extensiones**
- 3.1 🧠 ¿Por qué un rango de timeout **más ancho** baja la probabilidad de split vote pero **sube** el tiempo promedio hasta elegir líder? ¿Cómo elegirías el rango?
- 3.2 ✍️ Extendé la versión TS para simular `K` rondas: si hay split vote, re-sorteá timeouts y reintentá; contá cuántas rondas tarda en promedio con rango angosto vs ancho.
- 3.3 🧠 En 3b, ¿qué pasaría si el "líder" elegido se cae justo después? Relacionalo con el heartbeat del módulo 10.

---

## Build 4 — Un lock con fencing token

**El problema.** Un recurso (un "archivo") tiene que ser escrito por **un solo** cliente a la vez. Usamos un lock con lease (TTL). Vamos a reproducir el caso de Kleppmann (módulo 12): un cliente se **pausa** justo después de tomar el lock, el lease expira, otro lo toma, y al despertar el primero **corrompe** el recurso. Después lo arreglamos con un fencing token.

### 4a) Lock con lease, SIN fencing → dos dueños corrompen

```ts
// lock-roto.ts
class ServicioDeLock {
  private dueno: string | null = null
  private expiraEn = 0
  // adquiere si está libre o si el lease venció; devuelve true/false
  adquirir(cliente: string, ahora: number, ttlMs: number): boolean {
    if (this.dueno === null || ahora >= this.expiraEn) {
      this.dueno = cliente
      this.expiraEn = ahora + ttlMs
      return true
    }
    return false
  }
}

class Recurso {
  contenido = 'inicial'
  escribir(_cliente: string, dato: string): void {
    this.contenido = dato   // ❌ no valida NADA: escribe quien sea
  }
}

const lock = new ServicioDeLock()
const recurso = new Recurso()

// t=0: A adquiere (TTL 30)
console.log('A adquiere:', lock.adquirir('A', 0, 30))   // true
// A se "congela" (GC pause) ANTES de escribir...

// t=31: el lease de A venció; B adquiere
console.log('B adquiere:', lock.adquirir('B', 31, 30))  // true
recurso.escribir('B', 'datos-de-B')                      // B escribe legítimamente
console.log('tras B:', recurso.contenido)                // 'datos-de-B'

// t=32: A DESPIERTA, se cree dueño todavía, y escribe encima
recurso.escribir('A', 'datos-de-A-viejo')                // ❌ corrupción: A pisa a B
console.log('tras A revive:', recurso.contenido)         // 'datos-de-A-viejo'  <- MAL
```

**Qué observás.** El recurso queda con `'datos-de-A-viejo'`: el cliente A, que ya **no** era dueño del lock (su lease venció a t=30), escribió encima de B. El lock funcionó "bien" (le dio el lease a B), pero **no impidió** que A escribiera, porque el recurso no verifica nada. Dos dueños efectivos → corrupción.

### 4b) El fix: fencing token monotónico validado por el recurso

```ts
// lock-ok.ts
class ServicioDeLockFenced {
  private dueno: string | null = null
  private expiraEn = 0
  private token = 0                 // monotónico: sube en cada adquisición exitosa
  adquirir(cliente: string, ahora: number, ttlMs: number): number | null {
    if (this.dueno === null || ahora >= this.expiraEn) {
      this.dueno = cliente
      this.expiraEn = ahora + ttlMs
      this.token += 1               // nuevo dueño -> nuevo token, siempre mayor
      return this.token
    }
    return null
  }
}

class RecursoFenced {
  contenido = 'inicial'
  private ultimoToken = 0           // recuerda el mayor token que vio
  escribir(token: number, dato: string): void {
    if (token < this.ultimoToken) {
      throw new Error(`token ${token} rechazado (ya vi ${this.ultimoToken})`)  // ✅ corta al viejo
    }
    this.ultimoToken = token
    this.contenido = dato
  }
}

const lk = new ServicioDeLockFenced()
const rec = new RecursoFenced()

const tokenA = lk.adquirir('A', 0, 30)      // 1
// A se congela antes de escribir...
const tokenB = lk.adquirir('B', 31, 30)     // 2
rec.escribir(tokenB!, 'datos-de-B')          // ok: token 2 ≥ 0
console.log('tras B:', rec.contenido)        // 'datos-de-B'

// A despierta y escribe con su token VIEJO (1)
try {
  rec.escribir(tokenA!, 'datos-de-A-viejo')  // token 1 < 2 -> rechazado
} catch (e) {
  console.log('A rechazado:', (e as Error).message)  // token 1 rechazado (ya vi 2)
}
console.log('final:', rec.contenido)          // 'datos-de-B'  <- INTACTO ✅
```

**Qué observás.** Cuando A despierta e intenta escribir con su token viejo (`1`), el recurso lo **rechaza** porque ya vio el token `2` de B. El contenido queda intacto en `'datos-de-B'`. El lock no cambió de comportamiento — lo que cambió es que **el recurso valida el token**, y los tokens son monotónicos, así que el escritor "del pasado" siempre pierde.

**Qué demuestra.** La lección central del módulo 12: **un lock distribuido no es seguro por sí solo**. La exclusión real la hace el **recurso** validando un número monotónico. Si no controlás el recurso (no puede validar tokens), el lock es best-effort y tu diseño tiene que tolerar dos dueños — o usar consenso de verdad (etcd) para correctitud.

**Extensiones**
- 4.1 🧠 ¿Por qué el token tiene que ser **monotónico** y no, por ejemplo, un UUID aleatorio? ¿Qué garantía perderías?
- 4.2 ✍️ Hacé que el `ServicioDeLockFenced` viva en "otro proceso" (una función async con `await` que cede el control) y disparía dos `adquirir` concurrentes con `Promise.all`; confirmá que los tokens nunca se repiten.
- 4.3 🧠 Si el recurso es S3 (no podés agregarle validación de token), ¿qué opciones de diseño te quedan para evitar los dos escritores?

---

## Build 5 — Un hash ring (consistent hashing)

**El problema.** Repartimos 10 000 claves entre nodos. Con `hash(clave) % N`, agregar **un** nodo va a remapear casi todo. Con un **ring consistente** (+ virtual nodes), solo se mueve `~1/N`. Vamos a **contar** cuántas claves se mueven en cada caso.

### 5a) `% N` → rehash masivo

```ts
// modulo.ts
import { createHash } from 'node:crypto'

function h(s: string): number {
  // hash estable -> entero de 32 bits
  return parseInt(createHash('md5').update(s).digest('hex').slice(0, 8), 16)
}

function asignarModulo(clave: string, nNodos: number): number {
  return h(clave) % nNodos
}

const claves = Array.from({ length: 10_000 }, (_, i) => `clave-${i}`)

// mapa con 4 nodos vs 5 nodos: ¿cuántas claves cambian de nodo?
let movidas = 0
for (const k of claves) {
  if (asignarModulo(k, 4) !== asignarModulo(k, 5)) movidas++
}
console.log(`% N: al pasar de 4 a 5 nodos se movieron ${movidas} / 10000 claves`)
// % N: al pasar de 4 a 5 nodos se movieron ~8000 / 10000 claves  -> CATÁSTROFE
```

**Qué observás.** Al pasar de 4 a 5 nodos, **~80%** de las claves cambian de nodo. Solo agregaste **un** nodo y casi todo el dataset se tiene que mover (y un cache se invalidaría casi entero). Eso es el rehash masivo del módulo 17: `% N` no tiene nada de "consistente".

### 5b) El fix: ring consistente con virtual nodes

```ts
// ring.ts
import { createHash } from 'node:crypto'

function h(s: string): number {
  return parseInt(createHash('md5').update(s).digest('hex').slice(0, 8), 16)
}

class HashRing {
  // posición en el ring -> id del nodo físico
  private anillo: { pos: number; nodo: string }[] = []
  constructor(private vnodesPorNodo = 150) {}

  agregarNodo(nodo: string): void {
    for (let v = 0; v < this.vnodesPorNodo; v++) {
      this.anillo.push({ pos: h(`${nodo}#${v}`), nodo })   // muchas posiciones por nodo
    }
    this.anillo.sort((a, b) => a.pos - b.pos)
  }
  quitarNodo(nodo: string): void {
    this.anillo = this.anillo.filter(e => e.nodo !== nodo)
  }
  // primer vnode en sentido horario desde hash(clave)
  asignar(clave: string): string {
    const p = h(clave)
    for (const e of this.anillo) if (e.pos >= p) return e.nodo
    return this.anillo[0].nodo   // wrap-around: volvés al principio del círculo
  }
}

const claves = Array.from({ length: 10_000 }, (_, i) => `clave-${i}`)

const ring = new HashRing(150)
for (const n of ['n1', 'n2', 'n3', 'n4']) ring.agregarNodo(n)

const antes = new Map(claves.map(k => [k, ring.asignar(k)]))
ring.agregarNodo('n5')                                   // agrego UN nodo
let movidas = 0
for (const k of claves) if (antes.get(k) !== ring.asignar(k)) movidas++

console.log(`ring: al agregar el 5º nodo se movieron ${movidas} / 10000 claves`)
// ring: al agregar el 5º nodo se movieron ~2000 / 10000 claves  -> solo ~1/5
```

**Qué observás.** Al agregar el 5º nodo al ring, se mueven **~2000** claves (≈ `1/5` del total) — y todas van **al nodo nuevo**, no se reorganiza el resto. Compará con el ~80% del `% N`. Esa es la diferencia entre un rebalanceo que tira el sistema y uno que apenas se nota.

**Qué demuestra.** Consistent hashing (módulo 17) reduce el movimiento de `~todo` a `~1/N` porque la pertenencia de una clave depende solo de su vecino horario en el ring, no de N. Los **virtual nodes** (acá 150 por nodo) son lo que hace que el reparto sea **parejo**: probá bajar `vnodesPorNodo` a `1` y vas a ver el desbalance (algunos nodos con muchas más claves que otros) y que al caer un nodo toda su carga va a un solo vecino.

**Extensiones**
- 5.1 ✍️ Agregá un método `distribucion()` que devuelva cuántas claves tiene cada nodo; corré con `vnodesPorNodo = 1` y con `150` y compará el desbalance (máx/mín).
- 5.2 🧠 ¿Por qué con `vnodesPorNodo = 1` la caída de un nodo concentra **toda** su carga en un solo vecino, y por qué con 150 no?
- 5.3 ✍️ Implementá `quitarNodo` + medición: confirmá que al sacar un nodo solo se mueven **sus** claves (`~1/N`) y al siguiente en el ring.

---

## Soluciones de las extensiones

**1.1** D arranca en `{A:0,B:0,C:0,D:0}`. Si recibe el vector de B (`{A:2,B:1,C:0,D:0}`) y luego el de C, hace `max` posición a posición de ambos y suma 1 en D. Un evento posterior de D tendrá `≥` en todas las posiciones de B y C y `>` en D → `comparar` devuelve `'despues'` contra cualquier evento de B o C que D ya haya visto. (Si D no vio un evento, sale `'concurrente'`.)

**1.2** Porque la regla modela "primero **incorporo** lo que el mensaje me trae (lo que el emisor ya sabía), y recién **después** registro que YO hice un evento nuevo (la recepción)". Si incrementaras primero, contarías tu evento antes de incorporar el conocimiento del emisor y podrías quedar con un `V[self]` que no refleja causalidad correctamente respecto del mensaje. El `max` une el conocimiento; el `+1` posterior marca *este* evento como nuevo.

**1.3** En `comparar`, además de los flags, juntá `adelante = [k for k in claves if x[k] > y[k]]`. Si `x` y `y` son concurrentes, `adelante` (lo que x vio de más) y su simétrico (lo que y vio de más) son **ambos no vacíos** — eso *es* la concurrencia hecha explícita: cada uno vio eventos que el otro no.

**2.1** Con una réplica caída y `W=2`, el `for` saltea la caída y escribe en las 2 vivas restantes → ok. Con `W=3` y solo 2 vivas, nunca llega a 3 confirmaciones → tira `no se alcanzó W=3`. Muestra el costo de `W` alto: cada réplica caída te acerca a no poder escribir (menos disponibilidad de escritura).

**2.2** Con `W=3,R=1` (N=3): lectura **siempre fresca** desde cualquier réplica (`R+W=4>3` y además todas tienen todo). Pero si **una** réplica se cae, **no podés escribir** (`W=3` exige las 3) → disponibilidad de escritura nula ante una sola caída. Es el extremo "consistencia de lectura total, escritura frágil".

**2.3** Dos escrituras con `version:1` pero valores `'X'` y `'Y'`, cada una con `W=2`, pueden dejar réplicas con valores distintos en la **misma** versión. El `reduce` por `version` no puede desempatar (son iguales) → el quórum te garantiza "leés *una* escritura con quórum", pero no **cuál** ni que haya una sola ganadora. Sin líder que ordene, hacen falta vector clocks (build 1) para detectar que chocaron y guardarlas como siblings.

**3.1** Un rango más ancho hace menos probable que dos nodos sorteen el **mismo mínimo** (menos split votes), pero el timeout promedio (y por ende el tiempo hasta detectar ausencia de líder y elegir uno) sube. Se elige el rango balanceando: suficientemente ancho para que los empates sean raros dado N, pero acotado para que el failover no tarde demasiado (Raft típico: 150–300ms).

**3.2** Bucle: sorteás timeouts, llamás `elegir`; si `lider===null`, incrementás contador de rondas y re-sorteás; cortás cuando hay líder. Promediando muchas corridas, un rango angosto (p. ej. `[150,160]`) da más rondas (más empates); uno ancho (`[150,300]`) suele resolver en 1 ronda. Confirma 3.1 empíricamente.

**3.3** Si el líder elegido se cae justo después, deja de mandar heartbeats (`AppendEntries` vacíos, módulo 10); los followers no los reciben, vence **su** election timeout y arranca otra elección con un término mayor. El heartbeat es lo que mantiene la autoridad del líder vivo; sin él, el ciclo de elección se reactiva.

**4.1** Monotónico porque la garantía es "el escritor más nuevo tiene siempre el token **mayor**, y el recurso rechaza tokens menores que el último visto". Con un UUID aleatorio no hay orden: el recurso no podría decidir cuál token es "más nuevo" → perdería la capacidad de rechazar al escritor del pasado. La monotonía codifica el tiempo lógico de adquisición (es, de hecho, un reloj lógico del lock).

**4.2** Hacé `adquirir` async con un `await new Promise(r => setTimeout(r, 0))` antes de incrementar. Con `Promise.all([adquirir(A), adquirir(B)])`: como Node es single-thread, el incremento de `token += 1` es atómico respecto del event loop (no hay preempción en medio de la línea), así que los tokens salen `1` y `2`, nunca repetidos. (Si el servicio fuera multi-nodo de verdad, el contador necesitaría ser atómico/consensuado — p. ej. el índice del log de etcd.)

**4.3** Opciones: (a) escribir a un **path único por token** (`archivo-v{token}`) y tener un puntero "actual" actualizado con un CAS/condición que valide el token → el recurso intermedio hace de validador; (b) usar versionado de objetos + escritura condicional (`If-Match` con ETag) para que la escritura de A falle si el objeto cambió; (c) aceptar que el lock es best-effort y diseñar la operación como **idempotente**/conmutativa para que dos escritores no corrompan. En todos, alguien tiene que validar orden — si no es el recurso, lo emulás con un nivel de indirección.

**5.1** `distribucion()` recorre las claves, cuenta por nodo y devuelve el mapa. Con `vnodesPorNodo=1`, vas a ver una relación máx/mín alta (p. ej. un nodo con 4000 y otro con 1500) — desbalance fuerte. Con `150`, la relación máx/mín se acerca a ~1.1–1.2: reparto parejo por la ley de los grandes números (muchos arcos chicos promedian).

**5.2** Con 1 vnode por nodo, cada nodo ocupa **un** arco contiguo del ring; al caer, **todo** ese arco pasa al único nodo siguiente en sentido horario → ese vecino duplica (o peor) su carga (hot node). Con 150 vnodes, los arcos de un nodo están **dispersos** por todo el ring, y cada uno es seguido por un vnode de un nodo **distinto** → al caer, su carga se reparte entre **muchos** nodos, no uno.

**5.3** `quitarNodo` filtra los vnodes del nodo. Midiendo `antes`/`después` con `asignar`, solo cambian las claves que pertenecían a los vnodes del nodo quitado (van a su siguiente horario respectivo) → `~1/N` del total se mueve, y **solo** esas; las demás claves no se tocan. Simétrico a agregar.
