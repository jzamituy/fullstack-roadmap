# TypeScript: teoría + ejercicios para reactivar tu conocimiento

**Para frontend en transición a full stack · de lo básico a lo avanzado · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y recién después contrastá con la sección final. Empezá con `tsconfig` en modo estricto (`"strict": true`); es como vas a trabajar en producción y como aprendés de verdad. Podés hacer todo en el [TypeScript Playground](https://www.typescriptlang.org/play) sin instalar nada.

**Índice de módulos**
1. Tipos básicos e inferencia
2. Funciones bien tipadas
3. Objetos: `interface` vs `type`
4. Uniones, literales y narrowing
5. Arrays, tuplas y enums
6. Generics
7. Utility types
8. Discriminated unions y type guards
9. Async / Promises tipadas
10. Clases y modificadores (orientado a backend)
11. Tipos avanzados: mapped, conditional y template literal types

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Tipos básicos e inferencia

**Teoría.** TypeScript es JavaScript + un sistema de tipos que se chequea en tiempo de compilación. Los tipos primitivos son `string`, `number`, `boolean`, `null`, `undefined`, `symbol`, `bigint`. TS **infiere** el tipo cuando inicializás una variable, así que no hace falta anotar todo:

```ts
let nombre = "Jorge";   // inferido: string
let edad = 30;          // inferido: number
const activo = true;    // inferido: literal true (por const)
```

Evitá `any` (apaga el chequeo de tipos). Cuando no conocés el tipo pero querés seguridad, usá `unknown`: te obliga a verificar antes de usarlo. Anotá explícitamente cuando declarás sin inicializar o en límites públicos (parámetros, retornos).

```ts
let dato: unknown = JSON.parse(input);
// dato.toUpperCase()  // ❌ error: hay que estrechar primero
if (typeof dato === "string") dato.toUpperCase(); // ✅
```

**Ejercicios 1**
1.1 Declará variables para: un email (string), un precio (number), un flag de "publicado" (boolean) y una fecha de borrado que puede no existir todavía. Anotá solo lo que sea necesario.
1.2 Tenés `const config = { puerto: 3000, host: "localhost" }`. ¿Qué tipo infiere TS para `config.puerto`? ¿Y si fuera `let`? Explicá la diferencia entre `any` y `unknown` en una frase.
1.3 Escribí una función `parseEntrada(raw: unknown): number` que devuelva el número si `raw` es un número, y `0` en cualquier otro caso (sin usar `any`).

---

## Módulo 2 — Funciones bien tipadas

**Teoría.** Tipás parámetros y, opcionalmente, el retorno (TS lo infiere, pero anotarlo documenta y atrapa errores). Parámetros opcionales con `?`, valores por defecto, y rest params tipados.

```ts
function saludar(nombre: string, formal: boolean = false): string {
  return formal ? `Estimado ${nombre}` : `Hola ${nombre}`;
}

const sumar = (...nums: number[]): number => nums.reduce((a, b) => a + b, 0);
```

Para tipar una función como valor usás un *function type*:

```ts
type Operacion = (a: number, b: number) => number;
const dividir: Operacion = (a, b) => a / b;
```

Funciones que no retornan nada usan `void`; las que nunca terminan (lanzan o loop infinito) usan `never`.

**Ejercicios 2**
2.1 Tipá una función `crearUsuario(nombre: string, rol?: string)` donde `rol` por defecto sea `"user"`. Que retorne un objeto `{ nombre, rol }`.
2.2 Declará un tipo `Validador` que represente una función que recibe un `string` y devuelve `boolean`. Implementá dos validadores (`esEmail`, `noVacio`) con ese tipo.
2.3 Escribí `lanzarError(msg: string): never` que siempre lance una excepción. Explicá por qué `never` y no `void`.

---

## Módulo 3 — Objetos: `interface` vs `type`

**Teoría.** Para describir la forma de un objeto podés usar `interface` o `type`. Ambos sirven; reglas prácticas: usá `interface` para formas de objetos/clases que podrían extenderse, y `type` para uniones, intersecciones, tuplas o alias. Las propiedades opcionales usan `?`, las inmutables `readonly`.

```ts
interface Usuario {
  readonly id: number;
  nombre: string;
  email: string;
  edad?: number;          // opcional
}

type ID = string | number;            // type para unión
type UsuarioConRol = Usuario & { rol: string }; // intersección
```

Las interfaces se pueden extender (`interface Admin extends Usuario { ... }`). Cuidado con el *excess property check*: TS se queja si pasás propiedades de más en un objeto literal.

**Ejercicios 3**
3.1 Modelá una `interface Producto` con: `id` readonly numérico, `nombre`, `precio`, `stock`, y `descripcion` opcional.
3.2 Creá un `type ProductoConDescuento` que sea `Producto` más una propiedad `descuento: number` (usá intersección).
3.3 Definí `interface Repositorio` con un método `obtenerPorId(id: number): Producto | undefined`. (Anticipo del patrón Repository que vas a usar en backend.)

---

## Módulo 4 — Uniones, literales y narrowing

**Teoría.** Una *unión* (`A | B`) significa "uno u otro". Los *literal types* restringen a valores concretos (`"GET" | "POST"`). El *narrowing* es cómo TS reduce el tipo dentro de un bloque usando `typeof`, `in`, `instanceof` o comparaciones.

```ts
type Metodo = "GET" | "POST" | "PUT" | "DELETE";

function manejar(valor: string | number) {
  if (typeof valor === "string") {
    return valor.trim();      // acá valor es string
  }
  return valor.toFixed(2);    // acá valor es number
}
```

Los literales evitan los "stringly-typed bugs": el compilador te frena si escribís `"PETCH"`.

**Ejercicios 4**
4.1 Definí un tipo `Estado = "pendiente" | "activo" | "cancelado"`. Escribí `puedeEditar(estado: Estado): boolean` que sea `true` solo si está `"pendiente"` o `"activo"`.
4.2 Escribí `formatear(valor: string | number | Date): string` que devuelva una representación en texto según el tipo (usá narrowing con `typeof` e `instanceof`).
4.3 ¿Por qué `type Metodo = "GET" | "POST"` es mejor que `string` para el método de un request? Respondé en una o dos frases.

---

## Módulo 5 — Arrays, tuplas y enums

**Teoría.** Arrays se tipan `number[]` o `Array<number>`. Las *tuplas* son arrays de longitud y tipos fijos por posición: `[string, number]`. Los `enum` agrupan constantes con nombre, aunque en 2026 muchos prefieren *uniones de literales* o `as const` por ser más livianos y sin código generado.

```ts
const ids: number[] = [1, 2, 3];
const par: [string, number] = ["edad", 30];

// enum clásico
enum Rol { Admin, User, Guest }

// alternativa moderna recomendada
const ROLES = ["admin", "user", "guest"] as const;
type Rol2 = typeof ROLES[number]; // "admin" | "user" | "guest"
```

**Ejercicios 5**
5.1 Tipá una variable `coordenada` como tupla `[number, number]` (lat, lng) e inicializala.
5.2 Definí `ESTADOS_PEDIDO` con `as const` y derivá un tipo `EstadoPedido` a partir del array (técnica muy usada para evitar enums).
5.3 Escribí `primero<T>(arr: T[]): T | undefined` que devuelva el primer elemento o `undefined` si está vacío. (Sí, ya es genérico: prepara el módulo 6.)

---

## Módulo 6 — Generics

**Teoría.** Los *generics* permiten escribir código reutilizable que preserva el tipo. `<T>` es un parámetro de tipo que se resuelve al usarse. Podés restringirlos con `extends`.

```ts
function envolver<T>(valor: T): { data: T } {
  return { data: valor };
}
const r = envolver("hola"); // r: { data: string }

// restricción: T debe tener id numérico
function porId<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find((i) => i.id === id);
}
```

Son la base de las APIs tipadas (un `ApiResponse<T>`, un `Repository<T>`, etc.).

**Ejercicios 6**
6.1 Escribí `ultimo<T>(arr: T[]): T | undefined`.
6.2 Definí `interface ApiResponse<T>` con `{ ok: boolean; data: T; error?: string }`. Tipá una respuesta de lista de usuarios y una de un solo producto.
6.3 Escribí `agruparPor<T, K extends keyof T>(items: T[], clave: K)` que devuelva un objeto agrupando los items por el valor de esa clave. (Reto: el tipo de retorno puede ser `Record<string, T[]>`.)

---

## Módulo 7 — Utility types

**Teoría.** TS trae tipos utilitarios que transforman otros tipos. Los más usados en backend:

- `Partial<T>` — todas las props opcionales (ideal para updates).
- `Required<T>` — todas obligatorias.
- `Pick<T, K>` — selecciona un subconjunto de props.
- `Omit<T, K>` — quita props (ideal para DTOs sin `id`/`password`).
- `Record<K, V>` — objeto con claves `K` y valores `V`.
- `Readonly<T>` — todo inmutable.
- `ReturnType<F>` / `Parameters<F>` — extraen tipos de funciones.

```ts
interface Usuario { id: number; nombre: string; email: string; password: string; }

type NuevoUsuario = Omit<Usuario, "id">;         // para crear
type UsuarioPublico = Omit<Usuario, "password">; // para responder
type UpdateUsuario = Partial<Omit<Usuario, "id">>; // para editar
```

**Ejercicios 7**
7.1 Dado `interface Tarea { id: number; titulo: string; hecha: boolean; creada: Date }`, definí: `CrearTareaDTO` (sin `id` ni `creada`), `ActualizarTareaDTO` (todo opcional excepto que nunca incluya `id`), y `TareaPreview` (solo `id` y `titulo`).
7.2 Usando `Record`, tipá un objeto `contadoresPorEstado` que mapee cada `EstadoPedido` (del módulo 5) a un `number`.
7.3 Tenés `function crearToken(userId: number, rol: string) { return { token: "...", exp: 3600 }; }`. Definí un tipo `Token` usando `ReturnType` sin reescribir la forma a mano.

---

## Módulo 8 — Discriminated unions y type guards

**Teoría.** Una *discriminated union* es una unión de objetos que comparten una propiedad "discriminante" (un literal) que TS usa para estrechar con seguridad. Es el patrón rey para modelar estados (loading/success/error) y resultados.

```ts
type Resultado<T> =
  | { estado: "ok"; data: T }
  | { estado: "error"; mensaje: string };

function manejar<T>(r: Resultado<T>) {
  switch (r.estado) {
    case "ok":    return r.data;       // TS sabe que hay data
    case "error": return r.mensaje;    // TS sabe que hay mensaje
  }
}
```

Un *type guard* propio se escribe con `parametro is Tipo`:

```ts
function esString(x: unknown): x is string {
  return typeof x === "string";
}
```

El `never` sirve para *exhaustiveness checking*: si agregás un caso nuevo a la unión y no lo manejás, TS te avisa.

**Ejercicios 8**
8.1 Modelá un tipo `EstadoCarga<T>` con tres variantes: `{ tipo: "cargando" }`, `{ tipo: "exito"; datos: T }`, `{ tipo: "fallo"; error: string }`. Escribí una función que reciba ese estado y devuelva un mensaje string para cada caso.
8.2 Agregá *exhaustiveness checking* a la función anterior (que TS falle en compilación si en el futuro agregás una cuarta variante y no la manejás).
8.3 Escribí un type guard `esUsuario(x: unknown): x is Usuario` que verifique en runtime que `x` tiene `id` numérico y `email` string.

---

## Módulo 9 — Async / Promises tipadas

**Teoría.** Una función `async` siempre devuelve `Promise<T>`. Tipás lo que resuelve, no la promesa entera. Combiná con generics y discriminated unions para resultados seguros.

```ts
async function obtenerUsuario(id: number): Promise<Usuario | null> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) return null;
  return (await res.json()) as Usuario;
}
```

Cuidado: `await res.json()` devuelve `any` (o `unknown` en libs modernas); en producción validás con algo como Zod, pero para practicar alcanza con un cast consciente.

**Ejercicios 9**
9.1 Tipá `esperar(ms: number): Promise<void>` que resuelva tras un `setTimeout`.
9.2 Escribí `obtenerProducto(id: number): Promise<Resultado<Producto>>` (usando el `Resultado<T>` del módulo 8) que simule una búsqueda: si `id <= 0`, resuelve con error; si no, con un producto de ejemplo.
9.3 Tipá una función `reintentar<T>(fn: () => Promise<T>, intentos: number): Promise<T>` que reintente `fn` hasta `intentos` veces antes de propagar el error.

---

## Módulo 10 — Clases y modificadores (orientado a backend)

**Teoría.** En backend (sobre todo con NestJS) vas a usar clases con modificadores de acceso (`public`, `private`, `protected`, `readonly`), constructores con *parameter properties*, e implementación de interfaces. La inyección de dependencias se apoya en tipar dependencias por su interfaz.

```ts
interface Logger { log(msg: string): void; }

class UsuarioService {
  // parameter property: declara y asigna this.repo automáticamente
  constructor(
    private readonly repo: Repositorio,
    private readonly logger: Logger,
  ) {}

  obtener(id: number): Producto | undefined {
    this.logger.log(`Buscando ${id}`);
    return this.repo.obtenerPorId(id);
  }
}
```

Tipar la dependencia como `Repositorio` (interfaz) y no como una clase concreta es lo que habilita testear con mocks: el corazón de Clean/Hexagonal.

**Ejercicios 10**
10.1 Creá una clase `CuentaBancaria` con saldo `private`, métodos `depositar(monto)` y `retirar(monto)` (que no permita saldo negativo) y un getter `balance`.
10.2 Definí `interface NotificadorPort { enviar(destino: string, msg: string): Promise<void> }` y una clase `EmailNotificador` que la implemente (simulá el envío con un `console.log` y `Promise.resolve`).
10.3 Escribí una clase `PedidoService` que reciba un `NotificadorPort` por constructor (parameter property) y un método `confirmar(email: string)` que use el notificador. Explicá por qué tipamos contra la interfaz y no contra `EmailNotificador`.

---

## Módulo 11 — Tipos avanzados: mapped, conditional y template literal types

**Teoría.** Nivel "me hace ver senior":

- **Mapped types** — generan tipos iterando claves: `{ [K in keyof T]: ... }`.
- **Conditional types** — `T extends U ? X : Y`.
- **`keyof` / indexed access** — `keyof T`, `T[K]`.
- **Template literal types** — construyen literales: `` `on${Capitalize<E>}` ``.

```ts
type Opcional<T> = { [K in keyof T]?: T[K] };          // como Partial
type SoloLectura<T> = { readonly [K in keyof T]: T[K] }; // como Readonly
type Evento = "click" | "focus";
type Handlers = `on${Capitalize<Evento>}`;             // "onClick" | "onFocus"
type NoNulo<T> = T extends null | undefined ? never : T;
```

No necesitás escribir estos a diario, pero entenderlos te permite leer librerías y tipar APIs flexibles.

**Ejercicios 11**
11.1 Escribí un mapped type `Nullable<T>` que haga cada propiedad `T[K] | null`.
11.2 Dado un `type Rutas = "users" | "products"`, generá con template literals un tipo `Endpoints` igual a `"/api/users" | "/api/products"`.
11.3 Escribí un conditional type `ElementoDeArray<T>` que, si `T` es `U[]`, devuelva `U`, y si no, devuelva `T`. (Pista: `infer`.)

---

# Soluciones

> Mirá esto solo después de intentarlo. Si tu solución difiere pero compila y es type-safe, probablemente también esté bien — hay más de un camino.

### Módulo 1
```ts
// 1.1
let email: string = "a@b.com";   // o sin anotar, se infiere
let precio = 19.99;
let publicado = false;
let borradaEl: Date | undefined; // declarada sin inicializar → conviene anotar

// 1.2
// config.puerto se infiere como `number`. Con `const config = {...}` las props
// internas siguen siendo `number`/`string` (no literales), igual que con `let`.
// any: desactiva el chequeo de tipos (peligroso). unknown: acepta cualquier valor
// pero te obliga a estrechar el tipo antes de usarlo (seguro).

// 1.3
function parseEntrada(raw: unknown): number {
  return typeof raw === "number" ? raw : 0;
}
```

### Módulo 2
```ts
// 2.1
function crearUsuario(nombre: string, rol: string = "user") {
  return { nombre, rol };
}

// 2.2
type Validador = (valor: string) => boolean;
const esEmail: Validador = (v) => /\S+@\S+\.\S+/.test(v);
const noVacio: Validador = (v) => v.trim().length > 0;

// 2.3
function lanzarError(msg: string): never {
  throw new Error(msg);
}
// `never` porque la función NUNCA retorna un valor (corta el flujo lanzando).
// `void` significa "retorna, pero sin valor útil"; acá ni siquiera retorna.
```

### Módulo 3
```ts
// 3.1
interface Producto {
  readonly id: number;
  nombre: string;
  precio: number;
  stock: number;
  descripcion?: string;
}

// 3.2
type ProductoConDescuento = Producto & { descuento: number };

// 3.3
interface Repositorio {
  obtenerPorId(id: number): Producto | undefined;
}
```

### Módulo 4
```ts
// 4.1
type Estado = "pendiente" | "activo" | "cancelado";
function puedeEditar(estado: Estado): boolean {
  return estado === "pendiente" || estado === "activo";
}

// 4.2
function formatear(valor: string | number | Date): string {
  if (typeof valor === "string") return valor;
  if (typeof valor === "number") return valor.toFixed(2);
  return valor.toISOString(); // instanceof Date implícito por descarte
}

// 4.3
// Con literales el compilador valida los valores posibles: un typo como "PETCH"
// es error de compilación, y el autocompletado sugiere solo los métodos válidos.
```

### Módulo 5
```ts
// 5.1
const coordenada: [number, number] = [-34.9, -56.2];

// 5.2
const ESTADOS_PEDIDO = ["pendiente", "pagado", "enviado", "entregado"] as const;
type EstadoPedido = typeof ESTADOS_PEDIDO[number];
// "pendiente" | "pagado" | "enviado" | "entregado"

// 5.3
function primero<T>(arr: T[]): T | undefined {
  return arr[0];
}
```

### Módulo 6
```ts
// 6.1
function ultimo<T>(arr: T[]): T | undefined {
  return arr[arr.length - 1];
}

// 6.2
interface ApiResponse<T> {
  ok: boolean;
  data: T;
  error?: string;
}
const listaUsuarios: ApiResponse<Usuario[]> = { ok: true, data: [] };
const unProducto: ApiResponse<Producto> = {
  ok: true,
  data: { id: 1, nombre: "Mate", precio: 10, stock: 5 },
};

// 6.3
function agruparPor<T, K extends keyof T>(items: T[], clave: K): Record<string, T[]> {
  return items.reduce((acc, item) => {
    const k = String(item[clave]);
    (acc[k] ??= []).push(item);
    return acc;
  }, {} as Record<string, T[]>);
}
```

### Módulo 7
```ts
interface Tarea { id: number; titulo: string; hecha: boolean; creada: Date; }

// 7.1
type CrearTareaDTO = Omit<Tarea, "id" | "creada">;
type ActualizarTareaDTO = Partial<Omit<Tarea, "id">>;
type TareaPreview = Pick<Tarea, "id" | "titulo">;

// 7.2
const contadoresPorEstado: Record<EstadoPedido, number> = {
  pendiente: 0, pagado: 0, enviado: 0, entregado: 0,
};

// 7.3
function crearToken(userId: number, rol: string) {
  return { token: "...", exp: 3600 };
}
type Token = ReturnType<typeof crearToken>; // { token: string; exp: number }
```

### Módulo 8
```ts
// 8.1 y 8.2 (con exhaustiveness checking)
type EstadoCarga<T> =
  | { tipo: "cargando" }
  | { tipo: "exito"; datos: T }
  | { tipo: "fallo"; error: string };

function describir<T>(estado: EstadoCarga<T>): string {
  switch (estado.tipo) {
    case "cargando": return "Cargando...";
    case "exito":    return `Listo: ${JSON.stringify(estado.datos)}`;
    case "fallo":    return `Error: ${estado.error}`;
    default: {
      const _exhaustivo: never = estado; // ❌ compila solo si cubriste todo
      return _exhaustivo;
    }
  }
}

// 8.3
function esUsuario(x: unknown): x is Usuario {
  return (
    typeof x === "object" && x !== null &&
    "id" in x && typeof (x as any).id === "number" &&
    "email" in x && typeof (x as any).email === "string"
  );
}
```

### Módulo 9
```ts
// 9.1
function esperar(ms: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

// 9.2
async function obtenerProducto(id: number): Promise<Resultado<Producto>> {
  if (id <= 0) return { estado: "error", mensaje: "id inválido" };
  return { estado: "ok", data: { id, nombre: "Demo", precio: 9.99, stock: 1 } };
}

// 9.3
async function reintentar<T>(fn: () => Promise<T>, intentos: number): Promise<T> {
  let ultimoError: unknown;
  for (let i = 0; i < intentos; i++) {
    try {
      return await fn();
    } catch (e) {
      ultimoError = e;
    }
  }
  throw ultimoError;
}
```

### Módulo 10
```ts
// 10.1
class CuentaBancaria {
  private saldo = 0;
  depositar(monto: number): void {
    if (monto > 0) this.saldo += monto;
  }
  retirar(monto: number): void {
    if (monto > 0 && monto <= this.saldo) this.saldo -= monto;
    else throw new Error("Fondos insuficientes");
  }
  get balance(): number {
    return this.saldo;
  }
}

// 10.2
interface NotificadorPort {
  enviar(destino: string, msg: string): Promise<void>;
}
class EmailNotificador implements NotificadorPort {
  async enviar(destino: string, msg: string): Promise<void> {
    console.log(`Email a ${destino}: ${msg}`);
  }
}

// 10.3
class PedidoService {
  constructor(private readonly notificador: NotificadorPort) {}
  async confirmar(email: string): Promise<void> {
    await this.notificador.enviar(email, "Tu pedido fue confirmado");
  }
}
// Tipamos contra NotificadorPort (interfaz) y no contra EmailNotificador para
// poder inyectar otra implementación (SMS, push, o un mock en tests) sin tocar
// PedidoService. Es el principio de inversión de dependencias (la "D" de SOLID).
```

### Módulo 11
```ts
// 11.1
type Nullable<T> = { [K in keyof T]: T[K] | null };

// 11.2
type Rutas = "users" | "products";
type Endpoints = `/api/${Rutas}`; // "/api/users" | "/api/products"

// 11.3
type ElementoDeArray<T> = T extends (infer U)[] ? U : T;
// ElementoDeArray<number[]> = number ; ElementoDeArray<string> = string
```

---

## Siguientes pasos

Cuando estos 11 módulos te salgan fluidos, estás listo para NestJS sin fricción: vas a reconocer DI, DTOs, interfaces de repositorio y discriminated unions en cada archivo. Recomendado a continuación: configurar un proyecto con `tsconfig` estricto, sumar **Zod** para validar datos externos en runtime (cierra el hueco del `any` en `JSON.parse`/`res.json()`), y reescribir alguno de tus ejercicios como un mini-endpoint real.

Si querés, puedo armarte la **tanda 2** (ejercicios más difíciles tipo entrevista) o un set enfocado 100% en patrones de NestJS.
