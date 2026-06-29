# Seguridad de sistemas a escala: revocación, secretos, borde y aislamiento

**Revocar un JWT a escala, gestión de secretos (KMS), identidad servicio-a-servicio (mTLS), defensa en el borde (WAF/DDoS) y aislamiento multi-tenant de datos · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es la **seguridad a nivel de arquitectura distribuida** — no la mecánica de un login (eso está en [Autenticación](autenticacion.md): hashing, JWT, refresh, RBAC) ni el contrato de auth de la API (eso está en [Diseño de APIs](api-design.md) M10). Acá respondemos las preguntas que aparecen cuando el sistema **crece**: *"emitís JWT stateless — ¿cómo cortás la sesión de un usuario comprometido si ningún servicio consulta la base?"*, *"¿dónde viven tus secretos y qué pasa cuando uno se filtra?"*, *"un tenant, ¿puede leer los datos de otro?"*. Es la diferencia entre "sé hacer login" y "sé diseñar un sistema que no se cae cuando alguien lo ataca".

**Lo que asumimos.** Que sabés autenticación vs autorización, JWT (firma, `exp`, el ataque `alg:none`), access + refresh tokens y RBAC ([Autenticación](autenticacion.md)); el contrato de auth (API key vs Bearer, scopes, `401`/`403`) de [Diseño de APIs](api-design.md) M10; el borde (reverse proxy, rate limiting, anycast) de [Reverse proxy](reverse-proxy.md) y [Networking](networking.md); y el multi-tenancy de [Datos a escala](datos-escala.md). No re-explicamos eso: lo **reusamos**.

> ⚠️ **Nota sobre datos volátiles.** Los servicios gestionados (AWS KMS/Secrets Manager, HashiCorp Vault, Cognito/Auth0, los WAF de Cloudflare/AWS) cambian de API, límites y precios seguido. Todo lo marcado con ⚠️ verificalo en la doc oficial. Los **patrones** (envelope encryption, introspection vs validación local, RLS, defensa en profundidad) no cambian; los detalles del proveedor sí.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Índice de módulos**
1. Revocar un JWT a escala: el triángulo stateless / revocable / latencia
2. Gestión de secretos: KMS, envelope encryption y rotación
3. Identidad servicio-a-servicio: mTLS y zero trust
4. Defensa en el borde: WAF, DDoS y rate limiting
5. Aislamiento multi-tenant de **datos** (≠ noisy neighbor)
6. Defensa en profundidad y mínimo privilegio
7. El criterio: cuánta seguridad, y gestionado vs propio

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Revocar un JWT a escala: el triángulo stateless / revocable / latencia

**Teoría.** Un JWT es **stateless**: cualquier servicio lo valida con la firma, **sin consultar la base** ([Autenticación](autenticacion.md) M3). Esa es su gran ventaja a escala (cada servicio valida solo, sin un cuello de botella central) y su gran problema: **un JWT válido no se puede revocar antes de que expire.** Si detectás que una cuenta fue comprometida, no hay forma de "cortar" un access token stateless ya emitido — sirve hasta su `exp`. Tenés tres palancas, y **no podés tener las tres a la vez** (el triángulo):

- **Validación local (stateless puro):** cada servicio verifica la firma y `exp`, nada más. **Rápido y desacoplado**, pero **no revocable**: el token vale hasta expirar.
- **Introspection (stateful):** en cada request, el servicio le pregunta al **servidor de auth** "¿este token sigue activo?" (RFC 7662 ⚠️). **Revocable al instante**, pero agrega **latencia** y **acopla** todos los servicios al de auth (que se vuelve un SPOF y un cuello de botella). Es lo opuesto al stateless.
- **Tokens de vida corta + refresh:** la solución práctica que ya viste ([Autenticación](autenticacion.md) M4). El access token dura **poco** (~15 min), así que la **ventana de revocación = su TTL**: para "cortar" a alguien, invalidás su **refresh token** (que sí es stateful, se consulta al renovar) y en ≤15 min el access muere solo. No es instantáneo, pero acota el daño sin pagar introspection en cada request.

El híbrido más usado a escala suma una **denylist** (blocklist): una lista chica de **token IDs (`jti`) revocados** en un store compartido rápido ([Redis](redis.md)) que cada servicio chequea. El `jti` entra a la denylist **cuando revocás** (logout, cuenta comprometida — el mismo evento que invalida el refresh stateful), con **TTL = `exp − now`** (lo que le queda de vida al access token): después de eso el token expira solo y ya no hace falta tenerlo en la lista, así que la denylist **se autolimpia** sin barridos. Como solo contiene los **pocos** tokens revocados-pero-no-expirados (no todos los tokens), el chequeo es barato. Recuperás revocabilidad **casi instantánea** pagando un lookup chico — un punto intermedio entre el stateless puro y el introspection.

> El triángulo, en una frase: **statelessness, revocabilidad instantánea y baja latencia — elegí dos.** Stateless puro = rápido + desacoplado pero no revocable. Introspection = revocable pero lento + acoplado. Vida corta + denylist = el balance práctico (revocación casi instantánea, lookup barato, ventana acotada).

> 🔥 **Falla en 4 capas — cuenta comprometida con JWT stateless.**
> (1) **Qué se rompe:** detectás un token robado, pero como validás local (stateless), el atacante **sigue entrando** hasta que el token expire.
> (2) **Por qué a esta escala:** con N servicios validando local sin estado compartido, no hay un punto donde "apagar" el token.
> (3) **Corto plazo:** invalidar el refresh token (corta la renovación) + agregar el `jti` a la **denylist** que los servicios chequean → el access muere en el próximo lookup.
> (4) **Largo plazo:** access tokens **de vida corta** (acota la ventana sin denylist), denylist en un store rápido para revocación inmediata, y rotación de refresh tokens con **detección de reuso** ([Autenticación](autenticacion.md) M10).

La frase mental: **un JWT stateless es rápido y desacoplado pero NO revocable antes de su `exp`. Triángulo: stateless / revocable-ya / baja-latencia — elegí dos. Introspection revoca al instante pero acopla y agrega latencia; vida corta + refresh acota la ventana al TTL; una denylist de `jti` revocados en Redis te da revocación casi instantánea con un lookup barato. El balance práctico es vida corta + denylist.**

**Ejercicios 1**
1.1 🔁 ¿Por qué un access token JWT stateless no se puede revocar antes de que expire? ¿Qué ventaja te da ese mismo statelessness?
1.2 🧠 Explicá el triángulo stateless / revocable / latencia. ¿Qué dos propiedades te da cada una de las tres estrategias (local, introspection, vida corta + denylist)?
1.3 ✍️ Implementá en TypeScript un `validarToken(token, store, nowEpoch)` que rechace el token si está **expirado** (`exp <= now`) o si su `jti` está en una **denylist** (`store.isRevoked(jti)`), y lo acepte si no. (La solución está al final; tiene que compilar en `--strict`.)

---

## Módulo 2 — Gestión de secretos: KMS, envelope encryption y rotación

**Teoría.** Tu sistema tiene **secretos**: el secreto de firma de JWT, la contraseña de la base, las API keys de terceros, las claves de cifrado. La primera regla, la que más se viola: **un secreto nunca va en el código ni en git ni en una imagen de Docker.** Si está en el repo, está comprometido (los bots escanean GitHub por keys filtradas en minutos). ¿Dónde van entonces?

- **Variables de entorno:** el mínimo aceptable para empezar, pero a escala se quedan cortas — quedan en logs, en el entorno de procesos, sin rotación ni auditoría.
- **Gestor de secretos:** AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager ⚠️. Un servicio dedicado que **guarda los secretos cifrados**, controla **quién** los lee (IAM/políticas), **audita** cada acceso, y soporta **rotación**. Tu app los pide en runtime con su identidad (no los lleva embebidos).

El mecanismo criptográfico clave es **envelope encryption** (cifrado en sobre), que usa un **KMS** (Key Management Service) — que, como todo lo criptográfico, **es un servicio gestionado** (AWS KMS, GCP KMS, Vault Transit ⚠️): vos **consumís su API**, no construís el módulo de hardware (módulo 7: *don't roll your own crypto*).

- Hay una **CMK / master key** que **nunca sale** del KMS (un módulo de hardware, HSM). No cifrás tus datos con ella directamente.
- Para cifrar datos, pedís al KMS una **data key**: el KMS te devuelve la data key **en claro** (para cifrar ahí mismo) **y cifrada** con la master key. Cifrás tus datos con la data key en claro, **tirás la data key en claro**, y guardás junto a los datos la **data key cifrada**.
- Para descifrar, le mandás al KMS la data key cifrada, el KMS la descifra con la master key (que nunca expone) y te devuelve la data key en claro para usarla.

¿Por qué este baile? Porque así la master key **nunca toca tus servidores ni tus discos** (vive en el HSM), podés cifrar **terabytes** sin pasarlos por el KMS (solo pasás las data keys, chicas), y **rotar** la master key no te obliga a re-cifrar todos los datos. Es el patrón detrás del cifrado at-rest de S3, RDS, etc.

> ⚠️ Matiz fino de la rotación: al rotar la master key, el KMS **conserva las versiones viejas** para poder **desenvolver** (unwrap) las data keys ya emitidas, y usa la nueva solo para **envolver** las nuevas. O sea: rotar la master **no re-protege retroactivamente** las data keys viejas (siguen envueltas con la versión anterior) salvo que hagas un **re-wrap** explícito. El "no re-cifrar todos los datos" es real y es la ventaja; pero no confundas rotar-la-master con re-proteger lo ya cifrado.

La **rotación** cierra el módulo: los secretos deben **cambiar periódicamente** (y de inmediato si se sospecha una filtración). La rotación automática (el gestor genera un secreto nuevo, actualiza el consumidor, revoca el viejo con un solapamiento) convierte "filtramos un secreto" de catástrofe a incidente acotado. Los **secretos dinámicos** de Vault van más lejos: credenciales **efímeras** generadas por uso (una credencial de base que vive 1 hora), de modo que un secreto filtrado expira casi solo.

> 🔥 **Falla en 4 capas — el secreto de firma de JWT se filtra.**
> (1) **Qué se rompe:** se filtra el secreto HS256; **cualquiera puede firmar tokens válidos** y hacerse pasar por cualquier usuario/rol.
> (2) **Por qué a esta escala:** un único secreto compartido firma **todo**; su blast radius es el sistema entero.
> (3) **Corto plazo:** **rotar** el secreto ya (invalida todos los tokens firmados con el viejo) y forzar re-login; revisar accesos.
> (4) **Largo plazo:** secretos en un gestor con rotación automática (no en env/código), **RS256** (clave privada que firma vive en un solo lugar, las demás solo verifican con la pública — el blast radius de filtrar la pública es nulo), y tokens de vida corta.

La frase mental: **un secreto nunca en código/git/imagen. A escala: gestor de secretos (Secrets Manager/Vault) con acceso por identidad, auditoría y rotación. Envelope encryption: la master key vive en el KMS/HSM y nunca sale; cifrás con data keys (devueltas en claro + cifradas), así rotás sin re-cifrar todo y la master nunca toca tus discos. Rotá periódicamente; secretos dinámicos/efímeros = un leak expira solo.**

**Ejercicios 2**
2.1 🔁 ¿Por qué un secreto nunca debe ir en el código? ¿Qué te da un gestor de secretos que una variable de entorno no?
2.2 🧠 Explicá envelope encryption: ¿por qué no cifrás los datos directamente con la master key, y qué ganás con las data keys?
2.3 🧠 Se filtró el secreto HS256 de firma de JWT. ¿Qué hacés de inmediato y qué cambio de diseño baja el blast radius a futuro?

---

## Módulo 3 — Identidad servicio-a-servicio: mTLS y zero trust

**Teoría.** Hasta acá la identidad era de **usuarios**. Pero en un sistema de microservicios, los **servicios** también se llaman entre sí — ¿cómo sabe el servicio de Pagos que quien lo llama es de verdad el de Pedidos y no un atacante que ya entró a la red? La respuesta vieja era "están en la red privada, confío en quien esté adentro" (**seguridad de perímetro**). La respuesta moderna es **zero trust**: **no confíes en la red; verificá cada request**, venga de donde venga. "Adentro de la VPC" no es una credencial.

El mecanismo principal para identidad servicio-a-servicio es **mTLS (mutual TLS)**. En el TLS normal ([Networking](networking.md) M2), el **cliente verifica al server** (por su certificado). En **mTLS**, **ambos** se verifican: el server también pide y valida el **certificado del cliente**. Así cada servicio tiene un **certificado que prueba su identidad**, y una llamada de servicio-a-servicio queda autenticada criptográficamente en las dos direcciones — sin secretos compartidos que se puedan filtrar. Es lo que hace una **service mesh** (Istio, Linkerd ⚠️) automáticamente: inyecta un *sidecar* que termina mTLS por vos, rota los certificados y aplica políticas de "quién puede llamar a quién".

mTLS le gana a la alternativa ingenua —una **API key compartida** entre servicios— porque la key es un secreto de larga vida que, si se filtra, sirve para siempre y para todos; los certificados de mTLS son **por-servicio, de vida corta y rotados** automáticamente. La identidad es criptográfica y revocable, no un string compartido.

> Conexión con el resto: zero trust combina **mTLS** (identidad del servicio) + **autorización por request** (qué puede hacer ese servicio, con scopes/políticas — [api-design](api-design.md) M10) + **mínimo privilegio** (módulo 6). El perímetro (firewall/VPC) sigue siendo una capa, pero **no la única**: defensa en profundidad (módulo 6).

La frase mental: **no confíes en la red ("estar adentro de la VPC" no es credencial): zero trust = verificá cada request. Para identidad servicio-a-servicio, mTLS (ambos lados presentan certificado, no solo el server) le gana a una API key compartida: certificados por-servicio, de vida corta y rotados, en vez de un secreto eterno que si se filtra sirve para todo. Una service mesh lo automatiza.**

**Ejercicios 3**
3.1 🔁 ¿Qué verifica el TLS normal y qué agrega mTLS? ¿Qué problema de identidad resuelve entre servicios?
3.2 🧠 ¿Qué significa "zero trust" y por qué "estar dentro de la red privada" dejó de considerarse suficiente?
3.3 🧠 ¿Por qué mTLS (certificados por servicio) es más seguro que compartir una API key entre tus microservicios?

---

## Módulo 4 — Defensa en el borde: WAF, DDoS y rate limiting

**Teoría.** El principio rector: **empujá la seguridad lo más lejos posible hacia afuera**, para que el tráfico malicioso **no llegue a tu origin**. El borde (CDN, reverse proxy, load balancer) ve todo el tráfico antes que tu app, así que es donde conviene filtrar. Tres capas:

- **WAF (Web Application Firewall):** inspecciona los requests HTTP y **bloquea patrones de ataque** conocidos — inyección SQL, XSS, path traversal, las reglas del OWASP Top 10, y reglas propias (ej. bloquear un user-agent o una ruta). Se para **delante** de tu app (en el CDN/ALB) y rechaza lo obvio antes de que toque tu código. No reemplaza validar input en la app (defensa en profundidad), pero corta el ruido.
- **Mitigación de DDoS:** un ataque volumétrico (millones de requests para tumbarte) se absorbe **en el borde**, no en tu origin. Acá conecta con [Networking](networking.md): **anycast** distribuye el tráfico entre muchos PoPs (un CDN como Cloudflare/CloudFront absorbe el volumen con su capacidad global), y el origin queda escondido detrás. La regla: tu servidor de app **nunca** debería recibir el grueso de un DDoS — para eso está la capa de borde con capacidad para diluirlo.
- **Rate limiting en el borde:** limitar requests por IP/cliente/API key **antes** de tu app ([Reverse proxy](reverse-proxy.md) M6, [api-design](api-design.md) M11). Frena fuerza bruta, scraping y abuso. A escala se combina con detección de bots y *challenges* (CAPTCHA/JS challenge) para el tráfico sospechoso.

El modelo mental: **capas concéntricas**. El borde filtra lo volumétrico y lo obviamente malicioso (DDoS, WAF, rate limit); tu app valida y autoriza lo que pasa (input validation, authz); la base aplica la última línea (RLS, módulo 5). Si confías **solo** en el borde, un atacante que lo esquive (o que ya esté adentro) tiene vía libre — por eso es **defensa en profundidad** (módulo 6), no "el WAF me cubre".

> 🔥 **Falla en 4 capas — un DDoS volumétrico contra el origin.**
> (1) **Qué se rompe:** un pico de tráfico malicioso satura tu servidor de app y tira el servicio para todos.
> (2) **Por qué a esta escala:** tu origin tiene capacidad finita; un atacante con un botnet la supera fácil si le pega directo.
> (3) **Corto plazo:** activar la mitigación del CDN/WAF, rate limiting agresivo, bloquear rangos/patrones, *challenge* al tráfico sospechoso.
> (4) **Largo plazo:** **esconder el origin** detrás del CDN/anycast (que nunca se exponga su IP directa), capacidad de borde que diluya el volumen, y autoscaling + *load shedding* para degradar antes que caer.

La frase mental: **empujá la seguridad hacia afuera: que lo malicioso no llegue al origin. Borde = WAF (bloquea SQLi/XSS/OWASP antes de tu app), mitigación de DDoS (anycast/CDN absorbe el volumen, el origin escondido) y rate limiting (frena fuerza bruta/scraping). Pero es UNA capa: el borde filtra, la app valida y autoriza, la base aplica RLS — defensa en profundidad, no "el WAF me cubre".**

**Ejercicios 4**
4.1 🔁 ¿Qué hace un WAF y dónde se ubica respecto de tu app? ¿Reemplaza validar input en la aplicación?
4.2 🧠 ¿Por qué un DDoS volumétrico se mitiga "en el borde" y no en tu servidor de app? Conectalo con anycast/CDN de [Networking](networking.md).
4.3 🧠 ¿Por qué confiar **solo** en el WAF es un error? ¿Qué otras capas tienen que estar?
4.4 ✍️ Implementá un `rateLimit(clientId, store, nowMs, opts)` de **ventana fija** en TypeScript: dado un `{ limit, windowMs }`, contá los requests del cliente en la ventana actual y devolvé si está **permitido** y cuántos le **quedan**. (La solución está al final; tiene que compilar en `--strict`. Pensá qué problema tiene la ventana fija en el borde entre dos ventanas.)

---

## Módulo 5 — Aislamiento multi-tenant de datos (≠ noisy neighbor)

**Teoría.** Ojo con esta distinción, que se mezcla y cae en entrevistas. [Datos a escala](datos-escala.md) trató el **noisy neighbor**: un tenant que consume tantos **recursos** que degrada a los demás — es un problema de **performance/disponibilidad** (se resuelve con quotas, shuffle sharding, celdas). Acá tratamos lo otro: **aislamiento de datos** — que el tenant A **no pueda leer ni escribir** los datos del tenant B. Es un problema de **seguridad/confidencialidad**, y su falla no es "el sistema va lento": es **una fuga de datos entre clientes**, de las peores que le pueden pasar a un SaaS.

Los tres modelos de aislamiento, de más barato/menos aislado a más caro/más aislado:

- **Shared schema (columna `tenant_id`):** todos los tenants en las mismas tablas, distinguidos por una columna `tenant_id`. **El más barato y denso**, pero **el más riesgoso**: cada query **debe** filtrar por `tenant_id`, y **un solo `WHERE` olvidado = fuga** (el tenant A ve filas del B). La seguridad depende de que **toda** consulta, en todo el código, para siempre, incluya el filtro. Frágil por diseño.
- **Schema-per-tenant:** un schema (namespace) por tenant en la misma base. Más aislado (los datos están separados lógicamente), más overhead de gestión (migraciones × N schemas).
- **Database-per-tenant:** una base por tenant. **El más aislado** (imposible cruzar datos por accidente; fácil de borrar/exportar por tenant; cumple aislamiento regulatorio) y **el más caro** (N bases que operar, no escala a millones de tenants chicos).

La defensa clave para el modelo *shared schema* (el más común) es **no depender de que el desarrollador no se olvide el `WHERE`**: aplicar el aislamiento en una capa donde sea **imposible** saltárselo. Dos formas:

- **Row-Level Security (RLS)** en la base (Postgres ⚠️): definís una política que filtra automáticamente **toda** query por el `tenant_id` de la sesión. La base **garantiza** que ninguna query devuelve filas de otro tenant, **aunque el `WHERE` falte en el código**. Es defensa en profundidad: la última línea no confía en la app.
  - ⚠️ **El gotcha que invalida todo si lo ignorás:** en Postgres, **el dueño de la tabla (y los superusers, y los roles con `BYPASSRLS`) saltean las políticas RLS por defecto**. Y es **muy común** que la app conecte con el **mismo rol que creó las tablas** (típico con ORMs y migraciones) → RLS **no hace nada**. Para que la garantía aplique de verdad, o conectás con un **rol no-dueño y sin `BYPASSRLS`**, o activás `ALTER TABLE ... FORCE ROW LEVEL SECURITY` (que fuerza la política **también** para el dueño). Sin esto, creés que tenés aislamiento y no lo tenés.
- **Una capa de acceso a datos obligatoria** (un repositorio que **inyecta** el `tenant_id` en cada query, sin forma de evitarlo) — más débil que RLS porque vive en la app, pero mejor que confiar en cada `WHERE` a mano.

> El blast radius manda el criterio: en *shared schema*, el blast radius de un bug es **todos los tenants**; en *database-per-tenant*, es **uno**. Cuanto más sensible el dato (salud, finanzas) y más estricto el aislamiento regulatorio, más te corrés hacia bases separadas — pagando el costo. Es el mismo razonamiento de blast radius de [Datos a escala](datos-escala.md), aplicado a confidencialidad en vez de a disponibilidad.

La frase mental: **aislamiento de DATOS (¿A puede leer datos de B? = seguridad) ≠ noisy neighbor (¿A degrada a B? = performance). Modelos: shared schema + `tenant_id` (barato, riesgoso — un `WHERE` olvidado es fuga), schema-per-tenant, database-per-tenant (caro, máximo aislamiento). Defensa del shared schema: NO confiar en el `WHERE` a mano → RLS en la base lo garantiza aunque el código se olvide. El blast radius de un bug decide cuánto aislás.**

**Ejercicios 5**
5.1 🔁 ¿En qué se diferencia el "aislamiento de datos multi-tenant" del "noisy neighbor"? ¿Qué tipo de problema es cada uno?
5.2 🧠 Listá los tres modelos de aislamiento (shared schema / schema-per-tenant / database-per-tenant) ordenados por aislamiento vs costo. ¿Cuál es el riesgo central del shared schema?
5.3 🧠 ¿Por qué Row-Level Security es más seguro que "acordarse de poner `WHERE tenant_id = ?` en cada query"? ¿Qué principio de diseño aplica?

---

## Módulo 6 — Defensa en profundidad y mínimo privilegio

**Teoría.** Dos principios transversales que ya asomaron en cada módulo y conviene nombrar explícitamente, porque son los que un entrevistador senior quiere oír:

- **Defensa en profundidad (defense in depth):** **múltiples capas** de seguridad, de modo que si una falla, otra detiene el ataque. Ninguna capa es "la" defensa. Lo viste repetido: el borde filtra (WAF/DDoS, módulo 4) **y** la app valida input **y** la base aplica RLS (módulo 5); mTLS autentica el servicio (módulo 3) **y** la autorización chequea permisos. Si confiás en una sola capa, el día que se la saltean estás desnudo. La pregunta senior: *"si esta capa falla, ¿qué te queda?"* — si la respuesta es "nada", te falta profundidad.

- **Mínimo privilegio (least privilege):** cada componente, credencial y persona tiene **solo los permisos que necesita, y nada más**. El servicio de Pedidos no necesita acceso de escritura a la tabla de usuarios; el token de una integración pide solo los scopes que usa ([api-design](api-design.md) M10); la credencial de la base de un servicio read-only es read-only. Así, cuando algo se compromete (y algo siempre se compromete), el **blast radius** está acotado a lo que ese componente podía hacer. Es el mismo hilo de blast radius de todo el hub: no se trata de evitar toda brecha (imposible), sino de que **una brecha no se vuelva total**.

Y dos básicos que sostienen todo lo anterior, para no darlos por obvios:

- **Cifrado in-transit y at-rest:** TLS para todo lo que viaja ([Networking](networking.md)), cifrado de los datos en disco (con envelope encryption/KMS, módulo 2). A escala ambos son default, no opción.
- **Auditoría y detección:** registrar **quién accedió a qué** (logs de auditoría inmutables — el primo del log append-only del [Caso 14](system-design-casos.md)) y **alertar** sobre lo anómalo. No podés responder a una brecha que no ves; la observabilidad ([Observabilidad](observabilidad.md)) es parte de la seguridad.

La frase mental: **defensa en profundidad: muchas capas, si una falla otra frena el ataque (borde + app + base + mTLS); preguntá "si esta capa cae, ¿qué me queda?". Mínimo privilegio: cada cosa con solo los permisos que usa, para que una brecha no se vuelva total (blast radius acotado). Y los básicos: cifrado in-transit/at-rest por default, y auditoría/alertas porque no respondés a lo que no ves.**

**Ejercicios 6**
6.1 🔁 Definí "defensa en profundidad" y "mínimo privilegio" en una frase cada uno. ¿Qué pregunta revela si te falta profundidad?
6.2 🧠 Tu servicio de reportes solo lee datos. ¿Por qué le das una credencial de base **read-only** y no la misma que usa el servicio que escribe? Conectalo con blast radius.
6.3 🧠 ¿Por qué la auditoría (logs de quién accedió a qué) es parte de la seguridad y no solo de la observabilidad?

---

## Módulo 7 — El criterio: cuánta seguridad, y gestionado vs propio

**Teoría.** El cierre, como en cada módulo del hub: la seguridad es un **espectro de costo/beneficio**, no un absoluto. Demasiada poca te deja expuesto; demasiada (para lo que el sistema amerita) frena al equipo sin ganancia real. El criterio:

**Calibrá la seguridad al valor de lo que protegés y al modelo de amenaza.** Un MVP interno no necesita mTLS + service mesh + database-per-tenant; un sistema de salud o finanzas sí. La pregunta no es "¿es seguro?" (nada lo es del todo) sino "**¿qué estoy protegiendo, de quién, y cuál es el costo de una brecha?**". Eso decide cuánto invertís en cada capa.

**Casi siempre, no lo construyas vos:**
- **Auth gestionada** (Cognito, Auth0, Clerk ⚠️): emisión/rotación de tokens, MFA, social login, ya resueltos y auditados. Implementar OAuth2/OIDC a mano es una fuente clásica de agujeros.
- **Secretos/KMS gestionados** (Secrets Manager, Vault, KMS): no inventes tu propio cifrado ni tu propio store de secretos. *"Don't roll your own crypto"* es una regla por algo.
- **WAF/DDoS gestionados** (Cloudflare, AWS WAF/Shield): capacidad y reglas que no podés igualar solo.

Lo que **sí** es siempre tuyo y nadie hace por vos: **la autorización de tu dominio** (quién puede hacer qué en *tu* lógica de negocio), **el aislamiento de tenants** (el modelo y el RLS), **el mínimo privilegio** en tus componentes, y **validar tu propio input**. Lo gestionado te da los ladrillos; el diseño seguro del sistema es tuyo.

El costo que se subestima: la seguridad mal calibrada **hacia arriba** también daña — fricción que el equipo termina esquivando (el clásico "guardamos el secreto en un Slack porque el Vault era un dolor"). La seguridad que el equipo evade no es seguridad. Apuntá al nivel que el sistema **necesita** y que el equipo **puede sostener**.

> **El cierre de criterio — no sobre-diseñar ni sub-diseñar.** ¿Un side-project? Auth gestionada, secretos en el gestor de la plataforma, HTTPS, y listo. ¿Un SaaS B2B con datos sensibles de varios clientes? Ahí sí: aislamiento de tenants pensado (RLS o DB-per-tenant según el dato), rotación de secretos, mTLS si hay microservicios, WAF, auditoría. La pregunta senior: *"¿cuál es el costo de una brecha acá, y estoy invirtiendo en proporción?"*.

La frase mental de cierre: **la seguridad se calibra al valor de lo que protegés y al modelo de amenaza, no es un absoluto. No construyas auth/crypto/WAF propios — usá gestionado (auditado, "don't roll your own crypto"). Lo tuyo siempre: la autorización de tu dominio, el aislamiento de tenants, el mínimo privilegio y validar tu input. Y ojo: la seguridad que el equipo evade (por fricción) no es seguridad.**

**Ejercicios 7**
7.1 🔁 ¿Qué partes de la seguridad conviene delegar en servicios gestionados y cuáles son siempre tuyas?
7.2 🧠 ¿Por qué "más seguridad" no es siempre mejor? Dá un ejemplo de seguridad mal calibrada hacia arriba.
7.3 🧠 ¿Qué pregunta resume cómo decidir cuánta seguridad invertir en un sistema dado?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Porque cada servicio lo valida **solo con la firma y `exp`, sin consultar un estado central** — no hay un lugar donde "marcarlo como inválido" que los servicios consulten. Mientras la firma sea válida y no haya expirado, **lo aceptan**. La **ventaja** de ese mismo statelessness: validación **rápida y desacoplada** (cada servicio valida sin pegarle a la base ni al servidor de auth → no hay cuello de botella central ni SPOF), que es justo lo que hace al JWT escalar.

**1.2** El triángulo: **statelessness, revocabilidad instantánea y baja latencia — no podés tener las tres.** **Validación local** = baja latencia + desacople (stateless), pero **no revocable**. **Introspection** (preguntar al servidor de auth en cada request) = revocable al instante + (sigue siendo "verificable"), pero **alta latencia + acoplamiento** (pierde el statelessness). **Vida corta + denylist** = revocación casi instantánea (denylist) + baja latencia (lookup chico), resignando el statelessness puro (hay un store compartido), pero acotado: la denylist solo tiene los pocos tokens revocados-no-expirados. Es el balance práctico.

**1.3**
```ts
interface RevocationStore {
  isRevoked(jti: string): Promise<boolean>;
}

interface AccessToken {
  jti: string; // id único del token (para la denylist)
  sub: string; // usuario
  exp: number; // expiración en epoch seconds
}

interface ResultadoValidacion {
  valido: boolean;
  razon?: "expirado" | "revocado";
}

async function validarToken(
  token: AccessToken,
  store: RevocationStore,
  nowEpoch: number,
): Promise<ResultadoValidacion> {
  // Chequeo barato primero: si ya expiró, ni tocamos la denylist.
  if (token.exp <= nowEpoch) {
    return { valido: false, razon: "expirado" };
  }
  // Solo los tokens vivos pueden estar en la denylist (las entradas se autolimpian al expirar).
  if (await store.isRevoked(token.jti)) {
    return { valido: false, razon: "revocado" };
  }
  return { valido: true };
}
```
Idea: la firma del JWT ya se verificó antes (esto asume un token con firma válida — [Autenticación](autenticacion.md) M3); acá agregamos las **dos** comprobaciones que el stateless puro no hace. Chequeamos `exp` **primero** (es gratis, en memoria) y solo si el token sigue vivo consultamos la **denylist** (un lookup en [Redis](redis.md)). La denylist solo necesita guardar los `jti` revocados **hasta su `exp`** (después expiran solos y el token ya no vale igual), por eso es chica y barata. Compila en `--strict`. En producción, `isRevoked` es un `EXISTS` en Redis con TTL = el `exp` restante del token.

**2.1** Porque el código vive en **git**, en imágenes, en las máquinas de todo el equipo y en CI — un secreto ahí está **expuesto a todos esos lugares** y los bots escanean repos públicos por keys en minutos. Un **gestor de secretos** te da, además de guardarlos cifrados: **control de acceso por identidad** (quién/qué puede leer cada secreto), **auditoría** (registro de cada acceso), **rotación** (cambiarlos sin tocar el código) y que la app los pida **en runtime** (no embebidos). Una variable de entorno no rota, no audita, y suele filtrarse en logs/dumps.

**2.2** Porque la **master key nunca debe salir** del KMS/HSM (si saliera para cifrar tus datos, estaría en tu servidor y en riesgo). En su lugar, el KMS te da una **data key** (en claro para cifrar ahí mismo, y cifrada para guardar); cifrás con la data key en claro, la descartás, y guardás la versión cifrada junto a los datos. Ganás: (1) la master **nunca toca tus discos** (vive en el HSM); (2) cifrás **volúmenes enormes** sin pasarlos por el KMS (solo viajan las data keys, chicas); (3) **rotar** la master key **no obliga a re-cifrar todos los datos** — solo se re-cifran las data keys. Por eso es el patrón del cifrado at-rest de S3/RDS.

**2.3** **De inmediato:** **rotar** el secreto de firma (al cambiarlo, **todos** los tokens firmados con el viejo dejan de validar → se cortan las sesiones del atacante), forzar re-login, y auditar accesos para ver el alcance. **Cambio de diseño:** sacar el secreto del env/código y ponerlo en un **gestor con rotación automática**; pasar a **RS256** (la clave **privada** que firma vive en un solo lugar protegido y los servicios solo tienen la **pública** para verificar — filtrar la pública no permite firmar, así que el blast radius de un servicio comprometido baja a cero); y tokens de **vida corta** para acotar la ventana.

**3.1** El **TLS normal** verifica la identidad **del server** (el cliente valida el certificado del server; el server no sabe quién es el cliente). **mTLS** agrega que **el server también verifica al cliente**: ambos presentan y validan certificado. Resuelve la **identidad servicio-a-servicio**: el servicio llamado puede probar criptográficamente **quién lo está llamando** (que es de verdad el servicio X y no un atacante en la red), sin depender de secretos compartidos.

**3.2** **Zero trust** = **no confiar en la red**: cada request se autentica y autoriza **venga de donde venga**, incluso "desde adentro" de la VPC. "Estar en la red privada" dejó de ser suficiente porque el modelo de **perímetro** (muralla afuera, confianza total adentro) colapsa apenas un atacante **entra** (phishing, una dependencia comprometida, un servicio con un bug): una vez adentro, tiene vía libre a todo. Zero trust asume que **el atacante ya puede estar adentro** y exige verificar cada salto igual.

**3.3** Porque una **API key compartida** es **un secreto de larga vida, igual para todos los servicios**: si se filtra (de un log, un repo, un dump), sirve **para siempre y para suplantar a cualquiera**, y rotarla obliga a actualizar a todos a la vez. Los **certificados de mTLS** son **por servicio, de vida corta y rotados automáticamente** (la service mesh lo hace): la identidad es criptográfica y específica, un certificado comprometido afecta a **un** servicio y expira solo, y no hay un único secreto cuyo leak comprometa todo. Mínimo privilegio + blast radius acotado.

**4.1** Un **WAF** inspecciona los requests HTTP y **bloquea patrones de ataque conocidos** (inyección SQL, XSS, path traversal, OWASP Top 10) **antes** de que lleguen a tu app; se ubica **delante** de la aplicación (en el CDN/ALB/reverse proxy). **No reemplaza** validar input en la app: el WAF corta lo obvio y el ruido, pero puede ser evadido y no conoce tu lógica de negocio — validar y sanitizar el input en la aplicación sigue siendo obligatorio (**defensa en profundidad**).

**4.2** Porque tu **origin tiene capacidad finita** y un ataque volumétrico (un botnet) la supera fácil si le pega directo → se satura y cae. El **borde** (CDN/anycast — [Networking](networking.md)) tiene **capacidad global distribuida**: con **anycast**, la misma IP se anuncia desde muchos PoPs y el tráfico (incluido el malicioso) se **reparte y diluye** entre todos ellos, absorbiéndose lejos de tu origin. Además el origin queda **escondido** detrás del CDN (su IP no se expone), así que el atacante no tiene a quién pegarle directo. El borde está dimensionado para esto; tu app no.

**4.3** Porque el WAF es **una sola capa** y puede ser **evadido** (un ataque nuevo que no matchea sus reglas, o un atacante que ya está adentro de la red y no pasa por el borde). Si es tu única defensa, el día que lo esquivan tenés **cero** protección. Tienen que estar también: **validación/sanitización de input en la app**, **autorización** por request, **RLS** en la base (módulo 5), mTLS entre servicios (módulo 3). Defensa en profundidad: el WAF filtra, pero cada capa de adentro asume que las de afuera **pueden haber fallado**.

**4.4**
```ts
interface CounterStore {
  // Incrementa el contador de esa ventana y devuelve el nuevo valor.
  // (En Redis: INCR + EXPIRE con TTL = windowMs, atómico.)
  incr(key: string, ttlMs: number): Promise<number>;
}

interface RateLimitResult {
  permitido: boolean;
  restantes: number;
}

async function rateLimit(
  clientId: string,
  store: CounterStore,
  nowMs: number,
  opts: { limit: number; windowMs: number },
): Promise<RateLimitResult> {
  // La clave incluye el NÚMERO de ventana actual: al cambiar de ventana, cambia la clave
  // y el contador arranca de cero (la entrada vieja expira sola por su TTL).
  const ventana = Math.floor(nowMs / opts.windowMs);
  const key = `ratelimit:${clientId}:${ventana}`;
  const usados = await store.incr(key, opts.windowMs);
  return {
    permitido: usados <= opts.limit,
    restantes: Math.max(0, opts.limit - usados),
  };
}
```
Idea: la **ventana fija** trocea el tiempo en bloques de `windowMs` y cuenta los requests de cada bloque, usando el número de ventana como parte de la clave (así no hay que limpiar nada: la clave vieja expira). El `incr` es atómico (en [Redis](redis.md), `INCR` + `EXPIRE`), clave para que dos requests concurrentes no pisen el contador. Compila en `--strict`. **El problema de la ventana fija** (lo que pedía pensar): en el **borde** entre dos ventanas, un cliente puede mandar `limit` requests al final de una y `limit` al principio de la siguiente → **2× el límite** en un instante. Lo resuelven el *sliding window* (ventana deslizante) o el *token bucket* — el "balde con goteo" que ya viste en el rate limiting de [Reverse proxy](reverse-proxy.md) M6 y [api-design](api-design.md) M11. Para el borde, esta versión simple suele alcanzar; cuando el burst del borde importa, subís a sliding window.

**5.1** El **aislamiento de datos multi-tenant** es si el tenant A puede **leer/escribir datos** del tenant B → problema de **seguridad/confidencialidad** (su falla es una **fuga de datos** entre clientes). El **noisy neighbor** ([Datos a escala](datos-escala.md)) es si un tenant consume tantos **recursos** que degrada a los demás → problema de **performance/disponibilidad** (su falla es "el sistema va lento"). Uno es confidencialidad, el otro es rendimiento; se resuelven con cosas distintas (RLS/aislamiento vs quotas/shuffle sharding).

**5.2** De **más barato/menos aislado** a **más caro/más aislado** (el mismo orden que la teoría): **shared schema** (todos en las mismas tablas con columna `tenant_id` — el más barato y denso) → **schema-per-tenant** (un schema por tenant en la misma base — aislamiento lógico, overhead de migraciones × N) → **database-per-tenant** (una base por tenant — máximo aislamiento, imposible cruzar datos por accidente, pero N bases que operar). El **riesgo central del shared schema**: cada query **debe** filtrar por `tenant_id`, y **un solo `WHERE` olvidado en cualquier parte del código filtra datos de un tenant a otro**. La seguridad depende de no equivocarse nunca, lo cual es frágil.

**5.3** Porque "acordarse del `WHERE`" pone la seguridad en manos de que **cada desarrollador, en cada query, para siempre**, no se olvide — un solo error es una fuga. **Row-Level Security** mueve el filtro a la **base**: una política que filtra **toda** consulta por el `tenant_id` de la sesión, **automáticamente**, aunque el `WHERE` falte en el código. El principio que aplica es **defensa en profundidad**: la última capa (la base) **no confía** en que la de arriba (la app) hizo todo bien, y garantiza el aislamiento por su cuenta.

**6.1** **Defensa en profundidad:** múltiples capas de seguridad, de modo que si una falla, otra detiene el ataque (ninguna capa es la única defensa). **Mínimo privilegio:** cada componente/credencial/persona tiene solo los permisos que necesita y nada más. La pregunta que revela si te falta profundidad: ***"si esta capa falla, ¿qué me queda?"*** — si la respuesta es "nada", dependés de una sola capa y te falta profundidad.

**6.2** Porque si el servicio de reportes (que solo lee) se **compromete** —un bug, una dependencia maliciosa, una inyección—, una credencial **read-only** limita el daño a **leer** datos; con una credencial de escritura, el atacante podría **borrar o alterar** la base. Dándole solo el permiso que **usa** (leer), el **blast radius** de comprometer ese servicio queda acotado a la lectura, no a la destrucción. Es **mínimo privilegio**: no se trata de evitar toda brecha, sino de que una brecha en un componente no se vuelva control total.

**6.3** Porque sin **registro de quién accedió a qué** no podés **detectar** una brecha, **responder** a ella (saber qué datos se tocaron, contener) ni **investigar** después (forense). La seguridad no es solo **prevenir** (que es imposible al 100%): es también **detectar y responder**, y eso requiere logs de auditoría (idealmente inmutables, como el log append-only del [Caso 14](system-design-casos.md)) y alertas sobre lo anómalo. No podés actuar sobre una brecha que **no ves** — por eso la auditoría es seguridad, no solo observabilidad.

**7.1** **Conviene delegar (gestionado):** la **autenticación** (Cognito/Auth0/Clerk — emisión/rotación de tokens, MFA, OAuth2/OIDC, que a mano son fuente de bugs), la **gestión de secretos y el cifrado** (Secrets Manager/Vault/KMS — "don't roll your own crypto"), y la **defensa de borde** (WAF/DDoS gestionados — capacidad que no igualás solo). **Siempre tuyas:** la **autorización de tu dominio** (quién puede hacer qué en tu lógica), el **aislamiento de tenants** (modelo + RLS), el **mínimo privilegio** en tus componentes, y **validar tu propio input**. Lo gestionado te da los ladrillos; el diseño seguro del sistema es responsabilidad tuya.

**7.2** Porque la seguridad tiene un **costo** (fricción, latencia, complejidad, velocidad del equipo) que tiene que estar en proporción al **valor de lo que protegés** y al **modelo de amenaza**. Mal calibrada **hacia arriba**, genera fricción que el equipo termina **esquivando** — el ejemplo clásico: el proceso de secretos es tan engorroso que alguien **pega el secreto en un Slack/Notion** "para zafar", y terminás **menos** seguro que con una solución más simple que el equipo sí use. La seguridad que la gente evade no protege nada.

**7.3** ***"¿Qué estoy protegiendo, de quién, y cuál es el costo de una brecha — y estoy invirtiendo en proporción?"*** Eso (el modelo de amenaza + el valor del activo) decide cuánta seguridad amerita cada capa: un side-project se conforma con auth gestionada + HTTPS + secretos en el gestor de la plataforma; un SaaS con datos sensibles de varios clientes justifica aislamiento de tenants serio, rotación, mTLS, WAF y auditoría. Ni sub-diseñar (te expone) ni sobre-diseñar (frena al equipo sin ganancia real).

---

> **Para seguir.** Las fuentes de verdad: **OWASP** (Top 10, cheat sheets — la referencia de ataques y defensas web), la doc de tu proveedor de secretos/KMS (**AWS KMS/Secrets Manager**, **HashiCorp Vault**), de auth gestionada (**Cognito/Auth0/Clerk**), de WAF/DDoS (**Cloudflare**, **AWS WAF/Shield**), y de **Postgres Row-Level Security**. Re-verificá lo marcado con ⚠️ (APIs y precios de los gestionados, RFC 7662 de introspection, capacidades de la service mesh): se mueven. Los puentes en el hub: [Autenticación](autenticacion.md) es la mecánica (JWT, refresh, RBAC) que este módulo lleva a escala; [Diseño de APIs](api-design.md) M10 es el contrato de auth (scopes, mínimo privilegio); [Networking](networking.md) es el borde (anycast/CDN, TLS) sobre el que se montan WAF/DDoS y mTLS; [Reverse proxy](reverse-proxy.md) M6 es el rate limiting de borde; [Datos a escala](datos-escala.md) es el multi-tenancy (y el noisy neighbor que **no** hay que confundir con el aislamiento de datos); y [Observabilidad](observabilidad.md) es la auditoría/detección sin la cual no respondés a una brecha.
