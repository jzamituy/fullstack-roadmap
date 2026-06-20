# Vector Databases: búsqueda semántica a escala (y RAG en Python)

**Cómo se guardan y se buscan vectores de verdad · FAISS, pgvector, Pinecone, Qdrant · stack Python + Voyage + Claude · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es el cuarto del **track Python AI**: asume [Python](python.md) (tipos, async, paquetes con `uv`) y [Prompt Engineering](prompt-engineering.md). Y asume que ya viste el módulo conceptual de [RAG](rag.md) del track general —los dos pipelines (ingest/query), chunking, reranking, evals—: **no lo repetimos**. Ahí el stack era pgvector + TypeScript y la tesis era "no necesitás una base aparte". Acá miramos el otro lado: **qué es realmente una base vectorial, cómo funciona por dentro (los algoritmos ANN), cuándo sí conviene una dedicada, y cómo se arma todo en Python**. El objetivo es doble: las **internas** (métricas de distancia, HNSW/IVF, cuantización) y el **criterio de arquitectura** (FAISS vs pgvector vs Pinecone vs Qdrant) que se evalúa en una entrevista de AI/ML Engineer.

**Lo que asumimos.** Python, el SDK de Anthropic, embeddings (qué son: vectores que capturan significado — lo viste en [LLMs](ia-llms.md) y [RAG](rag.md)), y nociones de RAG. NumPy básico ayuda.

**Para practicar.**

```bash
uv add faiss-cpu numpy voyageai anthropic
# opcionales según el módulo:  uv add qdrant-client pinecone
# export ANTHROPIC_API_KEY=...   export VOYAGE_API_KEY=...
```

> Nota sobre datos volátiles: las APIs de los clientes (FAISS, Pinecone, Qdrant, voyageai) y los IDs/dimensiones de modelos cambian seguido. Los ejemplos muestran la **forma** de cada API a mediados de 2026 (FAISS 1.x, Pinecone SDK v5/serverless, qdrant-client 1.x, `voyage-3.5` → 1024 dims). Verificá firmas exactas en la doc de cada uno, y los datos de modelos Claude con la skill `claude-api`, antes de implementar.

**Índice de módulos**
1. Qué es una vector database (y por qué a veces no la necesitás)
2. Búsqueda por similitud: por qué la fuerza bruta no escala y aparece ANN
3. Métricas de distancia y normalización (coseno, dot, L2)
4. ANN por dentro I: IVF (particionar el espacio)
5. ANN por dentro II: HNSW (el grafo navegable)
6. Cuantización: comprimir vectores para que entren en RAM (PQ, scalar, binary)
7. FAISS: la biblioteca in-process, hands-on en Python
8. El panorama: FAISS vs pgvector vs Pinecone vs Qdrant/Weaviate
9. Metadata filtering y multi-tenant en bases dedicadas
10. RAG en Python end-to-end: ingest + query con Voyage y Claude
11. El criterio de cierre: elegir la herramienta y el puente a agentes

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es una vector database (y por qué a veces no la necesitás)

**Teoría.** Una **vector database** es una base de datos especializada en una sola operación que las bases tradicionales hacen mal: **dado un vector de consulta, encontrar los vectores más parecidos entre millones**. Esa operación —*nearest neighbor search*, búsqueda de vecinos más cercanos— es el motor de la búsqueda semántica, del retrieval en [RAG](rag.md), de los sistemas de recomendación y de la deduplicación por similitud.

La pregunta que confunde a todos al principio: **¿qué tiene de especial? ¿no es solo guardar una lista de floats?** Guardar es trivial; el problema es **buscar rápido**. Una base relacional indexa con árboles B (un `WHERE precio > 100` salta directo a las filas que cumplen). Pero "¿cuáles de estos 10 millones de vectores de 1024 dimensiones están más cerca de *este*?" no es una condición de igualdad ni de rango sobre una columna: es una comparación geométrica contra **todos**. Un índice B-tree no sirve para eso. La vector database aporta **estructuras de índice distintas** (HNSW, IVF — módulos 4 y 5) diseñadas para esa pregunta.

Qué hace una vector DB, en concreto:

1. **Almacena** vectores (de dimensión fija) junto a un `id` y, normalmente, **metadata** (texto original, `tenant_id`, fecha…).
2. **Indexa** los vectores con una estructura ANN para búsqueda sub-lineal.
3. **Busca** los top-k más cercanos según una **métrica de distancia** (módulo 3), idealmente combinando con **filtros sobre la metadata** (módulo 9).

El abanico de "dónde viven los vectores" es más amplio de lo que parece, y elegir bien es el tema del módulo 8. Adelanto del espectro:

- **Una biblioteca en proceso** (FAISS): no hay servidor; el índice vive en la RAM de tu proceso Python. Rapidísimo, sin red, pero sin persistencia de metadata ni filtros ni multi-tenant de fábrica (módulo 7).
- **Una extensión de tu base relacional** (pgvector sobre Postgres): los vectores viven al lado de tu data, filtrás con `WHERE` (la tesis de [RAG](rag.md)).
- **Una base dedicada gestionada** (Pinecone): SaaS, te abstrae la operación, namespaces y filtros incluidos, pagás por uso.
- **Una base dedicada self-hosted** (Qdrant, Weaviate, Milvus): un servidor que corrés vos, con filtrado, payload y búsqueda híbrida nativos.

Y la advertencia que recorre todo el módulo (es la misma de [RAG](rag.md) y de [liderazgo técnico](liderazgo.md) — *escalá la herramienta al problema*): **una base vectorial dedicada es infraestructura, y la mayoría de los proyectos no la necesitan el día uno.** Si tenés decenas de miles de chunks y ya usás Postgres, pgvector alcanza y te ahorra un sistema entero que sincronizar. La base dedicada gana cuando aparecen **escala** (decenas de millones de vectores), **features vector-nativas** (cuantización agresiva, híbrido sparse-dense) o **carga de búsqueda** que tu Postgres no banca. La frase mental: **una vector DB resuelve "buscar el vecino más cercano entre muchos, rápido"; antes de montar una dedicada, preguntate si de verdad tenés "muchos" y "rápido" como problema.**

**Ejercicios 1**
1.1 ¿Qué operación hace que una vector DB sea especial, y por qué un índice B-tree relacional no sirve para resolverla?
1.2 Nombrá las tres cosas que hace una vector database (almacenar, indexar, buscar) y qué agrega la metadata.
1.3 Describí el espectro de "dónde viven los vectores" (biblioteca in-process, extensión relacional, dedicada gestionada, dedicada self-hosted) con un ejemplo de cada una.
1.4 ¿Cuál es el criterio para NO montar una base vectorial dedicada el día uno? ¿Qué tres cosas justifican subir a una?

---

## Módulo 2 — Búsqueda por similitud: por qué la fuerza bruta no escala y aparece ANN

**Teoría.** La forma exacta y obvia de encontrar los vecinos más cercanos es la **fuerza bruta** (en la jerga, *flat* o *exhaustive search*): comparás el vector de consulta contra **todos** los vectores guardados, calculás la distancia a cada uno, ordenás y te quedás con los k menores. Es **exacto** (devuelve los verdaderos k más cercanos) y trivial de implementar.

```python
import numpy as np

def knn_fuerza_bruta(query: np.ndarray, base: np.ndarray, k: int) -> np.ndarray:
    # base: (n, d) ya normalizada; query: (d,) normalizada → similitud coseno = producto interno
    sims = base @ query              # (n,) un producto punto por cada vector
    return np.argsort(-sims)[:k]     # índices de los k más similares
```

El costo es **O(n·d)** por consulta: con `n = 10M` vectores y `d = 1024`, son ~10.000 millones de multiplicaciones **por cada query**. Para un corpus chico (hasta ~10⁵–10⁶ con NumPy/BLAS vectorizado, o GPU) la fuerza bruta es perfectamente válida —y conviene, porque es exacta y sin parámetros que afinar—. El problema es la **latencia bajo escala y carga**: a millones de vectores y muchas queries por segundo, comparar contra todo no entra en el presupuesto de latencia de un request.

¿Por qué no usamos un árbol, como en datos de baja dimensión? Acá entra la **maldición de la dimensionalidad**: las estructuras de partición espacial exactas (k-d trees, ball trees) funcionan en pocas dimensiones (2, 3, 10), pero en cientos o miles de dimensiones **degeneran a fuerza bruta** —terminan teniendo que visitar casi todos los nodos porque en alta dimensión "todos los puntos están más o menos a la misma distancia" y la geometría deja de ayudar a podar—. No existe un índice **exacto** y **eficiente** para vecinos más cercanos en alta dimensión.

La salida es renunciar a lo exacto: **Approximate Nearest Neighbor (ANN)**. Un índice ANN devuelve vectores que están *casi seguro* entre los más cercanos, **la mayoría de las veces**, a cambio de ser órdenes de magnitud más rápido. Aparece la métrica que gobierna todo el resto del módulo: el **recall** —de los k vecinos verdaderos, ¿qué fracción devolvió el índice ANN?—. Un recall de 0.95 significa que, en promedio, 19 de los 20 vecinos reales aparecen. El trade-off central de toda vector DB es **recall ↔ latencia ↔ memoria**: podés tener dos de los tres baratos, nunca los tres a la vez. Cada algoritmo (IVF, HNSW) y cada perilla (`nprobe`, `efSearch`) es una forma de moverte sobre ese triángulo.

La frase mental: **la fuerza bruta es exacta pero O(n·d); en alta dimensión no hay índice exacto y eficiente, así que aceptamos vecinos aproximados (ANN) y medimos cuánto perdemos con el recall.**

**Ejercicios 2**
2.1 Describí la búsqueda por fuerza bruta y su costo. ¿Cuándo es la opción correcta a pesar de no escalar?
2.2 ¿Qué es la "maldición de la dimensionalidad" y por qué hace que un k-d tree no sirva para embeddings de 1024 dimensiones?
2.3 ¿Qué significa ANN y qué se sacrifica respecto a la fuerza bruta? Definí *recall* en este contexto.
2.4 Enunciá el trade-off de tres vías de una vector DB. ¿Qué representan las perillas como `nprobe`/`efSearch` dentro de ese trade-off?

---

## Módulo 3 — Métricas de distancia y normalización (coseno, dot, L2)

**Teoría.** "Más cercano" exige definir **cercanía**. Hay tres métricas que vas a ver en toda vector DB, y elegir la equivocada (o mezclar normalización con métrica) es un error sutil que arruina el recall sin dar error.

- **Distancia euclídea (L2)**: la distancia "de regla" entre dos puntos, `√Σ(aᵢ-bᵢ)²`. Menor = más cerca. Sensible a la **magnitud** del vector, no solo a su dirección.
- **Producto interno / dot product (IP)**: `Σ aᵢ·bᵢ`. Mayor = más similar. Mezcla **dirección y magnitud**: un vector más "largo" puntúa más alto aunque apunte al mismo lado.
- **Similitud coseno**: el coseno del ángulo entre los vectores, `(a·b)/(‖a‖‖b‖)`. Mide solo **dirección**, ignora la magnitud. Es la métrica por defecto para embeddings de texto, porque lo que importa es el *significado* (la dirección), no cuán "largo" salió el vector.

El hecho que conecta las tres y que tenés que saber: **sobre vectores normalizados (norma 1), la similitud coseno es exactamente el producto interno, y el ranking por L2 es el inverso del ranking por coseno.** Es decir: si normalizás todos los vectores a norma 1 una sola vez (en el ingest), podés usar **producto interno** —que es más barato de computar, sin la división— y obtenés ranking coseno gratis.

```python
import numpy as np

def normalizar(v: np.ndarray) -> np.ndarray:
    # divide cada vector por su norma L2 → norma 1; tras esto, dot == coseno
    return v / np.linalg.norm(v, axis=-1, keepdims=True)
```

Por eso el patrón idiomático en FAISS (módulo 7) es **normalizar los vectores y usar `IndexFlatIP`** (inner product): es la forma canónica de hacer "búsqueda coseno". La regla operativa: **elegí la métrica del índice acorde a cómo embeddeás, y sé consistente entre ingest y query.** Si normalizás en el ingest, normalizá también la consulta; si el índice está configurado para IP pero comparás vectores sin normalizar esperando coseno, el ranking sale mal y no hay excepción que te avise —solo recall malo silencioso—.

Un detalle que recomiendan los proveedores de embeddings (Voyage entre ellos): consultá su doc por la métrica esperada. La mayoría de los embeddings de texto modernos vienen pensados para **coseno** (o, equivalentemente, IP sobre vectores normalizados). Algunos ya salen normalizados; otros no. No lo asumas: verificalo.

La frase mental: **coseno mide dirección (significado), L2 mide distancia con magnitud, IP mezcla ambas; normalizá una vez y coseno = producto interno —y sé fanático de la consistencia entre cómo indexás y cómo consultás—.**

**Ejercicios 3**
3.1 Diferenciá L2, producto interno y coseno. ¿Cuál es la métrica por defecto para embeddings de texto y por qué?
3.2 ¿Qué relación hay entre coseno y producto interno cuando los vectores están normalizados, y qué ventaja práctica te da eso?
3.3 ¿Por qué un índice configurado para IP con vectores sin normalizar (esperando coseno) es un bug peligroso? (pista: ¿qué síntoma da?)
3.4 ¿Por qué hay que verificar la métrica esperada del modelo de embeddings en vez de asumirla?

---

## Módulo 4 — ANN por dentro I: IVF (particionar el espacio)

**Teoría.** El primer gran enfoque ANN es **IVF** (*Inverted File Index*), y la intuición es la de un bibliotecario: en vez de revisar todos los libros, primero decidís **en qué estante** buscar y solo mirás ese estante.

Cómo funciona, en dos fases:

1. **Entrenamiento (train).** Se corre **k-means** sobre una muestra representativa de los vectores para encontrar `nlist` **centroides** (los "estantes", técnicamente *celdas de Voronoi*). Cada centroide representa una región del espacio. Esta fase es obligatoria y **necesita datos representativos**: un IVF entrenado sobre pocos vectores o sobre datos no representativos parte el espacio mal y arruina el recall.
2. **Add.** Cada vector se asigna a la celda de su centroide más cercano (queda "archivado en ese estante").

En la **búsqueda**, en lugar de comparar la query contra los `n` vectores, comparás:
- primero contra los `nlist` centroides (barato) para elegir las celdas candidatas,
- y después, solo contra los vectores de las **`nprobe` celdas más cercanas** (no contra todos).

Las dos perillas y su trade-off:

- **`nlist`** (cuántas celdas): se fija al construir. Más celdas = celdas más chicas = menos vectores por celda = búsqueda más rápida, pero más riesgo de que el vecino verdadero quede en una celda vecina que no visitaste. Una heurística común es `nlist ≈ √n`.
- **`nprobe`** (cuántas celdas visitar en cada query): se ajusta **en tiempo de consulta**. `nprobe = 1` solo mira la celda del centroide más cercano (rápido, recall bajo); subirlo visita más celdas vecinas (más recall, más latencia). En el extremo `nprobe = nlist` equivale a fuerza bruta (recall perfecto, sin ganancia de velocidad).

El error de recall característico de IVF: el **problema del borde de celda**. Si el vector de consulta cae cerca del límite entre dos celdas, su vecino más cercano real puede estar del otro lado del borde, en una celda que no visitaste con `nprobe` bajo. Subir `nprobe` lo mitiga (visitás también las celdas vecinas), a costa de latencia. Por eso `nprobe` es **la** perilla recall↔latencia de IVF, y se calibra midiendo recall contra un set de prueba (como en [evals](evals.md)), no a ojo.

IVF rara vez se usa "pelado" (`IVFFlat`, que guarda los vectores completos en cada celda): su verdadero poder aparece combinado con **cuantización** (módulo 6) para comprimir lo que hay dentro de cada celda. La frase mental: **IVF achica el problema con un primer filtro grueso —mirá solo los estantes correctos—; `nlist` define cuántos estantes hay y `nprobe` cuántos abrís en cada búsqueda, y ahí vive el trade-off recall↔latencia.**

**Ejercicios 4**
4.1 Explicá la intuición de IVF (el bibliotecario) y sus dos fases de construcción (train y add). ¿Por qué el train necesita datos representativos?
4.2 Diferenciá `nlist` de `nprobe`: ¿cuál se fija al construir y cuál en cada query? ¿Qué le pasa al recall y a la latencia al subir cada uno?
4.3 ¿Qué es el "problema del borde de celda" y cómo lo mitigás?
4.4 ¿Qué pasa con `nprobe = nlist`, y por qué eso muestra que IVF es un punto en el trade-off, no magia?

---

## Módulo 5 — ANN por dentro II: HNSW (el grafo navegable)

**Teoría.** El segundo gran enfoque, y el **default de hecho** en casi toda vector DB moderna (pgvector, Qdrant, Weaviate, FAISS, Pinecone por debajo), es **HNSW** (*Hierarchical Navigable Small World*). En vez de particionar el espacio en celdas, construye un **grafo** donde cada vector es un nodo conectado a sus vecinos más cercanos, y buscás **navegando** ese grafo.

La intuición es la de un mapa de transporte con **capas**, como un skip-list en 2D:

- **Capas altas**: pocos nodos, conexiones de **largo alcance** (como vuelos entre ciudades). Te acercan rápido a la región correcta.
- **Capas bajas**: todos los nodos, conexiones **locales** (como calles). Refinan hasta el vecino exacto.

La búsqueda **entra por la capa más alta**, avanza *greedy* hacia el nodo más cercano a la query, **baja** una capa y repite, hasta la capa 0 donde encuentra los k vecinos. Es búsqueda **logarítmica** en la práctica: saltás de lejos a cerca sin visitar casi nadie. Esa es la razón de su velocidad y su altísimo recall.

Las perillas de HNSW (las vas a ver con estos nombres en casi toda base):

- **`M`** (build): cuántos vecinos por nodo (grado del grafo). Más `M` = grafo más conectado = mejor recall y navegación, pero **más memoria** (cada conexión se guarda) y build más lento. Típico: 16–64.
- **`efConstruction`** (build): cuán exhaustiva es la búsqueda de vecinos al insertar cada nodo. Más alto = grafo de mejor calidad = mejor recall, build más lento. Típico: 100–400.
- **`efSearch`** (query): cuántos candidatos mantiene "en juego" la búsqueda en cada query. Es la perilla recall↔latencia **en tiempo de consulta** (el análogo del `nprobe` de IVF): subirlo explora más nodos → más recall, más latencia.

El costo de HNSW, y su gran desventaja, es la **memoria**: el grafo completo (vectores + todas las aristas) vive **en RAM**, y con `M` alto ese overhead es grande. Segundo punto que muerde en producción: **HNSW es malo para borrados**. El grafo se construye incrementalmente y quitar un nodo sin degradar la navegabilidad es complicado; muchas implementaciones (FAISS entre ellas) **no soportan `remove`** sobre HNSW —marcás como borrado y reconstruís periódicamente—. Esto conecta con el "día 2" de [RAG](rag.md) (documentos que se editan y borran): si tu corpus cambia mucho, HNSW puro te obliga a reindexar seguido.

**HNSW vs IVF, el resumen de criterio:**

| | HNSW | IVF (+ PQ) |
|---|---|---|
| Recall a igual latencia | mejor | bueno |
| Velocidad de lectura | muy alta | alta |
| Memoria | **alta** (grafo en RAM) | baja (sobre todo con PQ) |
| Build | más lento, sin train | rápido, **requiere train** |
| Borrados | malos / no soportados | mejores |
| Brilla en | recall alto, corpus que entra en RAM | escala masiva, memoria acotada |

**Default razonable: HNSW**, salvo que la memoria o la escala (cientos de millones) lo vuelvan caro, donde IVF+PQ (módulo 6) es el camino. La frase mental: **HNSW navega un grafo multi-capa de "lejos a cerca" —rapidísimo y de alto recall—, pero paga en RAM y sufre con los borrados; `M`/`efConstruction` son calidad de build y `efSearch` es la perilla recall↔latencia de cada query.**

**Ejercicios 5**
5.1 Explicá la intuición de las capas de HNSW (las conexiones de largo alcance arriba y locales abajo) y cómo procede la búsqueda.
5.2 Diferenciá `M`, `efConstruction` y `efSearch`: ¿cuáles son de build y cuál de query? ¿Cuál es el análogo del `nprobe` de IVF?
5.3 ¿Cuáles son las dos grandes desventajas de HNSW, y con qué problema del "día 2" de RAG conecta la segunda?
5.4 Dado un corpus de 300 millones de vectores con presupuesto de RAM ajustado, ¿elegirías HNSW o IVF+PQ? Justificá con la tabla.

---

## Módulo 6 — Cuantización: comprimir vectores para que entren en RAM (PQ, scalar, binary)

**Teoría.** Un embedding de 1024 dimensiones en `float32` ocupa `1024 × 4 = 4096 bytes`. Diez millones de vectores son ~40 GB **solo de datos crudos**, sin contar el overhead del índice. La memoria es el cuello de botella real a escala, y la **cuantización** es la familia de técnicas para comprimir vectores aceptando una pérdida controlada de precisión. Es lo que hace viable buscar entre miles de millones de vectores.

Tres niveles, de menos a más agresivo:

- **Scalar Quantization (SQ)**: en vez de `float32` (4 bytes) por componente, usás `int8` (1 byte) mapeando el rango de valores a 256 niveles. **4× menos memoria**, pérdida de precisión baja. Es la compresión "fácil", casi siempre vale la pena.

- **Product Quantization (PQ)**: la técnica clave para escala masiva. La idea: partís el vector de `d` dimensiones en `m` **subvectores**, y para cada subespacio entrenás (k-means) un **codebook** de 256 centroides. Cada subvector se reemplaza por **el id de 1 byte** de su centroide más cercano. Un vector de 1024 dims con `m = 64` subvectores pasa de 4096 bytes a **64 bytes**: **64× de compresión**. Las distancias se aproximan con tablas precalculadas entre la query y los centroides (rapidísimo). El costo es **pérdida de precisión** (el vector reconstruido es aproximado), que se compensa con la etapa de IVF que lo precede.

- **Binary Quantization (BQ)**: el extremo —1 bit por dimensión (signo del valor)—, distancia por Hamming (XOR + popcount, vertiginoso). **32× de compresión** y búsqueda ultrarrápida, pero la mayor pérdida; funciona razonable solo con embeddings de dimensión muy alta y casi siempre con un **rescoring**: traés muchos candidatos con la versión binaria (barato) y los reordenás con los vectores full-precision (caro pero sobre pocos) —el mismo embudo recall→precision del reranking de [RAG](rag.md)—.

El combo canónico a escala masiva es **IVFPQ**: IVF para mirar solo unas pocas celdas (módulo 4) **y** PQ para que los vectores dentro de cada celda ocupen 64× menos. Así es como los índices de miles de millones de vectores entran en una sola máquina. El trade-off vuelve a ser el triángulo del módulo 2, ahora con la memoria como protagonista: **cuantizar baja memoria y sube velocidad, a costa de recall** —que recuperás parcialmente con `nprobe` más alto y/o rescoring full-precision—.

La regla de criterio: **no cuantices hasta que la memoria sea un problema real.** Para un corpus que entra cómodo en RAM, HNSW con vectores full-precision (o SQ) te da el mejor recall sin complicarte. PQ/IVFPQ son la respuesta a "no me entran los vectores en memoria", no un default. La frase mental: **la cuantización cambia bytes por precisión —SQ 4×, PQ hasta 64×, BQ 32× con rescoring—; es la palanca de la escala masiva (IVFPQ), no algo que prendas porque sí.**

**Ejercicios 6**
6.1 ¿Por qué la memoria es el cuello de botella a escala? Hacé la cuenta de cuánto ocupan 10M de vectores de 1024 dims en float32.
6.2 Explicá Product Quantization: subvectores, codebooks, cuántos bytes por vector con `m=64`, y qué se sacrifica.
6.3 ¿Qué es IVFPQ y por qué la combinación es la respuesta a la escala masiva?
6.4 ¿Qué es el rescoring full-precision y con qué patrón de RAG se parece? ¿Cuándo NO conviene cuantizar?

---

## Módulo 7 — FAISS: la biblioteca in-process, hands-on en Python

**Teoría.** **FAISS** (Facebook AI Similarity Search) es la biblioteca de referencia para búsqueda por similitud, y la mejor forma de **tocar** todo lo de los módulos anteriores. Lo central que tenés que entender de su naturaleza: **FAISS es una biblioteca, no un servidor.** El índice vive **en la memoria de tu proceso Python**; no hay red, no hay daemon, no hay tabla. Eso lo hace velocísimo y sin latencia de red —y a la vez le falta todo lo que un servidor te da gratis: persistencia, concurrencia, filtros por metadata, control de acceso—.

Los índices de FAISS son exactamente los algoritmos que ya viste. El patrón en Python (vectores como `np.float32`, shape `(n, d)`):

```python
import faiss
import numpy as np

d = 1024  # dimensión del modelo de embeddings (voyage-3.5)

# --- Flat: fuerza bruta exacta (módulo 2). Coseno = normalizar + IndexFlatIP (módulo 3) ---
index = faiss.IndexFlatIP(d)
faiss.normalize_L2(vectores)      # in-place; ahora producto interno == coseno
index.add(vectores)               # vectores: (n, d) float32
D, I = index.search(consultas, k=5)   # D: similitudes (n_q, k); I: índices (n_q, k)

# --- HNSW (módulo 5): sin train, alto recall, memoria alta ---
index = faiss.IndexHNSWFlat(d, 32)            # M = 32
index.hnsw.efConstruction = 200               # calidad de build
index.hnsw.efSearch = 64                       # perilla recall↔latencia de cada query
index.add(vectores)

# --- IVFPQ (módulos 4 y 6): escala masiva, requiere train ---
nlist, m, nbits = 4096, 64, 8                  # nlist celdas; PQ de m subvectores a 8 bits
quantizer = faiss.IndexFlatIP(d)
index = faiss.IndexIVFPQ(quantizer, d, nlist, m, nbits, faiss.METRIC_INNER_PRODUCT)
index.train(muestra_representativa)            # OBLIGATORIO: k-means de celdas + codebooks PQ
index.add(vectores)
index.nprobe = 16                              # cuántas celdas visitar por query
```

`index.search(consultas, k)` devuelve dos matrices: **`D`** (las distancias/similitudes) e **`I`** (los **índices posicionales** de los vectores en el orden en que los agregaste). Y ahí está el primer detalle que sorprende: **FAISS te devuelve enteros, no tus datos.** El índice no guarda el texto del chunk ni el `id` de tu base —solo el vector—. Sos vos quien mantiene un mapeo `posición → (id, texto, metadata)` aparte (una lista, un dict, una tabla SQL). Para usar tus propios ids en vez de las posiciones, envolvés con `IndexIDMap`:

```python
index = faiss.IndexIDMap(faiss.IndexFlatIP(d))
index.add_with_ids(vectores, ids)                       # ids: np.int64 (n,)
index.remove_ids(faiss.IDSelectorBatch(ids_a_borrar))   # borrado por id (en índices que lo soportan)
```

Las **limitaciones que definen cuándo NO usar FAISS solo** (y que son el puente al módulo 8):

- **Persistencia manual.** El índice está en RAM; si el proceso muere, se pierde. Lo serializás vos a disco y lo recargás:
  ```python
  faiss.write_index(index, "apuntes.faiss")
  index = faiss.read_index("apuntes.faiss")     # lo levantás en el arranque
  ```
- **Sin metadata ni filtros nativos.** No hay `WHERE tenant_id = 7`. El filtrado lo hacés vos por fuera (con un `IDSelector` o filtrando posiciones), lo cual es justo lo que un multi-tenant necesita y FAISS no te da cómodo (módulo 9).
- **Borrados pobres en grafos.** `remove_ids` no está soportado en HNSW (módulo 5); en IVF sí.
- **Un proceso, sin concurrencia gestionada.** No es una base multi-cliente; es una estructura en tu proceso.

Por eso FAISS brilla en un perfil muy concreto: **corpus grande, mayormente estático, read-heavy, donde querés el máximo control y velocidad sin operar un servidor** —y donde el filtrado/multi-tenant no es el centro—. Es también el motor que muchas bases dedicadas usan por debajo. La frase mental: **FAISS es ANN puro en tu proceso —rapidísimo y didáctico, los algoritmos al desnudo—; vos ponés persistencia, el mapeo a tus datos y los filtros, y por eso a partir de cierta complejidad operativa conviene una base dedicada.**

**Ejercicios 7**
7.1 ¿Por qué decimos que FAISS "es una biblioteca, no un servidor" y qué implica eso para persistencia y concurrencia?
7.2 ¿Qué devuelve `index.search(q, k)` en `D` e `I`? ¿Por qué tenés que mantener un mapeo aparte y qué resuelve `IndexIDMap`?
7.3 Mostrá (en código o en palabras) cómo armarías un índice coseno en FAISS y por qué se usa `normalize_L2` + `IndexFlatIP`.
7.4 Nombrá dos limitaciones de FAISS que empujan a una base dedicada, y describí el perfil de proyecto donde FAISS solo es la elección correcta.

---

## Módulo 8 — El panorama: FAISS vs pgvector vs Pinecone vs Qdrant/Weaviate

**Teoría.** El módulo de criterio puro —el que se evalúa en una entrevista—. No hay una "mejor vector DB"; hay la correcta para **tu** combinación de escala, frecuencia de cambios, necesidad de filtrado, apetito operativo y stack existente. Las cuatro categorías y su carácter:

**1. FAISS (biblioteca in-process).** Lo del módulo 7. *Elegí cuando*: corpus grande y estático, read-heavy, querés control y velocidad máximos sin operar un servidor, y el filtrado/multi-tenant no domina. *Evitá cuando*: el corpus cambia seguido, necesitás filtros por metadata, o varios servicios deben compartir el índice.

**2. pgvector (extensión de Postgres).** La tesis de [RAG](rag.md). *Elegí cuando*: **ya usás Postgres**, los vectores conviven con data relacional, y necesitás **filtrar por permisos/metadata en la misma query** (un `WHERE tenant_id = $1` transaccional, JOINs con tus tablas). Un solo sistema que operar, respaldar y asegurar. *Evitá cuando*: la escala (decenas/cientos de millones de vectores) o la carga de búsqueda superan lo que tu Postgres banca sin volverse el cuello de botella, o necesitás cuantización agresiva e híbrido sparse-dense nativos.

**3. Pinecone (dedicada gestionada / SaaS).** Un servicio: no operás nada, escala (serverless), tiene **namespaces** (aislamiento por tenant) y filtros por metadata de fábrica. *Elegí cuando*: querés escala y no querés operar infraestructura, y el costo por uso te cierra. *Evitá cuando*: necesitás que la data no salga de tu infra (compliance/datos sensibles), querés evitar el vendor lock-in, o el costo a tu volumen no cierra. La contracara del "no operás nada" es **no controlás la caja** y dependés de un tercero.

**4. Qdrant / Weaviate / Milvus (dedicada self-hosted, también con opción gestionada).** Servidores vector-nativos que corrés vos: filtrado rico por payload, búsqueda híbrida (sparse-dense), cuantización, sharding. *Elegí cuando*: necesitás features vector-nativas a escala y querés (o debés, por compliance) tener el sistema en tu infra. *Evitá cuando*: tu caso lo resuelve pgvector —entonces estás operando un sistema extra sin necesidad—.

La **matriz de decisión** mental:

| Tu situación | Elección razonable |
|---|---|
| Corpus chico/mediano, ya tengo Postgres, filtro por permisos | **pgvector** |
| Corpus grande, estático, read-heavy, sin filtros complejos, sin servidor | **FAISS** |
| Quiero escala sin operar infra y el SaaS me cierra | **Pinecone** |
| Necesito filtros ricos + híbrido + escala, en mi propia infra | **Qdrant/Weaviate/Milvus** |
| Decenas/cientos de millones de vectores, RAM acotada | dedicada con **IVFPQ** (módulo 6) |

El error de seniority invertida: **saltar a Pinecone/Qdrant "porque es lo serio" cuando pgvector resolvía el caso.** Operar una vector DB dedicada es trabajo real (deploy, backups, monitoreo, sincronizar permisos con tu base principal). La progresión sana, idéntica al criterio de todo el temario ([liderazgo](liderazgo.md), [RAG](rag.md), [Kubernetes](kubernetes.md)): **empezá por lo que ya tenés (pgvector o FAISS), y subí a una dedicada cuando un límite concreto —escala, features, carga— te lo exija y lo puedas nombrar.** La frase mental: **no existe "la mejor"; existe la que matchea tu escala, tus filtros, tu apetito operativo y tu stack —y la respuesta más común para empezar es la menos glamorosa—.**

**Ejercicios 8**
8.1 Para cada categoría (FAISS, pgvector, Pinecone, self-hosted) dá un caso donde es la elección correcta y uno donde la evitarías.
8.2 ¿Cuál es la principal ventaja de pgvector sobre una base dedicada, y cuándo deja de alcanzar?
8.3 ¿Qué ganás y qué perdés con una gestionada como Pinecone respecto a una self-hosted como Qdrant?
8.4 ¿Cuál es el "error de seniority invertida" y cuál es la progresión sana? Conectalo con un criterio del resto del temario.

---

## Módulo 9 — Metadata filtering y multi-tenant en bases dedicadas

**Teoría.** [RAG](rag.md) ya dejó clavado el principio de seguridad: **la búsqueda vectorial sola no respeta permisos** —trae los más *parecidos*, vengan de quien vengan—, y no filtrar es una fuga de datos servida por tu propio sistema. Ahí la solución era trivial porque con pgvector el filtro es un `WHERE` en la misma query. En una base **dedicada**, el filtrado por metadata es más sutil y vale entenderlo, porque es donde se cae la promesa de "es más escalable".

Cómo expresan el filtrado las bases dedicadas:

- **Pinecone**: cada vector lleva un dict de **metadata**, y la query acepta un `filter` (estilo Mongo: `{"tenant_id": {"$eq": 7}}`). Además **namespaces**: particiones lógicas dentro de un índice, ideales para aislar por tenant (cada tenant en su namespace, y la query apunta a uno).
- **Qdrant**: cada punto tiene un **payload** (JSON arbitrario), y filtrás con `Filter(must=[FieldCondition(key="tenant_id", match=MatchValue(value=7))])`. Conviene crear un **índice de payload** sobre los campos que filtrás.
- **Weaviate**: propiedades por objeto y un `where` en la query.

El problema técnico de fondo —el que separa a quien entendió de quien memorizó la API— es el **filtered ANN**, y es el mismo que [RAG](rag.md) marcó para pgvector pero acá más agudo. Hay dos estrategias ingenuas, ambas malas:

- **Post-filtering** (buscar y después filtrar): traés los top-k del índice ANN y descartás los que no pasan el filtro. **Rápido pero roto**: si el filtro es selectivo (este tenant tiene pocos vectores entre millones), de los k que trajo el índice puede que **ninguno o casi ninguno** pase el filtro, y te quedás con **menos de k** —o cero— resultados, aunque existían vectores válidos más allá del top-k.
- **Pre-filtering** (filtrar y después buscar exacto): te quedás con los vectores que pasan el filtro y hacés fuerza bruta sobre ellos. **Exacto pero lento** si el subconjunto sigue siendo grande, y no aprovecha el índice ANN.

Las bases dedicadas serias implementan **filtered search** que integra el filtro **dentro** de la navegación del índice (un HNSW que solo considera nodos que pasan el filtro mientras navega el grafo). Funciona bien hasta que el filtro es **muy selectivo**: ahí el grafo puede agotar su exploración antes de juntar k nodos válidos y degradar el recall —exactamente el síntoma que [RAG](rag.md) describió para el `WHERE tenant_id` muy selectivo sobre HNSW—. Mitigaciones: subir `efSearch`, **namespaces/colecciones por tenant** (en vez de filtrar, separás físicamente), o índices/particiones por tenant.

```python
# Qdrant: filtrado por payload integrado en la búsqueda (forma de la API; verificá versión)
from qdrant_client import QdrantClient
from qdrant_client.models import Filter, FieldCondition, MatchValue

hits = client.query_points(
    collection_name="apuntes",
    query=vector_pregunta,          # ya normalizado, dim 1024
    limit=5,
    query_filter=Filter(            # el filtro de seguridad va EN la búsqueda, no después
        must=[FieldCondition(key="tenant_id", match=MatchValue(value=7))]
    ),
).points
```

El principio de seguridad no cambia respecto a [RAG](rag.md), solo se vuelve más difícil de garantizar: **filtrá durante la búsqueda, nunca después; tratá el filtro de tenant como parte del esquema desde el día uno; y recordá que en una base dedicada los permisos viven en *otro* sistema que tu base de datos principal —tenés que mantenerlos sincronizados—**. Esto último es, de hecho, uno de los argumentos más fuertes a favor de pgvector cuando el multi-tenant manda: un solo lugar donde vive y se filtra la verdad. La frase mental: **el filtrado en bases dedicadas no es un `WHERE` gratis —es filtered ANN, con su propio trade-off de recall cuando el filtro es selectivo— y los permisos quedan en un sistema separado que hay que mantener en sync; filtrá durante la búsqueda y considerá namespaces por tenant.**

**Ejercicios 9**
9.1 ¿Cómo expresan el filtrado por metadata Pinecone (namespaces + filter) y Qdrant (payload + Filter)?
9.2 Explicá post-filtering vs pre-filtering y por qué ambos ingenuos son malos (uno por roto, otro por lento).
9.3 ¿Qué es filtered search integrado en el índice y cuándo degrada el recall? ¿Con qué punto de RAG sobre pgvector conecta?
9.4 ¿Por qué el multi-tenant es un argumento a favor de pgvector? ¿Qué problema de sincronización aparece con una base dedicada?

---

## Módulo 10 — RAG en Python end-to-end: ingest + query con Voyage y Claude

**Teoría.** Acá unimos todo en el stack del track: **Python + Voyage (embeddings) + Claude (generación)**, con FAISS como vector store local para que corra sin infra. La arquitectura es la de [RAG](rag.md) —dos pipelines, ingest y query— pero ahora en código Python concreto. No repetimos la teoría de chunking/reranking/grounding; los aplicamos.

**Embeddings con Voyage** (recordá de [RAG](rag.md): Anthropic no provee embeddings; el partner es Voyage, y se distingue `input_type` entre `document` y `query`):

```python
import voyageai

vo = voyageai.Client()  # lee VOYAGE_API_KEY

def embeddear(textos: list[str], tipo: str) -> list[list[float]]:
    # tipo: "document" en el ingest, "query" en la búsqueda (mejora el match)
    r = vo.embed(textos, model="voyage-3.5", input_type=tipo)
    return r.embeddings
```

**Pipeline de ingest** (offline): chunkear → embeddear como `document` → guardar en FAISS + un mapeo a los datos. Como FAISS solo guarda vectores e índices posicionales (módulo 7), mantenemos una lista paralela con el texto y la metadata:

```python
import faiss
import numpy as np

D = 1024
index = faiss.IndexIDMap(faiss.IndexFlatIP(D))   # coseno vía IP sobre vectores normalizados
catalogo: dict[int, dict] = {}                    # id -> {"texto":..., "fuente":..., "tenant_id":...}

def ingest(chunks: list[dict], desde_id: int = 0) -> None:
    # chunks: [{"texto":..., "fuente":..., "tenant_id":...}, ...]
    vecs = np.array(embeddear([c["texto"] for c in chunks], "document"), dtype="float32")
    faiss.normalize_L2(vecs)                       # normalizar → IP == coseno (módulo 3)
    ids = np.arange(desde_id, desde_id + len(chunks), dtype="int64")
    index.add_with_ids(vecs, ids)
    for i, c in zip(ids, chunks):
        catalogo[int(i)] = c
    faiss.write_index(index, "apuntes.faiss")      # persistencia manual (módulo 7)
```

**Pipeline de query** (online): embeddear la pregunta como `query` → buscar top-k → (filtrar permisos) → armar prompt grounded → generar con Claude.

```python
import anthropic

cliente = anthropic.Anthropic()

def query(pregunta: str, tenant_id: int, k: int = 5) -> str:
    qv = np.array(embeddear([pregunta], "query"), dtype="float32")
    faiss.normalize_L2(qv)                          # misma normalización que el ingest
    sims, ids = index.search(qv, k * 4)             # traemos de más para post-filtrar por tenant

    # filtrado por permisos (en FAISS toca hacerlo por fuera, módulo 9) — limitación, no virtud
    chunks = [catalogo[int(i)] for i in ids[0] if i != -1 and catalogo[int(i)]["tenant_id"] == tenant_id][:k]
    if not chunks:
        return "No tengo esa información."

    contexto = "\n\n".join(f"[{j+1}] {c['texto']}" for j, c in enumerate(chunks))
    msg = cliente.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        system="Respondés solo con el contexto provisto. Si no está, decís 'No tengo esa información'. Citá la fuente entre corchetes.",
        messages=[{
            "role": "user",
            "content": f"<contexto>\n{contexto}\n</contexto>\n\n<pregunta>\n{pregunta}\n</pregunta>",
        }],
    )
    return msg.content[0].text
```

Tres cosas para notar, que son criterio y no sintaxis:

1. **Mismo modelo de embeddings en ambos pipelines** ([RAG](rag.md) módulo 2): `voyage-3.5` en ingest y query. Si cambiás de modelo, reindexás todo.
2. **El filtrado por tenant en FAISS es manual y por post-filtering** —por eso traemos `k*4` y filtramos después, con todos los problemas del módulo 9 (podrías quedarte con menos de k)—. Esto es **exactamente** la limitación que en pgvector o Qdrant resolverías con un filtro integrado a la búsqueda. El ejemplo lo deja a propósito para que se vea el costo de elegir FAISS cuando hay multi-tenant.
3. **La generación es una llamada normal a la Messages API** ([Prompt Engineering](prompt-engineering.md)): grounding ("solo con el contexto"), permiso de no saber, etiquetas XML para separar contexto de pregunta. RAG no es una API nueva: es buen retrieval + un prompt bien armado.

Para producción, este mismo flujo se expone como endpoint con [FastAPI](fastapi.md) (el ingest va a una tarea de fondo; la query es el request), y se mide con [evals](evals.md) (recall@k, faithfulness). La frase mental: **el RAG en Python es Voyage para embeddear, un vector store para buscar, y Claude para generar —con el mismo modelo de embeddings en los dos pipelines y el filtrado de permisos donde tu vector store lo permita—.**

**Ejercicios 10**
10.1 ¿Por qué el ingest embeddéa con `input_type="document"` y la query con `"query"`, y por qué se usa el mismo modelo en ambos?
10.2 En el `ingest`, ¿por qué se mantiene el `catalogo` aparte y por qué `normalize_L2` + `IndexFlatIP`?
10.3 En el `query`, ¿por qué se traen `k*4` resultados y se filtra por `tenant_id` después? ¿Qué problema arrastra eso y cómo lo evitarías con pgvector/Qdrant?
10.4 ¿Qué tres ideas de Prompt Engineering aparecen en el prompt de generación?

---

## Módulo 11 — El criterio de cierre: elegir la herramienta y el puente a agentes

**Teoría.** El cierre, que ata las internas con la decisión de arquitectura. La pregunta de entrevista no es "¿sabés usar Pinecone?" sino "**¿por qué esta vector DB, este índice y estos parámetros para este caso?**". El árbol de decisión completo:

1. **¿Cuántos vectores y cuánto cambian?**
   - Miles a ~cientos de miles, ya tengo Postgres → **pgvector** (un sistema, filtros transaccionales).
   - Millones, mayormente estático, read-heavy, sin filtros complejos → **FAISS** (in-process, máximo control).
   - Decenas/cientos de millones → **dedicada con IVFPQ** (la cuantización es la que hace entrar los vectores en RAM).

2. **¿Necesito filtrar por permisos/metadata (multi-tenant)?** → pesa fuerte hacia **pgvector** (filtro = `WHERE` en la misma query, sin sincronizar permisos en dos sistemas) o, si la escala lo exige, una dedicada con **filtered search** y namespaces por tenant (módulo 9), sabiendo que el recall se degrada con filtros muy selectivos.

3. **¿Quiero operar infraestructura?** → no: **Pinecone** (gestionada) o pgvector (ya lo operás). Sí, y necesito features vector-nativas: **Qdrant/Weaviate/Milvus**.

4. **¿Qué índice y parámetros?**
   - Corpus que entra en RAM, prioridad recall → **HNSW** (`M`, `efConstruction` en build; `efSearch` la perilla de query).
   - Escala masiva, RAM acotada → **IVFPQ** (`nlist`, `m`/`nbits` en build; `nprobe` la perilla de query).
   - Y siempre: **métrica acorde al embedding** (coseno = normalizar + IP), consistente entre ingest y query (módulo 3).

5. **¿Es exacto crítico o tolera aproximación?** → casi siempre tolera ANN (recall 0.95+ es suficiente para RAG). Solo si necesitás exactitud absoluta y el corpus es chico, **Flat** (fuerza bruta).

Y el principio que es el alma de todo el temario, una vez más: **escalá la herramienta al problema.** El error caro no es elegir mal la base "óptima"; es montar infraestructura que no necesitabas —una Pinecone para 20.000 chunks que pgvector servía—, o quedarte con fuerza bruta cuando ya tenés cincuenta millones de vectores y la latencia se fue al techo. La decisión correcta se **mide** (recall@k y latencia p95 contra tu set de prueba, [evals](evals.md)), no se adivina por moda.

**El puente al resto del track Python AI.** Una vector DB rara vez es el producto final: es **un componente**. En [RAG](rag.md) es el retrieval. En un **agente** ([AI Agents](ai-agents-python.md)) la búsqueda semántica se vuelve **una herramienta más** que el modelo decide usar —"buscá en la base de conocimiento"— dentro de su loop de razonamiento y acción; el mismo retrieval que acá disparás vos, allá lo dispara el agente cuando lo necesita. Y todo —el retrieval, el agente— se valida con [evals](evals.md). El hilo del track: aprendiste a hablarle al modelo ([Prompt Engineering](prompt-engineering.md)), ahora a **darle tu información a escala** (este módulo), y lo que sigue es **dejarlo actuar** ([AI Agents](ai-agents-python.md)), servirlo por API ([FastAPI](fastapi.md)) y desplegarlo ([Deploy de IA](deploy-ai.md)).

La frase mental: **elegí la vector DB y el índice justificando escala, filtros, operación y métrica —no por moda—, medí la decisión con recall y latencia, y recordá que la base vectorial es un componente: en un agente, buscar es solo una herramienta más en el loop.**

**Ejercicios 11**
11.1 Recorré el árbol de decisión para: (a) 30.000 chunks de apuntes, ya usás Postgres, multi-tenant; (b) 80 millones de vectores de imágenes, RAM acotada, sin filtros; (c) corpus mediano estático, read-heavy, sin servidor, sin filtros.
11.2 ¿Por qué "elegir mal la base óptima" no es el error caro, y cuál sí lo es? Dá los dos extremos.
11.3 ¿Cómo se "mide" una decisión de vector DB en vez de adivinarla? (qué métricas, contra qué)
11.4 ¿Qué rol cumple la búsqueda vectorial dentro de un agente, y cómo cambia respecto a un RAG clásico quién dispara el retrieval?

---

## Proyecto integrador (capstone): un buscador semántico de tus apuntes, en Python

El ejercicio que cierra el módulo y que mostrás en una entrevista de AI/ML Engineer cuando te piden "¿armaste búsqueda vectorial?". Es el primo Python (y vector-store-agnóstico) del capstone de [RAG](rag.md). No te damos la solución; te damos los **criterios de aceptación**. Construilo en Python.

**Qué construir.** Una CLI con dos comandos sobre los `.md` de este repo:

1. **`ingest`** — chunkea los `.md` por estructura (encabezados) y tamaño en tokens (teoría en [RAG](rag.md) módulo 3), embeddéa cada chunk con Voyage como `"document"`, **normaliza** y los carga en un índice (empezá con FAISS `IndexIDMap(IndexFlatIP)`; opcional: HNSW). Mantené el mapeo `id → (texto, fuente, tenant_id)` y **persistí** el índice a disco.
2. **`search "<consulta>"`** — embeddéa la consulta como `"query"`, normaliza, recupera top-k, **filtra por `tenant_id`**, y muestra los chunks con su fuente y su score. *(Extensión: agregá un comando `ask` que pase esos chunks a Claude con un prompt grounded —el RAG completo del módulo 10—.)*

**Criterios de aceptación.**
- [ ] El código tipa con `mypy` razonablemente y corre con `uv run`.
- [ ] El índice se **persiste** y se recarga en el arranque (no se reembeddéa todo cada vez).
- [ ] Ingest y query usan **el mismo modelo de embeddings** y `input_type` correcto en cada uno (`document` vs `query`).
- [ ] Los vectores se **normalizan** y la métrica del índice es coherente (IP sobre normalizados = coseno).
- [ ] La búsqueda **filtra por `tenant_id`** (aunque sea post-filtering en FAISS) y nunca devuelve chunks de otro tenant.
- [ ] `search` devuelve resultados relevantes para 3 consultas cuya respuesta está en distintos módulos de estos apuntes, mostrando la fuente.

**Extensiones (suben el nivel).**
- Reemplazá FAISS por **Qdrant** (local con `:memory:` o Docker) y movés el filtro de `tenant_id` a un **payload filter integrado a la búsqueda** (módulo 9) — y notá la diferencia con el post-filtering manual de FAISS.
- Medí **recall@k**: armá un mini golden set de 10 consultas con el chunk esperado y compará HNSW (`efSearch` bajo vs alto) contra Flat (verdad exacta) — vas a *ver* el trade-off recall↔latencia del módulo 5.
- Cronometrá la **latencia por etapa** (embedding / búsqueda / generación) y mostrá el p95.

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 La búsqueda de vecinos más cercanos (nearest neighbor): dado un vector, encontrar los más
    parecidos entre millones. Un B-tree indexa igualdades/rangos sobre una columna; "el más
    cercano geométricamente entre todos" no es eso, requiere comparar contra todos los vectores,
    así que el B-tree no aplica. Por eso hacen falta índices distintos (HNSW, IVF).
1.2 Almacena vectores (+ id), los indexa con una estructura ANN para búsqueda sub-lineal, y busca
    los top-k por una métrica de distancia. La metadata (texto, tenant_id, fecha) permite filtrar
    y recuperar el dato original asociado al vector.
1.3 Biblioteca in-process: FAISS (índice en RAM del proceso, sin servidor). Extensión relacional:
    pgvector sobre Postgres (vectores al lado de la data, filtro con WHERE). Dedicada gestionada:
    Pinecone (SaaS, namespaces y filtros, pagás por uso). Dedicada self-hosted: Qdrant/Weaviate/
    Milvus (servidor propio con filtrado, payload e híbrido nativos).
1.4 No montarla si tenés pocos chunks (decenas de miles) y ya usás Postgres: pgvector alcanza y
    ahorra un sistema a sincronizar. Subir a una dedicada lo justifican: escala (decenas de
    millones), features vector-nativas (cuantización agresiva, híbrido), o carga de búsqueda que
    tu Postgres no banca.
```

### Módulo 2
```
2.1 Comparar la query contra TODOS los vectores, calcular distancia, ordenar y tomar los k
    menores. Costo O(n·d) por consulta. Es correcta cuando el corpus es chico (hasta ~10^5-10^6
    con BLAS/GPU): es exacta, sin parámetros que afinar, y la latencia todavía entra.
2.2 En alta dimensión, las estructuras de partición espacial exactas (k-d trees) degeneran a
    fuerza bruta: todos los puntos quedan a distancias parecidas y la geometría ya no permite
    podar ramas, así que el árbol termina visitando casi todos los nodos. No hay índice exacto
    y eficiente en cientos/miles de dimensiones.
2.3 Approximate Nearest Neighbor: devuelve vectores que casi seguro están entre los más cercanos,
    la mayoría de las veces, a cambio de ser mucho más rápido. Se sacrifica exactitud. Recall =
    de los k vecinos verdaderos, qué fracción devolvió el índice (0.95 = 19 de 20).
2.4 Recall ↔ latencia ↔ memoria: podés tener dos baratos, no los tres. nprobe/efSearch son
    perillas de query que te mueven sobre el eje recall↔latencia (más exploración = más recall,
    más latencia).
```

### Módulo 3
```
3.1 L2: distancia euclídea, sensible a magnitud. IP: producto punto, mezcla dirección y magnitud.
    Coseno: ángulo, solo dirección, ignora magnitud. La de texto es coseno porque importa el
    significado (la dirección del vector), no su "largo".
3.2 Sobre vectores normalizados (norma 1), coseno == producto interno (y el ranking L2 es el
    inverso del coseno). Ventaja: normalizás una vez en el ingest y usás IP, más barato (sin la
    división), obteniendo ranking coseno gratis.
3.3 Porque no lanza error: si comparás vectores sin normalizar con IP esperando coseno, la
    magnitud contamina el ranking y el recall sale malo de forma silenciosa. El síntoma es
    "resultados mediocres sin excepción", lo más difícil de debuggear.
3.4 Porque algunos modelos salen normalizados y otros no, y la métrica esperada varía; asumir mal
    rompe el ranking en silencio. Se verifica en la doc del proveedor de embeddings.
```

### Módulo 4
```
4.1 Como un bibliotecario que elige el estante antes de buscar. Fase train: k-means sobre una
    muestra para hallar nlist centroides (celdas de Voronoi). Fase add: cada vector se asigna a
    la celda de su centroide más cercano. El train necesita datos representativos porque define
    cómo se parte el espacio; mal entrenado, parte mal y el recall cae.
4.2 nlist (cuántas celdas) se fija al construir; nprobe (cuántas celdas visitar) se ajusta por
    query. Subir nlist: celdas más chicas, búsqueda más rápida, pero más riesgo de perder al
    vecino en una celda no visitada. Subir nprobe: más recall, más latencia.
4.3 El vecino más cercano real puede caer del otro lado del límite entre celdas, en una celda que
    no visitaste con nprobe bajo. Se mitiga subiendo nprobe (visitás también celdas vecinas) a
    costa de latencia.
4.4 nprobe = nlist visita todas las celdas = fuerza bruta: recall perfecto pero sin ganancia de
    velocidad. Muestra que IVF es un punto en el trade-off recall↔latencia que vos elegís con
    nprobe, no una aceleración gratis.
```

### Módulo 5
```
5.1 Capas altas: pocos nodos, conexiones de largo alcance (vuelos) que acercan rápido a la región.
    Capas bajas: todos los nodos, conexiones locales (calles) que refinan. La búsqueda entra por
    la capa más alta, avanza greedy hacia el más cercano a la query, baja una capa y repite hasta
    la capa 0; logarítmica en la práctica.
5.2 M (vecinos por nodo) y efConstruction (exhaustividad al insertar) son de build; efSearch
    (candidatos en juego por query) es de query y es el análogo del nprobe de IVF (perilla
    recall↔latencia).
5.3 (1) Memoria alta: el grafo entero (vectores + aristas) vive en RAM. (2) Borrados malos o no
    soportados (FAISS no permite remove en HNSW): hay que marcar y reconstruir. La segunda conecta
    con el "día 2" de RAG (documentos que se editan/borran): corpus muy cambiante obliga a
    reindexar seguido.
5.4 IVF+PQ: con 300M de vectores y RAM ajustada, HNSW no entra (grafo full-precision en RAM). IVF
    mira pocas celdas y PQ comprime los vectores ~64×, que es lo que los hace entrar en memoria;
    se acepta algo menos de recall (recuperable con nprobe/rescoring).
```

### Módulo 6
```
6.1 A escala, los vectores no entran en RAM y la RAM es el recurso caro. 10M × 1024 dims × 4 bytes
    (float32) = ~40 GB solo de datos crudos, sin el overhead del índice.
6.2 Se parte el vector en m subvectores; por cada subespacio se entrena (k-means) un codebook de
    256 centroides, y cada subvector se reemplaza por el id de 1 byte de su centroide más cercano.
    Con m=64 → 64 bytes por vector (vs 4096): 64× de compresión. Se sacrifica precisión (el vector
    queda aproximado); las distancias se estiman con tablas precalculadas.
6.3 IVFPQ = IVF (visitar solo unas celdas) + PQ (comprimir los vectores dentro de cada celda 64×).
    La combinación ataca las dos dimensiones del problema a escala: menos vectores que mirar y
    menos memoria por vector, lo que hace entrar miles de millones de vectores en una máquina.
6.4 Rescoring: traés muchos candidatos con los vectores comprimidos/binarios (barato) y los
    reordenás con los vectores full-precision (caro pero sobre pocos) — igual al embudo
    recall→precision del reranking de RAG. No conviene cuantizar si el corpus entra cómodo en RAM:
    ahí HNSW full-precision (o SQ) da mejor recall sin complicarte.
```

### Módulo 7
```
7.1 El índice vive en la RAM del proceso Python, sin servidor ni red. Implica: si el proceso
    muere se pierde (persistencia manual con write_index/read_index), y no hay concurrencia
    multi-cliente gestionada (es una estructura en tu proceso, no una base con conexiones).
7.2 D: distancias/similitudes de los k resultados; I: los índices POSICIONALES (orden en que
    agregaste), no tus ids ni tu texto. Por eso mantenés un mapeo posición→(id, texto, metadata)
    aparte. IndexIDMap te deja usar tus propios ids (add_with_ids) en vez de las posiciones.
7.3 normalize_L2(vectores) los lleva a norma 1, y sobre vectores normalizados el producto interno
    (IndexFlatIP) es la similitud coseno (módulo 3). Así "búsqueda coseno" = normalizar + IP.
7.4 (Dos de) persistencia manual (RAM, se pierde al morir el proceso); sin filtros/metadata
    nativos (el WHERE tenant_id lo hacés por fuera); borrados no soportados en HNSW; sin
    concurrencia gestionada. FAISS solo es correcto en: corpus grande, mayormente estático,
    read-heavy, máximo control y velocidad sin operar servidor, y sin filtrado/multi-tenant central.
```

### Módulo 8
```
8.1 FAISS — correcto: corpus grande estático read-heavy sin filtros, sin servidor; evitar: corpus
    muy cambiante o con multi-tenant. pgvector — correcto: ya usás Postgres y filtrás por permisos
    en la misma query; evitar: escala/carga que Postgres no banca. Pinecone — correcto: querés
    escala sin operar infra y el SaaS cierra; evitar: datos que no pueden salir de tu infra o
    querés evitar lock-in. Self-hosted (Qdrant…) — correcto: features vector-nativas a escala en
    tu infra; evitar: cuando pgvector ya resolvía (operás de más).
8.2 Ventaja: un solo sistema, con filtrado por permisos/metadata transaccional en la misma query
    (WHERE + JOINs) y sin sincronizar dos bases. Deja de alcanzar cuando la escala (decenas/cientos
    de millones) o la carga de búsqueda superan lo que Postgres sirve, o necesitás cuantización
    agresiva e híbrido nativos.
8.3 Con Pinecone (gestionada) ganás no operar nada y escala serverless; perdés control de la caja,
    portabilidad (lock-in) y que la data vive en un tercero (compliance). Con Qdrant (self-hosted)
    tenés control total y la data en tu infra, pero operás el servidor (deploy, backups, monitoreo).
8.4 Saltar a una dedicada "porque es lo serio" cuando pgvector resolvía el caso — operás
    infraestructura innecesaria. Progresión sana: empezar por lo que ya tenés (pgvector/FAISS) y
    subir a una dedicada solo cuando un límite concreto y nombrable (escala, features, carga) lo
    exija. Es el "escalá la herramienta al problema" de liderazgo/RAG/Kubernetes.
```

### Módulo 9
```
9.1 Pinecone: metadata como dict por vector + un filter estilo Mongo en la query, y namespaces
    (particiones lógicas) para aislar por tenant. Qdrant: payload JSON por punto + un Filter con
    FieldCondition/MatchValue, idealmente con índice de payload sobre los campos filtrados.
9.2 Post-filtering: traer top-k del índice y descartar los que no pasan el filtro — rápido pero
    roto, porque con filtro selectivo pueden pasar menos de k (o cero) aunque existían válidos más
    allá del top-k. Pre-filtering: filtrar primero y hacer fuerza bruta sobre el subconjunto —
    exacto pero lento si el subconjunto es grande y no usa el índice ANN.
9.3 Es integrar el filtro DENTRO de la navegación del índice (un HNSW que solo considera nodos que
    pasan el filtro). Degrada el recall cuando el filtro es muy selectivo: el grafo agota su
    exploración antes de juntar k nodos válidos. Es el mismo síntoma que RAG describió para el
    WHERE tenant_id muy selectivo sobre HNSW en pgvector.
9.4 Porque con pgvector los permisos y los vectores viven en el mismo sistema: filtrás con un WHERE
    transaccional, una sola fuente de verdad. Con una base dedicada, los permisos viven en otro
    sistema (tu base principal) y hay que mantenerlos sincronizados con la vector DB — más
    superficie de error y de fuga.
```

### Módulo 10
```
10.1 "document" marca el texto como contenido buscable y "query" como algo que busca; el hint
     mejora el match. Mismo modelo en ambos para que pregunta y chunks vivan en el mismo espacio
     vectorial (si no, la cercanía no significa nada); cambiar de modelo obliga a reindexar todo.
10.2 FAISS solo guarda vectores e índices posicionales (o ids), no el texto ni la metadata; el
     catalogo mantiene id→(texto, fuente, tenant_id) para reconstruir el resultado. normalize_L2 +
     IndexFlatIP = búsqueda coseno (módulo 3), consistente con la normalización de la query.
10.3 Se traen k*4 porque el filtro por tenant_id es post-filtering manual (FAISS no filtra
     nativamente): hay que sobre-traer para que, tras descartar otros tenants, queden ~k. Problema:
     con un tenant poco representado podés quedarte con menos de k igual (módulo 9). Con
     pgvector (WHERE en la query) o Qdrant (payload filter integrado) el filtro va DENTRO de la
     búsqueda y no sobre-traés.
10.4 Grounding ("respondés solo con el contexto provisto"), permiso de no saber ("si no está,
     'No tengo esa información'"), y etiquetas XML (<contexto>/<pregunta>) para separar datos de
     instrucción. (También: citar la fuente.)
```

### Módulo 11
```
11.1 (a) 30k chunks + Postgres + multi-tenant → pgvector: la escala es chica y el filtro por
     permisos en la misma query es la ventaja decisiva. (b) 80M vectores, RAM acotada, sin filtros
     → dedicada con IVFPQ: la cuantización hace entrar los vectores en RAM; sin filtros, no perdés
     nada por no tener filtered search rico. (c) corpus mediano estático read-heavy sin servidor ni
     filtros → FAISS (HNSW o Flat): máximo control y velocidad in-process.
11.2 No es caro porque varias opciones sirven y la diferencia de rendimiento entre dos razonables
     suele ser chica. El error caro es de escala de herramienta: montar infraestructura
     innecesaria (Pinecone para 20k chunks que pgvector servía) o quedarse en fuerza bruta con 50M
     de vectores y la latencia por las nubes.
11.3 Se mide corriendo un set de prueba (golden set) y calculando recall@k (contra la verdad
     exacta de un índice Flat) y latencia p95 end-to-end, comparando índices/parámetros. La
     decisión sale del número, no de la moda (es eval-driven, como el resto del temario).
11.4 En un agente, la búsqueda vectorial es UNA herramienta más que el modelo decide invocar dentro
     de su loop de razonamiento/acción, según la necesite. En un RAG clásico el retrieval lo
     disparás vos siempre, antes de generar; en el agente lo dispara el modelo cuando lo juzga
     necesario.
```

---

## Siguientes pasos

Con este módulo entendés qué hay **debajo** de la búsqueda semántica: por qué la fuerza bruta no escala y aparece ANN, las métricas de distancia y la normalización, los dos grandes algoritmos (IVF y HNSW) con sus perillas y trade-offs, la cuantización (PQ/IVFPQ) que hace viable la escala masiva, FAISS al desnudo en Python, el panorama de bases (FAISS / pgvector / Pinecone / Qdrant) con su criterio de elección, el filtrado por metadata y multi-tenant en bases dedicadas, y un RAG end-to-end en Python con Voyage y Claude. Todo con la regla que recorre el temario: **escalá la herramienta al problema y medí la decisión, no la adivines.** Lo que sigue en el track Python AI: [AI Agents](ai-agents-python.md) —donde la búsqueda vectorial pasa a ser una herramienta más en el loop del modelo—, [Voice AI](voice-ai.md), y [Deploy de aplicaciones de IA](deploy-ai.md). Y atravesando todo, [evals](evals.md), que es lo que convierte cada una de estas decisiones en un número que podés defender.
