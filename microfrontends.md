# Microfrontends robustos: del concepto a producción

**Module Federation, resiliencia y aislamiento · ejemplos en React · 2026**

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

---

> **Para seguir.** La fuente de verdad de este módulo son `module-federation.io`, `webpack.js.org`, `rspack.rs` y el artículo de [Micro Frontends de Cam Jackson en martinfowler.com](https://martinfowler.com/articles/micro-frontends.html). Antes de implementar, re-verificá lo marcado con ⚠️ — versiones y nombres de paquetes de MF, el comportamiento exacto de `strictVersion`, y las opciones de Vite. El puente natural: [TanStack Start](tanstack-start.md) y [RSC](rsc.md) para el lado de cada app, y el track de arquitectura para los patrones de boundaries.
