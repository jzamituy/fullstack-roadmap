# Agentes: un LLM en un loop que decide y actúa

**Agentes con Claude / Anthropic, MCP y el Claude Agent SDK · criterio de producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **tercer módulo del track de IA**: asume que ya hiciste [IA Generativa y LLMs](ia-llms.md) (sobre todo el **tool use**, módulo 5) y [RAG](rag.md). Un **agente** es la pieza que ata todo lo anterior: un LLM puesto en un **loop**, con **herramientas**, que **decide solo qué hacer** para cumplir un objetivo —llamar una tool, leer el resultado, decidir el próximo paso— hasta terminar. El tool use que viste era el ladrillo; el agente es el edificio. El stack es **Claude / Anthropic**, con el **Claude Agent SDK** y **MCP** (Model Context Protocol) como estándar de herramientas.

**Lo que asumimos.** TS, NestJS (DI, providers), el tool use del módulo de LLMs, el RAG del módulo anterior, async/await, manejo de errores, colas y resiliencia. El código usa `@anthropic-ai/sdk`.

**Para practicar.** El SDK y tu API key:

```bash
npm i @anthropic-ai/sdk
export ANTHROPIC_API_KEY="sk-ant-..."
```

> Nota sobre datos volátiles: IDs/precios de modelos, betas (`mcp-client-2025-11-20`) y la versión del Agent SDK cambian con cada release; verificá contra `docs.claude.com` al implementar.

**Índice de módulos**
1. Qué es un agente (y qué no): el loop
2. El criterio primero: llamada simple → workflow → agente
3. El agentic loop por dentro
4. Diseño de herramientas (tool design)
5. Patrones de workflow (los ladrillos antes del agente)
6. MCP: el estándar de herramientas
7. Gestión de contexto en agentes largos
8. Confiabilidad: errores, guardarraíles y human-in-the-loop
9. RAG como herramienta del agente
10. El Claude Agent SDK y dónde corre el loop
11. El criterio integrador: ¿vale la pena un agente?

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es un agente (y qué no): el loop

**Teoría.** En el módulo de LLMs viste el **tool use**: le declarás herramientas al modelo, él pide ejecutar una, vos la ejecutás y le devolvés el resultado. Un **agente** es eso puesto en un **loop autónomo**: el modelo decide qué tool usar, observa el resultado, **decide el próximo paso por sí mismo**, y repite hasta cumplir el objetivo. La definición de Anthropic: un agente es un **LLM que dirige dinámicamente su propio proceso y el uso de herramientas, manteniendo el control sobre cómo cumple la tarea.**

La diferencia que tenés que internalizar —y que define todo el módulo— es **quién decide el camino**:

- En un **workflow**, *vos* escribís el flujo: paso 1, después paso 2, después un `if`, después paso 3. El LLM ejecuta cada paso, pero **el camino lo decidís vos en código**. Es predecible y determinista.
- En un **agente**, *el modelo* decide el camino. Vos le das un objetivo y herramientas; él elige qué tools usar, en qué orden, cuántas veces, y cuándo terminó. Es flexible pero **no determinista**.

El loop conceptual de un agente:

```
objetivo → [ LLM decide: ¿qué hago? ] → ejecuta tool → observa resultado
                  ↑                                            │
                  └──────────── repite hasta terminar ─────────┘
```

Qué **no** es un agente, para sacarte la confusión:

- **No** es un LLM con muchas herramientas y nada más: sin el loop autónomo (sin que decida y vuelva a decidir), es solo tool use de un paso.
- **No** es magia: es la misma Messages API que ya conocés, llamada en un `while`, con la salida de cada tool reinyectada como entrada de la próxima vuelta.
- **No** es siempre la respuesta correcta: es **caro, lento y no determinista**, y muchísimos problemas se resuelven mejor con un workflow o una sola llamada (módulos 2 y 11).

La frase mental: **un agente cambia control por autonomía.** Le cedés al modelo las decisiones del camino a cambio de que resuelva tareas que no podrías programar paso a paso de antemano. Esa cesión es potente cuando la tarea es genuinamente abierta, y un desperdicio (o un riesgo) cuando no lo es.

**Ejercicios 1**
1.1 En una frase, ¿qué es un agente y de qué pieza del módulo de LLMs está hecho?
1.2 ¿Cuál es la diferencia central entre un workflow y un agente? (pista: quién decide algo)
1.3 ¿Por qué "un LLM con muchas herramientas" no es todavía un agente?

---

## Módulo 2 — El criterio primero: llamada simple → workflow → agente

**Teoría.** Pongo el criterio al principio porque es el error #1 en IA aplicada: **armar un agente cuando no hacía falta.** El consejo central de Anthropic es **empezá con lo más simple que resuelva el problema**, y subir complejidad solo cuando el problema lo exige de verdad. Hay una escalera:

1. **Una sola llamada al LLM** (con prompting, structured output, retrieval): resuelve muchísimos casos. Clasificar, resumir, extraer, responder con RAG. Si te alcanza, **terminá acá**.
2. **Un workflow**: varios pasos orquestados *por tu código* (módulo 5). Predecible, debuggeable, barato. La mayoría de los sistemas "con IA" de producción son workflows, no agentes.
3. **Un agente**: el modelo decide el camino. Solo cuando la tarea es abierta y no podés especificar los pasos de antemano.

Cada escalón que subís agrega **costo, latencia, no determinismo y superficie de error**. La regla: subí un escalón solo cuando el de abajo te queda corto, no por defecto ni "porque suena más avanzado".

El trade-off concreto entre workflow y agente:

| | Workflow | Agente |
|---|---|---|
| Decide el camino | Tu código | El modelo |
| Predecibilidad | Alta (determinista) | Baja (no determinista) |
| Costo/latencia | Acotados | Variables, potencialmente altos |
| Cuándo | Pasos conocibles de antemano | Tarea abierta, pasos no especificables |

La pregunta-criterio para elegir entre workflow y agente: **¿puedo escribir los pasos de antemano?** Si sí —"buscá, después resumí, después clasificá"— es un workflow. Si no —"resolvé este bug" o "investigá este tema", donde los pasos dependen de lo que vayas encontrando— recién ahí un agente gana lo suficiente para justificar su costo.

Esto es el mismo criterio del módulo 11 de LLMs ("¿de verdad necesito un LLM acá?") subido un nivel: **¿de verdad necesito un agente, o un workflow o una llamada me alcanzan?** Saber responder eso es lo que demuestra criterio senior.

**Ejercicios 2**
2.1 Nombrá los tres escalones de complejidad en orden y qué agrega cada salto.
2.2 ¿Cuál es la pregunta-criterio para elegir entre un workflow y un agente? Dá un ejemplo de cada lado.
2.3 ¿Por qué "empezá simple" es el consejo central, y qué error evita?

---

## Módulo 3 — El agentic loop por dentro

**Teoría.** Desmitifiquemos el agente escribiéndolo. No hay framework mágico: es el **loop de tool use** del módulo de LLMs, repetido. El ciclo:

1. Llamás a la Messages API con el objetivo y las tools.
2. Si `stop_reason === "end_turn"`, el modelo terminó → salís.
3. Si pidió tools (`stop_reason === "tool_use"`), ejecutás cada una, agregás los resultados al historial, y **volvés al paso 1**.

```ts
async function correrAgente(objetivo: string, tools: Anthropic.Tool[]): Promise<string> {
  const messages: Anthropic.MessageParam[] = [{ role: "user", content: objetivo }];

  while (true) {
    const res = await client.messages.create({
      model: "claude-opus-4-8",
      max_tokens: 4096,
      tools,
      messages,
    });

    if (res.stop_reason === "end_turn") {
      const texto = res.content.find((b) => b.type === "text");
      return texto?.type === "text" ? texto.text : "";
    }

    // El modelo pidió una o más tools: ejecutarlas y devolver resultados
    messages.push({ role: "assistant", content: res.content }); // guardar lo que pidió
    const resultados: Anthropic.ToolResultBlockParam[] = [];
    for (const bloque of res.content) {
      if (bloque.type === "tool_use") {
        const salida = await ejecutarTool(bloque.name, bloque.input); // TU código
        resultados.push({ type: "tool_result", tool_use_id: bloque.id, content: salida });
      }
    }
    messages.push({ role: "user", content: resultados }); // los resultados como "user"
  }
}
```

Lo que hay que ver en este código:

- **El estado vive en `messages`**: la API es *stateless* (módulo 2 de LLMs), así que vos acumulás todo el historial —lo que el modelo pidió y lo que las tools devolvieron— y lo reenviás completo cada vuelta. El "razonamiento" del agente es ese historial creciendo.
- **El modelo nunca ejecuta nada**: pide; *tu* código ejecuta (`ejecutarTool`). Acá ponés validación, permisos, logging, límites. Es tu punto de control sobre un proceso autónomo.
- **Parálisis y costo**: cada vuelta es una llamada (input + output) sobre un historial cada vez más largo. Sin límites, un agente puede quedar en loop o quemar tokens. Por eso ponés un **tope de iteraciones** (un `for` con máximo, no un `while` infinito en producción) — el guardarraíl del módulo 8.

Dos detalles de la API real: para **tools del lado del servidor** (web search, code execution) podés recibir `stop_reason === "pause_turn"` —el modelo pausó y querés reenviar para que siga, sin agregar un mensaje "continuá"—. Y el SDK trae un **tool runner** (`client.beta.messages.toolRunner`) que maneja este loop por vos; entenderlo a mano —como acá— es clave para saber qué hace por debajo y cuándo necesitás el control manual (gating, human-in-the-loop).

**Ejercicios 3**
3.1 Describí las tres partes del agentic loop. ¿Cuándo termina?
3.2 La API es stateless. ¿Dónde vive entonces el "estado" del agente y qué implica para cada llamada?
3.3 ¿Por qué en producción usás un tope de iteraciones y no un `while (true)`? ¿Qué dos riesgos evita?

---

## Módulo 4 — Diseño de herramientas (tool design)

**Teoría.** La calidad de un agente depende más de **sus herramientas** que del prompt. El modelo decide qué tool usar leyendo su nombre, su descripción y su schema; herramientas mal diseñadas = decisiones malas. La "ingeniería de tools" es tan importante como la de prompts.

Ya viste lo básico en el módulo 5 de LLMs (nombre, descripción, `input_schema`, descripción prescriptiva del *cuándo* usarla). Acá van las decisiones de diseño propias de un agente:

- **Una herramienta de propósito amplio (bash) vs. herramientas dedicadas.** Una tool `bash` le da al modelo enorme poder (puede hacer casi cualquier cosa), pero a *tu harness* le da solo un string opaco: no podés distinguir un `grep` inocuo de un `rm -rf`. Una **tool dedicada** (`buscar_archivo`, `borrar_tarea`) te da un hook con argumentos tipados que podés **interceptar, validar, gatear, renderizar o auditar**. La regla: empezá con tools amplias por cobertura; **promové a tool dedicada cuando necesités controlar esa acción** (seguridad, confirmación, UI, paralelización).
- **Reversibilidad como criterio de gating.** Las acciones difíciles de revertir (mandar un mail, borrar datos, cobrar) son candidatas naturales a herramientas dedicadas que podés gatear con confirmación. `enviar_email` es fácil de gatear; `bash -c "curl -X POST..."` no.
- **Pocas y bien elegidas.** Demasiadas herramientas confunden al modelo (elige la equivocada, o se marea). Un set chico y enfocado decide mejor. Si tenés cientos de tools, está el **tool search** (el modelo descubre las relevantes en vez de cargarlas todas), pero la primera respuesta suele ser: recortá el set.
- **Descripciones prescriptivas del cuándo.** En modelos recientes (Opus 4.x) que son más conservadores para llamar tools, decir en la descripción *cuándo* usarla ("Usala cuando el usuario pida el estado actual de una tarea") sube de forma medible la tasa de uso correcto, más que decir solo qué hace.

El principio integrador (del archivo senior y del módulo de seguridad de LLMs): **mínimo privilegio.** Cada tool del agente tiene solo el poder justo. Un agente con una tool `ejecutar_sql_arbitrario` es una inyección SQL esperando a pasar; uno con `obtener_tarea(id)` y `cerrar_tarea(id)` tiene un blast radius acotado. Diseñás las tools como diseñás una API interna: superficie mínima, contratos claros, validación en el borde.

**Ejercicios 4**
4.1 ¿Por qué una tool `bash` le da poder al modelo pero poco control a tu harness, y cuándo promovés una acción a tool dedicada?
4.2 ¿Qué criterio usás para decidir qué acciones gatear con confirmación? Dá un ejemplo.
4.3 Conectá con el principio de mínimo privilegio: ¿por qué `obtener_tarea(id)` es mejor diseño que `ejecutar_sql_arbitrario(query)` para un agente?

---

## Módulo 5 — Patrones de workflow (los ladrillos antes del agente)

**Teoría.** Antes de saltar al agente full, conocé los **patrones de workflow**: composiciones de llamadas LLM orquestadas *por tu código*. Resuelven la mayoría de los casos reales con más control y menos costo que un agente, y además son los **bloques** con los que después construís agentes. Los cinco patrones de Anthropic:

- **Prompt chaining (encadenamiento)**: descomponés la tarea en pasos secuenciales; la salida de una llamada es la entrada de la siguiente. Ej.: generar un esquema → expandirlo a texto → traducirlo. Útil cuando la tarea se parte limpio en subtareas fijas. Podés meter checks (`gates`) entre pasos.
- **Routing (enrutamiento)**: una primera llamada **clasifica** la entrada y la dirige a un prompt/modelo especializado. Ej.: clasificar un ticket (reclamo / consulta / factura) y mandarlo al flujo y modelo adecuados —incluso a Haiku para los simples y Opus para los difíciles—. Separa preocupaciones y permite optimizar cada rama.
- **Parallelization (paralelización)**: dos sabores. *Sectioning*: partir la tarea en subtareas independientes que corren en paralelo (más rápido). *Voting*: correr la misma tarea varias veces y combinar (más confiable, ej. varios chequeos de seguridad sobre el mismo código).
- **Orchestrator-workers**: un LLM "orquestador" descompone dinámicamente la tarea en subtareas y las delega a LLMs "trabajadores", después sintetiza. A diferencia de la paralelización fija, **las subtareas no se conocen de antemano** —las decide el orquestador—. Acá ya empezás a rozar lo "agéntico".
- **Evaluator-optimizer**: un LLM genera una respuesta, otro la **evalúa y da feedback**, y se itera en loop hasta que pasa el criterio. Ej.: escribir → criticar → reescribir. Potente cuando hay un criterio de calidad claro y la primera versión rara vez es perfecta.

El insight clave: **un agente es, en buena medida, estos patrones decididos dinámicamente por el modelo en vez de cableados por vos.** Cuando podés cablear el patrón (sabés que siempre es "clasificar → enrutar → responder"), un workflow es mejor: más barato, predecible y debuggeable. Cuando *no* podés —porque el patrón depende de lo que el modelo vaya descubriendo— subís al agente. Dominar estos patrones es lo que te deja elegir el nivel correcto en vez de tirar un agente a todo.

**Ejercicios 5**
5.1 Explicá prompt chaining y routing, y dá un caso de uso de cada uno.
5.2 ¿Qué diferencia a "orchestrator-workers" de la paralelización fija? (pista: quién conoce las subtareas)
5.3 ¿Cómo se relacionan estos patrones con un agente, y por qué un workflow cableado suele ser preferible cuando podés usarlo?

---

## Módulo 6 — MCP: el estándar de herramientas

**Teoría.** Tus tools custom son funciones que vos escribís. Pero hay un universo de capacidades ya construidas —GitHub, Slack, bases de datos, Google Drive— que tu agente podría usar. El problema clásico: cada integración es un adaptador a medida (N agentes × M herramientas = un lío). **MCP (Model Context Protocol)** es el estándar abierto que resuelve esto: un protocolo común para que un LLM se conecte a **servidores de herramientas**. Pensalo como el **"USB-C de las herramientas de IA"** — un mismo enchufe para conectar capacidades, en vez de un cable distinto por cada una.

Un **servidor MCP** expone herramientas (y recursos, y prompts) siguiendo el protocolo; tu agente, como **cliente MCP**, las consume sin saber cómo están implementadas por dentro. Ventaja: hay servidores MCP ya hechos para servicios comunes, y si exponés tu propio backend como servidor MCP, cualquier agente compatible lo usa sin integración a medida.

En la práctica con Claude, el **MCP connector** de la Messages API conecta a un servidor remoto del lado del servidor —Anthropic hace la conexión—. Se declaran dos cosas, **juntas**: el servidor en `mcp_servers` y un `mcp_toolset` en `tools` que lo referencia por nombre:

```ts
const res = await client.beta.messages.create({
  model: "claude-opus-4-8",
  max_tokens: 1024,
  betas: ["mcp-client-2025-11-20"],
  mcp_servers: [{ type: "url", url: "https://mi-server/mcp", name: "mi-tools" }],
  tools: [{ type: "mcp_toolset", mcp_server_name: "mi-tools" }], // referencia por nombre
  messages: [{ role: "user", content: "Usá las herramientas disponibles para..." }],
});
```

(El `mcp_server_name` del toolset debe coincidir con el `name` del servidor; omitir el `mcp_toolset` es un error de validación.)

El encaje con lo que sabés: MCP no cambia el modelo mental del agentic loop (módulo 3) ni del diseño de tools (módulo 4) —el modelo sigue pidiendo herramientas y observando resultados—. Cambia **de dónde vienen** las herramientas: en vez de funciones que escribiste en tu repo, son capacidades servidas por un protocolo estándar. Y como toda herramienta externa con poder, aplican las mismas reglas: **mínimo privilegio** (qué servidores y qué tools le habilitás), tratar la salida como no confiable, y cuidar las credenciales (MCP tiene su propio esquema de auth, distinto de las API keys del servicio).

**Ejercicios 6**
6.1 ¿Qué problema resuelve MCP y a qué analogía física se lo compara?
6.2 ¿Qué dos cosas declarás juntas para usar el MCP connector de Claude, y qué pasa si falta una?
6.3 ¿MCP cambia el agentic loop? ¿Qué cambia exactamente, y qué reglas de seguridad siguen aplicando?

---

## Módulo 7 — Gestión de contexto en agentes largos

**Teoría.** Un agente que corre muchas vueltas tiene un problema que un workflow corto no tiene: **el contexto crece sin parar.** Cada tool result, cada paso de razonamiento, se acumula en `messages` (módulo 3). Eventualmente chocás contra la ventana de contexto (costo + el fenómeno *lost in the middle* del módulo de LLMs: el modelo rinde peor en contextos enormes). Manejar esto es lo que separa un agente de demo de uno que corre tareas largas.

Las tres herramientas, de menor a mayor alcance:

- **Context editing (edición de contexto)**: **limpiar** del historial lo que ya no sirve —tool results viejos, bloques de thinking ya consumidos—. No resume: poda. Mantiene la conversación corta sin perder su estructura. Sirve cuando los resultados viejos ya no son relevantes para los próximos pasos.
- **Compaction (compactación)**: cuando te acercás al límite, **resumir** el historial anterior en un bloque compacto y seguir desde ahí. A diferencia de la edición (que borra), esto condensa: conserva la información en forma comprimida. Es lo que te deja correr conversaciones que exceden la ventana.
- **Memory (memoria)**: persistir información **entre sesiones**, no solo dentro de una. El agente lee/escribe archivos en un directorio de memoria que sobrevive al reinicio del proceso. Es para estado que tiene que durar más que una corrida (preferencias aprendidas, notas de tareas previas).

La distinción mental: edición y compactación operan **dentro de una sesión** (podar vs. resumir); memoria es **entre sesiones**. Muchos agentes largos usan las tres a la vez.

Un detalle que conecta con el caché: cambiar el system prompt o las tools a mitad de sesión **invalida el prompt caching** (módulo 9 de LLMs: la caché es prefix-match). Por eso los agentes serios evitan editar el prefijo estable; si necesitan inyectar instrucciones a mitad de camino, lo hacen *después* del prefijo cacheado (en un mensaje, no editando el `system`). El criterio de "lo estable primero" del módulo de caché vuelve, ahora en el diseño del agente.

**Ejercicios 7**
7.1 ¿Por qué un agente largo tiene un problema de contexto que un workflow corto no tiene? Nombrá los dos costos de un contexto enorme.
7.2 ¿Cuál es la diferencia entre context editing y compaction? ¿Y qué resuelve la memoria que ninguna de las dos resuelve?
7.3 ¿Por qué cambiar el system prompt a mitad de sesión es problemático, y con qué módulo conecta?

---

## Módulo 8 — Confiabilidad: errores, guardarraíles y human-in-the-loop

**Teoría.** Un agente es un proceso **autónomo y no determinista** que ejecuta acciones reales. Eso lo hace potente y peligroso: puede equivocarse, quedar en loop, o hacer algo irreversible mal. La confiabilidad no es un extra; es lo que separa un juguete de algo que pondrías en producción.

Las defensas, todas conectadas con lo que ya sabés:

- **Tope de iteraciones y de costo.** El `for` con máximo del módulo 3. Un agente sin límite puede quedar en loop infinito o quemar tu presupuesto de tokens. Instrumentá `usage` por corrida (módulo de costo de LLMs) y cortá si se pasa.
- **Manejar errores de tools, no solo de la API.** Una tool puede fallar (timeout, 404, dato inválido). Devolvele al modelo el error como `tool_result` con `is_error: true` y un mensaje útil; el modelo suele adaptarse (reintentar distinto, pedir aclaración). No tragues el error en silencio ni dejes que tumbe el loop.
- **Validar la salida y las acciones.** Nunca confíes en lo que el modelo pide (módulo de seguridad de LLMs). Validá los argumentos de cada tool antes de ejecutarla. Si una acción es destructiva, no la ejecutes a ciegas.
- **Human-in-the-loop para lo irreversible.** Antes de una acción difícil de revertir (borrar, cobrar, mandar, deployar), pausá y pedí confirmación humana. Acá rinde el diseño de tools del módulo 4: una tool dedicada (`borrar_proyecto`) podés gatearla; un `bash` opaco no. Es el "gate antes de acciones irreversibles" hecho parte del flujo.
- **Prompt injection es peor en un agente.** En RAG, una inyección podía sesgar una respuesta; en un agente con tools, una inyección en un documento o en la salida de una tool puede hacer que el agente **ejecute acciones**. El input externo es hostil: separalo de las instrucciones, y combiná con mínimo privilegio (módulo 4) para que aunque sea secuestrado, no pueda hacer daño grave.

El principio integrador (el del archivo senior): **defensa en profundidad.** Ninguna capa sola alcanza —topes, validación, gating, mínimo privilegio—; las apilás. Y observabilidad: logueá cada decisión y cada tool call del agente, porque cuando algo sale mal (y va a salir), necesitás reconstruir qué pensó y qué hizo. Un agente sin trazas es indebuggeable.

**Ejercicios 8**
8.1 Nombrá tres defensas de confiabilidad de un agente y qué riesgo mitiga cada una.
8.2 ¿Cómo manejás el error de una tool dentro del loop, y por qué no conviene tragarlo o dejar que tumbe el agente?
8.3 ¿Por qué el prompt injection es más peligroso en un agente que en un RAG, y qué dos defensas combinás contra eso?

---

## Módulo 9 — RAG como herramienta del agente

**Teoría.** Acá se cierra el track: el RAG del módulo anterior **se vuelve una herramienta del agente.** En vez de un pipeline fijo "siempre recuperá y después respondé" (un workflow), le das al agente una tool `buscar_en_base_de_conocimiento(consulta)` y **él decide cuándo y qué buscar** —y cuántas veces—.

La diferencia es real y vale entenderla:

- **RAG como workflow** (lo del módulo anterior): toda pregunta dispara una recuperación, siempre, antes de responder. Predecible y barato. Ideal cuando *siempre* necesitás la base.
- **RAG como tool de un agente**: el modelo decide si buscar. Para "hola" no busca; para "¿qué dice la cláusula 4 del contrato de ACME?" busca, lee, y si el resultado no alcanza, **refina la consulta y busca de nuevo** (*agentic retrieval*). Más flexible y potente para preguntas que requieren varias búsquedas o donde a veces no hace falta buscar.

```ts
const toolBuscar: Anthropic.Tool = {
  name: "buscar_en_base_de_conocimiento",
  description:
    "Busca fragmentos relevantes en la base de conocimiento de la empresa. " +
    "Usala cuando la pregunta requiera información de documentos internos, " +
    "contratos o políticas. Podés llamarla varias veces refinando la consulta.",
  input_schema: {
    type: "object",
    properties: { consulta: { type: "string", description: "Qué buscar, en lenguaje natural" } },
    required: ["consulta"],
  },
};
// En ejecutarTool: si name === "buscar_...", corrés el pipeline de RAG (módulo 5 de rag.md),
// filtrado por permisos del usuario (módulo 9 de rag.md), y devolvés los chunks como tool_result.
```

Dos cosas que se trasladan tal cual desde RAG y son **más** importantes en un agente: el **filtrado por permisos** (módulo 9 de RAG) sigue siendo obligatorio —un agente con una tool de búsqueda sin filtro puede filtrar data entre tenants con más libertad que un pipeline fijo—; y el **grounding** (módulo 8 de RAG: "respondé solo con lo recuperado, citá la fuente") se vuelve crítico cuando el agente encadena búsquedas y razona sobre los resultados.

El cierre del hilo conductor del track: el mismo sistema que arrancó como **una llamada simple**, pasó a **extractor estructurado**, después a **RAG sobre tus apuntes**, ahora es un **agente que decide cuándo consultar esos apuntes** (y qué otras tools usar). Lo único que falta para que sea de producción es saber **medirlo** — el último módulo del track: Evals.

**Ejercicios 9**
9.1 ¿Qué diferencia hay entre usar RAG como workflow y usarlo como herramienta de un agente? Dá un caso donde gane cada uno.
9.2 ¿Qué es el "agentic retrieval" y por qué es más potente para algunas preguntas?
9.3 ¿Qué dos cosas de RAG se trasladan al agente y se vuelven más críticas? ¿Por qué?

---

## Módulo 10 — El Claude Agent SDK y dónde corre el loop

**Teoría.** Podés escribir el agentic loop a mano (módulo 3) —y conviene entenderlo así—, pero para producción hay herramientas que te ahorran el andamiaje. La decisión clave de arquitectura es **dónde corre el loop y dónde se ejecutan las tools.**

- **Vos hospedás el loop (Claude Agent SDK / tool runner).** El SDK trae el loop, el manejo de tool calls, gestión de contexto y utilidades —el mismo loop del módulo 3, pero robusto y mantenido—. *Tu* infraestructura ejecuta las herramientas. Lo elegís cuando querés control total: tu propio runtime de tools, tus gates de aprobación, tu logging, integración en tu backend NestJS como un service más. Es el camino por defecto para un backend que ya tenés.

- **Anthropic hospeda el loop y un sandbox (Managed Agents).** Creás una configuración de agente persistente (modelo, prompt, tools) y abrís *sesiones*; Anthropic corre el loop **y** provee un contenedor aislado donde se ejecutan las tools (bash, archivos, código). Vos mandás mensajes y recibís un stream de eventos. Lo elegís cuando querés un agente stateful con workspace propio sin gestionar la compute —un agente de coding con su sandbox, un agente de research de larga duración—.

La regla de elección: **¿querés que Anthropic corra el loop y hostee el sandbox de ejecución, o querés hospedar vos la compute y tu runtime de tools?** Lo segundo (SDK propio) es lo natural cuando ya tenés un backend y querés el agente *adentro* de él. Lo primero (Managed) cuando querés delegar toda la infraestructura del agente.

Una decisión que no cambia con la herramienta: **el modelo mental es el mismo de todo el módulo** —loop, tools, contexto, confiabilidad—. El SDK o Managed Agents te dan el andamiaje; el criterio de diseño (qué tools, qué guardarraíles, cuándo un agente) sigue siendo tuyo. Y el consejo de Anthropic se mantiene: usá la abstracción más simple que resuelva tu caso; si una llamada directa a la API con tu propio loop chico alcanza, no metas un framework pesado encima.

**Ejercicios 10**
10.1 ¿Cuál es la decisión de arquitectura central al elegir cómo correr un agente?
10.2 ¿Cuándo elegís el Claude Agent SDK (loop propio) y cuándo Managed Agents? Dá la pregunta que los separa.
10.3 ¿Qué te dan estas herramientas y qué sigue siendo responsabilidad tuya?

---

## Módulo 11 — El criterio integrador: ¿vale la pena un agente?

**Teoría.** El módulo que se evalúa en una entrevista de AI engineer, y el cierre del módulo 2. Antes de construir un agente, pasá la tarea por **cuatro preguntas** (el gate de Anthropic). Si alguna da "no", quedate en un escalón más simple (workflow o llamada):

1. **Complejidad**: ¿la tarea es multi-paso y difícil de especificar de antemano? "Convertí este doc de diseño en un PR" sí; "extraé el título de este PDF" no (eso es una llamada).
2. **Valor**: ¿el resultado justifica el costo y la latencia extra de un agente? Un agente que ahorra 2 segundos en algo trivial no vale su complejidad.
3. **Viabilidad**: ¿Claude es realmente capaz en este tipo de tarea? Si el modelo falla seguido en el dominio, un loop solo amplifica los errores.
4. **Costo del error**: ¿los errores se pueden detectar y recuperar (tests, review, rollback)? Un agente que actúa sobre algo irreversible y sin red de seguridad es un riesgo, no una solución.

La síntesis de todo el módulo: un agente es la herramienta correcta cuando la tarea es **genuinamente abierta** (no podés cablear los pasos), el **valor es alto**, el modelo es **capaz**, y los **errores son recuperables**. Cuando esas cuatro se cumplen, la autonomía paga; cuando alguna falla, el costo y el no determinismo superan el beneficio.

Y el criterio que atraviesa todo el track de IA, una última vez: **la mejor solución es la más simple que resuelve el problema real.** Para un LLM era "¿necesito un modelo o un `if`?"; para RAG, "¿busco en muchos docs o trabajo con pocos?"; para agentes, **"¿necesito que el modelo decida el camino, o puedo escribirlo yo?"**. Saber responder esas tres —y elegir el nivel correcto en vez del más vistoso— es lo que demuestra criterio senior en IA aplicada. El paso que falta para cerrar el track es aprender a **medir** si el sistema que construiste de verdad funciona: las **Evaluations**.

**Ejercicios 11**
11.1 Nombrá las cuatro preguntas del gate para decidir si construir un agente.
11.2 Para cada caso, decí si es agente, workflow o llamada simple, y por qué: (a) clasificar un email en 3 categorías; (b) "resolvé este issue de GitHub" con tests que validan; (c) "buscá, resumí y traducí" siempre en ese orden.
11.3 ¿Cuál es la pregunta-criterio de agentes, y cómo se relaciona con las de LLMs y RAG del track?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Un agente es un LLM puesto en un loop que decide solo qué herramientas usar para cumplir
    un objetivo. Está hecho del tool use (function calling) del módulo de LLMs, repetido en
    un ciclo autónomo.
1.2 Quién decide el camino: en un workflow lo decidís vos en código (pasos fijos); en un
    agente lo decide el modelo (elige qué tools, en qué orden, cuándo terminó). Workflow =
    determinista; agente = no determinista.
1.3 Porque sin el loop autónomo —sin que el modelo decida, observe el resultado y vuelva a
    decidir— es solo tool use de un paso. El agente es ese ciclo de decisión repetido.
```

### Módulo 2
```
2.1 (1) Una sola llamada al LLM. (2) Un workflow (pasos orquestados por tu código). (3) Un
    agente (el modelo decide el camino). Cada salto agrega costo, latencia, no determinismo y
    superficie de error.
2.2 La pregunta: ¿puedo escribir los pasos de antemano? Si sí → workflow (ej. "buscá, resumí,
    clasificá"). Si no → agente (ej. "resolvé este bug", donde los pasos dependen de lo que
    vayas encontrando).
2.3 Porque cada escalón agrega costo/latencia/no determinismo; empezar simple evita el error
    #1 (armar un agente cuando un workflow o una llamada alcanzaban) y mantiene el sistema
    barato, predecible y debuggeable.
```

### Módulo 3
```
3.1 (1) Llamás a la API con el objetivo y las tools. (2) Si stop_reason es "end_turn", el
    modelo terminó → salís. (3) Si pidió tools, las ejecutás, agregás los resultados al
    historial y volvés al paso 1. Termina cuando el modelo responde sin pedir más tools.
3.2 El estado vive en el array de messages: como la API es stateless, acumulás todo el
    historial (lo que el modelo pidió y lo que las tools devolvieron) y lo reenviás completo
    en cada llamada. Implica que cada vuelta es más cara (historial creciente).
3.3 Para evitar (1) loops infinitos (el modelo nunca decide terminar) y (2) quemar el
    presupuesto de tokens (cada vuelta cuesta sobre un historial cada vez más grande). El tope
    es un guardarraíl.
```

### Módulo 4
```
4.1 bash le da al modelo casi cualquier capacidad, pero a tu harness solo un string opaco: no
    podés distinguir ni gatear acciones concretas. Promovés a tool dedicada cuando necesitás
    controlar esa acción: seguridad/gating, confirmación, UI propia o paralelización.
4.2 La reversibilidad: las acciones difíciles de revertir (mandar mail, borrar, cobrar) son
    las que gateás con confirmación. Ej.: una tool enviar_email es fácil de gatear; un bash
    con curl no.
4.3 Mínimo privilegio: obtener_tarea(id) le da al agente solo el poder justo y un blast radius
    acotado; ejecutar_sql_arbitrario(query) le da poder ilimitado sobre la base (una inyección
    SQL esperando a pasar). Diseñás tools como una API interna: superficie mínima.
```

### Módulo 5
```
5.1 Prompt chaining: descomponés en pasos secuenciales donde la salida de uno es la entrada
    del siguiente (ej. esquema → texto → traducción). Routing: una llamada clasifica la
    entrada y la dirige a un prompt/modelo especializado (ej. clasificar un ticket y mandarlo
    al flujo y modelo adecuados).
5.2 En orchestrator-workers las subtareas NO se conocen de antemano: el orquestador las decide
    dinámicamente según la entrada. En la paralelización fija vos definís las subtareas por
    adelantado.
5.3 Un agente es, en gran parte, estos patrones decididos dinámicamente por el modelo en vez
    de cableados por vos. Cuando podés cablear el patrón (lo conocés de antemano), el workflow
    es preferible: más barato, predecible y debuggeable.
```

### Módulo 6
```
6.1 El problema N×M: cada integración LLM↔herramienta era un adaptador a medida. MCP es un
    protocolo estándar para conectar LLMs con servidores de herramientas. Se lo compara con el
    "USB-C de las herramientas de IA": un mismo enchufe para todo.
6.2 El servidor en mcp_servers ({type, url, name}) y un mcp_toolset en tools que lo referencia
    por nombre (mcp_server_name). Si falta el mcp_toolset (o el nombre no coincide), es un
    error de validación.
6.3 No cambia el loop: el modelo sigue pidiendo tools y observando resultados. Cambia de dónde
    vienen las herramientas (un protocolo estándar en vez de funciones de tu repo). Siguen
    aplicando: mínimo privilegio (qué servidores/tools habilitás), tratar la salida como no
    confiable, y cuidar las credenciales (auth de MCP).
```

### Módulo 7
```
7.1 Porque un agente largo acumula cada tool result y cada paso de razonamiento en el
    historial, que crece sin parar; un workflow corto no. Los dos costos de un contexto
    enorme: plata (más tokens) y calidad (lost in the middle: el modelo rinde peor).
7.2 Context editing PODA (borra del historial lo que ya no sirve); compaction RESUME (condensa
    el historial anterior cuando te acercás al límite). La memoria persiste información ENTRE
    sesiones (sobrevive al reinicio), algo que ninguna de las dos —que operan dentro de una
    sesión— resuelve.
7.3 Porque invalida el prompt caching: la caché es prefix-match, así que cambiar el system
    prompt (o las tools) a mitad de sesión invalida todo el prefijo cacheado. Conecta con el
    módulo de prompt caching de LLMs ("lo estable primero, lo variable al final").
```

### Módulo 8
```
8.1 (1) Tope de iteraciones/costo → evita loops infinitos y quemar tokens. (2) Manejar errores
    de tools (tool_result con is_error) → el agente se adapta en vez de tumbarse. (3)
    Human-in-the-loop para acciones irreversibles → evita daño no recuperable. (También:
    validar argumentos antes de ejecutar; mínimo privilegio.)
8.2 Le devolvés el error al modelo como tool_result con is_error: true y un mensaje útil; el
    modelo suele adaptarse (reintentar distinto, pedir aclaración). No lo tragás en silencio
    (el agente seguiría a ciegas) ni dejás que tumbe el loop (perdés la corrida entera).
8.3 Porque en un RAG una inyección sesga una respuesta, pero en un agente con tools puede
    hacer que el agente EJECUTE acciones reales. Combinás: separar el input externo de las
    instrucciones (tratarlo como hostil) y mínimo privilegio (que aunque sea secuestrado, no
    pueda hacer daño grave).
```

### Módulo 9
```
9.1 Como workflow, toda pregunta dispara una recuperación siempre, antes de responder
    (predecible, barato; gana cuando siempre necesitás la base). Como tool de un agente, el
    modelo decide si buscar, qué buscar y cuántas veces (gana en preguntas que a veces no
    requieren la base o que necesitan varias búsquedas refinadas).
9.2 Es cuando el agente busca, evalúa si el resultado alcanza, y si no, refina la consulta y
    busca de nuevo. Es más potente porque preguntas complejas rara vez se resuelven con una
    sola búsqueda; el agente itera hasta tener lo que necesita.
9.3 (1) El filtrado por permisos: un agente con una tool de búsqueda sin filtro puede filtrar
    data entre tenants con más libertad que un pipeline fijo. (2) El grounding (responder solo
    con lo recuperado, citar la fuente): crítico cuando el agente encadena búsquedas y razona
    sobre los resultados.
```

### Módulo 10
```
10.1 Dónde corre el loop y dónde se ejecutan las tools: vos hospedás la compute (SDK propio) o
     Anthropic hospeda el loop y un sandbox (Managed Agents).
10.2 La pregunta: ¿querés que Anthropic corra el loop y hostee el sandbox de ejecución, o
     hospedar vos la compute y tu runtime de tools? SDK propio (Claude Agent SDK) cuando
     querés el agente adentro de tu backend con control total; Managed Agents cuando querés un
     agente stateful con workspace sin gestionar infraestructura.
10.3 Te dan el andamiaje (el loop robusto, manejo de tool calls, gestión de contexto, y en
     Managed también el sandbox). Sigue siendo tuyo el criterio de diseño: qué tools, qué
     guardarraíles, y cuándo conviene un agente.
```

### Módulo 11
```
11.1 (1) Complejidad (¿multi-paso y difícil de especificar de antemano?). (2) Valor (¿justifica
     el costo y la latencia extra?). (3) Viabilidad (¿Claude es capaz en esta tarea?). (4)
     Costo del error (¿los errores se detectan y recuperan?).
11.2 (a) Llamada simple (o routing): es un paso, especificable. (b) Agente: tarea abierta,
     multi-paso, con tests que hacen el error recuperable. (c) Workflow: los pasos son fijos y
     conocidos de antemano ("buscá → resumí → traducí"), no hace falta que el modelo decida.
11.3 La pregunta de agentes: ¿necesito que el modelo decida el camino, o puedo escribirlo yo?
     Es la misma lógica del track subida de nivel: en LLMs era "¿necesito un modelo o un if?";
     en RAG, "¿busco en muchos docs o trabajo con pocos?"; en agentes, "¿autonomía o control?".
     Las tres apuntan a lo mismo: la solución más simple que resuelve el problema real.
```

---

## Siguientes pasos

Con este módulo cerrás el armado de un agente con criterio de producción: qué es el loop y qué no, la escalera llamada→workflow→agente, el agentic loop por dentro, el diseño de herramientas, los patrones de workflow, MCP, la gestión de contexto, la confiabilidad y los guardarraíles, RAG como tool del agente, y dónde correr el loop (SDK propio vs Managed Agents) — más el gate de cuatro preguntas para decidir si un agente vale la pena. El último módulo del track de IA es **Evaluations**: cómo medir la calidad de un sistema LLM no determinista de punta a punta —el equivalente a tests + observabilidad para algo que no da siempre la misma salida—, que asomó en el módulo 11 de RAG (recall@k, faithfulness, LLM-as-judge) y es lo que separa un prototipo de un sistema en producción. Con eso, el hilo conductor del track queda completo: el mismo sistema que arrancó como una llamada simple, creció a extractor, a RAG sobre tus apuntes, a un agente que decide su camino, y finalmente queda **medido** —un proyecto de portfolio coherente y completo para el rol de AI engineer.
