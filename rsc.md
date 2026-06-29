# React Server Components a fondo

**El boundary server/client, React Flight y Server Actions · React 19 · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es el complemento natural de [TanStack Start](tanstack-start.md): ahí los RSC quedaron como "profundización opcional"; acá los abrimos enteros. **RSC es de lo más confuso del ecosistema React para quien viene de SPA** — no porque sea mágico, sino porque cambia *dónde corre tu código*. Si entendés el boundary, entendés todo lo demás.

**Lo que asumimos.** React (componentes, props, hooks como `useState`/`useEffect`), JSX, `async/await`, y tener una idea de qué es el bundling. Ayuda haber visto [TanStack Start](tanstack-start.md) (la dicotomía client-first vs server-first).

> **¿Te falta alguna base?** Si **SSR vs CSR** te suena nuevo, lo aclaramos en el módulo 2 (es central acá). Si nunca tocaste un meta-framework de React, leé primero [TanStack Start](tanstack-start.md): este módulo se apoya en esos conceptos. Convertir el prerrequisito en rampa, no en muro.

> ⚠️ **Nota sobre datos volátiles.** RSC es estable como *feature* de React 19, pero el ecosistema que lo implementa se mueve rápido (versiones de frameworks, estado de soporte, APIs de caché). Todo lo marcado con ⚠️ verificalo contra `react.dev` y los docs del framework que uses antes de tomarlo como definitivo. Las versiones citadas son las vigentes a mediados de 2026.

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere un proyecto armado).

**Índice de módulos**
1. Qué es un RSC y qué problema resuelve
2. La confusión #1: RSC NO es SSR
3. El boundary: Server Components, Client Components y `"use client"`
4. Composición vs importación: cómo conviven server y client
5. Serialización: qué cruza el boundary (y qué no)
6. React Flight y el modelo de rendering completo
7. Server Functions / Server Actions (`"use server"`)
8. Quién implementa RSC hoy (y por qué necesitás un framework)
9. Las trampas reales: boundary tax, waterfalls, librerías, caché
10. El criterio: cuándo RSC paga y cuándo es over-engineering

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es un RSC y qué problema resuelve

**Teoría.** La definición oficial de React: un Server Component es *"un nuevo tipo de componente que se renderiza ahead of time, antes del bundling, en un entorno separado de tu app cliente o de tu servidor SSR"*. Ese entorno "server" puede ser **build time** (corre una vez, en CI) o **request time** (corre por cada request, en un web server).

¿Qué resuelve? Dos cosas concretas:

- **Cero JavaScript al cliente para ese componente.** El código de un Server Component **nunca** se envía al browser — solo viaja su *output* renderizado. Si un componente formatea fechas con una librería de 50KB, esa librería no llega al cliente.
- **Acceso directo al data layer, sin construir una API.** Un Server Component puede leer la base de datos, el filesystem o un CMS directamente, porque corre en el server:

```jsx
// Server Component (en un framework RSC, esto es el default)
async function Page({ page }) {
  const content = await file.readFile(`${page}.md`)  // acceso directo, sin fetch a una API
  return <div>{sanitizeHtml(marked(content))}</div>
}
```

Fijate el `await` en el cuerpo del componente: los Server Components pueden ser **async**. Eso solo, para alguien de SPA, ya es raro — y es la punta del ovillo.

La frase mental: **un Server Component corre en el server y manda HTML/datos, no código. Reducís el JS del cliente a cambio de que ese componente no maneje interactividad por sí mismo** — pero sí puede convivir con piezas interactivas (módulo 4). (El cómo de ese límite, en el módulo 3.)

**Ejercicios 1**
1.1 🔁 ¿Cuáles son los dos problemas concretos que resuelve un Server Component?
1.2 🔁 ¿En qué dos momentos puede correr el entorno "server" de un RSC?
1.3 🧠 Un componente usa una librería pesada solo para formatear datos antes de mostrarlos. ¿Por qué moverlo a Server Component mejora el bundle del cliente?

---

## Módulo 2 — La confusión #1: RSC NO es SSR

**Teoría.** Esta es **la confusión más común de todo el tema**, y hasta que no la tengas clara, RSC te va a parecer humo. **RSC y SSR son cosas distintas que conviven, no compiten.**

| Eje | SSR clásico | RSC |
|---|---|---|
| Qué produce | **HTML** (un string) | **RSC payload** (descripción serializada del árbol, "Flight") |
| El JS del componente | **Sí** viaja al cliente (para hidratar) | **No** viaja (queda 100% en el server) |
| Qué hace el cliente | Hidrata ese HTML para volverlo interactivo | Reconstruye el árbol React desde el payload |
| Se ejecuta dos veces | Sí (server→HTML, cliente→hidratación) | No (Server Components solo corren en server) |

La frase clave: **SSR ataca *cuándo* ves el primer pixel (manda HTML temprano); RSC ataca *cuánto JS* mandás (no manda el código del componente).** Son ortogonales — de hecho, en una app real **se usan juntos**: RSC genera el payload, SSR lo convierte en HTML para el primer paint, y el cliente hidrata solo las partes interactivas (módulo 6).

El error típico es pensar "SSR ya renderiza en el server, ¿para qué RSC?". Porque SSR igual **te manda todo el JS del componente al cliente** para hidratarlo. RSC dice: para los componentes que no necesitan interactividad, **ni siquiera mandes su JS**.

**Ejercicios 2**
2.1 🔁 Nombrá dos diferencias concretas entre lo que produce SSR y lo que produce RSC.
2.2 🧠 "Si ya tengo SSR, RSC no me suma nada." ¿Por qué esta afirmación es falsa? Explicá qué sigue mandando SSR que RSC no.
2.3 🧠 ¿Por qué se dice que SSR y RSC son "ortogonales" y se usan juntos, en vez de ser alternativas?

---

## Módulo 3 — El boundary: Server Components, Client Components y `"use client"`

**Teoría.** En un framework RSC, **todo componente es Server Component por defecto**. No hay ninguna directiva para marcarlos como server — es el estado base. (Ojo: el malentendido común es creer que se marcan con `"use server"`. **No.** Eso es otra cosa, módulo 7.)

Para volver un componente **interactivo**, lo marcás como Client Component con la directiva `"use client"`:

```jsx
'use client'                 // al tope del archivo, sobre los imports
import { useState } from 'react'

export default function Counter() {
  const [n, setN] = useState(0)
  return <button onClick={() => setN(n + 1)}>{n}</button>
}
```

**Qué hace EXACTAMENTE `"use client"`** (y acá está el matiz que casi nadie entiende al principio): *introduce un boundary server-cliente en el árbol de **módulos**, creando un subárbol de módulos cliente.* O sea:

- Va **al principio del archivo**, sobre cualquier import (solo comentarios pueden ir arriba). Comillas simples o dobles, **nunca backticks**.
- Marca **todos los exports de ese archivo** como Client Components.
- El boundary es **a nivel de módulo (el grafo de imports), no del árbol de render**. El archivo marcado **y todas sus dependencias importadas** se evalúan en el cliente.

Esto lleva a la división de capacidades:

| | Server Component (default) | Client Component (`"use client"`) |
|---|---|---|
| `async/await` en render | ✅ Sí | ❌ No |
| Acceso directo a DB/fs | ✅ Sí | ❌ No |
| `useState`, `useEffect` | ❌ No | ✅ Sí |
| Event handlers (`onClick`) | ❌ No | ✅ Sí |
| APIs del browser | ❌ No | ✅ Sí |
| Su JS viaja al cliente | ❌ No | ✅ Sí |

> Ojo con el encuadre de la fila "APIs del browser": no es que **React lo prohíba**, es que en el server **no existe `window`** (ni `localStorage`, ni `document`). Un Server Component no corre en el browser, así que esas APIs simplemente no están — la "prohibición" es una consecuencia del entorno, no una regla.

La frase mental: **`"use client"` no marca un componente, marca una *frontera en el grafo de módulos*. Todo lo que ese archivo importe cruza con él al cliente.** (Esta idea es la base de la trampa #1 del módulo 9 — leela con atención.)

> En una línea, la consecuencia que vas a sufrir: si un archivo con `"use client"` hace `import { algo } from './pesado'`, ese `./pesado` **también** termina en el bundle del cliente, lo necesites ahí o no. **Importar arrastra.** (El anti-pattern completo, con el patrón para evitarlo, está en el módulo 9.)

**Ejercicios 3**
3.1 🔁 ¿Cuál es el componente "por defecto" en un framework RSC, y cómo marcás uno como interactivo?
3.2 🔁 Nombrá tres cosas que un Server Component NO puede hacer y dos que sí.
3.3 🧠 La directiva `"use client"` marca "un boundary en el grafo de módulos, no en el árbol de render". Explicá con tus palabras qué implica eso para los archivos que ese módulo importa.
3.4 ✍️ Tenés una `Page` que lee la DB (server) y muestra un `<Counter>` interactivo (`useState`) enterrado adentro. ¿Dónde ponés el `"use client"` para que **solo** el contador sea cliente y la página siga siendo server? Escribí el árbol y justificá.

---

## Módulo 4 — Composición vs importación: cómo conviven server y client

**Teoría.** Si los Server Components no pueden ser interactivos y los Client Components no pueden ser async ni acceder al server… ¿cómo los combinás? Acá está la regla de oro, y es **la que más cuesta interiorizar**:

> **Un Server Component puede renderizar un Client Component. Un Client Component NO puede *importar* un Server Component — pero SÍ puede recibirlo como `children` (o prop).**

La distinción clave es **importación (no) vs composición (sí)**:

```jsx
// ✅ Server Component (default): renderiza un Client Component y le pasa contenido server como children
async function Notes() {
  const notes = await db.notes.getAll()
  return (
    <Expandable>              {/* Client Component */}
      <ServerOnlyContent />   {/* pasa como children → sigue siendo server */}
    </Expandable>
  )
}

// Expandable.jsx
'use client'
export default function Expandable({ children }) {
  const [open, setOpen] = useState(false)
  return <div>{open && children}</div>   // renderiza children, NO los importa
}
```

¿Por qué funciona? Porque **podés pasar JSX como prop**. El `<ServerOnlyContent />` se renderiza en el server y cruza el boundary **ya renderizado** (como elemento), no como módulo a importar. Si en cambio `Expandable` hiciera `import ServerOnlyContent from './...'`, ese módulo cruzaría al cliente y dejaría de ser server-only.

La frase mental: **el contenido server entra al cliente por la "puerta de `children`", ya cocinado — nunca por la puerta del `import`.**

**Ejercicios 4**
4.1 🔁 ¿Un Client Component puede importar un Server Component? ¿Puede renderizarlo de alguna forma?
4.2 🧠 ¿Por qué pasar un Server Component como `children` lo mantiene server, pero importarlo dentro de un Client Component no?
4.3 ✍️ Tenés un `<Layout>` que necesita estado de tema (cliente) pero quiere envolver contenido que lee de la DB (server). Escribí el patrón correcto (un provider cliente que recibe `children`).

---

## Módulo 5 — Serialización: qué cruza el boundary (y qué no)

**Teoría.** Cuando un Server Component le pasa props a un Client Component, esas props tienen que **viajar por la red** (serializadas en el **payload Flight** — el formato serializado del árbol que viaja por la red; lo abrimos a fondo en el módulo 6). Por eso **solo lo serializable cruza**.

**Cruza (serializable):**
- Primitivos: string, number, bigint, boolean, null, undefined, Symbol registrado (`Symbol.for`).
- Estructuras: Array, objeto plano, Map, Set, Date, TypedArray, ArrayBuffer.
- **JSX / elementos React** (por eso `children` funciona, módulo 4).
- **Promises** (se crean en el server sin `await` y se consumen en el cliente con `use`).
- **Server Functions** (módulo 7).

**NO cruza:**
- **Funciones comunes** — incluidos los **event handlers**. (Si necesitás pasar comportamiento, va por Server Functions o se define dentro del Client Component.)
- Clases e instancias de clase.
- Objetos con prototipo `null` (`Object.create(null)`).
- Symbols no registrados globalmente.

> ⚠️ Esta lista refleja el comportamiento del runtime de serialización (Flight). React documenta los tipos serializables junto a las directivas (`'use client'` / `'use server'`); si dependés de un caso borde, confirmalo en `react.dev`.

Un patrón potente: pasar una **Promise sin resolver** desde el server y resolverla en el cliente, para no bloquear:

```jsx
// Server: NO hace await, pasa la promesa
function Page({ note }) {
  const commentsPromise = db.comments.get(note.id)
  return <Comments commentsPromise={commentsPromise} />
}

// Client
'use client'
import { use } from 'react'
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise)   // se suspende hasta que resuelve
  return <ul>{comments.map(c => <li key={c.id}>{c.text}</li>)}</ul>
}
```

La frase mental: **el boundary es la red. Si no se puede serializar y mandar por un cable, no cruza** — y "una función" no se puede mandar por un cable.

**Ejercicios 5**
5.1 🔁 Nombrá tres tipos que cruzan el boundary server→client y dos que no.
5.2 🧠 ¿Por qué un event handler (una función) no puede pasarse como prop de un Server Component a un Client Component? Relacionalo con "el boundary es la red".
5.3 🧠 ¿Qué ventaja da pasar una Promise sin `await` desde el server, en vez de esperarla antes de renderizar?
5.4 ✍️ Este Server Component intenta pasarle un `onDelete` (función) a un Client Component como prop: `<Item onDelete={() => db.delete(id)} />`. No cruza el boundary. Reescribilo con el patrón correcto (una Server Function como prop, o el handler definido dentro del Client Component) y explicá por qué.

---

## Módulo 6 — React Flight y el modelo de rendering completo

**Teoría.** **Flight** (o "RSC payload") es el formato en que se serializa el árbol de Server Components. **No es HTML**: es una descripción del árbol que contiene (a) el output renderizado de los Server Components, (b) **referencias de módulo** —placeholders— donde hay un Client Component, y (c) las props serializadas de cada uno. Se transmite como **stream** (en chunks), así el cliente empieza a procesarlo sin esperar la respuesta completa — esto se integra con `<Suspense>` para streaming.

El orden típico de una request en un framework RSC (ej. Next.js App Router) ⚠️ *—este pipeline lo orquesta el framework, no es spec de React—*:

1. **Server — render RSC**: ejecuta los Server Components (con sus `await`) y produce el **payload Flight**. Los Client Components quedan como referencias de módulo + props.
2. **Server — SSR** *(opcional)*: usa ese payload para generar **HTML**, que se **streamea** al browser → primer paint rápido (no interactivo todavía). Este paso se puede **omitir**: RSC sin SSR (solo el payload Flight) es válido — es el modelo **client-first** (el de TanStack Start / React Router en data-mode). React lo dice explícito: *"Optionally, that bundle can then be server-side rendered to create the initial HTML for the page."*
3. **Cliente — recepción**: llega el HTML (pintado inmediato) y, en paralelo, el payload Flight que React usa para construir/actualizar su árbol.
4. **Cliente — hidratación**: React hidrata **solo los Client Components** para volverlos interactivos.

En navegaciones siguientes, el cliente pide un **nuevo payload Flight** (no un HTML completo) y lo mergea, reusando layouts sin re-renderizarlos.

La frase mental: **RSC produce una "receta serializada" del árbol (Flight); SSR la cocina en HTML para el primer pixel; el cliente solo hidrata las islas interactivas.** Las tres etapas, una sola request.

> ⚠️ El formato interno de Flight es un detalle de implementación de bajo nivel (no sigue semver). No programes contra su estructura.

### De Flight a tu código: las convenciones que vas a escribir ⚠️

Todo lo anterior es el **motor**. La **interfaz** que tocás el día 1 son convenciones del framework (acá, **Next.js App Router**; otros tienen sus equivalentes —verificá el tuyo—). Las cuatro que más vas a usar:

- **`loading.tsx`** — el framework lo envuelve automáticamente en un `<Suspense>` alrededor de la ruta: mientras los Server Components async cargan, se muestra este fallback (un **skeleton**). Es la cara "de aplicación" del streaming del módulo 6: en vez de escribir el `<Suspense>` a mano, dejás un archivo y el framework arma el boundary.
- **`error.tsx`** — un **Error Boundary** de la ruta (es Client Component: lleva `"use client"`). Recibe `error` y una función `reset()` para reintentar el render del segmento. Captura los errores de los Server Components de ese segmento.
- **`not-found.tsx`** — la UI para `notFound()` (404 de un recurso que no existe).
- **`generateMetadata`** — función `async` que corre en el server y devuelve el `<title>`/`<meta>` para **SEO** a partir de los datos de la página (ej. el título de un post). Reemplaza el baile de `react-helmet` del mundo SPA.

```tsx
// app/posts/[id]/page.tsx — Server Component
export async function generateMetadata({ params }: { params: { id: string } }) {
  const post = await db.posts.get(params.id)   // corre en el server
  return { title: post.title, description: post.excerpt }
}
// app/posts/[id]/loading.tsx → <PostSkeleton />   (fallback automático mientras carga)
// app/posts/[id]/error.tsx   → 'use client' + {error, reset}  (boundary de error del segmento)
```

La idea: **el framework convierte `<Suspense>` y Error Boundary (que viste en [react-fundamentos](react-fundamentos.md)) en archivos con nombre fijo.** Entendés el motor; estos archivos son la palanca.

**Ejercicios 6**
6.1 🔁 ¿Qué contiene el payload Flight y por qué se dice que "no es HTML"?
6.2 🔁 En el pipeline de una request, ¿qué se hidrata en el cliente: todo el árbol o solo una parte?
6.3 🧠 En una navegación cliente (no la carga inicial), ¿qué pide el cliente y por qué eso es más barato que recargar la página?
6.4 🔁 ¿Para qué sirven `loading.tsx`, `error.tsx` y `generateMetadata` en Next App Router, y con qué primitivas de React se corresponden los dos primeros?
6.5 🧠 ¿Por qué `loading.tsx` es "el `<Suspense>` que no escribís a mano"? ¿Qué le da el framework por encima de poner un `<Suspense>` vos mismo?

---

## Módulo 7 — Server Functions / Server Actions (`"use server"`)

**Teoría.** Los Server Components son la **lectura** (traen data). Las **Server Functions** son la **escritura/mutación**: funciones async que corren en el server pero que **invocás desde el cliente**. Se marcan con `"use server"`:

```js
// (a) inline, al tope del cuerpo de la función
async function addToCart(data) {
  'use server'
  // corre en el server
}

// (b) a nivel módulo: TODOS los exports del archivo se vuelven Server Functions
'use server'
export async function updateName(formData) { /* ... */ }
```

El framework crea una **referencia** a esa función; al llamarla desde el cliente, React hace un **request al server**, la ejecuta y devuelve el resultado. Se integran bien con forms y dan **progressive enhancement** (el form puede enviarse antes de que cargue el JS):

```jsx
'use client'
import { updateName } from './actions'

function UpdateName() {
  return (
    <form action={updateName}>     {/* Server Function como action */}
      <input type="text" name="name" />
    </form>
  )
}
```

### ⚠️ El gotcha de seguridad más importante

**`"use server"` NO es el espejo de `"use client"`.** No marca código como "server-only" — marca un **endpoint invocable desde el cliente**. Consecuencia directa, en palabras de la doc oficial: *"los argumentos de una Server Function son completamente controlados por el cliente… tratalos siempre como input no confiable… validá y escapá".*

O sea: cualquiera puede llamar a esa función con cualquier payload, porque es un endpoint expuesto. **Validá y autorizá SIEMPRE adentro de la función.** Es el mismo criterio que un endpoint de API tradicional.

### El otro lado: auth y datos sensibles en la lectura

La escritura (Server Functions) ya la cubrimos; el lado **lectura** (Server Components) tiene su propio criterio. Como un Server Component corre en el server **en request time**, puede leer la **sesión/cookies/headers** directamente (en Next, vía `cookies()`/`headers()`; no hay `localStorage` ni `document` porque no es el browser, módulo 3) y **autorizar en la capa de datos** antes de devolver nada. Pero ojo con el boundary:

> ⚠️ **No serialices secretos al cliente.** Todo lo que un Server Component le pase como **prop a un Client Component cruza la red en el payload Flight** (módulo 5) y queda visible en el browser. Si tu objeto `user` trae el `passwordHash`, un token o datos de otro tenant, **los estás filtrando** aunque "no los muestres". Pasá al cliente solo los campos que el cliente necesita —volvé a "el boundary es la red": lo que cruza, se ve—.

**Hacé la autorización en el server (en el Server Component o en la capa de datos), nunca confíes en ocultar en el cliente.**

### Terminología (te va a confundir leyendo blogs) ⚠️

Hasta septiembre de 2024 React llamaba "Server Actions" a todas las Server Functions. Hoy: una Server Function es una **Server Action** solo si se pasa a una prop `action` (o se llama desde una action). **Toda Server Action es una Server Function, pero no al revés.**

La frase mental: **un Server Component lee; una Server Function escribe — y como es un endpoint que el cliente invoca, sus argumentos son tan poco confiables como los de cualquier API pública.**

**Ejercicios 7**
7.1 🔁 ¿Qué es una Server Function y en qué se diferencia, en propósito, de un Server Component?
7.2 🧠 ¿Por qué es un error de seguridad pensar que `"use server"` hace que el código sea "solo del servidor y por lo tanto seguro"? ¿Qué tenés que hacer siempre?
7.3 🧠 ¿Toda Server Function es una Server Action? Explicá la relación.
7.4 ✍️ Escribí un `<form>` que use una Server Function como `action` para guardar un nombre, con la validación de los argumentos hecha DENTRO de la función (acordate: los args son input no confiable).
7.5 🧠 Un Server Component lee el `user` completo de la DB (incluye `passwordHash` y un `apiToken`) y se lo pasa entero como prop a un `<Profile>` Client Component que solo muestra el nombre. ¿Qué problema de seguridad hay y cómo lo arreglás?

---

## Módulo 8 — Quién implementa RSC hoy (y por qué necesitás un framework)

**Teoría.** Acá un punto que cambia cómo encarás todo: **no podés hacer `npm install react` y usar RSC en un proyecto vacío.** RSC es, como lo describe Mark Erikson (mantenedor de Redux), una feature híbrida poco habitual: el core vive en React, pero **no funciona sin un bundler + framework/router** que parsee las directivas `"use client"`/`"use server"`, transforme el código y calcule el grafo de módulos server/cliente. Es arquitectural, no accidental.

El panorama de implementaciones ⚠️ (a mediados de 2026, **muy volátil**):

| Stack | Modelo | Madurez para producción |
|---|---|---|
| **Next.js App Router** | Server-first | ✅ Battle-tested (prod-ready desde 2023) |
| **TanStack Start** | Client-first (RSC como stream de datos) | 🟡 Framework en RC; **soporte RSC marcado "experimental"** en su doc (se mantiene así en early v1; no confundir ambos estados) |
| **Parcel** | Sin framework (bundler directo, `@parcel/rsc`) | 🟡 Beta |
| **React Router 7** | Opt-in sobre Framework/Data Mode | 🔴 Preview/inestable (el propio equipo NO lo recomienda en prod) |
| **Waku** | RSC-first minimalista | 🔴 1.0 alpha |

Dos lecturas importantes:
- **Next.js App Router** es la implementación de referencia y la única realmente battle-tested. Si vas a prod hoy con RSC, es la apuesta madura — asumiendo su modelo server-first y su caché (módulo 9).
- El resto está **llegando, no llegado**. Notá que las nuevas implementaciones (TanStack, React Router) hacen RSC **opt-in y no-breaking**: tu código actual sigue funcionando.

> ⚠️ Corrección a material viejo: vas a leer (incluso en artículos respetados) que "Next.js es la única forma de usar RSC". **Era cierto en 2023, ya no.** En 2026 hay varias implementaciones, aunque solo Next esté battle-tested.

La frase mental: **RSC no es "React puro": es React + un bundler/framework que entienda el boundary. Elegir RSC es elegir un framework.**

**Ejercicios 8**
8.1 🔁 ¿Por qué no podés usar RSC con solo `react` instalado, sin framework ni bundler especial?
8.2 🔁 ¿Cuál es hoy la única implementación de RSC considerada battle-tested para producción?
8.3 🧠 Leés un artículo de 2023 que dice "RSC = Next.js, no hay otra forma". ¿Qué le corregís con la info de 2026?

---

## Módulo 9 — Las trampas reales: boundary tax, waterfalls, librerías, caché

**Teoría.** Lo que la gente sufre de verdad en producción. Cuatro dolores principales.

### a) El "boundary tax": la propagación de `"use client"`

La trampa #1. Como `"use client"` marca un boundary de módulo (módulo 3), si lo ponés arriba de un archivo grande o compartido, **arrastrás ese archivo y toda su cadena de imports al bundle del cliente** — borrando el beneficio de RSC.

```jsx
// ❌ ANTI-PATTERN: un 'use client' en el layout arrastra TODO a cliente
'use client'
import { Header } from './header'    // ahora cliente
import { Sidebar } from './sidebar'  // ahora cliente
export default function Layout({ children }) {
  const [theme, setTheme] = useState('dark')   // solo necesitabas esto...
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
}
```

El patrón correcto: **aislar el estado en un wrapper cliente mínimo y pasar el resto por `children`** (módulo 4), así sigue siendo server:

```jsx
// ✅ El ThemeProvider es el ÚNICO cliente
'use client'
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark')
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
}

// layout.tsx — Server Component; Sidebar NO se importa dentro del boundary cliente
import { ThemeProvider } from './theme-provider'
import { Sidebar } from './sidebar'      // sigue siendo server
export default function Layout({ children }) {
  return <ThemeProvider><Sidebar />{children}</ThemeProvider>
}
```

### b) Waterfalls de data fetching

Como los Server Components son async, es fácil crear **waterfalls accidentales**: un padre `await`ea su fetch y un hijo independiente no arranca el suyo hasta que el padre termina. Peor: un Server Component lento **sin un `<Suspense>` por encima bloquea todo el stream**. Cómo evitarlo:
- **Paralelizar** los fetch que no dependen entre sí (`Promise.all()`).
- **`cache()` de React** para deduplicar/memoizar por render (patrón "preload").
- **`<Suspense>`** alrededor de lo lento, para que el resto del stream fluya.

### c) Librerías de terceros que asumen cliente

Cualquier librería que use `createContext`/providers (theme, i18n, state managers, animaciones) **asume cliente** y no corre directo en un Server Component. El patrón estándar: **envolver el provider de terceros en tu propio componente `"use client"`** y dejar ese wrapper como único boundary. Costo real: parte del beneficio de performance se diluye.

### d) La caché (el caso Next) ⚠️

En Next App Router la caché es **multicapa** y es **fuente constante de bugs** (datos stale, refetch inesperado). Cambió de defaults entre versiones (Next ≤14 cacheaba por defecto; Next 15+ no). Y hay confusión de nombres parecidos (`unstable_cache` vs la directiva `use cache` vs "Cache Components"). **No asumas el comportamiento de caché: verificalo en los docs de tu versión exacta.**

La frase mental: **RSC mueve trabajo al server, pero te cobra un "impuesto de boundary": cada `"use client"` mal puesto, cada `await` en cadena y cada librería que asume cliente te devuelve al punto de partida.**

**Ejercicios 9**
9.1 🔁 ¿Qué es el "boundary tax" y cómo lo dispara un `"use client"` mal ubicado?
9.2 🧠 ¿Por qué un Server Component async sin `<Suspense>` por encima puede arruinar el streaming de toda la página?
9.3 ✍️ Tenés una librería de animaciones que usa Context y rompe en un Server Component. Describí (o escribí) el patrón para usarla sin convertir media app en cliente.
9.4 ✍️ Este Server Component genera un waterfall: `const user = await getUser(id)` seguido de `const posts = await getPosts(id)` (los dos fetch son independientes). Reescribilo para que corran en paralelo, y decí dónde pondrías un `<Suspense>`.

---

## Módulo 10 — El criterio: cuándo RSC paga y cuándo es over-engineering

**Teoría.** El cierre: acá convertís todo lo anterior en una decisión que podés defender. Porque la pregunta no es "¿RSC es bueno?", sino "**¿RSC paga para MI app?**".

**RSC paga** (RSC, o híbrido RSC + TanStack Query):
- Sitios **content-heavy**: e-commerce, marketing, blogs, docs → RSC puro elimina JS de verdad, máxima velocidad.
- **Dashboards / SaaS** donde mucho contenido es server-render con **islas** de interactividad.

**RSC es over-engineering** (preferí SPA + loaders/TanStack Query):
- Apps **muy interactivas y de sesión larga**: editores, builders, paneles real-time, admin pesados. Ahí casi todo es estado de cliente; RSC no remueve trabajo real y solo suma boundaries y complejidad.

La alternativa honesta —y esto es clave— es **loaders + TanStack Query** (el modelo que viste en [TanStack Start](tanstack-start.md)): corren en el server en el primer render y en el cliente al navegar, con un **mental model más familiar para quien viene de SPA**, sin boundary tax ni serialización Flight. **Elegir eso NO es "quedarse atrás"** — es elegir la herramienta correcta.

La crítica honesta de la comunidad (Erikson y otros) no es "RSC es malo", sino que el mensaje "RSC es el futuro" hizo creer que es **obligatorio**, cuando el propio equipo de React aclara que son **opcionales**. El ecosistema quedó más **fracturado y complejo** que la simplicidad original de React.

La frase mental de cierre: **RSC es una herramienta para borrar JavaScript del cliente, no una obligación. Si tu app es casi toda interacción, el JS no se va a borrar — y RSC solo te cobra el impuesto sin darte el premio.** Elegí por el perfil de tu app, no por el hype.

**Ejercicios 10**
10.1 🧠 Para un blog/docs con poca interactividad, ¿RSC paga? ¿Por qué?
10.2 🧠 Vas a construir un editor de diagramas colaborativo en tiempo real. ¿RSC o SPA + TanStack Query? Justificá.
10.3 🧠 Un colega dice "hay que migrar todo a RSC porque es el futuro de React". ¿Qué le respondés, con el matiz de lo que dice el propio equipo de React?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** (a) **Cero JS al cliente** para ese componente: su código nunca viaja al browser, solo su output renderizado. (b) **Acceso directo al data layer** (DB, fs, CMS) sin tener que construir y consumir una API.

**1.2** En **build time** (corre una vez, en CI) o en **request time** (corre por cada request, en un web server).

**1.3** Porque el código del Server Component —y la librería pesada que importa para formatear— **nunca se incluye en el bundle del cliente**. El browser recibe solo el resultado ya formateado (texto/HTML), no la librería. El bundle adelgaza.

**2.1** SSR produce **HTML** (un string que el browser pinta y luego hidrata, mandando también el JS del componente). RSC produce un **payload Flight** (descripción serializada del árbol) y **no manda el JS** del Server Component al cliente.

**2.2** Es falsa porque **SSR igual manda todo el JS del componente al cliente** para poder hidratarlo. RSC ataca justo eso: para los componentes que no necesitan interactividad, no manda su código. SSR mejora el *time-to-first-pixel*; RSC reduce el *tamaño del bundle*. Cosas distintas.

**2.3** Porque atacan ejes diferentes (HTML temprano vs menos JS) y **se combinan en la misma request**: RSC genera el payload, SSR lo convierte en HTML para el primer paint, y el cliente hidrata solo las islas interactivas. No elegís uno u otro; el framework usa los dos.

**3.1** El **Server Component** es el default (no lleva directiva). Lo marcás como interactivo (Client Component) poniendo `"use client"` al tope del archivo.

**3.2** **NO puede**: usar `useState`/`useEffect`, definir event handlers, usar APIs del browser. **SÍ puede**: usar `async/await` en el render y acceder directo a DB/filesystem/CMS.

**3.3** Significa que el boundary lo dispara el **import**, no dónde se renderiza el componente. El archivo con `"use client"` **y todas sus dependencias importadas** se evalúan en el cliente y su JS viaja al browser — aunque algunas de esas dependencias no necesiten ser cliente. Por eso importa qué importás dentro de un archivo cliente.

**3.4** El `"use client"` va en el **archivo del `Counter`**, no en la `Page`. La `Page` sigue siendo Server Component (async, lee la DB) y simplemente **renderiza** `<Counter />`: un Server Component puede renderizar un Client Component. Solo el JS del `Counter` (y lo que él importe) cruza al cliente; la página y su acceso a datos quedan en el server.
```tsx
// counter.tsx
'use client'
import { useState } from 'react'
export function Counter() {
  const [n, setN] = useState(0)
  return <button onClick={() => setN(n + 1)}>{n}</button>
}

// page.tsx — Server Component (sin 'use client')
import { Counter } from './counter'
export default async function Page() {
  const data = await db.getStuff()     // sigue en el server
  return <article>{/* ...contenido server... */}<Counter /></article>
}
```
El error sería poner `"use client"` arriba de `page.tsx`: arrastraría la página entera (y el acceso a la DB) al cliente.

**4.1** **No** puede importarlo. Pero **sí** puede renderizarlo si lo recibe como `children` (o como prop JSX) desde un Server Component padre.

**4.2** Porque pasarlo como `children` significa que el Server Component se renderiza **en el server** y cruza el boundary **ya convertido en elemento** (output serializado). Importarlo dentro de un Client Component metería su **módulo** en el grafo de cliente, forzando su evaluación en el browser y rompiendo su naturaleza server-only.

**4.3**
```jsx
// theme-provider.tsx — el único 'use client'
'use client'
import { createContext, useState } from 'react'
export const ThemeContext = createContext('dark')
export function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('dark')
  return <ThemeContext.Provider value={theme}>{children}</ThemeContext.Provider>
}

// layout.tsx — Server Component
import { ThemeProvider } from './theme-provider'
async function Content() {
  const data = await db.getStuff()   // sigue siendo server
  return <main>{/* ... */}</main>
}
export default function Layout() {
  return <ThemeProvider><Content /></ThemeProvider>   // Content entra por children → server
}
```

**5.1** **Cruzan**: primitivos (string/number/boolean…), arrays/objetos planos/Map/Set/Date, JSX/elementos React, Promises, Server Functions. **No cruzan**: funciones comunes (incluidos event handlers), instancias de clase.

**5.2** Porque el boundary es **la red**: las props se serializan en el payload Flight y viajan al cliente. Una función es código con closures — **no se puede serializar y mandar por un cable** de forma significativa. Por eso los handlers se definen dentro del Client Component (o se usan Server Functions).

**5.3** Que **no bloqueás el render del server** esperando esa data. Mandás la promesa pendiente, el server sigue, y el cliente la resuelve con `use` (suspendiéndose solo en esa parte, idealmente bajo `<Suspense>`). Mejora el streaming y el time-to-paint.

**5.4** El `onDelete` es una **función común**, no cruza el boundary (módulo 5). Dos arreglos válidos: (a) pasar una **Server Function** (la mutación corre en el server, con validación adentro); (b) **definir el handler dentro del Client Component** (ej. recibe el `id` por prop —que sí es serializable— y arma el handler localmente).
```tsx
// (a) Server Function como prop — Item es Client, borrar() corre en el server
// actions.ts
'use server'
export async function borrar(id: string) {
  // validar/autorizar acá: el id es input no confiable
  await db.delete(id)
}
// server component:  <Item id={item.id} onDelete={borrar} />

// (b) handler dentro del Client Component — solo cruza el id (string, serializable)
'use client'
function Item({ id, onDelete }: { id: string; onDelete: (id: string) => void }) {
  return <button onClick={() => onDelete(id)}>Borrar</button>
}
```
La razón: "el boundary es la red" — una función con su closure no se serializa; un `id` string sí.

**6.1** Contiene el output renderizado de los Server Components, **referencias de módulo** donde hay Client Components, y las **props serializadas**. "No es HTML" porque es una **descripción serializada del árbol React** (Flight) que el cliente reconstruye, no markup listo para pintar.

**6.2** Solo se hidratan **los Client Components**. Los Server Components ya vienen renderizados y no se hidratan (su JS ni siquiera está en el cliente).

**6.3** Pide un **nuevo payload Flight** (no un HTML completo) y lo mergea en el árbol existente, **reusando layouts sin re-renderizarlos**. Es más barato porque transfiere menos, no recarga el documento entero y evita rehidratar lo que no cambió.

**6.4** `loading.tsx` = la UI de carga del segmento (skeleton), que el framework envuelve en un **`<Suspense>`** automático mientras los Server Components async cargan. `error.tsx` = un **Error Boundary** del segmento (es Client Component; recibe `error` y `reset()`). `generateMetadata` = función async que corre en el server y devuelve `<title>`/`<meta>` para **SEO** desde los datos de la página. Los dos primeros se corresponden con `<Suspense>` y Error Boundary de React.

**6.5** Porque `loading.tsx` **es** un `<Suspense>` que el framework arma por vos alrededor de la ruta: dejás un archivo con nombre fijo y obtenés el boundary de carga sin escribirlo. Lo que suma por encima de ponerlo a mano: lo cablea automáticamente con el **streaming** y la **navegación** del router (el fallback aparece al navegar al segmento y mientras se streamea su contenido), de forma consistente en toda la app, sin que tengas que ubicar cada `<Suspense>`.

**7.1** Una **Server Function** es una función async que corre en el server pero se **invoca desde el cliente** (vía un request que arma el framework); sirve para **mutaciones/escritura**. Un **Server Component** sirve para **lectura/render** de contenido en el server. Lectura vs escritura.

**7.2** Porque `"use server"` **no** hace el código "server-only/seguro": marca un **endpoint invocable desde el cliente**. Sus argumentos los controla el cliente y pueden ser cualquier cosa. Siempre tenés que **validar y autorizar dentro de la función**, igual que en cualquier endpoint de API público.

**7.3** No al revés: **toda Server Action es una Server Function, pero no toda Server Function es una Server Action**. Una Server Function se vuelve "Action" solo cuando se pasa a una prop `action` (o se llama desde dentro de una action). Es terminología que cambió en sep-2024.

**7.4**
```jsx
// actions.ts
'use server'
export async function saveName(formData) {
  const name = formData.get('name')
  if (typeof name !== 'string' || name.trim().length === 0) {
    throw new Error('Nombre inválido')   // validación ADENTRO: los args son no confiables
  }
  // acá también autorizás al usuario logueado antes de escribir
  await db.user.updateName(name.trim())
}

// el form (en un Client o Server Component)
import { saveName } from './actions'
function NameForm() {
  return (
    <form action={saveName}>
      <input name="name" />
      <button>Guardar</button>
    </form>
  )
}
```
El punto del ejercicio: la validación va **dentro** de la Server Function, no en el cliente, porque el endpoint es invocable con cualquier payload (módulo 7, gotcha de seguridad). Bonus: al pasar la función al `action` del form, funciona aun antes de que hidrate el JS (progressive enhancement).

**7.5** Problema: las props que un Server Component le pasa a un Client Component **cruzan la red en el payload Flight** y quedan **visibles en el browser** (DevTools, el HTML/payload). Pasar el `user` entero **filtra `passwordHash` y `apiToken`** aunque `<Profile>` solo muestre el nombre — "el boundary es la red: lo que cruza, se ve". Arreglo: pasar **solo los campos necesarios** (`<Profile name={user.name} />`), o construir un DTO seguro en el server (`{ name, avatarUrl }`) y pasar ese. La regla: serializá al cliente el mínimo, y nunca confíes en "no lo muestro" para datos sensibles.

**8.1** Porque RSC necesita que **un bundler + framework** parseen las directivas `"use client"`/`"use server"`, transformen el código (registrar Client Components y Server Functions) y calculen el **grafo de módulos** server/cliente. React define el contrato, pero la implementación la provee el framework/bundler. Sin eso, las directivas no hacen nada.

**8.2** **Next.js App Router** (prod-ready desde 2023, ~3 años de uso real).

**8.3** Que en 2026 hay **varias implementaciones** además de Next: TanStack Start (RC), Parcel (beta, RSC sin framework), React Router 7 (preview) y Waku (alpha). "Next es la única forma" era cierto en 2023; hoy Next es la **única battle-tested**, pero no la única que existe.

**9.1** Es el costo de arrastrar código innecesario al cliente. Un `"use client"` arriba de un archivo grande o compartido marca un boundary de módulo que **promueve ese archivo y toda su cadena de imports al bundle del cliente**, anulando el "cero JS" de RSC. Se evita aislando el estado en un wrapper mínimo y pasando el resto por `children`.

**9.2** Porque el render del server es un **stream**, y un Server Component async sin `<Suspense>` por encima **bloquea el stream hasta que su `await` termina** — la página entera espera por esa parte lenta. Con `<Suspense>` alrededor de lo lento, el resto del contenido fluye y la parte pesada llega después.

**9.3** El patrón: **crear un wrapper propio con `"use client"`** que importe y renderice el provider de la librería, y usar ese wrapper como **único boundary**, pasándole el contenido por `children`. Así solo el provider (y su árbol cliente) cruza al cliente; el resto de la app sigue siendo server. (Mismo patrón que el ThemeProvider del módulo 4/9a.)

**9.4**
```jsx
// ❌ waterfall: getPosts no arranca hasta que getUser termina, aunque no dependan
const user = await getUser(id)
const posts = await getPosts(id)

// ✅ en paralelo: arrancan juntos
const [user, posts] = await Promise.all([getUser(id), getPosts(id)])
```
El `<Suspense>` lo ponés alrededor de una parte **lenta e independiente** que no quieras que bloquee el resto del stream (ej. una sección de comentarios o recomendaciones): así el resto de la página se pinta mientras esa isla sigue cargando. Paralelizar mata el waterfall; `<Suspense>` evita que lo lento frene a lo rápido.

**10.1** **Sí paga.** Un blog/docs es casi todo contenido estático con poca interactividad → RSC puro elimina el JS de esos componentes, dando máxima velocidad y bundles mínimos. Es el caso ideal.

**10.2** **SPA + TanStack Query** (o similar). Un editor colaborativo en tiempo real es **casi todo estado e interacción de cliente**: RSC no puede borrar ese JS (lo necesitás en el browser), así que solo agregaría boundaries y complejidad sin el premio de "menos JS". El mental model de loaders/Query además es más simple para ese tipo de app.

**10.3** Le respondo: **RSC es opcional, no obligatorio** — el propio equipo de React lo aclara; el "es el futuro" se malinterpretó como "es mandatorio". RSC paga en apps content-heavy; en apps muy interactivas es over-engineering. Migrar "todo" sin mirar el perfil de cada app es decidir por hype, no por ingeniería. Evaluamos app por app.

---

> **Para seguir.** La fuente de verdad de este módulo son los docs oficiales de React (`react.dev/reference/rsc/*`) y los del framework que uses (Next.js, TanStack Start). Antes de implementar, re-verificá lo marcado con ⚠️ — versión de React, estado de las implementaciones (TanStack, Parcel, React Router, Waku) y el comportamiento de caché de tu framework. ⚠️ Un punto todavía inmaduro: el **testing de RSC** (componentes async y Server Functions) sigue en evolución —RTL/Vitest no manejan del todo los Server Components async, así que hoy lo más sólido es cubrirlos con **E2E (Playwright)** y testear la capa de datos por separado; verificá el estado de las herramientas al momento de implementar—. El puente natural: [TanStack Start](tanstack-start.md) (el modelo client-first y el RPC tipado) y el track de backend para el lado servidor.
