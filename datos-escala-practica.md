# Datos a escala: laboratorio práctico

**Construí, rompé y arreglá: hot shard, Snowflake ID, migración expand-contract, mini LSM-tree y shuffle sharding · Node + Go · 2026**

> Cómo usar esta guía: complemento de mano del módulo de teoría [Datos a escala y multi-tenancy](datos-escala.md). Ahí entendiste los conceptos; acá **construís cinco cosas**, cada una con el método: **(1)** el problema, **(2)** código que **falla a propósito**, **(3)** qué observás, **(4)** el fix, **(5)** qué demuestra.

**Lo que asumimos.** Haber leído el [módulo de teoría](datos-escala.md). Node 20+ y TypeScript. El Go se explica línea por línea.

**Los cinco builds**
1. **Hot shard** (sharding range vs hash con inserción secuencial)
2. **Snowflake ID** (64 bits ordenados por tiempo, sin coordinación)
3. **Migración expand-contract** (el rename que rompe, y cómo se hace sin downtime)
4. **Mini LSM-tree** (memtable → SSTables → compaction, y la read amplification)
5. **Shuffle sharding** (acotar el blast radius de un tenant tóxico)

> Convención: el código compila en `--strict` y las salidas de los comentarios están **verificadas ejecutando**. Las **extensiones** están al final de cada build; sus soluciones, en la última sección.

> Usamos un solo hash en varios builds (FNV-1a de 32 bits), que mezcla bien:
> ```ts
> // hash.ts — FNV-1a 32 bits (reusado en builds 1 y 5)
> export function hash(s: string): number {
>   let h = 2166136261 >>> 0
>   for (let i = 0; i < s.length; i++) {
>     h ^= s.charCodeAt(i)
>     h = Math.imul(h, 16777619) >>> 0   // multiplicación 32-bit + mezcla
>   }
>   return h >>> 0
> }
> ```

---

## Build 1 — Hot shard (range vs hash)

**El problema.** Sharding por una key **secuencial** (un id/timestamp creciente). Vamos a ver que el sharding **range-based** manda todas las inserciones nuevas al mismo shard (hot shard) y que el **hash-based** las reparte parejo.

```ts
// sharding.ts
import { hash } from './hash.js'
const N = 4   // 4 shards

// las "últimas 1000 inserciones": keys secuenciales crecientes (u09000..u09999)
const recientes = Array.from({ length: 1000 }, (_, i) => `u${String(9000 + i).padStart(5, '0')}`)

// RANGE: 4 rangos ordenados por el número de la key
function rangeShard(k: string): number {
  const n = parseInt(k.slice(1), 10)
  return n < 2500 ? 0 : n < 5000 ? 1 : n < 7500 ? 2 : 3
}
// HASH: hash(key) % N
function hashShard(k: string): number {
  return hash(k) % N
}

const distRange = [0, 0, 0, 0]
const distHash = [0, 0, 0, 0]
for (const k of recientes) { distRange[rangeShard(k)]++; distHash[hashShard(k)]++ }

console.log('range:', distRange)   // [0, 0, 0, 1000]        <- TODO al último shard: HOT SHARD
console.log('hash: ', distHash)    // [250, 250, 250, 250]   <- repartido parejo
```

**Qué observás.** Con sharding **range**, las 1000 inserciones recientes (keys `u09000`–`u09999`, todas en el rango `≥7500`) caen **todas en el shard 3**: `[0, 0, 0, 1000]`. Los shards 0-2 están ociosos mientras el 3 se satura → **hot shard de escritura**. Con **hash**, las mismas keys se reparten `[250, 250, 250, 250]` — perfectamente parejo.

**Qué demuestra.** El módulo 2 y 4: sharding range por una key secuencial (timestamp, auto-increment) concentra todas las escrituras nuevas en el último shard. El hash lo evita repartiendo, al precio de perder los range scans. Si necesitás scans por tiempo **y** reparto, una key compuesta (`hash(userId) + timestamp`) — extensión 1.3.

**Extensiones**
- 1.1 🧠 ¿Por qué el hot shard del sharding range "no escaló nada" aunque tengas 4 shards?
- 1.2 ✍️ Mostrá que el hash **sí** preserva el routing: la misma key siempre cae en el mismo shard (consultar `hashShard('u09500')` dos veces da lo mismo).
- 1.3 🧠 Querés range scans por tiempo Y reparto parejo de escritura. ¿Qué key compuesta usarías y qué ganás/perdés?

---

## Build 2 — Snowflake ID

**El problema.** Generar IDs únicos en varios nodos **sin coordinación** y que tengan **buena localidad de índice** (ordenados por tiempo), a diferencia de un UUID v4 random. Vamos a construir un Snowflake de 64 bits.

```ts
// snowflake.ts — 64 bits: 1 signo (en 0) | timestamp(41) | worker(10) | secuencia(12)
const EPOCH = 1_700_000_000_000n   // época propia (ms); recorta el timestamp

class Snowflake {
  private worker: bigint
  private seq = 0n
  private lastMs = 0n
  constructor(workerId: number) { this.worker = BigInt(workerId) }

  gen(nowMs: number): bigint {
    const ms = BigInt(nowMs)
    if (ms === this.lastMs) {
      this.seq = (this.seq + 1n) & 0xFFFn         // mismo ms: incrementá la secuencia (12 bits)
    } else {
      this.seq = 0n; this.lastMs = ms             // ms nuevo: reiniciá la secuencia
    }
    // timestamp corrido 22 bits (10 worker + 12 seq), luego worker corrido 12, luego seq
    return ((ms - EPOCH) << 22n) | (this.worker << 12n) | this.seq
  }
}

const sf = new Snowflake(1)
const id1 = sf.gen(1_700_000_005_000)   // ms = epoch + 5000
const id2 = sf.gen(1_700_000_005_000)   // MISMO ms -> secuencia 1
const id3 = sf.gen(1_700_000_006_000)   // ms +1000 -> secuencia reinicia a 0
console.log([id1, id2, id3].map(x => x.toString()))   // ['20971524096', '20971524097', '25165828096']
console.log('monótonos crecientes?', id1 < id2 && id2 < id3)   // true
```

**Qué observás.** Los IDs salen `20971524096 < 20971524097 < 25165828096`: **monótonos crecientes** porque el **timestamp va en los bits altos**. Dos IDs del mismo milisegundo se distinguen por la **secuencia** (`...096`, `...097`); un ms nuevo reinicia la secuencia y salta el bloque alto. Cada nodo (worker distinto) genera **localmente** sin hablar con nadie.

**Qué demuestra.** El módulo 13: un Snowflake es único **sin coordinación** (worker id distinto por nodo) y **ordenado por tiempo** → buena **localidad de índice** (los inserts nuevos van contiguos al final de un B-tree, módulo 10), a diferencia de un UUID v4 random que cae disperso. El precio: depende de relojes razonablemente sincronizados y de worker ids únicos.

**Extensiones**
- 2.1 🧠 ¿Por qué poner el timestamp en los **bits altos** (y no el worker o la secuencia) es lo que da la localidad de índice?
- 2.2 ✍️ Generá 5000 IDs en el mismo ms con un solo worker y mostrá qué pasa al desbordar la secuencia de 12 bits (4096). ¿Qué debería hacer el generador?
- 2.3 🧠 ¿Qué dos cosas tienen que cumplirse en el entorno para que Snowflake no genere IDs duplicados o que retrocedan?

---

## Build 3 — Migración expand-contract

**El problema.** Tenemos filas con `nombreCompleto` y queremos partirlo en `nombre` + `apellido` **sin downtime**, con código viejo y nuevo conviviendo. Vamos a ver que un "rename" de un paso rompe el lector viejo, y después hacerlo por etapas.

### 3a) El rename de un paso → el código viejo se rompe

```ts
// migracion.ts
type Fila = { id: number; nombreCompleto?: string; nombre?: string; apellido?: string }
const tabla: Fila[] = [{ id: 1, nombreCompleto: 'Ana Gómez' }]

// el código VIEJO (instancias que aún corren durante el rolling deploy) lee así:
const lectorViejo = (f: Fila): string => f.nombreCompleto ?? '(roto)'

// migración NAÏVE: "renombrar" = borrar el viejo y poner los nuevos de golpe
function renameDeUnPaso(f: Fila): void {
  const [nombre, apellido] = (f.nombreCompleto ?? '').split(' ')
  f.nombre = nombre; f.apellido = apellido
  delete f.nombreCompleto              // ❌ borro el viejo YA
}
renameDeUnPaso(tabla[0])
console.log('lector viejo tras rename:', lectorViejo(tabla[0]))   // (roto)  <- el código viejo dejó de funcionar
```

**Qué observás.** Apenas "renombramos" (borramos `nombreCompleto`), el `lectorViejo` —que sigue corriendo en las instancias no actualizadas— devuelve `(roto)`: la columna que lee ya no existe. En producción, durante el rolling deploy, eso son **errores reales** hasta que todas las instancias se actualicen.

### 3b) El fix: expand → migrate → contract

```ts
// (mismo tipo Fila)
const tabla2: Fila[] = [{ id: 1, nombreCompleto: 'Ana Gómez' }, { id: 2, nombreCompleto: 'Beto Ruiz' }]

// FASE 1 — EXPAND: agrego las columnas nuevas SIN tocar la vieja (compatible hacia atrás)
//   (en una DB real: ALTER TABLE ADD COLUMN nombre, apellido)

// FASE 2 — MIGRATE:
//   (a) doble escritura: el código nuevo escribe nombreCompleto Y nombre/apellido
function escribirDoble(f: Fila, nombre: string, apellido: string): void {
  f.nombre = nombre; f.apellido = apellido
  f.nombreCompleto = `${nombre} ${apellido}`   // sigo manteniendo el viejo: lector viejo no se rompe
}
//   (b) backfill: relleno las filas viejas (idempotente, por lotes en la vida real)
function backfill(filas: Fila[]): void {
  for (const f of filas) {
    if (f.nombre === undefined && f.nombreCompleto !== undefined) {   // idempotente: solo las que faltan
      const [n, a] = f.nombreCompleto.split(' '); f.nombre = n; f.apellido = a
    }
  }
}
backfill(tabla2)
//   (c) cambio las LECTURAS a lo nuevo (tras verificar con shadow reads que coinciden)
const lectorNuevo = (f: Fila): string => `${f.nombre} ${f.apellido}`

// FASE 3 — CONTRACT: cuando NADIE lee lo viejo, lo borro
function contract(filas: Fila[]): void { for (const f of filas) delete f.nombreCompleto }

console.log('lector viejo durante migrate:', tabla2.map(f => f.nombreCompleto))  // ['Ana Gómez','Beto Ruiz'] -> sigue OK
console.log('lector nuevo tras backfill:', tabla2.map(lectorNuevo))              // ['Ana Gómez','Beto Ruiz'] -> OK
contract(tabla2)
console.log('tras contract:', tabla2.map(lectorNuevo))                           // ['Ana Gómez','Beto Ruiz'] -> sigue OK
```

**Qué observás.** En la fase MIGRATE, **tanto** el lector viejo (`nombreCompleto`) **como** el nuevo (`nombre`/`apellido`) funcionan a la vez → el rolling deploy no rompe nada. El backfill es idempotente (solo toca las filas que faltan). Recién en CONTRACT, cuando ya nadie lee `nombreCompleto`, lo borramos — y el lector nuevo sigue OK.

**Qué demuestra.** Los módulos 6-8: cada paso es **compatible hacia atrás**, así que viejo y nuevo conviven. El cambio riesgoso de un paso se vuelve una secuencia de pasos chicos y reversibles, cero downtime. El orden (expand → doble escritura → backfill → switch de lecturas → contract) **no** se puede alterar.

**Extensiones**
- 3.1 🧠 ¿Por qué el backfill chequea `if (f.nombre === undefined)` y qué propiedad le da eso (pista: ¿qué pasa si lo corrés dos veces)?
- 3.2 🧠 ¿Qué pasaría si cambiaras las lecturas a `nombre`/`apellido` **antes** del backfill?
- 3.3 ✍️ Agregá una verificación de *shadow reads*: comparar `lectorViejo(f)` vs `lectorNuevo(f)` para todas las filas y abortar el switch si alguna no coincide.

---

## Build 4 — Mini LSM-tree

**El problema.** Construir un mini LSM: las escrituras van a una memtable en memoria, se vuelcan a SSTables inmutables, y una compaction las mergea. Vamos a ver la **read amplification** (una lectura toca varias SSTables) y cómo la compaction la reduce.

```ts
// lsm.ts
type Entrada = [string, string]
class MiniLSM {
  private memtable = new Map<string, string>()
  sstables: Entrada[][] = []             // SSTables inmutables, la MÁS NUEVA primero (pública para inspección)
  scans = 0                              // cuántas SSTables tocó el último get (read amplification)
  constructor(private umbral = 3) {}

  put(k: string, v: string): void {
    this.memtable.set(k, v)              // escritura: siempre a la memtable (rápido, en memoria)
    if (this.memtable.size >= this.umbral) this.flush()
  }
  private flush(): void {                // memtable llena -> SSTable inmutable y ordenada
    const tabla = [...this.memtable.entries()].sort((a, b) => (a[0] < b[0] ? -1 : 1))
    this.sstables.unshift(tabla)         // al frente: es la más nueva
    this.memtable.clear()
  }
  get(k: string): string | undefined {
    this.scans = 0
    if (this.memtable.has(k)) return this.memtable.get(k)    // 1º la memtable (lo más nuevo)
    for (const t of this.sstables) {                          // luego SSTables, NUEVA -> vieja
      this.scans++
      const e = t.find(([kk]) => kk === k)
      if (e) return e[1]                                      // la primera (más nueva) gana
    }
    return undefined
  }
  compact(): void {                       // mergea todas las SSTables en una (la más nueva gana)
    const merged = new Map<string, string>()
    for (let i = this.sstables.length - 1; i >= 0; i--)       // de vieja a nueva: la nueva pisa
      for (const [k, v] of this.sstables[i]) merged.set(k, v)
    this.sstables = [[...merged.entries()].sort((a, b) => (a[0] < b[0] ? -1 : 1))]
  }
}

const db = new MiniLSM(3)
// 7 escrituras; con umbral 3 se vuelcan 2 SSTables y queda 1 clave en memtable
for (const [k, v] of [['a','1'],['b','1'],['c','1'],['a','2'],['d','1'],['e','1'],['b','2']] as Entrada[]) db.put(k, v)

console.log('SSTables:', db.sstables.length)            // 2
console.log("get('a'):", db.get('a'), 'scans:', db.scans)   // 2 scans: 1  -> 'a' se actualizó a 2 en la SSTable nueva
console.log("get('b'):", db.get('b'), 'scans:', db.scans)   // 2 scans: 0  -> 'b'=2 está en la memtable
console.log("get('z'):", db.get('z'), 'scans:', db.scans)   // undefined scans: 2  -> PEOR CASO: tocó las 2 SSTables
db.compact()
console.log('SSTables tras compact:', db.sstables.length)   // 1
console.log("get('a') tras compact:", db.get('a'), 'scans:', db.scans)   // 2 scans: 1
```

**Qué observás.** Tras 7 escrituras hay **2 SSTables**. `get('a')` devuelve `2` (el valor más nuevo, en la SSTable más reciente) tocando 1 SSTable; `get('b')` devuelve `2` sin tocar ninguna (está en la memtable); `get('z')` (que no existe) es el **peor caso**: toca **las 2** SSTables antes de rendirse → **read amplification**. Tras `compact()`, queda **1 sola** SSTable y la búsqueda toca menos.

**Qué demuestra.** Los módulos 11-12: las escrituras son siempre append (memtable → SSTable inmutable), rapidísimas; las lecturas pueden tocar varias SSTables (read amplification), peor para keys que no existen. La compaction mergea y reduce la cantidad de SSTables. En un LSM real, un **Bloom filter** por SSTable evitaría el peor caso de `get('z')` (diría "no está acá" sin escanear) — extensión 4.2.

> Nota: `get('b')` devuelve `2` desde la **memtable**, aunque la SSTable compactada muestre `b='1'` (la `b='2'` todavía no se había volcado cuando compactamos). La compaction solo mergea **SSTables**; la memtable es aparte y siempre se lee primero.

**Extensiones**
- 4.1 🧠 ¿Por qué `get` recorre las SSTables de la más **nueva** a la más **vieja**, y por qué la primera coincidencia gana?
- 4.2 ✍️ Agregá un Bloom filter simulado por SSTable (un `Set` de las keys que contiene) y mostrá que `get('z')` ahora **saltea** las SSTables → 0 escaneos reales.
- 4.3 🧠 ¿Qué es la write amplification acá y en qué momento del código ocurre (pista: la compaction reescribe entradas)?

---

## Build 5 — Shuffle sharding

**El problema.** N workers, muchos tenants. Si todos los tenants comparten todos los workers, un tenant tóxico cae a todos (blast radius 100%). Con shuffle sharding, cada tenant usa un subconjunto aleatorio de `k` workers → vamos a medir cuánto se achica el blast radius.

```ts
// shuffle.ts
import { hash } from './hash.js'
const N = 12   // workers
const K = 3    // workers por tenant

// asignación DETERMINÍSTICA por tenant: ordena los workers por hash(tenant#worker) y toma K
function workersDe(tenant: string): number[] {
  return [...Array(N).keys()]
    .sort((a, b) => hash(`${tenant}#${a}`) - hash(`${tenant}#${b}`))
    .slice(0, K)
    .sort((a, b) => a - b)
}

const setA = workersDe('tenantA')
const setB = workersDe('tenantB')
console.log('tenantA:', setA, 'tenantB:', setB)        // tenantA: [5,8,9]  tenantB: [0,2,3]
console.log('overlap A∩B:', setA.filter(w => setB.includes(w)))   // []  -> NO comparten workers

// si tenantA se vuelve TÓXICO (envenena sus 3 workers), ¿a cuántos de 50 tenants alcanza?
let totalmenteAfectados = 0
const histoOverlap = [0, 0, 0, 0]    // cuántos tenants comparten 0,1,2,3 workers con A
for (let i = 0; i < 50; i++) {
  const t = workersDe(`t${i}`)
  const compartidos = t.filter(w => setA.includes(w)).length
  histoOverlap[compartidos]++
  if (t.every(w => setA.includes(w))) totalmenteAfectados++   // sus 3 workers ⊆ los tóxicos de A
}
console.log('100%-contenidos en el set tóxico de A (de 50):', totalmenteAfectados)   // 3
console.log('histo superposición [0,1,2,3]:', histoOverlap)   // [22, 19, 6, 3]
```

**Qué observás.** `tenantA` usa `[5,8,9]` y `tenantB` usa `[0,2,3]` — **no comparten ningún worker**. Si `tenantA` se vuelve tóxico y envenena sus 3 workers, de 50 tenants solo **3** quedan **totalmente** afectados (sus 3 workers ⊆ los de A); **22 no comparten ninguno** (siguen perfectos) y el resto comparte 1-2 (degradados, pero con workers sanos para seguir). Compará con "todos comparten todos los workers": el blast radius sería **50 de 50**.

**Qué demuestra.** El módulo 5: shuffle sharding baja el blast radius de un tenant tóxico de "todos" a una fracción chica, porque con `C(12,3)=220` combinaciones posibles (y `C(100,5)≈75M` en escala real) dos tenants rara vez comparten su set completo. Un tenant ruidoso solo puede tumbar a los pocos que cayeron en su mismo subconjunto.

**Extensiones**
- 5.1 🧠 ¿Por qué subir `N` (workers totales) o bajar `K` (workers por tenant) achica el blast radius? ¿Qué se pierde si `K` es muy chico?
- 5.2 ✍️ Calculá empíricamente el blast radius promedio: para cada uno de los 50 tenants tóxicos, contá cuántos otros quedan 100%-contenidos, y promediá.
- 5.3 🧠 ¿Qué agrega cell-based architecture sobre shuffle sharding, y por qué es más caro?

---

## Soluciones de las extensiones

**1.1** Porque "escalar" con sharding significa **repartir la carga** entre los shards. Si todas las escrituras nuevas caen en un solo shard (el hot shard), ese shard hace **todo** el trabajo de inserción y los otros 3 están ociosos → tu throughput de escritura es el de **un** shard, igual que sin shardear. Sumaste máquinas pero no repartiste el trabajo.

**1.2** `hashShard('u09500')` aplica `hash('u09500') % 4`, que es **determinístico**: la misma string siempre da el mismo número → la misma key siempre cae en el mismo shard. Eso es lo que permite el routing (una query por esa key va directo al shard correcto). Lo verificás llamándola dos veces y viendo el mismo resultado.

**1.3** Una key compuesta `hash(userId)` como prefijo de shard + `timestamp` como orden **dentro** del shard (p. ej. shard = `hash(userId) % N`, y dentro ordenás por tiempo). Ganás: reparto parejo por usuario (no hot shard) **y** range scans por tiempo **dentro** de un usuario/shard. Perdés: range scans **globales** por tiempo cruzando todos los usuarios (tendrías que consultar todos los shards). Es el compromiso típico (p. ej. Cassandra: partition key + clustering key).

**2.1** Porque al comparar dos IDs, los **bits más significativos** mandan. Con el timestamp arriba, IDs generados después tienen bits altos mayores → quedan **ordenados por tiempo**, y los inserts consecutivos van **contiguos** (al final del índice B-tree). Si el worker o la secuencia fueran los bits altos, IDs de distintos nodos/secuencias se intercalarían sin orden temporal → inserts dispersos, mala localidad (el problema del UUID random).

**2.2** Al pasar 4096 IDs en el mismo ms, `seq` desborda los 12 bits (`& 0xFFF` vuelve a 0) → empezás a **repetir** secuencias dentro del ms → **IDs duplicados**. El generador debe **esperar al siguiente milisegundo** (busy-wait hasta que `nowMs` avance) antes de seguir generando, garantizando que nunca repite seq dentro de un ms. (4096 IDs/ms/worker = ~4M IDs/s por worker, suele alcanzar; si no, agregás workers.)

**2.3** (1) **Relojes razonablemente sincronizados y que no retrocedan**: si el reloj de un nodo salta hacia atrás (NTP, M2/[consenso módulo 4](consenso.md)), podría generar IDs con timestamp menor → IDs que "retroceden" o chocan; los generadores reales detectan el retroceso y esperan/abortan. (2) **worker ids únicos**: dos nodos con el mismo worker id en el mismo ms generarían el mismo ID → la asignación de worker id tiene que ser única (coordinada al arranque, p. ej. vía ZooKeeper/etcd).

**3.1** Porque hace el backfill **idempotente**: solo rellena las filas que **todavía no** tienen `nombre` (las viejas), y saltea las que ya están migradas. Si lo corrés dos veces (o se corta y reanudás), no re-procesa ni corrompe lo ya hecho — exactamente lo que pide un backfill por lotes reanudable (módulo 7).

**3.2** Leerías datos **incompletos**: las filas **viejas** todavía no tienen `nombre`/`apellido` poblados (el backfill no corrió), así que `lectorNuevo` devolvería `undefined undefined` para ellas. Por eso el switch de lecturas va **después** del backfill, cuando lo nuevo está completo (histórico + escrituras recientes).

**3.3** Un `verificar(filas)` que recorre todas y compara `lectorViejo(f) === lectorNuevo(f)`; si alguna difiere, lanzás/abortás el switch (no cambiás las lecturas). Sirve para detectar bugs en la doble escritura o el backfill **antes** de depender de lo nuevo: corrés shadow reads un tiempo en producción y solo hacés el switch cuando el 100% coincide.

**4.1** Recorre de la más nueva a la más vieja porque una key puede tener **varias versiones** (se sobrescribió): la versión más reciente está en la SSTable más nueva (o en la memtable). La **primera coincidencia** (la más nueva) es la vigente, así que devolverla y cortar da el valor correcto sin seguir buscando versiones viejas. (`get('a')` devuelve 2, no 1, por esto.)

**4.2** Cada SSTable lleva un `Set<string>` con sus keys (su "Bloom filter" exacto). En `get`, antes de escanear una SSTable, chequeás `if (!bloom.has(k)) continue` → para `get('z')`, ninguna SSTable lo tiene → las salteás todas → `scans` real = 0. (Un Bloom filter real es probabilístico y compacto, con falsos positivos pero **cero** falsos negativos, así que nunca saltea una SSTable que sí tiene la key.)

**4.3** La write amplification ocurre en `compact()`: las entradas de las SSTables viejas se **reescriben** en la SSTable nueva mergeada (cada `merged.set` + el volcado final). Una entrada que sobrevive a varias compactions se escribe en disco **varias veces** a lo largo de su vida, aunque su `put` original haya sido un solo append. Es el costo de mantener pocas SSTables (módulo 11).

**5.1** Subir `N` o bajar `K` aumenta la cantidad de combinaciones posibles `C(N,K)` → es **menos probable** que dos tenants compartan su set completo → menor superposición → menor blast radius. Si `K` es muy chico (p. ej. 1), perdés **redundancia/disponibilidad** para el tenant: si su único worker se cae, ese tenant queda sin servicio. `K` balancea aislamiento (chico) vs tolerancia a fallos del propio tenant (grande, varios workers).

**5.2** Para cada tenant `tX` (tóxico hipotético), calculás su set y contás cuántos **otros** de los 50 tienen su set completamente contenido en el de `tX`; sumás y dividís por 50. Da un número bajo (cercano al ~3/50 que vimos para A), confirmando que el blast radius promedio de un tenant cualquiera es chico — no "todos".

**5.3** Cell-based aísla **todo el stack** (workers, base de datos, cache, red) en celdas independientes, mientras shuffle sharding aísla solo un pool de **workers** compartido. Cell-based contiene fallas que shuffle sharding no (un deploy malo, una corrupción de datos, una caída de la DB quedan dentro de la celda). Es más caro porque **duplicás infraestructura completa** por celda (cada celda necesita su stack entero) en vez de repartir un pool compartido.
