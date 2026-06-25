# Express: el framework minimalista de Node (y la base de NestJS)

**Routing, middleware y APIs REST con Express 5 · ejemplos en TypeScript · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es el **eslabón que faltaba** en la sección Backend: entre [Node por dentro](nodejs.md) (el runtime) y [NestJS](nestjs.md) (el framework opinado) está **Express**, el framework minimalista sobre el que se construyó medio Node del mundo. Y acá está el dato que cambia todo: **NestJS corre SOBRE Express por defecto.** Entender Express no es opcional aunque uses Nest — es entender el piso sobre el que Nest construye.

**Lo que asumimos.** Node básico (`npm`, módulos, `async/await`), HTTP (qué es un request, un método GET/POST, un status code), y TypeScript. Si no tenés claro el ciclo de vida de un request HTTP o el event loop de Node, repasá [Node por dentro](nodejs.md) primero: Express es una capa finita arriba del módulo `http` de Node, y se entiende mejor sabiendo qué hay debajo.

> ⚠️ **Nota sobre versiones.** Enseñamos **Express 5**, el default de npm desde 2025. Si te cruzás con tutoriales de Express 4 —y hay muchos—, ojo: varios ejemplos ya no aplican (cambiaron el manejo de errores async y la sintaxis de routing, lo vemos en su momento). Lo marcado con ⚠️ verificalo en `expressjs.com`. Las versiones exactas las dejamos para el módulo 7, que es donde las vas a tipear.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere editor).

**Índice de módulos**
1. Qué es Express y por qué sigue importando
2. El primer servidor: `app`, `req`, `res` y el ciclo request-response
3. Routing: rutas, params, query y routers modulares
4. El middleware chain: el corazón de Express
5. Manejo de errores (y el gran cambio de Express 5)
6. Construir una API REST de verdad
7. Express con TypeScript
8. Express vs NestJS: cuándo cada uno

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es Express y por qué sigue importando

**Teoría.** Express es un **framework web minimalista** para Node. La palabra clave es **minimalista**: no te impone estructura, ni carpetas, ni una forma "correcta" de hacer las cosas. Te da lo esencial —routing y middleware— y el resto lo armás vos. Esa libertad es su mayor virtud y su mayor trampa (módulo 8).

La definición mental, de la propia doc: **una aplicación Express es, en el fondo, una serie de llamadas a funciones middleware.** Guardá esa frase — es literalmente cómo funciona Express por dentro, y la vamos a desarmar en el módulo 4.

¿Por qué aprender Express en 2026 si existe NestJS, que es más estructurado? Tres razones:

1. **NestJS corre sobre Express.** Por defecto, Nest usa Express como su plataforma HTTP subyacente (`@nestjs/platform-express`). Todo lo de Nest corre **sobre el motor HTTP de Express**: el request entra y sale por Express, y el *middleware* de Nest **es** middleware de Express. (Los guards, interceptors y pipes son abstracciones propias del pipeline de Nest, pero montadas sobre ese motor.) Si no entendés Express, ese piso es una caja negra.
2. **Es el estándar de facto.** Millones de APIs, tutoriales, librerías y ofertas de laburo lo nombran. Las ofertas que dicen "Node.js (Express, NestJS)" lo piden **explícitamente**.
3. **Te enseña los fundamentos sin magia.** Express no esconde el request/response detrás de decoradores. Ves el flujo crudo, y eso es oro para entender qué hace cualquier framework.

La analogía: Express es como una **cocina vacía pero bien equipada** — tenés la heladera, la hornalla y los cuchillos, pero la receta y el orden los ponés vos. NestJS es la misma cocina con un chef que ya te organizó el mise en place y te dice dónde va cada cosa. Para cocinar en la de Nest, ayuda saber usar la cocina pelada primero.

La frase mental: **Express es minimalista: routing + middleware y nada más impuesto. "Una app Express es una serie de llamadas a middleware." Y aunque uses NestJS, lo estás usando — Nest corre sobre Express.**

**Ejercicios 1**
1.1 🔁 ¿Qué significa que Express es "minimalista"? ¿Cuál es la frase que define cómo funciona por dentro?
1.2 🧠 ¿Por qué tiene sentido aprender Express aunque pienses trabajar con NestJS? Dá la razón más fuerte.
1.3 🧠 La libertad de Express ("no te impone estructura") es su virtud y su trampa. ¿Por qué podría ser una trampa en un proyecto grande con varios devs?

---

## Módulo 2 — El primer servidor: `app`, `req`, `res` y el ciclo request-response

**Teoría.** El corazón operativo de Express son tres cosas: el objeto **`app`** (tu aplicación), y en cada pedido, los objetos **`req`** (request: lo que entra) y **`res`** (response: lo que sale). El ciclo es siempre el mismo: **entra un request → tu código lo procesa → mandás una response → se cierra el ciclo.**

El servidor mínimo:

```ts
import express, { Request, Response } from 'express'

const app = express()

app.use(express.json())   // middleware built-in: parsea el body JSON y lo deja en req.body

app.get('/', (req: Request, res: Response) => {
  res.json({ ok: true, mensaje: 'Hola Express' })
})

app.listen(3000, () => {
  console.log('Servidor en http://localhost:3000')
})
```

Tres cosas para anclar de entrada:

- **`express.json()` no es decorativo.** Es un middleware built-in que parsea el body JSON **cuando hay uno** y lo deja en `req.body`. Sin este middleware, `req.body` es siempre `undefined` — olvidarlo es el error #1 de los principiantes: "¿por qué `req.body` está vacío?". (Ojo: aun con `express.json()`, un request sin cuerpo —un GET, o un POST vacío— deja `req.body` en `undefined`; por eso validás siempre, módulo 6.)
- **`req`** te da todo lo que entró: `req.params` (segmentos de la URL), `req.query` (el `?orden=asc`), `req.body` (el cuerpo), `req.headers`, `req.method`.
- **`res`** termina el ciclo: `res.json(obj)`, `res.send(texto)`, `res.status(201).json(...)`. **Cada request necesita exactamente una response** — si no respondés, el cliente queda colgado; si respondés dos veces, error.

> **Si venís de Express 4** (y tropezás con tutoriales viejos): varias cosas cambiaron en v5. `req.body` antes era `{}` por defecto (ahora `undefined`); `req.query` ahora es **solo lectura** (un getter, no lo podés reasignar); las firmas `res.json(obj, status)` y `res.send(body, status)` **se eliminaron** → ahora se encadena `res.status(201).json(obj)`; y `res.send(404)` (status numérico) es ahora `res.sendStatus(404)`. Si seguís un ejemplo viejo y algo no compila o no anda, suele ser por esto.

La frase mental: **`app` es tu aplicación; en cada pedido, `req` es lo que entra y `res` lo que sale. Un request = una response, sí o sí. Y acordate de `express.json()`, o `req.body` viene vacío.**

**Ejercicios 2**
2.1 🔁 ¿Qué hace `express.json()` y qué pasa en Express 5 si te lo olvidás?
2.2 🔁 ¿Dónde encontrás los segmentos de URL, el query string y el cuerpo del request? (nombrá la propiedad de `req` de cada uno)
2.3 ✍️ Escribí un servidor Express mínimo en TypeScript que responda `{ status: 'ok' }` en `GET /health` y escuche en el puerto 3000.

---

## Módulo 3 — Routing: rutas, params, query y routers modulares

**Teoría.** El **routing** decide qué código corre según el **método HTTP** + el **path** del request. La forma básica: `app.METODO(path, handler)`.

```ts
app.get('/usuarios', (req, res) => { /* listar */ })
app.post('/usuarios', (req, res) => { /* crear */ })
app.get('/usuarios/:id', (req, res) => {
  res.json({ id: req.params.id })       // :id es un "route param", queda en req.params.id
})
```

- **Route params** (`:id`): segmentos nombrados de la URL. `/usuarios/42` → `req.params.id === '42'` (siempre string).
- **Query string** (`?orden=asc&page=2`): va en `req.query` (`req.query.orden`). No es parte del path.
- **`app.route()`** encadena varios métodos en un mismo path sin repetirlo:
  ```ts
  app.route('/libros').get(listar).post(crear)
  ```

⚠️ **Cambios de routing en Express 5** (subió a `path-to-regexp` v8, más estricto — si venís de tutoriales v4, esto te va a morder):
- El **wildcard `*` ahora necesita nombre**: `/*` ya no va, usás `/*splat` (los capturados quedan en `req.params.splat` como **array**).
- El **opcional `?` se eliminó**; ahora usás llaves: `/:archivo{.:ext}` en vez de `/:archivo.:ext?`.
- **Regex en strings de ruta** restringido: para varios paths, usá un array → `app.get(['/discusion/:slug', '/pagina/:slug'], handler)`.

**El router modular** (`express.Router()`) es lo que hace que tu app no sea un solo archivo de 2000 líneas. Un `Router` es una **"mini-app"**: un sistema completo de routing y middleware que después **montás** en un path. Pensalo como un **tablero eléctrico**: en vez de cablear cada llave en el tablero principal, armás sub-tableros por ambiente (uno de `usuarios`, otro de `pedidos`) y los **enchufás** al principal bajo una dirección. El `app.use('/api/usuarios', router)` es ese enchufe — todo lo que cuelga del router queda accesible bajo `/api/usuarios`:

```ts
// rutas/usuarios.ts
import { Router } from 'express'
const router = Router()

router.get('/', listarUsuarios)        // → GET /api/usuarios
router.post('/', crearUsuario)         // → POST /api/usuarios
router.get('/:id', obtenerUsuario)     // → GET /api/usuarios/:id

export default router

// app.ts
import usuariosRouter from './rutas/usuarios'
app.use('/api/usuarios', usuariosRouter)   // monta el router bajo /api/usuarios
```

Fijate el patrón: **un router por recurso** (`usuarios`, `pedidos`, `productos`), cada uno en su archivo, montado con un prefijo. Eso es lo que mantiene una API Express organizada — y conecta con la responsabilidad única de [SOLID](solid.md).

La frase mental: **el routing mapea método + path a un handler. Los `:params` van en `req.params`, el `?query` en `req.query`. Y para no tener un archivo gigante: un `Router()` por recurso, montado con su prefijo.**

**Ejercicios 3**
3.1 🔁 ¿Cuál es la diferencia entre un route param (`:id`) y un query param (`?orden=asc`)? ¿Dónde queda cada uno?
3.2 🧠 ¿Qué problema resuelve `express.Router()` y cómo se monta en la app?
3.3 ✍️ Armá un `Router()` para el recurso `productos` con `GET /` (listar), `POST /` (crear) y `GET /:id` (uno), y montalo en la app bajo `/api/productos`.

---

## Módulo 4 — El middleware chain: el corazón de Express

**Teoría.** Acá está **lo que de verdad es Express.** ¿Te acordás de "una app Express es una serie de llamadas a middleware"? Esto es eso. Un **middleware** es una función con la firma `(req, res, next)` que se ejecuta **en orden**, formando una **cadena** por la que pasa cada request.

```ts
import { Request, Response, NextFunction } from 'express'

function logger(req: Request, res: Response, next: NextFunction): void {
  console.log(`${req.method} ${req.url}`)
  next()   // ⚠️ pasa el control al SIGUIENTE middleware. Sin esto, la request queda COLGADA
}

app.use(logger)   // se aplica a todos los requests
```

Cada middleware puede hacer cuatro cosas: **ejecutar código**, **modificar `req`/`res`** (ej. agregar `req.usuario` tras validar un token), **terminar el ciclo** (mandar una response), o **llamar `next()`** para ceder el control al siguiente. Si no termina el ciclo **y** no llama `next()`, **la request queda colgada para siempre** — el gotcha clásico.

La analogía: el request es una valija que pasa por **varios puestos de control en una aduana**. Cada puesto (middleware) puede revisarla, ponerle un sello (`req.usuario = ...`), dejarla pasar al siguiente (`next()`), o frenarla y devolverla (mandar la response). El orden de los puestos importa: si el de "verificar pasaporte" está después del de "entregar equipaje", no sirve de nada.

**El orden es todo:** los middleware se ejecutan en el orden en que los cargás. Por eso `app.use(express.json())` va **arriba** (antes de las rutas que leen `req.body`), y un middleware de autenticación va **antes** de las rutas protegidas.

Los **cinco tipos** de middleware:
- **Application-level:** `app.use(...)` / `app.get(...)` — sobre la app entera.
- **Router-level:** `router.use(...)` — sobre un router.
- **Error-handling:** 4 argumentos (módulo 5).
- **Built-in:** `express.json()`, `express.urlencoded()`, `express.static()`.
- **De terceros:** `cors`, `helmet`, `morgan`, etc.

⚠️ Detalle fino: `app.use(mw)` corre para **todos los métodos** y matchea por **prefijo** de path; `app.get(path, mw)` matchea solo GET y el path exacto. Confundirlos es un clásico.

La frase mental: **un middleware es `(req, res, next)` y la app es una cadena de ellos, en orden. Cada uno revisa, sella, frena o pasa con `next()`. Si no respondés ni llamás `next()`, la request muere colgada. Y el orden de carga es el orden de ejecución.**

**Ejercicios 4**
4.1 🔁 ¿Cuáles son las cuatro cosas que puede hacer un middleware? ¿Qué pasa si no termina el ciclo ni llama `next()`?
4.2 🧠 ¿Por qué `app.use(express.json())` tiene que ir ANTES de las rutas que leen `req.body`? Relacionalo con "el orden es el orden de ejecución".
4.3 🧠 ¿Qué diferencia hay entre `app.use(mw)` y `app.get('/x', mw)` en cuanto a método y matcheo de path?
4.4 ✍️ Escribí un middleware `requiereApiKey` que lea el header `x-api-key`, y si falta o es incorrecto responda `401`, y si está bien llame `next()`. Aplicalo a toda la app.
4.5 ✍️ **El bug a propósito.** Montá una ruta `POST /eco` que devuelva `req.body`, pero poné `app.use(express.json())` **después** de la ruta. Probala con un POST con body JSON y observá qué llega en `req.body`. Después movés `express.json()` arriba y volvés a probar. Explicá en un comentario qué cambió y por qué.

---

## Módulo 5 — Manejo de errores (y el gran cambio de Express 5)

**Teoría.** El manejo de errores en Express tiene una pieza especial: el **error-handling middleware**, que se distingue por tener **CUATRO argumentos** en vez de tres: `(err, req, res, next)`.

```ts
// Va SIEMPRE al final, después de todas las rutas y middlewares
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(err.stack)
  res.status(500).json({ error: 'Algo salió mal' })
})
```

Dos cosas críticas, las dos caen en entrevista:
- **¿Por qué 4 argumentos?** Porque Express identifica el error handler por su **aridad** (la cantidad de parámetros). Una función con 3 args es un middleware normal; con 4, Express sabe que es un manejador de errores. Si te olvidás el cuarto, **Express no lo reconoce como error handler** y nunca captura los errores. (Sí, aunque no uses `next`, tenés que declararlo.)
- **¿Dónde va?** **Último.** Después de todas las rutas y middlewares. Cualquier error que ocurra "arriba" cae acá.

La analogía: el error handler es la **oficina de reclamos al final del pasillo**. La reconocés porque tiene **una ventanilla más** que los mostradores comunes (4 argumentos en vez de 3), y está **al fondo de todo** — ahí derivás cualquier cosa que salió mal en cualquier mostrador anterior. Si la ponés en el medio del pasillo (no última), los mostradores que vienen después nunca le mandan sus reclamos.

¿Cómo le mandás un error a ese handler? Llamando `next(err)` con el error como argumento:

```ts
app.get('/usuarios/:id', (req, res, next) => {
  const usuario = buscarSync(req.params.id)
  if (!usuario) {
    next(new Error('Usuario no encontrado'))   // salta directo al error handler
    return
  }
  res.json(usuario)
})
```

> ⚠️ **EL gran cambio de Express 5 (y la razón #1 para no usar v4 en proyectos nuevos).** En Express 4, si un handler `async` lanzaba o una promesa rechazaba, Express **NO lo capturaba**: la request quedaba colgada o el proceso crasheaba, salvo que envolvieras todo en `try/catch` + `next(err)` o usaras el paquete `express-async-errors`. Era el dolor más común del framework.
>
> **En Express 5, eso se arregló:** los handlers/middleware que **devuelven una promesa rechazada** (o que lanzan dentro de una `async function`) **reenvían el error automáticamente** al error handler, como si hubieras llamado `next(err)`. Mirá:
> ```ts
> // Express 5: si pedidoService.buscar() rechaza, el error va SOLO al error handler.
> // NO necesitás try/catch. (En v4 esto dejaba la request colgada.)
> app.get('/pedidos/:id', async (req, res) => {
>   const pedido = await pedidoService.buscar(req.params.id)
>   res.json(pedido)
> })
> ```
> **El límite que tenés que saber:** esto vale solo para **promesas / `async`**. Si tu error ocurre dentro del **callback de una API vieja estilo Node** (que no devuelve promesa), Express no puede verlo → ahí seguís necesitando `try/catch` + `next(err)` manual. La regla: usá `async/await` y dejá que v5 capture; los callbacks viejos, manejalos a mano.

Para que no te quede la duda de "¿entonces uso `next(err)` o no?", el resumen de un vistazo:

| Dónde ocurre el error | ¿Qué hacés en Express 5? |
|---|---|
| `throw` síncrono en un handler | Lo captura solo (o `next(err)` explícito si querés) |
| Promesa rechazada / `throw` en `async` | **Automático** — no hacés nada, v5 lo reenvía al error handler |
| Callback de API vieja (`fs.readFile`, etc.) | **A mano** — `try/catch` o chequear `err` y llamar `next(err)` |

La frase mental: **el error handler tiene 4 argumentos (Express lo reconoce por la aridad) y va último. Mandás errores con `next(err)`. Y en Express 5, los errores de `async`/promesas se reenvían solos — pero los de callbacks viejos, no: esos a mano.**

**Ejercicios 5**
5.1 🔁 ¿Cuántos argumentos tiene un error-handling middleware y por qué? ¿Dónde se coloca?
5.2 🧠 En Express 4, ¿qué pasaba si un handler `async` lanzaba una excepción sin `try/catch`? ¿Qué cambió en Express 5?
5.3 🧠 En Express 5, ¿en qué caso TODAVÍA necesitás `try/catch` + `next(err)` manual? Dá un ejemplo.
5.4 ✍️ Escribí un error handler global que: loggee el error, y responda con el `status` del error si lo tiene (`err.status`) o 500 si no, en formato JSON.

---

## Módulo 6 — Construir una API REST de verdad

**Teoría.** Juntemos todo en una API REST como las que vas a escribir en el laburo. Los ingredientes: routers por recurso (módulo 3), middleware (módulo 4), manejo de errores (módulo 5), **status codes correctos** y **validación** del input.

```ts
import { Router } from 'express'
import { z } from 'zod'   // Express NO valida solo: necesitás una librería

const router = Router()

// El "contrato" del input, validado en runtime
const CrearUsuario = z.object({
  email: z.string().email(),
  edad: z.number().int().positive(),
})

router.post('/', (req, res) => {
  const parsed = CrearUsuario.safeParse(req.body)
  if (!parsed.success) {
    res.status(400).json({ errores: parsed.error.issues })  // 400: input inválido
    return
  }
  const creado = { id: 1, ...parsed.data }
  res.status(201).json(creado)   // 201: creado (NO 200)
})

router.get('/:id', async (req, res) => {
  const usuario = await repo.buscar(req.params.id)
  if (!usuario) {
    res.status(404).json({ error: 'No encontrado' })   // 404: no existe
    return
  }
  res.json(usuario)   // 200 por defecto
})

export default router
```

Las reglas que separan una API prolija de una chapucera:
- **Status codes con sentido:** `200` OK, `201` creado, `400` input inválido, `401` no autenticado, `403` sin permiso, `404` no existe, `500` error del servidor. No devuelvas `200` para todo.
- **Express NO valida el input por vos.** El framework no trae validación de payloads — eso es decisión de diseño (es minimalista). Usás una librería: **`zod`**, `joi` o `express-validator`. Validar SIEMPRE lo que entra: nunca confíes en `req.body`.
- **Un 404 "catch-all" al final**, antes del error handler, para rutas que no existen:
  ```ts
  app.use((req, res) => res.status(404).json({ error: 'Ruta no encontrada' }))
  ```
- **El error handler de 4 args, último de todo** (módulo 5).

El orden del stack completo, de arriba a abajo: `express.json()` → middlewares globales (cors, helmet, logger) → routers de recursos → 404 catch-all → error handler.

La frase mental: **una API REST prolija = routers por recurso + status codes correctos + validación con una librería (Express no valida solo) + 404 catch-all + error handler al final. Nunca confíes en `req.body`: validalo.**

**Ejercicios 6**
6.1 🔁 ¿Express valida el body del request por vos? Si no, ¿qué usás y por qué es importante?
6.2 🧠 ¿Qué status code devolvés al crear un recurso con éxito, al recibir input inválido, y al pedir algo que no existe? ¿Por qué no usar `200` para todo?
6.3 🧠 ¿En qué orden van, de arriba a abajo, `express.json()`, los routers, el 404 catch-all y el error handler? ¿Por qué ese orden?
6.4 ✍️ Escribí un `POST /api/pedidos` que valide con zod un body `{ productoId: string, cantidad: number positivo }`, devuelva 400 con los errores si falla, y 201 con el pedido si pasa.

---

## Módulo 7 — Express con TypeScript

**Teoría.** En 2026 nadie escribe Express en JS pelado para algo serio. El setup tipado: instalás `express` (⚠️ a mediados de 2026 la versión vigente es **`5.2.1`**, y requiere **Node 18+**; verificá en npm antes de fijar la versión) y los tipos **`@types/express`** (⚠️ asegurate de que sea la línea **v5**, hoy `5.0.6` — no mezcles `@types/express@4` con `express@5`).

Los tres tipos centrales se importan de `'express'`: **`Request`**, **`Response`**, **`NextFunction`**.

```ts
import express, { Request, Response, NextFunction } from 'express'

const app = express()

// Handler tipado básico
app.get('/ping', (req: Request, res: Response) => {
  res.json({ pong: true })
})
```

Lo potente: **`Request` es genérico**, y podés tipar params, body y query. La firma es `Request<Params, ResBody, ReqBody, Query>`:

```ts
type Params = { id: string }
type Body = { email: string; edad: number }

app.put('/usuarios/:id', (req: Request<Params, unknown, Body>, res: Response) => {
  const id = req.params.id        // string ✅ (tipado)
  const email = req.body.email    // string ✅ (tipado)
  const edad = req.body.edad      // number ✅
  res.json({ id, email, edad })
})
```

Un par de gotchas:
- ⚠️ **No mezcles versiones de tipos.** `@types/express@5` con `express@5`. Mezclar mayores te da errores de tipo crípticos.
- **Preferí mandar la response y `return;` aparte.** `return res.status(400).json(...)` **compila bien** con `@types/express` v5 (el tipo de retorno del handler es `unknown`, no `void`). Aun así, la convención de mandar la response y poner `return;` en una línea aparte (como en el módulo 6) es más explícita y te cubre si en algún momento anotás el retorno como `: void` o usás reglas de lint como `@typescript-eslint/no-misused-promises`. Es estilo, no un error de compilación.
- **Tipar `req.body` no valida nada en runtime.** El tipo `Body` es una promesa del compilador, pero en runtime el body es lo que llegue. Por eso validás con zod (módulo 6) **además** de tipar. Tipo ≠ validación.

La frase mental: **`express` + `@types/express` v5 (no mezcles mayores). Importás `Request`, `Response`, `NextFunction`; `Request<Params, _, Body, Query>` te tipa todo. Pero tipar no valida en runtime — para eso, zod.**

**Ejercicios 7**
7.1 🔁 ¿De dónde importás `Request`, `Response` y `NextFunction`? ¿Qué paquete de tipos necesitás y qué versión con Express 5?
7.2 🧠 ¿Por qué tipar `req.body` con un type de TS no reemplaza a validar con zod? ¿Qué hace cada uno?
7.3 ✍️ Tipá un handler `GET /usuarios/:id` con un query opcional `?incluirInactivos=true`, de forma que `req.params.id` y `req.query` estén tipados.

---

## Módulo 8 — Express vs NestJS: cuándo cada uno

**Teoría.** El cierre, y la pregunta de criterio: **¿Express o NestJS?** Primero, el dato que casi nadie dice bien: **no son rivales en el mismo nivel.** NestJS **corre sobre Express** (por defecto usa `@nestjs/platform-express`; también puede usar Fastify). Nest es una **capa de arquitectura y opinión** encima de un motor HTTP que, las más de las veces, es Express. ⚠️ NestJS v11 (2026) ya trae Express 5 como motor por defecto — verificalo en su changelog.

Entonces la pregunta real no es "cuál es mejor", es **"cuánta estructura impuesta querés".**

| | **Express** | **NestJS** |
|---|---|---|
| Filosofía | Minimalista, sin opinión | Opinado, estructura impuesta |
| Estructura | La armás vos | Módulos, controllers, providers, DI |
| Curva | Baja para empezar | Más alta (decoradores, DI, conceptos) |
| Brilla en | Servicios chicos, prototipos, control total | Apps grandes, equipos, mantenibilidad a largo plazo |
| TypeScript | Opcional (lo sumás) | First-class, integrado |

Cuándo **Express**: un microservicio chico, un prototipo, un endpoint simple, o cuando querés **control total** sin la ceremonia de Nest. La contra: en un proyecto grande, esa libertad sin disciplina termina en el "plato de fideos" (cada dev estructura distinto) — ahí es donde [SOLID](solid.md) y la arquitectura por capas se vuelven tu responsabilidad manual.

Cuándo **NestJS**: una app grande, con varios equipos, donde la **estructura impuesta** (módulos, inyección de dependencias, separación clara) es una ventaja, no un estorbo. Nest te da gratis la disciplina que en Express tenés que imponer a mano. Por eso el hub tiene [tres módulos de NestJS](nestjs.md).

El criterio senior: **empezá entendiendo Express (el motor), y elegí NestJS cuando la escala del proyecto/equipo justifique la estructura.** Saber Express te hace mejor con Nest, porque entendés qué hace por debajo. Al revés no funciona: saber solo Nest te deja sin entender el piso.

La frase mental de cierre: **no es "Express vs NestJS" — Nest corre sobre Express. La pregunta es cuánta estructura impuesta querés: Express para control y proyectos chicos, NestJS para escala y equipos. Entendé el motor (Express) y subí a Nest cuando la escala lo pida.**

**Ejercicios 8**
8.1 🔁 ¿Es correcto decir "Express vs NestJS" como si fueran rivales del mismo nivel? Explicá la relación real entre los dos.
8.2 🧠 ¿En qué escenario elegirías Express pelado y en cuál NestJS? Dá el criterio, no solo ejemplos.
8.3 🧠 ¿Por qué "saber Express te hace mejor con NestJS, pero no al revés"?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** "Minimalista" significa que Express te da solo lo esencial (routing + middleware) y **no te impone estructura, carpetas ni una forma correcta** de hacer las cosas: el resto lo armás vos. La frase que lo define por dentro: **"una aplicación Express es, en el fondo, una serie de llamadas a funciones middleware."**

**1.2** La razón más fuerte: **NestJS corre sobre Express.** Por defecto Nest usa Express como su plataforma HTTP, así que todo corre sobre su motor HTTP (y el middleware de Nest **es** middleware de Express; guards/interceptors/pipes son abstracciones del pipeline de Nest sobre ese motor). Si no entendés Express, ese piso es una caja negra. Además es el estándar de facto y te enseña los fundamentos del request/response sin la magia de los decoradores.

**1.3** Porque "no impone estructura" significa que **cada dev la inventa a su manera**: en un equipo grande terminás con cinco formas distintas de organizar rutas, validación y errores → inconsistencia, difícil de mantener y de onboardear. La libertad sin disciplina escala mal. Ahí NestJS (estructura impuesta) o aplicar SOLID/capas a mano sobre Express se vuelve necesario.

**2.1** `express.json()` es un middleware built-in que **parsea el body JSON** del request y lo deja en `req.body`. Si te lo olvidás, en Express 5 `req.body` es **`undefined`** (en v4 era `{}`), así que cualquier intento de leer el cuerpo falla — es el error #1 de principiantes ("¿por qué `req.body` está vacío?").

**2.2** Segmentos de URL (`:id`) → **`req.params`**; query string (`?orden=asc`) → **`req.query`**; cuerpo del request → **`req.body`** (requiere `express.json()`).

**2.3**
```ts
import express, { Request, Response } from 'express'

const app = express()

app.get('/health', (req: Request, res: Response) => {
  res.json({ status: 'ok' })
})

app.listen(3000, () => console.log('http://localhost:3000'))
```

**3.1** Un **route param** (`:id`) es un **segmento del path** de la URL (`/usuarios/42`) y queda en **`req.params`** (`req.params.id === '42'`); identifica un recurso. Un **query param** (`?orden=asc`) va **después del `?`**, no es parte del path, y queda en **`req.query`** (`req.query.orden`); suele usarse para filtros/orden/paginación.

**3.2** `express.Router()` resuelve el **monolito de un solo archivo**: te deja agrupar las rutas de un recurso en su propio módulo (una "mini-app" con su propio routing y middleware), que después montás en la app con un prefijo: `app.use('/api/usuarios', usuariosRouter)`. Mantiene la app organizada (un router por recurso).

**3.3**
```ts
import { Router } from 'express'
const router = Router()

router.get('/', (req, res) => { res.json([]) })              // GET  /api/productos
router.post('/', (req, res) => { res.status(201).json({}) }) // POST /api/productos
router.get('/:id', (req, res) => { res.json({ id: req.params.id }) }) // GET /api/productos/:id

export default router

// en app.ts:
// app.use('/api/productos', router)
```

**4.1** Un middleware puede: (1) ejecutar código, (2) modificar `req`/`res`, (3) terminar el ciclo (mandar una response), o (4) llamar `next()` para pasar al siguiente. Si **no** termina el ciclo **y** **no** llama `next()`, **la request queda colgada para siempre** (el cliente nunca recibe respuesta).

**4.2** Porque los middleware se ejecutan **en el orden en que se cargan**. Si las rutas que leen `req.body` corren antes de que `express.json()` haya parseado el cuerpo, `req.body` todavía no existe. `express.json()` arriba garantiza que, cuando el request llega a tu ruta, el body ya está parseado.

**4.3** `app.use(mw)` corre para **todos los métodos HTTP** y matchea por **prefijo** de path (si lo montás en `/api`, corre para todo lo que empiece con `/api`). `app.get('/x', mw)` corre **solo para GET** y solo en el path `/x` (matcheo de ruta, no de prefijo). `use` = middleware general; `get`/`post`/etc = handler de una ruta específica.

**4.4**
```ts
import { Request, Response, NextFunction } from 'express'

function requiereApiKey(req: Request, res: Response, next: NextFunction): void {
  const apiKey = req.header('x-api-key')
  if (apiKey !== process.env.API_KEY) {
    res.status(401).json({ error: 'API key inválida o ausente' })
    return    // cortamos el ciclo; NO llamamos next()
  }
  next()      // ok: pasa al siguiente
}

app.use(requiereApiKey)
```

**4.5**
```ts
import express from 'express'
const app = express()

app.post('/eco', (req, res) => {
  res.json({ recibido: req.body })
})

app.use(express.json())   // ❌ MAL: va DESPUÉS de la ruta

// Probás:  curl -X POST localhost:3000/eco -H 'Content-Type: application/json' -d '{"hola":1}'
// Resultado: { "recibido": undefined }  ← el body nunca se parseó: cuando el request
//   llegó a /eco, express.json() todavía no había corrido (se carga después).

// ✅ ARREGLO: mover express.json() ARRIBA de las rutas
// app.use(express.json())
// app.post('/eco', (req, res) => res.json({ recibido: req.body }))
//   → { "recibido": { "hola": 1 } }
```
La lección: **el orden de carga es el orden de ejecución.** Un middleware cargado después de la ruta que lo necesita nunca llega a tiempo. Por eso `express.json()` (y todo lo global) va arriba.

**5.1** Tiene **cuatro** argumentos: `(err, req, res, next)`. La razón: Express identifica al error handler por su **aridad** (número de parámetros) — con 4 args sabe que es manejador de errores; con 3 lo trata como middleware normal. Se coloca **último**, después de todas las rutas y middlewares.

**5.2** En Express 4, un `throw` (o promesa rechazada) en un handler `async` **no era capturado** por Express: la request quedaba colgada o el proceso podía crashear, salvo que usaras `try/catch` + `next(err)` o el paquete `express-async-errors`. En Express 5, los handlers que **devuelven una promesa rechazada o lanzan dentro de una `async function`** reenvían el error **automáticamente** al error handler (como un `next(err)` implícito). Ya no necesitás el `try/catch` para ese caso.

**5.3** Cuando el error ocurre dentro del **callback de una API vieja estilo Node** (que no devuelve una promesa) — Express no puede "ver" ese error porque no hay promesa que rechace. Ejemplo:
```ts
app.get('/archivo', (req, res, next) => {
  fs.readFile('/ruta', (err, data) => {
    if (err) { next(err); return }   // ⬅️ a mano: v5 NO captura esto solo
    res.send(data)
  })
})
```
La regla: con `async/await` v5 captura solo; con callbacks viejos, `next(err)` manual.

**5.4**
```ts
import { Request, Response, NextFunction } from 'express'

interface ErrorConStatus extends Error { status?: number }

app.use((err: ErrorConStatus, req: Request, res: Response, next: NextFunction) => {
  console.error(err.stack)
  const status = err.status ?? 500
  res.status(status).json({ error: err.message || 'Error interno' })
})
```
(Mantenemos los 4 argumentos —incluido `next` aunque no se use— para que Express lo reconozca como error handler.)

**6.1** **No**, Express no valida el body por vos (es minimalista por diseño). Usás una librería: **`zod`**, `joi` o `express-validator`. Es importante porque **nunca podés confiar en `req.body`**: el cliente puede mandar cualquier cosa (campos faltantes, tipos incorrectos, datos maliciosos). Validar el input es la primera línea de defensa de la API.

**6.2** Crear con éxito → **`201`** (Created); input inválido → **`400`** (Bad Request); recurso inexistente → **`404`** (Not Found). No usar `200` para todo porque el status code **es información**: clientes, proxies, logs y monitoreo se apoyan en él para saber qué pasó. Un `200` con un body `{error: ...}` adentro es mentirle a toda la infraestructura.

**6.3** Orden, de arriba a abajo: **`express.json()`** → middlewares globales (cors, helmet, logger) → **routers de recursos** → **404 catch-all** → **error handler**. Ese orden importa porque: el body se parsea antes de las rutas; las rutas matchean primero; si ninguna matcheó, cae al 404; y cualquier error (de cualquier punto de arriba) cae al error handler, que va último por su aridad.

**6.4**
```ts
import { Router } from 'express'
import { z } from 'zod'

const router = Router()

const CrearPedido = z.object({
  productoId: z.string(),
  cantidad: z.number().int().positive(),
})

router.post('/', (req, res) => {
  const parsed = CrearPedido.safeParse(req.body)
  if (!parsed.success) {
    res.status(400).json({ errores: parsed.error.issues })
    return
  }
  const pedido = { id: 1, ...parsed.data }
  res.status(201).json(pedido)
})

export default router  // montar con app.use('/api/pedidos', router)
```

**7.1** Los tres se importan del paquete **`'express'`**: `import { Request, Response, NextFunction } from 'express'`. Necesitás el paquete de tipos **`@types/express`**, y con Express 5 tiene que ser la **línea v5** (hoy `5.0.6`) — no mezclar `@types/express@4` con `express@5`.

**7.2** **Tipar** `req.body` (con un type de TS) es una promesa **en tiempo de compilación**: le dice al compilador "asumí que el body tiene esta forma", pero **en runtime no chequea nada** — el body real es lo que el cliente mande. **Validar** con zod chequea **en runtime** que los datos lleguen como esperás y rechaza lo que no. Tipo = ayuda del compilador y autocompletado; validación = defensa real contra input malicioso o malformado. Necesitás los dos.

**7.3**
```ts
import { Request, Response } from 'express'

type Params = { id: string }
type Query = { incluirInactivos?: string }   // los query params llegan como string

app.get('/usuarios/:id', (req: Request<Params, unknown, unknown, Query>, res: Response) => {
  const id = req.params.id                          // string
  const incluir = req.query.incluirInactivos === 'true'  // string → boolean
  res.json({ id, incluir })
})
```
(Nota: los query params siempre llegan como string, por eso `incluirInactivos?: string` y la comparación `=== 'true'`.)

**8.1** No es del todo correcto: **no son rivales del mismo nivel.** NestJS **corre sobre Express** (por defecto usa `@nestjs/platform-express`; puede usar Fastify). Express es el **motor HTTP minimalista**; NestJS es una **capa de arquitectura/opinión** (módulos, DI, decoradores) **encima** de ese motor. La pregunta real no es "cuál gana" sino "cuánta estructura impuesta querés".

**8.2** **Express pelado**: microservicios chicos, prototipos, endpoints simples, o cuando querés **control total** sin ceremonia. **NestJS**: apps grandes, varios equipos, donde la **estructura impuesta** (módulos, inyección de dependencias, separación clara) es una ventaja para la mantenibilidad. El criterio: **¿la escala del proyecto/equipo justifica el costo de la estructura?** Si sí, Nest; si es chico y querés libertad, Express.

**8.3** Porque NestJS está **construido sobre** Express: corre sobre su motor HTTP y su middleware es middleware de Express (guards/interceptors/pipes son abstracciones propias de Nest montadas sobre ese motor). Si entendés Express, entendés qué hace Nest realmente (no es magia). Al revés no: si solo sabés Nest, no entendés el piso sobre el que corre, y cuando algo falla a nivel HTTP/middleware quedás sin el modelo mental para diagnosticarlo.

---

> **Para seguir.** La fuente de verdad es [expressjs.com](https://expressjs.com) — en particular la [guía de routing](https://expressjs.com/en/guide/routing.html), [using middleware](https://expressjs.com/en/guide/using-middleware.html), [error handling](https://expressjs.com/en/guide/error-handling.html) y, clave para 2026, la [guía de migración 4→5](https://expressjs.com/en/guide/migrating-5.html). Antes de copiar lo marcado con ⚠️ (versión exacta de `express` y `@types/express`, el motor por defecto de NestJS, la sintaxis de routing de v5), re-verificá: Express 5 cambió APIs y los tutoriales viejos de v4 abundan. El puente natural desde acá: [NestJS](nestjs.md) para subir a la versión opinada y estructurada, [SOLID](solid.md) y [DDD](ddd.md) para la arquitectura que en Express imponés a mano, y [Reverse proxy y la capa web](reverse-proxy.md) para lo que va delante de tu Express en producción.
