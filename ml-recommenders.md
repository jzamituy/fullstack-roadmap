# 📊 ML clásico y sistemas de recomendación

Este módulo es deliberadamente distinto a los demás del track de IA. Hasta acá todo fue *construir sobre foundation models* (LLMs vía API, RAG, agentes). Esto es el otro lado: **machine learning supervisado clásico** — el rol que el GenAI **no** reemplaza.

Encuadre honesto de entrada, porque define todo: **esto es territorio de ML Engineer / Data Scientist, no de AI app developer.** El solape con tu perfil es **bajo**. Entonces, ¿por qué está en el temario? Por dos razones de criterio, no de día a día:

1. **Para saber cuándo NO usar un LLM.** El error más caro de alguien que solo sabe LLMs es meter un LLM donde corresponde un modelo clásico.
2. **Para hablar con equipos de ML/DS**: vocabulario compartido sobre datos, features, métricas, drift, recomendadores.

Por eso las secciones de MLOps y entrenamiento van **conceptuales y breves**: el objetivo es criterio y conversación, no que entrenes modelos de scoring en producción. El proyecto de referencia (que el curso de Saras usa para esto) es un **motor de descubrimiento de e-commerce con recomendaciones**.

> **Formato:** teoría → ejercicios → **soluciones al final**. Código Python ilustrativo. Conecta con [Vector DBs](vector-dbs.md) (el puente clave: embeddings + ANN), [Evaluations](evals.md), [Deploy de IA](deploy-ai.md) y [Fine-tuning](fine-tuning.md).

---

## Módulo 1 — Cuándo NO es un LLM la respuesta

El reflejo más valioso que te podés llevar: **si la entrada es una tabla (filas = entidades, columnas = features) y la salida es un número o una etiqueta —churn sí/no, score de fraude, riesgo crediticio, forecast de demanda, ranking de leads— es ML supervisado clásico, no un problema de LLM.**

Cinco argumentos concretos de por qué un LLM es la herramienta equivocada ahí:

1. **Precisión.** En predicción numérica/categórica sobre datos estructurados, un árbol con gradient boosting le gana a un LLM. Los LLMs "tienen dificultad para entender estructuras de tablas y extraer información con precisión". (Excepción legítima: si una columna es **texto libre** —notas, descripciones—, ahí el LLM sí ayuda a featurizar.)
2. **Costo y latencia.** Un árbol predice en microsegundos a milisegundos, con costo por predicción **predecible**. La inferencia LLM es cara, lenta e impredecible (round-trip de red, tokens). Para fraude/scoring en tiempo real es descalificante.
3. **Determinismo.** Un árbol entrenado es una función fija: misma entrada → misma salida, **siempre**. El LLM es estocástico (incluso con `temperature=0` no hay garantía dura de reproducibilidad bit-a-bit, por batching/hardware) y sufre version-drift (el proveedor cambia el modelo bajo tus pies). Para decisiones auditadas (crédito, fraude), la reproducibilidad es requisito duro.
4. **Interpretabilidad / regulación.** Los árboles exponen *feature importance*, SHAP y *reason codes* por predicción — necesarios para crédito (avisos de adverse-action), seguros, salud. (Conecta con [seguridad-ia.md](seguridad-ia.md): el EU AI Act exige explicabilidad en alto riesgo.)
5. **El patrón híbrido honesto** (no vendas el "nunca"): la arquitectura defendible es **el ML decide/puntúa, el LLM narra**. El árbol clasifica los datos estructurados; el LLM genera la explicación, el resumen o el mensaje personalizado a partir de ese output. **ML para la decisión, LLM para el lenguaje.**

> **Heurística para enseñar:** salida número/etiqueta + entradas en columnas → **ML clásico (gradient boosting)**. Entradas/salidas en lenguaje natural, imágenes, código, generación abierta → **LLM**. Mixto (tabular + columna de texto libre) → backbone de árbol, opcionalmente featurizás el texto con embeddings o un LLM extrae features y se las pasás al árbol.

---

## Módulo 2 — ML supervisado clásico: el flujo

El backbone, que no cambió y no es obsoleto:

> **datos → features → split train/val/test → entrenar → evaluar → servir**

**Split y validación.** Ratios típicos 60/20/20 o 70/15/15. Con pocos datos, **k-fold cross-validation** (entrenás k veces rotando qué fold es validación); **stratified k-fold** para preservar las proporciones de clase.

**El pecado capital: fuga de datos (*data leakage*)** — cuando información que no estaría disponible en el momento de predecir se filtra al entrenamiento, inflando la métrica de forma engañosa. Tres formas:
- **Target leakage**: un feature codifica la respuesta (ej. "fecha de cancelación" para predecir churn — solo existe si ya canceló).
- **Temporal leakage**: usás datos del futuro para predecir el pasado (crítico en forecasting). Siempre splitear por tiempo en series temporales.
- **Preprocessing leakage**: ajustás el scaler/imputer sobre **todo** el dataset en lugar de solo el fold de entrenamiento. La media/desvío "ven" el test.

La regla: **todo el preprocesamiento se ajusta dentro del fold de train**, nunca sobre el dataset completo. Para tuning honesto, **nested cross-validation**.

---

## Módulo 3 — Métricas: cuándo importa cada una

**Clasificación.**

> **La accuracy engaña con clases desbalanceadas** — y churn/fraude SON desbalanceados (los positivos son raros). Un modelo que predice "todo negativo" tiene 99% de accuracy si el fraude es el 1%. Enseñalo como la trampa #1.

- **Precision** vs **recall** = el dial del costo del error. Optimizá **precision** cuando los falsos positivos cuestan (marcar spam un mail legítimo); **recall** cuando los falsos negativos cuestan (no detectar un fraude o un cáncer).
- **F1** = media armónica de ambos, cuando necesitás un número que balancee.
- **AUC-ROC** = calidad de ranking independiente del umbral, **pero** engaña bajo desbalance severo (casi cualquier clasificador da 0.90+). Bajo desbalance fuerte, **PR-AUC** (precision-recall) es la curva más honesta.

**Regresión.**
- **MAE** (error absoluto medio): en unidades originales, robusto a outliers.
- **RMSE**: penaliza fuerte los errores grandes — usalo cuando los grandes fallos cuestan desproporcionadamente.
- **R²**: proporción de varianza explicada; bueno para comparar modelos, débil con no-linealidad/outliers.

Reportá varias juntas; ninguna sola cuenta la historia completa.

> **Calibración** (clave si usás la *probabilidad*, no solo el ranking): que el modelo diga "0.8" debe significar "pasa 8 de cada 10 veces". Cuidado: técnicas de desbalance como `scale_pos_weight` (módulo 5) **rompen la calibración** — inflan las probas hacia la clase rara. Si necesitás probas confiables (umbral de fraude, pricing de riesgo), recalibrá con **Platt scaling** o **isotonic regression** sobre un set aparte. AUC/ranking no se entera de la mala calibración; un score mal calibrado sí arruina decisiones basadas en el valor de la proba.

---

## Módulo 4 — Feature engineering

En datos tabulares, **la calidad de los features suele importar más que la elección del algoritmo.** Buena parte de las soluciones tabulares ganadoras en competencias dependieron más del feature engineering que del modelo. Crear el feature correcto (ratios, agregaciones, lags temporales, encodings de categóricas) es donde se gana o se pierde.

**El problema de producción: *training-serving skew*.** El feature se computa de una forma en el notebook de entrenamiento (batch, pandas) y de otra en el path de serving (online, otro lenguaje/librería). Resultado: el modelo ve features distintos en train y en prod, y se degrada en silencio. Los **feature stores** existen precisamente para garantizar **una sola definición** del feature en ambos lados (volvemos en el Módulo 11).

---

## Módulo 5 — Gradient boosting: el caballo de batalla

En 2026, el **GBDT (Gradient-Boosted Decision Trees)** sigue siendo el default para predicción tabular en producción. Tres implementaciones, y cuándo cada una:

| Librería | Elegila cuando |
|---|---|
| **XGBoost** | Default de propósito general; control fino, maduro, regularización L1+L2, maneja faltantes. La elección segura. |
| **LightGBM** | El más rápido y de menor memoria. Millones de filas, alta dimensionalidad, velocidad en GPU, ranking a escala. |
| **CatBoost** | El mejor manejo nativo de **categóricas** (sin encoding manual) y los mejores defaults out-of-the-box. El más "plug-and-play". |

```python
import xgboost as xgb
from sklearn.metrics import average_precision_score  # PR-AUC

model = xgb.XGBClassifier(
    n_estimators=400, max_depth=6, learning_rate=0.05,
    subsample=0.8, colsample_bytree=0.8,
    scale_pos_weight=99,   # compensa desbalance 1:99 (fraude) — pero descalibra las probas (ver abajo)
    eval_metric="aucpr",   # PR-AUC, no accuracy
    early_stopping_rounds=50,  # XGBoost 2.x: va en el constructor; para cuando val deja de mejorar
)
model.fit(X_train, y_train, eval_set=[(X_val, y_val)], verbose=False)
print(average_precision_score(y_val, model.predict_proba(X_val)[:, 1]))
```

**¿Por qué los árboles le ganan al deep learning en tabular?** El paper canónico es Grinsztajn, Oyallon & Varoquaux (NeurIPS 2022, *"Why do tree-based models still outperform deep learning on typical tabular data?"*). Tres razones de sesgo inductivo: (1) las redes tienden a funciones demasiado suaves, los árboles aprenden funciones irregulares con facilidad; (2) los features no informativos perjudican a las redes pero no a los árboles; (3) los árboles preservan la orientación de cada feature (en tabular, cada columna significa algo distinto). Sigue siendo la cita de referencia en 2026.

**La frontera viva (sé honesto):** los **tabular foundation models** —**TabPFN-2.5** (un transformer pre-entrenado en millones de datasets sintéticos que predice por in-context learning, sin entrenar)— lideran benchmarks como **TabArena** y le ganan a XGBoost default **en datasets chicos** (100% de win-rate sin tunear en el régimen ≲10k filas / ≲500 features). La 2.5 amplió el soporte hasta ~50k filas / 2k features (con win-rate aún alto), y trae destilación a MLP/árbol para servir con baja latencia. Igual: a escala grande el soporte de regresión va por detrás y conviene medir antes de confiar. El consenso de practicantes: **el GBDT sigue en el trono para producción**, sobre todo a escala. Presentá TabPFN como "baseline fuerte a probar en datos chicos", no como reemplazo. El titular "le ganó a XGBoost sin tunear" es cierto **solo** en el régimen de datos chicos.

---

## Módulo 6 — Recommender systems I: collaborative y content-based

**Collaborative filtering (CF)**: recomendar usando el comportamiento de muchos usuarios ("a quienes compraron X también les gustó Y"), sin mirar atributos del ítem.

> **El punto práctico #1 para e-commerce: feedback implícito vs explícito.** Casi nunca tenés ratings de estrellas (explícito). Tenés clicks, vistas, add-to-cart, compras (**implícito**). El feedback implícito se trata como **datos positive-only con confianza ponderada** (formulación weighted-ALS de Hu-Koren-Volinsky), **no** como ratings. Es un error tratar "no interactuó" como "no le gusta".

- **kNN user/item** (los usuarios/ítems más parecidos): fundacional y pedagógico.
- **Matrix factorization (ALS implícito)**: el método clásico que **todavía va a producción** como fuente de candidatos. Descompone la matriz usuario×ítem en dos matrices de embeddings de baja dimensión.

```python
# implicit: el default vivo para CF de feedback implícito (Python)
import implicit
from scipy.sparse import csr_matrix

# matriz usuario×ítem con "confianza" (ej. # de interacciones), no ratings
model = implicit.als.AlternatingLeastSquares(factors=64, regularization=0.05)
model.fit(user_item_matrix)              # csr_matrix user×item (¡la orientación cambió en implicit 0.5.0: antes era item×user!)
recs = model.recommend(user_id, user_item_matrix[user_id], N=10)  # user_item_matrix[user_id] debe quedar como CSR de 1 fila (2D), no 1D
```

**Content-based**: recomendar por atributos del ítem (categoría, marca, texto, precio, imagen) → representar ítems como vectores → similitud. La realización **moderna** es **basada en embeddings** (de texto/imagen) — exactamente el puente con el vector search que ya sabés ([vector-dbs.md](vector-dbs.md)). Las features de contenido son la palanca principal para el **cold-start de ítems** (ítems nuevos sin historial de interacciones).

---

## Módulo 7 — Recommender systems II: híbridos, cold-start y el puente con vector search

**Híbrido** = combinar CF (para ítems/usuarios "calientes", con historial) con content-based (para los "fríos", sin historial). Resuelve el **cold-start**:
- **Usuario nuevo** → demografía, onboarding, contexto.
- **Ítem nuevo** → features de contenido, hasta acumular interacciones.

La librería de libro para híbridos es **LightFM** (fusiona CF + features de contenido con losses de ranking). **Caveat de mantenimiento importante:** LightFM está **efectivamente sin mantener** (último release 2023, builds rotos en Python 3.12+). Enseñalo como **el algoritmo híbrido canónico** (vale conceptualmente), pero **no lo recomiendes como default de producción 2026** sin verificar compatibilidad. En cambio, **`implicit` sigue vivo** y es el default seguro para CF.

**El puente con vector search (tu terreno conocido):** el *retrieval* de un recomendador **es** búsqueda vectorial aplicada. Representás usuarios e ítems como embeddings y buscás los vecinos más cercanos con **ANN** (HNSW/IVF). La vector DB que ya conocés ([vector-dbs.md](vector-dbs.md)) te da ese índice; lo nuevo de recsys es **cómo producir los embeddings** (matrix factorization, two-tower), no el índice. FAISS es la librería ANN dominante; HNSW es el algoritmo que ya usás dentro de tu vector DB.

---

## Módulo 8 — La arquitectura a escala: retrieval → ranking

El concepto más importante de "cómo funcionan los recomendadores grandes" (YouTube, Netflix, Amazon). Es un **embudo de dos (o tres) etapas**:

1. **Retrieval / generación de candidatos** — reducir millones de ítems a ~cientos/miles. Barato y rápido. Enfoque dominante: **modelo two-tower** (una torre de usuario y una de ítem, redes separadas que producen embeddings en un espacio compartido). Los embeddings de ítems se **precomputan offline**; en serving, embebés al usuario y hacés **búsqueda ANN** sobre el índice de ítems. (Literalmente vector search.)
2. **Ranking** — un modelo más pesado re-puntúa esos ~cientos de candidatos usando **cross-features** usuario×ítem (que el two-tower no modela, porque sus torres son independientes para poder usar ANN). Optimiza el objetivo de negocio (prob. de compra, revenue esperado).

> El embudo **barato-y-amplio → caro-y-preciso** es el modelo mental durable. Muchos sistemas reales agregan una etapa de *pre-ranking* en el medio (tres etapas).

**Learning-to-rank (etapa de ranking).** El baseline dominante y durable es **LambdaMART** — un GBDT optimizado para ranking, implementado como `rank:ndcg` en XGBoost y en LightGBM. Sobre features tabulares, sigue siendo robusto, interpretable y difícil de batir. Los modelos de ranking *deep* (DLRM, transformers) se usan a la máxima escala, pero el debate deep-vs-GBDT sigue abierto en 2026: **no le digas a nadie que el ranking deep "ganó"; LambdaMART/GBDT es el default pragmático** para la mayoría del e-commerce.

> **Dos sesgos que un recomendador serio tiene que manejar:** (1) **Position bias** — los clicks están sesgados por la posición en que mostraste el ítem (lo de arriba se clickea más, sea o no más relevante); entrenar el ranker con esos clicks crudos lo refuerza. Se corrige con position-debiasing (modelar la probabilidad de examen por posición). Es el gemelo, dentro del ranking, del sesgo de logging del módulo 9. (2) **Popularity bias / diversity** — el embudo tiende a amplificar lo ya popular y a empobrecer la diversidad/novedad; medí y, si hace falta, re-rankeá por diversidad (no todo es relevancia pura).

---

## Módulo 9 — Métricas de recomendación: offline vs online

**Métricas offline:**
- **Precision@k, Recall@k, Hit Rate**: binarias, ignoran la posición dentro del top-k.
- **MAP, MRR, NDCG**: *rank-aware*, premian poner lo relevante más arriba. **NDCG** es la más usada (y la que optimizan XGBoost/LightGBM en ranking); **MRR** para "primer acierto relevante".

> **El punto que carga peso: las métricas offline NO predicen de forma confiable el lift online.** El eval offline está sesgado por la *política de logging* (solo ves feedback de los ítems que el sistema **viejo** mostró — nunca sabés si el usuario habría amado un ítem que nunca le mostraste). El **A/B testing es el estándar de oro** para medir impacto de negocio real.

Patrón a enseñar: **offline para iterar rápido y filtrar modelos malos; online (A/B, interleaving) para la decisión final.** Es el mismo espíritu de online vs offline que viste en [evals.md](evals.md), aplicado a recsys.

---

## Módulo 10 — LLMs y embeddings en recommenders (2026): real vs hype

**Real y en producción:**
- **Recomendación semántica / basada en embeddings**: usar embeddings de texto/imagen (incluidos los de LLMs) para representar contenido y resolver cold-start. Mainstream, bajo riesgo.
- **LLMs para cold-start**: embeddings generados de la metadata/descripción de un ítem nuevo para sembrarlo antes de tener interacciones.
- **Foundation models de recomendación**: Netflix y Spotify construyeron recomendadores tipo-LLM (transformers entrenados sobre cientos de miles de millones de interacciones, con tokenización aprendida / semantic IDs). Son despliegues genuinos — **a escala FAANG**, con la ingeniería que eso implica.

**Frontera / research-leaning (saber que existe, no implementar):**
- **Generative recommendation / TIGER / Semantic IDs**: replantear "predecir el próximo ítem" como un seq2seq que *genera* el ID semántico del próximo ítem. Área caliente 2025-26, con gains reportados grandes, pero con bloqueadores de producción reales (latencia de decodificación serial, expresividad limitada de IDs cortos). **No es la arquitectura default que un equipo e-commerce normal debería construir hoy.**

> **El patrón honesto a internalizar (igual que el Módulo 1):** en un recomendador moderno, **el modelo clásico decide/rankea y el LLM aporta lenguaje** (explicaciones, descripciones, cold-start semántico). No reemplaza al stack de dos etapas.

**Recomendación para el proyecto de referencia:** construí el **sistema clásico de dos etapas** (retrieval con ALS-implícito o two-tower + ANN; ranking con GBDT/LambdaMART; features de contenido híbridas para cold-start), usá **embeddings de LLM** para contenido/cold-start, y tratá la generative recommendation como tema "frontera, saber-que-existe".

Stack Python: `implicit` (CF implícito, default seguro) · XGBoost/LightGBM (LambdaMART) · FAISS o tu vector DB (retrieval) · `LightFM` solo conceptual (sin mantener).

---

## Módulo 11 — MLOps para ML clásico y criterio de cierre

Breve y conceptual, como dijimos. Lo único de esta sección que vos, como app dev / AI QA, **vas a tocar con las manos** es el **experiment tracking aplicado a tus apps LLM** (ver abajo).

- **Experiment tracking — MLflow vs Weights & Biases.** MLflow = estándar open-source agnóstico (runs, params, métricas, artefactos); W&B = suite comercial con la mejor UI de visualización. Suelen usarse juntos. **La noticia que te toca**: el **pivote GenAI de MLflow (3.0, 2025)** agregó **tracing OpenTelemetry** (captura cada paso de una cadena LLM/agente con latencia + tokens, auto-instrumentando 50+ frameworks) y **versionado de prompts** + eval con LLM-as-judge. Esto sí es para tus apps (puente con [evals.md](evals.md) y [observabilidad.md](observabilidad.md)).
- **Model registry**: "fuente de verdad" de modelos entrenados (como PyPI pero para modelos): versiones + metadata + transiciones de stage (Staging→Production→Archived) + rollback. Es la superficie que usa el CI/CD para promover/revertir.
- **Feature stores**: resuelven el *training-serving skew* (Módulo 4) centralizando la definición del feature. El espacio **se está consolidando dentro de plataformas** (ya no es un producto standalone obligatorio); no lo sobrevendas. Opciones: Feast (OSS), Tecton (adquirida por Databricks), variantes cloud.
- **Drift y reentrenamiento**: *data drift* = cambia la distribución de entrada; *concept drift* = cambia la relación entrada→etiqueta. Los modelos clásicos **se degradan** porque el mundo cambia bajo el modelo → reentrenar con datos nuevos. **Contraste durable con LLMs**: un foundation model no se "degrada" por drift de tus datos (no tenés un loop de feedback que lo degrade); su "obsolescencia" es el knowledge cutoff (se ataca con RAG/tools) y sus "regresiones" vienen de cambios de prompt/modelo/versión, no de drift estadístico.
- **Servir clásico vs LLM**: un modelo clásico es chico, corre en **CPU**, latencia sub-100ms, sin tokens, deploy como API REST simple o batch job; costo de infra finito y predecible. Un LLM es API de terceros (pago por token) o self-host en **GPU** (vLLM, cuantización, KV-cache). (Puente con [deploy-ai.md](deploy-ai.md).)

### La frontera AI Engineer ≠ ML Engineer

El origen de la distinción (swyx, *"The Rise of the AI Engineer"*, 2023; Chip Huyen, *AI Engineering*, 2025): los foundation models crearon una capa nueva entre el research de ML y la ingeniería de producto. El skillset se corre **de ML/estadística hacia software engineering**: prompt engineering, construcción de contexto, RAG, evaluación, fiabilidad de producto.

- **ML engineering** = apps sobre ML tradicional → tabular, feature engineering, **entrenamiento**, drift/reentrenamiento. (Casi todo este módulo.)
- **AI engineering** = apps sobre foundation models → prompting, RAG, evals, agentes. (El resto de tu track.)

> **Veredicto de scope para tu objetivo:** lo de este módulo es **"entenderlo lo suficiente para (1) no agarrar un LLM donde va un modelo clásico y (2) hablar con equipos de ML"** — no skills de día a día. Lo único que tocás con las manos es el tracing/eval estilo MLflow para tus propias apps LLM. Si te entusiasma el ML clásico, es un camino válido — pero es **otro rol**, no el de AI app developer / AI QA al que apunta el resto del temario.

Conexiones: [vector-dbs.md](vector-dbs.md) (embeddings + ANN = retrieval de recsys), [evals.md](evals.md) (offline vs online, A/B), [deploy-ai.md](deploy-ai.md) (servir CPU vs GPU), [fine-tuning.md](fine-tuning.md) (otro pedazo de territorio ML-eng), [observabilidad.md](observabilidad.md) (tracing), [seguridad-ia.md](seguridad-ia.md) (explicabilidad en alto riesgo).

---

## Ejercicios

> El foco es **criterio**: cuándo ML clásico vs LLM, y entender las decisiones de un recomendador.

### Ejercicio 1 — ¿LLM o ML clásico?
Para cada tarea decidí la herramienta y justificá: a) predecir qué clientes van a cancelar la suscripción el mes próximo (tenés su historial de uso en una tabla); b) responder preguntas de soporte sobre la documentación del producto; c) estimar el precio óptimo de un producto según demanda, estacionalidad y competencia; d) escribir la descripción de marketing de un producto nuevo; e) detectar transacciones fraudulentas en tiempo real.

### Ejercicio 2 — La trampa de la accuracy
Entrenás un detector de fraude y reportás **99.2% de accuracy**. Tu jefe está feliz. ¿Por qué no deberías estarlo, y qué métricas pedirías en su lugar?

### Ejercicio 3 — Data leakage
Predecís churn y obtenés AUC 0.99 en validación (sospechosamente alto). Revisando los features, encontrás uno llamado `dias_desde_cancelacion`. ¿Qué tipo de leakage es y por qué destruye el modelo en producción?

### Ejercicio 4 — Diseñá el recomendador del e-commerce
Tenés que recomendar productos en la home. Catálogo de 2M de ítems, clicks/compras (no ratings). Describí: (a) qué tipo de feedback es y cómo lo tratás; (b) la arquitectura de dos etapas y qué modelo pondrías en cada una; (c) cómo manejás un producto recién agregado al catálogo (cold-start); (d) dónde aparece el vector search que ya conocés.

### Ejercicio 5 — Offline se ve genial, online no mueve la aguja
Tu nuevo modelo de ranking mejora el NDCG offline de 0.42 a 0.51, pero el A/B test no muestra lift en compras. Da dos razones plausibles de por qué la métrica offline no se tradujo en resultado de negocio.

### Ejercicio 6 — El patrón híbrido ML + LLM
Diseñá un flujo para "explicar al usuario por qué le recomendamos este producto" que use **lo mejor de cada mundo**. ¿Qué decide el modelo clásico y qué hace el LLM?

---

## Soluciones

### Solución 1
a) **ML clásico** (clasificación, GBDT): tabular, salida etiqueta, necesita determinismo/interpretabilidad. b) **LLM** (RAG): lenguaje natural sobre conocimiento. c) **ML clásico** (regresión/forecasting): tabular, salida numérica, auditable. d) **LLM**: generación de lenguaje abierto. e) **ML clásico** (clasificación): tabular, **tiempo real** (latencia/costo del LLM lo descalifican) y auditable. Patrón: número/etiqueta sobre columnas → clásico; lenguaje → LLM.

### Solución 2
Porque el fraude es una **clase rara** (quizá <1%): un modelo que dice "nunca hay fraude" logra ~99% de accuracy y **no detecta ningún fraude**. La accuracy está dominada por la clase mayoritaria. Pedís: **precision y recall** de la clase fraude (¿cuántos fraudes atrapás? ¿cuántas alarmas son falsas?), **PR-AUC** (la curva honesta bajo desbalance), y según el costo del error, ajustar el umbral hacia recall (no perder fraudes) o precision (no molestar clientes legítimos).

### Solución 3
**Target leakage**: `dias_desde_cancelacion` solo existe **después** de que el cliente canceló — es prácticamente la respuesta disfrazada de feature. En entrenamiento "predice" perfecto porque está mirando el futuro. En producción, en el momento de predecir (cliente activo), ese feature **no existe o es nulo**, así que el modelo no tiene la señal de la que dependía y colapsa. Hay que eliminar todo feature que no esté disponible en el instante real de la predicción.

### Solución 4
a) **Feedback implícito**: clicks/compras como señales positivas con confianza ponderada (más interacciones = más confianza); **no** tratar "no vio" como negativo. b) **Dos etapas**: *retrieval* con ALS-implícito o two-tower (reduce 2M → ~500 candidatos, vía ANN); *ranking* con LambdaMART/GBDT usando cross-features usuario×ítem sobre esos ~500. (A 2M ítems, muchos sistemas reales intercalan un **pre-ranking** entre retrieval y ranking — las tres etapas del módulo 8.) c) **Cold-start**: features de contenido (categoría, marca, embedding de la descripción/imagen) hasta que acumule interacciones — un enfoque híbrido. d) **Vector search**: en el retrieval — los embeddings de ítems se indexan con ANN (HNSW/IVF en tu vector DB) y buscás los vecinos del embedding del usuario.

### Solución 5
1) **Sesgo de la política de logging**: el eval offline solo puntúa sobre los ítems que el sistema viejo mostró; el modelo nuevo quizá es mejor justo en ítems que nunca estuvieron en los logs, y eso el offline no lo puede ver (ni premiar ni castigar). 2) **La métrica offline (NDCG sobre clicks/relevancia histórica) no es el objetivo de negocio (compras)**: mejorar el orden de ítems "relevantes" según el log no necesariamente mueve la conversión — pueden ser ítems que el usuario igual no compra. Por eso el A/B online es el árbitro final.

### Solución 6
**El modelo clásico decide**: el recomendador de dos etapas (retrieval + ranking) selecciona y ordena los productos, y expone **por qué** en términos estructurados (ej. feature importance / señales: "comprado junto con X", "misma categoría que tu última compra", "popular en tu región"). **El LLM narra**: toma esas señales estructuradas y genera una explicación en lenguaje natural, personalizada y con el tono de marca ("Como compraste la cafetera la semana pasada, esto te puede servir..."). El LLM **no** decide el ranking (sería caro, no determinista, no auditable); solo convierte la decisión del modelo en lenguaje. Es el patrón "ML decide, LLM narra" del Módulo 1 y 10.
