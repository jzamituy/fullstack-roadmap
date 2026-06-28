# Service Workers en el browser a fondo

**El proxy de red programable: ciclo de vida, Cache API, offline, push y el criterio de cuándo (no) usarlos · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es de plataforma web pura — no depende de React ni de ningún framework. Es el motor que hay debajo de las **PWA**, del *offline*, de las *push notifications* y de buena parte de las apps "instalables". Si venís de SPA, el service worker es **lo más cercano a tener un mini-servidor corriendo dentro del browser del usuario**, y casi toda la confusión nace de no entender su ciclo de vida.

**Lo que asumimos.** JavaScript moderno (`Promise`, `async/await`, `fetch`), una idea básica de HTTP (request/response, headers, status codes) y haber visto el `fetch` del browser. Ayuda —pero no es obligatorio— saber qué es un Web Worker (hilo separado del main thread).

> **¿Te falta alguna base?** Si **caché HTTP** (los headers `Cache-Control`, `ETag`) te suena nuevo, no te frenes: lo tocamos en el módulo 10 justo para contrastarlo con el service worker, que es **otra capa de caché distinta**. Y si "main thread" te resulta vago, alcanza con esta idea: es el único hilo donde corre tu UI y tu código de página; todo lo que lo bloquea, congela la pantalla.

> ⚠️ **Nota sobre soporte y APIs.** El service worker en sí es estable y soportado en todos los browsers modernos desde hace años. Pero algunas APIs que se apoyan en él —**Background Sync**, **Periodic Background Sync**, **Push en iOS**— tienen soporte desigual entre browsers y plataformas. Todo lo marcado con ⚠️ verificalo en `caniuse.com` o en MDN antes de tomarlo como garantizado, sobre todo si tu target incluye Safari/iOS.

**Tipos de ejercicio** (para saber cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere un proyecto servido por HTTPS o `localhost`).

**Índice de módulos**
1. Qué es un Service Worker y qué problema resuelve
2. El modelo mental: un proxy de red programable (y por qué NO es un Web Worker común)
3. Registro, scope y el requisito de HTTPS
4. El ciclo de vida: `install` → `waiting` → `activate` (la parte más confusa)
5. La Cache API: guardar respuestas, no datos
6. El evento `fetch`: interceptar y responder
7. Estrategias de caching (cache-first, network-first, stale-while-revalidate…)
8. Versionado y actualización: el usuario "atascado" en una versión vieja
9. Más allá del offline: Push, Background Sync y Periodic Sync
10. El lado servidor en Node: web-push, el `manifest` y los headers correctos
11. Limitaciones, trampas y debugging
12. El criterio: cuándo un Service Worker paga (y cuándo es over-engineering)

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es un Service Worker y qué problema resuelve

**Teoría.** Un **service worker** es un script de JavaScript que el browser corre **en segundo plano**, en un hilo separado de tu página, y que actúa como **intermediario entre tu app y la red**. Una vez instalado, **puede interceptar cada request HTTP que sale de las páginas bajo su control** y decidir qué hacer: ir a la red, responder desde una caché, fabricar una respuesta, o combinar las tres.

¿Qué problema resuelve? La web clásica tiene una dependencia dura: **sin red, no hay nada**. Si el usuario pierde conexión, la app se cae. El service worker rompe esa dependencia. Concretamente habilita:

- **Offline y mala conexión.** La app sigue funcionando (total o parcialmente) sin internet, sirviendo desde caché lo que ya descargó.
- **Performance.** Servir assets desde una caché local controlada por vos es más rápido y predecible que depender solo de la caché HTTP del browser.
- **Capacidades "tipo app nativa".** Push notifications, sincronización en segundo plano, e instalación como PWA — todas se apoyan en el service worker.

La frase mental: **un service worker es código tuyo que se mete en el medio de la red del browser y la programa.** No es una librería que importás; es un proceso que vive entre tu app y el servidor.

Una aclaración temprana que ahorra confusión: el service worker **no acelera mágicamente nada por existir**. Es un *mecanismo*. La velocidad y el offline los conseguís vos escribiendo la lógica de caché correcta (módulos 5–7). Un service worker mal escrito puede dejarte **peor** que sin él (módulo 8).

**Ejercicios 1**
1.1 🔁 ¿Qué posición ocupa un service worker respecto de tu app y la red?
1.2 🔁 Nombrá tres capacidades que habilita un service worker.
1.3 🧠 "Instalo un service worker y mi sitio se vuelve más rápido solo." ¿Por qué esta afirmación es engañosa?

---

## Módulo 2 — El modelo mental: un proxy de red programable (y por qué NO es un Web Worker común)

**Teoría.** Un service worker **es** un tipo de Web Worker (corre fuera del main thread, no tiene acceso al DOM), pero tiene tres rasgos que lo hacen una bestia distinta:

| Rasgo | Web Worker común | Service Worker |
|---|---|---|
| Vínculo con la página | 1:1, vive y muere con la pestaña que lo creó | **Independiente de las páginas**: controla varias pestañas a la vez y sobrevive a que las cierres |
| Para qué sirve | Mover cálculo pesado fuera del main thread | **Interceptar red** y eventos del sistema (push, sync) |
| Ciclo de vida | Mientras la página viva | **Propio**, dirigido por eventos; el browser lo *arranca y mata* cuando quiere |

Tres consecuencias prácticas que tenés que internalizar desde ya:

1. **No tiene acceso al DOM.** No podés tocar `document`, ni `window`, ni leer el HTML. Si necesitás comunicarte con la página, usás mensajes (`postMessage`) o APIs como `Clients`.
2. **Es efímero.** El browser **termina** el service worker cuando está ocioso (a veces en segundos) y lo **revive** cuando llega un evento (un `fetch`, un `push`). Por eso **no podés guardar estado en variables globales** y esperar que sobreviva: entre dos eventos, todo se borra. El estado va a IndexedDB o a la Cache API.
3. **Es asíncrono hasta la médula.** No hay APIs síncronas de almacenamiento (nada de `localStorage`). Todo es `Promise`.

La frase mental: **pensalo como una función serverless (tipo AWS Lambda) que vive en el browser**: sin estado entre invocaciones, despertada por eventos, y muerta apenas termina. Esa analogía te ahorra el 80% de los bugs de principiante. (Un matiz, para no estirarla de más: la analogía es por el **ciclo** —se despierta ante eventos y muere ocioso—, **no** por la concurrencia. Hay **una sola instancia** del service worker por scope, no un pool que escala como Lambda.)

**Ejercicios 2**
2.1 🔁 ¿Por qué no podés acceder al `document` desde un service worker?
2.2 🧠 Un colega guarda un contador en una variable global del service worker y se sorprende de que "se resetea solo". ¿Qué le explicás?
2.3 🧠 ¿Por qué la analogía con una función serverless (sin estado, dirigida por eventos) es útil para razonar sobre service workers?

---

## Módulo 3 — Registro, scope y el requisito de HTTPS

**Teoría.** El service worker no se activa solo: lo **registrás** desde tu página, normalmente al cargar.

```js
// en tu app (main thread), por ejemplo en main.js
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('/sw.js')
      .then(reg => console.log('SW registrado, scope:', reg.scope))
      .catch(err => console.error('Falló el registro:', err))
  })
}
```

Tres reglas que **siempre** confunden:

- **HTTPS obligatorio.** Los service workers solo funcionan en **contextos seguros** (`https://`). La única excepción es `http://localhost`, justamente para que puedas desarrollar. Esto es por seguridad: un service worker que intercepta toda la red sería un arma brutal para un atacante *man-in-the-middle* si funcionara sobre HTTP.
- **El scope lo define la ubicación del archivo.** Un service worker controla las páginas que estén **dentro de su carpeta o más abajo**. Un `sw.js` en la **raíz** (`/sw.js`) controla **todo el sitio**. Uno en `/app/sw.js` controla solo `/app/...`. Por eso casi siempre lo querés en la raíz. (Podés ampliar el scope con el header `Service-Worker-Allowed`, pero no reducirlo más allá de la ubicación del archivo.)
- **Registrar ≠ controlar (la primera vez).** Cuando registrás un service worker por primera vez, la página que lo registró **no queda controlada todavía**. El control empieza en la *próxima* carga (salvo que fuerces `clients.claim()`, módulo 4). Esto descoloca a todos al principio: "lo registré pero el `fetch` no se dispara". Es esperado.

La frase mental: **el scope lo define el path del archivo `sw.js`, no dónde lo registrás ni qué importás. Archivo en la raíz = control de todo el origen; archivo enterrado = control solo de esa subcarpeta hacia abajo.**

**Ejercicios 3**
3.1 🔁 ¿Por qué los service workers exigen HTTPS, y cuál es la única excepción?
3.2 🧠 Tenés `sw.js` en `/static/js/sw.js`. ¿Por qué tu home en `/` no queda bajo control? ¿Dónde lo moverías?
3.3 🧠 Registrás el SW y notás que en esa misma carga no intercepta nada. ¿Es un bug? Explicá.

---

## Módulo 4 — El ciclo de vida: `install` → `waiting` → `activate` (la parte más confusa)

**Teoría.** Acá está **el corazón de todo el tema**, y donde se pierde casi todo el mundo. El service worker tiene un ciclo de vida propio, dirigido por eventos:

```
download → install → (waiting) → activate → (controlando) → redundant
```

- **`install`**: se dispara una vez, cuando el browser baja un service worker **nuevo o cambiado**. Acá típicamente **precacheás** los assets del *app shell* (HTML, CSS, JS, logo). Si el `install` falla, el service worker se descarta.
- **`waiting`**: este es el paso traicionero. Si ya hay un service worker **viejo controlando pestañas abiertas**, el nuevo queda **en espera** (waiting) y **no toma control** hasta que **todas** las pestañas controladas por el viejo se cierren. El browser hace esto a propósito: garantiza que **nunca convivan dos versiones** de tu lógica de red. La consecuencia es la que te muerde: el usuario actualiza, ve "nueva versión disponible"… pero sigue con la vieja hasta que cierre todas las pestañas.
- **`activate`**: se dispara cuando el nuevo finalmente toma control. Acá típicamente **limpiás cachés viejas** (módulo 8).

Dos métodos para alterar este baile, que tenés que usar **con criterio**:

- **`self.skipWaiting()`** (en `install`): le dice al nuevo service worker "no esperes, activate ya". Saltea el estado *waiting*.
- **`self.clients.claim()`** (en `activate`): hace que el service worker tome control de las pestañas **ya abiertas** inmediatamente, sin esperar a la próxima carga.

```js
// sw.js
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('app-shell-v1').then(cache =>
      cache.addAll(['/', '/index.html', '/styles.css', '/app.js'])
    )
  )
  // self.skipWaiting()  // ⚠️ ojo: lo discutimos abajo
})

self.addEventListener('activate', (event) => {
  event.waitUntil(self.clients.claim())
})
```

⚠️ **El `skipWaiting()` no es gratis.** Si lo activás a ciegas, podés terminar con una pestaña cargada con el HTML/JS **viejo** pero servida de pronto por un service worker **nuevo** con otra lógica de caché → respuestas incoherentes, assets que no matchean. El patrón seguro es: mostrarle al usuario un aviso "hay una actualización", y llamar a `skipWaiting()` **solo cuando él acepta recargar**. Lo cerramos en el módulo 8.

Notá el **`event.waitUntil(promise)`**: le dice al browser "no consideres terminado este evento (ni me mates) hasta que esta promesa resuelva". Es **esencial** porque el service worker es efímero (módulo 2): sin `waitUntil`, el browser podría matarlo a mitad del precache.

La frase mental: **un service worker nuevo se instala enseguida pero NO toma el mando mientras el viejo tenga pestañas abiertas. Esa espera es una garantía de seguridad, no un bug — y `skipWaiting` la rompe a propósito, así que usalo conscientemente.**

**Ejercicios 4**
4.1 🔁 ¿Qué solés hacer en `install` y qué en `activate`?
4.2 🧠 ¿Por qué el browser deja a un service worker nuevo en estado *waiting* en vez de activarlo directo? ¿Qué garantiza?
4.3 🧠 ¿Para qué sirve `event.waitUntil()` y por qué es crítico dado que el service worker es efímero?
4.4 🧠 ¿Cuál es el riesgo de llamar `skipWaiting()` sin avisarle al usuario?

---

## Módulo 5 — La Cache API: guardar respuestas, no datos

**Teoría.** La **Cache API** (`caches`) es el almacén que usa el service worker para guardar **pares request/response**. Punto clave que distingue a la gente que entendió: **la Cache API guarda objetos `Response` HTTP completos**, no JSON ni strings sueltos. Es una caché de *recursos de red*, no una base de datos.

```js
const cache = await caches.open('app-shell-v1')   // abre (o crea) una caché con nombre

await cache.add('/styles.css')                    // hace fetch y guarda la respuesta
await cache.addAll(['/', '/app.js', '/logo.png']) // varios de una; atómico: si uno falla, falla todo

const res = await cache.match('/styles.css')      // busca; devuelve Response o undefined
await cache.put(request, response)                // guarda un par a mano (útil al interceptar)
await cache.delete('/old.js')                     // borra una entrada
await caches.delete('app-shell-v0')               // borra una caché entera (por nombre)
await caches.keys()                               // lista los nombres de todas las cachés
```

Cuatro cosas que conviene tener claras:

- **Las cachés tienen nombre** (`app-shell-v1`). Vos manejás esos nombres, y de ahí sale el versionado (módulo 8): "versión nueva = nombre nuevo".
- **Es persistente.** Sobrevive a cerrar el browser. No se borra sola al cerrar la pestaña (a diferencia de `sessionStorage`).
- **No expira sola.** La Cache API **no respeta** los headers `Cache-Control` ni `max-age`. Una vez que guardaste una respuesta, **se queda ahí hasta que vos la borres**. La expiración la administrás vos (o una librería como Workbox).
- **¿Cuándo NO es la herramienta?** Para datos estructurados que consultás/filtrás (registros de usuario, estado de la app offline) usá **IndexedDB**, no la Cache API. Regla: **assets y respuestas HTTP → Cache API; datos de la app → IndexedDB.**

⚠️ **El almacenamiento no es infinito ni garantizado.** El browser asigna una cuota por origen y puede **evictar** (purgar) tu caché bajo presión de disco. Si tu offline depende de datos críticos, pedí almacenamiento persistente con `navigator.storage.persist()` y nunca asumas que la caché está ahí — siempre tené un fallback.

La frase mental: **la Cache API es un diccionario persistente de `request → Response`, controlado por vos, que no expira solo. Es para recursos de red; los datos de la app van a IndexedDB.**

**Ejercicios 5**
5.1 🔁 ¿Qué tipo de objeto guarda la Cache API: JSON, strings, o `Response`?
5.2 🔁 ¿La Cache API respeta el `max-age` del header `Cache-Control`? ¿Quién administra la expiración entonces?
5.3 🧠 Querés guardar offline una lista de tareas que el usuario filtra y ordena. ¿Cache API o IndexedDB? ¿Por qué?
5.4 🧠 ¿Por qué `cache.addAll([...])` es conveniente para precachear el app shell, y qué pasa si uno de los recursos da 404?

---

## Módulo 6 — El evento `fetch`: interceptar y responder

**Teoría.** Acá es donde el service worker se gana el título de "proxy programable". Cada vez que una página bajo su control hace una request (un `fetch`, cargar una imagen, un `<script>`, navegar a una URL), se dispara el evento **`fetch`** en el service worker. Vos podés **interceptarlo** y decidir la respuesta con `event.respondWith()`:

```js
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => {
      // si está en caché lo devolvemos; si no, vamos a la red
      return cached || fetch(event.request)
    })
  )
})
```

Reglas que evitan los dolores típicos:

- **`event.respondWith(responsePromise)`** define la respuesta. Si **no** llamás `respondWith`, el browser hace el request normal (red), como si el service worker no existiera. O sea: **interceptar es opt-in por request**. Podés filtrar y dejar pasar las que no querés tocar.
- **Devolvés una `Promise<Response>`.** Puede venir de la caché, de `fetch()`, o construida a mano (`new Response('...')`).
- **No interceptes lo que no debés.** Requests `POST`/`PUT` (mutaciones), llamadas a analytics, requests a otros orígenes que no controlás… normalmente conviene **dejarlas pasar** (no llamar `respondWith`). Cachear un `POST` no tiene sentido y la Cache API ni siquiera lo permite por defecto.
- **Cuidado con cachear respuestas malas.** Antes de guardar una respuesta nueva, verificá que sea válida (`response.ok`, status correcto). Cachear un 404 o un 500 te deja sirviendo basura desde la caché.
- **Las respuestas `opaque`.** Cuando hacés fetch *cross-origin* sin CORS (modo `no-cors`), recibís una respuesta **opaca**: no podés leer su status ni su contenido, y ocupa más cuota de la que parece. Ojo con un detalle que confunde: en una respuesta opaca, **`response.ok` siempre es `false`** y `status` es `0`, así que **no podés usar `response.ok` como criterio** para decidir si cachearla. Cachear opacas es una decisión explícita (y cara en cuota): hacela solo si sabés lo que hacés.

Ejemplo más realista, con guarda, clonado **y fallback offline**:

```js
self.addEventListener('fetch', (event) => {
  // Solo manejamos GET; lo demás pasa de largo
  if (event.request.method !== 'GET') return

  event.respondWith((async () => {
    const cached = await caches.match(event.request)
    if (cached) return cached

    try {
      const response = await fetch(event.request)
      // clonamos: una Response es un stream y se consume una sola vez
      if (response.ok) {
        const cache = await caches.open('runtime-v1')
        cache.put(event.request, response.clone())
      }
      return response
    } catch {
      // sin red Y sin caché: NO dejes que respondWith reciba un rechazo.
      // Devolvé un fallback (una offline.html precacheada, o una Response a mano).
      return (await caches.match('/offline.html'))
        ?? new Response('Sin conexión', { status: 503, statusText: 'Offline' })
    }
  })())
})
```

⚠️ **El `.clone()` no es opcional.** Un `Response` (y un `Request`) tiene un body que es un *stream*: se puede leer **una sola vez**. Si lo devolvés a la página **y** lo guardás en caché, necesitás dos copias → `response.clone()`. Olvidarlo es uno de los bugs más comunes del tema.

⚠️ **Nunca dejes que `respondWith` reciba una promesa rechazada.** Si `fetch()` falla (offline) y no hay nada en caché, un `await fetch(...)` sin `try/catch` propaga el error a `respondWith` y la request **rompe** — justo el caso offline que el service worker debería cubrir. Siempre cerrá con un fallback: una página `/offline.html` precacheada en `install`, o una `Response` construida a mano. Esta es la diferencia entre "tengo un service worker" y "tengo offline".

La frase mental: **el evento `fetch` te da la última palabra sobre cada request. `respondWith` = "yo me encargo"; no llamarlo = "que siga su curso". Interceptás selectivamente, validás antes de cachear, y clonás porque el body se consume una sola vez.**

**Ejercicios 6**
6.1 🔁 ¿Qué pasa si en el handler de `fetch` no llamás a `event.respondWith()`?
6.2 🔁 ¿Por qué hay que clonar la `Response` si la querés devolver y cachear a la vez?
6.3 🧠 ¿Por qué normalmente no interceptás requests `POST` ni llamadas de analytics?
6.4 🧠 ¿Qué problema trae cachear una respuesta sin chequear `response.ok` primero?
6.5 🧠 En el ejemplo realista, ¿qué pasa si sacás el `try/catch` y el usuario está offline pidiendo algo que no está en caché? ¿Por qué es justo el caso que el SW debía cubrir?

---

## Módulo 7 — Estrategias de caching

**Teoría.** "Usar service worker para cachear" no es una decisión binaria: hay **estrategias**, y elegir la correcta **por tipo de recurso** es la diferencia entre una app sólida y una que sirve contenido viejo. Las cinco clásicas:

| Estrategia | Qué hace | Cuándo conviene |
|---|---|---|
| **Cache First** | Mira la caché; si está, la devuelve sin tocar la red. Solo va a la red si falta. | Assets con hash en el nombre (`app.3f9a.js`), fuentes, logos — cosas **inmutables**. Máxima velocidad. |
| **Network First** | Va a la red; si falla (offline), cae a la caché. | HTML de navegación y datos que **deben** estar frescos pero querés un fallback offline. |
| **Stale-While-Revalidate** | Devuelve la caché **ya** (rápido) y **en paralelo** actualiza la caché desde la red para la próxima vez. | Recursos donde "un poco viejo está bien": avatares, contenido semi-dinámico. Lo mejor de velocidad + frescura diferida. |
| **Cache Only** | Solo caché, nunca red. | App shell precacheado que sabés que siempre está. |
| **Network Only** | Solo red, nunca caché. | Requests no cacheables: pagos, mutaciones, analytics. (Equivale a dejar pasar.) |

Ejemplo de **stale-while-revalidate** (la más usada para contenido dinámico):

```js
async function staleWhileRevalidate(request) {
  const cache = await caches.open('swr-v1')
  const cached = await cache.match(request)

  const networkFetch = fetch(request)
    .then(response => {
      if (response.ok) cache.put(request, response.clone())
      return response
    })
    .catch(() => cached)  // ⚠️ sin esto, un fallo de red queda sin manejar

  // devolvemos la caché si existe (rápido); si no, esperamos la red
  return cached || networkFetch
}
```

⚠️ **Ese `.catch()` no es decorativo.** Sin él, `return cached || networkFetch` tiene un agujero: si **no** hay caché y la red falla (offline), `respondWith` recibe una promesa **rechazada** y la request rompe. Y aunque haya caché, el rechazo de `networkFetch` queda como *unhandled rejection*. El `.catch(() => cached)` garantiza que el revalidado nunca tire un error suelto. **Regla de oro del SW: ninguna promesa que termine en `respondWith` debe poder rechazar.**

El criterio que separa a quien entendió: **no hay una estrategia "mejor" — hay una por tipo de recurso.** Un sitio real combina varias:
- JS/CSS con hash → **Cache First** (son inmutables, el hash cambia si cambian).
- `index.html` / navegación → **Network First** (querés la versión nueva, con offline de respaldo).
- API de contenido → **Stale-While-Revalidate** o **Network First** según cuán crítica sea la frescura.
- `/api/pagos` → **Network Only**.

**Navigation Preload: el costo de arranque escondido.** Hay un detalle de performance que muerde en *Network First* de navegación: cuando el service worker está dormido (módulo 2), el browser tiene que **arrancarlo** antes de que pueda hacer el `fetch` de la navegación → ese arranque agrega latencia justo en la request más importante (el HTML de la página). **Navigation Preload** lo resuelve: el browser **dispara la request de red en paralelo** mientras el service worker arranca, y vos consumís esa respuesta ya en curso.

```js
self.addEventListener('activate', (event) => {
  event.waitUntil(self.registration.navigationPreload?.enable())
})

self.addEventListener('fetch', (event) => {
  if (event.request.mode !== 'navigate') return
  event.respondWith((async () => {
    const preloaded = await event.preloadResponse  // la red ya arrancó en paralelo
    if (preloaded) return preloaded
    try { return await fetch(event.request) }
    catch { return (await caches.match('/offline.html')) ?? Response.error() }
  })())
})
```

⚠️ Soporte: estable en Chromium y Firefox; **Safari aún no lo implementa** — verificá en `caniuse.com`. Es una mejora progresiva: donde no está, el `event.preloadResponse` resuelve a `undefined` y caés al `fetch` normal, así que no rompe nada.

> 🧰 **En la práctica usás Workbox.** Escribir todo esto a mano es tedioso y propenso a bugs (expiración, límites de entradas, broadcast de updates, y el versionado manual del módulo 8). **Workbox** (de Google) te da estas estrategias como funciones listas, manejo de expiración, navigation preload, y routing por patrón de URL. Para producción, casi nadie escribe el `fetch` handler a mano: usás Workbox o el plugin de tu framework (`vite-plugin-pwa`, etc.). Entender los módulos 4–8 es para **saber qué hace Workbox por debajo** y poder debuggearlo — no para reimplementarlo, ni para sufrir el versionado a mano que vas a ver en el módulo 8.

La frase mental: **la estrategia se elige por recurso, no por app. Inmutable → cache-first; navegación → network-first; contenido tolerante → stale-while-revalidate; sensible → network-only. En producción, Workbox.**

**Ejercicios 7**
7.1 🔁 Asociá cada recurso con su estrategia ideal: (a) `app.8f2c.js`, (b) `index.html`, (c) avatar de usuario, (d) `POST /api/checkout`.
7.2 🧠 Explicá por qué *stale-while-revalidate* es un buen balance, y cuál es su contra (qué ve el usuario en la primera carga tras un cambio).
7.3 🧠 ¿Por qué *Cache First* es seguro para `app.8f2c.js` pero peligroso para `app.js` (sin hash)?
7.4 🧠 ¿Qué aporta Workbox sobre escribir el handler de `fetch` a mano, y por qué igual conviene entender el mecanismo crudo?
7.5 🧠 ¿Qué problema de latencia resuelve Navigation Preload, y por qué aparece justo cuando el service worker está dormido?

---

## Módulo 8 — Versionado y actualización: el usuario "atascado" en una versión vieja

**Teoría.** Este módulo resuelve **el problema operativo más serio** de los service workers, el que genera tickets de soporte: *"deployé un fix hace una semana y hay usuarios que siguen viendo la versión vieja"*.

¿Por qué pasa? Combinación de dos cosas que ya viste:
1. La Cache API **no expira sola** (módulo 5): lo que precacheaste se queda.
2. El service worker nuevo queda en **waiting** y no toma control mientras haya pestañas viejas abiertas (módulo 4). Y mucha gente **nunca cierra del todo** la pestaña (móvil, PWA).

El patrón de versionado correcto tiene tres piezas:

**a) Versioná el nombre de la caché.** Cada release usa un nombre nuevo, y en `activate` borrás los viejos:

```js
const CACHE = 'app-shell-v3'   // ← bumpeás esto en cada deploy

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE).then(c => c.addAll(['/', '/app.js', '/styles.css']))
  )
})

self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then(names =>
      Promise.all(
        names.filter(n => n !== CACHE)   // todo lo que no sea la versión actual
             .map(n => caches.delete(n)) // se borra
      )
    )
  )
})
```

**b) El browser detecta el cambio del `sw.js`.** Cuando el usuario vuelve a entrar (y periódicamente), el browser **re-baja `sw.js`** y lo compara **byte a byte** con el que tiene. Si cambió (basta cambiar la constante `CACHE`), arranca el ciclo `install` del nuevo. ⚠️ **Importante:** asegurate de que el **propio `sw.js` no se sirva con caché HTTP larga**, o el browser no se enterará de que cambiaste el service worker. Servilo con `Cache-Control: no-cache` (o `max-age=0`).

Dos precisiones que cierran esta trampa del todo:

- **El browser ya limita la caché del propio `sw.js` a 24h.** Desde hace años, por defecto el browser **ignora** cualquier `Cache-Control` de más de 24 horas **para el script del service worker** (no para los recursos que cachea, ojo — solo para el `sw.js` mismo). O sea: aun si te olvidás el header, la actualización llega como mucho en un día. Pero no dependas de eso: serví el `sw.js` con `no-cache` para que el cambio se detecte enseguida.
- **`updateViaCache` controla esto explícitamente.** En el `register()` podés decirle al browser que **nunca** use la caché HTTP para chequear el `sw.js` (ni sus `importScripts`):

```js
navigator.serviceWorker.register('/sw.js', { updateViaCache: 'none' })
// 'none'    → nunca usa caché HTTP para el sw.js (recomendado)
// 'imports' → cachea el sw.js pero no sus importScripts
// 'all'     → comportamiento viejo (cachea todo); evitalo
```

**c) Avisale al usuario y dejá que él active.** El patrón seguro (ver módulo 4): cuando hay un service worker en *waiting*, mostrás un toast "Nueva versión disponible — Recargar". Al aceptar, le mandás `skipWaiting()` al worker en espera y recargás:

```js
// en la página
navigator.serviceWorker.register('/sw.js', { updateViaCache: 'none' }).then(reg => {
  reg.addEventListener('updatefound', () => {
    const nuevo = reg.installing
    nuevo.addEventListener('statechange', () => {
      if (nuevo.state === 'installed' && navigator.serviceWorker.controller) {
        mostrarToast('Nueva versión disponible', () => {
          nuevo.postMessage({ type: 'SKIP_WAITING' })
        })
      }
    })
  })
})

// recargamos cuando el nuevo SW toma control
navigator.serviceWorker.addEventListener('controllerchange', () => {
  window.location.reload()
})
```

```js
// en sw.js
self.addEventListener('message', (e) => {
  if (e.data?.type === 'SKIP_WAITING') self.skipWaiting()
})
```

La frase mental: **versión nueva = nombre de caché nuevo + borrar los viejos en `activate`. Serví `sw.js` sin caché HTTP. Y para no dejar al usuario atascado, detectá el waiting, avisale, y activá con su consentimiento.**

**Ejercicios 8**
8.1 🔁 ¿En qué evento del ciclo de vida conviene borrar las cachés de versiones anteriores?
8.2 🧠 ¿Por qué un usuario podría seguir viendo la versión vieja "para siempre", y qué dos mecanismos lo causan?
8.3 🧠 ¿Por qué es un error servir `sw.js` con `Cache-Control: max-age=31536000`? ¿Qué red de seguridad pone igual el browser, y por qué no deberías depender solo de ella?
8.4 ✍️ Describí (o codeá) el flujo "nueva versión disponible → el usuario acepta → se activa y recarga".
8.5 🔁 ¿Qué hace `updateViaCache: 'none'` en el `register()` y por qué conviene?

---

## Módulo 9 — Más allá del offline: Push, Background Sync y Periodic Sync

**Teoría.** El offline es el caso estrella, pero el service worker —al ser un proceso que el browser puede **despertar sin que la página esté abierta**— habilita un conjunto de APIs "tipo app nativa":

**Push Notifications (Push API + Notifications API).** El servidor le manda un mensaje al *push service* del browser; este **despierta al service worker** (aunque la app esté cerrada) y dispara un evento `push`, donde mostrás la notificación:

```js
self.addEventListener('push', (event) => {
  const data = event.data?.json() ?? {}
  event.waitUntil(
    self.registration.showNotification(data.title ?? 'Novedad', {
      body: data.body,
      icon: '/icon-192.png',
    })
  )
})

self.addEventListener('notificationclick', (event) => {
  event.notification.close()
  event.waitUntil(clients.openWindow('/'))  // abre/enfoca la app al tocar
})
```

⚠️ Requiere **permiso explícito** del usuario (`Notification.requestPermission()`), suscripción vía `pushManager.subscribe()` con claves **VAPID**, y un backend que envíe el push. En **iOS** llegó tarde y con una condición dura: Web Push existe **desde iOS 16.4 (marzo 2023)** y **solo funciona si el usuario agregó la PWA a la pantalla de inicio** (no en Safari como pestaña normal). En Android/desktop, en cambio, funciona desde el browser sin instalar. Verificá soporte y versión mínima antes de prometer push en iOS.

**Background Sync ⚠️.** Si el usuario hace una acción offline (mandar un mensaje, postear un comentario), registrás un *sync*; el browser lo **reintenta automáticamente** cuando vuelve la conexión, despertando al service worker:

```js
// en la página, cuando falla por estar offline
const reg = await navigator.serviceWorker.ready
await reg.sync.register('enviar-mensajes')

// en sw.js
self.addEventListener('sync', (event) => {
  if (event.tag === 'enviar-mensajes') {
    event.waitUntil(enviarMensajesPendientes())  // lee de IndexedDB y postea
  }
})
```

⚠️ **Ojo: Background Sync NO es cross-browser.** Aunque suene a estándar maduro, en 2026 la *one-shot* Background Sync API sigue siendo **esencialmente Chromium** — **Safari y Firefox no la implementan**. No es solo Periodic Sync el que falla afuera; este también. Por eso el patrón **portable** no se apoya en el evento `sync`: guardás la cola en **IndexedDB** y la drenás en **dos disparadores** que sí funcionan en todos lados — al **reabrir la app** (un check al cargar) y dentro del handler `fetch` cuando vuelve a haber red. Background Sync, donde existe, es la **mejora progresiva** encima de eso (reintenta aunque el usuario no reabra), no la base.

**Periodic Background Sync ⚠️.** Actualizar contenido cada tanto en segundo plano (ej. precargar el feed de noticias a la mañana). Soporte **muy limitado** (esencialmente Chromium, y atado al *engagement* del usuario con la PWA). No construyas features críticas sobre esto.

El hilo común de las tres: **el service worker puede ejecutarse cuando tu página no está abierta**. Eso es lo que lo separa de cualquier truco que puedas hacer solo con JS de página.

La frase mental: **el service worker no es solo caché — es el punto de enganche del browser para despertar tu código ante eventos del sistema (push, reconexión, timer). Push y Background Sync son las features "nativas" que eso desbloquea; ojo con el soporte desigual: Push exige PWA instalada en iOS, y Background/Periodic Sync son Chromium-only, así que el patrón portable siempre es IndexedDB + reintento.**

**Ejercicios 9**
9.1 🔁 ¿Qué tienen en común Push, Background Sync y Periodic Sync respecto del estado de la página?
9.2 🧠 El usuario manda un mensaje sin conexión. ¿Cómo combinás IndexedDB + Background Sync para que se envíe solo al volver la red?
9.3 🧠 ¿Por qué no deberías construir una feature crítica sobre Periodic Background Sync?
9.4 🧠 ¿Qué condición especial tiene Web Push en iOS que no tiene en Android/desktop?
9.5 🧠 Background Sync no existe en Safari/Firefox. ¿Cómo armás el envío offline para que funcione igual en todos los browsers, y dónde encaja Background Sync en ese diseño?

---

## Módulo 10 — El lado servidor en Node: web-push, el `manifest` y los headers correctos

**Teoría.** Hasta acá todo fue browser. Pero como venís hacia **Full Stack Node**, lo que el mercado te va a pedir no es solo escribir el `sw.js` — es **el otro extremo**: el servidor que hace que el service worker funcione. Tres piezas concretas que viven en tu Express/Nest.

**a) Servir el `sw.js` con los headers correctos.** El service worker no es un asset cualquiera: cómo lo servís define si las actualizaciones llegan (módulo 8) y hasta dónde alcanza su scope (módulo 3).

```ts
import express from 'express'
const app = express()

// El sw.js: sin caché HTTP, para que el browser detecte cambios enseguida (módulo 8)
app.get('/sw.js', (_req, res) => {
  res.set('Cache-Control', 'no-cache')
  // Solo si servís el sw.js desde una subcarpeta pero querés scope de raíz (módulo 3):
  res.set('Service-Worker-Allowed', '/')
  res.sendFile(`${process.cwd()}/public/sw.js`)
})
```

Las dos reglas que se traducen en headers: `Cache-Control: no-cache` para el `sw.js` (no para los assets con hash, que sí querés cachear largo), y `Service-Worker-Allowed` **solo** si necesitás ampliar el scope más allá de donde está el archivo.

**b) El `manifest.webmanifest`: lo que vuelve "instalable" a la PWA.** El service worker da el offline; el **manifest** da la instalabilidad (ícono en la pantalla de inicio, nombre, splash). Es un JSON que linkeás desde el HTML y servís con el content-type correcto:

```ts
app.get('/manifest.webmanifest', (_req, res) => {
  res.type('application/manifest+json')   // content-type que el browser espera
  res.json({
    name: 'Mi App', short_name: 'MiApp', start_url: '/', display: 'standalone',
    background_color: '#ffffff', theme_color: '#0b5',
    icons: [{ src: '/icon-192.png', sizes: '192x192', type: 'image/png' },
            { src: '/icon-512.png', sizes: '512x512', type: 'image/png' }],
  })
})
```
```html
<link rel="manifest" href="/manifest.webmanifest">
```

Para que el browser ofrezca "Instalar", se necesita: **HTTPS + service worker registrado + manifest válido** (con `name`, `start_url`, `display` e íconos de 192 y 512). Las tres cosas juntas.

**c) Emitir Push desde Node (`web-push` + VAPID).** En el módulo 9 viste el lado browser (`pushManager.subscribe`). El que **manda** el push es tu backend. El flujo:

1. Generás un par de claves **VAPID** una vez (identifican a tu server ante el push service): `npx web-push generate-vapid-keys`. La **pública** va al cliente; la **privada** queda **solo en el server** (es un secreto, como cualquier API key).
2. El cliente se suscribe con la clave pública y te manda su `subscription` (un objeto con endpoint + claves). **Lo guardás en tu DB**, asociado al usuario.
3. Cuando hay algo que notificar, tu server le pega al `endpoint` de la suscripción con la librería `web-push`:

```ts
import webpush from 'web-push'

webpush.setVapidDetails(
  'mailto:vos@dominio.com',
  process.env.VAPID_PUBLIC_KEY!,   // pública
  process.env.VAPID_PRIVATE_KEY!,  // privada — NUNCA al cliente
)

// subscription = lo que el browser mandó en el módulo 9 y guardaste en la DB
async function notificar(subscription: webpush.PushSubscription, payload: object) {
  try {
    await webpush.sendNotification(subscription, JSON.stringify(payload))
  } catch (err: any) {
    // 404/410 = la suscripción caducó o el usuario revocó: borrala de la DB
    if (err.statusCode === 404 || err.statusCode === 410) {
      await borrarSubscription(subscription.endpoint)
    } else {
      throw err
    }
  }
}
```

El detalle de producción que distingue: **las suscripciones caducan o se revocan**. Cuando `sendNotification` devuelve **410 Gone** (o 404), esa suscripción está muerta — hay que **borrarla de la DB**, o vas a acumular basura y reintentar contra endpoints muertos para siempre.

La frase mental: **el service worker es la mitad cliente de un sistema de dos extremos. La otra mitad vive en tu Node: servir `sw.js` con `no-cache`, servir el `manifest` para la instalabilidad, y emitir push con `web-push` + VAPID — limpiando las suscripciones muertas (410). Eso es lo que te van a pedir como backend.**

**Ejercicios 10**
10.1 🔁 ¿Qué tres cosas tienen que cumplirse juntas para que el browser ofrezca "Instalar" una PWA?
10.2 🧠 ¿Por qué la clave VAPID privada nunca puede viajar al cliente, y dónde vive cada una de las dos claves?
10.3 🧠 Tu server manda push y recibe un `410 Gone` de un endpoint. ¿Qué pasó y qué tenés que hacer?
10.4 ✍️ Escribí el handler de Express que sirve `sw.js` con los headers correctos para que las actualizaciones se detecten y, si hiciera falta, con scope de raíz desde una subcarpeta.

---

## Módulo 11 — Limitaciones, trampas y debugging

**Teoría.** El service worker es poderoso justamente porque se mete en el medio de todo — y eso lo hace **peligroso si lo manejás mal**. Las limitaciones y trampas que tenés que conocer:

**Limitaciones duras:**
- **Sin DOM, sin `window`, sin `localStorage`** (módulo 2). Comunicación con la página vía `postMessage`/`Clients`; estado en IndexedDB/Cache.
- **Solo HTTPS / localhost** (módulo 3).
- **Efímero**: se mata cuando está ocioso; nada de estado en globals (módulo 2).
- **Cuota de almacenamiento limitada y evictable** (módulo 5): el browser puede purgar tu caché bajo presión de disco.
- **Soporte desigual** en las APIs satélite (Background/Periodic Sync, Push en iOS).

**Trampas clásicas (las que te van a pasar):**
- **"Lo actualicé y no cambia nada."** Es el ciclo de vida (módulos 4 y 8): el nuevo está en *waiting*. En DevTools usás **"Update on reload"** y **"Skip waiting"** mientras desarrollás.
- **El service worker que sirve contenido viejo (el "zombie").** Cacheaste agresivo con cache-first sobre recursos que sí cambian, sin versionado. Resultado: usuarios congelados en una versión. **Siempre** tené un plan de actualización (módulo 8) antes de cachear agresivo.
- **El "kill switch".** Por las dudas, conviene poder **desactivar** un service worker roto en producción. Un `sw.js` que se desregistra a sí mismo y borra cachés, deployado de emergencia, te salva de un service worker que rompió a todos los usuarios. Es la red de seguridad del tema.
- **Olvidar `.clone()`** (módulo 6) → "body already used".
- **Cachear respuestas opacas o con error** sin querer → caché envenenada.

**Service Worker vs caché HTTP.** Pregunta frecuente: *"¿no hace lo mismo que el `Cache-Control`?"*. No. Son **dos capas distintas**:

| | Caché HTTP (`Cache-Control`, `ETag`) | Service Worker + Cache API |
|---|---|---|
| Quién la controla | El **browser**, según headers del server | **Tu código** JS, con lógica arbitraria |
| Offline | No (más allá de lo que el browser decida) | **Sí**, programable |
| Granularidad | Por header de respuesta | Por request, con cualquier `if` que quieras |
| Complejidad | Casi cero (configurás headers) | Alta (ciclo de vida, versionado, bugs) |

La caché HTTP es **gratis y suficiente** para el 80% de los casos. El service worker es la artillería pesada para offline real y control fino.

**Debugging.** En Chrome DevTools → pestaña **Application → Service Workers** (ver estado, forzar update, skip waiting, unregister) y **Application → Cache Storage** (inspeccionar y borrar cachés). El checkbox **"Bypass for network"** ignora el service worker mientras debugueás la página.

La frase mental: **el service worker te da poder a cambio de complejidad permanente. Cacheá agresivo solo con versionado y un kill switch listos. Y recordá que la caché HTTP, gratis, ya resuelve la mayoría de los casos.**

**Ejercicios 11**
11.1 🔁 Nombrá tres limitaciones duras de los service workers.
11.2 🧠 ¿Qué es un "service worker zombie" y cómo lo prevenís?
11.3 🧠 ¿Para qué sirve tener un "kill switch" preparado y qué haría ese `sw.js` de emergencia?
11.4 🧠 Diferenciá la caché HTTP de la caché vía service worker en dos ejes (quién controla y offline).

---

## Módulo 12 — El criterio: cuándo un Service Worker paga (y cuándo es over-engineering)

**Teoría.** El cierre: convertir todo lo anterior en una decisión defendible. La pregunta no es "¿los service workers son buenos?", sino "**¿este service worker paga para MI app?**".

**El service worker paga cuando:**
- **Necesitás offline real** o tolerancia a conexión inestable: PWA, apps de campo, transporte, retail con conexión pobre, lectores/notas.
- **Querés push notifications** o sincronización en segundo plano (las features "nativas" del módulo 9).
- **Tenés un app shell** estable que querés cargar instantáneo y servir aunque el server esté lento.
- **Vas a instalar la app** (PWA en pantalla de inicio): el service worker es **requisito** para que sea instalable.

**Es over-engineering cuando:**
- Es un **sitio de contenido simple** (landing, blog, dashboard online-only) donde **un buen `Cache-Control` + CDN** ya te da casi toda la velocidad, sin la complejidad del ciclo de vida.
- **No vas a mantener** la lógica de versionado/actualización. Un service worker sin plan de updates es una **bomba de contenido viejo** — peor que no tenerlo.
- Lo agregás "porque es PWA y suena moderno", sin un caso de offline/push concreto. El costo (debugging, soporte, usuarios atascados) supera el beneficio.

El matiz honesto: **el service worker no es una mejora gratis que activás "por las dudas".** Es una pieza de infraestructura con mantenimiento permanente. Si tu único objetivo es "que cargue rápido", **empezá por caché HTTP, CDN y optimización de assets** — son más baratos y resuelven la mayoría de los casos. Reservá el service worker para cuando necesites lo que **solo** él da: **funcionar sin red y reaccionar a eventos con la app cerrada.**

Y la herramienta práctica: si decidís que sí, **no lo escribas a mano para producción — usá Workbox** (o el plugin PWA de tu framework: Vite PWA, Next PWA, Angular service worker). Entender los módulos 4–11 es lo que te permite **usar esas herramientas con criterio y debuggearlas**, no reinventarlas.

La frase mental de cierre: **el service worker es la herramienta para offline, push e instalabilidad — no un acelerador genérico. Si solo querés velocidad, la caché HTTP es más barata. Si necesitás que la app viva sin red o reaccione con la pestaña cerrada, no hay sustituto — pero asumí el mantenimiento, versioná siempre, y tené un kill switch.**

**Ejercicios 12**
12.1 🧠 Una landing page de marketing, online-only, quiere cargar más rápido. ¿Service worker o caché HTTP + CDN? Justificá.
12.2 🧠 Una app de relevamiento de campo que se usa en zonas sin señal. ¿Paga el service worker? ¿Qué capacidades concretas usarías?
12.3 🧠 Un colega quiere agregar un service worker "porque las PWA son el futuro", sin un caso de offline ni push. ¿Qué le respondés, y qué riesgo concreto le señalás?
12.4 🧠 ¿Por qué para producción conviene Workbox y no un `fetch` handler artesanal, y por qué igual hay que entender el mecanismo crudo?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Se ubica **entre tu app y la red**: es un intermediario (proxy) que puede interceptar cada request HTTP de las páginas bajo su control y decidir qué responder.

**1.2** Cualquiera de: offline / tolerancia a mala conexión, performance vía caché controlada, push notifications, background sync, e instalabilidad como PWA.

**1.3** Porque el service worker es solo un **mecanismo**: no acelera nada por existir. La velocidad y el offline los conseguís **escribiendo la lógica de caché correcta**. Mal escrito, puede dejarte peor (sirviendo contenido viejo, sumando latencia).

**2.1** Porque corre en un **hilo separado del main thread**, fuera del contexto de la página. No tiene `document` ni `window`. Para tocar la UI se comunica con la página vía `postMessage`/`Clients`.

**2.2** Que el service worker es **efímero**: el browser lo termina cuando está ocioso y lo revive ante eventos. Entre dos invocaciones, las variables globales se pierden. El estado tiene que ir a **IndexedDB** o a la **Cache API**, no a memoria.

**2.3** Porque captura sus tres rasgos clave: **sin estado** entre invocaciones, **dirigido por eventos** (lo despiertan push/fetch/sync), y **muere al terminar**. Razonar así (como una Lambda) te evita asumir persistencia de memoria o ejecución continua.

**3.1** Porque un service worker intercepta **toda la red**: sobre HTTP sería un vector ideal para un atacante man-in-the-middle. Solo funciona en contextos seguros (`https://`); la única excepción es `http://localhost`, para poder desarrollar.

**3.2** Porque el **scope lo define el path del archivo**: un `sw.js` en `/static/js/` solo controla `/static/js/...`, no `/`. Lo moverías a la **raíz** (`/sw.js`) para que controle todo el sitio (o, si tenés que dejarlo en la subcarpeta, lo servís con el header `Service-Worker-Allowed: /` para ampliar el scope).

**3.3** No es un bug: en el **primer** registro, la página que lo registró **no queda controlada todavía**; el control empieza en la próxima carga, salvo que uses `clients.claim()`.

**4.1** En **`install`**: precachear el app shell (HTML/CSS/JS/iconos). En **`activate`**: limpiar las cachés de versiones anteriores (y, si querés, `clients.claim()`).

**4.2** Para **garantizar que nunca convivan dos versiones** de la lógica de red. El nuevo espera a que se cierren todas las pestañas controladas por el viejo, así no hay incoherencias entre la lógica vieja y la nueva.

**4.3** Le dice al browser "no des por terminado este evento (ni mates al service worker) hasta que esta promesa resuelva". Es crítico porque el SW es efímero: sin `waitUntil`, el browser podría matarlo a mitad de un precache o de procesar un push.

**4.4** Que podés terminar con una pestaña cargada con HTML/JS **viejo** pero servida por un service worker **nuevo** con otra lógica de caché → respuestas/assets incoherentes. El patrón seguro es avisar al usuario y hacer `skipWaiting()` solo cuando él acepta recargar.

**5.1** Guarda objetos **`Response`** (pares request/response HTTP completos), no JSON ni strings.

**5.2** **No** respeta `max-age` ni `Cache-Control`: lo guardado se queda hasta que vos lo borres. La expiración la administrás vos (o una librería como Workbox).

**5.3** **IndexedDB**, porque son datos estructurados que consultás, filtrás y ordenás. La Cache API es para respuestas HTTP/recursos de red, no para querear datos.

**5.4** Es conveniente porque baja y guarda varios recursos en una llamada. Es **atómico**: si **uno** da 404 (o falla), **toda** la operación falla y el `install` se rechaza, descartándose el service worker. Por eso el app shell tiene que listar solo recursos que existan seguro.

**6.1** El browser hace el request normal a la red, como si el service worker no existiera. Interceptar es **opt-in por request**: solo controlás las que pasás por `respondWith`.

**6.2** Porque el body de una `Response` es un **stream** que se consume **una sola vez**. Si la devolvés a la página y además la guardás en caché, necesitás dos copias → `response.clone()`.

**6.3** Porque cachear mutaciones no tiene sentido (y la Cache API no admite `POST` por defecto), y cachear analytics rompería su propósito (contar eventos reales). Conviene **dejarlas pasar** (no llamar `respondWith`) → red directa.

**6.4** Guardás una respuesta inválida (404/500) y después la servís desde la caché como si fuera buena → caché envenenada, el usuario ve errores persistentes. Por eso se valida `response.ok` antes de `cache.put`.

**6.5** Sin `try/catch`, el `await fetch(...)` **rechaza** (no hay red), ese rechazo llega a `respondWith`, y la request **rompe** con un error de red — el usuario ve la página caída. Es justo el caso que el SW debía cubrir: estar offline pidiendo algo nuevo. Con el `catch` devolvés un fallback (una `/offline.html` precacheada o una `Response` a mano) y la app degrada con gracia en vez de romperse.

**7.1** (a) `app.8f2c.js` → **Cache First** (inmutable por el hash). (b) `index.html` → **Network First** (querés fresco, con fallback offline). (c) avatar → **Stale-While-Revalidate** (un poco viejo está bien). (d) `POST /api/checkout` → **Network Only** (no se cachea).

**7.2** Balance: devuelve la caché **al instante** (rápido) y refresca en segundo plano para la próxima vez. La contra: tras un cambio, en **esa** carga el usuario ve la versión **vieja**; recién la siguiente carga trae la nueva.

**7.3** Porque con hash, si el contenido cambia, **el nombre del archivo cambia** → es realmente inmutable y cache-first es seguro. Sin hash (`app.js`), cache-first te deja sirviendo **para siempre** la versión vieja, porque la URL no cambia aunque el contenido sí.

**7.4** Workbox aporta las estrategias listas, manejo de **expiración y límite de entradas**, routing por patrón de URL, precaching con manifiesto y broadcast de updates — todo lo tedioso y bug-prone. Conviene entender el mecanismo crudo para **debuggear** Workbox y configurarlo con criterio (saber qué estrategia por recurso).

**7.5** Resuelve la **latencia de arranque del service worker**: cuando el SW está dormido (efímero, módulo 2), el browser tiene que arrancarlo antes de poder hacer el `fetch` de la navegación, y ese arranque demora la request más importante (el HTML). Navigation Preload **dispara la request de red en paralelo** mientras el SW arranca, y vos consumís esa respuesta vía `event.preloadResponse`. Aparece justo en SW dormido porque si ya estuviera despierto no habría costo de arranque que ocultar.

**8.1** En **`activate`**: ahí comparás los nombres de caché existentes contra la versión actual y borrás los que no coincidan.

**8.2** Porque (1) la Cache API **no expira sola** (lo precacheado se queda) y (2) el service worker nuevo queda en **waiting** y no toma control mientras haya pestañas viejas abiertas — y mucha gente nunca las cierra (móvil/PWA). Sin un flujo de actualización, el usuario queda congelado.

**8.3** Porque el browser **re-baja `sw.js`** para detectar cambios; si lo cacheás un año, podrías quedar sirviendo el viejo. La **red de seguridad**: el browser igual **ignora** cualquier `Cache-Control` de más de **24h para el propio `sw.js`**, así que la actualización llega como mucho en un día. Pero no dependas solo de eso —son 24h de retraso— : servilo con `no-cache`/`max-age=0` para que el cambio se detecte enseguida.

**8.4** Flujo: al registrar, escuchás `updatefound`; el nuevo worker pasa a `installed` con un `controller` activo (= hay versión vieja corriendo) → mostrás un toast "Nueva versión". Al aceptar el usuario, le mandás `postMessage({type:'SKIP_WAITING'})`; el SW hace `self.skipWaiting()`, dispara `controllerchange`, y ahí hacés `location.reload()`.

**8.5** Le dice al browser que **no use la caché HTTP** para chequear si el `sw.js` (ni sus `importScripts`) cambió: siempre lo valida contra el server. Conviene porque elimina por completo el riesgo de que una caché intermedia te haga servir un service worker viejo, sin depender del header ni del límite de 24h.

**9.1** Que pueden ejecutarse **con la página/app cerrada**: el browser despierta al service worker ante un push entrante, la recuperación de la red, o un timer. No dependen de que haya una pestaña abierta.

**9.2** Cuando el envío falla por estar offline, guardás el mensaje en **IndexedDB** y registrás un sync (`reg.sync.register('enviar-mensajes')`). En el evento `sync` del service worker, leés los pendientes de IndexedDB y los posteás; el browser dispara ese evento **automáticamente al volver la conexión**.

**9.3** Porque su **soporte es muy limitado** (esencialmente Chromium) y está atado al engagement con la PWA y a heurísticas del browser que no controlás. Una feature crítica fallaría en gran parte de los usuarios.

**9.4** En **iOS**, Web Push **solo funciona si la PWA está instalada** en la pantalla de inicio (no en Safari como pestaña normal), y desde **iOS 16.4+**, mientras que en Android/desktop funciona desde el browser sin instalar.

**9.5** La base portable **no se apoya en el evento `sync`** (Chromium-only): guardás la acción pendiente en **IndexedDB** y la drenás en dos disparadores que existen en todos lados — al **reabrir la app** (un check al cargar) y en el handler `fetch` cuando vuelve a haber red. **Background Sync** se suma como **mejora progresiva** donde existe: reintenta automáticamente aunque el usuario no reabra la app. O sea: IndexedDB + reintento manual es el piso garantizado; Background Sync es el bonus.

**10.1** **HTTPS** + un **service worker registrado** + un **manifest válido** (con `name`, `start_url`, `display` e íconos de 192 y 512 px). Las tres juntas; si falta una, el browser no ofrece "Instalar".

**10.2** Porque la privada **firma** los push e identifica a tu server ante el push service: si viajara al cliente, cualquiera podría enviar notificaciones en tu nombre. La **pública** va al cliente (la usa en `pushManager.subscribe`); la **privada** vive **solo en el server** (variable de entorno/secreto).

**10.3** La suscripción está **muerta** (el usuario revocó el permiso, desinstaló la PWA, o el push service la expiró). Hay que **borrarla de tu DB**: si no, acumulás suscripciones zombie y seguís reintentando contra endpoints muertos para siempre.

**10.4**
```ts
app.get('/sw.js', (_req, res) => {
  res.set('Cache-Control', 'no-cache')      // para detectar updates (módulo 8)
  res.set('Service-Worker-Allowed', '/')    // solo si necesitás scope de raíz desde subcarpeta
  res.sendFile(`${process.cwd()}/public/sw.js`)
})
```
La clave: `no-cache` para que el cambio del `sw.js` se detecte enseguida, y `Service-Worker-Allowed` únicamente si el archivo no está en la raíz pero querés que controle todo el origen.

**11.1** Cualquiera de: sin DOM/`window`/`localStorage`; solo HTTPS (o localhost); efímero (sin estado en globals); cuota de almacenamiento limitada y evictable; soporte desigual de APIs satélite.

**11.2** Un service worker que **sirve contenido viejo indefinidamente** porque cacheó agresivo (cache-first) sobre recursos que cambian, sin versionado. Se previene **versionando los nombres de caché**, borrando viejos en `activate`, sirviendo `sw.js` sin caché, y teniendo un flujo de actualización que avisa al usuario.

**11.3** Sirve para **desactivar de emergencia** un service worker roto que afecta a todos los usuarios en producción. Ese `sw.js` de emergencia se **desregistra a sí mismo** (`self.registration.unregister()`) y **borra todas las cachés**, devolviendo a los usuarios al comportamiento normal sin service worker.

**11.4** **Quién controla:** la caché HTTP la maneja el browser según headers del server; la del service worker la maneja **tu código** con lógica arbitraria. **Offline:** la HTTP no da offline programable; el service worker **sí**, porque vos decidís servir desde caché cuando la red falla.

**12.1** **Caché HTTP + CDN.** Para un sitio online-only, los headers de caché y un CDN dan casi toda la velocidad sin el costo del ciclo de vida, versionado y debugging del service worker. Agregar un SW ahí es complejidad sin beneficio proporcional.

**12.2** **Sí paga**, claramente. Usarías: precaching del app shell para que abra sin red, Cache API + IndexedDB para los datos del relevamiento offline, y una **cola en IndexedDB + reintento** (con **Background Sync** como mejora progresiva donde exista) para subir lo capturado automáticamente cuando vuelve la señal. Es justo el caso donde el SW no tiene sustituto.

**12.3** Que el service worker **no es una mejora gratis**: es infraestructura con mantenimiento permanente. Sin un caso concreto de offline o push, el costo (ciclo de vida, versionado, debugging) supera el beneficio. El **riesgo concreto**: dejar usuarios atascados en versiones viejas (service worker zombie) si el versionado no se mantiene bien. Para "solo velocidad", caché HTTP + CDN.

**12.4** Workbox te da estrategias, expiración, routing y manejo de updates probados en producción — escribir eso a mano es tedioso y propenso a bugs sutiles (clonado, expiración, opacas, broadcast). Igual hay que entender el mecanismo crudo (ciclo de vida, fetch, versionado) para **configurar y debuggear** Workbox con criterio, no a ciegas.

---

> **Cierre.** El service worker es la pieza que convierte una página web en algo que puede vivir sin red, reaccionar con la pestaña cerrada e instalarse como una app. Pero ese poder viene con un ciclo de vida exigente y mantenimiento permanente: versioná siempre, serví `sw.js` sin caché, tené un kill switch, y reservalo para cuando necesites lo que **solo** él da. Para todo lo demás, la caché HTTP ya estaba ahí, gratis. Y cuando lo uses en serio, apoyate en **Workbox** — sabiendo, gracias a esta guía, qué hace por debajo.
