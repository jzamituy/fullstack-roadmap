# NoSQL hands-on: MongoDB y DynamoDB

**Bases NoSQL con criterio, viniendo de PostgreSQL · MongoDB (documentos) + DynamoDB (key-value gestionado) · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya dominás [PostgreSQL a nivel senior](postgresql.md) (índices, `EXPLAIN`, transacciones ACID, JSONB) y que conocés [AWS](aws.md) (donde DynamoDB ya apareció conceptualmente). El ángulo es deliberado: **no venís a aprender NoSQL desde cero, venís a entender en qué se diferencia de lo relacional que ya sabés, y cuándo conviene cada uno.** El error que separa a un junior de un senior no es "saber MongoDB", es **saber cuándo NoSQL es la herramienta correcta y cuándo es relacional disfrazado de moderno**. Cubrimos dos bases que representan dos familias distintas: **MongoDB** (documentos, flexible, auto-hospedado o Atlas) y **DynamoDB** (key-value/wide-column, gestionado, serverless, AWS).

**Lo que asumimos.** TS, NestJS, PostgreSQL senior (sobre todo índices, ACID y JSONB), AWS básico, async/await. El código usa Mongoose (`@nestjs/mongoose`) para Mongo y el AWS SDK v3 para DynamoDB.

**Para practicar.**

```bash
# MongoDB local
docker run -d --name mongo -p 27017:27017 mongo
npm i @nestjs/mongoose mongoose
# DynamoDB local + SDK
docker run -d --name dynamodb -p 8000:8000 amazon/dynamodb-local
npm i @aws-sdk/client-dynamodb @aws-sdk/lib-dynamodb
```

> Nota sobre datos volátiles: límites (tamaño de ítem en DynamoDB, capacidades) y nombres de comandos del SDK cambian; verificá contra la doc oficial al implementar.

**Índice de módulos**
1. Qué es NoSQL y por qué existe
2. Las cuatro familias de NoSQL
3. MongoDB: el modelo de documentos
4. Modelar en MongoDB: embeber vs referenciar
5. Queries y el aggregation pipeline
6. Índices en MongoDB
7. DynamoDB: key-value gestionado a escala
8. DynamoDB: partition key, sort key y diseñar por access patterns
9. Consistencia: CAP, eventual vs fuerte
10. Atomicidad y transacciones sin ACID completo
11. El criterio: SQL vs NoSQL

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es NoSQL y por qué existe

**Teoría.** "NoSQL" es un nombre malo: no significa "no usar SQL", sino **"not only SQL"** — una familia de bases que abandonan parte del modelo relacional a cambio de otra cosa. Para entenderlo, primero recordá qué te da una base relacional (tu Postgres):

- **Esquema rígido**: tablas con columnas tipadas, definidas de antemano.
- **Normalización**: la data se reparte en tablas relacionadas, sin duplicar, y la unís con `JOIN`.
- **ACID fuerte**: transacciones con garantías estrictas (lo del módulo de PostgreSQL).
- **Queries ad-hoc**: con SQL preguntás cualquier cosa, aunque no la hayas previsto.

Eso es genial y es el default correcto para la mayoría de los sistemas. Pero tiene costos cuando llevás Postgres a una escala o forma de datos para la que no fue pensado: los `JOIN` entre tablas enormes se vuelven caros, escalar horizontalmente (repartir la data en muchas máquinas, *sharding*) es difícil con un esquema relacional, y a veces tu data **no encaja** en filas y columnas (documentos anidados, grafos).

NoSQL nació para esos casos. La idea central: **cada familia NoSQL resigna algo del modelo relacional para ganar otra cosa** —escala horizontal, flexibilidad de esquema, velocidad para un patrón de acceso específico—. No hay magia: hay **trade-offs**. La frase que tenés que internalizar y que es la trampa #1: **NoSQL no es "más rápido" ni "más escalable" en abstracto.** Es más rápido/escalable *para ciertos patrones de acceso*, a cambio de perder flexibilidad de consulta, joins, o consistencia fuerte. Elegir NoSQL "porque es moderno" o "porque escala" sin entender qué resignás es el error clásico.

El matiz senior que conecta con lo que ya sabés: **Postgres moderno borró buena parte de la frontera.** Con **JSONB** (módulo de PostgreSQL) Postgres guarda documentos flexibles sin abandonar ACID ni los joins. Así que la pregunta real no es "¿SQL o NoSQL?" sino "¿este caso necesita algo que mi Postgres no me da bien?" — y muchas veces la respuesta es no (módulo 11).

**Ejercicios 1**
1.1 ¿Qué significa realmente "NoSQL" y cuál es la idea central que comparten todas sus familias?
1.2 Nombrá dos cosas que da una base relacional y dos costos que aparecen al llevarla a cierta escala o forma de datos.
1.3 ¿Por qué es un error elegir NoSQL "porque es más escalable"? ¿Qué pregunta deberías hacerte en su lugar?

---

## Módulo 2 — Las cuatro familias de NoSQL

**Teoría.** "NoSQL" no es una cosa, son **cuatro familias** con modelos de datos muy distintos. Confundirlas es como decir "lenguajes que no son Java": no dice nada. Las cuatro:

- **Documentos** (MongoDB, Couchbase): guardan **documentos** tipo JSON (anidados, con estructura libre). Cada documento es auto-contenido. Bueno cuando tu data tiene forma de objeto con sub-objetos (un usuario con su perfil y sus direcciones embebidas) y la leés junta. Es la familia más parecida a lo que un dev de aplicaciones piensa naturalmente.

- **Key-value** (Redis, DynamoDB en su forma más simple): un diccionario gigante. Le das una **clave**, te devuelve un **valor**. Súper rápido y escalable, pero solo buscás por la clave. Ya lo viste en su forma más pura: **Redis** (módulo de caché) es una base key-value.

- **Wide-column / column-family** (Cassandra, DynamoDB, Bigtable): filas con columnas dinámicas, optimizadas para escribir y leer a escala masiva por una clave de partición. DynamoDB es key-value con capacidades wide-column.

- **Grafos** (Neo4j): nodos y relaciones como ciudadanos de primera clase. Brillan cuando lo importante son las **conexiones** (redes sociales, detección de fraude, recomendaciones "amigos de amigos"). Un `JOIN` recursivo en SQL que es una pesadilla, en un grafo es trivial.

La regla mental: **la familia la elige la forma de tu data y tu patrón de acceso, no la moda.** ¿Documentos anidados que leés juntos? Documental. ¿Lookups por clave a escala brutal? Key-value/wide-column. ¿Lo central son las relaciones entre entidades? Grafo. En este módulo profundizamos **documentos (MongoDB)** y **wide-column/key-value (DynamoDB)** porque son las dos que más vas a cruzar en backend y en búsquedas laborales de AI/backend engineer.

**Ejercicios 2**
2.1 Nombrá las cuatro familias de NoSQL y una base de ejemplo de cada una.
2.2 ¿Qué familia ya usaste en el temario y en qué módulo? ¿De qué familia es?
2.3 ¿Para qué tipo de problema brilla una base de grafos, y por qué en SQL sería costoso?

---

## Módulo 3 — MongoDB: el modelo de documentos

**Teoría.** MongoDB es la base documental más usada. Su unidad es el **documento**: una estructura tipo JSON (internamente BSON, JSON binario con más tipos). Los documentos viven en **colecciones** (el análogo de una tabla). El mapeo mental viniendo de Postgres:

| PostgreSQL | MongoDB |
|---|---|
| Tabla | Colección |
| Fila | Documento |
| Columna | Campo |
| `id` (PK) | `_id` (auto, un `ObjectId`) |
| `JOIN` | embeber, o `$lookup` (módulo 5) |
| Esquema rígido | esquema flexible (por defecto) |

Un documento de una tarea de tu Task API:

```js
{
  _id: ObjectId("..."),
  titulo: "Escribir el módulo de NoSQL",
  estado: "en_progreso",
  prioridad: "alta",
  etiquetas: ["docs", "backend"],          // un array, sin tabla intermedia
  asignado: { id: 7, nombre: "Jorge" },    // un sub-documento embebido
  creada: ISODate("2026-06-19")
}
```

Lo que cambia de cabeza respecto a SQL:

- **Esquema flexible**: por defecto, dos documentos de la misma colección pueden tener campos distintos. No hay `ALTER TABLE` para agregar un campo: simplemente empezás a escribirlo. Esto es potente para iterar rápido y para data heterogénea, **pero es un arma de doble filo**: sin disciplina, terminás con una colección donde nadie sabe qué forma tienen los datos. Por eso en producción se usa **validación de esquema** (Mongo la soporta) o un ODM como **Mongoose** que define y valida la forma desde el código.
- **Data anidada de primera clase**: arrays y sub-documentos son normales, no algo a "aplanar" en tablas. La tarea de arriba, en Postgres normalizado, serían 3 tablas (tareas, etiquetas, asignaciones); en Mongo es **un solo documento que leés de un saque**.

En NestJS, Mongoose te da el patrón de siempre (un schema + un model inyectable como provider), igual que un repositorio:

```ts
@Schema()
export class Tarea {
  @Prop({ required: true }) titulo: string;
  @Prop({ enum: ["pendiente", "en_progreso", "hecha"], default: "pendiente" }) estado: string;
  @Prop([String]) etiquetas: string[];
}
export const TareaSchema = SchemaFactory.createForClass(Tarea);
```

La flexibilidad no te exime de pensar el modelo: te lo *permite posponer*, lo cual es bueno para prototipar y peligroso para mantener. El modelado serio viene en el módulo siguiente.

**Ejercicios 3**
3.1 Traducí el vocabulario: ¿qué es en MongoDB una tabla, una fila, una columna y la PK?
3.2 ¿Qué significa "esquema flexible" y por qué es un arma de doble filo? ¿Cómo se le pone disciplina en producción?
3.3 La tarea de ejemplo en Postgres normalizado serían varias tablas; en Mongo es un documento. ¿Qué ganás y qué resignás con eso? (lo profundiza el módulo 4)

---

## Módulo 4 — Modelar en MongoDB: embeber vs referenciar

**Teoría.** **La decisión de modelado de MongoDB**, la que define si tu base vuela o sufre. En SQL siempre normalizás (separás en tablas y unís con `JOIN`). En Mongo tenés **dos opciones** por cada relación, y elegir mal es la causa #1 de un Mongo lento o inmanejable:

- **Embeber (embedding)**: meter la data relacionada **dentro** del documento. La tarea con su `asignado` y sus `etiquetas` adentro. Lo leés todo en una query, sin joins. **Es el default de Mongo** y donde gana de verdad frente a SQL.

- **Referenciar (referencing)**: guardar solo el **id** de la data relacionada en otra colección, y "unir" en una segunda query (o con `$lookup`, el join de Mongo). Es lo más parecido a la normalización de SQL.

La regla de decisión, que tenés que poder recitar:

| Embebé si… | Referenciá si… |
|---|---|
| La data se **lee junta** casi siempre | La data se usa/consulta **por separado** |
| La relación es **"contiene"/"pertenece a"** (1:pocos) | La relación es **muchos-a-muchos** |
| La data embebida es **acotada** y no crece sin límite | La data crece **sin límite** (ej. comentarios de un post popular) |
| Cambia poco, o cambia junto al padre | Cambia de forma **independiente** y se referencia desde muchos lados |

El criterio que lo ordena: **modelás según cómo LEÉS la data, no según cómo se relaciona "en teoría".** En SQL modelás por la estructura de las entidades; en Mongo modelás por los **patrones de acceso**. Si tu pantalla muestra "tarea con su asignado", embebé el asignado. Si el mismo usuario está asignado a 10.000 tareas y querés actualizar su nombre, embeberlo significa actualizar 10.000 documentos — ahí referenciás.

Las dos trampas clásicas:

- **Documentos sin límite de crecimiento**: embeber un array que crece para siempre (los logs de una entidad, los comentarios de un post viral) hace que el documento se infle hasta chocar con el **límite de 16 MB por documento** de Mongo y vuelve lentas todas las lecturas. Eso se referencia.
- **Duplicación y consistencia**: embeber duplica data (el nombre del asignado está en cada tarea). Si cambia, tenés que actualizar en todos lados — el problema que la normalización de SQL evitaba. Aceptás algo de duplicación a cambio de lecturas rápidas; es un trade-off consciente, no un descuido.

**Ejercicios 4**
4.1 ¿Cuáles son las dos formas de modelar una relación en Mongo y en qué se diferencian al leer?
4.2 Da la regla: ¿cuándo embebés y cuándo referenciás? (al menos dos criterios de cada lado)
4.3 ¿Por qué embeber un array que crece sin límite es un problema en Mongo? ¿Y qué trade-off aceptás al embeber data que se duplica?

---

## Módulo 5 — Queries y el aggregation pipeline

**Teoría.** Buscar en Mongo no es SQL, pero el modelo mental se traslada. Las queries básicas usan un **documento de filtro**:

```js
// "SELECT * FROM tareas WHERE estado = 'hecha' AND prioridad = 'alta'"
db.tareas.find({ estado: "hecha", prioridad: "alta" })

// Operadores: $gt, $in, $exists... "WHERE prioridad IN ('alta','media')"
db.tareas.find({ prioridad: { $in: ["alta", "media"] } })

// Proyección (qué campos traer) + orden + límite
db.tareas.find({ estado: "hecha" }, { titulo: 1 }).sort({ creada: -1 }).limit(10)
```

Para lo que en SQL harías con `GROUP BY`, `JOIN`, funciones de agregación, Mongo tiene el **aggregation pipeline**: una **secuencia de etapas** donde cada una transforma los documentos y se los pasa a la siguiente (como un pipe de Unix, o como los streams que viste en Node). Las etapas clave:

- `$match`: filtra (el `WHERE`). Ponelo **primero** para reducir el volumen temprano.
- `$group`: agrupa y agrega (el `GROUP BY` + `COUNT`/`SUM`/`AVG`).
- `$sort`, `$limit`, `$project`: ordenar, limitar, elegir campos.
- `$lookup`: **el "join" de Mongo** — trae documentos de otra colección. Funciona, pero es la señal de que quizá tu modelo debería haber embebido (módulo 4): si vivís haciendo `$lookup`, estás usando Mongo como una SQL peor.

```js
// "Cuántas tareas hay por estado" → SELECT estado, COUNT(*) GROUP BY estado
db.tareas.aggregate([
  { $match: { proyecto_id: 42 } },                       // filtrá primero
  { $group: { _id: "$estado", total: { $sum: 1 } } },    // agrupá y contá
  { $sort: { total: -1 } },
])
```

El insight: el pipeline es **composición de transformaciones**, el mismo paradigma de los streams de Node y de `.map().filter().reduce()`. Cada etapa hace una cosa y pasa el resultado. Y el principio de rendimiento es idéntico al de SQL: **filtrá lo antes posible** (`$match` temprano, idealmente sobre un campo indexado) para no arrastrar documentos de más por el pipeline — el primo de "el `WHERE` antes que el `JOIN`".

**Ejercicios 5**
5.1 Escribí (en pseudo-Mongo) el equivalente a `SELECT titulo FROM tareas WHERE estado='hecha' ORDER BY creada DESC LIMIT 5`.
5.2 ¿Qué es el aggregation pipeline y con qué dos cosas del temario (una de Node, una de SQL) lo relacionás?
5.3 ¿Qué hace `$lookup` y por qué usarlo mucho es una señal de alerta sobre tu modelo?

---

## Módulo 6 — Índices en MongoDB

**Teoría.** Buena noticia: **lo que sabés de índices en PostgreSQL se traslada casi entero.** Mongo usa **B-tree** igual que Postgres, y la lógica es la misma: sin índice, una query recorre **todos** los documentos de la colección (un *collection scan*, el equivalente del *full table scan*); con índice, va directo. La regla no cambia: indexás los campos por los que **filtrás, ordenás o unís** con frecuencia.

```js
db.tareas.createIndex({ estado: 1 })                 // índice simple (1 = ascendente)
db.tareas.createIndex({ proyecto_id: 1, estado: 1 }) // índice compuesto
```

Lo que tenés que recordar de Postgres y aplica igual:

- **El `_id` ya está indexado** automáticamente (como la PK en Postgres).
- **Índices compuestos y la regla del prefijo**: un índice en `{ proyecto_id: 1, estado: 1 }` sirve para filtrar por `proyecto_id`, o por `proyecto_id + estado`, pero **no** para filtrar solo por `estado` (igual que en Postgres: el orden de las columnas del índice importa, y se usa de izquierda a derecha).
- **`explain()` es tu `EXPLAIN`**: `db.tareas.find({...}).explain("executionStats")` te dice si usó un índice (`IXSCAN`) o si hizo un collection scan (`COLLSCAN`). Ver un `COLLSCAN` en una query frecuente es exactamente la misma alarma que ver un `Seq Scan` en Postgres.
- **El mismo trade-off**: cada índice acelera lecturas pero **ralentiza escrituras** (hay que mantenerlo) y ocupa espacio. No indexes todo "por las dudas".

Particularidades de Mongo que no tenés en Postgres estándar:

- **Índices multikey**: si indexás un campo que es un **array** (`etiquetas`), Mongo crea una entrada por cada elemento, así podés buscar eficientemente "tareas con la etiqueta 'backend'". Gratis y muy útil con el modelo de documentos.
- **Índices de texto** y geoespaciales para búsqueda full-text y por ubicación.

El principio es el mismo que en todo el temario: **medí antes de optimizar.** Corré `explain()`, mirá si pega el índice, y agregá índices según los patrones de query reales — no a ciegas. Tu intuición de Postgres acá es casi 100% transferible.

**Ejercicios 6**
6.1 ¿Qué pasa al correr una query sin índice en Mongo, y cómo se llama el equivalente de `EXPLAIN`? ¿Qué señal de alarma buscás en su salida?
6.2 Tenés un índice `{ proyecto_id: 1, estado: 1 }`. ¿Sirve para filtrar solo por `estado`? ¿Por qué? (la regla del prefijo)
6.3 ¿Qué es un índice multikey y por qué encaja tan bien con el modelo de documentos?

---

## Módulo 7 — DynamoDB: key-value gestionado a escala

**Teoría.** Cambiamos de familia. **DynamoDB** es la base NoSQL gestionada de AWS (ya apareció en `aws.md`): key-value con capacidades wide-column, **serverless** (no administrás servidores), y diseñada para **escala masiva con latencia de un dígito de milisegundos, predecible**. Es una bestia distinta de Mongo, con una filosofía casi opuesta a la de Postgres.

Lo que la define:

- **Totalmente gestionada y serverless**: no hay instancia que mantener, parchar ni escalar a mano. AWS reparte tu data en particiones automáticamente. Pagás por capacidad (modo **on-demand**: pagás por request, sin planificar; o **provisioned**: reservás capacidad de lectura/escritura — RCU/WCU). Esto encaja con lo que viste de serverless y FinOps en AWS.

- **Performance predecible a cualquier escala**: una tabla con 10 ítems y una con 10.000 millones responden **igual de rápido** si la diseñaste bien. Eso es lo que Postgres no te garantiza (sus queries se degradan con el volumen si no cuidás índices y joins).

- **El precio de esa garantía — y esto es lo central**: DynamoDB **te obliga a diseñar alrededor de tus patrones de acceso, de antemano.** No hay queries ad-hoc, no hay `JOIN`, no hay "después se me ocurre buscar por este campo". Si no lo previste en el diseño de las claves, la query no existe (o es carísima). Es **lo opuesto a la flexibilidad de SQL**: Postgres te deja preguntar cualquier cosa después; Dynamo te exige saber qué vas a preguntar antes.

La frase mental: **con Postgres modelás la data y después consultás; con DynamoDB modelás las consultas y después guardás la data.** Por eso Dynamo brilla en sistemas con patrones de acceso **conocidos y de alto volumen** (el carrito de un e-commerce, sesiones, un feed) y es una pésima elección para analítica exploratoria o reportes ad-hoc (ahí querés SQL). El módulo siguiente es cómo se hace ese diseño-por-acceso.

**Ejercicios 7**
7.1 Nombrá tres cosas que definen a DynamoDB (gestión, performance, modelo de consulta).
7.2 ¿Qué garantía da Dynamo que Postgres no da, y cuál es el precio de esa garantía?
7.3 Completá y explicá: "con Postgres modelás la data y después consultás; con DynamoDB modelás ___ y después ___". ¿Para qué tipo de sistema es mala elección?

---

## Módulo 8 — DynamoDB: partition key, sort key y diseñar por access patterns

**Teoría.** El corazón de Dynamo es su **clave primaria**, que determina cómo se reparte y se accede la data. Dos formas:

- **Partition key sola** (clave hash): un identificador único. Dynamo aplica un hash y eso decide en qué **partición** físico vive el ítem. Buscás por esa clave → O(1), instantáneo. Es key-value puro.
- **Partition key + sort key** (clave compuesta): la partition key agrupa ítems relacionados en la misma partición, y la sort key los ordena/distingue **dentro** de esa partición. Esto habilita el patrón clave: **"traeme todos los ítems de esta partición, ordenados/filtrados por la sort key"** en una sola operación eficiente.

El ejemplo canónico, tu Task API:

```
PK = "PROYECTO#42"   SK = "TAREA#001"   { titulo: "...", estado: "..." }
PK = "PROYECTO#42"   SK = "TAREA#002"   { ... }
PK = "PROYECTO#42"   SK = "META#info"   { nombre: "Mi proyecto", owner: 7 }
```

Con una sola `Query` de `PK = "PROYECTO#42"` traés el proyecto y todas sus tareas juntas, ordenadas por SK. Esto es el famoso **single-table design**: en vez de una tabla por entidad (como en SQL), metés **varios tipos de entidad en una sola tabla**, diseñando las claves para que cada patrón de acceso sea una query eficiente. Suena raro viniendo de SQL —y lo es— pero es lo que hace que Dynamo rinda.

Las operaciones, y la distinción que tenés que clavar:

- **`Query`**: lee ítems por partition key (y opcionalmente filtra por sort key). **Eficiente** — usa las claves. Es lo que querés hacer siempre.
- **`Scan`**: recorre **toda la tabla**. **Carísimo y lento** — es el `SELECT *` sin `WHERE`, el `COLLSCAN`/`Seq Scan` de Dynamo. En producción se evita casi siempre; verlo es señal de que tu diseño de claves no contempla ese acceso.

¿Y si necesitás buscar por un campo que no es la clave (ej. "todas las tareas asignadas a Jorge")? Para eso están los **índices secundarios**: un **GSI (Global Secondary Index)** te da una partition/sort key **alternativa** sobre los mismos datos, habilitando otro patrón de acceso. Cada patrón de acceso que tu app necesita = una clave o un GSI que lo soporte. Por eso el diseño **empieza listando los access patterns** ("buscar tareas por proyecto", "buscar tareas por asignado", "buscar el proyecto por id") y *después* se diseñan las claves y GSIs para cubrirlos. Es ingeniería al revés de SQL: las consultas primero, el esquema después.

**Ejercicios 8**
8.1 ¿Qué hacen la partition key y la sort key, y qué patrón habilita tenerlas juntas?
8.2 ¿Qué es el single-table design y en qué se opone al modelado relacional?
8.3 ¿Cuál es la diferencia entre `Query` y `Scan`, y cuál evitás en producción? ¿Para qué sirve un GSI?

---

## Módulo 9 — Consistencia: CAP, eventual vs fuerte

**Teoría.** El concepto que explica *por qué* las bases NoSQL distribuidas se comportan distinto a tu Postgres de una sola máquina. El **teorema CAP** dice que un sistema distribuido, ante una **partición de red** (P, cuando los nodos no se pueden comunicar — y en un sistema distribuido las particiones *van a pasar*), tiene que elegir entre:

- **Consistencia (C)**: toda lectura ve la última escritura (o falla). Todos los nodos coinciden.
- **Disponibilidad (A)**: toda request recibe respuesta (aunque sea data vieja).

No podés tener las dos durante una partición: o respondés con data posiblemente desactualizada (elegís A), o rechazás la request para no dar data inconsistente (elegís C). Muchas bases NoSQL eligen **AP** (disponibilidad sobre consistencia) y ofrecen **consistencia eventual**.

**Consistencia eventual** (eventual consistency): después de una escritura, las lecturas pueden devolver el valor **viejo** por un ratito, hasta que el cambio se propaga a todas las réplicas. "Eventualmente" todos convergen al valor nuevo, pero no instantáneamente. Ejemplo concreto: escribís un ítem en Dynamo y lo leés 10 ms después desde otra réplica — podés recibir la versión anterior. Para un contador de "me gusta" o un feed, da igual (nadie nota 50 ms de retraso). Para el **saldo de una cuenta bancaria**, es inaceptable.

La contracara es la **consistencia fuerte** (strong consistency): toda lectura refleja la última escritura, siempre. Es lo que tu Postgres te da por defecto (ACID, módulo de PostgreSQL). DynamoDB **te deja elegir por lectura**: lecturas eventualmente consistentes (más baratas y rápidas, el default) o fuertemente consistentes (más caras, pero ves siempre lo último). Mongo, según la config de réplicas y *read/write concern*, te da un espectro parecido.

El criterio senior: **la consistencia es una decisión de negocio, no técnica.** No preguntes "¿quiero consistencia fuerte?" (todos dirían que sí); preguntá "¿qué pasa si un usuario ve data desactualizada por 100 ms en *este* dato?". Para un like, nada → eventual (más rápido y barato). Para un saldo o un stock que no podés sobrevender → fuerte. Saber que **podés elegir, y que la elección tiene costo**, es lo que distingue al que entiende sistemas distribuidos del que pide "máxima consistencia" en todo y paga de más (o ni eso, porque a veces ni está disponible).

**Ejercicios 9**
9.1 Enunciá el teorema CAP: ¿entre qué dos cosas hay que elegir, y cuándo?
9.2 ¿Qué es la consistencia eventual? Da un caso donde es aceptable y uno donde no.
9.3 ¿Por qué la elección de consistencia es "de negocio, no técnica"? ¿Qué pregunta hacés para decidir?

---

## Módulo 10 — Atomicidad y transacciones sin ACID completo

**Teoría.** En Postgres diste por sentado las **transacciones ACID**: envolvés varias operaciones en un `BEGIN ... COMMIT` y o pasan todas o ninguna, con aislamiento. En NoSQL eso **no se da por sentado** —fue parte de lo que muchas bases resignaron para escalar— y entender qué garantías tenés es clave para no corromper data.

**MongoDB:**
- **Operaciones sobre un solo documento son atómicas**, siempre. Si actualizás varios campos de un documento (o un sub-documento embebido), o se aplican todos o ninguno. **Esto es un argumento fuerte a favor de embeber** (módulo 4): si la data que cambia junta vive en un solo documento, tu actualización es atómica gratis, sin transacciones.
- **Transacciones multi-documento existen** (desde la versión 4.0) y dan garantías ACID a través de varios documentos/colecciones, pero tienen **costo de performance** y van contra la filosofía documental. La señal: si necesitás transacciones multi-documento seguido, probablemente deberías haber embebido esa data (o quizá tu caso quería un relacional).

**DynamoDB:**
- Operaciones sobre un solo ítem son atómicas, y tiene **escrituras condicionales** (`condition expressions`): "escribí esto **solo si** se cumple X" — atómico, perfecto para evitar sobrescrituras y para el patrón optimista (no actualices si la versión cambió). Es la herramienta clave para concurrencia sin locks.
- Tiene `TransactWriteItems` para transacciones de hasta unos pocos ítems, con límites y costo extra (consume el doble de capacidad). Existe, pero es para casos puntuales, no el modo de operar.

El patrón que reemplaza muchas transacciones en NoSQL: **escrituras condicionales / optimistic concurrency.** En vez de bloquear (lock pesimista, lo de Postgres), guardás un número de versión y escribís "solo si la versión sigue siendo N"; si otro escribió primero, tu condición falla y reintentás. Es la misma idea del *optimistic locking* que puede aparecer en SQL, pero acá es **el** mecanismo, no una opción.

El criterio: **diseñá para minimizar la necesidad de transacciones multi-documento/multi-ítem.** En SQL las transacciones son baratas y las usás libremente; en NoSQL son caras o limitadas, así que el buen modelado (embeber lo que cambia junto, claves bien diseñadas) hace que la atomicidad de un solo documento/ítem te alcance para casi todo. Si te encontrás peleando con transacciones distribuidas en NoSQL, muchas veces la respuesta correcta es **reconsiderar el modelo** —o reconocer que el caso pedía un relacional**.

**Ejercicios 10**
10.1 ¿Qué garantía de atomicidad te da Mongo siempre, y cómo se relaciona eso con la decisión de embeber?
10.2 ¿Qué es una escritura condicional en Dynamo y para qué patrón de concurrencia sirve? ¿En qué se diferencia del lock pesimista de Postgres?
10.3 Si te encontrás necesitando transacciones multi-documento en Mongo todo el tiempo, ¿qué te está diciendo eso sobre tu diseño?

---

## Módulo 11 — El criterio: SQL vs NoSQL

**Teoría.** El módulo que se evalúa en una entrevista y el que más importa. Después de ver dos NoSQL, el riesgo es el péndulo opuesto al de un junior: en vez de "uso Postgres para todo", caer en "uso Mongo/Dynamo porque escala". Ambos extremos son falta de criterio. La verdad senior: **relacional (Postgres) es el default correcto para la enorme mayoría de los sistemas**, y NoSQL es la herramienta para casos específicos donde lo relacional sufre.

Cuándo **quedarte en relacional** (la opción por defecto):

- Tu data tiene **relaciones complejas** y hacés queries variadas que las cruzan (joins).
- Necesitás **transacciones ACID** sobre varias entidades de forma rutinaria (lo financiero, lo transaccional).
- Querés **consultas ad-hoc / analítica**: preguntas que no previste de antemano.
- Tu escala es "normal" (millones de filas, no miles de millones) — Postgres aguanta muchísimo más de lo que la gente cree.
- **Tu caso parece documental pero entra en JSONB**: Postgres te da documentos flexibles *sin* abandonar ACID, joins ni SQL (módulo de PostgreSQL). Esto descarta una *enorme* cantidad de "necesito Mongo".

Cuándo **NoSQL gana de verdad**:

- **MongoDB**: data con forma de documento que leés junta, esquema que evoluciona rápido, y no necesitás joins complejos ni ACID multi-entidad. Catálogos, perfiles, CMS, eventos.
- **DynamoDB**: patrones de acceso **conocidos y fijos**, **escala masiva** con latencia predecible, y querés cero administración (serverless). Carritos, sesiones, IoT, feeds de alto volumen.
- **Key-value (Redis)**: caché, sesiones, contadores, rate limiting (módulo de Redis).
- **Grafo**: cuando lo central son las relaciones (redes, fraude, recomendaciones).

La tabla de criterio:

| Si necesitás… | Elegí |
|---|---|
| Joins, ACID multi-entidad, queries ad-hoc, escala normal | **PostgreSQL** |
| Documentos flexibles que leés juntos, esquema cambiante | **MongoDB** (o Postgres+JSONB) |
| Acceso fijo conocido + escala masiva + serverless | **DynamoDB** |
| Caché / contadores / sesiones efímeras | **Redis** |
| El problema *es* las relaciones entre entidades | **Grafo** |

Y dos verdades senior que cierran el módulo y el criterio del temario:

- **Polyglot persistence**: los sistemas reales **no eligen una sola base**. Usás Postgres para tu data transaccional, Redis para caché, Dynamo para el feed de alto volumen, pgvector para RAG (módulo de RAG). La pregunta no es "¿qué base uso?" sino "¿qué base para *cada* parte del sistema?".
- **El criterio integrador de todo el temario**, una vez más: **la solución más simple que resuelve el problema real.** Para bases eso significa: empezá con Postgres (cubre casi todo, incluido JSONB para documentos), y adoptá NoSQL solo cuando tengas un problema concreto que lo justifique —un patrón de acceso a escala, una forma de data, una necesidad de serverless— no porque suene moderno. Saber *justificar* la elección con un patrón de acceso real, en vez de con una moda, es exactamente lo que demuestra criterio de Tech Lead.

**Ejercicios 11**
11.1 Nombrá tres situaciones donde te quedás en PostgreSQL en vez de saltar a NoSQL.
11.2 Para cada caso, elegí la base y justificá: (a) el carrito de un e-commerce con millones de usuarios concurrentes y acceso por user_id; (b) un sistema bancario con transferencias entre cuentas; (c) un CMS con artículos de estructura variable que se leen enteros.
11.3 ¿Qué es "polyglot persistence" y cómo se relaciona con el criterio integrador del temario ("la solución más simple que resuelve el problema real")?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Significa "not only SQL": una familia de bases que abandonan parte del modelo relacional a
    cambio de otra cosa. La idea central que comparten: cada una resigna algo (joins, esquema
    rígido, consistencia fuerte, queries ad-hoc) para ganar otra cosa (escala horizontal,
    flexibilidad, velocidad para un patrón). Son trade-offs, no magia.
1.2 Da (dos de): esquema rígido tipado, normalización sin duplicar, ACID fuerte, queries
    ad-hoc con SQL. Costos a escala (dos de): joins entre tablas enormes se vuelven caros,
    escalar horizontalmente (sharding) es difícil, data que no encaja en filas/columnas.
1.3 Porque NoSQL no es más escalable "en abstracto": lo es para ciertos patrones de acceso, a
    cambio de resignar joins/flexibilidad/consistencia. La pregunta correcta: "¿este caso
    necesita algo que mi Postgres no me da bien?" (y con JSONB, muchas veces la respuesta es no).
```

### Módulo 2
```
2.1 Documentos (MongoDB), key-value (Redis/DynamoDB), wide-column (Cassandra/DynamoDB), grafos
    (Neo4j).
2.2 Redis, en el módulo de caché y colas. Es de la familia key-value.
2.3 Para problemas donde lo central son las relaciones/conexiones entre entidades (redes
    sociales, fraude, recomendaciones "amigos de amigos"). En SQL serían joins recursivos
    costosos y complejos; en un grafo, recorrer relaciones es la operación nativa.
```

### Módulo 3
```
3.1 Tabla → colección; fila → documento; columna → campo; PK (id) → _id (un ObjectId
    autogenerado).
3.2 Que dos documentos de la misma colección pueden tener campos distintos sin un ALTER: agregás
    un campo escribiéndolo. Doble filo: bueno para iterar rápido, pero sin disciplina nadie sabe
    qué forma tienen los datos. Disciplina en producción: validación de esquema de Mongo o un ODM
    como Mongoose que define/valida la forma desde el código.
3.3 Ganás: lo leés todo en una sola query, sin joins (la tarea con su asignado y etiquetas de un
    saque). Resignás: duplicación de data y el riesgo de inflar el documento si la data anidada
    crece sin límite — el modelado del módulo 4.
```

### Módulo 4
```
4.1 Embeber (meter la data relacionada dentro del documento, se lee todo junto sin joins) y
    referenciar (guardar solo el id y unir en otra query o con $lookup, como la normalización
    de SQL). Al leer: embeber = una query; referenciar = dos (o un lookup).
4.2 Embebé si: se lee junta casi siempre, la relación es "contiene"/1:pocos, y la data es
    acotada. Referenciá si: se usa por separado, la relación es muchos-a-muchos, la data crece
    sin límite, o cambia de forma independiente y se referencia desde muchos lados.
4.3 Porque un array que crece para siempre infla el documento hasta el límite de 16 MB y vuelve
    lentas todas las lecturas → eso se referencia. Al embeber data duplicada aceptás el trade-off
    de tener que actualizarla en todos lados si cambia, a cambio de lecturas rápidas (sin joins).
```

### Módulo 5
```
5.1 db.tareas.find({ estado: "hecha" }, { titulo: 1 }).sort({ creada: -1 }).limit(5)
5.2 Es una secuencia de etapas donde cada una transforma los documentos y se los pasa a la
    siguiente. Se relaciona con los streams / .map().filter().reduce() de Node (composición de
    transformaciones) y con GROUP BY/JOIN/agregaciones de SQL.
5.3 $lookup trae documentos de otra colección (el "join" de Mongo). Usarlo mucho es alerta
    porque significa que estás uniendo colecciones constantemente: probablemente esa data debió
    embeberse, o tu caso era más relacional que documental (estás usando Mongo como una SQL peor).
```

### Módulo 6
```
6.1 Sin índice hace un collection scan (recorre todos los documentos), el equivalente del full
    table scan. El equivalente de EXPLAIN es .explain("executionStats"); la señal de alarma es
    ver un COLLSCAN (en vez de IXSCAN) en una query frecuente.
6.2 No. La regla del prefijo: un índice compuesto se usa de izquierda a derecha, así que
    { proyecto_id: 1, estado: 1 } sirve para filtrar por proyecto_id o por proyecto_id+estado,
    pero no para filtrar solo por estado (igual que en Postgres).
6.3 Un índice multikey indexa cada elemento de un campo array (una entrada por elemento), así
    podés buscar eficientemente "documentos con la etiqueta X". Encaja con el modelo de
    documentos porque los arrays son ciudadanos de primera clase ahí.
```

### Módulo 7
```
7.1 (1) Totalmente gestionada y serverless (no administrás servidores; pagás por capacidad
    on-demand o provisioned). (2) Performance predecible (latencia de pocos ms a cualquier
    escala). (3) Modelo de consulta restringido: sin queries ad-hoc, sin JOIN — diseñás por
    patrones de acceso de antemano.
7.2 Garantía: latencia predecible de un dígito de ms a cualquier escala (10 ítems o 10.000
    millones responden igual). Precio: te obliga a diseñar alrededor de tus patrones de acceso
    de antemano — si no lo previste, la query no existe o es carísima.
7.3 "modelás las consultas y después guardás la data". Es mala elección para analítica
    exploratoria o reportes ad-hoc (donde querés la flexibilidad de SQL para preguntar cosas no
    previstas).
```

### Módulo 8
```
8.1 La partition key (hash) decide en qué partición vive el ítem y permite lookup O(1); la sort
    key ordena/distingue ítems dentro de una misma partición. Juntas habilitan "traeme todos los
    ítems de esta partición, ordenados/filtrados por la sort key" en una operación eficiente.
8.2 Meter varios tipos de entidad en una sola tabla, diseñando las claves para que cada patrón
    de acceso sea una query eficiente. Se opone al modelado relacional (una tabla por entidad,
    normalizada): acá desnormalizás y agrupás por acceso, no por entidad.
8.3 Query lee por partition key (eficiente, usa las claves) — lo que querés siempre. Scan recorre
    toda la tabla (carísimo, el SELECT * sin WHERE) — lo evitás en producción. Un GSI da una
    partition/sort key alternativa sobre los mismos datos, para habilitar otro patrón de acceso
    (ej. buscar por asignado además de por proyecto).
```

### Módulo 9
```
9.1 Ante una partición de red (P), un sistema distribuido debe elegir entre Consistencia (toda
    lectura ve la última escritura, o falla) y Disponibilidad (toda request responde, aunque sea
    con data vieja). No podés tener ambas durante la partición.
9.2 Después de una escritura, las lecturas pueden devolver el valor viejo un rato hasta que se
    propaga a todas las réplicas (eventualmente convergen). Aceptable: un contador de "me gusta"
    o un feed (nadie nota 50 ms). Inaceptable: el saldo de una cuenta bancaria o un stock que no
    podés sobrevender.
9.3 Porque "¿quiero consistencia fuerte?" todos dirían que sí, pero tiene costo (más cara, más
    lenta, a veces menos disponible). La pregunta correcta es de negocio: "¿qué pasa si un
    usuario ve este dato desactualizado por 100 ms?" — y según el dato, elegís eventual o fuerte.
```

### Módulo 10
```
10.1 Mongo garantiza atomicidad sobre un solo documento siempre (incluidos sus sub-documentos
     embebidos). Se relaciona con embeber: si la data que cambia junta vive en un solo documento,
     tu actualización es atómica gratis, sin necesitar transacciones multi-documento.
10.2 "Escribí esto solo si se cumple X" (ej. solo si la versión sigue siendo N): es atómica y
     sirve para concurrencia optimista (no actualices si otro escribió primero; si falla,
     reintentás). Se diferencia del lock pesimista de Postgres en que no bloquea: asume que no
     habrá conflicto y solo falla/reintenta si lo hubo.
10.3 Que tu modelo probablemente está mal: la data que cambia junta debería estar embebida en un
     mismo documento (para tener atomicidad gratis), o tu caso en realidad pedía una base
     relacional. Pelear con transacciones multi-documento en Mongo es señal de modelo equivocado.
```

### Módulo 11
```
11.1 (Tres de:) relaciones complejas con queries variadas que las cruzan (joins); necesidad
     rutinaria de ACID sobre varias entidades; consultas ad-hoc/analítica no previstas; escala
     normal (millones, no miles de millones); caso "documental" que entra en JSONB sin perder
     ACID/joins/SQL.
11.2 (a) DynamoDB: patrón de acceso fijo (por user_id), escala masiva concurrente, latencia
     predecible, serverless. (b) PostgreSQL: transferencias = ACID multi-entidad y consistencia
     fuerte, el caso de manual de relacional. (c) MongoDB (o Postgres+JSONB): documentos de
     estructura variable que se leen enteros, sin joins ni ACID multi-entidad.
11.3 Polyglot persistence es usar varias bases en un mismo sistema, una por cada necesidad
     (Postgres transaccional + Redis caché + Dynamo feed + pgvector RAG). Se relaciona con el
     criterio integrador: no buscás "la" base, buscás la herramienta más simple y adecuada para
     CADA parte del problema real — empezando por Postgres y sumando NoSQL solo cuando se
     justifica.
```

---

## Siguientes pasos

Con este módulo tenés NoSQL con criterio: las cuatro familias, MongoDB hands-on (documentos, embeber vs referenciar, aggregation pipeline, índices), DynamoDB hands-on (serverless, partition/sort key, single-table design, query vs scan, GSI), los conceptos transversales (CAP y consistencia eventual vs fuerte, atomicidad sin ACID completo, escrituras condicionales) y —lo más importante— **el criterio de cuándo SQL y cuándo cada NoSQL**, con el default puesto en Postgres y la honestidad de que JSONB y polyglot persistence resuelven más de lo que parece. Este es el primer módulo del **track Tech Lead**. Lo que sigue en ese track, en orden sugerido: **Kubernetes** (fundamentos de orquestación de contenedores, sobre lo que ya viste de Docker), **observabilidad práctica** (OpenTelemetry y tracing distribuido — que engancha directo con el tracing del módulo 8 de Evals y el de microservicios), **TDD como disciplina** (red-green-refactor, profundizando el módulo de Testing) y **liderazgo técnico** (ADRs, métricas DORA, Team Topologies). Juntos completan la otra mitad del perfil senior contratable, la que ya no es solo "saber construir" sino "saber decidir y conducir".
