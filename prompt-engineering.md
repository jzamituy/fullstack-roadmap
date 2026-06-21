# Prompt Engineering: comunicarle la intención al modelo

**La disciplina de escribir prompts que funcionan · stack Claude / Anthropic · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya hiciste [LLMs](ia-llms.md) (sabés qué es un LLM, tokens, la Messages API) y que estás cómodo con [Python](python.md). El **prompt engineering** es la disciplina de **comunicarle al modelo qué querés, de forma que lo entregue de manera confiable**. No es "hablarle lindo a la IA": es ingeniería —especificación precisa, estructura, ejemplos, medición e iteración—. Para un AI Engineer es la habilidad transversal: aparece en cada llamada a un LLM, en cada paso de un [RAG](rag.md), en cada turno de un [agente](agentes.md). El objetivo del módulo es doble: las **técnicas** (claridad, estructura, few-shot, chain-of-thought, salida estructurada) y el **criterio** —cuándo un mejor prompt alcanza y cuándo el problema es otro (contexto, RAG, fine-tuning, una arquitectura distinta)—.

**Lo que asumimos.** [LLMs](ia-llms.md) (la Messages API, system/user/assistant, tokens, structured output básico), [Python](python.md), y el SDK de Anthropic.

**Para practicar.**

```bash
uv add anthropic
# export ANTHROPIC_API_KEY=...
```

> Nota sobre datos volátiles: los IDs/precios de modelos Claude y los detalles de la API (thinking, structured output, prefill) cambian. Para datos exactos al implementar, invocá la skill `claude-api`. Modelos vigentes a 2026: `claude-opus-4-8` ($5/$25 por 1M, contexto 1M), `claude-sonnet-4-6` ($3/$15), `claude-haiku-4-5` ($1/$5, 200K).

**Índice de módulos**
1. Qué es prompt engineering (y qué no)
2. Anatomía de un prompt: las cinco piezas
3. Claridad y especificidad: el principio #1
4. Estructura: delimitadores y etiquetas XML
5. Few-shot: enseñar con ejemplos
6. Chain-of-thought y el thinking del modelo
7. Rol y contexto: el system prompt
8. Controlar el formato de salida (structured output)
9. Iterar con medición, no a ojo
10. Grounding, anti-alucinación y robustez
11. El criterio: prompt, contexto, RAG o fine-tuning

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es prompt engineering (y qué no)

**Teoría.** El malentendido inicial: creer que el prompt engineering es encontrar "palabras mágicas" o trucos para engañar al modelo. No. **Es la práctica de especificar una tarea con la precisión suficiente para que el modelo la resuelva de forma confiable y repetible.** La palabra clave es *confiable*: un prompt que anda una vez no sirve; necesitás uno que ande el 95%+ de las veces sobre entradas variadas, en producción.

Por qué es una habilidad real (y no algo que "ya sé porque sé escribir"):

- **El modelo no lee tu mente; lee tu texto.** Lo que para vos es "obvio" (el formato que querés, los casos borde, qué hacer si no sabe) el modelo no lo infiere salvo que lo digas. La mayoría de los prompts malos fallan por **subespecificación**: pediste poco y el modelo llenó los huecos a su criterio, no al tuyo.
- **Es empírico, no teórico.** No hay una fórmula cerrada de "el prompt perfecto". Escribís una hipótesis (un prompt), la probás contra casos reales, ves dónde falla, y ajustás. Es el mismo ciclo de [TDD](tdd.md) y de [eval-driven development](evals.md), aplicado al lenguaje natural.
- **El modelo importa.** Un prompt afinado para un modelo viejo puede sobre- o sub-disparar en uno nuevo. Los modelos modernos (Claude Opus 4.8 y la familia 4.6+) **siguen las instrucciones mucho más literalmente** que los viejos: prompts escritos con lenguaje agresivo ("CRITICAL: SIEMPRE DEBÉS...") para vencer la reticencia de modelos antiguos hoy **sobre-disparan**. Con modelos actuales, instrucción clara y directa > gritar en mayúsculas.

La distinción con lo que viene después en el track:

- **Prompt engineering** = optimizar *qué le decís* al modelo en una llamada.
- **Context engineering** = optimizar *qué información* le das (qué metés en la ventana de contexto, en qué orden) —el corazón de [RAG](rag.md) y de los [agentes](agentes.md)—.

Son complementarios: el mejor prompt sobre el contexto equivocado falla, y el mejor contexto con un prompt vago también. La frase mental: **el prompt es la interfaz entre tu intención y la capacidad del modelo; el prompt engineering es diseñar bien esa interfaz, con el mismo rigor empírico que cualquier otra ingeniería.**

**Ejercicios 1**
1.1 ¿Por qué "un prompt que anduvo una vez" no es un buen prompt? ¿Qué propiedad buscás?
1.2 ¿Cuál es la causa más común de un prompt malo, y por qué el modelo no la "arregla solo"?
1.3 ¿Por qué el lenguaje agresivo ("CRITICAL: SIEMPRE...") puede ser contraproducente en modelos modernos como Claude Opus 4.8?
1.4 Diferenciá prompt engineering de context engineering. ¿Por qué son complementarios?

---

## Módulo 2 — Anatomía de un prompt: las cinco piezas

**Teoría.** Un prompt de producción rara vez es una sola frase. Tiene **componentes**, cada uno con un rol. Conocerlos te da un checklist para construir y debuggear prompts.

1. **Rol / system prompt**: quién es el modelo y cómo se debe comportar ("Sos un asistente que clasifica tickets de soporte"). Va en el campo `system` de la Messages API, separado de la conversación (módulo 7).
2. **Instrucción / tarea**: qué tiene que hacer, concreto y en imperativo ("Clasificá el siguiente ticket en una de estas categorías: ...").
3. **Contexto / datos**: la información sobre la que opera (el ticket, el documento, los datos a extraer). Lo que cambia en cada llamada.
4. **Ejemplos (few-shot)**: demostraciones de entrada→salida correcta (módulo 5).
5. **Formato de salida**: cómo querés la respuesta (JSON con tal esquema, una sola palabra, markdown) (módulo 8).

Un ejemplo que junta las piezas, con la Messages API de Claude:

```python
import anthropic
client = anthropic.Anthropic()

msg = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=512,
    system=(                                 # 1. ROL
        "Sos un clasificador de tickets de soporte de una app de tareas. "
        "Sos preciso y respondés solo con la categoría."
    ),
    messages=[{
        "role": "user",
        "content": (
            "Clasificá el ticket en UNA categoría: "        # 2. INSTRUCCIÓN
            "[facturación, bug, feature_request, cuenta].\n\n"
            "Categorías:\n"                                  # (definiciones = contexto)
            "- bug: algo no funciona como debería\n"
            "- feature_request: piden algo nuevo\n"
            "- facturación: cobros, planes, reembolsos\n"
            "- cuenta: login, contraseña, acceso\n\n"
            "Ticket:\n"                                      # 3. CONTEXTO (lo variable)
            "\"No puedo entrar, dice contraseña incorrecta y el reset no llega.\"\n\n"
            "Respondé solo con la categoría, sin explicación."  # 5. FORMATO
        ),
    }],
)
print(msg.content[0].text)   # → "cuenta"
```

El orden importa: lo **estable** (rol, instrucciones, ejemplos) primero; lo **variable** (el dato de esta llamada) al final. Eso no es solo prolijidad —es lo que permite el [prompt caching](#) (módulo 9): si el prefijo no cambia entre llamadas, Claude lo cachea y pagás ~0.1× por esos tokens—. Por eso el dato del ticket (que cambia) va al final.

La frase mental: **un prompt es un documento estructurado, no una frase suelta; pensalo por componentes y vas a saber qué pieza falta cuando algo falla.**

**Ejercicios 2**
2.1 Nombrá las cinco piezas de un prompt y qué aporta cada una.
2.2 ¿Cuál de las piezas cambia en cada llamada y cuál se mantiene? ¿Por qué importa el orden (estable primero)?
2.3 ¿Qué pieza pondrías para que el modelo no te devuelva una explicación de más cuando solo querés una etiqueta?
2.4 Conectá el orden "estable primero, variable al final" con el prompt caching: ¿qué ahorrás?

---

## Módulo 3 — Claridad y especificidad: el principio #1

**Teoría.** Si solo te llevás una cosa del módulo, que sea esta: **la mayoría de los problemas de prompts se resuelven siendo más específico.** El modelo es capaz; casi siempre el problema es que no le dijiste con precisión qué querías.

Las reglas concretas de la especificidad:

- **Decí qué hacer, no (solo) qué no hacer.** "No uses lenguaje técnico" es débil; "Explicá como si hablaras con alguien sin formación técnica, usando analogías cotidianas" es fuerte. Las instrucciones positivas guían mejor que las prohibiciones.
- **Definí los términos vagos.** "Hacé un resumen corto" → ¿cuán corto? "Resumí en máximo 3 oraciones". "Un tono profesional" → ¿qué significa para vos? Dale criterios medibles.
- **Enumerá los casos borde.** Lo que no especificás, el modelo lo resuelve a su criterio. "Extraé el email del texto" → ¿y si hay dos? ¿y si no hay ninguno? Decilo: "Si hay varios, devolvé el primero. Si no hay, devolvé null."
- **Dale los pasos si la tarea los tiene.** Para tareas multi-paso, descomponé: "Primero identificá X. Después, para cada X, hacé Y. Finalmente, devolvé Z." El modelo sigue secuencias explícitas mucho mejor que un "hacé todo esto" monolítico.
- **Mostrá el formato exacto que querés**, idealmente con un ejemplo del shape (módulos 5 y 8).

Un antes y después:

```text
❌ Vago:
"Analizá este review y decime si es positivo."

✅ Específico:
"Clasificá el sentimiento del review en exactamente una de: positivo, negativo, neutral.
- positivo: el usuario está satisfecho en general
- negativo: el usuario está insatisfecho en general
- neutral: mixto, o sin carga emocional clara
Si el review está vacío o no es un review, devolvé 'neutral'.
Respondé solo con la palabra, en minúscula."
```

El segundo no es "más largo porque sí": cada línea elimina una ambigüedad que, sin ella, el modelo resolvería de formas distintas en distintas entradas —que es exactamente la falta de confiabilidad del módulo 1—. La frase mental: **cuando un prompt falla, tu primer reflejo no es cambiar de modelo ni agregar "por favor"; es preguntarte qué dejaste ambiguo y especificarlo.**

**Ejercicios 3**
3.1 ¿Por qué "decí qué hacer" supera a "decí qué no hacer"? Reescribí "no seas verboso" como instrucción positiva.
3.2 Tomá "resumí esto en un tono profesional" y hacelo específico (definí "corto"/"profesional" con criterios medibles).
3.3 ¿Qué son los "casos borde" de un prompt y por qué especificarlos mejora la confiabilidad? Dá un ejemplo para "extraé la fecha del texto".
3.4 Para una tarea multi-paso, ¿por qué conviene enumerar los pasos en vez de pedir todo junto?

---

## Módulo 4 — Estructura: delimitadores y etiquetas XML

**Teoría.** Cuando un prompt mezcla instrucciones, datos y ejemplos en un bloque de texto plano, el modelo puede confundir qué es qué —¿esto es una instrucción para mí o parte del documento que tengo que procesar?—. La solución es **estructurar el prompt con delimitadores claros**, y la técnica que Anthropic recomienda específicamente para Claude son las **etiquetas XML**.

```python
system = "Sos un asistente que responde preguntas sobre documentos."

user = """\
Respondé la pregunta usando SOLO la información del documento.
Si la respuesta no está en el documento, decí "No está en el documento".

<documento>
La Task API permite crear, asignar y completar tareas dentro de proyectos.
Solo los miembros de un proyecto pueden ser asignados a sus tareas.
El plan gratuito permite hasta 3 proyectos.
</documento>

<pregunta>
¿Cuántos proyectos permite el plan gratuito?
</pregunta>"""
```

Por qué las etiquetas XML funcionan tan bien con Claude:

- **Separan sin ambigüedad** instrucción de datos. El modelo entiende que lo de adentro de `<documento>` es material a procesar, no órdenes a seguir. Esto es clave para evitar **prompt injection** (módulo 10): si un usuario mete "ignorá las instrucciones anteriores" dentro del documento, encerrarlo en `<documento>` ayuda a que el modelo lo trate como dato, no como instrucción.
- **Te dejan referenciar partes**: "Usando el `<documento>`, respondé la `<pregunta>`". El modelo sabe exactamente a qué te referís.
- **Estructuran la salida también**: podés pedir "poné tu razonamiento en `<razonamiento>` y la respuesta final en `<respuesta>`", y después parsear esos bloques.
- **Claude está entrenado para prestarles atención** —es una convención que el modelo reconoce especialmente bien—.

No tienen que ser XML "válido" en sentido estricto; son delimitadores semánticos. Otros delimitadores (`### Sección ###`, triple backticks, `---`) también ayudan, pero para Claude las etiquetas con nombres descriptivos (`<datos_usuario>`, `<ejemplos>`, `<formato_salida>`) son la opción más robusta. La regla: **cuanto más larga y mezclada la entrada, más te conviene estructurarla con etiquetas; en un prompt de producción con instrucciones + contexto + ejemplos, es casi obligatorio.**

**Ejercicios 4**
4.1 ¿Qué problema resuelven los delimitadores/etiquetas XML en un prompt que mezcla instrucciones y datos?
4.2 ¿Por qué encerrar el contenido del usuario en `<documento>...</documento>` ayuda contra prompt injection?
4.3 Dá dos usos de las etiquetas XML más allá de separar la entrada.
4.4 ¿Cuándo es "casi obligatorio" estructurar con etiquetas y cuándo podés tener un prompt plano?

---

## Módulo 5 — Few-shot: enseñar con ejemplos

**Teoría.** A veces explicar la tarea con palabras no alcanza —sobre todo cuando el formato o el criterio son difíciles de describir pero fáciles de *mostrar*—. Ahí entra el **few-shot prompting**: incluís en el prompt **ejemplos de entrada→salida correcta**, y el modelo infiere el patrón. (Sin ejemplos se llama **zero-shot**; con ejemplos, *few-shot* o *N-shot* según cuántos.)

Por qué los ejemplos son tan potentes: comunican **de forma implícita** lo que costaría párrafos explicar —el tono exacto, el formato preciso, cómo manejar ambigüedad, el nivel de detalle—. Un ejemplo bien elegido reemplaza una página de instrucciones.

La forma idiomática en la Messages API: **ejemplos como turnos user/assistant alternados** en el array `messages`, *antes* de la entrada real.

```python
import anthropic
client = anthropic.Anthropic()

client.messages.create(
    model="claude-opus-4-8",
    max_tokens=256,
    system="Extraés el sentimiento de reviews como una sola palabra.",
    messages=[
        # --- ejemplos few-shot (turnos user/assistant) ---
        {"role": "user", "content": "Review: La app es lenta y se cuelga todo el tiempo."},
        {"role": "assistant", "content": "negativo"},
        {"role": "user", "content": "Review: Hace exactamente lo que necesito, la uso a diario."},
        {"role": "assistant", "content": "positivo"},
        {"role": "user", "content": "Review: Funciona, aunque la interfaz podría mejorar."},
        {"role": "assistant", "content": "neutral"},
        # --- la entrada real ---
        {"role": "user", "content": "Review: No puedo creer lo útil que es, me cambió el flujo."},
    ],
)
# → "positivo"
```

⚠️ **Nota importante de la API (Claude actual).** Esto **no es prefill**. El "prefill" —terminar el array con un turno `assistant` *incompleto* para forzar que el modelo lo continúe— **ya no se soporta en los modelos modernos** (Opus 4.8, la familia 4.6+ y Fable devuelven error 400 si el último mensaje es del assistant). Los ejemplos few-shot son distintos: son pares **completos** user→assistant en el medio de la conversación, y el último mensaje es del **user** (la entrada real). Eso sí es válido. Para *forzar un formato de salida* —el uso clásico del prefill— hoy se usa structured output (módulo 8), no prefill.

Buenas prácticas de few-shot:

- **Pocos pero representativos** (3-5 suele bastar): cubrí los casos distintos, incluí algún caso borde, y mostrá la diversidad real de entradas.
- **Consistencia absoluta de formato**: el modelo copia el formato de tus ejemplos al pie de la letra. Si en los ejemplos respondés en minúscula, va a responder en minúscula. Un formato inconsistente entre ejemplos confunde.
- **Incluí casos difíciles**: si hay una ambigüedad recurrente, mostrá un ejemplo de cómo resolverla.
- **Cuidá el costo**: cada ejemplo son tokens de entrada en cada llamada. Si los ejemplos son estables, ponelos temprano y aprovechá el [prompt caching](#) (módulo 9).

La frase mental: **cuando una instrucción en prosa no logra el formato o el criterio que querés, no escribas tres párrafos más —mostrá tres ejemplos—.**

**Ejercicios 5**
5.1 Diferenciá zero-shot de few-shot. ¿Qué comunica un ejemplo que cuesta explicar con palabras?
5.2 ¿Cómo se pasan los ejemplos few-shot en la Messages API? ¿Quién manda el último mensaje y por qué?
5.3 ¿Qué es el prefill y por qué NO es lo que estás haciendo con few-shot en los modelos actuales? ¿Qué usás hoy para forzar el formato de salida?
5.4 Nombrá dos buenas prácticas de few-shot y explicá por qué la consistencia de formato entre ejemplos es crítica.

---

## Módulo 6 — Chain-of-thought y el thinking del modelo

**Teoría.** Para tareas que requieren **razonamiento** (matemática, lógica, análisis multi-paso, decisiones con varias restricciones), pedirle al modelo la respuesta de una sola vez suele dar peores resultados que dejarlo **razonar paso a paso antes de responder**. Esto es el **chain-of-thought (CoT)**.

La técnica clásica de CoT por prompt: pedir explícitamente el razonamiento.

```text
"Resolvé este problema. Pensá paso a paso antes de dar la respuesta final.
Mostrá tu razonamiento dentro de <razonamiento> y la respuesta en <respuesta>."
```

Por qué funciona: el modelo genera la respuesta token a token, condicionado por lo que ya escribió. Si lo forzás a escribir primero el razonamiento, cada paso de la respuesta final se apoya en un razonamiento explícito en vez de "adivinar" de una. Es darle espacio para "pensar en voz alta".

**La forma moderna en Claude: extended/adaptive thinking.** Los modelos actuales tienen un modo de **pensamiento** nativo: en vez de meter "pensá paso a paso" en el prompt, activás el parámetro `thinking` y el modelo razona en un bloque separado antes de responder.

```python
msg = client.messages.create(
    model="claude-opus-4-8",
    max_tokens=8192,                                  # dale aire: el thinking consume tokens del presupuesto
    thinking={"type": "adaptive", "display": "summarized"},  # adaptive: el modelo decide cuánto pensar
    messages=[{"role": "user", "content": "Un proyecto tiene 3 miembros..."}],
)
# La respuesta trae bloques type=="thinking" y type=="text" (la respuesta).
# OJO: en Opus 4.8 el default de display es "omitted" → los bloques thinking llegan
# con texto VACÍO. Para ver el resumen del razonamiento hay que pedir display:"summarized".
```

Puntos clave del thinking moderno (verificá con la skill `claude-api` al implementar):

- **`thinking: {"type": "adaptive"}`** deja que el modelo decida cuánto razonar según la dificultad. Es lo recomendado para tareas no triviales.
- El viejo `budget_tokens` (fijar un presupuesto de pensamiento) **está removido** en los modelos actuales (Opus 4.8/4.7 y Fable devuelven 400 si lo mandás); se controla la profundidad con `output_config.effort`: `low`/`medium`/`high`/`xhigh`/`max` (en Opus 4.8 el default es `high` si lo omitís; `xhigh` y `max` solo existen en Opus 4.6+/Fable).
- Por defecto `display` es `"omitted"` (los bloques `thinking` vienen sin texto); pasá `display: "summarized"` si querés mostrar u observar el resumen del razonamiento.
- Con thinking activado, **no necesitás** el "pensá paso a paso" en el prompt para razonamiento puro —el modelo ya lo hace—. El CoT por prompt sigue siendo útil cuando querés un *formato* de razonamiento específico y parseable, o en modelos sin thinking.

El criterio: **para tareas de razonamiento, dale espacio para pensar** —vía el parámetro `thinking` en Claude moderno, o vía "pensá paso a paso" como técnica de prompt—. No malgastes thinking en tareas triviales (una clasificación simple no lo necesita y agrega latencia/costo). Conecta con el [módulo de LLMs](ia-llms.md), donde viste el thinking, y con [agentes](agentes.md), donde el razonamiento entre tool calls es central.

**Ejercicios 6**
6.1 ¿Qué es chain-of-thought y por qué mejora las tareas de razonamiento? (pensá en cómo genera tokens el modelo)
6.2 ¿Cuál es la forma moderna de CoT en Claude y en qué se diferencia de meter "pensá paso a paso" en el prompt?
6.3 ¿Qué hace `thinking: {"type": "adaptive"}` y qué reemplazó al viejo `budget_tokens`?
6.4 ¿Cuándo NO conviene activar thinking, y por qué?

---

## Módulo 7 — Rol y contexto: el system prompt

**Teoría.** El **system prompt** (campo `system` de la Messages API) es donde definís **quién es el modelo y cómo se comporta**, separado de la conversación con el usuario. Es la instrucción de más alto nivel y persiste a través de todos los turnos.

Qué poner en el system prompt:

- **El rol/persona**: "Sos un asistente legal que ayuda a entender contratos en lenguaje simple." Darle un rol concreto enfoca el conocimiento y el tono del modelo hacia ese dominio.
- **Reglas de comportamiento generales**: tono, qué nunca debe hacer, cómo manejar lo que no sabe, restricciones de formato globales.
- **Contexto estable de la aplicación**: "Trabajás dentro de una app de gestión de tareas llamada X. Los usuarios tienen proyectos y tareas."

Qué **no** va en el system prompt: el dato variable de cada request (eso va en el mensaje `user`). El system es lo que se mantiene igual entre llamadas.

```python
client.messages.create(
    model="claude-opus-4-8",
    max_tokens=1024,
    system=(
        "Sos el asistente de soporte de TaskFlow, una app de gestión de tareas. "
        "Respondés en español, de forma concisa y amable. "
        "Solo respondés sobre TaskFlow; si preguntan otra cosa, lo decís y redirigís. "
        "Si no sabés algo con certeza, lo admitís en vez de inventar."
    ),
    messages=[{"role": "user", "content": "¿Cómo asigno una tarea a un compañero?"}],
)
```

Dos razones de peso para usar el system prompt (y no meter todo en el `user`):

1. **Jerarquía de instrucciones**: el modelo trata el system prompt como autoridad de más alto nivel —reglas que pesan más que lo que diga el usuario en un turno—. Esto es importante para seguridad (módulo 10): una regla en el system ("nunca reveles datos de otros usuarios") es más difícil de sobrescribir desde un mensaje del usuario que la misma regla metida en el `user`.
2. **Prompt caching y consistencia**: como el system es estable, se cachea (módulo 9) y garantiza el mismo comportamiento de base en cada llamada.

Detalle de la API (Claude Opus 4.8): además del system de arranque, podés inyectar instrucciones **a mitad de conversación** agregando un mensaje con `role: "system"` dentro de `messages` —útil para cambios de modo o contexto que aparece después—, sin reescribir el system original ni romper el caché. (Verificá soporte con la skill `claude-api`.)

La frase mental: **el system prompt es el "contrato de comportamiento" del modelo —rol, reglas, contexto estable—; el mensaje del usuario es la tarea puntual de esta llamada—.**

**Ejercicios 7**
7.1 ¿Qué va en el system prompt y qué va en el mensaje del usuario? Dá un ejemplo de cada uno.
7.2 ¿Por qué una regla de seguridad pesa más en el system prompt que metida en el mensaje del usuario?
7.3 ¿Qué ventaja de costo/consistencia te da que el system sea estable?
7.4 ¿Para qué sirve poder inyectar un mensaje `role: "system"` a mitad de conversación?

---

## Módulo 8 — Controlar el formato de salida (structured output)

**Teoría.** En una app real casi nunca querés "texto libre": querés datos que tu código pueda **parsear y usar** —un JSON con campos definidos, una categoría de un enum, un booleano—. Controlar el formato de salida es central, y hay un espectro de técnicas, de la más débil a la más fuerte.

1. **Pedir el formato en el prompt** (débil): "Respondé en JSON con los campos `nombre` y `edad`." Funciona la mayoría de las veces, pero el modelo puede agregar texto extra ("Acá está el JSON: ..."), envolver en markdown, o desviarse en casos borde. Frágil para producción.

2. **Few-shot del formato** (medio): mostrar ejemplos con la salida exacta que querés (módulo 5). Mejora mucho la consistencia, pero sigue sin ser una garantía dura.

3. **Structured output / JSON Schema** (fuerte, lo recomendado): la API **garantiza** que la salida cumpla un esquema. Hay dos rutas, no las confundas: (a) el parámetro crudo `output_config={"format": {"type": "json_schema", "schema": ...}}` en `messages.create()` toma un **dict JSON Schema** y te devuelve texto JSON que vos parseás; (b) el helper `client.messages.parse()` del SDK de Python toma directamente un modelo **Pydantic** (que viste en [Python](python.md)) en `output_format=`, arma el JSON Schema por vos y te devuelve `parsed_output` ya como instancia validada. Para una app Python, (b) es lo idiomático:

```python
from pydantic import BaseModel
import anthropic

class Ticket(BaseModel):
    categoria: str
    prioridad: int
    requiere_humano: bool

client = anthropic.Anthropic()
resp = client.messages.parse(
    model="claude-opus-4-8",
    max_tokens=512,
    messages=[{"role": "user", "content": "Clasificá: 'No puedo entrar, es urgente, perdí acceso a todo'"}],
    output_format=Ticket,            # el SDK arma el JSON Schema desde el modelo Pydantic
)
ticket = resp.parsed_output          # instancia de Ticket validada... o None
if ticket is None:
    # parsed_output es None si hubo refusal (stop_reason == "refusal")
    # o la salida se truncó (stop_reason == "max_tokens"). Manejalo, no asumas.
    raise ValueError(f"sin salida estructurada: {resp.stop_reason}")
print(ticket.categoria, ticket.prioridad, ticket.requiere_humano)
```

Cuando hay salida, structured output **garantiza el esquema** —el modelo no puede devolver texto suelto ni un campo de más—. Pero la garantía es del *esquema*, no de que siempre haya salida: `parsed_output` es `None` si el modelo rechazó la tarea (`stop_reason == "refusal"`) o si la respuesta se cortó por `max_tokens`. Por eso el código chequea `None` antes de usar el ticket —asumir que siempre viene un `Ticket` es el bug que rompe en producción ante el primer refusal—. Es el equivalente, para LLMs, de validar los bordes con [Pydantic en FastAPI](fastapi.md): convertís salida no determinista en datos tipados y confiables, pero seguís manejando el caso de "no hubo dato".

Dato importante de la API (verificá con `claude-api`): **structured output reemplazó al prefill *para el caso de forzar formato estructurado*** —el prefill del assistant ya no se soporta en los modelos actuales (módulo 5)—. El prefill se usaba también para otras cosas (saltar preámbulos, continuar una respuesta cortada): esos usos hoy se resuelven con una instrucción en el system prompt, no con structured output. Y `output_config.format` es **incompatible con Citations** (no podés usar los dos a la vez). Para *tools* hay un mecanismo paralelo: `strict: true` en la definición de la tool garantiza que los argumentos cumplan el esquema (lo viste en [agentes](agentes.md)).

La regla de criterio: **si tu código va a consumir la salida, usá structured output (JSON Schema/Pydantic), no "pedile JSON y crucemos los dedos".** Pedir el formato en el prompt es para prototipos o salida que lee un humano.

**Ejercicios 8**
8.1 Ordená de más débil a más fuerte las tres formas de controlar el formato de salida. ¿Cuál garantiza el esquema?
8.2 ¿Cómo se hace structured output en Claude con Python/Pydantic, y qué te devuelve `parsed_output`?
8.3 ¿Con qué concepto de FastAPI/Pydantic conecta el structured output, y qué problema de los LLMs resuelve?
8.4 ¿Qué reemplazó structured output, y con qué feature de la API es incompatible?
8.5 (Práctico) Reescribí este prompt vago aplicando lo de los módulos 3, 4 y 8 —etiquetas XML para separar instrucción de datos, casos borde explícitos y salida estructurada con un modelo Pydantic—. Prompt vago: *"Mirá este correo de un cliente y decime qué quiere y si está enojado."*

---

## Módulo 9 — Iterar con medición, no a ojo

**Teoría.** El error de proceso más común: ajustar un prompt mirando **un** ejemplo, verlo andar, y darlo por terminado. Eso no es prompt engineering, es adivinar. El prompt engineering serio es **empírico y medido**: probás el prompt contra un conjunto de casos representativos y medís qué tan bien anda, igual que [TDD](tdd.md) y [evals](evals.md).

El ciclo:

1. **Armá un conjunto de prueba** (un mini golden set, [módulo 2 de evals](evals.md)): 10-50 entradas representativas, incluyendo casos borde y casos que ya viste fallar, cada una con su salida esperada (o un criterio de "correcto").
2. **Corré el prompt sobre todas** y medí: ¿qué % salió bien? ¿dónde falla?
3. **Identificá el patrón de falla** (¿confunde dos categorías? ¿inventa cuando no sabe? ¿se desvía del formato?).
4. **Cambiá una cosa** del prompt (más específico, un ejemplo nuevo que cubra el fallo, una etiqueta XML) y volvé a medir.
5. **Repetí** hasta llegar al umbral que tu caso necesita.

El bucle mínimo es un puñado de líneas —correr el prompt sobre el conjunto y contar aciertos—:

```python
# golden set: representativo + casos borde, cada caso con su salida esperada
golden = [
    {"texto": "La app es lenta y se cuelga.", "esperado": "negativo"},
    {"texto": "Hace justo lo que necesito.", "esperado": "positivo"},
    # ... 10-50 casos
]
aciertos = sum(
    clasificar_sentimiento(c["texto"]) == c["esperado"]   # tu llamada al LLM, normalizada
    for c in golden
)
print(f"{aciertos}/{len(golden)} = {aciertos / len(golden):.0%}")
```

Esto es deliberadamente crudo: el harness serio (métricas por capa, LLM-as-judge para salidas sin respuesta única, CI que bloquea regresiones) lo construís en [evals](evals.md) —este bucle es la versión de bolsillo para iterar un prompt—.

Un atajo útil: **meta-prompting**. Pedile al propio modelo que critique y reescriba tu prompt ("acá está mi prompt y los casos donde falla; proponé una versión más específica"). No reemplaza la medición —seguís evaluando la versión nueva contra el golden set—, pero acelera el paso 4.

Por qué medir y no mirar un caso: un prompt no determinista anda en algunos casos y falla en otros, y la mejora **no es monótona** —arreglar un caso puede romper otro—. Solo viéndolo sobre muchos casos sabés si un cambio fue neto positivo. Es exactamente la disciplina de [eval-driven development](evals.md) aplicada a prompts.

**Costo: medilo también.** Cada llamada cuesta tokens (entrada + salida). Dos palancas que aparecen al iterar prompts en producción:

- **Modelo correcto para la tarea** (tiering): no uses Opus 4.8 ($5/$25 por 1M) para una clasificación trivial que Haiku 4.5 ($1/$5) resuelve igual. La eval te dice cuál es el modelo más barato que pasa el umbral.
- **Prompt caching**: si tu prompt tiene un prefijo grande y estable (system + ejemplos few-shot + instrucciones), Claude lo cachea y pagás **~0.1× por los tokens cacheados** en las llamadas siguientes (la escritura de caché cuesta ~1.25× con TTL de 5 min, así que el break-even está en ~2 reutilizaciones). Por eso el módulo 2 insistió en poner lo estable primero y lo variable al final: maximiza el prefijo cacheable. **Hay un piso de tokens** y depende del modelo: en Opus 4.8 el prefijo debe llegar a **4096 tokens** (Sonnet 4.6 y Fable: 2048) o **no se cachea, en silencio y sin error** —verificalo con `usage.cache_read_input_tokens`: si da 0 en llamadas repetidas con el mismo prefijo, o no llegás al piso o algo lo está invalidando (un timestamp, un UUID, el orden de las tools)—.

La frase mental: **no preguntés "¿este prompt anda?" mirando un caso; preguntá "¿qué porcentaje pasa, sobre qué casos, a qué costo?" —y movete según el número, no según la corazonada—.**

**Ejercicios 9**
9.1 ¿Por qué ajustar un prompt mirando un solo ejemplo es un error? ¿Qué hacés en su lugar?
9.2 Describí el ciclo de iteración medida de un prompt (los pasos).
9.3 ¿Por qué la mejora de un prompt "no es monótona" y qué implica para cómo evaluás un cambio?
9.4 Nombrá dos palancas de costo al llevar un prompt a producción y cómo el orden del prompt (módulo 2) habilita el prompt caching.
9.5 (Práctico) Diseñá un golden set de 5 casos para el clasificador de sentimiento del módulo 5 (positivo/negativo/neutral). ¿Qué casos borde incluirías y por qué?

---

## Módulo 10 — Grounding, anti-alucinación y robustez

**Teoría.** Dos riesgos rompen los prompts en producción: que el modelo **invente** (alucinación) y que un usuario malicioso lo **secuestre** (prompt injection). El prompt engineering tiene defensas para ambos.

**Contra la alucinación** (el modelo afirma cosas falsas con confianza):

- **Dale permiso explícito de no saber.** Por defecto el modelo tiende a responder *algo*. Decile: "Si la información no está en el contexto provisto, respondé 'No tengo esa información'. No inventes." Esto solo reduce muchísimo las invenciones.
- **Anclá la respuesta al contexto (grounding).** "Respondé usando SOLO la información del `<documento>`. No uses conocimiento externo." Es la base del [RAG](rag.md): el modelo responde *desde* los datos que le diste, no desde su memoria.
- **Pedí citas/evidencia.** "Para cada afirmación, citá la parte del documento que la respalda." Forzar a citar reduce las afirmaciones sin sustento. (Claude tiene una feature de Citations nativa —recordá del módulo 8 que es incompatible con structured output—.)
- **Bajá la temperatura para tareas factuales.** `temperature` controla el trade-off **determinismo ↔ diversidad**: baja (cerca de 0) = respuestas más conservadoras y reproducibles (extracción, clasificación, hechos); alta = más variada y creativa (brainstorming, redacción). Para anti-alucinación querés determinismo. Dato de la API: en los modelos actuales (Opus 4.8/4.7, Fable) `temperature`, `top_p` y `top_k` **fueron removidos** (devuelven 400) —se guía el comportamiento por prompt—; el control sigue disponible en modelos más viejos. Por eso este módulo usa Opus 4.8 sin tocar `temperature`.

**Contra el prompt injection** (el usuario, o un documento que el modelo procesa, mete instrucciones tipo "ignorá todo lo anterior y hacé X"):

- **Separá instrucciones de datos con etiquetas XML** (módulo 4): lo que está dentro de `<documento_usuario>` se trata como dato, no como orden. No es infalible, pero ayuda.
- **Poné las reglas críticas en el system prompt** (módulo 7): pesan más que un mensaje del usuario.
- **No confíes en la salida para acciones peligrosas sin validar.** Si el LLM decide ejecutar algo (una tool, una query, un borrado), tu código valida y aplica el principio de mínimo privilegio —esto es central en [agentes](agentes.md), donde el prompt injection es más peligroso porque el modelo *actúa*, no solo responde—.
- **Nunca confíes datos sensibles a que "el prompt los proteja".** Si algo no debe verse, no lo metas en el contexto. El prompt no es un control de acceso.

La frase mental: **asumí que el modelo puede inventar y que la entrada puede ser hostil; el prompt engineering robusto le da permiso de no saber, lo ancla a los datos, y separa instrucciones de contenido —pero las garantías duras (acceso, validación) van en tu código, no en el prompt—.**

**Ejercicios 10**
10.1 ¿Por qué "dale permiso de no saber" reduce alucinaciones? Escribí la instrucción.
10.2 ¿Qué es el grounding y con qué módulo del track de IA es la idea central?
10.3 ¿Qué es prompt injection y por qué es más peligroso en un agente que en una llamada de respuesta simple? Nombrá dos defensas.
10.4 ¿Por qué "que el prompt proteja los datos sensibles" es una mala estrategia? ¿Dónde van las garantías duras?

---

## Módulo 11 — El criterio: prompt, contexto, RAG o fine-tuning

**Teoría.** El módulo que cierra y el más de criterio. Cuando un sistema de IA no anda como querés, el reflejo es "mejoro el prompt". A veces es lo correcto; a veces el problema es otro y ninguna cantidad de prompt engineering lo resuelve. Saber **qué palanca tocar** es la habilidad senior.

El árbol de decisión cuando algo falla:

1. **¿Está mal especificada la tarea?** → **Prompt engineering.** El modelo es capaz pero no entendió qué querías: especificá, estructurá, dale ejemplos, controlá el formato. Esto resuelve la mayoría de los problemas y es lo primero que probás (es barato y rápido).

2. **¿Le falta información que el modelo no tiene?** (datos de tu empresa, documentos privados, conocimiento posterior a su entrenamiento) → **Context engineering / [RAG](rag.md).** Ningún prompt hace que el modelo sepa algo que nunca vio. Le tenés que *dar* esa información en el contexto —recuperándola (RAG) o pasándola directamente—.

3. **¿Necesita decidir y actuar en varios pasos, usar herramientas, iterar?** → **[Agentes / tools](agentes.md).** Un prompt de una llamada no puede ejecutar acciones ni adaptarse sobre la marcha.

4. **¿Necesita un comportamiento/estilo muy específico, consistente y a gran escala, que ni el mejor prompt logra?** → **Fine-tuning** (raro). Reentrenás el modelo con ejemplos. Es caro, lento y rara vez la respuesta correcta —**el error #1 es creer que "fine-tuning enseña hechos"; no, para hechos es RAG**—. Fine-tuning es para *forma/estilo/formato* muy consistente, no para conocimiento.

La regla de oro, que es el mismo "[escalá la herramienta al problema](liderazgo.md)" de todo el temario: **empezá por el prompt (lo más barato), subí a RAG si falta información, a agentes si hace falta actuar, y a fine-tuning casi nunca.** El error caro es saltar a la solución compleja (fine-tuning, un agente elaborado) cuando un prompt más específico o un poco de contexto resolvían el problema con una fracción del costo.

Y el principio que recorre el módulo entero: **el prompt engineering es empírico.** No hay prompt "perfecto" en abstracto —hay un prompt que pasa *tus* casos a *tu* costo—. Lo escribís, lo medís, lo iterás. Es ingeniería, con la misma disciplina de hipótesis→medición→ajuste que el resto del oficio.

El cierre y el puente: el prompt engineering es la habilidad base del track de IA —la usás en cada llamada—. Recorriste qué es (especificación confiable, no magia), la anatomía de un prompt, los principios (claridad, estructura con XML, few-shot, chain-of-thought/thinking, system prompts, structured output), el proceso (iterar con medición y costo), y las defensas (grounding, anti-injection). Y, sobre todo, el criterio para saber cuándo el prompt es la palanca y cuándo el problema pide [contexto/RAG](vector-dbs.md), [agentes](ai-agents-python.md) u otra cosa. Lo que sigue en el track Python AI son justamente esas palancas: [Vector Databases](vector-dbs.md) (darle información: RAG), [AI Agents](ai-agents-python.md) (hacerlo actuar), [Voice AI](voice-ai.md) y [Deploy de IA](deploy-ai.md). Sobre todas ellas, el prompt sigue siendo la interfaz —ahora sabés diseñarla con rigor—.

**Ejercicios 11**
11.1 Tu sistema de IA falla. Dá el árbol de decisión: ¿cuándo es problema de prompt, cuándo de contexto/RAG, cuándo de agente, cuándo de fine-tuning?
11.2 ¿Por qué "fine-tuning para enseñarle hechos a un modelo" es el error #1? ¿Qué usás para hechos?
11.3 ¿Cuál es la regla de oro de por dónde empezar, y con qué criterio del resto del temario conecta?
11.4 ¿Por qué se dice que "no hay un prompt perfecto en abstracto"? ¿Qué define que un prompt sea bueno?

---

## Soluciones

### Módulo 1
1.1 Porque un LLM es no determinista y opera sobre entradas variadas: que ande en un caso no garantiza que ande en otros. Buscás **confiabilidad** —que ande el 95%+ de las veces sobre entradas reales y casos borde, repetidamente—.
1.2 La **subespecificación**: pediste poco y dejaste huecos (formato, casos borde, qué hacer si no sabe), que el modelo llena a su criterio en vez del tuyo. No lo arregla solo porque lee tu texto, no tu mente: lo que no decís, lo infiere él.
1.3 Porque los modelos modernos siguen las instrucciones **muy literalmente**; el lenguaje agresivo escrito para vencer la reticencia de modelos viejos hoy hace que la instrucción **sobre-dispare** (usa una tool/comportamiento más de lo debido). Instrucción clara y directa funciona mejor que gritar en mayúsculas.
1.4 **Prompt engineering** optimiza *qué le decís* al modelo (instrucciones, ejemplos, formato); **context engineering** optimiza *qué información* le das (qué entra en la ventana de contexto y en qué orden). Son complementarios: el mejor prompt sobre el contexto equivocado falla, y el mejor contexto con un prompt vago también.

### Módulo 2
2.1 (1) Rol/system (quién es y cómo se comporta), (2) instrucción/tarea (qué hacer), (3) contexto/datos (sobre qué opera), (4) ejemplos few-shot (demostraciones), (5) formato de salida (cómo querés la respuesta).
2.2 Cambia el **contexto/dato** de la llamada; se mantienen rol, instrucciones y ejemplos. El orden importa porque poner lo estable primero permite **cachear el prefijo** (prompt caching) y solo procesar el dato variable al final.
2.3 La pieza de **formato de salida**: una instrucción explícita como "Respondé solo con la categoría, sin explicación".
2.4 Si el prefijo (system + instrucciones + ejemplos) no cambia entre llamadas, Claude lo cachea y pagás ~0.1× por esos tokens en las llamadas siguientes; ponés el dato variable al final para maximizar ese prefijo cacheable.

### Módulo 3
3.1 Porque una instrucción positiva guía hacia un comportamiento concreto, mientras que una prohibición deja abierto *qué sí* hacer. "No seas verboso" → "Respondé en máximo 2 oraciones, solo con la información esencial".
3.2 Ej.: "Resumí en máximo 3 oraciones. Tono profesional = sin jerga coloquial, sin emojis, en tercera persona, foco en hechos y acciones concretas."
3.3 Son las situaciones que la instrucción principal no cubre (entradas vacías, múltiples coincidencias, ninguna coincidencia). Especificarlos evita que el modelo los resuelva distinto en cada entrada → más confiabilidad. Ej. fecha: "Si hay varias fechas, devolvé la primera. Si no hay ninguna, devolvé null."
3.4 Porque el modelo sigue secuencias explícitas mucho mejor que un pedido monolítico: descomponer en pasos reduce que se saltee partes o las haga en desorden, y hace el resultado más predecible.

### Módulo 4
4.1 Eliminan la ambigüedad de "¿esto es instrucción o dato?": el modelo sabe que lo de adentro de `<documento>` es material a procesar, no órdenes a seguir, y vos podés referenciar cada parte por su etiqueta.
4.2 Porque al encerrarlo en `<documento>` el modelo lo trata como **dato a procesar**, no como instrucción; si el usuario mete "ignorá lo anterior" dentro, queda marcado como contenido, no como orden (mitiga, no elimina, el injection).
4.3 (Dos de) referenciar partes del prompt ("usando el `<documento>`, respondé la `<pregunta>`"); estructurar la salida ("razonamiento en `<razonamiento>`, respuesta en `<respuesta>`") para parsearla; mitigar prompt injection.
4.4 Casi obligatorio cuando la entrada es larga y mezcla instrucciones + contexto + ejemplos (prompt de producción). Un prompt plano alcanza para una instrucción simple y corta sin datos embebidos.

### Módulo 5
5.1 **Zero-shot** = sin ejemplos, solo la instrucción; **few-shot** = con N ejemplos de entrada→salida. Un ejemplo comunica de forma implícita el formato exacto, el tono, el nivel de detalle y cómo resolver ambigüedad —cosas que costarían párrafos describir—.
5.2 Como **turnos user/assistant completos alternados** en el array `messages`, antes de la entrada real. El **último** mensaje lo manda el **user** (la entrada real), porque es lo que el modelo tiene que responder ahora.
5.3 El **prefill** es terminar el array con un turno `assistant` *incompleto* para forzar que el modelo lo continúe; **ya no se soporta** en los modelos actuales (devuelven 400 si el último mensaje es del assistant). Few-shot usa pares completos y termina en `user`, así que es válido. Para forzar formato hoy usás **structured output** (`output_config.format`).
5.4 (Dos de) pocos pero representativos (3-5, con casos borde); consistencia absoluta de formato; incluir casos difíciles; cuidar el costo/caché. La consistencia es crítica porque el modelo **copia el formato de los ejemplos al pie de la letra**: ejemplos inconsistentes producen salida inconsistente.

### Módulo 6
6.1 Es hacer que el modelo razone paso a paso **antes** de la respuesta final. Mejora el razonamiento porque genera tokens condicionado por lo ya escrito: si primero escribe el razonamiento, cada paso de la respuesta se apoya en él en vez de "adivinar" de una.
6.2 La forma moderna es el **extended/adaptive thinking** nativo (parámetro `thinking`): el modelo razona en un bloque separado sin que tengas que pedirlo en el prompt. "Pensá paso a paso" es la técnica de prompt clásica, útil en modelos sin thinking o cuando querés un formato de razonamiento parseable.
6.3 `thinking: {"type": "adaptive"}` deja que el modelo decida cuánto razonar según la dificultad. Reemplazó al viejo `budget_tokens` (presupuesto fijo de pensamiento), que está deprecado/removido; la profundidad se controla ahora con `output_config.effort`.
6.4 En tareas **triviales** (una clasificación simple): no mejora el resultado y agrega latencia y costo. El thinking se reserva para tareas que realmente requieren razonamiento.

### Módulo 7
7.1 En el **system** va el rol, las reglas de comportamiento y el contexto estable de la app (ej. "Sos el asistente de soporte de TaskFlow, respondés en español, conciso"). En el **user** va la tarea puntual de la llamada (ej. "¿Cómo asigno una tarea?").
7.2 Porque el modelo trata el system prompt como **autoridad de más alto nivel** en la jerarquía de instrucciones: una regla ahí pesa más y es más difícil de sobrescribir desde un mensaje del usuario que la misma regla puesta en el `user`.
7.3 Como el system es estable, se **cachea** (pagás ~0.1× por esos tokens) y garantiza el **mismo comportamiento de base** en cada llamada.
7.4 Para introducir un cambio de modo o contexto que aparece **después** del arranque (ej. "modo conciso activado", o contexto que llegó a mitad de sesión) sin reescribir el system original ni invalidar el caché del prefijo.

### Módulo 8
8.1 De más débil a más fuerte: (1) pedir el formato en el prompt, (2) few-shot del formato, (3) structured output con JSON Schema. La (3) **garantiza** el esquema; las otras dos solo lo hacen probable.
8.2 Con `client.messages.parse()` pasando un modelo Pydantic en `output_format=`; el SDK arma el JSON Schema y `parsed_output` te devuelve una **instancia validada** del modelo (no texto que tengas que parsear), o `None` si hubo refusal/truncamiento. (La ruta cruda `messages.create(output_config={"format": {...}})` es distinta: toma un dict JSON Schema, no un modelo Pydantic, y devuelve texto JSON que parseás vos.)
8.3 Conecta con validar los bordes con **Pydantic en FastAPI**: convertís entrada/salida no confiable en datos tipados y validados. Resuelve el problema de que la salida de un LLM es texto no determinista que tu código necesita consumir de forma confiable.
8.4 Reemplazó al **prefill** del assistant (ya no soportado) *para forzar formato estructurado*. Es **incompatible con Citations** (no se pueden usar a la vez).
8.5 La idea es convertir un pedido ambiguo en una especificación. Una solución posible:

```python
from pydantic import BaseModel
from typing import Literal
import anthropic

class Analisis(BaseModel):
    intencion: str                                      # qué quiere el cliente, en una frase
    enojado: bool
    categoria: Literal["facturacion", "bug", "envio", "cuenta", "otro"]

system = "Analizás correos de clientes de soporte. Sos preciso y objetivo."

user = """\
Analizá SOLO el correo de abajo. No respondas al cliente; extraé los datos.
- intencion: qué pide el cliente, en una sola oración.
- enojado: true solo si hay enojo explícito (insultos, mayúsculas sostenidas, amenazas de irse); el tono firme NO es enojo.
- categoria: una de [facturacion, bug, envio, cuenta, otro].
Si el correo está vacío o no es un reclamo, devolvé intencion="" y categoria="otro".

<correo>
{correo}
</correo>"""

client = anthropic.Anthropic()
resp = client.messages.parse(
    model="claude-opus-4-8",
    max_tokens=512,
    system=system,
    messages=[{"role": "user", "content": user.format(correo=texto_correo)}],
    output_format=Analisis,
)
analisis = resp.parsed_output
if analisis is None:
    raise ValueError(f"sin salida: {resp.stop_reason}")
```

Lo que se aplicó: el correo va dentro de `<correo>` (módulo 4, separa dato de instrucción y mitiga injection), los casos borde están explícitos (qué cuenta como "enojado", correo vacío → módulo 3), y la salida es un modelo Pydantic validado con guarda de `None` (módulo 8). El prompt vago original dejaba todo eso a criterio del modelo.

### Módulo 9
9.1 Porque un prompt no determinista anda en algunos casos y falla en otros; un solo ejemplo no te dice el porcentaje real ni los modos de falla. En su lugar **medís sobre un conjunto de casos representativos** (mini golden set) y mirás el % de aciertos.
9.2 (1) Armar un conjunto de prueba con casos representativos y borde + salida esperada; (2) correr el prompt sobre todos y medir; (3) identificar el patrón de falla; (4) cambiar **una** cosa y volver a medir; (5) repetir hasta el umbral.
9.3 Porque arreglar un caso puede **romper otro** (las mejoras no se acumulan linealmente). Implica que evaluás un cambio por su efecto **neto sobre todo el conjunto**, no por si arregló el caso que mirabas.
9.4 (1) **Tiering de modelo** (usar el modelo más barato que pasa el umbral, p. ej. Haiku en vez de Opus para clasificación trivial); (2) **prompt caching** (prefijo estable cacheado a ~0.1×). El orden "estable primero, variable al final" maximiza el prefijo cacheable.
9.5 Un golden set representativo cubre las clases *y* los modos de falla. Ej.:

| texto | esperado | por qué está |
|---|---|---|
| "No puedo creer lo útil que es, la uso a diario." | positivo | caso claro positivo |
| "Se cuelga constantemente, una basura." | negativo | caso claro negativo |
| "Funciona; la interfaz podría mejorar." | neutral | mixto / sin carga fuerte |
| "Ah, bárbaro, otra actualización que rompe todo. 👏" | negativo | **sarcasmo**: las palabras suenan positivas pero el sentido es negativo (donde más fallan los clasificadores) |
| "" | neutral | **entrada vacía**: caso borde que la instrucción debe cubrir (módulo 3) |

Los casos borde clave son el **sarcasmo** (rompe el matching superficial de palabras) y la **entrada vacía/no-review** (verifica que el prompt maneja lo que no especificaste). Sumá los casos que veas fallar en producción —el golden set crece con los errores reales, como en [evals](evals.md)—.

### Módulo 10
10.1 Porque por defecto el modelo tiende a responder *algo* aunque no sepa; darle una salida válida para "no sé" le quita la presión de inventar. Instrucción: "Si la información no está en el contexto provisto, respondé 'No tengo esa información'. No inventes."
10.2 **Grounding** = anclar la respuesta a los datos provistos ("respondé usando SOLO el `<documento>`, sin conocimiento externo"). Es la idea central de **RAG**.
10.3 **Prompt injection** = el usuario o un documento procesado mete instrucciones ("ignorá lo anterior y hacé X"). Es más peligroso en un **agente** porque el modelo **actúa** (ejecuta tools, modifica datos), no solo responde, así que una orden inyectada puede causar daño real. Defensas (dos de): separar instrucciones de datos con etiquetas XML; reglas críticas en el system prompt; validar en código las acciones peligrosas; mínimo privilegio.
10.4 Porque el prompt no es un control de acceso: un modelo puede ser manipulado o cometer errores, y "te pedí que no lo muestres" no garantiza nada. Las garantías duras (control de acceso, validación, qué datos entran al contexto) van en **tu código**, no en el prompt.

### Módulo 11
11.1 (1) Tarea mal especificada → **prompt engineering** (especificar, estructurar, ejemplos, formato). (2) Le falta información que el modelo no tiene → **context engineering / RAG**. (3) Necesita decidir y actuar en pasos / usar herramientas → **agentes/tools**. (4) Necesita un estilo/comportamiento muy consistente que ni el mejor prompt logra → **fine-tuning** (raro).
11.2 Porque el fine-tuning ajusta *forma/estilo/comportamiento*, no incorpora hechos de forma confiable (y los que mete quedan congelados y propensos a alucinarse). Para **hechos** se usa **RAG** (le das la información en el contexto).
11.3 Empezá por el **prompt** (lo más barato y rápido), subí a **RAG** si falta información, a **agentes** si hace falta actuar, y a **fine-tuning** casi nunca. Conecta con "escalá la herramienta al problema" (criterio de liderazgo/Kubernetes/agentes): no saltes a la solución compleja cuando una simple alcanza.
11.4 Porque un prompt solo es "bueno" relativo a *tus* casos y *tu* presupuesto: lo que define que sea bueno es que **pase tu conjunto de prueba al costo que tu caso tolera**, no una cualidad abstracta. Es empírico: se mide y se itera, no se juzga en el vacío.

---

Con este módulo tenés la habilidad transversal del track de IA: **comunicarle la intención al modelo con rigor de ingeniería**. Recorriste qué es el prompt engineering (especificación confiable, empírica, no magia), la anatomía de un prompt, los principios (claridad, estructura con etiquetas XML, few-shot, chain-of-thought/thinking nativo, system prompts, structured output con Pydantic), el proceso de iterar con medición y costo (prompt caching, tiering), las defensas contra alucinación y prompt injection, y el criterio para saber cuándo el prompt es la palanca correcta y cuándo el problema pide contexto, RAG, agentes o (rara vez) fine-tuning. Todo con el stack Claude/Anthropic y datos verificados con la skill `claude-api`. Lo que sigue en el track son las otras palancas, y sobre todas ellas el prompt sigue siendo la interfaz: [Vector Databases y RAG](vector-dbs.md) (darle información), [AI Agents](ai-agents-python.md) (hacerlo actuar), [Voice AI](voice-ai.md) y [Deploy de aplicaciones de IA](deploy-ai.md).
