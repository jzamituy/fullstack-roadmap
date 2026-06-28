# Streams y analytics: laboratorio práctico

**Construí, rompé y arreglá: particiones y orden, offsets que pierden/duplican, ventanas por event-time, replay estilo Kappa y scan OLTP vs OLAP · Node + Go · 2026**

> Cómo usar esta guía: complemento de mano del módulo de teoría [Datos en movimiento: streams, batch y analytics](streams.md). Ahí entendiste los conceptos; acá **construís cinco cosas**, cada una con el método: **(1)** el problema, **(2)** código que **falla a propósito**, **(3)** qué observás, **(4)** el fix, **(5)** qué demuestra.

**Lo que asumimos.** Haber leído el [módulo de teoría](streams.md) y, de fondo, [Event-driven](event-driven.md) (idempotencia, exactly-once processing). Node 20+ y TypeScript. El Go se explica línea por línea.

**Los cinco builds**
1. **Particiones y orden** (el orden que solo vale por partición; key → partición)
2. **Offsets del consumidor** (at-most-once pierde, at-least-once duplica, idempotencia lo cierra)
3. **Ventana por event-time** (tumbling + watermark + un evento tardío)
4. **Replay estilo Kappa** (reprocesar por event-time da el mismo resultado; por processing-time, otro)
5. **Scan OLTP vs OLAP** (el agregado que lee 1 columna en vez de 40)

> Convención: el código compila en `--strict`. Las **extensiones** están al final de cada build; sus soluciones, en la última sección.

---

## Build 1 — Particiones y orden

**El problema.** Un topic con varias particiones. Vamos a ver que el orden **solo** está garantizado dentro de una partición, que la **key** decide la partición, y que mezclar particiones pierde el orden.

### 1a) Sin key → round-robin → el orden del agregado se pierde

```ts
// particiones.ts — hash propio, sin dependencias (así el build corre suelto)
function hash(s: string): number {
  let h = 0
  for (let i = 0; i < s.length; i++) h = (h * 31 + s.charCodeAt(i)) >>> 0   // djb2-ish, 32 bits
  return h
}

type Evento = { key: string | null; payload: string; seq: number }

class Topic {
  particiones: Evento[][]
  private rr = 0
  constructor(n: number) { this.particiones = Array.from({ length: n }, () => []) }

  producir(e: Evento): number {
    let p: number
    if (e.key === null) {
      p = this.rr++ % this.particiones.length          // sin key: round-robin
    } else {
      p = hash(e.key) % this.particiones.length         // con key: hash(key) % N
    }
    this.particiones[p].push(e)
    return p
  }
}

// el historial de la cuenta 'c1': crear -> depositar -> retirar (ORDEN IMPORTA)
const sinKey = new Topic(3)
const eventos: Evento[] = [
  { key: null, payload: 'c1:crear',     seq: 1 },
  { key: null, payload: 'c1:depositar', seq: 2 },
  { key: null, payload: 'c1:retirar',   seq: 3 },
]
for (const e of eventos) console.log(`${e.payload} -> partición ${sinKey.producir(e)}`)
// c1:crear -> partición 0 ; c1:depositar -> partición 1 ; c1:retirar -> partición 2
```

**Qué observás.** Los tres eventos de la **misma** cuenta cayeron en **particiones distintas** (0, 1, 2). Como cada partición la procesa un consumidor a su ritmo, nada garantiza que `crear` se procese antes que `retirar` → el consumidor podría intentar retirar de una cuenta que "todavía no existe". El orden del agregado se perdió.

### 1b) El fix: key = la entidad que necesita orden

```ts
// (mismo Topic)
const conKey = new Topic(3)
const eventos2: Evento[] = [
  { key: 'c1', payload: 'c1:crear',     seq: 1 },
  { key: 'c1', payload: 'c1:depositar', seq: 2 },
  { key: 'c1', payload: 'c1:retirar',   seq: 3 },
  { key: 'c2', payload: 'c2:crear',     seq: 1 },   // otra cuenta: puede ir a otra partición
]
for (const e of eventos2) console.log(`${e.payload} -> partición ${conKey.producir(e)}`)
// c1:crear -> partición 1 ; c1:depositar -> partición 1 ; c1:retirar -> partición 1 ; c2:crear -> partición 2

// dentro de una partición, el orden de escritura == orden de lectura:
console.log('partición de c1:', conKey.particiones[1].map(e => e.payload))
// ['c1:crear', 'c1:depositar', 'c1:retirar']  -> ORDEN PRESERVADO
```

**Qué observás.** Con `key = 'c1'`, los tres eventos de la cuenta van **todos a la partición 1**, en orden. La cuenta `c2` puede ir a otra partición (y procesarse en paralelo), lo cual está bien: cuentas distintas no necesitan orden entre sí. Conseguiste orden **donde importa** sin perder el paralelismo.

**Qué demuestra.** El módulo 4: Kafka ordena **por partición**, no global; la **partition key** es la herramienta de diseño para hacer caer juntas (y en orden) las cosas que lo necesitan. El precio (extensión 1.2): una key demasiado popular concentra carga en una partición (hot partition).

**Extensiones**
- 1.1 🧠 ¿Por qué procesar en paralelo cuentas distintas (en particiones distintas) NO rompe nada, pero paralelizar eventos de la **misma** cuenta sí?
- 1.2 ✍️ Producí 10 000 eventos con `key = userId` donde el 90% son del mismo usuario; contá eventos por partición y mostrá el hot partition.
- 1.3 🧠 Si necesitás más paralelismo que el que da el número de particiones, ¿qué opciones tenés (y qué cuesta cada una)?

---

## Build 2 — Offsets del consumidor (perder vs duplicar)

**El problema.** Un consumidor procesa mensajes y commitea su offset. Vamos a ver que commitear **antes** de procesar **pierde** mensajes ante un crash, que commitear **después** los **duplica**, y que la idempotencia cierra el caso.

### 2a) at-most-once (commit antes) → mensaje perdido

```ts
// offsets.ts
type Mensaje = { offset: number; valor: string }
const log: Mensaje[] = [
  { offset: 0, valor: 'pago:A' },
  { offset: 1, valor: 'pago:B' },
  { offset: 2, valor: 'pago:C' },
]

function consumirAtMostOnce(crashEnOffset: number): { procesados: string[]; committed: number } {
  const procesados: string[] = []
  let committed = -1
  for (const m of log) {
    committed = m.offset                 // (1) COMMIT primero
    if (m.offset === crashEnOffset) {     // 💥 crash justo después de commitear, antes de procesar
      console.log(`💥 crash tras commitear offset ${m.offset}, sin procesarlo`)
      break
    }
    procesados.push(m.valor)              // (2) procesar
  }
  return { procesados, committed }
}

const r1 = consumirAtMostOnce(1)
console.log('procesados:', r1.procesados, '| committed:', r1.committed)
// procesados: ['pago:A'] | committed: 1   -> pago:B quedó committeado pero NUNCA procesado: PERDIDO
```

**Qué observás.** El crash cae tras commitear el offset 1 pero antes de procesar `pago:B`. Al reiniciar, el consumidor arranca desde el offset **2** (ya "leído" el 1) → **`pago:B` se perdió**. Cero duplicados, pero un pago no se procesó nunca.

### 2b) at-least-once (commit después) → mensaje duplicado

```ts
// (mismo log)
function consumirAtLeastOnce(crashEnOffset: number): { procesados: string[]; committed: number } {
  const procesados: string[] = []
  let committed = -1
  for (const m of log) {
    procesados.push(m.valor)              // (1) procesar primero
    if (m.offset === crashEnOffset) {     // 💥 crash tras procesar, antes de commitear
      console.log(`💥 crash tras procesar offset ${m.offset}, sin commitear`)
      break
    }
    committed = m.offset                  // (2) COMMIT después
  }
  return { procesados, committed }
}

const r2 = consumirAtLeastOnce(1)
console.log('procesados antes del crash:', r2.procesados, '| committed:', r2.committed)
// procesados: ['pago:A', 'pago:B'] | committed: 0  -> al reiniciar desde offset 1, pago:B se procesa OTRA VEZ
```

**Qué observás.** El crash cae tras procesar `pago:B` pero antes de commitearlo (committed quedó en 0). Al reiniciar, arranca desde el offset **1** → **`pago:B` se procesa de nuevo** → duplicado (¡cobraste dos veces!). No se perdió nada, pero se duplicó.

### 2c) El fix: at-least-once + idempotencia

```ts
// procesador idempotente: deduplica por id de operación (módulo 6)
const yaProcesados = new Set<string>()
function procesarIdempotente(idOperacion: string, efecto: () => void): void {
  if (yaProcesados.has(idOperacion)) {
    console.log(`  ${idOperacion} ya procesado, se ignora (idempotente)`)
    return
  }
  efecto()
  yaProcesados.add(idOperacion)
}

// 'pago:B' llega dos veces (el duplicado de 2b); el efecto ocurre UNA sola vez
let cobros = 0
procesarIdempotente('pago:B', () => { cobros++ })   // procesa
procesarIdempotente('pago:B', () => { cobros++ })   // reproceso del crash: ignorado
console.log('cobros a pago:B:', cobros)              // 1  -> exactly-once en el EFECTO
```

**Qué observás.** Aunque `pago:B` se entregue dos veces (por el reproceso del crash de 2b), el efecto (`cobros++`) ocurre **una sola vez**, porque el procesador deduplica por `idOperacion`. At-least-once + idempotencia = exactly-once en el **efecto observable**, que es lo único alcanzable (módulo 6).

**Qué demuestra.** El trade-off del módulo 6: commitear antes pierde, commitear después duplica; el default sano es **at-least-once + idempotencia**, no rezar por que no haya crashes. Exactly-once "de verdad" (sin idempotencia) cruzando a un sistema externo no existe.

**Extensiones**
- 2.1 🧠 ¿Por qué at-most-once igual existe y se usa? Dá un caso donde perder un mensaje es preferible a duplicarlo.
- 2.2 ✍️ Reemplazá el `Set` en memoria por una "tabla" (Map persistente simulado) con `INSERT ... ON CONFLICT` y mostrá que sobrevive a un "reinicio" del consumidor.
- 2.3 🧠 ¿En qué cambia este panorama si todo el pipeline vive dentro de Kafka (transacciones / EOS) y por qué ahí sí podés hablar de exactly-once?

---

## Build 3 — Ventana por event-time (con un evento tardío)

**El problema.** Agregamos eventos en ventanas tumbling de 1 minuto **por event-time**. Un watermark cierra cada ventana. Vamos a ver qué pasa cuando un evento llega **tarde** (después de que su ventana cerró): se descarta, salvo que uses allowed lateness.

```ts
// ventana.ts
type EventoTs = { ts: number; valor: number }      // ts en segundos (event-time)
const VENTANA = 60                                   // ventana tumbling de 60s

const ventanaDe = (ts: number): number => Math.floor(ts / VENTANA) * VENTANA   // inicio de la ventana

class Agregador {
  private ventanas = new Map<number, number>()       // inicioVentana -> suma
  private cerradas = new Set<number>()
  private watermark = -Infinity

  // procesa un evento; ALLOWED_LATENESS = cuántos segundos toleramos tras cerrar
  procesar(e: EventoTs, allowedLateness = 0): 'contado' | 'tardío-descartado' | 'tardío-corregido' {
    const w = ventanaDe(e.ts)
    // avanza el watermark: "vi todo hasta el mayor event-time menos un margen" (margen 0 acá para simplificar)
    this.watermark = Math.max(this.watermark, e.ts)
    if (this.cerradas.has(w)) {
      if (e.ts >= w && this.watermark - (w + VENTANA) <= allowedLateness) {
        this.ventanas.set(w, (this.ventanas.get(w) ?? 0) + e.valor)   // dentro del lateness: corrige
        return 'tardío-corregido'
      }
      return 'tardío-descartado'                                      // ya cerró y fuera de lateness
    }
    this.ventanas.set(w, (this.ventanas.get(w) ?? 0) + e.valor)
    return 'contado'
  }
  // cierra todas las ventanas cuyo fin quedó por debajo del watermark
  cerrarHasta(): void {
    for (const inicio of this.ventanas.keys()) {
      if (inicio + VENTANA <= this.watermark) this.cerradas.add(inicio)
    }
  }
  total(w: number): number { return this.ventanas.get(w) ?? 0 }
}

// eventos de la ventana [0,60): llegan en orden, suman 30
const agg = new Agregador()
console.log(agg.procesar({ ts: 10, valor: 10 }))   // contado
console.log(agg.procesar({ ts: 50, valor: 20 }))   // contado
// llega un evento de event-time 70 -> avanza el watermark a 70 -> la ventana [0,60) puede cerrar
console.log(agg.procesar({ ts: 70, valor: 5 }))    // contado (va a la ventana [60,120))
agg.cerrarHasta()                                    // cierra [0,60) porque 60 <= watermark(70)

// AHORA llega tarde un evento de event-time 30 (pertenecía a [0,60), ya cerrada)
console.log('total [0,60) antes del tardío:', agg.total(0))   // 30
console.log(agg.procesar({ ts: 30, valor: 100 }, 0))          // 'tardío-descartado'
console.log('total [0,60) tras el tardío (lateness 0):', agg.total(0))   // 30  -> el tardío NO se contó
```

**Qué observás.** El evento de event-time 30 llegó **después** de que la ventana `[0,60)` cerró (el watermark ya estaba en 70). Con `allowedLateness = 0`, se **descarta**: el total de esa ventana queda en 30, sin sus 100. El total mintió por bajo — exactamente la falla del intro del módulo.

**El fix con allowed lateness:**

```ts
const agg2 = new Agregador()
agg2.procesar({ ts: 10, valor: 10 })
agg2.procesar({ ts: 50, valor: 20 })
agg2.procesar({ ts: 70, valor: 5 })      // watermark -> 70
agg2.cerrarHasta()                         // cierra [0,60)
// con allowedLateness=30: el watermark (70) - fin de ventana (60) = 10 <= 30 -> se corrige
console.log(agg2.procesar({ ts: 30, valor: 100 }, 30))      // 'tardío-corregido'
console.log('total [0,60) con lateness 30:', agg2.total(0))  // 130  -> ¡se corrigió!
```

**Qué demuestra.** Los módulos 8-10: agrupás por event-time (el evento 30 pertenece a `[0,60)` aunque llegue último); el watermark decide cuándo cerrar; un tardío fuera del margen se pierde, pero `allowedLateness` mantiene la ventana viva un rato para corregir el resultado. El trade-off completitud-vs-latencia hecho código.

**Extensiones**
- 3.1 🧠 ¿Por qué este agregador agrupa por `e.ts` (event-time) y no por el orden de llegada? Mostrá qué pasaría con el evento tardío si agrupara por orden de llegada.
- 3.2 ✍️ Agregá un margen al watermark (`watermark = maxTs - margen`) y mostrá que un margen mayor reduce los descartes pero demora el cierre de las ventanas.
- 3.3 🧠 Más allá del allowed lateness, ¿a dónde mandarías los eventos "muy tardíos" y cómo corregirías el total histórico?

---

## Build 4 — Replay estilo Kappa

**El problema.** Tenemos un bug en la agregación y queremos recalcular el mes pasado. Vamos a ver que con **event-time** reprocesar el log da el resultado correcto (Kappa), y que con **processing-time** daría otro resultado cada vez.

```ts
// kappa.ts — el log es inmutable y releíble (módulo 3)
type Registro = { eventTime: number; valor: number }
const logInmutable: Registro[] = [
  { eventTime: 10, valor: 1 },
  { eventTime: 50, valor: 2 },
  { eventTime: 70, valor: 4 },
  { eventTime: 90, valor: 8 },
]
const VENTANA = 60
const ventanaDe = (ts: number) => Math.floor(ts / VENTANA) * VENTANA

// agrupa por EVENT-TIME: el resultado depende solo de los datos, no de cuándo proceso
function agregarPorEventTime(log: Registro[]): Map<number, number> {
  const out = new Map<number, number>()
  for (const r of log) {
    const w = ventanaDe(r.eventTime)
    out.set(w, (out.get(w) ?? 0) + r.valor)
  }
  return out
}

// versión 1 (con "bug": suma). Reproceso desde el offset 0 con la versión corregida.
const v1 = agregarPorEventTime(logInmutable)
console.log('v1 (event-time):', [...v1])   // [[0,3],[60,12]]  ventana [0,60)=1+2, [60,120)=4+8

// "arreglamos el bug" y reprocesamos EL MISMO log -> mismo resultado, reproducible
const v2 = agregarPorEventTime(logInmutable)
console.log('reproceso == original:', JSON.stringify([...v1]) === JSON.stringify([...v2]))  // true
```

**Qué observás.** Reprocesar el log por event-time da **exactamente** el mismo resultado: `[0,60)=3`, `[60,120)=12`. Es reproducible — por eso podés releer el log desde el offset 0 con una versión corregida del job y confiar en el resultado (Kappa, módulo 15).

**Contraste — agrupar por processing-time NO es reproducible:**

```ts
// agrupa por ORDEN DE LLEGADA (processing-time simulado): la ventana depende de CUÁNDO proceso
function agregarPorProcessingTime(log: Registro[]): Map<number, number> {
  const out = new Map<number, number>()
  log.forEach((r, indiceLlegada) => {
    const w = ventanaDe(indiceLlegada * 30)   // el "tiempo" es el momento de proceso, no del evento
    out.set(w, (out.get(w) ?? 0) + r.valor)
  })
  return out
}

const p1 = agregarPorProcessingTime(logInmutable)
// si reprocesamos con OTRO orden de llegada (la red reordenó), el resultado cambia:
const reordenado = [logInmutable[2], logInmutable[0], logInmutable[3], logInmutable[1]]
const p2 = agregarPorProcessingTime(reordenado)
// p1: [[0,3],[60,12]]   vs   p2: [[0,5],[60,10]]   -> los MISMOS valores caen en otras ventanas según el orden
console.log('processing-time reproducible?', JSON.stringify([...p1]) === JSON.stringify([...p2]))  // false
```

**Qué observás.** Agrupando por processing-time, reprocesar el **mismo** dato en otro orden de llegada da un resultado **distinto** (`false`): los eventos caen en ventanas distintas según cuándo se procesaron. No es reproducible → no podés recalcular confiando en el resultado.

**Qué demuestra.** El módulo 7 y 15: la reproducibilidad de Kappa **depende** de agrupar por event-time. Un log inmutable y releíble + event-time = podés reprocesar para arreglar bugs o cambiar la lógica, y el resultado es determinístico. Processing-time rompe esa propiedad.

**Extensiones**
- 4.1 🧠 ¿Por qué Kappa necesita las DOS cosas (log releíble Y event-time) y no le alcanza con una sola?
- 4.2 ✍️ Simulá el "switch" de Kappa: corré el job viejo y el nuevo (otra agregación, p. ej. `max` en vez de `sum`) sobre el mismo log y mostrá que ambos resultados conviven hasta que cortás al nuevo.
- 4.3 🧠 ¿Qué ventaja tiene Kappa (una base de código) sobre Lambda (dos) y en qué caso Lambda sigue teniendo sentido?

---

## Build 5 — Scan OLTP (fila) vs OLAP (columna)

**El problema.** Una tabla de ventas con muchas columnas. Una consulta analítica (`avg(monto)`) sobre millones de filas. Vamos a **contar cuántos valores lee** un row-store vs un column-store y ver por qué el columnar gana en analytics.

```ts
// storage.ts
type Venta = { id: number; usuario: string; region: string; monto: number; fecha: string; /* +36 cols */ }
const COLUMNAS = 40                                   // la tabla tiene 40 columnas en total

// --- ROW STORE: las filas están contiguas; un scan lee filas ENTERAS ---
function avgMontoRowStore(filas: Venta[]): { avg: number; valoresLeidos: number } {
  let suma = 0
  // para escanear la tabla, el row-store trae cada fila completa del disco (todas sus columnas)
  for (const f of filas) suma += f.monto
  const valoresLeidos = filas.length * COLUMNAS       // leíste N filas × 40 columnas de bytes
  return { avg: suma / filas.length, valoresLeidos }
}

// --- COLUMN STORE: cada columna está contigua; leés SOLO la columna que necesitás ---
function avgMontoColumnStore(columnaMonto: number[]): { avg: number; valoresLeidos: number } {
  let suma = 0
  for (const m of columnaMonto) suma += m
  const valoresLeidos = columnaMonto.length           // leíste N valores de UNA columna
  return { avg: suma / columnaMonto.length, valoresLeidos }
}

const N = 1_000_000
const filas: Venta[] = Array.from({ length: N }, (_, i) =>
  ({ id: i, usuario: `u${i % 1000}`, region: 'AR', monto: i % 100, fecha: '2026-06' }))
const columnaMonto = filas.map(f => f.monto)          // en un column-store esta columna vive aparte

const r = avgMontoRowStore(filas)
const c = avgMontoColumnStore(columnaMonto)
console.log('row-store    leyó:', r.valoresLeidos, 'valores')   // 40_000_000
console.log('column-store leyó:', c.valoresLeidos, 'valores')   // 1_000_000
console.log('ratio I/O:', r.valoresLeidos / c.valoresLeidos, 'x')  // 40x menos I/O
```

**Qué observás.** Para el mismo `avg(monto)`, el row-store "lee" 40 millones de valores (1M filas × 40 columnas, porque las filas están contiguas y el scan las trae enteras) y el column-store lee 1 millón (solo la columna `monto`) → **40× menos I/O**, justo el número de columnas que la consulta **no** necesita. En una tabla real con cientos de columnas, la diferencia es brutal.

**Qué demuestra.** El módulo 13: el column-store gana en analytics porque **lee solo las columnas de la consulta** (no las filas enteras). Sumá la mejor compresión (una columna tiene valores homogéneos → comprime 10×, extensión 5.2) y la ejecución vectorizada, y tenés por qué los warehouses son columnares y por qué no corrés analytics sobre tu OLTP row-store de producción.

**Extensiones**
- 5.1 🧠 Si la consulta fuera `SELECT * FROM ventas WHERE id = 123` (traer una fila entera), ¿qué store gana y por qué? Conectalo con OLTP.
- 5.2 ✍️ Simulá compresión: una columna `region` con 1M de `'AR'` se comprime con run-length a ~1 valor + count. Estimá el ratio vs guardarla fila por fila.
- 5.3 🧠 ¿Por qué un column-store es **malo** para escrituras transacionales (insertar/actualizar una fila)? Conectalo con por qué OLTP sigue siendo row-store.

---

## Soluciones de las extensiones

**1.1** Cuentas distintas son **independientes**: el orden entre eventos de `c1` y `c2` no afecta la corrección (procesar `c2:crear` antes o después de `c1:depositar` da lo mismo). Eventos de la **misma** cuenta tienen dependencia causal (`crear` → `depositar` → `retirar`): procesarlos fuera de orden corrompe el estado (depositar en una cuenta inexistente). Por eso particionás por cuenta: paraleliza entre cuentas, ordena dentro de cada una.

**1.2** Producís 10 000 con `key` = `'u0'` el 90% y `'u1'..'uN'` el resto; contás `topic.particiones[i].length`. Vas a ver una partición (la de `u0`) con ~9000 eventos y las otras con el reparto del 10% restante → **hot partition**: su consumidor se satura mientras los demás están casi ociosos. La key de alta frecuencia rompe el balance.

**1.3** Opciones: (a) **más particiones** (subís el techo de paralelismo, pero más particiones = más overhead de metadata, más rebalanceos, y no podés reducirlas fácil); (b) **sub-key** / particionar más fino (p. ej. `userId:tipoEvento`) si no necesitás orden total por usuario sino por una clave más granular (perdés el orden global del usuario); (c) procesar **dentro** del consumidor en paralelo respetando el orden por sub-stream. Todas cambian orden por paralelismo: elegís según qué orden necesitás de verdad.

**2.1** At-most-once se usa cuando **duplicar es peor que perder** o cuando el volumen es enorme y un dato suelto no importa: métricas/telemetría best-effort (perder una muestra de CPU no cambia el promedio, pero contarla dos veces sí lo sesga), logs de alta frecuencia, o casos donde reprocesar tiene un costo/efecto colateral inaceptable y el dato es desechable.

**2.2** Reemplazás `yaProcesados` por un `Map<string, true>` que represente una tabla con constraint único; `procesarIdempotente` hace el equivalente a `INSERT idOperacion ON CONFLICT DO NOTHING` y solo ejecuta el efecto si la inserción "ganó". Como la tabla es durable (sobrevive al reinicio), tras un crash el reproceso encuentra el id ya insertado y no duplica. (En prod: una tabla de idempotencia en la misma DB que el efecto, en la misma transacción.)

**2.3** Dentro de Kafka, las **transacciones (EOS)** atan "leer offset → procesar → escribir a otro topic → commitear el offset" en una sola unidad atómica: si falla, se aborta todo y no hay efecto parcial ni offset avanzado. Ahí sí hablás de exactly-once **processing** porque no hay un sistema externo no transaccional en el medio; el efecto (escribir a Kafka) y el avance del offset commitean juntos. Cruzando a una DB/API externa, volvés a necesitar idempotencia.

**3.1** Porque la ventana de un evento debe depender de **cuándo ocurrió** (su `ts`), no de cuándo lo viste. El evento tardío de `ts=30` **pertenece** a la ventana `[0,60)` aunque llegue último; agrupando por event-time podemos (con lateness) sumarlo a la ventana correcta. Si agrupáramos por orden de llegada, ese evento caería en una ventana "actual" equivocada (junto a eventos de `ts=70+`), ensuciando dos ventanas a la vez y dando un total irreproducible.

**3.2** Cambiás el avance a `this.watermark = Math.max(this.watermark, e.ts - margen)`. Con `margen=20`, tras ver `ts=70` el watermark es `50`, así que `[0,60)` **no** cierra todavía (espera más eventos posibles) → el evento de `ts=30` que llega después **todavía entra** sin necesidad de lateness. Pero la ventana `[0,60)` emite su resultado más tarde (cuando el watermark supere 60, es decir cuando veas un `ts≥80`). Margen mayor = menos descartes, más latencia de cierre.

**3.3** Los "muy tardíos" (fuera del allowed lateness) van a un **side output** (un topic/tabla aparte). Corregís el total histórico **en batch**: un job periódico relee esos tardíos y **actualiza** los agregados ya emitidos en el store de servicio (un `UPDATE` sobre la ventana afectada). Es la idea de Lambda: el stream da el número rápido y aproximado, el batch lo reconcilia (módulo 15).

**4.1** Necesita el **log releíble** para poder volver atrás y reprocesar (sin él no hay "desde el offset 0"). Necesita **event-time** para que ese reproceso sea **determinístico** (mismos datos → mismas ventanas → mismo resultado). Con log releíble pero processing-time, reprocesar daría otro resultado (build 4) → inútil para recalcular. Con event-time pero sin log releíble, no tendrías los datos viejos para reprocesar. Las dos juntas habilitan Kappa.

**4.2** Corrés `agregarPorEventTime` (sum) y una variante `agregarPorEventTimeMax` (max) sobre el **mismo** `logInmutable`, escribiendo a dos destinos distintos (p. ej. dos Maps / dos topics de salida). Ambos resultados existen a la vez (el viejo sigue sirviendo a los consumidores actuales); cuando el job nuevo alcanzó el presente, redirigís los lectores al resultado nuevo y dás de baja el viejo. Es el "switch" de Kappa sin downtime.

**4.3** Kappa tiene **una sola base de código** (un job de stream) → menos mantenimiento, sin riesgo de que la lógica batch y la stream diverjan (el dolor central de Lambda, que tiene dos). Lambda sigue teniendo sentido cuando el batch hace algo que el stream no puede o no conviene: joins masivos contra todo el histórico, entrenamiento de ML pesado, o cuando reprocesar todo el log por stream sería más caro/lento que un batch optimizado.

**5.1** Gana el **row-store** (OLTP): traer la fila 123 entera necesita **una** lectura contigua (la fila está junta en disco), mientras que el column-store tendría que ir a **40 columnas distintas** y reensamblar la fila (40 accesos dispersos). Las consultas puntuales por clave que tocan toda la fila son justo el patrón OLTP, y por eso las DBs transaccionales son row-store.

**5.2** Run-length encoding guarda `('AR', 1_000_000)` en vez de un millón de `'AR'` → ~2 valores vs 1M → ratio ~500_000×, idealizado (en la práctica hay bloques, igual es enorme). Fila por fila no podés comprimir así porque el `'AR'` está intercalado con valores de otras columnas (heterogéneos). La homogeneidad de la columna es lo que habilita la compresión brutal.

**5.3** Insertar/actualizar **una fila** en un column-store obliga a tocar **todas** las columnas en sus ubicaciones **separadas** (40 escrituras dispersas), y rompe la compresión por bloques de cada columna (hay que descomprimir/recomprimir). En un row-store, la fila entera es una escritura contigua. Por eso OLTP (muchas escrituras/updates de filas) sigue siendo row-store, y el columnar se reserva para analytics (cargas masivas + lecturas por columna, pocas escrituras puntuales).
