# Multi-tenancy full stack: aislar clientes de la base al frontend

**Tenant ≠ usuario, el espectro silo↔pool, modelos de aislamiento de datos, RLS de Postgres y scoping en el ORM, el contexto de tenant por todo el stack (AsyncLocalStorage), identidad y RBAC por tenant, frontend multi-tenant (subdominios, theming), y operar N clientes (vecino ruidoso, migraciones, caché, borrado) · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el módulo **canónico y full-stack** de multi-tenancy: el tema estaba repartido —el aislamiento de datos en [Datos a escala](datos-escala.md), la seguridad en [Seguridad de sistemas](seguridad-sistemas.md), el RBAC en [Autenticación](autenticacion.md), el BI embebido en [Analytics embebido](analytics-embebido.md)— y acá lo **juntamos de punta a punta**: cómo un mismo request de un cliente atraviesa frontend → backend → base **aislado** del resto. No re-enseñamos esos módulos: los conectamos.

**Lo que asumimos.** Que sabés qué es un **JWT** y qué es **RBAC** ([Autenticación](autenticacion.md)), que entendés **índices, transacciones y sharding** a nivel conceptual ([PostgreSQL](postgresql.md), [Datos a escala](datos-escala.md)), que viste **middleware** en un backend Node ([Express](express.md)) y que conocés **caché** ([Redis](redis.md)). No asumimos que hayas construido una SaaS multi-tenant.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Índice de módulos**
1. Qué es multi-tenancy: tenant ≠ usuario, y el espectro silo↔pool
2. Modelos de aislamiento de datos: columna, schema o base por tenant
3. Que el filtro nunca se olvide: RLS en Postgres y scoping en el ORM
4. El contexto de tenant por todo el stack: resolución y propagación
5. Identidad y permisos: usuarios en varios tenants y RBAC por tenant
6. El frontend multi-tenant: subdominios, theming y el switch de tenant
7. Operar N clientes: vecino ruidoso, migraciones, caché y borrado
8. El criterio: elegir modelo, tiers y cuándo evolucionar

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es multi-tenancy: tenant ≠ usuario, y el espectro silo↔pool

**Teoría.** Un **tenant** es un **cliente/organización** que usa tu producto — **no** un usuario individual. Una SaaS "VentasPro" contratada por **Acme** (5 empleados) y **Globex** (30 empleados) tiene **35 usuarios** pero **2 tenants**. La regla de negocio que lo define: un empleado de Acme ve **todo** lo de Acme y **nada** de Globex. La confusión tenant/usuario es el error #1 de quien arranca el tema; fijalo primero.

**Multi-tenant** = **una sola instancia de la app y su infraestructura sirven a muchos clientes a la vez**, cada uno aislado como si la app fuera solo suya. Lo opuesto, **single-tenant**, es desplegar una copia separada por cliente (más caro de operar, se justifica solo en casos de compliance extremo). La multi-tenancy es lo que hace **económicamente viable** el modelo SaaS: una base de código, un deploy, costos compartidos entre todos los clientes.

El aislamiento no es blanco o negro: es un **espectro**, y el vocabulario que se usa en la industria (AWS SaaS terminology) es:

- **Pool** (recursos **compartidos**): todos los tenants comparten la misma base, las mismas tablas, el mismo cómputo. Máxima eficiencia de costo y de operación (una migración, un backup), **mínimo aislamiento físico** (el aislamiento es puramente lógico, por software).
- **Silo** (recursos **dedicados**): cada tenant tiene su propia base (o hasta su propia infra). Máximo aislamiento (un tenant no puede tocar a otro ni por bug), **máximo costo y operación** (N bases que migrar, respaldar, monitorear).
- **Bridge** (híbrido): un punto intermedio (p. ej. base compartida pero un schema por tenant).

Casi ninguna SaaS es 100% pool o 100% silo: lo común es **tiered** (por plan) — los clientes chicos van a **pool** (baratos, densos), los *enterprise* que pagan por aislamiento y compliance van a **silo**. La decisión no es "cuál es mejor" sino "**cuánto aislamiento paga cada tenant**".

La frase mental: **tenant = cliente/organización, no usuario (un tenant tiene muchos usuarios). Multi-tenant = una instancia sirve a todos los clientes aislados, y eso es lo que hace viable el SaaS. El aislamiento es un espectro: pool (todo compartido, barato, aislamiento lógico) ↔ silo (todo dedicado, caro, aislamiento físico), con bridge en el medio. En la práctica es tiered: pool para los chicos, silo para enterprise.**

**Ejercicios 1**
1.1 🔁 ¿Cuál es la diferencia entre un tenant y un usuario? Dá un ejemplo con números.
1.2 🧠 Explicá el espectro pool↔silo: ¿qué ganás y qué perdés en cada extremo?
1.3 🧠 ¿Por qué la multi-tenancy es lo que hace económicamente viable a una SaaS, y por qué la mayoría termina con un modelo *tiered*?

---

## Módulo 2 — Modelos de aislamiento de datos: columna, schema o base por tenant

**Teoría.** El espectro del módulo 1 se materializa, sobre todo, en **cómo guardás los datos**. Tres modelos clásicos, de más pool a más silo:

**(a) Base y schema compartidos, columna `tenant_id`** (pool puro). Una sola tabla para todos; cada fila lleva de quién es.

```sql
| id | tenant_id | monto | producto |
|----|-----------|-------|----------|
| 1  | acme      | 1200  | Laptop   |
| 2  | globex    |  800  | Monitor  |
```

El aislamiento es **puramente lógico**: los datos están **físicamente mezclados** y lo único que separa a Acme de Globex es que **toda** query lleve `WHERE tenant_id = 'acme'`. Máxima densidad (miles de tenants en una base), operación mínima (una migración sirve a todos), pero el aislamiento **depende 100% de que el código nunca se olvide el filtro** (módulo 3). La `tenant_id` debe ir **primero en los índices** (`(tenant_id, ...)`) para que las queries de un tenant no escaneen los datos de los demás.

**(b) Base compartida, un schema por tenant** (bridge). Misma base, pero `acme.ventas` y `globex.ventas` son tablas separadas dentro de schemas distintos. Más aislamiento (más difícil filtrar mal), backup/restore por tenant más simple, pero se degrada con **miles** de schemas (el catálogo de la base sufre) y las migraciones ya son "por schema".

**(c) Una base por tenant** (silo). Acme y Globex tienen bases físicamente separadas. Aislamiento máximo (un bug no cruza datos; el *blast radius* de un incidente es un solo tenant), restore/borrado/compliance por tenant triviales — pero **N bases** que migrar y operar, y un problema real de **explosión de conexiones** (cada base necesita su pool → miles de tenants = miles de pools, y Postgres topea en `max_connections` —cientos, no miles—, así que a escala necesitás un pooler sí o sí; ver [PostgreSQL](postgresql.md)).

| | Pool (columna) | Bridge (schema) | Silo (base) |
|---|---|---|---|
| Densidad de tenants | Altísima | Media | Baja |
| Aislamiento | Lógico (frágil) | Medio | Físico (fuerte) |
| Migraciones | **Una** | Por schema | **N (fan-out)** |
| Backup/restore por tenant | Difícil | Medio | Trivial |
| *Blast radius* de un bug | Todos | Acotado | Un tenant |
| Costo por tenant | Mínimo | Medio | Alto |

El **eje que decide** casi siempre es: *¿cuántos tenants vas a tener y cuánto aislamiento/compliance exigen?* Miles de clientes chicos → **pool**. Decenas de clientes enterprise regulados → **silo**. Y la **evolución natural** de la mayoría de las SaaS: arrancás **pool** (rápido, barato) y **promovés a silo** los tenants grandes cuando el aislamiento empieza a pagar (esto es el modelo *tiered* del M1). En [Datos a escala](datos-escala.md) viste el sharding genérico; **el tenant es una sharding key natural** — repartir tenants entre shards (o bases) es la versión a escala del silo.

La frase mental: **el aislamiento de datos tiene tres modelos: columna `tenant_id` (pool: una tabla para todos, denso, aislamiento solo lógico, filtro obligatorio, tenant_id primero en el índice), schema por tenant (bridge), base por tenant (silo: aislamiento físico, blast radius de un tenant, pero N migraciones y explosión de conexiones). Se elige por cantidad de tenants × aislamiento exigido; lo normal es arrancar pool y promover a silo los grandes. El tenant es una sharding key natural.**

**Ejercicios 2**
2.1 🔁 Nombrá los tres modelos de aislamiento de datos y ordenálos de más pool a más silo.
2.2 🧠 En el modelo de columna `tenant_id`, ¿por qué la `tenant_id` tiene que ir **primero** en los índices compuestos? ¿Qué pasa si no?
2.3 🧠 Una SaaS con 3 clientes enterprise de salud (datos regulados) vs. una con 50.000 usuarios freemium. ¿Qué modelo para cada una y por qué? ¿Qué es el "blast radius" y cómo cambia entre modelos?

---

## Módulo 3 — Que el filtro nunca se olvide: RLS en Postgres y scoping en el ORM

**Teoría.** En el modelo pool, **todo** el aislamiento se reduce a una promesa frágil: que **cada** query lleve `WHERE tenant_id = ?`. Una sola query que se olvide el filtro —un reporte nuevo, un endpoint de admin, un `JOIN` mal armado— y un tenant ve datos de otro. **Confiar en que 200 queries escritas a mano nunca se equivoquen no es una estrategia de seguridad.** Necesitás que el filtro sea **estructural**, no disciplina humana. Dos capas de defensa:

**1. Row-Level Security (RLS) — la base lo impone.** Postgres puede aplicar el filtro **él mismo**, en el motor, para toda query sobre la tabla. Definís una **policy** y la base la agrega a cada `SELECT/UPDATE/DELETE` automáticamente:

```sql
-- 1. la tabla exige RLS
ALTER TABLE ventas ENABLE ROW LEVEL SECURITY;

-- 2. la policy: cada fila visible solo si su tenant_id == el del contexto de sesión
CREATE POLICY aislar_tenant ON ventas
  USING (tenant_id = current_setting('app.tenant_id', true)); -- 2 args (missing_ok): si no se seteó, NULL en vez de error

-- 3. en cada transacción, el backend fija el tenant actual (transaction-scoped)
SET LOCAL app.tenant_id = 'acme';
-- desde acá, TODA query en esta transacción ve solo filas de acme, aunque
-- el SQL diga  SELECT * FROM ventas  sin ningún WHERE.
```

La belleza: aunque un dev escriba `SELECT * FROM ventas` sin filtro, la base **solo** devuelve las de `acme`. El aislamiento deja de depender del código de aplicación. El `, true` (*missing_ok*) del `current_setting` es deliberado: si por lo que sea no se fijó el tenant, devuelve `NULL`, y `tenant_id = NULL` **no matchea ninguna fila** → *fail-closed natural* (sin el `true`, la query directamente **falla** — también fail-closed, pero ruidoso). Y RLS **no es gratis**: la policy se agrega como un **predicado a cada query**, así que tener el índice con `tenant_id` primero (M2) no es solo por el scan eficiente, sino para que el planner resuelva ese predicado barato. ⚠️ **Dos trampas reales:** (a) el **dueño de la tabla y los superusuarios bypassean RLS** por default — la app tiene que conectarse con un rol **sin** privilegios de dueño, o declarar `FORCE ROW LEVEL SECURITY`; (b) `SET LOCAL` es **por transacción**, lo cual es justo lo que querés con un **pooler en modo transacción** (PgBouncer), donde un `SET` de sesión se filtraría a la próxima request que reuse la conexión. Fijar el tenant **por transacción** es lo correcto y lo seguro.

**2. Scoping en el ORM — la app lo agrega.** La otra capa (complementaria, no excluyente): que el acceso a datos pase por un punto único que **inyecta** el filtro de tenant en cada query, en vez de repartir `WHERE` a mano. Casi todos los ORMs lo permiten: **Prisma** con *client extensions* que interceptan cada query, **Sequelize** con `scopes`, o un **repositorio propio** que toma el tenant del contexto (módulo 4) y lo agrega. La idea es la misma: **un solo lugar** donde el filtro se pone, imposible de olvidar porque nadie escribe queries crudas sueltas.

El principio que gobierna todo esto es **fail-closed** (denegar por defecto): si por lo que sea **no hay** un tenant en el contexto, la operación **falla**, no "trae todo". Un bug que hace fallar es infinitamente mejor que uno que filtra datos en silencio.

> 🔥 **Falla en 4 capas — el `WHERE tenant_id` olvidado (fuga entre tenants).**
> (1) **Qué se rompe:** un endpoint nuevo (un reporte, un export) ejecuta una query sin el filtro de tenant; un cliente ve datos de otro. Incidente crítico de una SaaS B2B.
> (2) **Por qué a esta escala:** en pool, los datos están físicamente mezclados; el aislamiento depende de que **cada** query —escrita por cualquiera, en cualquier momento— lleve el filtro. La probabilidad de un olvido tiende a 1 con el tamaño del equipo.
> (3) **Corto plazo:** revisar y tapar la query culpable; auditar los accesos a datos que no pasan por el repositorio central.
> (4) **Largo plazo:** hacer el filtro **estructural** — **RLS en la base** (fail-closed, imposible de bypassear desde la app) + **scoping único en el ORM** + prohibir queries crudas sueltas. El aislamiento no puede depender de que nadie se equivoque nunca.

La frase mental: **en pool, el aislamiento se cae a "que ninguna query olvide el WHERE tenant_id" — y eso no es seguridad. Hacelo estructural en dos capas: RLS en Postgres (el motor agrega el filtro a toda query vía una policy + `SET LOCAL app.tenant_id` por transacción; ojo: el dueño de la tabla bypassea salvo FORCE) y scoping único en el ORM (un punto que inyecta el filtro, sin WHERE sueltos). Principio: fail-closed — sin tenant en contexto, la operación falla, no trae todo.**

**Ejercicios 3**
3.1 🔁 ¿Qué es Row-Level Security en Postgres y por qué el aislamiento por RLS es más fuerte que el `WHERE` a mano?
3.2 🧠 ¿Por qué se fija el tenant con `SET LOCAL` (por transacción) y no con un `SET` de sesión? Relacionalo con un pooler en modo transacción.
3.3 🧠 Explicá el principio *fail-closed* en multi-tenancy. ¿Por qué un bug que hace fallar la request es preferible a uno que devuelve datos de más?

---

## Módulo 4 — El contexto de tenant por todo el stack: resolución y propagación

**Teoría.** Para que el filtro del módulo 3 funcione, la base (o el ORM) necesita saber **qué tenant es** en cada request. Eso son dos problemas: **resolverlo** (¿de dónde saco el tenant?) y **propagarlo** (¿cómo lo hago llegar hasta la query sin ensuciar todo el código?).

**Resolver el tenant** — de dónde sale el identificador, por orden de robustez:

- **Subdominio:** `acme.app.com` vs `globex.app.com`. El más común en SaaS B2B; el tenant es visible, cacheable por CDN, y da branding natural (M6).
- **Path:** `app.com/t/acme/...`. Simple, sin wildcard DNS ni certificados por subdominio.
- **Claim en el JWT / sesión:** el tenant va **dentro del token** que emitió tu backend al loguear. El más seguro para *autorizar* (no se puede falsificar).
- **Header** (`X-Tenant-Id`): común entre servicios internos, **nunca** como única fuente si viene del navegador.

**La regla de oro (otra vez, porque es la que se rompe):** el `tenant_id` que usás para **filtrar/autorizar** sale de una fuente **firmada por tu servidor** (el JWT/sesión), **nunca** de un valor que el cliente controla a piacere (un query param, un header editable). El subdominio/path sirven para *rutear y dar UX*, pero la autorización se ancla en el token. Si el subdominio dice `acme` pero el token del usuario es de `globex`, **gana el token** (y probablemente sea un intento de acceso cruzado que hay que rechazar).

**Propagar el tenant** — una vez resuelto, tiene que llegar hasta la capa de datos. La forma naíf es **pasarlo como parámetro** por cada función (`getVentas(tenantId, ...)`, `getUsuario(tenantId, ...)`) — funciona pero **contamina toda la firma** y, peor, si alguien se olvida de pasarlo, volvés al problema del filtro olvidado. La forma limpia en Node es **`AsyncLocalStorage`** (de `node:async_hooks`): un **almacenamiento por request** que "viaja" con el flujo asíncrono sin pasarlo explícitamente. Un middleware lo setea al entrar el request, y **cualquier** capa lo lee desde el contexto:

```
Request entra (acme.app.com, JWT de Ana)
   │
Middleware: resuelve tenant, verifica token, valida coherencia → AsyncLocalStorage.run({ tenantId:'acme', userId:'ana', roles:[...] })
   │
Handler → Servicio → Repositorio      (nadie pasa tenantId a mano)
   │
Repositorio lee currentTenant() del contexto → SET LOCAL app.tenant_id / WHERE tenant_id
```

Es el mismo patrón que un *request context* para trazas/logs ([Observabilidad](observabilidad.md)): setear una vez al borde, leer en cualquier profundidad.

La frase mental: **el tenant se resuelve (subdominio > path > claim de JWT > header) y se propaga hasta la query. Regla de oro: para filtrar/autorizar, el tenant sale del JWT/sesión firmado, NUNCA de un valor que el cliente controla; el subdominio rutea y da UX, el token autoriza, y si no coinciden gana el token. Para propagarlo sin ensuciar cada firma de función, usá AsyncLocalStorage (contexto por request): el middleware lo setea al borde, el repositorio lo lee al fondo.**

**Ejercicios 4**
4.1 🔁 Nombrá cuatro formas de resolver el tenant de un request y cuál es la fuente de verdad para *autorizar*.
4.2 🧠 El subdominio dice `acme` pero el JWT del usuario es de `globex`. ¿Qué hacés y por qué?
4.3 ✍️ Implementá en TypeScript un contexto de tenant con `AsyncLocalStorage`: `runWithTenant(ctx, fn)` que corre `fn` dentro del contexto, y `currentTenant()` que devuelve el contexto actual y **lanza** si no hay ninguno (fail-closed). El contexto tiene `tenantId`, `userId` y `roles`. Tipá todo; tiene que compilar en `--strict`. (Solución al final.)

---

## Módulo 5 — Identidad y permisos: usuarios en varios tenants y RBAC por tenant

**Teoría.** Acá se juntan multi-tenancy y [Autenticación](autenticacion.md), y aparece la sutileza que se pregunta en entrevista: **un usuario puede pertenecer a más de un tenant, con roles distintos en cada uno**. La misma persona (un consultor, un proveedor) puede ser `admin` en Acme y `viewer` en Globex. Por eso el modelo no es "el usuario tiene un rol" sino "el usuario tiene **membresías**, y cada membresía es (tenant, roles)".

```
Usuario "ana"
  ├── Membership { tenant: acme,   roles: [admin] }
  └── Membership { tenant: globex, roles: [viewer] }
```

Esto cambia la autorización: no alcanza con "¿Ana puede escribir ventas?". La pregunta correcta es **"¿Ana puede escribir ventas EN acme?"** — el permiso siempre se evalúa **dentro de un tenant**. Chequear el permiso sin anclar el tenant es un agujero: Ana podría usar su rol de admin de Acme para escribir en Globex.

El **RBAC** que ya viste ([Autenticación](autenticacion.md)) se aplica **scoped al tenant**: los roles y sus permisos se resuelven contra la **membresía en ese tenant**, no globalmente. La verificación tiene **dos pasos** que no se pueden saltear:

1. **¿El usuario es miembro de este tenant?** Si no tiene membresía en `acme`, denegado — antes incluso de mirar roles.
2. **¿Alguno de sus roles *en ese tenant* otorga el permiso?** Recién ahí evaluás el RBAC, con los roles de **esa** membresía.

Un detalle senior sobre el paso 1: para un recurso de **otro** tenant conviene devolver **404** (no existe), no **403** (prohibido) — un `403` le **confirma al atacante que ese recurso/tenant existe**; el `404` no filtra ni siquiera su existencia.

El token/sesión que emite tu backend suele llevar el tenant **activo** y los roles **en ese tenant** (`{ userId, tenantId, roles }`), justo lo que propagás por el contexto del módulo 4. Cuando el usuario **cambia de tenant** (el switcher del M6), se re-emite el token con la nueva membresía — y como el token **anterior** (con el tenant viejo) sigue válido hasta expirar, usá **tokens de vida corta** + una lista de revocación si el switch tiene que cortar el acceso previo al instante. Y ojo con un rol especial: el **super-admin de la plataforma** (tu equipo, no un cliente) que legítimamente ve *todos* los tenants — ese cruza el aislamiento a propósito, así que es el acceso más sensible de auditar.

> 🔥 **Falla en 4 capas — el permiso sin anclar al tenant (escalada cruzada).**
> (1) **Qué se rompe:** el chequeo es "¿Ana tiene rol admin?" (global) en vez de "¿Ana es admin **en este tenant**?"; Ana, admin en Acme y viewer en Globex, escribe datos de Globex.
> (2) **Por qué a esta escala:** con usuarios multi-tenant, un permiso sin tenant es ambiguo y se resuelve con el rol "más alto" que tenga la persona en cualquier lado.
> (3) **Corto plazo:** revisar los guards de autorización que no reciben/usan el tenant del contexto.
> (4) **Largo plazo:** autorización en **dos pasos obligatorios** (¿miembro de este tenant? → ¿rol en *esta* membresía otorga el permiso?), con el tenant tomado del contexto (M4), no de la elección del cliente.

La frase mental: **un usuario tiene membresías (tenant, roles), no un rol global — puede ser admin en Acme y viewer en Globex. Por eso el permiso SIEMPRE se evalúa dentro de un tenant, en dos pasos: (1) ¿es miembro de este tenant? (2) ¿sus roles EN esa membresía otorgan el permiso? El RBAC de Autenticación, pero scoped al tenant. El token lleva el tenant activo + roles en ese tenant; el switch re-emite el token. Cuidado con el super-admin de plataforma que cruza a propósito.**

**Ejercicios 5**
5.1 🔁 ¿Por qué el modelo correcto es "usuario → membresías (tenant, roles)" y no "usuario → rol"? Dá un ejemplo.
5.2 🧠 ¿Cuáles son los dos pasos de una verificación de permiso multi-tenant y por qué no se puede saltear el primero?
5.3 ✍️ Implementá en TypeScript `can(user, tenantId, permiso)`: el `User` tiene `memberships: { tenantId, roles }[]`; la función devuelve `true` solo si el usuario es miembro de `tenantId` **y** alguno de sus roles *en esa membresía* otorga el `permiso` (usá un mapa rol→permisos). Tipá todo; compila en `--strict`. (Solución al final.)

---

## Módulo 6 — El frontend multi-tenant: subdominios, theming y el switch de tenant

**Teoría.** El full-stack incluye el navegador, y la multi-tenancy también se ve del lado del cliente — pero con una advertencia que no se repite lo suficiente: **el frontend es presentación, no seguridad.** Todo lo que el front haga con el tenant es para *UX y ruteo*; el aislamiento **real** ya se garantizó en el backend/base (M3-M5). Un tenant malicioso puede reescribir el JavaScript del front a gusto — si tu aislamiento dependiera de código del navegador, no tenés aislamiento.

Dicho eso, el front resuelve cosas concretas:

- **Ruteo por tenant:** del subdominio (`acme.app.com`) o el path (`/t/acme`) el front deriva a qué tenant está mirando el usuario, y arma las llamadas a la API en consecuencia. El tenant viaja en el **token** (header `Authorization`), no como un dato que el usuario tipea.
- **Theming / white-label:** cada tenant puede tener su logo, colores, hasta su dominio propio (`analytics.acme.com` apuntando a tu app). Se resuelve cargando una **config de branding por tenant** al iniciar (tokens de diseño, feature flags). Es una de las razones por las que el **subdominio** gana como método de resolución: da identidad visual natural.
- **El switch de tenant:** para el usuario multi-tenant (Ana en Acme y Globex), un selector para cambiar de organización. Clave: **cambiar de tenant NO es un filtro de UI** — re-emite el token/sesión con la nueva membresía (M5) y **recarga el contexto** (datos, permisos, branding). Si el switch solo cambiara lo que se muestra sin renovar el token, tendrías datos de un tenant con permisos de otro.

Un detalle de seguridad que cruza front y back: **no filtres en el cliente lo que ya trajiste del servidor**. Si la API devuelve "todo" y el front oculta lo que no corresponde, los datos igual viajaron al navegador → fuga. El servidor debe devolver **solo** lo del tenant activo; el front nunca "esconde" datos que no debería haber recibido.

La frase mental: **el frontend multi-tenant hace ruteo (del subdominio/path deriva el tenant), theming/white-label (config de branding por tenant) y el switch de tenant — pero es presentación, no seguridad: el aislamiento real vive en backend/base. El switch NO es un filtro de UI: re-emite el token con la nueva membresía y recarga el contexto. Y nunca filtres en el cliente lo que el server no debió mandar: si viajó al navegador, ya se filtró.**

**Ejercicios 6**
6.1 🔁 ¿Qué tres cosas resuelve el frontend en una app multi-tenant, y cuál NO es (seguridad)?
6.2 🧠 ¿Por qué cambiar de tenant en el switcher tiene que re-emitir el token en vez de solo cambiar lo que se muestra?
6.3 🧠 Un dev implementa "el admin ve todos los tenants" trayendo todo del server y ocultando en el front lo que no corresponde. ¿Qué está mal?

---

## Módulo 7 — Operar N clientes: vecino ruidoso, migraciones, caché y borrado

**Teoría.** Multi-tenancy no termina en el aislamiento de datos: aparece un conjunto de problemas **operativos** que solo existen cuando muchos clientes comparten recursos. Los que más se preguntan:

- **Vecino ruidoso (*noisy neighbor*):** en pool, un tenant que dispara consultas pesadas o mucho tráfico **degrada a todos** los demás (comparten CPU, conexiones, IO). Se mitiga con **cuotas y rate limiting *por tenant*** (no global), y a escala, promoviendo al tenant pesado a **silo** (M2) para que su carga no toque a nadie. El rate limiting por tenant es el de [Seguridad de sistemas](seguridad-sistemas.md)/[Concurrencia](concurrencia.md), pero con la **tenant_id como clave del límite**.
- **Caché envenenado entre tenants:** si cacheás con una clave que **no incluye el tenant** (`cache["ventas:top10"]`), el resultado de Acme se le sirve a Globex. **La `tenant_id` es parte de la clave de caché, siempre** (`cache["acme:ventas:top10"]`). Es exactamente el problema del aislamiento de caché de pre-aggs de [Analytics embebido](analytics-embebido.md) M5, generalizado a **todo** caché ([Redis](redis.md)).
- **Migraciones (*fan-out*):** en pool, una migración de schema corre **una vez** y afecta a todos — simple, pero **arriesgado** (un error toca a todos los tenants a la vez). En silo, la misma migración corre **N veces** (una por base) — aislás el riesgo pero tenés que orquestar N ejecuciones, tolerar fallas parciales (el tenant 347 falló) y manejar que queden versiones distintas transitoriamente. Es un trade-off directo del modelo del M2.
- **Observabilidad por tenant:** meté la `tenant_id` en **logs, métricas y trazas** ([Observabilidad](observabilidad.md)). Sin eso, no podés responder "¿por qué Acme ve lentitud?" ni atribuir costo/uso. Es también la base del **billing por uso** (metering por tenant).
- **Provisioning y offboarding:** dar de alta un tenant (crear su fila/schema/base, seed inicial, config) y **darlo de baja** — que en multi-tenant es delicado: **borrar/exportar los datos de un solo tenant** (por GDPR o fin de contrato). En pool es un `DELETE WHERE tenant_id` masivo (y probar que no quede nada colgado); en silo es *drop* de la base (trivial y limpio) — otra vez el modelo del M2 decide cuán fácil es. Y ojo con la falsa sensación de *"borré, listo"*: el dato eliminado **sigue en backups, WAL y réplicas** ([Replicación](replicacion.md)) hasta que esos expiren, así que el borrado *de verdad* (GDPR) incluye la **política de retención**, no solo el `DELETE`.

> 🔥 **Falla en 4 capas — el vecino ruidoso tira la performance de todos.**
> (1) **Qué se rompe:** un tenant lanza un reporte carísimo (o un bucle de requests) y **todos** los demás sufren latencia; el aislamiento de *datos* estaba bien, pero no el de *recursos*.
> (2) **Por qué a esta escala:** en pool se comparten CPU/conexiones/IO; sin límites por tenant, uno puede acaparar el pool entero.
> (3) **Corto plazo:** rate limiting y timeouts **por tenant**; matar la query que acapara.
> (4) **Largo plazo:** cuotas por tenant (según plan), *bulkheads*/pools separados para los pesados, y **promover a silo** los tenants que justifican recursos dedicados. Aislás recursos, no solo datos.

La frase mental: **multi-tenancy suma problemas de operación: vecino ruidoso (un tenant degrada a todos → rate limit/cuotas POR tenant, y silo para los pesados), caché envenenado (la tenant_id SIEMPRE en la clave de caché), migraciones (pool: una, riesgo compartido; silo: N, riesgo aislado pero orquestado), observabilidad (tenant_id en logs/métricas/trazas → debugging y billing), y provisioning/borrado por tenant (fácil en silo, DELETE masivo en pool). El modelo de datos del M2 decide cuán duro es cada uno.**

**Ejercicios 7**
7.1 🔁 ¿Qué es el problema del "vecino ruidoso" y con qué se mitiga? ¿Por qué el rate limiting tiene que ser por tenant?
7.2 🧠 ¿Por qué la `tenant_id` tiene que ser parte de la clave de caché? Dá el ejemplo concreto de qué pasa si no.
7.3 🧠 Compará hacer una migración de schema en pool vs. en silo: ¿qué es más simple y qué es más riesgoso en cada uno?

---

## Módulo 8 — El criterio: elegir modelo, tiers y cuándo evolucionar

**Teoría.** El cierre senior, como en los casos del banco: **no hay un modelo "correcto"; hay un modelo correcto para tu combinación de tenants, aislamiento y escala** — y la respuesta madura suele ser **evolucionar**, no elegir de entrada el monumento.

**El criterio, condensado:**
- **Arrancás pool** (columna `tenant_id` + RLS + scoping en el ORM) casi siempre: es lo más rápido de construir, lo más denso y lo más barato de operar. Para el 90% de las SaaS B2B en su etapa temprana, esto **es** la respuesta.
- **Promovés a bridge/silo** cuando un tenant lo **justifica**: enterprise que paga por aislamiento, requisito de compliance (datos en su propia base/región), o un tenant tan grande que el vecino ruidoso ya no se contiene con cuotas. Esto es el modelo **tiered**.
- **Vas a base/shard por tenant** cuando la escala de un tenant sola no entra en la base compartida — el tenant como **sharding key** ([Datos a escala](datos-escala.md)).

**El anti-patrón inverso es igual de real:** arrancar **silo "por las dudas"** para tres clientes te hace pagar desde el día uno la explosión de conexiones y las N migraciones (M2) sin que ningún tenant lo justifique todavía. Sub-aislar es un riesgo de **seguridad**; sobre-aislar es un **impuesto operativo** — los dos son errores, y por eso el default es *pool y evolucionar*.

**La respuesta Staff narra una evolución** (como en [Analytics embebido](analytics-embebido.md) y el banco de casos): *v1* pool con RLS (lo más simple que aísla de verdad) → *v2* cuotas por tenant + caché con tenant en la clave + observabilidad por tenant (cuando crece el número de clientes y aparece el vecino ruidoso) → *v3* tiers: silo para enterprise, sharding por tenant para los gigantes (cuando el aislamiento/escala de algunos lo paga). Mostrar **qué cuello concreto dispara cada salto** demuestra criterio; dibujar el silo desde el día uno para tres clientes es sobre-ingeniería.

Estructurá cualquier pregunta de "diseñá una SaaS multi-tenant" con **las tres cosas**: **happy path** (resolver tenant del subdominio/token → contexto por request → RLS + scoping en la base → RBAC por membresía → front que solo rutea/tematiza), **modelo de falla** (filtro olvidado → fuga; permiso sin anclar al tenant → escalada; caché sin tenant → envenenamiento; vecino ruidoso → degradación), **recuperación** (RLS fail-closed, autorización en dos pasos, tenant en la clave de caché, cuotas por tenant, y promoción a silo del que lo necesite).

La frase mental de cierre: **no elijas el monumento: arrancá pool (tenant_id + RLS + scoping), es la respuesta para casi toda SaaS temprana. Promové a bridge/silo el tenant que lo justifique (enterprise, compliance, escala) — modelo tiered — y a shard por tenant cuando uno solo no entra. La respuesta Staff narra la evolución v1→v2→v3 nombrando el cuello que dispara cada salto, no dibuja el silo para tres clientes. Las tres cosas: aislar datos (RLS), aislar permisos (RBAC por membresía) y aislar recursos (cuotas por tenant) son ejes distintos — necesitás los tres.**

**Ejercicios 8**
8.1 🔁 ¿Por qué "arrancá pool" es el default para casi toda SaaS temprana?
8.2 🧠 Nombrá tres disparadores concretos que justifican promover un tenant de pool a silo.
8.3 🧠 Contá la evolución v1→v2→v3 de una SaaS multi-tenant, nombrando el cuello que dispara cada salto. Aplicá "las tres cosas".

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Un **tenant** es un **cliente/organización**; un **usuario** es una persona que se loguea. Un tenant tiene **muchos** usuarios. Ejemplo: la SaaS la usan Acme (5 empleados) y Globex (30 empleados) → **2 tenants, 35 usuarios**. El aislamiento se define entre tenants (Acme no ve nada de Globex), no entre usuarios del mismo tenant.

**1.2** **Pool** (compartido): máxima densidad y mínima operación (una base, una migración, un backup) y costo mínimo por tenant, pero **aislamiento solo lógico** (frágil, depende del software) y **blast radius** = todos. **Silo** (dedicado): **aislamiento físico** fuerte (un bug no cruza datos, blast radius = un tenant), backup/restore/borrado por tenant triviales, pero **costo y operación altos** (N bases que migrar/monitorear) y baja densidad. Ganás aislamiento y perdés eficiencia a medida que te movés de pool a silo.

**1.3** Porque **una sola instancia/infra/base de código sirve a todos los clientes**, así que los costos fijos (deploy, operación, cómputo) se **amortizan** entre todos en vez de replicarse por cliente — eso es lo que hace rentable cobrar una suscripción mensual. Termina **tiered** porque los clientes no son homogéneos: los chicos quieren precio bajo (→ pool denso) y los enterprise pagan por aislamiento/compliance/performance dedicada (→ silo). Un solo modelo no sirve a ambos: pool para los chicos, silo para los grandes.

**2.1** (más pool → más silo) (a) **Base y schema compartidos con columna `tenant_id`** (pool); (b) **base compartida, un schema por tenant** (bridge); (c) **una base por tenant** (silo).

**2.2** Porque las queries **siempre** filtran por tenant (`WHERE tenant_id = ?`), así que un índice que **empieza** por `tenant_id` (`(tenant_id, fecha)`, `(tenant_id, producto)`) le permite a la base ir directo a las filas de ese tenant y recorrer solo esas. Si `tenant_id` **no** va primero (o no está en el índice), la query de un tenant termina escaneando/filtrando sobre datos de **todos** los tenants → lento y con el índice desperdiciado. A escala, con el tenant primero, cada tenant "ve" prácticamente su propia partición del índice.

**2.3** **Salud regulada, 3 enterprise:** **silo** (base por tenant) — el compliance (datos aislados, a veces en región propia), el blast radius acotado a un tenant y el borrado/export por cliente lo justifican, y con solo 3 el costo de N bases es manejable. **50.000 freemium:** **pool** (columna `tenant_id`) — la densidad es obligatoria (50.000 bases sería inviable en costo y operación) y esos clientes no pagan por aislamiento físico. El **blast radius** es el alcance del daño de un incidente (bug, fuga, caída): en pool un problema puede tocar a **todos** los tenants; en silo queda contenido a **uno**. Por eso los datos sensibles/regulados empujan a silo.

**3.1** **RLS (Row-Level Security)** es una feature de Postgres donde definís **policies** que la base aplica a **toda** query sobre una tabla, filtrando las filas que cada sesión puede ver (p. ej. solo las del `tenant_id` fijado en el contexto). Es más fuerte que el `WHERE` a mano porque el filtro lo impone el **motor**, no el código de aplicación: aunque un dev escriba `SELECT * FROM ventas` sin filtro, la base solo devuelve las del tenant actual. Deja de depender de que **cada** query —presente y futura— recuerde el filtro (disciplina humana), que es exactamente lo que falla.

**3.2** Porque `SET LOCAL` es **transaction-scoped**: el valor vive solo mientras dura la transacción y se limpia al terminar. Con un **pooler en modo transacción** (PgBouncer), una misma conexión física se **reutiliza** entre requests de distintos tenants; un `SET` de **sesión** persistiría en la conexión y se **filtraría** a la próxima request que la reuse (le pondrías el tenant equivocado). `SET LOCAL` dentro de la transacción de cada request garantiza que el tenant se fija y se descarta **con** esa unidad de trabajo, sin fugarse a la siguiente.

**3.3** **Fail-closed** = ante la ausencia de información o un error, **denegar** por defecto (no permitir). En multi-tenancy: si no hay un tenant en el contexto, la operación **falla**, en vez de "traer todo" o "no filtrar". Un bug que **falla** es preferible a uno que **devuelve datos de más** porque el primero es **ruidoso y visible** (rompe, se detecta, se arregla) mientras que el segundo es **silencioso**: la app "funciona", nadie nota nada, y estás filtrando datos entre clientes hasta que alguien lo descubre (o lo explota). En seguridad, romper es más seguro que sobre-exponer.

**4.1** (1) **Subdominio** (`acme.app.com`), (2) **path** (`/t/acme`), (3) **claim en el JWT/sesión**, (4) **header** (`X-Tenant-Id`). La **fuente de verdad para autorizar** es el **JWT/sesión firmado por tu backend**: es lo único que el cliente no puede falsificar. El subdominio y el path sirven para **rutear y dar UX**, pero no para decidir a qué datos accede alguien.

**4.2** **Gana el token**: se sirve (o se deniega) según el tenant del **JWT**, no según el subdominio. Si el usuario tiene una sesión de `globex` pero entró por `acme.app.com`, o es un error de ruteo o es un **intento de acceso cruzado** — lo correcto es **rechazar/redirigir** (no servir datos de acme con un token de globex, ni al revés). El subdominio es una pista de UX manipulable (cualquiera escribe otra URL); la autorización se ancla en la fuente firmada.

**4.3**
```ts
import { AsyncLocalStorage } from 'node:async_hooks';

interface TenantContext {
  tenantId: string;
  userId: string;
  roles: readonly string[];
}

const storage = new AsyncLocalStorage<TenantContext>();

// corre fn dentro del contexto de un tenant (lo setea el middleware al borde del request)
function runWithTenant<T>(ctx: TenantContext, fn: () => T): T {
  return storage.run(ctx, fn);
}

// lee el contexto actual; fail-closed: si no hay tenant, lanza (no asume nada)
function currentTenant(): TenantContext {
  const ctx = storage.getStore();
  if (ctx === undefined) {
    throw new Error('Sin contexto de tenant: operación denegada');
  }
  return ctx;
}
```
Idea: `AsyncLocalStorage` mantiene un valor **por cadena asíncrona** (por request), así que el middleware llama `runWithTenant(ctx, () => handler())` una vez al entrar y **cualquier** capa por debajo (servicio, repositorio) llama `currentTenant()` sin recibir el tenant como parámetro. El `getStore()` devuelve `TenantContext | undefined`; el chequeo explícito de `undefined` es lo que hace el **fail-closed** (y satisface `--strict`): sin contexto, se lanza en vez de devolver algo inseguro. Compila en `--strict`/`--noUncheckedIndexedAccess`.

**5.1** Porque una misma persona puede pertenecer a **varios tenants con roles distintos en cada uno**: un consultor puede ser `admin` en Acme y `viewer` en Globex. Con "usuario → rol" (global) no podés representar eso — o le das admin en todos lados o en ninguno. Con "usuario → **membresías** (tenant, roles)", el rol es **relativo al tenant**, que es como funciona de verdad: el permiso de Ana depende de **en qué tenant** está actuando.

**5.2** (1) **¿Es miembro de este tenant?** — buscar la membresía del usuario para ese `tenantId`; si no existe, **denegar** ya. (2) **¿Algún rol de *esa* membresía otorga el permiso?** — recién ahí evaluás el RBAC, usando los roles de **esa** membresía (no los de otro tenant). No se puede saltear el primero porque si evaluás el permiso con "los roles del usuario" sin anclar el tenant, usás el rol **más alto que tenga en cualquier lado** → Ana (admin en Acme) escribiría en Globex donde solo es viewer. El paso 1 ata la autorización al tenant correcto.

**5.3**
```ts
type Permission = 'ventas:leer' | 'ventas:escribir' | 'admin';

interface Membership {
  tenantId: string;
  roles: readonly string[];
}

interface User {
  id: string;
  memberships: readonly Membership[];
}

const ROLE_PERMISSIONS: Record<string, readonly Permission[]> = {
  viewer: ['ventas:leer'],
  editor: ['ventas:leer', 'ventas:escribir'],
  admin: ['ventas:leer', 'ventas:escribir', 'admin'],
};

function can(user: User, tenantId: string, permiso: Permission): boolean {
  // paso 1: ¿es miembro de ESTE tenant?
  const membership = user.memberships.find((m) => m.tenantId === tenantId);
  if (membership === undefined) return false; // no es miembro → denegado

  // paso 2: ¿algún rol de ESA membresía otorga el permiso?
  return membership.roles.some((rol) => {
    const perms = ROLE_PERMISSIONS[rol];
    return perms !== undefined && perms.includes(permiso);
  });
}
```
Idea: `find` implementa el **paso 1** (sin membresía en ese tenant, `false` inmediato, antes de mirar roles). El `some` implementa el **paso 2**, evaluando el RBAC solo contra los roles de **esa** membresía. El `perms !== undefined` es obligatorio con `--noUncheckedIndexedAccess`: indexar `ROLE_PERMISSIONS` por un `string` arbitrario (un rol desconocido) devuelve `readonly Permission[] | undefined`, y un rol que no esté en el mapa simplemente no aporta permisos. Compila en `--strict`.

**6.1** El frontend resuelve (1) **ruteo por tenant** (del subdominio/path deriva qué tenant se mira y arma las llamadas a la API), (2) **theming/white-label** (branding, colores, dominio propio por tenant), y (3) el **switch de tenant** (selector para el usuario multi-tenant). Lo que **NO** es: **seguridad/aislamiento** — eso vive en backend/base (RLS, RBAC por membresía). El front es presentación; un tenant puede reescribir el JS del navegador, así que el aislamiento nunca puede depender de él.

**6.2** Porque el tenant activo determina **datos, permisos y branding**, y todo eso se ancla en el **token/sesión** (M4-M5), no en el estado de la UI. Si el switch solo cambiara lo que se muestra, seguirías con el **token del tenant anterior**: pedirías datos de Globex con un token de Acme (la API, bien hecha, te los niega) o —peor, si la API confiara en la UI— verías datos de un tenant con los permisos de otro. Re-emitir el token con la **nueva membresía** hace que backend y base apliquen el aislamiento correcto para el tenant nuevo. Cambiar de tenant es cambiar de contexto de seguridad, no de filtro visual.

**6.3** Que **los datos de todos los tenants viajaron al navegador**. "Ocultar en el front" es un filtro visual: los datos ya salieron del servidor, están en la respuesta HTTP / memoria del cliente, y cualquiera los ve en las DevTools o en la pestaña de red → **fuga**. Lo correcto es que el **servidor** devuelva solo lo que corresponde (para un super-admin legítimo, con auditoría; para un usuario normal, solo su tenant). El front nunca debe recibir datos que después "esconde".

**7.1** El **vecino ruidoso** es cuando un tenant, en recursos compartidos (pool), **consume tanto** (queries pesadas, mucho tráfico) que **degrada la performance de los demás**. Se mitiga con **rate limiting, cuotas y timeouts** y, a escala, promoviendo al tenant pesado a **silo** (recursos dedicados). El rate limiting tiene que ser **por tenant** (con la `tenant_id` como clave del límite) porque un límite **global** o bien no frena al abusador (si el global es alto) o bien castiga a los tenants buenos por culpa del ruidoso (si es bajo); por tenant, cada uno tiene su cuota y el que se pasa se frena **solo a sí mismo**.

**7.2** Porque una entrada de caché es un valor **compartido en memoria**: si la clave no distingue el tenant (`"ventas:top10"`), la **primera** respuesta que se cachee (la de Acme) se le sirve a **cualquiera** que pida esa clave, incluido Globex → Globex ve datos de Acme. Ejemplo: Acme pide su top 10 de ventas, se cachea bajo `"ventas:top10"`; Globex pide lo suyo, hay *cache hit* y recibe **el de Acme**. Con la `tenant_id` en la clave (`"acme:ventas:top10"` vs `"globex:ventas:top10"`) cada tenant tiene su propia entrada y no hay cruce. Es el mismo problema del aislamiento de pre-aggs de [Analytics embebido](analytics-embebido.md), generalizado.

**7.3** **Pool:** la migración corre **una sola vez** sobre la base compartida → **simple** de ejecutar y orquestar, pero **riesgosa** porque un error (una migración que rompe) impacta a **todos** los tenants a la vez (blast radius máximo). **Silo:** la misma migración corre **N veces** (una por base) → **más compleja** de orquestar (N ejecuciones, fallas parciales como "el tenant 347 falló", versiones distintas transitoriamente entre tenants), pero el **riesgo está aislado**: un error afecta a un tenant, no a todos, y podés hacer *rollout* gradual (canary por tenant). Es el trade-off simplicidad-vs-blast-radius del modelo del M2.

**8.1** Porque es lo **más rápido de construir, lo más denso y lo más barato de operar** (una base, una migración, un backup, un deploy), y para casi toda SaaS en etapa temprana el aislamiento **lógico** (columna `tenant_id` + RLS + scoping) es **suficiente**: los clientes chicos no pagan por —ni necesitan— aislamiento físico. Montar silo desde el día uno para pocos clientes es **sobre-ingeniería**: pagás la operación de N bases sin que ningún tenant lo justifique todavía. Arrancás simple y evolucionás cuando un cuello real aparece.

**8.2** (1) **Compliance/regulación:** un cliente exige sus datos en una base/región propia (salud, finanzas, sector público). (2) **Aislamiento pagado (enterprise):** un cliente grande paga por performance dedicada y garantía de que ningún vecino lo afecte. (3) **Escala/vecino ruidoso:** un tenant tan grande o tan intensivo que ya no se contiene con cuotas dentro del pool, o que directamente **no entra** en la base compartida (→ base/shard propio). Cualquiera de los tres justifica el costo extra del silo para *ese* tenant (tiered).

**8.3** **v1 — pool:** columna `tenant_id` + **RLS** (fail-closed) + scoping en el ORM + RBAC por membresía; resuelve el aislamiento de datos y permisos de forma barata y densa. **Cuello que dispara v2:** crece el número de clientes y aparece el **vecino ruidoso** + necesidad de *debuggear/facturar* por cliente → se agregan **cuotas y rate limiting por tenant**, **tenant_id en la clave de caché** y **observabilidad por tenant** (logs/métricas/trazas). **Cuello que dispara v3:** algunos tenants exigen **compliance/aislamiento** o crecen tanto que no entran → **tiers**: **silo** (base por tenant) para enterprise y **sharding por tenant** para los gigantes, dejando el pool para la larga cola de clientes chicos. **Las tres cosas:** *happy path* (resolver tenant del token → contexto por request → RLS + scoping → RBAC por membresía → front que rutea/tematiza); *modelo de falla* (filtro olvidado → fuga; permiso sin tenant → escalada; caché sin tenant → envenenamiento; vecino ruidoso → degradación); *recuperación* (RLS fail-closed, autorización en dos pasos, tenant en la clave de caché, cuotas por tenant, promoción a silo del que lo necesite).

---

> **Para seguir.** Las fuentes de verdad: la doc de **Postgres Row-Level Security** (`CREATE POLICY`, `FORCE ROW LEVEL SECURITY`, `SET LOCAL`/`set_config`), la de tu **ORM** (Prisma *client extensions*, Sequelize *scopes*) y la de **`node:async_hooks`** (`AsyncLocalStorage`). Re-verificá lo marcado con ⚠️ (el bypass de RLS por el dueño de la tabla y la semántica de `SET LOCAL` con poolers cambian con la versión y la config). Los puentes en el hub: [Datos a escala](datos-escala.md) es el sharding donde **el tenant es la sharding key** y el aislamiento de datos a escala; [Autenticación](autenticacion.md) es el **RBAC** que acá aplicamos *scoped* al tenant; [Seguridad de sistemas](seguridad-sistemas.md) es el aislamiento y el rate limiting; [Analytics embebido](analytics-embebido.md) M5 es este mismo aislamiento aplicado al **BI multi-tenant** (el `queryRewrite` y el caché de pre-aggs por tenant); [Redis](redis.md) es el caché donde la **tenant_id va en la clave**; [Observabilidad](observabilidad.md) es la **tenant_id en logs/métricas/trazas**; y [PostgreSQL](postgresql.md) es la base donde vive la RLS. **Y hacia dónde va el tema:** el aislamiento por tenant se está llevando también a los datos de **IA** — cada tenant con su propio espacio en la [vector database](vector-dbs.md) para que el [RAG](rag.md) de un cliente nunca recupere documentos de otro (el mismo `WHERE tenant_id`, ahora en el filtro de metadata del vector store).
