# Replicación y conflictos: laboratorio práctico

**Construí, rompé y arreglá: replication lag, LWW que pierde escrituras, CRDTs que convergen, outbox y el bloqueo de 2PC · Node + Go · 2026**

> Cómo usar esta guía: complemento de mano del módulo de teoría [Replicación, multi-región y conflictos](replicacion.md). Ahí entendiste los conceptos; acá **construís cinco cosas**, cada una con el método: **(1)** el problema, **(2)** código que **falla a propósito**, **(3)** qué observás, **(4)** el fix, **(5)** qué demuestra.

**Lo que asumimos.** Haber leído el [módulo de teoría](replicacion.md) y, para los conflictos, los **vector clocks** de [Consenso](consenso.md). Node 20+ y TypeScript. El Go se explica línea por línea.

**Los cinco builds**
1. **Replication lag** (read-your-writes y monotonic reads rotos, y el fix por LSN/sticky)
2. **LWW pierde una escritura** (clock skew) y la alternativa: detectar y guardar siblings
3. **CRDTs que convergen** (G-Counter, PN-Counter, OR-Set venga el orden que venga)
4. **Dual-write vs outbox** (el evento que se pierde, y cómo el outbox lo salva)
5. **2PC y el bloqueo del coordinador** (participantes in-doubt con los locks tomados)

> Convención: el código compila en `--strict`. Las **extensiones** están al final de cada build; sus soluciones, en la última sección.

---

## Build 1 — Replication lag (leer el pasado)

**El problema.** Un líder y un follower asíncrono con lag. El cliente escribe en el líder y lee del follower. Vamos a ver dos anomalías —**read-your-writes** y **monotonic reads**— y a taparlas con un timestamp lógico (LSN).

### 1a) Read-your-writes roto

```ts
// replicacion.ts
type Escritura = { lsn: number; clave: string; valor: string }

class Nodo {
  log: Escritura[] = []                       // historial ordenado por lsn
  estado = new Map<string, string>()
  aplicar(e: Escritura): void {
    this.log.push(e)
    this.estado.set(e.clave, e.valor)
  }
  lsnActual(): number { return this.log.at(-1)?.lsn ?? 0 }
  leer(clave: string): string | undefined { return this.estado.get(clave) }
}

class Lider extends Nodo {
  private seq = 0
  escribir(clave: string, valor: string): Escritura {
    const e: Escritura = { lsn: ++this.seq, clave, valor }
    this.aplicar(e)
    return e                                   // devuelve el lsn de ESTA escritura
  }
}

const lider = new Lider()
const follower = new Nodo()

// el follower replica con LAG: aplicamos al líder, pero al follower "más tarde"
const w = lider.escribir('perfil:ana', 'bio nueva')   // lsn 1, OK al cliente
// (el follower todavía NO recibió w: lag de red)

const loQueLeo = follower.leer('perfil:ana')
console.log('tras escribir, leo del follower:', loQueLeo)   // undefined  -> NO veo mi escritura
```

**Qué observás.** Escribiste `'bio nueva'` y recibiste OK (`lsn 1`), pero leer del follower devuelve `undefined`: el follower todavía no recibió la escritura. Para el usuario es "guardé y desapareció". Es read-your-writes: tu propia escritura no se ve.

### 1b) El fix: read-your-writes por LSN

El cliente recuerda el `lsn` de su última escritura y **exige** una réplica al menos tan nueva; si el follower está atrasado, cae al líder.

```ts
// (mismo Lider/Nodo de 1a)
function leerConsistente(
  clave: string,
  lsnRequerido: number,
  follower: Nodo,
  lider: Lider,
): string | undefined {
  if (follower.lsnActual() >= lsnRequerido) return follower.leer(clave)  // el follower ya alcanzó
  return lider.leer(clave)                                               // si no, leé del líder
}

const lider2 = new Lider()
const follower2 = new Nodo()
const w2 = lider2.escribir('perfil:ana', 'bio nueva')      // lsn 1
// follower2 sigue en lsn 0 (lag)
console.log('lsn follower:', follower2.lsnActual())         // 0
const visto = leerConsistente('perfil:ana', w2.lsn, follower2, lider2)
console.log('read-your-writes:', visto)                     // 'bio nueva'  -> SIEMPRE veo lo mío
```

**Qué observás.** Como el cliente pasa `w2.lsn = 1` y el follower está en `0 < 1`, la lectura cae al líder y devuelve `'bio nueva'`. La garantía es sobre **tus** escrituras, sin obligar consistencia fuerte global: solo subís al líder cuando la réplica no alcanzó tu propio LSN.

### 1c) Monotonic reads roto (el tiempo va para atrás)

```ts
// dos followers con distinto lag: leer de uno y luego del otro retrocede
const followerRapido = new Nodo()
const followerLento = new Nodo()
const e1: Escritura = { lsn: 1, clave: 'post:1', valor: 'hola' }
followerRapido.aplicar(e1)                 // el rápido ya tiene el post
// el lento todavía no

// dos lecturas seguidas del MISMO cliente: el balanceador mandó la 1ª al rápido y la 2ª al lento
const lectura1 = followerRapido.leer('post:1')   // 'hola'   (cae en el rápido)
const lectura2 = followerLento.leer('post:1')    // undefined (cae en el lento) -> ¡retrocedió!
console.log({ lectura1, lectura2 })              // { lectura1: 'hola', lectura2: undefined }
```

**Qué observás.** Dos lecturas seguidas del "mismo" dato dan `'hola'` y luego `undefined`: el usuario vio el post y al refrescar desapareció. El fix de monotonic reads: rutear cada usuario **siempre a la misma réplica** (sticky por hash del user-id) → nunca salta a una más atrasada. (Extensión 1.3.)

**Qué demuestra.** El lag no es un bug: es la consistencia eventual. Las garantías de **sesión** (read-your-writes por LSN, monotonic por sticky) tapan la anomalía concreta sin pagar linealizabilidad global — la jugada del módulo 5.

**Extensiones**
- 1.1 ✍️ Agregá una `replicar(follower, lider)` que copie del log del líder hasta un lsn dado, y mostrá que tras replicar, `leerConsistente` ya lee del follower (no sube al líder).
- 1.2 🧠 ¿Por qué `leerConsistente` no rompe la escalabilidad de lectura en el caso común (follower al día)?
- 1.3 ✍️ Implementá `replicaParaUsuario(userId, replicas)` con sticky por hash y mostrá que dos lecturas del mismo usuario nunca retroceden.

---

## Build 2 — LWW pierde una escritura

**El problema.** Dos réplicas aceptan escrituras concurrentes a la misma clave (multi-leader). Con **Last-Write-Wins por timestamp físico** y un poco de **clock skew**, vamos a ver cómo se descarta la escritura que de verdad fue última. Después: detectarlo con vector clocks y **no** perder datos.

### 2a) LWW con clock skew → escritura perdida

```ts
// lww.ts
type ValorLWW = { valor: string; ts: number }   // ts = timestamp físico (reloj de pared)

function mergeLWW(a: ValorLWW, b: ValorLWW): ValorLWW {
  return a.ts >= b.ts ? a : b                     // gana el timestamp más alto
}

// Réplica A tiene el reloj ADELANTADO 100ms; B está en hora.
// Cronología real: primero escribe A (t_real=1000), DESPUÉS escribe B (t_real=1050).
const escrituraA: ValorLWW = { valor: 'titulo-A', ts: 1000 + 100 }  // reloj de A: 1100
const escrituraB: ValorLWW = { valor: 'titulo-B', ts: 1050 }         // reloj de B: 1050 (real, posterior)

const ganador = mergeLWW(escrituraA, escrituraB)
console.log('LWW se queda con:', ganador.valor)   // 'titulo-A'  <- ¡MAL! B fue la última real
```

**Qué observás.** B se escribió **después** en el tiempo real (1050 > 1000), pero como el reloj de A estaba adelantado 100ms, su timestamp (1100) es mayor, y LWW se queda con `'titulo-A'`. **Se perdió la escritura que de verdad fue última**, silenciosamente. El reloj decidió, no la causalidad.

### 2b) El fix: detectar concurrencia con vector clocks y guardar ambas

Reusamos la idea de [vector clocks de Consenso](consenso.md): si las versiones son concurrentes, **no** elegimos una — guardamos las dos (siblings) y dejamos que la app/usuario mergee.

```ts
// siblings.ts
type Vector = Record<string, number>
type Version = { valor: string; vc: Vector }

function comparar(x: Vector, y: Vector): 'antes' | 'despues' | 'concurrente' | 'igual' {
  const claves = new Set([...Object.keys(x), ...Object.keys(y)])
  let menor = false, mayor = false
  for (const k of claves) {
    const xv = x[k] ?? 0, yv = y[k] ?? 0
    if (xv < yv) menor = true
    if (xv > yv) mayor = true
  }
  if (menor && mayor) return 'concurrente'
  if (menor) return 'antes'
  if (mayor) return 'despues'
  return 'igual'
}

// guarda una versión nueva: descarta las dominadas, conserva las concurrentes (siblings)
function guardar(siblings: Version[], nueva: Version): Version[] {
  const resultado: Version[] = []
  let dominada = false
  for (const s of siblings) {
    const rel = comparar(nueva.vc, s.vc)
    if (rel === 'antes' || rel === 'igual') dominada = true   // la nueva es vieja/igual: no aporta
    if (rel === 'despues') continue                           // la nueva domina a s: descarto s
    resultado.push(s)                                         // concurrente o domina: conservo s
  }
  if (!dominada) resultado.push(nueva)
  return resultado
}

// A escribe con vc {A:1}; B escribe CONCURRENTE con vc {B:1} (ninguno vio al otro)
let estado: Version[] = []
estado = guardar(estado, { valor: 'titulo-A', vc: { A: 1 } })
estado = guardar(estado, { valor: 'titulo-B', vc: { B: 1 } })
console.log('siblings:', estado.map(v => v.valor))   // ['titulo-A', 'titulo-B']  -> NO se pierde nada

// más tarde, alguien mergea y escribe una versión que VIO a ambas: vc {A:1, B:1, C:1}
estado = guardar(estado, { valor: 'titulo-merged', vc: { A: 1, B: 1, C: 1 } })
console.log('tras merge:', estado.map(v => v.valor))  // ['titulo-merged']  -> domina a ambas
```

**Qué observás.** Las dos escrituras concurrentes se conservan como **siblings** (`['titulo-A', 'titulo-B']`) en vez de perder una. Cuando llega una versión que vio a ambas (`{A:1,B:1,C:1}`), **domina** a las dos y las reemplaza: el conflicto se resolvió sin descartar nada por el camino.

**Qué demuestra.** El módulo 9 en acción: LWW es simple pero pierde datos por clock skew; detectar concurrencia con vector clocks te deja **elegir** la política (guardar siblings, mergear) en vez de que el reloj decida por vos. Usás LWW solo si perder una escritura concurrente es tolerable.

**Extensiones**
- 2.1 🧠 ¿Por qué `mergeLWW` es determinista (todas las réplicas convergen al mismo valor) **a pesar** de perder datos? ¿Por qué eso lo hace un CRDT válido (LWW-Register) aunque "pierda"?
- 2.2 ✍️ Agregá a `guardar` un desempate determinístico para cuando dos siblings quedan concurrentes y querés colapsar a uno (p. ej. mayor valor lexicográfico) — y discutí qué perdés.
- 2.3 🧠 Mostrá una secuencia donde A escribe, B **lee y luego escribe** (vio a A): ¿por qué ahí NO hay conflicto y B gana limpio?

---

## Build 3 — CRDTs que convergen

**El problema.** Dos réplicas reciben actualizaciones en **distinto orden** (y algunas repetidas, porque la red duplica). Vamos a construir tres CRDTs y verificar que **siempre convergen al mismo estado**, sin coordinación.

### 3a) G-Counter y PN-Counter

```ts
// crdt-counter.ts
type GCounter = Record<string, number>   // una casilla por réplica

const gInc = (g: GCounter, replica: string, n = 1): GCounter =>
  ({ ...g, [replica]: (g[replica] ?? 0) + n })

const gMerge = (a: GCounter, b: GCounter): GCounter => {
  const out: GCounter = { ...a }
  for (const k of Object.keys(b)) out[k] = Math.max(out[k] ?? 0, b[k])   // max casilla a casilla
  return out
}
const gValor = (g: GCounter): number => Object.values(g).reduce((s, n) => s + n, 0)

// Réplica A y B incrementan en paralelo, sin verse
let a: GCounter = {}, b: GCounter = {}
a = gInc(a, 'A'); a = gInc(a, 'A')   // A suma 2
b = gInc(b, 'B')                      // B suma 1

// se mergean en CUALQUIER orden -> mismo resultado
const m1 = gMerge(gMerge(a, b), a)   // con un duplicado de 'a' encima (idempotente)
const m2 = gMerge(b, a)
console.log('valor m1:', gValor(m1), 'valor m2:', gValor(m2))   // 3 y 3  -> convergen
```

```ts
// PN-Counter: dos G-Counters (P incrementos, N decrementos)
type PNCounter = { P: GCounter; N: GCounter }

const pnInc = (c: PNCounter, r: string, n = 1): PNCounter => ({ P: gInc(c.P, r, n), N: c.N })
const pnDec = (c: PNCounter, r: string, n = 1): PNCounter => ({ P: c.P, N: gInc(c.N, r, n) })
const pnMerge = (x: PNCounter, y: PNCounter): PNCounter => ({ P: gMerge(x.P, y.P), N: gMerge(x.N, y.N) })
const pnValor = (c: PNCounter): number => gValor(c.P) - gValor(c.N)

let c: PNCounter = { P: {}, N: {} }
c = pnInc(c, 'A', 5)   // +5
c = pnDec(c, 'A', 2)   // -2
console.log('PN-Counter:', pnValor(c))   // 3
```

**Qué observás.** `m1` y `m2` dan **3** aunque se mergearon en distinto orden y con un duplicado de `a` encima (el `max` es idempotente: re-mergear no cambia nada). El PN-Counter da `3` (`5-2`) decrementando vía el contador `N`.

### 3b) OR-Set (add-wins)

```ts
// or-set.ts — cada add lleva un tag único; remove borra solo los tags observados
type ORSet = { adds: Map<string, Set<string>>; tombstones: Set<string> }   // elemento -> tags

const crearORSet = (): ORSet => ({ adds: new Map(), tombstones: new Set() })

function orAdd(s: ORSet, elem: string, tag: string): void {
  if (!s.adds.has(elem)) s.adds.set(elem, new Set())
  s.adds.get(elem)!.add(tag)                       // tag único por add (réplica+contador)
}
function orRemove(s: ORSet, elem: string): void {
  const tags = s.adds.get(elem)
  if (tags) for (const t of tags) s.tombstones.add(t)   // mata SOLO los tags que veo ahora
}
function orTiene(s: ORSet, elem: string): boolean {
  const tags = s.adds.get(elem)
  if (!tags) return false
  return [...tags].some(t => !s.tombstones.has(t))      // vive si queda algún tag no-tombstoneado
}
function orMerge(a: ORSet, b: ORSet): ORSet {
  const out = crearORSet()
  for (const src of [a, b]) {
    for (const [elem, tags] of src.adds) {
      if (!out.adds.has(elem)) out.adds.set(elem, new Set())
      for (const t of tags) out.adds.get(elem)!.add(t)
    }
    for (const t of src.tombstones) out.tombstones.add(t)
  }
  return out
}

// A remueve 'x' (tag t1) mientras B lo agrega de nuevo CONCURRENTE (tag t2)
const ra = crearORSet(), rb = crearORSet()
orAdd(ra, 'x', 't1'); orAdd(rb, 'x', 't1')   // ambas vieron el add original t1
orRemove(ra, 'x')                              // A borra: tombstone t1
orAdd(rb, 'x', 't2')                           // B re-agrega CONCURRENTE: tag nuevo t2

const merged = orMerge(ra, rb)
console.log("'x' tras merge:", orTiene(merged, 'x'))   // true  -> ADD-WINS (t2 no fue observado por el remove)
```

**Qué observás.** Tras mergear un `remove` (A) concurrente con un `add` (B), `'x'` **sigue en el set**: el remove solo mató el tag `t1` que había observado, pero el add de B trajo `t2`, que sobrevive → **add-wins**. Un set naíf (con un booleano) habría perdido el resultado según el orden de merge; el OR-Set converge sin ambigüedad.

**Qué demuestra.** Los módulos 10-11: conmutatividad + idempotencia (el `max`, la unión de tags) hacen que el orden y los duplicados no importen → Strong Eventual Consistency. Y el OR-Set muestra el truco de los tags para que add/remove concurrentes tengan una semántica definida (add-wins) en vez de pisarse.

**Extensiones**
- 3.1 🧠 ¿Por qué un contador implementado como un solo número entero `n` con merge `max(a,b)` **no** funciona como CRDT cuando dos réplicas incrementan en paralelo? (Mostralo: A y B parten de 5, cada uno suma 1 → merge da 6, no 7.)
- 3.2 ✍️ Implementá `gValor`/`pnValor` y verificá que `pnMerge` es conmutativo: `pnMerge(x,y)` y `pnMerge(y,x)` dan el mismo valor.
- 3.3 🧠 El OR-Set acumula tombstones para siempre. ¿Qué problema operativo trae a largo plazo y cómo lo atacan los sistemas reales?

---

## Build 4 — Dual-write vs outbox

**El problema.** Una operación tiene que **guardar el pedido en la DB** y **publicar un evento** en un broker. Vamos a ver el dual-write perder el evento cuando el proceso muere en el medio, y al outbox arreglarlo escribiendo el evento en la **misma** transacción.

### 4a) Dual-write → evento perdido

```ts
// dual-write.ts
class DB {
  pedidos: string[] = []
  eventos: string[] = []                      // tabla outbox (la usamos en 4b)
  tx<T>(fn: () => T): T { return fn() }        // transacción local atómica (todo o nada)
}
class Broker { publicados: string[] = []; publicar(e: string): void { this.publicados.push(e) } }

const db = new DB(), broker = new Broker()

function crearPedidoDualWrite(id: string, matarAntesDePublicar: boolean): void {
  db.tx(() => db.pedidos.push(id))            // (1) commit local: pedido guardado
  if (matarAntesDePublicar) throw new Error('💥 el proceso muere acá')   // crash entre los dos writes
  broker.publicar(`PedidoCreado:${id}`)        // (2) NUNCA llega
}

try { crearPedidoDualWrite('p1', true) } catch (e) { console.log((e as Error).message) }
console.log('pedidos en DB:', db.pedidos)            // ['p1']            <- existe
console.log('eventos publicados:', broker.publicados) // []               <- NUNCA se publicó
// inconsistencia: el pedido existe pero nadie se entera (no se cobra, no se envía)
```

**Qué observás.** El pedido `p1` quedó en la DB pero el evento **nunca** se publicó: el proceso murió entre el commit local y el `publicar`. Nadie aguas abajo se entera del pedido. No hay try/catch que lo salve: el crash puede caer justo en el medio.

### 4b) El fix: outbox (el evento viaja en la misma transacción)

```ts
// outbox.ts — (mismos DB/Broker de 4a)
const db2 = new DB(), broker2 = new Broker()

function crearPedidoOutbox(id: string): void {
  db2.tx(() => {
    db2.pedidos.push(id)                       // pedido
    db2.eventos.push(`PedidoCreado:${id}`)     // evento EN LA MISMA tx -> single-write atómico
  })
}

// proceso relay aparte: lee la outbox y publica (reintentable e idempotente)
function relayOutbox(): void {
  while (db2.eventos.length > 0) {
    const e = db2.eventos[0]                    // sin sacarlo todavía
    broker2.publicar(e)                          // si esto falla, el evento sigue en la tabla -> reintento
    db2.eventos.shift()                          // recién al confirmar el publish, lo saco
  }
}

crearPedidoOutbox('p2')
console.log('antes del relay -> outbox:', db2.eventos)   // ['PedidoCreado:p2']  (durable en la DB)
relayOutbox()
console.log('pedidos:', db2.pedidos, '| publicados:', broker2.publicados)
// pedidos: ['p2'] | publicados: ['PedidoCreado:p2']   -> nunca hay pedido sin evento
```

**Qué observás.** El pedido y el evento se escriben **atómicamente** (misma transacción): si el proceso muere, o están **los dos** o **ninguno**. El evento queda durable en la tabla outbox; el relay lo publica después, y si el publish falla, el evento **sigue ahí** para reintentar. Nunca hay "pedido sin evento".

**Qué demuestra.** El módulo 12: el dual-write no se arregla con orden ni try/catch, se **elimina** convirtiéndolo en un single-write local (outbox) + publicación asíncrona reintentable. Es lo que se usa en serio (con CDC/Debezium leyendo el WAL en vez de un poller). Conecta con [event-driven](event-driven.md).

**Extensiones**
- 4.1 🧠 El relay publica **at-least-once** (si muere tras publicar pero antes del `shift`, re-publica). ¿Por qué eso obliga a que el consumidor sea **idempotente**?
- 4.2 ✍️ Agregá un `id` único por evento y un `Set` de procesados en el consumidor para deduplicar; mostrá que re-publicar el mismo evento no duplica el efecto.
- 4.3 🧠 ¿Qué ventaja tiene leer la outbox con **CDC sobre el WAL** en vez de un poller que hace `SELECT` cada X ms?

---

## Build 5 — 2PC y el bloqueo del coordinador

**El problema.** Un coordinador y dos participantes quieren commitear atómicamente. Vamos a correr el happy path y después **matar al coordinador justo después del prepare** para ver el bloqueo: los participantes quedan in-doubt, **con los locks tomados**, sin poder decidir solos.

### 5a) 2PC y el blocking problem (Go, donde el bloqueo concurrente se ve claro)

```go
// dos_pc.go
package main

import (
	"errors"
	"fmt"
)

type Participante struct {
	nombre   string
	bloqueado bool   // true entre prepare y la decisión: lock tomado, in-doubt
	commited bool
}

// prepare: hace el trabajo, toma el lock y vota. Queda in-doubt hasta recibir la decisión.
func (p *Participante) prepare() (bool, error) {
	p.bloqueado = true                       // 🔒 toma el lock AHORA
	fmt.Printf("  %s: preparado (lock tomado, in-doubt), voto YES\n", p.nombre)
	return true, nil
}
func (p *Participante) commit() {
	p.commited = true
	p.bloqueado = false                      // 🔓 libera el lock SOLO al recibir la decisión
	fmt.Printf("  %s: COMMIT, lock liberado\n", p.nombre)
}

func coordinador(parts []*Participante, coordinadorMuereTrasPrepare bool) error {
	fmt.Println("Fase 1 — PREPARE:")
	for _, p := range parts {
		ok, _ := p.prepare()
		if !ok {
			return errors.New("alguien votó NO -> ABORT")
		}
	}
	if coordinadorMuereTrasPrepare {
		fmt.Println("💥 el coordinador MUERE antes de mandar la decisión")
		return errors.New("coordinador caído: participantes in-doubt")
	}
	fmt.Println("Fase 2 — COMMIT:")
	for _, p := range parts {
		p.commit()
	}
	return nil
}

func main() {
	fmt.Println("=== Happy path ===")
	a, b := &Participante{nombre: "A"}, &Participante{nombre: "B"}
	_ = coordinador([]*Participante{a, b}, false)

	fmt.Println("\n=== Coordinador muere tras prepare ===")
	c, d := &Participante{nombre: "C"}, &Participante{nombre: "D"}
	err := coordinador([]*Participante{c, d}, true)
	fmt.Println("error:", err)
	// los participantes quedaron bloqueados: votaron YES pero no saben la decisión
	fmt.Printf("estado C: bloqueado=%v commited=%v\n", c.bloqueado, c.commited)
	fmt.Printf("estado D: bloqueado=%v commited=%v\n", d.bloqueado, d.commited)
	// C y D NO pueden decidir solos: no saben si el otro votó YES -> locks tomados indefinidamente
}
```

**Qué observás.** En el happy path, todos preparan, votan YES, y commitean liberando los locks. Pero cuando el coordinador muere tras el prepare, `C` y `D` quedan `bloqueado=true commited=false`: **votaron YES (locks tomados) pero no saben la decisión global**. No pueden commitear (quizás alguien votó NO) ni abortar (quizás todos votaron YES y hay un COMMIT en camino). Quedan **in-doubt, bloqueados, hasta que el coordinador vuelva** — y mientras, sus locks bloquean a todos los demás.

### 5b) El "fix" no es arreglar 2PC: es no usarlo (saga)

No hay un parche que haga a 2PC no-bloqueante en una red real (3PC asume red síncrona y tampoco sirve — módulo 13). La salida de producción es **rediseñar sin commit atómico distribuido**: una **saga** de transacciones locales con compensación (cada participante commitea **solo** localmente, sin quedar in-doubt; si un paso falla, se compensan los anteriores). En código, cada paso es un `db.tx` local + un evento (outbox del build 4), nunca un lock distribuido esperando una decisión remota.

```ts
// saga.ts — bosquejo del contraste: pasos locales + compensación, sin in-doubt
type Paso = { hacer: () => void; compensar: () => void }

function ejecutarSaga(pasos: Paso[]): boolean {
  const hechos: Paso[] = []
  for (const p of pasos) {
    try { p.hacer(); hechos.push(p) }            // cada paso COMMITEA local (no queda in-doubt)
    catch {
      for (const h of hechos.reverse()) h.compensar()   // falló: deshago hacia atrás
      return false
    }
  }
  return true
}

const ok = ejecutarSaga([
  { hacer: () => console.log('reservar stock'),  compensar: () => console.log('liberar stock') },
  { hacer: () => console.log('cobrar pago'),     compensar: () => console.log('reembolsar') },
])
console.log('saga ok:', ok)   // cada paso vive o se compensa; NADIE queda bloqueado in-doubt
```

**Qué demuestra.** El módulo 13-14: 2PC da atomicidad real pero el coordinador es un SPOF y el prepare deja a todos in-doubt con locks → un freeze que se propaga. Por eso los sistemas distribuidos de alta disponibilidad evitan 2PC y usan **saga + outbox + idempotencia**: pierden el aislamiento (hay estados intermedios visibles) pero nadie queda bloqueado, y el estado vive en logs reintentables, no en un coordinador frágil.

**Extensiones**
- 5.1 🧠 ¿Por qué un participante in-doubt no puede simplemente "abortar por timeout" para liberar el lock? ¿Qué se rompería?
- 5.2 🧠 La saga sacrifica el **aislamiento** que 2PC sí da. Dá un ejemplo concreto de un estado intermedio visible y cómo lo harías tolerable (estado explícito / reserva con TTL).
- 5.3 ✍️ Agregá a `ejecutarSaga` que un paso de compensación pueda **fallar** y mostrá por qué las compensaciones también deben ser idempotentes y reintentables.

---

## Soluciones de las extensiones

**1.1** `replicar(follower, lider, hastaLsn)` recorre `lider.log` y aplica al follower las escrituras con `lsn ≤ hastaLsn` que le falten. Tras `replicar(follower2, lider2, w2.lsn)`, `follower2.lsnActual()` pasa a `1 ≥ 1`, así que `leerConsistente` toma la rama del follower y devuelve `'bio nueva'` **sin** ir al líder. Muestra que el fallback al líder es solo transitorio (mientras dura el lag).

**1.2** Porque en régimen normal el lag es de ms y el follower **casi siempre** ya alcanzó el lsn requerido → la lectura va al follower (escala). Solo cae al líder en la ventana corta tras una escritura propia, o si la réplica está muy atrasada. No es "todo al líder": es "al líder solo cuando tu réplica todavía no te alcanzó".

**1.3** `replicaParaUsuario` hace `replicas[hash(userId) % replicas.length]` → el mismo usuario siempre pega a la misma réplica. Como esa réplica solo avanza (su lsn no retrocede), dos lecturas seguidas del usuario nunca ven un estado más viejo que el anterior → monotonía. (No garantiza ver lo último, solo no retroceder.)

**2.1** Es determinista porque `mergeLWW` es una función pura del par `(valor, ts)`: dadas las mismas dos versiones, **todas** las réplicas eligen la misma (mayor ts, con un desempate fijo por id ante empate). Por eso converge y es un CRDT válido (LWW-Register): cumple conmutatividad/idempotencia. "Perder" una escritura concurrente no rompe la convergencia — simplemente la política elegida descarta una; converger y no-perder son cosas distintas.

**2.2** Agregás: si quedan dos siblings concurrentes y querés colapsar, ordenás por `(valor)` y te quedás con el mayor lexicográfico (determinístico en todas las réplicas). Perdés la otra escritura (igual que LWW), pero al menos es **explícito** y bajo tu control, no a merced del reloj. Solo hacelo si perder esa escritura es tolerable para el dato.

**2.3** A escribe `{A:1}`. B **lee** (ve `{A:1}`) y luego escribe incorporando ese conocimiento: `{A:1, B:1}`. `comparar({A:1,B:1}, {A:1})` = `'despues'` → B **domina** a A (vio todo lo de A y agregó lo suyo). No es concurrente: hay relación causal (B vio a A), así que B gana limpio sin conflicto ni siblings.

**3.1** Un solo entero con `max` no cuenta incrementos paralelos: A y B parten de 5, cada uno suma 1 localmente → ambos tienen 6 → `max(6,6)=6`, pero el valor correcto es **7** (dos incrementos). El `max` no puede distinguir "el mismo incremento que ya vi" de "otro incremento distinto". El G-Counter lo arregla dando a cada réplica **su propia casilla**: partiendo de `{base:5}`, A sube la suya (`{base:5, A:1}`) y B la suya (`{base:5, B:1}`); el merge hace `max` **por casilla** → `{base:5, A:1, B:1}` y el valor es la **suma** = `5+1+1 = 7`. La clave es separar las contribuciones por réplica para que el `max` nunca tenga que distinguir un incremento de otro.

**3.2** `gValor` suma las casillas; `pnValor` = `gValor(P) - gValor(N)`. Como `gMerge` usa `max` (conmutativo) en cada casilla, `pnMerge(x,y)` y `pnMerge(y,x)` producen los mismos `P` y `N` → mismo valor. Verificable con cualquier par de estados.

**3.3** Los tombstones (tags borrados) **crecen sin límite** → el estado del set se infla con basura de elementos ya removidos. Los sistemas reales lo atacan con **garbage collection** de tombstones una vez que se sabe que todas las réplicas los vieron (estabilidad causal / version vectors), o con variantes que compactan tags. Es el costo oculto de los CRDTs de conjunto.

**4.1** Porque si el relay muere **después** de `publicar(e)` pero **antes** del `shift`, al reiniciar vuelve a publicar el mismo evento → el broker entrega el evento **dos veces** (at-least-once). Si el consumidor no es idempotente (p. ej. "cobrar"), cobraría dos veces. Idempotencia = procesar el duplicado no cambia el resultado.

**4.2** Cada evento lleva un `id` (uuid). El consumidor guarda los `id` ya procesados en un `Set` (o una tabla con constraint único): al recibir uno, si su `id` ya está, lo ignora. Re-publicar el mismo evento → el consumidor lo deduplica → el efecto (cobro) ocurre una sola vez. (En prod, el "Set" es una tabla de idempotencia con `INSERT ... ON CONFLICT`.)

**4.3** CDC sobre el WAL lee los cambios **directamente del log de la DB** (Debezium) → no hay polling (menos latencia y menos carga: no hacés `SELECT` cada X ms contra una tabla caliente), capturás **todos** los cambios en orden sin perder ninguno entre polls, y no competís con la carga transaccional. El poller es más simple pero agrega latencia (el intervalo) y presión sobre la DB.

**5.1** Si un participante abortara por timeout, podría abortar **mientras el coordinador ya decidió COMMIT** (los demás sí lo recibieron) → unos commitean y otro abortó: **se rompe la atomicidad** (el todo-o-nada). El participante in-doubt no puede saber si su timeout coincide con "coordinador caído" o con "decisión COMMIT en camino que aún no llegó" → no puede decidir unilateralmente sin arriesgar inconsistencia. Por eso queda bloqueado.

**5.2** Estado intermedio visible: tras "reservar stock" pero antes de "cobrar pago", otro cliente ve el stock **ya descontado** aunque la compra podría fallar y compensarse. Lo hacés tolerable con un estado explícito (`RESERVADO` con **TTL**: si la saga no confirma en N minutos, el stock se libera solo) en vez de descontarlo como "vendido". Así el estado parcial no engaña y se auto-recupera.

**5.3** Si `compensar()` puede fallar, `ejecutarSaga` tiene que **reintentarlo** (no podés dejar una compensación a medias: quedaría stock reservado para siempre o un cobro sin reembolsar). Como reintentás, la compensación debe ser **idempotente** (reembolsar dos veces no devuelve doble) y **reintentable** (persistir qué compensaciones faltan, p. ej. en un log/outbox, y reintentarlas hasta que tengan éxito — un workflow engine como Temporal hace justo esto).
