# AI Agents en Python: LangGraph, AutoGen y el multi-agente

**Frameworks de agentes en el ecosistema Python · grafos de estado, conversación multi-agente y criterio de producción · stack Claude · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Es el quinto módulo del **track Python AI**: asume [Python](python.md), [Prompt Engineering](prompt-engineering.md) y [Vector Databases](vector-dbs.md). Y asume el módulo conceptual de [Agentes](agentes.md) del track general —qué es un agente (un LLM en un loop que decide y actúa), la escalera llamada→workflow→agente, el diseño de tools, MCP, la confiabilidad y el gate de cuatro preguntas—: **no repetimos esos conceptos, los damos por sabidos**. Ahí el stack era TS + Claude Agent SDK y el loop lo escribías a mano. Acá miramos el otro lado: **los frameworks de agentes de Python que automatizan ese loop —LangGraph y AutoGen— y el salto al multi-agente**, que es lo que más aparece en los avisos de AI Engineer. El objetivo es doble: el **cómo** (el agente como grafo de estado, la conversación entre agentes, checkpointing, human-in-the-loop) y el **criterio** (framework vs a mano, single vs multi-agente) que separa al que ensambla cajas del que entiende lo que ensambla.

**Lo que asumimos.** Python (tipos, async, `TypedDict`/Pydantic), el agentic loop conceptual de [Agentes](agentes.md) (loop, tools, contexto, guardarraíles), retrieval de [Vector Databases](vector-dbs.md), y el SDK de Anthropic.

**Para practicar.**

```bash
uv add langgraph langchain-anthropic langgraph-checkpoint-sqlite
# multi-agente (opcional): uv add autogen-agentchat autogen-ext
# export ANTHROPIC_API_KEY=...
```

> Nota sobre datos volátiles: los frameworks de agentes son de las APIs que **más cambian** del ecosistema (LangGraph evoluciona rápido; AutoGen tuvo una reescritura grande de la v0.2 a la v0.4). Los ejemplos muestran la **forma** de cada API a 2026; verificá nombres de paquetes, clases y firmas en la doc oficial antes de implementar. Cuando un ejemplo no compile, la fuente de verdad es la doc de LangGraph (publican guía de migración entre versiones) y el paquete `langchain-anthropic` —`create_react_agent`, por caso, cambió de paquete y de firma entre versiones—. Datos de modelos Claude (IDs, precios, thinking), con la skill `claude-api`.

**Índice de módulos**
1. Por qué un framework de agentes en Python (el puente desde el loop a mano)
2. El panorama: LangGraph, AutoGen, CrewAI y los SDKs de proveedores
3. LangGraph I: el agente como grafo de estado (StateGraph, nodos, edges)
4. LangGraph II: el loop agéntico como ciclo del grafo (tools + conditional edges)
5. LangGraph III: persistencia, memoria e interrupts (checkpointing + human-in-the-loop)
6. AutoGen: agentes que conversan (el paradigma multi-agente por diálogo)
7. Multi-agente: supervisor, group chat y cuándo NO usarlo
8. Tools, RAG y MCP en el mundo Python (conectar las piezas del track)
9. Confiabilidad y guardarraíles en frameworks
10. Observabilidad y evaluación de agentes (la trayectoria, no solo la respuesta)
11. El criterio de cierre: framework vs a mano, y el puente del track

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Por qué un framework de agentes en Python (el puente desde el loop a mano)

**Teoría.** En [Agentes](agentes.md) escribiste el agentic loop **a mano**: un `while`/`for` que llama a la Messages API, mira `stop_reason`, ejecuta las tools que el modelo pidió, reinyecta los resultados al historial y vuelve a llamar, con un tope de iteraciones. Esa versión es la base —entenderla es no negociable— pero deja **todo el andamiaje** a tu cargo: el estado, la persistencia, reanudar tras una caída, pausar para pedir confirmación humana, coordinar varios agentes, observar qué pasó. Para un agente de juguete alcanza con el loop a mano; para uno de producción, reimplementar ese andamiaje bien (y a prueba de fallos) es mucho trabajo y fácil de hacer mal.

Eso es lo que resuelven los **frameworks de agentes** de Python. Es la misma capa de frameworks que viste en [Vector Databases](vector-dbs.md) módulo 11 (LangChain/LlamaIndex para RAG), ahora para agentes: **automatizan el loop y le agregan la maquinaria de producción**. Concretamente, te dan:

- **El loop ya resuelto**: pedir tools → ejecutarlas → reinyectar → repetir, con su tope de recursión.
- **Estado explícito y tipado**: qué información viaja entre pasos, y cómo se acumula.
- **Persistencia / checkpointing**: guardar el estado del agente para reanudar tras una caída, recordar entre turnos, o "viajar en el tiempo" a un paso anterior.
- **Human-in-the-loop de primera clase**: pausar el agente antes de una acción y esperar input humano (el *gating* de [Agentes](agentes.md) módulo 8, pero como primitiva del framework).
- **Multi-agente**: coordinar varios agentes especializados (módulos 6–7).
- **Observabilidad**: tracing de cada decisión y cada tool call (módulo 10).

Y el criterio, idéntico al del framework layer de RAG ([Vector Databases](vector-dbs.md) módulo 11): **un framework es leverage cuando entendés lo que automatiza, y un pasivo cuando es lo único que sabés.** Por eso el orden importa: primero el loop a mano ([Agentes](agentes.md)), después el framework. Si saltás directo a LangGraph sin entender el loop, vas a tratar al framework como magia —y cuando el agente entre en loop, alucine o gaste de más, no vas a tener con qué razonarlo—. La frase mental: **los frameworks de agentes no reemplazan al loop a mano; lo industrializan —estado, persistencia, HITL, multi-agente y observabilidad— para lo que el loop a mano hace incómodo o frágil—.**

**Ejercicios 1**
1.1 ¿Qué deja a tu cargo el agentic loop escrito a mano que un framework de agentes resuelve? Nombrá cuatro cosas.
1.2 ¿Por qué el orden "primero el loop a mano, después el framework" no es un capricho pedagógico?
1.3 Conectá esta capa con el framework layer de RAG ([Vector Databases](vector-dbs.md) módulo 11): ¿qué tienen en común el criterio de uno y otro?

---

## Módulo 2 — El panorama: LangGraph, AutoGen, CrewAI y los SDKs de proveedores

**Teoría.** El ecosistema Python de agentes tiene varias piezas con filosofías distintas. Conocer el mapa es lo que te deja elegir, no seguir la moda.

- **LangGraph** (equipo de LangChain). Modela el agente como un **grafo de estado**: vos definís los nodos y las conexiones, y el control de flujo es **explícito** (incluyendo ciclos). Es de más bajo nivel que los "agentes" de LangChain, pensado para producción: estado tipado, checkpointing, human-in-the-loop, durabilidad. **Es el más pedido** en los avisos y el que más profundizamos (módulos 3–5). Filosofía: *el agente como máquina de estados que vos diseñás y el modelo recorre.*

- **AutoGen** (Microsoft). Modela el problema como **agentes que conversan** entre sí para resolver una tarea (multi-agente por diálogo). Más orientado a investigación y a la coordinación emergente. Tuvo una **reescritura grande** (v0.2 → v0.4, ahora async y event-driven). Filosofía: *una sociedad de agentes que dialogan.* (Módulo 6.)

- **CrewAI**. Multi-agente **basado en roles**: definís un "crew" de agentes con rol, objetivo y backstory, y tareas. Opinado y rápido de arrancar; menos control fino que LangGraph. Filosofía: *un equipo con roles que ejecuta un proceso.* (No le damos ejemplo propio a propósito: es muy opinado y de bajo techo de control, y si dominás el grafo explícito de LangGraph y el diálogo de AutoGen, CrewAI se aprende en una tarde. Aparece en avisos, así que conviene reconocerlo —no profundizarlo.)

- **Los SDKs de proveedores**:
  - **Claude Agent SDK** (Anthropic, también en Python): el harness de producción de [Agentes](agentes.md) módulo 10 —loop + tools built-in + contexto + subagentes + MCP—, atado al modelo de Anthropic.
  - **OpenAI Agents SDK** (ex-Swarm): liviano, centrado en *handoffs* entre agentes.

El eje que ordena el panorama es **control explícito vs coordinación emergente**:

| Framework | Modelo mental | Control | Brilla en |
|---|---|---|---|
| **LangGraph** | Grafo de estado | Explícito (vos cableás el flujo + ciclos) | Producción, estado/durabilidad, HITL |
| **AutoGen** | Agentes que conversan | Emergente (el diálogo decide) | Multi-agente exploratorio, research |
| **CrewAI** | Equipo con roles | Medio (proceso + roles) | Prototipos multi-agente rápidos |
| **Claude/OpenAI SDK** | Harness del proveedor | Atado al proveedor | Un agente con el stack del proveedor |

La regla de criterio: **empezá por el que te dé el control que tu caso necesita con la menor complejidad.** Para un agente único, robusto, dentro de tu backend, LangGraph (o el loop a mano + el SDK del proveedor) suele ganar por previsibilidad. Para experimentar con varios agentes que colaboran, AutoGen/CrewAI arrancan más rápido. Y la advertencia de siempre ([Agentes](agentes.md) módulo 2): **no metas multi-agente porque suena avanzado** —subí ese escalón solo cuando un agente único te queda corto (módulo 7)—. La frase mental: **el panorama se ordena por control explícito (LangGraph) vs coordinación emergente (AutoGen/CrewAI); elegí por el control y la complejidad que tu caso pide, no por cuál suena más impresionante.**

**Ejercicios 2**
2.1 Diferenciá la filosofía de LangGraph y la de AutoGen en una frase cada una.
2.2 ¿Qué eje ordena el panorama de frameworks de agentes? Ubicá LangGraph y AutoGen en ese eje.
2.3 ¿Cuándo elegirías LangGraph y cuándo AutoGen/CrewAI, según el tipo de agente que querés construir?

---

## Módulo 3 — LangGraph I: el agente como grafo de estado (StateGraph, nodos, edges)

**Teoría.** La abstracción central de LangGraph es el **`StateGraph`**: un grafo donde el agente es un conjunto de **nodos** (pasos) conectados por **edges** (transiciones), operando sobre un **estado** compartido. Es "el agente como máquina de estados". Las tres piezas:

1. **El estado**: un objeto tipado (un `TypedDict` o un modelo Pydantic) que **viaja por el grafo**. Cada nodo lo lee y devuelve una actualización parcial. Es la "memoria de trabajo" del agente —el equivalente al array `messages` que acumulabas a mano en [Agentes](agentes.md) módulo 3, ahora declarado como un esquema—.
2. **Los nodos**: funciones Python `(estado) -> actualización_parcial`. Un nodo típico llama al LLM; otro ejecuta tools; otro hace una validación. El nodo no muta el estado: **devuelve** los campos a actualizar.
3. **Los edges**: las conexiones. Hay **edges normales** (de A siempre se pasa a B) y **edges condicionales** (una función mira el estado y decide a qué nodo ir). `START` y `END` son los nodos especiales de entrada y salida.

El grafo más simple —un solo nodo que llama al modelo—:

```python
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import HumanMessage

class Estado(TypedDict):
    # add_messages es un REDUCER: define que las actualizaciones se APPENDEAN, no sobrescriben (módulo 4)
    messages: Annotated[list, add_messages]

llm = ChatAnthropic(model="claude-opus-4-8")

def nodo_agente(estado: Estado) -> dict:
    # lee el historial, llama al modelo, devuelve el mensaje nuevo (se appendea al estado)
    return {"messages": [llm.invoke(estado["messages"])]}

grafo = StateGraph(Estado)
grafo.add_node("agente", nodo_agente)
grafo.add_edge(START, "agente")          # entrada → agente
grafo.add_edge("agente", END)            # agente → fin
app = grafo.compile()                     # compila a algo invocable

resultado = app.invoke({"messages": [HumanMessage("Explicá qué es HNSW en una frase.")]})
print(resultado["messages"][-1].content)
```

Por qué un grafo y no solo un `while`: el grafo hace **explícito el control de flujo**. Ves de un vistazo qué pasos hay, cómo se conectan, dónde se ramifica y dónde cicla —y eso se puede **visualizar, versionar, testear y reanudar**—. El `while` a mano tiene el flujo enredado en `if`s; el grafo lo declara. Esa explicitud es la base de todo lo que viene (ciclos para el loop agéntico, checkpointing, interrupts). La frase mental: **LangGraph modela el agente como un grafo de estado —nodos que transforman un estado tipado, conectados por edges normales o condicionales—; el control de flujo deja de estar implícito en un `while` y pasa a ser una estructura explícita que podés inspeccionar y persistir.**

**Ejercicios 3**
3.1 Nombrá las tres piezas de un `StateGraph` y qué hace cada una.
3.2 ¿Por qué un nodo **devuelve** una actualización parcial en vez de mutar el estado? ¿Con qué estructura del loop a mano de [Agentes](agentes.md) se corresponde el estado?
3.3 ¿Qué diferencia hay entre un edge normal y uno condicional?
3.4 ¿Qué ganás modelando el agente como grafo explícito en vez de un `while` con `if`s?

---

## Módulo 4 — LangGraph II: el loop agéntico como ciclo del grafo (tools + conditional edges)

**Teoría.** Acá LangGraph muestra su razón de ser: el **agentic loop** de [Agentes](agentes.md) módulo 3 —el modelo pide tools, las ejecutás, reinyectás, repetís— es exactamente un **ciclo en el grafo**. Dos nodos y un edge condicional:

- Nodo **`agente`**: llama al LLM (con las tools *bindeadas*). El modelo decide: ¿responde, o pide ejecutar tools?
- Nodo **`tools`**: ejecuta las tools que el modelo pidió y mete los resultados en el estado.
- **Edge condicional** desde `agente`: si el último mensaje del modelo trae *tool calls* → ir a `tools`; si no → `END`.
- **Edge normal** `tools → agente`: tras ejecutar, **volver al agente** para que observe los resultados y decida el próximo paso. **Ese edge de vuelta es el ciclo** —el `while` del loop a mano, hecho arista—.

```python
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.tools import tool

@tool
def buscar_en_kb(consulta: str) -> str:
    """Busca fragmentos relevantes en la base de conocimiento. Usala cuando la pregunta
    requiera información de documentos internos."""
    # acá corrés tu retrieval (vector DB del módulo de Vector Databases) y devolvés texto
    return retrieval(consulta)

llm_con_tools = llm.bind_tools([buscar_en_kb])   # el modelo ahora "sabe" de la tool

def nodo_agente(estado: Estado) -> dict:
    return {"messages": [llm_con_tools.invoke(estado["messages"])]}

grafo = StateGraph(Estado)
grafo.add_node("agente", nodo_agente)
grafo.add_node("tools", ToolNode([buscar_en_kb]))     # ejecutor de tools, ya provisto
grafo.add_edge(START, "agente")
grafo.add_conditional_edges("agente", tools_condition) # ¿hay tool_calls? → "tools"; si no → END
grafo.add_edge("tools", "agente")                      # ← EL CICLO: tras las tools, vuelve al agente
app = grafo.compile()

app.invoke({"messages": [HumanMessage("¿Qué dice la política de vacaciones?")]},
           {"recursion_limit": 25})   # el tope contra loops infinitos de Agentes módulo 8, ahora como config
```

Dos piezas listas para usar que conviene conocer:
- **`ToolNode`**: el nodo que ejecuta las tools pedidas (te ahorra escribir el "para cada tool_call, ejecutá y armá el tool_result" del loop a mano).
- **`tools_condition`**: la función de routing estándar que mira si el último mensaje trae tool calls.
- Y el **`recursion_limit`**: el tope de pasos del grafo —el guardarraíl contra loops infinitos de [Agentes](agentes.md) módulo 8, ahora un parámetro—. Si el agente lo supera, LangGraph corta con error en vez de quemar tokens para siempre. Ojo con la cuenta: cuenta **super-steps del grafo**, no turnos del modelo —cada vuelta `agente → tools → agente` consume ~2 super-steps—, así que `recursion_limit=25` permite ~12 vueltas, no 25. Dimensionalo en ≈ 2× la profundidad de pasos que esperás, no como número mágico.

Para el caso común hay un atajo que arma todo este grafo por vos:

```python
from langgraph.prebuilt import create_react_agent
app = create_react_agent(llm, tools=[buscar_en_kb])   # el grafo agente↔tools, prearmado
```

Pero el valor de LangGraph aparece cuando **NO** usás el atajo: cuando insertás nodos propios (validación, un gate humano, un router a otro subgrafo) en medio del ciclo. Ahí el grafo explícito te deja meter mano donde un loop cerrado no. La frase mental: **el loop agéntico es un ciclo en el grafo —nodo `agente` ↔ nodo `tools` cerrado por un edge condicional—; `create_react_agent` te lo da hecho, pero el poder de LangGraph es poder abrir ese ciclo e insertar tus propios nodos—.**

**Ejercicios 4**
4.1 Expresá el agentic loop de [Agentes](agentes.md) módulo 3 como grafo: ¿qué nodos, qué edge condicional, y cuál es exactamente "el ciclo"?
4.2 ¿Qué hacen `ToolNode` y `tools_condition`, y qué parte del loop a mano te ahorran?
4.3 ¿Qué es el `recursion_limit` y con qué guardarraíl de [Agentes](agentes.md) se corresponde?
4.4 ¿Qué te da `create_react_agent` y por qué el verdadero valor de LangGraph aparece cuando NO lo usás?

---

## Módulo 5 — LangGraph III: persistencia, memoria e interrupts (checkpointing + human-in-the-loop)

**Teoría.** Lo que separa a LangGraph de un loop a mano —y la razón principal para usarlo— es el **checkpointing**: la capacidad de **persistir el estado del grafo** después de cada paso. Le pasás un *checkpointer* al compilar, y cada ejecución se asocia a un **`thread_id`**. De ahí salen cuatro superpoderes que a mano costaría muchísimo:

1. **Memoria entre turnos**: con el mismo `thread_id`, el agente "recuerda" la conversación anterior sin que vos reenvíes el historial —el checkpointer lo tiene guardado—.
2. **Durabilidad**: si el proceso se cae a mitad de una tarea larga, reanudás desde el último checkpoint en vez de empezar de cero.
3. **Human-in-the-loop**: podés **pausar** el grafo antes de un paso, inspeccionar el estado, y reanudar (con o sin modificarlo).
4. **Time travel**: volver a un checkpoint anterior y reanudar por otro camino (debugging, "qué hubiera pasado si").

```python
from langgraph.checkpoint.memory import MemorySaver   # en prod: SqliteSaver / PostgresSaver

app = grafo.compile(checkpointer=MemorySaver())
config = {"configurable": {"thread_id": "usuario-7"}}   # el "hilo" de esta conversación

app.invoke({"messages": [HumanMessage("Me llamo Jorge.")]}, config)
r = app.invoke({"messages": [HumanMessage("¿Cómo me llamo?")]}, config)
print(r["messages"][-1].content)
# → "Jorge": recuerda, porque el checkpointer guardó el estado del thread "usuario-7"
```

**`MemorySaver` es para demos; en producción el checkpointer es persistente y el `thread_id` es por-usuario.** `MemorySaver` guarda en RAM: se pierde al reiniciar el proceso y no se comparte entre workers. Cuando servís el agente detrás de una API ([FastAPI](fastapi.md)) con varios requests concurrentes, dos cosas cambian: (1) el checkpointer va a **`SqliteSaver`/`PostgresSaver`** para que el estado sobreviva a un reinicio y lo vean todos los workers; (2) el `thread_id` se **deriva de la sesión/usuario** (`f"user-{user_id}-{conv_id}"`), no es una constante global —si todos comparten `"usuario-7"`, las conversaciones se pisan—. Esa es la diferencia entre el demo que anda en tu consola y el endpoint multiusuario que no mezcla a dos personas.

El **human-in-the-loop** es el caso que más rinde, y es el *gating* de acciones irreversibles de [Agentes](agentes.md) módulo 8 vuelto primitiva. Compilás el grafo para que **se interrumpa antes** de un nodo sensible (p. ej. el que ejecuta tools que escriben), inspeccionás qué iba a hacer, y recién entonces reanudás:

```python
# Pausa ANTES de ejecutar tools: revisás la acción propuesta y decidís si seguir
app = grafo.compile(checkpointer=MemorySaver(), interrupt_before=["tools"])

app.invoke({"messages": [HumanMessage("Borrá los proyectos archivados.")]}, config)
# el grafo corre hasta el borde de "tools" y PAUSA; acá inspeccionás el estado / pedís confirmación
estado = app.get_state(config)              # ver qué tool y con qué args iba a llamar
app.invoke(None, config)                     # invoke(None) = REANUDAR desde el checkpoint
```

(LangGraph moderno también ofrece la función `interrupt()` dentro de un nodo, que pausa y se reanuda con `Command(resume=...)` —más flexible que `interrupt_before`; verificá la versión—.)

Por qué esto es difícil a mano: tendrías que serializar el estado completo del loop, guardarlo, poder reconstruirlo, manejar la reanudación parcial y la concurrencia de threads. El checkpointer te da todo eso como infraestructura probada. **Es, junto al multi-agente, el argumento más fuerte para usar el framework en vez del loop a mano.** La frase mental: **el checkpointing persiste el estado del grafo por `thread_id` y de ahí salen memoria entre turnos, durabilidad, time travel y —sobre todo— el human-in-the-loop como pausa/reanudación de primera clase; replicar eso a mano es justamente lo que no querés reimplementar—.**

**Ejercicios 5**
5.1 ¿Qué es el checkpointing en LangGraph y qué rol cumple el `thread_id`?
5.2 Nombrá los cuatro superpoderes que habilita el checkpointing.
5.3 ¿Cómo se implementa el human-in-the-loop, y con qué defensa de [Agentes](agentes.md) módulo 8 se corresponde?
5.4 ¿Por qué el checkpointing es el argumento más fuerte para usar el framework en vez del loop a mano?

---

## Módulo 6 — AutoGen: agentes que conversan (el paradigma multi-agente por diálogo)

**Teoría.** **AutoGen** (Microsoft) parte de una idea distinta a la de LangGraph: en vez de que vos cablees un grafo, modelás **varios agentes que conversan entre sí** para resolver la tarea. Cada agente tiene un rol (system prompt) y un modelo, y "hablan" en una conversación —se pasan mensajes, se critican, se corrigen— hasta llegar a una solución o a una **condición de terminación**. El control es más **emergente**: el resultado surge del diálogo, no de un flujo que vos dibujaste.

Las piezas clásicas del modelo de AutoGen:

- **Agente asistente** (`AssistantAgent`): un agente con LLM y un rol (ej. "escribís código", "criticás código").
- **Proxy de usuario / ejecutor**: la pieza que aporta input externo o *actúa*. Ojo con la versión: en **v0.2** el `UserProxyAgent` representaba al humano **y** llevaba un code executor (ejecutaba el código que los otros proponían). En **v0.4** se separó: `UserProxyAgent` solo **representa al humano** (pide input vía una función de input) y **quien ejecuta código es `CodeExecutorAgent`** (o se modela como una tool). Si leés "el UserProxyAgent ejecuta código", es material de v0.2.
- **Group chat**: varios agentes en una conversación, con un **manager** que decide quién habla en cada turno.
- **Condición de terminación**: cuándo para la conversación (un mensaje "TERMINATE", un máximo de turnos, una meta cumplida).

```python
# AutoGen AgentChat (v0.4+) — forma de la API; hubo reescritura grande desde v0.2, VERIFICÁ versión
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import MaxMessageTermination

# model_client: un cliente de modelo (Anthropic/OpenAI). Ver doc de autogen-ext para el de Claude.
escritor = AssistantAgent("escritor", model_client=modelo, system_message="Escribís borradores concisos.")
critico  = AssistantAgent("critico",  model_client=modelo, system_message="Criticás y proponés mejoras concretas.")

equipo = RoundRobinGroupChat([escritor, critico], termination_condition=MaxMessageTermination(6))
# escritor y critico se turnan: escribe → critica → reescribe → ... hasta 6 mensajes
resultado = await equipo.run(task="Escribí un párrafo explicando qué es un agente.")
```

Fijate que esto es el patrón **evaluator-optimizer** de [Agentes](agentes.md) módulo 5 (escribir → criticar → reescribir), pero en vez de cablearlo vos, **emerge de la conversación entre dos agentes**. Esa es la promesa de AutoGen: patrones de colaboración expresados como diálogo. Y su riesgo: la coordinación emergente es **menos predecible y más difícil de debuggear** que un grafo explícito —dos agentes pueden quedar dando vueltas, repetirse, o no converger—, por eso las condiciones de terminación y los topes son críticos.

Nota de versión importante: AutoGen pasó de la **v0.2** (la de `ConversableAgent`/`GroupChat` síncrona, la que verás en la mayoría de los tutoriales viejos) a la **v0.4**, una reescritura **async y event-driven** con `autogen-core` (bajo nivel) y `autogen-agentchat` (alto nivel). Si seguís un tutorial, fijate de qué versión es —las APIs no son compatibles—. La frase mental: **AutoGen modela la tarea como agentes que conversan hasta converger (multi-agente emergente); es expresivo para patrones colaborativos como escribir↔criticar, pero menos predecible que el grafo explícito de LangGraph, así que las condiciones de terminación no son opcionales—.**

**Ejercicios 6**
6.1 ¿En qué se diferencia el modelo mental de AutoGen del de LangGraph? (cableado explícito vs …)
6.2 Nombrá las piezas clásicas de AutoGen (asistente, proxy/ejecutor, group chat, terminación) y qué hace cada una.
6.3 ¿Con qué patrón de workflow de [Agentes](agentes.md) módulo 5 se corresponde "escritor ↔ crítico", y qué cambia respecto a cablearlo vos?
6.4 ¿Por qué las condiciones de terminación son críticas en AutoGen, y qué cuidado tenés con las versiones (v0.2 vs v0.4)?

---

## Módulo 7 — Multi-agente: supervisor, group chat y cuándo NO usarlo

**Teoría.** El multi-agente —varios agentes que colaboran en vez de uno solo— es lo que más se hypea y lo que peor se usa. Antes de los patrones, el criterio, porque es donde se cae la mayoría: **un sistema multi-agente agrega coordinación, costo, latencia y no determinismo por encima de los de un agente único.** La recomendación (de Anthropic y del sentido común de [Agentes](agentes.md) módulo 2) es **empezar por un agente único bien diseñado con buenas tools**, y subir a multi-agente solo cuando ese agente único genuinamente no alcanza.

Cuándo un agente único **sí** alcanza (la mayoría de los casos): si la tarea se resuelve con un modelo capaz, un buen set de tools y, si hace falta, sub-llamadas. Meter tres agentes a hablar entre sí no lo hace más inteligente; lo hace más caro y más frágil.

Cuándo el multi-agente **gana de verdad**:
- **Especialización fuerte**: subtareas que requieren prompts/tools/conocimiento muy distintos (un agente "investigador" con tools de búsqueda, uno "redactor", uno "verificador"), donde meter todo en un solo agente lo confundiría.
- **Paralelismo real**: subtareas independientes que corren a la vez (acelera, como el *sectioning* de [Agentes](agentes.md) módulo 5).
- **Separación de responsabilidades / contextos**: aislar contextos para que no se contaminen, o dividir un problema demasiado grande para una sola ventana.

Los patrones de coordinación (los mismos del *orchestrator-workers* de [Agentes](agentes.md) módulo 5, ahora como arquitecturas multi-agente):

- **Supervisor / orchestrator-worker**: un agente **supervisor** recibe la tarea, la descompone y **enruta** a agentes trabajadores especializados, después sintetiza sus salidas. Es el patrón más usado y el más controlable. En LangGraph se modela natural: el supervisor es un nodo router con edges condicionales hacia cada worker.
- **Group chat / network**: los agentes conversan en un canal común y un manager decide quién habla (el modelo de AutoGen, módulo 6). Más flexible, menos predecible.
- **Hierarchical**: equipos de agentes con supervisores de supervisores, para problemas grandes.

La elección de framework mapea al patrón: **LangGraph** para supervisor/jerárquico con control explícito (vos diseñás el routing); **AutoGen/CrewAI** para group chat / equipos con rol donde la coordinación es más conversacional. La frase mental: **el multi-agente es potente para especialización fuerte, paralelismo real o separación de contextos —y un desperdicio caro para todo lo demás—; empezá por un agente único con buenas tools y subí a un supervisor (el patrón más controlable) solo cuando puedas nombrar por qué uno solo no alcanza—.**

**Ejercicios 7**
7.1 ¿Qué le agrega un sistema multi-agente por encima de un agente único, y cuál es la recomendación de arranque?
7.2 Nombrá tres situaciones donde el multi-agente gana de verdad.
7.3 Describí el patrón supervisor/orchestrator-worker y por qué es el más controlable. ¿Cómo se modela en LangGraph?
7.4 ¿Cómo mapea la elección LangGraph vs AutoGen/CrewAI a los patrones de coordinación?

---

## Módulo 8 — Tools, RAG y MCP en el mundo Python (conectar las piezas del track)

**Teoría.** Un agente sin buenas tools es un chat; el diseño de tools (de [Agentes](agentes.md) módulo 4: pocas y enfocadas, descripciones prescriptivas del *cuándo*, mínimo privilegio) vale **igual** acá —el framework no te exime de ese criterio—. Lo que cambia es **cómo** se declaran las tools en Python y cómo se enchufan las piezas que ya construiste en el track.

**Definir tools.** En LangChain/LangGraph, el decorador `@tool` convierte una función Python en una tool: el **nombre**, el **docstring** (que es la descripción que lee el modelo) y los **type hints** (que arman el schema) salen de la función. Por eso en Python el docstring de una tool **no es documentación opcional: es el prompt que decide si el modelo la usa bien**.

```python
from langchain_core.tools import tool

@tool
def cerrar_tarea(tarea_id: int) -> str:
    """Cierra una tarea por su id. Usala SOLO cuando el usuario confirme que la tarea está
    terminada. No la uses para tareas en progreso."""   # ← este texto guía la decisión del modelo
    ...
```

**RAG como tool del agente** (el cierre con [Vector Databases](vector-dbs.md) y [RAG](rag.md)): el retrieval deja de ser un pipeline fijo y se vuelve **una herramienta que el agente decide usar** —el *agentic retrieval* de [Agentes](agentes.md) módulo 9—. La tool `buscar_en_kb` del módulo 4 es exactamente eso: por dentro corre tu búsqueda vectorial (FAISS/pgvector/Qdrant del módulo de Vector Databases), **filtrada por permisos del usuario**, y devuelve los chunks. El agente decide cuándo buscar, con qué consulta, y si refina y vuelve a buscar.

El puente concreto entre el retrieval del track y la tool —el andamiaje que el capstone da por sabido—. La tool no abre el cliente vectorial en cada llamada: lo recibe ya instanciado (por *closure* o como variable de módulo), aplica el filtro de permisos **dentro** de la query, y devuelve un string (lo que el modelo lee):

```python
from langchain_core.tools import tool

# vectordb: el cliente del capstone de Vector Databases, ya conectado (pgvector/Qdrant/FAISS).
# tenant_id: NO viene del modelo; lo fija el código desde la sesión del usuario (mínimo privilegio).
def hacer_tool_buscar(vectordb, tenant_id: str):
    @tool
    def buscar_en_kb(consulta: str) -> str:
        """Busca fragmentos en la base de conocimiento. Usala cuando la pregunta
        requiera información de documentos internos."""
        # el filtro por tenant va EN la query, no se confía al modelo
        resultados = vectordb.similarity_search(consulta, k=4, filter={"tenant_id": tenant_id})
        if not resultados:
            return "Sin resultados en la base de conocimiento."
        # devolvemos texto + fuente para el grounding (el agente cita de acá)
        return "\n\n".join(f"[{d.metadata['fuente']}] {d.page_content}" for d in resultados)
    return buscar_en_kb

# en el armado del grafo: tool = hacer_tool_buscar(vectordb, tenant_id="acme")
```

El `tenant_id` se cierra por closure desde la sesión —el modelo nunca lo elige, así que no puede pedir data de otro tenant—. Dos cosas de RAG que se vuelven **más** críticas en un agente (igual que en [Agentes](agentes.md) módulo 9): el **filtrado por permisos** (un agente con búsqueda sin filtro puede filtrar data entre tenants con más libertad que un pipeline fijo) y el **grounding** (responder solo con lo recuperado, citar la fuente).

**MCP en Python.** El estándar de herramientas de [Agentes](agentes.md) módulo 6 (el "USB-C de las tools de IA") tiene SDK de Python y adaptadores para los frameworks: podés exponer un servidor MCP y consumir sus tools desde un agente LangGraph/AutoGen sin escribir un adaptador a medida por cada servicio. El modelo mental no cambia —el agente sigue pidiendo tools y observando resultados—; cambia **de dónde vienen** (un protocolo estándar en vez de funciones de tu repo), y siguen aplicando mínimo privilegio, tratar la salida como no confiable, y cuidar credenciales.

La frase mental: **en Python las tools se declaran con `@tool` y su docstring ES el prompt que decide su uso; el retrieval del track entra como una tool (agentic retrieval, con permisos y grounding intactos), y MCP te trae tools externas estándar —el criterio de diseño de tools de [Agentes](agentes.md) no cambia, solo su sintaxis—.**

**Ejercicios 8**
8.1 ¿De dónde saca el modelo el nombre, la descripción y el schema de una tool definida con `@tool`? ¿Por qué el docstring no es opcional?
8.2 ¿Cómo entra el RAG del track en un agente, y qué dos cosas de RAG se vuelven más críticas? (conectá con [Agentes](agentes.md) módulo 9)
8.3 ¿Qué aporta MCP en el mundo Python y qué del modelo mental del agente NO cambia al usarlo?

---

## Módulo 9 — Confiabilidad y guardarraíles en frameworks

**Teoría.** Las defensas de confiabilidad de [Agentes](agentes.md) módulo 8 —topes, manejo de errores de tools, validación, human-in-the-loop, defensa contra prompt injection, mínimo privilegio— **no desaparecen porque uses un framework; cambian de forma**. El riesgo nuevo es creer que el framework "ya se encarga": automatiza el andamiaje, no el criterio. Cómo se expresa cada defensa:

- **Tope de iteraciones**: el `recursion_limit` de LangGraph (módulo 4) / la condición de terminación de AutoGen (módulo 6). Sin él, un agente cicla o quema tokens. El framework te da la perilla; ponerla es tuyo.
- **Manejo de errores de tools**: una tool puede fallar (timeout, 404, dato inválido). El framework suele capturar la excepción y devolverla al modelo como resultado de error para que se adapte —pero **vos** decidís qué error es recuperable (que el modelo reintente) y cuál debe abortar el grafo. No tragues el error en silencio. En la práctica hay dos lugares para hacerlo: capturar dentro del cuerpo de la tool y devolver un string de error que el modelo pueda leer, o configurar el `ToolNode` para que maneje las excepciones (la firma exacta cambia entre versiones —verificá la doc—):

  ```python
  @tool
  def cerrar_tarea(tarea_id: int) -> str:
      """Cierra una tarea por su id."""
      try:
          api.cerrar(tarea_id)
          return f"Tarea {tarea_id} cerrada."
      except TareaNoEncontrada:
          # error RECUPERABLE: el modelo puede pedir otra id o avisar al usuario
          return f"Error: no existe la tarea {tarea_id}. Verificá el id."
      # un error NO recuperable (DB caída, permiso denegado) lo dejás propagar
      # para abortar el grafo, no lo escondas en un string

  # alternativa: que el nodo capture y reinyecte el error como tool_result
  # ToolNode([cerrar_tarea], handle_tool_errors=True)  # verificá la firma de tu versión
  ```

  La regla: un error que el modelo puede sortear cambiando de plan vuelve como `tool_result` de error; uno que no (sin permisos, infra caída) corta el grafo.
- **Validación de argumentos**: los type hints + Pydantic de la tool validan la *forma* (que `tarea_id` sea int), pero **no la autorización** (que este usuario pueda cerrar esa tarea). La validación de negocio/permisos va en el cuerpo de la tool o en un nodo previo —nunca confíes en que el modelo "pida bien"—.
- **Human-in-the-loop**: el `interrupt`/`interrupt_before` de LangGraph (módulo 5) es el gating de acciones irreversibles, ya como primitiva. Las acciones difíciles de revertir (borrar, cobrar, mandar, deployar) pasan por una pausa de confirmación.
- **Prompt injection, peor en un agente**: en [RAG](rag.md) una inyección sesga una respuesta; en un agente con tools puede hacer que **ejecute acciones**. Y en **multi-agente** el riesgo escala: una inyección en la salida de un agente entra como input a otro, propagándose por la conversación. Tratá toda salida externa (y la de otros agentes) como no confiable, separala de las instrucciones, y combiná con mínimo privilegio para acotar el blast radius.

El principio integrador no cambia, es el del archivo senior: **defensa en profundidad.** El framework te da más capas listas (recursion limit, interrupts, captura de errores), pero apilarlas y poner el criterio —qué gatear, qué validar, qué tool tiene qué poder— sigue siendo tu trabajo. Y un agente sin **trazas** (módulo 10) es indebuggeable: cuando algo salga mal —y va a salir—, necesitás reconstruir qué decidió y qué hizo. La frase mental: **el framework no te exime de la confiabilidad; te da las perillas (recursion limit, interrupts, captura de errores de tools) y vos seguís poniendo el criterio —validación de permisos, qué gatear, mínimo privilegio—, con el agravante de que en multi-agente la inyección se propaga entre agentes—.**

**Ejercicios 9**
9.1 ¿Cómo se expresan en LangGraph/AutoGen el tope de iteraciones y el human-in-the-loop de [Agentes](agentes.md) módulo 8?
9.2 Los type hints + Pydantic validan la forma de los argumentos de una tool. ¿Qué NO validan, y dónde va esa validación?
9.3 ¿Por qué el prompt injection escala en un sistema multi-agente? ¿Cómo lo acotás?
9.4 ¿Cuál es el riesgo de "el framework ya se encarga", y qué sigue siendo responsabilidad tuya?

---

## Módulo 10 — Observabilidad y evaluación de agentes (la trayectoria, no solo la respuesta)

**Teoría.** Un agente es un proceso **no determinista y de varios pasos**, y eso rompe dos cosas: es **difícil de debuggear** (¿por qué tomó ese camino?) y **difícil de evaluar** (¿anduvo bien, o tuvo suerte?). Las dos respuestas —observabilidad y evals— son lo que separa un demo de algo que mantenés en producción, y conectan con [evals](evals.md) y [observabilidad](observabilidad.md).

**Observabilidad / tracing.** Necesitás ver **la trayectoria completa**: cada llamada al modelo, cada tool call con sus argumentos y su resultado, cada decisión del router, el estado en cada paso, los tokens y la latencia. Herramientas como **LangSmith** (de LangChain, integra directo con LangGraph) o tracing con **OpenTelemetry** ([observabilidad](observabilidad.md)) capturan ese árbol de ejecución. Sin esto, cuando un agente hace algo raro estás a ciegas: un agente sin trazas es indebuggeable (el principio de [Agentes](agentes.md) módulo 8).

**Evaluación de agentes.** Acá está el salto respecto a evaluar un LLM de una sola llamada o un RAG ([RAG](rag.md) módulo 11): en un agente no alcanza con evaluar la **respuesta final**, porque el agente puede llegar a la respuesta correcta por un camino malo (caro, peligroso, o de pura suerte) o dar una respuesta pasable habiendo hecho algo indebido en el medio. Se evalúan **dos cosas**:

1. **El resultado final** (*outcome*): ¿la respuesta/acción es correcta? Como en cualquier eval —golden set, y LLM-as-judge para salida en texto libre ([evals](evals.md))—.
2. **La trayectoria** (*trajectory*): ¿el **camino** fue bueno? ¿Usó las tools correctas, en un orden razonable, sin pasos inútiles, sin acciones peligrosas? Se mide comparando la secuencia de tool calls contra una trayectoria esperada, o juzgándola con un LLM. Una respuesta correcta tras 30 tool calls innecesarias **falló** la eval de trayectoria aunque pase la de outcome.

Y las métricas **operacionales**, que en un agente son críticas porque varían con cada corrida: **costo por tarea** (un agente puede disparar decenas de llamadas), **latencia** (suma de pasos en serie), y **tasa de finalización** (¿cuántas tareas termina sin chocar el `recursion_limit` o quedar trabado?). La frase mental: **un agente se evalúa por el resultado Y por la trayectoria —llegar a la respuesta correcta por un camino malo es un fallo—, y se observa con tracing de cada paso (LangSmith/OTel); sin trazas no lo debuggeás y sin evaluar la trayectoria no sabés si anduvo o tuvo suerte—.**

**Ejercicios 10**
10.1 ¿Por qué un agente es difícil de debuggear, y qué necesitás capturar para poder hacerlo?
10.2 ¿Por qué evaluar solo la respuesta final de un agente no alcanza? ¿Qué es la evaluación de trayectoria?
10.3 Nombrá tres métricas operacionales de un agente y por qué importan especialmente en este caso.

---

## Módulo 11 — El criterio de cierre: framework vs a mano, y el puente del track

**Teoría.** El cierre, que ata todo con la decisión de arquitectura. Tres preguntas en orden, cada una un escalón que **no** subís por defecto:

**1. ¿Necesito un agente?** Antes que nada, el gate de cuatro preguntas de [Agentes](agentes.md) módulo 11 (complejidad, valor, viabilidad, costo del error) y la escalera llamada→workflow→agente. La mayoría de los "casos de agente" son workflows. Si la tarea tiene pasos especificables de antemano, **no es un agente** —y ningún framework cambia eso—.

**2. Si es un agente, ¿framework o a mano?**
- **A mano** (el loop de [Agentes](agentes.md) módulo 3, o el `tool runner` del SDK del proveedor): un agente simple, de pocos pasos, sin estado entre sesiones, dentro de un backend donde querés mínimas dependencias y control total. Menos magia, menos deps, más claro de debuggear.
- **Framework (LangGraph)**: cuando necesitás lo que el framework industrializa y a mano sería frágil —**checkpointing/durabilidad, memoria entre turnos, human-in-the-loop, time travel, o un grafo de control no trivial**—. Si tu agente necesita pausar para confirmación, reanudar tras una caída o recordar conversaciones, LangGraph paga su complejidad.
- La trampa, idéntica a la del framework layer de [Vector Databases](vector-dbs.md) módulo 11: meter LangGraph para un agente de 20 líneas que el loop a mano resolvía más claro.

**3. Si es un agente, ¿single o multi?** El criterio del módulo 7: **single por defecto**; multi solo con especialización fuerte, paralelismo real o separación de contextos que puedas nombrar. Y dentro de multi: supervisor (LangGraph, control explícito) antes que group chat (AutoGen, emergente), salvo que el problema sea genuinamente conversacional.

El principio que atraviesa **todo** el track, una última vez: **la mejor solución es la más simple que resuelve el problema real, y la decisión se mide, no se adivina.** Para el LLM era "¿modelo o `if`?"; para RAG, "¿buscar en muchos docs o trabajar con pocos?"; para vector DBs, "¿base dedicada o pgvector?"; para agentes, **"¿que el modelo decida el camino, o lo escribo yo? y si decide, ¿con qué andamiaje y cuántos agentes?"**.

**El puente del track Python AI.** Con esto tenés el agente —el componente que ata prompting, retrieval y tools en un sistema que decide y actúa—. Lo que sigue: servirlo por API ([FastAPI](fastapi.md): el agente detrás de un endpoint con streaming), [Voice AI](voice-ai.md) (agentes que escuchan y hablan), [Deploy de IA](deploy-ai.md) (correrlo en producción: GPU, escalado, costo), y atravesando todo, [evals](evals.md), que convierte cada decisión de este módulo en un número que podés defender. La frase mental: **decidí en tres escalones —¿agente?, ¿framework o a mano?, ¿single o multi?— subiendo solo cuando el de abajo te queda corto y podés nombrar por qué; es el mismo criterio de simplicidad medida que recorre el track entero, ahora aplicado a la pieza más poderosa y más fácil de sobre-diseñar—.**

**Ejercicios 11**
11.1 Recorré los tres escalones de decisión (¿agente?, ¿framework o a mano?, ¿single o multi?) y qué justifica subir cada uno.
11.2 ¿Cuándo escribirías el loop a mano y cuándo usarías LangGraph? Dá el factor decisivo.
11.3 ¿Cuál es la trampa compartida con el framework layer de RAG ([Vector Databases](vector-dbs.md) módulo 11)?
11.4 Enunciá la pregunta-criterio de agentes y mostrá cómo es la misma lógica de simplicidad que la de LLMs, RAG y vector DBs del track.

---

## Proyecto integrador (capstone): un agente de soporte sobre tus apuntes, con LangGraph

El ejercicio que cierra el módulo y que mostrás en una entrevista de AI Engineer cuando te piden "¿armaste un agente?". Es la evolución del capstone de [Vector Databases](vector-dbs.md): aquel **buscaba**; este **decide cuándo buscar y actúa**. No te damos la solución; te damos los **criterios de aceptación**. Construilo en Python con LangGraph.

**Qué construir.** Un agente de consola que responde preguntas sobre los apuntes de este sitio y puede ejecutar acciones acotadas:

1. Un **grafo agente↔tools** (módulo 4) con, al menos, dos tools: `buscar_en_kb(consulta)` —que corre el retrieval del capstone de [Vector Databases](vector-dbs.md), filtrado por `tenant_id`— y una tool de **acción** acotada (ej. `crear_nota(texto)` que escribe en un archivo).
2. **Checkpointing** con `thread_id` (módulo 5) para que recuerde la conversación entre turnos.
3. **Human-in-the-loop**: la tool de acción (`crear_nota`) pasa por una **pausa de confirmación** (`interrupt_before`) antes de ejecutarse (módulo 5 y 9).
4. Un **`recursion_limit`** como tope (módulos 4 y 9), y manejo del error de una tool sin tumbar el grafo.

**Criterios de aceptación.**
- [ ] El grafo cicla agente↔tools y termina solo cuando el modelo responde sin pedir tools.
- [ ] El agente **decide** cuándo buscar: para "hola" no busca; para una pregunta sobre los apuntes, busca (agentic retrieval).
- [ ] `buscar_en_kb` filtra por `tenant_id` y el grounding está en el system prompt (responde solo con lo recuperado, cita la fuente).
- [ ] La acción `crear_nota` **se interrumpe y pide confirmación** antes de ejecutar; reanuda con la decisión humana.
- [ ] Con el mismo `thread_id`, una segunda pregunta usa el contexto de la primera (memoria).
- [ ] El agente respeta el `recursion_limit` (no cicla infinito) y si una tool falla, se adapta en vez de morir.

**Extensiones (suben el nivel).**
- Agregá tracing con **LangSmith** (o logs estructurados) y mostrá la **trayectoria** completa de una corrida (módulo 10).
- Evaluá: armá 5 tareas con su trayectoria esperada (qué tools deberían usarse) y medí **outcome + trayectoria** (módulo 10).
- Convertilo en **supervisor + workers** (módulo 7): un supervisor que enruta entre un agente "buscador" y uno "redactor". Notá el costo de coordinación.
- Exponé el agente detrás de un endpoint **[FastAPI](fastapi.md)** con streaming.

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 (Cuatro de) la gestión del estado, la persistencia/checkpointing (reanudar tras caída,
    recordar entre turnos), el human-in-the-loop (pausar/reanudar), la coordinación multi-agente,
    y la observabilidad/tracing. El loop a mano solo te da el ciclo pedir→ejecutar→reinyectar.
1.2 Porque el framework automatiza el ANDAMIAJE, no el criterio: si no entendés el loop por
    debajo, tratás al framework como magia y cuando el agente cicla, alucina o gasta de más no
    tenés con qué razonarlo. Es leverage si entendés qué automatiza; pasivo si es lo único que sabés.
1.3 El mismo criterio: un framework (de RAG o de agentes) es leverage cuando entendés lo que
    automatiza y un pasivo cuando es lo único que sabés; por eso primero la implementación a mano
    (chunking/retrieval, o el loop) y después la capa que lo industrializa.
```

### Módulo 2
```
2.1 LangGraph: el agente como grafo de estado que vos diseñás y el modelo recorre (control
    explícito). AutoGen: una sociedad de agentes que conversan hasta converger (coordinación
    emergente).
2.2 El eje control explícito vs coordinación emergente. LangGraph en el extremo explícito (vos
    cableás nodos, edges y ciclos); AutoGen en el emergente (el resultado surge del diálogo entre
    agentes).
2.3 LangGraph para un agente robusto de producción con estado/durabilidad/HITL y control de flujo
    fino (incluido supervisor multi-agente). AutoGen/CrewAI para experimentar con varios agentes
    que colaboran de forma conversacional y arrancar rápido un multi-agente.
```

### Módulo 3
```
3.1 (1) El estado: un objeto tipado que viaja por el grafo (la memoria de trabajo). (2) Los nodos:
    funciones (estado)->actualización parcial (llamar al LLM, ejecutar tools, validar). (3) Los
    edges: las transiciones, normales (A→B siempre) o condicionales (una función decide según el
    estado), con START y END como entrada/salida.
3.2 Porque LangGraph aplica las actualizaciones al estado vía reducers (módulo 4): el nodo devuelve
    qué cambió y el framework lo mergea, lo que permite checkpointing y ejecución determinista. El
    estado se corresponde con el array `messages` que acumulabas a mano en el loop de Agentes.
3.3 Un edge normal va siempre del nodo A al B. Un edge condicional ejecuta una función que mira el
    estado y elige a qué nodo ir (ramificación), p. ej. "si hay tool_calls → tools, si no → END".
3.4 El control de flujo se vuelve explícito e inspeccionable: ves los pasos, ramas y ciclos, y eso
    permite visualizar, versionar, testear y —clave— persistir/reanudar el grafo, cosa que un while
    con ifs enredados no da.
```

### Módulo 4
```
4.1 Nodos: "agente" (llama al LLM con tools bindeadas) y "tools" (ejecuta las pedidas). Edge
    condicional desde "agente": si el último mensaje trae tool_calls → "tools", si no → END. El
    ciclo es el edge normal "tools → agente": tras ejecutar, vuelve al agente a observar y decidir.
4.2 ToolNode ejecuta las tools que el modelo pidió (te ahorra el "para cada tool_call, ejecutá y
    armá el tool_result"). tools_condition es la función de routing que detecta si el último mensaje
    trae tool calls. Juntos te ahorran el cuerpo del loop a mano.
4.3 El recursion_limit es el máximo de pasos del grafo; si se supera, LangGraph corta con error. Se
    corresponde con el tope de iteraciones de Agentes módulo 8 (evita loops infinitos y quemar tokens).
4.4 create_react_agent te arma el grafo agente↔tools prehecho. El valor real de LangGraph aparece
    cuando NO lo usás: cuando abrís el ciclo para insertar nodos propios (validación, gate humano,
    router a un subgrafo) que un loop cerrado no te deja meter.
```

### Módulo 5
```
5.1 El checkpointing persiste el estado del grafo tras cada paso (con un checkpointer). El thread_id
    identifica una conversación/ejecución: el estado se guarda y se recupera por ese id.
5.2 (1) Memoria entre turnos (mismo thread_id recuerda sin reenviar historial). (2) Durabilidad
    (reanudar tras una caída). (3) Human-in-the-loop (pausar, inspeccionar, reanudar). (4) Time
    travel (volver a un checkpoint anterior y reanudar por otro camino).
5.3 Compilando el grafo con interrupt_before=["tools"] (o usando interrupt() en un nodo): el grafo
    pausa antes del nodo sensible, inspeccionás el estado/pedís confirmación, y reanudás con
    invoke(None, config) (o Command(resume=...)). Es el gating de acciones irreversibles de Agentes
    módulo 8, vuelto primitiva.
5.4 Porque replicar a mano la serialización del estado, su reconstrucción, la reanudación parcial y
    la concurrencia por thread es mucho trabajo y fácil de hacer mal; el checkpointer te lo da como
    infraestructura probada. Es (con el multi-agente) la razón más fuerte para usar el framework.
```

### Módulo 6
```
6.1 LangGraph cablea un grafo explícito que vos diseñás; AutoGen modela agentes que CONVERSAN entre
    sí y el resultado emerge del diálogo (coordinación emergente, no flujo prediseñado).
6.2 AssistantAgent: agente con LLM y un rol. UserProxyAgent (proxy/ejecutor): representa al humano
    y/o ejecuta código (la pieza que actúa). Group chat: varios agentes en una conversación con un
    manager que decide quién habla. Condición de terminación: cuándo para (mensaje TERMINATE, máximo
    de turnos, meta cumplida).
6.3 Con evaluator-optimizer (escribir → criticar → reescribir). El cambio: en vez de cablear vos el
    loop generar/evaluar/iterar, emerge de la conversación entre el agente escritor y el crítico.
6.4 Porque la coordinación emergente puede no converger (agentes que se repiten o quedan dando
    vueltas): la condición de terminación/tope evita que la conversación sea infinita. Cuidado de
    versión: v0.2 (síncrona, ConversableAgent/GroupChat) y v0.4 (reescritura async event-driven,
    autogen-core/autogen-agentchat) NO son compatibles; fijate de qué versión es el tutorial.
```

### Módulo 7
```
7.1 Agrega coordinación, costo, latencia y no determinismo por encima de un agente único.
    Recomendación: empezar por un agente único bien diseñado con buenas tools y subir a multi-agente
    solo cuando ese único genuinamente no alcanza.
7.2 (1) Especialización fuerte (subtareas con prompts/tools/conocimiento muy distintos). (2)
    Paralelismo real (subtareas independientes que corren a la vez). (3) Separación de
    responsabilidades/contextos (aislar contextos o dividir un problema demasiado grande para una
    ventana).
7.3 Un agente supervisor recibe la tarea, la descompone y enruta a workers especializados, después
    sintetiza. Es el más controlable porque el routing es explícito (no emergente). En LangGraph se
    modela como un nodo router con edges condicionales hacia cada worker.
7.4 LangGraph para supervisor/jerárquico con control explícito (vos diseñás el routing);
    AutoGen/CrewAI para group chat / equipos con rol donde la coordinación es conversacional y
    emergente.
```

### Módulo 8
```
8.1 Del nombre de la función, su docstring (la descripción que lee el modelo) y sus type hints (que
    arman el schema). El docstring no es opcional porque ES el prompt que decide si el modelo usa la
    tool y cuándo: una descripción pobre = decisiones malas.
8.2 El retrieval se vuelve una tool (buscar_en_kb) que el agente decide usar (agentic retrieval): por
    dentro corre la búsqueda vectorial del track, y el agente elige cuándo/qué buscar y si refina.
    Más críticas: el filtrado por permisos (un agente con búsqueda sin filtro puede filtrar data entre
    tenants) y el grounding (responder solo con lo recuperado, citar fuente).
8.3 MCP trae tools externas estándar (SDK de Python + adaptadores) sin escribir un adaptador por
    servicio. No cambia el modelo mental: el agente sigue pidiendo tools y observando resultados;
    cambia de dónde vienen (un protocolo estándar), y siguen aplicando mínimo privilegio, salida no
    confiable y cuidado de credenciales.
```

### Módulo 9
```
9.1 Tope de iteraciones: el recursion_limit de LangGraph / la condición de terminación de AutoGen.
    Human-in-the-loop: el interrupt/interrupt_before de LangGraph (pausar antes de un nodo sensible,
    confirmar, reanudar).
9.2 Validan la FORMA (que tarea_id sea int, etc.), no la AUTORIZACIÓN (que este usuario pueda cerrar
    esa tarea) ni la lógica de negocio. Esa validación va en el cuerpo de la tool o en un nodo previo;
    nunca confiar en que el modelo "pida bien".
9.3 Porque la salida de un agente entra como input a otro: una inyección en un agente se propaga por
    la conversación al resto. Se acota tratando toda salida (externa y de otros agentes) como no
    confiable, separándola de las instrucciones, y con mínimo privilegio para limitar el blast radius.
9.4 El riesgo es creer que "el framework ya se encarga": automatiza el andamiaje (perillas: recursion
    limit, interrupts, captura de errores), no el criterio. Sigue siendo tuyo: validar permisos, qué
    acciones gatear, qué poder tiene cada tool (mínimo privilegio) y apilar las capas (defensa en
    profundidad).
```

### Módulo 10
```
10.1 Porque es no determinista y de varios pasos: ¿por qué tomó ese camino? Para debuggearlo necesitás
     capturar la trayectoria completa: cada llamada al modelo, cada tool call con args y resultado,
     cada decisión de router, el estado por paso, tokens y latencia (LangSmith / OpenTelemetry).
10.2 Porque el agente puede llegar a la respuesta correcta por un camino malo (caro, peligroso, de
     suerte) o dar una respuesta pasable habiendo hecho algo indebido. La evaluación de trayectoria
     mide si el CAMINO fue bueno: tools correctas, orden razonable, sin pasos inútiles ni acciones
     peligrosas (vs una trayectoria esperada o juzgada por un LLM).
10.3 Costo por tarea (un agente dispara muchas llamadas), latencia (suma de pasos en serie) y tasa de
     finalización (cuántas tareas termina sin chocar el recursion_limit o trabarse). Importan porque
     varían con cada corrida y definen si el agente es viable en producción.
```

### Módulo 11
```
11.1 (1) ¿Agente? Gate de 4 preguntas + escalera llamada→workflow→agente; si los pasos son
     especificables, es workflow, no agente. (2) ¿Framework o a mano? A mano si es simple y sin
     estado; framework si necesitás checkpointing/HITL/durabilidad/grafo no trivial. (3) ¿Single o
     multi? Single por defecto; multi solo con especialización/paralelismo/separación de contextos.
     Cada escalón se sube solo cuando el de abajo queda corto.
11.2 A mano: agente simple, pocos pasos, sin estado entre sesiones, dentro de un backend con mínimas
     deps y control total. LangGraph: cuando necesitás checkpointing, memoria entre turnos,
     human-in-the-loop, time travel o un grafo de control no trivial. Factor decisivo: si necesitás
     persistencia/durabilidad/HITL, paga el framework; si no, el loop a mano es más claro.
11.3 Meter el framework pesado (LangGraph) para un agente trivial que el loop a mano resolvía más
     claro — la misma trampa que usar LangChain/LlamaIndex para un RAG de 50 líneas.
11.4 "¿Que el modelo decida el camino, o lo escribo yo? y si decide, ¿con qué andamiaje y cuántos
     agentes?". Es la misma lógica de simplicidad medida del track: LLM ("¿modelo o if?"), RAG
     ("¿muchos docs o pocos?"), vector DBs ("¿dedicada o pgvector?"), agentes ("¿autonomía o
     control? ¿uno o varios?"). Siempre: la solución más simple que resuelve el problema real, medida.
```

---

## Siguientes pasos

Con este módulo tenés la capa de frameworks de agentes en Python: por qué existe (industrializar el loop a mano), el panorama (LangGraph / AutoGen / CrewAI / SDKs de proveedores) ordenado por control explícito vs emergente, LangGraph a fondo (el agente como grafo de estado, el loop como ciclo, checkpointing/memoria/human-in-the-loop), AutoGen y el paradigma multi-agente por diálogo, los patrones multi-agente (supervisor/group chat) con el criterio de cuándo NO usarlos, cómo enchufar tools/RAG/MCP, la confiabilidad y los guardarraíles dentro de un framework, y la observabilidad y evaluación de agentes (resultado **y** trayectoria). Todo con la regla del track: **la solución más simple que resuelve el problema real, medida y no adivinada.** Lo que sigue en el track Python AI: [Voice AI](voice-ai.md) (agentes que escuchan y hablan: STT/TTS, pipelines de voz, tiempo real) y [Deploy de aplicaciones de IA](deploy-ai.md) (correr todo esto en producción: GPU, escalado, costo). Y atravesando todo, [evals](evals.md) —lo que convierte cada decisión de este módulo en un número que podés defender—, y [FastAPI](fastapi.md) para servir el agente por API. El hilo del track Python AI queda casi completo: aprendiste el lenguaje, a servir APIs, a hablarle al modelo, a darle tu información a escala, y ahora a **dejarlo actuar con criterio**.
