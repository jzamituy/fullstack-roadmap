# Networking y protocolos a escala: lo que pasa debajo de tu API

**L4 vs L7, el costo de una conexión, HTTP/1.1→2→3 (QUIC), keep-alive y pools, y cómo el tráfico llega a la región correcta · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y recién después contrastá al final. Este módulo es el **system design de la red**: no la config de nginx (eso está en [Reverse proxy](reverse-proxy.md)) ni el contrato de la API (eso está en [Diseño de APIs](api-design.md)), sino **qué pasa por debajo** cuando un request cruza internet hasta tu backend, y por qué ese "por debajo" decide latencia, escala y resiliencia. Es la capa que casi nadie estudia ordenada y que el entrevistador staff sí pregunta: *"gRPC va sobre HTTP/2 — ¿y qué te da HTTP/2 exactamente?"*, *"¿por qué tu p99 se dispara cuando el pool se vacía?"*, *"un usuario en Tokio, ¿cómo llega a tu región más cercana y qué pasa cuando esa región se cae?"*.

**Lo que asumimos.** Que sabés qué es HTTP (request/response, métodos, status, headers), qué es TCP por arriba (conexión, IP:puerto) y que desplegaste algo detrás de un proxy. No re-explicamos nginx (está en [Reverse proxy](reverse-proxy.md)) ni multi-región (está en [Replicación](replicacion.md)): acá conectamos los puntos a nivel diseño.

> **¿Te falta alguna base?** Si "reverse proxy", "load balancer" o "terminar TLS" te suenan nuevos, hacé primero [Reverse proxy](reverse-proxy.md) módulos 1, 5 y 7 — este módulo es la **teoría de sistemas** detrás de esa config. Si nunca viste latencias intra-AZ vs cross-región, el módulo 2 de [Diseño de sistemas](system-design.md) (la matemática de servilleta) las ancla.

> ⚠️ **Nota sobre datos volátiles.** La adopción de HTTP/3, el soporte de QUIC en librerías y balanceadores, y los defaults de TLS se mueven rápido. Todo lo marcado con ⚠️ verificalo en la fuente (los RFCs: HTTP/2 = RFC 9113, HTTP/3 = RFC 9114, QUIC = RFC 9000, TLS 1.3 = RFC 8446; `caniuse.com` para soporte de browser; la doc de tu CDN/LB). Los conceptos no cambian; los números de adopción y los flags sí.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís/corregís código).

**El marco que usamos en las fallas.** Para cada modo de falla de red aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala/tráfico, (3) **control de corto plazo** que frena el *blast radius*, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**.

**Índice de módulos**
1. Las capas que importan: L4 vs L7
2. El costo de una conexión: TCP + TLS handshake
3. Keep-alive y connection pooling: amortizar el handshake
4. HTTP/1.1: head-of-line blocking a nivel aplicación
5. HTTP/2: multiplexing sobre una sola conexión (lo que aprovecha gRPC)
6. HTTP/3 y QUIC: matar el head-of-line blocking de TCP
7. Distribución global del tráfico: GeoDNS, anycast y GSLB
8. El criterio: cuándo esto te importa y cuándo te lo da la plataforma

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Las capas que importan: L4 vs L7

**Teoría.** Cuando un balanceador o un proxy se para en el medio del tráfico, la pregunta que lo define todo es: **¿qué tan adentro del paquete mira?** Eso es la diferencia entre **L4** y **L7** — los números vienen del modelo OSI, pero en la práctica solo te importan dos.

- **L4 (capa de transporte — TCP/UDP).** El balanceador ve **IP de origen, IP de destino, puertos**. **No** ve la URL, ni los headers, ni si es HTTP, gRPC o un protocolo binario propio. Toma la decisión de a qué backend mandar **mirando solo la 4-tupla** (a veces con un hash de origen para *stickiness*), y de ahí en más **reenvía los bytes tal cual**, sin entender qué transportan. Es el **NLB** de AWS, `ipvs`, HAProxy en modo `tcp`.
- **L7 (capa de aplicación — HTTP).** El balanceador **entiende el protocolo de aplicación**: parsea la request HTTP, ve el **path, los headers, el método, las cookies**. Puede **rutear por contenido** (`/api/*` a un servicio, `/img/*` a otro), terminar TLS, reescribir headers, hacer retries inteligentes, agregar/quitar campos. Es el **ALB** de AWS, nginx, Envoy, un API gateway.

La analogía: el **L4 es el cartero que reparte por dirección** — mira el sobre (IP:puerto), lo deja en la puerta correcta y **nunca lo abre**. El **L7 es la recepción que abre el sobre, lee el asunto y decide a qué departamento mandarlo** (y de paso puede traducirlo, sellarlo o rechazarlo).

El trade-off no es "uno es mejor": es **velocidad y opacidad (L4) vs inteligencia y costo (L7)**.

| | **L4** | **L7** |
|---|---|---|
| Qué ve | IP:puerto (la 4-tupla) | el request HTTP entero (path, headers, método) |
| Decisión de ruteo | por origen/destino + hash | por contenido (path, host, header, cookie) |
| Termina TLS | **no** (pasa el cifrado tal cual) | **sí** (descifra para leer el HTTP) |
| Costo / latencia | mínimo, casi line-rate | mayor (parsear, a veces descifrar) |
| Protocolo | **agnóstico** (cualquier cosa sobre TCP/UDP) | atado a HTTP/gRPC/WebSocket |
| Ejemplos | AWS NLB, IPVS, HAProxy `tcp` | AWS ALB, nginx, Envoy, API gateway |

**Cuándo cada uno.** Querés **L4** cuando necesitás throughput extremo, baja latencia, o pasar **TLS de punta a punta** sin terminarlo en el borde (el backend ve el cifrado original — *TLS passthrough*), o cuando el protocolo **no es HTTP** (una base de datos, un broker, un juego sobre UDP). Querés **L7** cuando necesitás **rutear por contenido**, terminar TLS para inspeccionar, aplicar auth/rate-limit/transformaciones en el borde, o hacer *canary* por header. En sistemas grandes es común **ambos en capas**: un L4 al frente repartiendo conexiones a una flota de L7, y los L7 haciendo el trabajo fino.

> Puente con [Reverse proxy](reverse-proxy.md): el módulo 7 de ahí ordena la familia proxy/LB/gateway/CDN. Acá agregamos el eje que ese módulo solo menciona de pasada: **dónde mira cada uno**. Un nginx haciendo `proxy_pass` con `proxy_set_header` es L7 puro (lee y reescribe HTTP); un NLB que solo reparte conexiones TCP es L4.

La frase mental: **L4 mira el sobre (IP:puerto) y reenvía a ciegas — rápido, agnóstico, no termina TLS. L7 abre el sobre y lee el HTTP — rutea por path/header, termina TLS, hace el trabajo fino, pero cuesta más. A escala, los combinás: L4 al frente, L7 atrás.**

**Ejercicios 1**
1.1 🔁 ¿Qué ve un balanceador L4 y qué ve uno L7? ¿Cuál puede rutear `/api/*` a un backend distinto que `/img/*`?
1.2 🧠 Querés que el TLS llegue **cifrado hasta el backend** (passthrough), sin terminarlo en el borde. ¿Elegís L4 o L7? ¿Por qué el otro no sirve para eso?
1.3 🧠 Tenés que balancear tráfico de **PostgreSQL** entre réplicas de lectura. ¿L4 o L7? Justificá pensando en qué protocolo es.

---

## Módulo 2 — El costo de una conexión: TCP + TLS handshake

**Teoría.** Antes de que viaje **un solo byte** de tu request, abrir una conexión nueva cuesta varios **round-trips** (RTT = ida y vuelta hasta el server). Y a escala, ese costo —que en local es invisible— **domina la latencia**.

Los apretones de manos, en orden:

1. **TCP handshake (3-way):** SYN → SYN-ACK → ACK. Cuesta **1 RTT** antes de poder mandar datos.
2. **TLS handshake (si es HTTPS):**
   - **TLS 1.2:** **2 RTT** adicionales (intercambio de claves + certificados).
   - **TLS 1.3:** **1 RTT** (RFC 8446 lo recortó a la mitad). Y con **0-RTT resumption** (sesión ya conocida), el cliente puede mandar datos **en el primer paquete** — 0 RTT de handshake, a costa de un riesgo de *replay* en requests no idempotentes (⚠️ por eso 0-RTT solo para GETs/idempotentes).

Hacé la cuenta (la matemática de servilleta de [system design](system-design.md) M2). Con un RTT de **100ms** (un usuario cruzando un océano hasta tu región):

| | TCP | + TLS 1.2 | + TLS 1.3 | + TLS 1.3 0-RTT |
|---|---|---|---|---|
| RTTs de handshake | 1 | 3 | 2 | 1 (TCP) |
| Latencia antes del 1er byte | 100ms | **300ms** | **200ms** | **100ms** |

**300ms tirados antes de empezar.** Si cada request abre una conexión nueva, **pagás esto en cada request** — y la transferencia de datos en sí podría ser de 5ms. Ahí está el origen de dos verdades de diseño:

- **Reusá conexiones** (módulo 3): amortizás el handshake sobre muchos requests.
- **Acercá el server al usuario** (módulo 7): si el RTT baja de 100ms a 5ms (un PoP cercano), el handshake deja de doler. Por eso los CDN no solo cachean: **terminan TLS cerca del usuario** y el handshake caro pasa sobre el "último kilómetro" corto.

> 🔥 **Falla en 4 capas — el "connection storm" / tormenta de handshakes.**
> (1) **Qué se rompe:** una flota de clientes pierde sus conexiones a la vez (un deploy reinició el pool, un LB hizo *failover*, expiró un keep-alive masivo) y **todos reabren al mismo tiempo** → un pico de handshakes TCP+TLS que satura CPU del borde (el TLS handshake es **caro en CPU**: criptografía asimétrica) y dispara el p99.
> (2) **Por qué a esta escala:** con miles de clientes, el costo no es el dato sino el **establecimiento sincronizado** — el *thundering herd* de conexiones.
> (3) **Corto plazo:** *jitter* en los reintentos de reconexión (no todos a la vez), *connection warm-up* gradual, y límites de *new connections/s* en el borde.
> (4) **Largo plazo:** conexiones **persistentes y de larga vida** (keep-alive agresivo), **session resumption** de TLS (handshake barato al reconectar), y escalonar deploys para no cortar todo el pool de golpe.

La frase mental: **antes del primer byte pagás RTTs: 1 (TCP) + 2 (TLS 1.2) o + 1 (TLS 1.3), y 0-RTT si reanudás sesión. A 100ms de RTT eso son 100–300ms tirados por conexión. La defensa: reusá conexiones y acercá el server (TLS cerca del usuario).**

**Ejercicios 2**
2.1 🔁 ¿Cuántos RTT cuesta abrir una conexión HTTPS con TLS 1.2? ¿Y con TLS 1.3? ¿Qué hace 0-RTT y cuál es su riesgo?
2.2 🧠 Tu API tiene un p99 altísimo solo para clientes lejanos, aunque el handler responde en 3ms. El tráfico abre una conexión nueva por request. ¿Qué está pasando y cuáles son las dos palancas para arreglarlo?
2.3 🧠 ¿Por qué 0-RTT de TLS 1.3 es seguro para un `GET /productos` pero peligroso para un `POST /transferencias`? (pista: replay)

---

## Módulo 3 — Keep-alive y connection pooling: amortizar el handshake

**Teoría.** Si abrir una conexión cuesta 1–3 RTT (módulo 2), la respuesta obvia es **no cerrarla**: reusarla para muchos requests. Eso es **keep-alive** (mantener la conexión TCP viva tras la respuesta) y **connection pooling** (mantener un **conjunto** de conexiones abiertas listas para usar).

- **Keep-alive** (HTTP persistent connections): tras responder, el server **no cierra** el socket; el siguiente request del mismo cliente lo reusa, **saltándose el handshake**. Es el default de HTTP/1.1. Lo controla el header `Connection: keep-alive` y *timeouts* a ambos lados.
- **Connection pool:** del lado del **cliente** (tu backend llamando a otro servicio, a la base, a Redis), mantenés N conexiones abiertas. Cuando necesitás hacer una llamada, **tomás una del pool** (`acquire`), la usás y la **devolvés** (`release`). Si todas están ocupadas, **esperás** (o fallás con timeout). Es exactamente el patrón de un **connection pool de base de datos** (PgBouncer, el pool de `pg`), de un **HTTP agent** (`http.Agent` de Node con `keepAlive: true`), o de un **gRPC channel** (que multiplexea muchas llamadas sobre pocas conexiones HTTP/2 — módulo 5).

El parámetro que importa es el **tamaño del pool**, y tiene dos fallas simétricas:

- **Pool muy chico:** bajo carga, todas las conexiones están ocupadas, los requests **se encolan esperando una libre** → la latencia se dispara aunque el backend esté ocioso. Es el **pool exhaustion**.
- **Pool muy grande:** abrís más conexiones de las que el destino aguanta. Una base de datos tiene un límite duro de conexiones (cada una cuesta memoria/un proceso); 50 instancias de tu app × 100 conexiones cada una = 5000 conexiones contra una DB que aguanta 500 → **la tumbás**. Acá entra **PgBouncer** (un pooler delante de Postgres que multiplexa miles de clientes sobre pocas conexiones reales).

> Puente con [Reverse proxy](reverse-proxy.md) M4: el `keepalive 32;` del bloque `upstream` de nginx es **esto** — el pool de conexiones reusadas de nginx **hacia** tu Node, que por eso necesita `proxy_http_version 1.1` y vaciar el header `Connection`.

> Puente con [Redis](redis.md) y [PostgreSQL](postgresql.md): los pools del lado cliente son la razón por la que tu app no abre una conexión nueva por query. El tamaño del pool de la DB es uno de los números que más se configura mal en producción.

La frase mental: **no cierres la conexión: keep-alive la reusa y te ahorra el handshake; el pool mantiene N abiertas y listas. El tamaño es el filo de la navaja: muy chico → requests encolados (pool exhaustion); muy grande → tumbás al destino (la DB tiene un límite duro). Un pooler como PgBouncer multiplexa miles de clientes sobre pocas conexiones reales.**

**Ejercicios 3**
3.1 🔁 ¿Qué ahorra el keep-alive? ¿Qué pasa cuando un connection pool se queda sin conexiones libres?
3.2 🧠 Tenés 40 instancias de tu API, cada una con un pool de 50 conexiones a la misma Postgres, que aguanta 400. ¿Qué problema tenés y cómo lo resolvés sin agrandar la DB?
3.3 ✍️ Implementá en TypeScript un `ConnectionPool<T>` con `acquire(): Promise<T>` y `release(conn: T): void`. El pool se inicializa con un **conjunto fijo de conexiones** (ese es su tope); si no hay conexiones libres al hacer `acquire`, el llamador **espera en una cola** hasta que alguien haga `release`. (La solución está al final; tiene que compilar en `--strict`.)

---

## Módulo 4 — HTTP/1.1: head-of-line blocking a nivel aplicación

**Teoría.** HTTP/1.1 tiene una limitación estructural: **una conexión TCP procesa un request a la vez**. Mandás un request, **esperás** la respuesta completa, recién entonces mandás el siguiente. El intento de arreglarlo —**pipelining** (mandar varios sin esperar)— exigía que el server respondiera **en orden**, así que si el primer request era lento, **bloqueaba a los de atrás**. Eso es **head-of-line (HOL) blocking a nivel aplicación**: el de adelante de la fila frena a toda la fila. El pipelining quedó **roto en la práctica** (los proxies lo manejaban mal) y nadie lo usa.

¿Cómo lo esquivaron los browsers? **Abriendo varias conexiones TCP en paralelo** al mismo host — típicamente **6 por dominio** (⚠️ el número varía por browser). Seis conexiones = seis requests en vuelo a la vez. Y como 6 era poco para una página con 100 recursos, nació el hack de **domain sharding**: servir los assets desde `img1.cdn.com`, `img2.cdn.com`, etc., para que el browser abriera 6 conexiones **por cada subdominio** y paralelizara más.

El costo de todo esto: **cada conexión paga su propio handshake** (módulo 2) y consume recursos en ambos lados. Seis conexiones = seis TCP+TLS handshakes. El domain sharding **multiplicaba** ese costo. Era un parche, no una solución — y es exactamente el problema que HTTP/2 vino a resolver (módulo 5), al punto que con HTTP/2 el domain sharding se vuelve **contraproducente**.

La frase mental: **HTTP/1.1 = un request a la vez por conexión; el pipelining para arreglarlo se rompió por HOL blocking a nivel app. Los browsers lo parchearon abriendo ~6 conexiones por host (y el hack de domain sharding para abrir más), pero cada conexión paga su handshake. HTTP/2 hace ese parche innecesario — y al sharding, dañino.**

**Ejercicios 4**
4.1 🔁 ¿Por qué se dice que HTTP/1.1 maneja "un request a la vez por conexión"? ¿Qué es el HOL blocking a nivel aplicación que tenía el pipelining?
4.2 🧠 ¿Por qué los browsers abren ~6 conexiones por dominio en HTTP/1.1, y qué problema de costo trae eso? ¿Qué era el domain sharding?
4.3 🧠 Adelanto: ¿por qué el domain sharding, que ayudaba en HTTP/1.1, se vuelve **contraproducente** con HTTP/2? (volvé a este después del módulo 5)

---

## Módulo 5 — HTTP/2: multiplexing sobre una sola conexión (lo que aprovecha gRPC)

**Teoría.** HTTP/2 (RFC 9113) resuelve el problema del módulo 4 con una idea central: **multiplexing**. Sobre **una sola conexión TCP**, podés tener **muchos requests/responses concurrentes** ("streams") **entrelazados**. Ya no necesitás 6 conexiones ni domain sharding — una alcanza, y los streams van y vienen en paralelo.

Cómo lo logra:

- **Binary framing.** HTTP/2 deja de ser texto: parte todo en **frames binarios** etiquetados con un **stream ID**. Frames de distintos streams se intercalan en el mismo cable y se reensamblan del otro lado. Por eso el de adelante ya no bloquea al de atrás **a nivel HTTP**.
- **HPACK (header compression).** Los headers HTTP son repetitivos y verbosos (cookies, user-agent, etc.). HPACK los comprime y mantiene una **tabla de headers ya vistos**, así no remandás lo mismo en cada request. A muchos requests chicos, esto es un ahorro grande.
- **Una conexión, muchos streams.** Un handshake amortizado sobre toda la sesión.
- **Server push:** existió (el server adelantaba recursos), pero **se deprecó** ⚠️ — los browsers lo retiraron porque rara vez ayudaba y era difícil de usar bien. No lo cuentes como ventaja viva.

**Acá se cierra el círculo con [api-design](api-design.md):** ese módulo dice "**gRPC va sobre HTTP/2**" sin explicar qué da HTTP/2. Esto es lo que da: **gRPC multiplexea miles de llamadas RPC concurrentes sobre pocas conexiones HTTP/2**, con framing binario (que encaja perfecto con Protobuf, también binario) y **streaming bidireccional** (los streams de HTTP/2 son full-duplex). Por eso gRPC es tan eficiente service-to-service: no abre una conexión por llamada, las **multiplexea**.

**Pero HTTP/2 no es la última palabra**, y el motivo es sutil: **el HOL blocking se mudó de capa.** HTTP/2 eliminó el HOL blocking **a nivel HTTP**, pero **TCP sigue siendo un único stream de bytes ordenado**. Si se **pierde un paquete TCP**, TCP **retiene todo lo que llegó después** hasta retransmitir el perdido —porque debe entregar en orden— y eso **frena a TODOS los streams de HTTP/2** que viajaban en esa conexión, aunque sus datos ya hubieran llegado. **HOL blocking a nivel transporte (TCP).** En una red con pérdida (móvil, wifi malo), HTTP/2 puede rendir **peor** que varias conexiones HTTP/1.1, porque puso todos los huevos en una canasta TCP. Ese es justo el problema que ataca HTTP/3 (módulo 6).

Y por eso (ejercicio 4.3) **el domain sharding se vuelve contraproducente con HTTP/2**: si abrís 6 subdominios, rompés el multiplexing en 6 conexiones separadas, cada una con su handshake y sin compartir la tabla HPACK — exactamente lo que HTTP/2 quería evitar.

Y de paso, el límite de **~6 conexiones por host** del módulo 4 **desaparece**: con HTTP/2 te alcanza **una** conexión multiplexada (por eso el domain sharding sobra).

La frase mental: **HTTP/2 = multiplexing: muchos streams concurrentes sobre UNA conexión TCP (binary framing + HPACK), lo que mata el HOL blocking a nivel HTTP y es exactamente lo que gRPC explota. Adiós a las ~6 conexiones por host: con una alcanza. Pero el HOL blocking se mudó a TCP: un paquete perdido frena TODOS los streams. Server push: deprecado.**

**Ejercicios 5**
5.1 🔁 ¿Qué es el multiplexing de HTTP/2 y qué dos mecanismos lo hacen posible (el framing y la compresión de headers)?
5.2 🧠 Cuando [api-design](api-design.md) dice "gRPC va sobre HTTP/2", ¿qué le da concretamente HTTP/2 a gRPC? Nombrá dos cosas.
5.3 🧠 HTTP/2 eliminó el HOL blocking… pero solo "movió" el problema. Explicá dónde quedó el HOL blocking y por qué en una red con pérdida de paquetes HTTP/2 puede ir peor que HTTP/1.1.

---

## Módulo 6 — HTTP/3 y QUIC: matar el head-of-line blocking de TCP

**Teoría.** HTTP/3 (RFC 9114) ataca la última pieza: el **HOL blocking de TCP** (módulo 5). Y para eso hace algo radical: **abandona TCP**. HTTP/3 corre sobre **QUIC** (RFC 9000), un protocolo de transporte construido **sobre UDP** que reimplementa lo bueno de TCP (entrega confiable, control de congestión) pero con **streams independientes**.

Las tres ganancias que importan:

- **Streams verdaderamente independientes (adiós HOL blocking de transporte).** En QUIC, cada stream tiene su **propio control de orden**. Si se pierde un paquete del stream A, **solo el stream A espera la retransmisión**; los streams B, C, D siguen entregándose. Esto es lo que TCP no podía hacer (un solo orden global). En redes con pérdida, HTTP/3 brilla justo donde HTTP/2 sufría.
- **Handshake integrado (transporte + cripto en 1 RTT, o 0-RTT).** QUIC **fusiona** el handshake de transporte con el de TLS 1.3 en **un solo apretón** → conexión establecida en **1 RTT**, y **0-RTT** al reconectar a un server conocido. (Compará con el módulo 2: TCP + TLS por separado eran 2–3 RTT.)
- **Connection migration.** Una conexión QUIC se identifica por un **connection ID**, no por la 4-tupla IP:puerto. Si tu celular **cambia de wifi a datos móviles** (cambia de IP), la conexión QUIC **sobrevive** — no hay que rehacer el handshake. Para apps móviles esto es enorme: cero cortes al moverte. (TCP, atado a la 4-tupla, te obliga a reconectar.)

El costo / los matices (⚠️ datos volátiles):
- Va sobre **UDP**, que algunas redes corporativas o firewalls **bloquean o degradan** → por eso los clientes hacen *fallback* a HTTP/2 sobre TCP. HTTP/3 es una **optimización con red de seguridad**, no un reemplazo total. El cliente **descubre** que el server habla HTTP/3 por el header **`Alt-Svc`** (o un registro DNS **HTTPS/SVCB**), así que la **primera** conexión casi siempre arranca por HTTP/2 sobre TCP y recién la siguiente migra a HTTP/3 — el fallback y el upgrade son transparentes.
- El cifrado es **obligatorio** (no hay QUIC en claro): siempre cifrado.
- Mover el trabajo de TCP (kernel) a QUIC (userspace, sobre UDP) puede costar **más CPU** ⚠️ — y ese costo **no es un impuesto fijo**: sobre enlaces de **alto ancho de banda** (fibra, >~500 Mbps) el overhead de CPU en userspace se vuelve el **cuello de botella**, y ahí HTTP/3 puede quedar **por debajo de HTTP/2 en throughput** ⚠️ (medido en *"QUIC is not Quick Enough over Fast Internet"* y réplicas posteriores). La síntesis: **HTTP/3 paga en latencia y pérdida, no en banda ancha.**

**El criterio.** Para una **API interna service-to-service** en tu propia red (baja pérdida, RTT bajo, y a menudo **alto ancho de banda**), HTTP/2 ya es óptimo — HTTP/3 no te compra casi nada y puede incluso rendir peor en throughput. HTTP/3 paga cuando el cliente está en una **red impredecible y con pérdida**: usuarios móviles, last-mile, internacional. Por eso lo primero que lo adoptó en serio fueron los **CDN y los browsers** (Google, Cloudflare, etc.), no las mallas internas.

La frase mental: **HTTP/3 = HTTP sobre QUIC (sobre UDP): streams independientes (un paquete perdido NO frena a los demás → mata el HOL blocking de TCP), handshake en 1 RTT / 0-RTT, y connection migration (sobrevive el cambio de wifi↔datos). Brilla en redes con pérdida (móvil, internacional); internamente HTTP/2 ya alcanza. Fallback a TCP si UDP está bloqueado.**

**Ejercicios 6**
6.1 🔁 ¿Sobre qué corre QUIC y cómo logra que un paquete perdido en un stream no frene a los demás? ¿Cuántos RTT cuesta su handshake?
6.2 🧠 ¿Qué es la connection migration de QUIC y por qué es una ganancia tan grande para apps móviles? ¿Por qué TCP no podía hacerlo?
6.3 🧠 Tenés dos sistemas: (a) 12 microservicios internos hablando gRPC en la misma VPC, (b) una app móvil global pegándole a tu API desde redes 4G dispares. ¿En cuál priorizás HTTP/3 y por qué en el otro casi no mueve la aguja?

---

## Módulo 7 — Distribución global del tráfico: GeoDNS, anycast y GSLB

**Teoría.** Tenés tu sistema replicado en varias regiones ([Replicación](replicacion.md) M16). Falta la pregunta de red: **¿cómo hacés que un usuario en Tokio pegue contra tu región de Tokio y no contra la de Virginia (cruzando un océano y sumando 150ms de RTT)?** Y la pregunta de resiliencia: **¿qué pasa con ese ruteo cuando una región se cae?** Tres mecanismos, que suelen combinarse.

- **GeoDNS (DNS geográfico).** El truco está en el **DNS**: cuando el usuario resuelve `api.tuapp.com`, el servidor DNS mira **de qué región viene la consulta** (por la IP del resolver) y **devuelve una IP distinta** según la geografía — la del PoP más cercano. Mismo nombre, distinta respuesta según dónde estés. Simple y muy usado.
  - **El talón de Aquiles: el TTL.** Las respuestas DNS se **cachean** (en el resolver del ISP, en el SO, en el browser) durante el **TTL**. Si una región se cae y querés redirigir el tráfico cambiando el DNS, **los clientes con la IP cacheada siguen yendo a la región muerta hasta que expire el TTL**. Por eso el DNS es un mecanismo de failover **lento e impreciso** (TTLs bajos ayudan pero no eliminan el problema, y muchos resolvers ignoran TTLs muy cortos ⚠️).
  - **El segundo talón: geolocaliza al resolver, no al usuario.** GeoDNS decide por la **IP del resolver recursivo**, no la del cliente final. Si el usuario usa un **resolver público o centralizado** (`8.8.8.8`, DoH), puede caer en la región **equivocada** porque el resolver está en otro lado. La mitigación es **EDNS Client Subnet (ECS)**, donde el resolver le pasa al DNS autoritativo un prefijo de la IP real del cliente — pero solo funciona si **ambos** lo soportan ⚠️.

- **Anycast.** Acá el truco está en el **ruteo de red (BGP)**, no en el DNS. Anunciás **la misma IP desde muchos sitios físicos** a la vez, y la **red de internet (BGP) rutea cada paquete al sitio más cercano** topológicamente. Un solo IP, decenas de ubicaciones; el usuario "cae" en la más cercana sin que el DNS decida nada. Es como funcionan los **DNS raíz**, los resolvers públicos (`1.1.1.1`, `8.8.8.8`) y el borde de los **CDN**.
  - **La ventaja sobre GeoDNS para failover:** si un sitio anycast se cae, **BGP deja de anunciar esa IP desde ahí** y los paquetes **se reenrutan solos** al siguiente sitio más cercano — **sin esperar TTLs**. Failover de red, mucho más rápido que el de DNS. (El matiz ⚠️: la *convergencia* de BGP no es instantánea y las conexiones en vuelo pueden cortarse, pero es órdenes de magnitud mejor que el caché de DNS.)

- **GSLB (Global Server Load Balancing).** Es la pieza que une todo con **conciencia de salud**: un balanceador global que conoce el **estado de cada región** (health checks), su **carga** y la **latencia** hacia el cliente, y dirige a la mejor región **sana** — normalmente **implementado vía DNS inteligente y/o anycast**. La diferencia con un GeoDNS pelado: el GSLB **saca de rotación una región caída** (no te manda a la que no responde) y puede balancear por carga, no solo por geografía. Los servicios gestionados de esto son Route 53 (con health checks + routing policies), Cloudflare Load Balancing, Akamai GTM, etc.

**Cómo se combinan en un sistema real.** Anycast lleva al usuario al **borde/CDN más cercano** (rápido, con failover de red). Ahí termina TLS cerca del usuario (módulo 2). El borde, si necesita ir al origin, usa **GSLB/health-aware routing** para elegir la **región de origin sana** más conveniente. Y GeoDNS sigue siendo válido para casos más simples donde no querés anycast.

> Puente con [Replicación](replicacion.md): ese módulo te dio el *qué* (active-passive vs active-active, RPO/RTO, promover la región pasiva). Este te da el *cómo llega el tráfico ahí* y por qué el failover por DNS es lento (el TTL) mientras el de anycast es rápido (BGP). La línea de M16.2 de replicación — "redirijo con DNS/anycast" — **es esto**.

> 🔥 **Falla en 4 capas — failover de región que "no toma" por el caché de DNS.**
> (1) **Qué se rompe:** se cae `us-east`, promovés `eu-west` y cambiás el registro DNS… pero **el tráfico sigue cayendo en la región muerta** durante minutos.
> (2) **Por qué a esta escala:** millones de clientes tienen la IP vieja **cacheada por el TTL** en resolvers que ni siquiera respetan TTLs cortos.
> (3) **Corto plazo:** TTLs bajos **preventivos** en los registros críticos, y para conexiones nuevas, *health-checked DNS* (Route 53) que deja de devolver la IP muerta.
> (4) **Largo plazo:** **anycast** para el failover (BGP reenruta sin esperar TTL) y/o un GSLB que combine health checks con anycast — no dependas del caché de DNS para tu RTO.

La frase mental: **para llevar al usuario a la región correcta: GeoDNS (el DNS responde una IP distinta según geo — simple, pero el failover es lento por el TTL cacheado); anycast (misma IP desde muchos sitios, BGP rutea al más cercano — failover rápido sin esperar TTL); GSLB (ruteo global health-aware que saca de rotación lo caído). En la práctica los combinás: anycast al borde + GSLB al origin.**

**Ejercicios 7**
7.1 🔁 Explicá en una frase cada uno: GeoDNS, anycast, GSLB. ¿Cuál usa el TTL del DNS y cuál usa BGP?
7.2 🧠 Hacés failover de región cambiando el registro DNS, pero el tráfico tarda minutos en moverse. ¿Por qué? ¿Qué mecanismo lo arregla y por qué es más rápido?
7.3 🧠 ¿Por qué un GeoDNS "pelado" puede mandarte a una región **caída** y un GSLB no? ¿Qué le agrega el GSLB?

---

## Módulo 8 — El criterio: cuándo esto te importa y cuándo te lo da la plataforma

**Teoría.** El cierre, igual que en [Reverse proxy](reverse-proxy.md) M10: aprendiste cómo funciona la capa de red, pero la pregunta senior es **cuánto de esto tenés que manejar vos**. Casi siempre, **menos de lo que parece** — pero saber **qué decisiones quedan abiertas** es lo que te separa.

**Te lo da la plataforma (no lo armás):**
- **El protocolo en el borde:** poné tu app detrás de un **CDN/ALB/Cloudflare** y obtenés HTTP/2 (y HTTP/3 ⚠️ donde esté disponible) hacia el cliente **sin tocar nada** — el borde negocia el mejor protocolo con cada cliente y te habla HTTP/1.1 o /2 por detrás. El handshake caro se termina cerca del usuario, gratis.
- **La distribución global:** Route 53 / Cloudflare LB te dan **GeoDNS + health-checked routing + anycast** como producto. No corrés BGP a mano.
- **TLS y su renovación:** ACM, cert-manager, Caddy (ver [Reverse proxy](reverse-proxy.md) M5/M8).

**Las decisiones que SÍ son tuyas, siempre:**
- **El tamaño de tus connection pools** (módulo 3) hacia la DB, Redis y servicios internos — nadie lo configura por vos, y mal puesto te tumba la base o te encola los requests. Es el error de red **#1** en producción.
- **L4 vs L7** para cada tramo (módulo 1): ¿terminás TLS en el borde o passthrough? ¿ruteás por path?
- **Cuándo HTTP/3 paga** (módulo 6): activarlo para clientes móviles/globales sí; obsesionarte con él para tráfico interno no.
- **No depender del DNS para tu RTO** (módulo 7): si tu plan de failover es "cambio el DNS", tu RTO real incluye el TTL cacheado — diseñá con anycast/GSLB si necesitás failover rápido.

La frase mental de cierre: **la plataforma te da el protocolo del borde, la distribución global y el TLS — no los reinventes. Lo que es tuyo y te rompe en producción: el tamaño de los pools, elegir L4 vs L7 por tramo, saber cuándo HTTP/3 paga, y no atar tu RTO al caché del DNS. Saber cómo funciona la capa no es para configurarla toda a mano: es para tomar esas cuatro decisiones bien.**

**Ejercicios 8**
8.1 🔁 Nombrá dos cosas de red que te da la plataforma sin que las armes, y dos decisiones que igual son tuyas.
8.2 🧠 ¿Cuál es, según este módulo, el error de configuración de red más común en producción y por qué duele?
8.3 🧠 Un compañero quiere "implementar HTTP/3 nosotros mismos en el balanceador interno entre microservicios". ¿Qué le decís, y qué le proponés en cambio?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Un **L4** ve solo la capa de transporte: **IP de origen/destino y puertos** (la 4-tupla), no el contenido. Un **L7** entiende el protocolo de aplicación: ve el **request HTTP entero** (path, método, headers, cookies). El que puede rutear `/api/*` distinto de `/img/*` es el **L7**, porque para eso hay que **leer el path**, y el L4 no lo ve.

**1.2** **L4**, con **TLS passthrough**: como el L4 no entiende ni descifra el contenido, **reenvía los bytes cifrados tal cual** hasta el backend, que termina el TLS. El **L7 no sirve para passthrough** porque para rutear por contenido **necesita descifrar** (terminar TLS) y leer el HTTP — si lo descifra en el borde, ya no llega cifrado al backend. (El L7 puede re-cifrar hacia el backend, pero entonces termina y reabre TLS; no es passthrough de la conexión original.)

**1.3** **L4.** PostgreSQL **no habla HTTP** — habla su propio protocolo binario sobre TCP. Un balanceador L7 está atado a HTTP/gRPC, así que no entiende el protocolo de Postgres; un **L4 es agnóstico al protocolo** (solo mira IP:puerto y reenvía bytes), así que puede balancear conexiones TCP a las réplicas sin entender qué transportan. (Para ruteo más fino —separar lecturas de escrituras— se usa un pooler/proxy que sí entiende el protocolo de Postgres, como PgBouncer/Pgpool, no un L7 HTTP.)

**2.1** **TLS 1.2:** 2 RTT de handshake TLS (sobre 1 RTT de TCP = 3 en total). **TLS 1.3:** 1 RTT de TLS (2 en total con TCP). **0-RTT:** al reconectar a un server ya conocido (session resumption), el cliente manda **datos en el primer paquete**, sin esperar el handshake → 0 RTT de handshake. **Riesgo:** los datos de 0-RTT son vulnerables a **replay** (un atacante puede reenviar ese primer paquete), así que solo se usa para requests **idempotentes** (GET), nunca para algo que muta estado.

**2.2** Está pagando el **handshake TCP+TLS en cada request**, porque abre una conexión nueva cada vez. Para clientes lejanos el RTT es alto (ej. 100ms), así que el handshake suma 100–300ms **antes** de que el handler de 3ms siquiera empiece — eso es el p99. Las **dos palancas**: (1) **reusar conexiones** (keep-alive + connection pool, módulo 3) para amortizar el handshake sobre muchos requests; (2) **acercar el server al usuario** (un PoP/CDN cercano que **termine TLS cerca**, módulos 2 y 7) para que el RTT del handshake sea de pocos ms, no de 100.

**2.3** Porque 0-RTT es vulnerable a **replay**: un atacante que capture el primer paquete puede **reenviarlo**. En un `GET /productos` reenviar el request no causa daño (es idempotente, leer dos veces da lo mismo). En un `POST /transferencias` un replay podría **ejecutar la transferencia dos veces** → mueve dinero de más. Por eso 0-RTT solo se habilita para métodos seguros/idempotentes. Ojo con el matiz fino: 0-RTT **no es "replay-proof"** ni siquiera con la anti-replay del protocolo — RFC 8446 §8 solo garantiza "a lo sumo una vez **por instancia de server**", así que el número de repeticiones queda acotado por la cantidad de instancias del deployment, no a una. Usarlo solo en idempotentes **acota el daño** (repetir no rompe nada); la protección del protocolo lo **limita pero no lo elimina**. (Conecta con la idempotencia de [api-design](api-design.md) M4: la defensa de fondo es una idempotency key, pero no le pases requests con efectos a 0-RTT.)

**3.1** El keep-alive **ahorra el handshake TCP+TLS**: en vez de abrir y cerrar una conexión por request, la mantiene viva y la reusa, saltándose 1–3 RTT por request siguiente. Cuando un connection pool se queda **sin conexiones libres** (todas ocupadas), los nuevos `acquire` **se encolan esperando** que alguien haga `release` (o fallan por timeout) → la latencia se dispara aunque el backend esté ocioso. Es el **pool exhaustion**.

**3.2** 40 instancias × 50 = **2000 conexiones potenciales** contra una DB que aguanta **400** → la **saturás** (rechaza conexiones / se queda sin memoria). Agrandar la DB no es la respuesta. Soluciones: (1) **bajar el tamaño de pool por instancia** (40 × 10 = 400, al límite — mejor con margen); (2) meter un **connection pooler** tipo **PgBouncer** delante de Postgres, que multiplexa miles de clientes sobre **pocas conexiones reales** a la DB (modo transaction pooling); (3) reducir el fan-out de instancias si es posible. La clave: el límite de conexiones de la DB es **duro y global**, y la suma de todos los pools no puede superarlo.

**3.3**
```ts
class ConnectionPool<T> {
  private readonly free: T[] = [];
  private readonly waiters: Array<(conn: T) => void> = [];

  // Recibe el conjunto fijo de conexiones ya creadas: el tope del pool es `connections.length`.
  constructor(connections: readonly T[]) {
    this.free = [...connections];
  }

  acquire(): Promise<T> {
    const conn = this.free.pop();
    if (conn !== undefined) {
      return Promise.resolve(conn);
    }
    // No hay libres: el llamador espera en la cola hasta un release.
    return new Promise<T>((resolve) => {
      this.waiters.push(resolve);
    });
  }

  release(conn: T): void {
    const waiter = this.waiters.shift();
    if (waiter !== undefined) {
      // Si alguien esperaba, le pasamos la conexión directo (no vuelve al pool).
      waiter(conn);
    } else {
      this.free.push(conn);
    }
  }
}
```
Idea: `free` son las conexiones disponibles; `waiters` es la cola FIFO de llamadores esperando. `acquire` toma una libre si hay; si no, devuelve una promesa que se resuelve cuando otro haga `release`. `release` se la entrega **directo** al primer waiter (si lo hay) en vez de devolverla al pool, así no hay ventana donde una conexión libre conviva con un llamador esperando. El **tope del pool es `connections.length`** (las conexiones que le pasás al constructor): no hay un parámetro `max` separado porque el conjunto inicial *es* el límite — nunca se crean más. Compila en `--strict` (incluido `--noUncheckedIndexedAccess`: por eso el chequeo `=== undefined` tras `pop()`/`shift()`, que pueden devolver `undefined`). En producción agregarías timeout en `acquire`, validación de conexiones muertas y, si quisieras un tope mayor que el conjunto inicial, **creación perezosa** hasta un `max` explícito.

**4.1** Porque en HTTP/1.1 una conexión TCP **procesa un request y espera su respuesta completa antes del siguiente** — serial. El **pipelining** permitía mandar varios sin esperar, pero **exigía respuestas en orden**: si el primero era lento, **bloqueaba a todos los de atrás** aunque ya estuvieran listos. Eso es el **HOL blocking a nivel aplicación** (el primero de la fila frena la fila). Por eso el pipelining quedó roto/sin usar.

**4.2** Porque con una request por conexión a la vez, abrir **~6 conexiones en paralelo** al mismo host deja **6 requests en vuelo simultáneos** — la única forma de paralelizar en HTTP/1.1. El **costo:** cada conexión paga su **propio handshake TCP+TLS** y consume recursos en ambos lados. El **domain sharding** era el hack de servir assets desde varios subdominios (`img1.`, `img2.`…) para que el browser abriera 6 conexiones **por subdominio** y paralelizara aún más — multiplicando el costo de handshakes.

**4.3** Porque HTTP/2 **multiplexa muchos streams sobre una sola conexión** (módulo 5). Si hacés domain sharding, **rompés ese multiplexing en 6 conexiones separadas**, cada una con su propio handshake y **sin compartir la tabla de compresión HPACK** — perdés justo lo que HTTP/2 vino a dar. Lo que en HTTP/1.1 era una optimización (más paralelismo) en HTTP/2 es un **anti-patrón** (fragmentás la conexión única). Con HTTP/2 querés **consolidar** en un dominio, no shardear.

**5.1** El **multiplexing** es tener **muchos requests/responses concurrentes ("streams") entrelazados sobre una sola conexión TCP**, sin que uno bloquee al otro a nivel HTTP. Lo hacen posibles: (1) **binary framing** — HTTP/2 parte todo en frames binarios etiquetados con un **stream ID**, que se intercalan en el cable y se reensamblan; (2) **HPACK** — compresión de headers con una tabla de los ya vistos, para no remandar headers repetidos en cada request.

**5.2** Dos de: (1) **multiplexing** — gRPC manda miles de llamadas RPC concurrentes sobre **pocas conexiones** en vez de una por llamada; (2) **framing binario** — encaja con Protobuf (también binario), eficiente de serializar; (3) **streaming bidireccional** — los streams de HTTP/2 son full-duplex, lo que habilita los modos streaming de gRPC; (4) **una conexión persistente** con handshake amortizado y HPACK comprimiendo metadata repetida. (Con cualquiera dos alcanza.)

**5.3** HTTP/2 eliminó el HOL blocking **a nivel HTTP** (los streams no se bloquean entre sí en la capa de aplicación), pero **TCP sigue siendo un único flujo de bytes ordenado**: si se **pierde un paquete TCP**, TCP **retiene todo lo posterior** hasta retransmitirlo (debe entregar en orden), y eso **frena a TODOS los streams** que viajaban en esa conexión, aunque sus datos ya hubieran llegado. Es **HOL blocking a nivel transporte (TCP)**. En una red con **pérdida de paquetes** (móvil, wifi malo), una sola conexión HTTP/2 puede ir **peor** que varias conexiones HTTP/1.1, porque todas las requests comparten el destino de esa única conexión TCP; con varias conexiones, una pérdida solo afecta a una. Eso es lo que HTTP/3 resuelve (módulo 6).

**6.1** QUIC corre sobre **UDP**. Logra streams independientes porque **cada stream maneja su propio orden/retransmisión**: un paquete perdido del stream A solo hace esperar al stream A, mientras B, C, D siguen entregándose (TCP no podía, porque impone un **único orden global**). Su **handshake fusiona transporte + TLS 1.3 en 1 RTT**, y **0-RTT** al reconectar a un server conocido.

**6.2** La **connection migration** es que una conexión QUIC se identifica por un **connection ID**, no por la 4-tupla IP:puerto. Si el dispositivo **cambia de red** (wifi → datos móviles) y por ende de IP, la conexión QUIC **sobrevive sin rehacer el handshake** — se sigue usando con el mismo connection ID. Para móviles es enorme: cero cortes ni re-handshakes al moverte (cambiar de celda, salir de casa, etc.). **TCP no puede** porque su conexión **está definida por la 4-tupla**: si cambia la IP, es —por definición— otra conexión, y hay que reconectar (nuevo handshake TCP+TLS).

**6.3** Priorizás HTTP/3 en **(b)**, la app móvil global: sus clientes están en **redes 4G dispares con pérdida y cambios de red**, justo donde QUIC paga — streams independientes (sin HOL blocking de TCP cuando hay pérdida), handshake en 1/0-RTT y **connection migration** (sobrevive al cambio de red). En **(a)**, microservicios en la misma VPC, la red tiene **baja pérdida, RTT bajo y alto ancho de banda**, no hay cambios de red, y HTTP/2 ya multiplexa eficiente: HTTP/3 casi no mueve la aguja, suma complejidad (UDP, más CPU ⚠️) y sobre esos enlaces rápidos hasta puede **rendir peor en throughput** (el overhead de CPU de QUIC en userspace se vuelve el cuello de botella, módulo 6). Regla: **HTTP/3 paga en el last-mile impredecible (latencia/pérdida), no en la malla interna de banda ancha.**

**7.1** **GeoDNS:** el servidor DNS devuelve una **IP distinta según la geografía** del que consulta, para mandarlo al PoP más cercano. **Anycast:** se anuncia la **misma IP desde muchos sitios** y la red (**BGP**) rutea cada paquete al sitio más cercano. **GSLB:** ruteo global **consciente de salud y carga** que dirige a la mejor región **sana** (vía DNS inteligente y/o anycast). El que usa el **TTL del DNS** es **GeoDNS**; el que usa **BGP** es **anycast**.

**7.2** Porque las respuestas DNS se **cachean por el TTL** en resolvers del ISP, el SO y el browser: tras cambiar el registro, los clientes con la IP vieja **siguen yendo a la región caída hasta que expire el TTL** (y muchos resolvers ignoran TTLs muy cortos ⚠️). Lo arregla **anycast**: como la IP es la misma desde muchos sitios, cuando uno se cae **BGP deja de anunciarla desde ahí y reenruta los paquetes solo** al siguiente sitio más cercano — **sin esperar ningún TTL**, porque el failover ocurre en la **capa de ruteo de red**, no en el caché de DNS.

**7.3** Un **GeoDNS pelado** solo sabe de **geografía**: devuelve la IP de la región más cercana **aunque esté caída**, porque no chequea su salud. Un **GSLB** le agrega **health checks** (y conciencia de carga/latencia): **saca de rotación la región que no responde** y devuelve solo regiones **sanas**, y puede balancear por carga, no solo por cercanía. En una frase: el GSLB es GeoDNS **+ salud + carga**, para no mandarte nunca a una región muerta.

**8.1** **Te da la plataforma:** el protocolo del borde (HTTP/2 y HTTP/3 ⚠️ negociados por el CDN/ALB sin tocar nada), la distribución global (Route 53/Cloudflare LB = GeoDNS + health routing + anycast), y TLS con su renovación (ACM/cert-manager/Caddy). **Son tuyas:** el tamaño de los connection pools, elegir L4 vs L7 por tramo, decidir cuándo activar HTTP/3, y no atar tu RTO al caché del DNS. (Con dos de cada lado alcanza.)

**8.2** El **tamaño de los connection pools** (módulo 3) hacia la DB/Redis/servicios. Duele en los dos extremos: **muy chico** → bajo carga los requests **se encolan** esperando una conexión libre y el p99 explota aunque el backend esté ocioso (pool exhaustion); **muy grande** → la suma de todos los pools **supera el límite duro de conexiones de la DB** y la tumbás. Nadie lo configura por vos y los defaults rara vez sirven a tu escala.

**8.3** Le digo que **no tiene sentido ahí**: entre microservicios en la misma red (baja pérdida, RTT bajo, sin cambios de red y alto ancho de banda) **HTTP/2 ya es óptimo** y HTTP/3 casi no mueve la aguja —e incluso puede **rendir peor en throughput** sobre enlaces rápidos, donde el overhead de CPU de QUIC en userspace se vuelve el cuello de botella ⚠️—, mientras suma complejidad real (UDP que algunos entornos degradan, librerías menos maduras para server-to-server). Además, "implementarlo nosotros" reinventa transporte serio — eso lo dan el CDN/LB/librerías, no se escribe a mano. **En cambio** le propongo: (1) dejar HTTP/2 internamente; (2) si querés ganar latencia real, **activar HTTP/3 en el borde hacia los clientes** (móviles/globales), donde sí paga, vía el CDN/LB gestionado; (3) revisar primero los **connection pools** y el ruteo L4/L7, que es donde está el ROI.

---

> **Para seguir.** Las fuentes de verdad: los RFCs (HTTP/1.1 = RFC 9112, HTTP/2 = RFC 9113, HTTP/3 = RFC 9114, QUIC = RFC 9000, TLS 1.3 = RFC 8446), `caniuse.com` para soporte de browser, y la doc de tu CDN/LB (Cloudflare, AWS ALB/NLB/Route 53) para qué protocolos y políticas de routing soporta. Re-verificá lo marcado con ⚠️ (adopción de HTTP/3, costo de CPU de QUIC, comportamiento de TTLs cortos, deprecación de server push): la capa se mueve. Los puentes en el hub: [Reverse proxy](reverse-proxy.md) es la versión operativa (config de nginx de esta capa); [Diseño de APIs](api-design.md) M2 elige el estilo (y acá entendiste qué le da HTTP/2 a gRPC); [Replicación](replicacion.md) M16 es el multi-región que GeoDNS/anycast/GSLB hacen alcanzable; [Tiempo real](tiempo-real.md) son los WebSockets que viven sobre esta conexión persistente; y [Diseño de sistemas](system-design.md) M2 es la matemática de servilleta con la que estimaste el costo de los RTT.
