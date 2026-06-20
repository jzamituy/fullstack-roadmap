# Evaluations: medir un sistema de IA no determinista

**Evals para LLMs, RAG y agentes · el "testing + observabilidad" de la IA · Claude / Anthropic · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **cuarto y último módulo del track de IA**: asume que ya hiciste [LLMs](ia-llms.md), [RAG](rag.md) y [Agentes](agentes.md), y que conocés [Testing práctico](testing.md) (Jest/Vitest, CI). Las **evals** son lo que separa un prototipo de IA de un sistema de producción: la forma de **medir si tu sistema LLM realmente funciona**. El problema central es que un LLM es **no determinista** —la misma entrada puede dar salidas distintas—, así que no podés testearlo con `assert(salida === esperado)`. Necesitás una disciplina nueva: datasets de referencia, métricas que toleran variación, y jueces automáticos. Es el **equivalente a tests + observabilidad** para algo que no da siempre el mismo resultado.

**Lo que asumimos.** TS, testing (lo del módulo de Testing), el track de IA completo (sobre todo el structured output del módulo 6 de LLMs y el módulo 11 de RAG donde asomaron recall@k y faithfulness), CI/CD, y nociones de observabilidad.

**Para practicar.** El SDK de Claude (para jueces y para el sistema bajo prueba) y, opcionalmente, una herramienta de tracing como Langfuse:

```bash
npm i @anthropic-ai/sdk
export ANTHROPIC_API_KEY="sk-ant-..."
# Langfuse (open source) es una opción para tracing/evals; hay otras.
```

> Nota sobre datos volátiles: IDs/precios de modelos y nombres de herramientas (Langfuse, etc.) cambian; verificá al implementar.

**Índice de módulos**
1. Por qué evaluar: el problema de lo no determinista
2. El golden set: tu conjunto de referencia
3. Dos familias de métricas: code-based vs LLM-as-judge
4. LLM-as-judge por dentro
5. Evaluar por capas: componente vs end-to-end
6. Eval-driven development: el TDD de la IA
7. Offline vs online: evals en CI y en producción
8. Observabilidad y tracing
9. Guardrails: la última línea en producción
10. El criterio: qué medir, cuánto y cuándo

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Por qué evaluar: el problema de lo no determinista

**Teoría.** En el módulo de Testing aprendiste a verificar código con asserts exactos: `expect(suma(2,2)).toBe(4)`. Eso funciona porque el código es **determinista**: misma entrada, misma salida, siempre. Un LLM rompe esa premisa (módulo 1 de LLMs): **la misma entrada puede dar salidas distintas**, y "correcto" no es una sola string, sino un rango de respuestas aceptables. `expect(respuesta).toBe("...")` es inservible: la próxima corrida da otra redacción igual de válida y tu test "falla" sin que nada esté roto.

Las **evals** son la respuesta a ese problema. Una eval es una forma sistemática de medir la calidad de las salidas de tu sistema de IA, tolerando la variación. La diferencia con un test tradicional:

| | Test (código) | Eval (LLM) |
|---|---|---|
| Salida correcta | Una, exacta | Un rango de aceptables |
| Resultado | Pasa / falla | Un **score** (ej. 0.87) |
| Determinismo | Sí | No (corrés varias veces) |
| Compara contra | Valor esperado | Criterios / referencia / juez |

Por qué no es opcional en producción:

- **No podés mejorar lo que no medís.** Cambiás el prompt, el modelo o el chunking "a ojo" y no sabés si mejoró o empeoró. Es el "medí antes de optimizar" del archivo senior, ahora sobre algo que **no tiene** una métrica obvia.
- **Las regresiones son invisibles.** Tocás un prompt para arreglar un caso y rompés otros tres sin enterarte —no hay un compilador ni un test rojo que te avise—. Solo una eval lo detecta.
- **Lo no determinista te exige medir en agregado.** Una sola corrida no te dice nada (pudo salir bien o mal por azar). Medís sobre **muchos casos** y mirás la tasa de acierto, no un caso suelto.

La frase mental: **una eval es un test que devuelve un score sobre un conjunto, no un pasa/falla sobre un caso.** Construir esa capacidad es lo que te deja iterar tu sistema de IA con confianza en vez de a ciegas.

**Ejercicios 1**
1.1 ¿Por qué no podés testear un LLM con `assert(salida === esperado)`? Conectá con una propiedad del módulo de LLMs.
1.2 Nombrá dos diferencias entre un test tradicional y una eval.
1.3 ¿Por qué hay que medir sobre muchos casos y no sobre uno solo?

---

## Módulo 2 — El golden set: tu conjunto de referencia

**Teoría.** No podés evaluar contra el vacío: necesitás un **golden set** (o *eval dataset*), el conjunto de casos de prueba contra los que medís tu sistema. Es el cimiento de todo lo demás —sin un buen dataset, las métricas más sofisticadas miden ruido—. Cada caso tiene, según el sistema:

- **Una entrada** (la pregunta del usuario, el documento a procesar).
- **Una salida esperada o un criterio de éxito** (la respuesta correcta, o "debe mencionar X y no inventar Y"). A veces es una respuesta exacta; muchas veces es una **rúbrica** de qué hace buena a una respuesta.
- Para RAG: además, **qué chunk(s) contienen la respuesta** (para medir recall@k, módulo 11 de RAG).

Cómo se arma uno bueno —y esto es trabajo de ingeniería, no un detalle—:

- **Representativo del tráfico real.** Si tus usuarios preguntan cosas cortas y ambiguas, tu golden set debe tenerlas; un dataset de preguntas perfectas y largas te da métricas optimistas y falsas.
- **Cubre los casos borde y los modos de falla.** No solo el "camino feliz": las preguntas sin respuesta en la data (¿el sistema dice "no sé" o alucina?), las ambiguas, las maliciosas (prompt injection), las de permisos cruzados (multi-tenant). Los casos donde tu sistema *podría* fallar son los más valiosos.
- **Empezá chico y crecé.** 20-50 casos bien elegidos ya te dan señal; no necesitás miles para arrancar. Y crecé el set con los **fallos reales de producción**: cada bug que aparece se vuelve un caso nuevo del golden set (igual que en testing escribís un test que reproduce el bug antes de arreglarlo).

De dónde sale: lo curás a mano (lo más confiable), lo sacás de logs de producción reales, o —con cuidado— lo generás sintéticamente con un LLM y lo revisás. La trampa a evitar: un golden set generado por el mismo tipo de modelo que evaluás puede tener los mismos sesgos. La revisión humana del dataset no es opcional.

El principio: **el golden set es tu definición operacional de "bueno".** Si está mal armado —no representativo, sin casos borde— vas a optimizar para la métrica equivocada y tu sistema va a "mejorar" en la eval mientras empeora para usuarios reales (el clásico "optimizar la métrica en vez del objetivo").

**Ejercicios 2**
2.1 ¿Qué es un golden set y qué tiene cada caso (en general y en RAG)?
2.2 ¿Por qué es clave que sea representativo del tráfico real y que cubra casos borde? Da un caso borde concreto.
2.3 ¿De dónde crece un golden set con el tiempo, y qué práctica del módulo de Testing se parece?

---

## Módulo 3 — Dos familias de métricas: code-based vs LLM-as-judge

**Teoría.** ¿Cómo puntuás una salida? Hay **dos familias** de métricas, y elegir la correcta para cada cosa es la mitad del trabajo.

**1. Métricas basadas en código (deterministas).** Cuando la corrección se puede chequear con código, hacelo —es barato, rápido, exacto y reproducible—:

- **Exact match / regex**: la salida es una de N categorías (clasificación), o contiene cierto patrón.
- **Validación de schema**: la salida es JSON válido contra tu schema (el structured output del módulo 6 de LLMs es validable así).
- **Métricas de retrieval**: recall@k, precision@k (módulo 11 de RAG) — pura comparación de IDs.
- **Métricas operacionales**: latencia, costo en tokens (módulo de costo de LLMs), tasa de error. Estas siempre se miden por código. Ojo con la latencia: medila en **percentiles (p95/p99)**, no en promedio —el promedio esconde la cola lenta que el usuario sí siente—, igual que en el módulo de observabilidad.

**2. LLM-as-judge (basadas en modelo).** Cuando la corrección es subjetiva o sobre texto libre —"¿esta respuesta es fiel al contexto?", "¿este resumen es bueno?", "¿el tono es apropiado?"— no hay regex que lo capture. Ahí usás **otro LLM como juez** que puntúa la salida según una rúbrica (módulo 4). Es más caro y menos exacto, pero captura calidad que el código no puede.

La regla de oro: **usá código siempre que puedas; LLM-as-judge solo cuando no se pueda con código.** Es el mismo criterio del módulo 11 de LLMs ("no metas un LLM donde una regex alcanza") aplicado a la evaluación. Un error común es usar un LLM caro para juzgar algo que un `===` resolvía (¿la categoría es "factura"? es exact match, no necesitás un juez).

En la práctica, un sistema serio combina ambas: las métricas de código corren en cada commit (rápidas, baratas, en CI), y las de LLM-as-judge en un set más chico o con menos frecuencia (caras). Y muchas métricas de código —recall@k, latencia, costo— son **no negociables**: siempre las medís, porque son objetivas y baratas.

**Ejercicios 3**
3.1 Nombrá dos métricas basadas en código y dos cosas que solo un LLM-as-judge puede evaluar.
3.2 ¿Cuál es la regla de oro para elegir entre las dos familias, y con qué criterio del módulo de LLMs conecta?
3.3 Da un ejemplo de usar mal un LLM-as-judge donde el código alcanzaba.

---

## Módulo 4 — LLM-as-judge por dentro

**Teoría.** El **LLM-as-judge** es la técnica clave para evaluar calidad subjetiva a escala: un LLM (idealmente uno capaz, como Opus) recibe una salida y la **puntúa según una rúbrica**. Pero un juez ingenuo da scores ruidosos e inconsistentes. Lo que lo hace confiable:

- **Una rúbrica clara y específica.** No "¿es buena la respuesta?" (vago, cada corrida puntúa distinto), sino criterios explícitos: "Puntuá la fidelidad de 1 a 5: 5 = toda afirmación está respaldada por el contexto; 1 = contradice o inventa información". Cuanto más específica la rúbrica, más reproducible el juez.
- **Structured output para el score.** Pedile al juez la salida como JSON `{ score, razon }` (el módulo 6 de LLMs): garantiza que recibís un número parseable más la justificación, no texto libre que tenés que parsear. La `razon` además te da trazabilidad de *por qué* puntuó así.

```ts
import { z } from "zod";
import { zodOutputFormat } from "@anthropic-ai/sdk/helpers/zod";
// `client` es el cliente Anthropic del módulo 6 de LLMs

const RubricaSchema = z.object({
  score: z.number().min(1).max(5),
  razon: z.string(),
});

async function juezFidelidad(contexto: string, respuesta: string) {
  const res = await client.messages.parse({
    // Idealmente, un modelo DISTINTO al que generó la respuesta (evita self-preference)
    model: "claude-opus-4-8",
    max_tokens: 1024,
    messages: [{
      role: "user",
      content: `Evaluá la FIDELIDAD de la respuesta al contexto, de 1 a 5.
5 = toda afirmación está respaldada por el contexto; 1 = inventa o contradice.

<contexto>${contexto}</contexto>
<respuesta>${respuesta}</respuesta>`,
    }],
    output_config: { format: zodOutputFormat(RubricaSchema) },
  });

  // parsed_output es NULLABLE: null si el parseo falla o el juez rechaza
  // (stop_reason: "refusal"). Un score ausente NO es 0 — es "re-evaluar/descartar".
  if (res.parsed_output === null) {
    throw new Error("El juez no devolvió un score válido (parseo fallido o refusal)");
  }
  return res.parsed_output; // { score, razon } validado
}
```

Dos detalles que el código deja ver y la teoría no debe omitir: `parsed_output` es **nullable** —el juez también es un LLM y puede fallar el parseo o rechazar la tarea (`stop_reason: "refusal"`)—; tratá el score ausente como un caso a re-evaluar o descartar, nunca como 0. Y el structured output es **incompatible con la feature Citations** (devuelve 400): si querés que el juez señale en qué parte del contexto se apoya, ponelo como un campo más del JSON (`evidencia: string`), no con Citations.

Los **sesgos** del juez que tenés que conocer (un juez es un LLM, con sus debilidades):

- **Sesgo de posición**: en comparaciones A-vs-B, tiende a preferir la primera (o la última). Mitigación: aleatorizá el orden, o promediá las dos posiciones.
- **Sesgo de verbosidad**: tiende a puntuar mejor las respuestas largas aunque no sean mejores. Mitigación: rúbrica explícita que penalice el relleno.
- **Self-preference**: un modelo tiende a preferir salidas de su misma familia. Mitigación: usá un juez de modelo distinto al evaluado cuando puedas.

Y la regla que cierra el círculo: **al juez también hay que evaluarlo.** ¿Cómo sabés que tu juez puntúa bien? Lo calibrás contra un conjunto de casos puntuados **por humanos** (unos 20-50 alcanzan para arrancar) y medís el **acuerdo juez↔humano** con una métrica concreta, no "a ojo": el **% de acuerdo** si la rúbrica es categórica, o **Cohen's kappa** / **correlación de Spearman** si es ordinal (1-5). Un juez con kappa bajo no es usable aunque su rúbrica se vea impecable. Sin esa calibración, estás midiendo con una regla que no verificaste.

**Ejercicios 4**
4.1 ¿Qué dos cosas hacen confiable a un LLM-as-judge? Explicá cada una.
4.2 Nombrá dos sesgos de un juez LLM y una mitigación de cada uno.
4.3 ¿Cómo sabés que tu juez puntúa bien? ¿Por qué eso importa?

---

## Módulo 5 — Evaluar por capas: componente vs end-to-end

**Teoría.** Un sistema de IA real —un RAG, un agente— tiene **muchas piezas**, y "el sistema responde mal" no te dice cuál falló. La disciplina clave (que ya asomó en el módulo 11 de RAG): **evaluá cada capa por separado, además del sistema entero.**

En un **RAG**, las dos capas:

- **Retrieval** (métricas de código): recall@k, precision@k. ¿Se recuperaron los chunks correctos? Si esto mide mal, el problema está antes del LLM —chunking, embeddings, retrieval— y tocar el prompt no sirve.
- **Generación** (LLM-as-judge): faithfulness, relevancia. Aunque el retrieval traiga lo correcto, ¿el modelo respondió bien?

En un **agente** (módulo de Agentes), las capas son distintas y más complejas:

- **Resultado final** (end-to-end): ¿el agente cumplió el objetivo? A veces verificable por código (¿el test pasa?, ¿el archivo se creó?), a veces por juez.
- **Trayectoria** (el camino): ¿usó las herramientas correctas, en orden razonable, sin pasos inútiles ni loops? Un agente puede llegar al resultado correcto por un camino carísimo, o fallar a mitad. Evaluar la trayectoria —no solo el resultado— es propio de los agentes.
- **Uso de tools**: ¿llamó las tools con argumentos válidos? ¿manejó los errores?

La razón de fondo: **evaluar por capas localiza la falla.** Si solo medís end-to-end, sabés que algo anda mal pero no qué. Con métricas por capa, sabés exactamente dónde atacar —y evitás el clásico de "mejorar el prompt" cuando el problema era el chunking, o de tocar el retrieval cuando el modelo alucinaba con buen contexto—. Es el mismo principio de un buen test suite (módulo de Testing): tests unitarios que aíslan el componente + tests de integración que cubren el flujo. Acá: evals de componente + evals end-to-end.

**Ejercicios 5**
5.1 ¿Cuáles son las dos capas de un RAG que evaluás por separado y con qué familia de métrica cada una?
5.2 En un agente, ¿qué es evaluar la "trayectoria" y por qué no alcanza con evaluar solo el resultado final?
5.3 ¿Por qué evaluar por capas en vez de solo end-to-end? Conectá con el módulo de Testing.

---

## Módulo 6 — Eval-driven development: el TDD de la IA

**Teoría.** El cambio de mentalidad más grande del módulo: **las evals no son algo que hacés al final, son cómo iterás.** En desarrollo tradicional de IA, la trampa es prompt-tunear a ojo —cambiar el prompt, probar dos casos a mano, "parece mejor", deployar—. **Eval-driven development** le da la disciplina del TDD a esto: el ciclo **rojo → verde → refactor** del módulo de Testing, aplicado a IA.

El ciclo:

1. **Definí el éxito primero** (rojo): armás el golden set y las métricas *antes* de optimizar. Decidís qué es "bueno" en números, no en sensaciones.
2. **Iterá contra las evals** (verde): cambiás el prompt / modelo / chunking / herramientas y corrés las evals. La métrica te dice si mejoraste, empeoraste, o arreglaste un caso rompiendo otros.
3. **Refactorizá con red de seguridad**: como tenés evals, podés cambiar el sistema (modelo más barato, prompt más corto) y confirmar que no degradaste la calidad —exactamente la confianza que dan los tests para refactorizar código—.

Por qué esto importa tanto en IA:

- **Las mejoras en IA no son monótonas.** Arreglar un caso suele romper otro (cambiás el prompt para que deje de alucinar en X y empieza a ser demasiado cauto en Y). Sin evals, no lo ves; con evals sobre un conjunto, lo ves al instante en la métrica agregada.
- **Te saca del "parece mejor".** Dos personas miran la misma salida y discrepan. Un número sobre 50 casos no discrepa. Es la diferencia entre opinar y medir.
- **Permite comparar opciones objetivamente.** ¿Opus o Sonnet para esta tarea? ¿Este prompt o aquel? Corrés la eval con cada uno y comparás scores, costo y latencia. El model tiering del módulo 9 de LLMs ("el modelo más chico que cumple tu calidad") **se decide con evals**: bajás de modelo y verificás que la métrica aguante.

La frase mental: **escribí la eval antes de optimizar el prompt, como escribís el test antes del código.** Eso convierte "tunear prompts" de un arte impreciso en ingeniería medible.

**Ejercicios 6**
6.1 ¿Qué es eval-driven development y con qué ciclo del módulo de Testing se corresponde?
6.2 ¿Por qué "las mejoras en IA no son monótonas" hace indispensables las evals al iterar?
6.3 ¿Cómo se decide con evals la elección de modelo (model tiering del módulo de LLMs)?

---

## Módulo 7 — Offline vs online: evals en CI y en producción

**Teoría.** Hay **dos momentos** para evaluar, y un sistema serio usa los dos.

**Evals offline** (antes de deployar): corrés tu golden set contra el sistema en desarrollo. Es lo de los módulos anteriores. Acá encaja la pieza que conecta con el módulo de Testing: **las evals corren en CI.** Igual que el pipeline corre tus tests en cada PR, corre tus evals —al menos las de código (recall@k, validación de schema, latencia, costo), que son baratas y deterministas—. Si una métrica clave cae por debajo de un umbral, el pipeline **bloquea el merge**. Eso convierte "no rompas la calidad de la IA" en una **regresión detectable**, igual que un test rojo. Las evals de LLM-as-judge, más caras, podés correrlas con menos frecuencia (nightly, o solo en cambios de prompt) o sobre un subconjunto.

**Evals online** (en producción, sobre tráfico real): el golden set, por bueno que sea, no cubre todo lo que los usuarios reales mandan. Las evals online monitorean el sistema **en vivo**:

- **Muestreo de tráfico real**: tomás una fracción de las interacciones reales y las puntuás (con jueces LLM o señales de código) para medir calidad en producción, no solo en tu dataset.
- **Detección de drift**: la calidad puede degradarse con el tiempo —cambia el tipo de preguntas, cambia la data, el proveedor actualiza el modelo—. El monitoreo online lo detecta antes de que sea un incidente.
- **Señales de usuario**: thumbs up/down, tasa de reformulación de preguntas, abandono. Son evals implícitas que el usuario te regala gratis.

La distinción: **offline mide antes de soltar (¿es seguro deployar?); online mide después (¿sigue funcionando con tráfico real?).** Conecta directo con la observabilidad de producción del archivo senior y del track de DevOps: no alcanza con que pase en tu máquina, tenés que medir en producción. Y conecta con el módulo siguiente: las evals online son la base del monitoreo y los guardarraíles.

**Ejercicios 7**
7.1 ¿Qué diferencia hay entre evals offline y online? ¿Qué responde cada una?
7.2 ¿Cómo encajan las evals en CI y por qué eso convierte la calidad de IA en una "regresión detectable"? ¿Qué tipo de evals correrías en cada commit vs. con menos frecuencia?
7.3 Nombrá dos cosas que las evals online detectan que el golden set offline no puede.

---

## Módulo 8 — Observabilidad y tracing

**Teoría.** No podés evaluar —ni debuggear— lo que no podés ver. Un sistema de IA, sobre todo un agente, es una **caja negra de muchos pasos**: una pregunta dispara un retrieval, varias llamadas al LLM, varias tool calls, decisiones intermedias. Cuando algo sale mal, "la respuesta fue mala" es inútil sin saber qué pasó adentro. La **observabilidad** es lo que te da esa visibilidad.

La unidad es el **trace** (traza): el registro completo de todo lo que pasó al procesar una request, descompuesto en **spans** (cada paso: cada llamada al LLM con su prompt/respuesta/tokens/latencia, cada retrieval con sus chunks, cada tool call con sus argumentos y resultado). Es el mismo concepto de tracing distribuido que se usa para microservicios (el del track de observabilidad), aplicado a un pipeline de IA. De hecho, el estándar **OpenTelemetry** —el de tracing general— tiene convenciones semánticas para IA generativa (todavía **experimentales y sujetas a cambio**, así que verificá la versión al implementar), y herramientas como **Langfuse** (open source) están construidas sobre esa idea: capturan cada llamada, su costo, su latencia y sus inputs/outputs, y te dejan inspeccionar traces individuales y agregar métricas.

Para qué sirve concretamente:

- **Debuggear un fallo**: el usuario reporta una respuesta mala → abrís el trace → ves que el retrieval trajo el chunk equivocado (no era el prompt). Sin el trace, adivinás.
- **Medir costo y latencia reales** por paso: qué llamada quema más tokens, qué tool es el cuello de botella. Es el "instrumentá `usage` desde el día uno" del módulo de costo, hecho sistema.
- **Alimentar las evals online**: los traces de producción *son* el tráfico real que muestreás y puntuás (módulo 7), y los fallos que encontrás en los traces se vuelven casos nuevos del golden set (módulo 2). El círculo se cierra: observás → encontrás fallos → los agregás al dataset → mejorás → medís.

El principio (del archivo senior): **observabilidad no es un lujo, es parte del sistema.** Un sistema de IA sin tracing es indebuggeable —tenés un no determinista del que no sabés qué hizo—. Lo instrumentás desde el inicio, no cuando ya tenés un incidente en producción y no podés reconstruir qué pensó el modelo.

**Ejercicios 8**
8.1 ¿Qué es un trace y un span en un sistema de IA, y con qué concepto del track de observabilidad/microservicios conecta?
8.2 Da un caso concreto donde el tracing te ahorra adivinar cuál capa falló.
8.3 ¿Cómo se conectan los traces de producción con las evals online y con el golden set? (el círculo)

---

## Módulo 9 — Guardrails: la última línea en producción

**Teoría.** Las evals (offline y online) miden calidad, pero hay una clase de fallos que no podés permitir que lleguen al usuario, ni siquiera una vez: que el sistema filtre datos de otro tenant, devuelva contenido tóxico, exponga un secreto, o ejecute una acción peligrosa por prompt injection. Los **guardrails (guardarraíles)** son los chequeos **en tiempo real**, en el camino del request, que validan input y output **antes** de que la salida llegue al usuario o de que una acción se ejecute.

La diferencia con las evals: una eval mide calidad **en agregado** (¿el sistema es bueno en promedio?); un guardarraíl es una **decisión por request** (¿esta salida específica es segura de devolver, sí o no?). Las evals son medición; los guardrails son control.

Tipos típicos:

- **De input**: detectar y bloquear prompt injection, contenido prohibido, o requests fuera de alcance antes de procesarlos.
- **De output**: validar que la salida no contenga PII, secretos, ni afirmaciones sin grounding (en RAG: ¿la respuesta cita fuentes reales del contexto, o inventó?). Acá rinde el structured output (validar la forma) más una validación de dominio del contenido (módulo 10 de LLMs).
- **De acción** (en agentes): el **human-in-the-loop** del módulo de Agentes —antes de una acción irreversible (borrar, cobrar, enviar), un gate que la confirma o un check automático que la valida—.

Cómo se implementan: a veces con código (regex/validación para PII, schema para forma), a veces con un **LLM clasificador** rápido y barato (Haiku) que juzga "¿esta salida es segura?" en línea —un primo en tiempo real del LLM-as-judge, optimizado para latencia, no para profundidad—. Ojo: ese guardarraíl-LLM agrega una llamada extra en el **camino crítico de cada request** (latencia + costo), así que se reserva para los fallos catastróficos, se corre en paralelo cuando se puede, y se prefiere código siempre que el chequeo lo permita (la regla de oro del módulo 3).

El principio integrador, el de siempre: **defensa en profundidad y mínimo privilegio.** Las evals te dicen que el sistema es bueno en general; los guardrails atajan el caso individual catastrófico que el promedio esconde. Ninguna capa sola alcanza: golden set + evals en CI + monitoreo online + guardrails en el request, apilados. Para un AI engineer, saber que **medir (evals) y controlar (guardrails) son cosas distintas y complementarias** es criterio senior.

**Ejercicios 9**
9.1 ¿Cuál es la diferencia entre una eval y un guardarraíl? (pista: agregado vs. por request)
9.2 Nombrá un guardarraíl de input, uno de output y uno de acción.
9.3 ¿Por qué los guardrails no reemplazan a las evals ni al revés? Conectá con defensa en profundidad.

---

## Módulo 10 — El criterio: qué medir, cuánto y cuándo

**Teoría.** El módulo de criterio que cierra el track entero. Tenés todo el arsenal —golden set, métricas de código, LLM-as-judge, evals por capa, eval-driven, offline/online, tracing, guardrails—. El error ahora es el opuesto al de no medir: **sobre-evaluar**, gastar más en infraestructura de evals que lo que vale el sistema, o medir métricas vanidosas que no mueven ninguna decisión.

El criterio, escalado a lo que necesita el sistema:

- **Un prototipo**: unos pocos casos a mano y métricas de código (¿el JSON valida?, ¿la latencia es tolerable?). No montes LLM-as-judge ni un pipeline de evals para un experimento. Empezá simple, igual que con todo el track.
- **Un sistema en producción**: golden set representativo, evals de código en CI (bloquean regresiones), LLM-as-judge para la calidad subjetiva, monitoreo online y tracing. Acá la inversión se justifica porque los errores tienen costo real.
- **Un sistema crítico** (salud, legal, finanzas): todo lo anterior más guardrails estrictos, calibración humana del juez, y muestreo online agresivo. El costo del error manda.

La regla que ordena: **medí lo que mueve una decisión.** Una métrica que mirás pero que no cambia nada que hagas es ruido (una "vanity metric"). Las que sí mueven decisiones: recall@k (¿mejoro el retrieval?), faithfulness (¿confío en deployar?), costo/latencia (¿bajo de modelo?), tasa de fallo en producción (¿hay un incidente?). Si una métrica no responde una pregunta que te lleva a actuar, no la midas.

Y el criterio que atravesó **todo el track de IA**, una última vez, ahora completo: la mejor solución es la más simple que resuelve el problema real. En LLMs era "¿necesito un modelo o un `if`?"; en RAG, "¿busco en muchos docs o trabajo con pocos?"; en Agentes, "¿el modelo decide el camino o lo escribo yo?"; en Evals, **"¿qué necesito medir de verdad para tomar las decisiones de este sistema?"**. Saber responder las cuatro —y construir solo la complejidad que el problema justifica— es lo que demuestra criterio senior de AI engineer. Con evals cerrás el ciclo: ya no solo construís sistemas de IA, sabés **probar que funcionan**.

**Ejercicios 10**
10.1 ¿Cuál es el error opuesto a no medir, y cómo escalás la inversión en evals según el sistema?
10.2 ¿Qué es una "vanity metric" y cuál es la regla para decidir si una métrica vale la pena medir?
10.3 Repasá la pregunta-criterio de cada uno de los cuatro módulos del track de IA (LLMs, RAG, Agentes, Evals).

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Porque un LLM es no determinista (módulo 1 de LLMs): la misma entrada puede dar salidas
    distintas y "correcto" es un rango de respuestas, no una sola string. Un assert exacto
    "falla" ante una redacción distinta igual de válida.
1.2 (Dos de:) el test da pasa/falla, la eval da un score; el test compara contra un valor
    exacto, la eval contra criterios/referencia/juez; el test es determinista, la eval se
    corre varias veces sobre muchos casos.
1.3 Porque una sola corrida no dice nada: por azar pudo salir bien o mal. Lo no determinista
    se mide en agregado —tasa de acierto sobre muchos casos—, no en un caso suelto.
```

### Módulo 2
```
2.1 El golden set es el conjunto de casos de prueba contra los que medís. Cada caso tiene una
    entrada y una salida esperada o criterio de éxito (a veces una rúbrica); en RAG, además,
    qué chunk(s) contienen la respuesta.
2.2 Porque un dataset no representativo o sin casos borde da métricas optimistas y falsas:
    optimizás para la métrica equivocada y el sistema "mejora" en la eval mientras empeora para
    usuarios reales. Caso borde: una pregunta sin respuesta en la data (¿dice "no sé" o
    alucina?).
2.3 Crece con los fallos reales de producción: cada bug se vuelve un caso nuevo del golden set.
    Se parece a escribir, en testing, un test que reproduce el bug antes de arreglarlo.
```

### Módulo 3
```
3.1 Código: exact match/regex (clasificación), validación de schema, recall@k, latencia, costo
    (dos cualesquiera). Solo LLM-as-judge: fidelidad al contexto, calidad/coherencia de un
    resumen, tono apropiado, relevancia (dos cualesquiera sobre texto libre/subjetivo).
3.2 Usá código siempre que puedas; LLM-as-judge solo cuando no se pueda con código. Conecta con
    el criterio del módulo 11 de LLMs: no metas un LLM donde una regex/cálculo alcanza.
3.3 Usar un LLM caro para juzgar si una clasificación dio "factura": eso es exact match (===),
    no necesita juez —agrega costo, latencia y no determinismo sin beneficio.
```

### Módulo 4
```
4.1 (1) Una rúbrica clara y específica (criterios explícitos por nivel) en vez de "¿es buena?":
    la hace reproducible. (2) Structured output para el score ({score, razon}): garantiza un
    número parseable más la justificación, en vez de texto libre.
4.2 (Dos de:) sesgo de posición (prefiere la primera/última en A-vs-B) → aleatorizar orden o
    promediar; sesgo de verbosidad (puntúa mejor lo largo) → rúbrica que penalice el relleno;
    self-preference (prefiere su propia familia de modelo) → usar un juez de modelo distinto.
4.3 Calibrándolo contra casos puntuados por humanos (20-50 alcanzan): medís el acuerdo
    juez↔humano con una métrica concreta —% de acuerdo si es categórica, Cohen's kappa o
    correlación de Spearman si es ordinal (1-5)— y solo confiás en él para escalar si el
    acuerdo es alto. Importa porque sin calibrar estás midiendo con una regla que no
    verificaste; un juez con kappa bajo no sirve aunque su rúbrica se vea impecable.
```

### Módulo 5
```
5.1 Retrieval (métricas de código: recall@k, precision@k) y generación (LLM-as-judge:
    faithfulness, relevancia).
5.2 Evaluar la trayectoria es chequear el camino del agente: qué tools usó, en qué orden, sin
    pasos inútiles ni loops. No alcanza el resultado final porque un agente puede llegar a la
    respuesta correcta por un camino carísimo o equivocado, o fallar a mitad —y eso solo se ve
    mirando la trayectoria.
5.3 Porque evaluar por capas localiza la falla: solo end-to-end te dice que algo anda mal pero
    no qué. Es el mismo principio del módulo de Testing: tests unitarios que aíslan el
    componente + tests de integración que cubren el flujo.
```

### Módulo 6
```
6.1 Es usar las evals como la forma de iterar (definir el éxito en números primero, iterar
    contra las métricas, refactorizar con red de seguridad), no algo del final. Corresponde al
    ciclo rojo → verde → refactor del TDD.
6.2 Porque arreglar un caso suele romper otro (dejás de alucinar en X y te volvés demasiado
    cauto en Y). Sin evals no lo ves; con una métrica agregada sobre muchos casos, lo ves al
    instante.
6.3 Corrés la misma eval con cada modelo y comparás score, costo y latencia. Bajás de modelo
    (ej. Opus → Sonnet) y verificás que la métrica de calidad aguante: así "el modelo más chico
    que cumple tu calidad" se decide con datos, no a ojo.
```

### Módulo 7
```
7.1 Offline: corrés el golden set antes de deployar (¿es seguro soltar esto?). Online: medís
    sobre tráfico real en producción (¿sigue funcionando con usuarios reales?).
7.2 Las evals corren en el pipeline como los tests: si una métrica clave cae bajo un umbral,
    bloquea el merge —convierte "no degrades la calidad" en una regresión detectable. En cada
    commit corrés las de código (baratas, deterministas: recall@k, schema, latencia, costo); las
    de LLM-as-judge (caras) con menos frecuencia o sobre un subconjunto.
7.3 (Dos de:) preguntas reales que el golden set no cubre; drift de calidad en el tiempo
    (cambia el tráfico, la data, o el modelo del proveedor); señales implícitas de usuario
    (thumbs down, reformulaciones, abandono).
```

### Módulo 8
```
8.1 Un trace es el registro completo de todo lo que pasó al procesar una request; los spans son
    cada paso (cada llamada al LLM, retrieval, tool call) con su prompt, resultado, tokens y
    latencia. Conecta con el tracing distribuido de microservicios (OpenTelemetry), aplicado a
    un pipeline de IA.
8.2 El usuario reporta una respuesta mala; abrís el trace y ves que el retrieval trajo el chunk
    equivocado, no que el prompt estaba mal —así sabés que el problema es el retrieval y no
    perdés tiempo tuneando el prompt.
8.3 Los traces de producción son el tráfico real que muestreás y puntuás (evals online), y los
    fallos que encontrás en ellos se vuelven casos nuevos del golden set. El círculo: observás
    → encontrás fallos → los agregás al dataset → mejorás → medís.
```

### Módulo 9
```
9.1 Una eval mide calidad en agregado (¿el sistema es bueno en promedio?); un guardarraíl es
    una decisión por request (¿esta salida específica es segura de devolver?). Evals = medición;
    guardrails = control.
9.2 Input: detectar/bloquear prompt injection o contenido prohibido antes de procesar. Output:
    validar que no haya PII/secretos ni afirmaciones sin grounding. Acción (agentes):
    human-in-the-loop antes de una acción irreversible (borrar, cobrar, enviar).
9.3 Porque miden cosas distintas: las evals aseguran que el sistema es bueno en general; los
    guardrails atajan el caso individual catastrófico que el promedio esconde. Es defensa en
    profundidad: golden set + evals en CI + monitoreo online + guardrails, apilados; ninguna
    capa sola alcanza.
```

### Módulo 10
```
10.1 El error opuesto es sobre-evaluar: gastar más en infraestructura de evals que lo que vale
     el sistema, o medir métricas que no mueven decisiones. Escalás: prototipo = pocos casos a
     mano + métricas de código; producción = golden set + evals de código en CI + LLM-as-judge
     + online + tracing; crítico = todo eso + guardrails estrictos + calibración humana +
     muestreo agresivo.
10.2 Una vanity metric es una que mirás pero que no cambia ninguna decisión que tomás (ruido).
     La regla: medí lo que mueve una decisión; si una métrica no responde una pregunta que te
     lleva a actuar, no la midas.
10.3 LLMs: ¿necesito un modelo o un if/regex? RAG: ¿busco en muchos docs o trabajo con pocos?
     Agentes: ¿el modelo decide el camino o lo escribo yo? Evals: ¿qué necesito medir de verdad
     para tomar las decisiones de este sistema? Las cuatro: la solución más simple que resuelve
     el problema real.
```

---

## Siguientes pasos

Con este módulo **cerrás el track de IA completo**. Tenés el ciclo entero de un AI engineer: consumir un LLM con criterio (LLMs), darle tu propia data (RAG), hacerlo decidir y actuar (Agentes), y —ahora— **probar que todo eso funciona** (Evals): golden set, métricas de código vs LLM-as-judge, evaluación por capas, eval-driven development, evals offline en CI y online en producción, observabilidad con tracing, y guardrails como última línea. El hilo conductor del track queda completo: el mismo sistema que arrancó como una **llamada simple** creció a **extractor estructurado**, a **RAG sobre tus apuntes**, a un **agente que decide su camino**, y finalmente queda **medido y monitoreado** —un proyecto de portfolio coherente, de punta a punta, para el rol de Software Engineer AI—. Desde acá, los próximos pasos del roadmap general apuntan al track de Tech Lead (NoSQL hands-on, Kubernetes, observabilidad práctica, TDD como disciplina, liderazgo técnico), que profundizan la otra mitad del perfil senior contratable.
