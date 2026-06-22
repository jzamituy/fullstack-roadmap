# 📱 Mobile System Design (2026)

> Diseñar el sistema de una app móvil no es "frontend con otro lenguaje". Es un problema de sistemas distribuidos donde **el cliente vive fuera de tu control**: corre versiones viejas, pierde la red, se queda sin batería y guarda estado que después hay que sincronizar. Este módulo toma los conceptos clásicos de *mobile system design* (los típicos de entrevista) y los pone al día a 2026, conectándolos con lo que ya sabés de **React Native**.

Venís de React / React Native, así que tenés ventaja: ya sufriste listas que tironean, estado que no se actualiza y builds eternos. Acá le ponemos nombre a esos problemas y los miramos como decisiones de arquitectura, no como bugs sueltos.

---

## Por qué el mobile system design es su propio mundo

En backend vos controlás el servidor: lo escalás, lo redeployás, lo observás. En una app móvil **casi nada de eso es cierto**. Estas son las restricciones que no existen (o existen mucho menos) del lado del servidor:

- **El cliente es incontrolable y conviven N versiones en producción.** No podés forzar a nadie a actualizar. Una API tuya tiene que seguir respondiéndole bien a una app de hace 18 meses. *"A diferencia de un sitio web, una app móvil no se puede actualizar al instante para todos los usuarios."*
- **La red es intermitente y mentirosa.** No es online/offline binario: hay *lie-fi* (conectado pero sin throughput), transiciones WiFi↔celular, latencia que varía 10× entre una llamada y la siguiente. *Cada request agrega latencia y consume batería.*
- **Los recursos son finitos y compartidos.** Batería, memoria, CPU con límite térmico, y datos móviles que **le cuestan plata al usuario**. El SO te mata el proceso si te excedés.
- **La distribución pasa por las stores.** Review delays, rollouts escalonados, y no podés hacer un hotfix instantáneo… salvo con OTA, que tiene límites (ver Módulo 8).
- **Heterogeneidad brutal.** Miles de modelos, versiones de OS, densidades de pantalla, niveles de RAM.
- **El estado vive en el dispositivo.** Esto convierte a la **sincronización y la resolución de conflictos** en un problema de primera clase, no en un detalle.

### Backend system design vs Mobile system design

| Eje | Backend / Web | Mobile |
|---|---|---|
| Deploy de fixes | Inmediato, vos controlás | Mediado por store + N versiones viejas vivas |
| Estado | Centralizado en servidor/DB | Distribuido en cada dispositivo |
| Red | Asumida estable, baja latencia | Intermitente, alta variabilidad, con costo |
| Recursos | Escalás horizontalmente | Fijos por dispositivo (batería/RAM/térmico) |
| Identidad del problema | Throughput, consistencia, disponibilidad | Offline, sync, batería, tamaño de app, arranque |
| Observabilidad | Logs/trazas que ya tenés | Tenés que instrumentar y *esperar* a que el dato suba |

> 🎯 **Regla mental:** en mobile, todo problema interesante nace de una de estas tres tensiones — *red poco confiable*, *recursos finitos* o *cliente que no podés actualizar*. Cuando en una entrevista no sepas por dónde arrancar, preguntate cuál de las tres aplica.

---

## Módulo 1 — Arquitectura de la app

### Thick client vs thin client (la primera decisión)

Antes de elegir un patrón, decidís **cuánto trabajo vive en el dispositivo**:

- **Thin client** — el servidor hace casi todo; la app solo pinta. Más simple, pero inútil sin red y con más round-trips.
- **Thick (fat) client** — la app guarda estado, valida y calcula localmente. Es la base del **offline-first**: rápido y resiliente, a costa de complejidad (sync, conflictos, migraciones).

La app móvil moderna tiende al **thick client**: el usuario espera que funcione en el subte. Esa decisión arrastra casi todo lo que sigue (Módulos 3 y 4).

### Patrones de presentación

El objetivo siempre es el mismo: **separar la UI de la lógica** para poder testear, reusar y no terminar con un "view controller" de 3000 líneas.

- **MVC** — el origen. En iOS clásico el Controller terminaba siendo un *Massive View Controller*. Punto de partida histórico, no a dónde querés llegar.
- **MVVM** (Model–View–ViewModel) — la View observa un ViewModel que expone estado. Dominante en Android (Jetpack + `ViewModel` + `StateFlow`) y en SwiftUI (`@Observable`).
- **MVI** (Model–View–Intent) — **flujo unidireccional**: la View emite *intents* → se reduce a un nuevo *state* inmutable → la View se re-renderiza. Si venís de Redux/`useReducer`, ya lo conocés. Predecible y fácil de debuggear porque el estado es una sola fuente.
- **VIPER / TCA** (iOS nativo) — VIPER es muy modular y ceremonioso; TCA (*The Composable Architecture*, Swift) es MVI con side-effects explícitos. *Cultura general*: reconocelos en una entrevista, pero como venís de RN no los vas a tocar — tu foco real es **MVI + Clean Architecture**.

> En **React Native** el flujo unidireccional (props bajan, eventos suben) es MVI de fábrica. El "ViewModel" suele ser un hook + store (Zustand/Jotai) o el cache de **TanStack Query**. No tenés que elegir un acrónimo; tenés que poder *nombrar* qué capa hace qué.

### Arquitectura en capas (Clean Architecture)

La idea que importa para una entrevista: **las dependencias apuntan hacia adentro**.

```
┌─────────────────────────────────────────┐
│  Presentation  (UI, ViewModels, hooks)   │  → depende de Domain
├─────────────────────────────────────────┤
│  Domain        (casos de uso, entidades) │  → no depende de nadie (puro)
├─────────────────────────────────────────┤
│  Data          (repos, API, DB local)    │  → implementa interfaces de Domain
└─────────────────────────────────────────┘
```

La capa de **Domain** no sabe si los datos vienen de la red, de SQLite o de un mock. Eso te da: testing sin red, intercambiar la fuente de datos (clave para offline-first) y onboarding más rápido.

### Modularización por features

> **Concepto 40 del post.** App dividida en módulos por feature (`Auth`, `Feed`, `Checkout`) en vez de por capa técnica.

**Por qué importa de verdad:** en un monolito, tocar un archivo recompila todo. Con módulos, **solo recompila lo que cambió** → builds incrementales mucho más rápidos. Airbnb, Uber y Spotify modularizaron principalmente para domar el tiempo de build y el ownership por equipo.

El **costo real** no es dividir, es gobernar los bordes:

- **Dependencias circulares**: `Feed` importa `Checkout` que importa `Feed`. Se rompe con un módulo `:core`/`:shared` y reglas de dependencia.
- Necesitás un **dependency graph** (Gradle module graph, Nx, herramientas de visualización) para ver el grafo y evitar que un módulo "hoja" se vuelva un hub del que depende todo.

```
feature/auth ─┐
feature/feed ─┼─► core/network ─► core/model
feature/cart ─┘            └────► core/storage
```

Regla: las features **no se conocen entre sí**; comparten a través de `core/*`. La navegación entre features se hace por contrato (rutas/deep links), no por import directo.

### Lo que cambió en React Native (2026)

Desde **RN 0.76 (oct 2024) la New Architecture es el default**. Después: la arquitectura vieja se **congeló en 0.80 (jun 2025)** y quedó **always-on / no desactivable en 0.82 (oct 2025)**. Si trabajás con **Expo**, la frontera práctica es el SDK: **SDK 54 fue el último que permitía legacy; SDK 55 (RN 0.83) ya corre solo New Architecture**. Para system design importan tres piezas:

- **JSI** (JavaScript Interface) — JS llama a C++/nativo **directo, sin serializar a JSON** por un "puente".
- **Fabric** — el nuevo renderer; permite layout síncrono y mejor interoperabilidad con vistas nativas.
- **TurboModules** — módulos nativos cargados *lazy*, lo que baja el tiempo de arranque.
- **Bridgeless** — desaparece el viejo *bridge* asíncrono como cuello de botella.

> En una entrevista, el dato útil no es "Fabric existe", sino *por qué*: el viejo bridge serializaba todo a JSON de forma asíncrona, y eso era el origen de jank en gestos/animaciones y de arranques lentos. La New Architecture ataca exactamente esas dos métricas.

---

## Módulo 2 — Networking y diseño de API para mobile

### REST vs GraphQL vs gRPC

| | Cuándo | Pro | Contra |
|---|---|---|---|
| **REST** | Default razonable | Simple, cacheable por HTTP, universal | Over/under-fetching; varias llamadas por pantalla |
| **GraphQL** | Pantallas que arman datos de muchas fuentes | El cliente pide exactamente lo que necesita; una sola llamada | Caché más compleja; costo en el servidor |
| **gRPC** | Comunicación interna / streaming / baja latencia | Binario (Protobuf), rápido, contrato fuerte | No nativo en browser; más fricción en mobile |

En mobile el peso de cada decisión se multiplica porque **cada request cuesta batería y datos**. La pregunta no es "¿cuál es mejor?" sino "¿cuántos round-trips hago para pintar esta pantalla?".

> Sobre **gRPC en mobile**: rara vez es la API *de cara al cliente* (gRPC-Web todavía tiene fricción en el dispositivo). Aparece más bien **interno, detrás del BFF**, entre microservicios, o para *streaming* de baja latencia. No lo sobreestimes como reemplazo de REST/GraphQL para tu app.

### Backend for Frontend (BFF)

> **Concepto 41 del post.** Una capa de backend dedicada al cliente móvil.

El BFF **agrega** datos de varios microservicios y devuelve **una sola respuesta optimizada para esa pantalla**. Netflix mantiene BFFs separados para iOS, Android, TV y web, porque cada plataforma necesita forma y tamaño de payload distintos.

```
[App] ──1 request──► [BFF mobile] ──► svc perfil
                                  ├──► svc catálogo
                                  └──► svc recomendaciones
        ◄── 1 respuesta a medida ──┘
```

- **Gana:** menos round-trips (= menos batería/latencia), payload a medida, el cliente no orquesta nada.
- **Cuesta:** un servicio más que mantener, y el riesgo de **meter lógica de negocio en el BFF**. Regla: el BFF *agrega y adapta*, el negocio vive en los servicios de origen.
- Conexión con tu mundo: un BFF es justo el tipo de servicio para el que **NestJS** (gateway/aggregator) o **FastAPI** encajan perfecto.

### Paginación

- **Offset/page** (`?page=2&size=20`): simple pero se rompe si la lista cambia mientras scrolleás (ítems duplicados o salteados) y es cara en DB con offsets grandes.
- **Cursor-based** (`?after=<cursor>`): el cliente manda un puntero opaco al último ítem visto. Estable ante inserciones, eficiente. **Es el default para feeds infinitos.**

### Eficiencia de red (acá se gana la batería)

- **Compresión** (gzip/Brotli) y **HTTP/2/3** para multiplexar y reusar conexión (el handshake TLS es caro en celular).
- **Backoff exponencial + jitter** en reintentos para no hacer *thundering herd* cuando vuelve la red.
- **Idempotency keys** en mutaciones (POST de pago) para que un reintento no cobre dos veces. La clave se genera **una sola vez al crear la mutación** y se persiste junto a ella; los reintentos reusan la misma (regenerarla en cada intento rompe la idempotencia).
- **Coalescing / dedup**: si tres pantallas piden el mismo dato al toque, una sola request. (TanStack Query lo hace por vos.)
- **Persisted queries** (GraphQL): mandás un hash en vez del query completo → menos bytes subiendo.
- **Caché HTTP** (`Cache-Control`, `ETag` / `If-None-Match`): para datos poco cambiantes, dejá que el protocolo trabaje — un `304 Not Modified` no transfiere body. Es la capa de caché *gratis* antes de cualquier caché de app.

```ts
// Reintento con backoff + jitter — el patrón que se repite en todo cliente móvil serio
async function fetchWithRetry(url: string, opts: RequestInit, max = 4): Promise<Response> {
  for (let attempt = 0; ; attempt++) {
    try {
      const res = await fetch(url, opts);
      if (res.status < 500) return res;        // 4xx no se reintenta
      if (attempt >= max) return res;
    } catch (err) {
      if (attempt >= max) throw err;            // red caída
    }
    const base = Math.min(1000 * 2 ** attempt, 16_000);
    const jitter = Math.random() * base;        // evita el thundering herd
    await new Promise((r) => setTimeout(r, jitter));
  }
}
```

### Subir archivos grandes: resumable uploads

Subir una foto o video de 50 MB por celular y que se corte al 90% es la regla, no la excepción. La solución es **resumable / chunked uploads**: partís el archivo en *chunks*, subís cada uno con su offset, y si se corta, **retomás desde el último chunk confirmado** en vez de empezar de cero. El protocolo de referencia es **tus.io**; el *multipart upload* de S3/GCS implementa la misma idea.

---

## Módulo 3 — Offline-first y sincronización (el corazón del tema)

Es el módulo que separa una app "que tiene caché" de una **offline-first**. Pensalo como una escalera de madurez:

1. **Read cache** — guardás la última respuesta y la mostrás mientras refrescás.
2. **Offline read** — la app abre y muestra datos útiles sin red.
3. **Offline write** — el usuario crea/edita estando offline; encolás las mutaciones.
4. **Full sync bidireccional** — cliente y servidor convergen solos, con resolución de conflictos.

### Estrategias de caché

- **Cache-then-network**: pintás del cache al instante y luego actualizás (lo que sentís como "rápido").
- **Stale-while-revalidate**: servís lo viejo, refrescás en background, swappeás. Es el modelo de **TanStack Query**, que ya es estándar de facto en RN.
- **Invalidación de caché** (el problema difícil de verdad): **TTL** (expira por tiempo), **ETag/versión** (revalidás contra el server) o **event-based** (un push marca el dato como sucio). Sin política de invalidación, el offline-first te muestra datos viejos para siempre.

### Escrituras offline: el outbox / mutation queue

El usuario toca "enviar" sin red. Vos:

1. Aplicás un **optimistic update** (la UI ya muestra el cambio).
2. Encolás la mutación en una **outbox** persistente (SQLite).
3. Cuando vuelve la red, drenás la cola **en orden**, con idempotency keys.
4. Si el server rechaza, hacés **rollback** del optimistic update y avisás.

```ts
type PendingMutation = {
  id: string;            // idempotency key (lo genera el cliente)
  type: 'CREATE_POST' | 'LIKE' | 'DELETE';
  payload: unknown;
  createdAt: number;
  retries: number;
};

// La outbox vive en SQLite, no en memoria: sobrevive a que el SO mate el proceso.
async function drainOutbox(queue: PendingMutation[]) {
  for (const m of queue.sort((a, b) => a.createdAt - b.createdAt)) {
    try {
      await api.send(m.type, m.payload, { idempotencyKey: m.id });
      await outbox.remove(m.id);
    } catch (e) {
      if (isPermanent(e)) { await outbox.remove(m.id); rollback(m); }
      else break; // error transitorio: corto y reintento en el próximo ciclo
    }
  }
}
```

### Resolución de conflictos

Dos dispositivos editan el mismo dato offline. ¿Quién gana?

- **Last-Write-Wins (LWW)** — el timestamp más nuevo gana. Simple pero **pierde datos** silenciosamente.
- **Version vectors / `updatedAt` + detección** — detectás el conflicto y decidís (o se lo mostrás al usuario).
- **CRDTs** (Conflict-free Replicated Data Types) — estructuras que **convergen sin conflicto por diseño**. Para texto colaborativo y listas: **Yjs**, **Automerge**. Es lo correcto cuando "el último gana" no alcanza (ej. edición colaborativa).

> **No es "CRDT siempre gana".** LWW es perfectamente válido para campos escalares independientes (un toggle, un nombre, una preferencia). Los CRDT recién valen la pena cuando hay edición concurrente de **estructura o texto**, y no son gratis: arrastran metadata/*tombstones* y el documento crece con el tiempo.

### Delta sync y consistencia eventual

- **Delta sync**: al sincronizar, no mandes el dataset entero — mandá **solo lo que cambió desde el último sync** (un cursor / `updatedAt` / changeset). Menos bytes, menos batería, sync más rápido. Es lo que hacen por dentro los sync engines de abajo.
- **Consistencia eventual**: en offline-first asumís que cliente y servidor **divergen temporalmente** y convergen cuando vuelve la red. La UI tiene que tolerar ese desfasaje (mostrar "sincronizando…", no bloquear). No busques consistencia fuerte en mobile: es cara y rompe la experiencia offline.

### Sync engines (el panorama 2026)

Esto es lo que más cambió y donde podés sonar actualizado:

- **PowerSync** — mantiene una SQLite local en sync con Postgres/MongoDB/MySQL/SQL Server (madurez dispar: **Postgres** sólido, **Mongo** GA, **MySQL** beta, **SQL Server** alpha — verificá el estado antes de comprometerte). Leés/escribís local y sincroniza en background. Consiguió **SOC 2 e HIPAA en enero 2026** (señal de madurez para enterprise).
- **ElectricSQL** — sync open-source de Postgres → SQLite local, con resolución de conflictos.
- **Zero (Rocicorp)** y **Replicache** — sync con foco en latencia y reactividad.
- **WatermelonDB** — base reactiva sobre SQLite **pensada para React Native** con datasets grandes (funciona bien con Expo SDK 54+).
- **Legend-State + Legend-Sync** — estado observable con sync, popular en RN.

**¿Cuál elegir?** A grandes rasgos:

| Si tu caso es… | Mirá… |
|---|---|
| Backend SQL existente (Postgres/Mongo) y querés sync transparente | **PowerSync**, **ElectricSQL** (SQL-first) |
| App RN con datasets grandes y modelo reactivo propio | **WatermelonDB** |
| Edición colaborativa / local-first con merge automático | CRDTs (**Yjs/Automerge**), **Zero / Replicache** |
| Ya manejás estado observable y solo querés sumarle sync | **Legend-State + Legend-Sync** |

> ⚠️ **Cambio importante 2026:** **MongoDB Realm / Atlas Device Sync llegó a su End-of-Life el 30 de septiembre de 2025.** El SDK local de Realm sigue como open source, pero **el sync en la nube se apagó**. Si en una entrevista o en código viejo ves "Realm Sync", la respuesta correcta hoy es: migrar a **Couchbase Lite**, **PowerSync** o un sync propio. No propongas Atlas Device Sync para un proyecto nuevo.

> 🧪 **Cómo se testea esto:** la capa de sync es no-determinista por naturaleza. Se prueba con **red simulada** (online/offline/lie-fi a voluntad) y **reloj controlado** (para forzar el orden de timestamps y reproducir conflictos a pedido). Conecta con tu lado de [Testing práctico](testing.md): fakes de red + *time-travel* de conflictos en vez de esperar a que pasen "en vivo".

---

## Módulo 4 — Almacenamiento local y persistencia

Qué herramienta para qué dato:

| Necesidad | Opción |
|---|---|
| Datos estructurados / queries | **SQLite** (la base de casi todo): `expo-sqlite`, `op-sqlite`, Room (Android), GRDB (iOS) |
| Key-value rápido (flags, settings) | **MMKV** (RN), `UserDefaults` (iOS), `SharedPreferences`/DataStore (Android) |
| Secretos (tokens, claves) | **Keychain** (iOS) / **Keystore** (Android), vía `expo-secure-store`. **Nunca** AsyncStorage para tokens. |
| Objetos reactivos / ORM móvil | WatermelonDB, Realm (local), **SwiftData** (iOS moderno, reemplaza Core Data) |

Puntos finos que distinguen a un senior:

- **Migraciones de schema**: la app vieja y la nueva conviven; necesitás migraciones versionadas y a prueba de fallos (la migración corre en el arranque, no la podés "rollbackear" en el dispositivo del usuario).
- **Cifrado at-rest**: **SQLCipher** para SQLite sensible; el secure storage del SO para las claves.
- **Qué NO guardar**: PII innecesaria, tokens en almacenamiento no cifrado, datos que cambian todo el tiempo (mejor cache con TTL).

---

## Módulo 5 — Estado, rendimiento y batería

### Métricas que importan

- **Tiempo de arranque**: *cold* (proceso desde cero), *warm*, *hot*. El cold start es el que más duele y el que miran las stores.
- **TTI (Time To Interactive)**: cuándo el usuario *puede tocar algo útil*, no cuándo apareció el splash.
- **Frame budget**: 60fps = **16,6 ms por frame**; 120fps (ProMotion/pantallas modernas) = **8,3 ms**. Pasarte de ese presupuesto = *jank* (tirones).

### Las dos fuentes clásicas de jank

1. **Listas largas sin virtualizar.** Renderizar 10.000 filas mata la memoria. Solución: virtualización — **FlashList** (RN, mejor que `FlatList`), `LazyColumn` (Compose), `UICollectionView` con prefetch (iOS).
2. **Trabajo pesado en el hilo principal.** Parseo/imagen/crypto en el UI thread. Movelo a workers/hilos de fondo.

### Ejecución en background y batería

El SO **defiende la batería de tu app**:

- **Doze / App Standby** (Android) y límites de background (iOS) suspenden tu proceso cuando la pantalla está apagada.
- Para trabajo diferible y garantizado: **WorkManager** (Android), **BGTaskScheduler** (iOS). No abras sockets ni hagas polling "a mano" en background.
- **Batchea** red y wakeups: 10 requests juntas gastan mucho menos radio que 10 espaciadas (la antena celular se queda "encendida" un rato tras cada uso — *tail energy*).

---

## Módulo 6 — Imágenes y media

En un feed, **las imágenes son el 80% de los bytes y de la memoria**. Diseñar bien acá es diseñar la app.

- **Librerías de carga+caché** (no reinventes): **Expo Image** / Coil (Android) / Nuke / SDWebImage (iOS). Dan caché en memoria + disco, dedup y decodificación off-thread. `react-native-fast-image` quedó **deprecado**; en RN moderno se usa **Expo Image**.
- **Formato**: **WebP** y **AVIF** pesan mucho menos que JPEG/PNG a igual calidad.
- **Tamaño responsivo desde el CDN**: pedí la imagen al tamaño del contenedor (`?w=320`), no la de 4K para mostrarla en un thumbnail.
- **Placeholders**: *blurhash*/thumbnail diminuto mientras carga → percepción de velocidad.
- **Prefetch** de las próximas imágenes del feed mientras el usuario lee las actuales.

---

## Módulo 7 — Push notifications y tiempo real

### Push

- Canales nativos: **APNs** (Apple) y **FCM** (Android/cross). Ojo dato 2026: la **API legacy de FCM se discontinuó (jun 2024)** → hay que usar **FCM HTTP v1**. En Expo, **Expo Push** abstrae ambos.
- **Notification messages** (las muestra el SO) vs **data messages / silent push** (despiertan tu código para sincronizar en background, con límites estrictos del SO).
- **Ciclo de vida del token**: registrás, el token rota, lo refrescás y lo limpiás en logout. Mandar push a tokens viejos ensucia métricas.
- **La entrega NO está garantizada.** Diseñá idempotente y no asumas que la push llegó: la verdad la define un *sync* al abrir, no la notificación.
- **Live Activities** (iOS) para estados en vivo (pedido, partido) en pantalla de bloqueo.

### Tiempo real

**WebSocket** (bidireccional) para chat y presencia; **SSE** (solo server→cliente, y en RN normalmente necesitás un polyfill de `EventSource`) para streams de solo lectura como precios en vivo; **long-poll** como fallback. En mobile, lo difícil **no es abrir el socket, es mantenerlo**: reconexión con backoff, heartbeats, y soltar/recuperar la conexión cuando la app va a background (el SO te la corta). Conecta directo con tu módulo de **Tiempo real (WebSockets)**.

---

## Módulo 8 — Versionado, backward compatibility y releases

> El módulo donde "no podés actualizar al cliente" se vuelve estrategia.

### Versionado de API y Expand–Contract

> **Conceptos 42–43 del post.**

Como conviven apps viejas y nuevas, versionás (`/v1/users`, `/v2/users`) y migrás con **Expand–Contract** (igual que migraciones de DB sin downtime):

1. **Expand** — agregás lo nuevo *sin romper* lo viejo (campo nuevo opcional, endpoint v2).
2. **Migrate** — los usuarios actualizan la app con el tiempo.
3. **Contract** — cuando el uso de lo viejo cae por debajo de un umbral, lo retirás.

Stripe es el ejemplo canónico de API versionada que nunca rompe clientes viejos.

### Forced update / soft update / kill switch

- **Soft update**: banner "hay versión nueva".
- **Forced update**: pantalla bloqueante si la versión es menor a un mínimo soportado (lo decide el servidor o remote config).
- **Kill switch**: apagar remotamente una feature rota sin esperar review de la store. Esto es, literalmente, un feature flag.

### Feature flags & Remote Config

> **Concepto 43.** Encender/apagar features y cambiar textos/límites/layout **sin publicar**.

- Herramientas: **Firebase Remote Config**, LaunchDarkly, Statsig, ConfigCat.
- Habilitan **A/B testing** (Airbnb los usa fuerte) y rollouts por porcentaje.
- **Deuda de flags**: si no tienen ciclo de vida (dueño + fecha de baja), el código se llena de `if (flag)` muertos. Política: todo flag nace con fecha de defunción.

### Staged / phased rollouts

Publicás al **1% → 10% → 50% → 100%**, mirando crash-free rate entre escalones (ver Módulo 10). Tanto las stores (Google Play staged rollout, iOS phased release) como los feature flags te lo dan. Si la métrica cae, **frenás o revertís**.

### OTA updates (over-the-air) — panorama 2026

Actualizar el **bundle JS/assets sin pasar por la store**. Cambios clave:

- **Microsoft App Center se retiró el 31 de marzo de 2025**, llevándose el CodePush hosteado. Microsoft liberó el server de **CodePush como open source** → podés self-hostearlo.
- **EAS Update (Expo)** es hoy la opción más sólida para la mayoría. **Expo SDK 55 (feb 2026, RN 0.83 — ya solo New Architecture, ver Módulo 1)** sumó **diffing de bytecode Hermes** (updates más chicos), **Rollouts** (liberar a un % de usuarios) y **Republish/rollback** (revertir un update malo).
- **Límite legal/store**: OTA solo para **JS y assets**, no código nativo, y **sin cambiar el propósito de la app** (regla de Apple/Google). No uses OTA para meter una feature que la store rechazaría.

### Deep linking

> **Concepto 44.** Abrir una pantalla concreta desde una URL.

- **URI schemes** (`miapp://producto/42`) — frágiles, no verifican dominio.
- **Universal Links** (iOS) / **App Links** (Android) — basados en HTTPS + verificación de dominio (`apple-app-site-association` / `assetlinks.json`); si la app no está, abre la web. Es el estándar.
- **Deferred deep linking** — el usuario toca un link, **instala** la app, y al abrir por primera vez aterriza en el destino correcto. Clave para growth/atribución (Branch, AppsFlyer).

---

## Módulo 9 — Seguridad mobile

Premisa: **el binario de tu app está en manos del atacante.** Se puede descompilar. Por lo tanto:

- **Nunca secretos en el bundle/código.** Cualquier API key embebida es pública. Los secretos viven en el servidor.
- **Secure storage** para tokens: Keychain/Keystore (`expo-secure-store`), nunca en texto plano.
- **Tokens**: access token corto + refresh token; el refresh en secure storage.
- **Certificate / public-key pinning**: la app solo confía en *tu* certificado → frena man-in-the-middle con un proxy. Ojo: hay que poder rotarlo o brickeás la app.
- **Attestation**: **App Attest** (iOS) y **Play Integrity API** (Android) le prueban al servidor que la request viene de una app genuina en un dispositivo no comprometido (anti-bots, anti-tampering).
- **Biometría**: `LocalAuthentication` (iOS) / `BiometricPrompt` (Android) para gatear acciones sensibles.
- **Ofuscación**: R8/ProGuard (Android) sube el costo de reversear; no es seguridad por sí sola.
- **Marco de referencia**: **OWASP MASVS / MASTG** — el checklist estándar de seguridad móvil. Citálo en una entrevista.

> ⚠️ **Ninguna de estas defensas es inviolable.** En un dispositivo rooteado/jailbreakeado o con herramientas como **Frida**, el pinning, la attestation y la ofuscación se pueden saltar. No *garantizan* seguridad: **suben el costo del ataque**. La verdad de seguridad vive en el servidor; el cliente solo eleva la barrera.

### Privacidad y permisos (2026)

> **Conceptos 46, 50 del post.**

- **Permisos en runtime**: pedilos *en contexto* y justo cuando se usan; manejá el "denegado" con degradación elegante. Para ubicación, ofrecé **precisa vs aproximada**.
- **App Tracking Transparency (ATT)** en iOS: consentimiento para tracking cross-app.
- **Privacy manifests** (iOS 17+): obligatorios; las SDKs deben declarar qué datos recolectan y por qué.
- **Data Safety** (Google Play) y **scoped storage** (Android): acceso acotado al almacenamiento del usuario.

---

## Módulo 10 — Observabilidad y operación post-ship

> Los conceptos finales del post viven acá: **shipear es el principio, no el final.**

### Crash reporting

- **Crashlytics**, **Sentry**, Embrace, Bugsnag.
- **Symbolication**: los stack traces vienen ofuscados; subís los **dSYM** (iOS) y el **mapping de ProGuard/R8** (Android) para des-ofuscarlos. Sin esto, los crashes son ilegibles.
- **ANR** (Application Not Responding, Android): el hilo principal bloqueado >5s. Métrica propia, separada de crashes.

### Crash-free rate como SLI

> **Concepto del post: crash-free rate como SLI.**

Conecta directo con tu módulo de **Observabilidad** (SLI/SLO):

- **SLI**: `crash-free sessions` o `crash-free users` (ej. 99,7% de sesiones sin crash).
- **SLO**: el objetivo (ej. ≥99,5%). Es la métrica que **gobierna los staged rollouts**: si en el 10% el crash-free baja del SLO, **se frena el rollout automáticamente**.

### Performance monitoring / RUM

- **RUM** (Real User Monitoring): arranque, latencia de red real, *frozen frames*, *slow frames*, vistos en dispositivos reales, no en tu emulador.
- Herramientas: **Firebase Performance**, **Sentry/Datadog RUM**, y cada vez más **OpenTelemetry** para mobile (trazas que cruzan app → BFF → servicios, unificando con tu observabilidad de backend).

### Releasing safely (el loop completo)

```
feature flag OFF en prod
   → staged rollout 1% + flag a cohorte interna
   → mirar crash-free rate / ANR / errores de red
   → ¿supera SLO? subir a 10% → 50% → 100%
   → ¿no lo supera? kill switch (flag OFF) — sin esperar a la store
```

---

## Módulo 11 — Accesibilidad, i18n y degradación elegante

> **Conceptos 47–49 del post.** Lo que diferencia una demo de un producto.

- **Accesibilidad (a11y)**: labels para lectores de pantalla (**VoiceOver** / **TalkBack**), respeto del *Dynamic Type* (tamaño de fuente del sistema), contraste, áreas táctiles ≥44pt. En RN: `accessibilityLabel`, `accessibilityRole`.
- **i18n / l10n**: pluralización (ICU MessageFormat), formatos de fecha/número por locale, y **RTL** (árabe/hebreo invierten el layout entero — `I18nManager` en RN).
- **Degradación elegante**: cada pantalla necesita estados de **vacío**, **error** y **offline**, con *skeletons* y botón de reintento. Mostrar **datos parciales** > pantalla en blanco. Si falla la sección de "recomendados", el feed igual carga.

---

## Marco para una entrevista de mobile system design

Cuando te tiran "diseñá el feed de Instagram" / "diseñá WhatsApp" / "diseñá Spotify offline", seguí este orden y vas a cubrir lo que el entrevistador busca:

1. **Aclarar requisitos.** Funcionales (qué hace) y **no funcionales** (¿offline? ¿tiempo real? ¿escala? ¿qué OS mínimos?). *Acá es donde demostrás que pensás en mobile.*
2. **API y modelo de datos.** Endpoints, forma del payload, ¿BFF?, paginación cursor.
3. **Arquitectura del cliente.** Capas (presentation/domain/data), patrón (MVVM/MVI), modularización.
4. **Estrategia offline/sync.** ¿Qué nivel de la escalera del Módulo 3? Cache, outbox, conflictos.
5. **Deep dives** según la app: feed → imágenes + scroll virtualizado + prefetch; chat → WebSocket + push + entrega; media → streaming + caché.
6. **Transversales:** seguridad (tokens, pinning), rendimiento (arranque, jank), batería y **costo** — los datos móviles los paga el usuario, el egress de CDN lo pagás vos, y mantener N versiones de API y feature flags vivos es deuda operativa. En entrevistas senior, nombrar el costo de cada decisión suma.
7. **Operación:** feature flags, staged rollout, crash-free SLI, OTA.

> 💡 El error típico es saltar directo a "uso Redux y FlatList". El entrevistador quiere ver que entendés las **tres tensiones** (red, recursos, cliente incontrolable) y que tus decisiones salen de ahí.

---

## Ejercicios

> Resolvé primero, después mirá las **Soluciones** al final. Son de diseño: no hay una única respuesta correcta, sino trade-offs bien justificados. Los **ejercicios 11-14** (puente full stack y offline) cierran su solución con una **🚩 red flag** — el error que delataría a un junior — para que te autoevalúes.

**1.** Tu API móvil recibe el reporte de que usuarios con la app de hace un año empezaron a recibir errores 400 tras un deploy. ¿Qué violaste y cómo lo arreglás sin romper a nadie?

**2.** Una pantalla de detalle de producto hace 5 llamadas distintas (precio, stock, reviews, recomendados, perfil del vendedor) y tarda 2,5 s en celular. Proponé una arquitectura para bajar eso, con su trade-off.

**3.** Diseñá la escritura offline de un "me gusta": el usuario lo toca sin red, cierra la app, vuelve la red 3 horas después. Listá los componentes y qué pasa si el server responde 409 (ya no existe el post).

**4.** Dos usuarios editan offline el mismo documento colaborativo. Compará LWW vs CRDT para este caso y elegí, justificando.

**5.** En 2026, un proyecto nuevo te pide "usá Realm Sync para offline como en el tutorial". ¿Qué le respondés y qué proponés?

**6.** El feed tironea (jank) al scrollear rápido. Enumerá las dos causas más probables y la solución concreta de cada una en React Native.

**7.** Tenés que lanzar una feature riesgosa a 50 millones de usuarios. Describí el pipeline de release para poder apagarla en minutos si algo sale mal, sin pasar por review de la store.

**8.** ¿Por qué no podés guardar el access token en `AsyncStorage`/`UserDefaults` y dónde va? ¿Y por qué una API key "secreta" embebida en la app no es secreta?

**9.** Definí un SLI y un SLO de estabilidad para tu app y explicá cómo ese SLO gobierna un staged rollout.

**10.** Te piden meter una feature nueva "por OTA, sin pasar por la store, para mañana". ¿Cuándo es legítimo y cuándo te van a rechazar/banear? ¿Qué herramienta usás en 2026?

**11.** *(Puente full stack)* Tomá el "me gusta" offline del ejercicio 3 y diseñá el **lado servidor en NestJS**: el endpoint que recibe la mutación, cómo garantizás idempotencia con la key del cliente, y el contrato de **delta sync** que le devuelve al cliente solo lo que cambió desde su último `updatedAt`.

**12.** *(Puente full stack)* La home hace 6 llamadas (perfil, feed, stories, notificaciones, banners, saldo) y tarda en celular. Diseñá un **BFF en NestJS** que las colapse en una respuesta: el contrato de salida, cómo paraleliza las 6 llamadas internas, y qué hace si **una falla** (degradación parcial vs error total).

**13.** *(Offline)* Diseñá una **mutation queue completa** con rollback, más allá del caso simple: los estados de cada mutación, el orden de drenado, la política de reintentos, y el caso difícil — el usuario crea un post offline y **comenta ese mismo post** antes de que el primero sincronice (dependencia entre mutaciones).

**14.** *(Offline)* Elegí y justificá la estrategia de conflicto para tres dominios: (a) el contador de likes de un post, (b) un mensaje de chat, (c) un documento de texto colaborativo. Decidí entre LWW, contador conmutativo, append-only o CRDT — sin caer en "CRDT para todo".

---

## Soluciones

**1 — Backward compatibility.** Rompiste un contrato del que dependen apps viejas que **no podés actualizar**. Probablemente hiciste un cambio *breaking* (campo obligatorio nuevo, tipo cambiado, endpoint removido) sin versionar. Arreglo con **Expand–Contract**: revertí el breaking en la versión actual del endpoint, hacé el cambio nuevo *aditivo* (campo opcional o `/v2`), dejá conviviendo ambas, y recién retirá la vieja cuando la telemetría muestre que casi nadie la usa. Lección: en mobile, **toda API es para siempre** hasta que las métricas digan lo contrario.

**2 — BFF + composición.** Metés un **Backend for Frontend** que hace las 5 llamadas en paralelo *server-side* y devuelve un solo payload a medida de esa pantalla → 1 round-trip en vez de 5, menos latencia y batería. Trade-off: un servicio más para mantener y el riesgo de filtrar lógica de negocio al BFF (debe solo agregar/adaptar). Alternativa si ya tenés GraphQL: una sola query con los 5 campos. Bonus: lo no crítico (recomendados) puede venir *lazy* o degradar si falla.

**3 — Outbox / mutation queue.** Componentes: (a) **optimistic update** que pinta el like al instante; (b) **outbox persistente en SQLite** con la mutación + **idempotency key** generada por el cliente + `createdAt`; (c) un **drainer** que se dispara cuando vuelve la conectividad y manda en orden con la key (así un reintento no duplica el like). Si el server responde **409** (el post ya no existe): es un error **permanente** → sacás la mutación de la outbox, hacés **rollback** del optimistic update y, si corresponde, avisás al usuario suavemente. Los 5xx/timeouts, en cambio, se reintentan con backoff.

**4 — LWW vs CRDT.** Para un **documento colaborativo**, LWW está mal: si gana "el último timestamp", **perdés las ediciones del otro** sin avisar. Elegí **CRDT** (Yjs/Automerge): las dos ediciones convergen al mismo estado sin perder caracteres, que es justo lo que un editor colaborativo necesita. LWW sería aceptable solo para datos donde la última intención reemplaza a la anterior (ej. "toggle de notificaciones"), no para texto que se *fusiona*.

**5 — Realm Sync está EOL.** Le decís que **Atlas Device Sync llegó a End-of-Life el 30 de septiembre de 2025**: el SDK local sigue como open source, pero **el sync en la nube ya no existe**, así que no es opción para un proyecto nuevo. Proponés un sync engine vigente: **PowerSync** (SQLite ↔ Postgres/Mongo, con SOC2/HIPAA desde ene-2026), **ElectricSQL**, o **WatermelonDB** si querés algo nativo de RN; **Couchbase Lite** es el camino de migración que recomienda el propio ecosistema Realm.

**6 — Jank en el feed.** Causa 1: **lista no virtualizada** renderizando todas las filas → usá **FlashList** en vez de `FlatList`/`ScrollView` (recicla vistas, menos memoria). Causa 2: **trabajo pesado en el JS/UI thread** (parseo, decodificación de imágenes grandes, cálculos en `render`) → mové la decodificación a la librería de imágenes (Expo Image, off-thread), memoizá, y sacá cómputo del render. Verificás con el profiler de la New Architecture mirando frames que pasan los 16,6 ms.

**7 — Pipeline de release con escape hatch.** La feature nace detrás de un **feature flag apagado** en prod. La activás primero para una **cohorte interna**, después **staged rollout** 1% → 10% → 50% → 100% mirando crash-free rate y errores entre escalones. Si algo explota, **kill switch**: apagás el flag desde Remote Config y la feature desaparece **en minutos, sin pasar por la store**. El flag te da el apagado instantáneo que un release normal no puede dar.

**8 — Secure storage y binario abierto.** El access token va en **Keychain (iOS) / Keystore (Android)** vía `expo-secure-store`, no en AsyncStorage/UserDefaults, que son texto plano accesibles en un device rooteado/con backup. Y una "API key secreta" embebida **no es secreta** porque el binario se puede descompilar y extraerla: todo lo que viaja en la app es público. Los verdaderos secretos viven en el servidor; la app se autentica y, si necesitás probar que es genuina, usás **App Attest / Play Integrity**.

**9 — SLI/SLO de estabilidad.** **SLI**: `crash-free sessions` (% de sesiones sin crash). **SLO**: ≥ 99,5% (ejemplo). Cómo gobierna el rollout: definís que cada escalón del staged rollout solo avanza si el crash-free de esa cohorte se mantiene ≥ SLO; si baja (regresión introducida por la nueva versión), el rollout **se frena o revierte automáticamente** antes de llegar al 100%. El SLO convierte "parece que anda" en una compuerta objetiva.

**10 — OTA legítimo vs prohibido.** **Legítimo**: arreglar un bug en JS, cambiar textos/estilos, togglear features ya revisadas — todo lo que sea **JS y assets**. **Te rechazan/banean** si por OTA cambiás **código nativo** o **alterás el propósito de la app** respecto a lo que la store aprobó (regla de Apple/Google). Herramienta 2026: **EAS Update** (con rollouts y republish/rollback desde Expo SDK 55), o **CodePush self-hosteado** ahora que App Center se retiró (31-mar-2025).

**11 — El lado servidor del like (NestJS + delta sync).** El endpoint (`POST /likes`) recibe la mutación con la **idempotency key** del cliente en un header (`Idempotency-Key`). En el server guardás esa key en una tabla con índice único (o en Redis con TTL): si llega repetida, devolvés el resultado ya guardado en vez de re-ejecutar → el reintento del cliente es seguro. Para el **delta sync**, el cliente manda su último cursor (`?since=<updatedAt>`) y devolvés solo las filas con `updated_at > since` (más los *tombstones* de lo borrado) y un cursor nuevo. En Postgres, índice en `(updated_at)` para que el delta sea barato. 🚩 **Red flag:** un endpoint que no es idempotente "porque el cliente ya no reintenta" — en mobile *siempre* reintenta.

**12 — BFF de la home (NestJS).** El BFF expone `GET /home` y por dentro dispara las 6 llamadas **en paralelo** con `Promise.allSettled` (no `all`). El contrato de salida es un objeto con una clave por *sección* de la pantalla. Con `allSettled`, una llamada que falla **no tumba la respuesta**: esa sección viene `null`/vacía y el cliente la degrada (Módulo 11) mientras el resto carga. Devolvés `200` con secciones parciales, no un `500` total — salvo que falle algo esencial (el perfil). Trade-off: el BFF es un punto único que hay que escalar y observar, y el negocio debe quedarse en los servicios de origen. 🚩 **Red flag:** usar `Promise.all` (un fallo tira las 6) o encadenar las llamadas en serie (sumás las latencias).

**13 — Mutation queue con dependencias.** Cada mutación tiene estado (`pending → sending → done | failed`), `createdAt` para ordenar, `retries` y backoff. El drenado es **secuencial y FIFO** para preservar la causalidad. El caso difícil (comentar un post aún no sincronizado) se resuelve con **IDs de cliente**: el post offline nace con un UUID local, el comentario referencia ese UUID, y cuando el post sincroniza el server devuelve el ID real y **remapeás** las referencias pendientes (o el server acepta el UUID de cliente como idempotency key y resuelve el grafo). Si una mutación con dependientes falla de forma permanente, el rollback es **en cascada**. 🚩 **Red flag:** drenar en paralelo o sin orden → el comentario llega antes que el post y el server lo rechaza.

**14 — Estrategia de conflicto por dominio.** (a) **Contador de likes**: ni LWW ni CRDT de texto — un **contador conmutativo** (cada cliente manda el delta `+1/-1`, el server suma); LWW perdería incrementos concurrentes. (b) **Mensaje de chat**: **append-only**, no hay conflicto real — cada mensaje es inmutable con su ID y timestamp, ordenás por la hora del server. (c) **Documento colaborativo**: **CRDT** (Yjs/Automerge), porque hay edición concurrente de texto que debe *fusionarse*. Moraleja: la estrategia sale del **tipo de dato y de operación**, no de una regla única. 🚩 **Red flag:** proponer CRDT para los tres "porque es lo más avanzado" — es overkill para un contador y para un chat.

---

## Conexiones con el resto del temario

- **BFF** → lo construís con [NestJS](nestjs.md) o [FastAPI](fastapi.md) como gateway/aggregator.
- **Sync engines** se apoyan en [PostgreSQL](postgresql.md) y [NoSQL](nosql.md); los conflictos y eventos conectan con [Event-driven](event-driven.md).
- **Tiempo real** (sockets, reconexión) → [Tiempo real (WebSockets)](tiempo-real.md).
- **Crash-free rate como SLI/SLO**, RUM y OpenTelemetry → [Observabilidad práctica](observabilidad.md).
- **Tokens, pinning, attestation** → [Autenticación y autorización](autenticacion.md).
- **Push/colas de mutaciones y caché** → [Redis](redis.md) del lado servidor.
- **Testear la capa offline/sync** (red simulada, reloj controlado, time-travel de conflictos) → [Testing práctico](testing.md).

## Fuentes

- Jangid, S. & Kim, N. — *Mobile System Design* (newsletter System Design One, jun 2026).
- [React Native — New Architecture is here](https://reactnative.dev/blog/2024/10/23/the-new-architecture-is-here) (default desde 0.76; always-on desde 0.82).
- [Expo — New Architecture & EAS Update](https://docs.expo.dev/guides/new-architecture/).
- [Microsoft — Visual Studio App Center Retirement](https://learn.microsoft.com/en-us/appcenter/retirement) (31-mar-2025).
- [MongoDB — Atlas Device Sync End-of-Life](https://www.mongodb.com/community/forums/t/atlas-device-sync-end-of-life-and-deprecation/296687) (30-sep-2025).
- [PowerSync](https://powersync.com/) · [ElectricSQL](https://electric-sql.com/) · [WatermelonDB](https://watermelondb.dev/).
- OWASP **MASVS / MASTG** — estándar de seguridad móvil.
