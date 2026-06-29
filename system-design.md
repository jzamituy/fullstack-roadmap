# Diseño de sistemas backend: el método y el criterio

**Cómo encarar y defender un diseño · escala, datos, resiliencia y trade-offs · para entrevistas senior y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y recién después contrastá con la sección final. Este módulo es distinto a los demás del hub: no enseña una tecnología, enseña **a pensar**. System Design no se trata de memorizar arquitecturas, sino de tomar requisitos vagos, hacer supuestos explícitos y **justificar cada decisión con su trade-off**. Es lo que separa "sé usar Postgres y Redis" de "sé diseñar un sistema". Acá esas piezas que ya viste se vuelven herramientas de un razonamiento más grande.

**Lo que asumimos.** Que ya conocés las piezas: bases de datos relacionales ([PostgreSQL](postgresql.md)) y NoSQL ([NoSQL](nosql.md)), caché y colas ([Redis](redis.md)), arquitectura event-driven ([Event-driven](event-driven.md)) y observabilidad ([Observabilidad](observabilidad.md)). Acá no las re-explicamos: las **combinamos con criterio**.

> **¿Te falta alguna base?** System Design es la capa que **une** todo lo anterior, así que sin las piezas queda abstracto. Si nunca pensaste en réplicas o transacciones, repasá [PostgreSQL](postgresql.md); si "caché" o "cola" te suenan nuevos, [Redis](redis.md); y si no viste consistencia eventual, [Event-driven](event-driven.md) — sin esas bases, los módulos 4, 5 y 6 te van a costar el doble. No es un muro: es la rampa que te lleva de "conozco las herramientas" a "sé cuándo usar cada una".

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código). Ojo: este módulo es casi todo criterio, así que vas a ver muchos 🧠 — es a propósito.

**Índice de módulos**
1. El método: cómo encarar y conducir cualquier problema de diseño
2. Las métricas y la matemática de servilleta: latencia, throughput, nueves y estimación
3. Escalar: vertical, horizontal y el load balancer
4. Datos a escala: réplicas, sharding y el teorema CAP
5. Caching a fondo: dónde, cómo y el problema de la invalidación
6. Consistencia: fuerte vs eventual (CAP y PACELC)
7. Comunicación entre servicios: sync vs async
8. Mensajería y colas: entrega, idempotencia y backpressure
9. Monolito vs microservicios: el criterio honesto
10. Resiliencia: diseñar para que falle
11. Observabilidad: operar lo que diseñaste
12. Caso completo: diseñar un acortador de URLs + rate limiter

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El método: cómo encarar cualquier problema de diseño

**Teoría.** El error de junior en System Design es saltar a la solución ("uso Kafka y Redis") antes de entender el problema. El senior sigue un **método**, siempre el mismo:

1. **Aclarar requisitos.** Funcionales (qué hace: "acortar URLs, redirigir") y **no funcionales** (cómo de bien: ¿cuánta escala? ¿lecturas vs escrituras? ¿latencia tolerable? ¿consistencia fuerte o eventual?). Los no funcionales son los que definen la arquitectura.
2. **Estimar (back-of-the-envelope).** Números aproximados: usuarios, requests por segundo (RPS), volumen de datos, ancho de banda. No para acertar, sino para saber si necesitás una máquina o mil.
3. **Definir la API y el modelo de datos.** El contrato y qué guardás.
4. **Diseño de alto nivel.** Las cajas y flechas: clientes → load balancer → servicios → datos/caché.
5. **Profundizar donde duele.** El cuello de botella real (la tabla más leída, el endpoint más caliente) y cómo lo resolvés.
6. **Discutir trade-offs.** Toda decisión tiene un costo. Decirlo en voz alta **es** la señal de seniority.

> **La regla mental:** en System Design **no hay respuestas correctas, hay decisiones justificadas.** "Depende" es una respuesta válida si seguís con "...depende de si priorizamos X o Y, y dado el requisito Z, elijo X porque...".
>
> **Y en la entrevista, vos manejás el tablero:** declarás tus supuestos en voz alta, confirmás los requisitos antes de dibujar nada, y **narrás tu razonamiento** mientras pensás (el silencio está bien si lo acompañás con "estoy viendo si conviene A o B"). El entrevistador no evalúa si adivinás su solución — evalúa **cómo pensás**.

**Conducir la conversación: el guion de los 45 minutos.** Los 6 pasos son *qué* decidir; esto es *cómo* manejar el tiempo. Un candidato senior no espera que el entrevistador encuentre los problemas: **los propone él**. Presupuesto típico de una ronda de ~45 min:

| Fase | Tiempo | Qué hacés |
|---|---|---|
| **Requisitos** | ~5 min | Funcionales (3 features core, "el usuario puede…") + **no funcionales cuantificados** ("< 200 ms p99", "10M DAU", "lectura-intensivo 100:1"). Los no funcionales mandan. |
| **Entidades centrales** | ~2 min | Los sustantivos del dominio (User, Post, Follow) **antes** del API: el API y el esquema *caen* de ahí. |
| **API / contrato** | ~5 min | Un endpoint por feature core, sobre las entidades (`POST /tweets`, `GET /feed`). |
| **Diseño de alto nivel** | ~10-15 min | Cajas y flechas, recorriendo cada endpoint. Sin complejidad prematura. |
| **Deep dives** | ~10-15 min | Endurecés el diseño contra los no funcionales: los 2-3 cuellos de botella y los casos borde. |
| **Wrap-up** | ~3-5 min | Qué quedó flojo, qué monitorearías, qué harías con más tiempo. |

> **La diferencia Senior → Staff.** Dos candidatos con el mismo conocimiento se separan en cómo **conducen**:
> - **Scopeás vos el problema.** "Diseñá Twitter" es deliberadamente vago: un Staff decide el alcance ("asumo 100M DAU, foco en timeline y publicación, dejo afuera DMs y trending — ¿de acuerdo?") en vez de esperar que se lo acoten. Sos *navegador, no ejecutor*.
> - **Cortás complejidad sin piedad.** Ir a multi-región y 12 servicios en *toda* pregunta es mala señal en 2026. "¿Esto hace falta a esta escala?" — un nodo si alcanza. Simplicidad = criterio, no debilidad.
> - **Decidís, no devolvés opciones.** "Voy con Postgres porque X" gana sobre "podríamos usar A, B o C" sin elegir. Ponés la decisión y su trade-off sobre la mesa.
> - **Liderás los deep dives.** "Me preocupan tres cosas: la hot key del feed, la consistencia del contador y el multi-región. Empiezo por la de más riesgo" — sin que te lo pidan.
> - **Manejás el reloj.** Si quedan 10 min, no te trabás en el esquema: vas al cuello de botella que más señal da.

**Ejercicios 1**
1.1 🔁 Nombrá los 6 pasos del método y decí cuál define la arquitectura.
1.2 🧠 Te piden "diseñá Twitter". ¿Cuáles son las **primeras tres preguntas** que hacés antes de dibujar nada, y por qué?
1.3 🧠 ¿Por qué los requisitos **no funcionales** pesan más en la arquitectura que los funcionales? Dá un ejemplo donde el mismo requisito funcional lleve a dos arquitecturas distintas según el no funcional.
1.4 🧠 Te dan "diseñá Instagram" y 35 minutos. Hacé el presupuesto de tiempo de la ronda y decí **qué dos cosas dejás explícitamente fuera de scope** y por qué.
1.5 🧠 El enunciado es vago a propósito ("diseñá un sistema de notificaciones"). ¿Qué hace **distinto** un candidato Staff respecto de uno Senior en los primeros 5 minutos?
1.6 🧠 A los 25 minutos el entrevistador te dice "esa shard key no escala". ¿Cómo respondés sin tirar abajo todo el diseño ni ponerte a la defensiva?

---

## Módulo 2 — Las métricas y la matemática de servilleta: latencia, throughput, nueves y estimación

**Teoría.** No podés diseñar para "rápido" ni "confiable": necesitás números.

- **Latencia** = cuánto tarda UNA operación. **Throughput** = cuántas operaciones por segundo aguantás. Son distintas y a veces compiten: batchear sube el throughput pero sube la latencia de **cada request individual** (cada una espera a que se llene el batch).
- **Percentiles, no promedios.** El promedio miente: si 99 requests tardan 10ms y una tarda 5s, el promedio (60ms) esconde el desastre. Por eso se mide **p50** (mediana), **p95**, **p99**. El **p99** es la experiencia del peor 1%.

> **El ancla — tail latency.** Parece "solo el 1%", pero pensalo así: si para armar una pantalla tu request dispara 100 llamadas internas y espera a todas, alcanza con que **una** caiga en su p99 para que toda la pantalla sea lenta. Con 100 dependencias, casi *todos* tus usuarios pegan ese p99 en alguna llamada. O sea: **el p99 no es el caso raro, es la experiencia típica.** Por eso se optimiza la cola, no el promedio.
- **Disponibilidad: los nueves.** Cuánto downtime tolerás al año:

| Disponibilidad | Downtime/año |
|---|---|
| 99% ("dos nueves") | ~3.65 días |
| 99.9% ("tres nueves") | ~8.76 horas |
| 99.99% ("cuatro nueves") | ~52.6 minutos |
| 99.999% ("cinco nueves") | ~5.26 minutos |

- **SLI / SLO / SLA.** El **SLI** es lo que medís (ej. % de requests bajo 200ms). El **SLO** es tu objetivo interno (ej. 99.9%). El **SLA** es el contrato con el cliente (con penalización si lo incumplís). Regla: SLA < SLO, para tener margen.

> **Modelo mental:** cada "nueve" extra cuesta aproximadamente **10× más** en ingeniería y plata. Cinco nueves no es "mejor que tres", es una decisión de negocio carísima. La pregunta senior no es "¿cómo llego a cinco nueves?" sino "¿cuántos nueves justifica este sistema?".

---

**La matemática de servilleta (back-of-the-envelope).** Antes de dibujar, un número decide la arquitectura: ¿una máquina o mil? ¿sharding o una tabla? No buscás precisión, buscás el **orden de magnitud**.

**Números canónicos (2026) para tener en la cabeza.** Cuidado: usar números de 2015 es el error más común — hoy un solo nodo aguanta lo que antes exigía un cluster.

| Latencias | | Capacidad / throughput | |
|---|---|---|---|
| Lectura de RAM | ~100 ns | RAM por servidor | 512 GB – 24 TB |
| Lectura random SSD | ~16 µs | SSD local | hasta ~60 TB |
| RTT misma AZ | < 1 ms | Red intra-DC | 25 Gbps (100+ en nodos grandes) |
| RTT cross-AZ (misma región) | 1–2 ms | S3 / object store | ~ilimitado |
| RTT cross-región | 50–150 ms | QPS por instancia (orden) | ~1k–10k según el trabajo |

Tamaños de referencia: timestamp 8 B · UUID 16 B · una URL ~100 B · un tweet ~300 B · una fila de metadata ~0.1–1 KB.

**Potencias de 10 (el truco para hacerlo de cabeza).** Un día tiene 86 400 s ≈ **10⁵ s**. Entonces "X por día → por segundo" es dividir por ~100 000. Ej.: 1 M req/día ≈ **~12 req/s**; 1 B (10⁹) eventos/día ≈ **~10⁴/s**. Redondeá: el objetivo es el exponente, no el decimal.

**La receta de 5 cálculos, en orden:**
1. **QPS** promedio = volumen/día ÷ 10⁵; **QPS pico** = promedio × 2–10 (dimensionás para el pico, no el promedio).
2. **Storage** = filas/día × bytes/fila × retención (años × 365).
3. **Ancho de banda** = QPS × tamaño del payload.
4. **Memoria de cache** = regla 80/20: cacheás el ~20% caliente del working set diario, no todo.
5. **Nº de servidores** = QPS pico ÷ QPS por instancia. Y para la **concurrencia**, *Little's Law*: con `λ` = 1000 req/s y `W` = 0.2 s de latencia, hay `L = λ × W = 200` requests en vuelo → ése es el orden de conexiones/threads que tenés que sostener.

> **El matiz Staff:** estimá **solo cuando el número cambia una decisión**, no como ritual. "Son ~40 escrituras/s → no necesito shardear todavía, alcanza con réplicas + cache" es estimación que *conduce* el diseño. Calcular siete cifras que no mueven ninguna decisión es perder tiempo del reloj. (El **Módulo 12** aplica esta receta completa de punta a punta.)

**Ejercicios 2**
2.1 🔁 ¿Por qué se reporta p99 en vez del promedio de latencia?
2.2 🧠 Un servicio hace 5 llamadas internas en paralelo y espera a todas. Si cada dependencia tiene p99 de 100ms, ¿por qué la latencia p99 del servicio compuesto es **peor** que 100ms? (pista: tail latency)
2.3 🧠 Tu jefe dice "quiero cinco nueves". ¿Qué dos preguntas le hacés antes de aceptar, y por qué?
2.4 🧠 Un sistema recibe 500 M de tweets/día, ~300 B cada uno. Estimá (mostrando la cuenta, con potencias de 10) las **escrituras por segundo** y el **storage a 1 año**.
2.5 🧠 Calculás que un sistema recibe ~50 escrituras/s y ~5000 lecturas/s. ¿Qué **decisión de arquitectura** cambia ese número — y qué cálculo NO hacía falta hacer?

---

## Módulo 3 — Escalar: vertical, horizontal y el load balancer

**Teoría.** Dos formas de aguantar más carga:

- **Vertical (scale up):** una máquina más grande. Simple, sin cambios de código, pero tiene techo (la máquina más grande que existe) y es un único punto de falla.
- **Horizontal (scale out):** más máquinas. Sin techo práctico y tolera fallas, pero exige que tu servicio sea **stateless** (sin estado en memoria local) para que cualquier instancia atienda cualquier request.

La clave que lo habilita es **sacar el estado del servicio**: las sesiones van a Redis o a un token (no a memoria local), los archivos a un object store (S3), los datos a la base. Servicio stateless = escalás poniendo más copias detrás de un **load balancer**.

> **Si venís de React, ya lo conocés:** un servicio stateless es como un **componente puro** — misma entrada, misma salida, sin estado oculto adentro. Por eso podés correr cualquier copia en cualquier instancia y da igual, lo mismo que renderizás un componente puro donde sea.

El **load balancer** reparte tráfico entre instancias. Dos sabores:

- **L4 (transporte):** rutea por IP/puerto, sin mirar el contenido. Rapidísimo, pero "tonto".
- **L7 (aplicación):** entiende HTTP, así que puede rutear por path/header/cookie (`/api` a un pool, `/static` a otro), terminar TLS y hacer health checks ricos. Más flexible, un poco más caro.

> **El insight que separa niveles:** "escalar" casi siempre empieza por **hacer el servicio stateless**, no por agregar máquinas. Si tu servicio guarda estado en memoria local, agregar instancias te rompe (el usuario cae en otra instancia que no tiene su sesión). Primero apátrida, después horizontal.

**Ejercicios 3**
3.1 🔁 ¿Cuál es la diferencia entre escalar vertical y horizontalmente, y qué requisito impone el horizontal sobre tu servicio?
3.2 🧠 Un servicio guarda el carrito de compras en memoria local. Lo escalás a 3 instancias y los usuarios se quejan de que el carrito "se vacía solo". ¿Qué pasó y cómo lo arreglás?
3.3 🧠 ¿Cuándo elegirías un load balancer L4 sobre uno L7 a pesar de que L7 es más flexible?

---

## Módulo 4 — Datos a escala: réplicas, sharding y el teorema CAP

**Teoría.** La base de datos es casi siempre el primer cuello de botella, porque escalar datos es más difícil que escalar servicios (los datos tienen estado por definición).

- **Réplicas de lectura (read replicas):** copiás la base a varios nodos de solo-lectura. Las lecturas se reparten; las escrituras siguen yendo al primario. Ideal cuando leés mucho más de lo que escribís (el caso común). Costo: **replication lag** — una réplica puede estar atrasada milisegundos/segundos, así que "leé tu propia escritura" puede fallar.
- **Particionado / sharding:** partís los datos entre varias bases por una **shard key** (ej. `user_id`). Cada shard tiene un pedazo. Escala las **escrituras** (que las réplicas no escalan), pero agrega complejidad enorme: queries cross-shard, rebalanceo, y el riesgo de **hot partition** (una shard key mal elegida concentra el tráfico en un nodo).
  - **Estrategias:** por rango (simple, pero hotspots), por **hash** de la key (reparte parejo, pero pierde queries por rango), o por directorio (una tabla de lookup, flexible pero un punto más).
- **Teorema CAP:** ante una **partición de red** (P, que en sistemas distribuidos *va a pasar*), tenés que elegir entre **Consistencia** (todos ven el mismo dato, o error) y **Disponibilidad** (todos reciben respuesta, aunque sea vieja). No podés tener las tres. Lo profundizamos en el módulo 6.

> **Regla de oro del orden:** la mayoría de los sistemas escala datos en este orden — (1) **índices** y queries eficientes, (2) **caché** (módulo 5), (3) **read replicas**, y solo cuando las escrituras no dan más, (4) **sharding**. Shardear de entrada es over-engineering: es la decisión más cara y difícil de revertir.

**Ejercicios 4**
4.1 🔁 ¿Qué problema escala una read replica y cuál NO, y para qué sirve el sharding?
4.2 🧠 Elegís `country` como shard key de una app global. ¿Qué problema concreto vas a tener y qué key elegirías mejor?
4.3 🧠 Un usuario actualiza su perfil y al recargar ve los datos viejos. Sospechás de read replicas. Explicá la causa y dá **dos** formas de mitigarla.

---

## Módulo 5 — Caching a fondo: dónde, cómo y el problema de la invalidación

**Teoría.** El caché cambia las reglas: una lectura que costaba 50ms a la base cuesta 1ms a memoria. Pero "solo hay dos problemas difíciles en computación: invalidar caché y nombrar cosas".

**Dónde cachear** (en capas, del cliente al dato):
- **Cliente/navegador** (HTTP cache), **CDN** (estáticos y respuestas cacheables cerca del usuario), **caché de aplicación** (Redis, compartido entre instancias), **caché de la base** (buffer pool).

**Estrategias de escritura:**
- **Cache-aside (lazy loading):** la app lee del caché; si no está (*miss*), lee de la base y lo guarda. La más común. Costo: la primera lectura es lenta y puede servir datos viejos si no invalidás.
- **Write-through:** escribís a caché y base a la vez. Caché siempre fresco, pero cada escritura es más lenta.
- **Write-back (write-behind):** escribís al caché y a la base **después**, en batch. Rapidísimo, pero si el caché se cae perdés datos.

**El problema real es la invalidación:** ¿cómo te asegurás de que el caché no sirva datos viejos? Opciones: **TTL** (expira solo; simple, pero tolerás staleness hasta que venza) o **invalidación explícita** (borrás la key al escribir; preciso, pero fácil de olvidar y propenso a races).

> **Si usaste React Query o SWR, esta pelea ya la diste:** es exactamente `staleTime` (TTL: el dato vale hasta que expira) vs `invalidateQueries` (invalidación explícita: lo marcás viejo a mano cuando algo cambia). El mismo trade-off, ahora del lado del server.

> **Trampas clásicas que te van a preguntar:**
> - **Cache stampede / thundering herd:** una key popular expira y mil requests pegan a la base a la vez. Se mitiga con *locks* o *early recomputation*.
> - **Cache penetration:** consultas de keys que no existen pasan siempre de largo al backend. Se mitiga cacheando también el "no existe".

**Ejercicios 5**
5.1 🔁 Explicá cache-aside vs write-through en una frase cada una, con su trade-off.
5.2 🧠 Una key muy popular (la home page) expira a medianoche y la base colapsa un segundo. ¿Cómo se llama el problema y cómo lo prevenís?
5.3 🧠 ¿Por qué la **invalidación** es más difícil que el cacheo en sí? Dá un ejemplo donde una invalidación olvidada cause un bug sutil.

---

## Módulo 6 — Consistencia: fuerte vs eventual (CAP y PACELC)

**Teoría.** Acá está el corazón del diseño distribuido.

- **Consistencia fuerte:** después de una escritura, *toda* lectura ve el valor nuevo. Intuitiva, pero cara: exige coordinación entre nodos (más latencia) y, ante una partición, sacrifica disponibilidad.
- **Consistencia eventual:** las lecturas pueden ver datos viejos por un rato, pero "eventualmente" convergen. Más disponible y rápida, pero tu código tiene que tolerar el desfase.

> **El ancla:** ya lo viviste como usuario — publicás un comentario, recargás y no aparece; recargás de nuevo y ahí está. Eso es consistencia eventual: tu escritura todavía no se propagó a la réplica que te tocó leer. Y del lado de React es el mismo espíritu del **optimistic update**: mostrás el like al toque (asumís que va a salir bien) y reconciliás con el server después.

**CAP** dice: ante una partición (P), elegís C o A. **PACELC** lo completa con lo que CAP olvida: **E**lse (cuando NO hay partición), igual elegís entre **L**atencia y **C**onsistencia. O sea: el trade-off consistencia/latencia existe **siempre**, no solo en fallas.

**¿Cuándo cada una?** Es por dato, no por sistema:
- **Fuerte:** saldo de una cuenta, stock de un producto, una transacción de pago. Servir un dato viejo acá es un bug grave.
- **Eventual:** contador de likes, feed de noticias, "última vez visto". Que tarde un segundo en propagarse no le importa a nadie.

> **El criterio senior:** no preguntes "¿el sistema es consistente?", preguntá "**¿qué dato necesita qué consistencia?**". Un mismo sistema usa consistencia fuerte para el pago y eventual para el contador de vistas. Forzar consistencia fuerte en todo es lento y caro; forzar eventual en un saldo es un fraude esperando a pasar.

**Ejercicios 6**
6.1 🔁 ¿Qué le agrega PACELC a CAP que CAP deja afuera?
6.2 🧠 Para cada dato, elegí consistencia fuerte o eventual y justificá: (a) saldo de billetera, (b) cantidad de retweets, (c) "¿este username está disponible?" al registrarte.
6.3 🧠 ¿Por qué "nuestro sistema es fuertemente consistente en todo" suele ser una mala señal de diseño y no una virtud?

---

## Módulo 7 — Comunicación entre servicios: sync vs async

**Teoría.** Cuando partís un sistema en servicios, tienen que hablarse. Dos modos:

- **Síncrono (request/response):** el servicio A llama a B y **espera** la respuesta. REST (sobre HTTP, simple y universal) o **gRPC** (sobre HTTP/2 + Protobuf: binario, rápido, contratos tipados, ideal service-to-service interno). Simple de razonar, pero **acopla**: si B está lento o caído, A se cuelga. Y los fallos se propagan (cascada).
- **Asíncrono (eventos/mensajes):** A publica un evento en un broker y **sigue**; B lo procesa cuando puede. Desacopla en el tiempo, absorbe picos y tolera que B esté caído. Costo: complejidad (consistencia eventual, orden, duplicados) y que el flujo es más difícil de seguir y debuggear.

| | Sync (REST/gRPC) | Async (eventos/colas) |
|---|---|---|
| Acoplamiento | Alto (A espera a B) | Bajo (A no conoce a B) |
| Latencia percibida | La suma de la cadena | Inmediata para A |
| Si B se cae | A falla | El mensaje espera en la cola |
| Razonar/debuggear | Fácil | Más difícil |
| Caso ideal | "Necesito la respuesta ahora" | "Esto puede pasar después" |

> **La pregunta que decide:** "¿el llamador **necesita** la respuesta para continuar?" Si sí (validar un pago antes de confirmar) → sync. Si no (mandar el email de confirmación) → async. Mezclar mal esto es la causa #1 de microservicios frágiles. Lo aplicado está en [Event-driven](event-driven.md).

**Ejercicios 7**
7.1 🔁 ¿Cuándo elegís gRPC sobre REST para comunicación interna?
7.2 🧠 En un checkout: (a) validar el stock, (b) cobrar la tarjeta, (c) mandar el email, (d) actualizar analytics. ¿Cuáles harías sync y cuáles async, y por qué?
7.3 🧠 ¿Por qué una cadena de 5 servicios síncronos es un riesgo de disponibilidad? Calculá: si cada uno tiene 99.9% de disponibilidad, ¿cuál es la del conjunto?

---

## Módulo 8 — Mensajería y colas: entrega, idempotencia y backpressure

**Teoría.** Si elegís async (módulo 7), entrás al mundo de los **message brokers** (colas y streams). Hay que entender sus garantías, porque son contraintuitivas. *(Aviso: este es el módulo más denso del recorrido — si hace falta, leelo en dos pasadas.)*

> **El ancla — por qué duplicás.** Un consumidor procesa un mensaje y, justo antes de mandar el **ACK** ("listo, ya lo procesé"), se corta la red. El broker nunca recibió el ACK, así que asume que falló y **reentrega** el mensaje. Resultado: lo procesaste dos veces. Por eso "al menos una vez" es lo normal y vas a recibir duplicados — no es un bug raro, es el modelo.

**Garantías de entrega:**
- **At-most-once:** se entrega 0 o 1 vez (podés perder mensajes). Rara vez aceptable.
- **At-least-once:** se entrega 1 o más veces (nunca perdés, pero podés **duplicar**). Lo normal.
- **Exactly-once:** el santo grial... y en la práctica un **mito** en sistemas distribuidos. Lo que existe es **"effectively-once"**: at-least-once + **consumidores idempotentes**.

**Idempotencia = la herramienta clave.** Una operación es idempotente si ejecutarla N veces da el mismo resultado que ejecutarla una. Como vas a recibir duplicados, tu consumidor tiene que tolerarlos: con una **idempotency key** que registrás como procesada.

```ts
// Consumidor idempotente: ignora un mensaje ya procesado (compila en --strict)
async function handle(msg: { id: string; amount: number }): Promise<void> {
  const yaProcesado = await store.exists(`msg:${msg.id}`);
  if (yaProcesado) return;                 // duplicado → no-op
  await aplicarCobro(msg.amount);
  await store.set(`msg:${msg.id}`, "done", { ttlSeconds: 86_400 });
}
```

> **Ojo con el orden (carrera sutil):** en el código de arriba, entre el `exists` y el `set` hay una ventana donde dos copias del mensaje pueden pasar el chequeo a la vez y cobrar dos veces. Para garantía real, **reservá la idempotency key ANTES del efecto** (un `SET key val NX` atómico, o un `INSERT` con PK única que falle si ya existe) — o hacé que el efecto y el marcado ocurran en la **misma transacción**. Marcar *después* del efecto, como muestra el ejemplo simplificado, no alcanza bajo concurrencia.

**Backpressure:** si los productores generan más rápido de lo que los consumidores procesan, la cola crece sin fin hasta explotar. Lo manejás con: límites de tamaño, consumidores que escalan, y un **dead-letter queue (DLQ)** para los mensajes que fallan repetidamente (en vez de bloquear la cola para siempre).

> **El modelo mental honesto:** dejá de buscar "exactly-once" en el broker. Diseñá consumidores **idempotentes** y asumí at-least-once. Esa es la práctica real; lo demás es marketing.

**Ejercicios 8**
8.1 🔁 ¿Por qué "exactly-once" es un mito y qué se usa en su lugar?
8.2 🧠 ¿Cuál de estas operaciones ya es idempotente y cuál no, y cómo harías idempotente la que no lo es? (a) `SET saldo = 100`, (b) `saldo = saldo + 50`.
8.3 ✍️ Escribí una función `procesarUna(msg, store)` que aplique un efecto solo si el `msg.id` no fue procesado antes (idempotencia con una key). Usá un `store` con `exists`/`set`.

---

## Módulo 9 — Monolito vs microservicios: el criterio honesto

**Teoría.** La moda dice "microservicios". El criterio dice "depende", y casi siempre empieza por monolito.

- **Monolito:** una sola unidad desplegable. Simple de desarrollar, testear, deployar y debuggear; una transacción ACID resuelve la consistencia. Su problema **no es técnico, es organizacional**: cuando muchos equipos tocan el mismo código, se pisan.
- **Microservicios:** servicios independientes, cada uno con su base y su deploy. Permiten que equipos escalen en paralelo y escalar partes por separado. Costo brutal: red entre servicios (latencia, fallas), consistencia distribuida (sagas, no transacciones), observabilidad difícil, y complejidad operativa (orquestación, [Kubernetes](kubernetes.md)).

**El camino sano** que recomienda la industria en 2026:
1. Empezá con un **monolito modular** (módulos bien separados *dentro* de un mismo deploy, con límites claros — acá ayuda [DDD](ddd.md) y los bounded contexts).
2. Extraé a microservicio **solo** lo que tenga una razón real: escala muy distinta, equipo dedicado, o ciclo de release independiente.

> **La frase que demuestra criterio:** "microservicios resuelven un problema **de escala organizacional**, no de performance". Si tenés un equipo y un producto que arranca, microservicios te dan toda la complejidad distribuida sin ninguno de los beneficios. El monolito modular es el default correcto, no el premio consuelo.

**Ejercicios 9**
9.1 🔁 ¿Qué tipo de problema resuelven realmente los microservicios?
9.2 🧠 Una startup de 4 devs quiere arrancar con 12 microservicios "para estar preparados". ¿Qué les decís y qué proponés en cambio?
9.3 🧠 Dado un monolito modular sano, ¿qué tres señales te indican que un módulo SÍ vale la pena extraer a microservicio?

---

## Módulo 10 — Resiliencia: diseñar para que falle

**Teoría.** En sistemas distribuidos, las fallas no son una posibilidad: son una certeza. La red se corta, un servicio se cuelga, un disco se llena. El diseño senior **asume la falla** y la contiene.

Patrones clave:

- **Timeouts:** nunca esperes para siempre. Sin timeout, un servicio lento cuelga a todos los que lo llaman (agotamiento de threads/conexiones). Es la defensa #1, y la más olvidada.
- **Retries con backoff exponencial + jitter:** reintentás un fallo transitorio, pero esperando cada vez más (1s, 2s, 4s...) y con un **jitter** aleatorio para que mil clientes no reintenten todos al mismo tiempo (*retry storm*). *(Si configuraste `retry` y `retryDelay` en React Query, es la misma idea — ahora del lado server, donde un retry storm puede tumbar tu propia base.)*
- **Circuit breaker:** si un servicio falla repetidamente, dejá de llamarlo un rato (estado **open**) en vez de seguir golpeándolo. Cada tanto probás (estado **half-open**); si responde, volvés a **closed**. Le da aire al servicio caído y falla rápido en vez de colgarte.
- **Bulkhead:** aislá recursos (pools de conexiones separados por dependencia) para que una dependencia saturada no se lleve puesto a todo el servicio.
- **Graceful degradation:** si el servicio de recomendaciones se cae, mostrá productos genéricos, no un error 500. Degradá funcionalidad, no disponibilidad.

```ts
// Backoff exponencial con jitter (compila en --strict)
function backoffMs(intento: number, baseMs = 100, maxMs = 10_000): number {
  const exp = Math.min(maxMs, baseMs * 2 ** intento);
  return Math.round(Math.random() * exp);   // full jitter: 0..exp
}
// intento 0 → 0..100ms, intento 3 → 0..800ms, ... cap a 10s
```

> **El cambio de mentalidad:** dejá de preguntar "¿cómo evito que falle?" (no podés) y empezá a preguntar "**¿cómo se comporta cuando falle?**". Un sistema resiliente no es uno que no falla, es uno que falla de forma contenida y predecible.

**Ejercicios 10**
10.1 🔁 ¿Qué hace un circuit breaker y cuáles son sus tres estados?
10.2 🧠 ¿Por qué los retries **sin** backoff ni jitter pueden empeorar una caída en vez de ayudar? (describí el *retry storm*)
10.3 ✍️ Escribí `reintentarConBackoff(fn, maxIntentos)` que reintente una función async con backoff exponencial + jitter y propague el error si se agotan los intentos.

---

## Módulo 11 — Observabilidad: operar lo que diseñaste

**Teoría.** Un sistema que no podés observar no lo podés operar: estás a ciegas. Observabilidad ≠ monitoreo. Monitoreo responde "¿está roto?" (preguntas que sabías de antemano); observabilidad responde "**¿por qué** está roto?" (preguntas nuevas, que no anticipaste).

Los **tres pilares**:
- **Logs:** eventos discretos ("usuario X falló login"). Ricos, pero caros a escala. Estructurados (JSON) y con un **trace id** para correlacionar.
- **Métricas:** números agregados en el tiempo (RPS, latencia p99, uso de CPU). Baratos y perfectos para dashboards y alertas.
- **Traces (distribuidos):** el viaje de UNA request a través de todos los servicios. Indispensable en microservicios para ver dónde se fue el tiempo. El estándar es **OpenTelemetry**.

**Qué medir — los cuatro golden signals** (Google SRE):
1. **Latencia** (¿cuánto tarda?) — separá la de requests exitosas de la de errores.
2. **Tráfico** (¿cuánta demanda?) — RPS.
3. **Errores** (¿cuántas fallan?) — tasa de error.
4. **Saturación** (¿qué tan lleno está?) — CPU, memoria, cola.

> **El error clásico:** alertar sobre **causas** (CPU al 90%) en vez de **síntomas** (latencia p99 arriba del SLO). El CPU alto puede ser normal; lo que le duele al usuario es la latencia. Alertá sobre lo que el usuario siente, e investigá las causas con los logs/traces. Lo práctico está en [Observabilidad](observabilidad.md).

**Ejercicios 11**
11.1 🔁 ¿Cuáles son los tres pilares de la observabilidad y qué responde cada uno?
11.2 🧠 ¿Por qué conviene alertar sobre "latencia p99 > SLO" y no sobre "CPU > 90%"? ¿Cuándo el CPU sí sería una buena alerta?
11.3 🧠 En un sistema de microservicios, una request está lenta pero ningún servicio individual muestra problemas en sus métricas. ¿Qué herramienta de observabilidad usás y por qué?

---

## Módulo 12 — Caso completo: diseñar un acortador de URLs + rate limiter

**Teoría.** Apliquemos el método (módulo 1) de punta a punta. Problema: **un acortador de URLs** (tipo bit.ly).

**1. Requisitos.**
- Funcionales: dado un URL largo, generar uno corto; al visitar el corto, redirigir al largo.
- No funcionales: **lectura-intensivo** (se crea una vez, se visita millones); latencia de redirect bajísima; alta disponibilidad; los short links no caducan.

**2. Estimación.** Digamos 100M URLs nuevas/mes y ratio lectura:escritura de 100:1 → ~40 escrituras/s y ~4000 lecturas/s. En 5 años, ~6000M URLs. Caben en una base con sharding o en un KV store; el volumen por fila es chico.

**3. API y datos.**
```
POST /shorten  { url }      → { shortCode }
GET  /{shortCode}           → 301/302 redirect
```
Tabla: `shortCode (PK) | longUrl | createdAt`. El `shortCode` es el corazón.

**4. La decisión clave — generar el shortCode.** Opciones:
- **Hash del URL (ej. MD5) y tomar 7 chars:** el problema no es la fortaleza de MD5, es que al **truncar** a 7 chars colapsás el espacio a ~62⁷ y aparecen colisiones (paradoja del cumpleaños) → necesitás detección de colisión + reintento con otro slice/salt. Además dos URLs iguales dan el mismo code (¿querés eso?).
- **Contador autoincremental + Base62:** un número global que codificás en `[a-zA-Z0-9]` (62 símbolos). 7 chars Base62 = 62⁷ ≈ **3.5 billones (3.5×10¹²)** de combinaciones. Sin colisiones por diseño. El reto: el contador global es un cuello de botella → se resuelve con **rangos pre-asignados** (cada instancia toma un bloque de 1000 ids y los reparte localmente).

**5. Escala (donde duele = las lecturas).** El redirect es lo caliente: va a **caché** (Redis) `shortCode → longUrl` con altísimo hit rate (los links populares se visitan mucho); miss → base → poblar caché. Una **CDN** delante puede cachear los redirects más populares. La base, con **read replicas** porque es lectura-intensiva.

**6. Trade-offs discutidos.** ¿301 (permanente, el browser cachea y no vuelve a pegar — menos carga, pero no podés contar clicks ni cambiar el destino) o 302 (temporal, cada visita pega al server — podés analizar, pero más carga)? Si querés analytics, 302. Decisión justificada, no automática.

**El rate limiter** (lo necesitás en `/shorten` para que no te abusen). El algoritmo clásico es **token bucket**: un balde con N tokens que se rellena a `r` tokens/s; cada request consume uno; si el balde está vacío, rechazás (429). Permite ráfagas (hasta N) pero limita el promedio (`r`).

```ts
// Token bucket en memoria (la idea; en prod va en Redis para ser distribuido). Compila --strict.
class TokenBucket {
  private tokens: number;
  constructor(
    private readonly capacity: number,
    private readonly refillPerSec: number,
    private lastRefill: number,        // epoch ms
  ) {
    this.tokens = capacity;
  }
  permitir(ahoraMs: number): boolean {
    const elapsedSec = (ahoraMs - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsedSec * this.refillPerSec);
    this.lastRefill = ahoraMs;
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    return false;   // sin tokens → 429 Too Many Requests
  }
}
```

*(El `TokenBucket` de arriba asume un reloj monótono: si `ahoraMs` retrocede, el refill se rompe; en prod usás un reloj monótono o un `Math.max(0, …)` en el elapsed.)*

**Retrospectiva del método.** Fijate lo que hicimos: **(1)** aclaramos requisitos (lectura-intensivo, no caduca), **(2)** estimamos (40 escrituras/s, 4000 lecturas/s), **(3)** definimos API y datos, **(4–5)** diseñamos de alto nivel y **profundizamos donde duele** (el shortCode y el redirect caliente), y **(6)** discutimos trade-offs (301 vs 302, hash vs contador). Ese es el guion de los 6 pasos del módulo 1 — **el mismo que repetís en cualquier entrevista**, sea un acortador, un chat o un feed.

> **El cierre de criterio — no sobre-diseñar.** Todo esto asume escala de bit.ly. Si tu acortador es interno para 100 usuarios, **una tabla en Postgres y nada más** es la respuesta correcta; meter sharding, CDN y rangos de contador sería over-engineering de manual. La marca de seniority no es diseñar el sistema más grande: es diseñar **el que el problema necesita**, y saber qué agregarías *cuando* la escala lo pida. "Empezá simple, escalá cuando duela" — esa frase vale toda la entrevista.

**Ejercicios 12**
12.1 🧠 ¿Por qué un acortador de URLs es un caso **lectura-intensivo**, y qué dos piezas de infraestructura son las más importantes por eso?
12.2 🧠 ¿Qué problema tiene el contador global autoincremental para generar el shortCode, y cómo lo resolvés sin perder la unicidad?
12.3 🧠 Te piden agregar "cuántos clicks tuvo cada link". ¿Cómo cambia tu elección entre redirect 301 y 302, y por qué?
12.4 ✍️ Extendé el `TokenBucket`: agregá un método `tokensDisponibles(ahoraMs)` que devuelva cuántos tokens hay (con el refill aplicado) **sin** consumir ninguno.

---

# Soluciones

> Mirá esto solo después de intentarlo. Casi todas estas son de **criterio**: si tu razonamiento difiere pero justifica bien el trade-off, probablemente también esté bien. En System Design, el *cómo* justificás importa más que el *qué* elegís.

### Módulo 1
**1.1** (1) Aclarar requisitos, (2) estimar, (3) API y modelo de datos, (4) diseño de alto nivel, (5) profundizar donde duele, (6) discutir trade-offs. Los **requisitos no funcionales** (paso 1) son los que definen la arquitectura.

**1.2** Tres preguntas posibles: (a) **¿Qué features están en scope?** (¿tweets y timeline? ¿DMs? ¿trending? "Twitter" entero es inabarcable). (b) **¿Qué escala?** (¿1000 o 300M usuarios? cambia todo). (c) **¿Cuál es el ratio lectura/escritura y la tolerancia de consistencia del timeline?** (un timeline puede ser eventual). Se pregunta primero porque sin scope y escala, cualquier dibujo es adivinanza.

**1.3** Porque el *qué hace* (acortar URLs, postear un tweet) suele ser simple y similar entre sistemas; lo que dispara decisiones arquitectónicas distintas es el *cómo de bien* (escala, latencia, consistencia, disponibilidad). Ejemplo: "guardar y mostrar posts" con 100 usuarios = una tabla Postgres; con 300M usuarios y feed en tiempo real = sharding + caché + fan-out + consistencia eventual. Mismo requisito funcional, dos arquitecturas según el no funcional.

**1.4** Presupuesto para 35 min: requisitos ~4 min, entidades ~2, API ~4, alto nivel ~10, deep dives ~12, wrap-up ~3. Dos cosas fuera de scope (ejemplos válidos): **DMs/mensajería** (es casi otro sistema, con su propio almacenamiento y tiempo real) y **el algoritmo de ranking del feed** (cae en ML/relevancia, no en la arquitectura de almacenamiento/fan-out). Se acotan para profundizar en el core (publicar foto + feed + media + likes) en vez de barnizar todo. Lo clave no es *qué* dejás afuera sino **declararlo y confirmarlo** en voz alta, no resolverlo en silencio.

**1.5** Un Senior pregunta los requisitos y espera que se los acoten; un **Staff scopea él mismo**: propone una escala concreta ("asumo 50M DAU"), elige las 2-3 features core y dice qué deja afuera, identifica el *crux* (la parte de verdad difícil: fan-out, dedup, entrega garantizada) y declara la decisión consistencia/latencia — todo en los primeros minutos y **confirmándolo**, en vez de devolverle al entrevistador una lista de opciones para que elija. Maneja la ambigüedad tomando decisiones, no evitándolas.

**1.6** La objeción es una **señal, no un ataque** — muchas veces el entrevistador prueba si defendés (o corregís) con criterio. El reflejo: (1) no te pongas a la defensiva ni tires todo abajo; confirmá que entendiste ("decís que `country` concentra la carga en el país más grande, ¿no?"). (2) Evaluá si tiene razón. (3) Si la tiene: nombrá el trade-off, aplicá el **fix localizado** (`hash(user_id)` en vez de `country`) y seguí desde ahí, sin reescribir lo que ya funciona. (4) Si tu decisión era defendible: justificala con su trade-off, pero quedate abierto. Lo que evalúan es **adaptabilidad con criterio**: corregir el rumbo rápido sin perder el hilo ni la compostura — ni aferrarte ni colapsar.

### Módulo 2
**2.1** Porque el promedio esconde los outliers que sí duelen: si 99% de las requests van bien y 1% tarda 5s, el promedio se ve sano pero hay usuarios sufriendo. El p99 expone ese peor 1%, que en sistemas con muchas dependencias termina afectando a casi todos en alguna request.

**2.2** Porque al esperar a las 5 en paralelo, la latencia del compuesto es la del **más lento** de los 5, no la de uno. La probabilidad de que *al menos una* de las 5 caiga en su p99 (su cola lenta) es mucho mayor que para una sola: `1 - 0.99⁵ ≈ 4.9%`. Así que el p99 del compuesto se acerca más al p99.x individual — la *tail latency* se amplifica con el fan-out. Mitigación: hedged requests, timeouts agresivos, menos dependencias en el camino crítico.

**2.3** (a) **"¿Qué le cuesta al negocio un minuto de downtime?"** — cinco nueves (~5 min/año) cuesta ~10× más que tres nueves (~8.76 h/año) en ingeniería; solo se justifica si el downtime cuesta una fortuna (pagos, trading). (b) **"¿Sobre qué SLI exactamente, y medido desde dónde?"** — cinco nueves de disponibilidad del server no sirven si la red del usuario o un tercero del que dependés tienen menos. Se pregunta para no firmar un objetivo carísimo o imposible sin entender el costo/beneficio.

**2.4** Escrituras/s: 500 M/día ÷ 10⁵ s ≈ **~5 000 escrituras/s** (promedio; el pico sería ×2–10). Storage/año: 500 M × 300 B = 1.5×10¹¹ B ≈ **~150 GB/día**; × 365 ≈ **~55 TB/año** (solo el texto, sin índices ni media). La conclusión que *conduce* el diseño: 55 TB/año no entra cómodo en un solo nodo a largo plazo → vas a necesitar sharding o un store distribuido; pero 5 000 escrituras/s todavía es manejable. El número cambió la decisión. (Ojo: el acortador del Módulo 12 da ~40 escrituras/s con la misma receta — otro volumen y otro ratio. El número siempre **depende del problema**; por eso se estima, no se memoriza.)

**2.5** Cambia la decisión de **escalado de datos**: ~50 escrituras/s es bajísimo → **no shardeás** (una primaria aguanta de sobra), y ~5000 lecturas/s con ratio 100:1 dice que es **lectura-intensivo** → invertís en **caché + read replicas**, no en particionar escrituras. Lo que NO hacía falta calcular con precisión: el ancho de banda exacto o el storage al decimal, porque a esta escala ninguno cambia esa decisión. Estimás lo que decide; lo demás es ruido del reloj.

### Módulo 3
**3.1** Vertical = una máquina más grande (simple, con techo, single point of failure). Horizontal = más máquinas (sin techo, tolera fallas). El horizontal exige que el servicio sea **stateless**: nada de estado en memoria local, porque cualquier instancia debe poder atender cualquier request.

**3.2** El carrito está en memoria local de una instancia; con 3 instancias detrás del load balancer, cada request del usuario puede caer en una instancia distinta que no tiene su carrito → "se vacía". Solución: sacar el estado afuera — guardar el carrito en **Redis** (o en la base), de modo que las 3 instancias lo compartan. (Alternativa peor: *sticky sessions* atando al usuario a una instancia, pero rompe el balanceo y la tolerancia a fallas.)

**3.3** Cuando necesitás máxima performance/throughput y no te importa el contenido: balanceo TCP puro (ej. tráfico de base de datos, conexiones de larga vida, protocolos no-HTTP), o cuando querés que el TLS lo terminen los servicios y no el LB. L4 es más rápido y simple porque no inspecciona el paquete; pagás flexibilidad (no podés rutear por path/header).

### Módulo 4
**4.1** Una read replica escala las **lecturas** (repartís reads entre réplicas), pero NO las **escrituras** (siguen yendo al primario). El sharding parte los datos por una key entre varias bases y es lo que escala las **escrituras**, a costa de complejidad (queries cross-shard, rebalanceo, hot partitions).

**4.2** `country` distribuye muy desparejo: el shard de un país grande (o donde tengas más usuarios) recibe muchísimo más tráfico que el de uno chico → **hot partition** (un nodo saturado mientras otros están ociosos). Mejor una key de alta cardinalidad y reparto parejo, como `hash(user_id)`, que distribuye uniforme sin importar la geografía.

**4.3** Causa: **replication lag**. La escritura fue al primario, pero la lectura del reload pegó a una réplica que todavía no recibió ese cambio. Mitigaciones (dos): (a) **read-your-writes**: rutear las lecturas de un usuario que *acaba* de escribir al primario (o a la réplica ya actualizada) por un tiempo corto; (b) leer del **primario** los datos críticos sensibles a frescura, o esperar confirmación de replicación (consistencia más fuerte a costa de latencia).

### Módulo 5
**5.1** **Cache-aside:** la app consulta el caché y, si hay miss, lee de la base y puebla el caché — trade-off: la primera lectura es lenta y puede haber staleness. **Write-through:** se escribe a caché y base juntas — trade-off: caché siempre fresco, pero cada escritura es más lenta.

**5.2** Es un **cache stampede** (o *thundering herd*): al expirar la key, todas las requests concurrentes hacen miss y pegan a la base a la vez. Prevención: un **lock** para que solo una request recompute la key mientras las demás esperan o sirven el valor viejo; o **early/probabilistic recomputation** (recalcular *antes* de que expire); o jitter en los TTL para que no expiren todas juntas.

**5.3** Porque cachear es solo "guardar una copia", pero invalidar exige saber **exactamente cuándo** esa copia dejó de ser válida — y los datos cambian desde muchos lugares. Ejemplo de bug sutil: cacheás el perfil de un usuario; un proceso de admin actualiza el email directo en la base sin pasar por el código que invalida la key → la app sirve el email viejo desde caché por horas, y nadie entiende por qué "no se actualizó". La fuente de la escritura se olvidó de invalidar.

### Módulo 6
**6.1** PACELC agrega el caso **sin** partición: CAP solo habla del trade-off cuando hay una partición de red (C vs A), pero PACELC dice que **Else** (cuando todo está sano) igual elegís entre **L**atencia y **C**onsistencia. O sea, el trade-off consistencia/latencia existe siempre, no solo en fallas.

**6.2** (a) Saldo de billetera → **fuerte**: servir un saldo viejo permite gastar dinero que no existe. (b) Cantidad de retweets → **eventual**: que el número tarde un segundo en cuadrar no le importa a nadie y exigir fuerte sería carísimo. (c) Username disponible al registrarte → **fuerte** en el momento del commit: necesitás una garantía de unicidad (una constraint única en la base), porque dos personas no pueden quedarse con el mismo username; mostrar "disponible" de forma eventual y después fallar es mala UX, pero la *verdad* final tiene que ser fuerte.

**6.3** Porque consistencia fuerte en todo significa pagar coordinación (latencia, menor disponibilidad, menor escala) **incluso para datos a los que no les importa** (contadores, feeds, "visto por última vez"). Es sobre-ingeniería: gastás el presupuesto de performance en garantías que nadie necesitaba. Un buen diseño usa fuerte donde duele y eventual donde alcanza; "fuerte en todo" suele delatar que no se pensó por dato.

### Módulo 7
**7.1** Cuando es comunicación **interna** service-to-service de alto volumen donde querés contratos tipados (Protobuf), payloads binarios eficientes y baja latencia (HTTP/2, multiplexing, streaming). REST gana en APIs públicas/de browser por universalidad y simplicidad; gRPC gana puertas adentro.

**7.2** (a) Validar stock → **sync** (necesitás saber si hay antes de cobrar). (b) Cobrar la tarjeta → **sync** (el resultado decide si el pedido se confirma). (c) Mandar el email → **async** (puede pasar después; que el broker lo reintente). (d) Analytics → **async** (no debe frenar ni romper el checkout jamás). Regla: lo que el usuario necesita para que el checkout responda va sync; lo demás, async.

**7.3** Porque las disponibilidades se **multiplican** en una cadena síncrona: si A necesita a B que necesita a C..., todos tienen que estar arriba a la vez. Con 5 servicios al 99.9%: `0.999⁵ ≈ 0.995` → **99.5%**, o sea ~43.7 h de downtime/año, mucho peor que el 99.9% (~8.76 h) de cada uno. Por eso conviene cortar dependencias síncronas del camino crítico (caché, async, defaults).

### Módulo 8
**8.1** Porque garantizar exactly-once de punta a punta en un sistema distribuido (con fallas de red, reinicios y duplicados) es prácticamente imposible sin un costo prohibitivo. Lo que se usa es **at-least-once + consumidores idempotentes** = "effectively-once": aceptás que puede llegar duplicado y hacés que reprocesar no tenga efecto.

**8.2** (a) `SET saldo = 100` ya es **idempotente**: ejecutarla 1 o 5 veces deja el saldo en 100. (b) `saldo = saldo + 50` **no** lo es: cada ejecución suma de nuevo (un duplicado cobra 50 de más). Para hacerla idempotente: registrar una **idempotency key** por operación (ej. el id del mensaje/transacción) y aplicar el `+50` solo si esa key no fue procesada antes.
```ts
// (concepto de la solución de 8.3 aplicado a este caso)
```

**8.3**
```ts
interface IdempotencyStore {
  exists(key: string): Promise<boolean>;
  set(key: string, value: string, opts?: { ttlSeconds: number }): Promise<void>;
}

async function procesarUna(
  msg: { id: string; amount: number },
  store: IdempotencyStore,
  efecto: (amount: number) => Promise<void>,
): Promise<"aplicado" | "duplicado"> {
  const key = `msg:${msg.id}`;
  if (await store.exists(key)) return "duplicado";
  await efecto(msg.amount);
  await store.set(key, "done", { ttlSeconds: 86_400 });
  return "aplicado";
}
// Nota: entre el exists y el set hay una ventana de carrera; en prod se cierra con
// una operación atómica (SET NX en Redis, o un INSERT con PK única que falle en duplicado).
```

### Módulo 9
**9.1** Un problema **organizacional / de escala de equipos**: permiten que muchos equipos desarrollen, deployen y escalen partes del sistema de forma independiente sin pisarse. No son una optimización de performance (de hecho, agregan latencia de red).

**9.2** Les decís que con 4 devs, 12 microservicios significan 12 deploys, 12 bases, red entre todos, consistencia distribuida y un infierno operativo — toda la complejidad de microservicios sin el beneficio (que es escalar equipos que no tienen). Propuesta: un **monolito modular** con límites internos claros (bounded contexts), que les permite moverse rápido ahora y extraer servicios *después*, cuando una parte tenga una razón real para separarse.

**9.3** (a) **Escala muy distinta:** un módulo necesita 10× los recursos del resto y escalarlo aparte ahorra plata. (b) **Equipo dedicado:** un equipo entero trabaja solo en ese módulo y el deploy compartido lo frena. (c) **Ciclo de release independiente:** ese módulo necesita deployar 20 veces al día mientras el resto deploya una vez por semana (o tiene requisitos de compliance/aislamiento distintos).

### Módulo 10
**10.1** Un circuit breaker corta las llamadas a un servicio que está fallando, para no seguir golpeándolo y para fallar rápido. Estados: **closed** (todo normal, pasan las llamadas), **open** (demasiados fallos → se rechazan las llamadas sin intentar, por un tiempo), **half-open** (pasado ese tiempo, deja pasar una prueba; si funciona vuelve a closed, si falla vuelve a open).

**10.2** Sin backoff, los clientes reintentan inmediatamente y en masa justo cuando el servicio ya está sobrecargado → le agregan *más* carga y lo terminan de tumbar (efecto bola de nieve). Sin jitter, aunque haya backoff, todos reintentan en el **mismo instante** (1s, 2s, 4s sincronizados) → picos coordinados que reproducen el problema. El **retry storm** convierte una falla transitoria en una caída total. Backoff espacía en el tiempo; jitter desincroniza a los clientes entre sí.

**10.3**
```ts
function backoffMs(intento: number, baseMs = 100, maxMs = 10_000): number {
  const exp = Math.min(maxMs, baseMs * 2 ** intento);
  return Math.round(Math.random() * exp);
}
function esperar(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}
async function reintentarConBackoff<T>(
  fn: () => Promise<T>,
  maxIntentos: number,
): Promise<T> {
  if (maxIntentos < 1) throw new RangeError("maxIntentos debe ser >= 1");
  let ultimoError: unknown;
  for (let intento = 0; intento < maxIntentos; intento++) {
    try {
      return await fn();
    } catch (e) {
      ultimoError = e;
      if (intento < maxIntentos - 1) await esperar(backoffMs(intento));
    }
  }
  throw ultimoError;
}
```

### Módulo 11
**11.1** **Logs** (eventos discretos: "¿qué pasó exactamente acá?"), **métricas** (números agregados en el tiempo: "¿está sano el sistema? ¿cuál es la tendencia?") y **traces** (el viaje de una request entre servicios: "¿dónde se fue el tiempo / dónde falló en la cadena?").

**11.2** Porque la latencia p99 sobre el SLO es un **síntoma** que el usuario realmente siente; el CPU al 90% es una **causa posible** que muchas veces es normal (un servicio bien aprovechado puede vivir al 90% sin problema). Alertar sobre causas genera ruido y falsas alarmas. El CPU sí sería buena alerta como señal de **saturación** predictiva (ej. "CPU sostenido al 95% y subiendo" para anticipar que te vas a quedar sin capacidad), pero como complemento, no como la alerta principal orientada al usuario.

**11.3** **Tracing distribuido** (OpenTelemetry). Las métricas por servicio muestran cada uno "sano" en promedio, pero no ven el camino completo de *esa* request: el tiempo puede irse en los saltos de red, en una cola, o en la suma de muchas latencias chicas. Un trace reconstruye la request entera con cada span y su duración, y ahí ves exactamente dónde se acumuló el tiempo.

### Módulo 12
**12.1** Es lectura-intensivo porque un URL se crea **una vez** pero se visita (redirige) **muchísimas** veces — ratio típico 100:1 o más. Por eso las dos piezas clave son el **caché** (Redis con `shortCode → longUrl`, hit rate altísimo) y las **read replicas** de la base (más una CDN delante de los redirects populares). Optimizás el camino de lectura, que es el caliente.

**12.2** El contador global autoincremental es un **único cuello de botella y punto de falla**: cada escritura necesita pedirle el próximo número, serializando todo. Solución sin perder unicidad: **rangos pre-asignados** — cada instancia toma un bloque (ej. ids 1000–1999) de un coordinador, los reparte localmente sin coordinación, y pide otro bloque cuando se agota. Sigue siendo único (los rangos no se solapan) y elimina el cuello de botella por request.

**12.3** Si querés contar clicks, necesitás que **cada visita pegue a tu servidor**, así que usás **302 (temporal)**: el browser no cachea el redirect y vuelve a consultarte cada vez, dándote el evento para contar (y la libertad de cambiar el destino). El **301 (permanente)** hace que el browser cachee el redirect y vaya directo al URL largo sin pasar por vos → menos carga, pero perdés la visibilidad de los clicks. Es el trade-off carga vs analytics.

**12.4**
```ts
class TokenBucket {
  private tokens: number;
  constructor(
    private readonly capacity: number,
    private readonly refillPerSec: number,
    private lastRefill: number,
  ) {
    this.tokens = capacity;
  }
  private refill(ahoraMs: number): void {
    const elapsedSec = (ahoraMs - this.lastRefill) / 1000;
    this.tokens = Math.min(this.capacity, this.tokens + elapsedSec * this.refillPerSec);
    this.lastRefill = ahoraMs;
  }
  permitir(ahoraMs: number): boolean {
    this.refill(ahoraMs);
    if (this.tokens >= 1) {
      this.tokens -= 1;
      return true;
    }
    return false;
  }
  tokensDisponibles(ahoraMs: number): number {
    this.refill(ahoraMs);     // aplica el refill pero no consume
    return Math.floor(this.tokens);
  }
}
```

---

## Siguientes pasos

Cuando estos 12 módulos te salgan fluidos, tenés lo que de verdad evalúan en una entrevista de diseño senior: **un método para encarar cualquier problema y el criterio para justificar cada trade-off.** Recomendado a continuación:

- **Practicá el método en voz alta** con problemas clásicos: feed de red social, chat en tiempo real, sistema de notificaciones, rate limiter distribuido, "diseñá YouTube/Uber/WhatsApp". Lo importante no es la respuesta, es el recorrido (requisitos → estimación → diseño → trade-offs).
- **Auto-evaluate con el [checklist de entrevista](system-design-checklist.md):** los conceptos senior/staff que el interviewer inyecta (consenso, quórums, watermarks, consistent hashing, etc.), cada uno resuelto en 4 capas, con el mapa hacia el módulo que lo profundiza.
- **Profundizá donde el método se queda corto:** este módulo da el criterio; los cinco módulos del roadmap de system design lo llevan a fondo — [Consenso](consenso.md), [Replicación](replicacion.md), [Streams](streams.md), [Datos a escala](datos-escala.md) y [Building blocks](building-blocks.md), cada uno con su laboratorio.
- **Practicá con casos resueltos:** el [Banco de casos](system-design-casos.md) aplica este mismo método de punta a punta en 8 sistemas (feed, chat, typeahead, pagos, etc.).
- **Conectá con los módulos del hub:** cada decisión de este módulo se implementa en otro — caché en [Redis](redis.md), eventos en [Event-driven](event-driven.md), datos en [PostgreSQL](postgresql.md), operación en [Observabilidad](observabilidad.md), orquestación en [Kubernetes](kubernetes.md).
- **Diseñá tu propio proyecto del portfolio "a escala imaginaria":** tomá tu "Task API" y escribí un documento de una página respondiendo "¿qué cambiaría si tuviera 10M de usuarios?". Ese ejercicio, hecho con honestidad (sin sobre-diseñar), demuestra criterio senior.
