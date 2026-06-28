# Datos en movimiento: streams, batch y analytics a fondo

**Kafka por dentro, event-time vs processing-time, watermarks, exactly-once y OLTP vs OLAP · Node + Go · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **M3** del roadmap profundo de system design (M1 [Consenso](consenso.md), M2 [Replicación](replicacion.md)). Trata los **datos en movimiento**: cómo Kafka ordena y reparte un flujo infinito, qué es el *tiempo* en un stream (y por qué hay dos), cómo se cierran ventanas con watermarks, qué significa exactly-once de verdad, y por qué las consultas analíticas viven en un mundo columnar aparte. Su complemento es el [laboratorio práctico](streams-practica.md): acá entendés, allá **construís** (particiones y orden, offsets con at-least/at-most-once, ventanas por event-time con un evento tardío, reprocesamiento estilo Kappa, y el costo de scan OLTP vs OLAP).

**Lo que asumimos.** El método de [system-design](system-design.md), y que viste [Event-driven](event-driven.md) (colas, idempotencia, **exactly-once processing vs delivery**, outbox/CDC — los reusamos sin re-explicarlos) y [Redis](redis.md) (BullMQ, DLQ). De [Consenso](consenso.md) reusamos la idea de **orden** (un log es un orden total por partición). No asumimos Go: los ejemplos se explican línea por línea.

> ⚠️ **Sobre los lenguajes.** El procesamiento de streams es **lógica de flujo**. Los ejemplos van en **Node/TS** (tu stack) y en **Go** donde varios consumidores concurrentes se ven más claros. La idea importa más que el runtime.

### El marco que usamos en todo el módulo

Igual que en M1/M2: **cuatro capas** ante fallas —(1) qué se rompe, (2) por qué a *esta* escala/forma de tráfico, (3) control de corto plazo, (4) rediseño— y **las tres cosas** al diseñar: happy path, modelo de falla, recuperación.

> **El ejemplo que fija el tono — "el dashboard mostró menos de lo real".** Respuesta débil: *"habrá un delay"*. Respuesta profunda, en cascada: **un evento de las 10:00 viaja por una red móvil y llega a las 10:05 → la ventana de "ventas de 10:00 a 10:01" ya se cerró por *processing-time* a las 10:01 → ese evento cayó fuera → el total de esa ventana quedó bajo, para siempre**. No es un delay: es haber usado el reloj equivocado (processing-time en vez de event-time) y no haber esperado con un watermark. Lo arreglás en los módulos 7-10.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (el grueso va al [laboratorio](streams-practica.md)).

**Índice de módulos**

*Etapa 1 — El cambio de mentalidad*
1. Batch vs stream: datos en reposo vs en movimiento
2. Por qué procesar en stream (y qué se complica)

*Etapa 2 — Kafka por dentro (el log particionado)*
3. El log de commits como abstracción
4. Particiones y orden (la garantía que sí y la que no)
5. Consumer groups, offsets y rebalancing
6. Garantías de entrega: at-most / at-least / exactly-once

*Etapa 3 — El tiempo en streams*
7. Event-time vs processing-time
8. Watermarks: decidir "ya vi todo hasta T"
9. Windowing: tumbling, sliding, session
10. Eventos tardíos y desordenados

*Etapa 4 — Correctitud en streams*
11. Exactly-once en procesamiento (checkpoints y estado)
12. Stateful stream processing

*Etapa 5 — Arquitecturas analíticas*
13. OLTP vs OLAP (fila vs columna)
14. Warehouse, lake, lakehouse (ETL vs ELT)
15. Lambda vs Kappa

*Etapa 6 — Cierre*
16. Cómo hablar de streams/analytics en una entrevista

Las soluciones de **todos** los ejercicios están al final.

---

## Etapa 1 — El cambio de mentalidad

## Módulo 1 — Batch vs stream: datos en reposo vs en movimiento

**Teoría.** Hay dos formas de procesar datos, y la diferencia no es la herramienta, es **el modelo mental**:

- **Batch:** los datos están **en reposo** (un archivo, una tabla, un dump de ayer). Tienen **principio y fin**. Procesás "todo lo de ayer" de una corrida, y cuando termina, terminó. Throughput altísimo, latencia alta (esperás a juntar el lote). Es Hadoop/Spark/un cron con un `SELECT` gigante.
- **Stream:** los datos están **en movimiento** — un flujo **sin fin** de eventos que llegan uno tras otro (clicks, pagos, mediciones de sensores). No hay "fin del dataset": procesás cada evento (o ventanas chiquitas) **a medida que llega**. Latencia baja (segundos o menos), pero tenés que razonar sobre un infinito.

> El salto conceptual que hay que internalizar: **un stream es un dataset que nunca termina.** Eso cambia todo. En batch podés preguntar "¿cuál es el total?" porque hay un fin. En un stream, "el total" no existe — existe "el total **hasta ahora**" o "el total **de esta ventana de tiempo**". Casi todo lo difícil de streams (ventanas, watermarks, event-time) sale de esta sola cosa: **no hay un final donde pararte a contar.**

Una idea que une los dos mundos (y reaparece en Kappa, módulo 15): **un batch es un caso particular de un stream acotado**. Si tratás "el archivo de ayer" como "un stream que empieza en el offset 0 y termina en el último registro", el mismo motor sirve para los dos. Por eso los sistemas modernos (Flink, Spark Structured Streaming) unifican batch y stream bajo la misma API.

**Ejercicios 1**
1.1 🔁 ¿Cuál es la diferencia de modelo mental entre batch y stream (no la herramienta)?
1.2 🧠 ¿Por qué en un stream "¿cuál es el total?" es una pregunta mal planteada, y cómo la reformulás?
1.3 🧠 Explicá en qué sentido un batch es un caso particular de un stream.

---

## Módulo 2 — Por qué procesar en stream (y qué se complica)

**Teoría.** Pasás a stream por dos razones reales:

1. **Latencia:** querés reaccionar **ahora**, no mañana. Detección de fraude, alertas, un feed que se actualiza, un dashboard en vivo. Esperar el batch nocturno no sirve.
2. **Datos verdaderamente infinitos:** telemetría, logs, IoT, clickstream — nunca hay un "fin" para correr un batch.

Lo que se complica respecto de batch (y es el temario del módulo):

- **El tiempo se vuelve ambiguo:** en batch, "lo de ayer" es claro. En stream, ¿"lo de ayer" según cuándo **pasó** el evento o cuándo lo **procesaste**? (módulo 7).
- **Desorden:** los eventos no llegan en orden — la red, los reintentos y las particiones los reordenan. Un evento de las 10:00 puede llegar después de uno de las 10:05.
- **No podés esperar al fin:** tenés que decidir *cuándo* emitir un resultado parcial sin haber visto "todo" (watermarks, módulo 8).
- **Fallas a mitad de un infinito:** si el procesador se cae, ¿desde dónde retoma sin perder ni duplicar? (offsets y exactly-once, módulos 6 y 11).

> La frase mental: **batch te deja la comodidad de un dataset finito y ordenado; stream te quita las dos cosas (infinito y desordenado) a cambio de latencia baja.** Todo el resto del módulo es recuperar, una a una, las garantías que el batch te daba gratis.

**Ejercicios 2**
2.1 🔁 Dá las dos razones para procesar en stream en vez de batch.
2.2 🧠 Nombrá tres cosas que se complican en stream y no existen en batch.
2.3 🧠 ¿Por qué "no poder esperar al fin del dataset" es el problema raíz del que salen ventanas y watermarks?

---

## Etapa 2 — Kafka por dentro (el log particionado)

## Módulo 3 — El log de commits como abstracción

**Teoría.** Kafka (y Pulsar, Kinesis, Redpanda) no es "una cola". Es un **log de commits distribuido**: una secuencia **append-only** de mensajes, **inmutable y ordenada**, que los consumidores leen a su ritmo. Tres diferencias con una cola tradicional (RabbitMQ/SQS) que tenés que tener clarísimas:

1. **El mensaje no se borra al leerlo.** En una cola, consumir = sacar. En Kafka, el mensaje **queda** en el log; se borra por **retención** (por tiempo: "7 días", o por tamaño), no por consumo. Eso permite **releer** (reprocesar desde un offset viejo) y que **varios** consumidores independientes lean el mismo log.
2. **El consumidor lleva su posición (offset).** Kafka no trackea "qué entregó"; cada consumidor recuerda **hasta qué offset leyó**. Avanzar o retroceder es mover un número.
3. **Es un registro de eventos, no una bandeja de tareas.** Un mismo evento (`PagoConfirmado`) lo consumen el servicio de envíos, el de facturación y el de analytics, cada uno con su offset. El log es **la fuente de la verdad** del flujo.

> Tipos de retención: por **tiempo/tamaño** (el default: borra lo viejo) o **log compaction** (conserva **el último valor por clave**, descartando versiones viejas — útil para "el estado actual de cada entidad", como un changelog). La compaction es la base del *event sourcing* y de los changelog topics (módulo 12).

La abstracción "log inmutable y releíble" es lo que hace a Kafka el **sistema nervioso** de las arquitecturas event-driven: desacopla productores de consumidores en el tiempo (el consumidor puede estar caído una hora y ponerse al día después) y permite reprocesar (Kappa, módulo 15).

**Ejercicios 3**
3.1 🔁 ¿Por qué en Kafka un mensaje no desaparece al consumirlo, y qué lo borra?
3.2 🧠 ¿Qué dos cosas te habilita que el log sea releíble e inmutable, que una cola tradicional no?
3.3 🧠 ¿Qué conserva el log compaction y para qué caso de uso sirve?

---

## Módulo 4 — Particiones y orden (la garantía que sí y la que no)

**Teoría.** Un *topic* se parte en **particiones**: cada partición es un log independiente, y las particiones se reparten entre brokers. Particionar es lo que da **escala** (paralelismo: más particiones = más consumidores en paralelo) y es donde vive **la garantía de orden más malentendida del sistema**:

> **Kafka garantiza orden DENTRO de una partición, NO entre particiones.** Los mensajes de una partición se leen exactamente en el orden en que se escribieron (por offset). Pero dos mensajes en **particiones distintas** no tienen orden relativo garantizado.

¿Quién decide la partición de un mensaje? La **partition key**: `partición = hash(key) % numParticiones`. Misma key → misma partición → **orden garantizado entre esos mensajes**. Sin key → round-robin (reparte parejo, pero sin orden).

Eso te da la herramienta de diseño central: **elegí la key para que las cosas que necesitan orden caigan en la misma partición.** ¿Querés que todos los eventos de un usuario se procesen en orden? Key = `userId`. ¿De una cuenta? Key = `accountId`.

> 🔥 **Falla en 4 capas — eventos del mismo agregado procesados fuera de orden.**
> 1. **Qué se rompe:** `CuentaCreada` y `CuentaActualizada` del mismo usuario caen en particiones distintas; el consumidor procesa el update antes que el create → error o estado corrupto.
> 2. **Por qué a esta escala:** sin key (o con key mal elegida), el round-robin los dispersa; con varios consumidores en paralelo, cada partición avanza a su ritmo → el orden global se pierde.
> 3. **Control de corto plazo:** tolerar el desorden en el consumidor (buffer + reordenar por versión/timestamp; descartar updates de entidades que aún no existen y reintentar).
> 4. **Largo plazo:** **particionar por la entidad** (`key = aggregateId`) para que todo su historial vaya a una partición y se procese en orden. El precio: una key "caliente" (un usuario con muchísimo tráfico) puede sobrecargar **una** partición (hot partition) → revisar la cardinalidad de la key.

**Ejercicios 4**
4.1 🔁 ¿Dónde garantiza Kafka el orden y dónde no?
4.2 🧠 Querés procesar en orden todos los eventos de cada pedido. ¿Qué key elegís y por qué funciona?
4.3 🧠 ¿Qué problema trae elegir una key de baja cardinalidad (pocos valores distintos) o con un valor muy popular?

---

## Módulo 5 — Consumer groups, offsets y rebalancing

**Teoría.** Un **consumer group** es un conjunto de consumidores que se reparten las particiones de un topic para procesar **en paralelo**. La regla de oro:

> **Cada partición es consumida por exactamente UN consumidor del grupo.** Por eso el paralelismo máximo de un grupo = la **cantidad de particiones**. Si tenés 4 particiones y 6 consumidores, **2 quedan ociosos**. Si tenés 4 particiones y 2 consumidores, cada uno toma 2.

Varios grupos distintos leen el **mismo** topic independientemente (cada grupo con sus propios offsets) — así envíos, facturación y analytics consumen lo mismo sin pisarse.

El **offset commit** es cuándo el consumidor le dice a Kafka "ya procesé hasta acá". Acá vive el trade-off de entrega del módulo 6: ¿commiteás **antes** o **después** de procesar? Puede ser automático (cada N ms) o manual (vos controlás cuándo).

> ⚠️ **Dos mecanismos distintos de expulsión** (pregunta clásica de entrevista: *"¿qué echa al consumidor, el heartbeat o el poll lento?"*): el **heartbeat** corre en un **thread aparte** y, si falla por `session.timeout.ms`, detecta que el **proceso murió**; `max.poll.interval.ms` detecta que el **procesamiento es lento** (tardaste demasiado entre dos `poll()`) **aunque el heartbeat siga latiendo**. Son ortogonales: un consumidor puede estar vivo (heartbeat OK) pero ser expulsado por procesar lento, o caerse de golpe y que lo eche el heartbeat.

El **rebalancing** es lo que pasa cuando un consumidor entra o sale del grupo (se cae, escalás, deploy): Kafka **redistribuye** las particiones entre los consumidores vivos. Es necesario, pero es un punto de dolor:

> 🔥 **Falla en 4 capas — el rebalanceo que congela el consumo (stop-the-world).**
> 1. **Qué se rompe:** durante un rebalanceo clásico (*eager*), **todos** los consumidores sueltan **todas** sus particiones y esperan la nueva asignación → el grupo entero deja de procesar por unos segundos.
> 2. **Por qué a esta escala:** con muchos consumidores y deploys frecuentes (cada deploy = consumidores que entran/salen), los rebalanceos son constantes; si además el procesamiento por mensaje es lento, un consumidor lento dispara el `max.poll.interval` y lo expulsan → otro rebalanceo → efecto dominó.
> 3. **Control de corto plazo:** subir `max.poll.interval.ms`, bajar `max.poll.records`, hacer el procesamiento por mensaje más rápido o asíncrono.
> 4. **Largo plazo:** **cooperative/incremental rebalancing** (las particiones no se sueltan todas: solo se mueven las que cambian de dueño, el resto sigue procesando) — el default moderno. Y *static membership* para que un reinicio rápido no dispare rebalanceo.

**Ejercicios 5**
5.1 🔁 ¿Cuántos consumidores de un grupo pueden leer una misma partición a la vez? ¿Qué pasa si hay más consumidores que particiones?
5.2 🧠 ¿Por qué dos consumer groups distintos pueden leer el mismo topic sin interferir?
5.3 🧠 Explicá por qué un procesamiento lento por mensaje puede disparar un ciclo de rebalanceos en cascada.

---

## Módulo 6 — Garantías de entrega: at-most / at-least / exactly-once

**Teoría.** ¿Qué pasa con un mensaje si el consumidor se cae a mitad de procesarlo? Depende de **cuándo commiteás el offset**:

- **At-most-once (commit ANTES de procesar):** marcás "leído", después procesás. Si te caés en el medio, el mensaje queda commiteado pero **no procesado** → **se pierde**. Cero duplicados, pero perdés mensajes. Aceptable solo si perder uno no duele (métricas best-effort).
- **At-least-once (commit DESPUÉS de procesar):** procesás, después commiteás. Si te caés entre procesar y commitear, al reiniciar **reprocesás** el mensaje → **duplicado**. No perdés nada, pero podés procesar dos veces. **Es el default sano** — combinado con idempotencia, es la base de casi todo.
- **Exactly-once:** ni pierde ni duplica el **efecto**. Es lo más caro y lo más malentendido (ver abajo).

> ⚠️ **El matiz que separa al senior — exactly-once DELIVERY no existe, exactly-once PROCESSING sí.** Entregar un mensaje exactamente una vez sobre una red que pierde y reordena es **imposible** en el caso general (es el teorema de los dos generales, ya visto en [event-driven](event-driven.md)). Lo que SÍ se logra es que el **efecto observable** sea como si se hubiera procesado una vez. Dos caminos:
> - **At-least-once + idempotencia:** reprocesar es inofensivo (la operación da el mismo resultado). El 90% de los casos.
> - **Exactly-once de Kafka (EOS):** productor idempotente + **transacciones** que atan "leer del topic A → procesar → escribir al topic B → commitear el offset" en **una sola transacción atómica**. Si algo falla, se aborta todo. Funciona **dentro del mundo Kafka** (Kafka Streams); cruzando a un sistema externo (una DB, un mail) volvés a necesitar idempotencia en el sink.

La respuesta de entrevista: *"exactly-once de punta a punta con efectos externos no existe; uso at-least-once + idempotencia, o las transacciones de Kafka si todo el pipeline vive en Kafka."*

**Ejercicios 6**
6.1 🔁 ¿Qué arriesga commitear el offset ANTES de procesar, y qué arriesga commitearlo DESPUÉS?
6.2 🧠 ¿Por qué "exactly-once delivery" es imposible pero "exactly-once processing" sí se logra?
6.3 🧠 ¿Hasta dónde llega la garantía exactly-once de Kafka y dónde necesitás volver a idempotencia?

---

## Etapa 3 — El tiempo en streams

## Módulo 7 — Event-time vs processing-time

**Teoría.** En un stream hay **dos relojes**, y confundirlos es el error más caro del área:

- **Event-time:** cuándo el evento **ocurrió de verdad** (el timestamp que trae embebido: "este click fue a las 10:00:03").
- **Processing-time:** cuándo el sistema **lo procesó** (la hora del reloj del procesador cuando lo recibió: "lo vi a las 10:00:09").

Entre los dos hay un **desfase** (lag), y nunca es cero: red, colas, reintentos, un celular que estuvo offline y manda sus eventos al reconectar. Ese evento puede llegar **minutos u horas** después de haber ocurrido.

¿Por qué importa? Porque si agrupás por **processing-time**, tus resultados dependen de **cuándo procesaste**, no de cuándo pasaron las cosas → **no son reproducibles** y mienten:

- Si reprocesás el mismo stream mañana (Kappa), los eventos caen en ventanas distintas (otro processing-time) → **otro resultado** para los mismos datos.
- Un evento tardío de las 10:00 que llega a las 10:05 se cuenta en la ventana de las 10:05 → la métrica de las 10:00 queda mal **para siempre**.

> La regla: **para resultados correctos y reproducibles, agrupá por event-time.** Processing-time solo sirve para cosas tipo "throughput del sistema ahora" (que sí son sobre el procesador, no sobre el dominio). El precio de event-time: como los eventos llegan tarde y desordenados, **no sabés cuándo cerrar una ventana** — y de ahí nacen los watermarks (módulo 8).

**Ejercicios 7**
7.1 🔁 Definí event-time y processing-time en una frase cada uno.
7.2 🧠 ¿Por qué agrupar por processing-time hace que reprocesar el mismo stream dé un resultado distinto?
7.3 🧠 ¿Cuándo SÍ tiene sentido usar processing-time?

---

## Módulo 8 — Watermarks: decidir "ya vi todo hasta T"

**Teoría.** Si agrupás por event-time pero los eventos llegan tarde, ¿**cuándo** cerrás la ventana "10:00–10:01" y emitís el total? Si esperás "para siempre" por si llega un rezagado, nunca emitís nada. Si cerrás apenas pasa el minuto de processing-time, perdés a los tardíos. La respuesta es el **watermark**.

Un **watermark** es una marca que el sistema inyecta en el stream y que afirma: **"no espero más eventos con event-time menor que T"**. Cuando el watermark pasa el final de una ventana, esa ventana **se cierra** y emite su resultado.

El watermark es una **heurística**, no una verdad — el sistema lo estima (p. ej. "el event-time máximo que vi, menos un margen de 30s"). Y ahí está el trade-off, que es **el mismo de la detección de fallas** de [consenso módulo 3](consenso.md): completitud vs latencia.

> 🔥 **Falla en 4 capas — watermark mal calibrado.**
> 1. **Qué se rompe:** un watermark **agresivo** (margen chico) cierra las ventanas rápido pero **descarta** los eventos que llegan después → totales bajos. Un watermark **conservador** (margen grande) no pierde eventos pero **demora** cada resultado (el dashboard va "atrasado").
> 2. **Por qué a esta escala:** el desfase event/processing no es uniforme — los usuarios móviles, una región con mala red o un re-envío masivo meten colas largas; un margen fijo o no alcanza (perdés) o sobra (demorás).
> 3. **Control de corto plazo:** ajustar el margen del watermark al p99 real del lag observado.
> 4. **Largo plazo:** **allowed lateness** (mantener el estado de la ventana un rato más tras cerrarla, para actualizar el resultado si llega un tardío) + **side output** (mandar los que llegan aún más tarde a un canal aparte para corregir en batch). Módulo 10.

**Ejercicios 8**
8.1 🔁 ¿Qué afirma un watermark y qué dispara cuando pasa el fin de una ventana?
8.2 🧠 Explicá el trade-off entre un watermark agresivo y uno conservador.
8.3 🧠 ¿Por qué un watermark es una heurística y no una garantía?

---

## Módulo 9 — Windowing: tumbling, sliding, session

**Teoría.** Como no hay "fin del stream", agrupás en **ventanas** de tiempo (por event-time). Tres tipos canónicos:

- **Tumbling (fijas, sin solape):** cortás el tiempo en bloques contiguos de tamaño fijo: `[10:00–10:01)`, `[10:01–10:02)`, … Cada evento cae en **exactamente una** ventana. Para "ventas por minuto", "requests por hora".
- **Sliding / hopping (deslizantes, con solape):** ventanas de tamaño fijo que **avanzan de a un paso menor** que su tamaño: ventana de 5 min que avanza cada 1 min → cada evento cae en **varias** ventanas. Para "promedio móvil de los últimos 5 minutos, actualizado cada minuto".
- **Session (por sesión):** ventanas de **tamaño variable** definidas por **inactividad**: agrupan eventos hasta que pasa un *gap* sin actividad (p. ej. 30 min). Para "sesión de navegación de un usuario", "una llamada". El tamaño lo decide el comportamiento, no un reloj fijo.

| Ventana | Tamaño | Solape | Caso típico |
|---|---|---|---|
| Tumbling | Fijo | No | Métricas por minuto/hora |
| Sliding | Fijo | Sí | Promedio móvil, alertas |
| Session | Variable (gap) | No | Actividad de usuario |

> La elección sale de la pregunta de negocio: ¿"cada bloque por separado"? → tumbling. ¿"los últimos X, actualizado seguido"? → sliding. ¿"agrupar lo que pasó junto, sin un corte fijo"? → session. Y todas se cierran con un watermark (módulo 8): la ventana emite cuando el watermark pasa su fin.

> Nota para una repregunta de staff: el **tipo de ventana** (cuándo *termina*) es ortogonal al **trigger** (cuándo *emitir* un resultado). Además del trigger por watermark, un motor como Flink permite **early firing** (emitir resultados parciales **antes** de que cierre la ventana, p. ej. cada 10s, para un dashboard que no quiere esperar) y **late firing** (re-emitir cuando llega un tardío dentro del allowed lateness del módulo 10). El detalle fino vive en Flink; alcanza con saber que "qué ventana" y "cuándo disparo" se eligen por separado.

**Ejercicios 9**
9.1 🔁 Diferencia entre tumbling y sliding en una frase.
9.2 🧠 ¿Qué define el tamaño de una session window y para qué caso es la indicada?
9.3 🧠 "Promedio de temperatura de los últimos 10 minutos, recalculado cada minuto." ¿Qué ventana usás y por qué?

---

## Módulo 10 — Eventos tardíos y desordenados

**Teoría.** Aunque agrupes por event-time y uses watermark, **siempre** habrá eventos que lleguen **después** de que su ventana cerró (el celular que estuvo dos horas sin señal). ¿Qué hacés con un evento tardío? Tres políticas, de la más simple a la más completa:

1. **Descartar (drop):** lo ignorás. Simple, pero la métrica queda incompleta. Aceptable si los tardíos son raros y la exactitud no es crítica.
2. **Allowed lateness:** mantenés el **estado** de la ventana un tiempo extra tras cerrarla; si llega un tardío dentro de ese margen, **actualizás** el resultado ya emitido (corregís el total). Cuesta memoria (estado que no liberás).
3. **Side output (dead-letter de tardíos):** los que llegan **demasiado** tarde (después del allowed lateness) van a un canal aparte para procesarlos en **batch** y corregir después. Es la idea de Lambda (módulo 15): el stream da una respuesta rápida y aproximada, el batch la corrige.

El desorden (out-of-order) es el primo del tardío: los eventos no vienen ordenados por event-time ni dentro de una partición necesariamente (sí por offset, pero el offset es processing-order). El watermark + las ventanas por event-time **absorben** el desorden hasta el margen del watermark; lo que cae más allá es "tardío".

> La frase mental: **no existe "ya tengo todos los eventos" en un stream — existe "tengo suficientes como para emitir, y un plan para los que lleguen tarde".** Decidir la política de tardíos (drop / corregir / batch aparte) es una decisión de negocio sobre cuánta exactitud vale cuánta complejidad.

**Ejercicios 10**
10.1 🔁 Nombrá las tres políticas para un evento tardío.
10.2 🧠 ¿Qué cuesta `allowed lateness` y por qué no podés tenerlo "para siempre"?
10.3 🧠 ¿Cómo absorbe el desorden la combinación watermark + ventana por event-time, y qué queda afuera?

---

## Etapa 4 — Correctitud en streams

## Módulo 11 — Exactly-once en procesamiento (checkpoints y estado)

**Teoría.** En el módulo 6 vimos exactly-once para "leer-procesar-escribir" de un mensaje. Pero un procesador de streams **stateful** (que mantiene contadores, ventanas, agregados) tiene un problema extra: si se cae, **¿cómo restaura el estado** sin perder ni duplicar las actualizaciones que ya hizo? La respuesta es el **checkpointing**.

Un **checkpoint** es una foto consistente de **(a) el estado del procesador** y **(b) los offsets de entrada** correspondientes, guardada de forma durable, **atómicamente**. La clave es ese "atómicamente": el estado y el offset tienen que reflejar el mismo punto del stream. Cuando el procesador se cae:

1. Restaura el último checkpoint (estado + offsets).
2. **Reprocesa** desde esos offsets.

Como el estado y el offset eran consistentes, reprocesar desde ahí **no duplica** lo ya reflejado en el estado ni pierde lo posterior → el efecto es exactly-once **dentro del procesador**.

> El algoritmo canónico es el **distributed snapshot de Chandy-Lamport** (lo que usa **Flink**): inyecta "barreras" en el stream que viajan con los datos y disparan que cada operador guarde su estado en el mismo punto lógico, sin parar el flujo. **Kafka Streams** lo logra distinto: con sus **transacciones** (el estado vive en un *changelog topic* y se commitea junto con los offsets en una transacción Kafka). Distinto mecanismo, misma garantía: estado + offset atómicos.

Y el límite, otra vez: exactly-once vale **adentro** del pipeline (Kafka→procesador→Kafka). Si escribís a una DB externa, esa escritura tiene que ser **idempotente** (o transaccional con el sink) o el reprocesamiento del checkpoint la duplicará. El checkpoint te salva el estado interno, no el efecto externo.

**Ejercicios 11**
11.1 🔁 ¿Qué dos cosas captura un checkpoint y por qué tienen que ser atómicas entre sí?
11.2 🧠 Explicá por qué restaurar un checkpoint y reprocesar no duplica el estado ya reflejado.
11.3 🧠 ¿Por qué el checkpoint no te garantiza exactly-once contra una DB externa?

---

## Módulo 12 — Stateful stream processing

**Teoría.** Muchos cálculos de stream necesitan **memoria**: un contador por usuario, el total de una ventana, un join de dos streams, "el último valor de cada sensor". Ese es el **estado** del procesador, y manejarlo a escala tiene su técnica:

- **Estado local + particionado:** el estado se guarda **junto al procesador** (no en una DB remota por cada evento — sería lento). Como el stream está particionado por key (módulo 4), cada instancia del procesador maneja **su** subconjunto de keys y **su** porción del estado, en paralelo. (Kafka Streams/Flink guardan ese estado en **RocksDB** embebido, en disco local, para que no tenga que entrar todo en memoria.)
- **Tolerancia a fallas con changelog:** si el estado vive en disco local y la instancia muere, ¿se pierde? No: cada cambio de estado se escribe también a un **changelog topic** en Kafka (compactado, módulo 3). Al reiniciar (o al moverse a otra instancia tras un rebalanceo), el estado se **reconstruye** releyendo el changelog. El estado es efímero; el changelog es la verdad durable.

> El insight de arquitectura: **el estado del stream es, él mismo, un stream** (el changelog de sus cambios). Por eso "estado local rápido + changelog durable en Kafka" es el patrón: tenés la velocidad del acceso local y la durabilidad/recuperación del log replicado. Es la misma dualidad estado↔log de una máquina de estados replicada ([consenso módulo 7](consenso.md)): el estado se reconstruye reproduciendo el log.

**Ejercicios 12**
12.1 🔁 ¿Por qué el estado de un procesador de streams se guarda local y no en una DB remota por evento?
12.2 🧠 Si el estado local se pierde cuando muere la instancia, ¿cómo se recupera sin perder datos?
12.3 🧠 Explicá la frase "el estado del stream es él mismo un stream".

---

## Etapa 5 — Arquitecturas analíticas

## Módulo 13 — OLTP vs OLAP (fila vs columna)

**Teoría.** Las consultas se parten en dos mundos que quieren **layouts de almacenamiento opuestos**:

- **OLTP (Online Transaction Processing):** las operaciones de tu app — "traé el pedido 123", "insertá este usuario". Muchas, chiquitas, **por fila** (tocan todas las columnas de pocas filas). Storage **orientado a filas** (row-store): una fila junta en disco → leés/escribís una fila entera barato. Es Postgres/MySQL ([postgresql](postgresql.md)).
- **OLAP (Online Analytical Processing):** las consultas analíticas — "promedio de ventas por región del último año sobre 500 millones de filas". Pocas, enormes, **por columna** (tocan **pocas** columnas de **muchísimas** filas). Storage **orientado a columnas** (column-store): cada columna junta en disco.

¿Por qué columnar para analytics? Dos razones de peso:

1. **Lee solo lo que necesita:** `SELECT avg(monto) FROM ventas` sobre un column-store lee **solo la columna `monto`**, no las otras 40 columnas de cada fila. En un row-store, leer una columna te obliga a pasar por filas enteras → muchísimo I/O desperdiciado.
2. **Comprime muchísimo mejor:** una columna tiene **valores del mismo tipo y dominio** (todos los `país` son strings parecidos, todos los `monto` números) → run-length encoding, diccionarios y delta comprimen 10× más que una fila heterogénea. Menos bytes en disco = menos I/O = más rápido. Y permite **ejecución vectorizada** (procesar la columna en bloques con SIMD).

> La regla: **no corras analytics pesado sobre tu DB OLTP de producción** (la satura y compite con el tráfico de la app). Mové los datos a un sistema columnar (warehouse) para eso. La separación OLTP/OLAP es la razón de ser de los data warehouses (módulo 14). Lo medís con tus manos en el [laboratorio](streams-practica.md): el mismo agregado lee 1 columna vs 40.

**Ejercicios 13**
13.1 🔁 ¿Qué layout de storage usa OLTP y cuál OLAP, y a qué patrón de consulta responde cada uno?
13.2 🧠 Dá las dos razones por las que un column-store es más rápido para un agregado sobre millones de filas.
13.3 🧠 ¿Por qué no conviene correr una consulta analítica pesada sobre la DB OLTP de producción?

---

## Módulo 14 — Warehouse, lake, lakehouse (ETL vs ELT)

**Teoría.** ¿Dónde viven los datos para analytics? Tres arquetipos, y conviene ubicarlos:

- **Data warehouse:** un sistema **columnar** optimizado para consultas analíticas, con datos **estructurados y modelados** (esquema en estrella: tablas de hechos + dimensiones). Snowflake, BigQuery, Redshift. Caro por consulta, pero rapidísimo y con SQL cómodo.
- **Data lake:** almacenamiento **barato y crudo** (archivos en S3/GCS, típicamente **Parquet** —que es columnar— u ORC). Guardás **todo** sin esquema previo ("schema-on-read"). Flexible y económico, pero "un pantano" sin gobierno: fácil que se vuelva inmanejable.
- **Lakehouse:** lo mejor de los dos — el storage barato del lake **más** una capa de tabla transaccional encima (**Delta Lake, Iceberg, Hudi**) que le da ACID, schema y time-travel a los archivos del lake. Es la tendencia 2026.

Y la decisión de **cuándo transformás**:

- **ETL (Extract-Transform-Load):** transformás **antes** de cargar → en el warehouse entran datos ya limpios y modelados. Clásico, controlado, pero rígido (si querés otra transformación, re-extraés).
- **ELT (Extract-Load-Transform):** cargás el dato **crudo** primero (al lake/warehouse) y transformás **adentro**, con el poder de cómputo del warehouse (dbt es el estándar). Es lo dominante hoy: storage y cómputo son baratos, así que guardás todo crudo y transformás a demanda, pudiendo rehacer transformaciones sin re-extraer.

> El puente con streams: los datos **llegan** al warehouse/lake por un pipeline (a menudo un stream: CDC desde la DB OLTP vía Debezium, o eventos de Kafka). "OLTP → CDC → lake/warehouse → dbt → dashboards" es el pipeline analítico moderno típico. El stream alimenta el analytics; no son mundos separados. Y como ese CDC es **at-least-once** (módulo 6), puede reentregar filas → el **merge/upsert por clave** en el warehouse (un `MERGE`/`INSERT ON CONFLICT` que escribe la última versión de cada fila) es **la idempotencia del sink analítico**: reprocesar el mismo cambio no duplica la fila.

**Ejercicios 14**
14.1 🔁 Diferencia entre data warehouse, data lake y lakehouse en una frase cada uno.
14.2 🧠 ¿Qué cambió para que ELT le ganara a ETL como default?
14.3 🧠 ¿Cómo llegan típicamente los datos de tu DB OLTP al warehouse, y qué rol juega el stream ahí?

---

## Módulo 15 — Lambda vs Kappa

**Teoría.** ¿Cómo combinás "rápido pero aproximado" (stream) con "lento pero exacto" (batch)? Dos arquitecturas, y es una pregunta clásica de diseño:

- **Lambda architecture:** corrés **dos** pipelines en paralelo. El **batch layer** reprocesa **todos** los datos periódicamente y produce la "verdad" exacta (lento, horas). El **speed layer** procesa el stream en vivo y da resultados recientes y aproximados (rápido, segundos). El **serving layer** **mergea** los dos: servís el batch para lo histórico y el speed para lo reciente, hasta que el próximo batch lo absorbe. Problema: **dos bases de código** que tienen que dar el mismo resultado → doble mantenimiento, sutilezas de divergencia.
- **Kappa architecture:** **un solo** pipeline, todo stream. La "verdad" y lo "reciente" salen del mismo motor. Si tenés un bug o querés recalcular, **reprocesás** releyendo el log desde el offset 0 (Kafka guarda todo, módulo 3) con una nueva versión del job, y cuando alcanza el presente, hacés el switch. Una sola base de código. **Requiere** que el log sea reproducible y que proceses por **event-time** (módulo 7) para que reprocesar dé el mismo resultado.

> La tendencia 2026 es **Kappa** (o pipelines unificados batch+stream como Flink/Spark Structured Streaming que difuminan la distinción): una sola base de código, reprocesamiento por replay. Lambda sobrevive donde el batch hace algo que el stream no puede (ML pesado, joins gigantes contra históricos). La pregunta de entrevista —*"¿cómo recalculás si encontrás un bug en tu agregación de hace un mes?"*— se responde con Kappa: **reproceso el log por event-time desde el offset de hace un mes**. Lo construís en el [laboratorio](streams-practica.md).

**Ejercicios 15**
15.1 🔁 ¿Cuál es la diferencia central entre Lambda y Kappa en cantidad de pipelines/bases de código?
15.2 🧠 ¿Qué dos condiciones necesita Kappa para que reprocesar dé el resultado correcto?
15.3 🧠 "Encontraste un bug en la agregación del mes pasado." ¿Cómo lo recalculás con Kappa?

---

## Etapa 6 — Cierre

## Módulo 16 — Cómo hablar de streams/analytics en una entrevista

**Teoría.** Cierre operativo. Ante un tema de streaming/pipeline/analytics, estructurá con **las tres cosas** y, ante fallas, **las cuatro capas**.

**Las tres cosas (ejemplo: "diseñá un pipeline de métricas en vivo"):**
- **(a) Happy path:** "eventos a Kafka particionados por `userId` (orden por usuario); un consumer group procesa por event-time, ventanas tumbling de 1 min cerradas por watermark; el agregado va a un store columnar para los dashboards".
- **(b) Modelo de falla:** "si un consumidor se cae, at-least-once + estado por checkpoint → reprocesa desde el offset sin perder; eventos tardíos con allowed lateness, los muy tardíos a un side output que corrige en batch".
- **(c) Recuperación:** "si hubo un bug en la agregación, reproceso el log por event-time desde el offset afectado (Kappa) y hago el switch; el changelog reconstruye el estado".

**Las muletillas que suenan a que entendiste:**
- *"Kafka ordena por partición, no global → particiono por la entidad que necesita orden."*
- *"Agrupo por event-time, no processing-time, para que reprocesar sea reproducible."*
- *"Exactly-once de punta a punta con efectos externos no existe: at-least-once + idempotencia, o transacciones de Kafka si todo vive en Kafka."*
- *"El watermark es el dial completitud-vs-latencia; los tardíos los trato con allowed lateness o un side output."*
- *"Analytics pesado va a un column-store, no a la DB OLTP de producción."*

> **Cierre.** Un stream es un dataset que **nunca termina y llega desordenado**, y todo el oficio es recuperar las garantías que el batch te daba gratis: **orden** (particionar por la key correcta), **un punto donde contar** (ventanas + watermark por event-time), **no perder ni duplicar ante una caída** (offsets + checkpoints + idempotencia), y **poder recalcular** (un log releíble → Kappa). Y cuando el dato deja de moverse y hay que **analizarlo**, cambia de forma: de filas (OLTP) a columnas (OLAP). El que sabe nombrar **cuál garantía** está recuperando en cada decisión —y la cascada de 4 capas cuando algo se desordena o llega tarde— ya razona como senior. Ahora pasá al [laboratorio práctico](streams-practica.md) y **construí**: el orden que solo vale por partición, los offsets que pierden o duplican, una ventana por event-time con un evento tardío, un reprocesamiento estilo Kappa, y el scan que lee 1 columna en vez de 40.

**Ejercicios 16**
16.1 🔁 Listá "las tres cosas" aplicadas a un pipeline de métricas en vivo.
16.2 🧠 Te preguntan "¿cómo garantizás que no perdés eventos si un consumidor se cae?". Respondé con offsets + idempotencia.
16.3 🧠 ¿Por qué "agrupá por event-time" es el consejo que más se repite en este módulo?

---

## Soluciones

**1.1** Batch trata los datos como un conjunto **finito en reposo** (con principio y fin: procesás "todo lo de ayer" de una vez); stream los trata como un flujo **infinito en movimiento** (procesás cada evento a medida que llega, sin un "fin del dataset").

**1.2** Porque un stream no tiene fin, así que "el total" (que presupone haber visto todo) no está definido. Lo reformulás como "el total **hasta ahora**" (acumulado corriente) o "el total **de esta ventana de tiempo**" (p. ej. por minuto).

**1.3** Un batch es un stream **acotado**: si tratás "el archivo/tabla" como un flujo que empieza en el primer registro y termina en el último, el mismo motor de stream lo procesa. Por eso Flink/Spark unifican ambos: batch = stream con un fin conocido.

**2.1** (1) Latencia: reaccionar **ahora** (fraude, alertas, dashboards en vivo) en vez de esperar el batch. (2) Datos infinitos: telemetría/logs/IoT/clickstream que nunca tienen un "fin" para correr un batch.

**2.2** Tres de: el tiempo se vuelve ambiguo (event-time vs processing-time); los eventos llegan desordenados; no podés esperar al fin del dataset para emitir un resultado; recuperarte de una caída a mitad de un infinito (desde qué offset, sin perder/duplicar).

**2.3** Porque en batch el "fin" es donde te parás a calcular el resultado final. Sin fin, tenés que **decidir cuándo emitir** un resultado parcial sin haber visto todo → necesitás cortar el infinito en **ventanas** y decidir cuándo cerrarlas (**watermarks**). Ambos nacen de "no hay un fin natural donde contar".

**3.1** Porque el log es **append-only e inmutable**: consumir es **leer** (avanzar un offset), no sacar. El mensaje se borra por **retención** (por tiempo o tamaño configurado), no por consumo.

**3.2** (1) **Releer/reprocesar**: podés mover el offset hacia atrás y volver a procesar (base de Kappa). (2) **Múltiples consumidores independientes**: varios consumer groups leen el mismo log con sus propios offsets sin pisarse. Una cola tradicional borra al consumir, así que no permite ninguna de las dos.

**3.3** Conserva **el último valor por clave** (descarta versiones viejas de la misma key). Sirve para representar "el estado actual de cada entidad" como un log (changelog/event sourcing): releyendo el topic compactado reconstruís el último estado de todo.

**4.1** Garantiza orden **dentro de una partición** (se lee en el orden de escritura, por offset); **no** garantiza orden **entre particiones** distintas.

**4.2** Key = `pedidoId` (o `aggregateId`). Como `partición = hash(key) % N`, todos los eventos del mismo pedido caen en la **misma** partición → se procesan en el orden en que se escribieron. Distintos pedidos pueden ir a particiones distintas y procesarse en paralelo, lo cual está bien (no necesitan orden entre sí).

**4.3** Baja cardinalidad o un valor muy popular concentra muchos mensajes en **pocas/una** partición → **hot partition**: esa partición (y su único consumidor) se sobrecarga mientras las demás están ociosas, perdiendo el paralelismo y volviéndose un cuello. Hay que elegir una key con buena distribución.

**5.1** **Una sola** (cada partición la consume exactamente un consumidor del grupo). Si hay más consumidores que particiones, los sobrantes quedan **ociosos** (el paralelismo máximo = número de particiones).

**5.2** Porque cada consumer group lleva sus **propios offsets** y Kafka no borra el mensaje al leerlo (módulo 3). Dos grupos leen el mismo log de forma independiente, cada uno a su ritmo y su posición, sin afectarse.

**5.3** Si procesar un mensaje tarda más que `max.poll.interval.ms`, Kafka asume que el consumidor murió y lo **expulsa** del grupo → dispara un **rebalanceo** → se redistribuyen particiones (y en el modo eager, todos paran) → el reinicio del consumo es lento y otro consumidor puede volver a tardar → otro rebalanceo. Cascada. Se corta acelerando el procesamiento, subiendo el intervalo o bajando `max.poll.records`.

**6.1** Commit ANTES (at-most-once): si te caés tras commitear y antes de procesar, el mensaje queda marcado leído pero **sin procesar** → **se pierde**. Commit DESPUÉS (at-least-once): si te caés tras procesar y antes de commitear, al reiniciar **reprocesás** → **duplicado**.

**6.2** Porque entregar exactamente una vez sobre una red que pierde/duplica/reordena es imposible (problema de los dos generales): no podés distinguir "no llegó" de "llegó pero se perdió el ack", así que reintentás y podés duplicar. Pero el **efecto** sí puede ser exactly-once: con at-least-once + **idempotencia**, procesar el duplicado no cambia el resultado → observable como una sola vez.

**6.3** La garantía exactly-once de Kafka (productor idempotente + transacciones) vale **dentro del mundo Kafka**: leer-procesar-escribir entre topics + commit de offset en una transacción atómica (Kafka Streams). Cuando el efecto sale a un **sistema externo** (DB, mail, API), esa escritura tiene que ser **idempotente** o transaccional con el sink; el reprocesamiento la duplicaría si no.

**7.1** Event-time: cuándo **ocurrió** el evento (timestamp embebido). Processing-time: cuándo el **sistema lo procesó** (reloj del procesador al recibirlo).

**7.2** Porque al reprocesar, el processing-time es **otro** (lo procesás otro día/otra hora) → los mismos eventos caen en ventanas distintas → el resultado cambia. Con event-time, el timestamp del evento es fijo, así que cae siempre en la misma ventana → reprocesar da el mismo resultado (reproducible).

**7.3** Cuando lo que medís **es sobre el procesador, no sobre el dominio**: throughput/latencia del sistema "ahora", carga actual, métricas operativas. Ahí "cuándo lo procesé" es justo lo que querés medir.

**8.1** Afirma "**no espero más eventos con event-time menor que T**". Cuando el watermark pasa el **fin** de una ventana, esa ventana **se cierra** y emite su resultado.

**8.2** Agresivo (margen chico): cierra ventanas rápido → **baja latencia** pero **descarta** más eventos tardíos (totales incompletos). Conservador (margen grande): espera más → **no pierde** tardíos pero **demora** cada resultado. Es el dial completitud-vs-latencia.

**8.3** Porque el sistema **no puede saber** si llegará un evento más viejo: lo **estima** (p. ej. event-time máximo visto menos un margen). Es una apuesta heurística; por eso siempre puede llegar un evento "tardío" que la viole (módulo 10).

**9.1** Tumbling: ventanas fijas **contiguas sin solape** (cada evento en una sola). Sliding: ventanas fijas que **avanzan de a un paso menor** que su tamaño → **se solapan** (cada evento en varias).

**9.2** Lo define la **inactividad**: la ventana agrupa eventos hasta que pasa un *gap* sin actividad (p. ej. 30 min sin eventos del usuario). Es la indicada para agrupar "lo que pasó junto" sin un corte de reloj fijo: sesión de navegación, una llamada, una visita.

**9.3** **Sliding** de 10 minutos con paso de 1 minuto: querés "los últimos 10 min" (tamaño fijo) "recalculado cada minuto" (avanza de a 1) → ventanas solapadas. Un tumbling de 10 min solo te daría un valor cada 10 min, no actualizado cada minuto.

**10.1** (1) Descartar (drop), (2) allowed lateness (mantener el estado de la ventana y actualizar el resultado si llega un tardío), (3) side output (mandar los muy tardíos a un canal aparte para corregir en batch).

**10.2** Cuesta **memoria/estado**: mantenés vivas las ventanas ya cerradas para poder actualizarlas → no liberás ese estado. Para siempre sería estado infinito (memoria que crece sin parar); por eso el allowed lateness tiene un límite tras el cual la ventana se descarta de verdad.

**10.3** El watermark le da a la ventana por event-time un **margen** para esperar a los rezagados: mientras el desorden quepa dentro de ese margen, los eventos caen igual en su ventana correcta (se absorben). Lo que llega **después** de que el watermark cerró la ventana queda afuera → es "tardío" y va a la política de tardíos (drop / lateness / side output).

**11.1** (a) El **estado** del procesador y (b) los **offsets de entrada** correspondientes. Tienen que ser atómicos entre sí porque el estado debe reflejar **exactamente** los eventos hasta esos offsets; si no, al restaurar habría desajuste (estado que incluye eventos posteriores al offset, o al revés) → pérdida o duplicación.

**11.2** Porque el checkpoint guardó estado y offset **en el mismo punto del stream**: al restaurar, reprocesás desde ese offset, y los eventos que reprocesás son justo los que el estado **todavía no** reflejaba (los posteriores al checkpoint). Lo anterior ya está en el estado y no se vuelve a leer. No hay solape → ni duplica ni pierde.

**11.3** Porque el checkpoint solo abarca el **estado interno** del procesador y los offsets de Kafka. Una escritura a una **DB externa** ocurrió "fuera" del checkpoint: al restaurar y reprocesar, esa escritura se **repite**. Para que no duplique, el sink externo tiene que ser idempotente (o participar en una transacción con el commit).

**12.1** Porque ir a una DB remota **por cada evento** agrega latencia de red en el camino caliente y no escala al volumen de un stream. El estado local (en memoria/RocksDB en disco local) se accede sin red, y el particionado por key reparte el estado entre instancias en paralelo.

**12.2** Cada cambio de estado se escribe también a un **changelog topic** en Kafka (compactado). Si la instancia muere (o el estado se mueve a otra tras un rebalanceo), el estado se **reconstruye** releyendo el changelog desde Kafka. El estado local es efímero/un cache; el changelog replicado es la verdad durable.

**12.3** Porque el estado no es más que el **resultado de aplicar** la secuencia de cambios, y esa secuencia es un stream (el changelog). Guardás los cambios como un log y el estado actual se obtiene reproduciéndolo — igual que una máquina de estados replicada reconstruye su estado reproduciendo su log de comandos ([consenso](consenso.md)).

**13.1** OLTP: storage **por filas** (row-store), para operaciones chiquitas que tocan **muchas columnas de pocas filas** (traer/insertar un registro). OLAP: storage **por columnas** (column-store), para consultas que tocan **pocas columnas de muchísimas filas** (agregados sobre millones).

**13.2** (1) **Menos I/O**: un agregado lee **solo las columnas que necesita** (la columna `monto`), no las filas enteras con sus 40 columnas. (2) **Mejor compresión**: una columna tiene valores homogéneos (mismo tipo/dominio) → comprime mucho más (RLE, diccionario, delta) → menos bytes a leer, y permite ejecución vectorizada (SIMD).

**13.3** Porque una consulta analítica pesada hace **scans enormes** que consumen I/O, memoria y CPU, **compitiendo** con el tráfico transaccional de la app (latencia de los usuarios) y pudiendo saturar/bloquear la DB de producción. Además el row-store es ineficiente para esos scans. Se mueve a un sistema columnar (warehouse) dedicado.

**14.1** Warehouse: sistema columnar con datos **estructurados y modelados** para SQL analítico (Snowflake/BigQuery). Lake: storage **barato y crudo** (archivos en S3, Parquet), schema-on-read. Lakehouse: storage de lake **+** una capa transaccional (Delta/Iceberg/Hudi) que le da ACID/schema/time-travel.

**14.2** El storage y el cómputo se volvieron **baratos y elásticos** (cloud): conviene cargar **todo crudo** y transformar **adentro** (ELT) con el poder del warehouse, pudiendo rehacer transformaciones sin re-extraer del origen. Antes (cómputo/almacenamiento caros) había que transformar y filtrar **antes** de cargar (ETL).

**14.3** Por un **pipeline**, típicamente un stream: **CDC** (Debezium leyendo el WAL de la DB OLTP) o eventos de Kafka que llevan los cambios al lake/warehouse, donde dbt los transforma. El stream es el transporte; "OLTP → CDC → lake/warehouse → transformación → dashboards" es el pipeline analítico moderno.

**15.1** Lambda corre **dos** pipelines/bases de código (batch exacto + speed aproximado, mergeados en el serving). Kappa corre **uno solo** (todo stream); recalcula reprocesando el log.

**15.2** (1) Un **log reproducible** (Kafka guarda todo y podés releer desde un offset viejo). (2) Procesar por **event-time** (no processing-time), para que reprocesar los mismos eventos los ponga en las mismas ventanas → mismo resultado.

**15.3** Reproceso el log por event-time **desde el offset de hace un mes** con la versión corregida del job (un consumer group nuevo), dejo que alcance el presente y hago el **switch** al nuevo resultado. Como Kafka guarda el log y agrupo por event-time, el recálculo da el resultado correcto sin tocar el origen.

**16.1** (a) Happy path: eventos a Kafka particionados por la entidad que necesita orden, consumer group por event-time, ventanas tumbling cerradas por watermark, agregado a un store columnar. (b) Modelo de falla: at-least-once + checkpoint del estado → reprocesa desde offset; tardíos con allowed lateness, muy tardíos a side output. (c) Recuperación: reproceso por event-time desde el offset afectado (Kappa) y switch; el changelog reconstruye el estado.

**16.2** "Commiteo el offset **después** de procesar (at-least-once): si me caigo entre procesar y commitear, al reiniciar reproceso desde el último offset commiteado → no pierdo nada. Como eso puede duplicar, hago el procesamiento **idempotente** (o transaccional con el sink) para que el reproceso no cambie el resultado. El estado lo recupero del último checkpoint/changelog."

**16.3** Porque agrupar por event-time es lo que hace los resultados **correctos y reproducibles**: el evento cae en la ventana de cuándo **pasó**, no de cuándo lo procesaste → reprocesar (Kappa, recuperación de fallas) da el mismo resultado, y los tardíos van a su ventana real. Casi todas las anomalías de streaming (totales que mienten, recálculos que difieren) salen de haber usado processing-time.
