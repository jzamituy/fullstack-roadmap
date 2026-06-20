# RAG: que un LLM responda con tu propia data

**Retrieval-Augmented Generation con criterio de producción · pgvector + Claude · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **segundo módulo del track de IA**: asume que ya hiciste el de [IA Generativa y LLMs](ia-llms.md) (Messages API, tokens, prompting, structured output, embeddings) y que conocés [PostgreSQL](postgresql.md) (índices, `EXPLAIN`, transacciones). RAG es la técnica que cierra el hueco más importante de un LLM para el backend: **un modelo no conoce tu data privada** (tus apuntes, los documentos de tu empresa, la base de tickets) ni nada posterior a su corte de entrenamiento. RAG resuelve eso **recuperando** los fragmentos relevantes de tu data y **metiéndolos en el prompt** para que el modelo responda basándose en ellos. El stack es **pgvector sobre tu Postgres** (no una base vectorial aparte) y **Claude / Anthropic** para la generación.

**Lo que asumimos.** TS, NestJS (DI, providers, el patrón Repository), Postgres + un ORM, async/await, y el módulo de LLMs anterior (sobre todo embeddings y prompting). El código usa `@anthropic-ai/sdk`, el cliente de Voyage AI para embeddings, y `pgvector`.

**Para practicar.** Postgres con la extensión `vector`, una API key de Anthropic y otra de Voyage AI:

```bash
npm i @anthropic-ai/sdk voyageai
# en Postgres, una sola vez:
#   CREATE EXTENSION IF NOT EXISTS vector;
export ANTHROPIC_API_KEY="sk-ant-..."
export VOYAGE_API_KEY="pa-..."     # Anthropic NO tiene embeddings propios (módulo 4)
```

> Nota sobre datos volátiles: IDs/precios de modelos y dimensiones de embeddings son los vigentes a mediados de 2026; verificá contra `docs.claude.com` y `docs.voyageai.com` al implementar.

**Índice de módulos**
1. Qué es RAG y qué problema resuelve
2. La arquitectura: dos pipelines (ingest y query)
3. Chunking: el error número uno
4. Embeddings y la vector database (pgvector)
5. Retrieval: búsqueda vectorial sobre pgvector
6. Retrieval híbrido (vectorial + BM25) y reranking
7. Contextual Retrieval: la técnica de Anthropic
8. La generación: armar el prompt con el contexto
9. Permisos y multi-tenant: el RAG seguro
10. El criterio: RAG vs fine-tuning vs long-context
11. Evaluar un sistema RAG (recall@k, faithfulness)

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es RAG y qué problema resuelve

**Teoría.** Un LLM tiene dos límites duros que ya viste: **no conoce tu data privada** y **no sabe nada posterior a su corte de entrenamiento**. Si le preguntás "¿qué dice nuestro contrato con el cliente X?", no tiene forma de saberlo. Tenés tres salidas:

1. **Meter todo en el prompt** (long-context): pegás el documento entero en cada llamada. Funciona para un documento, no para 10.000 (módulo 10).
2. **Fine-tuning**: reentrenar el modelo con tu data. Caro, lento, y —clave— **no es la herramienta para enseñar hechos** (módulo 10).
3. **RAG** (Retrieval-Augmented Generation): en cada pregunta, **buscás** los fragmentos relevantes de tu data y **se los pasás al modelo en el prompt** para que responda basándose en ellos.

RAG es, para el backend, lo más parecido a lo que ya sabés: es **una búsqueda seguida de una llamada al LLM**. El modelo no "aprende" tu data; la **lee** en el momento, como vos leés un archivo antes de responder un mail. La frase mental: **RAG le da al modelo un examen a libro abierto** — no le pedís que recuerde, le pasás la página correcta.

Las ventajas que lo hacen el default en producción:

- **Data fresca**: actualizás la base de conocimiento y el sistema responde con lo nuevo, sin reentrenar nada.
- **Citabilidad**: como sabés qué fragmentos recuperaste, podés mostrar la fuente ("según el contrato, cláusula 4"). Esto reduce alucinaciones y da trazabilidad.
- **Control de acceso**: filtrás qué fragmentos puede ver cada usuario (módulo 9) — imposible con fine-tuning, donde la data queda fundida en los pesos.

El hilo conductor del track: el sistema que venimos construyendo crece otra vez. De **una llamada simple** (módulo de LLMs) a un **extractor estructurado**, ahora a un **RAG sobre tus propios apuntes** (este módulo de estudio podría ser la base de conocimiento), después a un **agente**, todo cubierto por **evals**.

**Ejercicios 1**
1.1 En una frase, ¿qué hace RAG y a qué situación de examen se parece?
1.2 ¿Por qué un LLM no puede responder sobre el contrato privado de tu empresa, y cuáles son las tres salidas posibles?
1.3 Nombrá dos ventajas de RAG sobre fine-tuning para responder con data privada.

---

## Módulo 2 — La arquitectura: dos pipelines (ingest y query)

**Teoría.** Todo sistema RAG son **dos pipelines separados** que la gente confunde al principio. Tenerlos claros es la mitad del diseño.

**Pipeline de ingest (offline, una vez por documento):**

```
documento → chunking → embeddings → guardar (texto + vector) en la base
```

Procesás tus documentos: los partís en fragmentos (**chunks**), convertís cada chunk en un vector (**embedding**), y guardás texto + vector en la base. Esto corre **fuera del request del usuario** — es trabajo batch, ideal para tus colas (módulo de Redis/BullMQ): subís un documento → encolás un job de ingest.

**Pipeline de query (online, en cada pregunta):**

```
pregunta → embedding de la pregunta → buscar los k chunks más cercanos
         → (opcional: rerank) → armar prompt con esos chunks → LLM → respuesta
```

Cuando el usuario pregunta, embeddéas **la pregunta** con el mismo modelo del ingest, buscás los chunks más parecidos, los metés en el prompt y llamás al LLM.

La asimetría importante: **el ingest es caro pero infrecuente; el query es barato pero por cada request.** Por eso optimizás distinto cada uno (el ingest tolera latencia y conviene cachear/batchear embeddings; el query tiene presupuesto de latencia ajustado).

El principio que ordena todo: **mismo modelo de embeddings en ambos pipelines.** La pregunta y los chunks tienen que vivir en el mismo "espacio vectorial" para que la cercanía signifique algo. Si embeddeás los documentos con un modelo y la pregunta con otro, la búsqueda devuelve basura. (Corolario: si cambiás de modelo de embeddings, tenés que **reindexar todo**.)

**Ejercicios 2**
2.1 Nombrá los pasos de cada pipeline (ingest y query) en orden.
2.2 ¿Por qué el ingest conviene correrlo en una cola y no en el request? Conectá con un módulo anterior.
2.3 ¿Qué pasa si embeddeás los documentos con un modelo y la pregunta con otro distinto? ¿Qué implica al cambiar de modelo de embeddings?

---

## Módulo 3 — Chunking: el error número uno

**Teoría.** **El chunking mal hecho es la causa #1 de un RAG que no funciona.** Un *chunk* es un fragmento de documento — la unidad que embeddeás, guardás y recuperás. El tamaño y los cortes determinan todo lo demás: si los chunks son malos, ni el mejor retrieval ni el mejor modelo te salvan.

Por qué importa tanto:

- **Demasiado grande** (un capítulo entero): el embedding se vuelve un promedio difuso de muchos temas, la búsqueda pierde precisión, y desperdiciás tokens metiendo texto irrelevante en el prompt.
- **Demasiado chico** (una oración suelta): el chunk pierde contexto y deja de ser auto-suficiente. "Sube un 40%" no significa nada sin saber **qué** sube.

El error clásico es el **chunking ingenuo**: cortar cada N caracteres a ciegas. Parte oraciones por la mitad, separa una afirmación de su sujeto, mezcla el final de un tema con el principio de otro. Mejores estrategias:

- **Por estructura del documento**: cortar por encabezados, secciones o párrafos (respetar los límites semánticos que el autor ya marcó). Para Markdown —como estos apuntes— cortar por `##`/`###` es naturalmente bueno.
- **Con solapamiento (overlap)**: que chunks contiguos compartan un poco de texto (ej. 10-15%), para no perder una idea que cae justo en el corte.
- **Tamaño objetivo en tokens, no en caracteres**: apuntá a un rango (ej. ~200-500 tokens) medido con el tokenizador, no con `.length`.

La regla práctica: **un buen chunk es auto-contenido** — alguien que lo lee aislado entiende de qué habla. Si para entenderlo necesitás el párrafo anterior, el chunk está mal cortado (o le falta contexto — eso lo arregla el módulo 7, Contextual Retrieval).

```ts
// Chunking ingenuo (MAL): corta a ciegas, parte oraciones
function chunkIngenuo(texto: string, tam = 1000): string[] {
  const chunks: string[] = [];
  for (let i = 0; i < texto.length; i += tam) chunks.push(texto.slice(i, i + tam));
  return chunks;
}

// Mejor: por estructura (encabezados markdown) — respeta límites semánticos
function chunkPorSecciones(markdown: string): string[] {
  return markdown
    .split(/\n(?=#{1,3}\s)/) // corta antes de cada encabezado H1-H3
    .map((s) => s.trim())
    .filter((s) => s.length > 0);
}
```

**Ejercicios 3**
3.1 ¿Por qué se dice que el chunking es el error #1 de un RAG? Da el problema de chunks muy grandes y de chunks muy chicos.
3.2 ¿Qué es el "chunking ingenuo" y por qué es malo? Nombrá dos estrategias mejores.
3.3 ¿Cuál es la prueba de que un chunk está bien cortado? (la regla de auto-contención)

---

## Módulo 4 — Embeddings y la vector database (pgvector)

**Teoría.** Ya sabés del módulo de LLMs que un **embedding** es un vector que captura el significado de un texto, y que **Anthropic no provee embeddings** — el partner recomendado es **Voyage AI**. En RAG, cada chunk se embeddéa una vez (en el ingest) y el vector se guarda para buscar después.

¿Dónde se guardan esos vectores? En una **vector database**: una base optimizada para buscar "los vectores más cercanos a este". La decisión de arquitectura clave para vos: **no necesitás una base nueva.** Con la extensión **pgvector**, tu Postgres de siempre guarda vectores y hace búsqueda por similitud. Ventaja enorme: los chunks viven **al lado de tu data relacional** (usuarios, proyectos, permisos), así que un `JOIN` o un `WHERE tenant_id = $1` filtra por permisos en la misma query (módulo 9). Una base vectorial aparte (Pinecone, Weaviate) te obliga a sincronizar dos sistemas.

El esquema, sobre lo que ya sabés de Postgres:

```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE chunks (
  id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  documento_id bigint NOT NULL REFERENCES documentos(id),
  tenant_id   bigint NOT NULL,          -- para filtrar por permisos (módulo 9)
  contenido   text   NOT NULL,          -- el texto del chunk (va al prompt)
  embedding   vector(1024)              -- el vector (dim según el modelo de Voyage)
);
```

La dimensión (`1024` arriba) la fija el modelo de embeddings: la columna `vector(N)` debe coincidir con la salida del modelo (ej. `voyage-3` produce 1024 dimensiones). Si cambiás de modelo con otra dimensión, cambia el esquema y reindexás (módulo 2).

Para buscar rápido necesitás un **índice vectorial**. Sin índice, Postgres compara la pregunta contra **todos** los vectores (full scan) — funciona con miles de filas, no con millones. pgvector ofrece **HNSW** (el default razonable hoy: muy rápido en lectura, buena precisión) e **IVFFlat**. Esto conecta con lo que viste en PostgreSQL: igual que un índice B-tree acelera un `WHERE`, el índice HNSW acelera la búsqueda por cercanía — y, como todo índice, **es una aproximación con un trade-off** (velocidad vs. recall: puede no devolver el vecino exacto). Lo verificás con `EXPLAIN`, igual que cualquier query.

```sql
-- Índice para distancia coseno (el operador de búsqueda será <=>, módulo 5)
CREATE INDEX ON chunks USING hnsw (embedding vector_cosine_ops);
```

**Ejercicios 4**
4.1 ¿Qué es una vector database y qué ventaja concreta da usar pgvector sobre Postgres en vez de una base vectorial aparte?
4.2 ¿Qué fija la dimensión de la columna `vector(N)` y qué pasa si cambiás de modelo de embeddings?
4.3 ¿Para qué sirve un índice HNSW y con qué concepto del módulo de PostgreSQL lo conectás? ¿Qué trade-off tiene?

---

## Módulo 5 — Retrieval: búsqueda vectorial sobre pgvector

**Teoría.** El corazón del pipeline de query: dada una pregunta, traer los **k chunks más cercanos**. Tres pasos:

1. Embeddear la pregunta (mismo modelo del ingest, módulo 2).
2. Buscar en pgvector los k vectores más cercanos por **distancia coseno**.
3. Devolver el **texto** de esos chunks (que irá al prompt).

En pgvector la cercanía se mide con **operadores de distancia**. El más usado en RAG es `<=>` (**distancia coseno**): cuanto menor, más parecido. Ordenás por esa distancia y limitás a k:

```sql
-- $1 = embedding de la pregunta; $2 = tenant del usuario (permisos, módulo 9)
SELECT id, contenido, embedding <=> $1 AS distancia
FROM chunks
WHERE tenant_id = $2
ORDER BY embedding <=> $1   -- el índice HNSW acelera este ORDER BY
LIMIT 5;
```

Nota técnica clave: el índice HNSW se usa **solo si el `ORDER BY` matchea el operador del índice** (`vector_cosine_ops` ↔ `<=>`). Si ordenás por otra distancia, el índice no aplica y caés en full scan — exactamente el tipo de detalle que verificás con `EXPLAIN`, como en el módulo de PostgreSQL.

En el código, envuelto detrás de una interfaz (el patrón Repository / puerto que ya usás, para no acoplar tu lógica a pgvector):

```ts
async function recuperar(pregunta: string, tenantId: number, k = 5): Promise<string[]> {
  // 1. mismo modelo de embeddings que en el ingest
  const [vector] = await embeddear([pregunta], "query"); // ver nota sobre input_type abajo
  // 2. búsqueda vectorial filtrada por permisos
  const filas = await db.query(
    `SELECT contenido FROM chunks
     WHERE tenant_id = $2
     ORDER BY embedding <=> $1
     LIMIT $3`,
    [toSqlVector(vector), tenantId, k],
  );
  return filas.map((f) => f.contenido);
}
```

Un detalle que sube la calidad gratis: muchos modelos de embeddings (Voyage entre ellos) distinguen **`input_type`** — embeddear como `"document"` en el ingest y como `"query"` en la búsqueda. Le dice al modelo "esto es algo que se va a buscar" vs. "esto es algo buscable", y mejora el match. Mismo modelo, distinto hint.

¿Cuántos chunks traer (k)? Pocos (1-2) arriesgan dejar afuera la respuesta; muchos (20) meten ruido y queman tokens. Un punto de partida típico es **k = 5**, ajustado con evaluación (módulo 11). Y ojo: la búsqueda vectorial **siempre devuelve algo** —los k más cercanos— aunque ninguno sea relevante. "Cercano" no es "correcto"; por eso vienen el rerank (módulo 6) y los evals (módulo 11).

**Ejercicios 5**
5.1 Describí los tres pasos del retrieval vectorial dado una pregunta.
5.2 ¿Qué hace el operador `<=>` en pgvector, y por qué el `ORDER BY` debe matchear el operador del índice?
5.3 La búsqueda siempre devuelve k chunks. ¿Por qué eso es un problema y qué dos módulos lo mitigan?

---

## Módulo 6 — Retrieval híbrido (vectorial + BM25) y reranking

**Teoría.** La búsqueda vectorial es genial para **significado** ("olvidé mi clave" matchea "resetear contraseña"), pero floja para **coincidencias exactas**: un código de error `ERR_4021`, el nombre propio `Zamit`, un SKU, un número de cláusula. Esos los encuentra mejor la búsqueda **léxica por palabras clave** — el clásico **BM25** (lo que hace un buscador full-text tradicional, incluido el de Postgres con `tsvector`).

El **retrieval híbrido** combina las dos: corrés la búsqueda vectorial **y** la léxica, y fusionás los resultados (una técnica común es **Reciprocal Rank Fusion**, que combina los rankings sin necesitar que las puntuaciones sean comparables). Capturás lo mejor de ambos mundos: semántica + exactitud. Saltarse el híbrido y quedarse solo en vectorial es un error frecuente cuando el dominio tiene muchos términos exactos (código, legal, productos).

El segundo refuerzo es el **reranking**, y **saltárselo es otro de los errores típicos**. El problema: el retrieval (vectorial o híbrido) está optimizado para ser **rápido sobre millones de chunks**, no para ser **preciso**. Trae, digamos, los 20 candidatos "más o menos buenos". Un **reranker** es un modelo más pesado (un cross-encoder, ej. `rerank-2` de Voyage) que mira **la pregunta y cada candidato juntos** y los reordena por relevancia real. Es caro por candidato, así que solo se aplica a los pocos que el retrieval ya filtró:

```
retrieval (rápido, trae 20)  →  reranker (preciso, reordena)  →  top 5 al prompt
       recall                            precision
```

La división mental es exactamente la de un embudo de dos etapas: **recall primero** (no perderse la respuesta entre millones, barato) y **precision después** (poner lo mejor arriba, caro pero sobre pocos). Pasarle al LLM los 5 mejores **reordenados** en vez de los 5 primeros "crudos" sube la calidad de la respuesta de forma notable, porque el modelo presta más atención a lo que va primero en el contexto.

**Ejercicios 6**
6.1 ¿En qué es mejor la búsqueda vectorial y en qué la léxica (BM25)? Dá un ejemplo de cada una.
6.2 ¿Qué es el retrieval híbrido y qué error evita?
6.3 ¿Qué hace un reranker y por qué se aplica después del retrieval y no en su lugar? Explicá el embudo recall→precision.

---

## Módulo 7 — Contextual Retrieval: la técnica de Anthropic

**Teoría.** Volvamos al problema del módulo 3: un chunk aislado **pierde el contexto del documento**. Tomá este chunk real:

> "La facturación subió un 12% respecto al trimestre anterior."

¿De qué empresa? ¿Qué trimestre? Si el usuario pregunta "¿cuánto creció ACME en el Q2?", este chunk —que es la respuesta— quizá no se recupera, porque no menciona ni "ACME" ni "Q2". El embedding no tiene de dónde sacar esa información.

**Contextual Retrieval** (la técnica de Anthropic) resuelve esto: **antes** de embeddear cada chunk, le antepone un breve contexto generado por un LLM que lo sitúa dentro del documento completo. El chunk de arriba se transforma en:

> "Este fragmento es del reporte financiero Q2 2026 de ACME Corp; la sección compara contra el Q1 2026. La facturación subió un 12% respecto al trimestre anterior."

Ahora el embedding **sí** captura "ACME", "Q2", "reporte financiero" — y se recupera correctamente. Esto se aplica en el **ingest**: por cada chunk, una llamada al LLM con `(documento completo + chunk) → "situá este chunk en una o dos frases"`, y embeddeás el resultado. Anthropic reporta que esto **reduce drásticamente la tasa de fallo del retrieval** (y combinado con BM25 y reranking, todavía más).

"¿No es carísimo, una llamada al LLM por cada chunk de cada documento?" Acá entra lo que aprendiste del **prompt caching** (módulo 9 de LLMs): el documento completo se repite en todas las llamadas de ese documento (es el prefijo estable), y el chunk cambia (va al final). Cacheás el documento y cada llamada lee el prefijo a **~0.1× del costo**. Es exactamente el criterio de "lo estable primero, lo variable al final" — y es lo que hace a Contextual Retrieval **viable en producción**. Además es ingest (offline, en tu cola): tolera latencia.

El encaje con todo lo anterior: Contextual Retrieval ataca la **calidad del chunk** (módulo 3), el híbrido + rerank atacan la **calidad del ranking** (módulo 6). Son capas complementarias, no alternativas — el sistema serio usa las tres.

**Ejercicios 7**
7.1 Con el ejemplo de la facturación, explicá por qué un chunk aislado falla en el retrieval y qué hace Contextual Retrieval para arreglarlo.
7.2 ¿En qué pipeline se aplica (ingest o query) y por qué eso importa para el costo?
7.3 ¿Cómo hace el prompt caching que Contextual Retrieval sea viable? Conectá con el criterio "lo estable primero".

---

## Módulo 8 — La generación: armar el prompt con el contexto

**Teoría.** Ya recuperaste y reordenaste los mejores chunks. El último paso es la **G** de RAG: **Generation**. Armás un prompt que combina los chunks recuperados con la pregunta del usuario, y se lo das al modelo. Acá se juega que el sistema sea confiable o que alucine.

El prompt típico:

```ts
const prompt = `Respondé la pregunta del usuario usando ÚNICAMENTE el contexto de abajo.
Si el contexto no contiene la respuesta, decí "No tengo esa información" — no inventes.
Citá la fuente entre corchetes cuando la uses.

<contexto>
${chunks.map((c, i) => `[${i + 1}] ${c}`).join("\n\n")}
</contexto>

<pregunta>
${preguntaUsuario}
</pregunta>`;
```

Las decisiones que separan un RAG serio de uno que alucina:

- **"Usá únicamente el contexto"**: instrucción explícita de **basarse (grounding) solo en lo recuperado**, no en el conocimiento general del modelo. Sin esto, el modelo mezcla tu data con lo que "recuerda" y no sabés de dónde salió cada afirmación.
- **"Si no está, decilo"**: darle permiso de **decir que no sabe**. Sin esta cláusula, el modelo siente la presión de responder igual y **alucina**. Es la mitigación de alucinación más barata y efectiva.
- **Separar contexto de instrucciones con etiquetas XML** (`<contexto>`, `<pregunta>`): lo que aprendiste en prompting — distingue tus instrucciones del dato, y es la primera defensa contra **prompt injection** (módulo de LLMs): un documento malicioso podría contener "ignorá lo anterior y...". Acá el riesgo es real porque el contexto viene de **tu data, que puede incluir input de usuarios**.
- **Citar la fuente**: como sabés qué chunk usaste, podés mostrar la cita. Da trazabilidad y deja al usuario verificar.

Conexión con lo anterior: la generación es una **llamada normal a la Messages API** (módulo 2 de LLMs), con todo lo que ya sabés —streaming para mostrar la respuesta a medida que sale, structured output si querés la respuesta como JSON con campos `respuesta` + `fuentes`, model tiering para elegir el modelo. RAG no es una API nueva: es **buena recuperación + un prompt bien armado** sobre la API que ya conocés.

**Ejercicios 8**
8.1 ¿Qué dos instrucciones en el prompt son las que más reducen la alucinación en RAG, y qué hace cada una?
8.2 ¿Por qué hay que separar el contexto recuperado de las instrucciones con etiquetas? ¿Por qué el riesgo de prompt injection es especialmente real en RAG?
8.3 Conectá la generación con el módulo de LLMs: ¿qué tres cosas que ya sabés aplicás tal cual en este paso?

---

## Módulo 9 — Permisos y multi-tenant: el RAG seguro

**Teoría.** El módulo que separa un demo de un sistema de producción, y **no filtrar permisos es uno de los errores más graves** (y más comunes) de un RAG. El problema: si tu base de conocimiento tiene documentos de varios clientes, equipos o niveles de acceso, **la búsqueda vectorial por sí sola no respeta permisos**. Trae los chunks más *parecidos*, vengan de donde vengan. Sin filtrar, el usuario A puede recibir en su respuesta un fragmento del documento confidencial del usuario B. Es una **fuga de datos** servida por tu propio sistema.

La solución es directa **gracias a haber elegido pgvector** (módulo 4): como los chunks viven en Postgres al lado de tu data relacional, filtrás permisos **en la misma query**, con un `WHERE` de toda la vida:

```sql
SELECT contenido FROM chunks
WHERE tenant_id = $2              -- el filtro de seguridad
  AND ($3::bigint IS NULL OR proyecto_id = $3)
ORDER BY embedding <=> $1
LIMIT 5;
```

Esto es lo que una base vectorial aparte hace difícil: tendrías que mantener los permisos sincronizados en dos sistemas, o filtrar **después** de la búsqueda (trayendo de más y descartando — frágil y lento). Con pgvector, el filtro y la búsqueda son una sola query atómica, y los permisos viven donde ya viven (tu modelo relacional).

Dos detalles de implementación que importan:

- **Filtrá ANTES o DURANTE la búsqueda, nunca después.** Si traés los k más cercanos y *después* descartás los que el usuario no puede ver, podés quedarte con menos de k (o cero) resultados relevantes, y estuviste procesando data prohibida. El `WHERE` va en la misma query que el `ORDER BY ... <=>`.
- **El filtro y el índice conviven**: pgvector aplica el índice HNSW combinado con el `WHERE` (filtered search). En conjuntos muy selectivos verificá con `EXPLAIN` que el plan sea sano —el mismo reflejo de PostgreSQL.

El principio integrador (del archivo senior): **defensa en profundidad** y **mínimo privilegio**. El `tenant_id` en cada chunk no es opcional ni un "lo agrego después": es parte del esquema desde el día uno (lo pusimos en el `CREATE TABLE` del módulo 4 a propósito). Tratá la base de conocimiento como tratás cualquier tabla con datos de varios dueños.

**Ejercicios 9**
9.1 ¿Por qué la búsqueda vectorial sola no respeta permisos, y qué puede pasar en un sistema multi-tenant si no filtrás?
9.2 ¿Por qué pgvector hace el filtrado por permisos más simple que una base vectorial aparte?
9.3 ¿Por qué hay que filtrar antes/durante la búsqueda y no después? Da el problema concreto de filtrar después.

---

## Módulo 10 — El criterio: RAG vs fine-tuning vs long-context

**Teoría.** El módulo de criterio, el que se evalúa en una entrevista de AI engineer. Tenés tres herramientas para "darle data al modelo" y elegir mal es caro. La confusión más peligrosa: **creer que el fine-tuning sirve para enseñarle hechos al modelo. No es así.**

- **RAG** — para **conocimiento que cambia o es específico/privado**: documentos, base de tickets, catálogo de productos, políticas internas. Data fresca (actualizás la base, no el modelo), citable, con control de acceso. **Es el default** para "responder con mi data".

- **Long-context** (meter todo en el prompt) — para **pocos documentos que entran en la ventana** y se usan en una sola interacción: "resumí este contrato", "compar~á estos 3 reportes". Simple, sin infraestructura. Pero no escala (no podés pegar 10.000 documentos en cada request), es caro por tokens en cada llamada, y sufre *lost in the middle* en contextos enormes. La regla: **¿necesito buscar entre muchos documentos, o trabajar con unos pocos que ya tengo a mano?** Lo segundo no necesita RAG.

- **Fine-tuning** — para enseñarle al modelo un **comportamiento, formato o estilo**, NO hechos. Sirve para "respondé siempre en este formato JSON de mi dominio", "adoptá este tono de marca", "clasificá según mi taxonomía". **No** sirve para "que sepa el contenido de mis documentos": meter hechos en los pesos es caro, se desactualiza al instante (cada documento nuevo = reentrenar), no es citable y no respeta permisos. Si querés que el modelo *sepa* algo, es RAG; si querés que se *comporte* distinto, es fine-tuning.

La tabla mental:

| Necesito… | Herramienta |
|---|---|
| Responder con data privada que cambia, buscando entre muchos docs | **RAG** |
| Trabajar con unos pocos documentos que ya tengo, en una interacción | **Long-context** |
| Cambiar el comportamiento/formato/estilo del modelo | **Fine-tuning** |

Y el consejo de "empezá simple" del módulo de LLMs aplica acá: **no armes un RAG si long-context te alcanza.** Montar ingest, embeddings, vector DB, retrieval, rerank y evals es trabajo real. Si tu caso es "resumir un documento que el usuario sube", una sola llamada con el documento en el prompt es la respuesta correcta. Subís a RAG cuando de verdad necesitás **buscar entre un corpus grande y cambiante**.

**Ejercicios 10**
10.1 ¿Por qué el fine-tuning NO es la herramienta para enseñarle hechos al modelo? Nombrá dos razones.
10.2 ¿Cuándo usás long-context en vez de RAG? Da la pregunta-criterio que los separa.
10.3 Para cada caso, decí qué herramienta usarías: (a) un chatbot sobre 50.000 artículos de soporte que cambian seguido; (b) que el modelo responda siempre en un JSON con tu esquema; (c) resumir el PDF que el usuario acaba de subir.

---

## Módulo 11 — Evaluar un sistema RAG (recall@k, faithfulness)

**Teoría.** El módulo que separa un prototipo de un sistema de producción —y la antesala del próximo módulo del track (Evals). Un RAG tiene **muchas piezas** (chunking, embeddings, retrieval, rerank, prompt), y "parece que anda" no es una métrica. Cuando falla, necesitás saber **dónde**. La clave: un RAG falla en **dos lugares distintos**, y se miden por separado.

**1. ¿Falla el retrieval?** (¿se recuperaron los chunks correctos?) La métrica base es **recall@k**: de las preguntas de tu set de prueba, ¿en qué fracción el chunk correcto apareció entre los k recuperados? Si el recall@k es bajo, el problema está **antes** del LLM —en el chunking, los embeddings o el retrieval— y mejorar el prompt no te va a salvar. Acá es donde medís el impacto de Contextual Retrieval, del híbrido y del rerank: corrés el mismo set con y sin cada técnica y comparás el recall.

**2. ¿Falla la generación?** Aunque el retrieval traiga lo correcto, el modelo puede responder mal. Dos métricas clave:
- **Faithfulness (fidelidad)**: ¿la respuesta se basa de verdad en el contexto recuperado, o el modelo agregó cosas de su cabeza (alucinó)? Una respuesta puede sonar bien y ser infiel al contexto.
- **Relevancia**: ¿la respuesta efectivamente contesta la pregunta?

Cómo se mide en la práctica: armás un **golden set** —un conjunto de preguntas con su respuesta correcta y el/los chunk(s) que la contienen—. El recall@k se calcula con código (¿el chunk esperado está entre los k?). La faithfulness y la relevancia, al ser sobre texto libre, se evalúan con **LLM-as-judge**: otro LLM (o el mismo) puntúa si la respuesta es fiel al contexto y relevante a la pregunta. Esto es exactamente el puente al **módulo de Evals** del track, donde lo desarrollamos.

El principio (el mismo del archivo senior y del módulo de costo de LLMs): **medí antes de optimizar.** Sin separar retrieval de generación, vas a "mejorar el prompt" cuando el problema era el chunking, o a tocar el chunking cuando el modelo estaba alucinando con buen contexto. Instrumentá las dos capas por separado y atacá la que mide mal.

**Ejercicios 11**
11.1 ¿Cuáles son los dos lugares donde puede fallar un RAG, y por qué hay que medirlos por separado?
11.2 ¿Qué mide recall@k y qué te dice un recall@k bajo sobre dónde está el problema?
11.3 ¿Qué es la faithfulness y por qué no alcanza con que la respuesta "suene bien"? ¿Cómo se mide algo así sobre texto libre?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 RAG recupera los fragmentos relevantes de tu data y se los pasa al LLM en el prompt
    para que responda basándose en ellos. Se parece a un examen a libro abierto: no le
    pedís al modelo que recuerde, le pasás la página correcta.
1.2 Porque el LLM no conoce data privada ni nada posterior a su corte de entrenamiento; el
    contrato no está en sus pesos. Tres salidas: (1) meter todo en el prompt (long-context),
    (2) fine-tuning, (3) RAG.
1.3 (1) Data fresca: actualizás la base sin reentrenar. (2) Control de acceso: filtrás qué
    ve cada usuario. (También: citabilidad/trazabilidad.) Con fine-tuning la data queda
    fundida en los pesos: no es fresca, ni citable, ni filtrable.
```

### Módulo 2
```
2.1 Ingest: documento → chunking → embeddings → guardar (texto + vector). Query: pregunta →
    embedding de la pregunta → buscar los k chunks más cercanos → (rerank opcional) → armar
    prompt → LLM → respuesta.
2.2 Porque el ingest es trabajo pesado y batch (chunking + muchas llamadas de embeddings),
    no algo que el usuario deba esperar en un request. Conecta con el módulo de colas
    (Redis/BullMQ): subís un documento → encolás un job de ingest.
2.3 La búsqueda devuelve basura: la pregunta y los chunks viven en espacios vectoriales
    distintos y la "cercanía" deja de significar nada. Al cambiar de modelo de embeddings
    hay que reindexar TODO (reembeddear todos los chunks).
```

### Módulo 3
```
3.1 Porque si los chunks son malos, ni el mejor retrieval ni el mejor modelo recuperan la
    respuesta. Muy grandes: el embedding se vuelve un promedio difuso, pierde precisión y
    desperdicia tokens. Muy chicos: el chunk pierde contexto y deja de ser auto-suficiente.
3.2 Cortar cada N caracteres a ciegas; es malo porque parte oraciones, separa una afirmación
    de su sujeto y mezcla temas. Mejores: (1) cortar por estructura (encabezados/secciones/
    párrafos), (2) con solapamiento (overlap) entre chunks contiguos. (También: medir el
    tamaño en tokens, no en caracteres.)
3.3 La auto-contención: si alguien lo lee aislado, entiende de qué habla. Si para entenderlo
    necesita el párrafo anterior, está mal cortado o le falta contexto (Contextual Retrieval).
```

### Módulo 4
```
4.1 Una base optimizada para buscar "los vectores más cercanos a este". Con pgvector, los
    chunks viven en tu Postgres al lado de la data relacional: podés filtrar por permisos con
    un WHERE en la misma query, sin sincronizar dos sistemas (lo que obliga una base aparte).
4.2 La fija el modelo de embeddings: vector(N) debe coincidir con la dimensión de salida del
    modelo (ej. voyage-3 → 1024). Si cambiás a un modelo con otra dimensión, cambia el
    esquema y hay que reindexar todo.
4.3 Acelera la búsqueda por cercanía (los k vecinos más próximos) para no comparar contra
    todos los vectores (full scan). Se conecta con los índices del módulo de PostgreSQL:
    igual que un B-tree acelera un WHERE, HNSW acelera el ORDER BY por distancia. Trade-off:
    es una aproximación (velocidad vs. recall: puede no devolver el vecino exacto).
```

### Módulo 5
```
5.1 (1) Embeddear la pregunta con el mismo modelo del ingest. (2) Buscar en pgvector los k
    vectores más cercanos por distancia coseno. (3) Devolver el texto de esos chunks.
5.2 <=> calcula la distancia coseno entre dos vectores (menor = más parecido). El ORDER BY
    debe usar el mismo operador que el índice (vector_cosine_ops ↔ <=>) porque si no, el
    índice HNSW no se aplica y caés en full scan (verificable con EXPLAIN).
5.3 Porque "cercano" no es "correcto": devuelve los k más cercanos aunque ninguno sea
    relevante, lo que puede meter ruido en el prompt. Lo mitigan el rerank (módulo 6, reordena
    por relevancia real) y los evals (módulo 11, miden si se recuperó lo correcto).
```

### Módulo 6
```
6.1 Vectorial: significado/semántica — "olvidé mi clave" matchea "resetear contraseña".
    Léxica (BM25): coincidencias exactas — un código ERR_4021, un nombre propio, un SKU, un
    número de cláusula.
6.2 Correr la búsqueda vectorial Y la léxica y fusionar los resultados (ej. Reciprocal Rank
    Fusion). Evita el error de quedarse solo en vectorial y perder las coincidencias exactas
    en dominios con muchos términos precisos (código, legal, productos).
6.3 Un reranker (cross-encoder) mira la pregunta y cada candidato juntos y los reordena por
    relevancia real. Va después porque es caro por candidato: el retrieval es rápido y trae
    muchos candidatos (recall, sobre millones); el rerank es preciso pero solo sobre los pocos
    que el retrieval ya filtró (precision). Embudo: recall barato primero, precision cara
    después.
```

### Módulo 7
```
7.1 Aislado, "la facturación subió un 12%" no menciona ni la empresa ni el trimestre, así que
    su embedding no captura "ACME" ni "Q2" y no se recupera ante esa pregunta. Contextual
    Retrieval antepone, antes de embeddear, un contexto generado por un LLM que sitúa el chunk
    en el documento ("este fragmento es del reporte Q2 2026 de ACME..."), así el embedding sí
    captura esos términos.
7.2 En el ingest (offline). Importa para el costo porque tolera latencia (corre en la cola, no
    en el request) y porque permite cachear el documento entre llamadas.
7.3 El documento completo se repite en todas las llamadas de ese documento (prefijo estable) y
    el chunk cambia (al final). Con prompt caching, el documento se cachea y cada llamada lo
    lee a ~0.1× del costo. Es el criterio "lo estable primero, lo variable al final" — lo que
    vuelve la técnica viable en producción.
```

### Módulo 8
```
8.1 (1) "Usá únicamente el contexto": fuerza el grounding en lo recuperado y evita que el
    modelo mezcle su conocimiento general. (2) "Si no está, decí que no sabés": le da permiso
    de no responder, que es la mitigación de alucinación más barata y efectiva.
8.2 Para que el modelo distinga tus instrucciones del dato a procesar (como en prompting) y
    como primera defensa contra prompt injection. El riesgo es especialmente real en RAG
    porque el contexto viene de tu data, que puede incluir contenido subido por usuarios y por
    tanto instrucciones maliciosas.
8.3 (1) Es una llamada normal a la Messages API. (2) Streaming para mostrar la respuesta a
    medida que sale. (3) Structured output si querés la respuesta como JSON (respuesta +
    fuentes). (También: model tiering para elegir el modelo.)
```

### Módulo 9
```
9.1 Porque la búsqueda vectorial trae los chunks más PARECIDOS, sin importar de quién son. En
    multi-tenant, el usuario A puede recibir un fragmento del documento confidencial del
    usuario B: una fuga de datos servida por tu propio sistema.
9.2 Porque los chunks viven en Postgres al lado de la data relacional y los permisos, así que
    filtrás con un WHERE (tenant_id = $1) en la misma query que la búsqueda. Una base aparte
    obliga a sincronizar permisos en dos sistemas o a filtrar después (frágil y lento).
9.3 Porque si filtrás después de traer los k más cercanos, podés quedarte con menos de k (o
    cero) resultados relevantes, y además procesaste data prohibida. El WHERE debe ir en la
    misma query que el ORDER BY ... <=>.
```

### Módulo 10
```
10.1 Porque meter hechos en los pesos: (1) se desactualiza al instante (cada documento nuevo
     exige reentrenar), (2) no es citable ni respeta permisos. (También: es caro y los hechos
     se mezclan/alucinan.) Fine-tuning sirve para comportamiento/formato/estilo, no para saber
     hechos.
10.2 Cuando trabajás con pocos documentos que ya tenés a mano, en una sola interacción, y
     entran en la ventana de contexto. La pregunta-criterio: ¿necesito BUSCAR entre muchos
     documentos (RAG) o trabajar con unos pocos que ya tengo (long-context)?
10.3 (a) RAG (corpus grande y cambiante que hay que buscar). (b) Fine-tuning (es un
     comportamiento/formato, no un hecho). (c) Long-context (un solo documento, una
     interacción — una llamada con el PDF en el prompt).
```

### Módulo 11
```
11.1 (1) El retrieval (¿se recuperaron los chunks correctos?) y (2) la generación (¿el modelo
     respondió bien con el contexto?). Se miden por separado porque la causa y el arreglo son
     distintos: si falla el retrieval, mejorar el prompt no sirve; si falla la generación, el
     contexto ya estaba bien.
11.2 Recall@k mide en qué fracción de las preguntas el chunk correcto apareció entre los k
     recuperados. Un recall@k bajo dice que el problema está ANTES del LLM (chunking,
     embeddings o retrieval), no en el prompt.
11.3 Faithfulness es si la respuesta se basa de verdad en el contexto recuperado y no en cosas
     que el modelo inventó. No alcanza con que "suene bien" porque una respuesta fluida puede
     ser infiel al contexto (alucinada). Al ser texto libre se mide con LLM-as-judge: otro LLM
     puntúa si la respuesta es fiel al contexto y relevante a la pregunta (puente al módulo de
     Evals).
```

---

## Siguientes pasos

Con este módulo podés construir un RAG con criterio de producción: los dos pipelines, chunking que no sabotea todo, pgvector sobre tu Postgres, retrieval vectorial + híbrido + reranking, Contextual Retrieval con prompt caching, una generación que no alucina, filtrado por permisos multi-tenant, el criterio de cuándo NO usar RAG, y cómo evaluarlo. El recorrido del track de IA desde acá: **(1) Agentes** — el loop de tool use (módulo 5 de LLMs) llevado a sistemas que deciden su propio camino y usan RAG como una herramienta más, con **MCP** y el **Claude Agent SDK**; **(2) Evaluations** — cómo medir la calidad de un sistema LLM de punta a punta (lo que asomó en el módulo 11), que es lo que separa un prototipo de un sistema de producción. El hilo conductor sigue en pie: el mismo sistema que arrancó como una llamada simple ahora **busca en tus propios apuntes**; el próximo paso es darle herramientas y dejarlo decidir.
