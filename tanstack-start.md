# TanStack Start: full-stack React con type-safety de punta a punta

**Meta-framework sobre TanStack Router + Vite + Nitro · comparado con Next.js · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo encaja justo en tu transición **Frontend → Full Stack**: ya sabés React y venís de SPA con React; TanStack Start es lo que te deja agregar **servidor** (SSR, lógica server-only, data loading) **sin renunciar al modelo mental client-first** que ya tenés. Lo vamos a mirar siempre en contraste con **Next.js**, que es el otro gran camino, para que la elección sea una decisión informada y no una moda.

**Lo que asumimos.** React, TypeScript (cómodo con genéricos e inferencia), `async/await`, qué es SSR vs CSR, y haber tocado alguna vez **TanStack Query** (antes React Query). No asume Next.js, pero si lo conocés vas a entender más rápido los contrastes.

**Para practicar.** Node 20+, y el scaffold oficial:

```bash
npx @tanstack/cli@latest create   # CLI oficial: pregunta package manager + add-ons (Tailwind, ESLint)
# alternativa: clonar un ejemplo oficial
npx gitpick TanStack/router/tree/main/examples/react/start-basic start-basic
```

> ⚠️ **Nota sobre datos volátiles (importante en este módulo más que en ningún otro).** TanStack Start es **nuevo y se movió muchísimo en 2025-2026**: versión, estado de estabilidad, soporte de RSC, comando de scaffold y forma de configurar cambiaron varias veces. **Todo lo marcado con ⚠️ verificalo contra `tanstack.com/start` y los releases de GitHub antes de citarlo como definitivo.** Vas a encontrar tutoriales de 2024/2025 que ya están desactualizados (ver módulo 7). Las versiones de Next.js que se citan son las vigentes a mediados de 2026.

**Índice de módulos**
1. Qué es TanStack Start y qué problema resuelve
2. La decisión de fondo: client-first vs server-first (y Next.js)
3. Routing type-safe: TanStack Router por debajo
4. Server Functions: `createServerFn` y el RPC tipado
5. SSR, streaming y Selective SSR
6. Data loading: loaders isomórficos + TanStack Query
7. El motor por debajo: Vite + Nitro (y por qué murió Vinxi)
8. Implementación práctica: estructura, scaffold y deploy
9. TanStack Start vs Next.js: la comparación con criterio
10. El criterio: cuándo SÍ y cuándo NO usarlo

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es TanStack Start y qué problema resuelve

**Teoría.** **TanStack Start es un meta-framework full-stack** construido sobre tres piezas que ya son maduras por separado: **TanStack Router** (el routing type-safe), **Vite** (el bundler/dev server) y **Nitro** (el motor de servidor para el build de producción). Lo mantiene el equipo de **Tanner Linsley** — el mismo de TanStack Query, Router y Table. Te da SSR de documento completo, streaming, **server functions** (lógica server-only con RPC type-safe), y **deploy agnóstico** (Node, Cloudflare, Bun, Lambda… sin lock-in).

El problema que resuelve, para alguien que viene de SPA con React: tenés tu app client-side andando, pero necesitás **servidor** — SEO con SSR, hacer queries a la DB sin exponer credenciales, traer data antes de renderizar. Históricamente la respuesta era "migrá a Next.js", pero Next te impone su modelo mental **server-first** (todo es Server Component por defecto). TanStack Start ofrece el otro camino: **agregar servidor a tu mundo client-first**, sin cambiar de paradigma.

La frase mental: **TanStack Start es "tu SPA de React + TanStack Router, pero ahora con servidor"** — SSR aditivo, no un reemplazo del modelo cliente.

**Estado y versión** ⚠️: el **v1.0 Release Candidate** se anunció el **23-sep-2025**. El versionado interno de los paquetes (`@tanstack/react-start`) va muy avanzado (rango `1.16x.x` a mediados de 2026), pero el rótulo oficial en el sitio osciló entre "RC" y "estable" según el momento. **Tratá el estado como "estable de facto, etiqueta conservadora"**: producción-ready para early adopters, pero verificá el badge oficial antes de afirmar "1.0". Soporta hoy **React, Solid y Vue** (paquetes `react-start`, `solid-start`, `vue-start`).

**Ejercicios 1**
1.1 En una frase, ¿qué es TanStack Start y sobre qué tres piezas se construye?
1.2 ¿Qué problema concreto resuelve para alguien que ya tiene una SPA de React andando?
1.3 ¿Por qué conviene tratar con cuidado la afirmación "TanStack Start ya es 1.0 estable"?

---

## Módulo 2 — La decisión de fondo: client-first vs server-first

**Teoría.** Esta es **la diferencia más importante de todo el módulo**, y es **filosófica, no de features**. Si entendés esto, el resto cae por su propio peso.

- **Next.js (App Router) es *server-first*.** Cada componente es un **Server Component (RSC)** por defecto; vos hacés *opt-out* con `"use client"` cuando necesitás interactividad. RSC no es una herramienta que sumás: es **el paradigma alrededor del cual construís todo**. Pensás en boundaries server/client permanentemente.

- **TanStack Start es *client-first* con SSR aditivo.** El SSR es "additive to React — not a replacement". Arrancás del modelo cliente que ya conocés (componentes normales, hooks, estado) y el servidor **suma** capacidades (render inicial, server functions) sin obligarte a partir tu cabeza en dos mundos.

Esto tiene una consecuencia directa sobre **RSC** ⚠️: TanStack Start **está sumando soporte de React Server Components, pero como una *primitiva opt-in*, no como el default**. Existe un paquete dedicado (`@tanstack/react-start-rsc`, todavía `0.x` / experimental a mediados de 2026). En el modelo de Start, un RSC es **"un stream de bytes" (React Flight) — una primitiva de datos más** que producís explícitamente y consumís con los patrones que ya conocés de Query/Router. Es lo opuesto a Next, donde los RSC los tenés por default y "peleás para opt-out".

> **Honestidad obligatoria:** vas a leer artículos de 2026 que dicen "TanStack Start NO soporta RSC". Eso quedó **desactualizado** — sí hay soporte. Pero tampoco es RSC maduro: es nuevo, experimental, y filosóficamente distinto al de Next. El RSC de Next lleva **años** de battle-testing; el de Start recién arranca. **Ninguna de las dos frases extremas ("no lo soporta" / "ya es igual que Next") es cierta.**

La frase mental: **Next te hace pensar desde el servidor hacia afuera; Start te hace pensar desde el cliente hacia adentro.** No hay un ganador universal — hay un encaje según tu app (módulo 10).

**Ejercicios 2**
2.1 Explicá en tus palabras la diferencia entre "server-first" y "client-first". ¿Cuál describe a Next y cuál a Start?
2.2 ¿Por qué la frase "TanStack Start no soporta RSC" es a la vez un mito desactualizado y algo que no podés desmentir del todo? Matizalo.
2.3 Si tu app es un dashboard altamente interactivo con mucho estado en el cliente, ¿cuál de los dos paradigmas te genera menos fricción y por qué?

---

## Módulo 3 — Routing type-safe: TanStack Router por debajo

**Teoría.** El routing de Start **es** TanStack Router, y acá está su carta más fuerte: **type-safety de punta a punta en compile time**, sin que escribas anotaciones. Soporta routing **file-based** (un archivo por ruta, generado a un árbol tipado) **o** code-based.

Una ruta file-based se ve así:

```ts
import { createFileRoute } from '@tanstack/react-router'

export const Route = createFileRoute('/posts/$postId')({
  // params está TIPADO: postId: string, inferido del path
  loader: ({ params }) => fetchPost(params.postId),
  component: PostComponent,
})
```

Lo que te da gratis y que en otros routers tenés que cablear a mano:

- **Route params tipados**: si escribís `params.postID` (con D mayúscula) en vez de `params.postId`, **TypeScript te lo marca en el editor**. El árbol de rutas se genera en build.
- **Search params como ciudadanos de primera clase**: tipados, parseados/serializados, y **validables con Zod**. Esto es enorme — el estado en la URL (`?filtro=activos&page=2`) deja de ser `string` crudo que parseás a mano:

```ts
export const Route = createFileRoute('/posts')({
  validateSearch: (search) => postSearchSchema.parse(search), // Zod
  // a partir de acá, Route.useSearch() devuelve el tipo validado
})
```

- **Returns de loaders tipados**: lo que devuelve el loader, el componente lo consume con el tipo correcto vía `Route.useLoaderData()`.

La frase mental: **en TanStack Router la URL es estado tipado y validado, no un string que rezás que venga bien.**

**Ejercicios 3**
3.1 ¿Qué significa "type-safety de punta a punta en compile time" aplicado a una ruta? Dá un ejemplo de un error que el compilador atrapa.
3.2 ¿Por qué tratar los search params como tipados y validables (con Zod) es una ventaja real frente a leerlos como `URLSearchParams`?
3.3 (Implementación) Escribí una ruta `/products/$category` cuyo loader reciba `category` (tipado) y un search param `sort` validado a `'price' | 'name'` con un default.

---

## Módulo 4 — Server Functions: `createServerFn` y el RPC tipado

**Teoría.** Las **server functions** son el corazón del lado servidor de Start. Es lógica que **solo corre en el servidor** (acceso a DB, variables de entorno, filesystem) pero que **invocás desde cualquier lado** —loaders, componentes, hooks, otras server functions— como si fuera una función normal y **type-safe**.

```ts
import { createServerFn } from '@tanstack/react-start'

// Lectura simple (GET por defecto)
export const getServerTime = createServerFn().handler(async () => {
  return new Date().toISOString() // este código NUNCA llega al bundle del cliente
})

// Mutación con validación de input
export const saveUser = createServerFn({ method: 'POST' })
  .inputValidator((d: { id: string; name: string }) => d) // ⚠️ se llama inputValidator, NO validator
  .handler(async ({ data }) => {
    // data está tipado; acá tocás la DB, env, fs...
    return db.user.update(data)
  })
```

**Cómo funciona el RPC por debajo** (esto es lo elegante, y entenderlo te salva de bugs). El compilador —que es un **plugin de Vite**— genera **tres versiones** de cada server function:

1. **Handler de servidor**: el body real, vive solo en el server.
2. **Client stub**: en el bundle del cliente, el body se **reemplaza por un `fetch('/_serverFn/[id]')`** — una llamada HTTP al endpoint que expone el handler.
3. **SSR wrapper**: durante el render en servidor, hace un import dinámico del handler y lo ejecuta **local, sin round-trip HTTP**.

Resultado: escribís una función, la llamás igual desde cliente o server, y el framework resuelve el transporte. Es **type-safe sobre el límite de red** con serialización automática. La frase mental: **una server function es una función que sabés que corre en el server, pero que llamás como si fuera local — el RPC es invisible y tipado.**

> **Seguridad.** Las server functions son **RPC same-origin**, NO una API pública. Traen CSRF middleware por defecto (verifican `Origin`/`Referer`/`Sec-Fetch-Site`). Para endpoints públicos, webhooks o una API REST de verdad, usás **server routes**, no server functions.

**Contraste con Next.js** ⚠️: el equivalente de Next son las **Server Actions** (`"use server"`). Diferencia clave: las Server Actions cruzan un boundary serializado donde **el tipado se rompe** — la Action puede recibir datos que no matchean lo que TS espera, así que **necesitás validación runtime igual**. Las server functions de Start son **RPC tipado de verdad**. Es la diferencia central de type-safety entre ambos (módulo 9).

**Ejercicios 4**
4.1 ¿Qué es una server function y desde dónde la podés invocar?
4.2 Explicá las tres versiones que el compilador genera de cada server function y para qué sirve cada una.
4.3 ¿Por qué una server function NO es lo mismo que un endpoint de API pública? ¿Qué usás si necesitás un webhook?
4.4 (Trampa común) ¿Cómo se llama el método para validar el input de una server function? ¿Por qué este detalle importa?

---

## Módulo 5 — SSR, streaming y Selective SSR

**Teoría.** Start hace **SSR de documento completo** con **streaming**: el servidor manda el HTML apenas lo tiene listo y el cliente hidrata progresivamente. Hasta acá, lo esperable de un meta-framework.

Lo distintivo es **Selective SSR**: decidís el comportamiento de render **por ruta**, con la propiedad `ssr`:

```ts
export const Route = createFileRoute('/dashboard')({
  ssr: true,        // default: beforeLoad + loader + render, todo en el server
  // ssr: false,    // todo en el cliente (para APIs browser-only: localStorage, canvas, window)
  // ssr: 'data-only', // corre los loaders en el server, pero renderiza en el cliente
})
```

Incluso podés decidirlo en **runtime** con la forma funcional:

```ts
ssr: ({ params, search }) => (search.preview ? 'data-only' : true)
```

¿Por qué importa? Porque te da el **control granular** que en un modelo todo-o-nada no tenés. Una ruta que usa `localStorage` o `canvas` no puede correr en el server → `ssr: false`. Una que necesita data pero cuyo render es pesado en cliente → `'data-only'`. Esto es coherente con la filosofía client-first del módulo 2: **el SSR es una capacidad que activás donde suma, no una imposición global.**

**Ejercicios 5**
5.1 ¿Qué hace el streaming en SSR y qué problema de UX ataca?
5.2 Explicá las tres variantes de `ssr` por ruta y dá un caso de uso para cada una.
5.3 Tenés una ruta que renderiza un gráfico con una librería que usa `window`. ¿Qué valor de `ssr` le ponés y por qué?

---

## Módulo 6 — Data loading: loaders isomórficos + TanStack Query

**Teoría.** Los **loaders** de Start son **isomórficos**: corren en el **servidor** durante el request inicial (SSR) y en el **cliente** durante las navegaciones siguientes. Eso evita el "waterfall" clásico de SPA (renderizo → recién ahí pido data → muestro spinner).

El patrón ganador es **loader + TanStack Query**: prefetcheás en el loader y consumís con `useQuery` en el componente. Query maneja la hidratación, el cache, el stale-while-revalidate y las invalidaciones:

```ts
export const Route = createFileRoute('/posts')({
  // el loader asegura que la data esté en el cache de Query antes de renderizar
  loader: ({ context }) =>
    context.queryClient.ensureQueryData(postsQueryOptions()),
  component: Posts,
})

function Posts() {
  // misma queryOptions: si ya está en cache (del loader), no re-fetchea
  const { data } = useSuspenseQuery(postsQueryOptions())
  return <PostList posts={data} />
}
```

La gran ventaja sobre el modelo de Next: si ya vivís en el ecosistema **TanStack Query** (cosa probable si venís de SPA con React), este modelo te resulta **natural** — es el mismo Query que ya usás, ahora con SSR. Tenés SWR, optimistic updates e invalidación granular "gratis", y evitás el re-fetch ciego en cada navegación.

**Contraste con Next.js**: Next usa **async Server Components que hacen `fetch` directo** + el sistema de caché del framework (en Next 16, con "Cache Components" y `"use cache"` opt-in). Es más conciso para páginas content-heavy, pero es **otro modelo mental** de caché que tenés que aprender. Start reusa el que ya conocés.

**Ejercicios 6**
6.1 ¿Qué significa que un loader sea "isomórfico" y qué problema de las SPA resuelve?
6.2 En el patrón loader + Query, ¿por qué el componente no vuelve a hacer fetch si el loader ya trajo la data?
6.3 ¿Cuál es la ventaja de data loading de Start específicamente para alguien que ya usa TanStack Query en su SPA?

---

## Módulo 7 — El motor por debajo: Vite + Nitro (y por qué murió Vinxi)

**Teoría.** Acá hay un cambio histórico que **invalida casi todos los tutoriales viejos**, así que prestá atención.

**Cómo es hoy** ⚠️:
- En **desarrollo**, Start usa el **dev server de Vite directamente** (cold start de cientos de ms, no segundos).
- Para **producción**, construye con **Nitro** (el motor de servidor sobre H3), que hace el deploy **"adapterless"**: elegís un preset y deployás a cualquier target sin reescribir nada.
- Se configura con el **plugin `tanstackStart()` en `vite.config.ts`**.

```ts
// vite.config.ts (forma ACTUAL)
import { defineConfig } from 'vite'
import { tanstackStart } from '@tanstack/react-start/plugin/vite'

export default defineConfig({
  plugins: [tanstackStart()],
})
```

**Cómo era antes (y por qué importa)** ⚠️: Start originalmente usaba **Vinxi** como capa de orquestación, y se configuraba con `defineConfig` de `@tanstack/react-start/config`. Alrededor de **v1.120/1.121 (2025)** el equipo **abandonó Vinxi** y pasó a Vite directo. Esto fue un **breaking change**: la config cambió de raíz.

> ⚠️ **Las tres correcciones que te salvan de tutoriales desactualizados:**
> 1. Ya **NO se usa Vinxi** ni `@tanstack/react-start/config` → es el plugin `tanstackStart()` en `vite.config.ts`.
> 2. El scaffold actual es **`npx @tanstack/cli@latest create`**, no `create-tsrouter-app` ni `create-start-app`.
> 3. El validador de server functions es **`inputValidator`**, no `validator`.

La lección de fondo (y esto es criterio senior): **Start es nuevo, y "nuevo" significa que la documentación de terceros envejece rápido.** Cuando trabajes con tecnología joven, **la fuente de verdad son los docs oficiales y los releases**, no el primer blog que aparece en Google. Es así de fácil.

**Ejercicios 7**
7.1 ¿Qué hace Vite en Start y qué hace Nitro? ¿En qué etapa interviene cada uno?
7.2 ¿Qué significa que el deploy sea "adapterless" y qué ventaja te da?
7.3 Encontrás un tutorial que te dice de configurar Start con `defineConfig` de `@tanstack/react-start/config` y Vinxi. ¿Qué hacés y por qué?

---

## Módulo 8 — Implementación práctica: estructura, scaffold y deploy

**Teoría.** Bajemos a tierra.

**Scaffold** ⚠️:

```bash
npx @tanstack/cli@latest create   # pregunta package manager + add-ons
```

**Estructura de proyecto típica** (con la convención de sufijos para separar código por dónde puede importarse):

```
src/
├── routes/
│   ├── __root.tsx          # layout raíz (el <html>, providers globales)
│   ├── index.tsx           # ruta /
│   └── posts.tsx           # ruta /posts
├── start.ts                # createStart(): middleware global, defaultSsr, CSRF
├── router.tsx              # createRouter(): registra el árbol de rutas + queryClient
└── utils/
    ├── users.functions.ts  # server functions (createServerFn) — importables en cualquier lado
    ├── users.server.ts     # código SOLO-server (acceso a DB), importado dentro de handlers
    └── schemas.ts          # client-safe: zod, types compartidos
```

La convención de sufijos (`.functions.ts` / `.server.ts` / `.ts`) **no es obligatoria pero es buena práctica**: deja claro de un vistazo qué código puede llegar al cliente y cuál nunca. Es Screaming Architecture aplicada al boundary cliente/servidor.

**Deploy (presets de Nitro)** ⚠️: `node-server`, `cloudflare-workers`, `netlify`, `vercel`, `bun`, `railway`, y más. Cloudflare y Netlify usan plugins de Vite dedicados. Los partners oficiales destacados son **Cloudflare, Netlify y Railway**.

**Ecosistema**: ejemplos oficiales con Clerk, Convex, Supabase, WorkOS, Material UI, React Query y tRPC; e integración profunda con el resto de TanStack (Query, Form, Store, Table).

**Ejercicios 8**
8.1 ¿Para qué sirve la convención de sufijos `.functions.ts` / `.server.ts` / `.ts`? ¿Qué deja explícito?
8.2 ¿Qué rol cumple `__root.tsx`?
8.3 Tu empresa exige self-hosting on-prem (nada de Vercel, por residencia de datos). ¿El deploy de Start es un problema? ¿Por qué?

---

## Módulo 9 — TanStack Start vs Next.js: la comparación con criterio

**Teoría.** Ya tenés las piezas. Acá las juntamos en una comparación **honesta, sin hype para ningún lado**. Tabla primero, matices después.

| Dimensión | Next.js (App Router, v16) | TanStack Start (v1) |
|---|---|---|
| **Filosofía** | Server-first: todo es RSC por defecto, opt-out con `"use client"` | Client-first: SSR aditivo, el cliente decide |
| **Bundler dev** | Turbopack (default estable) | Vite (cold start más rápido) |
| **RSC** ⚠️ | Paradigma central, implícito, **maduro** | Opt-in, experimental (`react-start-rsc` 0.x), modelo distinto |
| **Lógica server** | Server Actions (`"use server"`) | Server Functions (`createServerFn`, RPC tipado) |
| **Type-safety de rutas** | Superficial (plugin de IDE para links) | **End-to-end en compile time** |
| **Search params** | `URLSearchParams` sin tipos, parseo manual | **First-class, tipados, validables (Zod)** |
| **Data fetching** | async RSC + `fetch` con caché del framework | Loaders isomórficos + TanStack Query |
| **Deploy** | Óptimo en Vercel; self-host con asteriscos | Agnóstico vía Nitro (Node, CF, Bun, Lambda) |
| **Madurez / ecosistema** ⚠️ | Enorme (~130K stars, años en producción) | Joven (v1 reciente; el ecosistema base sí es maduro) |

**Las tres diferencias que realmente deciden:**

1. **Server-first vs client-first (lo filosófico, módulo 2).** No es una feature: es cómo pensás la app entera. Next: boundaries server/client todo el tiempo, RSC maduro. Start: arrancás del cliente, el server suma. Ganás transparencia y control en Start, pero escribís más "plumbing" que en Next, donde mucho es implícito.

2. **Type-safety — acá Start es netamente superior, y no es opinión.** Routing tipado de punta a punta + server functions con RPC tipado real. Las Server Actions de Next cruzan un boundary donde el tipo se rompe y necesitás validación runtime igual. Si end-to-end type-safety es un **requisito** (no un nice-to-have), Start no tiene rival hoy.

3. **Vendor lock-in.** Next fuera de Vercel **funciona, pero con asteriscos**: ISR queda single-region, la caché multi-instancia necesita un handler compartido (Redis), image optimization consume memoria. Start **nace agnóstico** vía Nitro. Para on-prem, Cloudflare o residencia de datos, ventaja concreta de Start.

> **Mitos a desactivar** (los vas a encontrar dando vueltas):
> - "Start no soporta RSC" → desactualizado (lo soporta opt-in desde 2026), pero no es RSC maduro.
> - Las cifras de descargas/stars **varían muchísimo entre fuentes** → tratalas como orden de magnitud, no como números exactos.
> - Benchmarks de "5x throughput" a favor de Start salen de blogs/vendors, no de un estándar neutral → con pinzas.
> - El RSC de Next sigue siendo **el más maduro hoy**; el de Start es más transparente pero más nuevo y más manual.

**Ejercicios 9**
9.1 Nombrá las tres diferencias que realmente deciden entre Start y Next, y explicá una en profundidad.
9.2 ¿Por qué se dice que en type-safety Start es "netamente superior"? Dá el ejemplo concreto de Server Actions vs Server Functions.
9.3 "Next.js te encierra en Vercel." ¿Es literalmente cierto? Matizá la afirmación con dos ejemplos concretos de qué sufre Next self-hosteado.

---

## Módulo 10 — El criterio: cuándo SÍ y cuándo NO usarlo

**Teoría.** El cierre, que es lo que separa al que sigue modas del que decide con criterio. **No hay un ganador universal — hay un encaje.** El mismo principio que recorre todo: *la mejor herramienta es la que resuelve tu problema real, y la decisión se mide contra tu contexto, no contra el hype de Twitter.*

**Elegí Next.js si:**
- Sitio **content-heavy**: marketing, e-commerce, CMS, blogs — donde brillan SSG, ISR y el RSC maduro.
- Vas a deployar en **Vercel** y querés cero fricción.
- Necesitás **contratar rápido** o apoyarte en un ecosistema masivo (más gente lo sabe, más boilerplates, más respuestas en SO).
- Querés el modelo RSC más battle-tested y **estabilidad/soporte garantizados hoy**.

**Elegí TanStack Start si:**
- App **dinámica**: dashboards, SaaS, herramientas internas, admin — mucho estado en el cliente y en la URL.
- La **type-safety end-to-end** (rutas, search params, RPC) es un requisito real.
- Necesitás **deploy agnóstico** / evitar lock-in (Cloudflare, on-prem, residencia de datos).
- El equipo **ya vive en el ecosistema TanStack** (Query/Router/Table) — la curva es mínima.
- Tolerás ser **early adopter**: API que aún se mueve, menos recursos, "impuesto de productividad" las primeras semanas.

**Cuándo NO usar Start todavía** (honestidad):
- Equipo grande que necesita contrataciones y ecosistema maduros **ya**.
- Necesitás RSC maduro y battle-tested **hoy**.
- Querés docs exhaustivas y mil ejemplos copy-paste.

**El consejo que vale oro, de la propia comunidad:** **"No migres por migrar."** No reescribas un Next que funciona. La performance sola (bundles más chicos, dev server más rápido) **no justifica** un rewrite. Probá Start en un proyecto nuevo que encaje con su perfil, o migrá ruta por ruta. La frase mental de cierre: **elegí el framework por el encaje con tu app y tu equipo, no por cuál salió en el último video de YouTube.** Es así de fácil.

**Ejercicios 10**
10.1 Tu equipo arranca un **SaaS dashboard** nuevo, todos vienen de SPA con TanStack Query, y el deploy es en Cloudflare. ¿Cuál elegís y por qué?
10.2 Te toca un **e-commerce** con foco en SEO, mucho contenido estático, deploy en Vercel, y necesitás sumar 3 devs rápido. ¿Cuál elegís y por qué?
10.3 Tenés una app en Next.js que funciona bien en producción. Un compañero quiere migrarla a Start "porque es más rápido y más moderno". ¿Qué le respondés?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** TanStack Start es un meta-framework full-stack de React (también Solid y Vue) construido sobre **TanStack Router** (routing type-safe), **Vite** (bundler/dev server) y **Nitro** (motor de servidor para el build de producción).

**1.2** Le permite **agregar servidor a una SPA existente** —SSR para SEO, queries a la DB sin exponer credenciales, data loading antes del render— **sin abandonar el modelo mental client-first**. La alternativa histórica (migrar a Next) te impone su paradigma server-first.

**1.3** Porque el estado de versión es **volátil y las fuentes se contradicen**: el RC se anunció en sep-2025, el versionado interno está muy avanzado, pero el rótulo oficial osciló entre "RC" y "estable". Lo correcto es verificar el badge oficial en `tanstack.com/start` antes de afirmarlo.

**2.1** **Server-first** (Next): cada componente es Server Component por defecto; hacés opt-out con `"use client"` para interactividad — pensás desde el servidor hacia afuera. **Client-first** (Start): arrancás del modelo cliente y el SSR suma capacidades sin obligarte a partir la app en dos mundos — pensás desde el cliente hacia adentro.

**2.2** Es un **mito desactualizado** porque Start sí sumó soporte de RSC (paquete `react-start-rsc`). Pero **no podés desmentirlo del todo** porque ese soporte es nuevo, experimental (`0.x`), opt-in y filosóficamente distinto al de Next (un RSC es una primitiva de datos opt-in, no el default). Ni "no lo soporta" ni "ya es igual que Next" son ciertos.

**2.3** **Client-first (Start)** genera menos fricción: un dashboard altamente interactivo vive sobre todo en el cliente (estado, eventos, re-renders), así que el modelo donde el cliente es el punto de partida y el SSR es aditivo encaja mejor que el server-first, donde estarías marcando `"use client"` en casi todos lados.

**3.1** Significa que TypeScript conoce los tipos de params, search params y returns de loaders **en tiempo de compilación**, sin anotaciones manuales, gracias al árbol de rutas generado. Ejemplo de error atrapado: escribir `params.postID` cuando el path declara `$postId` → el editor lo marca al instante.

**3.2** Porque el estado en la URL deja de ser un `string` crudo que parseás y validás a mano (propenso a errores y sin tipos). Con search params tipados + Zod, obtenés validación en runtime **y** tipos en compile time: `Route.useSearch()` te devuelve el objeto ya validado y tipado.

**3.3**
```ts
import { createFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

const searchSchema = z.object({
  sort: z.enum(['price', 'name']).default('name'),
})

export const Route = createFileRoute('/products/$category')({
  validateSearch: (search) => searchSchema.parse(search),
  loader: ({ params }) => fetchProducts(params.category), // category: string, tipado
  component: ProductList,
})
// en el componente: const { sort } = Route.useSearch() // 'price' | 'name'
```

**4.1** Es lógica que **solo corre en el servidor** (DB, env, fs) pero invocable **desde cualquier lado** —loaders, componentes, hooks, otras server functions— como una función normal y type-safe. El framework resuelve el transporte.

**4.2** (1) **Handler de servidor**: el body real, solo vive en el server. (2) **Client stub**: en el bundle del cliente el body se reemplaza por un `fetch('/_serverFn/[id]')`. (3) **SSR wrapper**: durante el render en server, importa el handler dinámicamente y lo ejecuta **local, sin HTTP**. Así la misma llamada funciona desde cliente (vía fetch) o desde server (local).

**4.3** Porque es un **RPC same-origin** con CSRF middleware por defecto, pensado para que tu propio frontend hable con tu backend — no para exponer una API pública. Para un **webhook** (que viene de un tercero, cross-origin) o una API REST de verdad, usás **server routes**.

**4.4** Se llama **`inputValidator`** (no `validator`). Importa porque es un error común —incluso los LLMs lo equivocan— y usar el nombre viejo te da un fallo silencioso o de tipos difícil de rastrear.

**5.1** El streaming manda el HTML al cliente **apenas está listo**, en chunks, en vez de esperar a renderizar todo. Ataca el **time-to-first-byte / contenido visible**: el usuario ve algo antes, y las partes lentas (que esperan data) llegan después sin bloquear el resto.

**5.2** `ssr: true` (default): `beforeLoad`, `loader` y render, todo en el server — para contenido que se beneficia de SSR/SEO. `ssr: false`: todo en el cliente — para rutas que dependen de APIs browser-only (`localStorage`, `canvas`, `window`). `ssr: 'data-only'`: loaders en el server (data lista) pero render en el cliente — cuando querés la data prefetcheada pero el componente solo funciona client-side.

**5.3** `ssr: false` (o `'data-only'` si necesitás prefetchear data en server). La librería usa `window`, que no existe en el server, así que renderizarla server-side rompería. Con `ssr: false` el componente solo se monta en el cliente.

**6.1** "Isomórfico" = el mismo loader corre en el **server** durante el SSR inicial y en el **cliente** durante las navegaciones siguientes. Resuelve el **waterfall** de las SPA puras (renderizo → recién ahí pido data → spinner): con loaders, la data se pide **antes** del render.

**6.2** Porque el loader hace `ensureQueryData(postsQueryOptions())` —deja la data en el cache de TanStack Query— y el componente consume **las mismas `queryOptions`**. Query ve que ya está en cache (y fresca) y **no re-fetchea**: reusa lo que el loader trajo.

**6.3** Que es **el mismo TanStack Query que ya usás en tu SPA**, ahora con SSR encima. No tenés que aprender un modelo de caché nuevo (como el de Next): reusás SWR, optimistic updates, invalidaciones y `queryOptions` que ya conocés. Curva de aprendizaje mínima.

**7.1** **Vite**: el dev server (HMR rápido) y el bundler. Interviene en **desarrollo** y en el **build** del cliente. **Nitro**: el motor de servidor (sobre H3) que construye el artefacto de **producción** y lo hace deployable a cualquier target ("adapterless"). Interviene al **buildear para producción**.

**7.2** "Adapterless" = no necesitás un adapter específico por plataforma escrito a mano; elegís un **preset** de Nitro (`node-server`, `cloudflare-workers`, etc.) y el mismo código deploya a cualquier lado. Ventaja: **portabilidad sin lock-in** y sin reescribir la app para cambiar de hosting.

**7.3** **No lo seguís.** Ese tutorial está desactualizado: Vinxi y `@tanstack/react-start/config` ya **no se usan** (se abandonaron en ~v1.120). La config actual es el plugin `tanstackStart()` en `vite.config.ts`. Verificás en los **docs oficiales** (`tanstack.com/start`), que son la fuente de verdad para tecnología que se mueve rápido.

**8.1** Separa el código por **dónde puede importarse / ejecutarse**: `.server.ts` es código que **nunca** debe llegar al cliente (DB, secrets); `.functions.ts` son las server functions (invocables desde ambos lados); `.ts` es client-safe (zod, types compartidos). Deja explícito el boundary cliente/servidor de un vistazo.

**8.2** `__root.tsx` es el **layout raíz**: define el documento (`<html>`, `<head>`, `<body>`), envuelve toda la app y aloja providers globales (QueryClientProvider, etc.). Es el ancestro común de todas las rutas.

**8.3** **No es un problema, es una ventaja de Start.** Gracias a Nitro, el deploy es agnóstico: hay preset `node-server` (y otros) que te permiten self-hostear on-prem sin lock-in. Justo el escenario donde Start le gana a Next, que self-hosteado sufre con ISR single-region, caché multi-instancia e image optimization.

**9.1** Las tres: (1) **server-first vs client-first** (filosofía de render); (2) **type-safety** (Start netamente superior, end-to-end); (3) **vendor lock-in** (Start agnóstico, Next óptimo en Vercel). En profundidad, p. ej. la (1): Next te hace pensar en boundaries server/client permanentemente con RSC por default; Start arranca del cliente y suma SSR — distinto modelo mental, no distinta feature.

**9.2** Porque las **Server Functions** de Start son un **RPC tipado de verdad**: input validado, return type y contexto chequeados en compile time. Las **Server Actions** de Next cruzan un boundary serializado donde el tipo **se rompe** — la Action puede recibir datos que no matchean lo que TS espera, así que **necesitás validación runtime igual** para estar seguro. Sumado al routing tipado, Start gana la dimensión.

**9.3** **No literalmente.** Next **funciona** fuera de Vercel, pero ciertas features brillan solo ahí y self-hostearlas bien cuesta. Dos ejemplos: (a) **ISR** queda single-region y no persiste a almacenamiento durable fuera de Vercel; (b) la **caché multi-instancia** por defecto es filesystem local → con más de una instancia necesitás un cache handler compartido (Redis). No es lock-in absoluto, es "funciona con asteriscos".

**10.1** **TanStack Start.** Encaje casi perfecto: app dinámica (dashboard), el equipo ya vive en TanStack Query (curva mínima), deploy en Cloudflare (Nitro lo soporta, agnóstico). La type-safety end-to-end es un plus directo para una herramienta interna con mucho estado en URL.

**10.2** **Next.js.** E-commerce con foco SEO + mucho contenido estático = SSG/ISR/RSC maduro brillan. Deploy en Vercel = cero fricción. Necesitás sumar devs rápido = ecosistema masivo y mucha gente que ya sabe Next. Todos los factores apuntan a Next.

**10.3** Le digo: **"No migramos por migrar."** La app funciona en producción; un rewrite es costo y riesgo enormes que la performance sola **no justifica**. Si Start nos interesa, lo probamos en un **proyecto nuevo** que encaje con su perfil (algo dinámico, type-safety crítica, deploy agnóstico), no reescribiendo lo que ya anda. "Más moderno" no es un argumento de ingeniería; el encaje con el problema sí.

---

> **Para seguir.** La fuente de verdad de este módulo, dado lo volátil que es, son los docs oficiales: `tanstack.com/start`, `tanstack.com/router` y los releases en `github.com/TanStack/router`. Antes de implementar, re-verificá todo lo marcado con ⚠️ — versión, estado de RSC, comando de scaffold y forma de configurar. Es así de fácil: con tecnología joven, **leés la fuente, no el primer blog de Google.**
