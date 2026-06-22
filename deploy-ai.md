# Deploy de aplicaciones de IA: de la API a la GPU

**Poner en producción sistemas de IA · llamar a una API vs servir tu propio modelo, GPU, vLLM, escalado y costo · stack Python / Claude · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Es el **módulo que cierra el track Python AI**: asume [Python](python.md), [FastAPI](fastapi.md) (servir + streaming), y todo lo que construiste —[Prompt Engineering](prompt-engineering.md), [Vector Databases](vector-dbs.md), [AI Agents](ai-agents-python.md), [Voice AI](voice-ai.md)—. Y asume los módulos **genéricos** de deploy del temario, que **no repetimos**: [Docker, CI/CD y deploy](docker-deploy.md) (imágenes, pipelines, healthchecks, graceful shutdown), [AWS](aws.md) (cómputo gestionado, colas, escalado) y [Observabilidad](observabilidad.md) (logs, métricas, tracing, OTel, SLO). Este módulo es el **delta de IA**: qué cambia cuando lo que desplegás llama a un LLM o, peor, *es* un LLM. El objetivo es doble: las **técnicas** (servir modelos con GPU y vLLM, métricas de inferencia, escalado) y el **criterio** que se evalúa en una entrevista —**la decisión #1 es llamar a una API o servir tu propio modelo**, y la mayoría de las veces la respuesta sensata es la primera—.

**Lo que asumimos.** Docker y deploy genérico ([Docker](docker-deploy.md)), nociones de cloud ([AWS](aws.md)), observabilidad ([Observabilidad](observabilidad.md)), FastAPI con streaming, y el costo/tiering de [LLMs](ia-llms.md) y [Prompt Engineering](prompt-engineering.md).

**Para practicar.**

```bash
# La app que llama a una API: nada nuevo de infra (Docker + FastAPI del track)
uv add anthropic fastapi uvicorn
# Servir tu propio modelo (módulos 3-5) requiere GPU; vLLM se instala donde hay CUDA:
#   uv add vllm        # o correrlo como contenedor en una instancia con GPU
```

> Nota sobre datos volátiles: precios de GPU y de tokens, modelos open-weight, y las features de los servidores de inferencia y plataformas (Bedrock, SageMaker, Modal, Replicate…) cambian constantemente. Los ejemplos muestran la **forma** y los **órdenes de magnitud** a 2026; verificá precios y modelos vigentes al implementar. Datos de modelos Claude, con la skill `claude-api`.

**Índice de módulos**
1. La bifurcación: llamar a una API vs servir tu propio modelo
2. Desplegar la app que llama a una API: un web service (casi) normal
3. Servir tu propio modelo: GPU y VRAM (cuándo y por qué)
4. El servidor de inferencia: vLLM, continuous batching y PagedAttention
5. Las métricas de serving: throughput, TTFT, TPOT y el trade-off del batching
6. Escalar inferencia: por qué es difícil (GPUs caras, cold starts de minutos)
7. Dónde correrlo: serverless-GPU vs endpoints gestionados vs GPU propia
8. Ingeniería de costos: la métrica que manda en IA
9. Observabilidad de IA: los tres pilares + tokens, costo y calidad
10. Confiabilidad: outages, fallbacks, rate limits y degradación
11. El criterio de cierre: API vs self-host, dónde, y el cierre del track

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — La bifurcación: llamar a una API vs servir tu propio modelo

**Teoría.** Antes de cualquier detalle, la decisión que parte el mundo del deploy de IA en dos —y el error de principiante es no verla—: **¿tu sistema llama a un modelo que corre en la infraestructura de otro (una API gestionada como Claude/Anthropic), o vos servís tu propio modelo (open-weight, en tu GPU)?** Son dos problemas de deploy completamente distintos.

**Camino A — Llamar a una API gestionada** (Claude, y similares). Tu modelo vive en Anthropic; vos mandás un request HTTP y recibís la respuesta. Para tu deploy, **el LLM es una dependencia externa, como una base de datos gestionada**. No tenés GPU, ni pesos de modelo, ni servidor de inferencia. Tu app es un **web service normal** (FastAPI) que hace llamadas de red. Esto es lo que usa **la enorme mayoría** de los sistemas de IA en producción, y es el default sensato.

**Camino B — Servir tu propio modelo.** Tomás un modelo open-weight (Llama, Mistral, Qwen, un modelo fine-tuneado tuyo), lo cargás en una **GPU** que vos operás, y exponés tu propio endpoint de inferencia. Ahora el deploy incluye GPU, VRAM, un servidor de inferencia (vLLM), y el escalado de todo eso. Es muchísimo más complejo y caro de operar.

Cuándo justifica el camino B (y solo entonces):
- **Privacidad / compliance**: la data no puede salir de tu infraestructura (salud, legal, gobierno).
- **Costo a escala**: a volumen **alto y sostenido**, una GPU propia puede salir más barata que pagar por token (el crossover del módulo 8).
- **Latencia / control**: sin saltos de red a un tercero, sin rate limits ajenos, control total de versión del modelo.
- **Modelo propio**: usás un modelo open-weight o fine-tuneado que ninguna API ofrece.

El principio, el mismo de todo el temario ([liderazgo](liderazgo.md), y el "empezá simple" de [Agentes](agentes.md)): **empezá llamando a una API; pasate a servir tu propio modelo solo cuando un requisito concreto —privacidad, costo a escala, latencia, un modelo específico— lo exija y lo puedas nombrar.** El error caro es montar un cluster de GPUs "porque es lo serio" cuando una API resolvía el caso con una fracción del esfuerzo. La frase mental: **el deploy de IA se bifurca en llamar a una API gestionada (tu app es un web service normal con el LLM de dependencia externa —el default) o servir tu propio modelo (GPU + servidor de inferencia, mucho más complejo) —y solo subís al segundo por privacidad, costo a escala, latencia o un modelo propio—.**

**Ejercicios 1**
1.1 Describí los dos caminos de deploy de IA y por qué son problemas distintos.
1.2 En el camino A (API), ¿a qué se parece el LLM desde el punto de vista de tu deploy?
1.3 Nombrá las cuatro razones que justifican servir tu propio modelo (camino B).
1.4 ¿Cuál es el default sensato y cuál el error de principiante? Conectá con el criterio del temario.

---

## Módulo 2 — Desplegar la app que llama a una API: un web service (casi) normal

**Teoría.** Si elegiste el camino A (lo más común), buena noticia: **el grueso del deploy es el genérico que ya sabés** —Docker, imagen multi-stage, CI/CD, healthchecks, graceful shutdown ([Docker](docker-deploy.md)), cómputo gestionado y autoscaling ([AWS](aws.md))—. Tu app FastAPI que llama a Claude se despliega como cualquier API. Lo que **sí cambia** —el delta de IA— son cinco cosas:

- **Requests largos.** Una llamada a un LLM puede tardar **segundos** (o más con razonamiento), no milisegundos. Esto rompe supuestos: timeouts de load balancer y de cliente más largos, y **streaming casi obligatorio** —mandar la respuesta token a token con SSE/WebSocket ([FastAPI](fastapi.md))— para que el usuario vea progreso y no un spinner de 10 s. El streaming a su vez exige que tu LB/proxy **no buffere** la respuesta y mantenga la conexión.
- **Concurrencia I/O-bound.** Tu servidor pasa la mayor parte del tiempo **esperando** la respuesta del LLM (red), no computando. Con **async** ([Python](python.md): asyncio), un solo proceso maneja **cientos de llamadas concurrentes** mientras esperan. Acá el modelo de concurrencia de FastAPI/async rinde de verdad: no necesitás muchos workers para muchas requests, necesitás no bloquear el event loop.
- **Trabajo pesado a colas.** Lo que no es interactivo —el **ingest** de [RAG](rag.md) (chunking + embeddings de miles de docs), procesar un batch— va a una **cola / background job**, no al request (igual que cualquier trabajo batch en [AWS](aws.md)/colas).
- **Rate limits y reintentos del proveedor.** La API tiene límites (requests/tokens por minuto) y puede devolver `429`. Tu cliente necesita **backoff exponencial y reintentos** (módulo 10), cosa que con una DB propia no pensabas.
- **El costo es operacional.** Cada request cuesta plata (tokens). El costo deja de ser solo "infra" y pasa a ser una métrica de producto que monitoreás (módulos 8 y 9).

Y los secretos: la **API key** es un secreto como cualquier otro —va en el secret manager, nunca en la imagen ni en el repo ([Docker](docker-deploy.md) módulo 5, [AWS](aws.md) IAM)—. La frase mental: **una app que llama a una API de IA se despliega como un web service normal (Docker + autoscaling del track), con cinco deltas: requests largos (→ streaming, timeouts), concurrencia I/O-bound (→ async maneja cientos de llamadas en espera), trabajo pesado a colas, manejo de rate limits/reintentos, y el costo como métrica de primera clase—.**

**Ejercicios 2**
2.1 Si llamás a una API, ¿qué parte del deploy es la genérica que ya sabés y cuál es el delta de IA?
2.2 ¿Por qué los requests largos vuelven al streaming casi obligatorio, y qué exige eso de tu load balancer?
2.3 ¿Por qué un servidor que llama a un LLM es I/O-bound, y qué implica para cuántas requests concurrentes maneja con async?
2.4 ¿Qué trabajo mandás a una cola en vez de al request, y por qué? (pensá en el ingest de RAG)

---

## Módulo 3 — Servir tu propio modelo: GPU y VRAM (cuándo y por qué)

**Teoría.** Camino B. Para correr un LLM vos mismo necesitás una **GPU**, y entender por qué cambia todo. Un LLM es básicamente multiplicaciones de matrices gigantes; las GPUs hacen eso **masivamente en paralelo** (miles de núcleos), mientras una CPU lo haría en serie y tardaría una eternidad. Pero la GPU impone una restricción dura que domina todo el self-hosting: **la VRAM (memoria de la GPU)**.

El modelo tiene que **entrar en VRAM**, y su tamaño se calcula: **parámetros × bytes por parámetro**. En precisión `fp16` (16 bits = 2 bytes), un modelo de **7.000 millones de parámetros (7B)** ocupa **~14 GB** solo de pesos; uno de **70B**, **~140 GB** —no entra en una sola GPU, necesitás varias—. Por eso aparece la **cuantización** (la misma idea que en [Vector Databases](vector-dbs.md) módulo 6, ahora sobre los pesos del modelo): bajar la precisión a `int8` (~1 byte/param → ~7 GB para 7B) o `int4` (~0.5 byte → ~3.5 GB), aceptando una pérdida pequeña de calidad para que el modelo entre en menos VRAM o en una GPU más barata. Ojo: esos GB son solo los **pesos** —el KV cache (lo que sigue) se suma encima, y cuantizar los pesos **no** lo achica—.

Y un consumo de VRAM que los principiantes olvidan: **el KV cache**. Durante la generación, el modelo guarda en memoria los estados intermedios (claves y valores de atención) de **todos los tokens del contexto**, y eso **crece con la longitud del contexto y con la cantidad de requests concurrentes**. En serving real, el KV cache puede ocupar tanta o más VRAM que los pesos. Por eso "¿entra el modelo?" no es solo `pesos < VRAM`: es `pesos + KV cache de todas las requests en vuelo < VRAM`.

Las GPUs, además, son **caras y escasas**: las de datacenter (A100, H100 y sucesoras) cuestan varios dólares por hora y **se pagan estén o no en uso** (módulo 6). La frase mental: **servir tu propio modelo exige GPU, y la restricción que manda es la VRAM: el modelo entra según parámetros × bytes/param (7B fp16 ≈ 14 GB), la cuantización (int8/int4) lo achica a costa de calidad, y el KV cache —que crece con contexto y concurrencia— compite por esa misma memoria—.**

**Ejercicios 3**
3.1 ¿Por qué se usa GPU y no CPU para servir un LLM?
3.2 Calculá cuánta VRAM ocupan los pesos de un modelo 7B en fp16, int8 e int4. ¿Qué se sacrifica al cuantizar?
3.3 ¿Qué es el KV cache y por qué "¿entra el modelo en VRAM?" no se responde solo con el tamaño de los pesos?
3.4 ¿Por qué el costo de las GPUs es un problema aun cuando no las estás usando? (adelanta el módulo 6)

---

## Módulo 4 — El servidor de inferencia: vLLM, continuous batching y PagedAttention

**Teoría.** Tener el modelo en la GPU no alcanza: necesitás un **servidor de inferencia** que reciba requests, los procese eficientemente y exponga un endpoint. Y acá está el salto que separa un juguete de algo productivo: **cómo servís importa más que qué GPU tenés.** El estándar de facto es **vLLM** (open-source), y entender sus dos innovaciones es entender por qué el serving naïve es terrible.

El **serving naïve** (correr el modelo con la librería `transformers` de Hugging Face, un request a la vez): procesás una request, después la siguiente. La GPU —que puede hacer miles de operaciones en paralelo— pasa la mayor parte del tiempo **subutilizada**, esperando. Throughput pésimo. Las dos ideas de vLLM lo arreglan:

- **Continuous batching (batching continuo).** En vez de procesar de a una request, o esperar a juntar un "lote" fijo (static batching, que hace esperar a las requests rápidas por las lentas), vLLM **mete y saca requests del batch en cada paso de generación**: en cuanto una secuencia termina, libera su lugar y entra una nueva, sin esperar a que el batch entero termine. La GPU queda **siempre llena de trabajo útil** → throughput muchísimo mayor.
- **PagedAttention.** El KV cache (módulo 3) tradicionalmente se reserva en bloques contiguos y rígidos, desperdiciando VRAM por fragmentación. PagedAttention lo gestiona como la **memoria virtual paginada** de un sistema operativo: KV cache en "páginas" no contiguas, asignadas bajo demanda. Resultado: **mucho menos desperdicio de VRAM** → cabe más KV cache → batches más grandes → más throughput.

```bash
# Servir un modelo open-weight con vLLM, con un endpoint compatible con la API de OpenAI
vllm serve meta-llama/Llama-3.1-8B-Instruct --max-model-len 8192
# expone /v1/chat/completions; tu app FastAPI le pega como si fuera una API más
# Para cuantizar, apuntá a un checkpoint YA cuantizado (no al fp16 de arriba):
#   vllm serve <repo>-AWQ --quantization awq   # AWQ/GPTQ requieren pesos pre-cuantizados
```

Otros servidores que vas a oír: **TGI** (Hugging Face), **TensorRT-LLM** (NVIDIA, máximo rendimiento, más complejo), y **Ollama** (genial para correr modelos en tu máquina en desarrollo, **no** para servir producción a escala). El detalle de criterio: vLLM suele exponer una **API compatible con la de OpenAI**, así que tu app cambia de proveedor (API gestionada ↔ tu vLLM) tocando casi solo la URL base —la portabilidad del [framework layer](vector-dbs.md)—. La frase mental: **servir un LLM bien es un problema de throughput, no solo de tener GPU; vLLM lo resuelve con continuous batching (la GPU siempre llena, sin esperar lotes fijos) y PagedAttention (KV cache paginado, sin desperdicio de VRAM) —el serving naïve un-request-a-la-vez desaprovecha la GPU—.**

**Ejercicios 4**
4.1 ¿Por qué tener el modelo en la GPU no alcanza, y qué problema tiene el serving naïve (un request a la vez)?
4.2 Explicá el continuous batching y en qué se diferencia del static batching (lote fijo).
4.3 ¿Qué problema de VRAM resuelve PagedAttention y con qué concepto de sistemas operativos se lo compara?
4.4 ¿Qué ventaja de portabilidad da que vLLM exponga una API compatible con la de OpenAI?

---

## Módulo 5 — Las métricas de serving: throughput, TTFT, TPOT y el trade-off del batching

**Teoría.** "Anda rápido" no es una métrica. Servir un LLM tiene métricas propias, y confundirlas lleva a optimizar lo que no es. La generación de un LLM tiene **dos fases** con costos distintos:

- **Prefill**: procesar el prompt de entrada (en paralelo, rápido) → produce el **primer token**.
- **Decode**: generar los tokens de salida **uno por uno** (cada uno depende del anterior, secuencial) → el grueso del tiempo en respuestas largas.

De ahí salen las tres métricas que importan:

- **TTFT (Time To First Token)**: cuánto tarda en salir el primer token (≈ el tiempo de prefill + cola). Es lo que el usuario percibe como "cuánto tarda en empezar a responder" —clave para interactividad y para [voz](voice-ai.md)—.
- **TPOT / ITL (Time Per Output Token / Inter-Token Latency)**: cuánto tarda **cada** token siguiente. Define qué tan fluido sale el texto en streaming. TTFT + TPOT × n_tokens ≈ latencia total.
- **Throughput**: total de **tokens por segundo** (o requests/seg) que el servidor produce **sumando todas las requests concurrentes**. Es la métrica de **capacidad** (cuántos usuarios bancás) y la que define el costo por token.

Y el trade-off central, que es la decisión de tuning del serving: **batching sube el throughput pero empeora la latencia por request.** Más requests en el batch = la GPU produce más tokens/seg en total (mejor throughput, menor costo por token) pero cada request individual espera más (peor TTFT/TPOT). Por eso:
- Carga **interactiva** (chat, voz): priorizás **latencia** (TTFT/TPOT bajos), aceptás menos throughput.
- Carga **batch** (procesar un corpus offline): priorizás **throughput** (tokens/seg, costo), la latencia por request no importa.

Es el mismo "medí, no adivines" de [observabilidad](observabilidad.md) (mirá percentiles, p95/p99, no promedios) aplicado a inferencia. La frase mental: **la inferencia se mide con TTFT (cuánto tarda en empezar), TPOT (cuánto tarda cada token, la fluidez) y throughput (tokens/seg agregados, la capacidad y el costo); y el batching cambia throughput por latencia —priorizás latencia en lo interactivo y throughput en lo batch—.**

**Ejercicios 5**
5.1 Diferenciá las fases de prefill y decode. ¿Cuál domina el tiempo en una respuesta larga y por qué?
5.2 Definí TTFT, TPOT y throughput. ¿Cuál percibe el usuario como "tarda en empezar"?
5.3 Explicá el trade-off del batching entre throughput y latencia.
5.4 Para chat interactivo vs procesar un corpus offline, ¿qué métrica priorizás en cada uno y por qué?

---

## Módulo 6 — Escalar inferencia: por qué es difícil (GPUs caras, cold starts de minutos)

**Teoría.** Escalar un web service normal es un problema resuelto ([AWS](aws.md): autoscaling, más réplicas cuando sube la carga, scale-to-zero cuando no hay tráfico). **Escalar inferencia en GPU propia es mucho más difícil**, y las razones son específicas:

- **Las GPUs son caras y se pagan 24/7.** Una instancia con GPU cuesta varios dólares la hora **esté o no procesando**. En un web service CPU podés tener réplicas baratas y escalar agresivo; con GPUs, cada réplica ociosa quema plata.
- **El scale-to-zero casi no existe en la práctica.** Apagar la GPU cuando no hay tráfico suena ideal, pero **prenderla de nuevo tarda minutos**: hay que provisionar la instancia (GPUs escasas, a veces ni hay disponibilidad), bajar la imagen, y **cargar los pesos del modelo en VRAM** (decenas de GB). Ese **cold start de minutos** hace inviable el scale-to-zero para tráfico interactivo —el primer usuario tras el apagado espera una eternidad—.
- **Autoscaling con señales distintas.** No escalás por CPU; escalás por **utilización de GPU**, **profundidad de la cola de requests** o **TTFT** subiendo. Y como agregar una réplica tarda minutos, el autoscaling **reacciona tarde** —tenés que anticipar la demanda o mantener un colchón—.

Las salidas, en orden de "cuánto operás vos":
- **Sobre-aprovisionar** (always-on): mantenés N GPUs prendidas para el pico. Simple, caro (pagás el ocio).
- **Cola + batch**: para trabajo no interactivo, encolás requests y los procesás en batch con throughput alto (módulo 5), exprimiendo cada GPU.
- **Serverless-GPU gestionado** (módulo 7): plataformas que manejan el cold start y el escalado por vos.

El contraste que cierra el módulo: **si llamás a una API gestionada (camino A), todo esto no es tu problema** —Anthropic escala el modelo, vos escalás solo tu web service liviano (módulo 2), que sí hace scale-to-zero barato—. El escalado difícil de inferencia es **un costo del camino B** que hay que poner en la balanza. La frase mental: **escalar inferencia propia es difícil porque las GPUs son caras y se pagan 24/7, el scale-to-zero choca con cold starts de minutos (cargar los pesos), y el autoscaling reacciona tarde; se mitiga sobre-aprovisionando, encolando para batch, o con serverless-GPU —y nada de esto existe si llamás a una API gestionada—.**

**Ejercicios 6**
6.1 ¿Por qué escalar inferencia en GPU propia es más difícil que escalar un web service CPU normal?
6.2 ¿Por qué el scale-to-zero es inviable para tráfico interactivo con modelo propio? (explicá el cold start)
6.3 ¿Por qué no escalás por CPU, y qué señales usás en su lugar? ¿Qué problema agrega que provisionar tarde minutos?
6.4 ¿Cómo cambia todo este problema si elegís el camino A (API gestionada)?

---

## Módulo 7 — Dónde correrlo: serverless-GPU vs endpoints gestionados vs GPU propia

**Teoría.** Si vas por el camino B (modelo propio), hay un espectro de **dónde** correrlo, de más control a menos operación —el mismo patrón build-vs-buy de [AWS](aws.md) (gestionado vs autogestionado), ahora para inferencia—:

- **GPU propia / autogestionada** (instancias EC2 con GPU, o hardware propio): vos instalás vLLM, manejás el escalado, los cold starts, todo. **Máximo control y, a alta utilización sostenida, el menor costo por token.** También el máximo trabajo de operación. Para equipos con volumen alto y capacidad de ops.
- **Endpoints gestionados de modelo** (Amazon **Bedrock**, **SageMaker**, Google **Vertex AI**): el cloud te sirve modelos (open-weight o propios) detrás de un endpoint; vos no tocás la GPU ni vLLM. Menos control, mucha menos operación. Nota: Bedrock & co. también ofrecen modelos *como API* —ahí te acercás de vuelta al camino A, pero dentro de tu cloud/VPC (útil para compliance)—.
- **Serverless-GPU** (Modal, Replicate, RunPod, Baseten…): plataformas que corren tu modelo y **manejan el cold start y el autoscaling** (incluido un scale-to-zero más razonable que a mano). Pagás por uso de GPU. El punto dulce entre control y operación para muchos: traés tu modelo, ellos manejan la infra de GPU.

Y siempre, recordá el camino A: **una API gestionada (Claude) es la opción de "cero operación de modelo".** El espectro completo, de más a menos que operás vos:

| Opción | Operás | Control | Cuándo |
|---|---|---|---|
| GPU autogestionada | Todo (vLLM, escalado, cold start) | Máximo | Volumen alto sostenido + equipo de ops |
| Serverless-GPU (Modal/Replicate) | Tu modelo; ellos la GPU | Alto | Modelo propio sin querer operar GPUs |
| Endpoint gestionado (Bedrock/SageMaker) | Casi nada | Medio | Modelo en tu cloud/VPC, mínima ops |
| API gestionada (Claude) | Nada del modelo | Bajo (sobre el modelo) | El default: la mayoría de los casos |

La regla, otra vez "escalá la herramienta al problema": **bajá en la tabla (hacia menos operación) salvo que un requisito concreto te empuje a subir.** La frase mental: **dónde correr la inferencia es un espectro build-vs-buy —GPU autogestionada (máximo control y ops), serverless-GPU (ellos la GPU, vos el modelo), endpoint gestionado (mínima ops, en tu cloud), API gestionada (cero ops de modelo, el default)— y elegís la de menos operación que cumpla tus requisitos—.**

**Ejercicios 7**
7.1 Ordená las opciones de menos a más operación de tu parte y dá el caso de uso de cada una.
7.2 ¿Qué resuelve una plataforma serverless-GPU (Modal/Replicate) respecto a una GPU autogestionada?
7.3 ¿En qué se diferencia un endpoint gestionado (Bedrock/SageMaker) de la API gestionada de Claude, y cuándo el primero ayuda con compliance?
7.4 ¿Cuál es la regla para elegir en este espectro y con qué criterio del temario conecta?

---

## Módulo 8 — Ingeniería de costos: la métrica que manda en IA

**Teoría.** En un backend tradicional el costo es un tema de infra; en IA **el costo es protagonista del diseño**, porque cada request puede costar mucho y escala con el uso. Un AI Engineer que no controla el costo construye sistemas que no llegan a producción. Las palancas, casi todas ya vistas en el track, ahora como disciplina de deploy:

- **El crossover API vs self-host.** Una **API** cuesta **por token, con cero costo ocioso** (no usás, no pagás; escala a cero). Una **GPU propia** cuesta **fijo 24/7** sin importar el uso. Entonces self-host **solo sale más barato a partir de un volumen alto y sostenido** que mantenga la GPU bien utilizada; por debajo de ese punto de cruce, la API es más barata *y* más simple. Calcular ese crossover (tu volumen × precio por token vs costo de la GPU/hora × utilización) es la cuenta que justifica —o no— el camino B.
- **Model tiering.** No uses Opus 4.8 ($5/$25 por 1M) para lo que Haiku ($1/$5) resuelve igual ([LLMs](ia-llms.md), [Prompt Engineering](prompt-engineering.md)). Enrutá: tareas simples al modelo barato, difíciles al caro. La eval te dice el modelo más barato que pasa el umbral.
- **Caching.** El **prompt caching** ([Prompt Engineering](prompt-engineering.md)): prefijos estables (system + ejemplos + contexto) se **leen** a ~0.1×, pero **escribir** la caché cuesta ~1.25× (TTL 5 min) o ~2× (1 h) —así que recién rinde a partir de ~2 lecturas del mismo prefijo, no con una request suelta—. Y el **semantic caching**: si una pregunta es casi idéntica a una ya respondida (cercanía de embeddings, [Vector Databases](vector-dbs.md)), devolvés la respuesta cacheada sin llamar al LLM. Ambos recortan costo y latencia.
- **Batch API.** Para trabajo no interactivo, muchas APIs ofrecen un modo **batch** con descuento fuerte (del orden del 50%) a cambio de latencia (procesa cuando puede). Ideal para el ingest/procesamiento offline (módulo 2).
- **Token budgets y límites.** Acotás `max_tokens`, recortás el contexto al presupuesto ([RAG](rag.md) módulo 8), y ponés límites por usuario para que un abuso no te vacíe la cuenta. Y en conversaciones o agentes largos, **compactar el contexto** (resumir o recortar la historia que ya no aporta) frena el crecimiento del costo por turno —cada token de historia se re-paga en **cada** llamada—.

La disciplina, el "medí antes de optimizar" de [observabilidad](observabilidad.md): **medí el costo por request / por usuario / por feature, y optimizá la palanca que más pesa** —no caches todo a ciegas ni saltes a self-host sin la cuenta del crossover—. La frase mental: **en IA el costo es un problema de diseño, no de infra: la API cuesta por token sin ocio y el self-host cuesta fijo 24/7 (hay un crossover de volumen), y lo bajás con tiering, prompt/semantic caching, batch API y token budgets —medido, atacando la palanca que más pesa—.**

**Ejercicios 8**
8.1 ¿Por qué el costo es protagonista en IA y no un tema secundario de infra?
8.2 Explicá el crossover API vs self-host: ¿por qué la API gana por debajo de cierto volumen y el self-host por encima?
8.3 Nombrá tres palancas para bajar el costo de un sistema que llama a una API, y qué módulo del track desarrolla cada una.
8.4 ¿Qué es la Batch API y para qué tipo de trabajo conviene?

---

## Módulo 9 — Observabilidad de IA: los tres pilares + tokens, costo y calidad

**Teoría.** Toda la [observabilidad](observabilidad.md) genérica aplica —logs estructurados con correlación, métricas (RED, golden signals), tracing distribuido, OTel, SLOs—. Un sistema de IA es un sistema distribuido y necesita todo eso. Pero agrega **señales propias** que, sin ellas, estás operando a ciegas:

- **Uso de tokens** (input/output por request): es a la vez una **métrica de capacidad** y el **driver del costo** (módulo 8). La logueás y la agregás por usuario/feature.
- **Costo por request / por usuario / por feature**: derivado de los tokens, monitoreado como un golden signal de IA. Una feature que se dispara en costo se ve acá.
- **Latencia desglosada**: TTFT y TPOT (módulo 5), no solo "latencia total". El p95/p99 de TTFT te dice si la experiencia interactiva se degrada. Y en sistemas con tool-use ([AI Agents](ai-agents-python.md)), instrumentá **spans por paso del agente** (cada tool-use, cada llamada al LLM): un número agregado esconde en qué step se va el costo y la latencia.
- **Cache hit rate** (prompt y semantic, módulo 8): si cae, el costo sube; es una métrica a vigilar.
- **Calidad en producción (online evals)**: lo más específico de IA. La salida es no determinista, así que no alcanza con que "no haya errores 500" —la respuesta puede ser un `200 OK` y ser **mala**—. Muestreás un porcentaje del tráfico real y lo evaluás (con **LLM-as-judge** y/o feedback de usuarios), midiendo en vivo lo que en [evals](evals.md) medías offline. Es la extensión natural de las evals al runtime.
- **Versiones de modelo y prompt**: registrás qué versión de prompt y qué modelo atendió cada request, para correlacionar un cambio de calidad con un cambio de prompt/modelo (un deploy de prompt es un deploy).
- **Disparos de guardarraíles**: cuántas veces se activó un filtro de seguridad / moderación / validación.

Un cuidado que viene directo de [observabilidad](observabilidad.md) módulo 10 (cardinalidad): **no metas el prompt completo ni el `user_id` como label de una métrica** —cardinalidad explosiva—; esos van en logs/traces, no en las dimensiones de una métrica. La frase mental: **a los tres pilares genéricos, la IA suma señales propias: tokens y costo por request, latencia TTFT/TPOT, cache hit rate, calidad en producción vía online evals (muestrear y juzgar la salida, porque un 200 puede ser una respuesta mala), y versiones de prompt/modelo —porque un cambio de prompt es un deploy que puede degradar la calidad sin tirar ningún error—.**

**Ejercicios 9**
9.1 ¿Qué de la observabilidad genérica aplica a un sistema de IA?
9.2 Nombrá cuatro señales específicas de IA que agregás a los tres pilares y qué te dice cada una.
9.3 ¿Por qué "no hay errores 500" no alcanza para saber si un sistema de IA anda bien, y qué son las online evals?
9.4 ¿Por qué registrás la versión de prompt/modelo por request, y qué cuidado de cardinalidad tenés con los tokens/prompts?

---

## Módulo 10 — Confiabilidad: outages, fallbacks, rate limits y degradación

**Teoría.** Un sistema de IA en producción depende de piezas que **fallan de formas propias**, y la confiabilidad ([Docker](docker-deploy.md) healthchecks, [AWS](aws.md) resiliencia) se extiende con defensas específicas:

- **El proveedor se cae o se degrada.** Si tu LLM es una API externa, su outage es tu outage. Defensa: **fallback multi-modelo / multi-proveedor** —si Claude no responde, caés a otro modelo (o a una versión más chica, o a una respuesta cacheada)—. Diseñá el cliente para que el modelo sea **configurable**, no hardcodeado.
- **Rate limits (`429`).** La API tiene límites por minuto; bajo carga los vas a tocar. Lo primero, antes de escribir una línea: **el SDK `anthropic` ya reintenta 429 y 5xx solo**, con backoff y respetando el header `retry-after` (`max_retries=2` por defecto, configurable) —no lo reimplementes a ciegas—. Para el caso común subís `max_retries` y listo. El bucle manual es solo para **control fino** (encolar, idempotencia, métricas propias), y entonces **desactivás el retry del SDK** (`max_retries=0`) para no apilar reintento-sobre-reintento (5 manuales × 3 del SDK = 15 llamadas reales). Las defensas al gestionarlo vos: respetar **`retry-after`** cuando viene, **backoff exponencial con jitter** (sin jitter, todos los clientes reintentan en los mismos instantes → *thundering herd*), y una **cola** que suavice los picos.

```python
import anthropic, random, time

# Caso común: dejá que el SDK reintente solo (respeta retry-after y hace backoff)
client = anthropic.Anthropic(max_retries=5)
client.messages.create(model="claude-opus-4-8", max_tokens=1024, messages=[...])

# Control fino (cola, idempotencia): gestionás vos → desactivá el retry del SDK
# (max_retries=0) para no apilarlos
client_manual = anthropic.Anthropic(max_retries=0)

def llamar_con_reintentos(client, intentos=5, **kwargs):
    for i in range(intentos):
        try:
            return client.messages.create(**kwargs)
        except anthropic.RateLimitError as e:           # 429
            if i == intentos - 1:
                raise
            espera = e.response.headers.get("retry-after")   # respetá retry-after si viene;
            time.sleep(float(espera) if espera             # si no, backoff exponencial + jitter
                       else 2 ** i + random.uniform(0, 1))
        except anthropic.APIStatusError as e:
            if e.status_code >= 500 and i < intentos - 1:
                time.sleep(2 ** i + random.uniform(0, 1))    # transitorio del servidor: backoff + jitter
            else:
                raise
```

- **Timeouts y respuestas parciales.** Una generación puede colgarse. Poné **timeouts**, y con **streaming** una respuesta parcial es mejor que nada (y mejora el TTFT percibido).
- **Degradación gradual (graceful degradation).** Cuando algo falla, degradá en vez de romper: respuesta cacheada, un modelo más simple, o un mensaje honesto ("no puedo procesar esto ahora"). Mejor un sistema más tonto que uno caído.
- **Idempotencia en los reintentos.** Reintentar una llamada **cara** o con **efectos** (un agente que ejecuta una acción, [AI Agents](ai-agents-python.md)) sin idempotencia puede cobrar dos veces o duplicar la acción. Misma disciplina de idempotencia que las colas ([AWS](aws.md)/[RAG](rag.md) ingest).
- **Guardarraíles en el borde.** Validación de entrada/salida, moderación, límites por usuario —las defensas de [Prompt Engineering](prompt-engineering.md) y [Agentes](agentes.md), ahora como parte del runtime de producción—.

El principio, el de siempre: **defensa en profundidad.** Ninguna capa sola alcanza; apilás fallbacks, reintentos, timeouts, degradación y guardarraíles. La frase mental: **la confiabilidad de IA suma defensas propias: fallback multi-modelo ante outages, backoff + cola ante rate limits, timeouts y respuestas parciales por streaming, degradación gradual en vez de caída, e idempotencia al reintentar llamadas caras o con efectos —defensa en profundidad sobre un servicio que depende de un modelo no determinista y un proveedor externo—.**

**Ejercicios 10**
10.1 ¿Qué hacés para que el outage de tu proveedor de LLM no sea tu outage total?
10.2 ¿Cómo manejás los rate limits (`429`) de la API? (nombrá las dos defensas)
10.3 ¿Qué es la degradación gradual y por qué "un sistema más tonto" puede ser mejor que uno caído?
10.4 ¿Por qué la idempotencia importa al reintentar, y en qué caso de IA es especialmente peligrosa? (pensá en agentes)

---

## Módulo 11 — El criterio de cierre: API vs self-host, dónde, y el cierre del track

**Teoría.** El cierre del módulo y del track, con las decisiones en orden:

**1. ¿API gestionada o servir tu propio modelo?** (módulo 1) La decisión #1, y el default es **API**. Subís a self-host solo por un requisito concreto y nombrable: **privacidad/compliance** (la data no puede salir), **costo a escala** (volumen alto y sostenido que pasa el crossover del módulo 8), **latencia/control**, o un **modelo propio**. Si ninguna se cumple, montar GPUs es complejidad y costo que no necesitás.

**2. Si servís tu propio modelo, ¿dónde?** (módulo 7) El espectro build-vs-buy: GPU autogestionada (máximo control + ops) → serverless-GPU (Modal/Replicate) → endpoint gestionado (Bedrock/SageMaker). Bajá hacia menos operación salvo que un requisito te empuje a subir. Y servilo con **vLLM** (módulo 4): continuous batching + PagedAttention, no serving naïve.

**3. ¿Cómo lo medís y operás?** Costo como métrica de primera clase (módulo 8: tiering, caching, batch), observabilidad de IA (módulo 9: tokens, costo, TTFT/TPOT, online evals), y confiabilidad propia (módulo 10: fallbacks, backoff, degradación). El deploy no termina en "está corriendo": termina en "lo veo, lo mido y aguanta cuando algo falla".

El principio que cierra **todo el track**, una última vez: **la mejor solución es la más simple que resuelve el problema real, y la decisión se mide.** En deploy de IA: "¿de verdad necesito servir mi propio modelo, o una API alcanza? y si sirvo, ¿dónde con la menor operación posible?". Es la misma lógica que recorrió el track entero —LLM ("¿modelo o `if`?"), RAG ("¿muchos docs o pocos?"), vector DBs ("¿dedicada o pgvector?"), agentes ("¿autonomía o control?"), voz ("¿la modalidad importa?"), deploy ("¿API o GPU propia?")—.

**El cierre del track Python AI.** Recorriste el camino completo de un AI Engineer en Python: el [lenguaje](python.md), [servir APIs](fastapi.md), [hablarle al modelo](prompt-engineering.md), [darle tu información a escala](vector-dbs.md), [dejarlo actuar](ai-agents-python.md), [darle voz](voice-ai.md), y ahora **llevarlo a producción con criterio de costo, escala y confiabilidad**. El sistema que arrancó como una llamada simple es hoy un servicio de IA desplegable, observable y económicamente viable. Lo que atraviesa todo y nunca termina: [evals](evals.md) (medir que de verdad funciona) y [observabilidad](observabilidad.md) (verlo en producción). La frase mental: **el deploy de IA se decide en orden —¿API o modelo propio?, ¿dónde con menos operación?, ¿cómo lo mido y lo hago confiable?— con el default puesto en la API gestionada; es el mismo criterio de simplicidad medida de todo el track, aplicado al paso final de poner la IA frente a usuarios reales—.**

**Ejercicios 11**
11.1 ¿Cuál es la decisión #1 del deploy de IA, cuál es el default, y qué la cambia?
11.2 Si servís tu propio modelo, ¿cómo decidís dónde correrlo y con qué servidor de inferencia?
11.3 Más allá de "está corriendo", ¿qué tres cosas incluye operar un sistema de IA en producción?
11.4 Enunciá la pregunta-criterio del deploy de IA y mostrá cómo es la misma lógica de simplicidad medida del resto del track.

---

## Proyecto integrador (capstone): llevar a producción el sistema del track

El ejercicio que cierra el módulo —y el track—: **desplegar de verdad** el agente sobre tus apuntes (capstones de [AI Agents](ai-agents-python.md)/[Vector Databases](vector-dbs.md)). No te damos la solución; te damos los **criterios de aceptación**. Es deploy real, con el camino A (API), que es el sensato para este caso.

**Qué construir.** Exponé el agente RAG detrás de una **API [FastAPI](fastapi.md)** desplegada, con todo lo de producción:

1. **Dockerizá** la app (multi-stage, [Docker](docker-deploy.md)) y desplegala en una plataforma con autoscaling. La **API key** va en el secret manager, no en la imagen.
2. **Streaming**: el endpoint de chat responde token a token (SSE), con el LB configurado para no buffear.
3. **Ingest a cola**: subir/indexar apuntes dispara un background job, no bloquea el request (módulo 2).
4. **Costo + observabilidad**: logueá tokens y costo por request; exponé métricas de TTFT, costo por request y cache hit rate (módulo 9).
5. **Confiabilidad**: reintentos con backoff ante `429`/5xx, timeout, y una degradación honesta si el LLM no responde (módulo 10).

**Criterios de aceptación.**
- [ ] La app corre dockerizada con la API key como secreto (no en la imagen ni el repo).
- [ ] El endpoint de chat **streamea** la respuesta (se ve el progreso, no un spinner).
- [ ] El ingest corre en background; el request de subida responde sin esperar a indexar todo.
- [ ] Se loguean **tokens y costo por request**, y hay al menos una métrica de latencia (TTFT o total p95).
- [ ] Ante un `429` simulado, el cliente reintenta con backoff en vez de fallar de una.
- [ ] Si el LLM no responde (timeout), el usuario recibe un mensaje honesto, no un 500 crudo.

**Extensiones (suben el nivel).**
- Agregá **prompt caching** (prefijo estable) y/o **semantic caching** (cachear por similitud de la pregunta) y medí el ahorro (módulo 8).
- Implementá **fallback multi-modelo** (Claude → un modelo más chico) y probalo apagando el principal (módulo 10).
- Agregá **online evals**: muestreá un % del tráfico y puntualo con LLM-as-judge, graficando la calidad en el tiempo (módulo 9).
- (Avanzado, camino B) Serví un modelo open-weight con **vLLM** en una GPU (o serverless-GPU) y compará costo/latencia contra la API para *tu* volumen — calculá el crossover (módulos 4, 7, 8).

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 (A) Llamar a una API gestionada: el modelo corre en la infra de otro (Claude/Anthropic), tu app
    le pega por HTTP. (B) Servir tu propio modelo: cargás un modelo open-weight en tu GPU y exponés
    tu endpoint. Son distintos porque B agrega GPU, VRAM, servidor de inferencia y el escalado de
    todo eso; A no tiene nada de eso.
1.2 A una dependencia externa, como una base de datos gestionada: tu app es un web service normal
    que hace llamadas de red; no tenés GPU ni pesos ni servidor de inferencia.
1.3 Privacidad/compliance (la data no puede salir), costo a escala (volumen alto y sostenido que
    pasa el crossover), latencia/control (sin saltos a un tercero ni rate limits ajenos), y usar un
    modelo propio (open-weight o fine-tuneado) que ninguna API ofrece.
1.4 Default: llamar a una API. Error de principiante: montar GPUs "porque es lo serio" cuando una API
    resolvía el caso. Es el "escalá la herramienta al problema"/"empezá simple" del temario: subís a
    self-host solo cuando un requisito concreto lo exige.
```

### Módulo 2
```
2.1 Genérico (ya lo sabés): Docker, multi-stage, CI/CD, healthchecks, graceful shutdown, autoscaling.
    Delta de IA: requests largos (streaming/timeouts), concurrencia I/O-bound (async), trabajo pesado
    a colas, manejo de rate limits/reintentos, y el costo como métrica.
2.2 Porque una llamada al LLM tarda segundos; sin streaming el usuario ve un spinner largo. El
    streaming manda la respuesta token a token (SSE/WebSocket) para mostrar progreso, y exige que el
    LB/proxy NO buffere la respuesta y mantenga la conexión abierta.
2.3 Porque pasa la mayor parte del tiempo ESPERANDO la respuesta del LLM (red), no computando. Con
    async, un solo proceso maneja cientos de llamadas concurrentes mientras esperan; la clave es no
    bloquear el event loop, no tener muchos workers.
2.4 El ingest de RAG (chunking + embeddings de muchos docs) y cualquier procesamiento batch: es
    trabajo pesado y no interactivo, así que va a una cola/background job para no bloquear el request
    del usuario.
```

### Módulo 3
```
3.1 Porque un LLM es multiplicación de matrices gigante, y la GPU lo hace masivamente en paralelo
    (miles de núcleos); una CPU lo haría en serie y tardaría una eternidad.
3.2 7B en fp16 (2 bytes/param) ≈ 14 GB; en int8 (~1 byte) ≈ 7 GB; en int4 (~0.5 byte) ≈ 3.5 GB. Al
    cuantizar se sacrifica algo de calidad/precisión del modelo a cambio de menos VRAM.
3.3 El KV cache son los estados intermedios (claves/valores de atención) de todos los tokens del
    contexto, que el modelo guarda durante la generación. Crece con la longitud de contexto y la
    concurrencia, así que "¿entra?" es pesos + KV cache de todas las requests en vuelo < VRAM, no
    solo los pesos.
3.4 Porque una instancia con GPU se paga por hora esté o no procesando (caras y escasas); una GPU
    ociosa quema plata igual. (Lo desarrolla el módulo 6.)
```

### Módulo 4
```
4.1 Porque sin un buen servidor de inferencia la GPU queda subutilizada. El serving naïve (transformers,
    un request a la vez) procesa de a uno: la GPU, que puede paralelizar miles de operaciones, pasa la
    mayor parte del tiempo esperando → throughput pésimo.
4.2 Continuous batching mete y saca requests del batch en CADA paso de generación: cuando una secuencia
    termina, libera el lugar y entra otra, sin esperar. El static batching arma un lote fijo y hace que
    las requests rápidas esperen a las lentas; el continuo mantiene la GPU siempre llena.
4.3 La fragmentación/desperdicio de VRAM del KV cache (que tradicionalmente se reserva en bloques
    contiguos rígidos). PagedAttention lo gestiona como la memoria virtual paginada de un SO: páginas
    no contiguas asignadas bajo demanda → menos desperdicio → batches más grandes → más throughput.
4.4 Que tu app cambia entre la API gestionada y tu propio vLLM tocando casi solo la URL base (misma
    interfaz tipo OpenAI): portabilidad, evitás reescribir el cliente.
```

### Módulo 5
```
5.1 Prefill: procesa el prompt de entrada en paralelo (rápido), produce el primer token. Decode:
    genera los tokens de salida uno por uno, secuencial (cada uno depende del anterior). En respuestas
    largas domina el decode, porque son muchos pasos secuenciales.
5.2 TTFT = tiempo hasta el primer token (≈ prefill + cola). TPOT = tiempo por cada token siguiente
    (fluidez del streaming). Throughput = tokens/seg agregados de todas las requests (capacidad/costo).
    El usuario percibe el TTFT como "cuánto tarda en empezar".
5.3 Más requests en el batch = más tokens/seg en total (mejor throughput, menor costo por token) pero
    cada request individual espera más (peor TTFT/TPOT). Subir throughput cuesta latencia y viceversa.
5.4 Interactivo (chat/voz): priorizás latencia (TTFT/TPOT bajos), aceptás menos throughput. Batch
    offline: priorizás throughput (tokens/seg, costo); la latencia por request no importa.
```

### Módulo 6
```
6.1 Porque las GPUs son caras y se pagan 24/7 (una réplica ociosa quema plata), el scale-to-zero choca
    con cold starts de minutos, y el autoscaling reacciona tarde porque provisionar una GPU tarda
    minutos. En un web service CPU las réplicas son baratas y escalás agresivo.
6.2 Porque prender una GPU de nuevo tarda minutos: provisionar la instancia (GPUs escasas), bajar la
    imagen y cargar los pesos (decenas de GB) en VRAM. El primer usuario tras el apagado esperaría
    minutos, inaceptable para tráfico interactivo.
6.3 Porque la GPU, no la CPU, es el recurso saturado; escalás por utilización de GPU, profundidad de
    cola o TTFT subiendo. Que provisionar tarde minutos implica que el autoscaling reacciona tarde:
    tenés que anticipar la demanda o mantener un colchón.
6.4 Desaparece como problema tuyo: el proveedor (Anthropic) escala el modelo; vos escalás solo tu web
    service liviano, que sí hace scale-to-zero barato. El escalado difícil de inferencia es un costo
    del camino B.
```

### Módulo 7
```
7.1 De menos a más operación: API gestionada (Claude — cero ops de modelo, el default) → endpoint
    gestionado (Bedrock/SageMaker — mínima ops, modelo en tu cloud/VPC) → serverless-GPU
    (Modal/Replicate — ellos la GPU, vos el modelo) → GPU autogestionada (todo vos, máximo control,
    volumen alto + equipo de ops).
7.2 El cold start y el autoscaling de la GPU (incluido un scale-to-zero más razonable): traés tu
    modelo y ellos manejan la infra de GPU, sin que instales vLLM ni gestiones escalado a mano.
7.3 Un endpoint gestionado te sirve un modelo (open-weight/propio) detrás de un endpoint en TU
    cloud/VPC, sin tocar GPU; la API de Claude es el modelo de Anthropic en su infra. El endpoint
    gestionado (o Bedrock como API dentro de tu VPC) ayuda con compliance porque la data se queda en
    tu nube.
7.4 Bajá hacia menos operación salvo que un requisito concreto te empuje a subir. Es el build-vs-buy /
    "escalá la herramienta al problema" del temario (AWS gestionado vs autogestionado).
```

### Módulo 8
```
8.1 Porque cada request puede costar mucho y el costo escala con el uso; un sistema cuyo costo por
    request no cierra no llega a producción. El costo es una restricción de diseño, no un detalle de
    infra.
8.2 La API cuesta por token con cero costo ocioso (escala a cero); la GPU propia cuesta fijo 24/7
    aunque no la uses. Por debajo de cierto volumen la API es más barata (no pagás ocio); por encima
    de ese crossover, la GPU bien utilizada sale más barata por token. El cruce depende de tu volumen
    × precio por token vs costo de GPU/hora × utilización.
8.3 (Tres de) model tiering (modelo barato para lo simple — LLMs/Prompt Engineering); prompt caching
    (prefijo estable a ~0.1× — Prompt Engineering); semantic caching (cachear por similitud de la
    pregunta — Vector Databases); Batch API; token budgets (RAG módulo 8).
8.4 Un modo de la API que procesa requests no interactivos con descuento fuerte (~50%) a cambio de
    latencia (cuando puede). Conviene para trabajo offline/batch como el ingest o procesamiento masivo.
```

### Módulo 9
```
9.1 Todo: logs estructurados con correlación, métricas (RED/golden signals), tracing distribuido,
    OpenTelemetry y SLOs. Un sistema de IA es un sistema distribuido y necesita los tres pilares.
9.2 (Cuatro de) tokens por request (capacidad + driver de costo); costo por request/usuario/feature;
    latencia TTFT/TPOT (no solo total); cache hit rate (prompt/semantic); versión de prompt/modelo por
    request; disparos de guardarraíles. Cada una expone una dimensión que las métricas genéricas no ven.
9.3 Porque la salida es no determinista: una respuesta puede ser 200 OK y aun así ser mala, y eso no
    lo detecta un error rate. Online evals = muestrear un % del tráfico real y evaluarlo en vivo (con
    LLM-as-judge y/o feedback de usuarios), llevando la disciplina de evals al runtime.
9.4 Para correlacionar un cambio de calidad con un cambio de prompt/modelo (un deploy de prompt es un
    deploy). Cuidado de cardinalidad: no metas el prompt completo ni el user_id como label de una
    métrica (explosión de cardinalidad); van en logs/traces, no en dimensiones de métricas.
```

### Módulo 10
```
10.1 Fallback multi-modelo / multi-proveedor: si Claude no responde, caés a otro modelo (o uno más
     chico, o una respuesta cacheada). Diseñás el cliente con el modelo configurable, no hardcodeado.
10.2 Lo primero: el SDK anthropic ya reintenta 429/5xx solo (max_retries, respeta retry-after);
     subís el límite y alcanza para el caso común. Si gestionás a mano (cola, idempotencia),
     desactivás el del SDK (max_retries=0) y aplicás: respetar retry-after si viene, backoff
     exponencial con jitter (para no provocar thundering herd) y una cola que suavice los picos.
10.3 Degradar en vez de romper: ante un fallo, devolver una respuesta cacheada, usar un modelo más
     simple, o un mensaje honesto ("no puedo procesar esto ahora"). Un sistema más tonto pero
     disponible es mejor que uno caído.
10.4 Porque reintentar una llamada cara o con efectos sin idempotencia puede cobrar dos veces o
     duplicar la acción. Es especialmente peligroso en un agente que EJECUTA acciones (AI Agents): un
     reintento podría re-ejecutar un borrado/cobro. Misma idempotencia que las colas.
```

### Módulo 11
```
11.1 La #1 es ¿API gestionada o servir tu propio modelo? El default es API. La cambia un requisito
     concreto: privacidad/compliance, costo a escala (crossover), latencia/control, o un modelo propio.
11.2 Dónde: el espectro build-vs-buy (GPU autogestionada → serverless-GPU → endpoint gestionado),
     bajando hacia menos operación salvo que un requisito te empuje a subir. Con qué: vLLM (continuous
     batching + PagedAttention), no serving naïve.
11.3 Costo como métrica de primera clase (tiering, caching, batch), observabilidad de IA (tokens,
     costo, TTFT/TPOT, online evals) y confiabilidad propia (fallbacks, backoff, degradación). El
     deploy termina en "lo veo, lo mido y aguanta cuando algo falla", no en "está corriendo".
11.4 "¿De verdad necesito servir mi propio modelo o una API alcanza? y si sirvo, ¿dónde con la menor
     operación posible?" Es la misma lógica de simplicidad medida del track: LLM (¿modelo o if?), RAG
     (¿muchos docs o pocos?), vector DBs (¿dedicada o pgvector?), agentes (¿autonomía o control?), voz
     (¿la modalidad importa?), deploy (¿API o GPU propia?). Siempre: lo más simple que resuelve el
     problema real, medido.
```

---

## Siguientes pasos

Con este módulo cerrás el track Python AI: la bifurcación entre llamar a una API gestionada (el default) y servir tu propio modelo, cómo desplegar la app que llama a una API (un web service normal con sus deltas de IA), los fundamentos de GPU/VRAM y cuantización, el servidor de inferencia (vLLM, continuous batching, PagedAttention), las métricas de serving (TTFT/TPOT/throughput y el trade-off del batching), por qué escalar inferencia es difícil, dónde correrla (serverless-GPU vs endpoints gestionados vs GPU propia), la ingeniería de costos (el crossover, tiering, caching, batch), la observabilidad de IA (tokens, costo, online evals) y la confiabilidad (fallbacks, rate limits, degradación). Con la regla de siempre: **la solución más simple que resuelve el problema real, medida —y en deploy, el default es la API, el self-host se gana con un requisito concreto—.** Con esto el track Python AI queda **completo**: del [lenguaje](python.md) y las [APIs](fastapi.md), pasando por [prompting](prompt-engineering.md), [retrieval a escala](vector-dbs.md), [agentes](ai-agents-python.md) y [voz](voice-ai.md), hasta **producción**. Lo que nunca termina y atraviesa todo: [evals](evals.md) para saber que funciona, y [observabilidad](observabilidad.md) para verlo correr. El sistema que arrancó como una llamada simple es hoy un servicio de IA completo, desplegado, medido y económicamente viable —el portfolio coherente de un AI Engineer en Python—.
