# Patrones de diseño en NestJS

**De los que el framework te impone a los que aplicás por criterio · orientado a lo laboral**

> Cómo usar esta página: leé la teoría de cada módulo, intentá los ejercicios **sin mirar las soluciones**, y contrastá al final. Los ejemplos asumen que ya configuraste un proyecto Nest (`nest new`) con `tsconfig` estricto y `experimentalDecorators` activado (viene por defecto). No hace falta correr todo: el objetivo es entender el *porqué* de cada patrón. Los snippets **omiten los imports** por brevedad (`@nestjs/common`, `class-validator`, `class-transformer`, `rxjs`, `@nestjs/cqrs`); en un proyecto real los traés de esos paquetes.

**Índice**
1. Inversión de Control e Inyección de Dependencias (DI)
2. Patrón Módulo
3. Pipeline de request: guards, pipes, interceptors y filters
4. Repository pattern
5. DTO + Mapper
6. Provider patterns (useClass / useValue / useFactory) y Strategy
7. CQRS + Mediator (introducción)
8. Ports & Adapters (Hexagonal)
9. Custom decorators (parameter decorators)

Las soluciones de todos los ejercicios están al final.

---

## Módulo 1 — Inversión de Control e Inyección de Dependencias (DI)

**Teoría.** Es el corazón de NestJS. En vez de que una clase **cree** sus dependencias (`new EmailService()`), las **recibe** desde afuera. Un contenedor (IoC container) sabe cómo construir cada provider y se las inyecta. Marcás una clase como inyectable con `@Injectable()` y la pedís por el constructor.

```ts
@Injectable()
class EmailService {
  enviar(a: string, msg: string) { /* ... */ }
}

@Injectable()
class UsuarioService {
  // El contenedor inyecta EmailService automáticamente
  constructor(private readonly email: EmailService) {}

  registrar(correo: string) {
    this.email.enviar(correo, "Bienvenido");
  }
}
```

¿Por qué importa? Porque `UsuarioService` no depende de *cómo* se construye `EmailService`. En un test podés inyectar un mock. Ese desacople es lo que habilita todo lo demás (Repository, Hexagonal, etc.).

Los providers son **singleton por defecto** (una sola instancia para toda la app). Existen otros scopes para casos puntuales: **`REQUEST`** crea una instancia por request (útil para datos del request actual, p. ej. multi-tenant o el usuario autenticado), y **`TRANSIENT`** entrega una instancia nueva a cada consumidor (cuando cada uno necesita su propio estado). Ambos tienen costo de performance, así que el default singleton es lo normal.

**Ejercicios 1**
1.1 Explicá en una o dos frases la diferencia entre "crear" una dependencia con `new` dentro de una clase y "inyectarla" por constructor. ¿Qué problema de testeo resuelve la inyección?
1.2 Tenés un `PagoService` que necesita un `LoggerService`. Escribí ambas clases con los decoradores correctos y la inyección por constructor.
1.3 ¿Qué significa que un provider sea "singleton" en Nest? Si dos servicios distintos inyectan el mismo `LoggerService`, ¿reciben la misma instancia o instancias distintas (con el scope por defecto)?

---

## Módulo 2 — Patrón Módulo

**Teoría.** Nest organiza la app en **módulos**: unidades que agrupan controllers y providers relacionados a una feature. Un módulo declara qué provee (`providers`), qué expone a otros módulos (`exports`), qué importa (`imports`) y qué controllers tiene.

```ts
@Module({
  imports: [],
  controllers: [UsuarioController],
  providers: [UsuarioService, EmailService],
  exports: [UsuarioService], // otros módulos pueden inyectar UsuarioService
})
export class UsuarioModule {}
```

Reglas clave: un provider solo es visible fuera de su módulo si está en `exports`. Para usar un provider de otro módulo, hay que `import`ar ese módulo. Esto te fuerza a pensar **límites** entre features — la base de un *monolito modular* limpio.

Los **módulos dinámicos** (`forRoot`/`forFeature`) permiten configurar un módulo al importarlo (típico en módulos de DB, config, etc.) y se apoyan en el patrón Factory (módulo 6).

**Ejercicios 2**
2.1 Tenés un `AuthService` definido en `AuthModule` y querés usarlo dentro de `UsuarioService` (en `UsuarioModule`). Listá los dos cambios necesarios (uno en cada módulo) para que la inyección funcione.
2.2 Verdadero o falso, justificá: "Si un provider está en `providers` de un módulo, cualquier otro módulo puede inyectarlo automáticamente".
2.3 Diseñá (solo los decoradores `@Module`, sin lógica) tres módulos: `ProductoModule`, `PedidoModule` y un `SharedModule` que exporte un `LoggerService` usado por los otros dos.

---

## Módulo 3 — Pipeline de request: guards, pipes, interceptors y filters

**Teoría.** Nest aplica programación orientada a aspectos: separa *cross-cutting concerns* (autorización, validación, logging, errores) del código de negocio. Cada request pasa por una cadena:

`Middleware → Guard → Interceptor (antes) → Pipe → Handler → Interceptor (después) → Exception Filter (si hay error)`

- **Guard** — decide si el request sigue (autorización). Devuelve `boolean | Promise<boolean> | Observable<boolean>` (puede ser async). Implementa `CanActivate`.
- **Pipe** — transforma/valida el input antes de llegar al handler (ej. `ValidationPipe` con DTOs).
- **Interceptor** — envuelve el handler: podés transformar la respuesta, medir tiempo, cachear. Patrón decorador/proxy.
- **Exception Filter** — captura excepciones y arma la respuesta de error centralizada.

```ts
@Injectable()
class RolesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    return req.user?.rol === "admin";
  }
}
```

La gracia: estos se aplican con decoradores (`@UseGuards`, `@UsePipes`, `@UseInterceptors`) a nivel método, controller o global, sin ensuciar la lógica.

Un detalle fino del orden: como el **guard corre antes que los pipes**, dentro de un guard el body **todavía no está validado ni transformado** por el `ValidationPipe`. Si tu guard necesitara datos del body ya validados, repensá el diseño (eso va en un interceptor o en el handler, no en el guard).

**Ejercicios 3**
3.1 Para cada necesidad, indicá cuál de los cuatro (guard / pipe / interceptor / filter) corresponde:
  (a) Rechazar el request si el usuario no tiene rol admin.
  (b) Convertir todas las respuestas exitosas a `{ data: ... , timestamp }`.
  (c) Validar que el body cumpla un DTO.
  (d) Devolver siempre un JSON de error uniforme cuando algo lanza excepción.
3.2 Escribí un `AuthGuard` que implemente `CanActivate` y devuelva `true` solo si existe el header `authorization`.
3.3 Escribí un `LoggingInterceptor` que loguee cuánto tardó el handler (usá `intercept` y el stream que devuelve; basta con describir el flujo si no recordás RxJS).

---

## Módulo 4 — Repository pattern

**Teoría.** El Repository abstrae el acceso a datos detrás de una **interfaz**. El service depende de la interfaz, no de la base de datos concreta. Así podés cambiar de Postgres a otra cosa, o usar un mock en tests, sin tocar la lógica.

```ts
// El puerto (interfaz)
interface UsuarioRepository {
  obtenerPorId(id: number): Promise<Usuario | null>;
  guardar(u: Usuario): Promise<Usuario>;
}

@Injectable()
class UsuarioService {
  constructor(
    // inyectamos por un token, contra la interfaz
    @Inject("UsuarioRepository") private readonly repo: UsuarioRepository,
  ) {}

  buscar(id: number) { return this.repo.obtenerPorId(id); }
}
```

Como TypeScript borra las interfaces en runtime, en Nest se inyecta usando un **token** (string o symbol) y un provider que mapea ese token a la implementación concreta. Ese registro vive en el módulo, y el token del `@Inject` **debe coincidir exactamente** con el `provide`:

```ts
export const USUARIO_REPO = Symbol("UsuarioRepository"); // token centralizado

@Module({
  providers: [
    { provide: USUARIO_REPO, useClass: UsuarioRepositoryPostgres },
  ],
})
export class UsuarioModule {}

// en el service:
// constructor(@Inject(USUARIO_REPO) private readonly repo: UsuarioRepository) {}
```

Usá un **`Symbol` en una constante** (no un string mágico repetido): evita colisiones y errores de tipeo entre el `provide` y el `@Inject`.

**Ejercicios 4**
4.1 ¿Por qué tipar el service contra `UsuarioRepository` (interfaz) y no contra `UsuarioRepositoryPostgres` (clase concreta) facilita los tests?
4.2 Definí una interfaz `ProductoRepository` con `listar(): Promise<Producto[]>` y `crear(p: Producto): Promise<Producto>`. Escribí una implementación en memoria `ProductoRepositoryMemoria` que la cumpla (útil para tests).
4.3 Escribí el provider (objeto `{ provide, useClass }`) que registraría `ProductoRepositoryMemoria` bajo el token `"ProductoRepository"` dentro de un módulo.

---

## Módulo 5 — DTO + Mapper

**Teoría.** Un **DTO** (Data Transfer Object) define la forma de los datos que entran o salen por la API, separada de tu entidad de dominio/persistencia. Esto te permite: validar entradas, no exponer campos sensibles (ej. `password`), y versionar la API sin tocar el dominio.

```ts
// Entrada: lo que el cliente manda para crear
class CrearUsuarioDTO {
  @IsEmail() email!: string;
  @MinLength(8) password!: string;
}

// Salida: lo que devolvemos (sin password)
class UsuarioPublicoDTO {
  id!: number;
  email!: string;
}
```

El **Mapper** es la función/clase que convierte entidad ↔ DTO. Mantener ese mapeo en un solo lugar evita que datos sensibles se filtren por accidente.

La forma idiomática de Nest para no exponer campos sensibles, además del mapper manual, es **`class-transformer` + `ClassSerializerInterceptor`**: marcás los campos a ocultar con `@Exclude()` en la entidad y Nest los filtra al serializar la respuesta:

```ts
import { Exclude } from "class-transformer";

class Usuario {
  id!: number;
  email!: string;
  @Exclude() password!: string; // nunca sale en la respuesta
}
// se activa global con: app.useGlobalInterceptors(new ClassSerializerInterceptor(reflector))
```

El mapper manual da más control (DTOs distintos por endpoint y por versión de API); el `ClassSerializerInterceptor` es más rápido de aplicar. Las dos vías se usan en producción, y conocer la segunda es lo que se espera en entrevistas.

**Ejercicios 5**
5.1 Nombrá dos riesgos concretos de devolver directamente la entidad de la base de datos en la respuesta HTTP en vez de un DTO de salida.
5.2 Dado `interface Usuario { id: number; email: string; password: string; creadoEl: Date }`, escribí un mapper `aUsuarioPublico(u: Usuario)` que devuelva solo `{ id, email }` (tipado, sin `password`).
5.3 Escribí un `CrearProductoDTO` con `nombre` (string no vacío) y `precio` (number positivo), usando decoradores de `class-validator` (`@IsString`, `@IsNotEmpty`, `@IsNumber`, `@Min`).

---

## Módulo 6 — Provider patterns y Strategy

**Teoría.** Nest no solo inyecta clases; podés controlar *cómo* se construye cada provider:

- `useClass` — instancia una clase (lo más común).
- `useValue` — provee un valor/objeto ya hecho (ideal para config o mocks en tests).
- `useFactory` — ejecuta una función para construir el provider, con dependencias inyectadas (Factory pattern). Sirve para configuración asíncrona o condicional.
- `useExisting` — alias de otro provider.

```ts
const config = { provide: "CONFIG", useValue: { puerto: 3000 } };

const repo = {
  provide: "UsuarioRepository",
  useFactory: (cfg: AppConfig) => cfg.usarMemoria
    ? new UsuarioRepositoryMemoria()
    : new UsuarioRepositoryPostgres(),
  inject: [AppConfig],
};
```

El **patrón Strategy** aparece de forma natural en auth: cada estrategia de Passport (JWT, local, Google) es una implementación intercambiable de "cómo autenticar". Elegís la estrategia sin cambiar el resto.

**Ejercicios 6**
6.1 Para cada caso elegí `useClass`, `useValue`, `useFactory` o `useExisting`:
  (a) Inyectar un objeto de configuración fijo leído de variables de entorno.
  (b) Decidir en runtime, según config, qué implementación de repositorio usar.
  (c) Registrar un service normal.
  (d) Exponer un provider ya existente bajo un segundo token (alias).
6.2 Escribí un provider con `useValue` que registre bajo el token `"MAILER"` un mock con un método `enviar` que no haga nada (para tests).
6.3 Explicá con tus palabras por qué Passport + estrategias es un ejemplo del patrón Strategy. ¿Qué es lo "intercambiable"?

---

## Módulo 7 — CQRS + Mediator (introducción)

**Teoría.** **CQRS** (Command Query Responsibility Segregation) separa las operaciones que **cambian estado** (commands) de las que **leen** (queries). Cada comando/consulta tiene su *handler*. Un **Mediator** (en Nest, el `CommandBus`/`QueryBus`) recibe el comando y lo despacha al handler correcto, desacoplando quien dispara la acción de quien la ejecuta.

```ts
class CrearPedidoCommand {
  constructor(public readonly usuarioId: number, public readonly items: string[]) {}
}

@CommandHandler(CrearPedidoCommand)
class CrearPedidoHandler implements ICommandHandler<CrearPedidoCommand> {
  async execute(cmd: CrearPedidoCommand) {
    // lógica de creación
  }
}

// en el controller: this.commandBus.execute(new CrearPedidoCommand(id, items))
```

¿Cuándo usarlo? En dominios complejos donde lecturas y escrituras tienen necesidades muy distintas, o donde querés trazabilidad por evento. **No** es para un CRUD simple: ahí agrega complejidad sin beneficio.

**Ejercicios 7**
7.1 Explicá la "Q" y la "C" de CQRS: ¿qué tipo de operación va en cada lado y por qué separarlas?
7.2 ¿Qué rol cumple el `CommandBus` (Mediator)? ¿Qué desacopla?
7.3 Da un ejemplo de aplicación donde CQRS valga la pena y uno donde sea sobre-ingeniería.

---

## Módulo 8 — Ports & Adapters (Hexagonal)

**Teoría.** La arquitectura hexagonal (puertos y adaptadores) lleva la DI a nivel arquitectónico: el **núcleo de dominio** (entidades y casos de uso) no conoce ni la base de datos, ni el framework, ni HTTP. Se comunica con el exterior a través de **puertos** (interfaces). Las implementaciones concretas (Postgres, un cliente de email, un controller HTTP) son **adaptadores** que enchufan en esos puertos.

```
        HTTP / CLI (adaptador de entrada)
                  │
            Caso de uso (dominio) ──► Puerto (interfaz)
                                          ▲
                          Adaptador (PostgresRepo, EmailSMTP)
```

En Nest esto se logra: definís interfaces (puertos), inyectás contra ellas con tokens, y registrás los adaptadores en el módulo. El beneficio: testeás el dominio sin levantar nada, y cambiás infraestructura sin tocar la lógica. Es el patrón más valorado en entrevistas mid/senior.

**Ejercicios 8**
8.1 Clasificá cada elemento como **puerto**, **adaptador** o **dominio**:
  (a) `interface NotificadorPort`
  (b) `class EmailNotificador implements NotificadorPort`
  (c) `class CalcularPrecioPedido` (lógica de negocio pura)
  (d) `class PedidoController` (HTTP)
8.2 Definí un puerto `PagoPort` con `cobrar(monto: number, token: string): Promise<boolean>` y dos adaptadores que lo implementen: `StripeAdapter` y `PagoFalsoAdapter` (este último para tests, siempre devuelve `true`).
8.3 Explicá en dos o tres frases por qué un caso de uso que depende de `PagoPort` (y no de `StripeAdapter`) es más fácil de testear y de mantener.

---

## Módulo 9 — Custom decorators (parameter decorators)

**Teoría.** Además de los decoradores que trae Nest, podés crear los tuyos con `createParamDecorator` para extraer datos del request de forma declarativa y reutilizable. El caso típico es `@CurrentUser()`: saca el usuario que un guard de auth dejó en `request.user`, evitando repetir `req.user` (y su casteo) en cada handler.

```ts
import { createParamDecorator, ExecutionContext } from "@nestjs/common";

export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext) => {
    const req = ctx.switchToHttp().getRequest<{ user?: unknown }>();
    return req.user;
  },
);

// uso en el handler:
@Get("perfil")
perfil(@CurrentUser() user: Usuario) {
  return user;
}
```

Es uno de los patrones más usados y preguntados en NestJS real: encapsula el acceso al request, mantiene los handlers limpios y es trivialmente testeable. Combinado con un guard de auth, `@CurrentUser()` es el estándar de facto. El parámetro `data` del decorador permite parametrizarlo (ej. `@CurrentUser("email")` para extraer solo un campo).

**Ejercicios 9**
9.1 ¿Qué ventaja da `@CurrentUser()` frente a leer `req.user` con `@Req()` en cada handler?
9.2 Escribí un decorator `@HeaderValue(nombre)` con `createParamDecorator` que extraiga un header arbitrario del request por su nombre (usá el parámetro `data`).

---

# Soluciones

> Mirá esto solo después de intentarlo. Los snippets omiten imports por brevedad y asumen los tipos de dominio `Usuario` y `Producto`.

### Módulo 1
```
1.1 Con `new` la clase queda acoplada a una implementación concreta y no podés
    sustituirla; con inyección recibís la dependencia desde afuera, así que en
    un test podés pasar un mock en vez de la implementación real.
```
```ts
// 1.2
@Injectable()
class LoggerService { log(msg: string) { console.log(msg); } }
// (En producción se usa el Logger de @nestjs/common en vez de console.log directo.)

@Injectable()
class PagoService {
  constructor(private readonly logger: LoggerService) {}
  cobrar(monto: number) { this.logger.log(`Cobrando ${monto}`); }
}
```
```
1.3 Singleton = una única instancia compartida en toda la app (con el scope por
    defecto). Los dos services reciben LA MISMA instancia de LoggerService.
```

### Módulo 2
```
2.1 (1) En AuthModule: agregar AuthService a `exports`.
    (2) En UsuarioModule: agregar AuthModule a `imports`.
2.2 Falso. Un provider solo es inyectable desde otro módulo si está en `exports`
    de su módulo y ese módulo es importado. `providers` lo hace visible solo
    dentro del propio módulo.
```
```ts
// 2.3
@Module({ providers: [LoggerService], exports: [LoggerService] })
export class SharedModule {}

@Module({ imports: [SharedModule], controllers: [], providers: [] })
export class ProductoModule {}

@Module({ imports: [SharedModule], controllers: [], providers: [] })
export class PedidoModule {}
```

### Módulo 3
```
3.1 (a) Guard  (b) Interceptor  (c) Pipe  (d) Exception Filter
```
```ts
// 3.2
@Injectable()
class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest();
    return Boolean(req.headers["authorization"]);
  }
}

// 3.3 (con RxJS)
@Injectable()
class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<unknown> {
    const inicio = Date.now();
    return next
      .handle()
      .pipe(tap(() => console.log(`Tardó ${Date.now() - inicio}ms`)));
  }
}
// Flujo: guardás el tiempo de inicio, dejás correr el handler con next.handle(),
// y cuando el stream emite (tap), logueás la diferencia.
```

### Módulo 4
```
4.1 Porque en el test podés inyectar una implementación falsa/en memoria que
    cumpla la interfaz, sin necesidad de una base de datos real. El service no
    sabe ni le importa cuál implementación recibe.
```
```ts
// 4.2
interface ProductoRepository {
  listar(): Promise<Producto[]>;
  crear(p: Producto): Promise<Producto>;
}

@Injectable()
class ProductoRepositoryMemoria implements ProductoRepository {
  private items: Producto[] = [];
  async listar(): Promise<Producto[]> { return this.items; }
  async crear(p: Producto): Promise<Producto> { this.items.push(p); return p; }
}

// 4.3
const productoRepoProvider = {
  provide: "ProductoRepository",
  useClass: ProductoRepositoryMemoria,
};
```

### Módulo 5
```
5.1 (1) Filtrás datos sensibles sin querer (password, tokens, flags internos).
    (2) Acoplás tu API a la forma de la tabla: cualquier cambio de esquema rompe
        el contrato con los clientes.
```
```ts
// 5.2
interface Usuario { id: number; email: string; password: string; creadoEl: Date; }
function aUsuarioPublico(u: Usuario): { id: number; email: string } {
  return { id: u.id, email: u.email };
}

// 5.3
class CrearProductoDTO {
  @IsString() @IsNotEmpty() nombre!: string;
  @IsNumber() @Min(0.01) precio!: number;
}
```

### Módulo 6
```
6.1 (a) useValue   (b) useFactory   (c) useClass   (d) useExisting
6.3 Es Strategy porque "cómo autenticar" es un algoritmo intercambiable: JWT,
    local, Google son estrategias distintas con la misma interfaz. El resto de
    la app pide "autenticar" sin saber cuál se usa; cambiar de estrategia no
    cambia el código que la consume.
```
```ts
// 6.2
const mailerMock = {
  provide: "MAILER",
  useValue: { enviar: (_a: string, _msg: string) => undefined },
};
```

### Módulo 7
```
7.1 La "C" (Command) son operaciones que cambian estado (crear, actualizar,
    borrar). La "Q" (Query) son lecturas. Se separan porque sus necesidades
    difieren: las lecturas pueden optimizarse/cachearse y las escrituras
    necesitan validación e invariantes; separarlas simplifica cada lado.
7.2 El CommandBus (Mediator) recibe un comando y lo despacha al handler
    registrado para ese tipo. Desacopla a quien dispara la acción (ej. un
    controller) de quien la ejecuta (el handler).
7.3 Vale la pena: una plataforma de e-commerce/finanzas con reglas de negocio
    complejas y auditoría por evento. Sobre-ingeniería: un CRUD de notas o una
    ABM simple, donde un service directo alcanza.
```

### Módulo 8
```
8.1 (a) puerto   (b) adaptador   (c) dominio   (d) adaptador
```
```ts
// 8.2
interface PagoPort {
  cobrar(monto: number, token: string): Promise<boolean>;
}

@Injectable()
class StripeAdapter implements PagoPort {
  async cobrar(_monto: number, _token: string): Promise<boolean> {
    // acá iría la llamada real a Stripe usando monto y token
    return true;
  }
}

@Injectable()
class PagoFalsoAdapter implements PagoPort {
  async cobrar(_monto: number, _token: string): Promise<boolean> {
    return true; // siempre OK, para tests
  }
}
```
```
8.3 Porque el caso de uso depende de una abstracción (PagoPort), no de Stripe.
    En tests inyectás PagoFalsoAdapter y probás la lógica sin red ni claves; en
    producción cambiás de proveedor de pago implementando otro adaptador, sin
    tocar el dominio.
```

### Módulo 9
```
9.1 Encapsula y nombra la intención (`user`), evita repetir el acceso crudo a
    `req.user` y su casteo en cada handler, y centraliza el cambio si la forma del
    request cambia. Deja los handlers más limpios y declarativos.
```
```ts
// 9.2
export const HeaderValue = createParamDecorator(
  (nombre: string, ctx: ExecutionContext) => {
    const req = ctx
      .switchToHttp()
      .getRequest<{ headers: Record<string, string | undefined> }>();
    return req.headers[nombre.toLowerCase()];
  },
);
// uso: perfil(@HeaderValue("x-tenant-id") tenant?: string) { ... }
```

---

## Cómo seguir

Construí un módulo Nest real aplicando, en orden: **DI** (módulo 1) → **Module + Repository** (2 y 4) → **DTO + Pipe de validación** (3 y 5). Cuando eso te salga sin pensar, refactorizá a **Hexagonal** (módulo 8). Dejá **CQRS** (módulo 7) para cuando tengas un dominio que de verdad lo pida. Ese recorrido cubre lo que se evalúa en entrevistas de full stack mid.
