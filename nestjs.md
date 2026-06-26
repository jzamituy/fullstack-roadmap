# NestJS: teoría + ejercicios para tu primer backend

**Para frontend en transición a full stack · de lo básico a lo avanzado · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y recién después contrastá con la sección final. NestJS es un framework de Node.js sobre TypeScript que usa **decoradores** y **inyección de dependencias**; si venís de React, pensalo como "Angular para el backend". Para practicar, arrancá un proyecto con `npm i -g @nestjs/cli` y `nest new mi-api` (o probá los snippets mentalmente; el foco acá es entender los patrones). El `tsconfig` que genera el CLI ya viene en modo estricto y con `experimentalDecorators` + `emitDecoratorMetadata`. Este último, junto con `reflect-metadata`, es lo que habilita la **DI por tipo**: Nest lee el tipo del parámetro del constructor para saber qué inyectar. Por eso las **clases** se inyectan sin token, pero las **interfaces** (que no existen en runtime) sí lo necesitan.

**Lo que asumimos.** TypeScript con soltura (clases, interfaces, genéricos, decoradores y modificadores de acceso), Node.js (`npm`, módulos, `async/await`) y nociones de HTTP (request/response, métodos GET/POST, status codes). Si venís de Angular, la DI y los decoradores te van a resultar familiares.

> **¿Te falta alguna base?** Nest se para sobre dos pilares. (1) El **sistema de tipos de TS**: la inyección de dependencias lee el tipo del constructor en runtime, así que clases, interfaces y decoradores no son opcionales acá. Si los ves flojos, repasá [TypeScript](typescript.md) —sobre todo clases e interfaces (módulo 10)— antes de seguir, o los módulos 3, 5 y 11 te van a costar el doble. (2) **Node por dentro**: Nest es una capa sobre el módulo `http` de Node; si el ciclo de vida de un request o el event loop te suenan nuevos, pegale una mirada a [Node por dentro](nodejs.md). Convertir el prerrequisito en rampa, no en muro.

**Índice de módulos**
1. Arquitectura: módulos, controladores y providers
2. Controladores y rutas HTTP
3. Providers e inyección de dependencias
4. Módulos: organización y encapsulamiento
5. DTOs y validación con `class-validator`
6. Pipes: transformación y validación
7. Manejo de errores y excepciones
8. Guards: autenticación y autorización
9. Interceptors y middleware
10. Configuración y variables de entorno
11. Persistencia: el patrón Repository
12. Testing: unitario con mocks y e2e

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Arquitectura: módulos, controladores y providers

**Teoría.** NestJS organiza todo en tres piezas que se repiten en cada feature:

- **Controlador** (`@Controller`): recibe requests HTTP y devuelve responses. Es la capa de entrada; no debería tener lógica de negocio.
- **Provider / Service** (`@Injectable`): contiene la lógica de negocio. Es reutilizable e inyectable.
- **Módulo** (`@Module`): agrupa controladores y providers relacionados. La app es un árbol de módulos a partir de `AppModule`.

```ts
// users.service.ts
@Injectable()
export class UsersService {
  findAll(): string[] {
    return ["Ana", "Luis"];
  }
}

// users.controller.ts
@Controller("users")
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  findAll(): string[] {
    return this.usersService.findAll();
  }
}

// users.module.ts
@Module({
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

El flujo siempre es el mismo: **request → controlador → service → respuesta**. El controlador delega; el service trabaja.

**Ejercicios 1**
1.1 Explicá en una frase la diferencia de responsabilidad entre un controlador y un service.
1.2 Tenés un service `ProductsService` con un método `findAll()`. Escribí el `ProductsController` que exponga `GET /products` delegando en ese service (usá `constructor` con inyección).
1.3 Escribí el `ProductsModule` que registre el controlador y el service del ejercicio anterior.

---

## Módulo 2 — Controladores y rutas HTTP

**Teoría.** Cada método HTTP tiene su decorador: `@Get`, `@Post`, `@Put`, `@Patch`, `@Delete`. Los datos del request se extraen con decoradores de parámetro:

- `@Param("id")` — segmentos de la ruta (`/users/:id`).
- `@Query("q")` — query string (`/users?q=ana`).
- `@Body()` — el cuerpo del request (JSON).
- `@Headers()`, `@Req()`, `@Res()` — acceso de bajo nivel (evitá `@Res` salvo que lo necesites).

```ts
@Controller("users")
export class UsersController {
  @Get(":id")
  findOne(@Param("id") id: string): string {
    return `usuario ${id}`;
  }

  @Post()
  create(@Body() body: { nombre: string }): { creado: string } {
    return { creado: body.nombre };
  }
}
```

Nest serializa automáticamente lo que retornás a JSON y responde `200` (o `201` en `@Post`). Para cambiar el código usás `@HttpCode(204)`. Ojo con `@Res()`: si respondés a mano (`res.json(...)`), **desactivás la serialización y el manejo de status de Nest** salvo que pases `@Res({ passthrough: true })`. Por eso conviene evitar `@Res()` salvo necesidad real.

**Ejercicios 2**
2.1 Escribí un método que maneje `GET /products/:id` y devuelva `{ id }` (el `id` viene como string del param).
2.2 Escribí un método para `GET /products?categoria=...` que reciba la query `categoria` (opcional) y devuelva un string indicando por qué categoría se filtra.
2.3 Escribí un `DELETE /products/:id` que responda con status `204` (sin contenido). ¿Qué decorador necesitás?

---

## Módulo 3 — Providers e inyección de dependencias

**Teoría.** La **inyección de dependencias (DI)** es el corazón de Nest. Vos declarás qué necesita una clase en su constructor, y Nest construye y entrega esas dependencias por vos (las resuelve desde el módulo). Esto se apoya en lo que practicaste en TypeScript: tipás contra **interfaces/clases** y dejás que el contenedor decida la implementación concreta.

```ts
@Injectable()
export class UsersService {
  // Nest inyecta una instancia de Database automáticamente
  constructor(private readonly db: Database) {}
}
```

Cuando querés inyectar algo que **no es una clase** (una interfaz, un valor, una config), usás un **custom provider** con un *token* y `@Inject`:

```ts
@Module({
  providers: [
    { provide: "CONFIG", useValue: { apiKey: "abc" } },
  ],
})
export class AppModule {}

@Injectable()
export class ServicioX {
  constructor(@Inject("CONFIG") private readonly config: { apiKey: string }) {}
}
```

Las interfaces de TS **no existen en runtime**, por eso para inyectar "una interfaz" usás un token (string o `Symbol`) y `useClass`/`useValue`. En producción se prefiere un **`Symbol` centralizado en una constante** (`export const MAILER = Symbol('MAILER')`) en vez de strings mágicos repetidos: evita colisiones y errores de tipeo entre el `provide` y el `@Inject`.

**Ejercicios 3**
3.1 Tenés `@Injectable() class Logger { log(msg: string): void {} }`. Escribí un `OrdersService` que lo reciba por constructor y tenga un método `crear()` que loguee `"orden creada"`.
3.2 Definí una interfaz `MailerPort { enviar(to: string): Promise<void> }`. Registrala en un módulo con el token `"MAILER"` usando `useClass` con una implementación `ConsoleMailer`.
3.3 Escribí el service que inyecte ese `MAILER` con `@Inject("MAILER")` tipado como `MailerPort`. Explicá en una frase por qué hace falta el token.

---

## Módulo 4 — Módulos: organización y encapsulamiento

**Teoría.** Un `@Module` declara cuatro listas:

- `controllers` — los controladores del módulo.
- `providers` — los servicios disponibles **dentro** del módulo.
- `imports` — otros módulos cuyos *exports* querés usar acá.
- `exports` — los providers que este módulo hace visibles a quien lo importe.

Por defecto un provider es **privado** del módulo. Si `UsersModule` quiere usar `AuthService` de `AuthModule`, entonces `AuthModule` debe exportarlo **y** `UsersModule` debe importarlo.

```ts
@Module({
  providers: [AuthService],
  exports: [AuthService], // sin esto, nadie afuera puede inyectarlo
})
export class AuthModule {}

@Module({
  imports: [AuthModule], // ahora UsersService puede inyectar AuthService
  providers: [UsersService],
})
export class UsersModule {}
```

Para servicios transversales (config, logging) que querés en todos lados, marcás el módulo como `@Global()`. Usalo con moderación: rompe el encapsulamiento.

**Ejercicios 4**
4.1 ¿Qué dos cosas tienen que pasar para que `ServicioA` (de `ModuloA`) pueda inyectarse en `ServicioB` (de `ModuloB`)?
4.2 Escribí un `DatabaseModule` que provea y **exporte** un `DatabaseService`.
4.3 Escribí un `UsersModule` que importe `DatabaseModule` y tenga un `UsersService` que inyecte `DatabaseService`. ¿Necesita `UsersModule` listar `DatabaseService` en sus propios `providers`? Justificá.

---

## Módulo 5 — DTOs y validación con `class-validator`

**Teoría.** Un **DTO** (Data Transfer Object) es una clase que define la forma de los datos que entran o salen. En Nest se modelan como **clases** (no `interface`) porque los decoradores de validación necesitan existir en runtime. Con `class-validator` + `class-transformer` y un `ValidationPipe` global, Nest valida el body automáticamente.

```ts
import { IsString, IsInt, Min, IsOptional } from "class-validator";

export class CreateProductDto {
  @IsString()
  nombre!: string;

  @IsInt()
  @Min(0)
  precio!: number;

  @IsOptional()
  @IsString()
  descripcion?: string;
}
```

Para activarlo globalmente (en `main.ts`):

```ts
app.useGlobalPipes(
  new ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true }),
);
```

`whitelist` borra propiedades no declaradas; `forbidNonWhitelisted` directamente rechaza el request si sobran props; `transform: true` hace que `class-transformer` **instancie el DTO** y convierta tipos primitivos (p. ej. un `@Param` numérico llega como `number`, sin `ParseIntPipe` manual). En producción casi siempre se activan los tres juntos. Esto cierra el hueco del `any` que viste al hacer `JSON.parse` en el módulo de TypeScript: los datos externos quedan validados antes de tocar tu lógica.

**Ejercicios 5**
5.1 Escribí un `CreateUserDto` con `email` (string con formato email), `password` (string, mínimo 8 caracteres) y `edad` opcional (entero, mínimo 18). Pista: `@IsEmail`, `@MinLength`, `@IsOptional`, `@IsInt`, `@Min`.
5.2 ¿Por qué los DTOs se definen como `class` y no como `interface` o `type`? Respondé en una o dos frases.
5.3 Escribí la línea de `main.ts` que active la validación global descartando propiedades no declaradas en el DTO.

---

## Módulo 6 — Pipes: transformación y validación

**Teoría.** Un **pipe** transforma o valida un argumento antes de que llegue al handler. Nest trae varios built-in:

- `ParseIntPipe`, `ParseBoolPipe`, `ParseUUIDPipe` — convierten/validan params.
- `ValidationPipe` — valida DTOs (módulo 5).
- `DefaultValuePipe` — valor por defecto si falta.

```ts
@Get(":id")
findOne(@Param("id", ParseIntPipe) id: number): string {
  // si /products/abc → 400 automático; acá id ya es number
  return `producto ${id}`;
}
```

Podés escribir pipes propios implementando `PipeTransform`:

```ts
@Injectable()
export class TrimPipe implements PipeTransform<string, string> {
  transform(value: string): string {
    return value.trim();
  }
}
```

Los pipes resuelven el "stringly-typed" de los params: todo entra como string en HTTP, y el pipe lo convierte a su tipo real (y falla con `400` si no puede).

**Ejercicios 6**
6.1 Escribí un handler `GET /products/:id` donde `id` se reciba ya como `number` (y un id no numérico devuelva `400` automáticamente).
6.2 Escribí un handler `GET /products?activos=...` donde `activos` se reciba como `boolean`, con valor por defecto `true` si no viene. Pista: combiná `DefaultValuePipe` y `ParseBoolPipe`.
6.3 Escribí un pipe `UpperCasePipe` que transforme un string a mayúsculas implementando `PipeTransform`.

---

## Módulo 7 — Manejo de errores y excepciones

**Teoría.** Nest tiene una capa de excepciones HTTP. Lanzás una excepción y el framework la convierte en la response correcta:

```ts
import { NotFoundException, BadRequestException } from "@nestjs/common";

@Get(":id")
findOne(@Param("id", ParseIntPipe) id: number): Producto {
  const prod = this.service.find(id);
  if (!prod) throw new NotFoundException(`Producto ${id} no existe`);
  return prod;
}
```

Excepciones comunes: `BadRequestException` (400), `UnauthorizedException` (401), `ForbiddenException` (403), `NotFoundException` (404), `ConflictException` (409). Para casos a medida usás `throw new HttpException(mensaje, status)`.

Para centralizar el formato de errores escribís un **exception filter** con `@Catch`:

```ts
@Catch(HttpException)
export class HttpErrorFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost): void {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse();
    const status = exception.getStatus();
    res.status(status).json({ ok: false, status, mensaje: exception.message });
  }
}
```

La regla: el **service** lanza excepciones de dominio; la **capa HTTP** las traduce a status codes. No devuelvas `null` silencioso desde un endpoint cuando el caso correcto es un `404`.

**Ejercicios 7**
7.1 Escribí un método de service `obtener(id: number): Usuario` que lance `NotFoundException` si no encuentra el usuario (simulá la búsqueda con un array).
7.2 Escribí un `crear(email: string)` que lance `ConflictException` si el email ya existe en una lista dada.
7.3 ¿Por qué conviene lanzar `NotFoundException` desde el service en vez de retornar `undefined` y que el controlador decida? Respondé en una o dos frases.

---

## Módulo 8 — Guards: autenticación y autorización

**Teoría.** Un **guard** decide si un request puede continuar. Devuelve `true` (pasa) o `false`/excepción (se bloquea con `403`). Es el lugar canónico para **auth**: validar un token, chequear roles, etc. Implementan `CanActivate`:

```ts
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context.switchToHttp().getRequest<{ headers: Record<string, string> }>();
    const auth = req.headers["authorization"];
    return typeof auth === "string" && auth.startsWith("Bearer ");
  }
}
```

Se aplican con `@UseGuards(AuthGuard)` sobre un controlador o un método. Para roles se combinan con metadata: un decorador `@Roles("admin")` setea metadata que el guard lee con `Reflector`.

El guard corre **antes** que el handler y los pipes de validación de body, así que es la primera barrera real de seguridad.

**Ejercicios 8**
8.1 Escribí un `ApiKeyGuard` que implemente `CanActivate` y permita el request solo si el header `x-api-key` es igual a `"secreto"`.
8.2 Mostrá cómo aplicarías ese guard a todo un `AdminController`.
8.3 ¿Cuál es la diferencia de propósito entre un guard y un pipe? Respondé en una frase.

---

## Módulo 9 — Interceptors y middleware

**Teoría.** Ambos envuelven el request, pero en capas distintas:

- **Middleware**: corre antes que todo (incluso antes de los guards). Acceso crudo a `req`/`res`. Ideal para logging básico, CORS, parseos. Se registra en el módulo con `configure(consumer)`.
- **Interceptor**: envuelve la ejecución del handler con RxJS; puede transformar la **respuesta** o medir tiempos. Implementa `NestInterceptor`.

```ts
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<{ ok: boolean; data: unknown }> {
    return next.handle().pipe(map((data) => ({ ok: true, data })));
  }
}
```

Orden de ejecución de un request: **middleware → guards → interceptors (antes) → pipes → handler → interceptors (después) → exception filters**. Conocer este orden te ahorra horas de debugging. Un detalle clave: como los **guards corren antes que los pipes**, dentro de un guard el body **todavía no está validado ni transformado**.

**Ejercicios 9**
9.1 Escribí un `LoggingInterceptor` que mida cuánto tarda el handler y lo loguee al terminar. Pista: tomá un timestamp antes y usá `tap(() => ...)` de RxJS en el `pipe`.
9.2 Escribí un interceptor `EnvelopeInterceptor` que envuelva cualquier respuesta en `{ ok: true, data }`.
9.3 Ordená estas etapas según se ejecutan: pipes, handler, guards, middleware, interceptors (antes y después).

---

## Módulo 10 — Configuración y variables de entorno

**Teoría.** Nunca hardcodees secretos ni URLs. Nest tiene `@nestjs/config`: cargás un `.env` y leés valores tipados con `ConfigService`.

```ts
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
})
export class AppModule {}

@Injectable()
export class DbService {
  constructor(private readonly config: ConfigService) {
    const url = this.config.get<string>("DATABASE_URL");
  }
}
```

Buenas prácticas: validá el `.env` al arrancar (con un schema de Joi o Zod) para fallar rápido si falta una variable; nunca commitees el `.env` (sí un `.env.example`); tipá los `get<T>()`.

**Ejercicios 10**
10.1 ¿Por qué conviene validar las variables de entorno al iniciar la app en lugar de cuando se usan?
10.2 Escribí la importación de `ConfigModule` para que esté disponible globalmente sin reimportarlo en cada módulo.
10.3 Escribí un service que lea `PORT` (number) desde `ConfigService`, con `3000` como valor por defecto si no está definida.

---

## Módulo 11 — Persistencia: el patrón Repository

**Teoría.** El **patrón Repository** abstrae el acceso a datos detrás de una interfaz. Tu service depende de la interfaz (`ProductRepository`), no de la base concreta (TypeORM, Prisma, un array en memoria). Esto es exactamente la inversión de dependencias que viste en TypeScript, aplicada a la BD.

```ts
export interface ProductRepository {
  findById(id: number): Promise<Producto | null>;
  save(p: Producto): Promise<Producto>;
}

@Injectable()
export class ProductsService {
  constructor(
    @Inject("PRODUCT_REPO") private readonly repo: ProductRepository,
  ) {}

  obtener(id: number): Promise<Producto | null> {
    return this.repo.findById(id);
  }
}
```

En tests inyectás un repositorio en memoria; en producción, uno con Prisma/TypeORM. El service no se entera. Esta separación es lo que hace tu código testeable y portable entre bases de datos.

**Ejercicios 11**
11.1 Definí una interfaz `UserRepository` con `findById(id: number): Promise<Usuario | null>` y `save(u: Usuario): Promise<Usuario>`.
11.2 Escribí una implementación `InMemoryUserRepository` que cumpla esa interfaz usando un `Map` interno.
11.3 Escribí el provider que registre `InMemoryUserRepository` bajo el token `"USER_REPO"` (con `useClass`) en un módulo.

---

## Módulo 12 — Testing: unitario con mocks y e2e

**Teoría.** Nest está diseñado para testear. En un **test unitario** de un service, inyectás *mocks* de sus dependencias (gracias a que dependés de interfaces/tokens). En un **test e2e** levantás la app entera y le pegás con `supertest`.

```ts
// unitario: mockeamos el repositorio
describe("ProductsService", () => {
  it("devuelve null si no existe", async () => {
    const repoMock: ProductRepository = {
      findById: async () => null,
      save: async (p) => p,
    };
    const service = new ProductsService(repoMock);
    expect(await service.obtener(99)).toBeNull();
  });
});
```

Para tests que necesitan el contenedor de DI, usás `Test.createTestingModule({...}).compile()` y `module.get(...)`. La clave: como el service depende de una interfaz, el mock es trivial (un objeto que la cumple), sin tocar la base real.

Un test **e2e** levanta la app real y le pega con `supertest`:

```ts
import { Test } from "@nestjs/testing";
import { INestApplication } from "@nestjs/common";
import request from "supertest";
import { AppModule } from "../src/app.module";

describe("Products (e2e)", () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();
    app = moduleRef.createNestApplication();
    await app.init();
  });

  it("GET /products devuelve 200", () => {
    return request(app.getHttpServer()).get("/products").expect(200);
  });

  afterAll(async () => {
    await app.close();
  });
});
```

**Ejercicios 12**
12.1 Escribí un mock de `UserRepository` (del módulo 11) donde `findById` siempre devuelva un usuario fijo.
12.2 Escribí un test unitario de un `UsersService` que verifique que `obtener(1)` devuelve el usuario del mock.
12.3 ¿Por qué depender de la interfaz `UserRepository` (y no de la implementación concreta) hace que este test sea simple? Respondé en una frase.

---

# Soluciones

> Mirá esto solo después de intentarlo. Si tu solución difiere pero es correcta y type-safe, probablemente también esté bien — hay más de un camino.
>
> Los snippets de soluciones **omiten los imports** por brevedad (`@nestjs/common`, `class-validator`, `rxjs`, etc.) y asumen estos tipos de dominio, usados en varios módulos:
>
> ```ts
> interface Usuario { id: number; email: string; }
> interface Producto { id: number; nombre: string; precio: number; stock: number; }
> ```

### Módulo 1
```ts
// 1.1
// El controlador maneja la entrada/salida HTTP (rutas, params, status) y delega;
// el service contiene la lógica de negocio reutilizable e independiente del HTTP.

// 1.2
@Controller("products")
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Get()
  findAll(): string[] {
    return this.productsService.findAll();
  }
}

// 1.3
@Module({
  controllers: [ProductsController],
  providers: [ProductsService],
})
export class ProductsModule {}
```

### Módulo 2
```ts
// 2.1
@Get(":id")
findOne(@Param("id") id: string): { id: string } {
  return { id };
}

// 2.2
@Get()
findByCategoria(@Query("categoria") categoria?: string): string {
  return categoria ? `filtrando por ${categoria}` : "sin filtro";
}

// 2.3
@Delete(":id")
@HttpCode(204) // necesitás @HttpCode para forzar el 204 No Content
remove(@Param("id") id: string): void {
  // ... eliminar
}
```

### Módulo 3
```ts
// 3.1
@Injectable()
export class OrdersService {
  constructor(private readonly logger: Logger) {}
  crear(): void {
    this.logger.log("orden creada");
  }
}

// 3.2
interface MailerPort {
  enviar(to: string): Promise<void>;
}

@Injectable()
export class ConsoleMailer implements MailerPort {
  async enviar(to: string): Promise<void> {
    console.log(`mail a ${to}`);
  }
}

@Module({
  providers: [{ provide: "MAILER", useClass: ConsoleMailer }],
})
export class MailModule {}

// 3.3
@Injectable()
export class NotificationsService {
  constructor(@Inject("MAILER") private readonly mailer: MailerPort) {}
  async avisar(to: string): Promise<void> {
    await this.mailer.enviar(to);
  }
}
// Hace falta el token porque las interfaces de TS no existen en runtime: Nest no
// puede usarlas como identificador para inyectar, así que usás un string/Symbol.
```

### Módulo 4
```ts
// 4.1
// (1) ModuloA debe EXPORTAR ServicioA (exports: [ServicioA]).
// (2) ModuloB debe IMPORTAR ModuloA (imports: [ModuloA]).

// 4.2
@Module({
  providers: [DatabaseService],
  exports: [DatabaseService],
})
export class DatabaseModule {}

// 4.3
@Module({
  imports: [DatabaseModule],
  providers: [UsersService], // NO se lista DatabaseService acá
})
export class UsersModule {}
// No: DatabaseService ya está provisto y exportado por DatabaseModule. Al importar
// ese módulo, su export queda disponible para inyección. Listarlo de nuevo crearía
// una segunda instancia y rompería el singleton del módulo original.
```

### Módulo 5
```ts
// 5.1
import { IsEmail, MinLength, IsOptional, IsInt, Min } from "class-validator";

export class CreateUserDto {
  @IsEmail()
  email!: string;

  @MinLength(8)
  password!: string;

  @IsOptional()
  @IsInt()
  @Min(18)
  edad?: number;
}

// 5.2
// Porque los decoradores de class-validator (@IsEmail, etc.) necesitan existir en
// runtime para validar; las interfaces y los type se borran al compilar.

// 5.3
app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
```

### Módulo 6
```ts
// 6.1
@Get(":id")
findOne(@Param("id", ParseIntPipe) id: number): { id: number } {
  return { id };
}

// 6.2
@Get()
findActivos(
  @Query("activos", new DefaultValuePipe(true), ParseBoolPipe) activos: boolean,
): { activos: boolean } {
  return { activos };
}
// DefaultValuePipe solo aplica si `activos` viene ausente/undefined. Si llega vacío
//  (`?activos=`), ParseBoolPipe recibe "" y falla con 400.

// 6.3
@Injectable()
export class UpperCasePipe implements PipeTransform<string, string> {
  transform(value: string): string {
    return value.toUpperCase();
  }
}
```

### Módulo 7
```ts
// 7.1
@Injectable()
export class UsersService {
  private readonly usuarios: Usuario[] = [];
  obtener(id: number): Usuario {
    const u = this.usuarios.find((x) => x.id === id);
    if (!u) throw new NotFoundException(`Usuario ${id} no existe`);
    return u;
  }
}

// 7.2
crear(email: string): Usuario {
  const existe = this.usuarios.some((u) => u.email === email);
  if (existe) throw new ConflictException("El email ya está registrado");
  const nuevo: Usuario = { id: this.usuarios.length + 1, email };
  this.usuarios.push(nuevo);
  return nuevo;
}

// 7.3
// Centraliza la regla de dominio en un solo lugar y la hace consistente: cualquier
// controlador que use el service obtiene el 404 correcto sin reimplementar el chequeo,
// y Nest traduce la excepción al status HTTP automáticamente.
```

### Módulo 8
```ts
// 8.1
@Injectable()
export class ApiKeyGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const req = context
      .switchToHttp()
      .getRequest<{ headers: Record<string, string | undefined> }>();
    return req.headers["x-api-key"] === "secreto";
  }
}
// En producción la API key se lee de ConfigService/env (NUNCA hardcodeada, como dice
//  el módulo 10) y la comparación debería ser de tiempo constante
//  (crypto.timingSafeEqual) para no filtrar info por el tiempo de respuesta.

// 8.2
@UseGuards(ApiKeyGuard)
@Controller("admin")
export class AdminController {}

// 8.3
// El guard decide SI el request puede continuar (autorización); el pipe transforma
// o valida los datos de un argumento que ya pasó esa barrera.
```

### Módulo 9
```ts
// 9.1
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    const inicio = Date.now();
    return next.handle().pipe(
      tap(() => console.log(`tardó ${Date.now() - inicio}ms`)),
    );
  }
}

// 9.2
@Injectable()
export class EnvelopeInterceptor implements NestInterceptor {
  intercept(
    _ctx: ExecutionContext,
    next: CallHandler,
  ): Observable<{ ok: boolean; data: unknown }> {
    return next.handle().pipe(map((data) => ({ ok: true, data })));
  }
}

// 9.3
// Orden de ejecución: middleware → guards → interceptors (antes) → pipes →
//  handler → interceptors (después) → exception filters.
```

### Módulo 10
```ts
// 10.1
// Para fallar rápido ("fail fast"): si falta o está mal una variable, la app no
// arranca, en vez de explotar más tarde en runtime cuando un usuario use esa ruta.

// 10.2
@Module({
  imports: [ConfigModule.forRoot({ isGlobal: true })],
})
export class AppModule {}

// 10.3
@Injectable()
export class ServerConfig {
  constructor(private readonly config: ConfigService) {}
  get port(): number {
    return this.config.get<number>("PORT") ?? 3000;
  }
}
```

### Módulo 11
```ts
// 11.1
export interface UserRepository {
  findById(id: number): Promise<Usuario | null>;
  save(u: Usuario): Promise<Usuario>;
}

// 11.2
@Injectable()
export class InMemoryUserRepository implements UserRepository {
  private readonly store = new Map<number, Usuario>();

  async findById(id: number): Promise<Usuario | null> {
    return this.store.get(id) ?? null;
  }

  async save(u: Usuario): Promise<Usuario> {
    this.store.set(u.id, u);
    return u;
  }
}

// 11.3
@Module({
  providers: [{ provide: "USER_REPO", useClass: InMemoryUserRepository }],
  exports: ["USER_REPO"],
})
export class UsersModule {}
```

### Módulo 12
```ts
// 12.1
const usuarioFijo: Usuario = { id: 1, email: "ana@test.com" };
const repoMock: UserRepository = {
  findById: async () => usuarioFijo,
  save: async (u) => u,
};

// 12.2
describe("UsersService", () => {
  it("obtener(1) devuelve el usuario del mock", async () => {
    const service = new UsersService(repoMock);
    expect(await service.obtener(1)).toEqual(usuarioFijo);
  });
});
// (asumiendo un UsersService cuyo constructor recibe el UserRepository y cuyo
//  obtener(id) hace return this.repo.findById(id))

// 12.3
// Porque el mock es solo un objeto que cumple la interfaz: no hay que levantar una
// base de datos real ni el contenedor de DI para probar la lógica del service.
```

---

## Siguientes pasos

Con estos 12 módulos ya entendés la columna vertebral de NestJS: el flujo request → controlador → service, la DI con tokens, validación con DTOs, y la separación por capas (guards, pipes, interceptors, repositorios). El recorrido natural por el temario desde acá: **PostgreSQL + ORM** para reemplazar el repositorio en memoria por uno real (Prisma/TypeORM sobre la interfaz `Repository`), el módulo de **Autenticación y autorización** (JWT/Passport sobre el `AuthGuard` que viste acá), **Testing** para profundizar unit + e2e, y **Swagger** (en el módulo de *Tiempo real + Swagger*) para documentar la API. Más adelante, **Patrones de diseño en NestJS** y **NestJS nivel Senior** llevan todo esto a escala de producción.
