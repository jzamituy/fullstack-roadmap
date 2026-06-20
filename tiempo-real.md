# Tiempo real y documentación de API: el cierre full stack

**WebSockets para features en vivo + OpenAPI/Swagger · conectá tu frontend al backend · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el cierre del temario: dos temas que completan tu backend para que sea, de verdad, full stack. **WebSockets** te da features en **tiempo real** (chat, notificaciones en vivo, presencia) — el diferenciador que mejor aprovecha tu experiencia de React/React Native, porque del otro lado del socket está una app que ya sabés construir. Y **OpenAPI/Swagger** documenta tu API de forma viva y navegable, lo primero que mira quien evalúa tu portfolio o consume tu backend. Juntos cierran el círculo: un backend que no solo responde requests, sino que empuja datos y se explica solo.

**Lo que asumimos.** Tu "Task API" con Postgres, auth (JWT), tests, deploy y Redis. Reusamos todo: autenticamos sockets con tu JWT y escalamos WebSockets con tu Redis.

**Para practicar.** NestJS usa **Socket.IO** por defecto para WebSockets:

```bash
npm i @nestjs/websockets @nestjs/platform-socket.io socket.io
npm i @socket.io/redis-adapter redis   # escalar WebSockets entre instancias (módulo 6)
npm i @nestjs/swagger                   # documentación OpenAPI
```

**Índice de módulos**
1. Tiempo real: por qué HTTP no alcanza
2. WebSockets: conexión persistente y bidireccional
3. Gateways en NestJS
4. Emitir eventos: a uno, a todos, a rooms
5. Autenticar el socket (con tu JWT)
6. Escalar WebSockets entre instancias (con Redis)
7. Cuándo WebSockets y cuándo alternativas
8. Documentar la API: por qué OpenAPI/Swagger
9. Swagger en NestJS, en concreto
10. El círculo full stack: React / React Native + tu backend

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Tiempo real: por qué HTTP no alcanza

**Teoría.** Todo lo que viste hasta acá usa el modelo **request/response** de HTTP: el cliente **pregunta**, el servidor **responde**, y la conexión se cierra. Funciona perfecto para "traeme mis proyectos". Pero no sirve cuando el servidor necesita **avisarle al cliente** que algo pasó sin que el cliente pregunte: un mensaje nuevo en un chat, una tarea que otro usuario completó, una notificación.

Con HTTP puro, la única forma de "enterarte" de un cambio es **preguntar de nuevo**. Las técnicas previas a WebSockets:

- **Polling**: el cliente pregunta cada X segundos ("¿hay algo nuevo?"). Simple, pero ineficiente: la mayoría de las respuestas son "no, nada", desperdiciando requests, batería y datos — algo que como dev de React Native sabés que importa.
- **Long polling**: el cliente pregunta y el servidor **retiene** la respuesta hasta que haya novedad. Mejor, pero sigue siendo un parche sobre un modelo que no fue pensado para esto.

El problema de fondo: HTTP es **unidireccional** en la iniciativa (siempre arranca el cliente). Para tiempo real real necesitás que el **servidor pueda hablar primero**. Eso es lo que resuelve WebSockets: una conexión **persistente y bidireccional** donde cualquiera de los dos lados puede mandar datos en cualquier momento.

**Ejercicios 1**
1.1 ¿Por qué el modelo request/response de HTTP no sirve para que el servidor avise al cliente de un evento (ej. un mensaje nuevo)?
1.2 ¿Qué es el polling y cuál es su principal desperdicio? (Pensá en una app mobile.)
1.3 ¿Cuál es la limitación de fondo de HTTP que WebSockets viene a resolver?

---

## Módulo 2 — WebSockets: conexión persistente y bidireccional

**Teoría.** Un **WebSocket** es una conexión que se abre una vez y **queda abierta**, permitiendo enviar mensajes en **ambas direcciones** sin el overhead de abrir una conexión HTTP por cada uno. Empieza como una request HTTP normal que se "actualiza" (un *handshake* con el header `Upgrade`) y, a partir de ahí, el canal queda persistente.

```
HTTP (request/response):           WebSocket (persistente, bidireccional):
  cliente → "¿novedades?" → server   cliente ⇄═════════════⇄ server
  cliente ← "no"          ← server        (conexión abierta)
  cliente → "¿novedades?" → server   server → "mensaje nuevo!" → cliente  (sin que pregunte)
  ...repetir cada 5s                  cliente → "escribiendo..." → server
```

**Socket.IO** (lo que usa Nest por defecto) es una librería sobre WebSockets que agrega comodidades de producción: **reconexión automática** si se cae la red (clave en mobile, donde la conexión es inestable), *fallback* a otras técnicas si WebSocket no está disponible, **rooms** (agrupar conexiones, módulo 4) y un modelo de **eventos con nombre** (`emit("mensaje", data)` / `on("mensaje", ...)`) en vez de mensajes crudos. Del lado del cliente usás `socket.io-client`, tanto en React como en React Native.

> **Pregunta de entrevista: ¿Socket.IO es WebSocket?** No exactamente. Socket.IO implementa un **protocolo propio sobre Engine.IO** (con su handshake, sus reconexiones y su empaquetado de eventos), que *usa* WebSocket como transporte cuando puede y cae a HTTP long-polling cuando no. Consecuencia práctica: un cliente `socket.io-client` **no** puede hablar con un servidor WebSocket nativo (`ws`/la API `WebSocket` del navegador) ni al revés — son protocolos distintos. Si necesitás interoperar con clientes WS estándar, usás `ws` o WebSockets nativos; si querés las comodidades (rooms, reconexión, fallback), Socket.IO de punta a punta.

**Seguridad de transporte.** Igual que tu API va sobre HTTPS, los WebSockets en producción van sobre **WSS** (`wss://`, WebSocket sobre TLS): sin cifrar, el token del handshake y los mensajes viajan en claro. Y el `Origin` del handshake hay que validarlo del lado del servidor (lo vemos en el módulo 5) — conecta con el HTTPS del módulo de deploy.

El modelo mental: en vez de "endpoints" (rutas), pensás en **eventos** que viajan en ambos sentidos. El cliente `emit`e eventos y el servidor `on`-los escucha, y viceversa.

**Ejercicios 2**
2.1 ¿En qué se diferencia una conexión WebSocket de una serie de requests HTTP, en cuanto a apertura y dirección?
2.2 Nombrá dos comodidades que Socket.IO agrega sobre WebSockets crudos y por qué importan (una pensando en mobile).
2.3 ¿Cómo cambia el modelo mental al pasar de HTTP a WebSockets (de "rutas" a qué)?

---

## Módulo 3 — Gateways en NestJS

**Teoría.** En Nest, el equivalente a un controlador para WebSockets es un **gateway**. Se marca con `@WebSocketGateway()` y maneja eventos con `@SubscribeMessage()` (el análogo a `@Get`/`@Post`, pero para eventos en vez de rutas HTTP). Es un provider normal: vive en un módulo y usa **DI** como cualquier service.

```ts
import {
  WebSocketGateway, WebSocketServer, SubscribeMessage,
  MessageBody, ConnectedSocket, OnGatewayConnection, WsException,
} from "@nestjs/websockets";
import { UsePipes, ValidationPipe } from "@nestjs/common";
import { IsString, MaxLength } from "class-validator";
import { Server, Socket } from "socket.io";

// DTO validado, igual que en HTTP — pero ojo, ver la nota de abajo
class MensajeDto {
  @IsString()
  @MaxLength(2000)
  texto!: string;
}

@WebSocketGateway({
  // allowlist por entorno, NUNCA "*" en una app autenticada (ver módulo 5)
  cors: { origin: process.env.ALLOWED_ORIGINS?.split(",") ?? false },
})
export class ChatGateway implements OnGatewayConnection {
  @WebSocketServer() private server!: Server; // referencia al servidor, para emitir

  constructor(private readonly chatService: ChatService) {} // DI normal

  handleConnection(client: Socket): void {
    console.log(`Cliente conectado: ${client.id}`);
  }

  @SubscribeMessage("mensaje") // escucha el evento "mensaje" que emite el cliente
  @UsePipes(new ValidationPipe({ transform: true })) // ¡necesario! no corre solo en WS
  async onMensaje(
    @MessageBody() data: MensajeDto,
    @ConnectedSocket() client: Socket,
  ): Promise<void> {
    if (!data.texto.trim()) throw new WsException("Mensaje vacío");
    const guardado = await this.chatService.guardar(data.texto);
    this.server.emit("mensaje", guardado); // reenvía a TODOS los conectados
  }
}
```

Las piezas:

- `@WebSocketServer()` te da el **servidor** Socket.IO, con el que emitís eventos hacia los clientes.
- `@SubscribeMessage("evento")` registra un handler para un evento entrante (lo que el cliente `emit`e).
- `@MessageBody()` y `@ConnectedSocket()` son los decoradores de parámetro (análogos a `@Body()` y `@Req()`).
- Los **lifecycle hooks** `OnGatewayConnection` / `OnGatewayDisconnect` te avisan cuando un cliente se conecta o desconecta — útil para presencia ("quién está online") y limpieza.

**Trampa de producción #1 — la validación NO es automática.** En HTTP, si registrás un `ValidationPipe` global (módulo de Nest), tus DTOs se validan solos. **En gateways WebSocket eso no pasa por defecto**: tenés que aplicar el pipe explícitamente con `@UsePipes(new ValidationPipe())` en el handler o el gateway. Si te olvidás, el `@MessageBody() data: MensajeDto` es una **mentira de tipos**: en runtime entra cualquier cosa que el cliente mande, sin validar — puerta abierta a payloads maliciosos o malformados. Es exactamente el tipo de detalle que un entrevistador senior pregunta.

**Trampa de producción #2 — los errores no se manejan como en HTTP.** Lanzar una `HttpException` en un gateway no sirve. El equivalente es **`WsException`**, y para darle forma a los errores que llegan al cliente se usa un **exception filter de WS** (`@Catch(WsException)` con `BaseWsExceptionFilter`). Sin esto, un throw no controlado puede tumbar el handler sin avisarle nada útil al cliente.

Como es un provider con DI, el gateway inyecta tus services (la lógica de negocio sigue en el service, no en el gateway: la misma separación de capas de siempre).

**Ejercicios 3**
3.1 ¿Cuál es el equivalente WebSocket de un `@Controller` y de un `@Get`/`@Post` en NestJS?
3.2 ¿Para qué sirve `@WebSocketServer()` y para qué `@SubscribeMessage()`?
3.3 ¿Dónde debería vivir la lógica de negocio: en el gateway o en un service inyectado? ¿Por qué?
3.4 ¿Por qué un `@MessageBody() data: MiDto` sin `@UsePipes(ValidationPipe)` es una "mentira de tipos" en un gateway, y qué riesgo abre?
3.5 ¿Qué excepción se lanza en un gateway para reportar un error al cliente, y por qué no sirve `HttpException`?
3.6 (teclado) Implementá el `ChatGateway` con el DTO validado. Desde un cliente (o Postman/wscat) emití un `"mensaje"` con `texto` ausente o de 5000 caracteres y verificá que la validación lo rechaza.

---

## Módulo 4 — Emitir eventos: a uno, a todos, a rooms

**Teoría.** Una vez que tenés el servidor, el control fino está en **a quién** le emitís. Tres alcances:

```ts
// 1. A TODOS los conectados (broadcast global)
this.server.emit("notificacion", data);

// 2. A UN cliente específico (el que mandó el mensaje, por ejemplo)
client.emit("confirmacion", { ok: true });

// 3. A un GRUPO (room): solo a quienes están en esa "sala"
this.server.to(`proyecto:${proyectoId}`).emit("tarea-actualizada", tarea);
```

El concepto clave es el de **rooms** (salas): agrupaciones lógicas de conexiones. Un cliente se une a una room con `client.join("nombre")`, y vos emitís `to("nombre")` para llegar **solo** a los miembros de esa sala. Es lo que te permite que los cambios de un proyecto lleguen únicamente a quienes lo están mirando, no a todos los usuarios de la app:

```ts
@SubscribeMessage("ver-proyecto")
async onVerProyecto(
  @MessageBody() id: number,
  @ConnectedSocket() client: Socket,
): Promise<void> {
  // ⚠️ AUTORIZAR antes de unir: nada impide que un cliente pida una sala ajena.
  const usuarioId = client.data.user.sub; // viene del handshake autenticado (módulo 5)
  const esMiembro = await this.proyectosService.esMiembro(id, usuarioId);
  if (!esMiembro) throw new WsException("Sin acceso a este proyecto");

  client.join(`proyecto:${id}`); // recién ahora escucha eventos de ese proyecto
}

// cuando alguien actualiza una tarea de ese proyecto:
this.server.to(`proyecto:${id}`).emit("tarea-actualizada", tarea);
```

> **Trampa de producción #3 — autenticar no es autorizar.** Que el socket esté autenticado (módulo 5) dice *quién* es, pero **nada impide** que un cliente emita `ver-proyecto` con el id de un proyecto que no es suyo y se cuele en su sala, recibiendo datos ajenos. Por eso, antes de `client.join("proyecto:X")` (o de procesar cualquier evento sobre un recurso) hay que **chequear pertenencia** contra el service, igual que un guard de autorización en HTTP. Podés encapsularlo en un `WsGuard` por evento, pero el chequeo explícito ya cubre lo esencial.

Las rooms resuelven el problema de **a quién le importa cada evento**. Sin ellas, o le mandás todo a todos (ruido, fugas de datos entre usuarios) o tenés que llevar a mano la lista de a quién notificar. Con rooms, Socket.IO mantiene esa membresía por vos. Casos típicos: una sala por proyecto, por conversación de chat, o por usuario (`usuario:42`) para notificaciones personales.

**Ejercicios 4**
4.1 ¿Cuál es la diferencia entre `server.emit(...)`, `client.emit(...)` y `server.to(room).emit(...)`?
4.2 ¿Qué problema resuelven las rooms? ¿Qué pasaría si emitieras todos los eventos a todos los conectados?
4.3 Querés que las notificaciones de un usuario le lleguen solo a él (en todos sus dispositivos). ¿Cómo lo modelarías con rooms?
4.4 Un cliente autenticado emite `ver-proyecto` con el id de un proyecto ajeno. ¿Qué pasa si solo hacés `client.join` sin chequear nada, y cómo lo prevenís?

---

## Módulo 5 — Autenticar el socket (con tu JWT)

**Teoría.** Un WebSocket también necesita saber **quién** está del otro lado — y reusás tu auth del módulo de JWT. La diferencia con HTTP: no hay un header `Authorization` en cada mensaje (la conexión es persistente), así que el token se valida **al conectar**, en el handshake.

El cliente manda el token al abrir la conexión, y el gateway lo verifica en `handleConnection`; si es inválido, **desconecta** el socket:

```ts
@WebSocketGateway()
export class ChatGateway implements OnGatewayConnection {
  constructor(private readonly jwt: JwtService, private readonly config: ConfigService) {}

  async handleConnection(client: Socket): Promise<void> {
    try {
      const token = client.handshake.auth?.token; // el cliente lo manda al conectar
      if (typeof token !== "string") return client.disconnect(); // sin token → fuera
      const payload = await this.jwt.verifyAsync(token, {
        secret: this.config.getOrThrow("JWT_ACCESS_SECRET"),
      });
      client.data.user = payload;               // guardamos el usuario en el socket
      client.join(`usuario:${payload.sub}`);    // sala personal para notificaciones
    } catch {
      client.disconnect();                       // token inválido/expirado → fuera
    }
  }
}
```

Del lado del cliente (React / React Native):

```ts
const socket = io(URL, { auth: { token: accessToken } });
```

A partir de ahí, `client.data.user` te dice quién es en cada evento, igual que `request.user` en HTTP. También existen **guards de WebSocket** (`@UseGuards` con un guard que implemente la lógica para el contexto WS) para proteger eventos específicos, pero validar en `handleConnection` y guardar el usuario en el socket cubre la mayoría de los casos. La regla del módulo de auth sigue: autenticá primero (quién sos), después autorizá (si podés unirte a *esta* sala / disparar *este* evento — módulo 4).

**El problema que casi nadie ve: el token vence con el socket abierto.** Validás el JWT *una vez*, al conectar. Pero la conexión es persistente y puede durar horas, mientras que el access token dura minutos (módulo de auth). ¿Qué pasa cuando expira? Si solo validás en el handshake, **el socket queda autenticado para siempre** aunque el token ya venció o la sesión fue revocada — un agujero de seguridad clásico que un entrevistador senior va a buscar. Estrategias reales:

- **Reconexión con token fresco**: como el access token es de vida corta, cerrá el socket al vencer (un timer en el server con `setTimeout(() => client.disconnect(), msHastaExpirar)` calculado desde el `exp` del payload) y dejá que el cliente reconecte con un token renovado (Socket.IO reconecta solo; del lado del cliente actualizás `socket.auth.token` antes de reconectar).
- **Re-auth por evento**: el cliente emite un evento `"reauth"` con el nuevo token antes de que expire; el server re-verifica y actualiza `client.data.user`.
- **Revocación**: para cortar una sesión robada *ya* (no esperar a que expire), chequeá contra la **lista de sesiones/refresh tokens en Redis** (módulo de Redis) — si la sesión fue revocada, desconectás.

Para tu Task API, lo más simple y correcto es la **reconexión con token fresco**: aprovecha que el token ya es de vida corta y que Socket.IO reconecta automáticamente.

**Ejercicios 5**
5.1 ¿Por qué el token JWT se valida al conectar (handshake) y no en cada mensaje, como sí pasa en HTTP?
5.2 ¿Qué hacés con un socket cuyo token es inválido o ausente? ¿Dónde guardás el usuario una vez validado?
5.3 Conectá con el módulo de auth: una vez autenticado el socket, ¿qué falta chequear antes de dejar que un cliente se una a la sala de un proyecto ajeno?
5.4 El access token dura 15 min pero el socket queda abierto 3 horas. ¿Qué problema de seguridad aparece si solo validás en el handshake, y cómo lo resolverías?
5.5 ¿Cómo cortarías *inmediatamente* una sesión robada que ya tiene un socket abierto, sin esperar a que el token expire? (Pista: módulo de Redis.)

---

## Módulo 6 — Escalar WebSockets entre instancias (con Redis)

**Teoría.** Acá vuelve el problema del **escalado horizontal** del módulo de Node, con una vuelta de tuerca. Con WebSockets, cada cliente mantiene una conexión **persistente con UNA instancia** concreta. Si escalás a 3 instancias detrás de un balanceador, los clientes quedan repartidos: Ana conectada a la instancia A, Luis a la B.

El problema: si Ana (en A) manda un mensaje y hacés `server.emit(...)`, **solo llega a los clientes conectados a la instancia A**. Luis, en B, nunca lo recibe — porque cada instancia solo conoce sus propias conexiones. El tiempo real se rompe al escalar.

La solución es el **adapter de Redis** (`@socket.io/redis-adapter`): usa el **pub/sub de Redis** (el que viste en el módulo de Redis) para que las instancias se **reenvíen los eventos entre sí**. Cuando A emite, publica el evento en Redis; B y C están suscriptas y lo reemiten a *sus* clientes. Resultado: un `emit` llega a todos los clientes, sin importar a qué instancia estén conectados.

En NestJS esto se implementa con un **adapter custom** que extiende `IoAdapter`. El patrón real (no alcanza con crear clientes sueltos: hay que aplicar `createAdapter()` al server de Socket.IO en `createIOServer`):

```ts
// redis-io.adapter.ts
import { IoAdapter } from "@nestjs/platform-socket.io";
import { ServerOptions } from "socket.io";
import { createAdapter } from "@socket.io/redis-adapter";
import { createClient } from "redis";

export class RedisIoAdapter extends IoAdapter {
  private adapterConstructor!: ReturnType<typeof createAdapter>;

  async connectToRedis(url: string): Promise<void> {
    const pubClient = createClient({ url });
    const subClient = pubClient.duplicate();
    await Promise.all([pubClient.connect(), subClient.connect()]);
    this.adapterConstructor = createAdapter(pubClient, subClient);
  }

  createIOServer(port: number, options?: ServerOptions): unknown {
    const server = super.createIOServer(port, options);
    server.adapter(this.adapterConstructor); // ← acá se engancha el pub/sub
    return server;
  }
}
```

```ts
// main.ts
const redisIoAdapter = new RedisIoAdapter(app);
await redisIoAdapter.connectToRedis(process.env.REDIS_URL!);
app.useWebSocketAdapter(redisIoAdapter); // ahora los emits se propagan entre instancias
```

**La otra mitad que casi siempre se olvida: sticky sessions.** El adapter de Redis propaga los `emit` entre instancias, pero **no** resuelve un problema distinto: Socket.IO arranca el handshake con **HTTP long-polling** (y cae a polling como fallback), y eso son **varias requests HTTP** que **deben llegar todas a la misma instancia**. Si el balanceador las reparte round-robin entre A, B y C, el handshake se rompe con errores tipo *"Session ID unknown"*. Por eso, con varias instancias necesitás:

- **Session affinity / sticky sessions** en el balanceador o ingress (afinidad por IP o cookie), para que todas las requests de un cliente caigan en la misma instancia; **o**
- forzar `transports: ["websocket"]` en cliente y server, saltando el polling — más simple, pero perdés el fallback (si la red bloquea WebSocket, no hay plan B).

La regla para la entrevista: **el adapter de Redis y las sticky sessions resuelven problemas diferentes y complementarios** — Redis propaga eventos *entre* instancias; las sticky sessions garantizan que el handshake de *un* cliente no rebote entre instancias. Necesitás los dos.

Esto vuelve a apoyarse en que tu app sea **stateless** (el módulo de Node) y en Redis como **estado/coordinación compartida** (el módulo de Redis): las instancias no comparten memoria, se coordinan a través de Redis. Es el patrón estándar y la respuesta correcta si en una entrevista te preguntan "¿cómo escalás un chat con WebSockets?".

**Ejercicios 6**
6.1 ¿Por qué un `server.emit(...)` no llega a todos los clientes cuando tenés varias instancias de tu app?
6.2 ¿Cómo resuelve el adapter de Redis ese problema? ¿Qué capacidad de Redis usa?
6.3 Conectá con módulos anteriores: ¿por qué este escalado de WebSockets depende de que la app sea stateless y de Redis como coordinación compartida?
6.4 ¿Qué problema NO resuelve el adapter de Redis y sí resuelven las sticky sessions? ¿Por qué Socket.IO las necesita (pensá en el transporte polling)?
6.5 (teclado) Levantá dos instancias de tu API con el adapter de Redis (y sticky sessions o `transports: ["websocket"]`). Conectá dos clientes que caigan en instancias distintas y verificá que un `emit` desde un cliente le llega al otro.

---

## Módulo 7 — Cuándo WebSockets y cuándo alternativas

**Teoría.** WebSockets es potente, pero **no siempre es la herramienta correcta** — y saber cuándo *no* usarlo es criterio (el mismo espíritu del archivo senior con los microservicios). Mantener conexiones persistentes tiene costo (memoria por conexión, complejidad de escalado del módulo 6). El criterio según el patrón de comunicación:

- **WebSockets** — cuando hay comunicación **bidireccional y frecuente**, o el servidor empuja seguido: chat, edición colaborativa, juegos, presencia en vivo, dashboards que se actualizan al instante.
- **Server-Sent Events (SSE)** — cuando el flujo es **solo del servidor al cliente** (unidireccional) y te alcanza con eso: un feed de notificaciones, el progreso de un job, actualizaciones de estado. Es más simple que WebSockets (va sobre HTTP normal, reconecta solo) y suele bastar para "el servidor avisa".
- **Polling** — cuando los cambios son **poco frecuentes** y el tiempo real no es crítico: chequear el estado de algo cada 30s. Simple, sin conexión persistente; perfectamente válido si la latencia no importa.

El error es usar WebSockets "porque es lo más avanzado" para algo que un SSE o un polling cada 30s resolvían sin la complejidad. La pregunta correcta: *¿realmente necesito bidireccionalidad y baja latencia, o solo que el cliente se entere de cambios ocasionales?* Para tu portfolio, un chat o notificaciones en vivo justifican WebSockets de sobra; para "avisar cuando un reporte está listo", un SSE o consultar el estado del job (módulo de Redis) alcanza.

> Dato fino (SSE): en **HTTP/1.1** el navegador limita a ~6 conexiones por dominio, y cada SSE ocupa una — un detalle que tumba apps con varias pestañas. Con **HTTP/2** (multiplexado) ese límite desaparece. Saber esto distingue a un senior cuando defiende "SSE vs WS".

**Lo que WebSockets NO te da gratis (entrega y salud de la conexión).** Que el canal esté abierto no significa que tus mensajes lleguen siempre. Tres cosas que tenés que operar y que caen en entrevista:

- **ACKs (confirmación de recepción)**: por defecto `emit` es *fire-and-forget* — emitís y no sabés si llegó. Socket.IO soporta un **callback de ACK** para confirmar: `socket.emit("mensaje", data, (resp) => { /* el server confirmó */ })`. Úsalo cuando necesitás saber que el otro lado recibió (ej. "mensaje entregado").
- **Heartbeats (ping/pong)**: Socket.IO manda pings periódicos (`pingInterval`/`pingTimeout`) para detectar conexiones muertas — una red mobile que se cae no siempre dispara un "close" limpio. Si no hay pong a tiempo, se considera caído y dispara la reconexión.
- **Eventos perdidos al reconectar**: este es el grande. Mientras un cliente estuvo desconectado (túnel, cambio de red), **los `emit` que el server hizo en ese lapso se perdieron** — Socket.IO reconecta el socket, pero no reenvía lo que te perdiste. El patrón de producción es **re-sincronizar estado al reconectar**: en el evento de reconexión, el cliente pide el estado actual por REST (ej. "traeme los mensajes desde el timestamp X") en vez de asumir que el stream en vivo está completo. WebSockets es para *lo nuevo en vivo*; la fuente de verdad sigue siendo tu API REST + Postgres.

> Mención al pasar (el roadmap los nombra como nociones): **GraphQL** ofrece *subscriptions* para tiempo real (aparecen en stacks con GraphQL/Apollo, normalmente sobre WebSockets por debajo), y **gRPC** tiene *streaming* bidireccional (típico en comunicación **entre microservicios internos**, no navegador↔servidor). Para una API REST con features en vivo de cara al cliente, WebSockets/Socket.IO es lo más directo y pedido.

**Ejercicios 7**
7.1 ¿En qué caso usarías SSE en lugar de WebSockets? ¿Cuál es la diferencia clave?
7.2 Dá un ejemplo donde un polling cada 30s sea preferible a WebSockets, y justificá.
7.3 ¿Por qué "usar WebSockets porque es lo más avanzado" puede ser un error? Conectá con el criterio de complejidad del archivo senior.
7.4 ¿Qué es un ACK en Socket.IO y cuándo lo usarías en vez de un `emit` fire-and-forget?
7.5 Un cliente mobile pierde la red 30s y reconecta. ¿Por qué no alcanza con confiar en el stream en vivo, y cuál es el patrón para no perderse lo que pasó mientras estuvo caído?

---

## Módulo 8 — Documentar la API: por qué OpenAPI/Swagger

**Teoría.** Cambiamos de tema al segundo pilar de este cierre. Una API que nadie sabe cómo usar vale poco. **OpenAPI** (antes "Swagger") es un **estándar** para describir una API REST: qué endpoints tiene, qué parámetros y body reciben, qué devuelven, qué status codes, qué auth requiere. Esa descripción es un documento (JSON/YAML) que las herramientas saben leer.

Lo valioso no es el documento en sí, sino lo que **habilita**:

- **Documentación viva y navegable**: **Swagger UI** genera una página web interactiva desde la spec, donde se ven todos los endpoints y se pueden **probar en vivo** desde el navegador (mandar un request real y ver la respuesta). Reemplaza al "leé el código para entender la API".
- **Contrato compartido**: el frontend (vos mismo, o tu equipo) sabe exactamente qué esperar sin preguntar. En equipos, el front puede empezar a trabajar contra el contrato antes de que el backend esté listo.
- **Generación de código**: desde la spec se pueden **generar clientes tipados** (un SDK de TypeScript para tu app React/React Native), tests, mocks. La spec es la fuente.

Para tu portfolio es un diferenciador concreto: una API con Swagger UI desplegado dice "esto es profesional, se entiende solo y se puede probar", frente a un README que hay que creerle. Es justo lo que el roadmap pide documentar en el proyecto estrella.

**Ejercicios 8**
8.1 ¿Qué describe una spec OpenAPI de una API REST? Nombrá tres cosas.
8.2 ¿Qué es Swagger UI y qué podés hacer en él que un README estático no permite?
8.3 Nombrá dos cosas que la spec OpenAPI habilita más allá de la documentación para leer.

---

## Módulo 9 — Swagger en NestJS, en concreto

**Teoría.** NestJS tiene integración de primera con OpenAPI vía `@nestjs/swagger`. Lo bueno: **aprovecha tus DTOs y decoradores existentes** para generar gran parte de la spec automáticamente — la documentación no es un trabajo aparte, sale de tu código.

Se configura en `main.ts` con el `DocumentBuilder`:

```ts
import { DocumentBuilder, SwaggerModule } from "@nestjs/swagger";

const config = new DocumentBuilder()
  .setTitle("Task API")
  .setDescription("API de proyectos y tareas")
  .setVersion("1.0")
  .addBearerAuth()                 // documenta que usa JWT (del módulo de auth)
  .build();

const document = SwaggerModule.createDocument(app, config);

// Proteger /docs en producción: no la expongas pública por defecto.
if (process.env.NODE_ENV !== "production") {
  SwaggerModule.setup("docs", app, document); // Swagger UI en /docs solo fuera de prod
}
// En prod: o no la montás, o la protegés con un guard / basic-auth detrás de la URL.
```

> **Por qué proteger `/docs`:** `SwaggerModule.setup` deja la UI **pública** en esa ruta. Exponer el mapa completo de tu API (endpoints, esquemas, auth) a cualquiera en producción es regalarle reconocimiento a un atacante. El default sano: servirla solo en dev/staging, o protegerla con auth básica/guard en prod.

Y enriquecés los DTOs y controladores con decoradores `@Api*` para que la doc sea precisa — incluyendo **los errores como parte del contrato**:

```ts
export class CreateProjectDto {
  @ApiProperty({ example: "Lanzamiento Q3", description: "Nombre del proyecto" })
  @IsString()
  nombre!: string;
}

// El formato de error TAMBIÉN es parte del contrato: documentalo con un DTO
export class ErrorResponseDto {
  @ApiProperty({ example: 400 }) statusCode!: number;
  @ApiProperty({ example: "Validation failed" }) message!: string;
  @ApiProperty({ example: "Bad Request" }) error!: string;
}

@ApiTags("proyectos")               // agrupa los endpoints en la UI
@ApiBearerAuth()                    // este controlador requiere token
@Controller({ path: "proyectos", version: "1" }) // versionado: /v1/proyectos
export class ProyectosController {
  @ApiResponse({ status: 201, description: "Proyecto creado", type: ProjectDto })
  @ApiResponse({ status: 400, description: "Datos inválidos", type: ErrorResponseDto })
  @ApiResponse({ status: 401, description: "No autenticado", type: ErrorResponseDto })
  @ApiResponse({ status: 403, description: "Sin permiso", type: ErrorResponseDto })
  @ApiResponse({ status: 409, description: "Conflicto (duplicado)", type: ErrorResponseDto })
  @Post()
  crear(@Body() dto: CreateProjectDto) { /* ... */ }
}
```

Dos decisiones de diseño que separan a un mid de un senior:

- **Documentar los errores, no solo el camino feliz**: un consumidor necesita saber qué forma tiene un `400`/`401`/`403`/`409` para manejarlos. El formato de error es contrato igual que el de éxito.
- **Versionado de la API**: NestJS soporta versionado por URI (`/v1/...`), header o media type (`app.enableVersioning()`). Versionar desde el principio te permite evolucionar la API sin romper los clientes ya generados desde la spec — clave si otros (o tu app mobile ya publicada) consumen tu backend.

Fijate la sinergia con módulos anteriores: los mismos DTOs con `class-validator` (módulo de Nest) ahora también documentan; el `@ApiBearerAuth` refleja tu auth JWT; los status codes documentados son los que aprendiste a usar bien (`201`, `400`, `401`, `403`, `404`, `409`). La spec se genera de tu código real, así que **no se desactualiza** mientras la mantengas con los decoradores — el problema clásico de la documentación a mano que miente.

**Ejercicios 9**
9.1 ¿Por qué `@nestjs/swagger` puede generar gran parte de la doc automáticamente? ¿De qué se aprovecha?
9.2 ¿Para qué sirve `addBearerAuth()` / `@ApiBearerAuth()` en la spec? Conectá con el módulo de auth.
9.3 ¿Cuál es la ventaja de que la spec se genere del código real en vez de escribirse a mano en un documento aparte?
9.4 ¿Por qué conviene documentar las respuestas de error (400/401/403/409), y no solo el 201? ¿Para quién es parte del contrato?
9.5 ¿Por qué exponer `/docs` público en producción es un riesgo, y qué hacés en su lugar? ¿Qué problema previene versionar la API (`/v1`) desde el inicio?

---

## Módulo 10 — El círculo full stack: React / React Native + tu backend

**Teoría.** Llegamos al cierre del temario y del roadmap: conectar un **frontend que ya sabés hacer** (React / React Native) al backend que construiste. Acá es donde tu transición se completa y donde tu perfil se vuelve un diferenciador real: muy pocos backenders saben mobile, y vos venís de ahí.

Cómo se conectan las piezas que aprendiste, de punta a punta:

- **REST + auth**: tu app hace `fetch`/axios a los endpoints, guarda el **access token en memoria y el refresh en un store seguro** (módulo de auth: en mobile, `SecureStore`/`Keychain`, el equivalente a la cookie httpOnly del web), y renueva con el refresh cuando el access expira.
- **Tiempo real**: `socket.io-client` se conecta pasando el token (módulo 5) y escucha eventos — una notificación, un mensaje, una tarea actualizada llegan **sin que la app pregunte**. La reconexión automática de Socket.IO es oro en mobile, donde la red se corta seguido.
- **El contrato**: con Swagger (módulo 9) podés **generar un cliente TypeScript tipado** desde la spec, así tu app consume la API con tipos de punta a punta — la misma seguridad de tipos que tenés en el backend, ahora en el front.
- **Deploy**: tu backend vive en una URL pública (módulo de Docker/deploy); tu app apunta ahí. El círculo está cerrado.

El proyecto **capstone** que sugiere el roadmap y que mejor muestra esto: una plataforma tipo Trello/Linear o un chat en tiempo real, con tu frontend React (o tu app React Native) + backend Nest + Postgres + auth + WebSockets + deploy. Eso es, literalmente, todo el temario en un solo producto desplegado — la mejor pieza posible de tu portfolio, y la prueba de que la transición de frontend a full stack está hecha.

**Criterios de aceptación (para que sea verificable, no "más o menos funciona").** Un capstone que muestra nivel cumple, como mínimo:

1. **Auth completa**: registro/login con JWT access+refresh, refresh en store seguro (mobile) o cookie httpOnly (web), y logout que revoca (módulo de auth + Redis).
2. **CRUD real con permisos**: un usuario solo ve/edita sus proyectos y tareas; un recurso ajeno devuelve `403` (autorización, no solo autenticación).
3. **Tiempo real verificable**: abrir la app en dos clientes y ver que un cambio en uno aparece en el otro **sin recargar** (rooms por proyecto), con el socket autenticado por JWT.
4. **Escala**: corre con ≥2 instancias detrás de un balanceador (adapter de Redis + sticky sessions) y el tiempo real sigue funcionando entre instancias.
5. **Documentada**: Swagger UI navegable (protegida en prod), con errores documentados.
6. **Desplegada**: URL pública (backend) + app instalable o web desplegada, con CI que corre los tests (módulo de Docker/deploy).
7. **Con tests**: al menos los flujos críticos (auth, permisos, un caso de tiempo real) cubiertos (módulo de testing).

Si tu proyecto cumple esos siete puntos, no es un "to-do app de tutorial": es la evidencia concreta de que cerrás el círculo frontend↔backend.

**Ejercicios 10**
10.1 En una app React Native, ¿dónde guardás el access token y dónde el refresh token, y por qué? (Conectá con el módulo de auth.)
10.2 ¿Qué ventaja concreta da generar un cliente TypeScript desde la spec OpenAPI para tu frontend?
10.3 ¿Por qué un capstone full stack (front React/RN + backend Nest + tiempo real + deploy) es la mejor pieza de portfolio para tu transición, y qué demuestra de tu perfil?
10.4 (proyecto) Tomá los 7 criterios de aceptación y autoevaluá tu capstone: ¿cuáles cumplís hoy y cuál es el próximo que vas a cerrar?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Porque en request/response la iniciativa siempre es del cliente: el servidor solo
    contesta lo que le preguntan, no puede iniciar la comunicación. Para avisar de un
    evento, el cliente tendría que estar preguntando todo el tiempo.
1.2 El cliente pregunta cada X segundos si hay novedades. Su desperdicio: la mayoría de
    las respuestas son "nada", gastando requests, batería y datos (crítico en mobile)
    para, casi siempre, no recibir nada nuevo.
1.3 Que HTTP es unidireccional en la iniciativa (siempre arranca el cliente). WebSockets
    permite que el servidor también pueda enviar datos cuando quiera (bidireccional).
```

### Módulo 2
```
2.1 El WebSocket se abre una vez y queda persistente, y permite enviar mensajes en ambas
    direcciones en cualquier momento. HTTP abre y cierra una conexión por cada
    intercambio y siempre lo inicia el cliente.
2.2 (1) Reconexión automática: si la red se cae (común en mobile) el cliente se reconecta
    solo. (2) Rooms y eventos con nombre: agrupar clientes y modelar la comunicación por
    eventos en vez de mensajes crudos. (También: fallback si WebSocket no está disponible.)
2.3 De pensar en rutas/endpoints (request→response) a pensar en eventos con nombre que
    viajan en ambos sentidos: el cliente emite y escucha eventos, y el servidor también.
2.4 (extra) Socket.IO NO es WebSocket puro: usa un protocolo propio sobre Engine.IO (que
    usa WebSocket como transporte cuando puede y cae a polling). Por eso socket.io-client
    no interopera con un server WS nativo (ws / API WebSocket del navegador) ni viceversa.
    En prod va sobre WSS (TLS), igual que HTTPS.
```

### Módulo 3
```
3.1 El @WebSocketGateway() es el equivalente del @Controller; el @SubscribeMessage("ev")
    es el equivalente de @Get/@Post pero para un evento entrante en vez de una ruta HTTP.
3.2 @WebSocketServer() inyecta la referencia al servidor Socket.IO, que usás para EMITIR
    eventos a los clientes. @SubscribeMessage() registra un handler para un evento que el
    cliente emite (escuchar).
3.3 En un service inyectado, no en el gateway. El gateway es la capa de entrada (como un
    controlador): recibe el evento y delega. Mantener la lógica en el service preserva la
    separación de capas y permite reusarla/testearla sin WebSockets.
3.4 Porque en gateways el ValidationPipe global NO corre automáticamente (a diferencia de
    HTTP): sin @UsePipes(new ValidationPipe()), el tipo del DTO no se valida en runtime y
    entra cualquier payload que el cliente mande. El riesgo: datos malformados o
    maliciosos llegan al service sin filtro. Hay que aplicar el pipe explícito en el
    gateway/handler.
3.5 Se lanza una WsException (de @nestjs/websockets), no una HttpException: el contexto es
    WebSocket, no HTTP, así que las HttpException y los exception filters HTTP no aplican.
    Para dar forma a los errores hacia el cliente se usa un filtro WS (@Catch +
    BaseWsExceptionFilter).
3.6 (teclado) Con el DTO + @UsePipes, un "mensaje" con texto ausente o de 5000 chars es
    rechazado por class-validator (@IsString / @MaxLength) antes de llegar al service; sin
    el pipe, pasaría sin validar.
```

### Módulo 4
```
4.1 server.emit envía a TODOS los clientes conectados; client.emit envía solo a ESE
    cliente puntual; server.to(room).emit envía solo a los clientes que están en esa room
    (grupo).
4.2 Resuelven "a quién le importa cada evento": agrupan conexiones para emitir solo a las
    que corresponde. Sin rooms, mandarías todo a todos: ruido, desperdicio y fuga de datos
    entre usuarios (alguien recibiría eventos de proyectos que no le incumben).
4.3 Uniendo cada conexión del usuario a una room personal por su id (ej. usuario:42) al
    conectarse; las notificaciones se emiten con server.to(`usuario:42`).emit(...) y
    llegan a todos sus dispositivos conectados, solo a él.
4.4 Si solo hacés client.join sin chequear, el cliente se cuela en la sala de un proyecto
    ajeno y recibe sus eventos (fuga de datos): autenticar dice quién es, no qué puede ver.
    Se previene chequeando pertenencia contra el service (¿es miembro del proyecto?) ANTES
    del join, y rechazando con WsException si no lo es (idealmente vía un WsGuard).
```

### Módulo 5
```
5.1 Porque la conexión es persistente: se establece una vez y queda abierta, así que no
    hay un header por mensaje donde mandar el token. Se valida en el handshake (al
    conectar) y, si es válido, esa conexión queda autenticada para toda su vida.
5.2 Si el token es inválido, desconectás el socket (client.disconnect()). Si es válido,
    guardás el usuario en el socket (client.data.user = payload) para saber quién es en
    cada evento posterior.
5.3 Falta la autorización: chequear que ese usuario tenga permiso sobre ESE proyecto
    (que sea suyo o colaborador) antes de hacer client.join de su room. Autenticar dice
    quién es; autorizar, si puede unirse a ese recurso (igual que en HTTP).
5.4 Si solo validás en el handshake, el socket queda autenticado mientras viva la conexión
    aunque el access token haya expirado o la sesión se haya revocado: una conexión de
    horas con un token de minutos sigue activa indefinidamente. Se resuelve cerrando el
    socket al vencer el token (timer según el exp del payload) y dejando que el cliente
    reconecte con un token fresco, o con un evento de re-auth periódico.
5.5 Chequeando contra la lista de sesiones/refresh tokens en Redis (módulo de Redis): si
    la sesión fue revocada (DEL de la clave), el server lo detecta y desconecta el socket
    de inmediato, sin esperar a que expire el JWT. Un JWT por sí solo no se puede revocar
    antes de su expiración.
```

### Módulo 6
```
6.1 Porque cada instancia solo conoce las conexiones abiertas con ELLA. Un server.emit
    en la instancia A llega solo a los clientes de A; los conectados a B o C no se enteran,
    ya que las instancias no comparten la memoria con la lista de sockets.
6.2 Usa el pub/sub de Redis: cuando una instancia emite, publica el evento en Redis y las
    demás (suscriptas) lo reemiten a sus propios clientes. Así un emit llega a todos sin
    importar a qué instancia esté conectado cada cliente.
6.3 Porque las instancias no comparten estado en memoria (son stateless): necesitan un
    canal externo para coordinarse, y ese canal es Redis (pub/sub). Es el mismo principio
    de "estado/coordinación compartida en Redis" de los módulos de Node y Redis.
6.4 El adapter de Redis propaga los emits ENTRE instancias, pero no garantiza que las
    varias requests HTTP del handshake de UN cliente caigan en la misma instancia.
    Socket.IO arranca/cae a HTTP long-polling, que necesita afinidad: sin sticky sessions
    (afinidad por IP/cookie en el balanceador), el handshake en polling rebota entre
    instancias y falla ("Session ID unknown"). Alternativa: forzar transports:
    ["websocket"] (perdiendo el fallback). Son problemas distintos y complementarios.
6.5 (teclado) Con dos instancias + adapter de Redis, dos clientes conectados a instancias
    distintas igual se ven los eventos: el emit de uno se publica en Redis y la otra
    instancia lo reemite a su cliente. Sin el adapter, el segundo cliente no recibiría nada.
```

### Módulo 7
```
7.1 Usás SSE cuando el flujo es solo del servidor al cliente (unidireccional): un feed de
    notificaciones, el progreso de un job. La diferencia clave: SSE no permite que el
    cliente envíe por el mismo canal (eso lo hace por HTTP normal) y es más simple.
7.2 Chequear el estado de algo que cambia poco y donde la latencia no importa (ej. "¿el
    reporte ya está listo?" cada 30s). Es preferible porque evita la complejidad de
    mantener conexiones persistentes para algo ocasional.
7.3 Porque agrega complejidad (conexiones persistentes, escalado con Redis, más memoria)
    que quizás no necesitás: si un SSE o un polling resuelven, los WebSockets son
    complejidad accidental. El criterio del senior: elegí por el problema, no por lo
    "moderno".
7.4 Un ACK es un callback de confirmación: el receptor avisa que recibió el evento
    (socket.emit("ev", data, (resp) => ...)). Lo usás cuando necesitás certeza de entrega
    o una respuesta (ej. "mensaje entregado", confirmar que se guardó), en vez del emit
    fire-and-forget donde no sabés si llegó.
7.5 Porque mientras el cliente estuvo desconectado, los emits que el server hizo en ese
    lapso se perdieron: Socket.IO reconecta el socket pero NO reenvía lo que te perdiste.
    El patrón: al reconectar, pedir el estado actual por REST (ej. mensajes desde el último
    timestamp/id conocido) y re-sincronizar, tratando a Postgres/REST como fuente de verdad
    y al stream en vivo solo para lo nuevo.
```

### Módulo 8
```
8.1 Los endpoints, sus parámetros y body de entrada, lo que devuelven (forma y status
    codes) y la autenticación que requieren. (Cualquier tres de estas.)
8.2 Swagger UI es una página interactiva generada desde la spec donde se ven todos los
    endpoints y se pueden PROBAR en vivo (mandar requests reales desde el navegador y ver
    la respuesta), algo que un README estático no permite.
8.3 Generar clientes tipados (un SDK para el frontend), tests o mocks; y servir de
    contrato compartido para que el front trabaje contra él sin esperar al backend.
```

### Módulo 9
```
9.1 Porque se aprovecha de tus DTOs y decoradores existentes (class-validator, tipos):
    de ahí infiere la forma de las entradas/salidas. La doc sale del código que ya
    escribiste, no es un trabajo separado.
9.2 Documentan que la API usa autenticación con Bearer token (JWT): en Swagger UI aparece
    el candado para cargar el token y probar endpoints protegidos. Refleja el esquema de
    auth del módulo de JWT en la spec.
9.3 Que la doc no se desactualiza ni miente: al generarse del código real (DTOs,
    decoradores), refleja la API tal como es. La doc escrita a mano en un documento aparte
    queda vieja apenas cambia un endpoint y nadie la actualiza.
9.4 Porque el consumidor (tu front, otro equipo) necesita saber qué forma tienen los
    errores para manejarlos: el formato de un 400/401/403/409 es parte del contrato, igual
    que el del 201. Documentar solo el camino feliz deja al cliente adivinando cómo reacciona
    ante fallos.
9.5 Exponer /docs público en prod revela el mapa completo de la API (endpoints, esquemas,
    auth) a cualquiera, dándole reconocimiento gratis a un atacante: por eso se sirve solo
    en dev/staging o se protege con guard/basic-auth. Versionar (/v1) desde el inicio
    permite evolucionar la API (cambios incompatibles en /v2) sin romper a los clientes ya
    publicados que consumen /v1 (ej. tu app mobile en las tiendas).
```

### Módulo 10
```
10.1 El access token en memoria (variable/estado de la app) por ser de vida corta; el
     refresh token en un store seguro del dispositivo (SecureStore/Keychain), el
     equivalente mobile de la cookie httpOnly: protegido del acceso de otras apps/JS. Es
     la estrategia del módulo de auth aplicada a mobile.
10.2 Tipado de punta a punta: el front consume la API con los mismos tipos que define el
     backend, así un cambio en un endpoint rompe la compilación del front (lo detectás
     antes de runtime) en vez de fallar silenciosamente. Menos bugs de integración.
10.3 Porque reúne TODO el temario en un solo producto desplegado (REST, DB, auth, tiempo
     real, deploy) y demuestra que cerrás el círculo frontend↔backend. De tu perfil
     muestra el diferenciador: sabés mobile/React Y backend, algo que pocos backenders
     tienen y que te separa en una búsqueda.
10.4 (proyecto) Autoevaluación honesta contra los 7 criterios (auth completa, permisos,
     tiempo real verificable en 2 clientes, escala con ≥2 instancias, doc protegida,
     deploy con CI, tests de flujos críticos). Lo importante es identificar el próximo
     criterio a cerrar y atacarlo; un capstone que cumple los 7 ya no es un proyecto de
     tutorial.
```

---

## Cómo seguir (y cierre del temario)

Con este módulo cerraste el recorrido completo: empezaste con TypeScript y NestJS, sumaste **PostgreSQL**, **autenticación**, los **fundamentos de Node**, **testing**, **Docker/deploy/CI-CD**, **Redis** y ahora **tiempo real + documentación**. Eso es, exactamente, el stack que el roadmap define como "full stack contratable": TypeScript + NestJS + PostgreSQL + Redis + Docker + JWT/OAuth + arquitectura en capas evolucionando a hexagonal.

Lo que queda no es más teoría, es **construir y mostrar**:

1. **El capstone**: una app full stack end-to-end (front React/RN + backend Nest + DB + auth + WebSockets + deploy). Es tu proyecto estrella y la prueba viviente de la transición.
2. **Portfolio pulido**: cada proyecto desplegado, con README, diagrama de arquitectura, tests y Swagger UI público. Calidad sobre cantidad: 3 proyectos sólidos pesan más que 10 a medias.
3. **System design y entrevistas**: practicá explicar en voz alta lo que estudiaste — el event loop, cuándo NO usar microservicios, el patrón Outbox, cómo escalás un chat. Si lo podés explicar claro, lo dominás.

El temario te dio las piezas y el criterio. Ahora el 80% es construir: agarrá el roadmap, elegí un proyecto y empezá. Tu ventaja —ya sabés programar y venís del frontend— hace que el camino más corto a full stack sea, simplemente, **hacerlo**.
