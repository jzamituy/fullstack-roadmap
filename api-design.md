# Diseño de APIs y contratos: el método y el criterio

**Cómo diseñar el contrato, no solo implementarlo · REST, versionado, idempotencia, paginación, errores y evolución · para entrevistas senior y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y recién después contrastá con la sección final. Este módulo es sobre **diseño**, no sobre mecánica: no enseña a levantar un servidor (eso está en [Express](express.md)) ni a probar una API (eso está en [API Testing](api-testing.md)) ni el schema de GraphQL (eso está en [GraphQL](graphql.md)). Enseña a **diseñar el contrato**: la interfaz pública que tus consumidores van a usar durante años y que es **carísima de cambiar** una vez publicada. Es la diferencia entre "sé hacer un endpoint" y "sé diseñar una API que sobreviva a su propio éxito".

**Lo que asumimos.** Que ya sabés HTTP básico (métodos, status codes, headers), que escribiste alguna API REST ([Express](express.md), [NestJS](nestjs.md)) y que viste validación de input. Acá no re-explicamos cómo se rutea: nos preguntamos **qué contrato exponer y por qué**.

> **¿Te falta alguna base?** Si nunca armaste una API REST, hacé primero [Express](express.md) módulo 6; si "idempotencia" o "at-least-once" te suenan nuevos, el módulo 8 de [Diseño de sistemas](system-design.md) los ancla. Sin esas piezas, los módulos 4 (idempotencia) y 8 (evolución) te van a costar el doble.

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código). Este módulo es mayormente criterio (vas a ver muchos 🧠), porque diseñar una API **es** tomar decisiones y defenderlas.

**El marco que usamos en las fallas.** Para cada decisión de contrato que puede romper a un consumidor, aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a esta escala/forma de uso, (3) **control de corto plazo** que frena el daño, (4) **cambio de diseño** que baja la probabilidad de que se repita. Y una buena respuesta de diseño de API tiene **tres cosas**: una **forma para el happy path** (el contrato feliz), un **modelo de falla** (qué devolvés cuando algo sale mal) y un **camino de evolución** (cómo cambiás sin romper). Lo cerramos en el módulo 14.

**Índice de módulos**
1. El contrato primero: qué es diseñar una API
2. Estilos de API: REST vs gRPC vs GraphQL (el criterio de elección)
3. Recursos, métodos y status codes: el núcleo REST
4. Idempotencia: métodos seguros, idempotentes y las idempotency keys
5. Paginación a escala: offset vs cursor (keyset)
6. Filtrado, ordenamiento y selección de campos
7. Versionado de APIs: estrategias y cuándo (y cuándo no)
8. Evolución sin romper: breaking vs non-breaking y expand-contract
9. El contrato de error: problem+json (RFC 9457)
10. Rate limiting y cuotas en el contrato (429 y Retry-After)
11. Caching HTTP y concurrencia optimista: ETag, 304 y If-Match
12. Webhooks: el contrato saliente
13. OpenAPI como fuente de verdad: spec-first y contract testing
14. Caso completo: diseñar la API pública de la Task API

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El contrato primero: qué es diseñar una API

**Teoría.** Una API es un **contrato**: una promesa pública sobre cómo se la usa. El error de junior es pensar la API como "los endpoints que salen de mi código" (un reflejo de la base de datos hacia afuera). El senior la piensa al revés: **diseña el contrato que el consumidor necesita** y después lo implementa, no expone su esquema interno y reza.

La distinción clave: **el contrato es lo público y caro de cambiar; la implementación es lo privado y barato de cambiar.** Podés reescribir el servicio entero, cambiar de base, partir un monolito en microservicios — y mientras el contrato no cambie, **ningún consumidor se entera**. Al revés también: un cambio chico en el contrato (renombrar un campo) puede romper a cien clientes que no controlás. Por eso el contrato se diseña con cuidado y se cambia con disciplina (módulo 8).

Dos principios que guían el diseño:

- **Diseñá para el consumidor, no para tu base de datos.** El contrato modela el dominio que el cliente entiende (`order`, `invoice`, `subscription`), no tus tablas. Si tu API devuelve `user_role_join_id`, estás filtrando tu esquema interno hacia afuera y atándote las manos para refactorizar.
- **Ley de Postel (principio de robustez), con asterisco:** *"sé conservador en lo que enviás, liberal en lo que aceptás"*. Útil como instinto (tolerá campos extra que no conocés, no explotes), pero **con límite**: ser *demasiado* liberal aceptando input inválido esconde bugs y vuelve el contrato impreciso. El criterio moderno: liberal con lo que **ignorás** (campos desconocidos), estricto con lo que **validás** (lo que sí usás).

> **El cambio de mentalidad:** una API no es "mi código expuesto por HTTP". Es una **interfaz publicada** con consumidores que no controlás, que la van a usar de formas que no imaginaste, y que se va a tener que mantener compatible durante años. Diseñala como diseñarías una librería pública: la firma importa más que la implementación, y romperla tiene costo real.

**Ejercicios 1**
1.1 🔁 ¿Cuál es la diferencia entre el *contrato* de una API y su *implementación*, y por qué importa para refactorizar?
1.2 🧠 Tu endpoint `GET /usuarios` devuelve un campo `pwd_hash_v2`. ¿Qué dos problemas tiene exponer eso, y qué principio de diseño se violó?
1.3 🧠 "Sé liberal en lo que aceptás" llevado al extremo: un endpoint acepta `{ "edad": "treinta" }` y lo convierte a 30 en silencio. ¿Por qué esto es mala idea pese a sonar a la Ley de Postel?

---

## Módulo 2 — Estilos de API: REST vs gRPC vs GraphQL

**Teoría.** No hay un estilo "mejor": hay un estilo que **encaja con el contexto**. Los tres que vas a cruzar:

- **REST (sobre HTTP/JSON):** recursos identificados por URL, manipulados con métodos HTTP. **Universal** (cualquier cliente, browser, curl), cacheable por la infraestructura HTTP, simple de debuggear. Es el default para **APIs públicas** y de browser. Costo: over-fetching/under-fetching (traés de más o tenés que hacer N llamadas) y contratos menos tipados que gRPC.
- **gRPC (sobre HTTP/2 + Protobuf):** llamadas a procedimientos con contratos binarios fuertemente tipados (`.proto`). **Rápido y eficiente** (binario, multiplexing, streaming bidireccional), contratos estrictos con codegen. Ideal para **comunicación interna service-to-service** de alto volumen. Costo: no es nativo del browser (necesitás gRPC-Web + proxy), binario = más difícil de inspeccionar a mano.
- **GraphQL:** un solo endpoint donde el **cliente pide exactamente los campos que necesita**. Resuelve el over/under-fetching de REST y es excelente cuando **muchos clientes distintos** (web, mobile, terceros) necesitan **formas distintas** de los mismos datos. Costo: caching más difícil (no es GET cacheable), complejidad de servidor (el problema N+1, ver [GraphQL](graphql.md)), y autorización/rate-limiting por query es más sutil.

| | REST | gRPC | GraphQL |
|---|---|---|---|
| Transporte | HTTP/1.1+ / JSON | HTTP/2 / Protobuf | HTTP / JSON |
| Tipado del contrato | Opcional (OpenAPI) | Fuerte (`.proto`) | Fuerte (schema) |
| Caso ideal | API pública / browser | Interno service-to-service | Muchos clientes, formas variadas |
| Caching HTTP | Nativo (GET) | No | Difícil (POST) |
| Browser nativo | Sí | No (gRPC-Web) | Sí |
| Over/under-fetching | Sí | Acotable | No (cliente elige) |

> **La pregunta que decide:** *¿quién consume esta API y qué necesita?* API pública para terceros → **REST** (universalidad y cacheo ganan). Tráfico interno de alto volumen entre tus servicios → **gRPC** (eficiencia y contratos tipados). Un front rico (o varios) que arma pantallas con muchas entidades relacionadas → **GraphQL**. Y la respuesta senior incluye el matiz: **podés mezclar** — gRPC puertas adentro, un gateway REST/GraphQL puertas afuera.

**Ejercicios 2**
2.1 🔁 ¿Para qué caso de uso brilla cada uno: REST, gRPC, GraphQL?
2.2 🧠 Una app mobile se queja de que para armar el perfil tiene que hacer 6 llamadas REST y cada una trae campos que no usa. ¿Qué estilo lo resuelve y por qué? ¿Qué nuevo problema te traés?
2.3 🧠 Tenés 12 microservicios internos que se llaman entre sí miles de veces por segundo. ¿Por qué gRPC suele ganarle a REST/JSON acá, y qué resignás respecto de poder debuggear con curl?

---

## Módulo 3 — Recursos, métodos y status codes: el núcleo REST

**Teoría.** REST modela **recursos** (sustantivos), no acciones (verbos). La acción la lleva el **método HTTP**; el recurso, la **URL**.

**Diseño de URLs:**
- Sustantivos en plural, jerarquía por anidamiento: `/proyectos`, `/proyectos/42`, `/proyectos/42/tareas`, `/proyectos/42/tareas/7`.
- **Nada de verbos en la URL:** `POST /crearTarea` es RPC disfrazado. La acción ya la dice el método: `POST /tareas`.
- Las acciones que **no** son CRUD puro (no encajan en crear/leer/actualizar/borrar) son la excepción honesta: `POST /tareas/7/archivar` o modelarla como un sub-recurso de estado. No te tortures forzando todo a CRUD, pero que sea la excepción, no la regla.

**Métodos HTTP y su semántica:**

| Método | Qué hace | ¿Seguro? | ¿Idempotente? |
|---|---|---|---|
| `GET` | Leer | Sí | Sí |
| `HEAD` | Leer solo headers | Sí | Sí |
| `POST` | Crear / acción | No | **No** |
| `PUT` | Reemplazar completo | No | Sí |
| `PATCH` | Modificar parcial | No | No (depende) |
| `DELETE` | Borrar | No | Sí |

(*Seguro* = no modifica estado. *Idempotente* = ejecutarlo N veces deja el mismo estado que ejecutarlo una. Lo profundiza el módulo 4.)

**Status codes que tenés que usar bien** (el status **es** parte del contrato):

- **2xx — éxito:** `200 OK` (lectura/actualización con cuerpo), `201 Created` (creaste un recurso; devolvé `Location` con su URL), `202 Accepted` (lo aceptaste pero se procesa async, ver módulo 4), `204 No Content` (éxito sin cuerpo, típico en `DELETE`).
- **4xx — error del cliente:** `400 Bad Request` (malformado), `401 Unauthorized` (no autenticado — mal nombrado, en realidad es "no identificado"), `403 Forbidden` (autenticado pero sin permiso), `404 Not Found`, `409 Conflict` (choca con el estado actual: duplicado, edición concurrente), `422 Unprocessable Entity` (sintaxis OK pero viola reglas de negocio/validación), `429 Too Many Requests` (rate limit, módulo 10).
- **5xx — error del servidor:** `500 Internal Server Error` (algo explotó de tu lado), `502/503/504` (gateway, no disponible, timeout). Regla: un `5xx` es **culpa tuya**; nunca devuelvas `500` por un input malo del cliente (eso es `4xx`).

> **El error clásico — el `200` mentiroso.** Devolver `200 OK` con `{ "error": "no encontrado" }` adentro. Rompe todo: los clientes, los proxies y los reintentadores miran el **status** para decidir. Un `200` le dice a la infraestructura "salió bien, cacheá/seguí", cuando en realidad falló. El status code no es decorativo: es la primera línea del contrato que lee toda la cadena HTTP.

> **El debate 400 vs 422.** Cuándo cada uno: `400` para lo que ni siquiera se puede parsear (JSON roto, header faltante); `422` para lo que se parsea bien pero no pasa las reglas (`email` con formato inválido, `cantidad` negativa). No todos coinciden y está bien elegir una convención y **ser consistente** — la consistencia importa más que ganar la discusión semántica.

**Ejercicios 3**
3.1 🔁 ¿Qué status code devolvés al crear un recurso, y qué header acompaña esa respuesta?
3.2 🧠 Un dev propone `POST /usuarios/42/desactivar`. ¿Qué tiene de no-RESTful y cómo lo modelarías? ¿Cuándo aceptarías igual una URL así?
3.3 🧠 Para cada caso, elegí el status: (a) el cliente manda un `email` con formato inválido; (b) pide un recurso que no existe; (c) está logueado pero quiere editar un proyecto ajeno; (d) tu base de datos se cayó.

---

## Módulo 4 — Idempotencia: métodos seguros, idempotentes y las idempotency keys

**Teoría.** **Idempotente** = ejecutar la operación N veces deja el sistema en el mismo estado que ejecutarla una vez. Es una propiedad **central del contrato**, porque internet pierde respuestas: el cliente manda un `POST`, la red se corta antes de que llegue la respuesta, y el cliente **no sabe** si se ejecutó. ¿Reintenta? Si la operación no es idempotente, reintentar puede cobrar dos veces.

Por eso `GET`, `PUT`, `DELETE` se diseñan idempotentes (reintentarlos es seguro) y `POST` **no lo es** (crea algo nuevo cada vez). El problema: justo `POST` (crear un pago, una orden) es lo que más duele duplicar.

**La solución de contrato: idempotency keys.** El cliente genera una clave única (un UUID) por operación y la manda en un header:

```http
POST /pagos
Idempotency-Key: 9f8e7d6c-...
Content-Type: application/json

{ "monto": 5000, "moneda": "UYU" }
```

El servidor, la **primera** vez, procesa y **guarda el resultado asociado a esa key**. Si llega un reintento con la **misma** key, no reprocesa: devuelve el resultado guardado. Así el cliente puede reintentar sin miedo. (Es una convención de la industria — Stripe la popularizó — y hay un *draft* de IETF, `Idempotency-Key`, en camino a estandarizarla.)

```ts
// Middleware de idempotencia (la idea; compila en --strict).
interface IdemStore {
  // reserva atómica: true si la key es nueva (la insertó), false si ya existía
  reservar(key: string): Promise<boolean>;
  guardarRespuesta(key: string, status: number, body: unknown): Promise<void>;
  leerRespuesta(key: string): Promise<{ status: number; body: unknown } | null>;
}

async function conIdempotencia(
  key: string,
  store: IdemStore,
  ejecutar: () => Promise<{ status: number; body: unknown }>,
): Promise<{ status: number; body: unknown }> {
  const esNueva = await store.reservar(key);          // atómico: INSERT ... ON CONFLICT
  if (!esNueva) {
    const guardada = await store.leerRespuesta(key);
    if (guardada) return guardada;                    // reintento → respuesta cacheada
    // key reservada pero sin respuesta aún = request original todavía en vuelo
    return { status: 409, body: { error: "request en proceso, reintentá luego" } };
  }
  const res = await ejecutar();
  await store.guardarRespuesta(key, res.status, res.body);
  return res;
}
```

> **Ojo con el orden (la carrera sutil):** la reserva de la key tiene que ser **atómica y previa al efecto** — un `INSERT` con PK única que falle si ya existe, o un `SET NX`. Si chequeás "¿existe?" y después insertás en dos pasos, dos reintentos concurrentes pasan ambos el chequeo y cobrás dos veces. Es la misma race del check-then-act de [Concurrencia](concurrencia.md) módulo 5 y del consumidor idempotente de [Diseño de sistemas](system-design.md) módulo 8.

> **4 capas — el doble cobro por reintento:**
> 1. **Qué se rompe:** un cliente cobra dos veces porque reintentó un `POST /pagos` cuya respuesta se perdió.
> 2. **Por qué a esta escala/uso:** invisible en pruebas (la red anda); aparece bajo **timeouts y reintentos automáticos** (móviles con mala señal, un cliente HTTP con retry, un gateway que reintenta). La ventana es el tiempo entre el efecto y el ACK que nunca llegó.
> 3. **Control de corto plazo:** aceptar y exigir `Idempotency-Key` en los `POST` con efecto monetario; deduplicar por esa key.
> 4. **Cambio de diseño:** reserva atómica de la key antes del efecto + guardar la respuesta para devolverla en el reintento; documentar en el contrato que esos endpoints son idempotentes por key.

**Para operaciones largas — el patrón async (202).** Si crear el recurso tarda (un reporte, un video), no cuelgues el `POST`: respondé `202 Accepted` con la URL de un recurso de "trabajo" que el cliente pollea (`GET /trabajos/123` → `pending`/`done` + link al resultado). El contrato deja de mentir sobre lo que ya pasó.

**Ejercicios 4**
4.1 🔁 ¿Qué significa que `PUT` sea idempotente y `POST` no? Dá un ejemplo del problema que causa el `POST` no idempotente.
4.2 🧠 ¿Por qué la idempotency key tiene que reservarse de forma atómica **antes** de ejecutar el efecto, y no marcarse después?
4.3 ✍️ Escribí `manejarPago(key, store, cobrar)` que use un `IdemStore` para no cobrar dos veces ante reintentos con la misma key, devolviendo la respuesta cacheada si la key ya se procesó.

---

## Módulo 5 — Paginación a escala: offset vs cursor (keyset)

**Teoría.** Ninguna colección que crece se devuelve entera. La pregunta de diseño es **cómo paginás**, y hay dos familias con un trade-off fuerte.

**Offset/limit (paginación por página):**
```http
GET /tareas?limit=20&offset=40      # "saltá 40, dame 20" (página 3)
```
- **A favor:** intuitiva, permite saltar a una página arbitraria ("ir a la página 7"), trivial de implementar (`LIMIT 20 OFFSET 40`).
- **En contra (lo que la rompe a escala):** dos problemas serios.
  1. **Costo O(n):** `OFFSET 1000000` obliga a la base a **recorrer y descartar** un millón de filas para devolver 20. Las páginas profundas son lentísimas.
  2. **Drift por inserción:** si entre la página 1 y la 2 alguien inserta una fila al principio, todo se corre un lugar — ves un ítem repetido o te salteás uno. La paginación no es estable bajo escritura concurrente.

**Cursor / keyset (paginación por cursor):**
```http
GET /tareas?limit=20&cursor=eyJjcmVhdGVkQXQiOiIuLi4iLCJpZCI6N30
```
El cursor es un puntero opaco al **último ítem visto** (codificado, ej. base64 de `{createdAt, id}`). La query siguiente pide "lo que viene después de este punto":
```sql
SELECT * FROM tareas
WHERE (created_at, id) < ('2026-06-01T10:00', 12345)   -- "después" del cursor
ORDER BY created_at DESC, id DESC
LIMIT 20;
```
*(Ojo con el sentido de la comparación: el operador sigue al orden. Con `ORDER BY ... DESC` querés "lo más viejo que el cursor", así que va `<`; con `ORDER BY ... ASC` sería `>`. Si invertís el orden y te olvidás de invertir el operador, la página siguiente te devuelve para atrás.)*
- **A favor:** **O(1) por página** si hay índice sobre `(created_at, id)` — la base salta directo al punto, no recorre y descarta. Y es **estable**: las inserciones no corren las páginas, porque navegás por *valor de clave*, no por posición.
- **En contra:** no podés saltar a "la página 7" (solo siguiente/anterior), y necesitás un orden total y estable (por eso el desempate con `id`: dos filas con el mismo `created_at` necesitan un criterio único o el cursor es ambiguo).

> **El criterio senior:** **offset para datasets chicos y UIs con números de página** (un admin con 200 filas); **cursor/keyset para feeds, scroll infinito y APIs públicas a escala**. La marca de que alguien no diseñó para escala es una API pública con `?page=99999` que tumba la base. El cursor opaco además te deja **cambiar la estrategia por detrás** sin romper al cliente (él solo reenvía el token que le diste).

> **4 capas — la página profunda que mata la base:**
> 1. **Qué se rompe:** una query con `OFFSET` grande tarda segundos y satura la base; bajo concurrencia, agota el connection pool y degrada **toda** la API.
> 2. **Por qué a esta escala:** invisible con pocas filas; aparece cuando la tabla crece a millones y alguien (un scraper, un export, un bug de cliente) pagina hasta el fondo.
> 3. **Control de corto plazo:** **cap al offset/limit** (rechazá `offset` arriba de un máximo con `400`), límite de `limit` (ej. 100).
> 4. **Cambio de diseño:** migrar a cursor/keyset con índice sobre la clave de orden; exponer cursores opacos en el contrato para poder evolucionar la implementación.

**El sobre de la respuesta.** Devolvé los datos **y** los metadatos de navegación, de forma consistente:
```json
{
  "data": [ /* ...20 tareas... */ ],
  "page": { "nextCursor": "eyJ...", "hasMore": true }
}
```

**Ejercicios 5**
5.1 🔁 Nombrá los dos problemas de la paginación por offset que el cursor/keyset resuelve.
5.2 🧠 ¿Por qué el cursor necesita un orden **total** (ej. `(created_at, id)`) y no alcanza con ordenar por `created_at` solo?
5.3 ✍️ Escribí `codificarCursor`/`decodificarCursor` para un cursor opaco que lleva `{ createdAt: string; id: number }` (base64 de JSON), tipado en `--strict`.

---

## Módulo 6 — Filtrado, ordenamiento y selección de campos

**Teoría.** Más allá de paginar, una buena API de colección deja **filtrar**, **ordenar** y a veces **elegir campos**, todo por query params — sin convertirse en "SQL por HTTP".

**Filtrado.** Por query params, con convención clara:
```http
GET /tareas?estado=abierta&proyecto=42&creadaDespuesDe=2026-01-01
```
Para operadores (rangos, mayor/menor) hay dos escuelas: sufijos (`?monto_gte=100&monto_lte=500`) o sintaxis estructurada (`?filter[monto][gte]=100`). Elegí una y sé consistente. **Regla de seguridad:** filtrá solo por campos que **decidiste exponer** (una allowlist), nunca pasando el query param directo a la base — eso es inyección y fuga de campos internos.

**Ordenamiento.** Un parámetro `sort` con allowlist de campos y dirección:
```http
GET /tareas?sort=-creadaEn,titulo      # '-' = descendente; campos separados por coma
```

**Selección de campos (sparse fieldsets).** Dejar que el cliente pida solo lo que usa, para no over-fetchear:
```http
GET /tareas?fields=id,titulo,estado
```
Es la respuesta "lite" de REST al over-fetching (lo que GraphQL resuelve de raíz). Útil para clientes móviles; sumá complejidad solo si tenés el problema.

> **El insight de diseño:** estos parámetros son **opcionales y con defaults sanos** (un `GET /tareas` sin nada debe devolver algo razonable, no un error). Y son **acotados**: allowlist de campos filtrables/ordenables, límites de tamaño. Una API que deja filtrar y ordenar por *cualquier* columna es a la vez un agujero de performance (ordenás por una columna sin índice) y de seguridad (filtrás por una columna interna). Libertad para el cliente, dentro de un contrato que vos controlás.

**Ejercicios 6**
6.1 🔁 ¿Qué tres capacidades de una colección se exponen por query params, y por qué deben ser opcionales con defaults?
6.2 🧠 ¿Por qué filtrar/ordenar por *cualquier* campo que mande el cliente es un riesgo doble (performance y seguridad)? ¿Cómo lo acotás?
6.3 🧠 ¿Qué problema de REST resuelven los *sparse fieldsets* (`?fields=...`) y con qué estilo de API ese problema directamente no existe?

---

## Módulo 7 — Versionado de APIs: estrategias y cuándo

**Teoría.** Tarde o temprano necesitás cambiar el contrato de forma incompatible. El versionado te deja hacerlo **sin romper** a los consumidores viejos: conviven la `v1` y la `v2`. Tres estrategias:

- **En la URI:** `GET /v1/tareas` → `GET /v2/tareas`. **La más común y visible**, trivial de rutear y de probar en un browser. Crítica purista: la URL debería identificar el *recurso*, no su representación, y `/v1/tareas` y `/v2/tareas` son "la misma tarea". Pragmáticamente, gana por simplicidad.
- **En un header / media type (content negotiation):** `Accept: application/vnd.miapp.v2+json`. **Más "correcto" en teoría** (la URL queda limpia), pero invisible, más difícil de probar a mano y de cachear. Lo usan APIs muy formales (GitHub lo soportó así).
- **En un query param:** `GET /tareas?version=2`. Simple pero ensucia el cacheo y mezcla versionado con filtros. Poco recomendado.

**Una cuarta vía — versionado por fecha (date-based).** Lo usa Stripe: el cliente manda `Stripe-Version: 2026-06-01` en un header; el servidor mantiene transformaciones entre versiones. Excelente para APIs públicas con muchísimos consumidores, a costa de mantener la maquinaria de migración por dentro.

> **El criterio que más vale — cuándo NO versionar.** Versionar es caro: mantenés dos (o cinco) contratos vivos, con su código, sus tests y su confusión. La mayoría de los cambios **no necesitan** una versión nueva, porque son **compatibles hacia atrás** (módulo 8): agregar un campo opcional, un endpoint nuevo, un valor nuevo de enum tolerado. **Reservá la versión nueva para los cambios que sí rompen** (quitar un campo, renombrar, cambiar un tipo, cambiar la semántica). Saltar a `/v2` por cada cambio chico es un olor de que no se sabe distinguir breaking de non-breaking.

> **El error de versionado:** publicar `v1`, `v2`, `v3` en seis meses porque cada feature "merecía" su versión. Cada versión viva es deuda: código duplicado, superficie de bugs, y clientes confundidos sobre cuál usar. Una API bien diseñada vive años en `v1` porque **evoluciona compatiblemente** y solo corta una versión nueva cuando de verdad rompe algo.

**Ejercicios 7**
7.1 🔁 Nombrá tres estrategias de versionado y la crítica principal de la versión en URI.
7.2 🧠 ¿Por qué "versionar en cada cambio" es un antipatrón? ¿Qué clase de cambios *no* requieren versión nueva?
7.3 🧠 Una API pública con miles de integradores externos necesita cambiar contratos seguido sin romper a nadie. ¿Por qué el versionado por fecha (estilo Stripe) encaja mejor que `/v2`, `/v3`... y qué costo interno pagás?

---

## Módulo 8 — Evolución sin romper: breaking vs non-breaking y expand-contract

**Teoría.** El corazón de mantener una API viva: saber qué cambios **rompen** a un consumidor y cuáles no.

**Cambios NO breaking (compatibles, seguros sin versión nueva):**
- Agregar un **endpoint** nuevo.
- Agregar un **campo opcional** a una respuesta (un cliente que no lo conoce lo ignora — por eso los clientes deben tolerar campos extra, módulo 1).
- Agregar un **parámetro opcional** con default.
- Agregar un **valor** nuevo a un enum *si el cliente está preparado para tolerar valores desconocidos* (ojo: esto sí puede romper clientes con un `switch` exhaustivo).

**Cambios breaking (rompen — necesitan versión nueva):**
- **Quitar o renombrar** un campo o endpoint.
- **Cambiar el tipo** de un campo (`string` → `number`) o su formato.
- **Volver obligatorio** un parámetro que era opcional.
- **Cambiar la semántica** de un campo (antes `total` incluía impuestos, ahora no) — el más traicionero, porque el contrato "se ve igual" pero miente.
- Cambiar status codes o la estructura del error (módulo 9).

**El patrón para migrar sin romper: expand-contract (parallel change).** Tres fases:
1. **Expand:** agregá lo nuevo **sin quitar lo viejo**. Si renombrás `nombre` → `nombreCompleto`, devolvé **los dos** campos un tiempo. Aceptá los dos en la entrada.
2. **Migrate:** los consumidores migran a su ritmo al campo nuevo; vos medís el uso del viejo (telemetría: ¿quién sigue mandando/leyendo `nombre`?).
3. **Contract:** cuando el uso del viejo cae a cero (o se acaba el período de deprecación anunciado), lo quitás.

```jsonc
// Fase expand: ambos campos vivos. El cliente viejo lee `nombre`; el nuevo, `nombreCompleto`.
{
  "id": 42,
  "nombre": "Ada Lovelace",         // deprecado, se quita en v2 / fecha X
  "nombreCompleto": "Ada Lovelace"  // nuevo, preferido
}
```

> **Deprecación con avisos, no por sorpresa.** Antes de quitar algo, **anuncialo**: header `Deprecation` y `Sunset` (RFC 8594) con la fecha de corte, changelog, y si podés, telemetría para avisarles a los que todavía lo usan. Romper sin avisar es la forma más rápida de perder la confianza de tus integradores.

> **4 capas — el cambio de semántica silencioso:**
> 1. **Qué se rompe:** cambiás qué incluye `total` (antes con impuestos, ahora sin) sin cambiar el nombre ni el tipo; los clientes siguen leyendo `total` y ahora muestran/cobran mal.
> 2. **Por qué pasa:** el cambio "parece" no-breaking (mismo campo, mismo tipo), así que se cuela sin versión; los tests de schema pasan porque la *forma* no cambió, solo el *significado*.
> 3. **Control de corto plazo:** revertir y tratarlo como breaking; comunicar a los afectados.
> 4. **Cambio de diseño:** introducir un campo **nuevo** con la semántica nueva (`totalSinImpuestos`) vía expand-contract y deprecar el viejo; tratar el significado como parte del contrato, no solo la forma.

**Ejercicios 8**
8.1 🔁 Clasificá breaking / no-breaking: (a) agregar un campo opcional a la respuesta, (b) renombrar un campo, (c) volver obligatorio un parámetro que era opcional, (d) agregar un endpoint nuevo.
8.2 🧠 Explicá el patrón expand-contract para renombrar `email` → `correo` en una respuesta, sin romper clientes.
8.3 🧠 ¿Por qué un cambio de **semántica** (qué significa un campo) es más peligroso que un cambio de **forma** (tipo/nombre), y por qué los tests de schema no lo cazan?

---

## Módulo 9 — El contrato de error: problem+json (RFC 9457)

**Teoría.** El camino de error es **parte del contrato tanto como el happy path**. Una API que devuelve errores inconsistentes (a veces `{ "error": "x" }`, a veces `{ "message": "y" }`, a veces texto plano) obliga a cada cliente a programar contra el caos. El diseño senior define **una** forma de error y la respeta en toda la API.

**El estándar: Problem Details (RFC 9457, que reemplaza al RFC 7807).** Un cuerpo de error con `Content-Type: application/problem+json` y campos estándar:

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://miapp.com/errores/validacion",
  "title": "La solicitud no pasó la validación",
  "status": 422,
  "detail": "El campo 'email' no tiene un formato válido.",
  "instance": "/tareas",
  "errors": [
    { "campo": "email", "mensaje": "formato inválido" }
  ]
}
```

- **`type`** (URI): identifica la *clase* de error, estable y documentable. Es la clave que el cliente debe usar para discriminar programáticamente — no el `title`, que es legible para humanos y puede cambiar.
- **`title`:** resumen legible, estable por `type`.
- **`status`:** repite el HTTP status (conveniencia).
- **`detail`:** explicación específica de *esta* ocurrencia.
- **`instance`:** qué recurso/ocurrencia falló.
- **Campos de extensión** (`errors`, etc.): podés agregar los tuyos (ej. la lista de errores de validación por campo). *(Acá usamos nombres en español por la convención del proyecto, pero ojo: los campos estándar de RFC 9457 van en inglés, y en una API pública real los de extensión también suelen ir en inglés (`field`, `message`) por interoperabilidad. El nombre lo elegís vos; elegí inglés si esperás consumidores de afuera.)*

```ts
// El tipo del cuerpo de error, compartido por toda la API (compila en --strict).
interface ProblemDetails {
  type: string;          // URI estable que identifica la clase de error
  title: string;         // resumen legible
  status: number;        // HTTP status
  detail?: string;       // específico de esta ocurrencia
  instance?: string;     // recurso afectado
  errors?: Array<{ campo: string; mensaje: string }>;  // extensión propia
}

function problema(status: number, type: string, title: string, detail?: string): ProblemDetails {
  return { type, title, status, detail };
}
```

> **Reglas de oro del contrato de error:**
> - **Consistencia ante todo:** la *misma* forma en los 400, 403, 404, 422, 500. Un cliente parsea errores **una vez**.
> - **Discriminá por `type` (o un `code` estable), no por el texto.** El mensaje humano puede cambiar/traducirse; el `type`/`code` es el contrato máquina-a-máquina.
> - **No filtres internos:** nunca un stack trace, una query SQL o un path del servidor en un error de producción. Eso es una fuga de información (y un regalo para un atacante). Logueá el detalle internamente con un `traceId` y devolvé ese `traceId` al cliente para soporte.

> **El anti-patrón clásico:** devolver `200 OK` con `{ "success": false, "error": "..." }`. Ya lo vimos en el módulo 3: rompe a los proxies, a los reintentadores y a los clientes que miran el status. El error va con su status `4xx`/`5xx` **y** un cuerpo `problem+json` consistente. Las dos cosas.

**Ejercicios 9**
9.1 🔁 ¿Qué `Content-Type` usa Problem Details y cuál es el rol del campo `type`?
9.2 🧠 ¿Por qué el cliente debe discriminar errores por `type`/`code` y no por el texto del mensaje?
9.3 🧠 ¿Por qué nunca devolver un stack trace o una query SQL en el cuerpo de un error de producción, y qué devolvés en su lugar para poder dar soporte?

---

## Módulo 10 — Rate limiting y cuotas en el contrato (429 y Retry-After)

**Teoría.** Toda API pública necesita límites de uso: para protegerse del abuso, repartir capacidad con justicia y no caer por un cliente desbocado. El **algoritmo** (token bucket, etc.) está en [Diseño de sistemas](system-design.md) módulo 12; acá nos importa **cómo lo expone el contrato**.

**El status: `429 Too Many Requests`.** Cuando el cliente excede su límite, no le des `403` ni `500`: `429`, que comunica exactamente "frená, volvé a intentar después".

**El header clave: `Retry-After`.** Decile al cliente **cuándo** reintentar (segundos, o una fecha HTTP):
```http
HTTP/1.1 429 Too Many Requests
Retry-After: 30
```
Sin esto, los clientes reintentan a ciegas y empeoran la sobrecarga (el *retry storm* de [Concurrencia](concurrencia.md) módulo 19). Con esto, un cliente bien hecho espera lo indicado. *(`Retry-After` también se usa en `503 Service Unavailable`.)*

**Headers de cuota (para que el cliente se autorregule antes de chocar):**
```http
RateLimit-Limit: 100
RateLimit-Remaining: 12
RateLimit-Reset: 30
```
Hay un *draft* de IETF (`RateLimit` fields) que estandariza esto. Ojo a la versión: las revisiones recientes del draft unifican todo en **un solo header** con sintaxis de *structured fields* (`RateLimit: limit=100, remaining=12, reset=30`), mientras que la forma de campos separados (`RateLimit-Limit/Remaining/Reset`) es de un draft anterior y los `X-RateLimit-*` son el de-facto que más vas a ver en producción hoy. Lo importante del **contrato** no es qué header exacto, sino exponer límite, restante y cuándo se resetea, para que un cliente educado baje el ritmo **antes** de comerse un `429`.

> **4 capas — el `429` sin `Retry-After`:**
> 1. **Qué se rompe:** los clientes que reciben `429` reintentan de inmediato y en masa; el rate limiter rechaza más rápido pero el volumen de requests (aunque rechazadas) satura el borde.
> 2. **Por qué a esta escala:** con pocos clientes no se nota; con miles reintentando sincronizados, el propio mecanismo de defensa amplifica la carga (retry storm).
> 3. **Control de corto plazo:** agregar `Retry-After` a toda respuesta `429`/`503`; documentar que el cliente debe respetarlo.
> 4. **Cambio de diseño:** exponer headers de cuota (`RateLimit-*`) para autorregulación proactiva; recomendar en el contrato backoff exponencial **con jitter** del lado cliente.

> **El insight:** el rate limiting no es solo "rechazar"; es **comunicar**. Un buen contrato de límites le da al cliente todo para portarse bien: el `429`, el `Retry-After` y la cuota visible. Un mal contrato solo dice "no" y deja que el cliente adivine — y adivinar, bajo carga, significa retry storm.

**Ejercicios 10**
10.1 🔁 ¿Qué status code y qué header son el núcleo del contrato de rate limiting?
10.2 🧠 ¿Por qué un `429` sin `Retry-After` puede empeorar una sobrecarga? Conectá con el retry storm.
10.3 🧠 ¿Para qué sirven los headers `RateLimit-Limit/Remaining/Reset` si ya devolvés `429` cuando se pasa? (pista: proactivo vs reactivo)

---

## Módulo 11 — Caching HTTP y concurrencia optimista: ETag, 304 e If-Match

**Teoría.** El módulo 2 dijo que REST es "cacheable por la infraestructura HTTP". Eso no es magia: es **contrato**. Vos decidís, en los headers de respuesta, qué pueden cachear el browser, la CDN y los proxies — y de paso resolvés un problema de concurrencia que el módulo 3 dejó abierto (el `409`).

**Cache-Control: la política de cacheo.** El header con el que el servidor declara qué se puede guardar y por cuánto:
- `Cache-Control: max-age=3600` — cacheable 1 hora.
- `Cache-Control: no-store` — nunca cachear (datos sensibles).
- `Cache-Control: private` — solo el browser del usuario, no las CDNs/proxies compartidos (datos por-usuario); `public` permite cachés compartidos.

**ETag: el validador de versión.** Un `ETag` es una "huella" de la representación actual del recurso (un hash, o un número de versión). El servidor lo manda en la respuesta; el cliente lo guarda y, la próxima vez, pregunta *"¿cambió respecto de esta versión?"* con un **GET condicional**:

```http
GET /tareas/7
If-None-Match: "v3"

→ 304 Not Modified        # no cambió: el cliente reusa su copia, sin cuerpo en la respuesta
→ 200 OK + ETag: "v4"     # cambió: te mando la versión nueva
```

El `304 Not Modified` ahorra **ancho de banda y serialización**: el servidor no retransmite un cuerpo que el cliente ya tiene. Es el mismo trade-off `staleTime` vs revalidación que ya diste con React Query/SWR, ahora en el protocolo.

**Concurrencia optimista: `If-Match` → `412`.** El mismo ETag, del lado de la **escritura**, resuelve el *lost update* (dos clientes editan el mismo recurso y el segundo pisa al primero sin enterarse). El cliente manda la versión que leyó:

```http
PUT /tareas/7
If-Match: "v3"            # "actualizá solo si sigue en la versión v3"

→ 200 OK                  # seguía en v3: se aplicó, nueva ETag "v4"
→ 412 Precondition Failed # ya estaba en v4 (otro la editó): rechazado, releé y reintentá
```

Esto es **optimistic locking a nivel contrato** — el primo HTTP del lock optimista con columna de versión de [PostgreSQL](postgresql.md) y del `409 Conflict` del módulo 3. El `412` le dice al cliente "tu copia quedó vieja, no te dejo pisar"; el cliente releé, re-aplica su cambio sobre la versión fresca y reintenta.

> **4 capas — el *lost update* por edición concurrente:**
> 1. **Qué se rompe:** dos usuarios abren la misma tarea, ambos editan, el segundo en guardar **pisa** el cambio del primero sin que nadie se entere. Se pierde una escritura.
> 2. **Por qué a esta escala/uso:** invisible con un solo editor; aparece cuando varios usuarios (o varias pestañas) editan el mismo recurso, o con un `PATCH` lento y otro rápido encima.
> 3. **Control de corto plazo:** exigir `If-Match` con el ETag en los `PUT`/`PATCH` de recursos editables concurrentemente; rechazar con `412` si no coincide.
> 4. **Cambio de diseño:** versionar el recurso (columna `version`/`updated_at` que alimenta el ETag) y documentar en el contrato que esas escrituras son condicionales; el cliente sabe que debe leer-modificar-reintentar.

> **El insight:** caching y concurrencia optimista son **la misma pieza** (el ETag) mirada desde dos lados: en lectura ahorra transferencia (`304`), en escritura previene pisar cambios (`412`). Un contrato senior expone ETags en los recursos que se leen mucho y se editan concurrentemente — gratis te da las dos cosas.

**Ejercicios 11**
11.1 🔁 ¿Qué responde el servidor a un `GET` con `If-None-Match` cuando el recurso **no** cambió, y qué se ahorra con eso?
11.2 🧠 Explicá cómo `If-Match` + `412` previene el *lost update*, y con qué patrón de [PostgreSQL](postgresql.md) se corresponde.
11.3 🧠 ¿Cuándo pondrías `Cache-Control: private` en vez de `public`, y qué pasaría si te equivocás y marcás `public` un recurso por-usuario?

---

## Módulo 12 — Webhooks: el contrato saliente

**Teoría.** Hasta acá tu API **recibe** llamadas. Un **webhook** invierte la dirección: es **tu API la que llama al consumidor** cuando pasa algo (un pago se confirmó, un trabajo terminó). El consumidor registra una URL (`POST /webhooks` con `{ url, eventos }`) y vos le hacés `POST` a esa URL con cada evento. Es el contrato **saliente**, y tiene sus propias reglas — las mismas que ya viste, ahora del otro lado.

**Entrega: at-least-once, así que el receptor debe ser idempotente.** La red entre vos y el consumidor también pierde respuestas: mandás el evento, su servidor lo procesa pero el ACK se pierde, vos reintentás → lo recibe dos veces. Es el at-least-once de [Diseño de sistemas](system-design.md) módulo 8. Por eso cada evento lleva un **id único** y el contrato le pide al receptor que **deduplique por ese id** (la idempotency key del módulo 4, ahora del lado de quien recibe).

**Reintentos y DLQ.** Si la URL del consumidor está caída o responde error, reintentás con **backoff exponencial** (no de inmediato), y tras N fallos parás y marcás el evento como no entregable (una *dead-letter*, ver [event-driven](event-driven.md) y [Redis](redis.md)). El contrato documenta la política: cuántos reintentos, con qué espaciado, y cómo reenviar manualmente.

**Seguridad: firma HMAC.** El receptor necesita probar que el `POST` vino **de vos** y no de un atacante que conoce su URL. La forma estándar: firmás el cuerpo con un **secreto compartido** (HMAC) y mandás la firma en un header; el receptor recalcula la firma y compara. Sumá un **timestamp** firmado para evitar *replay* (reenviar un evento viejo capturado).

```ts
import { createHmac, timingSafeEqual } from "node:crypto";

// Lado RECEPTOR: verificar que el webhook es auténtico (compila en --strict).
function webhookEsValido(cuerpoRaw: string, firmaRecibida: string, secreto: string): boolean {
  const esperada = createHmac("sha256", secreto).update(cuerpoRaw).digest("hex");
  const a = Buffer.from(esperada, "hex");
  const b = Buffer.from(firmaRecibida, "hex");
  // longitudes distintas → timingSafeEqual tira; chequear antes evita la excepción y el leak de timing
  if (a.length !== b.length) return false;
  return timingSafeEqual(a, b);   // comparación en tiempo constante: no filtra info por timing
}
```

> **El receptor responde rápido y procesa async.** El handler del webhook debe devolver `2xx` **enseguida** (encolá el evento y procesalo después), no hacer el trabajo pesado en línea. Si tardás, el emisor te marca como lento/caído y reintenta — y terminás procesando duplicados por tu propia lentitud. "Aceptá rápido, procesá después": el `202` del módulo 4, ahora como receptor.

> **4 capas — el webhook procesado dos veces:**
> 1. **Qué se rompe:** el consumidor recibe el mismo evento `pago.confirmado` dos veces y le acredita el saldo al cliente dos veces.
> 2. **Por qué a esta escala/uso:** garantizado por el modelo (at-least-once): basta un ACK perdido o un receptor lento para que reintentes. No es un caso raro, es **el** caso.
> 3. **Control de corto plazo:** el receptor deduplica por el `id` del evento (lo registra como procesado antes de aplicar el efecto, atómico — la race del módulo 4).
> 4. **Cambio de diseño:** contrato explícito de at-least-once + id de evento estable + firma HMAC; del lado emisor, reintentos con backoff y DLQ con alerta (un evento no entregado es un problema que alguien debe ver).

**Evolución del payload.** El cuerpo de un webhook es un contrato como cualquier otro: agregar un campo es no-breaking, renombrar/quitar es breaking (módulo 9). Versioná el formato del evento igual que un endpoint y avisá las deprecaciones — tus consumidores tienen código que parsea ese JSON.

**Ejercicios 12**
12.1 🔁 ¿Por qué un webhook se entrega "al menos una vez" y qué le exige eso al receptor?
12.2 🧠 ¿Para qué sirve la firma HMAC de un webhook y por qué la comparación se hace en tiempo constante (`timingSafeEqual`)?
12.3 🧠 ¿Por qué el handler de un webhook debería responder `2xx` rápido y procesar el trabajo después, en vez de hacerlo en línea?

---

## Módulo 13 — OpenAPI como fuente de verdad: spec-first y contract testing

**Teoría.** Un contrato que vive solo en la cabeza (o en un wiki desactualizado) no es un contrato: es folklore. **OpenAPI** (antes Swagger) es la forma estándar de **escribir el contrato como un documento** (YAML/JSON) que describe endpoints, parámetros, schemas de request/response y errores.

**Dos enfoques:**
- **Code-first:** escribís el código con anotaciones/decoradores y generás el OpenAPI desde ahí (lo que hace `@nestjs/swagger`, o FastAPI automáticamente). Cómodo, el spec sigue al código — pero el contrato queda **subordinado** a la implementación (riesgo de filtrar internos).
- **Spec-first (design-first):** escribís el OpenAPI **primero**, lo acordás con los consumidores, y de ahí generás stubs de servidor, clientes tipados y mocks. El contrato es **el** artefacto, diseñado antes de implementar. Encaja con todo lo de este módulo: diseñás la interfaz, no la derivás.

**Qué te habilita tener el contrato como spec:**
- **Codegen:** clientes tipados y stubs de servidor en varios lenguajes, gratis.
- **Mock servers:** los consumidores desarrollan contra un mock del contrato antes de que exista el backend.
- **Documentación** siempre sincronizada (Swagger UI / Redoc).
- **Contract testing:** verificar que la implementación **cumple** el spec, y que un cambio no rompe a los consumidores. Esto enlaza directo con [API Testing](api-testing.md): validación de schema contra el OpenAPI, testing dirigido por spec (Schemathesis) y consumer-driven contracts (Pact) para el problema N×M entre servicios.

> **El criterio:** el valor de OpenAPI no es "generar lindos docs"; es tener **una sola fuente de verdad del contrato** que sirve a la vez de documentación, de generador de código y de **gate de CI** (el contract testing del módulo de API Testing rompe el build si la implementación se desvía del spec). Diseñar spec-first te fuerza a pensar el contrato antes de teclear el primer handler — que es, justamente, de lo que trató todo este módulo.

**Ejercicios 13**
13.1 🔁 ¿Qué describe un documento OpenAPI y cuál es la diferencia entre code-first y spec-first?
13.2 🧠 Nombrá tres cosas que te habilita tener el contrato como spec OpenAPI, más allá de la documentación.
13.3 🧠 ¿Cómo se conecta OpenAPI con el contract testing de [API Testing](api-testing.md) para evitar que un cambio rompa a un consumidor en producción?

---

## Módulo 14 — Caso completo: diseñar la API pública de la Task API

**Teoría.** Apliquemos todo a la "Task API" del hub (usuarios ↔ proyectos ↔ tareas), ahora como **API pública** que van a consumir terceros.

**1. Recursos y método (módulo 3).**
```
GET    /proyectos?limit=&cursor=&sort=         # listar (paginado, ordenable)
POST   /proyectos                              # crear → 201 + Location
GET    /proyectos/{id}                         # leer
PATCH  /proyectos/{id}                         # modificar parcial
DELETE /proyectos/{id}                         # borrar → 204
GET    /proyectos/{id}/tareas?estado=&cursor=  # tareas de un proyecto, filtrables
POST   /proyectos/{id}/tareas                  # crear tarea
POST   /tareas/{id}/completar                  # acción no-CRUD (excepción justificada)
```

**2. Paginación (módulo 5).** Cursor/keyset en todas las colecciones, porque es API pública y las colecciones crecen. Nada de `?page=N` sin tope.
```json
{ "data": [ /* ... */ ], "page": { "nextCursor": "eyJ...", "hasMore": true } }
```

**3. Idempotencia (módulo 4).** `POST /proyectos/{id}/tareas` acepta `Idempotency-Key` para que un cliente con mala conexión pueda reintentar sin crear tareas duplicadas. Documentado en el contrato.

**4. Errores (módulo 9).** Una sola forma `application/problem+json` en toda la API, discriminable por `type`. Validación → `422`; recurso ajeno → `403`; inexistente → `404`; conflicto de edición concurrente → `409`.

**5. Rate limiting (módulo 10).** `429` + `Retry-After` + headers `RateLimit-*` para que los integradores se autorregulen.

**6. Caching y concurrencia optimista (módulo 11).** `ETag` en `GET /tareas/{id}` (lectura barata vía `304`) y `If-Match` → `412` en `PATCH /tareas/{id}` para que dos editores concurrentes no se pisen.

**7. Webhooks (módulo 12).** Los integradores se suscriben a eventos (`tarea.completada`) y los reciben firmados con HMAC, con id de evento para deduplicar y reintentos con backoff de nuestro lado.

**8. CORS (si es API de browser).** Si un front en otro origen la consume, el contrato incluye CORS: el browser manda un *preflight* `OPTIONS` y nosotros respondemos qué orígenes, métodos y headers permitimos (`Access-Control-Allow-Origin`, etc.). Es contrato del lado del browser, no implementación: definirlo mal (un `*` permisivo en una API con credenciales) es un agujero de seguridad.

**9. Versionado y evolución (módulos 8-9).** Arranca en `/v1`. Los cambios compatibles (campos opcionales, endpoints nuevos) entran sin versión nueva; un breaking real dispara `/v2` con expand-contract y headers `Deprecation`/`Sunset` para la `v1`.

**10. El contrato como spec (módulo 13).** Todo lo anterior vive en un OpenAPI spec-first, que genera el cliente tipado, alimenta el mock para los integradores y es el gate de contract testing en CI.

**Retrospectiva del método.** Diseñamos **el contrato primero** (recursos, formas, errores, evolución) y dejamos la implementación para después. Ese es el orden senior: la interfaz pública es lo caro y duradero; el código que la cumple es reemplazable.

> **El cierre de criterio — no sobre-diseñar.** Todo esto asume una API pública con integradores externos y escala. Si tu API es interna, para un solo front que controlás, mucho de esto es over-engineering: offset alcanza, una versión informal alcanza, y un error simple y consistente alcanza. La marca de seniority no es aplicar las 12 cosas siempre — es **saber cuáles pide tu contexto**. "Diseñá el contrato que tus consumidores necesitan, ni más ni menos" — esa frase vale todo el módulo.

**Las 3 cosas (cierre del marco).** Una respuesta completa de diseño de API tiene: **(a)** una forma para el happy path (recursos, métodos, status, paginación), **(b)** un modelo de falla (errores `problem+json` consistentes, `429`/`Retry-After`, idempotencia ante reintentos) y **(c)** un camino de evolución (no-breaking por default, expand-contract y versionado solo cuando rompe). Si tu diseño cubre las tres, pensaste el contrato como un sistema vivo, no como una foto.

**Ejercicios 14**
14.1 🧠 Para `POST /proyectos/{id}/tareas`, listá las decisiones de contrato que tomarías (status de éxito, header de idempotencia, forma del error de validación, status si el proyecto no existe).
14.2 🧠 Te piden agregar un campo `prioridad` a las tareas. ¿Es breaking? ¿Cómo lo introducís sin tocar la versión?
14.3 🧠 ¿Qué tres cosas (happy path, modelo de falla, evolución) explicitarías al presentar el diseño de esta API en una entrevista, y por qué las tres juntas demuestran criterio?
14.4 ✍️ Escribí un handler `crearTarea(req)` que reúna las piezas del contrato: si falta el título, devolvé `422` con `problem+json`; si el proyecto no existe, `404`; si todo va bien, `201` con header `Location`. Usá los tipos de abajo.
```ts
interface Req { proyectoId: number; body: { titulo?: string }; }
interface Res { status: number; headers?: Record<string, string>; body: unknown; }
interface TareaRepo {
  proyectoExiste(id: number): Promise<boolean>;
  crear(proyectoId: number, titulo: string): Promise<{ id: number }>;
}
```

---

# Soluciones

> Mirá esto solo después de intentarlo. Casi todas son de **criterio**: si tu razonamiento difiere pero justifica bien el trade-off, probablemente también esté bien. En diseño de API, el *cómo* justificás importa más que el *qué* elegís.

### Módulo 1
**1.1** El **contrato** es la interfaz pública (URLs, formas de request/response, status, semántica): lo que los consumidores ven y de lo que dependen. La **implementación** es el código privado que lo cumple. Importa para refactorizar porque podés reescribir la implementación entera (cambiar de base, partir en microservicios) **sin romper a nadie** mientras el contrato no cambie; y al revés, un cambio chico de contrato puede romper a consumidores que no controlás. Lo caro y duradero es el contrato.

**1.2** (a) **Fuga de información sensible** (un hash de password nunca debería salir de la API — es un regalo para un atacante). (b) **Acoplamiento al esquema interno:** exponer `pwd_hash_v2` ata el contrato a tu implementación (¿qué pasa cuando hay un `v3`?). Se violó "diseñá para el consumidor, no para tu base de datos": el contrato debe modelar el dominio del cliente, no reflejar tus columnas.

**1.3** Porque convertir `"treinta"` → 30 en silencio **esconde un bug del cliente** y vuelve el contrato impreciso e impredecible: hoy adivinás bien, mañana `"treintaiuno"` falla raro, y nadie sabe qué acepta realmente la API. La Ley de Postel bien entendida es liberal con lo que **ignorás** (campos extra desconocidos), no con lo que **validás**: el input que sí usás se valida estricto y, si está mal, devolvés `422` con un error claro. Fallar fuerte y temprano es mejor contrato que adivinar.

### Módulo 2
**2.1** **REST:** APIs públicas y de browser (universalidad, cacheo HTTP). **gRPC:** comunicación interna service-to-service de alto volumen (binario eficiente, contratos tipados, HTTP/2). **GraphQL:** muchos clientes distintos que necesitan formas distintas de los mismos datos (el cliente elige los campos, sin over/under-fetching).

**2.2** Lo resuelve **GraphQL**: una sola query pide exactamente las entidades y campos que la pantalla necesita, en una sola ida. El nuevo problema que te traés: caching más difícil (no es un `GET` cacheable), complejidad de servidor (el N+1, que hay que resolver con dataloaders, ver [GraphQL](graphql.md)), y autorización/rate-limiting por query más sutil que por endpoint. *(Alternativa REST-only: sparse fieldsets + un endpoint agregador/BFF, módulo 6.)*

**2.3** gRPC gana por **eficiencia y contratos**: payloads binarios (Protobuf) mucho más chicos y rápidos de serializar que JSON, HTTP/2 con multiplexing y conexiones persistentes, y contratos `.proto` fuertemente tipados con codegen (menos errores de integración entre equipos). A ese volumen interno, el ahorro es enorme. Lo que resignás: no podés pegarle con `curl` ni leer el cuerpo a ojo (es binario), así que debuggear y explorar a mano es más incómodo — necesitás herramientas (grpcurl, reflection).

### Módulo 3
**3.1** `201 Created`, acompañado del header **`Location`** con la URL del recurso recién creado (ej. `Location: /tareas/7`). Opcionalmente el cuerpo trae la representación del recurso creado.

**3.2** `POST /usuarios/42/desactivar` mete un **verbo** en la URL (RPC disfrazado de REST); la URL debería nombrar un recurso, no una acción. Modelado RESTful: tratar el estado como un dato y hacer `PATCH /usuarios/42` con `{ "activo": false }`, o modelar la activación como un sub-recurso. **Cuándo lo aceptarías igual:** cuando la acción no encaja en CRUD puro (procesos, transiciones complejas con efectos colaterales) — ahí una URL de acción explícita (`POST /tareas/7/archivar`) es más honesta que forzar un PATCH. Debe ser la excepción, no la regla.

**3.3** (a) email inválido (se parsea pero viola la regla) → **`422`** (o `400` si tu convención no distingue; lo importante es ser consistente). (b) recurso inexistente → **`404`**. (c) logueado pero sin permiso sobre recurso ajeno → **`403`** (autenticado pero prohibido; `401` sería "no autenticado"). (d) base de datos caída → **`500`** (o `503` si es indisponibilidad temporal) — es culpa del servidor, jamás un `4xx`.

### Módulo 4
**4.1** `PUT /recurso/1` con un cuerpo **reemplaza** ese recurso: ejecutarlo 1 o 5 veces deja el mismo estado final, es idempotente. `POST /recursos` **crea uno nuevo** cada vez: ejecutarlo 5 veces crea 5 recursos. El problema: si un cliente manda `POST /pagos`, se le corta la red antes de recibir la respuesta y **reintenta**, cobra dos veces — porque no sabe si el primero se ejecutó.

**4.2** Porque entre "chequear si la key existe" y "marcarla como procesada" hay una **ventana de carrera**: dos reintentos concurrentes con la misma key pueden pasar ambos el chequeo (ninguno la ve aún), ejecutar el efecto los dos, y recién después marcarla → doble cobro. La reserva tiene que ser **atómica y previa** (un `INSERT` con PK única que falle en duplicado, o `SET NX`): el que reserva primero gana y ejecuta; el segundo ve que ya está reservada y no reprocesa. Marcar *después* del efecto no cierra la ventana.

**4.3**
```ts
interface IdemStore {
  reservar(key: string): Promise<boolean>;
  guardarRespuesta(key: string, status: number, body: unknown): Promise<void>;
  leerRespuesta(key: string): Promise<{ status: number; body: unknown } | null>;
}

async function manejarPago(
  key: string,
  store: IdemStore,
  cobrar: () => Promise<{ status: number; body: unknown }>,
): Promise<{ status: number; body: unknown }> {
  const esNueva = await store.reservar(key);   // atómico: INSERT ... ON CONFLICT DO NOTHING
  if (!esNueva) {
    const guardada = await store.leerRespuesta(key);
    if (guardada) return guardada;             // reintento de algo ya cobrado → respuesta cacheada
    return { status: 409, body: { type: "https://miapp.com/errores/idem-en-proceso", title: "Request en proceso", status: 409 } };
  }
  const res = await cobrar();                   // primera vez: cobra de verdad
  await store.guardarRespuesta(key, res.status, res.body);
  return res;
}
// Nota: la atomicidad la da `reservar` (INSERT con PK única). El 409 cubre el caso raro de
// dos requests con la misma key llegando casi a la vez: el segundo ve la reserva sin respuesta aún.
// Usamos un `type` propio (no `about:blank`) para ser coherentes con la doctrina del módulo 9:
// el cliente discrimina por `type` estable, no por el texto.
```

### Módulo 5
**5.1** (1) **Costo O(n) en páginas profundas:** `OFFSET` grande obliga a la base a recorrer y descartar todas las filas saltadas. (2) **Drift por inserción/borrado concurrente:** como navega por *posición*, una fila insertada/borrada al principio corre todo y hace que veas duplicados o te saltees ítems. El cursor/keyset navega por *valor de clave*, así que es O(1) con índice y estable ante escrituras.

**5.2** Porque si dos (o más) filas comparten el mismo `created_at`, ordenar solo por `created_at` no las desempata: el cursor "después de las 10:00:00" es **ambiguo** — no sabés cuál de las filas con ese timestamp ya devolviste, y podés repetir o saltear en el borde de la página. Agregar un desempate único (`id`) da un **orden total** (`(created_at, id)`): cada fila tiene una posición inequívoca y el cursor apunta sin ambigüedad.

**5.3**
```ts
interface Cursor {
  createdAt: string;
  id: number;
}

function codificarCursor(c: Cursor): string {
  return Buffer.from(JSON.stringify(c), "utf8").toString("base64url");
}

function decodificarCursor(token: string): Cursor {
  const json = Buffer.from(token, "base64url").toString("utf8");
  const parsed: unknown = JSON.parse(json);
  if (
    typeof parsed !== "object" || parsed === null ||
    typeof (parsed as Record<string, unknown>).createdAt !== "string" ||
    typeof (parsed as Record<string, unknown>).id !== "number"
  ) {
    throw new Error("cursor inválido");
  }
  const c = parsed as Record<string, unknown>;
  return { createdAt: c.createdAt as string, id: c.id as number };
}
// El cliente trata el token como opaco: solo reenvía lo que le diste. Eso te deja
// cambiar qué hay adentro del cursor sin romper el contrato.
```

### Módulo 6
**6.1** **Filtrado, ordenamiento y selección de campos** (sparse fieldsets), todo por query params. Deben ser **opcionales con defaults sanos** porque un `GET /tareas` "pelado" tiene que devolver algo razonable (las primeras N, en un orden por defecto), no un error: el caso simple debe ser simple, y los parámetros refinan, no son obligatorios.

**6.2** **Performance:** ordenar/filtrar por una columna sin índice obliga a full scans o sorts caros → un cliente puede tumbar la base pidiendo orden por un campo arbitrario. **Seguridad:** filtrar por cualquier campo expone columnas internas (o permite inferir datos) y, si pasás el param directo a la query, abrís inyección. Se acota con una **allowlist** de campos filtrables/ordenables (solo los que decidiste exponer y tenés indexados) y límites de tamaño; nunca construir la query con el nombre de campo crudo del cliente.

**6.3** Resuelven el **over-fetching**: el cliente pide solo los campos que usa (`?fields=id,titulo`) en vez de recibir el objeto completo, ahorrando ancho de banda (clave en mobile). Con **GraphQL** ese problema directamente no existe, porque elegir los campos es el modelo nativo de la query.

### Módulo 7
**7.1** (1) **URI** (`/v1/...`), (2) **header / media type** (`Accept: application/vnd.app.v2+json`), (3) **query param** (`?version=2`). Crítica a la URI: la URL debería identificar el *recurso*, y `/v1/tareas` y `/v2/tareas` son "la misma tarea" con distinta representación, así que mezcla recurso con versión de contrato. (Pragmáticamente gana igual por simplicidad y visibilidad.)

**7.2** Porque cada versión viva es **deuda permanente**: código duplicado, más superficie de bugs, tests por versión y clientes confundidos sobre cuál usar. Versionar tiene sentido solo para cambios **breaking**. Los que **no** requieren versión nueva: agregar un endpoint, agregar un campo opcional a la respuesta, agregar un parámetro opcional con default — todos compatibles hacia atrás (módulo 8). Una API sana vive años en `v1` evolucionando compatiblemente.

**7.3** Porque con miles de integradores no podés pedirles que migren de `/v1` a `/v2` cada vez que cambia algo: el versionado por fecha deja que **cada cliente fije la versión del día en que integró** (`Stripe-Version: 2026-06-01`) y el servidor aplica transformaciones para mantenerlo estable, mientras los nuevos arrancan con la última. Da evolución continua sin migraciones masivas ni un zoo de `/v2`, `/v3`. El costo interno: mantenés la **maquinaria de transformación entre versiones** (cada cambio breaking necesita su shim), que es trabajo y complejidad reales puertas adentro.

### Módulo 8
**8.1** (a) campo opcional nuevo en la respuesta → **no breaking** (el cliente que no lo conoce lo ignora). (b) renombrar un campo → **breaking** (el cliente que lo lee deja de encontrarlo). (c) volver obligatorio un parámetro opcional → **breaking** (los clientes que no lo mandaban empiezan a fallar). (d) endpoint nuevo → **no breaking** (nadie dependía de él).

**8.2** **Expand:** la respuesta devuelve **los dos** campos a la vez, `email` (deprecado) y `correo` (nuevo), con el mismo valor; si también recibís input, aceptás ambos. **Migrate:** los consumidores cambian a `correo` a su ritmo; medís con telemetría quién sigue leyendo/mandando `email`, y anunciás la deprecación (headers `Deprecation`/`Sunset`, changelog). **Contract:** cuando el uso de `email` cae a cero o vence el plazo anunciado, lo quitás (eso sí, en una versión nueva si todavía hubiera quien lo use). Nunca quitás y agregás en el mismo paso.

**8.3** Porque un cambio de **forma** (tipo/nombre) **rompe visiblemente** —el campo desaparece o no parsea, y los tests de schema y los clientes lo detectan al toque—. Un cambio de **semántica** deja la forma intacta (mismo nombre, mismo tipo) pero cambia el *significado* (`total` antes con impuestos, ahora sin): el cliente sigue leyendo el campo sin error y muestra/cobra mal, en silencio. Los tests de schema validan **estructura**, no significado, así que pasan en verde mientras el contrato real ya se rompió. Por eso un cambio de semántica se trata como breaking y se hace con un campo nuevo.

### Módulo 9
**9.1** `Content-Type: application/problem+json` (RFC 9457). El campo **`type`** es una URI estable que identifica la **clase** de error: es lo que el cliente usa para discriminar programáticamente qué pasó (a diferencia del `title`/`detail`, que son para humanos y pueden cambiar o traducirse).

**9.2** Porque el texto del mensaje es para **humanos**: puede reescribirse, traducirse o ajustarse sin previo aviso, y si el cliente hace `if (error.message === "...")` se rompe en cuanto cambie una coma. El `type` (o un `code` estable) es el contrato **máquina-a-máquina**: estable, documentado y pensado para que el cliente ramifique su lógica (¿reintento?, ¿pido login?, ¿muestro qué campo está mal?) sin depender de la redacción.

**9.3** Porque un stack trace o una query SQL **filtran información interna** (estructura del código, nombres de tablas, versiones, rutas del servidor) que ayuda a un atacante y expone tu implementación. En su lugar devolvés un error `problem+json` limpio y consistente con un **`traceId`** (o `instance`), logueás el detalle completo internamente asociado a ese id, y el usuario/soporte te pasa el `traceId` para que vos encuentres el problema en tus logs. El cliente recibe lo justo para actuar; el detalle sensible se queda adentro.

### Módulo 10
**10.1** El status **`429 Too Many Requests`** y el header **`Retry-After`** (segundos o fecha HTTP), que le dice al cliente cuándo puede reintentar.

**10.2** Porque sin `Retry-After` los clientes reintentan **a ciegas e inmediatamente**, normalmente todos a la vez; el servidor ya saturado recibe una avalancha de reintentos (aunque los rechace con `429`, el volumen de conexiones lo golpea) → el mecanismo de defensa se vuelve amplificador de carga. Es el **retry storm** de [Concurrencia](concurrencia.md) módulo 19: la recuperación mal hecha agrava la falla. Con `Retry-After` (y backoff + jitter del lado cliente) los reintentos se espacian y desincronizan.

**10.3** Porque `429` es **reactivo** (ya chocaste con el límite), mientras que `RateLimit-Limit/Remaining/Reset` son **proactivos**: le dan al cliente visibilidad de cuánta cuota le queda y cuándo se resetea, para que **baje el ritmo antes** de comerse el `429`. Un cliente bien hecho mira `Remaining` y se autorregula; así evitás el rechazo del todo, que es mejor para ambos lados que rechazar y reintentar.

### Módulo 11
**11.1** Responde **`304 Not Modified`** (sin cuerpo). Se ahorra **ancho de banda y serialización**: el servidor no retransmite una representación que el cliente ya tiene en caché; el cliente reusa su copia. (Igual hay un round-trip de red, pero sin payload.)

**11.2** El cliente lee el recurso y guarda su `ETag` (ej. `"v3"`); al escribir, manda `If-Match: "v3"` ("actualizá solo si sigue en v3"). Si otro lo editó mientras tanto, el ETag actual ya es `"v4"`, no coincide, y el servidor responde **`412 Precondition Failed`** sin aplicar el cambio → no se pisa la escritura del otro. El cliente releé, re-aplica su cambio sobre la versión fresca y reintenta. Se corresponde con el **optimistic locking** de [PostgreSQL](postgresql.md) (columna `version` que se chequea en el `UPDATE ... WHERE version = ?`): misma idea, expresada en el contrato HTTP.

**11.3** `private` cuando el recurso es **por-usuario** (el perfil propio, el saldo): solo el browser de ese usuario puede cachearlo, nunca una CDN/proxy compartido. Si por error lo marcás `public`, un caché compartido puede **servirle a un usuario los datos cacheados de otro** — una fuga de datos entre usuarios. Por eso `private` (o `no-store`) es el default seguro para todo lo que dependa de quién pregunta.

### Módulo 12
**12.1** Porque la red entre el emisor y el receptor también pierde ACKs: el emisor reintenta ante la duda, así que el mismo evento puede llegar más de una vez (at-least-once). Le exige al receptor ser **idempotente**: deduplicar por el `id` del evento (registrarlo como procesado, atómicamente, antes de aplicar el efecto), para que recibir un duplicado no dispare el efecto dos veces.

**12.2** Sirve para **autenticar** el webhook: prueba que el `POST` lo mandó quien dice (que comparte el secreto), no un atacante que descubrió la URL del receptor. La comparación se hace en **tiempo constante** (`timingSafeEqual`) para no filtrar información por *timing*: una comparación normal (`===`) corta en el primer byte distinto, y midiendo cuánto tarda en fallar un atacante puede ir adivinando la firma byte a byte. Tiempo constante = el tiempo de comparar no depende de cuántos bytes coinciden.

**12.3** Porque el emisor espera un `2xx` **rápido** como acuse de recibo; si el handler hace el trabajo pesado en línea y tarda, el emisor lo interpreta como lento o caído y **reintenta** — y el receptor termina procesando el mismo evento varias veces por su propia lentitud, además de arriesgar timeouts. Patrón correcto: validar la firma, encolar el evento, responder `2xx` enseguida, y procesar en un worker aparte (el "aceptá rápido, procesá después" del `202`).

### Módulo 13
**13.1** Un documento OpenAPI describe el contrato de la API: endpoints, métodos, parámetros, schemas de request/response, errores y auth, en YAML/JSON estándar. **Code-first:** el código (con anotaciones) es la fuente y el spec se genera desde él (cómodo, pero el contrato queda subordinado a la implementación). **Spec-first (design-first):** el spec se escribe **primero** y de él se generan stubs, clientes y mocks — el contrato es el artefacto que se diseña antes de implementar.

**13.2** Tres (de varias): (1) **codegen** de clientes tipados y stubs de servidor en varios lenguajes; (2) **mock servers** para que los consumidores desarrollen contra el contrato antes de que exista el backend; (3) **contract testing** que verifica en CI que la implementación cumple el spec (y que un cambio no rompe a los consumidores). *(También: documentación siempre sincronizada.)*

**13.3** El OpenAPI es la **fuente de verdad** del contrato; el contract testing ([API Testing](api-testing.md)) lo usa como referencia para verificar, en CI, que la API real **cumple** lo que el spec promete (validación de schema, testing dirigido por spec con Schemathesis) y que los consumidores siguen siendo compatibles (consumer-driven contracts con Pact). Si un cambio desvía la implementación del contrato acordado, **el build falla antes de deployar**, no en producción frente al consumidor.

### Módulo 14
**14.1** (a) Éxito: **`201 Created`** con `Location: /tareas/{nuevoId}` (y opcionalmente la tarea en el cuerpo). (b) Header de idempotencia: aceptar **`Idempotency-Key`** para que un reintento no cree una tarea duplicada. (c) Error de validación (ej. `titulo` vacío): **`422`** con cuerpo `application/problem+json` discriminable por `type`. (d) Si el proyecto `{id}` no existe: **`404`** (con el mismo formato de error). *(Y `403` si el proyecto es de otro usuario.)*

**14.2** **No es breaking:** agregar un campo opcional a la respuesta es compatible (los clientes que no lo conocen lo ignoran). Lo introducís **sin tocar la versión**: agregás `prioridad` a la respuesta con un default sensato (ej. `"media"`) y, si se puede setear, como parámetro **opcional** en `POST`/`PATCH`. Documentás el campo nuevo en el changelog y listo — `v1` sigue viva.

**14.3** **(a) Happy path:** recursos, métodos, status codes y paginación (cómo se ve cuando todo sale bien). **(b) Modelo de falla:** errores `problem+json` consistentes, `429`/`Retry-After`, e idempotencia ante reintentos (qué pasa cuando algo sale mal). **(c) Evolución:** cambios no-breaking por default, expand-contract y versionado solo cuando rompe (cómo cambia el contrato sin romper consumidores). Las tres juntas demuestran que pensás la API como un **sistema vivo** con consumidores reales a lo largo del tiempo, no como un set de endpoints para el demo de hoy — y eso es exactamente lo que evalúa una entrevista de diseño senior.

**14.4**
```ts
interface Req { proyectoId: number; body: { titulo?: string }; }
interface Res { status: number; headers?: Record<string, string>; body: unknown; }
interface TareaRepo {
  proyectoExiste(id: number): Promise<boolean>;
  crear(proyectoId: number, titulo: string): Promise<{ id: number }>;
}

async function crearTarea(req: Req, repo: TareaRepo): Promise<Res> {
  const titulo = req.body.titulo?.trim();
  if (!titulo) {
    return {
      status: 422,
      headers: { "Content-Type": "application/problem+json" },
      body: {
        type: "https://miapp.com/errores/validacion",
        title: "La solicitud no pasó la validación",
        status: 422,
        errors: [{ campo: "titulo", mensaje: "requerido" }],
      },
    };
  }
  if (!(await repo.proyectoExiste(req.proyectoId))) {
    return {
      status: 404,
      headers: { "Content-Type": "application/problem+json" },
      body: { type: "https://miapp.com/errores/no-encontrado", title: "Proyecto inexistente", status: 404 },
    };
  }
  const tarea = await repo.crear(req.proyectoId, titulo);
  return {
    status: 201,
    headers: { Location: `/tareas/${tarea.id}` },
    body: tarea,
  };
}
// Las tres piezas del contrato en un handler: validación → 422 problem+json,
// recurso padre inexistente → 404, éxito → 201 + Location. (La idempotency key
// se sumaría envolviendo esto con el `manejarPago`/`conIdempotencia` del módulo 4.)
```

---

## Siguientes pasos

Cuando estos 12 módulos te salgan fluidos, tenés el criterio para diseñar un contrato que sobreviva a su éxito: **happy path claro, fallas previstas y evolución sin romper.** Recomendado a continuación:

- **Implementá el contrato** que diseñaste: [Express](express.md) o [NestJS](nestjs.md) para REST, [GraphQL](graphql.md) si elegiste ese estilo.
- **Probá el contrato:** [API Testing](api-testing.md) — afirmá el contrato (no solo el `200`), contract testing con Pact y testing dirigido por OpenAPI.
- **Conectá con el sistema:** la idempotencia y el rate limiting viven en [Diseño de sistemas](system-design.md) (módulos 8 y 12) y [Concurrencia](concurrencia.md) (idempotencia bajo carrera, retry storm); la auth del contrato, en [Autenticación](autenticacion.md).
- **Diseñá la API de tu portfolio "para terceros":** tomá tu Task API y escribí su OpenAPI spec-first con todo lo de este módulo. Ese documento, hecho con criterio (sin sobre-diseñar), demuestra seniority de diseño.
