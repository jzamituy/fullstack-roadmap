# Tiempo real y documentación de API: el cierre full stack

**WebSockets para features en vivo + OpenAPI/Swagger · conectá tu frontend al backend · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el cierre del temario: dos temas que completan tu backend para que sea, de verdad, full stack. **WebSockets** te da features en **tiempo real** (chat, notificaciones en vivo, presencia) — el diferenciador que mejor aprovecha tu experiencia de React/React Native, porque del otro lado del socket está una app que ya sabés construir. Y **OpenAPI/Swagger** documenta tu API de forma viva y navegable, lo primero que mira quien evalúa tu portfolio o consume tu backend. Juntos cierran el círculo: un backend que no solo responde requests, sino que empuja datos y se explica solo.

**Lo que asumimos.** Tu "Task API" con Postgres, auth (JWT), tests, deploy y Redis. Reusamos todo: autenticamos sockets con tu JWT y escalamos WebSockets con tu Redis.

**Para practicar.** NestJS usa **Socket.IO** por defecto para WebSockets:

```bash
npm i @nestjs/websockets @nestjs/platform-socket.io socket.io
npm i @nestjs/swagger          # documentación OpenAPI
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
  MessageBody, ConnectedSocket, OnGatewayConnection,
} from "@nestjs/websockets";
import { Server, Socket } from "socket.io";

@WebSocketGateway({ cors: { origin: "*" } }) // cors para que el front pueda conectarse
export class ChatGateway implements OnGatewayConnection {
  @WebSocketServer() private server!: Server; // referencia al servidor, para emitir

  constructor(private readonly chatService: ChatService) {} // DI normal

  handleConnection(client: Socket): void {
    console.log(`Cliente conectado: ${client.id}`);
  }

  @SubscribeMessage("mensaje") // escucha el evento "mensaje" que emite el cliente
  async onMensaje(
    @MessageBody() data: { texto: string },
    @ConnectedSocket() client: Socket,
  ): Promise<void> {
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

Como es un provider con DI, el gateway inyecta tus services (la lógica de negocio sigue en el service, no en el gateway: la misma separación de capas de siempre).

**Ejercicios 3**
3.1 ¿Cuál es el equivalente WebSocket de un `@Controller` y de un `@Get`/`@Post` en NestJS?
3.2 ¿Para qué sirve `@WebSocketServer()` y para qué `@SubscribeMessage()`?
3.3 ¿Dónde debería vivir la lógica de negocio: en el gateway o en un service inyectado? ¿Por qué?

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
onVerProyecto(@MessageBody() id: number, @ConnectedSocket() client: Socket): void {
  client.join(`proyecto:${id}`); // este cliente ahora escucha eventos de ese proyecto
}

// cuando alguien actualiza una tarea de ese proyecto:
this.server.to(`proyecto:${id}`).emit("tarea-actualizada", tarea);
```

Las rooms resuelven el problema de **a quién le importa cada evento**. Sin ellas, o le mandás todo a todos (ruido, fugas de datos entre usuarios) o tenés que llevar a mano la lista de a quién notificar. Con rooms, Socket.IO mantiene esa membresía por vos. Casos típicos: una sala por proyecto, por conversación de chat, o por usuario (`usuario:42`) para notificaciones personales.

**Ejercicios 4**
4.1 ¿Cuál es la diferencia entre `server.emit(...)`, `client.emit(...)` y `server.to(room).emit(...)`?
4.2 ¿Qué problema resuelven las rooms? ¿Qué pasaría si emitieras todos los eventos a todos los conectados?
4.3 Querés que las notificaciones de un usuario le lleguen solo a él (en todos sus dispositivos). ¿Cómo lo modelarías con rooms?

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
      const payload = await this.jwt.verifyAsync(token, {
        secret: this.config.getOrThrow("JWT_ACCESS_SECRET"),
      });
      client.data.user = payload;               // guardamos el usuario en el socket
      client.join(`usuario:${payload.sub}`);    // sala personal para notificaciones
    } catch {
      client.disconnect();                       // token inválido → fuera
    }
  }
}
```

Del lado del cliente (React / React Native):

```ts
const socket = io(URL, { auth: { token: accessToken } });
```

A partir de ahí, `client.data.user` te dice quién es en cada evento, igual que `request.user` en HTTP. También existen **guards de WebSocket** (`@UseGuards` con un guard que implemente la lógica para el contexto WS) para proteger eventos específicos, pero validar en `handleConnection` y guardar el usuario en el socket cubre la mayoría de los casos. La regla del módulo de auth sigue: autenticá primero (quién sos), después autorizá (si podés unirte a *esta* sala / disparar *este* evento).

**Ejercicios 5**
5.1 ¿Por qué el token JWT se valida al conectar (handshake) y no en cada mensaje, como sí pasa en HTTP?
5.2 ¿Qué hacés con un socket cuyo token es inválido? ¿Dónde guardás el usuario una vez validado?
5.3 Conectá con el módulo de auth: una vez autenticado el socket, ¿qué falta chequear antes de dejar que un cliente se una a la sala de un proyecto ajeno?

---

## Módulo 6 — Escalar WebSockets entre instancias (con Redis)

**Teoría.** Acá vuelve el problema del **escalado horizontal** del módulo de Node, con una vuelta de tuerca. Con WebSockets, cada cliente mantiene una conexión **persistente con UNA instancia** concreta. Si escalás a 3 instancias detrás de un balanceador, los clientes quedan repartidos: Ana conectada a la instancia A, Luis a la B.

El problema: si Ana (en A) manda un mensaje y hacés `server.emit(...)`, **solo llega a los clientes conectados a la instancia A**. Luis, en B, nunca lo recibe — porque cada instancia solo conoce sus propias conexiones. El tiempo real se rompe al escalar.

La solución es el **adapter de Redis** (`@socket.io/redis-adapter`): usa el **pub/sub de Redis** (otra capacidad de Redis que no vimos) para que las instancias se **reenvíen los eventos entre sí**. Cuando A emite, publica el evento en Redis; B y C están suscriptas y lo reemiten a *sus* clientes. Resultado: un `emit` llega a todos los clientes, sin importar a qué instancia estén conectados.

```ts
// main.ts: configurar el adapter de Redis para Socket.IO
const pubClient = createClient({ url: process.env.REDIS_URL });
const subClient = pubClient.duplicate();
await Promise.all([pubClient.connect(), subClient.connect()]);

const adapter = new RedisIoAdapter(app);
adapter.connectToRedis(pubClient, subClient);
app.useWebSocketAdapter(adapter); // ahora los emits se propagan entre instancias
```

Esto vuelve a apoyarse en que tu app sea **stateless** (el módulo de Node) y en Redis como **estado/coordinación compartida** (el módulo de Redis): las instancias no comparten memoria, se coordinan a través de Redis. Es el patrón estándar y la respuesta correcta si en una entrevista te preguntan "¿cómo escalás un chat con WebSockets?".

**Ejercicios 6**
6.1 ¿Por qué un `server.emit(...)` no llega a todos los clientes cuando tenés varias instancias de tu app?
6.2 ¿Cómo resuelve el adapter de Redis ese problema? ¿Qué capacidad de Redis usa?
6.3 Conectá con módulos anteriores: ¿por qué este escalado de WebSockets depende de que la app sea stateless y de Redis como coordinación compartida?

---

## Módulo 7 — Cuándo WebSockets y cuándo alternativas

**Teoría.** WebSockets es potente, pero **no siempre es la herramienta correcta** — y saber cuándo *no* usarlo es criterio (el mismo espíritu del archivo senior con los microservicios). Mantener conexiones persistentes tiene costo (memoria por conexión, complejidad de escalado del módulo 6). El criterio según el patrón de comunicación:

- **WebSockets** — cuando hay comunicación **bidireccional y frecuente**, o el servidor empuja seguido: chat, edición colaborativa, juegos, presencia en vivo, dashboards que se actualizan al instante.
- **Server-Sent Events (SSE)** — cuando el flujo es **solo del servidor al cliente** (unidireccional) y te alcanza con eso: un feed de notificaciones, el progreso de un job, actualizaciones de estado. Es más simple que WebSockets (va sobre HTTP normal, reconecta solo) y suele bastar para "el servidor avisa".
- **Polling** — cuando los cambios son **poco frecuentes** y el tiempo real no es crítico: chequear el estado de algo cada 30s. Simple, sin conexión persistente; perfectamente válido si la latencia no importa.

El error es usar WebSockets "porque es lo más avanzado" para algo que un SSE o un polling cada 30s resolvían sin la complejidad. La pregunta correcta: *¿realmente necesito bidireccionalidad y baja latencia, o solo que el cliente se entere de cambios ocasionales?* Para tu portfolio, un chat o notificaciones en vivo justifican WebSockets de sobra; para "avisar cuando un reporte está listo", un SSE o consultar el estado del job (módulo de Redis) alcanza.

> Mención al pasar (el roadmap los nombra como nociones): **GraphQL** ofrece *subscriptions* para tiempo real, y **gRPC** tiene *streaming* bidireccional. Son alternativas en contextos específicos; para una API REST con features en vivo, WebSockets/Socket.IO es lo más directo y pedido.

**Ejercicios 7**
7.1 ¿En qué caso usarías SSE en lugar de WebSockets? ¿Cuál es la diferencia clave?
7.2 Dá un ejemplo donde un polling cada 30s sea preferible a WebSockets, y justificá.
7.3 ¿Por qué "usar WebSockets porque es lo más avanzado" puede ser un error? Conectá con el criterio de complejidad del archivo senior.

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
SwaggerModule.setup("docs", app, document); // Swagger UI queda en /docs
```

Y enriquecés los DTOs y controladores con decoradores `@Api*` para que la doc sea precisa:

```ts
export class CreateProjectDto {
  @ApiProperty({ example: "Lanzamiento Q3", description: "Nombre del proyecto" })
  @IsString()
  nombre!: string;
}

@ApiTags("proyectos")               // agrupa los endpoints en la UI
@ApiBearerAuth()                    // este controlador requiere token
@Controller("proyectos")
export class ProyectosController {
  @ApiResponse({ status: 201, description: "Proyecto creado" })
  @ApiResponse({ status: 401, description: "No autenticado" })
  @Post()
  crear(@Body() dto: CreateProjectDto) { /* ... */ }
}
```

Fijate la sinergia con módulos anteriores: los mismos DTOs con `class-validator` (módulo de Nest) ahora también documentan; el `@ApiBearerAuth` refleja tu auth JWT; los status codes documentados son los que aprendiste a usar bien (`201`, `401`, `404`). La spec se genera de tu código real, así que **no se desactualiza** mientras la mantengas con los decoradores — el problema clásico de la documentación a mano que miente.

**Ejercicios 9**
9.1 ¿Por qué `@nestjs/swagger` puede generar gran parte de la doc automáticamente? ¿De qué se aprovecha?
9.2 ¿Para qué sirve `addBearerAuth()` / `@ApiBearerAuth()` en la spec? Conectá con el módulo de auth.
9.3 ¿Cuál es la ventaja de que la spec se genere del código real en vez de escribirse a mano en un documento aparte?

---

## Módulo 10 — El círculo full stack: React / React Native + tu backend

**Teoría.** Llegamos al cierre del temario y del roadmap: conectar un **frontend que ya sabés hacer** (React / React Native) al backend que construiste. Acá es donde tu transición se completa y donde tu perfil se vuelve un diferenciador real: muy pocos backenders saben mobile, y vos venís de ahí.

Cómo se conectan las piezas que aprendiste, de punta a punta:

- **REST + auth**: tu app hace `fetch`/axios a los endpoints, guarda el **access token en memoria y el refresh en un store seguro** (módulo de auth: en mobile, `SecureStore`/`Keychain`, el equivalente a la cookie httpOnly del web), y renueva con el refresh cuando el access expira.
- **Tiempo real**: `socket.io-client` se conecta pasando el token (módulo 5) y escucha eventos — una notificación, un mensaje, una tarea actualizada llegan **sin que la app pregunte**. La reconexión automática de Socket.IO es oro en mobile, donde la red se corta seguido.
- **El contrato**: con Swagger (módulo 9) podés **generar un cliente TypeScript tipado** desde la spec, así tu app consume la API con tipos de punta a punta — la misma seguridad de tipos que tenés en el backend, ahora en el front.
- **Deploy**: tu backend vive en una URL pública (módulo de Docker/deploy); tu app apunta ahí. El círculo está cerrado.

El proyecto **capstone** que sugiere el roadmap y que mejor muestra esto: una plataforma tipo Trello/Linear o un chat en tiempo real, con tu frontend React (o tu app React Native) + backend Nest + Postgres + auth + WebSockets + deploy. Eso es, literalmente, todo el temario en un solo producto desplegado — la mejor pieza posible de tu portfolio, y la prueba de que la transición de frontend a full stack está hecha.

**Ejercicios 10**
10.1 En una app React Native, ¿dónde guardás el access token y dónde el refresh token, y por qué? (Conectá con el módulo de auth.)
10.2 ¿Qué ventaja concreta da generar un cliente TypeScript desde la spec OpenAPI para tu frontend?
10.3 ¿Por qué un capstone full stack (front React/RN + backend Nest + tiempo real + deploy) es la mejor pieza de portfolio para tu transición, y qué demuestra de tu perfil?

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
```

---

## Cómo seguir (y cierre del temario)

Con este módulo cerraste el recorrido completo: empezaste con TypeScript y NestJS, sumaste **PostgreSQL**, **autenticación**, los **fundamentos de Node**, **testing**, **Docker/deploy/CI-CD**, **Redis** y ahora **tiempo real + documentación**. Eso es, exactamente, el stack que el roadmap define como "full stack contratable": TypeScript + NestJS + PostgreSQL + Redis + Docker + JWT/OAuth + arquitectura en capas evolucionando a hexagonal.

Lo que queda no es más teoría, es **construir y mostrar**:

1. **El capstone**: una app full stack end-to-end (front React/RN + backend Nest + DB + auth + WebSockets + deploy). Es tu proyecto estrella y la prueba viviente de la transición.
2. **Portfolio pulido**: cada proyecto desplegado, con README, diagrama de arquitectura, tests y Swagger UI público. Calidad sobre cantidad: 3 proyectos sólidos pesan más que 10 a medias.
3. **System design y entrevistas**: practicá explicar en voz alta lo que estudiaste — el event loop, cuándo NO usar microservicios, el patrón Outbox, cómo escalás un chat. Si lo podés explicar claro, lo dominás.

El temario te dio las piezas y el criterio. Ahora el 80% es construir: agarrá el roadmap, elegí un proyecto y empezá. Tu ventaja —ya sabés programar y venís del frontend— hace que el camino más corto a full stack sea, simplemente, **hacerlo**.
