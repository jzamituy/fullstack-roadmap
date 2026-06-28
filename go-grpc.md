# gRPC y Protobuf en Go: microservicios tipados

**El lenguaje de comunicación entre servicios · contract-first · HTTP/2 · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya hiciste [Go para backend](go-backend.md): structs, interfaces, `context`, errores como valores y un servidor HTTP. Acá damos el salto que define a un backend Go de microservicios: **gRPC**. No es casualidad que lo veas en Go — gRPC y Go nacieron en Google de la misma cultura, y Go es **el** lenguaje de referencia del ecosistema. Si REST es cómo tu API habla con el browser, **gRPC es cómo tus servicios hablan entre sí**.

**Lo que asumimos.** Go a nivel del módulo [Go para backend](go-backend.md), HTTP y REST/JSON, qué es un contrato de API (si hiciste [Diseño de APIs](api-design.md), mejor), y `context` para cancelación/timeouts. **No** asumimos que sepas Protocol Buffers ni HTTP/2 — eso lo construimos.

> ⚠️ **Nota sobre APIs que cambiaron.** El cliente de `grpc-go` cambió: **`grpc.Dial` está deprecado** y la forma actual (2026) es **`grpc.NewClient`** (desde grpc-go 1.63, 2024). El tooling de codegen también se movió: hoy lo idiomático es **`buf`** por encima de invocar `protoc` a mano. Todo lo marcado con ⚠️ verificalo contra la doc oficial (`grpc.io`, `protobuf.dev`, `buf.build`) antes de citarlo como definitivo.

**Índice de módulos**
1. Qué es gRPC y por qué se usa tanto en Go
2. Protocol Buffers: el contrato como código
3. Codegen: de `.proto` a Go con `buf`/`protoc`
4. El servidor: implementar y registrar el servicio
5. El cliente: `grpc.NewClient` y el stub tipado
6. Los cuatro tipos de RPC (unary y streaming)
7. Errores: `status` y códigos (no HTTP status)
8. `context`, metadata, deadlines e interceptors
9. Seguridad: TLS, mTLS y autenticación
10. Producción: health, reflection, observabilidad y balanceo
11. El criterio: gRPC vs REST vs GraphQL

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere `go` + toolchain de protobuf).

---

## Módulo 1 — Qué es gRPC y por qué se usa tanto en Go

**Teoría.** **gRPC** es un framework de **RPC (Remote Procedure Call)**: en vez de pensar en "endpoints y verbos HTTP", llamás a un **método de un servicio remoto como si fuera una función local** (`cliente.ObtenerUsuario(ctx, req)`), y el framework se encarga de serializar, mandar por la red y deserializar la respuesta.

Las cuatro piezas que lo definen, **en contraste con REST/JSON**:

- **Contract-first con Protocol Buffers.** El contrato (servicios, métodos, mensajes) se define en un archivo `.proto` y de ahí se **genera el código** de cliente y servidor. No hay drift entre doc y realidad: el `.proto` *es* la verdad. (En REST el contrato suele ser un OpenAPI que se escribe aparte y se desincroniza.)
- **Serialización binaria (Protobuf), no JSON.** Más compacto y mucho más rápido de (de)serializar. El costo: no es legible por humanos en el cable (necesitás herramientas).
- **HTTP/2 por debajo.** Multiplexa muchas llamadas sobre una conexión, soporta **streaming bidireccional** y header compression. REST clásico vive en HTTP/1.1 una-request-una-respuesta.
- **Tipado fuerte de punta a punta.** El stub generado te da tipos concretos en ambos lados; un campo mal escrito **no compila**, no falla en runtime.

¿Por qué Go? Porque gRPC, Protobuf, Kubernetes y Go salen de la misma cultura: el soporte es de primera, el código generado es idiomático, y el grueso de la infraestructura cloud-native que habla gRPC está escrita en Go. **Para comunicación service-to-service interna, gRPC es el default en backends Go.**

La frase mental: **REST es para el borde (browser, terceros); gRPC es para adentro (servicio a servicio), donde mandan la performance y el contrato fuerte.**

**Ejercicios 1**
1.1 🔁 Nombrá las cuatro piezas que definen gRPC frente a REST/JSON.
1.2 🧠 ¿Por qué se dice que gRPC es "contract-first" y qué problema de REST resuelve eso?
1.3 🧠 Un equipo expone su API pública para el browser y terceros con gRPC "porque es más rápido". ¿Qué le señalarías?

---

## Módulo 2 — Protocol Buffers: el contrato como código

**Teoría.** Un archivo `.proto` (proto3) define **mensajes** (los datos) y **servicios** (los métodos RPC). Cada campo lleva un **número** (el *field number*), que es lo que viaja en el cable — **no** el nombre. Esa es la clave de la compatibilidad.

```protobuf
syntax = "proto3";

package usuarios.v1;
option go_package = "miapp/gen/usuarios/v1;usuariosv1";

message Usuario {
  string id = 1;
  string nombre = 2;
  int32 edad = 3;
}

message ObtenerUsuarioRequest {
  string id = 1;
}

service UsuarioService {
  rpc ObtenerUsuario(ObtenerUsuarioRequest) returns (Usuario);
}
```

Reglas que importan en producción:

- **Los field numbers son sagrados.** Una vez asignado el `= 1`, **nunca lo cambies ni lo reuses** para otro campo: romperías a todos los clientes viejos. Para agregar, usás un número nuevo; para borrar, marcás el número como `reserved`.
- **Compatibilidad hacia adelante/atrás.** Agregar un campo nuevo (número nuevo) es compatible: los clientes viejos lo ignoran, los nuevos lo leen. Por eso Protobuf escala entre versiones de servicios.
- **Zero values, como en Go.** En proto3 no hay "campo ausente" para escalares: un `int32` no enviado llega como `0`, un `string` como `""`. Si necesitás distinguir "no enviado" de "cero", usás tipos *wrapper* (`google.protobuf.Int32Value`) o `optional`.
- **Versioná el `package`** (`usuarios.v1`): cuando haya un cambio incompatible, creás `v2` y convivís.

**Ejercicios 2**
2.1 🔁 ¿Qué viaja realmente en el cable: el nombre del campo o su número? ¿Qué implica eso?
2.2 🧠 ¿Por qué NUNCA hay que reusar un field number que ya estuvo en uso, y qué usás en su lugar al borrar un campo?
2.3 ✍️ Escribí un `.proto` con un mensaje `Producto` (`id`, `nombre`, `precio_centavos int64`) y un servicio `ProductoService` con un RPC `Crear(Producto) returns (Producto)`.

---

## Módulo 3 — Codegen: de `.proto` a Go con `buf`/`protoc`

**Teoría.** El `.proto` no se ejecuta: **genera código**. Para Go necesitás dos plugins:
- **`protoc-gen-go`** — genera los structs de los mensajes.
- **`protoc-gen-go-grpc`** — genera el cliente (stub) y la interfaz del servidor.

La forma vieja era invocar `protoc` con un comando largo y frágil. La forma moderna (2026) es **`buf`**: un wrapper que maneja dependencias, *linting* del `.proto`, detección de *breaking changes* y la generación con un `buf.gen.yaml` declarativo.

```yaml
# buf.gen.yaml
version: v2
plugins:
  - remote: buf.build/protocolbuffers/go
    out: gen
    opt: paths=source_relative
  - remote: buf.build/grpc/go
    out: gen
    opt: paths=source_relative
```

```bash
buf lint          # valida el .proto contra reglas de estilo
buf breaking --against '.git#branch=main'   # ¿rompiste compatibilidad?
buf generate      # genera el código Go en gen/
```

> 💡 **`buf breaking` es oro:** en CI, falla el build si un cambio en el `.proto` rompería a los clientes existentes (renombrar, cambiar un tipo, reusar un número). Es el guardrail que hace que "contract-first" sea seguro y no una promesa.

Lo generado: structs como `usuariosv1.Usuario`, una interfaz `UsuarioServiceServer` (que vos implementás) y un `UsuarioServiceClient` (que usás). **No edites el código generado** — se regenera.

**Ejercicios 3**
3.1 🔁 ¿Qué genera `protoc-gen-go` y qué genera `protoc-gen-go-grpc`?
3.2 🧠 ¿Qué ventaja te da `buf breaking` en CI sobre simplemente regenerar y compilar?
3.3 🧠 ¿Por qué no debés editar a mano el código generado, y dónde ponés entonces tu lógica?

---

## Módulo 4 — El servidor: implementar y registrar el servicio

**Teoría.** El codegen te da una **interfaz** `UsuarioServiceServer`. Tu trabajo es crear un struct que la **implemente** (acá vuelven las interfaces implícitas de Go) y registrarlo en un `grpc.Server`.

Detalle clave: el struct debe **embeber `pb.UnimplementedUsuarioServiceServer`**. Eso te da *forward compatibility*: si mañana agregás un método al `.proto` y aún no lo implementaste, el servicio sigue compilando y responde `Unimplemented` en vez de romper.

```go
package main

import (
	"context"
	"log"
	"net"

	"google.golang.org/grpc"
	pb "miapp/gen/usuarios/v1"
)

type servidor struct {
	pb.UnimplementedUsuarioServiceServer // embebido: forward compatibility
	repo UsuarioRepo
}

func (s *servidor) ObtenerUsuario(ctx context.Context, req *pb.ObtenerUsuarioRequest) (*pb.Usuario, error) {
	u, err := s.repo.Buscar(ctx, req.GetId()) // usá los getters generados (GetId), nil-safe
	if err != nil {
		return nil, err // (en el módulo 7 lo convertimos a un status code)
	}
	return &pb.Usuario{Id: u.ID, Nombre: u.Nombre, Edad: int32(u.Edad)}, nil
}

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatalf("listen: %v", err)
	}
	s := grpc.NewServer()
	pb.RegisterUsuarioServiceServer(s, &servidor{repo: nuevoRepo()})

	log.Println("gRPC escuchando en :50051")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("serve: %v", err)
	}
}
```

- **Usá los getters generados** (`req.GetId()`): son seguros ante `nil` (devuelven el zero value), a diferencia de `req.Id` que paniquea si `req` es `nil`.
- `grpc.NewServer()` acepta opciones (interceptors, TLS, límites); lo vemos en los módulos 8-9.

**Ejercicios 4**
4.1 🔁 ¿Para qué embebés `pb.UnimplementedUsuarioServiceServer` en tu struct de servidor?
4.2 🧠 ¿Por qué conviene `req.GetId()` en vez de `req.Id`?
4.3 ✍️ Escribí la firma del método `Crear` del `ProductoService` del Módulo 2 (receiver `*servidor`, `context`, `*pb.Producto`, retorno `(*pb.Producto, error)`).

---

## Módulo 5 — El cliente: `grpc.NewClient` y el stub tipado

**Teoría.** Del lado del cliente, abrís una **conexión** (`*grpc.ClientConn`) y la envolvés en el **stub** generado. La conexión es **multiplexada y de larga vida**: la creás una vez y la compartís (como el pool de `database/sql`), no una por llamada.

```go
import (
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	pb "miapp/gen/usuarios/v1"
)

func main() {
	// ⚠️ grpc.NewClient reemplaza al deprecado grpc.Dial
	conn, err := grpc.NewClient("localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials())) // solo dev: sin TLS
	if err != nil {
		log.Fatalf("conexión: %v", err)
	}
	defer conn.Close()

	cliente := pb.NewUsuarioServiceClient(conn)

	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	u, err := cliente.ObtenerUsuario(ctx, &pb.ObtenerUsuarioRequest{Id: "u1"})
	if err != nil {
		log.Fatalf("rpc: %v", err)
	}
	fmt.Println(u.GetNombre())
}
```

- **`grpc.NewClient`** (no `Dial`): no se conecta de inmediato, conecta *lazy* en la primera llamada — por eso no necesita el viejo `grpc.WithBlock`. Consecuencia clave: el `err` que devuelve `NewClient` solo valida el *target* y las opciones; **los fallos de red reales recién afloran en el primer RPC**, no acá (`WithBlock`/`FailOnNonTempDialError` quedaron deprecados en este flujo). Y ojo en Kubernetes: para un target sin esquema el *resolver* por defecto es `dns:///`, lo que importa para el balanceo (Módulo 10).
- **`defer conn.Close()`** descarta el error que devuelve `Close()`; en código real conviene capturarlo. Acá lo dejamos así por brevedad.
- **`insecure.NewCredentials()`** es **solo para desarrollo local**. En producción va TLS (Módulo 9). gRPC te obliga a ser explícito sobre la seguridad: no hay "modo inseguro por accidente".
- El `ctx` con timeout viaja con la llamada: si expira, el servidor lo ve cancelado (propagación de deadline, Módulo 8).

**Ejercicios 5**
5.1 🔁 ¿Por qué la `ClientConn` se crea una vez y se comparte, y no una por llamada?
5.2 🧠 ¿Qué cambió `grpc.NewClient` respecto del viejo `grpc.Dial` + `WithBlock` en cuanto a cuándo se establece la conexión?
5.3 🧠 ¿Por qué `insecure.NewCredentials()` no debe llegar a producción y qué lo reemplaza?

---

## Módulo 6 — Los cuatro tipos de RPC

**Teoría.** gRPC, gracias a HTTP/2, soporta **cuatro** formas de RPC. Esto es lo que no podés hacer (cómodo) con REST:

1. **Unary** — request → response. La llamada clásica (lo que vimos). `rpc Obtener(Req) returns (Resp);`
2. **Server streaming** — request → **flujo** de responses. El server manda muchos mensajes (ej: resultados paginados, notificaciones, un export grande). `rpc Listar(Req) returns (stream Item);`
3. **Client streaming** — **flujo** de requests → response. El cliente sube muchos mensajes y el server responde una vez al final (ej: subir métricas, ingestar un batch). `rpc Subir(stream Item) returns (Resumen);`
4. **Bidirectional streaming** — ambos flujos a la vez, independientes (ej: chat, telemetría en vivo). `rpc Chat(stream Msg) returns (stream Msg);`

Ejemplo de **server streaming** (el server envía mensajes en un loop hasta terminar):

```protobuf
service UsuarioService {
  rpc ListarUsuarios(ListarRequest) returns (stream Usuario);
}
```

```go
func (s *servidor) ListarUsuarios(req *pb.ListarRequest, stream pb.UsuarioService_ListarUsuariosServer) error {
	usuarios, err := s.repo.Todos(stream.Context()) // el context del stream
	if err != nil {
		return err
	}
	for _, u := range usuarios {
		if err := stream.Send(&pb.Usuario{Id: u.ID, Nombre: u.Nombre}); err != nil {
			return err // el cliente cortó; abortamos
		}
	}
	return nil // cerrar el stream = return nil
}
```

Del lado cliente, recibís en un loop hasta `io.EOF`:

```go
stream, err := cliente.ListarUsuarios(ctx, &pb.ListarRequest{})
if err != nil { /* ... */ }
for {
	u, err := stream.Recv()
	if err == io.EOF {
		break // el server terminó
	}
	if err != nil { /* error real */ }
	fmt.Println(u.GetNombre())
}
```

> 📝 Acá `err == io.EOF` se compara **directo** (no con `errors.Is`): los streams generados devuelven `io.EOF` sin envolver para señalar "fin del stream", así que es la convención idiomática de grpc-go — la excepción a la regla de `errors.Is` que viste en [Go para backend](go-backend.md).

**Client streaming** es el espejo: el server recibe en un loop y responde **una sola vez** al final con `SendAndClose`:

```go
func (s *servidor) SubirUsuarios(stream pb.UsuarioService_SubirUsuariosServer) error {
	var n int32
	for {
		u, err := stream.Recv()
		if err == io.EOF {
			return stream.SendAndClose(&pb.Resumen{Recibidos: n}) // fin del flujo: una respuesta
		}
		if err != nil {
			return err
		}
		_ = u // ... persistir cada uno ...
		n++
	}
}
```

(El **bidireccional** combina ambos: `Recv` y `Send` en goroutines independientes sobre el mismo stream.)

**Ejercicios 6**
6.1 🔁 Nombrá los cuatro tipos de RPC y un caso de uso de cada streaming.
6.2 🧠 Tenés que exportar 1 millón de registros a un cliente. ¿Unary o server streaming, y por qué?
6.3 ✍️ Escribí el loop del lado **cliente** que consume un server-stream `ListarUsuarios` y acumula los nombres en un slice (manejando `io.EOF`).

---

## Módulo 7 — Errores: `status` y códigos (no HTTP status)

**Teoría.** gRPC **no usa códigos HTTP** (404, 500). Tiene su propio conjunto de **status codes** en el paquete `codes`: `OK`, `NotFound`, `InvalidArgument`, `AlreadyExists`, `PermissionDenied`, `Unauthenticated`, `DeadlineExceeded`, `Unavailable`, `Internal`, etc. Devolver un error "pelado" desde un handler se traduce a `Unknown` — pobre para el cliente. Lo correcto es devolver un **`status` explícito**:

```go
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

func (s *servidor) ObtenerUsuario(ctx context.Context, req *pb.ObtenerUsuarioRequest) (*pb.Usuario, error) {
	if req.GetId() == "" {
		return nil, status.Error(codes.InvalidArgument, "id es obligatorio")
	}
	u, err := s.repo.Buscar(ctx, req.GetId())
	if errors.Is(err, ErrNoEncontrado) {
		return nil, status.Errorf(codes.NotFound, "usuario %s no existe", req.GetId())
	}
	if err != nil {
		return nil, status.Error(codes.Internal, "error interno") // no filtres detalles internos al cliente
	}
	return &pb.Usuario{Id: u.ID, Nombre: u.Nombre, Edad: int32(u.Edad)}, nil
}
```

Del lado cliente, inspeccionás el código con `status.FromError`:

```go
u, err := cliente.ObtenerUsuario(ctx, req)
if err != nil {
	st, _ := status.FromError(err)
	switch st.Code() {
	case codes.NotFound:
		// manejar 404 lógico
	case codes.Unauthenticated:
		// renovar token
	default:
		// otro
	}
}
```

- **Mapeo mental con HTTP** (útil si venís de REST): `InvalidArgument`≈400, `Unauthenticated`≈401, `PermissionDenied`≈403, `NotFound`≈404, `AlreadyExists`≈409, `Internal`≈500, `Unavailable`≈503.
- **Detalles enriquecidos:** `status` permite adjuntar mensajes estructurados (`google.rpc.ErrorInfo`, `BadRequest`) con `st.WithDetails(...)` — el equivalente tipado del *problem detail* de [Diseño de APIs](api-design.md).
- **No filtres errores internos:** un error de la DB se loguea del lado server y se devuelve como `Internal` genérico; nunca el mensaje crudo.

**Ejercicios 7**
7.1 🔁 ¿Qué pasa si devolvés un `error` común (ej. `errors.New("x")`) desde un handler gRPC, en términos del código que ve el cliente?
7.2 🧠 ¿Por qué un error de la base de datos se devuelve como `codes.Internal` y no con el mensaje original?
7.3 ✍️ Convertí esta validación a un `status`: si `req.GetEdad() < 0`, devolvé `InvalidArgument` con el mensaje "edad no puede ser negativa".

---

## Módulo 8 — `context`, metadata, deadlines e interceptors

**Teoría.** Tres mecanismos transversales:

- **Deadlines (no timeouts locales).** El `ctx` con `WithTimeout` del cliente **viaja con la llamada**: el server recibe el deadline y, si expira, su `ctx` se cancela. Esto propaga la cancelación por toda la cadena de servicios (A llama a B llama a C; si A se cansa, todos cortan). Es el patrón de [`context`](go-backend.md) llevado a la red.
- **Metadata** = los "headers" de gRPC: pares clave-valor para auth tokens, request IDs, tracing. Se leen del context:

```go
import "google.golang.org/grpc/metadata"

md, ok := metadata.FromIncomingContext(ctx)
if ok {
	tokens := md.Get("authorization") // []string
}
```

> 📝 Las claves de metadata son **case-insensitive**: gRPC las normaliza a minúsculas, así que `md.Get("authorization")` y `"Authorization"` son lo mismo. Buscá siempre en minúsculas.

- **Interceptors** = el **middleware de gRPC**. Una función que envuelve cada RPC; ideal para logging, auth, métricas, recovery de panics. Hay versión *unary* y *stream*.

```go
func logUnario(ctx context.Context, req any, info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler) (any, error) {
	inicio := time.Now()
	resp, err := handler(ctx, req) // llama al handler real
	slog.Info("rpc", "metodo", info.FullMethod, "dur", time.Since(inicio), "err", err)
	return resp, err
}

// al crear el server:
s := grpc.NewServer(grpc.ChainUnaryInterceptor(logUnario, authUnario))
```

> 💡 **Recovery obligatorio:** como viste en [Go para backend](go-backend.md), un `panic` en una goroutine tumba el proceso. En gRPC, cada RPC corre en su goroutine, así que un interceptor de **recover** (o `grpc_recovery` del ecosistema) que convierta el panic en `codes.Internal` es prácticamente obligatorio en producción.

**Ejercicios 8**
8.1 🔁 ¿Qué es la *metadata* en gRPC y para qué se usa típicamente?
8.2 🧠 ¿Por qué se dice que el deadline del cliente "viaja con la llamada", y qué problema de microservicios resuelve?
8.3 🧠 ¿Por qué un interceptor de recover es casi obligatorio en un servidor gRPC de producción?

---

## Módulo 9 — Seguridad: TLS, mTLS y autenticación

**Teoría.** gRPC corre sobre HTTP/2 y **espera TLS** en serio (el `insecure` del Módulo 5 es solo dev). Dos capas:

- **TLS del lado servidor** — encripta el canal y autentica al server ante el cliente. Cargás el certificado al crear el server:

```go
import "google.golang.org/grpc/credentials"

creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
if err != nil { log.Fatal(err) }
s := grpc.NewServer(grpc.Creds(creds))
```

- **mTLS (mutual TLS)** — además, el **server verifica al cliente** por su certificado. Es el estándar para tráfico service-to-service en una malla (service mesh como Istio/Linkerd lo hacen por vos). La identidad del servicio queda probada por el certificado, sin tokens. En código, mTLS no se arma con el `NewServerTLSFromFile` de arriba (eso es solo TLS server-side) sino con `credentials.NewTLS(&tls.Config{...})` configurando `ClientCAs` y `ClientAuth: tls.RequireAndVerifyClientCert`.

**Autenticación de usuario/llamador** (cuando no alcanza mTLS): el token va en la **metadata** (`authorization`), y un **interceptor** lo valida antes de llegar al handler — exactamente el patrón de middleware de auth, pero a nivel RPC:

```go
func authUnario(ctx context.Context, req any, info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler) (any, error) {
	md, _ := metadata.FromIncomingContext(ctx)
	tokens := md.Get("authorization")
	if len(tokens) == 0 || !tokenValido(tokens[0]) {
		return nil, status.Error(codes.Unauthenticated, "token inválido o ausente")
	}
	return handler(ctx, req)
}
```

- **Credencial por metadata, nunca en el mensaje.** Igual que en REST no mandás el token en la URL, acá no lo metés en el campo de un mensaje: va en metadata.
- Conecta con tu módulo de [Autenticación y autorización](autenticacion.md): el *qué* (validar JWT, scopes, 401 vs 403) es lo mismo; cambia el *dónde* (interceptor + metadata + `codes.Unauthenticated`/`PermissionDenied`).

**Ejercicios 9**
9.1 🔁 ¿Qué agrega mTLS por encima de TLS server-side?
9.2 🧠 ¿Dónde viaja un token de autenticación en gRPC y por qué NO en un campo del mensaje?
9.3 🧠 ¿Qué status code devolverías ante token ausente, y cuál ante un usuario autenticado pero sin permiso para la operación?

---

## Módulo 10 — Producción: health, reflection, observabilidad y balanceo

**Teoría.** Lo que separa un demo de un servicio gRPC en producción:

- **Health checking** — el protocolo estándar `grpc.health.v1.Health` (paquete `google.golang.org/grpc/health`). Lo registrás y Kubernetes/load balancers lo sondean para saber si tu servicio está vivo. Es el equivalente del `GET /salud` pero estandarizado para gRPC.
- **Server reflection** — registrar `reflection.Register(s)` permite que herramientas como `grpcurl` descubran tus servicios sin tener el `.proto` a mano. Útil en dev/debug; en prod se suele dejar solo en entornos internos.
- **Observabilidad** — instrumentás con **interceptors de OpenTelemetry** (`otelgrpc`): trazas y métricas por RPC automáticamente, con propagación del trace por la metadata. Esto enchufa directo con tu módulo de [Observabilidad](observabilidad.md): el `FullMethod` es tu etiqueta, y el deadline propagado ya te da el span padre.
- **Balanceo de carga** — gRPC usa conexiones **de larga vida** sobre HTTP/2, así que un balanceador L4 (por conexión) manda todo a un solo backend. Necesitás **balanceo L7 / client-side** (resolver + `round_robin`) o un proxy que entienda gRPC (Envoy, o el de tu service mesh). Es la trampa #1 de gRPC en Kubernetes: "escalé las réplicas y el tráfico no se reparte".
- **gRPC-Web / gateway** — el browser no habla gRPC nativo. Si necesitás exponer al frontend, usás **gRPC-Web** (con un proxy como Envoy) o **grpc-gateway** (genera un REST/JSON que traduce a gRPC). Esto refuerza el criterio del Módulo 11: gRPC adentro, REST en el borde.
- **ConnectRPC (Buf)** — ⚠️ evolución del ecosistema `buf` que aparece cada vez más en ofertas 2026: un servidor que habla **gRPC, gRPC-Web y REST/JSON desde el mismo handler**, sin proxy aparte. Resuelve de raíz el "gRPC adentro + browser/REST afuera" que en gRPC clásico te obliga a gateway/proxy. Tenelo en el radar como el camino que simplifica este problema.

**Ejercicios 10**
10.1 🔁 ¿Para qué sirve el health checking estándar de gRPC y quién lo consume?
10.2 🧠 Escalás tu servicio gRPC a 5 réplicas en Kubernetes con un Service normal y el tráfico sigue cayendo casi todo en un pod. ¿Por qué pasa y cómo se resuelve?
10.3 🧠 El equipo de frontend quiere consumir tu servicio gRPC desde React. ¿Qué dos opciones les ofrecés y cuál es el espíritu de la decisión?

---

## Módulo 11 — El criterio: gRPC vs REST vs GraphQL

**Teoría.** No es "gRPC mejor que REST": cada uno ocupa un lugar.

| Elegí… | Cuando… |
|---|---|
| **gRPC** | Comunicación **service-to-service interna**, baja latencia, contratos fuertes, streaming, polyglot (clientes en varios lenguajes desde el mismo `.proto`). El default entre microservicios. |
| **REST/JSON** | API **pública**, para el browser o terceros, donde importan la simplicidad, la cacheabilidad HTTP, los proxies/CDN y que cualquiera la consuma con `curl`. El borde. |
| **GraphQL** | El cliente (típicamente un frontend) necesita **componer datos de varias fuentes** y pedir exactamente los campos que usa, evitando over/under-fetching. Ver [GraphQL](graphql.md). |

Pros y contras honestos de gRPC:
- ✅ Performance (binario + HTTP/2), contrato fuerte y codegen, streaming de primera, cancelación/deadlines propagados, multi-lenguaje.
- ⚠️ No nativo en el browser (necesitás gRPC-Web/gateway), no legible en el cable (debug con herramientas), balanceo en K8s requiere L7, curva de tooling (protobuf, codegen), *overkill* para un CRUD chico o una API pública simple.

La regla mental, alineada con tu stack: **REST/GraphQL en el borde (lo que ya sabés con Node/Nest), gRPC para adentro cuando partís a microservicios.** Tener gRPC en el cinturón es lo que te habilita el rol de backend Go en arquitecturas distribuidas — y conecta con [Diseño de sistemas backend](system-design.md) y [Event-driven](event-driven.md), donde gRPC es el transporte síncrono y las colas el asíncrono.

**Ejercicios 11**
11.1 🧠 Para una API pública que consumen apps móviles de terceros, ¿gRPC o REST? Justificá en dos dimensiones.
11.2 🧠 ¿En qué punto de la vida de un sistema "monolito Node → microservicios" empieza a tener sentido introducir gRPC, y por qué no antes?
11.3 ✍️ Para un sistema de pedidos (servicio **Pedidos**, servicio **Pagos** y un **frontend React**): decidí qué comunicación va en gRPC y cuál en REST, marcá qué RPC convendría que sea *streaming*, y escribí el esqueleto `.proto` del servicio Pedidos (package versionado + un mensaje + un service con 2 RPCs).

---

## Soluciones

### Módulo 1
```
1.1 (a) Contract-first con Protocol Buffers (el .proto genera el código); (b) serialización
    binaria Protobuf (no JSON); (c) HTTP/2 por debajo (multiplexing + streaming); (d) tipado
    fuerte de punta a punta vía el stub generado.
1.2 Porque el contrato (.proto) es la fuente de verdad de la que se GENERA cliente y servidor,
    así que no puede desincronizarse del código. Resuelve el drift de REST, donde el OpenAPI
    se escribe aparte y termina mintiendo respecto de lo que la API hace de verdad.
1.3 Que gRPC no es nativo del browser ni amigable para terceros: necesitarían gRPC-Web + un
    proxy, no es legible/cacheable como REST, y la "velocidad" no es el factor que manda en el
    borde público. Para browser/terceros, REST (o GraphQL); gRPC es para servicio-a-servicio.
```

### Módulo 2
```
2.1 Viaja el NÚMERO del campo, no el nombre. Implica que podés renombrar un campo sin romper
    el cable (el número manda), pero NO podés cambiar el número, y los nombres son solo para
    el código generado.
2.2 Porque el número es la identidad del campo en el binario: si lo reusás para otro campo, un
    cliente viejo interpretará los bytes nuevos como el campo viejo → datos corruptos
    silenciosos. Al borrar un campo, marcás su número (y/o nombre) como `reserved` para que
    nadie lo reutilice.
```
```protobuf
// 2.3
syntax = "proto3";
package productos.v1;
option go_package = "miapp/gen/productos/v1;productosv1";

message Producto {
  string id = 1;
  string nombre = 2;
  int64 precio_centavos = 3;
}

service ProductoService {
  rpc Crear(Producto) returns (Producto);
}
```

### Módulo 3
```
3.1 protoc-gen-go genera los structs Go de los mensajes (Usuario, etc.). protoc-gen-go-grpc
    genera el cliente (stub UsuarioServiceClient) y la interfaz del servidor
    (UsuarioServiceServer) que vos implementás.
3.2 buf breaking detecta en CI si el cambio en el .proto rompería compatibilidad con los
    clientes existentes (renombrar, cambiar tipo, reusar un número) y falla el build ANTES de
    publicar. Regenerar y compilar solo te dice que TU código compila, no que no rompiste a
    los consumidores ya desplegados.
3.3 Porque se regenera cada vez que cambia el .proto y pisaría tus ediciones. Tu lógica va en
    el struct que IMPLEMENTA la interfaz del servidor (y en tus servicios/repos), no en el
    código generado.
```

### Módulo 4
```
4.1 Para forward compatibility: si agregás un método al .proto y todavía no lo implementaste,
    el struct sigue cumpliendo la interfaz (hereda los métodos "no implementados" que
    devuelven codes.Unimplemented) y el servicio compila y arranca igual, en vez de romper.
4.2 Porque los getters generados (GetId) son nil-safe: si el mensaje es nil devuelven el zero
    value en lugar de paniquear, mientras que el acceso directo req.Id desreferencia y puede
    tumbar el handler con un nil pointer.
```
```go
// 4.3
func (s *servidor) Crear(ctx context.Context, req *pb.Producto) (*pb.Producto, error) {
	// ... validar y guardar ...
	return req, nil
}
```

### Módulo 5
```
5.1 Porque la ClientConn es multiplexada sobre HTTP/2 (muchas llamadas concurrentes sobre una
    conexión) y de larga vida; crear una por llamada desperdicia handshakes y recursos, igual
    que abrir una conexión de DB por request. Se crea una vez y se comparte.
5.2 grpc.NewClient conecta de forma lazy: no establece la conexión hasta la primera llamada,
    así que ya no necesitás WithBlock para "esperar a que conecte". El viejo Dial (con
    WithBlock) intentaba conectar al inicio; NewClient simplificó eso y es la API vigente.
5.3 Porque deja el canal sin cifrar (texto plano en la red) y sin autenticar al servidor:
    inaceptable fuera de localhost. En producción lo reemplazás por credenciales TLS
    (credentials.NewClientTLSFromFile / config TLS), y para service-to-service, mTLS.
```

### Módulo 6
```
6.1 Unary (req→resp). Server streaming (req→flujo de resps): export grande, feed de
    notificaciones. Client streaming (flujo de reqs→resp): subir un batch/métricas. Bidi
    streaming (ambos flujos): chat, telemetría en vivo, juego en tiempo real.
6.2 Server streaming: el server manda los registros de a uno con stream.Send y el cliente los
    procesa a medida que llegan, sin cargar el millón en memoria de un lado ni del otro. Un
    unary tendría que construir y serializar la respuesta entera (memoria + latencia + riesgo
    de exceder el tamaño máximo de mensaje).
```
```go
// 6.3
stream, err := cliente.ListarUsuarios(ctx, &pb.ListarRequest{})
if err != nil {
	return nil, err
}
var nombres []string
for {
	u, err := stream.Recv()
	if err == io.EOF {
		break
	}
	if err != nil {
		return nil, err
	}
	nombres = append(nombres, u.GetNombre())
}
return nombres, nil
```

### Módulo 7
```
7.1 Se traduce a codes.Unknown: el cliente recibe un error genérico sin un código accionable
    (no puede distinguir "no encontrado" de "argumento inválido" de "error interno"). Por eso
    se devuelve siempre un status explícito.
7.2 Para no filtrar detalles internos (estructura de tablas, mensajes del driver, rutas) al
    cliente —es info sensible y acopla— y porque desde la óptica del llamador es un fallo del
    servidor (Internal), no algo que él pueda corregir. El detalle real se loguea del lado
    server.
```
```go
// 7.3
if req.GetEdad() < 0 {
	return nil, status.Error(codes.InvalidArgument, "edad no puede ser negativa")
}
```

### Módulo 8
```
8.1 Son los "headers" de gRPC: pares clave-valor que acompañan la llamada (fuera del mensaje).
    Se usan para tokens de auth, request IDs, info de tracing/propagación de contexto, etc.
8.2 Porque el ctx con deadline del cliente se serializa y se manda al server, que lo recibe
    como su propio deadline; si expira, su ctx se cancela y la cancelación se propaga a las
    llamadas que él haga aguas abajo. Resuelve el problema de cadenas de microservicios que
    siguen trabajando para una request que el cliente original ya abandonó.
8.3 Porque cada RPC corre en su goroutine y un panic no recuperado tumba TODO el proceso (no
    solo esa request). Un interceptor de recover atrapa el panic, lo convierte en
    codes.Internal y mantiene el servidor vivo para el resto de las llamadas.
```

### Módulo 9
```
9.1 mTLS hace que el SERVER también verifique al cliente por su certificado (autenticación
    mutua), no solo el cliente al server. Así la identidad del servicio llamador queda probada
    por el cert, ideal para tráfico service-to-service sin manejar tokens.
9.2 Viaja en la metadata (clave authorization), no en un campo del mensaje. Porque es una
    preocupación de transporte/transversal (la valida un interceptor antes del handler), no
    dato de dominio; meterlo en el mensaje lo mezcla con el payload, se loguea por accidente y
    obliga a tocar el .proto por algo que no es del contrato de negocio.
9.3 Token ausente/ inválido → codes.Unauthenticated (≈401). Usuario autenticado pero sin
    permiso para la operación → codes.PermissionDenied (≈403).
```

### Módulo 10
```
10.1 Es un protocolo estándar (grpc.health.v1) para reportar si el servicio está sano. Lo
     consumen orquestadores y balanceadores (Kubernetes liveness/readiness, load balancers)
     para decidir si mandarle tráfico o reiniciarlo. Es el "/salud" estandarizado de gRPC.
10.2 Porque gRPC usa una conexión HTTP/2 de larga vida y un Service/LB L4 balancea por
     conexión, no por request: una vez establecida, todas las llamadas van al mismo pod. Se
     resuelve con balanceo L7 / client-side (resolver headless + round_robin) o un proxy que
     entienda gRPC (Envoy / service mesh).
10.3 (a) grpc-gateway: genera una fachada REST/JSON que traduce a gRPC. (b) gRPC-Web con un
     proxy (Envoy). El espíritu: el browser no habla gRPC nativo, así que exponés REST en el
     borde y mantenés gRPC adentro — borde REST, interior gRPC.
```

### Módulo 11
```
11.1 REST. (a) Alcance/consumo: terceros y apps móviles consumen REST/JSON trivialmente (HTTP,
     curl, cualquier lenguaje) sin tooling de protobuf ni proxies; gRPC no es nativo del
     browser y es incómodo de exponer afuera. (b) Operación del borde: REST aprovecha caching
     HTTP, CDNs y proxies estándar, claves en una API pública.
11.2 Cuando ya tenés VARIOS servicios que se llaman entre sí y empezás a sentir el dolor de
     contratos REST que se desincronizan, serialización JSON pesada o falta de streaming.
     Antes (monolito o uno o dos servicios) gRPC suma tooling y complejidad sin pagar: la
     comunicación interna no es todavía el cuello de botella ni la fuente de bugs de contrato.
11.3 Pedidos↔Pagos: gRPC (service-to-service interno, contrato fuerte, baja latencia).
     React↔Pedidos: REST (o gRPC-Web/grpc-gateway/ConnectRPC) — el browser no habla gRPC
     nativo. Streaming: "seguir el estado del pedido" calza como server streaming (el server
     empuja cambios de estado en vivo). Esqueleto:
```
```protobuf
syntax = "proto3";
package pedidos.v1;
option go_package = "miapp/gen/pedidos/v1;pedidosv1";

message Pedido {
  string id = 1;
  string estado = 2;
  int64 total_centavos = 3;
}
message CrearPedidoRequest { repeated string producto_ids = 1; }
message SeguirRequest { string id = 1; }

service PedidoService {
  rpc CrearPedido(CrearPedidoRequest) returns (Pedido);
  rpc SeguirPedido(SeguirRequest) returns (stream Pedido); // server streaming: estado en vivo
}
```

---

## Siguientes pasos

Con este módulo sumás el **transporte síncrono de microservicios** que define a un backend Go en arquitecturas distribuidas. El recorrido natural desde acá: **(1)** volvé a [Go para backend](go-backend.md) y armá un servicio que exponga **gRPC adentro y REST en el borde** (grpc-gateway), aplicando el criterio del Módulo 11; **(2)** conectá la teoría con [Diseño de sistemas backend](system-design.md) y [Event-driven](event-driven.md): gRPC es el camino síncrono, las colas el asíncrono — saber cuándo usar cada uno es lo que se evalúa a nivel senior; **(3)** instrumentá tus RPCs con los interceptors de OpenTelemetry y cerrá el loop con [Observabilidad](observabilidad.md). La constante: gRPC no reemplaza a REST — te da la herramienta para el tráfico interno, donde el contrato fuerte y la performance mandan.
