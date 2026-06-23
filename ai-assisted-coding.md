# 🤖 Ingeniería de software asistida por IA

Usar IA para construir software dejó de ser un truco y pasó a ser una **habilidad de ingeniería en sí misma** — una que en 2026 es prácticamente requisito. Este módulo es el "10x coding with AI": no se trata de que la IA escriba código por vos, sino de que vos **dirijas** a un agente de coding con criterio.

Es meta: estás usando exactamente esta clase de herramienta para construir este temario. Y conecta con casi todo el track — un agente de coding **es un agente** ([agentes.md](agentes.md), [ai-agents-python.md](ai-agents-python.md)), se le habla con **prompts** ([prompt-engineering.md](prompt-engineering.md)), su código hay que **testearlo** ([testing.md](testing.md), [tdd.md](tdd.md)) y **revisarlo** con responsabilidad ([liderazgo.md](liderazgo.md)), y tiene su propia **superficie de seguridad** ([seguridad-ia.md](seguridad-ia.md)).

> **Aviso de honestidad:** no es magia. La IA **amplifica lo que ya sos**: un buen ingeniero produce más y mejor; uno flojo produce deuda más rápido. La habilidad que enseña este módulo es *dirigir y verificar*, no *delegar y olvidar*.

> **Formato:** teoría → ejercicios → **soluciones al final**. Menos código, más criterio y workflow (es un módulo de meta-habilidad). Español.

---

## Módulo 1 — De autocompletar a delegar (y el cambio de rol)

La IA en el código vive en un espectro de autonomía creciente:

1. **Autocompletado** (Copilot inline): completa la línea o el bloque que estás escribiendo. Vos seguís escribiendo cada decisión.
2. **Chat en el IDE**: le preguntás, te explica, te genera un snippet que copiás. Conversacional, pero vos integrás.
3. **Agente de coding** (Claude Code, Cursor agent, etc.): le das un **objetivo** y el agente lee archivos, escribe, corre comandos/tests, observa el resultado e itera **solo** hasta terminar.

El salto importante es del 2 al 3: tu rol cambia de **escribir cada línea** a **especificar, revisar e integrar**. Sos menos "tipeador" y más "tech lead de un junior muy rápido y muy literal". Esa es la habilidad nueva — y por qué un mal uso (aceptar todo sin mirar) es peligroso, no solo improductivo.

Por qué es table-stakes en 2026: la velocidad de generación se volvió commodity. Lo que diferencia a un ingeniero ya no es cuánto código escribe, sino **cuán bien dirige la generación y cuán bien verifica el resultado**.

---

## Módulo 2 — El espectro de herramientas y dónde encaja cada una

| Herramienta | Cuándo |
|---|---|
| **Autocomplete** (Copilot, etc.) | Código repetitivo, boilerplate, mientras ya sabés qué escribir. Bajo riesgo. |
| **Chat / inline edit** | Explicar código ajeno, refactors acotados, generar un snippet puntual. |
| **Agente de coding** (Claude Code, Cursor) | Tareas de varios pasos: implementar una feature, arreglar un bug que cruza archivos, escribir tests, migrar. |
| **Claude Agent SDK** | Embeber el harness de un agente de coding **en tu propia app** (CI que arregla código, pipelines). Ver Módulo 10. |

Un punto clave (puente con [ai-agents-python.md](ai-agents-python.md)): **Claude Code es el producto donde un humano dirige el loop interactivamente; el Claude Agent SDK es el mismo motor extraído para que tu *aplicación* dirija el agente.** Mismo loop, distinto conductor.

Regla de criterio: **escalá la herramienta a la tarea.** Boilerplate → autocomplete. Tarea acotada de varios pasos → agente con review. Decisión arquitectónica crítica → vos pensás, la IA es un par para explorar, no para decidir.

---

## Módulo 3 — Cómo funciona un agente de coding por dentro

Es el **agent loop** que ya viste, aplicado a un repositorio. Si entendiste [agentes.md](agentes.md), ya entendés esto:

```
objetivo → el modelo decide una acción (leer archivo / editar / correr comando)
         → tu entorno la ejecuta → el resultado vuelve al contexto
         → repetir hasta que la tarea esté hecha o se cumpla una condición de parada
```

Las **tools** típicas de un agente de coding: leer archivos, editar/escribir, ejecutar comandos (correr tests, linters, builds), buscar en el repo (grep/glob). El modelo no "ve" tu repo entero de una: **lo explora con esas tools**, como lo harías vos.

Esto importa porque explica los aciertos y los fallos del agente: cuando corre los tests y ve que fallan, **se corrige solo** (el loop observa el error). Cuando alucina la firma de una función, es porque no leyó el archivo correcto antes de editar. Darle las tools y el contexto para que **verifique su propio trabajo** (correr tests, leer antes de editar) es lo que lo vuelve confiable.

---

## Módulo 4 — Context engineering para código

El cuello de botella de un agente no es qué tan inteligente es el modelo, sino **qué tiene en su contexto**. Dos verdades:

- **El contexto es finito y se degrada** (*context rot*: a más tokens, peor recall). Volcarle todo el repo no ayuda — lo confunde.
- El patrón ganador es **just-in-time**: el agente trae el archivo o el fragmento que necesita **cuando lo necesita** (grep/read), en vez de precargar todo. Por eso un repo bien estructurado, con nombres claros y módulos chicos, es más fácil de operar para un agente (igual que para un humano).

Palancas concretas para darle buen contexto:

- **Archivos de instrucciones de proyecto** (ej. un `CLAUDE.md` en la raíz): convenciones, comandos para correr tests, estilo, qué no tocar. El agente lo lee y respeta. Es la forma más barata de subir la calidad de todo lo que genera.
- **Señalar los archivos relevantes** explícitamente cuando la tarea es acotada ("mirá `auth.service.ts` y seguí ese patrón").
- **No mezclar tareas no relacionadas** en una misma sesión larga: el contexto se ensucia.

```markdown
<!-- Ejemplo de CLAUDE.md: convenciones que el agente respeta -->
# Proyecto X
- Tests: `npm test`. Correr SIEMPRE antes de dar por terminada una tarea.
- Estilo: funcional, sin clases nuevas salvo en /domain.
- No tocar /legacy sin avisar.
- Commits en español, imperativo, sin trailers de coautoría.
```

---

## Módulo 5 — Spec-driven / plan-first development

El error #1 al dirigir un agente: tirarle una tarea grande y vaga ("hacé el sistema de pagos") y esperar que adivine. Falla por la misma razón que falla con un humano: **las decisiones implícitas se resuelven mal.**

El patrón que funciona, en orden:

1. **Especificá el *qué*** antes de pedir el *cómo*: qué entra, qué sale, casos borde, restricciones. Cuanto más claro el contrato, mejor el resultado (esto es prompt engineering aplicado, [prompt-engineering.md](prompt-engineering.md)).
2. **Pedí un plan primero**, revisalo, y *después* dejá que ejecute. Un agente que primero propone "voy a tocar estos 4 archivos así" te deja corregir el rumbo **antes** de que escriba 300 líneas en la dirección equivocada.
3. **Baby steps**: tareas chicas y verificables encadenadas le ganan a una tarea gigante. Cada paso con su test (puente con [tdd.md](tdd.md): el ciclo chico también disciplina al agente).

> El reflejo a internalizar: **revisar un plan de 5 líneas es barato; revisar 300 líneas de código en la dirección equivocada es caro.** Mové la corrección hacia adelante.

---

## Módulo 6 — Prompting para código

Las mismas técnicas de [prompt-engineering.md](prompt-engineering.md), aplicadas:

- **Instrucciones específicas** > vagas. "Agregá validación de email con la lib `zod` que ya usamos, devolviendo 400 con el mensaje de error" > "validá el input".
- **Ejemplos del codebase**: "seguí el patrón de `users.controller.ts`". El agente imita estilo bien si le mostrás el estilo.
- **Restricciones explícitas**: qué no tocar, qué librería usar, qué convención seguir. Lo que no decís, lo decide el modelo (y puede meter una dependencia nueva que no querías).
- **Pedile que razone/planifique** en tareas complejas antes de codear (los modelos de razonamiento lo hacen mejor si se lo pedís).

---

## Módulo 7 — Revisar código generado por IA

Acá está el centro de gravedad del rol nuevo. Cuando la generación es barata, **el cuello de botella se mueve al review** (esto ya lo anticipaba [liderazgo.md](liderazgo.md): "la IA en el rol").

Principios:

- **El que mergea es responsable.** No importa que lo haya escrito un agente: si lo aprobás, es tuyo. Esta regla, sola, evita la mayoría de los desastres.
- **Entendé antes de mergear.** Si no podés explicar qué hace el código, no estás en condiciones de aprobarlo. "Funciona en la demo" no es entender.
- **Buscá los fallos típicos de la IA**: APIs alucinadas (métodos que no existen), manejo de errores ausente, casos borde ignorados, código que pasa los tests felices pero no los adversos, sobre-ingeniería (te resuelve un problema que no tenías).
- **Cuidado con el *rubber-stamping***: aprobar en automático porque "se ve bien" y son muchos diffs. Es el modo de fallo más común y el más peligroso, porque escala.

El review de código asistido por IA **es** un trabajo de liderazgo técnico: revisás más volumen, así que tu criterio sobre qué mirar primero (seguridad, límites, contratos) importa más que nunca.

---

## Módulo 8 — Testing y evals del código generado (ángulo AI QA)

Si generás más rápido, necesitás verificar más rápido — y de forma **automática**, no a ojo.

- **TDD con IA**: escribir (o pedir) los tests **primero** le da al agente un objetivo verificable y un criterio de "terminado". El loop del agente se vuelve mucho más confiable cuando puede correr los tests y ver rojo→verde (puente con [tdd.md](tdd.md)).
- **Corré los tests siempre**, en el loop del agente y en CI. "Lo que no corre automáticamente no existe" ([api-testing.md](api-testing.md)).
- **Métrica de código generado**: *functional correctness* / **pass@k** — ¿el código pasa los tests? Es la métrica honesta para código (a diferencia de tareas subjetivas que necesitan LLM-judge, [evals.md](evals.md)).
- **El ángulo AI QA**: testear features construidas con asistencia de IA es tu terreno. Y un paso más: si tu producto *genera* código o artefactos con IA, necesitás **evals** de esa generación — un golden set de tareas con su criterio de correctitud, corriendo en CI como quality gate.

---

## Módulo 9 — Seguridad del código asistido por IA

El código generado **hereda los riesgos de su origen probabilístico**. Cuatro a vigilar (puente con [seguridad-ia.md](seguridad-ia.md)):

1. **Vulnerabilidades copiadas.** La IA aprende de código público, que incluye patrones inseguros. Puede generar SQL concatenado, validación ausente, secrets hardcodeados. **No asumas que es seguro porque funciona.**
2. **Dependencias alucinadas (*slopsquatting*).** El modelo puede sugerir importar un paquete que **no existe** — y un atacante puede registrar ese nombre con malware esperando que alguien lo instale. Verificá que las deps que sugiere son reales y mantenidas (riesgo de supply chain, LLM03 del OWASP LLM Top 10).
3. **Prompt injection en el agente de coding.** Si tu agente lee archivos de terceros (un issue, un README de una dependencia, un repo clonado), ese contenido puede contener instrucciones inyectadas. El caso real **MCPoison (CVE-2025-54136)** en Cursor mostró cómo una config aprobada se modificaba después para ejecutar código.
4. **Secrets y datos.** Cuidado con qué le das de contexto al agente (no le pegues secretos de producción) y qué manda a un servicio externo.

El reflejo: **el código de la IA pasa por el mismo (o más) escrutinio de seguridad que el tuyo**, no menos.

---

## Módulo 10 — El agente de coding como producto

Hasta acá usaste el agente *como herramienta*. El siguiente nivel: **embeberlo en tu propio software** con el **Claude Agent SDK** (el harness de Claude Code expuesto como librería).

Casos donde esto habilita lo que el CLI interactivo no puede:
- **CI que arregla código**: un pipeline que lintea, detecta un fallo y le pide al agente que lo corrija, con tests como gate.
- **Pipelines de migración** a escala (cientos de archivos, mismo patrón).
- **Flujos con aprobación custom** que bloquean operaciones peligrosas antes de ejecutarlas.
- Cualquier flujo que necesite **output estructurado** del agente en vez de una conversación.

Esto es, esencialmente, el proyecto "PromptLab" del tipo de curso que inspiró este módulo: una plataforma para diseñar, versionar, testear y optimizar el uso del modelo a escala. Cuándo construir esto vs usar Claude Code directo: **construí cuando la app dirige el agente** (sin humano en cada paso); **usá el CLI cuando vos dirigís**.

---

## Módulo 11 — Criterio de cierre

La idea que ordena todo: **la IA amplifica lo que hay.**

- Un buen ingeniero con IA = más rápido y con más alcance. Un ingeniero flojo con IA = **deuda técnica acelerada**. La herramienta no suple el criterio; lo multiplica (en cualquier dirección).
- **El cuello de botella se movió al review.** El que mergea responde. No delegues el *entendimiento*.
- **Escalá el uso a la tarea**: autocomplete para boilerplate, agente + review para tareas acotadas, vos al volante para lo crítico.
- **Verificá automáticamente**: tests en el loop y en CI; evals si tu producto genera artefactos con IA.
- **Tratá el código generado como código de un junior brillante pero literal**: rápido, capaz, sin contexto de negocio, capaz de alucinar con seguridad. Revisalo como tal.

Esto conecta con el resto del temario más de lo que parece: es el [agente](ai-agents-python.md) que ya estudiaste, dirigido con [prompts](prompt-engineering.md), verificado con [tests/TDD](tdd.md) y [evals](evals.md), revisado con responsabilidad de [Tech Lead](liderazgo.md) (DORA: la velocidad sin estabilidad no es velocidad), y con su propia [superficie de seguridad](seguridad-ia.md). Es, quizá, la habilidad de mayor leverage del 2026 — precisamente porque hace palanca sobre todas las demás.

---

## Ejercicios

### Ejercicio 1 — Elegí la herramienta
Para cada tarea, decidí autocomplete / chat / agente de coding / Agent SDK embebido, y por qué: a) escribir 12 getters y setters de un DTO; b) implementar un endpoint nuevo con su validación y tests, tocando 3 archivos; c) entender qué hace un módulo legacy que no escribiste; d) un bot de CI que, ante un test que falla, intente arreglarlo automáticamente y abra un PR.

### Ejercicio 2 — Por qué falla el prompt gigante
Un compañero le pide al agente "construí todo el sistema de autenticación con JWT, refresh tokens, RBAC y recuperación de contraseña" en un solo mensaje, y el resultado es un desastre inconsistente. Explicá por qué falló y reescribí el enfoque siguiendo el Módulo 5.

### Ejercicio 3 — Review de código generado
El agente te entrega una función que pasa todos los tests existentes. Listá **cinco** cosas que mirarías antes de mergear, más allá de "los tests pasan".

### Ejercicio 4 — La dependencia que no existe
El agente sugiere `import { sanitize } from "secure-input-validator"` y te dice que instales `secure-input-validator`. ¿Qué riesgo concreto hay y cómo lo verificás antes de instalar?

### Ejercicio 5 — Amplificación
Explicá, con un ejemplo, la frase "la IA amplifica lo que hay" en el caso de (a) un equipo con buena disciplina de tests y review, y (b) un equipo sin tests que mergea rápido.

### Ejercicio 6 — Dirigir vs delegar
Tu jefe dice "ahora con IA no hace falta revisar tanto, total el agente ya corre los tests". Respondé con un argumento técnico (no de autoridad) de por qué eso es peligroso, usando conceptos de este módulo y de [seguridad-ia.md](seguridad-ia.md).

---

## Soluciones

### Solución 1
a) **Autocomplete**: boilerplate repetitivo, bajo riesgo, ya sabés qué va. b) **Agente de coding**: tarea de varios pasos que cruza archivos y necesita tests — con plan-first y review. c) **Chat / inline**: pedirle que explique el módulo (no necesita editar nada). d) **Agent SDK embebido**: la app (el CI) dirige al agente sin humano en cada paso, con output estructurado y tests como gate — exactamente el caso del Módulo 10.

### Solución 2
Falló porque es una tarea enorme, vaga y llena de **decisiones implícitas** (qué lib de JWT, cómo guardar refresh tokens, qué modelo de RBAC, formato de errores...) que el agente resolvió cada una a su manera, sin coherencia entre partes. Enfoque correcto: (1) especificar el contrato de cada pieza; (2) pedir un **plan** y revisarlo; (3) ejecutar en **baby steps** — primero login con access token + su test, luego refresh + su test, luego RBAC, luego recuperación — revisando cada paso antes del siguiente. Cada incremento verificable. (Y de paso, ya tenés el módulo [autenticacion.md](autenticacion.md) como spec de referencia.)

### Solución 3
Cinco más allá de "pasan los tests": (1) **¿los tests cubren los casos adversos/borde**, o solo el happy path? (2) **manejo de errores**: ¿qué pasa con input inválido, timeouts, nulos? (3) **seguridad**: ¿hay SQL concatenado, secrets, validación ausente, deps nuevas? (4) **APIs reales**: ¿los métodos/firmas que usa existen de verdad o están alucinados? (5) **encaje con el codebase**: ¿sigue las convenciones, o metió un patrón/librería ajeno? Bonus: ¿está sobre-ingenierizado para lo que se pedía?

### Solución 4
Riesgo: **dependencia alucinada / slopsquatting** (supply chain, OWASP LLM03). El paquete puede no existir — y un atacante puede haber registrado ese nombre exacto con código malicioso, esperando que alguien siga la sugerencia de la IA y lo instale. Verificación antes de instalar: buscá el paquete en el registro oficial (npm/PyPI), mirá descargas, mantenimiento, repo, último release y reputación. Si no existe o es sospechoso, no lo instales; resolvé con una lib ya conocida del proyecto.

### Solución 5
"La IA amplifica lo que hay": (a) **equipo con disciplina**: la IA genera rápido, los tests atrapan los errores, el review filtra, y la deuda se mantiene baja → ganan velocidad **con** estabilidad (DORA sube). (b) **equipo sin tests que mergea rápido**: la IA genera rápido el doble de código sin red de seguridad ni review → la deuda y los bugs se acumulan más rápido que antes. Misma herramienta, resultados opuestos, porque amplifica la práctica existente.

### Solución 6
Argumento técnico: que los tests pasen prueba *functional correctness sobre los casos que los tests cubren*, no que el código sea correcto, seguro o mantenible. El agente puede (1) pasar los tests felices ignorando casos adversos, (2) introducir vulnerabilidades que ningún test funcional detecta (SQL injection, secrets, deps alucinadas — [seguridad-ia.md](seguridad-ia.md)), (3) alucinar APIs que fallan en runtime fuera del path testeado, (4) meter deuda/sobre-ingeniería que no rompe tests pero degrada el sistema. Como **el que mergea es responsable**, saltarse el review traslada esos riesgos a producción sin filtro. El review no es burocracia: es la verificación que los tests no cubren. Reducir review por "el agente corre los tests" es confundir *pasar tests* con *ser correcto y seguro*.
