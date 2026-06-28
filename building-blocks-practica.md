# Building blocks: laboratorio práctico

**Construí, rompé y arreglá: Bloom filter, Count-Min, HyperLogLog, leaderboard y geohash · Node + Go · 2026**

> Cómo usar esta guía: complemento de mano del módulo de teoría [Building blocks y diseño geoespacial](building-blocks.md). Ahí entendiste los conceptos; acá **construís cinco cosas**, cada una con el método: **(1)** el problema, **(2)** código, **(3)** qué observás, **(4)** la lección, **(5)** qué demuestra.

**Lo que asumimos.** Haber leído el [módulo de teoría](building-blocks.md). Node 20+ y TypeScript. El Go se explica línea por línea.

**Los cinco builds**
1. **Bloom filter** (membresía: cero falsos negativos, falsos positivos acotados)
2. **Count-Min Sketch** (frecuencias que sobreestiman, nunca subestiman)
3. **HyperLogLog** (cardinalidad en memoria fija, cuente mil o un millón)
4. **Leaderboard** (top-K y rank, el rank que SQL hace mal)
5. **Geohash** (proximidad por prefijo, y el edge case del borde)

> Convención: el código compila en `--strict` y las salidas de los comentarios están **verificadas ejecutando**. Las **extensiones** están al final de cada build; sus soluciones, en la última sección.

> Varios builds usan un hash de 32 bits con buena mezcla (FNV-1a + finalizador tipo murmur). La mezcla importa: las estructuras probabilísticas asumen bits ~uniformes.
> ```ts
> // hash.ts — FNV-1a + finalizador (reusado en builds 1, 2 y 3)
> export function h32(s: string, seed = 0): number {
>   let h = (2166136261 ^ seed) >>> 0
>   for (let i = 0; i < s.length; i++) { h ^= s.charCodeAt(i); h = Math.imul(h, 16777619) >>> 0 }
>   h ^= h >>> 16; h = Math.imul(h, 2246822507) >>> 0
>   h ^= h >>> 13; h = Math.imul(h, 3266489909) >>> 0; h ^= h >>> 16
>   return h >>> 0
> }
> ```

---

## Build 1 — Bloom filter

**El problema.** Queremos un filtro barato para "¿`x` ya lo vimos?" sin guardar todos los elementos. Vamos a ver que un Bloom **nunca** da un falso negativo y que sus falsos positivos están acotados.

```ts
// bloom.ts
import { h32 } from './hash.js'

class BloomFilter {
  private bits: Uint8Array
  constructor(private m: number, private k: number) { this.bits = new Uint8Array(m) }

  // k posiciones por doble hashing: a + i*b (mod m) — evita calcular k hashes independientes
  private posiciones(x: string): number[] {
    const a = h32(x, 1), b = h32(x, 2)
    const r: number[] = []
    for (let i = 0; i < this.k; i++) r.push((a + i * b) % this.m)
    return r
  }
  add(x: string): void { for (const i of this.posiciones(x)) this.bits[i] = 1 }
  has(x: string): boolean { return this.posiciones(x).every(i => this.bits[i] === 1) }
}

const bf = new BloomFilter(1000, 5)   // 1000 bits, 5 hashes
for (let i = 0; i < 100; i++) bf.add(`item${i}`)   // agrego 100 elementos

// (1) CERO falsos negativos: todo lo agregado da true
const falsosNeg = Array.from({ length: 100 }, (_, i) => `item${i}`).filter(x => !bf.has(x)).length
console.log('falsos negativos:', falsosNeg)         // 0  -> NUNCA falla en el "sí"

// (2) falsos positivos acotados: cosas NO agregadas que dan true
let fp = 0
for (let i = 1000; i < 11000; i++) if (bf.has(`item${i}`)) fp++
console.log(`falsos positivos: ${fp} de 10000 (${(fp / 100).toFixed(1)}%)`)   // 89 (0.9%)

console.log("has('item42'):", bf.has('item42'))     // true  (agregado)
console.log("has('itemXYZ'):", bf.has('itemXYZ'))   // false (definitivamente no está)
```

**Qué observás.** **Cero** falsos negativos: los 100 elementos agregados dan `true` siempre (si está, el Bloom no lo niega nunca). Y `~0.9%` de falsos positivos: de 10 000 elementos **no** agregados, 89 dan `true` por casualidad (sus 5 bits estaban prendidos por otros). `has('itemXYZ')` da `false` → "definitivamente no está", la respuesta exacta.

**Qué demuestra.** El módulo 4: la asimetría del Bloom — "quizás está" (con falsos positivos) vs "seguro no está" (exacto). Por eso sirve de **filtro previo barato**: si dice "no", te ahorrás ir al disco/red; si dice "quizás", verificás. El fill ratio (cuántos bits en 1) controla la tasa de FP: más elementos → más FP (extensión 1.2).

**Extensiones**
- 1.1 🧠 ¿Por qué `has` usa `.every` (todos los bits en 1) y no `.some`? ¿Qué pasaría con `.some`?
- 1.2 ✍️ Agregá 900 elementos más (1000 en total, el mismo `m=1000`) y mostrá cómo la tasa de falsos positivos se dispara (Bloom saturado, módulo 4).
- 1.3 🧠 ¿Por qué el doble hashing (`a + i*b`) es válido en vez de calcular `k` funciones de hash independientes?

---

## Build 2 — Count-Min Sketch

**El problema.** Contar frecuencias en un stream enorme sin un contador por elemento. Vamos a ver que el Count-Min **sobreestima pero nunca subestima**, y que sirve para encontrar los más frecuentes.

```ts
// count-min.ts
import { h32 } from './hash.js'

class CountMin {
  private m: Uint32Array[]
  constructor(private d: number, private w: number) {
    this.m = Array.from({ length: d }, () => new Uint32Array(w))   // d filas × w columnas
  }
  add(x: string, n = 1): void {
    for (let i = 0; i < this.d; i++) this.m[i][h32(x, i + 1) % this.w] += n   // +1 en cada fila
  }
  estimar(x: string): number {
    let min = Infinity
    for (let i = 0; i < this.d; i++) min = Math.min(min, this.m[i][h32(x, i + 1) % this.w])
    return min   // el MÍNIMO: el contador menos contaminado por colisiones
  }
}

const cms = new CountMin(4, 64)   // 4 hashes, 64 columnas
cms.add('/home', 500); cms.add('/about', 10); cms.add('/home', 300)   // /home real = 800, /about = 10
for (let i = 0; i < 200; i++) cms.add(`/p${i}`, 1)                     // 200 páginas de ruido

console.log("estimar('/home') (real 800):", cms.estimar('/home'))     // 800
console.log("estimar('/about') (real 10):", cms.estimar('/about'))    // 13  -> sobreestima por colisión
```

**Qué observás.** `estimar('/home')` da **800** (exacto, su contador es tan grande que el ruido no lo mueve), y `estimar('/about')` da **13** (real 10 — **sobreestima** por 3 por colisiones con el ruido, pero **nunca** da menos de 10). El error es siempre **hacia arriba**.

**Qué demuestra.** El módulo 5: el Count-Min estima frecuencias con una matriz fija chica, devolviendo el **mínimo** de los `d` contadores (el menos contaminado). Sobreestima por colisiones pero nunca subestima → perfecto para **heavy hitters** ("¿las URLs más frecuentes?") donde los grandes (como `/home`) dominan y un poco de ruido en los chicos no importa. Más columnas (`w`) = menos colisiones = más preciso (extensión 2.2).

**Extensiones**
- 2.1 🧠 ¿Por qué `/home` sale exacto (800) y `/about` sobreestima (13)? ¿Qué tiene que ver el tamaño del valor real?
- 2.2 ✍️ Bajá `w` a 8 y mostrá que la sobreestimación de `/about` empeora; subilo a 512 y mostrá que se acerca a 10.
- 2.3 🧠 Para sacar el **top-K** real necesitás algo más que el Count-Min. ¿Qué estructura combinarías y por qué?

---

## Build 3 — HyperLogLog

**El problema.** Contar elementos **únicos** (cardinalidad) con memoria **fija y diminuta**, cuente mil o un millón. Vamos a construir un HLL simplificado y ver que el error es ~1% y la memoria **no crece** con la cantidad.

```ts
// hll.ts
import { h32 } from './hash.js'

function clz32(x: number): number {   // count leading zeros de un uint32
  if (x === 0) return 32
  let n = 0
  while ((x & 0x80000000) === 0) { x = (x << 1) >>> 0; n++ }
  return n
}

class HyperLogLog {
  private m: number
  private reg: Uint8Array
  constructor(private b = 14) { this.m = 1 << b; this.reg = new Uint8Array(this.m) }   // m = 2^b registros

  add(x: string): void {
    const hash = h32(x, 42)
    const idx = hash >>> (32 - this.b)             // los b bits altos eligen el bucket
    const rest = (hash << this.b) >>> 0            // el resto: contamos ceros a la izquierda
    const rho = clz32(rest) + 1                    // posición del primer 1 (patrón "raro" = muchos ceros)
    if (rho > this.reg[idx]) this.reg[idx] = rho   // el bucket guarda el MÁXIMO visto
  }
  count(): number {
    const alpha = 0.7213 / (1 + 1.079 / this.m)
    let sum = 0
    for (const r of this.reg) sum += 2 ** (-r)
    let e = (alpha * this.m * this.m) / sum         // estimador HLL (media armónica de los buckets)
    if (e <= 2.5 * this.m) {                         // corrección de rango bajo (linear counting)
      let vacios = 0
      for (const r of this.reg) if (r === 0) vacios++
      if (vacios > 0) e = this.m * Math.log(this.m / vacios)
    }
    return Math.round(e)
  }
  get bytes(): number { return this.m }             // 1 byte por registro (un HLL real empaqueta a 6 bits)
}

for (const real of [1_000, 50_000, 1_000_000]) {
  const hll = new HyperLogLog(14)                    // m = 16384 registros = 16 KB
  for (let i = 0; i < real; i++) hll.add(`user${i}`)
  const est = hll.count()
  const err = ((Math.abs(est - real) / real) * 100).toFixed(2)
  console.log(`real ${real} -> est ${est} (error ${err}%, memoria ${hll.bytes} bytes)`)
}
// real 1000      -> est 1007    (error 0.70%, memoria 16384 bytes)
// real 50000     -> est 50690   (error 1.38%, memoria 16384 bytes)
// real 1000000   -> est 996590  (error 0.34%, memoria 16384 bytes)

// los DUPLICADOS no cambian el conteo:
const h = new HyperLogLog(14)
for (let i = 0; i < 50_000; i++) h.add(`u${i}`)
const antes = h.count()
for (let i = 0; i < 50_000; i++) h.add(`u${i}`)   // re-agrego los MISMOS
console.log('dup-test: igual antes y después?', antes === h.count())   // true
```

**Qué observás.** El error ronda el **1%** (0.34%–1.38%) y, lo central: la **memoria es constante en 16 KB** para 1000, 50 000 **y un millón** de únicos. Re-agregar los mismos elementos **no cambia** el conteo (el mismo hash cae en el mismo bucket con el mismo máximo). Un `Set` exacto para un millón de ids serían **decenas de MB**; el HLL, 16 KB siempre.

**Qué demuestra.** El módulo 6: HLL cuenta cardinalidad en memoria fija porque guarda un estadístico (máximo de ceros a la izquierda) por bucket, no los elementos. Los duplicados son inofensivos (cuenta únicos) y es **mergeable** (unir dos HLL por el máximo por bucket = cardinalidad de la unión, extensión 3.3) → ideal para conteos distribuidos. (Un HLL real empaqueta cada registro en 6 bits → ~12 KB; acá usamos 1 byte por simplicidad.)

**Extensiones**
- 3.1 🧠 ¿Por qué el error baja al subir `b` (más buckets `m`)? ¿Cuál es la fórmula aproximada del error?
- 3.2 🧠 ¿Por qué importa que el hash mezcle bien los bits (el finalizador murmur)? ¿Qué pasaría con un hash sesgado?
- 3.3 ✍️ Implementá `merge(otro)` (máximo por registro) y mostrá que unir el HLL de "shard A" (users 0–30k) con el de "shard B" (users 20k–60k) estima ~60k (la unión, sin doble-contar el solapamiento).

---

## Build 4 — Leaderboard

**El problema.** Top-K y "¿en qué puesto estoy?" a escala. Vamos a ver que el rank por conteo lineal (lo que hace un `COUNT(*)` en SQL) es `O(n)`, y cómo una estructura ordenada lo baja a `O(log n)`.

```ts
// leaderboard.ts
class LeaderboardNaive {
  private s = new Map<string, number>()
  set(p: string, score: number): void { this.s.set(p, score) }
  topK(k: number): [string, number][] {
    return [...this.s.entries()].sort((a, b) => b[1] - a[1]).slice(0, k)
  }
  rank(p: string): number {                       // ❌ O(n): cuenta cuántos te superan (el COUNT(*) del SQL)
    const sc = this.s.get(p)
    if (sc === undefined) return -1
    let higher = 0
    for (const v of this.s.values()) if (v > sc) higher++
    return higher + 1
  }
}

const lb = new LeaderboardNaive()
;([['ana', 1500], ['beto', 2300], ['caro', 900], ['dario', 2300], ['eva', 3100]] as [string, number][])
  .forEach(([p, s]) => lb.set(p, s))
console.log('top 3:', lb.topK(3))   // [['eva',3100],['beto',2300],['dario',2300]]
console.log('rank eva:', lb.rank('eva'), 'ana:', lb.rank('ana'), 'caro:', lb.rank('caro'))   // 1, 4, 5
```

**Qué observás.** `top 3` da `eva(3100), beto(2300), dario(2300)`; `rank eva=1, ana=4, caro=5`. Funciona, pero `rank` recorre **todos** los jugadores en cada llamada (`O(n)`) — es exactamente el `SELECT COUNT(*) WHERE score > miScore` del módulo 8, que a escala (millones de jugadores, miles de consultas de rank/s) satura.

**El fix: mantener los scores ordenados → rank por búsqueda binaria `O(log n)`:**

```ts
// leaderboard-fast.ts — rank vía binary search sobre scores ordenados
class LeaderboardFast {
  private byPlayer = new Map<string, number>()
  private sorted: number[] = []        // scores ordenados ascendente

  private upperBound(x: number): number {   // primer índice con score > x
    let lo = 0, hi = this.sorted.length
    while (lo < hi) { const m = (lo + hi) >> 1; if (this.sorted[m] <= x) lo = m + 1; else hi = m }
    return lo
  }
  set(p: string, score: number): void {
    if (this.byPlayer.has(p)) {                              // si ya existía, saco su score viejo
      const old = this.byPlayer.get(p)!
      this.sorted.splice(this.sorted.indexOf(old), 1)
    }
    this.byPlayer.set(p, score)
    this.sorted.splice(this.upperBound(score), 0, score)      // inserto ordenado
  }
  rank(p: string): number {
    const sc = this.byPlayer.get(p)
    if (sc === undefined) return -1
    return this.sorted.length - this.upperBound(sc) + 1       // cuántos > sc (binary search), +1
  }
}

const lf = new LeaderboardFast()
;([['ana', 1500], ['beto', 2300], ['caro', 900], ['dario', 2300], ['eva', 3100]] as [string, number][])
  .forEach(([p, s]) => lf.set(p, s))
console.log('fast rank eva:', lf.rank('eva'), 'ana:', lf.rank('ana'), 'caro:', lf.rank('caro'))   // 1, 4, 5
```

**Qué observás.** Mismos ranks (`1, 4, 5`), pero ahora `rank` es una **búsqueda binaria** `O(log n)` sobre el array ordenado, no un conteo lineal. (El `splice` del insert sigue siendo `O(n)` con un array — por eso Redis usa una **skip list** en su sorted set: `O(log n)` para **insertar y** rankear.)

**Qué demuestra.** El módulo 8: el rank por conteo es el anti-patrón de SQL; una estructura **ordenada** lo vuelve logarítmico. Un sorted set de Redis (skip list) da las cuatro operaciones —add, top-K, rank, rango— en `O(log n)`, por eso es **la** herramienta para leaderboards en vivo.

**Extensiones**
- 4.1 🧠 ¿Por qué el `rank` naive es el mismo problema que el `COUNT(*) WHERE score > x` de SQL?
- 4.2 🧠 El `LeaderboardFast` mejora el rank pero su `set` sigue siendo `O(n)` por el `splice`. ¿Qué estructura da `O(log n)` para **ambos** y por qué?
- 4.3 ✍️ Agregá `topK(k)` a `LeaderboardFast` devolviendo los k de mayor score (pista: el final del array ordenado, mapeando de score a jugador).

---

## Build 5 — Geohash

**El problema.** Indexar puntos para "cerca de mí" convirtiendo (lat, long) en un string donde **puntos cercanos comparten prefijo**. Vamos a construirlo y ver el edge case del borde.

```ts
// geohash.ts
const BASE32 = '0123456789bcdefghjkmnpqrstuvwxyz'

function geohash(lat: number, lon: number, prec = 7): string {
  let latR: [number, number] = [-90, 90]
  let lonR: [number, number] = [-180, 180]
  let s = '', bit = 0, ch = 0, even = true
  while (s.length < prec) {
    if (even) {                                  // bit de longitud
      const mid = (lonR[0] + lonR[1]) / 2
      if (lon >= mid) { ch = (ch << 1) | 1; lonR[0] = mid } else { ch = ch << 1; lonR[1] = mid }
    } else {                                     // bit de latitud (interleaving lon/lat)
      const mid = (latR[0] + latR[1]) / 2
      if (lat >= mid) { ch = (ch << 1) | 1; latR[0] = mid } else { ch = ch << 1; latR[1] = mid }
    }
    even = !even
    if (++bit === 5) { s += BASE32[ch]; bit = 0; ch = 0 }   // cada 5 bits -> un char base32
  }
  return s
}

const timesSquare = geohash(40.7580, -73.9855)   // 'dr5ru7v'
const dosCuadras  = geohash(40.7560, -73.9840)   // 'dr5ru7q'  -> comparte 'dr5ru7' con Times Square
const tokio       = geohash(35.6762, 139.6503)   // 'xn76cyd'  -> no comparte nada

console.log('Times Square:', timesSquare)
console.log('2 cuadras:   ', dosCuadras, '| prefijo común:', comun(timesSquare, dosCuadras))   // dr5ru7
console.log('Tokio:       ', tokio, '| prefijo común:', comun(timesSquare, tokio))             // (vacío)

function comun(a: string, b: string): string {
  let i = 0; while (i < a.length && a[i] === b[i]) i++
  return a.slice(0, i)
}
```

**Qué observás.** Times Square (`dr5ru7v`) y un punto a dos cuadras (`dr5ru7q`) comparten el prefijo **`dr5ru7`** (6 de 7 chars) → "cerca" = "prefijo común largo". Tokio (`xn76cyd`) no comparte **nada** → lejos. Buscar "cerca de Times Square" = buscar geohashes que empiecen con `dr5ru7` (un `LIKE 'dr5ru7%'` que un B-tree resuelve volando).

**El edge case del borde:**

```ts
// dos puntos a ~17 m, pero a distinto lado de un borde de celda
const bordeA = geohash(40.7580, -73.9820)   // 'dr5ru7z'
const bordeB = geohash(40.7580, -73.9818)   // 'dr5rueb'  -> ¡difiere en el 6º char!
console.log('borde A:', bordeA, 'borde B:', bordeB, '| común:', comun(bordeA, bordeB))   // dr5ru  (solo 5)
```

**Qué observás.** Dos puntos a **~17 metros** (`-73.9820` vs `-73.9818`) caen en celdas **distintas**: `dr5ru7z` y `dr5rueb`, que solo comparten `dr5ru` (5 chars), no `dr5ru7`. Si buscaras solo por el prefijo de 6 de Times Square (`dr5ru7%`), **te perderías** el punto en `dr5rue` aunque esté pegadísimo → por eso una búsqueda seria consulta la celda **y sus 8 vecinas**.

**Qué demuestra.** Los módulos 9-10: el geohash convierte proximidad 2D en un **prefijo 1D** indexable por B-tree. Y el edge case del borde es por qué nunca consultás solo tu celda: dos puntos cercanos pueden caer en celdas adyacentes con prefijos que se separan antes de lo esperado → consultás la celda propia + sus 8 vecinas (9 en total).

**Extensiones**
- 5.1 🧠 ¿Por qué el interleaving de bits lon/lat es lo que hace que el prefijo capture proximidad en **ambas** dimensiones?
- 5.2 ✍️ Implementá `vecinas(gh)` que devuelva las 8 celdas adyacentes (pista: decodificá a lat/lon, sumá/restá el tamaño de celda en cada dirección, re-codificá) y úsalas para una búsqueda de proximidad correcta.
- 5.3 🧠 ¿Cuándo preferirías un quadtree (módulo 11) sobre geohash para este mismo problema?

---

## Soluciones de las extensiones

**1.1** Usa `.every` porque un elemento se considera "presente" solo si **todos** sus `k` bits están en 1 (al agregarlo se prendieron todos). Con `.some` (basta un bit en 1), casi cualquier consulta daría `true` (un solo bit compartido alcanzaría) → la tasa de falsos positivos sería altísima e inútil. La conjunción de los `k` bits es lo que hace al filtro discriminante.

**1.2** Con 1000 elementos en `m=1000` bits, casi todos los bits quedan en 1 (el fill ratio se acerca a 1) → casi cualquier consulta encuentra sus 5 bits prendidos → la tasa de falsos positivos sube a decenas de % (el filtro "dice que sí a todo", deja de filtrar). Es el Bloom saturado del módulo 4: hay que dimensionar `m` y `k` para el `n` esperado.

**1.3** Porque generar `k` hashes como `h_i(x) = a + i·b` (con `a`, `b` dos hashes base independientes) produce, en la práctica, una familia de hashes con propiedades de dispersión equivalentes a `k` hashes independientes para un Bloom filter (resultado de Kirsch-Mitzenmacher). Ahorra calcular `k` funciones distintas: con dos hashes derivás los `k` que necesités.

**2.1** `/home` (real 800) sale exacto porque su contador es **tan grande** que el ruido que colisiona en sus celdas (unas pocas unidades) es despreciable frente a 800, y además tomamos el **mínimo** de 4 filas (la menos contaminada). `/about` (real 10) sobreestima porque el ruido que colisiona (unas pocas unidades) es **comparable** a su valor real → 10 + 3 = 13. Cuanto más chico el valor real, más relativo pesa el error de colisión.

**2.2** Con `w=8` hay muchas más colisiones (solo 8 columnas para 202 claves) → `/about` sobreestima bastante más (se le suma más ruido). Con `w=512`, casi no hay colisiones → `/about` se acerca a 10 (y `/home` sigue en 800). Muestra que **más columnas = menos colisiones = más preciso** (a costa de más memoria), el dial del Count-Min.

**2.3** Un Count-Min solo **estima la frecuencia de una key que le preguntás**; no sabe **cuáles** son las top-K (no itera elementos). Lo combinás con un **min-heap de tamaño K**: por cada elemento del stream, actualizás el Count-Min y, con su frecuencia estimada, lo mantenés o no en el heap de los K más frecuentes. El sketch da la frecuencia, el heap mantiene el ranking.

**3.1** El error baja porque con más buckets (`m`) cada uno ve menos elementos y el promedio (media armónica) de sus estimaciones tiene **menos varianza**. La fórmula aproximada es `error ≈ 1.04/√m`: con `m=16384` (b=14), `1.04/128 ≈ 0.8%`. Duplicar `b` cuadruplica `m` y halviea el error, al costo de más memoria.

**3.2** Porque el HLL asume que los bits del hash son **uniformes e independientes** (cada bit 0/1 con prob 1/2): de ahí sale que "ver muchos ceros a la izquierda es raro y proporcional a la cantidad de únicos". Con un hash **sesgado** (como FNV-1a crudo, cuyos bits altos/bajos no son uniformes), la distribución de ceros a la izquierda se desvía → el estimador da error mucho mayor (lo vimos: ~11% sin finalizador vs ~1% con él). El finalizador murmur "avalancha" los bits para acercarlos a uniformes.

**3.3** `merge(otro)` recorre los registros y se queda con `Math.max(this.reg[i], otro.reg[i])` por posición. Como cada registro guarda el máximo de ceros vistos para su bucket, el máximo entre dos HLL refleja "el patrón más raro visto por **cualquiera** de los dos" = la unión de los elementos. Unir A (0–30k) y B (20k–60k) estima ~60k (no 90k): los users 20k–30k que están en **ambos** caen en los mismos buckets con el mismo máximo → no se doble-cuentan. Esa es la magia para conteos distribuidos.

**4.1** Porque ambos **cuentan cuántos elementos superan tu score** recorriéndolos: el naive itera `this.s.values()` (`O(n)`), el SQL hace `COUNT(*) WHERE score > miScore` (escanea/cuenta una porción grande). En los dos casos, cada consulta de rank cuesta proporcional a la cantidad de jugadores → con millones y muchas consultas/s, satura.

**4.2** Una **skip list** (lo que usa Redis en su sorted set), o un **árbol balanceado con order-statistics** (cada nodo guarda el tamaño de su subárbol). Dan `O(log n)` para **insertar** (encontrás la posición y enlazás/rebalanceás en log n) **y** para **rank** (sumás tamaños de subárboles a la izquierda en log n). El array ordenado da log n para rank pero `O(n)` para insertar (el `splice` corre todos los elementos posteriores).

**4.3** `topK(k)` toma el **final** del array `sorted` (los scores más altos): `this.sorted.slice(-k).reverse()`, y para cada score buscás el/los jugador(es) con ese score. (Con scores repetidos —como beto y dario en 2300— necesitás un mapa inverso `score → jugadores`; un sorted set real desempata por el miembro lexicográficamente.)

**5.1** Porque al **alternar** un bit de longitud y uno de latitud, el prefijo va refinando **ambas** dimensiones de forma pareja: los primeros bits acotan una región grande en lon **y** lat, los siguientes la achican en lon **y** lat, etc. Así, compartir un prefijo largo significa coincidir en muchas subdivisiones de **las dos** coordenadas → estar en la misma región 2D. Si no interleaveáramos (todos los bits de lon primero), el prefijo capturaría proximidad en una dimensión y no en la otra.

**5.2** `vecinas(gh)`: decodificás `gh` a su celda (lat/lon del centro y el tamaño de celda según la precisión), y generás las 8 combinaciones de (lat ± alto, lon ± ancho, y los 4 diagonales), re-codificando cada una a geohash. La búsqueda de proximidad correcta = unir los puntos de `gh` **y** sus 8 vecinas, y después refinar por distancia real (Haversine). Así no perdés puntos al otro lado de un borde.

**5.3** Preferirías un **quadtree** cuando la **densidad de puntos es muy variable** (módulo 11): el geohash usa celdas de tamaño fijo, así que en el centro de una ciudad una celda puede tener millones de puntos (hot cell) mientras en el campo está casi vacía. El quadtree subdivide **adaptativamente** (celdas chicas donde hay densidad, grandes donde no), evitando la hot cell y el desperdicio. El costo: es un árbol que mantenés/rebalanceás aparte, más difícil de shardear que un prefijo de geohash.
