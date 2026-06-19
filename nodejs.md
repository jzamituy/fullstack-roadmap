# Node.js por dentro: el runtime que sostiene tu backend

**Event loop, async de verdad, streams y escalado · los fundamentos que se dan por sabidos · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Hasta acá usaste NestJS, Postgres y auth dando por hecho que "Node ejecuta tu código". Este módulo abre esa caja: **cómo** Node maneja miles de requests con **un solo hilo**, por qué `async/await` no es magia, qué pasa si bloqueás el event loop, y cómo escala un proceso Node. No es teoría académica: el event loop es **la pregunta de entrevista más común** de Node, y entender streams, concurrencia y graceful shutdown es lo que separa a quien "usa Express" de quien opera un backend en producción.

**Por qué importa ahora.** Todo lo que viste asume esto por debajo: el `await` de tus queries Drizzle, los timeouts de resiliencia del archivo senior, el graceful shutdown de Kubernetes, el "escalado horizontal" del roadmap. Sin entender el runtime, esos temas quedan como recetas sin el porqué.

**Para practicar.** Te alcanza con Node instalado (`node -v`, idealmente LTS) y correr los snippets con `node archivo.js` (o `ts-node` / `tsx` para TypeScript). Muchos ejercicios son de **predecir la salida**: hacelo en papel antes de correrlo.

**Índice de módulos**
1. El modelo de Node: un solo hilo, I/O no bloqueante
2. El event loop: cómo un hilo hace tantas cosas
3. Macrotasks vs. microtasks (el orden de ejecución)
4. De callbacks a async/await (y qué es async "de verdad")
5. Manejo de errores asíncronos
6. Concurrencia: secuencial vs. paralelo (`Promise.all` y amigos)
7. No bloquees el event loop (trabajo CPU-bound)
8. Streams y backpressure
9. EventEmitter: el patrón base de Node
10. Módulos: CommonJS vs. ESM
11. `process`, entorno y ciclo de vida (graceful shutdown)
12. Cómo escala Node: cluster, worker threads y horizontal

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El modelo de Node: un solo hilo, I/O no bloqueante

**Teoría.** La pregunta clásica: "¿cómo hace Node, con **un solo hilo** de JavaScript, para servir miles de conexiones a la vez?" La respuesta no es "hace muchas cosas al mismo tiempo" — es que **casi nunca espera**.

La clave es distinguir dos tipos de trabajo:

- **I/O-bound** (entrada/salida): leer un archivo, consultar la base, llamar a otra API. El tiempo se va **esperando** una respuesta externa, no usando CPU.
- **CPU-bound**: calcular un hash, procesar una imagen, ordenar un millón de elementos. El tiempo se va **usando la CPU**.

Node está optimizado para lo primero. Cuando tu código pide una operación de I/O (`await db.query(...)`), Node **no se queda esperando bloqueado**: delega esa operación al sistema operativo (vía `libuv`), **sigue ejecutando** otro código, y cuando la I/O termina, retoma con el resultado. Por eso un solo hilo alcanza para miles de requests concurrentes: mientras uno espera la base, el hilo atiende a otros.

```ts
// Bloqueante (MAL): el hilo se queda parado hasta que termina de leer
const datos = fs.readFileSync("archivo.txt"); // nada más corre mientras tanto

// No bloqueante (BIEN): Node delega la lectura y sigue; retoma cuando está lista
const datos = await fs.promises.readFile("archivo.txt"); // el hilo atiende otras cosas
```

El corolário, que vas a profundizar en el módulo 7: Node brilla con I/O, pero si le metés trabajo **CPU-bound** pesado en el hilo principal, lo **bloqueás** — y como es un solo hilo, *toda* la app se congela.

**Un matiz clave: el thread pool de libuv.** "No bloqueante en un solo hilo" no significa que *toda* la I/O sea mágica en el kernel. Hay que distinguir:

- **I/O de red** (sockets, HTTP, TCP): es realmente event-driven en el sistema operativo (`epoll`/`kqueue`); no usa hilos extra.
- **I/O de filesystem (`fs.*`), `dns.lookup`, y operaciones de librerías nativas como `crypto` (`pbkdf2`, `bcrypt`) o `zlib`**: corren en el **thread pool de libuv**, que por defecto tiene **4 hilos** (configurable con la variable `UV_THREADPOOL_SIZE`).

Esto importa en producción: si disparás muchos `bcrypt` o lecturas de archivo a la vez, podés **saturar esos 4 hilos** y formar una cola, aunque tu event loop esté libre. Por eso un `bcrypt` con factor de costo alto es, en la práctica, trabajo pesado a vigilar (conecta con el módulo 7).

**Ejercicios 1**
1.1 En una frase: ¿cómo puede un solo hilo de JavaScript atender miles de requests concurrentes?
1.2 Clasificá cada tarea como I/O-bound o CPU-bound: (a) consultar Postgres; (b) calcular el factorial de un número gigante; (c) llamar a la API de Stripe; (d) redimensionar una imagen.
1.3 ¿Por qué `fs.readFileSync` es peligroso en un servidor que atiende muchos requests, y `fs.promises.readFile` no?

---

## Módulo 2 — El event loop: cómo un hilo hace tantas cosas

**Teoría.** El **event loop** es el corazón de Node: un bucle que coordina qué se ejecuta y cuándo. Tu código JavaScript corre en el hilo principal; las operaciones de I/O se delegan; y cuando terminan, sus **callbacks** se encolan para que el event loop los ejecute en el momento adecuado.

El event loop pasa por **fases**, en orden, en cada vuelta ("tick"). Las que importan conocer:

1. **Timers**: ejecuta callbacks de `setTimeout` / `setInterval` vencidos.
2. **Pending callbacks**: ciertos callbacks de I/O diferidos (p. ej. algunos errores de TCP). Es una fase aparte, distinta de "poll".
3. **Poll**: acá se procesan la mayoría de los callbacks de I/O (lecturas, red, DB). Es donde Node "espera" eventos nuevos si no hay nada pendiente.
4. **Check**: ejecuta callbacks de `setImmediate`.
5. **Close**: callbacks de cierre (ej. un socket que se cerró).

(Node tiene además fases internas "idle/prepare" de uso interno que no necesitás manejar.)

```
   ┌─────────────┐
┌─>│   timers    │  setTimeout, setInterval
│  ├─────────────┤
│  │  pending    │  callbacks de I/O diferidos (errores TCP, etc.)
│  ├─────────────┤
│  │    poll     │  ← I/O: red, archivos, DB (acá se pasa la mayor parte del tiempo)
│  ├─────────────┤
│  │    check    │  setImmediate
│  ├─────────────┤
│  │    close    │
│  └─────────────┘
└──── (siguiente tick) ────
```

La idea central: el event loop solo ejecuta **una cosa a la vez** (es un solo hilo). Si una de esas "cosas" tarda mucho (un cálculo pesado, un `while` infinito), **bloquea todo el loop**: ningún timer, ninguna I/O, ningún request se atiende hasta que termine. Esa es la razón por la que "no bloquear el event loop" es un mandamiento en Node.

**Importante:** entre cada callback (y entre fases), Node vacía las **microtasks** (módulo 3). Ese detalle explica el orden de salida que confunde a todos.

**`setTimeout(fn, 0)` vs `setImmediate(fn)`** (pregunta clásica de entrevista): en el **nivel top-level** el orden **no es determinista** (depende de cuánto tardó en arrancar el loop). Pero **dentro de un callback de I/O** (fase poll), `setImmediate` **siempre** corre antes que `setTimeout(fn, 0)`, porque la fase *check* viene justo después de *poll*, mientras que los timers recién se ejecutan en la vuelta siguiente.

**Ejercicios 2**
2.1 ¿En qué fase del event loop se ejecutan los callbacks de `setTimeout`? ¿Y los de la mayoría de las operaciones de I/O?
2.2 Si dentro de un request ejecutás un `for` que tarda 5 segundos haciendo cálculos, ¿qué les pasa a los otros requests que llegan en ese momento? ¿Por qué?
2.3 ¿Verdadero o falso? "Node ejecuta varios callbacks de JavaScript en paralelo". Justificá.

---

## Módulo 3 — Macrotasks vs. microtasks (el orden de ejecución)

**Teoría.** Esta es **la** pregunta de entrevista. Hay dos colas con prioridades distintas:

- **Macrotasks** (o "tasks"): callbacks de `setTimeout`, `setInterval`, `setImmediate`, I/O. Se ejecutan en las fases del event loop (módulo 2).
- **Microtasks**: callbacks de **Promises** (`.then`, `await`) y `queueMicrotask`. Además, `process.nextTick` tiene una cola propia con prioridad **aún mayor**.

La regla de oro (en Node): **después de cada callback individual y entre cada fase del event loop** (no solo "después de cada macrotask"), Node vacía primero la cola de `process.nextTick` y luego la de microtasks/Promises, antes de continuar. Es una diferencia con el navegador, que las vacía solo al terminar cada tarea.

```ts
console.log("1: síncrono");

setTimeout(() => console.log("2: setTimeout (macrotask)"), 0);

Promise.resolve().then(() => console.log("3: promise (microtask)"));

process.nextTick(() => console.log("4: nextTick (microtask prioritaria)"));

console.log("5: síncrono");
```

Salida:

```
1: síncrono
5: síncrono
4: nextTick (microtask prioritaria)
3: promise (microtask)
2: setTimeout (macrotask)
```

El razonamiento: primero corre **todo el código síncrono** (`1`, `5`). Al terminar, Node vacía las microtasks: `nextTick` primero (`4`), luego las Promises (`3`). Recién entonces avanza a la fase de timers y ejecuta el `setTimeout` (`2`). Entender esto te explica por qué un `await` "se cuela" antes que un `setTimeout(…, 0)`.

**Cuidado con `process.nextTick`:** como tiene la máxima prioridad, si lo usás en recursión puede **matar de hambre** al event loop (nunca llega a la I/O). Por eso, para "diferir al próximo tick" sin riesgo, suele preferirse `setImmediate`.

**Ejercicios 3**
3.1 Predecí la salida exacta:
```ts
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
console.log("D");
```
3.2 ¿Por qué un callback de `Promise.then` se ejecuta antes que uno de `setTimeout(fn, 0)`, aunque el timeout sea "0 ms"?
3.3 ¿Qué riesgo tiene usar `process.nextTick` de forma recursiva?

---

## Módulo 4 — De callbacks a async/await (y qué es async "de verdad")

**Teoría.** El asincronismo en Node evolucionó en tres etapas. Vale conocerlas porque vas a leer código viejo en todas:

**1. Callbacks** (estilo "error-first"): pasás una función que se llama al terminar. El problema es el "callback hell": anidamiento ilegible cuando encadenás varios.

```ts
fs.readFile("a.txt", (err, data) => {
  if (err) return manejar(err);
  fs.readFile("b.txt", (err2, data2) => { /* ...anidado y frágil... */ });
});
```

**2. Promises**: un objeto que representa un valor futuro (`pending` → `fulfilled`/`rejected`). Se encadenan con `.then`/`.catch`, aplanando el anidamiento.

**3. async/await**: azúcar sintáctico **sobre Promises**. `await` "pausa" la función hasta que la Promise se resuelve, pero **sin bloquear el hilo** — Node sigue atendiendo otras cosas mientras tanto. El código asíncrono se lee como síncrono. Por dentro no es una transpilación literal a `.then`: el motor compila la función `async` como una **máquina de estados** (una corrutina) que **suspende** la ejecución en cada `await` y la **reanuda** cuando la Promise se resuelve, conservando las variables locales entre medio. Eso es lo que realmente es "async de verdad".

```ts
async function cargar(): Promise<string> {
  const a = await fs.promises.readFile("a.txt", "utf8");
  const b = await fs.promises.readFile("b.txt", "utf8");
  return a + b;
}
```

**El malentendido que hay que matar:** `async/await` **no hace tu código más rápido ni paralelo**. Un `await` detrás de otro corre **en secuencia** (esperás `a`, después pedís `b`). Y `async` solo significa "esta función devuelve una Promise". Si las dos lecturas son independientes, encadenarlas con `await` desperdicia tiempo — eso lo arregla la concurrencia del módulo 6.

Otro punto clave que conecta con TypeScript: una función `async` **siempre** devuelve una `Promise<T>`, nunca `T` directo. Por eso tipás `Promise<string>`, no `string` — exactamente como hiciste en los repositorios (`findById(id): Promise<Usuario | null>`).

**Ejercicios 4**
4.1 ¿`async/await` ejecuta tu código más rápido que las Promises con `.then`? ¿Qué es realmente `async/await`?
4.2 Estas dos lecturas son independientes. ¿Corren en paralelo o en secuencia tal como está? ¿Está bien?
```ts
const a = await leer("a.txt");
const b = await leer("b.txt");
```
4.3 Una función `async function f()` que hace `return 42`. ¿Qué tipo devuelve realmente? ¿Cómo lo tiparías en TypeScript?

---

## Módulo 5 — Manejo de errores asíncronos

**Teoría.** Con `async/await`, los errores se manejan con `try/catch` normal: una Promise rechazada hace que el `await` **lance** la excepción, que cae en el `catch`.

```ts
async function obtenerUsuario(id: number): Promise<Usuario> {
  try {
    const usuario = await repo.findById(id);
    if (!usuario) throw new NotFoundException(`Usuario ${id} no existe`);
    return usuario;
  } catch (err) {
    // logueás / traducís / re-lanzás
    throw err;
  }
}
```

Las trampas que hay que conocer:

- **`await` olvidado**: si llamás a una función async sin `await` (ni `.then/.catch`), el error que lance se vuelve una **unhandled rejection** que tu `try/catch` **no** atrapa, porque la función ya retornó. La Promise "flota" sola.

```ts
// MAL: el try/catch NO atrapa nada; si guardar() falla, es una unhandled rejection
try {
  repo.guardar(usuario); // falta await
} catch (err) { /* nunca entra acá */ }
```

- **`forEach` con async no espera**: `array.forEach(async ...)` lanza todas las Promises pero **no** las espera. Para esperar en un loop, usá `for...of` con `await`, o `Promise.all` (módulo 6).
- **Red de seguridad global**: registrá handlers de `process.on("unhandledRejection")` y `uncaughtException` para loguear y apagar ordenadamente; pero son la **última** línea, no el mecanismo normal de manejo. Importante (2026): en las versiones LTS actuales (Node 18/20/22) el comportamiento por defecto ante una unhandled rejection es **terminar el proceso** (`--unhandled-rejections=throw`); antes de Node 15 solo emitía un warning. Razón de más para `await`/`.catch` siempre.

La regla: **siempre** `await` (o `.catch`) toda Promise. Una Promise sin manejar es un error que se pierde — o peor, tira el proceso.

**Ejercicios 5**
5.1 ¿Por qué este `try/catch` no atrapa el error de `guardar()`?
```ts
try { repo.guardar(u); } catch (e) { console.log("ups"); }
```
5.2 Querés procesar un array de ids llamando a `await borrar(id)` para cada uno y esperar a que **todos** terminen. ¿Por qué `ids.forEach(async (id) => await borrar(id))` no hace lo que esperás? ¿Qué usarías?
5.3 ¿Qué es una "unhandled promise rejection" y por qué conviene tener un handler global de `unhandledRejection`?

---

## Módulo 6 — Concurrencia: secuencial vs. paralelo (`Promise.all` y amigos)

**Teoría.** Como Node delega la I/O, podés **disparar varias operaciones a la vez** y esperar a que terminen todas, en vez de una tras otra. Esto suele ser la optimización de mayor impacto en un backend.

```ts
// SECUENCIAL: 3 queries, una espera a la anterior → suma de los tiempos (~300ms si cada una tarda 100)
const usuario = await repo.usuario(id);
const proyectos = await repo.proyectos(id);
const tareas = await repo.tareas(id);

// PARALELO: las 3 arrancan juntas → tarda lo que la más lenta (~100ms)
const [usuario, proyectos, tareas] = await Promise.all([
  repo.usuario(id),
  repo.proyectos(id),
  repo.tareas(id),
]);
```

Las herramientas:

- **`Promise.all([...])`**: espera a que **todas** resuelvan; si **una** rechaza, falla todo de inmediato (rejects con el primer error). Usalo cuando necesitás todos los resultados y querés cortar ante cualquier fallo.
- **`Promise.allSettled([...])`**: espera a todas y te da el estado de **cada una** (`fulfilled`/`rejected`) sin cortar. Ideal cuando un fallo parcial es tolerable (ej. notificar a 100 usuarios; que falle uno no debe frenar al resto).
- **`Promise.race([...])`**: resuelve/rechaza con la **primera** que termine. Útil para timeouts ("lo que pase primero: la respuesta o un timeout de 3s").
- **`Promise.any([...])`**: la primera que **resuelve con éxito** (ignora las que fallan hasta que una funcione).

**Cuándo NO paralelizar:** si una operación **depende** del resultado de la anterior (necesitás el `usuario` para pedir *sus* proyectos), tiene que ser secuencial. Y cuidado con disparar miles de Promises a la vez contra la base: podés saturar el connection pool — ahí conviene paralelismo **acotado** (por lotes).

**Ejercicios 6**
6.1 Tenés tres llamadas a APIs externas independientes que tardan 200ms cada una. ¿Cuánto tarda aproximadamente si las hacés con `await` secuencial vs. con `Promise.all`?
6.2 Vas a enviar un email a 500 usuarios y querés que, si uno falla, el resto igual se envíe y al final saber cuáles fallaron. ¿`Promise.all` o `Promise.allSettled`? ¿Por qué?
6.3 ¿En qué caso NO podés reemplazar varios `await` secuenciales por un `Promise.all`?

---

## Módulo 7 — No bloquees el event loop (trabajo CPU-bound)

**Teoría.** Node es un solo hilo para tu JavaScript. Si una operación **CPU-bound** (un cálculo pesado, sincrónico) corre en el hilo principal, **bloquea el event loop** durante todo ese tiempo: ningún otro request se atiende, los timers no disparan, la app parece colgada. Esto es lo opuesto a la I/O, que Node maneja sin bloquear.

```ts
// MAL: este cálculo pesado bloquea el hilo; todos los demás requests esperan
app.get("/reporte", () => {
  const resultado = calcularPrimosHasta(100_000_000); // 4 segundos de CPU pura
  return resultado; // durante esos 4s, la app no responde a NADIE
});
```

Síntoma típico en producción: la latencia de **todos** los endpoints se dispara cuando uno solo hace trabajo pesado. El "event loop lag" es una métrica que conviene monitorear (conecta con observabilidad del senior).

Soluciones, según el caso:

- **Partir el trabajo** en pedazos y ceder el control entre ellos (con `setImmediate`), para que el loop atienda otras cosas en el medio.
- **Worker threads** (`worker_threads`): hilos reales para trabajo CPU-bound, separados del hilo principal. Tu event loop queda libre mientras el worker calcula.
- **Sacarlo del request**: mandarlo a una **cola de trabajos** (BullMQ + Redis, del roadmap) y procesarlo en background. El request responde "encolado" al instante y el resultado se entrega después. Suele ser la mejor opción para tareas largas (generar PDFs, procesar videos).

La regla: **Node es para I/O, no para crunchear números en el hilo principal.** Si tenés CPU-bound pesado, sacalo del event loop.

**Ejercicios 7**
7.1 ¿Por qué un cálculo CPU-bound de 4 segundos en un handler afecta a **todos** los requests, y no solo al que lo disparó?
7.2 Nombrá dos formas de evitar bloquear el event loop con una tarea CPU-bound pesada.
7.3 Para "generar un PDF que tarda 30 segundos al pedirlo un usuario", ¿lo harías en el handler del request o lo mandarías a una cola? Justificá.

---

## Módulo 8 — Streams y backpressure

**Teoría.** Un **stream** procesa datos **en pedazos** (chunks) a medida que llegan, en vez de cargar todo en memoria de una vez. Es esencial cuando los datos son grandes o continuos: archivos enormes, respuestas HTTP, exportar millones de filas.

```ts
// MAL: carga el archivo ENTERO en memoria; un archivo de 2GB te tumba el proceso
const todo = await fs.promises.readFile("enorme.csv");
res.send(todo);

// BIEN: lo procesa en chunks con memoria constante, y pipeline propaga errores y cierra bien
import { pipeline } from "node:stream/promises";
await pipeline(fs.createReadStream("enorme.csv"), res);
```

Los tipos: **Readable** (fuente, ej. leer un archivo), **Writable** (destino, ej. una respuesta HTTP), **Transform** (transforma en el medio, ej. comprimir). Se conectan con `pipeline()` (recomendado: maneja errores y cierra los streams) o el más viejo `.pipe()` (respeta backpressure pero **no** propaga errores).

El concepto que distingue a quien entiende streams: **backpressure** (contrapresión). Si la fuente produce datos **más rápido** de lo que el destino los consume (leés un archivo veloz pero lo escribís a una red lenta), los chunks se acumularían en memoria hasta reventar. Los streams manejan esto automáticamente al usar `pipeline()`/`.pipe()`: la fuente **pausa** cuando el destino está saturado y **reanuda** cuando se libera. Por eso conectar streams es mejor que leer todo y escribir a mano: respeta el backpressure por vos (y con `pipeline()`, además, propaga errores y cierra los streams correctamente).

Conexión con el resto: cuando un ORM "streamea" un resultado grande, o cuando subís/bajás archivos en tu API, estás usando streams. Entenderlos evita el clásico "se me cae el proceso por OOM (out of memory)" al manejar archivos grandes.

**Ejercicios 8**
8.1 ¿Cuál es la ventaja de un stream sobre leer un archivo completo con `readFile` cuando el archivo es de varios GB?
8.2 ¿Qué es el "backpressure" y qué problema evita? ¿Quién lo maneja cuando usás `.pipe()`?
8.3 ¿Qué tipo de stream usarías para comprimir datos mientras pasan de un archivo a una respuesta HTTP: Readable, Writable o Transform?

---

## Módulo 9 — EventEmitter: el patrón base de Node

**Teoría.** Buena parte de Node está construida sobre el patrón **observer**, materializado en `EventEmitter`: un objeto **emite** eventos con nombre y otros se **suscriben** con callbacks. Los streams, los sockets, el servidor HTTP — todos son `EventEmitter` por debajo.

```ts
import { EventEmitter } from "node:events";

class Pedidos extends EventEmitter {
  confirmar(id: number): void {
    // ...lógica...
    this.emit("confirmado", id); // notifica a quien esté escuchando
  }
}

const pedidos = new Pedidos();
pedidos.on("confirmado", (id: number) => console.log(`Pedido ${id} confirmado`));
pedidos.confirmar(42); // dispara el listener → "Pedido 42 confirmado"
```

Es **desacople**: quien emite no sabe quién escucha ni cuántos. Esto conecta directo con los **eventos de dominio** del archivo senior (`PedidoConfirmado`) y con el sistema de eventos de NestJS (`@nestjs/event-emitter`), que es justamente un `EventEmitter` integrado a la DI.

Detalles a conocer: los listeners de tus propios emisores corren de forma **síncrona** en el orden en que se registraron (aunque varios emisores nativos —como el evento `'connection'` de un server— disparan en un tick posterior); un error tirado dentro de un listener de `error` sin manejar puede tumbar el proceso (por eso siempre se escucha el evento `error` en streams y sockets); y hay un límite por defecto de listeners (avisa con un warning de posible "memory leak" si registrás demasiados sin querer).

**Ejercicios 9**
9.1 ¿Qué patrón de diseño implementa `EventEmitter` y qué desacopla?
9.2 ¿Qué relación tiene `EventEmitter` con los "eventos de dominio" que viste en el archivo senior?
9.3 ¿Por qué es importante siempre suscribirse al evento `"error"` de un stream o socket?

---

## Módulo 10 — Módulos: CommonJS vs. ESM

**Teoría.** Node tiene **dos** sistemas de módulos y vas a cruzarte con ambos:

- **CommonJS (CJS)**: el histórico. `require(...)` para importar, `module.exports` para exportar. Es **síncrono** y dinámico. Sigue siendo enorme en el código existente.
- **ES Modules (ESM)**: el estándar moderno de JavaScript. `import`/`export`. Es el que usás en el frontend (y el que NestJS/TypeScript compilan). Es estático (permite tree-shaking) y soporta `await` a nivel de módulo.

```ts
// CommonJS
const express = require("express");
module.exports = { miFuncion };

// ESM
import express from "express";
export { miFuncion };
```

Cómo Node decide cuál usar: por la extensión (`.mjs` = ESM, `.cjs` = CJS) o por `"type": "module"` en el `package.json` (entonces los `.js` son ESM). En proyectos TypeScript escribís `import`/`export` siempre, y el `tsconfig` (`module`) decide a qué se compila.

Lo que conviene saber para no sufrir: **mezclar los dos da fricción**. Históricamente un paquete "ESM-only" no se podía `require()` desde CJS; y `import` de un módulo CJS a veces necesita el default import. Matiz 2026: Node 20.19+ / 22+ ya permite **`require()` de módulos ESM síncronos** (sin top-level await), así que la interoperabilidad mejoró bastante respecto de años anteriores. La tendencia sigue siendo claramente hacia **ESM**, pero como hay años de librerías en CJS, entender la diferencia te ahorra errores tipo `ERR_REQUIRE_ESM` o `Cannot use import statement outside a module`.

**Ejercicios 10**
10.1 ¿Cuál es la diferencia de sintaxis básica para importar entre CommonJS y ESM?
10.2 ¿Cómo le indicás a Node que los archivos `.js` de tu proyecto son ESM?
10.3 Si ves el error `Cannot use import statement outside a module`, ¿qué está pasando probablemente?

---

## Módulo 11 — `process`, entorno y ciclo de vida (graceful shutdown)

**Teoría.** El objeto global `process` es tu interfaz con el entorno donde corre Node:

- **`process.env`**: las variables de entorno (tu `DATABASE_URL`, `JWT_SECRET`, `PORT`). Es de donde el `ConfigModule` de Nest las lee. Nunca hardcodees secretos: van acá (módulo de config).
- **`process.argv`**: los argumentos de línea de comandos (útil en scripts/CLIs).
- **`process.exit(code)`**: termina el proceso (`0` = OK, distinto de `0` = error). Usalo con cuidado: corta de golpe.
- **Señales**: el SO le manda señales al proceso. `SIGTERM` (apagado ordenado, lo que manda Kubernetes/Docker al hacer deploy) y `SIGINT` (Ctrl+C).

El concepto Senior acá es el **graceful shutdown** (apagado elegante), que ya apareció en el archivo senior. Cuando el orquestador manda `SIGTERM`, no querés cortar de golpe dejando requests a medias y conexiones colgadas. Querés: **dejar de aceptar requests nuevos**, **terminar los que están en curso**, **cerrar conexiones** (base, Redis, colas) y recién entonces salir.

```ts
process.on("SIGTERM", async () => {
  console.log("Apagando ordenadamente...");
  await server.close();        // deja de aceptar requests nuevos y espera los en curso
  await db.end();              // cierra el pool de conexiones
  await redis.quit();
  process.exit(0);
});
```

En NestJS esto se integra con los **lifecycle hooks** (`enableShutdownHooks()`, `OnApplicationShutdown`) que viste en el senior: Nest dispara el cierre ordenado de cada módulo. Sin graceful shutdown, cada deploy corta transacciones a la mitad y devuelve errores a usuarios que estaban a mitad de un request.

**Ejercicios 11**
11.1 ¿De dónde lee NestJS (vía ConfigModule) variables como `DATABASE_URL` o `JWT_SECRET`?
11.2 ¿Qué señal manda Kubernetes/Docker al apagar tu contenedor, y qué deberías hacer al recibirla en vez de cortar de golpe?
11.3 ¿Qué problemas concretos evita el graceful shutdown durante un deploy?

---

## Módulo 12 — Cómo escala Node: cluster, worker threads y horizontal

**Teoría.** "Si Node es un solo hilo, ¿no desperdicia los demás núcleos de la CPU?" Sí, un proceso Node usa un núcleo para tu JS. Para aprovechar más, hay tres caminos (que responden a problemas distintos):

- **`cluster` / múltiples procesos**: lanzás **varios procesos** Node (idealmente uno por núcleo), todos escuchando el mismo puerto; el SO reparte las conexiones. Cada proceso tiene su propio event loop. Sirve para **aprovechar varios núcleos ejecutando tu JavaScript en paralelo**: ayuda cuando el cuello de botella es el CPU/JS del hilo principal (parsing, serialización, validación, render), no la espera de I/O — un solo proceso ya maneja muchísima I/O concurrente. El resultado es más throughput de requests por máquina. (En la práctica moderna esto lo hace un gestor como PM2 o, más común, corriendo varias réplicas del contenedor.)
- **`worker_threads`**: hilos dentro de **un** proceso, pensados para trabajo **CPU-bound** (módulo 7), no para escalar I/O. Sacás el cálculo pesado del hilo principal.
- **Escalado horizontal**: correr **muchas instancias** de tu app (en contenedores) detrás de un **load balancer**. Es el modelo dominante en la nube y el que menciona el roadmap. La capacidad crece agregando réplicas.

El requisito que hace posible el escalado horizontal —y la pregunta que sigue en entrevistas— es que tu app sea **stateless**: que **no** guarde estado en la memoria del proceso (sesiones, contadores, caché local), porque cada request puede caer en una instancia distinta. El estado compartido va afuera: la **base de datos** (Postgres) y **Redis** (sesiones, caché, colas — el módulo del roadmap). Esto cierra el círculo: tu auth con refresh tokens en Redis, tu caché cache-aside, tus colas BullMQ — todo eso existe, en parte, para que tu backend pueda escalar a N instancias sin que importe en cuál caés.

> Respuesta que impresiona en entrevista: "Node escala horizontalmente con instancias stateless detrás de un balanceador; el estado vive en Postgres y Redis. `cluster` aprovecha varios núcleos en una máquina, y `worker_threads` es para el trabajo CPU-bound, no para escalar I/O."

**Ejercicios 12**
12.1 ¿Cuál es la diferencia de propósito entre `cluster` (múltiples procesos) y `worker_threads`?
12.2 ¿Qué significa que una app sea "stateless" y por qué es requisito para el escalado horizontal?
12.3 Si tu app guarda las sesiones de los usuarios en una variable en memoria y la escalás a 3 instancias detrás de un balanceador, ¿qué problema aparece? ¿Cómo lo resolvés?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Porque casi nunca espera bloqueado: delega la I/O al sistema operativo (libuv) y
    sigue ejecutando otro código; mientras un request espera la base, el hilo atiende
    a otros. La concurrencia viene de no esperar, no de varios hilos.
1.2 (a) I/O-bound  (b) CPU-bound  (c) I/O-bound  (d) CPU-bound.
1.3 readFileSync bloquea el único hilo hasta terminar de leer: durante ese tiempo
    ningún otro request se atiende. readFile (promesa) delega la lectura y libera el
    hilo para atender otras cosas mientras tanto.
```

### Módulo 2
```
2.1 setTimeout se ejecuta en la fase de "timers"; la mayoría de los callbacks de I/O,
    en la fase "poll".
2.2 Quedan en espera: el event loop está ocupado ejecutando ese for (un solo hilo), así
    que no puede atender ningún otro request, timer ni I/O hasta que el cálculo termine.
    La latencia de todos se dispara.
2.3 Falso. Node ejecuta un callback de JS a la vez (el hilo principal es único). Lo que
    ocurre "en paralelo" es la I/O delegada al SO, no la ejecución de tu JavaScript.
```

### Módulo 3
```
3.1 Salida:
    A
    D
    C
    B
    (Síncronos A y D primero; luego la microtask C de la promise; por último la
     macrotask B del setTimeout.)
3.2 Porque las microtasks (Promises) se vacían COMPLETAS al terminar el código
    síncrono actual, ANTES de pasar a la fase de timers donde corre el setTimeout. El
    "0 ms" no lo adelanta: igual espera a la próxima fase de timers.
3.3 Como nextTick tiene prioridad máxima y se vacía entero antes de seguir, una
    recursión de nextTick nunca le cede el turno a la fase de I/O/timers: mata de
    hambre al event loop (la app deja de atender I/O). Para diferir conviene setImmediate.
```

### Módulo 4
```
4.1 No, no es más rápido. async/await es azúcar sintáctico sobre Promises: el mismo
    mecanismo, escrito de forma que se lee como código síncrono. "async" solo significa
    que la función devuelve una Promise.
4.2 Corren en SECUENCIA: el segundo await no arranca hasta que termina el primero.
    Está MAL si son independientes (desperdiciás tiempo); debería ser Promise.all.
4.3 Devuelve Promise<number> (toda función async envuelve su retorno en una Promise).
    En TS: `async function f(): Promise<number> { return 42; }`.
```

### Módulo 5
```
5.1 Porque falta el await: guardar() devuelve una Promise que "flota". La función ya
    salió del try antes de que la Promise se rechace, así que el catch no la atrapa; el
    rechazo queda como unhandled rejection.
5.2 Porque forEach ignora el valor de retorno del callback async: dispara todas las
    Promises pero no las espera, así que el código sigue sin que terminen. Usás
    `for (const id of ids) { await borrar(id); }` (secuencial) o
    `await Promise.all(ids.map((id) => borrar(id)))` (en paralelo).
5.3 Es una Promise que se rechazó y nadie manejó con catch/try-catch. Conviene un
    handler global de unhandledRejection para loguearla y apagar ordenadamente: una
    rejection no manejada tumba el proceso por defecto desde Node 15+ (es el
    comportamiento estándar en Node 18/20/22).
```

### Módulo 6
```
6.1 Secuencial: ~600ms (200+200+200). Con Promise.all: ~200ms (las tres arrancan
    juntas y esperás a la más lenta).
6.2 Promise.allSettled: espera a todas sin cortar ante un fallo y te da el estado de
    cada una, así sabés cuáles fallaron y el resto igual se envió. Promise.all cortaría
    en el primer rechazo y no enviarías a los demás.
6.3 Cuando hay dependencia de datos: una operación necesita el resultado de la anterior
    (ej. traer el usuario y después traer SUS proyectos con el id obtenido). Ahí debe
    ser secuencial.
```

### Módulo 7
```
7.1 Porque el cálculo corre en el único hilo principal y bloquea el event loop durante
    esos 4s: mientras tanto Node no puede ejecutar ningún otro callback, request ni
    timer. Todos quedan esperando, no solo el que lo disparó.
7.2 (1) worker_threads: mover el cálculo a un hilo aparte. (2) Mandarlo a una cola de
    trabajos (BullMQ + Redis) y procesarlo en background. (También: partir el trabajo y
    ceder con setImmediate entre pedazos.)
7.3 A una cola. 30s en el handler bloquearía/colgaría el request y, si es CPU-bound,
    el event loop. Encolás el trabajo, respondés al instante ("en proceso") y entregás
    el resultado cuando esté (notificación, polling o webhook).
```

### Módulo 8
```
8.1 El stream procesa el archivo en chunks con memoria casi constante, sin importar el
    tamaño; readFile carga los GB enteros en RAM y puede tumbar el proceso por falta de
    memoria (OOM).
8.2 Backpressure es el mecanismo para cuando la fuente produce más rápido de lo que el
    destino consume: en vez de acumular chunks en memoria, la fuente se pausa hasta que
    el destino se libera. Con .pipe()/pipeline() lo maneja Node automáticamente.
8.3 Un Transform stream: recibe datos, los transforma (comprime) y los emite, en el
    medio entre el Readable (archivo) y el Writable (respuesta).
```

### Módulo 9
```
9.1 El patrón Observer (publicador/suscriptor): un objeto emite eventos y otros se
    suscriben. Desacopla a quien emite de quien escucha: el emisor no sabe quién ni
    cuántos están escuchando.
9.2 Los eventos de dominio (PedidoConfirmado, etc.) son el mismo patrón a nivel de
    negocio: el agregado emite un evento y otras partes reaccionan sin que él las
    conozca. NestJS lo ofrece con @nestjs/event-emitter, que es un EventEmitter
    integrado a la DI.
9.3 Porque un evento "error" emitido sin ningún listener que lo escuche hace que Node
    lo trate como una excepción no manejada y tumbe el proceso. Suscribirse permite
    manejarlo (loguear, cerrar el stream) en vez de morir.
```

### Módulo 10
```
10.1 CommonJS: const x = require("paquete") y module.exports = ... .
     ESM: import x from "paquete" y export ... .
10.2 Poniendo "type": "module" en el package.json (entonces los .js se tratan como
     ESM), o usando la extensión .mjs.
10.3 Estás usando sintaxis import (ESM) en un contexto que Node interpreta como
     CommonJS: falta "type": "module" en el package.json (o usar .mjs / compilar el TS
     al target de módulos correcto).
```

### Módulo 11
```
11.1 De process.env (las variables de entorno del proceso). El ConfigModule las lee de
     ahí (y de archivos .env que carga en process.env al arrancar).
11.2 SIGTERM. En vez de cortar de golpe: dejar de aceptar requests nuevos, terminar los
     en curso, cerrar conexiones (DB, Redis, colas) y recién entonces process.exit(0)
     — graceful shutdown.
11.3 Evita cortar requests a la mitad (errores a usuarios), dejar transacciones
     incompletas y conexiones colgadas. Permite un deploy sin downtime perceptible:
     la instancia vieja termina lo que tenía antes de morir.
```

### Módulo 12
```
12.1 cluster lanza varios PROCESOS Node (cada uno con su event loop) para aprovechar
     varios núcleos ejecutando JavaScript en paralelo (más throughput cuando el límite
     es el CPU/JS del hilo principal, no la I/O, que un solo proceso ya maneja de sobra).
     worker_threads son HILOS dentro de un proceso, pensados para sacar trabajo
     CPU-bound del hilo principal.
12.2 Stateless = el proceso no guarda en su memoria estado que deba sobrevivir entre
     requests (sesiones, caché local). Es requisito del escalado horizontal porque cada
     request puede caer en una instancia distinta: si el estado viviera en una sola, las
     demás no lo tendrían. El estado va afuera: Postgres, Redis.
12.3 Las sesiones quedarían en la memoria de UNA instancia; si un request del mismo
     usuario cae en otra de las 3, no encuentra su sesión (se "desloguea" al azar). Se
     resuelve sacando las sesiones a un store compartido (Redis), volviendo la app
     stateless.
```

---

## Siguientes pasos

Con este módulo cerrás los fundamentos del runtime que el resto del temario daba por sabidos: ya entendés *por qué* tus `await` no bloquean, *por qué* no debés crunchear números en el hilo principal, y *cómo* tu backend escala a varias instancias. El recorrido natural desde acá conecta los puntos: **(1)** la **cola de trabajos** (BullMQ + Redis) del roadmap, que es la respuesta a "no bloquees el event loop" y a "stateless"; **(2)** **Docker + deploy**, donde el graceful shutdown y el escalado horizontal dejan de ser teoría; **(3)** volvé a la sección de **resiliencia y observabilidad** del archivo senior — los timeouts, el circuit breaker y el "event loop lag" ahora tienen una base concreta. Si vas a entrevistas, practicá explicar el event loop y el orden macrotask/microtask en voz alta: es la pregunta de Node que más se repite.
