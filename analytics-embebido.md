# Analytics embebido y la capa semántica: del warehouse al dashboard multi-tenant

**OLTP vs OLAP, el data warehouse y el query-on-the-lake (Athena), la capa semántica (Cube.dev), pre-agregaciones, y row-level security para BI embebido en una SaaS multi-tenant · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo cubre el stack que aparece cuando tu producto **es** (o tiene) un dashboard de analytics que cada cliente ve con **sus** datos: el clásico de las SaaS B2B. Conecta cosas que ya viste por separado — el [warehouse/OLAP columnar](datos-escala.md) (M11-14), el [CDC/ETL](streams.md) que carga los datos, la [multi-tenancy](datos-escala.md) y el [RBAC](autenticacion.md), el [caché](redis.md) y la [economía del diseño](costo.md) — y los junta en una arquitectura concreta: **OLTP → pipeline → warehouse → capa semántica → dashboard embebido**.

**Lo que asumimos.** Que entendés índices y que un `GROUP BY ... SUM(...)` sobre millones de filas es caro, que viste el trade-off **row-store vs columnar** y la idea de **OLAP** a nivel conceptual ([Datos a escala](datos-escala.md) M11-14), que sabés qué es un **JWT** y qué es **RBAC / multi-tenancy** ([Autenticación](autenticacion.md), [Seguridad de sistemas](seguridad-sistemas.md)), y que conocés **S3** y el modelo de **pago por uso** de AWS ([AWS](aws.md), [AWS hands-on](aws-practica.md)). No asumimos que hayas tocado un warehouse, Athena ni Cube.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Índice de módulos**
1. Por qué no corrés analytics sobre tu Postgres de producción (OLTP vs OLAP)
2. El data warehouse y el query-on-the-lake: Athena, Parquet y el pago por scan
3. La capa semántica: definir la métrica una sola vez (Cube.dev)
4. Pre-agregaciones y caché: que el dashboard no pegue al warehouse en cada request
5. Analytics embebido multi-tenant: row-level security en serio
6. El criterio: build vs buy y la arquitectura completa

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Por qué no corrés analytics sobre tu Postgres de producción (OLTP vs OLAP)

**Teoría.** Tu base transaccional (la Postgres que sirve la app) está afinada para **OLTP** (*Online Transaction Processing*): muchísimas transacciones **chicas**, point reads/writes por clave primaria, escrituras concurrentes, datos **normalizados**. El motor es **row-oriented**: guarda cada fila junta en disco, porque cuando traés "el pedido #4012" querés **todas sus columnas** de una.

El analytics es el problema **opuesto** — **OLAP** (*Online Analytical Processing*): pocas consultas **enormes** que **agregan** (`SUM`, `COUNT`, `AVG ... GROUP BY`) sobre **millones de filas** pero **pocas columnas** ("facturación por mes y por región del último año"). Correr eso sobre un row-store es ineficiente por dos razones que se suman:

1. **El row-store lee de más.** Para sumar **una** columna (`monto`) sobre 50M de filas, igual tiene que leer de disco **filas enteras** (con las otras 30 columnas que no te interesan). El **columnar** guarda cada columna por separado: lee **solo `monto`**, y como una columna tiene valores homogéneos, **comprime muchísimo** (10×+) y se procesa **vectorizado**. Esa es la diferencia OLAP que ya viste en [Datos a escala](datos-escala.md) M13.
2. **Compite con la carga transaccional.** La query analítica de 40 segundos toma locks, llena el buffer cache de páginas que no le sirven a nadie más, y satura IO/CPU **mientras** los usuarios intentan hacer checkout. Una consulta pesada degrada **toda** la app.

El primer paso —y muchas veces el único que necesitás— es una **réplica de lectura** ([Replicación](replicacion.md)): mandás el analytics a un nodo aparte para que no pegue al primario. Resuelve el punto (2) (aislamiento), pero **no** el (1): la réplica sigue siendo un row-store, así que la query enorme sigue siendo lenta. Cuando el volumen crece, el dato analítico se mueve a un sistema **columnar** dedicado (el warehouse, módulo 2), cargado por un pipeline **ETL/ELT** o por **CDC** desde la OLTP ([Streams](streams.md)). El warehouse vive **desacoplado** de producción: que una query tarde 30s ahí no afecta a nadie.

> 🔥 **Falla en 4 capas — la query del dashboard tumba producción.**
> (1) **Qué se rompe:** alguien abre el panel de "métricas del año" y dispara un `GROUP BY` sobre toda la tabla de eventos en la **Postgres primaria**; la latencia transaccional de toda la app se va a las nubes.
> (2) **Por qué a esta escala:** un row-store lee filas enteras para agregar una columna y compite por IO/CPU/cache con el tráfico OLTP en la misma instancia.
> (3) **Corto plazo:** mover esas consultas a una **réplica de lectura** (aísla, no acelera) y ponerles `statement_timeout`.
> (4) **Largo plazo:** separar el plano analítico — **columnar/warehouse** cargado por ETL/CDC, desacoplado del OLTP, donde una query larga no le hace daño a nadie.

La frase mental: **OLTP (row-store, transacciones chicas, point access) y OLAP (columnar, agregaciones enormes sobre pocas columnas) son cargas opuestas. Correr analytics sobre tu Postgres de producción lee de más y compite con el tráfico real. Primer paso: réplica de lectura (aísla). Paso de escala: columnar/warehouse desacoplado, cargado por ETL/CDC.**

**Ejercicios 1**
1.1 🔁 ¿Qué diferencia a una carga OLTP de una OLAP, y por qué el almacenamiento columnar le gana al row-store en agregaciones?
1.2 🧠 Un dev propone "mandamos los reportes a una read replica y listo". ¿Qué problema resuelve eso y cuál **no**?
1.3 🧠 ¿Por qué el warehouse se carga con un proceso aparte (ETL/CDC) en vez de que el dashboard consulte la base de producción en vivo? Nombrá las dos razones.

---

## Módulo 2 — El data warehouse y el query-on-the-lake: Athena, Parquet y el pago por scan

**Teoría.** Hay dos formas de tener "una base columnar para analytics", y conviene distinguirlas porque la entrevista las mezcla:

- **El data warehouse clásico** (Amazon **Redshift**, Google **BigQuery**, **Snowflake**): vos **cargás** los datos en el sistema, que los guarda en **su** formato columnar optimizado y los sirve con su motor. Más rápido y con más features (índices, materialized views, concurrencia alta), pero pagás cómputo+almacenamiento del sistema y tenés que hacer el *load*.
- **El query-on-the-lake** (Amazon **Athena**): los datos **quedan en S3** (tu *data lake*, en archivos), y un motor de query SQL los lee **ahí mismo**, sin cargarlos a ningún lado. **Athena es Trino gestionado y serverless** (el engine v3 vigente se basa en **Trino**; las versiones previas, v1/v2, eran **Presto** ⚠️): no hay servidor que aprovisionar, le tirás SQL y lee los archivos de S3.

Athena es el caso que más aparece en stacks AWS por lo barato de arrancar, así que vale entender su mecánica:

- **Schema-on-read.** Athena no "tiene" los datos; tiene una **definición de tabla** (en el **AWS Glue Data Catalog**) que dice "los archivos en `s3://bucket/eventos/` son Parquet con estas columnas". El esquema se aplica **al leer**, no al escribir. Por eso podés apuntar Athena a datos que ya están en S3 sin migrarlos.
- **El formato importa muchísimo.** Si los archivos son **CSV/JSON** (texto, row-oriented), Athena lee **todo** el archivo en cada query. Si son **Parquet** (columnar + comprimido), lee **solo las columnas** que tu `SELECT` toca y se saltea el resto. Convertir el lake a Parquet es la optimización #1.
- **Pagás por byte escaneado.** El modelo de costo de Athena es **≈ US$5 por TB escaneado de S3** ⚠️ (no por tiempo de cómputo). Esto cambia *todo* el criterio de optimización: no optimizás por CPU, optimizás por **cuántos bytes lee la query**. Dos detalles que muerden y que un perfil senior nombra: hay un **mínimo de 10 MB facturados por query** ⚠️ (una query "chiquita" igual cuesta como 10 MB → muchos refrescos diminutos de un dashboard suman), y el **Glue Data Catalog** y los **requests S3 GET** se facturan **aparte**, no bajo el servicio Athena (la factura "real" de una consulta es scan + catálogo + S3). Dos palancas enormes para bajar los bytes:
  - **Particionado:** organizás el lake en carpetas por una clave de filtro frecuente (`.../año=2026/mes=06/...`). Una query con `WHERE año=2026 AND mes=6` solo lee esa carpeta — **partition pruning**. Sin particionar, un `WHERE fecha = ...` escanea **todo el histórico**.
  - **Columnar (Parquet) + `SELECT` acotado:** un `SELECT *` sobre Parquet pierde la ventaja (lee todas las columnas); pedí solo las que necesitás.

El punto senior: Athena es **excelente para consultas ad-hoc y exploración** sobre el lake, y para alimentar agregaciones. Pero **no** es la base que ponés **directamente detrás de un dashboard interactivo multi-usuario**: la latencia es de **segundos** (lee de S3, planifica, distribuye) y la **concurrencia** está acotada por cuotas de queries en paralelo. Para un panel con 200 usuarios refrescando, Athena directo es caro (cada refresh = otro scan facturado) y lento. Eso lo resuelve la capa semántica con pre-agregaciones (módulos 3-4).

> 🔥 **Falla en 4 capas — la factura de Athena se dispara.**
> (1) **Qué se rompe:** un `SELECT * FROM eventos WHERE user_id = 'x'` sobre un lake de CSV sin particionar escanea **todo** el histórico (varios TB) en cada ejecución; la cuenta de fin de mes explota.
> (2) **Por qué a esta escala:** Athena cobra por **byte escaneado**; CSV row-oriented + sin particiones = lee todo, siempre.
> (3) **Corto plazo:** límites de bytes por query (`per-query data usage control`), avisar/cortar al que escanea de más.
> (4) **Largo plazo:** **convertir el lake a Parquet** (columnar → lee solo las columnas pedidas), **particionar** por la clave de filtro frecuente (pruning), y comprimir; el mismo dato cuesta una fracción por query.

La frase mental: **dos sabores de columnar: el warehouse clásico (Redshift/BigQuery/Snowflake — cargás los datos en su motor) y el query-on-the-lake (Athena = Presto/Trino serverless que lee S3 in situ, schema-on-read vía Glue). Athena cobra por byte escaneado, así que optimizás por bytes: Parquet (columnar) + particionado (pruning) + SELECT acotado. Es genial para ad-hoc, no para un dashboard interactivo multi-usuario directo (latencia de segundos, scan por refresh).**

**Ejercicios 2**
2.1 🔁 ¿Qué diferencia hay entre un warehouse clásico (Redshift/BigQuery/Snowflake) y un motor query-on-the-lake como Athena? ¿Qué es Athena por dentro?
2.2 🧠 Athena cobra por byte escaneado. Nombrá **dos** decisiones de cómo guardás los datos en S3 que bajan drásticamente lo que paga cada query, y explicá por qué cada una funciona.
2.3 🧠 ¿Por qué Athena es buena para consultas ad-hoc pero mala como backend **directo** de un dashboard interactivo con cientos de usuarios?

---

## Módulo 3 — La capa semántica: definir la métrica una sola vez (Cube.dev)

**Teoría.** Tenés los datos en el warehouse. Ahora cinco superficies distintas necesitan la métrica **"ingreso recurrente mensual (MRR)"**: el dashboard embebido del cliente, el panel interno, un export a Excel, una alerta, un informe. Si cada una escribe **su propio SQL**, pasa lo inevitable: una incluye los reembolsos y otra no, una define "usuario activo" como login-en-30-días y otra como acción-en-7-días, y terminás con **tres números distintos para la misma métrica** en tres pantallas. Nadie confía en los datos.

La **capa semántica** (*semantic layer*) resuelve esto: es una capa que vive **entre el warehouse y los consumidores** donde definís las **métricas y dimensiones una sola vez**, de forma declarativa, y todas las superficies las consumen desde ahí. Es un **concepto**, no un producto: **Cube** (Cube.dev), el **dbt Semantic Layer** y **LookML** (de Looker) son instancias distintas de la misma idea. Acá usamos **Cube** porque es el open-source de referencia y el que pide el mercado para BI embebido — se la llama **"headless BI"**: la lógica de negocio de BI (métricas, joins, granularidad) sin la UI atada; vos ponés el front (tu dashboard, un export, lo que sea). Lo que aprendas acá transfiere a las otras.

El modelo de datos de Cube se define en **cubes** con tres piezas:

- **Measures** (medidas): los valores que se **agregan** — `count`, `sum(monto)`, `countDistinct(user_id)`. Acá definís MRR **una vez**, con su fórmula exacta. ⚠️ Ojo con un detalle que paga en el M4: no todas las measures son **aditivas**. `count`/`sum`/`min`/`max` sí (se re-suman desde un rollup); `countDistinct` y `avg` **no** (no podés sumar dos conteos de únicos sin volver a contar). Esto, que parece pedante, **decide si una pre-agregación puede servir la query** — lo abrimos en el M4.
- **Dimensions** (dimensiones): los ejes por los que **cortás/agrupás** — fecha, región, plan, estado.
- **Joins**: cómo se relacionan los cubes entre sí (pedidos ↔ usuarios), para que el consumidor pida "ingreso por país" sin saber el SQL del join.

```js
// schema/Orders.js — un cube define la métrica UNA vez
cube('Orders', {
  sql: `SELECT * FROM public.orders`,

  measures: {
    count: { type: 'count' },
    // MRR definido en UN lugar: neto de reembolsos. Punto.
    mrr: {
      type: 'sum',
      sql: `amount - refunded_amount`,
      filters: [{ sql: `${CUBE}.status = 'active'` }],
    },
  },

  dimensions: {
    status: { sql: `status`, type: 'string' },
    createdAt: { sql: `created_at`, type: 'time' },
  },

  joins: {
    Users: { relationship: 'belongsTo', sql: `${CUBE}.user_id = ${Users}.id` },
  },
});
```

El consumidor **no escribe SQL**: hace una **query declarativa** (`measures: ['Orders.mrr'], dimensions: ['Users.country'], timeDimensions: [...]`) y Cube **genera el SQL** contra el warehouse, aplica los joins y devuelve el resultado. Y lo expone por **varias APIs** sobre la misma definición: **REST**, **GraphQL**, una **SQL API** (habla el protocolo de Postgres, así que un BI tradicional o un `psql` se conectan como si fuera una base) y **MDX** (para Excel). Una sola fuente de verdad de las métricas, muchos consumidores.

> **Por qué esto es arquitectura, no una librería de charts.** La capa semántica es el punto donde **centralizás la definición del negocio**. Cuando marketing dice "cambiamos la definición de churn", lo cambiás en **un** cube y **todas** las superficies quedan consistentes al instante. Es el mismo principio DRY que aplicás al código ([SOLID](solid.md)), pero para las **métricas**: un solo lugar donde la verdad se define.

La frase mental: **sin capa semántica, cada pantalla reimplementa el SQL de cada métrica y los números no coinciden. La capa semántica (Cube = "headless BI") define measures/dimensions/joins UNA vez entre el warehouse y los consumidores; estos hacen queries declarativas (no SQL) y Cube genera el SQL. Una definición, muchas APIs (REST/GraphQL/SQL/MDX). DRY para las métricas.**

**Ejercicios 3**
3.1 🔁 ¿Qué problema concreto resuelve una capa semántica? ¿Qué son measures, dimensions y joins en Cube?
3.2 🧠 ¿Por qué se la llama "headless BI" y qué ventaja da exponer la misma definición por REST, GraphQL y una SQL API a la vez?
3.3 🧠 El equipo de finanzas redefine "ingreso neto" para excluir un tipo de transacción. Con capa semántica, ¿dónde tocás y qué pasa con las 6 pantallas que la muestran? ¿Y sin capa semántica?

---

## Módulo 4 — Pre-agregaciones y caché: que el dashboard no pegue al warehouse en cada request

**Teoría.** Problema: 200 usuarios abren el dashboard, cada uno con varios gráficos, cada gráfico es una query. Si **cada** una se traduce a SQL contra el warehouse (o peor, a un scan de Athena facturado), tenés **latencia de segundos por gráfico** y una **factura** que escala con los usuarios. El dato del warehouse es perfecto para *calcular*, pésimo para *servir interactivo*.

La solución central de Cube son las **pre-agregaciones** (*pre-aggregations*): **rollups materializados** que se calculan **por adelantado** y se guardan en un store columnar rápido (**Cube Store**), para que las queries del dashboard peguen al **rollup chico** en vez de a la tabla cruda enorme.

La idea: si tu dashboard siempre pide "MRR por mes por país", no hace falta re-escanear 50M de filas crudas cada vez. Pre-calculás **una tabla** `mrr × (mes, país)` —que tiene miles de filas, no millones— y la servís desde ahí en **milisegundos**:

```js
// dentro del cube Orders
preAggregations: {
  mrrByMonthCountry: {
    type: 'rollup',
    measures: [Orders.mrr],
    dimensions: [Users.country],
    timeDimension: Orders.createdAt,
    granularity: 'month',
    // refresca cada hora; el dato del dashboard tiene <1h de lag (aceptable en analytics)
    refreshKey: { every: '1 hour' },
  },
},
```

Cube hace **query rewriting**: cuando llega una query que **encaja** en una pre-agregación existente, la sirve desde el rollup **automáticamente**, sin tocar el warehouse. Si no encaja, cae al warehouse (*hit* vs *miss* de pre-agg). Pero "encaja" tiene **dos condiciones** que se preguntan en entrevista y que mucha gente pasa por alto:

- **Additividad de las measures.** El rollup guarda totales **ya agregados**; para servir una query a partir de él, Cube tiene que poder **re-agregar** esos totales. Eso solo funciona con measures **aditivas** (`count`, `sum`, `min`, `max`). Una measure **no aditiva** como `countDistinct` **no se puede re-sumar** (sumar los únicos de enero + los de febrero **no** da los únicos del bimestre: alguien que entró los dos meses se cuenta dos veces) → por defecto **no encaja en ningún rollup** y cae al warehouse. El truco real es declararla como `count_distinct_approx` (un **HyperLogLog**, que *sí* es aditivo a cambio de un error de ~2%) para volverla rollup-able. **La additividad es una condición de matching tan importante como la granularidad.**
- **Alineación de la granularidad temporal.** Una query encaja en una granularidad **más gruesa** solo si es un **múltiplo alineado** de la del rollup (de un rollup **diario** sacás semana, mes, trimestre, año; de uno **mensual**, trimestre y año). Cuidado: **semana y mes no se alinean** (una semana cruza el borde de mes), así que un rollup mensual **no** sirve una query semanal *ni al revés* — no es solo "más fina/más gruesa", es si una **divide exactamente** a la otra.

Esto es el **mismo trade-off de freshness vs costo/latencia** que ya viste en [Búsqueda](busqueda.md) M6 y [Streams](streams.md): el rollup es **datos derivados precomputados** (como el fan-out-on-write del feed o el índice de búsqueda), que cambian **latencia de lectura por staleness**. En analytics el staleness de minutos/una hora **casi siempre es aceptable** (nadie necesita el MRR al segundo), así que el trade-off es muy favorable. Y hay capas de caché encima: Cube cachea en memoria los resultados de queries repetidas (con un TTL ligado al `refreshKey`), igual que ponés [Redis](redis.md) delante de una base.

> 🔥 **Falla en 4 capas — el dashboard martilla el warehouse (cache stampede analítico).**
> (1) **Qué se rompe:** cada widget de cada usuario dispara una query al warehouse; con 200 usuarios concurrentes el warehouse se satura (o la factura de Athena se va por los scans repetidos) y los paneles tardan 10s+.
> (2) **Por qué a esta escala:** servís datos **interactivos** desde un motor pensado para **batch**; el costo escala con usuarios × widgets × refrescos.
> (3) **Corto plazo:** caché de resultados con TTL delante (memoria/Redis) para colapsar las queries repetidas.
> (4) **Largo plazo:** **pre-agregaciones** (rollups materializados en Cube Store) que el dashboard consume en ms; el warehouse solo se toca al **refrescar** el rollup (1×/hora), no en cada request. Cambiás un poco de freshness por orden(es) de magnitud en latencia y costo.
> **La ventana de rebuild:** mientras un rollup se reconstruye, el dashboard **sigue sirviendo la versión anterior** (Cube hace el swap recién cuando la nueva está lista) — no hay downtime ni un fallback masivo al warehouse en cada refresh. Para que ese rebuild no sea carísimo a su vez, se usan **refrescos incrementales por partición** (solo se recalcula la partición de tiempo que cambió, no todo el histórico). El lag máximo del dato es, justamente, el intervalo del `refreshKey`.

La frase mental: **servir un dashboard interactivo directo del warehouse es lento y caro (latencia de segundos + costo por query × usuarios). Las pre-agregaciones de Cube son rollups materializados precomputados (en Cube Store) que el dashboard consume en ms; Cube reescribe la query al rollup si encaja — y "encaja" exige measures aditivas (`countDistinct` no, salvo `count_distinct_approx`) + granularidad temporal alineada (semana y mes no se dividen). Es el trade-off freshness↔costo/latencia: cambiás minutos de staleness (aceptable en analytics) por leer un rollup chico en vez de millones de filas crudas. Durante el rebuild se sirve la versión vieja (sin downtime); refresco incremental por partición. Caché encima para queries repetidas.**

**Ejercicios 4**
4.1 🔁 ¿Qué es una pre-agregación y por qué hace que un dashboard sea viable sobre un warehouse?
4.2 🧠 ¿Qué trade-off introducen las pre-agregaciones, y por qué en analytics suele ser un cambio que conviene (a diferencia de un saldo bancario)? Relacionalo con el fan-out-on-write del feed.
4.3 🧠 Una query del dashboard pide "MRR por semana por país". Tenés una pre-agg de "MRR por mes por país". ¿Puede servirla desde el rollup? ¿Y si pidiera "MRR por mes por **ciudad**"? ¿Y si pidiera `countDistinct` de usuarios? Explicá las **dos** condiciones que hacen encajar (o no) una query en una pre-agg.
4.4 ✍️ Implementá en TypeScript una función `canServeFromRollup(query, rollup)` que decida si un rollup puede servir una query, chequeando las dos condiciones: (a) las measures de la query están en el rollup **y son aditivas**, (b) las dimensions de la query están en el rollup, (c) la granularidad temporal de la query es un **múltiplo alineado** de la del rollup. Modelá la additividad y la alineación con tablas explícitas (no asumas que "más gruesa" siempre sirve). Tipá todo; tiene que compilar en `--strict`. (Solución al final.)

---

## Módulo 5 — Analytics embebido multi-tenant: row-level security en serio

**Teoría.** **Analytics embebido** (*embedded BI*) es meter los dashboards **dentro de tu SaaS**: el cliente ve sus métricas sin salir de tu app, no en un Tableau aparte. La feature que lo define —y la que más se rompe con consecuencias graves— es el **aislamiento multi-tenant**: el cliente A tiene que ver **solo los datos del cliente A**, jamás los del B. Una sola query mal filtrada es una **fuga de datos entre clientes**, el peor incidente posible de una SaaS B2B.

La regla de oro, la que define todo lo demás: **el `tenant_id` jamás viene del cliente.** El navegador no es de confianza ([Seguridad de sistemas](seguridad-sistemas.md)). El flujo correcto:

1. El **backend** (que ya autenticó al usuario) firma un **JWT** con el `tenant_id` y el rol **del lado del servidor**, como un claim. Ese token va a Cube.
2. Cube expone esos claims como **security context** y los usa para **inyectar el filtro de tenant en TODAS las queries** vía `queryRewrite` — una función que recibe la query del usuario y le **agrega un filtro obligatorio** antes de generar el SQL.

```js
// cube.js — el filtro de tenant es OBLIGATORIO y server-side
module.exports = {
  // securityContext = claims del JWT firmado por tu backend (no por el cliente)
  queryRewrite: (query, { securityContext }) => {
    if (!securityContext.tenantId) {
      throw new Error('Sin tenant en el token: query denegada');
    }
    // se agrega a CUALQUIER query, el usuario no puede sacarlo
    query.filters.push({
      member: 'Orders.tenantId',
      operator: 'equals',
      values: [String(securityContext.tenantId)],
    });
    return query;
  },
};
```

Esto es **row-level security** aplicada arriba de la capa semántica: no importa qué query mande el front, **siempre** sale con `WHERE tenant_id = <el del token>`. Es el complemento del **RBAC** que ya viste ([Autenticación](autenticacion.md)): RBAC decide **qué measures/dimensions** puede tocar un rol (¿puede ver "costos internos"?), y el `queryRewrite` decide **qué filas** (las de su tenant). Dos preguntas distintas — *qué columnas/métricas* vs *qué filas* — y necesitás las dos.

Dos puntos finos de multi-tenancy que se preguntan:

- **Aislamiento del caché de pre-agregaciones.** Si dos tenants comparten la misma pre-agg sin el tenant en la clave, el rollup del A le puede responder al B. Cube te deja separar el contexto (`contextToAppId` / `COMPILE_CONTEXT`) para que cada tenant tenga su **propio data model y/o su propio caché de pre-aggs**. El filtro de fila no alcanza si el rollup ya viene mezclado.
- **El front es solo presentación.** El embed se hace pasándole al iframe/SDK un token de **sesión firmado por tu backend**; la librería de visualización (o un producto como **Embeddable**, especializado en BI embebido) **solo dibuja** lo que la capa segura devuelve. La seguridad **nunca** vive en el front.

> 🔥 **Falla en 4 capas — fuga de datos entre tenants.**
> (1) **Qué se rompe:** el `tenant_id` se toma de un parámetro del request (o se confía en un filtro que el front "promete" mandar); un cliente cambia el id y ve los datos de otro. Fuga cruzada de datos — incidente crítico.
> (2) **Por qué a esta escala:** en multi-tenant, **toda** request es potencialmente adversaria; cualquier filtro que dependa del cliente es bypasseable.
> (3) **Corto plazo:** sacar el `tenant_id` del input del cliente y leerlo **solo** del JWT verificado server-side; auditar las queries.
> (4) **Largo plazo:** filtro de tenant **obligatorio e inyectado** en la capa semántica (`queryRewrite`) que no se puede omitir, + **aislamiento del caché de pre-aggs por tenant** (`COMPILE_CONTEXT`), + RBAC para las métricas sensibles. Defensa en capas, no un `WHERE` que el front promete poner.

La frase mental: **BI embebido = dashboards multi-tenant dentro de tu SaaS. Regla de oro: el tenant_id viene del JWT firmado server-side, NUNCA del cliente. El queryRewrite inyecta el filtro de tenant en TODAS las queries (row-level security) — complementa al RBAC (qué measures/columnas) con el filtro de filas. Cuidá el aislamiento del caché de pre-aggs por tenant (COMPILE_CONTEXT). El front solo dibuja; la seguridad vive en el servidor.**

**Ejercicios 5**
5.1 🔁 En BI embebido multi-tenant, ¿de dónde tiene que venir el `tenant_id` y por qué jamás del cliente?
5.2 🧠 Distinguí qué resuelve el RBAC y qué resuelve el `queryRewrite` con el filtro de tenant. ¿Por qué necesitás los dos y no alcanza con uno?
5.3 ✍️ Implementá en TypeScript una función `secureQuery(userQuery, ctx)` que reciba una query analítica y un security context, y devuelva la query con el filtro de tenant **obligatorio** inyectado. Debe **lanzar** si no hay `tenantId` en el contexto, e **ignorar** cualquier filtro de tenant que venga en `userQuery` (no se puede confiar en el cliente). Tipá todo; tiene que compilar en `--strict`. (Solución al final.)

---

## Módulo 6 — El criterio: build vs buy y la arquitectura completa

**Teoría.** El cierre senior, igual que en los casos del banco: **¿de verdad necesitás todo este stack?** Como siempre, el criterio es no sobre-diseñar.

**Te alcanza con menos cuando:**
- **Pocos datos, pocos clientes:** un `GROUP BY` sobre una **read replica** de tu Postgres, con índices o **materialized views**, sirve dashboards perfectamente hasta volúmenes sorprendentemente altos. No montes un warehouse para 100k filas. Es el **escalón intermedio honesto** entre la read replica cruda (M1) y la artillería de warehouse + Cube: una **materialized view** *es* una pre-agregación, hecha dentro de Postgres. Pero conocé su límite antes de apoyarte en ella: en Postgres **no son incrementales de forma nativa** —`REFRESH MATERIALIZED VIEW` **recalcula todo** la vista— y el refresh **toma un lock exclusivo** (bloquea las lecturas) salvo que uses `REFRESH ... CONCURRENTLY` (que no bloquea, pero exige un índice único y es más lento). Cuando ese refresh full empieza a doler —tarda demasiado o el dato envejece mal— es la señal de pasar a un warehouse + pre-aggs incrementales por partición (M4).
- **Métricas simples y estables:** si hay tres números y no cambian, una capa semántica es burocracia. La necesitás cuando las métricas **proliferan** y **deben coincidir** entre superficies.

**Necesitás el stack completo cuando:**
- **Volumen OLAP real** (decenas de millones de filas, agregaciones que en row-store no dan) → **warehouse/lake columnar**.
- **Muchas superficies que deben dar el mismo número** → **capa semántica** (una definición, muchos consumidores).
- **Dashboard interactivo multi-tenant como parte del producto** → **pre-agregaciones** (latencia/costo) + **row-level security** (aislamiento).

**Build vs buy:** podés ensamblar (Athena/Redshift + Cube self-hosted + tu front de charts) o comprar una pieza gestionada (Cube Cloud, o un producto de embedded analytics como **Embeddable** que trae el front y parte de la seguridad). La decisión es la de siempre: cuánto del *no-diferenciador* querés operar vos. La seguridad multi-tenant **no** se terceriza sin entenderla: aunque uses un producto, **vos** sos responsable de que el tenant salga del token y no del cliente.

> **La arquitectura completa, de punta a punta** (el "happy path" del tema):
>
> ```
> Postgres OLTP ──CDC/ETL──▶ S3 (Parquet, particionado) / Warehouse columnar
>   (producción)              [Athena lee in situ · o Redshift/BigQuery/Snowflake]
>                                          │
>                                  Capa semántica (Cube)
>                          measures/dimensions UNA vez · pre-aggs · queryRewrite (RLS)
>                                          │
>                          token JWT firmado server-side (tenant_id, rol)
>                                          │
>                              Dashboard embebido en la SaaS (front / Embeddable)
> ```
>
> Cada flecha es un módulo de este temario: el **CDC/ETL** es [Streams](streams.md), el **columnar/warehouse** es [Datos a escala](datos-escala.md) M13 + Athena (M2 acá), la **capa semántica** y las **pre-aggs** son Cube (M3-4), el **JWT + RLS** es [Autenticación](autenticacion.md)/[Seguridad](seguridad-sistemas.md) (M5), y el **costo** por scan es [Costo](costo.md).

Estructurá cualquier pregunta de "diseñá el analytics de una SaaS" con **las tres cosas**: **happy path** (OLTP → pipeline → warehouse → capa semántica con pre-aggs → dashboard embebido con token firmado), **modelo de falla** (query que tumba el OLTP, scan que explota la factura, dashboard que martilla el warehouse, **fuga entre tenants**), **recuperación** (réplica/desacople, Parquet+particiones, pre-aggs+caché, RLS obligatoria + aislamiento de caché por tenant).

La frase mental de cierre: **no sobre-diseñes: read replica + materialized views aguantan mucho. El stack completo (warehouse columnar + capa semántica + pre-aggs + RLS) se justifica con volumen OLAP real, muchas superficies que deben coincidir, y un dashboard multi-tenant como producto. La arquitectura es una cadena que ya conocés módulo por módulo: OLTP → CDC/ETL → columnar → Cube (semántica + pre-aggs + queryRewrite) → embed con JWT server-side. La seguridad multi-tenant es tuya aunque compres las piezas.**

**Ejercicios 6**
6.1 🔁 Nombrá dos situaciones en las que NO necesitás warehouse + capa semántica, y con qué las resolvés.
6.2 🧠 Listá los tres disparadores que justifican el stack completo y qué pieza atiende cada uno.
6.3 🧠 Dibujá (en palabras) la arquitectura de punta a punta del analytics embebido de una SaaS multi-tenant y aplicá "las tres cosas" (happy path / modelo de falla / recuperación).

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** **OLTP** = muchas transacciones **chicas**, point read/write por clave, escrituras concurrentes, datos normalizados; se sirve mejor con un **row-store** (la fila junta, porque la traés entera). **OLAP** = pocas consultas **enormes** que **agregan** sobre millones de filas pero **pocas columnas**. El **columnar** gana en OLAP porque (a) lee **solo las columnas** que la query toca (el row-store lee filas enteras para sumar una columna), y (b) una columna tiene valores homogéneos → **comprime mucho** y se procesa **vectorizado**.

**1.2** La read replica resuelve el **aislamiento** (la query analítica no compite por IO/CPU/locks con el tráfico transaccional del primario). **No** resuelve la **eficiencia** de la query: la réplica sigue siendo un **row-store**, así que el `GROUP BY` sobre millones de filas sigue siendo lento y leyendo de más. Para eso necesitás **columnar** (warehouse/lake). Replica = aísla; warehouse = acelera.

**1.3** (1) **Aislamiento/desacople:** el plano analítico no debe competir con producción; cargando un sistema aparte, una query de 30s no afecta el checkout. (2) **Formato/optimización:** el warehouse es **columnar** y está afinado para agregaciones; la base de producción es row-store afinada para OLTP. El pipeline (ETL/CDC) además transforma/denormaliza el dato a un esquema analítico (star schema) que la query OLAP necesita.

**2.1** El **warehouse clásico** (Redshift/BigQuery/Snowflake) **ingiere** los datos y los guarda en **su** formato columnar optimizado, sirviéndolos con su motor (más rápido, más features, pagás el load y el sistema). El **query-on-the-lake** (Athena) deja los datos **en S3** y los lee **ahí mismo** sin cargarlos. **Athena es Trino gestionado y serverless** (engine v3; Presto en v1/v2): SQL sobre archivos de S3, con **schema-on-read** (la definición de tabla vive en el **Glue Data Catalog**, se aplica al leer).

**2.2** (1) **Formato columnar (Parquet) en vez de CSV/JSON:** Athena lee **solo las columnas** que el `SELECT` toca y se saltea el resto + comprime → escanea una fracción de los bytes. (2) **Particionado** (carpetas por clave de filtro, p. ej. `año=/mes=`): una query con `WHERE` sobre esa clave hace **partition pruning** y solo lee las carpetas relevantes, no todo el histórico. Ambas reducen **bytes escaneados**, que es exactamente lo que Athena factura (≈US$5/TB ⚠️). (Bonus: `SELECT` de columnas puntuales en vez de `SELECT *`.)

**2.3** Porque está pensada para **batch/ad-hoc**, no para **interactivo**: (a) **latencia de segundos** — lee de S3, planifica y distribuye la query, no hay índices calientes para un point lookup; (b) **concurrencia acotada** por cuotas de queries en paralelo; (c) **costo por scan**: cada refresh de cada widget de cada usuario es **otro scan facturado**, así que el costo escala con usuarios × widgets. Un dashboard de cientos de usuarios necesita **pre-agregaciones + caché** (M3-4) delante, no pegarle directo.

**3.1** Resuelve la **inconsistencia de métricas**: sin ella, cada superficie reimplementa el SQL de cada métrica y terminás con números distintos para "lo mismo" (uno cuenta reembolsos, otro no). En Cube: **measures** = los valores que se **agregan** (`sum`, `count`, `countDistinct`) — ahí definís la fórmula de la métrica una vez; **dimensions** = los ejes por los que **agrupás/filtrás** (fecha, país, plan); **joins** = cómo se relacionan los cubes para poder cruzar (ingreso por país sin escribir el join a mano).

**3.2** "**Headless BI**" porque provee la **lógica de BI** (métricas, joins, granularidad, seguridad) **sin la UI**: vos ponés el front. Exponer la misma definición por **REST / GraphQL / SQL API / MDX** a la vez significa que **una sola fuente de verdad** alimenta a todos los consumidores (tu dashboard por REST/GraphQL, un BI tradicional o un notebook por la SQL API que habla protocolo Postgres, Excel por MDX) — todos obtienen **el mismo número** porque sale de la misma definición, sin reimplementar el SQL en cada cliente.

**3.3** **Con** capa semántica: tocás la fórmula de la measure en **un** cube; las 6 pantallas quedan consistentes **al instante** (todas consumen esa definición). **Sin** capa semántica: tenés que encontrar y editar el SQL en **las 6** superficies, sin garantía de que queden iguales (alguna se te escapa, alguna lo interpreta distinto) → vuelven los números que no coinciden. Es DRY aplicado a las métricas: una definición, un lugar para cambiarla.

**4.1** Una **pre-agregación** es un **rollup materializado**: una tabla precomputada con la métrica ya agregada por ciertas dimensiones (p. ej. `MRR × mes × país`), guardada en un store columnar rápido (**Cube Store**). Hace viable el dashboard porque la query del panel pega a ese rollup —**miles** de filas, en **milisegundos**— en vez de re-escanear los **millones** de filas crudas del warehouse en cada request (lento y, en Athena, facturado por scan). Cube hace **query rewriting**: si la query encaja en un rollup, lo usa automáticamente.

**4.2** Introducen **staleness**: el rollup se refresca cada X (p. ej. 1h), así que el dato del dashboard está **levemente atrasado**. En analytics **conviene** porque nadie necesita el MRR exacto al segundo — minutos/una hora de lag es invisible para una decisión de negocio, y a cambio ganás órdenes de magnitud en latencia y costo. Es el **mismo trade-off del fan-out-on-write del feed** ([Datos a escala](datos-escala.md)/[Búsqueda](busqueda.md) M6): precomputás datos derivados, cambiando freshness por velocidad de lectura. En un **saldo bancario** ese trade-off **no** sirve (necesitás el número exacto ya), por eso ahí no precomputás; en analytics sí.

**4.3** Las **dos** condiciones de matching:
- "**MRR por semana por país**" sobre una pre-agg **mensual**: **no encaja**. Semana y mes **no se alinean** (una semana cruza el borde de mes), así que no podés reconstruir una semana sumando meses *ni al revés*: no es "más fina/más gruesa", es que **ninguna divide exactamente a la otra**. (De un rollup **diario** sí sacarías semana, mes, etc., porque el día divide a todas.)
- "**MRR por mes por ciudad**" sobre una pre-agg por **país**: tampoco. Ciudad es una dimensión **más granular** que país; el rollup ya **colapsó** las ciudades en países y ese detalle no se recupera.
- "**countDistinct** de usuarios" sobre cualquier rollup: tampoco, por **additividad**: sumar los únicos de dos particiones del rollup **no** da los únicos del total (los repetidos se cuentan dos veces). `countDistinct` no es aditivo → *miss*. El fix es `count_distinct_approx` (HyperLogLog, aditivo con ~2% de error).

**Regla completa:** una query encaja si (a) sus measures están en el rollup **y son aditivas**, (b) sus dimensions están en el rollup, y (c) su granularidad temporal es un **múltiplo alineado** de la del rollup. Si falla cualquiera, hay *miss* y cae al warehouse.

**4.4**
```ts
type Granularity = 'day' | 'week' | 'month' | 'quarter' | 'year';
type MeasureType =
  | 'count' | 'sum' | 'min' | 'max'
  | 'countDistinctApprox' // HyperLogLog: aditivo
  | 'countDistinct' | 'avg'; // NO aditivos

interface Measure {
  name: string;
  type: MeasureType;
}

interface Rollup {
  measures: readonly Measure[];
  dimensions: readonly string[];
  granularity: Granularity;
}

interface RollupQuery {
  measures: readonly Measure[];
  dimensions: readonly string[];
  granularity: Granularity;
}

// (a) qué measures se pueden re-agregar desde un rollup
const ADITIVAS: ReadonlySet<MeasureType> = new Set<MeasureType>([
  'count', 'sum', 'min', 'max', 'countDistinctApprox',
]);

// (c) qué granularidades de query puede servir un rollup de cada granularidad.
// Múltiplos alineados: el día divide a todo; la semana NO alinea con mes/trimestre/año.
const SERVIBLES: Record<Granularity, readonly Granularity[]> = {
  day: ['day', 'week', 'month', 'quarter', 'year'],
  week: ['week'],
  month: ['month', 'quarter', 'year'],
  quarter: ['quarter', 'year'],
  year: ['year'],
};

function canServeFromRollup(query: RollupQuery, rollup: Rollup): boolean {
  // (a) cada measure de la query está en el rollup Y es aditiva
  const enRollup = new Map<string, MeasureType>(
    rollup.measures.map((m): [string, MeasureType] => [m.name, m.type]),
  );
  for (const m of query.measures) {
    const tipo = enRollup.get(m.name);
    if (tipo === undefined) return false;   // la measure no está en el rollup
    if (!ADITIVAS.has(tipo)) return false;  // está, pero no se puede re-agregar (countDistinct, avg)
  }

  // (b) cada dimension de la query está en el rollup (no podés desagregar lo que ya colapsó)
  const dims = new Set(rollup.dimensions);
  for (const d of query.dimensions) {
    if (!dims.has(d)) return false;
  }

  // (c) la granularidad de la query es un múltiplo alineado de la del rollup
  return SERVIBLES[rollup.granularity].includes(query.granularity);
}
```
Idea: las tres condiciones del M4, hechas datos. La **additividad** (a) es una **tabla** (`ADITIVAS`), no una propiedad que se asume: `countDistinct`/`avg` quedan afuera a propósito, y por eso un rollup nunca los sirve (el fix sería declararlos `countDistinctApprox`). La **alineación temporal** (c) también es una **tabla** (`SERVIBLES`), no un orden grueso/fino: `week → ['week']` codifica que la semana no se re-agrega a mes ni el mes a semana. Compila en `--strict` (y `--noUncheckedIndexedAccess`: por eso el `tipo === undefined` tras `Map.get`; nótese que `SERVIBLES[rollup.granularity]` **no** da `undefined` porque `Record<Granularity, ...>` tiene todas las claves del union como propiedades explícitas, no un index signature). El anotado `(m): [string, MeasureType]` fuerza la tupla que espera `new Map`.

**5.1** El `tenant_id` tiene que venir de un **JWT firmado por tu backend** (server-side), como un claim, después de autenticar al usuario. **Jamás del cliente** (ni parámetro de request, ni body, ni un filtro que el front "promete" mandar) porque el navegador **no es de confianza**: cualquier valor que controle el cliente es **manipulable** → cambiás el id y ves los datos de otro tenant. La confianza vive en el token firmado, que el cliente no puede falsificar.

**5.2** El **RBAC** controla **qué measures/dimensions** (qué columnas/métricas) puede ver un **rol**: p. ej. un usuario normal no ve la measure "costo interno" o "margen". El **`queryRewrite` con filtro de tenant** controla **qué filas** ve: solo las de **su** tenant. Son preguntas **ortogonales** — *qué columnas/métricas* vs *qué filas* — y necesitás las dos: con solo RBAC, un tenant podría ver las **filas** de otro (mismas columnas permitidas, datos ajenos); con solo el filtro de tenant, un rol básico podría ver **métricas sensibles** de su propio tenant que no le corresponden. Defensa en capas.

**5.3**
```ts
type FilterOp = 'equals' | 'gt' | 'lt' | 'contains';

interface QueryFilter {
  member: string;
  operator: FilterOp;
  values: readonly string[];
}

interface AnalyticsQuery {
  measures: readonly string[];
  dimensions?: readonly string[];
  filters?: readonly QueryFilter[];
}

interface SecurityContext {
  tenantId?: string | number; // opcional: puede faltar en un token mal formado
  role?: string;
}

const TENANT_MEMBER = 'Orders.tenantId';

function secureQuery(userQuery: AnalyticsQuery, ctx: SecurityContext): AnalyticsQuery {
  // 1) sin tenant en el contexto → denegamos (no asumimos nada)
  if (ctx.tenantId === undefined || ctx.tenantId === null || ctx.tenantId === '') {
    throw new Error('Sin tenant en el security context: query denegada');
  }

  // 2) descartamos cualquier filtro de tenant que mande el cliente (no es de confianza)
  const userFilters = (userQuery.filters ?? []).filter(
    (f) => f.member !== TENANT_MEMBER,
  );

  // 3) inyectamos el filtro de tenant obligatorio, tomado SOLO del contexto server-side
  const tenantFilter: QueryFilter = {
    member: TENANT_MEMBER,
    operator: 'equals',
    values: [String(ctx.tenantId)],
  };

  return {
    ...userQuery,
    filters: [...userFilters, tenantFilter],
  };
}
```
Idea: la función **nunca confía** en el cliente. Primero **lanza** si no hay `tenantId` en el contexto (fail-closed: ante la duda, denegar). Después **filtra** cualquier filtro de tenant que el usuario haya intentado mandar (paso 2 — si no lo sacaras, el cliente podría sobrescribir el suyo), y recién **inyecta** el filtro tomado **solo** del `SecurityContext` server-side. Compila en `--strict` (y `--noUncheckedIndexedAccess`: por eso `filters ?? []` antes del `.filter`, y el chequeo explícito de `tenantId` contra `undefined`/`null`/`''`). En Cube esto es exactamente lo que hace el hook `queryRewrite`; acá lo escribimos a mano para ver la mecánica.

**6.1** (1) **Pocos datos / pocos clientes:** un `GROUP BY` sobre una **read replica** con índices o **materialized views** alcanza para dashboards hasta volúmenes altos — no montes warehouse para 100k filas. (2) **Métricas pocas y estables:** si son tres números que no cambian, la capa semántica es burocracia; la justificás cuando las métricas **proliferan** y **deben coincidir** entre muchas superficies.

**6.2** (1) **Volumen OLAP real** (decenas de millones de filas, agregaciones que el row-store no aguanta) → **warehouse/lake columnar**. (2) **Muchas superficies que deben dar el mismo número** → **capa semántica** (una definición, muchos consumidores). (3) **Dashboard interactivo multi-tenant como producto** → **pre-agregaciones** (latencia/costo bajo carga) + **row-level security** (aislamiento entre tenants).

**6.3** **Happy path:** la **OLTP** (Postgres) se replica por **CDC/ETL** a un **lake en S3 (Parquet, particionado) / warehouse columnar**; **Cube** define measures/dimensions **una vez** y mantiene **pre-agregaciones**; el backend firma un **JWT** con `tenant_id`+rol; el **dashboard embebido** consume vía REST/GraphQL y Cube sirve desde el rollup con el **filtro de tenant inyectado**. **Modelo de falla:** query que tumba el OLTP, scan de Athena que explota la factura, dashboard que martilla el warehouse, **fuga de datos entre tenants**, pre-agg que no encaja. **Recuperación:** réplica/desacople del plano analítico, **Parquet + particiones** (menos bytes), **pre-aggs + caché** (menos carga), **RLS obligatoria vía `queryRewrite` + aislamiento de caché por tenant** (`COMPILE_CONTEXT`), RBAC para métricas sensibles.

---

> **Para seguir.** Las fuentes de verdad: la doc de **Cube** (`cube.dev/docs` — data modeling, pre-aggregations, `queryRewrite`/security context, multitenancy) y la de **Amazon Athena** (formatos columnar, particiones, controles de costo por query; Athena engine v3 = Trino). Re-verificá lo marcado con ⚠️ (el precio de Athena ≈US$5/TB escaneado y las cuotas de concurrencia cambian por región y con el tiempo; además hay un modelo de **capacity reservations** además del por-scan). Los puentes en el hub: [Datos a escala](datos-escala.md) M11-14 es el origen del **columnar/OLAP** que acá aplicamos; [Streams](streams.md) es el **CDC/ETL** que carga el warehouse; [Replicación](replicacion.md) es la **read replica** del primer paso; [Redis](redis.md) es el **caché** delante de las queries repetidas; [Autenticación](autenticacion.md) y [Seguridad de sistemas](seguridad-sistemas.md) son el **JWT + RBAC + multi-tenancy** sobre los que se apoya la row-level security; [Costo](costo.md) es la **economía del pago por scan**; y [API design](api-design.md) es el contrato de las APIs que Cube expone. **Y hacia dónde va el tema:** la capa semántica se está volviendo el punto de integración para que **los LLM** consulten métricas de negocio sin alucinar SQL (text-to-query contra una definición controlada en vez de contra tablas crudas) — esa otra mitad se cruza con [RAG](rag.md) y [Agentes](agentes.md).
