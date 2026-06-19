# Testing práctico: de la teoría a tests que dan confianza

**Unitarios, integración y e2e · Jest, Supertest y testcontainers · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. El archivo senior ya te dio la *estrategia* (la pirámide, qué mockear y qué no, por qué 100% de cobertura engaña). Esta página la baja a tierra: cómo se escribe un test, cómo se mockea un repositorio, cómo se levanta la app entera para pegarle con HTTP, y cómo se prueba contra una **base de datos real** en un contenedor. El objetivo no es "tener tests", es **tener confianza** de que tu código hace lo que decís que hace — y poder refactorizar sin miedo.

**Lo que asumimos.** Que ya tenés tu "Task API" con el patrón Repository (en memoria y con Drizzle), DTOs y auth. Vamos a testear exactamente eso. El dominio sigue siendo usuarios ↔ proyectos ↔ tareas.

**Para practicar.** NestJS ya trae **Jest** configurado al crear el proyecto (`nest new`), con scripts `test`, `test:watch`, `test:cov` y `test:e2e`. Para los tests de integración con base real:

```bash
npm i -D @testcontainers/postgresql supertest @types/supertest
```

> **Jest o Vitest.** El roadmap menciona ambos. NestJS usa Jest por defecto, así que esta guía lo usa; **Vitest** tiene una API casi idéntica (`describe`/`it`/`expect`, mocks) y es más rápido — lo que aprendas acá se traslada casi sin cambios.

**Índice de módulos**
1. Por qué testear y la pirámide (en la práctica)
2. Anatomía de un test: AAA, `describe`/`it`/`expect`
3. Unit test de un service con un mock del repositorio
4. Dobles de prueba: mocks, stubs y spies
5. El `TestingModule` de Nest (DI en los tests)
6. Tests de integración contra Postgres real (testcontainers)
7. Tests e2e de endpoints con Supertest
8. Testear lo que está detrás de auth
9. Cobertura: qué mide y por qué 100% engaña
10. Tests confiables: deterministas, aislados y rápidos
11. Mockear el tiempo, lo aleatorio y la red

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Por qué testear y la pirámide (en la práctica)

**Teoría.** Un test es código que ejecuta tu código y **verifica** que el resultado es el esperado, de forma automática. Su valor real no es "encontrar bugs hoy", es **darte confianza para cambiar el código mañana**: si los tests pasan después de un refactor, sabés que no rompiste nada. Sin tests, cada cambio es una apuesta.

La **pirámide de tests** (del archivo senior) ordena el esfuerzo por capas:

```
        ╱ e2e ╲          pocos: lentos, frágiles, pero validan flujos completos
      ╱─────────╲
    ╱ integración ╲      algunos: tu código contra la DB/cola reales (en contenedor)
  ╱─────────────────╲
╱     unitarios       ╲  muchos: lógica de dominio, rápidos y estables
```

- **Unitarios**: prueban una unidad aislada (un service, una función) **sin** sus dependencias reales (las mockeás). Son rápidos (milisegundos) y estables. La mayoría de tus tests.
- **Integración**: prueban que tu código habla bien con algo **real** (la base, una cola), normalmente levantado en un contenedor. Más lentos, atrapan bugs que los unitarios no ven (mapeos, transacciones, constraints).
- **E2E (end-to-end)**: levantan la app entera y le pegan por HTTP, validando un flujo completo. Los más lentos y frágiles; pocos, para los caminos críticos.

La forma de pirámide responde a un criterio: invertí donde el test es **barato y estable** (abajo) y reservá los caros y frágiles (arriba) para lo que de verdad lo justifica. El anti-patrón opuesto (muchos e2e, pocos unitarios) se llama "cono de helado" y produce suites lentas que todos terminan ignorando.

**Ejercicios 1**
1.1 ¿Cuál es el valor principal de tener tests, más allá de encontrar bugs el día que los escribís?
1.2 Ordená de más a menos cantidad (según la pirámide): tests e2e, unitarios, de integración.
1.3 ¿Por qué conviene tener muchos unitarios y pocos e2e, y no al revés?

---

## Módulo 2 — Anatomía de un test: AAA, `describe`/`it`/`expect`

**Teoría.** Un test se organiza con tres funciones: `describe` agrupa, `it` (o `test`) define un caso, y `expect` hace la aserción.

El patrón interno que hace un test legible es **AAA** (Arrange, Act, Assert): **preparás** el escenario, **ejecutás** lo que querés probar, y **verificás** el resultado.

```ts
describe("suma", () => {
  it("suma dos números positivos", () => {
    // Arrange (preparar)
    const a = 2;
    const b = 3;

    // Act (ejecutar)
    const resultado = sumar(a, b);

    // Assert (verificar)
    expect(resultado).toBe(5);
  });
});
```

`expect(valor)` se combina con un **matcher**. Los más usados:

- `toBe(x)` — igualdad estricta (`===`), para primitivos.
- `toEqual(obj)` — igualdad profunda, para objetos y arrays.
- `toThrow()` — verifica que una función lance una excepción.
- `resolves` / `rejects` — para Promises: `await expect(p).rejects.toThrow(...)`.
- `toHaveBeenCalledWith(...)` — verifica cómo se llamó un mock (módulo 4).

Un buen test prueba **una cosa** y su nombre describe el comportamiento esperado ("devuelve 404 si el usuario no existe"), no la implementación. Si el nombre del test es claro, cuando falla ya sabés qué se rompió sin leer el código.

**Ejercicios 2**
2.1 ¿Qué significan las tres "A" de AAA y qué va en cada parte?
2.2 ¿Cuál es la diferencia entre `toBe` y `toEqual`? ¿Cuál usarías para comparar `{ id: 1 }` con `{ id: 1 }`?
2.3 Escribí un test que verifique que una función `dividir(10, 0)` lanza una excepción. Pista: `toThrow`.

---

## Módulo 3 — Unit test de un service con un mock del repositorio

**Teoría.** Acá se ve por qué valió la pena el patrón Repository. Tu `UsersService` depende de la **interfaz** `UserRepository`, no de la implementación concreta. En el test le pasás un **mock** (un objeto que cumple la interfaz) y probás toda la lógica del service **sin tocar la base**.

```ts
import { NotFoundException } from "@nestjs/common";

describe("UsersService", () => {
  it("devuelve el usuario si existe", async () => {
    // Arrange: mock del repositorio que devuelve un usuario fijo
    const usuarioFijo: Usuario = { id: 1, email: "ana@test.com", roles: ["user"] };
    const repoMock: UserRepository = {
      findById: async () => usuarioFijo,
      save: async (u) => ({ ...u, id: 1 }),
    };
    const service = new UsersService(repoMock);

    // Act
    const resultado = await service.obtener(1);

    // Assert
    expect(resultado).toEqual(usuarioFijo);
  });

  it("lanza NotFoundException si el usuario no existe", async () => {
    const repoMock: UserRepository = {
      findById: async () => null, // simulamos "no encontrado"
      save: async (u) => ({ ...u, id: 1 }),
    };
    const service = new UsersService(repoMock);

    await expect(service.obtener(99)).rejects.toThrow(NotFoundException);
  });
});
```

Fijate lo que NO hay: ni Postgres, ni el contenedor de DI de Nest, ni red. Solo lógica pura, en milisegundos. El segundo test verifica el **camino de error**, que suele ser donde están los bugs y donde los juniors no testean. Probá siempre el caso feliz **y** los bordes (no encontrado, datos inválidos, conflicto).

**Ejercicios 3**
3.1 ¿Por qué podés probar el `UsersService` sin levantar una base de datos real? ¿Qué se lo permite?
3.2 Escribí un mock de `UserRepository` cuyo `save` devuelva el usuario con un `id` asignado, y un test que verifique que `registrar(...)` del service devuelve ese usuario con `id`.
3.3 ¿Por qué es importante testear también el caso "no encontrado" / error, y no solo el camino feliz?

---

## Módulo 4 — Dobles de prueba: mocks, stubs y spies

**Teoría.** "Doble de prueba" (test double) es el término genérico para cualquier objeto falso que reemplaza una dependencia real. Los que más vas a usar:

- **Stub**: devuelve respuestas fijas (como el `repoMock` del módulo 3: "cuando te pidan `findById`, devolvé este usuario").
- **Mock**: además de responder, **registra cómo fue llamado** para que lo verifiques (¿se llamó?, ¿con qué argumentos?, ¿cuántas veces?).
- **Spy**: envuelve una función **real** para observar sus llamadas sin reemplazar su comportamiento (o reemplazándolo parcialmente).

Jest los crea con `jest.fn()`:

```ts
it("guarda el usuario con la contraseña hasheada", async () => {
  const repoMock = {
    findByEmail: jest.fn().mockResolvedValue(null), // stub: no existe ese email
    save: jest.fn().mockImplementation(async (u) => ({ ...u, id: 1 })),
  };
  const service = new AuthService(repoMock as unknown as UserRepository, jwtMock, configMock);

  await service.register({ email: "ana@test.com", password: "secreta123" });

  // Verificamos CÓMO se llamó save (mock), no solo el resultado:
  expect(repoMock.save).toHaveBeenCalledTimes(1);
  const guardado = repoMock.save.mock.calls[0][0];
  expect(guardado.passwordHash).not.toBe("secreta123"); // se hasheó, no se guardó en claro
});
```

`mockResolvedValue(x)` hace que la función async devuelva `x`; `mockImplementation(fn)` le da un comportamiento. Y `toHaveBeenCalledWith` / `toHaveBeenCalledTimes` verifican la **interacción**. Esto es clave para probar efectos: "¿el service llamó al repositorio para guardar?", "¿llamó al mailer una sola vez?".

Cuidado con sobre-mockear: si tu test verifica demasiados detalles de *cómo* se hizo algo (qué métodos internos se llamaron), se rompe ante cualquier refactor aunque el comportamiento siga bien. Preferí verificar **resultados** y reservá la verificación de interacción para los efectos que importan (que se haya guardado, que se haya enviado el email).

**Ejercicios 4**
4.1 ¿Cuál es la diferencia entre un stub y un mock?
4.2 ¿Qué hace `jest.fn().mockResolvedValue(null)` y cuándo lo usarías al testear un login?
4.3 ¿Qué riesgo tiene un test que verifica demasiados detalles de implementación (qué métodos internos se llamaron)?

---

## Módulo 5 — El `TestingModule` de Nest (DI en los tests)

**Teoría.** En el módulo 3 instanciamos el service a mano (`new UsersService(mock)`). Eso funciona para casos simples, pero cuando el service tiene muchas dependencias o querés probar la wiring de la DI, Nest te da `Test.createTestingModule`: arma un mini-módulo de prueba donde **reemplazás los providers reales por mocks** y pedís las instancias ya inyectadas.

```ts
import { Test, TestingModule } from "@nestjs/testing";

describe("UsersService (con TestingModule)", () => {
  let service: UsersService;
  const repoMock = { findById: jest.fn(), save: jest.fn() };

  beforeEach(async () => {
    const moduleRef: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: "USER_REPO", useValue: repoMock }, // mock en lugar del repo real
      ],
    }).compile();

    service = moduleRef.get(UsersService);
  });

  afterEach(() => jest.clearAllMocks()); // limpiamos los mocks entre tests

  it("delega en el repositorio", async () => {
    repoMock.findById.mockResolvedValue({ id: 1, email: "ana@test.com", roles: ["user"] });
    const u = await service.obtener(1);
    expect(u.id).toBe(1);
    expect(repoMock.findById).toHaveBeenCalledWith(1);
  });
});
```

Las piezas: `providers` arma el grafo de DI con tus mocks; `.compile()` lo construye; `moduleRef.get(...)` te da la instancia. Los **hooks** `beforeEach`/`afterEach` (y `beforeAll`/`afterAll`) preparan y limpian el escenario — `jest.clearAllMocks()` entre tests evita que las llamadas de un test contaminen al siguiente (uno de los errores más comunes: tests que pasan solos pero fallan juntos por estado compartido).

**Ejercicios 5**
5.1 ¿Qué ventaja te da `Test.createTestingModule` sobre instanciar el service con `new` a mano?
5.2 ¿Para qué sirve `{ provide: "USER_REPO", useValue: repoMock }` dentro de los `providers` del test?
5.3 ¿Por qué conviene llamar a `jest.clearAllMocks()` en un `afterEach`? ¿Qué problema evita?

---

## Módulo 6 — Tests de integración contra Postgres real (testcontainers)

**Teoría.** Los unitarios prueban tu lógica, pero **no** prueban que tu repositorio Drizzle hable bien con Postgres: los mapeos de columnas, las constraints, las transacciones, las queries reales. Esos bugs solo aparecen contra una **base de verdad**. La regla del archivo senior: **no mockees tu propia base de datos** en los tests de integración; usá una real, descartable, en un **contenedor**.

**testcontainers** levanta un Postgres efímero en Docker para tus tests y lo tira al terminar. Cada corrida arranca limpia, sin depender de una base instalada a mano:

```ts
import { PostgreSqlContainer, StartedPostgreSqlContainer } from "@testcontainers/postgresql";
import { drizzle, NodePgDatabase } from "drizzle-orm/node-postgres";
import { Pool } from "pg";

describe("DrizzleUserRepository (integración)", () => {
  let container: StartedPostgreSqlContainer;
  let db: NodePgDatabase;
  let repo: DrizzleUserRepository;

  beforeAll(async () => {
    container = await new PostgreSqlContainer("postgres:16").start();
    const pool = new Pool({ connectionString: container.getConnectionUri() });
    db = drizzle(pool);
    await migrar(db);                 // aplicás tus migraciones a la base efímera
    repo = new DrizzleUserRepository(db);
  }, 60_000);                          // timeout amplio: arrancar el contenedor tarda

  afterAll(async () => {
    await container.stop();            // se tira el contenedor al final
  });

  it("guarda y recupera un usuario por id", async () => {
    const creado = await repo.save({ email: "ana@test.com", roles: ["user"] });
    const encontrado = await repo.findById(creado.id);
    expect(encontrado?.email).toBe("ana@test.com");
  });

  it("falla al insertar un email duplicado (constraint UNIQUE)", async () => {
    await repo.save({ email: "dup@test.com", roles: ["user"] });
    await expect(repo.save({ email: "dup@test.com", roles: ["user"] })).rejects.toThrow();
    // este bug NUNCA aparecería contra un mock: solo la DB real conoce el UNIQUE
  });
});
```

El segundo test es el argumento entero a favor de la integración: la restricción `UNIQUE` vive en Postgres, no en tu código TypeScript; un mock la ignoraría y te daría falsa confianza. Estos tests son más lentos (levantar Docker tarda segundos), por eso son **menos** que los unitarios — pero atrapan una categoría de bugs que ningún unitario ve.

**Ejercicios 6**
6.1 ¿Por qué el archivo senior dice que NO hay que mockear tu propia base de datos en los tests de integración?
6.2 Nombrá dos tipos de bug que un test de integración contra Postgres atrapa y un unitario con mock no.
6.3 ¿Qué ventaja da usar testcontainers en lugar de apuntar los tests a una base de datos instalada manualmente en tu máquina?

---

## Módulo 7 — Tests e2e de endpoints con Supertest

**Teoría.** Un test **e2e** levanta tu aplicación Nest entera y le pega por HTTP, como lo haría un cliente real: valida el endpoint, los pipes de validación, los guards, la serialización, todo el pipeline junto. La herramienta es **Supertest**.

```ts
import { Test } from "@nestjs/testing";
import { INestApplication, ValidationPipe } from "@nestjs/common";
import request from "supertest";

describe("Usuarios (e2e)", () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [AppModule], // la app completa
    }).compile();

    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true })); // mismo setup que main.ts
    await app.init();
  });

  afterAll(async () => await app.close()); // graceful shutdown del módulo de Node

  it("POST /usuarios crea un usuario y devuelve 201", async () => {
    const res = await request(app.getHttpServer())
      .post("/usuarios")
      .send({ email: "ana@test.com", password: "secreta123" })
      .expect(201);

    expect(res.body.email).toBe("ana@test.com");
    expect(res.body.password).toBeUndefined(); // el DTO de salida NO expone la contraseña
  });

  it("POST /usuarios rechaza un email inválido con 400", async () => {
    await request(app.getHttpServer())
      .post("/usuarios")
      .send({ email: "no-es-email", password: "secreta123" })
      .expect(400); // lo frena el ValidationPipe + el DTO, sin tocar el service
  });
});
```

Lo potente: el segundo test verifica que tu `ValidationPipe` y tu DTO rechazan basura **antes** de llegar a la lógica — algo que un unitario del service no cubre, porque salta esa capa. Y el primero confirma que el DTO de salida no filtra la contraseña (la regla del módulo de patrones). Para el e2e, conectá la app a una base de prueba (testcontainers del módulo 6) o, si el flujo no toca la DB, a mocks.

Los e2e son los más lentos y frágiles (cualquier cambio en una ruta los rompe), por eso van **pocos**: los caminos críticos (registro, login, el flujo estrella), no cada endpoint.

**Ejercicios 7**
7.1 ¿Qué capas de tu app ejercita un test e2e que un unitario del service se saltea?
7.2 Escribí (en pseudocódigo o con Supertest) un test que verifique que `GET /usuarios/:id` con un id inexistente devuelve `404`.
7.3 ¿Por qué los e2e van "pocos y para lo crítico" y no uno por cada endpoint?

---

## Módulo 8 — Testear lo que está detrás de auth

**Teoría.** Tus endpoints están protegidos con `AuthGuard("jwt")` y `RolesGuard` (módulo de auth). En los tests tenés dos caminos según qué quieras probar:

**1. Probar el flujo real de auth** (e2e): te logueás de verdad, obtenés un token y lo mandás en el header. Verifica que la cadena completa funciona.

```ts
it("GET /proyectos sin token devuelve 401", async () => {
  await request(app.getHttpServer()).get("/proyectos").expect(401);
});

it("GET /proyectos con token válido devuelve 200", async () => {
  const { body } = await request(app.getHttpServer())
    .post("/auth/login")
    .send({ email: "ana@test.com", password: "secreta123" });

  await request(app.getHttpServer())
    .get("/proyectos")
    .set("Authorization", `Bearer ${body.accessToken}`)
    .expect(200);
});
```

**2. Saltarte el guard** cuando estás probando *otra cosa* y la auth es solo un estorbo: sobreescribís el guard en el `TestingModule` con uno que siempre deja pasar e inyecta un usuario fijo.

```ts
const moduleRef = await Test.createTestingModule({ imports: [AppModule] })
  .overrideGuard(AuthGuard("jwt"))
  .useValue({ canActivate: () => true }) // siempre pasa, en estos tests no probamos auth
  .compile();
```

El criterio: si lo que estás probando **es** la autorización (que sin token da 401, que sin rol admin da 403), usá el flujo real. Si estás probando la lógica de proyectos y la auth solo se interpone, sobreescribí el guard para no acoplar cada test al login. Tené al menos **un** test que sí verifique que la protección funciona — si todos saltan el guard, nadie comprueba que las rutas están realmente protegidas.

**Ejercicios 8**
8.1 ¿Cuándo conviene probar el flujo real de login (obtener token) y cuándo sobreescribir el guard para que siempre pase?
8.2 Escribí el test que verifique que un endpoint protegido devuelve `401` cuando no se manda token.
8.3 Si en TODOS tus tests sobreescribís el guard para que pase, ¿qué riesgo corrés?

---

## Módulo 9 — Cobertura: qué mide y por qué 100% engaña

**Teoría.** La **cobertura** (coverage) mide qué porcentaje de tu código fue **ejecutado** por los tests. Jest lo reporta con `npm run test:cov`, desglosado en líneas, ramas (`if`/`else`), funciones y statements.

```bash
npm run test:cov
# Statements: 84% | Branches: 71% | Functions: 90% | Lines: 85%
```

Es **útil como termómetro** (te muestra zonas sin ningún test), pero perseguir el **100% es una trampa**, como advierte el archivo senior. Dos razones:

1. **Mide ejecución, no verificación.** Un test puede ejecutar una función (sube la cobertura) sin una sola aserción significativa. 100% de líneas ejecutadas no significa 100% de comportamiento verificado.
2. **Empuja a tests inútiles.** Para arañar el último 10% se terminan escribiendo tests de getters triviales y ramas imposibles, que cuestan mantenimiento y no atrapan ningún bug real.

La **cobertura de ramas** (branches) suele ser más reveladora que la de líneas: te dice si probaste *ambos* lados de tus `if` (el camino feliz **y** el de error). Un 70-80% de cobertura **significativa**, con foco en la lógica de dominio y los caminos de error, vale más que un 100% inflado con tests vacíos. La pregunta correcta no es "¿qué % tengo?" sino "¿confío en poder refactorizar sin romper nada?".

**Ejercicios 9**
9.1 ¿Qué mide exactamente la cobertura, y qué NO garantiza?
9.2 ¿Por qué perseguir el 100% de cobertura puede llevar a tests de mala calidad?
9.3 ¿Por qué la cobertura de ramas (branches) suele ser más informativa que la de líneas?

---

## Módulo 10 — Tests confiables: deterministas, aislados y rápidos

**Teoría.** Un test que a veces pasa y a veces falla sin que el código cambie es un **test flaky**, y es peor que no tener test: erosiona la confianza hasta que el equipo empieza a ignorar la suite ("ah, ese falla a veces, dale igual"). Las propiedades de un buen test:

- **Determinista**: mismo input → mismo resultado, siempre. Las causas típicas de flakiness: depender de la **fecha/hora** real, de valores **aleatorios**, de **timers** reales, del **orden** de ejecución, o de **red** externa. Todo eso se controla (módulo 11).
- **Aislado**: no depende de otros tests ni del estado que dejaron. Cada test prepara su escenario (`beforeEach`) y limpia (`afterEach`). Un test no debería romperse porque otro corrió antes (o no corrió).
- **Rápido**: la suite tiene que ser lo bastante ágil como para correrla seguido. Por eso la pirámide: muchos unitarios veloces, pocos e2e lentos.
- **Que pruebe comportamiento, no implementación**: el test sobrevive a un refactor si verifica *qué* hace el código (su resultado observable), no *cómo* lo hace por dentro.

Un olor a problema: si tenés que correr los tests "en cierto orden" para que pasen, o si limpiar la base entre tests los arregla, tenés un problema de aislamiento. Cada test debe poder correr solo y en cualquier orden.

**Ejercicios 10**
10.1 ¿Qué es un test "flaky" y por qué es peor que no tener ese test?
10.2 Nombrá tres causas típicas de flakiness.
10.3 Tus tests pasan cuando corrés el archivo entero, pero uno falla si lo corrés solo. ¿Qué problema tenés y cómo lo encarás?

---

## Módulo 11 — Mockear el tiempo, lo aleatorio y la red

**Teoría.** Para que un test sea determinista (módulo 10), hay que **controlar lo incontrolable**: lo que cambia entre corridas. El criterio del archivo senior: mockeá lo que está **fuera de tu control** (el reloj, lo aleatorio, APIs de terceros) y lo lento.

**El tiempo.** Si tu código usa `Date.now()` o `new Date()`, el test daría distinto cada día. Jest tiene timers falsos:

```ts
it("marca la tarea con la fecha actual", () => {
  jest.useFakeTimers().setSystemTime(new Date("2026-06-19T10:00:00Z"));

  const tarea = crearTarea("Comprar pan");

  expect(tarea.creadaEl).toEqual(new Date("2026-06-19T10:00:00Z"));
  jest.useRealTimers(); // restaurar
});
```

**Lo aleatorio.** Si generás IDs o tokens con `Math.random()`/`crypto`, mockealo para que devuelva un valor fijo en el test, o inyectá un generador (otra vez, la inversión de dependencias rinde).

**La red / APIs de terceros.** Nunca pegues a la API real de Stripe o de un mailer en un test: es lento, frágil y podría cobrar de verdad. Acá brilla el patrón **Ports & Adapters** del archivo de patrones: tu caso de uso depende de `PagoPort`; en el test inyectás `PagoFalsoAdapter` y probás toda la lógica sin red ni claves, en milisegundos. "Testear el puerto, no el adaptador" — esa era, textualmente, la promesa de la arquitectura hexagonal cumpliéndose.

Qué **no** mockear (recordatorio del módulo 6): tu propia base en los tests de integración. Ahí justamente querés lo real (en contenedor). Mockeás lo de afuera y lo no determinista; lo tuyo, lo probás de verdad.

**Ejercicios 11**
11.1 ¿Por qué un test que usa `Date.now()` sin mockear puede volverse flaky? ¿Cómo lo controlás en Jest?
11.2 ¿Por qué nunca deberías pegarle a la API real de un proveedor de pagos en un test? ¿Qué inyectarías en su lugar?
11.3 Conectá con hexagonal: ¿cómo hace "depender de `PagoPort`" que testear el pago sea trivial y sin red?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Te dan confianza para CAMBIAR el código: si tras un refactor los tests siguen
    verdes, sabés que no rompiste el comportamiento. Son una red de seguridad que
    permite evolucionar el código sin miedo.
1.2 De más a menos: unitarios > integración > e2e.
1.3 Porque los unitarios son baratos, rápidos y estables (los corrés todo el tiempo) y
    los e2e son lentos y frágiles (se rompen por cualquier cambio). Muchos e2e dan una
    suite lenta que el equipo termina ignorando (anti-patrón "cono de helado").
```

### Módulo 2
```
2.1 Arrange (preparar el escenario: datos, mocks), Act (ejecutar lo que se prueba),
    Assert (verificar el resultado con expect).
2.2 toBe usa igualdad estricta (===), sirve para primitivos y misma referencia.
    toEqual compara en profundidad por valor. Para { id: 1 } vs { id: 1 } (objetos
    distintos con mismo contenido) usás toEqual.
```
```ts
// 2.3
it("dividir por cero lanza error", () => {
  expect(() => dividir(10, 0)).toThrow();
});
```

### Módulo 3
```
3.1 Porque el service depende de la INTERFAZ UserRepository (por un token), no de la
    implementación concreta. En el test le inyectás un mock que cumple la interfaz, así
    que no hace falta Postgres ni el contenedor de DI: probás solo la lógica.
3.3 Porque los caminos de error (no encontrado, datos inválidos, conflicto) son donde
    suelen esconderse los bugs y donde los juniors no testean. El camino feliz solo no
    da confianza: tenés que cubrir también los bordes.
```
```ts
// 3.2
it("registrar devuelve el usuario con id asignado", async () => {
  const repoMock: UserRepository = {
    findById: async () => null,
    save: async (u) => ({ ...u, id: 1 }),
  };
  const service = new UsersService(repoMock);
  const creado = await service.registrar({ email: "ana@test.com", roles: ["user"] });
  expect(creado.id).toBe(1);
});
```

### Módulo 4
```
4.1 Un stub solo devuelve respuestas fijas (no verifica nada). Un mock, además de
    responder, registra cómo fue llamado (argumentos, cantidad de veces) para que lo
    verifiques con aserciones de interacción.
4.2 Hace que una función async devuelva null al llamarla. Al testear un login lo usás
    para simular "el email no existe" (findByEmail resuelve null), y verificar que el
    service lanza UnauthorizedException.
4.3 Que el test se acopla a CÓMO está implementado, no a QUÉ hace. Cualquier refactor
    interno (que no cambia el comportamiento) lo rompe, generando tests frágiles que dan
    falsas alarmas y desincentivan refactorizar.
```

### Módulo 5
```
5.1 Construye el grafo de DI real con tus mocks: si el service tiene muchas
    dependencias, no las cableás a mano una por una, y probás que la inyección por
    tokens funciona como en producción. Más cercano a cómo Nest arma el service.
5.2 Reemplaza el provider real del token "USER_REPO" por tu mock: cuando el service
    pida ese token, Nest le inyecta el mock en lugar del repositorio real (Postgres).
5.3 Para resetear los registros de llamadas y las implementaciones de los mocks entre
    tests. Evita que las llamadas de un test "se filtren" al siguiente y produzcan
    tests que pasan o fallan según el orden (falta de aislamiento).
```

### Módulo 6
```
6.1 Porque mockear la base elimina justo lo que esos tests deben verificar: que tu
    código habla bien con Postgres (mapeos de columnas, transacciones, constraints,
    queries). Un mock siempre "se porta bien" y te da falsa confianza.
6.2 Por ejemplo: violaciones de constraints (UNIQUE, NOT NULL, foreign keys),
    errores de mapeo columna↔propiedad, comportamiento real de transacciones/rollback,
    y queries mal escritas. Un mock no conoce nada de eso.
6.3 testcontainers levanta una base efímera y limpia por corrida y la tira al terminar:
    reproducible, aislada y sin depender de que cada dev tenga una base instalada y en
    cierto estado. El mismo test corre igual en tu máquina y en el CI.
```

### Módulo 7
```
7.1 Ejercita el pipeline HTTP completo: routing, los pipes de validación (DTOs), los
    guards (auth), la serialización de la respuesta y los exception filters. El unitario
    del service salta todas esas capas y prueba solo la lógica.
7.3 Porque son lentos y frágiles: levantan la app entera y se rompen ante cualquier
    cambio de ruta/contrato. Se reservan para los flujos críticos (registro, login, el
    camino estrella); el resto se cubre con unitarios e integración, más baratos.
```
```ts
// 7.2
it("GET /usuarios/:id inexistente devuelve 404", async () => {
  await request(app.getHttpServer()).get("/usuarios/999999").expect(404);
});
```

### Módulo 8
```
8.1 Probás el flujo real (login → token → header) cuando lo que querés verificar ES la
    autorización (sin token 401, sin rol 403, con token OK). Sobreescribís el guard
    cuando probás OTRA cosa (la lógica de proyectos) y la auth solo se interpone: así no
    acoplás cada test al login.
8.3 Que nadie verifica que las rutas están realmente protegidas: un bug que deje un
    endpoint abierto pasaría todos los tests. Hay que tener al menos un test que use el
    guard real y compruebe el 401/403.
```
```ts
// 8.2
it("GET /proyectos sin token devuelve 401", async () => {
  await request(app.getHttpServer()).get("/proyectos").expect(401);
});
```

### Módulo 9
```
9.1 Mide qué porcentaje del código fue EJECUTADO por los tests (líneas, ramas,
    funciones). NO garantiza que ese código esté VERIFICADO: ejecutar una función sin
    aserciones significativas igual cuenta como cubierta.
9.2 Porque para llegar al 100% se terminan escribiendo tests triviales (getters, ramas
    imposibles) solo para tocar líneas, que cuestan mantenimiento y no atrapan bugs. El
    número sube, la confianza real no.
9.3 Porque las ramas te dicen si probaste AMBOS caminos de cada if (el feliz y el de
    error). Podés tener 100% de líneas ejecutando solo el camino feliz y dejar todos los
    casos de error sin probar; la cobertura de ramas lo delata.
```

### Módulo 10
```
10.1 Un test flaky pasa o falla de forma intermitente sin que el código cambie. Es peor
     que no tenerlo porque erosiona la confianza en la suite: el equipo empieza a
     ignorar fallos ("ese falla a veces") y deja pasar bugs reales.
10.2 Depender de la fecha/hora real, de valores aleatorios, de timers reales, del orden
     de ejecución entre tests, o de servicios de red externos. (Cualquiera de estas.)
10.3 Es un problema de aislamiento: ese test depende del estado que dejó otro (datos en
     la base, un mock no reseteado, una variable compartida). Lo encarás haciendo que
     cada test prepare su propio escenario en beforeEach y limpie en afterEach
     (clearAllMocks, resetear/transaccionar la base), de modo que corra solo y en
     cualquier orden.
```

### Módulo 11
```
11.1 Porque Date.now() devuelve algo distinto en cada corrida, así que una aserción
     sobre la fecha fallaría según el momento. En Jest lo controlás con
     jest.useFakeTimers().setSystemTime(fecha) para fijar un "ahora" determinista (y
     jest.useRealTimers() al final).
11.2 Porque es lento, frágil (la API puede estar caída) y peligroso (podrías generar
     un cobro real). En su lugar inyectás un doble: un adaptador falso (PagoFalsoAdapter)
     que cumple el puerto y devuelve un resultado fijo.
11.3 Como el caso de uso depende de la abstracción PagoPort y no de StripeAdapter, en el
     test le inyectás PagoFalsoAdapter (que implementa el puerto) y probás toda la lógica
     sin red, sin claves y en milisegundos. Es "testear el puerto, no el adaptador".
```

---

## Siguientes pasos

Con este módulo cerrás la teoría de testing del archivo senior con práctica concreta: unitarios con mocks para la lógica, integración con testcontainers para la persistencia, e2e con Supertest para los flujos críticos, y el criterio para que sean confiables. El recorrido natural desde acá: **(1)** llevá tu "Task API" a una cobertura **significativa** (>70% con foco en dominio y caminos de error), no a un 100% inflado; **(2)** integrá los tests al **pipeline de CI/CD** (el siguiente módulo, Docker + deploy): que cada push corra `lint → test → build` y no mergee si algo falla verde; **(3)** sumá **contract testing** el día que partas a microservicios (lo plantea el archivo senior). Tener una suite en la que confiás es lo que te deja refactorizar a hexagonal, cambiar de ORM o extraer un servicio sin miedo — todo lo que viene se apoya en esta red de seguridad.
