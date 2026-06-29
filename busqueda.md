# Búsqueda a escala: full-text, relevancia y el motor por dentro

**Índice invertido, análisis de texto, TF-IDF/BM25, la arquitectura de Lucene/Elasticsearch (segmentos, shards, scatter-gather) y cuándo te alcanza con Postgres · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es el **deep-dive** de la búsqueda full-text: en [Datos a escala](datos-escala.md) M14 viste *por qué* freshness y ranking pelean a escala (el trade-off conceptual); acá **construís el motor** — cómo se representa el texto, cómo se calcula la relevancia, y cómo Lucene/Elasticsearch lo escalan a millones de documentos. El propio M14 te mandó acá: *"la implementación de un motor (índice invertido, BM25, sharding del índice) da para un módulo aparte"*. Este es ese módulo.

**Lo que asumimos.** Que sabés qué es un índice de base de datos y un `O(log n)`, que viste el trade-off B-tree vs LSM ([Datos a escala](datos-escala.md) M11-12) — porque el motor de búsqueda **reusa la idea del LSM** (módulo 4) — y que entendés sharding y scatter-gather ([Datos a escala](datos-escala.md) M5). No asumimos que hayas usado Elasticsearch.

> **Ojo con qué problema resolvemos.** Esto **no** es el [typeahead/autocompletar del Caso 4](system-design-casos.md) (ese **completa prefijos** con un trie precomputado: "lap" → "laptop"). Acá hacés **full-text retrieval**: dado un texto de consulta, encontrar y **rankear por relevancia** los documentos que lo contienen ("zapatillas rojas para correr" sobre un catálogo de millones). Son dos problemas distintos con dos estructuras distintas (trie vs índice invertido).

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Índice de módulos**
1. El índice invertido: por qué `LIKE %x%` no escala
2. Del texto a términos: el pipeline de análisis (tokenización, normalización, stemming)
3. Relevancia: de TF-IDF a BM25
4. El motor por dentro: segmentos inmutables (= SSTables) y near-real-time
5. Escalar el índice: shards, réplicas y scatter-gather
6. Freshness vs ranking y los modos de falla
7. El criterio: ¿Elasticsearch o te alcanza con Postgres?

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El índice invertido: por qué `LIKE %x%` no escala

**Teoría.** El instinto de junior para buscar texto es `SELECT * FROM productos WHERE descripcion LIKE '%zapatilla%'`. **No escala**: el comodín al inicio (`%...`) impide usar cualquier índice, así que la base **escanea cada fila** comparando el patrón → `O(n)` sobre todo el corpus. A un millón de documentos, cada búsqueda los lee todos.

La estructura que lo resuelve es el **índice invertido**. La idea (la misma de un índice al final de un libro): en vez de ir documento → palabras, das vuelta el mapa a **término → lista de documentos que lo contienen**. Esa lista se llama **posting list**.

```
"zapatilla"  → [doc1, doc7, doc42, doc108, ...]
"roja"       → [doc7, doc42, doc55, ...]
"correr"     → [doc42, doc99, ...]
```

Buscar **"zapatilla roja"** ya no escanea nada: tomás las posting lists de `"zapatilla"` y `"roja"` y las **intersecás** (AND) → `[doc7, doc42]`. Si querés un OR, las **unís**. Como las posting lists están **ordenadas por doc id**, intersecarlas es un *merge* lineal sobre los candidatos, no un escaneo del corpus. El costo pasa de "cuántos documentos hay" a "cuántos documentos contienen estos términos" — que para términos específicos es **muchísimo** menos.

Dos detalles que el ejemplo simplifica pero que un motor real sí tiene: (1) las posting lists llevan **skip pointers** (punteros de salto) para que el AND **saltee** bloques de la lista larga en vez de hacer un merge lineal puro — clave cuando una lista tiene millones de entradas; (2) además del doc id, cada entrada guarda las **posiciones** del término dentro del documento. Eso es lo que habilita las **frases exactas** (`"zapatilla roja"` como secuencia contigua, no solo ambos términos sueltos) y el **highlighting** (resaltar el fragmento donde aparece el término en los resultados). El código del módulo 3 solo guarda el `tf` para simplificar; uno de verdad guarda también las posiciones.

Esto es **boolean retrieval**: encontrar el **conjunto** de documentos que matchean. Pero "encontrar" no alcanza — si `"zapatilla"` aparece en 50.000 productos, devolver los 50.000 en cualquier orden es inútil. Falta **rankear por relevancia** (módulo 3). Primero, cómo convertimos texto crudo en los **términos** del índice (módulo 2).

> Puente con [Datos a escala](datos-escala.md) M14: ahí definimos el índice invertido y el lag de freshness a nivel conceptual. Acá lo abrimos: posting lists, intersección, y de acá en adelante el análisis y el ranking.

La frase mental: **`LIKE %x%` escanea todo (`O(n)`, no usa índice). El índice invertido da vuelta el mapa: `término → posting list` de documentos. Buscar = intersecar/unir posting lists ordenadas, no escanear el corpus. Eso encuentra los documentos; rankearlos es otra historia.**

**Ejercicios 1**
1.1 🔁 ¿Qué es un índice invertido y qué es una posting list? ¿Por qué `LIKE '%x%'` no escala?
1.2 🧠 Tenés las posting lists de `"rojo"` (10.000 docs) y `"ferrari"` (200 docs) y querés los que tienen **ambos**. ¿Por qué conviene recorrer empezando por la lista más corta, y qué operación es "ambos" vs "cualquiera"?
1.3 🧠 El índice invertido encuentra el conjunto de documentos que matchean, pero eso no alcanza para una buena búsqueda. ¿Qué falta y por qué el conjunto "pelado" es insuficiente?

---

## Módulo 2 — Del texto a términos: el pipeline de análisis

**Teoría.** ¿Qué guardás exactamente en el índice como "término"? No las palabras crudas. Si indexás literal, `"Corriendo"` no matchea una búsqueda de `"correr"`, ni `"Café"` matchea `"cafe"`. La solución es pasar **tanto el texto a indexar como la consulta** por un **analyzer**: un pipeline que normaliza ambos al mismo vocabulario. Las etapas típicas:

1. **Tokenización:** partir el texto en *tokens* (normalmente por espacios/puntuación). `"Zapatillas rojas, talle 42"` → `["Zapatillas", "rojas", "talle", "42"]`.
2. **Normalización:** pasar a **minúsculas**, **quitar acentos/diacríticos** (`"café"` → `"cafe"`), a veces unificar variantes. → `["zapatillas", "rojas", "talle", "42"]`.
3. **Stop words (opcional):** descartar palabras vacías muy frecuentes y poco discriminantes (`"el"`, `"de"`, `"the"`, `"a"`). Ahorra espacio, aunque los motores modernos las conservan más seguido (BM25 ya les da poco peso — módulo 3).
4. **Stemming / lematización:** reducir a la **raíz**. *Stemming* corta heurísticamente (`"corriendo"`, `"corre"`, `"corrió"` → `"corr"`); *lematización* usa diccionario para llegar al lema real (`"mejor"` → `"bueno"`). Así variantes de la misma palabra caen en **un mismo término**.
5. **Sinónimos (opcional):** un *token filter* que expande o reescribe términos equivalentes (`"celu"` → `"celular"`, `"notebook"` ↔ `"laptop"`), para que buscar uno encuentre el otro. Se configura como una etapa más del analyzer (a menudo solo en query-time). Es una de las features que se afinan en un motor dedicado (módulo 7).

La regla de oro, la que más se rompe en la práctica: **el mismo analyzer en index-time y en query-time.** Si indexás con stemming pero la consulta no lo aplica (o usás otro idioma), los términos **no coinciden** y no matchea nada — aunque el documento "obviamente" contenga la palabra. El análisis **depende del idioma** (el stemmer de español no sirve para inglés), así que elegir el analyzer correcto por campo/idioma es una decisión de diseño real.

> Conexión con el módulo 1: lo que termina en las posting lists son estos **términos analizados**, no las palabras originales. Por eso buscar `"CORRER"` encuentra un documento que decía `"corriendo"`: ambos colapsan al término `"corr"`.

La frase mental: **el texto crudo no va al índice: pasa por un analyzer (tokenizar → minúsculas/sin acentos → stop words → stemming) que lo reduce a términos normalizados. Regla de oro: el MISMO analyzer en indexación y en consulta, o no matchea. Y depende del idioma.**

**Ejercicios 2**
2.1 🔁 Nombrá las etapas del pipeline de análisis y qué hace cada una. ¿Qué diferencia hay entre stemming y lematización?
2.2 🧠 Indexaste con un analyzer en español (con stemming) pero en query-time usás uno que solo tokeniza y pasa a minúsculas. Buscás `"zapatos"` y no encontrás un producto cuya descripción dice `"zapato"`. ¿Qué pasó?
2.3 🧠 ¿Por qué quitar acentos y pasar a minúsculas en **ambos** lados (index y query) es lo que hace que `"Café"` matchee `"cafe"`? ¿Qué pasaría si solo lo hicieras en uno?

---

## Módulo 3 — Relevancia: de TF-IDF a BM25

**Teoría.** El corazón de la búsqueda: dado que varios documentos matchean, **¿en qué orden los devolvés?** Necesitás un **score de relevancia** por documento. Dos intuiciones lo gobiernan:

- **TF (term frequency):** cuanto **más veces** aparece el término en un documento, más sobre ese término trata. Un artículo que dice "jazz" 30 veces es más sobre jazz que uno que lo dice una vez.
- **IDF (inverse document frequency):** cuanto **más raro** es el término en *todo el corpus*, más **discrimina**. Si buscás "el teorema de Bayes", la palabra "el" aparece en todos los documentos (no distingue nada, IDF bajo); "Bayes" aparece en poquísimos (IDF alto) → matchear "Bayes" vale muchísimo más.

**TF-IDF** combina las dos: `score = TF × IDF`, sumado sobre los términos de la consulta. Fue el ranking clásico durante décadas. Pero tiene dos defectos que **BM25** (el estándar actual) corrige:

1. **Saturación de TF.** En TF-IDF crudo, 200 apariciones puntúan 100× más que 2. Irreal: pasado cierto punto, repetir un término **no** lo hace más relevante. BM25 hace que el aporte de TF **sature** (crezca rápido al principio y se aplane) con un parámetro `k1` (≈1.2–2.0).
2. **Normalización por longitud.** Un documento largo tiene más palabras → más chances de TF alto solo por ser largo. BM25 **penaliza la longitud** relativa a la media (`avgdl`) con un parámetro `b` (≈0.75), para que un doc corto y al punto no pierda contra uno largo y disperso.

La fórmula de BM25 (no hay que memorizarla, sí entender las piezas):

```
                              f(q,D) · (k1 + 1)
score(D, Q) = Σ  IDF(q) ·  ───────────────────────────────────
            q ∈ Q           f(q,D) + k1 · (1 − b + b · |D|/avgdl)

donde  f(q,D) = veces que el término q aparece en D
       |D|    = longitud del documento D (en términos)
       avgdl  = longitud media de los documentos del corpus
       k1     ≈ 1.2  (controla la saturación de TF)
       b      ≈ 0.75 (controla la penalización por longitud)

       IDF(q) = ln(1 + (N − n(q) + 0.5) / (n(q) + 0.5))
                N = total de documentos, n(q) = docs que contienen q
```

Leelo así: **IDF** pondera cada término por lo raro que es; la fracción es **TF saturado y normalizado por longitud**. El numerador `f·(k1+1)` es lo que hace que, cuando el TF crece mucho, el aporte **tienda a `IDF·(k1+1)`** en vez de a infinito — esa es la saturación. Es el ranking por defecto de **Lucene/Elasticsearch** (desde 2016 ⚠️; antes era TF-IDF).

> **El `1 +` dentro del log del IDF no es decorativo.** El IDF original (Robertson/Spärck-Jones) es `ln((N − n + 0.5)/(n + 0.5))`, que se vuelve **negativo** para un término presente en **más de la mitad** de los documentos (`n > N/2`) — y un IDF negativo haría que un término común **reste** score. Lucene mete el `1 +` justamente para forzar **IDF ≥ 0**: un término ultra-frecuente aporta ~0, nunca negativo.

Y un punto senior: BM25 es la **relevancia textual**, pero el ranking final de un producto/posteo suele combinarla con **señales de negocio** — popularidad, recencia, rating, personalización — en una función de scoring por encima (o un modelo de *learning to rank*). BM25 ordena por "qué tan bien matchea el texto"; el negocio decide cuánto pesa eso contra "qué tanto se vende". Y un límite importante: BM25 es **léxico** — matchea **términos**, no **significado**. No sabe que `"auto"` y `"vehículo"` son parientes salvo que se lo digas con sinónimos. Esa brecha la cubre la **búsqueda semántica/vectorial** (embeddings), y en 2026 lo común es **combinarlas** (búsqueda híbrida — ver el cierre).

> 🔥 **Falla en 4 capas — un término ultra-común infla el ranking.**
> (1) **Qué se rompe:** una búsqueda como `"the office"` devuelve basura arriba porque `"the"` matchea medio corpus y arrastra documentos irrelevantes al top.
> (2) **Por qué a esta escala:** sin ponderar rareza, los términos frecuentes dominan por volumen de matches.
> (3) **Corto plazo:** confiar en el **IDF**, que con la corrección `1 +` ya le da peso ≈0 a `"the"` (no hace falta sacarlo); las stop words quedan como optimización opcional para **ahorrar índice**, no como la defensa principal.
> (4) **Largo plazo:** BM25 con IDF correcto + saturación de TF, de modo que el término raro (`"office"`) domine el score y el común no infle por repetición; y campos con *boost* (título > cuerpo).

La frase mental: **rankeás con un score: TF (más apariciones → más relevante) × IDF (término más raro → más discrimina). BM25 mejora a TF-IDF saturando el TF (la 200ª aparición no vale como la 2ª, `k1`) y normalizando por longitud del doc (`b`). Es el default de Lucene desde 2016. Encima va el negocio: popularidad, recencia, personalización.**

**Ejercicios 3**
3.1 🔁 ¿Qué capturan TF e IDF, y por qué un término raro debería pesar más que uno común?
3.2 🧠 ¿Cuáles son los dos problemas de TF-IDF que BM25 arregla, y qué parámetro controla cada uno?
3.3 ✍️ Implementá en TypeScript un mini índice invertido con scoring **BM25**: `addDoc(docId, tokens)` y `search(queryTokens)` que devuelva los doc ids ordenados por score descendente. Usá `k1 = 1.2`, `b = 0.75` y el IDF de BM25. (La solución está al final; tiene que compilar en `--strict`.)

---

## Módulo 4 — El motor por dentro: segmentos inmutables (= SSTables) y near-real-time

**Teoría.** Acá está el gancho que conecta búsqueda con lo que ya sabés de motores de almacenamiento. ¿Cómo guarda Lucene (el motor debajo de Elasticsearch/OpenSearch/Solr) el índice invertido en disco, soportando escrituras constantes? **Con la misma idea del LSM-tree** ([Datos a escala](datos-escala.md) M11): **archivos inmutables que solo se agregan + un merge de fondo.**

- Lucene escribe el índice en **segmentos**: cada segmento es un **mini índice invertido completo e inmutable**. Es el análogo exacto de la **SSTable** del LSM.
- Indexar un documento nuevo **no modifica** un segmento existente (son inmutables): se bufferiza en memoria y, en el siguiente **refresh**, se escribe un **segmento nuevo**. Igual que la memtable que se vuelca a una SSTable.
- **Borrar/actualizar** no edita el segmento: marca el doc como borrado con un **tombstone** (un bitset de "docs vivos"). El espacio se recupera después. Idéntico al tombstone del LSM.
- Un **merge** de fondo consolida segmentos chicos en grandes, descartando los docs borrados. Es la **compaction** del LSM con otro nombre.

De esto sale, sin misterio, el **lag de freshness** del que hablaba [Datos a escala](datos-escala.md) M14: un documento recién escrito **no es buscable hasta el próximo refresh** (en Elasticsearch, **~1s** por default ⚠️), porque hasta entonces no existe como segmento consultable. **Near-real-time, no real-time.** Bajar el refresh interval da más frescura pero **cuesta** (más segmentos chicos, más merges, más CPU) — el mismo tipo de trade-off que afinar un LSM.

> ⚠️ El "~1s" es el default de Elastic Stack/self-managed; en **ES Serverless** es **5s**. Y desde 7.x el refresh es **adaptativo**: un índice **idle** (sin búsquedas por ~30s) **deja de refrescar** hasta que llega una query — por eso, en un índice poco consultado, un doc puede tardar **más** de 1s en aparecer. El lag no es un número fijo, depende de la config y de si el índice está recibiendo búsquedas.

Y aparece la misma **read amplification**: como hay **varios segmentos**, una búsqueda consulta **todos** los segmentos del shard y combina sus resultados (igual que una lectura LSM toca varias SSTables). El merge mantiene ese número acotado.

> El paralelo completo, para fijarlo:
>
> | Lucene (búsqueda) | LSM-tree (almacenamiento) |
> |---|---|
> | Segmento (inmutable) | SSTable (inmutable) |
> | Buffer de indexación + refresh | Memtable + flush |
> | Tombstone (doc borrado) | Tombstone (key borrada) |
> | Merge de segmentos | Compaction |
> | Lag de freshness (~1s) | — (la escritura es visible al flush) |
>
> Misma familia de diseño: **escribir es barato (append a un archivo nuevo); leer toca varios archivos; un proceso de fondo consolida.**

La frase mental: **Lucene es un LSM para búsqueda: segmentos inmutables (= SSTables), un doc nuevo crea un segmento en el refresh (≈1s → near-real-time, de ahí el lag de freshness), borrar es un tombstone, y un merge de fondo (= compaction) consolida. Por eso una búsqueda toca varios segmentos (read amplification).**

**Ejercicios 4**
4.1 🔁 ¿Qué es un segmento de Lucene y con qué pieza del LSM-tree se corresponde? ¿Qué pasa al borrar un documento?
4.2 🧠 ¿De dónde sale exactamente el "lag de freshness" (que un doc recién escrito no se pueda buscar todavía)? ¿Qué ganás y qué perdés bajando el refresh interval?
4.3 🧠 ¿Por qué una búsqueda en un shard tiene "read amplification", y qué proceso la mantiene acotada? (relacionalo con la lectura de un LSM)

---

## Módulo 5 — Escalar el índice: shards, réplicas y scatter-gather

**Teoría.** Un índice de mil millones de documentos no entra en una máquina. Se **parte en shards**: cada **shard es un índice Lucene completo e independiente** (con sus propios segmentos), y los documentos se reparten entre shards (por hash del id). Dos consecuencias clave:

- **Réplicas:** cada shard puede tener **copias** (réplicas) en otros nodos → tolerancia a fallos (si cae el nodo del shard primario, una réplica lo reemplaza) **y** más throughput de lectura (las búsquedas se reparten entre primario y réplicas). Es la replicación de [Datos a escala](datos-escala.md)/[Replicación](replicacion.md) aplicada al índice.
- **Scatter-gather:** como un documento puede estar en **cualquier** shard, una búsqueda **va a TODOS los shards** (scatter), cada uno devuelve sus **top-k locales**, y un nodo **coordinador los mergea** en el **top-k global** (gather). Es el mismo *scatter-gather* que [Datos a escala](datos-escala.md) M5 describe como el costo de no poder dirigir la query a un shard — solo que en búsqueda es **inevitable** (no sabés en qué shard cayó lo relevante).

El flujo concreto en Elasticsearch es **query-then-fetch** en dos fases:
1. **Query:** el coordinador manda la consulta a un shard (primario o réplica) de cada partición; cada uno calcula scores y devuelve solo los **ids + scores** de sus top-k.
2. **Fetch:** el coordinador mergea, se queda con el top-k **global**, y recién entonces **pide los documentos completos** de esos k a los shards que los tienen. (Así no mueve documentos enteros de shards que no entran al top.)

Dos trampas que caen en entrevistas:

- **IDF distribuido.** El IDF (módulo 3) necesita estadísticas **globales** (en cuántos docs del corpus aparece el término), pero cada shard calcula el IDF con **sus** estadísticas locales → los scores **no son estrictamente comparables** entre shards. El desvío se nota sobre todo con **shards de pocos documentos** (índices recién creados, tests); a **millones de docs por shard** las stats locales **convergen a las globales** (ley de grandes números) y la aproximación es tan buena que nadie se da cuenta. Si igual necesitás exactitud, existe un modo que primero junta stats globales (`dfs_query_then_fetch` ⚠️) a costa de un round-trip extra — que rara vez vale la pena salvo con shards chicos.
- **Deep pagination.** Para devolver la **página 1000** (`from=10000, size=10`), cada shard debe devolver `from+size` = 10.010 resultados al coordinador para poder mergear bien → costo que **crece con la profundidad** y puede tumbar el cluster. Por eso Elasticsearch **limita** `from+size` (≈10.000 por default ⚠️) y para paginar profundo te da `search_after` / *point-in-time* (un cursor, como la **paginación por cursor** de [api-design](api-design.md) M5) en vez de offset.

La frase mental: **el índice se parte en shards (cada uno un índice Lucene completo) con réplicas (HA + throughput de lectura). Como lo relevante puede estar en cualquier shard, la búsqueda es scatter-gather: preguntás a todos, mergeás top-k locales en top-k global (query-then-fetch). Trampas: el IDF es local por shard (aprox) y la paginación profunda explota (usá cursor, no offset).**

**Ejercicios 5**
5.1 🔁 ¿Qué es un shard en un motor de búsqueda y por qué una query tiene que ir a todos? ¿Para qué sirven las réplicas?
5.2 🧠 Explicá query-then-fetch: ¿por qué primero se piden solo ids+scores y recién después los documentos completos?
5.3 🧠 ¿Por qué pedir la página 1000 con `from=10000` es tan caro en un sistema con scatter-gather, y qué mecanismo lo reemplaza? (conectalo con la paginación por cursor de [api-design](api-design.md))

---

## Módulo 6 — Freshness vs ranking y los modos de falla

**Teoría.** El cierre operativo, retomando el trade-off de [Datos a escala](datos-escala.md) M14 ahora que conocés el motor. **Freshness** (que lo recién escrito se pueda buscar ya) y **ranking** (que el orden sea bueno) **pelean** porque un buen ranking necesita estadísticas globales (IDF, señales) que recalcular sobre un índice que muta sin parar es caro. La resolución es la de siempre en este hub: **separar el camino rápido del caro** (la idea Lambda/Kappa de [Streams](streams.md)): indexación near-real-time para frescura + ranking refinado con stats actualizadas periódicamente o en una segunda pasada.

Ante un tema de búsqueda, estructurá con **las tres cosas**: **happy path** (analyzer consistente → índice invertido sharded → BM25 + señales → top-k), **modelo de falla** (reindex que satura, shard caliente, deep pagination, scores incomparables), **recuperación** (réplicas, reindex incremental con alias, límites de paginación). Dos fallas típicas en 4 capas:

> 🔥 **Falla en 4 capas — el reindex masivo que satura el cluster.**
> (1) **Qué se rompe:** cambiás el analyzer/mapping y hay que **reindexar** todo (el índice invertido no se puede "alterar en su lugar", igual que un segmento es inmutable); el reindex compite por CPU/IO con las búsquedas en vivo y la latencia se dispara.
> (2) **Por qué a esta escala:** reindexar millones de docs es escribir un índice nuevo entero mientras servís tráfico.
> (3) **Corto plazo:** *throttling* del reindex, hacerlo en horas valle, subir réplicas para absorber lecturas.
> (4) **Largo plazo:** **reindex contra un índice nuevo + swap por alias** (construís `productos_v2` en paralelo y al final apuntás el alias `productos` de v1 a v2 de forma atómica) — es el **expand-contract** de [Datos a escala](datos-escala.md) M9 aplicado al índice: nunca reindexás "en caliente" el que sirve.

> 🔥 **Falla en 4 capas — deep pagination tumba el cluster.**
> (1) **Qué se rompe:** un scraper pide `from=900000` y cada shard tiene que materializar 900.000+ resultados para mergear → memoria y CPU por las nubes.
> (2) **Por qué a esta escala:** el costo del offset crece linealmente con la profundidad **por shard** (módulo 5).
> (3) **Corto plazo:** límite duro de `max_result_window`; rechazar/420 al que pagina demasiado profundo.
> (4) **Largo plazo:** exponer solo **paginación por cursor** (`search_after`/PIT), no offset profundo — y para "exportar todo", una API de scroll/export dedicada, no la de búsqueda interactiva.

La frase mental: **freshness y ranking pelean (buen orden ⇒ stats globales caras sobre un índice que muta): se resuelven separando indexación rápida de ranking refinado (Lambda/Kappa). Fallas típicas: reindex masivo (→ índice nuevo + swap por alias, expand-contract) y deep pagination (→ cursor, no offset).**

**Ejercicios 6**
6.1 🔁 ¿Por qué freshness y ranking entran en tensión a escala? ¿Cuál es la estrategia general para resolverlo?
6.2 🧠 Tenés que cambiar el analyzer de un índice de 50M de documentos en producción sin downtime ni degradar las búsquedas en vivo. ¿Cómo lo hacés? (pista: alias + expand-contract)
6.3 🧠 Aplicá "las tres cosas" (happy path / modelo de falla / recuperación) a una búsqueda de catálogo a escala.

---

## Módulo 7 — El criterio: ¿Elasticsearch o te alcanza con Postgres?

**Teoría.** El cierre senior, igual que en los casos del banco: **¿de verdad necesitás un motor de búsqueda dedicado?** Muchas veces **no**, y montar un Elasticsearch (un sistema distribuido más para operar, sincronizar y pagar) es sobre-ingeniería.

**Te alcanza con la base de datos cuando:**
- **Prefijos / autocompletar simple:** `WHERE nombre LIKE 'lap%'` (sin comodín inicial) **sí** usa un índice B-tree; o `pg_trgm` para fuzzy/substring. (Y si es typeahead a escala de buscador, es el **trie del Caso 4**, no full-text.)
- **Full-text moderado:** **Postgres full-text search** (`tsvector`/`tsquery` + índice **GIN**) te da índice invertido, stemming por idioma y ranking (`ts_rank`) **dentro de tu base** — sin un sistema aparte ni sincronización. Aguanta muchísimo (cientos de miles a millones de docs, según hardware y query).

**Necesitás Elasticsearch/OpenSearch cuando la búsqueda *es* el producto a escala:**
- **Decenas/cientos de millones de documentos** con latencia baja y alta concurrencia de búsqueda.
- **Relevancia afinada** (boosts por campo, sinónimos, BM25 tuneado, learning-to-rank), **faceting/agregaciones** ("filtrar por marca, color, rango de precio" con conteos), **fuzzy/typo-tolerance** serio.
- Cuando ya **medís** que el full-text de Postgres no da (latencia, o compite con la carga transaccional de la misma base).

El costo de Elasticsearch que se suele subestimar: es **otra fuente de verdad que hay que mantener sincronizada** con tu base (vía CDC/[Streams](streams.md) o reindex), con su propio lag, su operación de cluster y su consistencia eventual respecto del dato canónico. No es "gratis y mejor": es una pieza distribuida más.

> **El cierre de criterio — no sobre-diseñar.** ¿Buscador sobre 50.000 productos? `tsvector` + GIN en la Postgres que ya tenés, y listo. ¿Marketplace global con cientos de millones de ítems, faceting y relevancia como ventaja competitiva? Ahí Elasticsearch gana lo que cuesta. La pregunta senior no es "¿qué motor uso?" sino "**¿la búsqueda relevante a escala es parte central del producto, o es un `WHERE` con onda?**".

La frase mental de cierre: **antes de Elasticsearch, probá Postgres: `LIKE 'x%'` para prefijos, `tsvector`+GIN para full-text moderado — sin otro sistema que sincronizar. Elasticsearch es para cuando la búsqueda relevante a escala (millones de docs, faceting, relevancia afinada) ES el producto. Su costo oculto: otra fuente de verdad que mantener en sync. No sobre-diseñes.**

**Ejercicios 7**
7.1 🔁 Nombrá dos formas de hacer búsqueda en Postgres sin un motor dedicado y para qué sirve cada una.
7.2 🧠 ¿Cuál es el costo "oculto" de meter Elasticsearch que la gente subestima al compararlo con la búsqueda en la base?
7.3 🧠 ¿Qué preguntas te hacés para decidir entre el full-text de Postgres y un Elasticsearch? Dá dos señales claras de que ya necesitás el motor dedicado.

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Un **índice invertido** es un mapa **`término → lista de documentos que lo contienen`**; esa lista es la **posting list** (ordenada por doc id). Buscar = **intersecar/unir** posting lists, no escanear documentos. `LIKE '%x%'` no escala porque el **comodín inicial** impide usar cualquier índice → la base hace un **full scan** comparando el patrón contra cada fila (`O(n)` sobre todo el corpus).

**1.2** "Ambos" es una **intersección (AND)**; "cualquiera" es una **unión (OR)**. Conviene **empezar por la lista más corta** (`"ferrari"`, 200 docs) y, para cada uno de esos 200, chequear si está en la de `"rojo"`: el trabajo queda acotado por la lista chica (≤200 comprobaciones) en vez de recorrer los 10.000 de `"rojo"`. El resultado de un AND nunca puede ser mayor que la posting list más corta, así que esa lista **acota** el costo.

**1.3** Falta **rankear por relevancia** (módulo 3). El conjunto pelado puede tener decenas de miles de documentos que matchean, todos "iguales" para el booleano; devolverlos sin orden (o por fecha/id) entierra lo relevante. El usuario mira las primeras posiciones, así que **el orden ES la búsqueda**: necesitás un score (TF-IDF/BM25 + señales de negocio) que ponga arriba lo más pertinente.

**2.1** **Tokenización** (partir en tokens), **normalización** (minúsculas, sin acentos), **stop words** (descartar palabras vacías frecuentes), **stemming/lematización** (reducir a la raíz). **Stemming** corta heurísticamente sin diccionario (`"corriendo"`→`"corr"`), puede producir raíces que no son palabras reales; **lematización** usa diccionario/morfología para llegar al **lema** correcto (`"mejor"`→`"bueno"`), más preciso pero más caro.

**2.2** Que **rompiste la regla de oro**: usaste **analyzers distintos** en index-time y query-time. Al indexar, `"zapato"` pasó por stemming y quedó como una raíz (p. ej. `"zapat"`); al consultar `"zapatos"` sin stemming, el término de la query es `"zapatos"` (o `"zapato"` solo en minúsculas), que **no coincide** con `"zapat"` en el índice → no hay match aunque el documento "obviamente" tenga la palabra. Solución: **el mismo analyzer en ambos lados**.

**2.3** Porque lo que se compara son los **términos analizados**, no el texto original. Si en **ambos** lados pasás a minúsculas y quitás acentos, `"Café"` (documento) y `"cafe"` (query) **colapsan al mismo término** `"cafe"` → matchean. Si lo hicieras en **uno solo** (p. ej. solo al indexar), el índice tendría `"cafe"` pero la query buscaría `"Café"` con acento/mayúscula → términos distintos, **no matchea**. La normalización solo sirve si es **simétrica**.

**3.1** **TF** (term frequency) captura **cuán presente** está el término en el documento (más apariciones → el doc trata más de eso). **IDF** (inverse document frequency) captura cuán **raro** es el término en el corpus. Un término raro pesa más porque **discrimina**: aparecer en pocos documentos significa que esos pocos son específicamente sobre eso; un término que está en todos (`"el"`) no distingue nada, así que su aporte al ranking debe ser ≈0.

**3.2** (1) **Saturación de TF:** en TF-IDF crudo el score crece **lineal** con las apariciones (200 ocurrencias ≈ 100× el peso de 2), lo cual es irreal; BM25 hace que **sature** (se aplane) con el parámetro **`k1`**. (2) **Normalización por longitud:** un documento largo acumula TF alto solo por ser largo; BM25 penaliza la longitud relativa a la media (`avgdl`) con el parámetro **`b`**. Así gana el documento que es **específicamente** sobre el término, no el más largo ni el que lo repite mil veces.

**3.3**
```ts
type DocId = number;

interface Posting {
  docId: DocId;
  tf: number; // veces que el término aparece en ese documento
}

class BM25Index {
  private readonly index = new Map<string, Posting[]>(); // término → posting list
  private readonly docLen = new Map<DocId, number>();     // longitud de cada doc
  private totalLen = 0;

  constructor(
    private readonly k1: number = 1.2,
    private readonly b: number = 0.75,
  ) {}

  addDoc(docId: DocId, tokens: readonly string[]): void {
    this.docLen.set(docId, tokens.length);
    this.totalLen += tokens.length;

    const freqs = new Map<string, number>();
    for (const term of tokens) {
      freqs.set(term, (freqs.get(term) ?? 0) + 1);
    }
    for (const [term, tf] of freqs) {
      const postings = this.index.get(term) ?? [];
      postings.push({ docId, tf });
      this.index.set(term, postings);
    }
  }

  private idf(term: string): number {
    const N = this.docLen.size;
    const n = this.index.get(term)?.length ?? 0; // docs que contienen el término
    // IDF de BM25: cuanto más raro el término, más alto.
    return Math.log(1 + (N - n + 0.5) / (n + 0.5));
  }

  search(queryTokens: readonly string[]): Array<{ docId: DocId; score: number }> {
    const avgdl = this.totalLen / Math.max(1, this.docLen.size);
    const scores = new Map<DocId, number>();

    for (const term of queryTokens) {
      const postings = this.index.get(term);
      if (postings === undefined) continue; // término no está en el corpus
      const idf = this.idf(term);
      for (const { docId, tf } of postings) {
        const len = this.docLen.get(docId) ?? 0;
        // BM25: IDF · TF saturado y normalizado por longitud
        const denom = tf + this.k1 * (1 - this.b + this.b * (len / avgdl));
        const termScore = idf * ((tf * (this.k1 + 1)) / denom);
        scores.set(docId, (scores.get(docId) ?? 0) + termScore);
      }
    }

    return [...scores.entries()]
      .map(([docId, score]) => ({ docId, score }))
      .sort((a, b) => b.score - a.score); // mayor score primero
  }
}
```
Idea: `addDoc` cuenta el TF por término y lo agrega a la posting list (más la longitud del doc para `avgdl`). `idf` usa la fórmula de BM25 (término raro → más peso). `search` suma, por cada término de la consulta, el `IDF · TF-saturado-y-normalizado`, y ordena por score. Compila en `--strict` (y `--noUncheckedIndexedAccess`: por eso los `?? 0` / `=== undefined` tras `Map.get`, que puede devolver `undefined`; el `for...of` sobre `postings` sí entrega `Posting`, no `Posting | undefined`). En un motor real esto vive **por segmento y por shard** (módulos 4-5), con las posting lists comprimidas en disco y el IDF idealmente global.

**4.1** Un **segmento** es un **mini índice invertido completo e inmutable** en disco; se corresponde con la **SSTable** del LSM-tree. Al **borrar** un documento, el segmento (inmutable) no se edita: se marca el doc con un **tombstone** (un bitset de docs vivos), y el espacio se recupera más tarde en el merge — igual que el tombstone del LSM.

**4.2** Sale de que indexar un doc **no modifica** un segmento (son inmutables): el doc se bufferiza y solo se vuelve **buscable cuando se escribe como segmento nuevo en el refresh** (~1s por default en Elasticsearch). Entre la escritura y el refresh, el doc existe pero no aparece en búsquedas → **near-real-time**, no real-time. Bajando el refresh interval **ganás** frescura (el doc aparece antes) pero **perdés** eficiencia: más segmentos chicos, más merges, más CPU/IO. Es el mismo trade-off que afinar un LSM.

**4.3** Porque el índice del shard está en **varios segmentos** (cada refresh crea uno; los merges los consolidan pero siempre hay más de uno), y una búsqueda debe consultar **todos** los segmentos y combinar resultados — igual que una lectura de LSM toca la memtable y varias SSTables. Eso es **read amplification**. El proceso que la mantiene acotada es el **merge** de segmentos (la **compaction** del LSM): fusiona segmentos chicos en grandes y descarta los docs borrados, reduciendo cuántos hay que abrir por búsqueda.

**5.1** Un **shard** es un **índice Lucene completo e independiente** (con sus propios segmentos) que contiene una **parte** de los documentos; el índice total se reparte entre shards porque no entra en una máquina. Una query va a **todos** los shards porque un documento relevante puede estar en **cualquiera** y no hay forma de saber en cuál de antemano (a diferencia de una query con la sharding key — acá no la tenés). Las **réplicas** (copias de cada shard) dan **tolerancia a fallos** (si cae el primario, la réplica sirve) y **más throughput de lectura** (las búsquedas se reparten entre primario y réplicas).

**5.2** **Query-then-fetch:** en la fase **query**, el coordinador pide a un shard de cada partición solo los **ids + scores** de sus top-k locales (barato de transmitir); mergea y obtiene el **top-k global**. Recién en la fase **fetch** pide los **documentos completos** de esos k al shard que los tiene. Se hace así para **no mover documentos enteros inútilmente**: si pidieras los docs completos en la primera fase, transmitirías `k × (nº shards)` documentos grandes para tirar casi todos al mergear. Trayendo solo ids+scores primero, movés los cuerpos completos **solo de los k que realmente entran** al resultado.

**5.3** Porque con scatter-gather, para devolver bien la página que arranca en `from=10000`, **cada shard** debe devolver `from+size` (≈10.010) resultados al coordinador —no puede saber cuáles de sus locales entran en el top global sin mandar todos los candidatos hasta esa profundidad— y el coordinador debe ordenar `(from+size) × nº_shards` ítems. El costo **crece con la profundidad**, multiplicado por shards → memoria/CPU que puede tumbar el cluster (por eso Elasticsearch limita `from+size` a ≈10.000 ⚠️). Lo reemplaza la **paginación por cursor** (`search_after` / point-in-time): igual que el **keyset/cursor de [api-design](api-design.md) M5**, pasás "el último visto" y traés los siguientes, costo **constante** por página sin importar la profundidad.

**6.1** Porque un **buen ranking** necesita estadísticas **globales** (IDF, señales de popularidad/recencia) y cómputo, y recalcular eso sobre un índice que **muta constantemente** (alta frescura) es caro; al revés, priorizar frescura absoluta deja poco margen para rankear bien en tiempo real. La estrategia general: **separar el camino rápido del caro** (idea Lambda/Kappa de [Streams](streams.md)): **indexación near-real-time** para frescura + **ranking refinado** con stats actualizadas periódicamente o en una segunda pasada — respuesta fresca y aproximada ya, orden afinado después.

**6.2** Con **reindex contra un índice nuevo + swap por alias** (expand-contract de [Datos a escala](datos-escala.md) M9). Tu app no apunta al índice físico sino a un **alias** (`productos` → `productos_v1`). Construís `productos_v2` con el analyzer nuevo **en paralelo**, reindexando desde la fuente (con throttling para no competir con el tráfico), lo dejás alcanzar a v1, y al final **mueves el alias** `productos` de v1 a v2 de forma **atómica**. Las búsquedas nunca pegan a un índice a medio reindexar; el cambio es instantáneo y reversible (volvés el alias a v1 si algo sale mal). Nunca reindexás "en caliente" el que sirve, porque el índice invertido es inmutable de hecho (como sus segmentos).

**6.3** **Happy path:** texto y query pasan por el **mismo analyzer** → se consulta el **índice invertido sharded** → se rankea con **BM25 + señales de negocio** → scatter-gather devuelve el **top-k global**. **Modelo de falla:** reindex masivo que satura, shard caliente, **deep pagination** que explota, scores no comparables entre shards (IDF local), lag de freshness. **Recuperación:** **réplicas** (HA + throughput), **reindex incremental con swap por alias**, **límites de paginación** + cursor, refresh interval ajustado, y `dfs_query_then_fetch` si la exactitud del score lo amerita.

**7.1** (1) **`LIKE 'x%'`** (comodín solo al final) o `pg_trgm` — para **prefijos/substring/fuzzy** simple, usando índices de Postgres. (2) **Full-text search de Postgres** (`tsvector`/`tsquery` + índice **GIN**, con `ts_rank` para ranking) — para **búsqueda full-text moderada** con stemming por idioma, **dentro de tu propia base**, sin un sistema aparte.

**7.2** Que Elasticsearch es **otra fuente de verdad que hay que mantener sincronizada** con la base canónica (vía CDC/[Streams](streams.md) o reindex), con su **propio lag**, su **consistencia eventual** respecto del dato real, y su **operación de cluster** (shards, nodos, capacidad). La gente lo compara como "más rápido y con mejor relevancia" y olvida que sumás un **sistema distribuido más** que puede quedar desincronizado, fallar por su cuenta y que hay que operar — costo que la búsqueda en la propia base no tiene.

**7.3** Preguntas clave: *¿cuántos documentos y a qué QPS de búsqueda?*, *¿la relevancia/faceting es ventaja competitiva o es un filtro simple?*, *¿ya medí que el full-text de Postgres no da o que compite con mi carga transaccional?*. **Dos señales claras** de que necesitás el motor dedicado: (1) **escala** — decenas/cientos de millones de docs con alta concurrencia y latencia baja, donde Postgres ya no rinde; (2) **features de búsqueda** — necesitás **faceting/agregaciones con conteos**, relevancia afinada (boosts, sinónimos, BM25 tuneado, learning-to-rank) o **typo-tolerance** serio, que en Postgres es incómodo o inexistente. (Otra: la búsqueda **es** el producto, no un accesorio.)

---

> **Para seguir.** Las fuentes de verdad: la doc de **Lucene** (`lucene.apache.org`) para segmentos/scoring, la de **Elasticsearch/OpenSearch** (analyzers, mapping, `search_after`, `dfs_query_then_fetch`, merge policy) y, del lado relacional, **Postgres full-text search** (`tsvector`/`tsquery`/GIN) y `pg_trgm`. Re-verificá lo marcado con ⚠️ (BM25 como default desde Lucene 6 / ES 5 en 2016, refresh ≈1s, `max_result_window` ≈10.000): los defaults cambian entre versiones. Los puentes en el hub: [Datos a escala](datos-escala.md) M11-14 es el origen (LSM/SSTable = segmentos, y el trade-off freshness↔ranking que acá profundizamos); el [Caso 4 — typeahead](system-design-casos.md) es el problema **vecino pero distinto** (completar prefijos con un trie, no rankear documentos); [api-design](api-design.md) M5 es la paginación por cursor que resuelve el deep pagination; [Streams](streams.md) es el CDC que mantiene el índice en sync con la base; y [Replicación](replicacion.md) es la idea de réplicas aplicada a los shards. **Y hacia dónde va el tema:** la búsqueda moderna combina **BM25 (léxico)** con **embeddings (semántica)** en lo que se llama **búsqueda híbrida** — el estándar de facto en 2026 para que matchee tanto el término exacto como el significado. Esa otra mitad está en [Vector Databases y RAG](vector-dbs.md) y [RAG](rag.md): BM25 encuentra "la palabra", el vectorial encuentra "la idea", y fusionás ambos rankings.
