# Microfrontends robustos: del concepto a producción

**Module Federation, resiliencia, aislamiento y multi-framework · ejemplos en React, Angular y Vue · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es la cara **arquitectónica** de la sección [Full Stack web (React)](tanstack-start.md): no es sobre una herramienta, es sobre **cómo partir un frontend grande entre varios equipos sin que se vuelva un caos**. El foco está en lo que casi nadie te enseña: la **robustez** — qué pasa cuando un pedazo falla, cómo aislás estilos y código, cómo versionás. El "hello world" de Module Federation lo encontrás en cualquier lado; un microfrontend que no se cae en producción, no.

**Lo que asumimos.** React, un bundler (idea de qué hacen Webpack o Vite), `npm`, `async/await`, y haber sufrido alguna vez un monorepo o un repo grande con varios equipos. Ayuda tener nociones de CI/CD.

> **¿Te falta alguna base?** Si nunca configuraste un bundler (Webpack/Vite), repasá eso primero — este módulo muestra config de bundler todo el tiempo. Si venís de una SPA y no tenés claro qué es un "build" y un "deploy" separados, leé el módulo 2 con calma: ahí está la clave de todo. Convertir el prerrequisito en rampa, no en muro.

> ⚠️ **Nota sobre datos volátiles.** El ecosistema de Module Federation se movió MUCHÍSIMO (Webpack → `@module-federation/enhanced`, Rspack, Vite, "MF 2.0"). Todo lo marcado con ⚠️ —versiones, nombres de paquetes, firmas de config— verificalo contra `module-federation.io`, `webpack.js.org` y `rspack.rs` antes de copiarlo: cambia entre minors. Las versiones citadas son las vigentes a mediados de 2026.

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere un proyecto armado).

**Índice de módulos**
1. Qué es un microfrontend y qué problema (organizacional) resuelve
2. Patrones de composición: cómo se ensamblan los pedazos
3. Module Federation: el modelo host/remote (con ejemplos)
4. El ecosistema hoy: Webpack, Rspack, Vite y "MF 2.0"
5. Shared dependencies: el problema de compartir React
6. Robustez I — resiliencia: qué pasa cuando un remote falla
7. Robustez II — aislamiento de CSS y JS
8. Routing y comunicación entre microfrontends
9. Testing, observabilidad y governance
10. El criterio: cuándo NO usar microfrontends
11. Multi-framework en el mundo real: cómo conviven React, Angular y Vue (y cómo se comunican)

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es un microfrontend y qué problema (organizacional) resuelve

**Teoría.** La definición de Martin Fowler / Cam Jackson: *"un estilo arquitectónico donde aplicaciones frontend **entregables de forma independiente** se componen en un todo mayor"*. La palabra clave es **entregables de forma independiente** — no es "dividir el código en carpetas", es dividir la **entrega**.

Pensalo como un **shopping**: para el cliente es UN solo edificio —entra, camina, compra sin notar las costuras— pero adentro cada local es **independiente**: distinto dueño, distinto equipo, cada uno renueva su vidriera y abre o cierra sin pedirle permiso al de al lado. Un microfrontend es exactamente eso: **una sola app para el usuario, pedazos independientes para los equipos**. Vamos a volver a esta imagen varias veces — porque casi todo lo difícil (qué pasa si un local cierra, que ninguno pinte el pasillo común, que todos respeten la cartelería) tiene su paralelo acá.

Y acá está lo que casi todo el mundo confunde: **el problema que resuelven NO es técnico, es organizacional.** Un frontend monolítico no se rompe por performance — se rompe por **escala de equipos**:

- Varios equipos tocando el mismo repo → conflictos de merge, releases coordinados, todos esperan al más lento.
- Releases en *lockstep*: nadie despliega hasta que todos terminan.
- No podés adoptar una herramienta nueva sin migrar todo de golpe.

Los microfrontends atacan esto con **equipos autónomos que dueñan una rebanada vertical** (UI + lógica), cada uno con su CI/CD y **deploy independiente**.

La analogía con **microservicios** es real, pero ojo con un costo que el backend no tiene: en microservicios cada servicio corre aislado en su proceso; en microfrontends **todo converge en un único navegador, un único DOM, un único runtime de JS**. Los "servicios" comparten memoria, CSS global y el usuario descarga todo. **De ahí salen casi todos los dolores** (módulos 5, 6, 7).

La frase mental: **un microfrontend es una solución a un problema de ORGANIGRAMA, no de código. Si no te duele la escala de equipos, no es tu herramienta.**

**Ejercicios 1**
1.1 🔁 ¿Cuál es la palabra clave de la definición de microfrontend, y por qué "dividir el código en carpetas" NO es un microfrontend?
1.2 🧠 ¿Por qué se dice que el problema que resuelven es organizacional y no técnico? Dá dos síntomas del monolito que atacan.
1.3 🧠 ¿Qué costo tiene el frontend que el backend (microservicios) no tiene, y por qué de ahí salen los problemas?

---

## Módulo 2 — Patrones de composición: cómo se ensamblan los pedazos

**Teoría.** La pregunta central de toda arquitectura de microfrontend es: **¿cuándo y dónde se unen las piezas?** El espectro va de build-time → server-side → runtime.

- **Build-time (paquetes npm):** cada MFE se publica como paquete npm y el contenedor lo importa como dependencia. **Suena limpio pero NO es realmente microfrontend**: para publicar un cambio tenés que re-buildear y re-desplegar el contenedor → volvés al **release en lockstep**. Es modularización, no microfrontend.
- **Server-side / Edge-side (SSI, ESI):** el servidor (o un CDN en el borde) ensambla fragmentos de HTML de distintos orígenes. Excelente para performance y SEO (el HTML llega armado), pero la interactividad rica entre fragmentos es más difícil. Encaja con stacks SSR/multipágina.
- **Client-side / runtime:** el contenedor decide en el navegador qué cargar y cuándo. Acá viven:
  - **iframes:** aislamiento brutal (CSS/JS/globals totalmente separados), pero comunicación incómoda (`postMessage`), routing y responsive sufren. El martillo cuando necesitás aislamiento duro.
  - **JavaScript en runtime:** cada MFE expone una función que el contenedor invoca. **Module Federation es la versión industrializada y moderna de esta idea** — y nuestro foco.
  - **Web Components:** cada MFE es un custom element (`<my-cart>`). Estándar del navegador, agnóstico, encapsulación vía Shadow DOM. Costo: fricción al integrar con React.
  - **Import maps:** estándar nativo para mapear módulos a URLs. Más bajo nivel, sin lock-in de tooling.

| Enfoque | Deploy independiente | Aislamiento | Complejidad |
|---|---|---|---|
| Build-time (npm) | ❌ (lockstep) | bajo | baja |
| Server-side / ESI | ✅ | medio | media |
| iframes | ✅ | máximo | baja-media |
| Module Federation | ✅ | bajo (comparte runtime) | alta |
| Web Components | ✅ | alto (Shadow DOM) | media |

La frase mental: **el eje no es "qué librería uso", es "en qué momento se unen los pedazos". Build-time te quita la independencia; runtime te la da pero te cobra en complejidad y aislamiento.**

**Ejercicios 2**
2.1 🔁 ¿Por qué la integración build-time (paquetes npm) no se considera un microfrontend de verdad?
2.2 🧠 Tu equipo necesita aislamiento total entre pedazos (que el CSS y el JS de uno jamás toquen al otro), y la comunicación entre ellos es mínima. ¿Qué enfoque de composición mirarías primero y qué costo aceptás?
2.3 🔁 ¿Cuál es la idea detrás de "JavaScript en runtime" y de qué herramienta moderna es la versión industrializada?

---

## Módulo 3 — Module Federation: el modelo host/remote (con ejemplos)

**Teoría.** Module Federation (MF) parte el mundo en dos roles:

- **Host (o shell):** la app contenedora que **consume** módulos remotos en runtime.
- **Remote:** la app que **expone** módulos para que otros los consuman.

La pieza mágica es el **`remoteEntry.js`** (el "container/manifiesto"): cada remote publica este archivo, y el host lo descarga **en runtime, no en build**. Por eso podés desplegar un remote de forma independiente y el host lo levanta **sin recompilarse**. Ese es *el* valor de MF.

**El remote expone componentes:**

```js
// remote/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container
const deps = require('./package.json').dependencies

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'cart',                  // nombre global del remote
      filename: 'remoteEntry.js',    // el manifiesto que el host descarga
      exposes: {
        './CartWidget': './src/CartWidget',  // qué módulo publica
      },
      shared: {
        react: { singleton: true, requiredVersion: deps.react },
        'react-dom': { singleton: true, requiredVersion: deps['react-dom'] },
      },
    }),
  ],
}
```

**El host declara qué remotes consume:**

```js
// host/webpack.config.js
new ModuleFederationPlugin({
  name: 'shell',
  remotes: {
    // alias  ->  nombreRemote@URL/remoteEntry.js
    cart: 'cart@http://localhost:3001/remoteEntry.js',
  },
  shared: {
    react: { singleton: true, requiredVersion: deps.react },
    'react-dom': { singleton: true, requiredVersion: deps['react-dom'] },
  },
})
```

> ⚠️ Nota de formato: con Module Federation **nativo de Webpack** el entry es `remoteEntry.js` (como en el ejemplo). Con **`@module-federation/enhanced` / Rspack** (módulo 4) el formato recomendado apunta al manifiesto: `cart@http://localhost:3001/mf-manifest.json`. El concepto es el mismo (un archivo que el host descarga en runtime); cambia el nombre/forma según el plugin.

**El host consume el remote** (lazy + dynamic import del módulo federado):

```tsx
// host/src/App.tsx
import React, { Suspense } from 'react'

const CartWidget = React.lazy(() => import('cart/CartWidget')) // "alias/exposeKey"

export default function App() {
  return (
    <Suspense fallback={<div>Cargando carrito…</div>}>
      <CartWidget />
    </Suspense>
  )
}
```

(Para que TypeScript no se queje del `import('cart/CartWidget')`, declarás el módulo en un `.d.ts`. MF 2.0 genera esos tipos automáticamente, módulo 4.)

La frase mental: **el host no "importa" el remote en build — lo descarga por su `remoteEntry.js` en runtime. Por eso cada equipo despliega su pedazo cuando quiere.**

**Ejercicios 3**
3.1 🔁 Definí "host" y "remote" en Module Federation. ¿Cuál expone y cuál consume?
3.2 🧠 ¿Por qué el hecho de que el host descargue `remoteEntry.js` en runtime es lo que habilita el deploy independiente?
3.3 ✍️ Escribí la config del `ModuleFederationPlugin` de un remote llamado `search` que exponga `./SearchBar` desde `./src/SearchBar`, con React como dependencia compartida.

---

## Módulo 4 — El ecosistema hoy: Webpack, Rspack, Vite y "MF 2.0"

**Teoría.** Acá hay mucha confusión de nombres, así que ordenemos ⚠️ (esto es de lo más volátil del módulo):

- **Webpack 5 nativo:** trae `ModuleFederationPlugin` de fábrica. Es la implementación original. Funciona, pero está atada a Webpack y le faltan las features nuevas.
- **"Module Federation 2.0" ⚠️:** **OJO — NO es una versión semántica 2.0.** Los maintainers lo aclaran: es el nombre de una **reescritura arquitectónica** (liderada por el equipo de ByteDance junto a Zack Jackson), que se materializa en el paquete **`@module-federation/enhanced`**. El release original fue en **abril de 2024**; se reportó como estable en **2026** ⚠️.
- **`@module-federation/enhanced` ⚠️:** el plugin moderno. Suma sobre lo nativo: **tipos de TypeScript automáticos** entre remotes, manifiesto `mf-manifest.json`, **runtime desacoplado del bundler** (un SDK que podés usar para cargar remotes por código), plugins de runtime y devtools. Funciona sobre Webpack y Rspack.
- **Rspack ⚠️:** soporta MF de forma first-class (vía `enhanced`). Hoy es **el camino más sólido** si querés velocidad: API compatible con Webpack pero builds mucho más rápidos.
- **Vite ⚠️:** usable, pero **con asteriscos** que tenés que conocer ANTES de comprometerte:
  - Paquete oficial: `@module-federation/vite`. (Existe también el de comunidad `@originjs/vite-plugin-federation` — no los mezcles.)
  - **Solo el host soporta dev mode con HMR**; el remote requiere `vite build` para generar el `remoteEntry`.
  - **No mezcles MFEs de Vite con MFEs de Webpack** en el mismo sistema: no hay garantía de que generen los mismos chunks → terminás con dos Reacts (módulo 5).

> Recomendación honesta para hoy: greenfield con foco en MF → **Rspack + `@module-federation/enhanced`**. Vite-MF anda, pero su asimetría dev/build y el riesgo de mezclar bundlers lo hacen menos predecible.

La frase mental: **"Module Federation 2.0" no es un número de versión: es la reescritura que vive en `@module-federation/enhanced`. Y si vas a producción hoy, Rspack es el caballo más confiable.**

**Ejercicios 4**
4.1 🔁 ¿Qué es "Module Federation 2.0" y por qué el nombre es engañoso? ¿En qué paquete vive?
4.2 🧠 Tu equipo usa Vite y otro equipo usa Webpack, y quieren federar sus MFEs juntos. ¿Qué riesgo concreto corren y por qué?
4.3 🧠 ¿Qué tres ventajas concretas da `@module-federation/enhanced` sobre el plugin nativo de Webpack?

---

## Módulo 5 — Shared dependencies: el problema de compartir React

**Teoría.** Acá está **la trampa #1 de Module Federation.** Si cada MFE bundlea su propio React, no solo el usuario descarga React N veces — lo grave es que hay **N instancias de React en memoria**. Y React guarda estado interno por módulo, así que con múltiples instancias **los hooks explotan** (`Invalid hook call`) y se rompe la hidratación.

La solución es el campo `shared`. Config defensiva para React:

```js
shared: {
  react: {
    singleton: true,        // UNA sola instancia en todo el grafo
    requiredVersion: '^18.2.0',
    strictVersion: true,    // ⚠️ tira ERROR en mismatch, en vez de cargar dos
  },
  'react-dom': { singleton: true, requiredVersion: '^18.2.0', strictVersion: true },
}
```

Qué hace cada flag:
- **`singleton: true`** — fuerza UNA sola copia en el "share scope". Si las versiones difieren, carga la más alta y **avisa** (warning). **Obligatorio para React, react-dom, react-router, tu state manager** — todo lo que tenga estado o contexto global.
- **`requiredVersion`** — el rango semver aceptable (por defecto, el de tu `package.json`).
- **`strictVersion: true`** — convierte ese warning en **error duro**. Preferís fallar en CI/deploy antes que comerte un bug fantasma de hooks en producción. **Ojo con el default**: para singletons como React, `strictVersion` viene en `false`, así que ponerlo en `true` es una **decisión deliberada** para fallar duro ante un mismatch. ⚠️ Está documentado en `webpack.js.org` pero no aparece explícito en la página `configure/shared` de `module-federation.io` — confirmá el comportamiento en tu versión.
- **`eager: true`** — mete la dep compartida en el bundle de entrada de forma síncrona (útil para el shell que necesita React al arranque), **pero infla el entry**. Usalo con criterio, no por default.

**El dilema de fondo** (y esto es criterio senior, lo dice Fowler): apenas compartís dependencias, **reintroducís acoplamiento en build** — un upgrade de React puede obligar a coordinar a TODOS los equipos a la vez, que es justo la independencia que querías evitar perder. No hay almuerzo gratis: **o duplicás (payload más grande) o acoplás (coordinación de versiones).** La gobernanza de versiones (módulo 9) es un problema organizacional, no de config.

La frase mental: **React tiene que ser `singleton` sí o sí, o los hooks se rompen. Pero compartir te devuelve algo de acoplamiento — el microfrontend nunca es 100% independiente.**

**Ejercicios 5**
5.1 🔁 ¿Qué pasa concretamente si dos MFEs cargan cada uno su propia instancia de React? ¿Qué error vas a ver?
5.2 🧠 ¿Para qué sirve `strictVersion: true` y por qué un senior preferiría "fallar en CI" antes que cargar dos versiones?
5.3 🧠 Explicá el dilema de Fowler: ¿por qué compartir dependencias reintroduce el acoplamiento que los microfrontends querían evitar?
5.4 ✍️ Escribí el campo `shared` para `react` y `react-dom` como singletons con `strictVersion`, y en un comentario explicá qué se rompe si sacás `singleton: true`.

---

## Módulo 6 — Robustez I: resiliencia, qué pasa cuando un remote falla

**Teoría.** Acá empieza lo que pediste: **la robustez.** En un microfrontend, los pedazos se cargan por red en runtime. Eso significa que **un pedazo puede fallar** — y tu trabajo es que, si "cart" se cae, el resto de la app **siga viva**. Hay **dos modos de falla distintos** y necesitás cubrir los dos:

| Falla | Cuándo | Mecanismo |
|---|---|---|
| **No carga el `remoteEntry`** | red caída, deploy roto, 404, CORS | runtime plugin / retry a nivel loader |
| **Error al renderizar** | el MFE cargó pero tira excepción | React **Error Boundary** |

### Patrón A — Error Boundary POR remote (el más importante)

```tsx
// host/src/RemoteErrorBoundary.tsx
import React from 'react'

type Props = { name: string; fallback: React.ReactNode; children: React.ReactNode }
type State = { hasError: boolean }

export class RemoteErrorBoundary extends React.Component<Props, State> {
  state: State = { hasError: false }

  static getDerivedStateFromError(): State {
    return { hasError: true }
  }

  componentDidCatch(error: Error) {
    // OBSERVABILIDAD: reportá QUÉ remote cayó (módulo 9)
    console.error(`[MFE:${this.props.name}] crash`, error)
    // sendToTelemetry({ remote: this.props.name, error })
  }

  render() {
    return this.state.hasError ? this.props.fallback : this.props.children
  }
}
```

```tsx
// host/src/App.tsx — un boundary POR CADA remote
const CartWidget = React.lazy(() => import('cart/CartWidget'))

<RemoteErrorBoundary
  name="cart"
  fallback={<div className="cart-degraded">El carrito no está disponible ahora.</div>}
>
  <Suspense fallback={<CartSkeleton />}>
    <CartWidget />
  </Suspense>
</RemoteErrorBoundary>
```

Esto garantiza que **si "cart" se cae, el shell y los demás remotes siguen funcionando.** Esa es la diferencia entre "un MFE caído" y "toda la app caída". Volvé al shopping: **un local cerrado por refacción no clausura el shopping** — el cliente sigue comprando en el resto. El Error Boundary es el cartelito de "cerrado por refacción" de ese local.

### Patrón B — retry / backoff (lo que MF NO te da de fábrica)

Para la falla de **red** (no carga el chunk), MF no trae retry por defecto. Versión mínima:

```ts
function loadWithRetry(factory: () => Promise<any>, retries = 3, delay = 500): Promise<any> {
  return factory().catch((err) => {
    if (retries === 0) throw err
    return new Promise((resolve) => setTimeout(resolve, delay))
      .then(() => loadWithRetry(factory, retries - 1, delay * 2)) // backoff exponencial
  })
}

const CartWidget = React.lazy(() => loadWithRetry(() => import('cart/CartWidget')))
```

> ⚠️ La doc oficial de MF recomienda usar el **Error Boundary de React para el caso de render** (es lo nativo) y reservar los runtime plugins (`errorLoadRemote`) para resiliencia de **red** (retry, URL de backup, cache-bust). Dos capas, dos problemas distintos.

> **Y esto es clave:** si no **testeás el camino de falla** (módulo 9), tu "resiliencia" es teórica. Simulá el remote caído y verificá que el fallback aparece y el resto sigue vivo.

La frase mental: **un microfrontend robusto asume que cada remote puede caerse. Error Boundary por remote para el render, retry/backoff para la red — y el fallback se testea, no se supone.**

**Ejercicios 6**
6.1 🔁 ¿Cuáles son los dos modos de falla de un remote y qué mecanismo cubre cada uno?
6.2 🧠 ¿Por qué conviene un Error Boundary POR remote y no uno solo que envuelva toda la app?
6.3 ✍️ Tenés un remote `recommendations` que a veces tarda o falla. Envolvelo de forma que, si se cae, el usuario vea "Recomendaciones no disponibles" pero el resto de la página siga funcionando.

---

## Módulo 7 — Robustez II: aislamiento de CSS y JS

**Teoría.** En el shopping, **ningún local puede pintar el pasillo común** ni meter su música en el local de al lado. En microfrontends, en cambio, todos los MFEs comparten el mismo DOM, el mismo `<head>` y el mismo `window`, así que **se pisan entre ellos** si no los aislás vos. Dos frentes:

### Aislamiento de CSS

| Técnica | Aislamiento | Notas |
|---|---|---|
| Naming / prefijos (`cart-button`) | Débil | Depende de disciplina; frágil |
| **CSS Modules** | Fuerte | **El default recomendado**: clases hasheadas por build, no colisionan |
| CSS-in-JS (styled/emotion) | Fuerte | ⚠️ si dos MFEs traen distinta versión de la lib, problemas |
| **Shadow DOM** | Total | El único aislamiento REAL, pero rompe portales de React, modales, theming global; y el CSS *heredable* (`color`, `font-family`) igual se filtra salvo reset |

Honestidad: **Shadow DOM es el único aislamiento de verdad**, pero rompe demasiadas cosas en una app React rica. En la práctica, la mayoría va con **CSS Modules + un design system compartido** (módulo 9) y reserva Shadow DOM para widgets verdaderamente independientes.

### Aislamiento de JS / globals

- **`window` es compartido.** Si dos MFEs escriben `window.config` o registran el mismo listener global, colisionan. Convención: namespacing (`window.__cart__`) o, mejor, **nada de globals**.
- MF aísla el **scope de módulos** (cada remote tiene su contenedor), pero **NO sandboxea el global scope**. Para eso necesitarías iframes. En la práctica: disciplina + linting.

La frase mental: **MF te aísla los módulos, pero el DOM, el `<head>` y el `window` siguen siendo de todos. El CSS y los globals son tierra compartida — protegelos con CSS Modules y cero globals.**

**Ejercicios 7**
7.1 🔁 ¿Cuál es la técnica de aislamiento de CSS recomendada por defecto, y por qué funciona?
7.2 🧠 Shadow DOM da el aislamiento más fuerte. ¿Por qué, aun así, la mayoría no lo usa para toda la app?
7.3 🧠 ¿Qué significa que "MF aísla el scope de módulos pero no el global scope"? Dá un ejemplo de colisión posible.
7.4 ✍️ Tenés un componente de un remote con una clase `.button` y el host también usa `.button`. Aislalo con CSS Modules y mostrá (en el JSX/CSS) por qué deja de colisionar.

---

## Módulo 8 — Routing y comunicación entre microfrontends

**Teoría.** Dos problemas relacionados, con un mismo principio rector: **la URL es el contrato de más bajo acoplamiento que existe.**

### Routing
- **El host es dueño del routing de alto nivel:** mapea segmentos a MFEs (ej. `/checkout/*` → MFE de checkout). El host decide *qué* MFE se monta.
- **Cada MFE es dueño de su sub-árbol de rutas** (nested routing): dentro de `/checkout/*`, el MFE maneja `/checkout/cart`, `/checkout/payment`. El host no conoce esas rutas internas.
- **Regla de oro: una ruta tiene UN SOLO dueño.** Si dos MFEs creen ser dueños de la misma ruta, es un bug de diseño.

### Comunicación
Mantra: **acoplamiento bajo o no es microfrontend.** Si los MFEs necesitan conocerse profundamente, perdiste la autonomía. Opciones, de **menos** a **más** acoplamiento (preferí las primeras):

1. **URL / routing** — estado compartido en la barra de direcciones. Declarativo, bookmarkeable, cero acoplamiento de código. La mejor para "en qué sección/recurso estamos".
2. **Custom events (event bus)** — comunicación indirecta: el emisor no sabe quién escucha. Bueno para "se agregó al carrito". Costo: contrato implícito, difícil de tipar.
3. **Props / callbacks** — el host pasa datos y funciones hacia abajo al montar el MFE. Explícito y tipado, pero acopla al host con la interfaz del MFE.
4. **Estado compartido (store global)** — el **más acoplante**. Recrea el monolito por la puerta de atrás. Si lo usás, solo para un mínimo transversal (ej. usuario autenticado), nunca para estado de dominio.

La frase mental: **preferí URL > eventos > props > estado compartido. Cada paso a la derecha es deuda de acoplamiento que pagás después. Un store global compartido es un monolito disfrazado.**

**Ejercicios 8**
8.1 🔁 ¿Quién es dueño del routing de alto nivel y quién del routing interno de cada sección?
8.2 🧠 Ordená de menor a mayor acoplamiento: estado compartido, URL, props/callbacks, custom events. Justificá los extremos.
8.3 🧠 ¿Por qué un store global compartido entre todos los MFEs es "un monolito disfrazado"?
8.4 ✍️ El host maneja `/checkout/*` y el MFE de checkout sus rutas internas. Escribí (config de router o pseudocódigo) qué ruta declara el host y qué rutas declara el MFE, marcando dónde está el límite de dueño.

---

## Módulo 9 — Testing, observabilidad y governance

**Teoría.** Lo que hace a un microfrontend **sostenible** en el tiempo (no solo que arranque).

### Testing (la integración es en runtime, así que la pirámide cambia)
1. **Unit/component por MFE:** normal, aislado, cada equipo en su repo.
2. **Contract testing (clave):** el contrato es la **interfaz del módulo expuesto** (props del componente, shape de eventos). Si el remote cambia las props de `CartWidget`, **rompe al host en runtime, sin error de compilación**. Versioná el contrato y testealo (los tipos de MF 2.0 ayudan).
3. **E2E con remotes reales** (Playwright/Cypress): la única capa que prueba la integración de verdad.
4. **Test del fallback:** bloqueá la URL del `remoteEntry` y verificá que el Error Boundary muestra el fallback. **Si no testeás el camino de falla, tu resiliencia es teórica.**

### Observabilidad
- En `componentDidCatch` y en los runtime plugins, reportá **el nombre del remote** + versión + URL a tu APM (Sentry/Datadog). Sin eso, ves "algo se rompió" pero no *cuál equipo*.
- Source maps por remote para des-minificar el stacktrace del equipo correcto.

### Governance (lo que el equipo de plataforma DEBE estandarizar)
Sin esto, microfrontends = caos distribuido:
1. **Versiones de shared deps** — política única de React/router como singletons, consistente en TODOS los repos.
2. **Design system compartido** (la pieza más importante) — componentes + tokens versionados como shared dep. Es lo que evita el Frankenstein visual **sin** Shadow DOM. Es la **cartelería unificada del shopping**: cada local es independiente, pero todos respetan la misma tipografía y señalética para que el cliente sienta que está en un solo lugar.
3. **Contrato de integración** — cómo se exponen módulos y cómo se comunican.
4. **Política de resiliencia compartida** — un loader común (con retry/telemetry) que todos usan, no copiado 12 veces.

La frase mental: **la integración es en runtime, así que el bug aparece en producción, no en el compilador. Contract testing, test del fallback y una plataforma que estandarice lo compartido — eso separa el microfrontend sostenible del caos.**

**Ejercicios 9**
9.1 🔁 ¿Por qué el contract testing es especialmente crítico en microfrontends? ¿Qué tipo de bug atrapa que el compilador no?
9.2 🧠 ¿Por qué un design system compartido es "la pieza más importante" de la governance?
9.3 🧠 En `componentDidCatch`, ¿qué información mínima tenés que reportar a tu APM y por qué?

---

## Módulo 10 — El criterio: cuándo NO usar microfrontends

**Teoría.** El cierre, y el módulo más importante de todos — porque la decisión correcta muchas veces es **no usarlos.**

**El "distributed monolith" (el peor de los mundos):** si tus MFEs comparten estado global, tienen dependencias acopladas o necesitan desplegarse juntos para no romperse, tenés **la complejidad de lo distribuido SIN la independencia**. Combinás los downsides de ambas arquitecturas. Es el resultado más común de adoptar microfrontends sin necesitarlos.

**Lo que dice Cam Jackson** (martinfowler.com): los microfrontends *"inevitablemente llevan a más cosas que gestionar — más repos, más herramientas, más pipelines, más servidores, más dominios"*. La pregunta antes de adoptarlos es si tu organización tiene *"la madurez técnica y organizacional para hacerlo sin crear caos"*.

**El paralelo con microservicios:** Fowler defiende empezar por el monolito —es su principio *MonolithFirst* (en otro artículo suyo, no en el de microfrontends, que es de Cam Jackson)—: no conviene arrancar un proyecto nuevo directamente con microservicios, aun creyendo que va a crecer lo bastante para justificarlos. Igual acá: **empezá con un monolito modular, extraé microfrontends cuando el dolor organizacional sea real y medible.** Un monolito frontend bien modularizado (workspaces, módulos por dominio, lazy routes) te da el **80% de los beneficios** (separación, code-splitting) sin el costo de runtime federado ni la coordinación de versiones.

> ⚠️ Vas a leer cifras tipo "la adopción cayó de 75% a 23%". **No tienen fuente primaria verificable** — tomalas como anécdota de desencanto, no como dato. La tendencia cualitativa (vuelta al monolito modular) sí está soportada; los números exactos, no.

**Usá microfrontends SOLO si:**
- ✅ Múltiples equipos autónomos chocando en un frontend grande.
- ✅ Necesidad real de deploys independientes por dominio.
- ✅ Madurez de CI/CD, observabilidad y plataforma YA existente.
- ❌ Un solo equipo, o app chica/mediana → **monolito modular**.
- ❌ Lo querés "para aprender" o porque "los grandes lo usan" → over-engineering.

La frase mental de cierre: **microfrontends resuelven un problema de organigrama. Si tu cuello de botella no es la escala de equipos, estás agregando sistemas distribuidos para no ganar nada. Empezá monolito modular; extraé cuando duela de verdad.**

**Ejercicios 10**
10.1 🧠 ¿Qué es un "distributed monolith" y por qué es el peor resultado posible?
10.2 🧠 Un equipo de 4 personas, una sola app de tamaño medio, quiere microfrontends "para estar a la moda". ¿Qué les recomendás y por qué?
10.3 🧠 ¿Qué te da un monolito modular bien hecho que cubre el 80% del beneficio sin el costo de los microfrontends?

---

## Módulo 11 — Multi-framework en el mundo real: cómo conviven React, Angular y Vue (y cómo se comunican)

**Teoría.** Hasta acá todo fue **React con React**. Pero el mundo real es más sucio: terminás con React, Angular, Vue y un legacy en jQuery **en la misma página**. Y ojo —esto no se elige porque sí—. Casi siempre llegás acá por una de tres razones honestas:

1. **Migración incremental** de un legacy: querés pasar de AngularJS/jQuery a React **sin un rewrite big-bang** (lo vemos al final, es el mejor caso de uso).
2. **Adquisiciones / fusiones:** dos empresas se juntan, cada una traía su stack, y de un día para el otro hay que mostrarlas en un solo producto.
3. **Equipos autónomos** que —ejerciendo la autonomía que el microfrontend les promete— eligieron frameworks distintos.

Pero antes de entusiasmarte, el criterio senior, y lo dice Luca Mezzalira (el autor canónico del tema, ex-DAZN): **no arranques un greenfield mezclando frameworks.** Multi-framework es una **capacidad de alto impuesto**, no un default. Volvé al shopping: que un local sea una hamburguesería y el de al lado una zapatería está bien; que CADA local hable un idioma distinto y use enchufes distintos es un costo que pagás todos los días. Mezclás frameworks cuando **el beneficio organizacional supera ese impuesto** —migración, adquisición, autonomía real— y lo tratás como **transición**, no como destino.

Este módulo tiene un corazón: **cómo se comunican y comparten datos pedazos hechos en frameworks distintos** (sección 11.5). Todo lo demás existe para llegar bien preparado a esa pregunta. El recorrido:

- **11.1 — el idioma común** (Web Components): cómo un framework habla con los demás.
- **11.2 — el portero** (single-spa): quién decide qué pedazo se monta y cuándo.
- **11.3 — el límite** (Module Federation entre frameworks): qué se puede federar y qué no.
- **11.4 — el costo** (Hydra of Lerna): qué pagás por sumar frameworks.
- **11.5 — 🫀 EL CORAZÓN** (comunicación y datos): cómo se hablan los pedazos.
- **11.6 — la realidad** (casos): quién lo hizo de verdad y qué aprendió.
- **11.7 — cuándo SÍ** (migración strangler fig): el caso de uso que lo justifica.

### 11.1 — Web Components: la lingua franca

Si React no entiende a Angular y Angular no entiende a Vue, **¿qué entienden todos?** La plataforma. Un **custom element** (`class X extends HTMLElement` + `customElements.define('mi-widget', X)`) es un elemento del DOM de verdad, con su ciclo de vida (`connectedCallback`, `disconnectedCallback`, `attributeChangedCallback`). Cualquier framework que renderice HTML puede **renderizarlo**; cualquiera que escuche eventos del DOM puede **escucharlo**. El contrato de interoperabilidad es **la plataforma misma** (atributos, propiedades, eventos del DOM, slots), no el modelo de componentes de ningún framework. Esa es la lingua franca.

> ⚠️ Dato volátil (mediados de 2026): en `custom-elements-everywhere.com` —el scoreboard de referencia— React, Angular, Vue, Svelte, Lit y compañía puntúan **100%**. Pero ojo: que React llegara al 100% recién pasó con **React 19** (dic 2024). Si tu host es React 18 o anterior, lo de abajo NO aplica del todo.

Pensalo como el **enchufe estándar del shopping**: cada local (cada framework) es distinto por dentro, pero todos respetan la misma toma de corriente. Esa toma es el custom element. Y tiene dos lados: **exportar** (publicar tu pedazo para que otro lo enchufe) y **consumir** (enchufar un pedazo ajeno).

**Exportar** — publicar un pedazo como custom element:

| Framework | Cómo se exporta | Soporte |
|---|---|---|
| **Angular** | `@angular/elements` + `createCustomElement(Component, { injector })`. Los `@Input()` se mapean a atributos dash-case; los `@Output()` emiten un `CustomEvent` con los datos en `event.detail` | de primera |
| **Vue 3** | `defineCustomElement(MiComp)` devuelve un constructor de `HTMLElement` listo para `customElements.define` | de primera |
| **React** | NO tiene API nativa: subclaseás `HTMLElement` a mano y montás con `createRoot(...).render(<App/>)` dentro del `connectedCallback` | artesanal ⚠️ |

(El caso React es el más áspero: además de manual, la delegación de eventos de React se complica dentro de un Shadow DOM.)

**Consumir** — enchufar un custom element ajeno. Acá el dolor histórico fue React:

| Framework | Cómo se consume |
|---|---|
| **React 19** | Si la prop coincide con una propiedad del elemento, la asigna como **propiedad** (pasan objetos y arrays bien); los eventos custom se escuchan con prefijo `on` (`onsay-hi={...}` escucha `say-hi`). Antes de React 19 serializaba las props no-string a string (`[1,2,3]` → `"1,2,3"`) y no podías bindear eventos de forma declarativa |
| **Angular** | `CUSTOM_ELEMENTS_SCHEMA` habilita usar elementos custom (dash-case) en el template sin que Angular se queje. En la práctica, con `[prop]="x"` setea la propiedad del DOM (así pasás objetos limpio) — comportamiento práctico, no documentado como tal; conviene aislar el elemento en un wrapper |
| **Vue** | `compilerOptions.isCustomElement` para que el compilador no trate a `<mi-widget>` como un componente Vue |

**Shadow DOM, el aislamiento que ya viste en el módulo 7**, ahora en clave cross-framework: te da scoping de CSS real entre equipos, pero las propiedades heredables (`color`, `font-family`) igual se filtran, y un design system global **no puede entrar**... salvo por un canal: **las CSS custom properties (variables CSS) SÍ atraviesan el límite del Shadow DOM**. Por eso los **design tokens como variables CSS** son el puente recomendado para mantener coherencia visual sin romper el aislamiento.

### 11.2 — single-spa: el orquestador agnóstico de frameworks

Module Federation resuelve "cómo cargo código de otro equipo", pero deja una pregunta abierta: **¿quién es el portero que decide qué pedazo se monta en cada ruta —y lo desmonta cuando te vas?** En el shopping es la administración: decide qué local abre según en qué pasillo estás. En microfrontends multi-framework ese portero es **single-spa**: un orquestador top-level que gestiona el **ciclo de vida** y el **routing** de apps independientes, varias en la misma página, sin refresh.

Cada app expone funciones de ciclo de vida asíncronas:
- **`bootstrap`** (una vez), **`mount`** (cuando su ruta se activa), **`unmount`** (cuando se desactiva — **acá limpiás DOM y listeners**, clave para no leakear).

El root config registra apps y dice cuándo están activas:

```js
// root-config.js
import { registerApplication, start } from 'single-spa'

registerApplication({
  name: '@org/navbar-react',
  app: () => import('@org/navbar-react'),
  activeWhen: () => true,            // siempre montada
})
registerApplication({
  name: '@org/checkout-angular',
  app: () => import('@org/checkout-angular'),
  activeWhen: ['/checkout'],         // solo en /checkout
})
registerApplication({
  name: '@org/dashboard-vue',
  app: () => import('@org/dashboard-vue'),
  activeWhen: ['/dashboard'],
})
start()
```

single-spa observa la URL y monta/desmonta cada app según su `activeWhen`. Si dos activeWhen son verdaderos a la vez, **las dos apps se montan juntas** (un navbar React + una página Angular conviviendo). Cada framework se envuelve con su adapter: `single-spa-react`, `single-spa-angular`, `single-spa-vue`.

**¿single-spa o Module Federation?** No es "o" — son **complementarios**: single-spa **orquesta** (routing + ciclo de vida), MF **comparte código**. El setup recomendado de single-spa usa **import maps** para que React/Angular/Vue se descarguen **una sola vez** y las apps los compartan. La doc oficial pide elegir **import maps O module federation** para las deps compartidas, no mezclar los dos mecanismos.

### 11.3 — Module Federation entre frameworks: qué SÍ y qué NO

Pregunta del millón: **¿puedo federar un remote Vue dentro de un host React?** Respuesta honesta: podés **cargarlo** (`loadRemote('vueRemote/x')` te devuelve el módulo), pero **los componentes NO interoperan**. ¿Por qué? Cada framework mantiene su propio **árbol interno** de componentes y un **motor** que lo sincroniza con el DOM — en React ese árbol son las *fibras* y el motor es el *reconciliador*; Vue tiene los suyos. Son **dos cerebros que no comparten neuronas**: `react-dom` no sabe renderizar un componente Vue, y al revés igual. En el shopping: dos locales con cajas registradoras incompatibles. Si intentás bridgear ingenuamente, **con cada re-render del host el componente Vue pierde su estado**. (Esto es razonamiento sobre cómo funcionan los runtimes, no una cláusula de la doc de MF.)

Y dos cosas que se rompen seguro:
- **NO podés compartir `react` como `singleton` entre frameworks.** El `shared` deduplica dentro de la **misma familia**; Angular y Vue ni siquiera importan React. (El código compilado de Angular usa APIs internas donde el semver ni aplica.)
- **Los dos runtimes se cargan igual.** Host React + remote Angular = los dos runtimes bootstrappeados en la misma pestaña.

**El patrón que SÍ funciona:** que el remote no exponga un componente crudo, sino una **función de montaje agnóstica de framework**. El contrato es: el host te pasa **un nodo del DOM + props**, y el remote monta SU propio framework adentro y te devuelve un `unmount`. Module Federation oficializó este patrón con el nombre **Bridge** (`createBridgeComponent` / `createRemoteAppComponent`, con un lifecycle `render`/`destroy`); lo de abajo es la versión a mano del mismo concepto, para que veas el mecanismo sin magia.

```ts
// remote Vue — expone una función de montaje, no un componente Vue
import { createApp } from 'vue'
import Widget from './Widget.vue'

export function mount(el: HTMLElement, props: Record<string, unknown> = {}): () => void {
  const app = createApp(Widget, props)
  app.mount(el)
  return () => app.unmount()   // el host llama esto al desmontar
}
```

```tsx
// host React — monta el remote agnóstico en un nodo que él controla
import { useEffect, useRef } from 'react'
import { loadRemote } from '@module-federation/enhanced/runtime'

type RemoteModule = { mount: (el: HTMLElement, props?: Record<string, unknown>) => () => void }

export function VueIsland(props: { userId: string }) {
  const ref = useRef<HTMLDivElement>(null)
  useEffect(() => {
    let unmount: (() => void) | undefined
    loadRemote<RemoteModule>('vueRemote/Widget').then((mod) => {
      if (ref.current && mod) unmount = mod.mount(ref.current, props)
    })
    return () => unmount?.()    // limpieza: React desmonta → Vue desmonta
  }, [props])
  return <div ref={ref} />
}
```

Fijate el patrón: **React es dueño del nodo del DOM, Vue es dueño de lo que pasa adentro**, y el límite es un contrato chico (`mount(el, props) => unmount`). Sin pelearse por el árbol de fibras.

### 11.4 — El costo real: N runtimes en memoria ("Hydra of Lerna")

Mezzalira tiene un nombre para el anti-patrón: **"Hydra of Lerna"** — la hidra mitológica a la que, por cada cabeza que cortabas, le crecían dos. Acá es igual: **cada framework que sumás hace crecer DOS costos a la vez, bundle y memoria.** Los frameworks **no están diseñados para bootstrappearse juntos en la misma pestaña**: en el shopping sería tener **tres generadores eléctricos prendidos** para abrir un solo local. Cargar React + Angular + Vue significa:

- ⚠️ **Payload (cifras volátiles, verificá en `bundlephobia.com` con fecha):** React + ReactDOM ≈ 45KB gz **en React 18** (ojo: en React 19 `react-dom` se reestructuró y bundlephobia lo reporta como un re-export delgado de ~1.4KB, que NO es comparable; el runtime real en el browser sigue pesando lo suyo), Vue ≈ 42-45KB gz (full build 3.5.x; el runtime-only es menor), Angular bastante más (es multi-paquete). En conjunto, **140KB+ gz de runtime ANTES de una sola línea de tu producto**.
- **N reconciliadores en memoria**, cada uno con su loop de render y su sistema de reactividad.

Mitigaciones: **compartir el runtime dentro de cada familia** (un solo React para todos los MFEs React), **lazy-load** de las islas de otro framework (no cargues Angular hasta que el usuario pise `/checkout`), y **limitar la diversidad** (dos frameworks no son tres). La regla de Geers es contundente: *"Avoid Micro Frontends Anarchy"*.

### 11.5 — Comunicación y datos entre frameworks (el corazón del módulo)

Acá está lo que casi nadie te explica bien. **¿Cómo le pasa datos un MFE React a uno Angular?** Y la primera respuesta es entender qué **NO** podés hacer:

> **No podés compartir un React Context con Angular. Punto.** Un Context de React, la inyección de dependencias de Angular, el `provide/inject` de Vue — **son construcciones internas del runtime de cada framework**. El Context de React vive en SU árbol de fibras; Angular no tiene forma de verlo. Dos runtimes bootstrappeados por separado **no comparten ningún grafo de objetos.**

Entonces, ¿qué comparten? **Solo la plataforma del navegador**: el DOM, `window`, los `CustomEvent`, la **URL**, el **web storage** y `BroadcastChannel`. Eso es todo lo que tienen en común. Y de ahí sale el mantra que repiten Mezzalira, Geers y la propia doc de single-spa:

> **NO COMPARTAS ESTADO, COMPARTÍ EVENTOS.**

Igual que en el módulo 8, ordenamos los mecanismos de **menor a mayor acoplamiento** — pero ahora el filtro es más duro, porque tiene que funcionar **entre frameworks distintos**:

| Mecanismo | Acoplamiento | Cross-framework | Para qué |
|---|---|---|---|
| **URL / routing** | mínimo | ✅ nativo | "en qué sección/recurso estamos" |
| **CustomEvents (event bus)** | bajo | ✅ nativo | "pasó algo" (se agregó al carrito, se deslogueó) |
| **Web storage + `storage`/`BroadcastChannel`** | bajo-medio | ✅ nativo | estado persistente o entre pestañas |
| **Props/callbacks en el límite de montaje** | medio | ✅ (vía mount function) | datos tipados que el host baja al remote |
| **Store global compartido (Redux/NgRx)** | máximo | ❌ ni siquiera funciona bien | — (anti-patrón, ver abajo) |

**El caballo de batalla es el event bus sobre `CustomEvent`** porque es lo único verdaderamente común y desacoplado: el emisor **no sabe quién escucha**. Es el **sistema de altavoces del shopping**: un local anuncia "promoción en caja 3" y lo escucha quien quiera, sin que el que habla sepa quién está oyendo. Veámoslo con código — un bus chiquito, tipado, que vive en el `window`:

```ts
// shared/event-bus.ts — agnóstico de framework. Vive sobre window.
// El "contrato" es el NOMBRE del evento + el SHAPE del detail.
type MfeEventMap = {
  'cart:add': { sku: string; qty: number }
  'auth:logout': { reason: string }
}

export function emit<K extends keyof MfeEventMap>(type: K, detail: MfeEventMap[K]): void {
  window.dispatchEvent(new CustomEvent(type, { detail }))
}

export function on<K extends keyof MfeEventMap>(
  type: K,
  handler: (detail: MfeEventMap[K]) => void,
): () => void {
  const listener = (e: Event) => handler((e as CustomEvent<MfeEventMap[K]>).detail)
  window.addEventListener(type, listener)
  return () => window.removeEventListener(type, listener) // devolvés el "off" para limpiar
}
```

Ahora **el mismo evento `cart:add`, emitido desde React y escuchado desde Angular y Vue al mismo tiempo:**

```tsx
// MFE en REACT — emite
import { emit } from 'shared/event-bus'

export function AddToCart({ sku }: { sku: string }) {
  return <button onClick={() => emit('cart:add', { sku, qty: 1 })}>Agregar</button>
}
```

```ts
// MFE en ANGULAR — escucha (y limpia en ngOnDestroy)
import { Component, OnDestroy, signal } from '@angular/core'
import { on } from 'shared/event-bus'

@Component({ selector: 'cart-badge', template: `🛒 {{ count() }}` }) // standalone es el default desde Angular 19
export class CartBadge implements OnDestroy {
  count = signal(0)
  private off = on('cart:add', ({ qty }) => this.count.update((n) => n + qty))
  ngOnDestroy(): void { this.off() } // sin esto, leak: el listener sobrevive al desmontaje
}
```

```vue
<!-- MFE en VUE — escucha (y limpia en onUnmounted) -->
<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import { on } from 'shared/event-bus'

const count = ref(0)
const off = on('cart:add', ({ qty }) => { count.value += qty })
onUnmounted(off) // misma disciplina de limpieza que Angular
</script>

<template>🛒 {{ count }}</template>
```

**Lo que tenés que ver acá:** React no sabe que existen Angular ni Vue. Solo grita "`cart:add`" al `window`. Los otros dos, cada uno en su runtime, **escuchan el mismo evento del navegador** y reaccionan con SU propio sistema de reactividad (signals en Angular, refs en Vue). **Cero acoplamiento de código entre ellos.** Y fijate la disciplina repetida: **siempre limpiás el listener al desmontar** (`ngOnDestroy`, `onUnmounted`, el cleanup de `useEffect`) — si no, en un orquestador como single-spa que monta y desmonta apps, los listeners se acumulan y tenés un leak.

**Tres advertencias finales sobre datos compartidos:**

1. **El `detail` es un contrato implícito sin tipos compartidos.** Ese `MfeEventMap` tipado es lindo en el código fuente, pero **en runtime no hay nada que lo valide**: cada MFE compila por separado, así que en el navegador el evento es solo un string y un objeto. Si el emisor cambia el shape del `detail`, **rompe a los oyentes en runtime, sin error de compilación** — exactamente el problema de contract testing del módulo 9, recargado. **Versioná y documentá el shape de cada evento.**
2. **La URL es tu mejor amiga.** Para "qué producto/sección estamos viendo", no inventes un evento: ponelo en la URL. Es declarativo, bookmarkeable, lo lee cualquier framework con `location`, y es el contrato de **menor** acoplamiento que existe.
3. **El store global compartido es el anti-patrón, recargado.** Compartir un Redux entre React y Angular no solo recrea el monolito (módulo 8): es que **ni siquiera funciona bien**, porque Angular no se entera de los cambios del store de React sin pegamento manual. single-spa lo dice explícito: **evitá librerías de estado global entre MFEs.** Si necesitás un mínimo transversal (ej. el usuario logueado), pasalo por evento + storage, no por un store compartido.

### 11.6 — Casos del mundo real (con honestidad)

Teoría linda, pero **¿quién hizo esto de verdad?** Y, más importante, **¿qué aprendieron?** (Ojo: varios de estos posts son históricos, de 2017-2021 — léelos como "qué decidieron entonces", no como "el estado del arte hoy".)

- **Spotify — y por qué ABANDONÓ la idea en la web.** En desktop, cada página era una app aislada en su propio **iframe**. Cuando llevaron eso a la web, lo **descartaron**: cada iframe traía su propio JS/CSS sin deps compartidas → *"long load times"* y *"demasiado complejo para el web player"*. Migraron a una SPA React+Redux y en 2021 unificaron el desktop sobre esa misma base. **La lección:** el aislamiento total (iframes) se paga en performance; a veces el monolito moderno gana.
- **IKEA — composición en el servidor.** Gustaf Nilsson Kotte: modelo de "páginas y fragmentos" compuestos principalmente con **Edge Side Includes (ESI)** — los servicios exponen HTML listo y el consumidor lo transcluye. Server-side, no runtime JS.
- **Zalando — la evolución honesta.** Empezaron con **Mosaic/Tailor** (un servicio que streamea layout componiendo `<fragment src=...>`). Para 2018 los fragmentos les dieron **inconsistencia de UX** y una barrera de entrada alta → los reemplazaron por su **Interface Framework** (Entities + Renderers). **La lección:** una arquitectura de MFE puede ser un paso evolutivo que después reemplazás.
- **DAZN — Mezzalira y el "decisions framework".** Split **vertical** (un equipo dueño de una vista entera como SPA independiente), el app shell **carga UN MFE a la vez** (no varios frameworks juntos en pantalla — esquivan la Hydra), routing en el borde con CloudFront + Lambda@Edge, estado por APIs de bootstrap. Principio: **"duplicación antes que abstracción"**.
- **American Express — One App** (open source): SSR en Node + composición de módulos vía un registro central (un `module-map.json`). En producción interna desde ~2016, open-source desde ~2019. **La lección:** los microfrontends también viven en el server, no solo en el browser.

### 11.7 — El mejor caso de uso honesto: migración incremental (strangler fig)

Si te quedás con UNA sola razón para mezclar frameworks, que sea esta. El patrón **strangler fig** (Martin Fowler, 2004) viene de la higuera estranguladora: la enredadera crece alrededor del árbol hasta sostenerse sola y el árbol original muere. Aplicado al software: **reemplazás el sistema legacy pieza por pieza**, mientras seguís entregando features — en vez del **big-bang rewrite**, que *"se cae en llamas la mayoría de las veces"*.

Los microfrontends son la herramienta perfecta para hacerlo **en el frontend**: ponés un shell (o single-spa) que enruta a "lo viejo o lo nuevo", y vas **estrangulando** el AngularJS/jQuery legacy una ruta a la vez, reemplazándola por React, **sin pedirle a nadie que pare a reescribir todo**. Casos documentados:

- **Canopy Tax:** migrando de AngularJS a React en la misma página fue **lo que dio origen a single-spa**. Lo más difícil: lograr que la SPA legacy **limpie todo en su `unmount`**.
- **Smartly.io:** de AngularJS a React vía un "bridge component"; llegaron a **más del 41% en React** sin rewrite, porque el rewrite era *"demasiado riesgoso"*.

> ⚠️ **El riesgo que nadie te cuenta:** el **"in-between permanente"** — la migración que **nunca termina** y te quedás manteniendo DOS frameworks para siempre (lo peor de ambos mundos, de nuevo el "distributed monolith" del módulo 10). Sam Newman, en *Monolith to Microservices*, insiste en lo mismo: planificá la **baja del legacy** y ponele fecha. Si no, no estás migrando: estás acumulando.

La frase mental de cierre: **entre frameworks no compartís objetos, compartís la plataforma — eventos, URL, storage. Web Components y single-spa te dejan convivir; el event bus te deja comunicar; pero el multi-framework es un impuesto que solo vale la pena para migrar, fusionar o por autonomía real. Tratalo como un puente, no como una casa.**

**Ejercicios 11**
11.1 🔁 ¿Qué es un custom element y por qué se lo llama la "lingua franca" entre frameworks? Nombrá cómo exporta uno Angular y cómo lo arregló React 19 para consumirlos.
11.2 🧠 ¿Qué hace single-spa y qué hace Module Federation? ¿Por qué se dice que son **complementarios** y no alternativas?
11.3 🧠 ¿Por qué federar un componente Vue dentro de un host React "no interopera", y cuál es el patrón que SÍ funciona para montarlo?
11.4 🧠 ¿Qué es la "Hydra of Lerna" y qué dos costos concretos pagás al cargar React + Angular + Vue en la misma pestaña? Dá una mitigación.
11.5 🔁 ¿Por qué no podés compartir un React Context con un MFE hecho en Angular? ¿Qué es lo ÚNICO que comparten dos frameworks bootstrappeados por separado?
11.6 🧠 Explicá el mantra "no compartas estado, compartí eventos". ¿Por qué un store global (Redux) compartido entre React y Angular es peor todavía que entre dos MFEs React?
11.7 ✍️ Escribí un event bus tipado y agnóstico de framework sobre `window`: una función `emit(type, detail)` y una `on(type, handler)` que devuelva una función para **dar de baja** el listener. Tipá los eventos con un mapa.
11.8 ✍️ Usando ese bus, tenés un MFE en React que emite `auth:logout` con `{ reason: string }` y un MFE en Vue que muestra un cartel cuando eso pasa. Escribí el emisor (React) y el oyente (Vue), incluyendo la **limpieza** del listener.
11.9 🧠 Vas a migrar un frontend AngularJS legacy a React con strangler fig. ¿Cuál es el principal riesgo del patrón y cómo lo evitás?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** La palabra clave es **"entregables de forma independiente"**. Dividir el código en carpetas no es microfrontend porque seguís teniendo **un solo build y un solo deploy**: no ganás entrega independiente, solo organización interna.

**1.2** Porque el monolito frontend no se rompe por performance sino por **escala de equipos**. Síntomas: (a) varios equipos en el mismo repo con conflictos de merge y releases coordinados (lockstep); (b) no poder adoptar una herramienta/framework nuevo sin migrar todo de golpe.

**1.3** Que **todo converge en un único navegador: un solo DOM, un solo `window`, un solo runtime de JS**. En microservicios cada servicio corre aislado en su proceso; acá los pedazos comparten memoria y CSS global. De esa convergencia salen los problemas de instancias duplicadas de React, colisión de CSS y de globals.

**2.1** Porque para publicar un cambio en un MFE-paquete tenés que **re-buildear y re-desplegar el contenedor** que lo importa → volvés al release en lockstep. Perdiste la entrega independiente, que es la definición misma de microfrontend. Es modularización, no microfrontend.

**2.2** **iframes.** Dan el aislamiento más fuerte (CSS, JS y globals totalmente separados) y, si la comunicación es mínima, su mayor costo (la comunicación incómoda vía `postMessage`) casi no te pega. Aceptás también routing/historial y responsive más complicados.

**2.3** La idea: cada MFE **expone una función** (o módulo) que el contenedor **invoca en runtime** para montarlo. Su versión industrializada moderna es **Module Federation**.

**3.1** **Host**: la app contenedora que **consume** módulos remotos en runtime. **Remote**: la app que **expone** módulos para que otros los usen. El remote expone (`exposes`), el host consume (`remotes`).

**3.2** Porque el host no tiene el código del remote "horneado" en su build: lo **descarga por la red en runtime** desde el `remoteEntry.js`. Entonces el remote puede re-desplegarse con cambios y el host los levanta en la próxima carga **sin recompilarse**. Build acoplado = lockstep; carga en runtime = independencia.

**3.3**
```js
const { ModuleFederationPlugin } = require('webpack').container
const deps = require('./package.json').dependencies

new ModuleFederationPlugin({
  name: 'search',
  filename: 'remoteEntry.js',
  exposes: {
    './SearchBar': './src/SearchBar',
  },
  shared: {
    react: { singleton: true, requiredVersion: deps.react },
    'react-dom': { singleton: true, requiredVersion: deps['react-dom'] },
  },
})
```

**4.1** Es el nombre de una **reescritura arquitectónica** de Module Federation (tipos automáticos, manifest, runtime desacoplado del bundler, plugins). El nombre es engañoso porque **NO es la versión semántica 2.0** — es un término de marca. Vive en el paquete **`@module-federation/enhanced`**.

**4.2** Riesgo concreto: Vite (Rollup) y Webpack **no garantizan generar los mismos chunks** (sobre todo para CommonJS), así que el mecanismo de `shared` puede fallar y terminás con **dos instancias de React** en runtime → `Invalid hook call`. Por eso la recomendación es no mezclar bundlers en el mismo grafo federado.

**4.3** (1) **Tipos de TS automáticos** entre remotes (vía `mf-manifest.json`/DTS). (2) **Runtime desacoplado del bundler**: podés cargar remotes por código con un SDK, lo que habilita resiliencia (retry, fallback). (3) **Plugins de runtime + devtools** (y preloading). Todo eso sobre Webpack y Rspack.

**5.1** Cada MFE descarga su propia copia de React (payload más grande) y, lo grave, hay **múltiples instancias de React en memoria**. Como React guarda estado interno por módulo, los hooks fallan: vas a ver **`Invalid hook call`** y se rompe la hidratación.

**5.2** `strictVersion: true` convierte el warning de versión incompatible en un **error duro**. Un senior prefiere fallar en CI/deploy porque un mismatch de versiones de React en runtime produce **bugs fantasma de hooks** dificilísimos de debuggear en producción — mejor que reviente temprano y visible.

**5.3** Porque para que el `shared` funcione (no duplicar React), los MFEs tienen que **ponerse de acuerdo en versiones compatibles**. Entonces un upgrade mayor de React obliga a **coordinar a todos los equipos a la vez** — que es exactamente el release en lockstep que los microfrontends querían eliminar. O duplicás (payload) o acoplás (coordinación): no hay almuerzo gratis.

**5.4**
```js
shared: {
  react: { singleton: true, requiredVersion: '^18.2.0', strictVersion: true },
  'react-dom': { singleton: true, requiredVersion: '^18.2.0', strictVersion: true },
  // Sin `singleton: true`, cada remote podría cargar SU propia instancia de React.
  // Con varias instancias en memoria, los hooks fallan (`Invalid hook call`) porque
  // React guarda estado interno por módulo. singleton fuerza una sola copia compartida.
}
```
El punto: `singleton` garantiza UNA instancia; `strictVersion` hace que un mismatch de versiones falle en CI en vez de romper silencioso en runtime.



**6.1** (1) **No carga el `remoteEntry`** (red, 404, CORS, deploy roto) → se cubre con retry/backoff o runtime plugin a nivel loader. (2) **Error al renderizar** (el MFE cargó pero tira excepción) → se cubre con un **React Error Boundary**.

**6.2** Porque un Error Boundary **por remote aísla la falla**: si "cart" revienta, solo se muestra el fallback de "cart" y el resto del shell y los demás remotes siguen vivos. Un único boundary que envuelva toda la app haría que un MFE caído **tire abajo la página entera** — justo lo que queremos evitar.

**6.3**
```tsx
const Recommendations = React.lazy(() => import('recommendations/Widget'))

<RemoteErrorBoundary
  name="recommendations"
  fallback={<div>Recomendaciones no disponibles</div>}
>
  <Suspense fallback={<div>Cargando…</div>}>
    <Recommendations />
  </Suspense>
</RemoteErrorBoundary>
```
El Error Boundary captura el error de render y muestra el fallback; el `<Suspense>` cubre la carga. El resto de la página, fuera de este boundary, sigue funcionando.

**7.1** **CSS Modules.** Funciona porque genera **nombres de clase hasheados y únicos por build**, así dos MFEs que usen `.button` no colisionan: cada uno termina con algo como `.button_a3f9`. Aislamiento fuerte con costo casi nulo.

**7.2** Porque Shadow DOM, al encapsular totalmente, **rompe demasiadas cosas de una app React rica**: portales (modales, tooltips, dropdowns que se montan en `document.body`), librerías que inyectan en `document.head`, y el theming global. Además el CSS heredable (`color`, `font-family`) igual se filtra salvo que lo resetees. Por eso se reserva para widgets realmente independientes.

**7.3** Significa que MF le da a cada remote su propio **contenedor de módulos** (sus imports no se mezclan), pero **el `window` y el DOM siguen siendo globales y compartidos**. Ejemplo de colisión: dos MFEs que hagan `window.config = ...` se pisan, o que registren el mismo `customElements.define('x-foo')` → error de "ya definido".

**7.4**
```tsx
// remote/CartButton.module.css
.button { background: rebeccapurple; }

// remote/CartButton.tsx
import styles from './CartButton.module.css'
export function CartButton() {
  return <button className={styles.button}>Comprar</button>
}
```
Por qué deja de colisionar: CSS Modules **hashea** el nombre de clase en build, así `.button` del remote termina como algo único (`.CartButton_button_a3f9`) y `.button` del host como otro hash distinto. Aunque ambos se llamen "button" en el código, en el DOM final nunca coinciden → no se pisan.



**8.1** El **host/shell** es dueño del routing de **alto nivel** (qué MFE se monta en cada segmento, ej. `/checkout/*` → MFE checkout). Cada **MFE** es dueño de su **routing interno** (`/checkout/cart`, `/checkout/payment`), que el host no conoce.

**8.2** De menor a mayor acoplamiento: **URL < custom events < props/callbacks < estado compartido**. La URL es el menor porque es declarativa y no acopla código (solo "dónde estamos"). El estado compartido es el mayor porque un store común entre MFEs los hace dependientes del shape del estado de los demás — recrea el monolito.

**8.3** Porque un store global compartido hace que cada MFE **dependa del shape del estado de los demás** y de cómo lo mutan. Eso reintroduce el acoplamiento fuerte que el microfrontend quería romper: ya no podés cambiar un MFE sin entender y no romper a los otros → es un monolito, solo que ahora también distribuido (lo peor de los dos).

**8.4**
```tsx
// HOST — dueño del routing de alto nivel. Mapea el segmento a un MFE,
// pero NO conoce las rutas internas del checkout.
<Routes>
  <Route path="/" element={<Home />} />
  <Route path="/checkout/*" element={<CheckoutRemote />} />  {/* el /* delega todo lo de adentro */}
</Routes>

// MFE CHECKOUT — dueño de su sub-árbol. El host no sabe que estas rutas existen.
<Routes>
  <Route path="cart" element={<Cart />} />        {/* -> /checkout/cart */}
  <Route path="payment" element={<Payment />} />  {/* -> /checkout/payment */}
</Routes>
```
El **límite de dueño** está en el `/*`: el host declara `/checkout/*` (es dueño de "qué MFE va acá"); de la barra para adentro (`cart`, `payment`) el dueño es el MFE de checkout. Una ruta, un solo dueño.



**9.1** Porque la integración es **en runtime**: si un remote cambia las props o el shape de lo que expone, **rompe al host cuando se cargan juntos en producción, sin ningún error de compilación** (se buildearon por separado). El contract testing atrapa justo ese desajuste de interfaz que el compilador no puede ver porque nunca vio los dos a la vez.

**9.2** Porque es lo que garantiza **consistencia visual y de comportamiento** entre MFEs de equipos distintos **sin** recurrir a Shadow DOM. Un design system compartido (componentes + tokens, versionado como shared dep) evita que cada equipo reinvente botones y que la app se vea como un Frankenstein — resuelve el problema de cohesión que el aislamiento por sí solo no resuelve.

**9.3** Mínimo: **el nombre del remote** (qué equipo/MFE falló), su **versión** y la **URL del entry**, además del error/stack. Por qué: sin el nombre del remote, en producción ves "algo se rompió" pero no podés saber **cuál equipo** tiene que arreglarlo — perdés la trazabilidad que justamente necesitás en un sistema con muchos dueños.

**10.1** Un "distributed monolith" es cuando tenés MFEs que **igual están acoplados** (comparten estado global, dependencias acopladas, o necesitan desplegarse juntos para no romperse). Es el peor resultado porque pagás **toda la complejidad de un sistema distribuido** (más repos, pipelines, infra, coordinación) **sin obtener la independencia** que era el único motivo para distribuirlo. Lo peor de ambos mundos.

**10.2** Les recomiendo **NO usar microfrontends** y quedarse con un **monolito modular**. Con un solo equipo chico no existe el problema organizacional (escala de equipos) que los microfrontends resuelven; "estar a la moda" no es una razón de ingeniería. Sumarían complejidad distribuida (repos, pipelines, versionado de shared deps, resiliencia) a cambio de cero beneficio real. Es over-engineering de manual.

**10.3** **Separación por dominio y code-splitting**: con workspaces, módulos por dominio y lazy routes, ya obtenés boundaries claros entre features y bajás solo el código de la página que el usuario visita. Eso cubre la mayor parte del beneficio "técnico" de los microfrontends **sin** el costo de runtime federado, duplicación de deps ni coordinación de versiones entre equipos.



**11.1** Un **custom element** es un elemento del DOM real definido por vos (`class X extends HTMLElement` + `customElements.define`), con ciclo de vida propio. Es la "lingua franca" porque el contrato de interoperabilidad es **la plataforma** (atributos, propiedades, eventos del DOM, slots), que **todos** los frameworks entienden — no el modelo de componentes de ninguno. Angular lo exporta con **`@angular/elements` + `createCustomElement(Component, { injector })`**. React 19 arregló el **consumo**: ahora asigna las props que coinciden con una propiedad del elemento como **propiedades** (pasan objetos/arrays, antes los serializaba a string) y permite escuchar eventos custom con prefijo `on` (`onsay-hi`).

**11.2** **single-spa orquesta; Module Federation comparte código.** single-spa es el **portero**: decide qué app (de qué framework) se monta en cada ruta, gestiona su ciclo de vida (bootstrap/mount/unmount) y maneja el routing top-level. Module Federation resuelve otra cosa: **cómo un host carga código de un remote en runtime** y cómo comparten dependencias. Son complementarios porque cubren ejes distintos —routing/ciclo de vida vs. carga/sharing de código—: podés usar single-spa para orquestar y MF (o import maps) para cargar y compartir. La doc de single-spa solo pide no **mezclar los dos mecanismos de sharing** de deps third-party (elegí import maps **o** MF), pero a nivel de arquitectura conviven sin problema.

**11.3** No interopera porque cada framework tiene su propio árbol interno y su motor de render: `react-dom` solo sabe renderizar **fibras de React** (su árbol + su reconciliador); un componente Vue corre sobre el renderer y la reactividad de **Vue**. Son dos cerebros que no comparten neuronas. Si lo bridgeás ingenuamente, con cada re-render del host el componente Vue **pierde su estado**. El patrón que SÍ funciona: el remote expone una **función de montaje agnóstica** —`mount(el, props) => unmount`—; el host React le da **un nodo del DOM que él controla** y Vue monta su propia app adentro. React es dueño del nodo, Vue de lo que pasa dentro; el límite es un contrato chico (MF lo oficializa como patrón **Bridge**).

**11.4** La **"Hydra of Lerna"** (término de Mezzalira) es el anti-patrón de bootstrappear varios frameworks en la misma pestaña, para lo que **no fueron diseñados** — por cada framework que sumás crecen dos costos a la vez. Dos costos concretos: (1) **payload** — el runtime de los tres (≈140KB+ gz ⚠️) se descarga **antes** de tu código de producto; (2) **N reconciliadores en memoria**, cada uno con su render loop y su sistema de reactividad. Mitigación (cualquiera): compartir el runtime dentro de la misma familia (un solo React), lazy-load de las islas de otro framework, o limitar la diversidad de frameworks.

**11.5** No podés porque un React Context es una **construcción interna del runtime de React** — vive en su árbol de fibras, y Angular (con su propio runtime y su DI) no tiene forma de verlo. Dos frameworks bootstrappeados por separado **no comparten ningún grafo de objetos**. Lo ÚNICO que comparten es **la plataforma del navegador**: el DOM, `window`, los `CustomEvent`, la URL, el web storage y `BroadcastChannel`.

**11.6** "No compartas estado, compartí eventos" significa: en vez de un objeto de estado común que todos leen y mutan (acoplamiento máximo), emitís **eventos** de "pasó algo" y cada MFE reacciona con su propio estado interno; el emisor no sabe quién escucha (acoplamiento mínimo). Un Redux compartido entre React y Angular es **peor** que entre dos MFEs React porque, además de recrear el monolito, **ni siquiera funciona**: la reactividad de Redux está atada a React; Angular no se entera de los cambios sin pegamento manual (suscripciones + disparar change detection a mano). O sea: pagás el acoplamiento y encima no obtenés la reactividad.

**11.7**
```ts
// shared/event-bus.ts — agnóstico de framework, vive sobre window
type MfeEventMap = {
  'auth:logout': { reason: string }
  // ...más eventos del contrato
}

export function emit<K extends keyof MfeEventMap>(type: K, detail: MfeEventMap[K]): void {
  window.dispatchEvent(new CustomEvent(type, { detail }))
}

export function on<K extends keyof MfeEventMap>(
  type: K,
  handler: (detail: MfeEventMap[K]) => void,
): () => void {
  const listener = (e: Event) => handler((e as CustomEvent<MfeEventMap[K]>).detail)
  window.addEventListener(type, listener)
  return () => window.removeEventListener(type, listener) // el "off"
}
```
Las claves: el mapa `MfeEventMap` + los genéricos `K extends keyof MfeEventMap` hacen que `emit`/`on` estén **tipados** (el `detail` se infiere por el nombre del evento). `on` devuelve la función de baja para que cada framework la llame al desmontar. Y ojo: ese tipado **solo existe en el código fuente** — en runtime el evento es un string y un objeto, así que el shape es un contrato implícito (módulo 9).

**11.8**
```tsx
// MFE React — emite
import { emit } from 'shared/event-bus' // emit('auth:logout', { reason })

export function LogoutButton() {
  return (
    <button onClick={() => emit('auth:logout', { reason: 'user-click' })}>
      Salir
    </button>
  )
}
```
```vue
<!-- MFE Vue — escucha y LIMPIA -->
<script setup lang="ts">
import { ref, onUnmounted } from 'vue'
import { on } from 'shared/event-bus'

const mensaje = ref<string | null>(null)
const off = on('auth:logout', ({ reason }) => {
  mensaje.value = `Sesión cerrada (${reason})`
})
onUnmounted(off) // sin esto, el listener sobrevive al desmontaje → leak
</script>

<template>
  <div v-if="mensaje" class="banner">{{ mensaje }}</div>
</template>
```
La clave: React solo despacha el `CustomEvent` al `window` (no conoce a Vue); Vue lo escucha y reacciona con su propio `ref`. Y el `onUnmounted(off)` garantiza que, si el MFE Vue se desmonta (ej. en single-spa), el listener se va con él.

**11.9** El principal riesgo es el **"in-between permanente"**: la migración nunca termina y quedás manteniendo **dos frameworks para siempre** — la complejidad distribuida sin el beneficio (un "distributed monolith", módulo 10). Lo evitás **planificando la baja del legacy y poniéndole fecha** (Sam Newman): tratás la coexistencia como un estado **transitorio con deadline**, medís el avance (% migrado) y priorizás cerrar la migración, no solo abrir features nuevas en lo nuevo.

---

> **Para seguir.** La fuente de verdad de este módulo son `module-federation.io`, `webpack.js.org`, `rspack.rs` y el artículo de [Micro Frontends de Cam Jackson en martinfowler.com](https://martinfowler.com/articles/micro-frontends.html). Para el módulo multi-framework (11), sumá: `single-spa.js.org`, el scoreboard `custom-elements-everywhere.com`, [micro-frontends.org de Michael Geers](https://micro-frontends.org/), las guías de Web Components de [angular.dev](https://angular.dev/guide/elements) y [vuejs.org](https://vuejs.org/guide/extras/web-components.html), el [strangler fig de Fowler](https://martinfowler.com/bliki/StranglerFigApplication.html), y el libro *Building Micro-Frontends* de Luca Mezzalira (2ª ed., 2025). Antes de implementar, re-verificá lo marcado con ⚠️ — versiones y nombres de paquetes de MF, el comportamiento exacto de `strictVersion`, las opciones de Vite, el soporte de custom elements de React 19, y las cifras de bundle size. El puente natural: [TanStack Start](tanstack-start.md) y [RSC](rsc.md) para el lado de cada app, y el track de arquitectura para los patrones de boundaries.
