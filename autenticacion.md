# Autenticación y autorización: el backend que protege de verdad

**De la teoría del guard a un sistema de login real · JWT, refresh, RBAC · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. En el módulo de guards de NestJS viste un `AuthGuard` que chequeaba si existía un header `Authorization`. Eso era el esqueleto; acá lo llenamos con un sistema de autenticación completo: registro con contraseña hasheada, login que emite tokens, sesiones con **access + refresh tokens**, y autorización por **roles (RBAC)**. Es semana propia en el roadmap, un proyecto de portfolio ("Auth Service reutilizable") y una pregunta fija de entrevista. Y es, probablemente, **lo que más confunde a quien viene del frontend**: dónde viven los tokens, cómo se renuevan, por qué una cookie httpOnly es más segura que `localStorage`.

**Lo que asumimos.** Que ya tenés usuarios persistidos en PostgreSQL (módulo anterior) y que entendés guards, providers y DI de NestJS. El dominio sigue siendo el mismo: **usuarios** que se registran, inician sesión y acceden a sus **proyectos** y **tareas**.

> **¿Te falta alguna base?** Auth se apoya en dos cosas concretas. (1) **Guards y DI de NestJS**: los tokens se validan en un guard y los servicios de auth se inyectan; si eso te suena nuevo, repasá [NestJS](nestjs.md) (módulo 8, guards) antes de seguir. (2) **Usuarios en PostgreSQL**: hasheás contraseñas y guardás sesiones contra una tabla real, así que si no hiciste [PostgreSQL](postgresql.md), el registro y el login te van a costar el doble. Convertir el prerrequisito en rampa, no en muro.

**Para practicar.** Vas a necesitar:

```bash
npm i @nestjs/jwt @nestjs/passport passport passport-jwt bcrypt
npm i -D @types/passport-jwt @types/bcrypt
```

**Índice de módulos**
1. Autenticación vs. autorización
2. Nunca guardes contraseñas: hashing con bcrypt/argon2
3. JWT: qué es y cómo se firma
4. La estrategia de dos tokens: access + refresh
5. Dónde viven los tokens (cookies httpOnly vs. localStorage)
6. Registro y login en NestJS (`AuthService`)
7. Passport y estrategias (el patrón Strategy en acción)
8. Guards de autenticación: proteger rutas
9. Autorización con roles (RBAC): `@Roles` + `RolesGuard`
10. Refresh token: rotación y revocación
11. Endurecer: rate limiting, fugas de información y OWASP
12. OAuth 2.1 / "login con Google" (panorama)
13. Segundo factor: 2FA/MFA con TOTP
14. Recuperar contraseña (password reset) seguro

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Autenticación vs. autorización

**Teoría.** Son dos cosas distintas que se confunden todo el tiempo:

- **Autenticación (authN)**: *¿quién sos?* Verificar la identidad. El login es autenticación: comprobás que quien dice ser "ana@test.com" conoce su contraseña.
- **Autorización (authZ)**: *¿qué podés hacer?* Una vez que sé quién sos, decido si tenés permiso para esta acción. "Sos Ana, pero solo un admin puede borrar proyectos ajenos."

El orden es siempre authN → authZ: primero te identifico, después chequeo permisos. En NestJS ambos viven en **guards** (lo viste en `nestjs.md`): un guard de autenticación valida el token e identifica al usuario; un guard de autorización chequea sus roles/permisos. El guard corre **antes** que el handler, así que es la primera barrera real (igual que aprendiste con el orden del pipeline de request).

Un error conceptual común: devolver `401` y `403` indistintamente. **`401 Unauthorized`** = "no sé quién sos" (falta o es inválido el token → fallo de autenticación). **`403 Forbidden`** = "sé quién sos, pero no podés hacer esto" (fallo de autorización). Usarlos bien comunica la causa real al cliente.

**Ejercicios 1**
1.1 Clasificá cada acción como autenticación o autorización: (a) verificar la contraseña al loguearse; (b) chequear que el usuario tenga rol `admin` para borrar; (c) validar que el JWT del request no esté vencido.
1.2 Un usuario logueado intenta editar un proyecto que no es suyo. ¿Qué status devolvés, `401` o `403`? ¿Por qué?
1.3 ¿Por qué la autorización siempre va después de la autenticación y no al revés?

---

## Módulo 2 — Nunca guardes contraseñas: hashing con bcrypt/argon2

**Teoría.** Regla número uno, sin excepciones: **nunca guardes la contraseña en texto plano** (ni "encriptada" de forma reversible). Si tu base se filtra, todas las cuentas quedan expuestas. En vez de eso, guardás un **hash**: una transformación de una sola vía. Cuando el usuario se loguea, hasheás lo que escribió y lo comparás con el hash guardado; nunca recuperás la contraseña original (ni vos podés).

Pero no cualquier hash sirve. `md5`/`sha256` son **demasiado rápidos**: un atacante prueba miles de millones por segundo. Para contraseñas se usan algoritmos **lentos a propósito** y con **salt** (un valor aleatorio por usuario que evita que dos contraseñas iguales den el mismo hash y frena los ataques con tablas precalculadas):

- **bcrypt**: el clásico, probado, suficiente para la mayoría. Tiene un "cost factor" (cuánto trabajo cuesta).
- **argon2 (variante `argon2id`)**: el ganador moderno (resistente a ataques con GPU/ASIC, que es donde bcrypt flaquea). Si podés elegir, es la recomendación 2026. Usá la variante **`argon2id`** y los defaults seguros de la librería (del orden de ~19 MiB de memoria, 2 iteraciones, paralelismo 1) ajustando según tu hardware.

```ts
import * as bcrypt from "bcrypt";

// al registrar: hasheás (bcrypt genera y guarda el salt dentro del propio hash)
const passwordHash = await bcrypt.hash(passwordPlano, 12); // 12 = cost factor

// al loguear: comparás sin "deshashear" nada
const coincide = await bcrypt.compare(passwordPlano, passwordHash);
```

El cost factor (12 es un buen default en 2026) hace que verificar tarde ~décimas de segundo: imperceptible para un login real, carísimo para un atacante que prueba millones. En la base guardás **solo** `passwordHash`, nunca la contraseña.

**Ejercicios 2**
2.1 ¿Por qué `sha256` es una mala elección para hashear contraseñas, si igual es un hash de una vía?
2.2 ¿Qué es el "salt" y qué ataque previene? ¿Tenés que guardarlo en una columna aparte si usás bcrypt?
2.3 Escribí las dos líneas: una que hashee una contraseña al registrar (cost 12) y otra que verifique un intento de login contra el hash guardado.

---

## Módulo 3 — JWT: qué es y cómo se firma

**Teoría.** Un **JWT** (JSON Web Token) es un string que representa "credenciales verificables". El servidor lo emite al loguearte, el cliente lo manda en cada request, y el servidor lo valida **sin consultar la base** (es *stateless*). Tiene tres partes separadas por puntos: `header.payload.signature`.

- **Header**: el algoritmo de firma (ej. `HS256`).
- **Payload (claims)**: los datos, ej. `{ "sub": 42, "email": "ana@test.com", "roles": ["user"], "exp": 1718...}`. `sub` es el ID del usuario; `exp` es el vencimiento.
- **Signature**: la firma criptográfica del header+payload con un **secreto** que solo el servidor conoce.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOjQyLCJlbWFpbCI6ImFuYUB0ZXN0LmNvbSJ9.K3xR...
└────── header ──────┘└─────────── payload ───────────┘└── signature ──┘
```

La clave para entenderlo: el payload **no está encriptado, solo codificado en base64** — cualquiera puede leerlo. Lo que el JWT garantiza no es secreto, es **integridad**: si alguien modifica el payload (se pone `roles: ["admin"]`), la firma deja de coincidir y el servidor lo rechaza, porque no conoce el secreto para re-firmarlo. Por eso:

- **Nunca pongas datos sensibles** en el payload (ni contraseñas, ni datos privados): viaja a la vista.
- **El secreto de firma es sagrado**: vive en variables de entorno, nunca en el código ni en git.

En NestJS se firma y verifica con `@nestjs/jwt`:

```ts
const token = await this.jwt.signAsync(
  { sub: usuario.id, email: usuario.email, roles: usuario.roles },
  { secret: process.env.JWT_SECRET, expiresIn: "15m" },
);
```

**HS256 vs. RS256.** `HS256` firma con un **secreto compartido** (el mismo que firma, verifica): simple, ideal cuando un único servicio emite y valida. `RS256`/`ES256` usan un **par de claves**: el servidor firma con la **privada** y cualquiera valida con la **pública** — necesario cuando varios servicios (o un cliente) deben **verificar** tokens sin poder **emitirlos**.

**El ataque que tenés que conocer (`alg:none` y confusión de algoritmo).** El `alg` viaja en el **header del token**, que el atacante controla. Dos ataques clásicos: poner `alg: "none"` (token sin firma, esperando que el server lo acepte) o cambiar `RS256`→`HS256` para que el verificador use la **clave pública** (que es conocida) como si fuera el secreto HMAC y así firmar tokens válidos. La defensa es una sola regla: **el servidor fija el algoritmo esperado y nunca confía en el `alg` del token**. En `passport-jwt`/`@nestjs/jwt` eso se hace pasando explícitamente `algorithms: ["HS256"]` al verificar (lo ves en el módulo 7).

**Ejercicios 3**
3.1 ¿El payload de un JWT está encriptado? ¿Qué garantiza entonces la firma?
3.2 Un usuario edita su propio JWT y se cambia `roles` a `["admin"]`. ¿Funciona? Explicá qué pasa cuando el servidor lo recibe.
3.3 Nombrá dos cosas que **nunca** deberías poner en el payload de un JWT y por qué.

---

## Módulo 4 — La estrategia de dos tokens: access + refresh

**Teoría.** Hay una tensión: querés tokens **de vida corta** (si se roban, sirven poco tiempo) pero **no querés que el usuario tenga que loguearse cada 15 minutos**. La solución estándar son **dos tokens**:

- **Access token**: JWT de vida corta (~15 min). Se manda en **cada request** para acceder a recursos protegidos. Si se filtra, expira rápido.
- **Refresh token**: de vida larga (~7-30 días). **No** se usa para acceder a recursos: su único trabajo es pedir un **nuevo access token** cuando el anterior vence. Se guarda más protegido y se usa poco.

El flujo:

```
1. Login        → servidor devuelve { accessToken (15m), refreshToken (7d) }
2. Requests     → cliente manda accessToken en cada uno
3. Access vence → cliente llama a POST /auth/refresh con el refreshToken
4. Refresh OK   → servidor devuelve un accessToken nuevo (y rota el refresh, módulo 10)
5. Logout       → servidor invalida el refreshToken
```

¿Por qué dos y no uno solo de vida larga? Porque un access token de larga duración que se filtra es una catástrofe (sirve semanas). Y uno solo de vida corta obligaría a re-loguearse todo el tiempo. La separación te da **seguridad** (el token que viaja en cada request expira enseguida) **y** **comodidad** (la sesión dura días). El refresh, además, es **revocable**: como lo consultás contra la base/Redis al renovar, podés invalidarlo (logout, robo detectado) — algo que un access token JWT puro, al ser stateless, no permite hasta que expira.

**Ejercicios 4**
4.1 ¿Cuál de los dos tokens se manda en cada request a un endpoint protegido, y cuál solo al renovar?
4.2 ¿Por qué no alcanza con un único access token de vida larga? Mencioná el riesgo concreto.
4.3 Si detectás que la cuenta de un usuario fue comprometida, ¿por qué podés "cortarle la sesión" vía el refresh token pero no instantáneamente vía un access token JWT stateless?

---

## Módulo 5 — Dónde viven los tokens (cookies httpOnly vs. localStorage)

**Teoría.** Esta es la pregunta que todo frontend se hace y casi nadie responde bien. ¿Dónde guarda el cliente los tokens? Tres opciones y sus riesgos:

- **`localStorage`**: simple, pero **accesible desde JavaScript** → si tu sitio sufre un **XSS** (código malicioso inyectado), el atacante lee el token y se hace pasar por el usuario. Cómodo pero vulnerable.
- **Cookie `httpOnly`**: el navegador la guarda y la manda sola, pero **JavaScript no puede leerla** (`httpOnly`) → un XSS no puede robar el token. A cambio, las cookies se envían automáticamente, lo que abre la puerta a **CSRF** (un sitio malicioso dispara una request a tu API con la cookie del usuario). Se mitiga con `SameSite`.
- **En memoria** (una variable JS): el access token vive solo mientras la pestaña está abierta; nada en disco. El más seguro contra robo persistente, pero se pierde al recargar.

El patrón profesional más usado en 2026 combina lo mejor de cada uno:

- **Access token en memoria** (variable de JS): corto, no persiste, no expuesto a robo de disco.
- **Refresh token en cookie `httpOnly` + `Secure` + `SameSite=Strict`** (o `Lax`): JS no lo puede leer (a salvo de XSS), solo viaja al endpoint de refresh, y `SameSite` lo protege de CSRF.

```ts
// el refresh token se setea como cookie httpOnly desde el backend:
res.cookie("refreshToken", refreshToken, {
  httpOnly: true,   // JS del navegador no lo puede leer → protege de XSS
  secure: true,     // solo se envía por HTTPS
  sameSite: "strict", // no se manda en requests cross-site → protege de CSRF
  maxAge: 7 * 24 * 60 * 60 * 1000, // 7 días
  path: "/auth/refresh", // solo viaja al endpoint que lo necesita
});
```

La conclusión que importa para vos como frontend: **`localStorage` para tokens es la opción cómoda y la menos segura**; si te preguntan en una entrevista "¿dónde guardás el JWT?", la respuesta que demuestra criterio es "el refresh en una cookie httpOnly/SameSite y el access en memoria, no en localStorage, por el riesgo de XSS".

**Matices de `SameSite` y CSRF (clave en entrevista).** Si tu access token viaja en el header `Authorization: Bearer` (no en cookie), tu API es **inmune a CSRF** en esos endpoints: CSRF explota el envío **automático** de cookies por el navegador, y un header lo setea tu propio JS a propósito. El riesgo de CSRF aparece cuando autenticás **por cookie**. Sobre `SameSite`: `Strict` es fuerte pero puede romper flujos legítimos (llegar desde un link externo, login con terceros); `Lax` suele ser el default razonable. Y si tu SPA y tu API están en **dominios distintos**, la cookie tiene que ser `SameSite=None; Secure` (cross-site) y ahí **sí** necesitás una defensa CSRF explícita: token de **doble envío** (double-submit) o verificación del header `Origin` (más en el módulo 11).

**Ejercicios 5**
5.1 ¿Por qué guardar el token en `localStorage` es riesgoso? ¿Qué tipo de ataque lo explota?
5.2 ¿Qué propiedad de una cookie evita que JavaScript la lea, y de qué ataque protege eso?
5.3 ¿Para qué sirve `SameSite=Strict` en la cookie del refresh token? ¿Qué ataque mitiga?

---

## Módulo 6 — Registro y login en NestJS (`AuthService`)

**Teoría.** Con las piezas anteriores armamos el `AuthService`. El **registro** hashea la contraseña y crea el usuario; el **login** verifica credenciales y emite tokens. Ambos delegan en el repositorio de usuarios (el `UserRepository` que conectaste a Postgres en el módulo anterior).

```ts
@Injectable()
export class AuthService {
  constructor(
    @Inject("USER_REPO") private readonly usuarios: UserRepository,
    private readonly jwt: JwtService,
    private readonly config: ConfigService,
  ) {}

  async register(dto: RegisterDto): Promise<Tokens> {
    const existente = await this.usuarios.findByEmail(dto.email);
    if (existente) throw new ConflictException("El email ya está registrado");

    const passwordHash = await bcrypt.hash(dto.password, 12);
    const usuario = await this.usuarios.save({
      email: dto.email,
      passwordHash,
      roles: ["user"],
    });
    return this.emitirTokens(usuario);
  }

  // hash "dummy" precalculado (un bcrypt de cualquier valor) para igualar tiempos
  private static readonly DUMMY_HASH = "$2b$12$abcdefghijklmnopqrstuv...";

  async login(dto: LoginDto): Promise<Tokens> {
    const usuario = await this.usuarios.findByEmail(dto.email);
    // Comparamos SIEMPRE contra un hash (real o dummy) para que el tiempo de respuesta
    // sea igual exista o no el email: si no, un timing attack revela qué emails existen.
    const hash = usuario?.passwordHash ?? AuthService.DUMMY_HASH;
    const ok = await bcrypt.compare(dto.password, hash);
    if (!usuario || !ok) throw new UnauthorizedException("Credenciales inválidas");

    return this.emitirTokens(usuario);
  }

  // familyId agrupa la "familia" de tokens de una sesión (módulo 10)
  private async emitirTokens(usuario: Usuario, familyId = randomUUID()): Promise<Tokens> {
    // El ACCESS lleva los datos de autorización (roles); el REFRESH es mínimo:
    // solo identifica usuario y sesión. Los roles NO viajan en el refresh (módulo 10).
    const accessToken = await this.jwt.signAsync(
      { sub: usuario.id, email: usuario.email, roles: usuario.roles, type: "access" },
      { secret: this.config.get<string>("JWT_ACCESS_SECRET"), expiresIn: "15m" },
    );
    const refreshToken = await this.jwt.signAsync(
      { sub: usuario.id, type: "refresh", familyId, jti: randomUUID() },
      { secret: this.config.get<string>("JWT_REFRESH_SECRET"), expiresIn: "7d" },
    );
    // guardamos el HASH del refresh (no el token en claro) bajo su familia
    await this.refreshStore.guardar(familyId, sha256(refreshToken));
    return { accessToken, refreshToken };
  }
}
```

Detalles que separan a un junior de un mid acá: el login devuelve **el mismo mensaje** falle por email inexistente o por contraseña incorrecta, y además **compara siempre contra un hash (real o dummy)** para no filtrar por **tiempo de respuesta** qué emails existen (timing attack, módulo 11); el `passwordHash` **jamás** se devuelve en la respuesta (DTO de salida del módulo de patrones); el secreto del access y el del refresh son **distintos**; y el access y el refresh llevan un claim `type` y **payloads distintos** — el refresh **no** lleva `roles`, para que al renovar los permisos se relean de la base y no queden "congelados" los viejos (módulo 10).

**Ejercicios 6**
6.1 ¿Por qué `login` devuelve "Credenciales inválidas" tanto si el email no existe como si la contraseña está mal, en vez de mensajes distintos?
6.2 Escribí el `RegisterDto` con `email` (formato email) y `password` (mínimo 12 caracteres), reusando lo que sabés de `class-validator`.
6.3 El `AuthService` inyecta el `UserRepository` por un token. ¿Qué ventaja te da eso para testear el login sin una base real? (Conectá con el patrón Repository.)

---

## Módulo 7 — Passport y estrategias (el patrón Strategy en acción)

**Teoría.** **Passport** es la librería estándar de autenticación en Node, y NestJS la integra con `@nestjs/passport`. Su idea central es la **estrategia**: cada forma de autenticar (validar un JWT, validar usuario+contraseña, login con Google) es una *estrategia* intercambiable con la misma interfaz. Esto es, literalmente, el **patrón Strategy** que viste en el módulo de patrones: "cómo autenticar" es un algoritmo que se cambia sin tocar el resto.

La estrategia JWT define **de dónde sacar el token** y **cómo validarlo**. Su método `validate` recibe el payload ya verificado (firma y expiración OK) y lo que retorna queda disponible como `request.user`:

```ts
import { PassportStrategy } from "@nestjs/passport";
import { ExtractJwt, Strategy } from "passport-jwt";

interface JwtPayload {
  sub: number;
  email: string;
  roles: string[];
  type: "access" | "refresh";
}

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(config: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(), // lee "Authorization: Bearer <token>"
      ignoreExpiration: false, // rechaza tokens vencidos
      secretOrKey: config.getOrThrow<string>("JWT_ACCESS_SECRET"),
      algorithms: ["HS256"], // FIJAR el algoritmo: nunca confiar en el `alg` del token (módulo 3)
    });
  }

  // solo se llama si la firma, la expiración y el algoritmo son válidos
  async validate(payload: JwtPayload): Promise<JwtPayload> {
    // un refresh token NO debe servir como access token
    if (payload.type !== "access") throw new UnauthorizedException("Token inválido");
    return { sub: payload.sub, email: payload.email, roles: payload.roles, type: "access" };
  }
}
```

Passport hace el trabajo pesado (extraer el token, verificar firma y expiración con el secreto); vos solo definís qué hacer con el payload validado. Registrás la estrategia como provider en el `AuthModule`.

**Ejercicios 7**
7.1 ¿Por qué se dice que Passport + estrategias es un ejemplo del patrón Strategy? ¿Qué es lo intercambiable?
7.2 ¿Qué garantiza Passport antes de llamar a tu método `validate` de la estrategia JWT? (Pensá en firma y expiración.)
7.3 ¿Dónde queda disponible lo que devuelve `validate`, y cómo lo usarías después en un controlador?

---

## Módulo 8 — Guards de autenticación: proteger rutas

**Teoría.** Con la estrategia lista, proteger una ruta es trivial: un guard que dispara la estrategia JWT. `@nestjs/passport` te da uno hecho, `AuthGuard("jwt")`:

```ts
import { AuthGuard } from "@nestjs/passport";

@Controller("proyectos")
export class ProyectosController {
  @UseGuards(AuthGuard("jwt")) // exige un access token válido
  @Get()
  misProyectos(@Req() req: { user: JwtPayload }) {
    // req.user viene de lo que devolvió validate() en la estrategia
    return this.proyectosService.findByUsuario(req.user.sub);
  }
}
```

Si no hay token, o está vencido, o la firma no coincide, el guard responde `401` automáticamente y el handler **ni se ejecuta**. Si todo está bien, `request.user` queda poblado con el payload — ese es el puente entre "quién sos" (autenticación) y la lógica del endpoint.

Patrón habitual: en vez de escribir `@Req()` y tipar a mano, se crea un **decorador de parámetro** `@CurrentUser()` que extrae `request.user` limpio. Y para no repetir `@UseGuards` en cada método, podés aplicar el guard a todo el controlador, o globalmente y marcar las rutas públicas con un decorador `@Public()`.

```ts
// decorador de parámetro reutilizable
export const CurrentUser = createParamDecorator(
  (_data: unknown, ctx: ExecutionContext): JwtPayload => {
    return ctx.switchToHttp().getRequest<{ user: JwtPayload }>().user;
  },
);

// uso limpio:
@Get()
misProyectos(@CurrentUser() user: JwtPayload) {
  return this.proyectosService.findByUsuario(user.sub);
}
```

**Ejercicios 8**
8.1 ¿Qué pasa con el handler si el access token está vencido y la ruta tiene `@UseGuards(AuthGuard("jwt"))`?
8.2 ¿De dónde sale `request.user` y qué lo pobló? Trazá el camino desde el header hasta el controlador.
8.3 Escribí el uso de un decorador `@CurrentUser()` en un handler `GET /perfil` que devuelva el email del usuario logueado.

---

## Módulo 9 — Autorización con roles (RBAC): `@Roles` + `RolesGuard`

**Teoría.** Ya sabemos *quién* es el usuario; ahora *qué puede hacer*. **RBAC** (Role-Based Access Control) asigna **roles** (`user`, `admin`) y protege acciones según el rol. Es simple y alcanza para la mayoría de las apps (el archivo senior lo contrasta con ABAC, más granular, para dominios complejos).

Se arma con tres piezas, combinando metadata + guard + `Reflector` (exactamente como se anticipó en el módulo de guards):

```ts
// 1. decorador que marca qué roles requiere una ruta (setea metadata)
export const ROLES_KEY = "roles";
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// 2. guard que lee esa metadata y la compara con los roles del usuario
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requeridos = this.reflector.getAllAndOverride<string[] | undefined>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );
    if (!requeridos || requeridos.length === 0) return true; // ruta sin restricción de rol

    const { user } = context.switchToHttp().getRequest<{ user: JwtPayload }>();
    return requeridos.some((rol) => user.roles.includes(rol));
  }
}

// 3. uso: combinás el guard de auth (quién sos) con el de roles (qué podés)
@UseGuards(AuthGuard("jwt"), RolesGuard)
@Roles("admin")
@Delete(":id")
borrar(@Param("id", ParseIntPipe) id: number) {
  return this.proyectosService.borrar(id);
}
```

El orden importa: `AuthGuard("jwt")` corre primero y pobla `request.user`; `RolesGuard` después lee ese `user.roles`. Sin autenticar primero, no hay roles que chequear.

**RBAC vs. dueño del recurso.** RBAC responde "¿es admin?", pero muchas reglas son "¿es **el dueño** de *este* proyecto?". Eso ya no es solo rol: necesitás chequear contra el dato (`proyecto.usuarioId === user.sub`). Es la frontera hacia **ABAC** (control por atributos) que menciona el archivo senior; para empezar, RBAC + un chequeo de propiedad en el service cubre casi todo.

**Ejercicios 9**
9.1 ¿Por qué `RolesGuard` tiene que correr **después** de `AuthGuard("jwt")` y no antes?
9.2 Escribí un endpoint `DELETE /usuarios/:id` que solo pueda ejecutar un `admin`, usando `@Roles` y los dos guards.
9.3 "Un usuario solo puede editar **sus propios** proyectos." ¿RBAC alcanza para expresar esa regla? Si no, ¿dónde y cómo la chequearías?

---

## Módulo 10 — Refresh token: rotación y revocación

**Teoría.** El endpoint de refresh toma un refresh token válido y devuelve un access token nuevo. Pero hacerlo bien implica dos prácticas de seguridad:

**Rotación.** Cada vez que se usa un refresh token, emitís uno **nuevo** e invalidás el anterior. Así, un refresh token solo sirve una vez. Si un atacante robó uno y lo usa, cuando el usuario legítimo intente usar **el mismo** (ya rotado), detectás el **reuso** → señal de robo → revocás toda la familia de tokens de ese usuario y lo forzás a re-loguearse.

**Revocación.** Para poder invalidar un refresh token (logout, robo) tenés que tener **estado**: guardás el refresh token (o su hash) en la base o en **Redis**. Al renovar, verificás que siga siendo válido y no esté revocado. Esto rompe el "stateless puro" del JWT, pero **a propósito**: el refresh es justamente el token que querés poder cortar.

**Familia de tokens, hash y roles frescos.** Tres prácticas que hacen la diferencia:

- **Familia/`jti`**: cada login abre una **familia** (`familyId`) y cada refresh lleva un `jti` (id único). La revocación opera sobre la **familia**: detectar un reuso corta toda la sesión, no un token suelto.
- **Guardá el hash**, no el token en claro (mismo criterio que las contraseñas).
- **Releé el usuario de la base al rotar**: los `roles` del nuevo access salen de la **DB**, nunca de los claims del refresh viejo. Si no, a un usuario al que le revocaste `admin` le seguirías emitiendo tokens con `admin` hasta que expire el refresh — una **escalada de privilegios** silenciosa. Por eso el refresh no lleva `roles` (módulo 6).

```ts
async refresh(refreshToken: string): Promise<Tokens> {
  // 1. verificar firma, expiración y algoritmo; exigir que sea un token de tipo refresh
  const payload = await this.jwt.verifyAsync<RefreshPayload>(refreshToken, {
    secret: this.config.getOrThrow("JWT_REFRESH_SECRET"),
    algorithms: ["HS256"], // fijar el algoritmo (módulo 3)
  });
  if (payload.type !== "refresh") throw new UnauthorizedException("Token inválido");

  // 2. ¿sigue vigente en el store? Comparamos el HASH, no el token en claro
  const hash = sha256(refreshToken);
  const vigente = await this.refreshStore.esValido(payload.familyId, hash);
  if (!vigente) {
    // ya fue usado/rotado → posible reuso por robo: cortar TODA la familia
    await this.refreshStore.revocarFamilia(payload.familyId);
    throw new UnauthorizedException("Refresh token inválido");
  }

  // 3. rotar: invalidar el viejo dentro de la familia
  await this.refreshStore.invalidar(payload.familyId, hash);

  // 4. RELEER el usuario de la base: los roles salen de la DB, no del token viejo
  const usuario = await this.usuarios.findById(payload.sub);
  if (!usuario) throw new UnauthorizedException("Usuario inexistente");
  return this.emitirTokens(usuario, payload.familyId); // par nuevo en la misma familia
}
```

El **logout** es entonces simple: revocás la familia del refresh token en el store (y el cliente descarta el access, que de todos modos expira en minutos).

**Ejercicios 10**
10.1 ¿Qué es la rotación de refresh tokens y qué ataque ayuda a detectar?
10.2 ¿Por qué para revocar un refresh token necesitás guardar estado (base/Redis), rompiendo el "stateless" del JWT? ¿Por qué está bien hacerlo solo con el refresh y no con el access?
10.3 ¿Cómo implementarías el logout con este esquema? ¿Qué pasa con el access token que el cliente todavía tiene?

---

## Módulo 11 — Endurecer: rate limiting, fugas de información y OWASP

**Teoría.** Un sistema de auth funcional no es lo mismo que uno seguro. Lo que un Senior endurece (conecta con la sección de seguridad del archivo senior y el OWASP Top 10):

- **Rate limiting**: limitá los intentos de login por IP/usuario para frenar **fuerza bruta** (probar miles de contraseñas). En Nest, `ThrottlerModule`. Sin esto, tu login es un blanco fácil.
- **Bloqueo tras N intentos**: complementá el rate limiting con un **lockout temporal** de la cuenta (o un captcha) ante muchos fallos seguidos — frena el credential stuffing dirigido a una cuenta concreta.
- **No filtrar qué emails existen**: el mensaje de login fallido es el mismo exista o no el usuario, y la respuesta tarda lo mismo (timing, módulo 6). Lo mismo en "recuperar contraseña": respondé siempre "si el email existe, te enviamos un link", nunca "ese email no está registrado" — eso es un **enumeration attack**.
- **Validar toda entrada**: DTOs con `class-validator`, y para datos externos en runtime, Zod. La protección contra **SQL injection** viene de **parametrizar** las queries (lo que el ORM hace por defecto), no del ORM "por arte de magia": una *raw query* con input interpolado a mano sigue siendo vulnerable.
- **Defensa CSRF** (solo si autenticás por **cookie**): además de `SameSite`, usá un **token CSRF de doble envío** o verificá el header `Origin`. Si tu access token va por header `Authorization: Bearer`, CSRF no aplica (módulo 5).
- **Headers de seguridad**: `helmet` en NestJS (HSTS, `X-Content-Type-Options`, etc.) y una **CSP** (Content-Security-Policy) que limita el daño de un XSS.
- **Secretos fuera del código**: `JWT_*_SECRET` en variables de entorno, nunca en git, y rotables. Validalos al arrancar (el `fail fast` del módulo de config).
- **HTTPS siempre** en producción: un token que viaja en HTTP plano se puede interceptar. La cookie con `Secure` ni siquiera se manda sin HTTPS.
- **Tiempos de expiración cortos** en el access token: limita la ventana de daño si se filtra.

El principio que ordena todo (del archivo senior): **mínimo privilegio** (cada quien solo lo que necesita) y **defensa en profundidad** (varias capas, porque cualquiera puede fallar). Ninguna de estas medidas sola alcanza; juntas, suben mucho el costo de atacarte.

**Ejercicios 11**
11.1 ¿Qué ataque frena el rate limiting en el endpoint de login? ¿Por qué un login sin límite de intentos es peligroso?
11.2 ¿Por qué "recuperar contraseña" debe responder lo mismo exista o no el email? ¿Cómo se llama el ataque que esto previene?
11.3 Mencioná dos razones por las que los secretos de firma JWT van en variables de entorno y no en el código.

---

## Módulo 12 — OAuth 2.1 / "login con Google" (panorama)

**Teoría.** Esto es para ubicar el concepto, no para implementarlo ahora. **OAuth 2.1** es el estándar para **delegar la autenticación** a un tercero ("login con Google/GitHub"): en vez de manejar vos la contraseña, el usuario se autentica en Google y Google te confirma quién es. **OpenID Connect (OIDC)** es la capa sobre OAuth que estandariza esa identidad.

El flujo (Authorization Code) en una frase: tu app redirige al usuario a Google → el usuario se loguea y autoriza → Google redirige de vuelta a tu app con un **código** → tu backend cambia ese código por los datos del usuario → creás (o encontrás) el usuario en tu base y le emitís **tus propios** tokens (los del módulo 4). Es decir: OAuth resuelve el "¿quién sos?" inicial, pero **tu** sistema de sesiones sigue siendo el de access/refresh que ya armaste.

**Lo que en 2026 no puede faltar (OAuth 2.1).** Tres piezas que separan una respuesta actualizada de una de 2018:

- **PKCE** (`code_verifier`/`code_challenge`): hace que un código interceptado **no sirva** sin el verificador. Es **obligatorio** en OAuth 2.1, no opcional.
- **`state`**: un valor aleatorio que mandás y verificás al volver, para proteger el callback de **CSRF**.
- **Validar el `id_token`** (OIDC): antes de confiar en la identidad, validá su **firma** contra las **JWKS** del proveedor y los claims `iss`, `aud`, `exp` y `nonce`. No alcanza con "Google me lo devolvió".

El viejo *implicit flow* (token en el redirect) está **obsoleto**; hoy siempre es Authorization Code + PKCE.

Por qué importa como concepto: a los usuarios les ahorra otra contraseña, y a vos te saca de encima el manejo de credenciales (no guardás contraseñas si todos entran por Google). En Nest se implementa con una **estrategia de Passport** (`passport-google-oauth20`) — el mismo patrón Strategy del módulo 7, otra estrategia intercambiable.

Cuándo sumarlo: cuando tu producto lo pida (usuarios que ya tienen Google y no quieren otra cuenta). No es obligatorio para un MVP, pero saber **cómo encaja** (delega authN, vos seguís emitiendo tus tokens) es lo que demuestra criterio en una entrevista.

**Ejercicios 12**
12.1 En "login con Google", ¿quién verifica la contraseña del usuario, tu backend o Google? ¿Qué recibís vos al final del flujo?
12.2 Después de que Google confirma la identidad, ¿qué tokens usa tu app para las siguientes requests: los de Google o los tuyos? ¿Por qué?
12.3 ¿Qué patrón de diseño (ya visto) se repite al implementar "login con Google" como otra estrategia de Passport?

---

## Módulo 13 — Segundo factor: 2FA/MFA con TOTP

**Teoría.** La **autenticación multifactor (MFA)** suma, además de "algo que sabés" (la contraseña), "algo que **tenés**" (tu teléfono). La forma más común sin SMS (los SMS son interceptables) es **TOTP** (Time-based One-Time Password, RFC 6238): apps como Google Authenticator o Authy generan un código de 6 dígitos que cambia cada 30 segundos.

¿Cómo funciona? En el **enrolamiento**, el servidor genera un **secreto** único por usuario y se lo muestra como **QR** (una URL `otpauth://...`) para que lo escanee con su app. Desde ahí, tanto el server como la app derivan el mismo código de 6 dígitos a partir de ese secreto + la hora actual. En el **login**, tras validar la contraseña, se pide el código y se verifica contra el secreto.

```ts
import { authenticator } from "otplib";

// 1. enrolamiento: generás el secreto, lo guardás CIFRADO y mostrás el QR
const secret = authenticator.generateSecret();
const otpauth = authenticator.keyuri(usuario.email, "MiApp", secret);
// → convertís `otpauth` en un QR para que el usuario lo escanee con su app

// 2. en el login (segundo paso): verificás el código de 6 dígitos
const ok = authenticator.verify({ token: codigoIngresado, secret });
if (!ok) throw new UnauthorizedException("Código 2FA inválido");
```

Detalles que importan: guardá el **secreto cifrado** en la base (no en claro, y **nunca** en el JWT — viaja a la vista). Generá **backup codes** de un solo uso (hasheados, como contraseñas) para que el usuario no quede afuera si pierde el teléfono. Y recién marcá el 2FA como activo cuando el usuario confirma un primer código válido (así sabés que escaneó bien el QR).

**Ejercicios 13**
13.1 ¿Qué dos factores combina 2FA y por qué un código TOTP cuenta como "algo que tenés"?
13.2 ¿Para qué sirven los backup codes y qué propiedad de seguridad deben tener (cómo se guardan)?
13.3 ¿Dónde guardás el secreto TOTP del usuario? ¿Estaría bien meterlo en el JWT? Justificá.

---

## Módulo 14 — Recuperar contraseña (password reset) seguro

**Teoría.** Casi toda app tiene "olvidé mi contraseña", y hacerlo mal es una vulnerabilidad clásica. El flujo seguro tiene dos endpoints:

**`POST /auth/forgot`** — recibe un email y responde **siempre lo mismo** ("si el email existe, te enviamos un link"), exista o no (anti-enumeration, módulo 11). Si el usuario existe, generás un token **aleatorio criptográficamente seguro**, guardás su **hash** con un `expiresAt` corto (15-60 min) y lo enviás por email (el token en claro viaja **solo** en el email).

**`POST /auth/reset`** — recibe el token + la nueva contraseña, busca por el **hash** del token, valida que no esté **vencido ni usado**, cambia el `passwordHash`, marca el token como **usado** (un solo uso) y **revoca las sesiones existentes** (las familias de refresh) para cerrar cualquier sesión del atacante.

```ts
import { randomBytes } from "node:crypto";

async forgotPassword(email: string): Promise<void> {
  const usuario = await this.usuarios.findByEmail(email);
  if (!usuario) return; // el controlador responde 200 igual: respuesta neutra
  const token = randomBytes(32).toString("hex");            // token en claro → va al email
  await this.resetStore.guardar(usuario.id, sha256(token), enMinutos(60)); // guardamos el HASH
  await this.mailer.enviarReset(email, token);
}

async resetPassword(token: string, nuevaPassword: string): Promise<void> {
  const registro = await this.resetStore.buscarPorHash(sha256(token));
  if (!registro || registro.usado || registro.expiraEl < new Date())
    throw new BadRequestException("Token inválido o expirado");
  const passwordHash = await bcrypt.hash(nuevaPassword, 12);
  await this.usuarios.actualizarPassword(registro.usuarioId, passwordHash);
  await this.resetStore.marcarUsado(registro.id);
  await this.refreshStore.revocarTodas(registro.usuarioId); // cerrar sesiones existentes
}
```

Por qué cada decisión: token **aleatorio** (no adivinable) y **hasheado** en la base (si te filtran la tabla, los hashes no sirven para resetear); **expiración corta** y **un solo uso** (limita la ventana); **respuesta neutra** (no filtrás emails); **revocar sesiones** (si el atacante ya tenía una sesión, el reset la corta).

**Ejercicios 14**
14.1 ¿Por qué `POST /auth/forgot` debe responder lo mismo exista o no el email?
14.2 ¿Por qué se guarda el **hash** del token de reset y no el token en claro? Nombrá las tres propiedades que debe tener el token (aleatoriedad, expiración, un solo uso).
14.3 Tras cambiar la contraseña, ¿por qué conviene revocar los refresh tokens existentes del usuario?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 (a) autenticación  (b) autorización  (c) autenticación.
1.2 403 Forbidden: el usuario SÍ está autenticado (sabemos quién es), pero NO tiene
    permiso sobre ese recurso. 401 sería si no supiéramos quién es (token ausente/
    inválido).
1.3 Porque para decidir QUÉ puede hacer alguien primero tenés que saber QUIÉN es.
    Sin identidad confirmada, no hay roles ni permisos contra los cuales chequear.
```

### Módulo 2
```
2.1 Porque es demasiado rápido: un atacante con el hash filtrado puede probar miles
    de millones de candidatos por segundo (fuerza bruta / diccionario). Las
    contraseñas necesitan un hash LENTO a propósito (bcrypt/argon2) con cost factor.

2.2 El salt es un valor aleatorio por usuario que se combina con la contraseña antes
    de hashear. Previene los ataques con tablas precalculadas (rainbow tables) y hace
    que dos usuarios con la misma contraseña tengan hashes distintos. Con bcrypt NO
    necesitás columna aparte: el salt queda embebido dentro del propio string del hash.
```
```ts
// 2.3
const passwordHash = await bcrypt.hash(passwordPlano, 12); // al registrar
const coincide = await bcrypt.compare(passwordPlano, usuario.passwordHash); // al loguear
```

### Módulo 3
```
3.1 No, el payload solo está codificado en base64 (legible por cualquiera). La firma
    garantiza INTEGRIDAD/AUTENTICIDAD: que el token lo emitió el servidor y no fue
    modificado, porque solo el servidor conoce el secreto para firmarlo.

3.2 No funciona. Al cambiar el payload, la firma deja de corresponder; el servidor
    re-calcula la firma del payload recibido con su secreto, ve que no coincide con la
    firma del token, y lo rechaza (401). El atacante no puede re-firmar sin el secreto.

3.3 (1) Contraseñas o su hash: el payload es legible. (2) Datos personales sensibles
    (documentos, datos de salud): viajan a la vista en cada request. El JWT da
    integridad, no confidencialidad.
```

### Módulo 4
```
4.1 El access token va en cada request a endpoints protegidos. El refresh token solo
    se usa al renovar (POST /auth/refresh).
4.2 Porque si un access token de vida larga se filtra, sirve por semanas: el atacante
    tiene acceso prolongado. El de vida corta limita la ventana de daño; el refresh
    (revocable y poco expuesto) mantiene la sesión cómoda sin ese riesgo.
4.3 Porque el refresh se valida contra un store (base/Redis) al renovar, así que lo
    podés marcar como revocado y la próxima renovación falla. El access token JWT es
    stateless: nadie lo consulta contra la base, así que sigue siendo válido hasta que
    expira por sí mismo (de ahí que se lo haga de vida corta).
```

### Módulo 5
```
5.1 Porque localStorage es accesible desde JavaScript: un ataque XSS (script malicioso
    inyectado en tu página) puede leer el token y robar la sesión.
5.2 httpOnly impide que JavaScript lea la cookie; protege de XSS (un script inyectado
    no puede robar ese token).
5.3 SameSite=Strict hace que la cookie NO se envíe en requests originadas desde otro
    sitio; mitiga CSRF (que una página maliciosa dispare una request a tu API usando
    la cookie del usuario sin que él lo sepa).
```

### Módulo 6
```
6.1 Para no revelar qué emails están registrados. Si dijeras "email no existe" vs.
    "contraseña incorrecta", un atacante podría enumerar cuentas válidas. Mismo mensaje
    (y mismo tiempo de respuesta, comparando contra un hash dummy) = no das información
    útil al atacante.
```
```ts
// 6.2
import { IsEmail, MinLength } from "class-validator";

export class RegisterDto {
  @IsEmail()
  email!: string;

  @MinLength(12) // longitud > reglas de composición (guía NIST 2026)
  password!: string;
}
// Mejor aún: validar contra listas de contraseñas filtradas (have-i-been-pwned/zxcvbn)
// en vez de exigir mayúsculas/símbolos, que empujan a passwords predecibles.
```
```
6.3 Como el AuthService depende de la INTERFAZ UserRepository (por token), en los tests
    le inyectás un repo en memoria o un mock que devuelva un usuario fijo, y probás
    toda la lógica de login (hash, comparación, emisión de tokens) sin levantar Postgres.
```

### Módulo 7
```
7.1 Porque "cómo autenticar" es un algoritmo intercambiable: JWT, local (usuario+
    contraseña), Google son estrategias distintas con la misma interfaz. El resto de la
    app pide "autenticar" sin saber cuál se usa; cambiar de estrategia no cambia el
    código que la consume.
7.2 Que la firma del token es válida (lo emitió el servidor, no fue modificado) y que
    no está vencido (ignoreExpiration: false). Recién entonces llama a validate con el
    payload ya verificado.
7.3 Lo que devuelve validate queda en request.user. En el controlador lo accedés con
    @Req() (o un decorador @CurrentUser()) una vez que el guard de auth corrió.
```

### Módulo 8
```
8.1 El guard responde 401 automáticamente y el handler NO se ejecuta. La lógica del
    endpoint nunca corre con un token vencido.
8.2 Camino: el cliente manda "Authorization: Bearer <token>" → AuthGuard("jwt") dispara
    la JwtStrategy → Passport extrae el token (ExtractJwt) y verifica firma+expiración
    con el secreto → llama a validate(payload) → lo que validate devuelve se asigna a
    request.user → el controlador lo lee.
```
```ts
// 8.3
@UseGuards(AuthGuard("jwt"))
@Get("perfil")
perfil(@CurrentUser() user: JwtPayload): { email: string } {
  return { email: user.email };
}
```

### Módulo 9
```
9.1 Porque RolesGuard necesita request.user (con user.roles) para decidir, y eso lo
    pobla AuthGuard("jwt") al autenticar. Si RolesGuard corriera primero, no habría
    user todavía y no tendría roles que chequear.
9.3 No alcanza con RBAC: "ser dueño" no es un rol, es una relación entre el usuario y
    ESE recurso concreto. Lo chequeás en el service (o un guard con acceso al dato),
    comparando proyecto.usuarioId === user.sub y lanzando ForbiddenException si no. Es
    la frontera hacia ABAC (control por atributos).
```
```ts
// 9.2
@UseGuards(AuthGuard("jwt"), RolesGuard)
@Roles("admin")
@Delete(":id")
borrar(@Param("id", ParseIntPipe) id: number): Promise<void> {
  return this.usuariosService.borrar(id);
}
```

### Módulo 10
```
10.1 Es emitir un refresh token nuevo (e invalidar el anterior) en cada uso, de modo
     que cada refresh sirva una sola vez. Ayuda a detectar el ROBO: si aparece el mismo
     refresh token ya usado, es señal de reuso por un atacante → se revoca toda la
     familia y se fuerza re-login.
10.2 Porque "revocar" implica recordar qué tokens ya no valen, y eso es estado: lo
     guardás en base/Redis y lo consultás al renovar. Está bien hacerlo solo con el
     refresh porque se usa poco (cada 15 min, no en cada request), así que el costo de
     consultarlo es mínimo; el access se mantiene stateless y de vida corta por
     performance.
10.3 Logout = borrar/revocar el refresh token del store. El access token que el cliente
     todavía tiene sigue siendo válido hasta que expira (minutos), porque es stateless;
     por eso conviene que sea de vida corta. (Si necesitás corte inmediato, se usa una
     denylist de access tokens, a costa de volverlo stateful.)
```

### Módulo 11
```
11.1 Frena la fuerza bruta / credential stuffing: probar muchas contraseñas (o muchos
     pares email+password robados) hasta acertar. Sin límite de intentos, un atacante
     automatiza miles de pruebas por minuto contra una cuenta.
11.2 Para no revelar qué emails están registrados (enumeration attack): si respondieras
     distinto según exista o no, un atacante armaría una lista de cuentas válidas para
     atacar. Respondés siempre "si el email existe, enviamos el link".
11.3 (1) Si están en el código y el repo se filtra (o es público), cualquiera puede
     firmar tokens válidos y suplantar a cualquier usuario. (2) En variables de entorno
     son rotables y distintos por entorno (dev/prod) sin tocar el código.
```

### Módulo 12
```
12.1 Lo verifica Google, no tu backend. Al final del flujo recibís un código que tu
     backend cambia por la identidad del usuario (email, nombre, id de Google).
12.2 Los tuyos (access/refresh propios). Tras confirmar la identidad con Google, creás
     o encontrás al usuario en tu base y emitís tus propios tokens; tu sistema de
     sesiones sigue siendo el mismo y no dependés de Google en cada request.
12.3 El patrón Strategy: "login con Google" es otra estrategia de Passport
     (passport-google-oauth20), intercambiable con la estrategia JWT/local sin cambiar
     el resto de la app.
```

### Módulo 13
```
13.1 "Algo que sabés" (la contraseña) + "algo que tenés" (el teléfono con la app TOTP).
     El código TOTP cuenta como "algo que tenés" porque se deriva de un secreto que solo
     está en TU dispositivo (lo escaneaste del QR); un atacante con tu contraseña pero
     sin tu teléfono no puede generarlo.
13.2 Son códigos de un solo uso para entrar si perdés el teléfono (el segundo factor).
     Se guardan HASHEADOS como las contraseñas (si te filtran la base, no sirven), y cada
     uno se invalida al usarse.
13.3 En la base, CIFRADO (no en claro). NUNCA en el JWT: el payload es legible (módulo 3),
     así que meter el secreto TOTP ahí lo expondría en cada request, anulando el segundo
     factor.
```

### Módulo 14
```
14.1 Para no filtrar qué emails están registrados (enumeration): si respondiera distinto
     según exista o no, un atacante armaría una lista de cuentas válidas. Respuesta
     neutra siempre: "si el email existe, enviamos el link".
14.2 Porque si te filtran la tabla, un hash no permite resetear cuentas (no es el token
     real); guardar el token en claro sería como guardar contraseñas en claro.
     Propiedades: (1) aleatorio/criptográficamente seguro (no adivinable), (2) con
     expiración corta, (3) de un solo uso (se invalida al consumirse).
14.3 Porque si un atacante ya tenía una sesión activa (un refresh válido), cambiar la
     contraseña no lo echaría: hay que revocar las familias de refresh del usuario para
     cerrar todas las sesiones existentes y forzar re-login.
```

---

## Siguientes pasos

Con este módulo tenés el "Auth Service" del roadmap: registro con hashing, login con access+refresh, RBAC y las prácticas de seguridad que se evalúan en entrevistas. El recorrido natural desde acá: **(1)** conectá este auth a tu "Task API" — protegé los endpoints de proyectos/tareas con `AuthGuard("jwt")` y filtrá por `user.sub` para que cada quien vea solo lo suyo; **(2)** mové el store de refresh tokens a **Redis** (entra el módulo de caché/colas del roadmap); **(3)** sumá **tests**: unitarios del `AuthService` con el repo en memoria (mockear lo externo) y e2e del flujo login → request protegida con Supertest. El siguiente tema del temario con más retorno para entrevistas es **Node.js por dentro** (event loop, async, streams): los fundamentos del runtime que el resto del backend da por sabidos.
