# 🔧 Fine-tuning y adaptación de modelos

Hay un malentendido que casi todo el mundo trae: *"si quiero que el modelo sepa de mi empresa, lo fine-tuneo"*. **Es falso**, y desarmar esa idea es el objetivo central de este módulo.

Encuadre honesto antes de empezar: **para un desarrollador de aplicaciones, el fine-tuning es conocimiento de "saber que existe y cuándo NO usarlo", no una habilidad del día a día.** La mayoría de los app devs nunca fine-tunean — y hacen bien. Entrenar modelos es trabajo de **ML engineer**. Pero necesitás el criterio para: (1) no agarrar fine-tuning donde corresponde [RAG](rag.md) o [prompting](prompt-engineering.md), (2) reconocer los pocos casos donde sí conviene, y (3) hablar con equipos de ML sin perderte.

> **Formato:** teoría por módulo → ejercicios → **soluciones al final**. Código en Python (HuggingFace TRL/PEFT). Conecta con [Prompt Engineering](prompt-engineering.md), [RAG](rag.md), [Vector DBs](vector-dbs.md), [Evaluations](evals.md) y [Deploy de IA](deploy-ai.md).

---

## Módulo 1 — El orden de operaciones (y por qué fine-tuning va casi al final)

Antes de fine-tunear, agotá lo barato. El orden que recomienda toda fuente seria (OpenAI, Anthropic, Chip Huyen) es:

> **evals (siempre, debajo de todo) → prompt engineering → RAG → fine-tuning → distillation**

- **Evals primero y siempre** ([evals.md](evals.md)): sin un set de evaluación no sabés si algo mejoró. Es la base de todo el resto.
- **Prompt engineering** resuelve el **60-80%** de lo que la gente cree que necesita fine-tuning. Instrucciones claras, few-shot, structured output.
- **RAG** para inyectar conocimiento que el modelo no tiene.
- **Fine-tuning** solo cuando lo anterior no alcanza, y para el tipo de problema correcto (Módulo 2).
- **Distillation** para abaratar una pipeline que *ya funciona* (Módulo 9).

Por qué esto se volvió **más** cierto en 2026, no menos: los modelos base cerraron los gaps que motivaban fine-tuning en 2023-24 (contexto largo real, tool use nativo, structured output por decodificación restringida). Cada año que pasa, hay menos razones para fine-tunear como app dev.

El costo oculto que define la frontera **app dev / ML engineer**: un modelo fine-tuneado en producción **se reentrena cada 1-3 meses** y consume una fracción permanente de la capacidad de un ingeniero **por modelo**. No es un proyecto, es un compromiso continuo. Por eso no es algo que tomás a la ligera.

---

## Módulo 2 — RAG vs fine-tuning vs long-context: "hechos vs forma"

La regla mnemotécnica de Chip Huyen, que tenés que internalizar:

> **RAG es para hechos. Fine-tuning es para forma.**

- **Forma** = estilo, formato, tono, estructura de salida, hábitos de razonamiento, vocabulario de dominio. *Cómo* responde el modelo.
- **Hechos** = información que el modelo no tiene (tu base de conocimiento, datos frescos, documentos privados). *Qué* sabe el modelo.

El **modelo de dos ejes** de OpenAI lo dice igual de claro:

| Tipo de gap | Síntoma | Herramienta correcta |
|---|---|---|
| **Conocimiento** | El modelo no sabe un hecho, o lo inventa | **RAG** (o long-context) |
| **Comportamiento / forma** | El modelo sabe pero responde con formato/tono/estilo equivocado | **Prompting**, y si no alcanza, **fine-tuning** |

Esto conecta directo con lo que ya viste en [rag.md](rag.md) y [vector-dbs.md](vector-dbs.md): ahí establecimos que *"fine-tuning no enseña hechos"*. Acá está el porqué completo: **el fine-tuning ajusta los pesos hacia un comportamiento, no es una base de datos.** Si fine-tuneás con tus documentos esperando que el modelo los "memorice", vas a obtener un modelo que *suena* como tus documentos pero **alucina los detalles** — lo peor de ambos mundos. Para hechos: RAG.

Y **long-context**: cuando son pocos documentos por consulta o estás prototipando, a veces alcanza con meter todo en el contexto (+ [prompt caching](prompt-engineering.md) para el costo). RAG gana cuando el conocimiento es grande, cambia, o necesita citas/permisos.

---

## Módulo 3 — Qué es fine-tuning por dentro

**Full fine-tuning**: tomás un modelo pre-entrenado y seguís entrenándolo con tus datos, **actualizando todos sus pesos**. Funciona, pero tiene tres problemas para un equipo normal:

1. **Memoria brutal.** No alcanza con la memoria de los pesos: el optimizer (Adam) guarda momentos por cada parámetro, más los gradientes, más las activaciones. La regla de pulgar es que entrenar necesita **varias veces** la memoria de solo inferir. Un modelo de 7B que infiere en ~14GB puede necesitar 60-80GB para full fine-tuning.
2. **Catastrophic forgetting.** Al mover todos los pesos hacia tu dataset chico, el modelo puede *olvidar* capacidades generales que tenía. Lo afilás para tu tarea y lo embrutecés para todo lo demás.
3. **Un checkpoint entero por tarea.** Cada fine-tune es una copia completa del modelo (decenas de GB). Servir 10 variantes = 10 modelos.

Por estos problemas nació PEFT.

---

## Módulo 4 — PEFT y LoRA: la técnica que domina

**PEFT** (Parameter-Efficient Fine-Tuning): en vez de mover todos los pesos, **congelás el modelo base** y entrenás una cantidad chica de parámetros nuevos.

**LoRA** (Low-Rank Adaptation) es el método dominante, sin discusión. La idea: en vez de actualizar la matriz de pesos `W` (enorme), aprendés dos matrices chicas `A` y `B` de **bajo rango** tales que el cambio es `ΔW = B·A`. Entrenás solo `A` y `B` (una fracción ínfima de los parámetros); en inferencia, sumás `B·A` a `W`.

Ventajas que resuelven los tres problemas del Módulo 3:
- **Memoria**: entrenás <1% de los parámetros → entra en una sola GPU.
- **Sin forgetting catastrófico**: el base queda intacto.
- **Multi-adapter**: un solo modelo base + muchos adapters LoRA chiquitos (MB, no GB). Podés servir decenas de tareas con un base en memoria (volvemos a esto en el Módulo 6).

```python
# LoRA con HuggingFace PEFT + TRL (esquema)
from peft import LoraConfig
from trl import SFTTrainer

lora_config = LoraConfig(
    r=16,                       # rank: 16 suele alcanzar (ver abajo)
    lora_alpha=16,
    target_modules="all-linear",  # CLAVE: todas las capas lineales, no solo attention
    lora_dropout=0.05,
    task_type="CAUSAL_LM",
)

trainer = SFTTrainer(
    model="meta-llama/Llama-3.1-8B",
    train_dataset=dataset,       # tus ejemplos {prompt, respuesta deseada}
    peft_config=lora_config,
    # learning rate ~2e-4: más alto que en full fine-tuning
)
trainer.train()
```

**El resultado clave 2025-26 — "LoRA Without Regret"** (Thinking Machines Lab, ya incorporado a los docs de TRL): **LoRA iguala a full fine-tuning** si se cumplen unas condiciones simples:
- Aplicar los adapters a **todas las capas lineales** (`all-linear`: attention **y** MLP), no solo a las projections de atención. Este es el error histórico más común.
- Un **rank modesto alcanza**: rank=16 logra ~99% de lo que da rank=256. Subir el rank casi no aporta.
- **Learning rate ~10× el de full fine-tuning** (~2e-4).

Corolario importante: el "zoo" de variantes de LoRA (DoRA, rsLoRA, PiSSA, VeRA, LoRA+...) que prometen superar a LoRA **colapsa cuando ajustás bien el learning rate** — un paper de 2026 mostró que las 9 variantes principales quedan dentro del 1-2% entre sí con el LR correcto. Mucha de esa literatura comparaba un método tuneado contra LoRA sin tunear. **DoRA** es la única con algo de tracción (un flag `use_dora=True`), pero su ventaja es marginal. **No te pierdas en el zoo: LoRA bien configurado es el estado del arte.**

---

## Módulo 5 — QLoRA y cuantización para entrenar

**QLoRA** = LoRA sobre un modelo base **cuantizado a 4 bits**. El truco: el base congelado no necesita precisión alta (no se entrena), así que lo comprimís a 4 bits (formato **NF4** vía `bitsandbytes`) y entrenás los adapters LoRA en precisión normal encima. Resultado: un modelo de 7B entra en una GPU de consumo de 24GB (una RTX 4090). Es el estándar de entrenamiento cuando estás limitado por memoria.

```python
from transformers import BitsAndBytesConfig
import torch

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",            # NormalFloat 4-bit
    bnb_4bit_compute_dtype=torch.bfloat16,
    bnb_4bit_use_double_quant=True,
)
# El base se carga en 4-bit; los adapters LoRA se entrenan en bf16 encima.
```

> **Cuidado con un cruce de conceptos** (puente con [deploy-ai.md](deploy-ai.md)): la cuantización aparece en **dos** lugares distintos y no son lo mismo:
> - **Para entrenar** (QLoRA, acá): comprimir el base congelado para que el *entrenamiento* entre en memoria.
> - **Para servir** (deploy-ai): comprimir los pesos para que la *inferencia* sea más barata/rápida. Ahí los formatos típicos son AWQ/GPTQ (GPU) o GGUF (llama.cpp/local), y recordá que cuantizar pesos **no achica el KV cache**.

---

## Módulo 6 — Full fine-tuning vs PEFT: cuándo cada uno

Tras "LoRA Without Regret", el balance se inclinó **más** hacia PEFT. La decisión por defecto hoy es **"LoRA salvo que tengas una razón específica para full fine-tuning"** (el inverso de hace unos años).

**Usá PEFT/QLoRA (el ~90% de los casos):**
- Adaptar a una tarea, dominio o estilo.
- Hardware de una sola GPU.
- **Serving multi-adapter**: un base + muchos LoRA. Si tenés 20 clientes con 20 estilos, full fine-tuning te pide 20 modelos; LoRA te pide 1 base + 20 adapters chicos.

**Considerá full fine-tuning solo cuando:**
- Tenés un dataset **grande** con un **cambio de distribución fuerte** y querés que el modelo *re-aprenda* (no solo se oriente). Ejemplo: un dominio muy alejado del pre-entrenamiento (código de un lenguaje raro, otro idioma poco representado).
- Controlás el deploy y preferís un único checkpoint limpio.

Sobre el supuesto "comeback de full fine-tuning" con modelos chicos (1-3B): es **factibilidad, no superioridad**. Que ahora full FT sea *barato* en modelos chicos no significa que sea *mejor* — LoRA bien tuneado lo iguala igual.

---

## Módulo 7 — Alignment: de RLHF a DPO

Hasta acá vimos **SFT** (supervised fine-tuning): le mostrás ejemplos de entrada→salida deseada y el modelo imita. Pero hay un segundo régimen: enseñar **preferencias** (qué respuesta es *mejor* entre dos), no solo imitar una respuesta correcta.

El método clásico es **RLHF** (Reinforcement Learning from Human Feedback): entrenás un *reward model* con comparaciones humanas y después optimizás el modelo con RL (**PPO**) contra ese reward. Funciona pero es pesado: reward model aparte, sampling online, un critic, mucha GPU. Hoy **PPO-RLHF clásico es sobre todo territorio de los labs frontera** (es como Anthropic hace Constitutional AI, ver [seguridad-ia.md](seguridad-ia.md)).

**DPO** (Direct Preference Optimization) lo reemplazó para el caso práctico. Colapsa todo RLHF en **una pérdida contrastiva supervisada**: le das pares `(respuesta preferida, respuesta rechazada)` y el modelo aprende a preferir la buena directamente — sin reward model, sin sampling online, sin critic. La mitad del footprint de GPU que PPO.

> **One-liner para recordar:** **DPO alinea el *gusto*** (qué le gusta a los humanos: tono, utilidad, seguridad).

Variantes como KTO, ORPO, SimPO existen pero son **nicho** — tanto que TRL las dejó en "experimental" mientras DPO está en el core estable. No las necesitás para empezar.

---

## Módulo 8 — RLVR / GRPO: entrenar razonamiento

El otro gran cambio de 2025-26. Cuando lo que querés mejorar no es el *gusto* sino la *competencia* en tareas con respuesta verificable (matemática, código), entra **RLVR** (Reinforcement Learning from Verifiable Rewards):

- En vez de un reward model aprendido (caro, subjetivo), usás un **verificador determinista**: ¿el código pasa los unit tests? ¿la respuesta matemática es exacta? Eso es el reward. Barato, sin humanos, sin sesgo de juez.
- El algoritmo dominante es **GRPO** (de DeepSeek-R1): elimina el value model y el critic, normalizando las recompensas dentro de un grupo de respuestas generadas.

> **Segundo one-liner:** **GRPO/RLVR entrena la *competencia*** (qué es *correcto*). No compite con DPO — un pipeline frontier moderno usa ambos: SFT → DPO → RLVR.

Dos honestidades importantes:
1. **"Afila, no crea."** Hay evidencia de que RLVR sube el `pass@1` (saca a la superficie capacidad ya latente) más que generar razonamiento genuinamente nuevo. No es magia.
2. **El problema del verificador.** RLVR es limpio donde hay un verificador objetivo (código, mate). Extenderlo a medicina, escritura, dominios subjetivos está **sin resolver** y sufre *reward hacking* (el modelo explota el reward sin resolver la tarea).

La cara "producto" de esto es **RFT** (Reinforcement Fine-Tuning) de OpenAI: adaptás un modelo de razonamiento con un **grader** (que define tu reward, p. ej. una función Python o un LLM-judge) en vez de ejemplos etiquetados. Es herramienta de especialista: solo sirve si podés escribir un grader que puntúe correctitud de forma limpia. **No es algo cotidiano del app dev.**

---

## Módulo 9 — Distillation: frontier → modelo chico

Esta es, probablemente, **la única forma de fine-tuning que un app dev podría tocar con las manos de verdad**.

**Distillation**: usás un modelo grande y caro (el *teacher*) para generar datos de alta calidad, y con esos datos entrenás (fine-tuneás) un modelo chico (el *student*) que corre la tarea a una fracción del costo y la latencia. El caso de uso: **abaratar una pipeline que ya funciona** con un modelo grande.

Cuándo conviene: tenés una tarea **angosta y de alto volumen** (clasificación, routing, extracción, generar SQL) donde el modelo grande funciona pero es caro/lento. Destilás a un modelo chico especializado. Evidencia real: modelos open-source chicos fine-tuneados superando a modelos cerrados grandes **en esa tarea específica** (Predibase *LoRA Land*, Together) — con la salvedad de que esas empresas venden fine-tuning.

Estado de los productos (dato volátil, verificar):
- **OpenAI** está **cerrando su fine-tuning self-serve** (deprecación escalonada hacia 2027). No apuestes la pipeline de distillation al producto first-party de OpenAI.
- **Anthropic / Claude**: **no hay API pública de fine-tuning**. Está disponible vía **Amazon Bedrock** (fine-tuning de Haiku, distillation desde Sonnet) y para enterprise. Anthropic apuesta, para el caso general, a **system prompt + few-shot + prompt caching** en vez de fine-tuning — lo cual refuerza todo el Módulo 1.
- Camino más durable: **Bedrock / Azure / modelos open-weights** que controlás vos.

---

## Módulo 10 — Datos sintéticos para entrenar

Distillation y RFT necesitan datos, y el etiquetado humano no escala. De ahí los **datos sintéticos**: generás ejemplos de entrenamiento con un modelo.

La receta consolidada:

> **seed real chico → teacher (modelo frontera) genera variaciones → filtro de calidad (LLM-judge / reward model) → chequeo de diversidad**

El **filtrado y la curaduría importan más que la generación**. Tirar un millón de samples sin filtro corteja el **model collapse**: entrenar de forma recursiva solo con datos sintéticos degrada el modelo de forma irreversible (estrecha sus capacidades). La mitigación probada: **acumular datos sintéticos *junto* a datos reales, no reemplazarlos**, y mantener diversidad de fuentes.

Para tu perfil: la generación masiva de datos sintéticos es **territorio de ML engineer / data curator**. Lo que **sí** vas a tocar como app dev / AI QA es generar **sets de evaluación y de test** sintéticos para tus propias apps (puente con [evals.md](evals.md): casos sintéticos para ampliar el golden set, marcando que son sintéticos).

---

## Módulo 11 — Criterio de cierre

La frontera, sin vueltas:

- **Para un app dev (tu perfil)**, casi todo este módulo es **literatura conceptual, no trabajo diario.** El día a día es **prompting + RAG + evals**. Fine-tuning, RLVR y datos sintéticos cruzan a territorio **ML engineer**.
- Lo único con chance real de servirte: **distillation para abaratar** una pipeline que ya funciona, y entender el resto lo suficiente para **decidir bien** y **hablar con equipos de ML**.

El criterio comprimido:

1. **Antes de fine-tunear**: evals → prompt → RAG. Agotá lo barato.
2. **RAG para hechos, fine-tuning para forma.** Si querés que "sepa" algo nuevo, es RAG. El error #1 es fine-tunear para inyectar conocimiento.
3. **Si fine-tuneás: LoRA/QLoRA**, a `all-linear`, rank modesto (~16), LR ~2e-4. No te pierdas en el zoo de variantes.
4. **DPO alinea el gusto; GRPO/RLVR entrena la competencia.** Dos regímenes distintos.
5. **Distillation** para bajar costo/latencia (vía Bedrock/Azure/open-weights).
6. **Datos sintéticos**: el filtrado pesa más que la generación; cuidado con el model collapse.

Stack a conocer: **HuggingFace PEFT** (canónica), **TRL** (con la receta LoRA-Without-Regret), **Unsloth** (rápido, single-GPU), **bitsandbytes** (NF4). Y recordá la señal más fuerte de todas: **Anthropic ni siquiera ofrece fine-tuning público para el caso general** — apuesta a prompt + contexto. Eso te dice dónde está el leverage para alguien que construye aplicaciones.

Conexiones: [prompt-engineering.md](prompt-engineering.md) (lo que probás antes), [rag.md](rag.md) / [vector-dbs.md](vector-dbs.md) ("hechos vs forma"), [evals.md](evals.md) (medís si sirvió; datos sintéticos para el golden set), [deploy-ai.md](deploy-ai.md) (cuantización para servir vs para entrenar), [seguridad-ia.md](seguridad-ia.md) (Constitutional AI = alignment).

---

## Ejercicios

> El foco es **criterio de decisión**, no memorizar hiperparámetros.

### Ejercicio 1 — ¿Fine-tuning, RAG o prompting?
Para cada caso, decidí la herramienta correcta y justificá en términos de "hechos vs forma":
a) Querés que el modelo responda siempre en JSON con un schema fijo.
b) Querés que el chatbot conozca las 4.000 páginas de documentación interna de tu empresa, que cambia cada semana.
c) Querés que el modelo escriba con el tono de marca de tu empresa (informal, con emojis, frases cortas).
d) Querés que un modelo chico y barato haga clasificación de tickets con la misma calidad que tu pipeline actual con un modelo grande.

### Ejercicio 2 — El error del conocimiento
Un colega fine-tuneó un modelo con los 500 PDFs de productos de la empresa para que "los aprenda". En la demo, el modelo inventa números de modelo y precios que no existen. Explicá qué pasó y qué debió hacer.

### Ejercicio 3 — Configurar LoRA
Tu primer intento de LoRA aplica adapters solo a las capas de atención (`q_proj`, `v_proj`) con rank=256 y learning rate 2e-5 (el mismo que usarías en full fine-tuning), y el resultado es mediocre. Según "LoRA Without Regret", ¿qué tres cosas cambiarías y por qué?

### Ejercicio 4 — DPO vs GRPO/RLVR
Para cada objetivo, decí si usarías DPO o RLVR/GRPO y por qué:
a) Que el modelo prefiera respuestas más concisas y educadas.
b) Que el modelo resuelva mejor problemas de programación competitiva (con tests automáticos).
c) Que el modelo escriba poesía que a los usuarios les guste más.

### Ejercicio 5 — Multi-adapter
Tenés 15 clientes, cada uno quiere el asistente con su propio estilo de marca. Compará servir esto con 15 modelos full-fine-tuneados vs 1 base + 15 adapters LoRA, en términos de memoria, costo de entrenamiento y operación.

### Ejercicio 6 — Frontera de rol
Te ofrecen dos tareas en un proyecto de IA: (A) construir el sistema RAG + agente + evals de una app de soporte; (B) armar un pipeline de generación de datos sintéticos + fine-tuning + reentrenamiento mensual de un modelo de scoring. Si tu objetivo es un puesto de **AI app developer / AI QA**, ¿cuál es más alineada y por qué? ¿Qué te llevás de la otra?

---

## Soluciones

### Solución 1
a) **Prompting** (structured output) primero; si aun así alucina campos en volumen, **fine-tuning** (es un problema de *forma*). b) **RAG**: es conocimiento, grande y que cambia semanalmente — fine-tunear acá garantiza alucinación de detalles y reentrenar cada semana es inviable. c) **Prompting** primero (un buen system prompt con ejemplos suele bastar para tono); si el estilo es muy específico y constante, **fine-tuning** (forma). d) **Distillation** (fine-tuning del modelo chico con datos del grande): tarea angosta, alto volumen, objetivo = bajar costo/latencia manteniendo calidad. Es el caso *canónico* donde fine-tuning gana.

### Solución 2
Confundió **hechos con forma**. Fine-tuning no es una base de datos: ajusta los pesos hacia un *comportamiento*, no memoriza datos de forma recuperable. El modelo aprendió a *sonar* como los PDFs (estructura, vocabulario) pero no a *recordar* los valores exactos → los inventa con confianza. Debió usar **RAG**: indexar los PDFs, recuperar los chunks relevantes y pasarlos al contexto con instrucción de responder solo con base en lo recuperado (grounding). Bonus: con RAG, actualizar un precio es reindexar un documento, no reentrenar.

### Solución 3
1) **`target_modules="all-linear"`** en vez de solo attention: aplicar a las capas MLP también es lo que cierra la brecha con full fine-tuning. 2) **Bajar el rank a ~16**: 256 casi no aporta sobre 16 y desperdicia cómputo/memoria. 3) **Subir el learning rate a ~2e-4** (~10× el de full fine-tuning): LoRA necesita LR más alto; 2e-5 es demasiado bajo y por eso converge mal. (El LR es, según la investigación reciente, el factor que más mueve la aguja — más que la elección de variante.)

### Solución 4
a) **DPO**: es una preferencia de *gusto* (conciso y educado), tenés pares mejor/peor, no hay un verificador objetivo. b) **RLVR/GRPO**: hay un **verificador determinista** (los tests pasan o no) — el caso ideal de RLVR. c) **DPO**: la calidad poética es subjetiva, no verificable; se modela como preferencia humana. RLVR no aplica porque no hay reward objetivo (y forzar uno llevaría a reward hacking).

### Solución 5
- **15 modelos full-fine-tuneados**: 15 checkpoints completos (decenas de GB c/u) en disco y en memoria de serving; 15 entrenamientos completos (caros); operar 15 modelos en producción (o cargar/descargar, con latencia). No escala.
- **1 base + 15 adapters LoRA**: el base (decenas de GB) cargado **una vez**; 15 adapters de unos MB c/u; 15 entrenamientos *baratos* (PEFT, una GPU); en serving, intercambiás el adapter activo según el cliente sobre el mismo base en memoria. **Mucho más eficiente** — es exactamente para esto que existe el multi-adapter serving.

### Solución 6
**(A) es la alineada** con AI app developer / AI QA: RAG + agentes + evals es construir *sobre* foundation models, que es el corazón del rol — y los evals son el costado de calidad que define al perfil AI QA. **(B) es territorio de ML engineer**: generación de datos, fine-tuning y reentrenamiento periódico de un modelo propio. De (B) te llevás el **criterio**: saber cuándo un problema es de modelo entrenado (scoring/tabular) y no de LLM, entender qué hace un pipeline de fine-tuning para poder colaborar con el equipo de ML, y reconocer el compromiso operativo del reentrenamiento. Pero no es tu día a día. (Este reparto de roles lo profundiza el módulo de [ML clásico y recommenders](ml-recommenders.md).)
