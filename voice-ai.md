# Voice AI: darle oídos y boca a un sistema de IA

**El pipeline de voz (STT → LLM → TTS), tiempo real y agentes de voz · stack Python, Claude como cerebro · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Es el sexto módulo del **track Python AI**: asume [Python](python.md), [Prompt Engineering](prompt-engineering.md), y sobre todo [AI Agents](ai-agents-python.md) y [Agentes](agentes.md) —porque un agente de voz **es** el agente que ya conocés, con un front de audio adelante y atrás—. La **IA de voz** es la capa que convierte un sistema de texto en uno con el que **hablás**: le transcribe lo que decís (oídos), razona (cerebro), y te contesta hablando (boca). El objetivo es doble: las **piezas** (reconocimiento de voz, síntesis, el problema de tiempo real, los turnos de habla) y el **criterio** (cuándo voz de verdad, arquitectura en cascada vs speech-to-speech, build vs framework) que separa un demo de juguete de un asistente de voz usable. Una aclaración de stack desde el arranque: **Anthropic/Claude es texto** —no provee STT ni TTS ni audio nativo—. Igual que en [RAG](rag.md) los embeddings venían de Voyage, acá los oídos y la boca vienen de **proveedores especializados** (Whisper, Deepgram, ElevenLabs, Cartesia…) y Claude pone el **cerebro**.

**Lo que asumimos.** Python (async es central acá), el agente de [AI Agents](ai-agents-python.md), WebSockets de [Tiempo real](tiempo-real.md), y servir con [FastAPI](fastapi.md).

**Para practicar.**

```bash
uv add faster-whisper anthropic
# framework de voz en tiempo real (opcional): uv add pipecat-ai
# los STT/TTS de proveedores (Deepgram, ElevenLabs, OpenAI Realtime…) requieren su SDK + API key
# export ANTHROPIC_API_KEY=...
```

> Nota sobre datos volátiles: el campo de voz se mueve rapidísimo —modelos de STT/TTS, APIs realtime (speech-to-speech), latencias y precios cambian seguido—. Los ejemplos muestran la **forma** de cada API a 2026; verificá modelos y firmas en la doc del proveedor. Que Anthropic no tenga audio nativo es lo vigente al escribir: confirmalo con la skill `claude-api` al implementar.

**Índice de módulos**
1. Qué es la IA de voz: el pipeline y dos arquitecturas (cascada vs speech-to-speech)
2. Fundamentos de audio que sí necesitás (sample rate, PCM, frames)
3. STT / ASR: reconocimiento de voz (Whisper, batch vs streaming, WER)
4. TTS: síntesis de voz (neural, streaming, SSML, latencia)
5. El presupuesto de latencia: por qué la voz es un problema de tiempo real
6. Turnos de habla: VAD, endpointing y barge-in (la parte difícil)
7. Speech-to-speech: los modelos de audio nativo en tiempo real
8. Transporte: WebSockets y WebRTC para audio bidireccional
9. El voice agent: el agente del track con oídos y boca (+ frameworks)
10. Evaluación y producción de sistemas de voz
11. El criterio de cierre: cascada vs S2S, build vs framework, y el puente del track

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es la IA de voz: el pipeline y dos arquitecturas

**Teoría.** Un sistema de voz conversacional —un *voice agent*— es, en su forma clásica, **tres piezas en un loop**:

```
🎤 vos hablás → [ STT: voz → texto ] → [ LLM: razona ] → [ TTS: texto → voz ] → 🔊 escuchás
                     oídos                  cerebro              boca
```

- **STT** (*Speech-to-Text*, también ASR): transcribe el audio del usuario a texto.
- **LLM**: el cerebro —prompting, tools, RAG, todo lo del track— opera sobre ese texto.
- **TTS** (*Text-to-Speech*): convierte la respuesta en audio.

Y se repite, turno a turno. Pero **cómo** armás esas piezas define dos arquitecturas que es la decisión central del módulo:

**1. Cascada (cascaded / pipeline).** Tres modelos distintos encadenados (STT → LLM → TTS), cada uno especializado y reemplazable. Ventajas: **modular** (cambiás de proveedor de STT sin tocar el resto), **el cerebro puede ser Claude** (que es texto), **debuggeable y logueable** (hay texto en el medio: ves qué se transcribió y qué se respondió), y maduro (tool use, RAG, structured output funcionan tal cual sobre el LLM). Desventajas: la **latencia se apila** (tres saltos en serie), se **pierde la información paralingüística** (tono, emoción, énfasis: el STT te da las palabras, no *cómo* se dijeron), y **los errores se encadenan** (si el STT transcribe mal, el LLM razona sobre basura).

**2. Speech-to-speech (S2S / end-to-end).** Un **único modelo multimodal** que recibe audio y produce audio directamente, sin pasar por texto (módulo 7). Ventajas: **latencia mucho menor** (un salto), **preserva y produce prosodia/emoción** (suena natural), y maneja **interrupciones** de forma nativa. Desventajas: **menos control y observabilidad** (no hay texto en el medio por defecto), integración de **tools/RAG más nueva**, y estás **atado al proveedor** que ofrece audio nativo —y como Claude es texto, S2S significa usar el modelo de audio de otro proveedor, no Claude—.

La frase mental: **un voice agent es STT→LLM→TTS en un loop; en cascada (tres modelos, modular, con Claude de cerebro, debuggeable, pero latencia apilada y sin prosodia) o speech-to-speech (un modelo de audio nativo, rápido y natural, pero menos control y sin Claude de cerebro) —y elegir entre las dos es la decisión de arquitectura del campo—.**

**Ejercicios 1**
1.1 Dibujá el pipeline de un voice agent y nombrá qué hace cada pieza (STT, LLM, TTS).
1.2 ¿Cuáles son las tres ventajas y las tres desventajas de la arquitectura en cascada?
1.3 ¿Por qué, si querés a Claude como cerebro, estás haciendo cascada y no speech-to-speech?
1.4 ¿Qué información se "pierde" en la cascada que el speech-to-speech preserva?

---

## Módulo 2 — Fundamentos de audio que sí necesitás

**Teoría.** No hace falta ser ingeniero de audio, pero unos pocos conceptos son la diferencia entre que tu pipeline ande o tire ruido. El audio es una onda continua (analógica) que se **digitaliza** muestreándola:

- **Sample rate (frecuencia de muestreo)**: cuántas muestras por segundo. **16 kHz es el estándar para voz** (Whisper y la mayoría de los STT esperan 16 kHz); **8 kHz** es telefonía (banda angosta, peor calidad; además suele venir codificada en **μ-law/G.711**, no PCM lineal —hay que decodificarla y resamplear a 16 kHz antes de pasarla al STT—); 44.1/48 kHz es música. El teorema de **Nyquist**: para capturar frecuencias hasta `f`, necesitás muestrear a `2f` —la voz humana vive bajo ~8 kHz, por eso 16 kHz alcanza—. *Gotcha clásico*: pasarle a un modelo audio a un sample rate distinto del que espera produce transcripciones basura; siempre **resampleás** a lo que el modelo pide.
- **Bit depth**: bits por muestra; **16-bit PCM** es lo típico para voz.
- **Canales**: para voz usás **mono** (un canal), no estéreo —menos datos, y no necesitás espacialidad—.
- **PCM**: las muestras crudas, sin comprimir. Los **contenedores/códecs** las empaquetan: **WAV** (PCM crudo, simple, pesado), **MP3** (comprimido), y **Opus** —el códec de **tiempo real** por excelencia: baja latencia, buena calidad a poco bitrate, el que usan WebRTC y los pipelines de voz en vivo—.
- **Frames / chunks**: en tiempo real no procesás un archivo entero; partís el audio en **chunks chiquitos** (ej. de 20 ms) y los procesás a medida que llegan. Esto es lo que permite la baja latencia (módulo 5) y el streaming (módulo 8).

Por qué importa para un AI Engineer: la mitad de los bugs de un pipeline de voz son de **formato** —sample rate equivocado, estéreo donde se esperaba mono, PCM crudo donde se esperaba WAV con header—. Saber esto te ahorra horas. La frase mental: **el audio se digitaliza muestreando (16 kHz mono 16-bit PCM es el default de voz), se transporta crudo (PCM/WAV) o comprimido (Opus para tiempo real), y se procesa en chunks chicos para streaming —y el error #1 del principiante es el sample rate mal—.**

**Ejercicios 2**
2.1 ¿Qué es el sample rate y cuál es el estándar para voz? ¿Qué dice Nyquist y por qué 16 kHz alcanza para la voz humana?
2.2 ¿Por qué para voz usás mono y no estéreo?
2.3 ¿Qué diferencia hay entre PCM, WAV y Opus, y cuál usarías para audio en tiempo real?
2.4 ¿Por qué se procesa el audio en chunks chicos en un sistema en vivo, y cuál es el bug de formato más común?

---

## Módulo 3 — STT / ASR: reconocimiento de voz (Whisper, batch vs streaming, WER)

**Teoría.** El **STT** (o **ASR**, *Automatic Speech Recognition*) son los oídos: audio → texto. El modelo abierto de referencia es **Whisper** (OpenAI, open-weight): multilingüe, robusto al ruido, con varios tamaños (`tiny` → `large`) que cambian el trade-off velocidad/precisión. Corre **local** (con `faster-whisper` o `whisper.cpp`, sin mandar audio a un tercero —importa para privacidad—) o vía API. Otros proveedores fuertes en STT en vivo: Deepgram, AssemblyAI.

```python
from faster_whisper import WhisperModel

modelo = WhisperModel("small", device="cpu", compute_type="int8")  # tiny/base/small/medium/large
segmentos, info = modelo.transcribe("pregunta.wav", language="es")
for s in segmentos:
    print(f"[{s.start:.1f}s–{s.end:.1f}s] {s.text}")
```

La distinción que define si sirve para un agente conversacional:

- **Batch (por lotes)**: le das un audio completo (o un chunk cerrado) y te devuelve la transcripción. Whisper es fundamentalmente batch. Perfecto para transcribir un archivo; **malo para una conversación en vivo**, porque esperás a que el usuario termine de hablar antes de tener una sola palabra.
- **Streaming**: el STT te devuelve **resultados parciales** (interim) a medida que la persona habla, y los va corrigiendo. Esto es lo que necesitás para baja latencia: empezás a procesar antes de que termine la frase. Los proveedores de streaming (Deepgram, etc.) lo dan; con Whisper "streaming" se simula procesando chunks solapados, con sus límites.

**Criterio práctico**: Whisper local cuando priorizás **batch, costo o privacidad** (el audio no sale de tu máquina); un STT de **streaming gestionado** (Deepgram, AssemblyAI) cuando la **latencia conversacional manda** y no querés pelear con chunks solapados a mano. No es "Whisper siempre"; es qué prioriza el caso.

La métrica de calidad es el **WER (Word Error Rate)**:

```
WER = (sustituciones + inserciones + eliminaciones) / palabras_de_referencia
```

Es la fracción de palabras que el STT erró respecto a una transcripción correcta (menor es mejor; 0 = perfecto). El WER se dispara con **acento, ruido de fondo, vocabulario de dominio** (nombres propios, términos técnicos, códigos) y **mala calidad de audio**. Palancas para bajarlo: dar el **hint de idioma**, **biasing** por palabras clave del dominio (muchos STT permiten sesgar hacia un vocabulario), y mejorar la captura (mejor micrófono, cancelación de ruido). Y recordá del módulo 1: un error de STT **se propaga** al LLM, que razona sobre lo mal transcripto —por eso el WER no es un detalle, es la base del pipeline—.

La frase mental: **el STT (Whisper el open-weight de referencia) transcribe voz a texto; batch para archivos, streaming con parciales para conversación en vivo; su calidad se mide con WER y se degrada con acento/ruido/jerga —y como sus errores se encadenan al LLM, es la base sobre la que se apoya todo el pipeline—.**

**Ejercicios 3**
3.1 ¿Qué es el STT/ASR y por qué Whisper es la referencia abierta? ¿Qué ventaja da correrlo local?
3.2 Diferenciá STT batch de streaming. ¿Por qué el batch puro es malo para un agente conversacional?
3.3 ¿Qué mide el WER y qué factores lo empeoran? Nombrá dos palancas para bajarlo.
3.4 ¿Por qué un WER alto es especialmente grave en un pipeline en cascada? (conectá con el módulo 1)

---

## Módulo 4 — TTS: síntesis de voz (neural, streaming, SSML, latencia)

**Teoría.** El **TTS** es la boca: texto → audio. La generación moderna es **neural** (modelos que producen voz natural), muy por encima de la voz robótica concatenativa de antes. Proveedores fuertes: ElevenLabs, Cartesia, OpenAI, Azure; abiertos como XTTS/Coqui. Las decisiones que importan:

- **Streaming TTS**: igual que el STT, podés **generar audio a medida que el texto llega**, sin esperar la respuesta completa del LLM. Es **clave para la latencia**: en cuanto el LLM emite la primera frase (o cláusula), el TTS ya empieza a hablar. La métrica acá es el **time-to-first-byte / first-audio** (cuánto tarda en salir el primer sonido), que pesa más en la percepción que el tiempo total.
- **SSML** (*Speech Synthesis Markup Language*): marcado para controlar la salida —pausas, énfasis, pronunciación, números/fechas, prosodia—. Ej.: `<break time="500ms"/>`, deletrear una sigla, decir "1/2" como "un medio". Sin SSML, el TTS adivina y a veces erra (lee "Dr." como "doctor" o "drive" según contexto).
- **Normalización de texto (text normalization)**: antes de sintetizar, el texto se *normaliza* a su forma hablada —números, fechas, montos, símbolos y siglas—: `$1.000` → "mil pesos", `Dr.` → "doctor", `12/3` → "doce de marzo", `API` → "a-pe-i" o "api" según convenga. La mayoría de los TTS lo hacen solos, pero erran con dominio/idioma/jerga; SSML ayuda a forzar casos puntuales pero **no siempre alcanza**, y la normalización mal hecha es una fuente real de errores *audibles* en producción (que no aparecen en el texto, solo al escucharlo).
- **Voice cloning / voces custom**: clonar una voz a partir de muestras. Potente y **delicado**: requiere **consentimiento** —clonar la voz de alguien sin permiso es un problema legal y ético, además del riesgo de deepfakes—.
- **Naturalidad y el valle inquietante**: la prosodia (entonación, ritmo) es lo que hace humana o robótica a una voz. Una voz casi-humana pero con prosodia rara cae en el *uncanny valley* y molesta más que una claramente sintética.

La frase mental: **el TTS neural convierte texto en voz natural; el streaming (empezar a hablar antes de tener toda la respuesta) es lo que lo hace usable en vivo —importa el time-to-first-audio, no el total—, SSML te da control fino sobre cómo se dice, y la clonación de voz exige consentimiento—.**

**Ejercicios 4**
4.1 ¿Qué es el streaming TTS y por qué es clave para la latencia? ¿Qué métrica importa más, el tiempo total o el time-to-first-audio?
4.2 ¿Para qué sirve SSML? Dá dos ejemplos de algo que controlás con él.
4.3 ¿Qué cuidado ético/legal tiene la clonación de voz?
4.4 ¿Qué es la prosodia y qué tiene que ver con el "valle inquietante"?
4.5 ¿Qué es la normalización de texto para TTS y por qué SSML no siempre alcanza? Dá un ejemplo de algo que se normaliza.

---

## Módulo 5 — El presupuesto de latencia: por qué la voz es un problema de tiempo real

**Teoría.** Acá está lo que hace difícil a la voz y fácil de subestimar: **es un problema de latencia, no solo de calidad.** En una conversación humana, el silencio entre que termino de hablar y empezás a responder es de ~200–500 ms. Si tu voice agent tarda 3 segundos, se siente roto —no importa cuán inteligente sea la respuesta—. El objetivo conversacional es responder en **menos de ~800 ms–1 s** de punta a punta.

Y el problema es que la latencia **se apila** a lo largo del pipeline (la desventaja de la cascada del módulo 1):

```
captura mic + endpointing (¿terminó de hablar?) + STT + LLM (time-to-first-token) + TTS (time-to-first-audio) + red + reproducción
```

Si cada etapa tarda "poco", la suma en serie te saca de presupuesto fácil. La técnica que salva todo es una sola idea: **streamear y pipelinear todo, no esperar a que cada etapa termine.**

- No esperes la transcripción completa para mandarle al LLM: usá **STT con parciales** (módulo 3).
- No esperes la respuesta completa del LLM para hablar: **streameá los tokens** y arrancá el TTS con la primera cláusula (módulos 4 y de [FastAPI](fastapi.md), streaming).
- **Solapá las etapas**: mientras el TTS dice la primera frase, el LLM ya está generando la segunda.

El insight clave de percepción: lo que el usuario siente como "lag" es sobre todo el **time-to-first-token** del LLM y el **time-to-first-audio** del TTS —cuánto tarda en **empezar** a responder—, no el tiempo total. Por eso dos palancas concretas: **streaming en cada etapa**, y **model tiering** —como el **time-to-first-token del LLM domina el lag percibido**, el cerebro conversacional debería ser el modelo **más rápido** (Haiku 4.5, el más veloz y barato) justamente para minimizar ese TTFT, reservando uno más pesado (Opus) solo si la tarea lo exige (criterio de costo/latencia de [LLMs](ia-llms.md) y [Prompt Engineering](prompt-engineering.md))—. No es "Haiku porque es barato"; es "Haiku porque el lag que sentís es cuánto tarda en *empezar* a hablar, y eso lo manda el TTFT". Y un costo que se olvida: el **endpointing** (decidir que el usuario terminó, módulo 6) **agrega latencia a propósito** —hay que esperar para estar seguro—, y es parte del presupuesto.

La frase mental: **la voz se siente rota arriba de ~1 s, y la latencia se apila por todo el pipeline; la solución es streamear y solapar cada etapa (parciales de STT, tokens del LLM, primer audio del TTS) y elegir un modelo rápido —porque lo que se percibe como lag es el tiempo en *empezar* a responder, no el total—.**

**Ejercicios 5**
5.1 ¿Cuál es el presupuesto de latencia conversacional y por qué importa tanto en voz?
5.2 Enumerá las etapas donde se apila la latencia de punta a punta.
5.3 ¿Cuál es la idea central para mantener la latencia baja, y cómo se aplica en STT, LLM y TTS?
5.4 ¿Por qué el time-to-first-token/first-audio importa más que el tiempo total? Nombrá dos palancas de latencia.

---

## Módulo 6 — Turnos de habla: VAD, endpointing y barge-in (la parte difícil)

**Teoría.** Lo que de verdad separa un voice agent usable de un demo no es el STT ni el TTS: es **manejar los turnos de habla** —saber cuándo el usuario empezó y terminó, y qué hacer si te interrumpe—. Es el problema más subestimado y el que más frameworks (módulo 9) existen para resolver. Tres conceptos:

- **VAD (Voice Activity Detection)**: detectar si en un chunk de audio **hay voz o silencio**. Es barato y rápido (modelos chicos como Silero VAD). Es el primer filtro: no mandás silencio al STT, y sabés cuándo alguien está hablando.

- **Endpointing / turn detection (detección de fin de turno)**: decidir que el usuario **terminó su turno** y le toca al agente. Suena trivial y es **muy difícil**: la gente hace pausas en medio de una idea ("quiero pedir… mmm… una pizza"). Si endpointeás muy rápido, **cortás al usuario** en medio de la frase; si esperás demasiado, la conversación se siente lenta. El enfoque básico es por **silencio** (X ms sin voz = fin de turno), pero el silencio fijo es tosco (un umbral corto corta a quien piensa en voz alta; uno largo demora a todos). Una mejora intermedia es el **timeout adaptativo**: ajustar el umbral al ritmo del hablante o al contexto, en vez de un número fijo. Y la frontera es el **endpointing semántico**: usar un modelo para decidir si lo que dijo es una frase *completa* (si pidió "quiero una…", claramente no terminó, aunque haya silencio).

- **Barge-in (interrupción)**: si el usuario **empieza a hablar mientras el agente está hablando**, hay que **cortar el TTS inmediatamente** y ponerse a escuchar —como en una conversación real, donde interrumpir es normal—. Requiere audio **full-duplex** (capturar y reproducir a la vez) y poder **cancelar** la reproducción en curso. Y un detalle duro que se subestima: con **altavoz abierto** (manos libres, no auriculares) el micrófono recaptura el propio TTS del agente, así que el barge-in fiable depende de **cancelación de eco acústico (AEC)** —separar la voz del usuario del eco del parlante—. La AEC es justo una de las cosas que **WebRTC trae de fábrica** (módulo 8) y el WebSocket crudo no, otra razón concreta para preferir WebRTC en clientes con altavoz. Un agente sin barge-in que te sigue hablando encima mientras intentás corregirlo es insufrible.

Estos tres juntos son la "máquina de turnos" de la conversación, y hacerla bien (sin cortar al usuario, sin demorar, manejando interrupciones) es donde se va la mayor parte del esfuerzo de ingeniería de un voice agent —y por qué casi nadie lo escribe a mano—. La frase mental: **el turno de habla es la parte difícil: VAD detecta voz vs silencio, el endpointing decide cuándo el usuario terminó (tosco por silencio, mejor semántico), y el barge-in corta al agente cuando lo interrumpen —manejar esto bien, no el STT/TTS, es lo que hace usable a un voice agent—.**

**Ejercicios 6**
6.1 ¿Qué hace el VAD y por qué es el primer filtro del pipeline?
6.2 ¿Por qué el endpointing es difícil? ¿Qué pasa si endpointeás muy rápido y si esperás demasiado? ¿Qué mejora el endpointing semántico sobre el de silencio fijo?
6.3 ¿Qué es el barge-in y qué requiere a nivel de audio para funcionar?
6.4 ¿Por qué se dice que el manejo de turnos —y no el STT/TTS— es lo que separa un demo de un agente usable?

---

## Módulo 7 — Speech-to-speech: los modelos de audio nativo en tiempo real

**Teoría.** La segunda arquitectura del módulo 1, ahora en detalle. Los modelos **speech-to-speech (S2S)** son multimodales: reciben **audio** y producen **audio** directamente, sin la traducción a texto en el medio. Ejemplos: la **Realtime API de OpenAI** y **Gemini Live** (Google), que operan sobre una conexión persistente (WebSocket/WebRTC, módulo 8) con audio entrando y saliendo en streaming.

Por qué son atractivos:
- **Latencia mínima**: un solo modelo en vez de tres saltos en serie → respuestas en cientos de ms, en rango conversacional.
- **Prosodia y emoción**: como no pasan por texto, **escuchan** *cómo* dijiste algo (tono, énfasis, duda) y **responden** con prosodia natural. Captan lo paralingüístico que la cascada pierde (módulo 1).
- **Interrupciones nativas**: el barge-in y los turnos los maneja el modelo, no vos.

Por qué **no** son siempre la respuesta:
- **Menos control y observabilidad**: no hay texto en el medio por defecto (aunque suelen emitir una transcripción), así que loguear, auditar y debuggear es más difícil que en la cascada.
- **Menos control del cerebro (no menos tools)**: ojo con un malentendido común —los modelos S2S (OpenAI Realtime, Gemini Live) **sí soportan function calling/tools**; lo que se pierde no es la capacidad de llamar herramientas, sino el **control fino del cerebro**: no podés poner a **Claude** específicamente, el prompting y la orquestación son más cerrados, y el ecosistema de RAG/retrieval sobre S2S es más nuevo que sobre un LLM de texto.
- **Lock-in del proveedor y del cerebro**: usás el modelo de audio de ese proveedor —no podés poner a Claude de cerebro—. La elección de arquitectura es también una elección de proveedor.
- **Costo por minuto de audio** en vez de por token: otro modelo mental de costos.

El criterio cascada vs S2S, que es la decisión central del campo: **S2S cuando la prioridad es latencia mínima y naturalidad y aceptás menos control** (un asistente de conversación casual, un compañero de voz); **cascada cuando necesitás control, observabilidad, tools/RAG maduros, o específicamente a Claude de cerebro** (un agente de soporte que ejecuta acciones, donde querés el texto logueado y el razonamiento auditible). No es "uno es mejor"; es qué prioriza tu caso. La frase mental: **speech-to-speech (OpenAI Realtime, Gemini Live) es audio→audio sin texto en el medio: latencia mínima, prosodia natural e interrupciones nativas, a cambio de menos control del cerebro/observabilidad y lock-in al proveedor (soporta tools, pero no podés usar Claude y el RAG es más nuevo) —elegís S2S por naturalidad y cascada por control—.**

**Ejercicios 7**
7.1 ¿Qué es un modelo speech-to-speech y en qué se diferencia de la cascada?
7.2 Nombrá las tres ventajas principales del S2S.
7.3 Nombrá tres razones para preferir la cascada a pesar de la mayor latencia.
7.4 Dá un caso de uso donde elegirías S2S y uno donde elegirías cascada, con la razón.

---

## Módulo 8 — Transporte: WebSockets y WebRTC para audio bidireccional

**Teoría.** El audio en vivo es **bidireccional, continuo y sensible a la latencia** —no es un request/response HTTP—. Necesitás un transporte de tiempo real, y hay dos opciones, que conectan directo con [Tiempo real](tiempo-real.md) y [FastAPI](fastapi.md):

- **WebSockets**: un canal full-duplex sobre una conexión persistente. Más **simple** y suficiente para muchos casos (server-a-server, o clientes controlados). Vos manejás el *chunking* del audio, el buffering y el formato. Va sobre **TCP**, que garantiza orden y entrega pero sufre *head-of-line blocking* (un paquete perdido frena los siguientes) —tolerable en redes buenas, problemático en redes con pérdida—. La telefonía suele entrar por acá: **Twilio Media Streams** manda el audio de una llamada (8 kHz) por WebSocket.

```python
from fastapi import FastAPI, WebSocket, WebSocketDisconnect

app = FastAPI()

@app.websocket("/voz")
async def voz(ws: WebSocket):
    await ws.accept()
    try:
        while True:
            chunk = await ws.receive_bytes()      # audio entrante en chunks (PCM/Opus)
            # pipeline: VAD → STT(parcial) → al terminar turno → LLM(stream) → TTS(stream)
            async for audio_out in pipeline(chunk):
                await ws.send_bytes(audio_out)     # audio saliente, también en streaming
    except WebSocketDisconnect:
        return    # el cliente cerró: es el camino de salida ESPERADO, no un error
```

> Sin el `try/except WebSocketDisconnect`, cada cierre del cliente te loguea un stack trace —la desconexión es el final normal de la conexión, no un fallo—. (Igual que cuidaste la desconexión en el streaming SSE de [FastAPI](fastapi.md).)

- **WebRTC**: el protocolo **pensado para media en tiempo real** (lo que usan las videollamadas). Va sobre **UDP**, y trae de fábrica lo que para voz en vivo es oro: **jitter buffer** (suaviza la variación de latencia de la red), **cancelación de eco** (que el micrófono no recapture lo que dice el parlante, clave para el barge-in del módulo 6), tolerancia a **pérdida de paquetes**, y *NAT traversal* (conectar clientes detrás de routers). Es más complejo de operar, pero es lo correcto para audio **navegador↔servidor o teléfono↔servidor** de verdad. Importante: en la práctica **casi nadie implementa WebRTC a mano** —el navegador no habla WebRTC directo contra tu lógica Python—; lo adoptás vía un **SFU/servidor de medios** (el que provee LiveKit) o una librería como `aiortc`. Es decir, WebRTC casi siempre llega **dentro de un framework** (módulo 9), no como un transporte que cableás vos —otro caso del criterio build-vs-framework—.

La regla: **WebSocket cuando controlás los extremos y la red es buena** (server-server, telefonía vía un proveedor que ya hace el WebRTC del lado del usuario); **WebRTC cuando el cliente es un navegador/teléfono en una red impredecible** y necesitás cancelación de eco y robustez a pérdida —cosas que a mano sobre WebSocket reimplementarías mal—. La frase mental: **el audio en vivo necesita transporte de tiempo real bidireccional; WebSocket (TCP, simple, vos manejás chunks; bueno server-server y telefonía) o WebRTC (UDP, con jitter buffer/cancelación de eco/tolerancia a pérdida; lo correcto para navegador o teléfono en redes reales)—.**

**Ejercicios 8**
8.1 ¿Por qué el audio en vivo no se transporta por HTTP request/response normal?
8.2 ¿Qué da WebRTC de fábrica que WebSocket no, y por qué eso importa para voz?
8.3 ¿Cuándo alcanza con WebSocket y cuándo necesitás WebRTC? Dá la regla.
8.4 ¿Cómo entra la telefonía (ej. una llamada) en este esquema? (pista: Twilio Media Streams)

---

## Módulo 9 — El voice agent: el agente del track con oídos y boca (+ frameworks)

**Teoría.** Acá se cierra el hilo: un **voice agent es el agente de [AI Agents](ai-agents-python.md)** —el LLM en un loop con tools, RAG, guardarraíles— **envuelto en STT (adelante) y TTS (atrás)**. Todo lo que sabés del agente sigue valiendo: el **tool use** funciona igual ("reservá una mesa para dos" → function call), el **RAG como tool** (agentic retrieval con permisos y grounding), la **confiabilidad** (topes, human-in-the-loop, mínimo privilegio). La voz no cambia el cerebro; le cambia la **interfaz**.

Pero la voz **agrega exigencias** que el agente de texto no tenía:
- **La latencia de las tools ahora se nota.** En texto, que una tool tarde 2 s es invisible; en voz, son 2 s de silencio incómodo. Solución conversacional: rellenar ("dejame chequear eso…") mientras la tool corre, o streamear un acuse antes del resultado.
- **Las respuestas tienen que ser "para escuchar", no "para leer".** Un párrafo con bullets y markdown es horrible dicho en voz. El prompt del cerebro debe pedir respuestas **cortas, habladas, sin formato visual** —es [prompt engineering](prompt-engineering.md) específico para voz—.
- **No todo lo que el LLM emite se habla.** El agente alterna entre **hablar** y **llamar tools**, y el loop tiene dos canales distintos: el "para actuar" (los `tool_use`, sus argumentos JSON, el razonamiento intermedio) y el "para hablar" (la respuesta al usuario). **Solo este último va al TTS** —si streameás al sintetizador el JSON de una llamada a herramienta o el texto intermedio, el usuario escucha basura—. Tenés que filtrar: al TTS van las cláusulas de respuesta, no las llamadas a tools ni el pensamiento.
- **Los turnos y el barge-in** (módulo 6) son parte del agente, no un detalle de UI.

Y como pasó en RAG y en agentes, hay una **capa de frameworks** que arma el pipeline de voz y —sobre todo— resuelve las partes difíciles de tiempo real (VAD, endpointing, barge-in, transporte) que no querés escribir a mano:

- **Pipecat** (open-source, Python): modela el pipeline de voz como una cadena de *frames* (audio, texto, eventos) que fluyen por etapas. Maneja STT/LLM/TTS, VAD, interrupción y transporte de forma pluggable.
- **LiveKit Agents**: infraestructura WebRTC + framework de agentes de voz.
- **Vapi** y similares: gestionados (el "Pinecone" de la voz).

```python
# Pipecat: el pipeline de voz como cadena de frames — forma CONCEPTUAL, verificá la API real
pipeline = Pipeline([
    transport.input(),     # audio del usuario (WebRTC/WebSocket)
    vad,                   # detección de voz + fin de turno (módulo 6)
    stt,                   # speech-to-text streaming (módulo 3)
    llm,                   # el cerebro: Claude con tools (el agente del track)
    tts,                   # text-to-speech streaming (módulo 4)
    transport.output(),    # audio al usuario, con barge-in
])
```

El criterio build vs framework es **más fuerte que en RAG**: las partes difíciles de la voz (interrupción full-duplex, endpointing, jitter, sincronización de streams) son exactamente lo que un framework hace bien y vos harías mal y lento. Salvo el caso más simple (transcribir un archivo y responder, sin tiempo real), **el framework casi siempre paga**. La frase mental: **un voice agent es el agente del track con STT/TTS alrededor —mismo cerebro, tools y guardarraíles, pero respuestas cortas para escuchar, latencia de tools que ahora se nota, y turnos/barge-in como parte del sistema—; y los frameworks (Pipecat/LiveKit) resuelven las partes duras de tiempo real que no querés escribir a mano—.**

**Ejercicios 9**
9.1 ¿Qué del agente de [AI Agents](ai-agents-python.md) sigue valiendo tal cual en un voice agent, y qué le cambia la voz?
9.2 Nombrá dos exigencias nuevas que la voz le agrega al agente, y cómo las resolvés.
9.3 ¿Qué resuelve un framework como Pipecat y por qué el criterio build-vs-framework es más fuerte acá que en RAG?
9.4 ¿Por qué no todo lo que emite el LLM se manda al TTS? ¿Qué dos canales separás y qué va a cada uno?

---

## Módulo 10 — Evaluación y producción de sistemas de voz

**Teoría.** "Suena bien en mi demo" no es una métrica. Un sistema de voz se evalúa por **capas y de punta a punta**, y conecta con [evals](evals.md) y [observabilidad](observabilidad.md). Las métricas que importan:

- **WER del STT** (módulo 3): la precisión de transcripción, la base del pipeline.
- **Latencia**: el **p95 de punta a punta** (no el promedio —la cola es la que arruina conversaciones—), y desglosada por etapa (time-to-first-token del LLM, time-to-first-audio del TTS) para saber dónde optimizar. Es el presupuesto del módulo 5 medido.
- **Naturalidad del TTS**: subjetiva, se mide con **MOS** (*Mean Opinion Score*, gente puntuando de 1 a 5) o juicios comparativos.
- **Task completion**: ¿el agente cumplió lo que el usuario quería? La métrica de resultado (como en agentes).
- **Calidad de turnos**: ¿cortó al usuario? ¿manejó bien las interrupciones? ¿tardó en responder?

La disciplina, igual que en RAG y agentes: **evaluá cada etapa por separado Y de punta a punta**, porque un fallo de voz puede estar en cualquier capa (mal WER → el LLM razona sobre basura; buen WER pero respuesta larga → el TTS la dice eterna; todo bien pero endpointing lento → se siente trabado). Sin desglosar, "optimizás el prompt" cuando el problema era el sample rate.

**QA de un pipeline de voz (el ángulo AI QA).** Medir no es lo mismo que **testear de forma reproducible**, y acá está el corazón del QA de sistemas de IA: el **no-determinismo** —el mismo audio puede dar distinta transcripción y, con un LLM detrás, distinta respuesta—, que rompe el `assert salida == esperada` clásico. Cómo se testea cada parte:

- **STT → golden set de transcripción.** Un dataset de audios etiquetados con su texto correcto; corrés el STT y medís WER contra esas etiquetas. Es tu **test de regresión de los oídos**: cuando cambiás de modelo o de proveedor, lo volvés a correr y ves si el WER subió. Incluí audios con la jerga de tu dominio (es donde más se degrada).
- **Respuesta del agente → testing por contrato, no por igualdad.** Como la salida no es determinista, no afirmás string exacto: fijás semilla/parámetros donde se pueda y evaluás por **contrato** (¿la respuesta contiene el dato correcto?, ¿llamó la tool correcta?) o con un **LLM-juez** ([evals](evals.md)), no por `==`. Es el mismo patrón de QA de IA de [Playwright](playwright.md)/[API testing](api-testing.md): separar lo determinista (¿el cableado funciona?) de lo no determinista (¿la respuesta es buena? → evals).
- **Turnos → fixtures de audio sintético.** Para testear barge-in y endpointing **de forma repetible** (no "a oído"), armás audios de prueba: uno con una **interrupción** (verificás que el TTS se corta) y uno con una **pausa en medio de una frase** (verificás que el endpointing NO corta antes de tiempo). Los turnos se testean con fixtures, igual que cualquier otro comportamiento.
- **Harness reproducible.** Audio de entrada fijo + parámetros fijos = test repetible en CI. Sin eso, "QA" es escuchar el demo y opinar —que no es QA—.

Producción suma sus propios temas: **costo** (la voz es **cara por conversación** —STT por minuto + TTS por carácter/minuto + tokens del LLM—, mucho más que una llamada de texto), **telefonía** (integración con Twilio para llamadas reales), **idiomas y acentos** (el WER varía por idioma), **accesibilidad** (la voz es una herramienta de accesibilidad enorme), y **observabilidad** (loguear el audio, la transcripción y la trayectoria del agente —módulo 10 de [AI Agents](ai-agents-python.md)— para poder debuggear qué pasó). La frase mental: **un sistema de voz se mide por capas (WER, latencia p95 por etapa, MOS de naturalidad) y de punta a punta (task completion, calidad de turnos); evaluás cada etapa por separado para saber dónde falla, y en producción pesan el costo por minuto, la telefonía y la observabilidad del audio+trayectoria—.**

**Ejercicios 10**
10.1 Nombrá cuatro métricas de un sistema de voz y qué capa evalúa cada una.
10.2 ¿Por qué se mira el p95 de latencia y no el promedio, y por qué desglosada por etapa?
10.3 ¿Por qué hay que evaluar cada etapa por separado además de punta a punta? Dá un ejemplo de un fallo mal diagnosticado.
10.4 ¿Por qué la voz es cara en producción y qué compone su costo?
10.5 ¿Por qué el no-determinismo rompe el `assert` clásico en QA de voz, y cómo testeás de forma reproducible (a) el STT, (b) la respuesta del agente y (c) los turnos?

---

## Módulo 11 — El criterio de cierre: cascada vs S2S, build vs framework, y el puente del track

**Teoría.** El cierre, con las decisiones en orden:

**1. ¿Voz, de verdad?** Antes que nada, el gate de simplicidad de todo el track. La voz agrega **muchísima complejidad** (latencia de tiempo real, transporte bidireccional, turnos, interrupción, costo alto). Solo vale cuando **la modalidad importa de verdad**: manos libres (manejando, cocinando), teléfono (call centers, IVR), accesibilidad, o una UX donde hablar es genuinamente mejor que tipear. Agregar voz "porque es cool" a algo que un chat de texto resolvía es el error caro equivalente a montar un agente cuando bastaba un workflow.

**2. ¿Cascada o speech-to-speech?** (módulos 1 y 7)
- **Cascada** (STT→Claude→TTS): cuando necesitás **control, observabilidad, tools/RAG maduros, o Claude de cerebro** —un agente que ejecuta acciones, con razonamiento auditable y texto logueado—.
- **S2S** (OpenAI Realtime/Gemini Live): cuando la prioridad es **latencia mínima y naturalidad** y aceptás menos control —conversación casual, compañía, baja fricción—.

**3. ¿Build o framework?** (módulo 9) Para cualquier cosa en tiempo real, **framework** (Pipecat/LiveKit): las partes difíciles (interrupción, endpointing, transporte, sincronización) son justo las que harías mal a mano. Build solo para el caso batch más simple.

**4. ¿Qué modelo en cada etapa?** El criterio de tiering del track: el **STT** según WER/latencia/privacidad (Whisper local vs proveedor streaming); el **cerebro** un modelo **rápido** para minimizar el time-to-first-token (Haiku conversacional, Opus si la tarea lo exige); el **TTS** según naturalidad/latencia/costo.

**Y dos ejes que cruzan todas las decisiones: costo y privacidad.** El **costo por conversación** es de primera clase, no un detalle de después: la cascada con Whisper local es **barata pero pelea la latencia**; el S2S es **fluido pero caro por minuto** de audio. Ese trade-off entra en la decisión de arquitectura misma. Y el audio es **dato sensible**: una grabación de voz es **PII**, exige consentimiento (más allá del *voice cloning*), una política de **retención** clara y cuidado de compliance —qué se loguea, por cuánto tiempo y quién accede es parte del diseño, no un agregado—.

El principio que cierra el track de IA, una última vez: **la mejor solución es la más simple que resuelve el problema real, y la decisión se mide.** En voz: "¿de verdad necesito voz? y si sí, ¿priorizo control (cascada) o naturalidad (S2S)?".

**El puente del track Python AI.** La voz es la **interfaz** sobre todo lo demás: el agente ([AI Agents](ai-agents-python.md)) con su RAG ([Vector Databases](vector-dbs.md)) y su prompting ([Prompt Engineering](prompt-engineering.md)), ahora hablado. Lo que cierra el track es **ponerlo en producción**: [Deploy de aplicaciones de IA](deploy-ai.md) —GPU, escalado, costo, servir modelos— es el último módulo, y [FastAPI](fastapi.md) + [evals](evals.md) atraviesan todo. La frase mental: **decidí en orden —¿voz de verdad?, ¿cascada o S2S?, ¿build o framework?, ¿qué modelo por etapa?— subiendo en complejidad solo cuando se justifica; la voz es la interfaz más natural y la más cara y compleja de construir, así que se agrega con criterio, no por moda—.**

**Ejercicios 11**
11.1 ¿Cuál es la primera pregunta antes de construir un sistema de voz, y cuándo se justifica la voz?
11.2 ¿Cuándo elegís cascada y cuándo speech-to-speech? Dá el factor decisivo de cada uno.
11.3 ¿Por qué el criterio build-vs-framework se inclina tan fuerte al framework en voz?
11.4 Enunciá la pregunta-criterio de la voz y mostrá cómo es la misma lógica de simplicidad medida del resto del track.

---

## Proyecto integrador (capstone): un asistente de voz sobre tus apuntes

El ejercicio que cierra el módulo y que mostrás cuando te piden "¿armaste algo con voz?". Es el capstone de [AI Agents](ai-agents-python.md) —el agente sobre tus apuntes— **ahora hablado**. No te damos la solución; te damos los **criterios de aceptación**. Construilo en Python.

**Qué construir.** Un asistente de voz de consola (o navegador) que responde por voz preguntas sobre los apuntes de este sitio:

1. **Captura** audio del micrófono en chunks (módulo 2), con **VAD** para detectar cuándo hablás y **endpointing** por silencio para saber cuándo terminaste (módulo 6).
2. **STT** con Whisper/`faster-whisper` (módulo 3) para transcribir tu pregunta.
3. **El cerebro**: el agente del capstone anterior (Claude con la tool `buscar_en_kb`, agentic retrieval con grounding) —reusá ese código—. **Prompt para voz**: respuestas cortas, habladas, sin markdown (módulo 9).
4. **TTS** con streaming para que empiece a hablar apenas el LLM emite la primera frase (módulos 4 y 5).

**Criterios de aceptación.**
- [ ] Detecta cuándo empezás y terminás de hablar (VAD + endpointing), sin cortarte en pausas cortas.
- [ ] Transcribe tu pregunta con un WER razonable en español (probá con jerga técnica de los apuntes).
- [ ] El agente decide cuándo buscar (agentic retrieval) y responde **solo** con lo recuperado, en frases cortas aptas para escuchar.
- [ ] El TTS **empieza a hablar antes** de que el LLM termine de generar (streaming) — medí el time-to-first-audio.
- [ ] Medís y mostrás la **latencia de punta a punta** y por etapa (módulos 5 y 10).
- [ ] Maneja **entrada degradada** sin romperse: silencio total, ruido, audio cortado o un STT que devuelve vacío → responde con una recuperación ("no te escuché, ¿podés repetir?"), no con un crash ni una respuesta inventada. (Es el ángulo QA: testealo a propósito.)

**Extensiones (suben el nivel).**
- Agregá **barge-in**: si empezás a hablar mientras el asistente habla, corta el TTS y te escucha (módulo 6).
- Servilo por **WebSocket con [FastAPI](fastapi.md)** (módulo 8) y un cliente de navegador.
- Compará la **cascada** (Whisper→Claude→TTS) contra un modelo **speech-to-speech** en latencia y naturalidad (módulo 7), y anotá los trade-offs que sentiste.
- Usá un STT de **streaming** (parciales) en vez de Whisper batch y medí cuánta latencia ganás (módulo 3).

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 🎤 → STT (voz→texto, los oídos) → LLM (razona, el cerebro) → TTS (texto→voz, la boca) → 🔊,
    en un loop turno a turno.
1.2 Ventajas: modular/reemplazable, el cerebro puede ser Claude (que es texto), y es debuggeable/
    logueable (hay texto en el medio). Desventajas: la latencia se apila (3 saltos en serie), se
    pierde lo paralingüístico (tono/emoción), y los errores se encadenan (STT mal → LLM razona
    sobre basura).
1.3 Porque Claude es un modelo de TEXTO (no tiene audio nativo); para usarlo de cerebro necesitás
    convertir voz↔texto alrededor, que es exactamente la cascada. El speech-to-speech usa un modelo
    de audio nativo de otro proveedor, donde Claude no entra.
1.4 La información paralingüística: cómo se dijo algo (tono, emoción, énfasis, duda). El STT te da
    las palabras pero no la prosodia; el S2S, al no pasar por texto, la conserva.
```

### Módulo 2
```
2.1 El sample rate es cuántas muestras por segundo se toman al digitalizar; el estándar de voz es
    16 kHz. Nyquist: para capturar frecuencias hasta f hay que muestrear a 2f; la voz humana vive
    bajo ~8 kHz, así que 16 kHz alcanza.
2.2 Porque la voz no necesita espacialidad (estéreo) y mono es la mitad de datos: menos ancho de
    banda y procesamiento, sin perder nada útil para transcribir.
2.3 PCM = muestras crudas sin comprimir; WAV = PCM con un header (simple, pesado); Opus = códec
    comprimido de baja latencia y buena calidad. Para tiempo real usás Opus (lo usa WebRTC).
2.4 Porque en vivo no tenés el audio completo: procesás chunks chicos (ej. 20 ms) a medida que
    llegan, lo que permite baja latencia y streaming. El bug más común es el sample rate equivocado
    (le das al modelo un rate distinto del que espera → transcripción basura).
```

### Módulo 3
```
3.1 STT/ASR transcribe audio a texto. Whisper es la referencia abierta: multilingüe, robusto al
    ruido, con varios tamaños. Correrlo local (faster-whisper/whisper.cpp) no manda el audio a un
    tercero → mejor privacidad (y sin costo por API).
3.2 Batch: le das audio completo y devuelve la transcripción (Whisper es batch). Streaming: devuelve
    parciales a medida que la persona habla. El batch puro es malo en conversación porque esperás a
    que el usuario termine de hablar antes de tener una sola palabra (latencia alta).
3.3 WER = (sustituciones + inserciones + eliminaciones) / palabras de referencia: la fracción de
    palabras erradas. Lo empeoran acento, ruido de fondo, vocabulario de dominio y mala calidad de
    audio. Palancas: hint de idioma, biasing por palabras clave del dominio (también: mejor captura).
3.4 Porque en cascada el error del STT se propaga: el LLM razona sobre el texto mal transcripto, así
    que un WER alto contamina toda la respuesta aunque el cerebro sea perfecto.
```

### Módulo 4
```
4.1 Streaming TTS = generar audio a medida que el texto llega, sin esperar la respuesta completa del
    LLM. Es clave para latencia porque el agente empieza a hablar con la primera cláusula. Importa
    más el time-to-first-audio (cuándo empieza a sonar) que el tiempo total.
4.2 SSML es marcado para controlar la salida del TTS: pausas, énfasis, pronunciación, prosodia.
    Ej.: insertar una pausa (<break time="500ms"/>) o forzar cómo se lee una sigla/número/fecha.
4.3 La clonación de voz requiere consentimiento: clonar la voz de alguien sin permiso es un problema
    legal y ético, además del riesgo de deepfakes/suplantación.
4.4 La prosodia es la entonación, el ritmo y el énfasis del habla. Una voz casi-humana pero con
    prosodia rara cae en el "valle inquietante" (uncanny valley) y molesta más que una claramente
    sintética; la prosodia es lo que hace sonar natural o robótico al TTS.
4.5 La normalización de texto convierte el texto a su forma hablada antes de sintetizar: números,
    fechas, montos, símbolos y siglas ("$1.000" → "mil pesos", "12/3" → "doce de marzo", "Dr." →
    "doctor"). SSML no siempre alcanza porque el TTS adivina por contexto/idioma y erra (sobre todo
    con jerga de dominio), y son errores AUDIBLES que no se ven en el texto —se notan solo al
    escuchar—. Ej. típico: un monto o una sigla leídos mal.
```

### Módulo 5
```
5.1 ~200–500 ms es el gap natural entre turnos humanos; el objetivo es responder en menos de ~1 s
    punta a punta. Importa porque arriba de eso la conversación se siente rota por más inteligente
    que sea la respuesta: la voz es un problema de tiempo, no solo de calidad.
5.2 Captura del micrófono + endpointing (decidir que terminó) + STT + LLM (time-to-first-token) +
    TTS (time-to-first-audio) + red + reproducción. Todo en serie, así que se suma.
5.3 Streamear y solapar todo, no esperar a que cada etapa termine: STT con parciales, tokens del LLM
    en streaming (arrancar el TTS con la primera cláusula), y solapar etapas (el TTS dice la frase 1
    mientras el LLM genera la 2).
5.4 Porque lo que se percibe como "lag" es cuánto tarda en EMPEZAR a responder, no el total; con
    streaming el usuario oye el inicio enseguida aunque la respuesta completa tarde. Palancas:
    streaming en cada etapa y model tiering (un modelo rápido como cerebro para bajar el TTFT).
```

### Módulo 6
```
6.1 El VAD detecta si un chunk de audio tiene voz o silencio. Es el primer filtro: no manda silencio
    al STT y permite saber cuándo alguien está hablando (es barato y rápido).
6.2 Porque la gente hace pausas en medio de una idea: distinguir "pausa" de "terminé" es ambiguo.
    Muy rápido → cortás al usuario en medio de la frase; muy lento → la conversación se siente
    trabada. El endpointing semántico usa un modelo para decidir si la frase está completa (no solo
    si hay silencio), evitando cortar un "quiero una…".
6.3 Barge-in = que el usuario interrumpa al agente mientras habla, cortando el TTS y poniéndose a
    escuchar. Requiere audio full-duplex (capturar y reproducir a la vez) y poder cancelar la
    reproducción en curso.
6.4 Porque el STT/TTS son piezas resueltas por proveedores, pero manejar bien los turnos (no cortar,
    no demorar, manejar interrupciones) es lo que hace que la conversación se sienta natural o rota
    —y es donde se va la mayor parte de la ingeniería—.
```

### Módulo 7
```
7.1 Un modelo S2S recibe audio y produce audio directamente, sin texto en el medio; la cascada usa
    tres modelos (STT→LLM→TTS) con texto entre ellos.
7.2 (1) Latencia mínima (un salto, no tres). (2) Prosodia y emoción (escucha y responde el cómo, no
    solo el qué). (3) Manejo nativo de interrupciones/turnos.
7.3 (Tres de) más control del cerebro y observabilidad (hay texto que loguear/auditar/debuggear);
    poder usar Claude de cerebro con orquestación/RAG más maduros (el S2S sí soporta tools, pero su
    ecosistema de RAG es más nuevo); y un modelo de costos por token en vez de por minuto de audio.
    Evita además el lock-in al proveedor de audio.
7.4 S2S: un compañero/asistente de conversación casual donde la naturalidad y la baja latencia
    mandan. Cascada: un agente de soporte que ejecuta acciones, donde querés control, texto logueado,
    tools/RAG maduros y a Claude razonando de forma auditable.
```

### Módulo 8
```
8.1 Porque el audio en vivo es bidireccional, continuo y sensible a la latencia; HTTP
    request/response no sirve para un stream full-duplex continuo de baja latencia.
8.2 WebRTC trae jitter buffer (suaviza variación de latencia), cancelación de eco, tolerancia a
    pérdida de paquetes (va sobre UDP) y NAT traversal. Importa porque en redes reales/navegador hay
    pérdida, jitter y eco que de otro modo arruinan el audio.
8.3 WebSocket cuando controlás los extremos y la red es buena (server-server, o telefonía vía un
    proveedor que ya hace el WebRTC del lado del usuario). WebRTC cuando el cliente es un
    navegador/teléfono en una red impredecible y necesitás cancelación de eco y robustez a pérdida.
8.4 Un proveedor de telefonía como Twilio manda el audio de la llamada (8 kHz) por WebSocket
    (Twilio Media Streams) a tu servidor, que corre el pipeline y devuelve el audio.
```

### Módulo 9
```
9.1 Sigue valiendo todo el cerebro: el loop, el tool use, el RAG como tool (con permisos y
    grounding) y la confiabilidad (topes, HITL, mínimo privilegio). La voz no cambia el cerebro, le
    cambia la interfaz (STT adelante, TTS atrás).
9.2 (Dos de) la latencia de las tools ahora se nota (en voz son silencios incómodos → rellenás con
    "dejame chequear…" o un acuse mientras corre); las respuestas deben ser cortas y habladas, sin
    markdown (prompt para voz); y los turnos/barge-in pasan a ser parte del agente.
9.3 Pipecat (y LiveKit/Vapi) arma el pipeline de voz y resuelve las partes duras de tiempo real
    (VAD, endpointing, barge-in, transporte, sincronización de streams). El criterio se inclina más
    al framework que en RAG porque esas partes son justo las que a mano harías mal y lento; salvo el
    caso batch más simple, el framework casi siempre paga.
9.4 Porque el loop del agente tiene dos canales: el "para actuar" (las llamadas a tools, sus
    argumentos JSON, el razonamiento intermedio) y el "para hablar" (la respuesta al usuario). Solo
    el segundo va al TTS. Si streameás al sintetizador el JSON de un tool_use o el texto intermedio,
    el usuario escucha basura; hay que filtrar y mandar al TTS solo las cláusulas de respuesta.
```

### Módulo 10
```
10.1 (Cuatro de) WER (precisión del STT), latencia p95 punta a punta y por etapa (TTFT del LLM,
     time-to-first-audio del TTS), MOS/naturalidad del TTS, task completion (resultado del agente) y
     calidad de turnos (¿cortó?, ¿manejó interrupciones?, ¿tardó?).
10.2 El p95 porque el promedio esconde la cola, y es la cola la que arruina conversaciones. Desglosada
     por etapa para saber DÓNDE optimizar (si el lag está en el TTFT del LLM, en el STT o en el
     endpointing).
10.3 Porque un fallo puede estar en cualquier capa y la causa/arreglo son distintos; sin desglosar,
     "optimizás el prompt" cuando el problema era el sample rate. Ej.: respuesta correcta pero el
     usuario la siente lenta → no es el LLM, es el endpointing demasiado conservador.
10.4 Porque paga por capa: STT por minuto de audio + TTS por carácter/minuto + tokens del LLM, mucho
     más caro por conversación que una interacción de solo texto.
10.5 Porque el mismo audio puede dar distinta transcripción y distinta respuesta del LLM, así que
     "salida == esperada" falla aunque el sistema esté bien. Se testea: (a) STT con un golden set de
     audios etiquetados → WER contra esas etiquetas (regresión de los oídos al cambiar de modelo);
     (b) la respuesta del agente por CONTRATO (¿contiene el dato?, ¿llamó la tool correcta?) o con
     LLM-juez, fijando semilla/parámetros donde se pueda, no por igualdad exacta; (c) los turnos con
     fixtures de audio sintético (un audio con interrupción → verificar que corta; uno con pausa en
     medio de frase → verificar que el endpointing NO corta antes). Todo con entrada y parámetros
     fijos para que sea repetible en CI.
```

### Módulo 11
```
11.1 "¿Necesito voz de verdad?" La voz se justifica cuando la modalidad importa: manos libres,
     teléfono/call center, accesibilidad, o una UX donde hablar es genuinamente mejor que tipear.
     Agregarla "porque es cool" a algo que un chat resolvía es el error caro.
11.2 Cascada cuando necesitás control, observabilidad, tools/RAG maduros o Claude de cerebro (agente
     que actúa, razonamiento auditable). S2S cuando priorizás latencia mínima y naturalidad y
     aceptás menos control (conversación casual). Factor decisivo: control vs naturalidad/latencia.
11.3 Porque las partes difíciles de la voz (interrupción full-duplex, endpointing, jitter,
     sincronización de streams en tiempo real) son justamente lo que un framework hace bien y vos
     harías mal y lento; solo el caso batch más simple justifica build.
11.4 "¿De verdad necesito voz? y si sí, ¿priorizo control (cascada) o naturalidad (S2S)?" Es la
     misma lógica de simplicidad medida del track: LLM ("¿modelo o if?"), RAG ("¿muchos docs o
     pocos?"), agentes ("¿autonomía o control?"), voz ("¿hace falta la modalidad, y qué prioriza?").
     Siempre: la solución más simple que resuelve el problema real, medida.
```

---

## Siguientes pasos

Con este módulo tenés la IA de voz con criterio de producción: el pipeline STT→LLM→TTS y las dos arquitecturas (cascada vs speech-to-speech), los fundamentos de audio que evitan los bugs típicos, el reconocimiento de voz (Whisper, batch vs streaming, WER), la síntesis (neural, streaming, SSML), el presupuesto de latencia y por qué la voz es un problema de tiempo real, el manejo de turnos (VAD, endpointing, barge-in) que es la parte difícil, los modelos speech-to-speech, el transporte (WebSocket vs WebRTC), el voice agent como el agente del track con oídos y boca (más los frameworks Pipecat/LiveKit), y cómo evaluar y operar todo. Con la regla de siempre: **la solución más simple que resuelve el problema real, medida —y la voz se agrega con criterio, no por moda—.** Lo que cierra el track Python AI es [Deploy de aplicaciones de IA](deploy-ai.md): cómo poner en producción todo lo que construiste —servir modelos, GPU, escalado, costo y latencia a escala—, con [FastAPI](fastapi.md) sirviendo y [evals](evals.md) midiendo de punta a punta. El hilo del track queda casi completo: aprendiste el lenguaje, a servir APIs, a hablarle al modelo, a darle tu información a escala, a dejarlo actuar, y ahora a **darle voz** —falta llevarlo a producción—.
