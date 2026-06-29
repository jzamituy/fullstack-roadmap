# GraphQL a fondo: del schema al cliente, con criterio

**Server con NestJS (code-first) + cliente Apollo · de lo básico a lo senior · TypeScript · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y recién después contrastá con la sección final. GraphQL no es "REST pero con un endpoint": es un **lenguaje de consulta tipado** donde el cliente pide exactamente lo que necesita. Acá lo vemos del lado del server (NestJS sobre Apollo) y del cliente (Apollo Client), porque venís de React y ahí está tu ventaja: del otro lado de la query hay una app que ya sabés construir.

**Lo que asumimos.** TypeScript con soltura (interfaces, genéricos, decoradores), [NestJS](nestjs.md) (módulos, providers, DI, guards) y nociones de HTTP y de API REST (para entender qué mejora y qué no). Si venís de React, vas a reconocer el modelo declarativo enseguida.

> **¿Te falta alguna base?** GraphQL en NestJS se apoya en dos cosas. (1) **Decoradores y DI de Nest**: los resolvers son providers inyectables y el schema se genera desde clases decoradas; si eso te suena nuevo, hacé primero [NestJS](nestjs.md) — sin esa base, los módulos 6 y 7 te van a costar el doble. (2) **El problema N+1** se entiende mejor sabiendo qué es una consulta a base de datos: si nunca tocaste el patrón Repository ni una query SQL, pegale una mirada a [PostgreSQL](postgresql.md) antes del módulo 5. Convertir el prerrequisito en rampa, no en muro.

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código — necesitás un proyecto Nest o un playground GraphQL).

**Índice de módulos**
1. Qué es GraphQL y por qué (vs REST): el criterio
2. El schema y el type system (SDL)
3. Queries, mutations y subscriptions
4. Resolvers: la unidad de ejecución
5. El problema N+1 y DataLoader
6. GraphQL en NestJS: code-first
7. Context, autenticación y autorización
8. Manejo de errores
9. Paginación: offset vs cursor (Relay)
10. Performance y seguridad
11. El cliente: Apollo Client
12. Federation y el criterio del "cuándo NO"

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es GraphQL y por qué (vs REST): el criterio

**Teoría.** GraphQL es una **especificación** (no una librería ni un framework) para una API: define un lenguaje de consulta y un sistema de tipos. La idea central: hay **un solo endpoint** (`/graphql`) y el cliente manda una *query* describiendo exactamente qué campos quiere; el server responde con esa forma, ni más ni menos.

El problema que ataca es doble, y lo conocés del frontend:

- **Over-fetching:** REST te devuelve el objeto entero aunque solo quieras el nombre. Pediste `/users/1` para mostrar un avatar y te bajaste 40 campos.
- **Under-fetching (y el N+1 de red):** para armar una pantalla necesitás el usuario, sus proyectos y las tareas de cada proyecto → 3 round-trips (o más) a 3 endpoints REST distintos.

Con GraphQL, una sola query trae el árbol completo:

```graphql
query {
  user(id: "1") {
    name
    projects {
      title
      tasks { title done }
    }
  }
}
```

> **Si venís de React, ya tenés el modelo mental.** Pedir solo los campos que necesitás es el mismo instinto que pasarle a un componente solo las props que usa, no el objeto entero. Y esa misma query, desde el cliente, se ve así con Apollo (lo profundizamos en el módulo 11):
>
> ```tsx
> const { data } = useQuery(GET_USER, { variables: { id: "1" } });
> ```
>
> El query que diseñás en el server es, literalmente, el que pedís desde el componente. Ese es el arco cliente-server que vamos a cerrar.

**Pero ojo —y esto es criterio senior—:** GraphQL no es gratis. Trae complejidad real (N+1 en el server, caché más difícil que REST, seguridad de queries arbitrarias). No es "mejor que REST"; es **distinto**, y brilla cuando tenés muchos clientes con necesidades de datos distintas (web + mobile + terceros) o grafos de datos muy relacionados. Para un CRUD interno con un solo cliente, REST suele ser más simple. El módulo 12 vuelve sobre esto.

**Ejercicios 1**
1.1 🔁 En una frase, ¿qué resuelve GraphQL que REST hace incómodo, y a costa de qué?
1.2 🧠 Tenés una API interna, un solo cliente web, CRUD clásico. ¿GraphQL o REST? Justificá en dos frases.
1.3 🧠 "GraphQL tiene un solo endpoint, así que es más simple que REST." Matizá esa afirmación: ¿en qué es más simple y en qué es más complejo?

---

## Módulo 2 — El schema y el type system (SDL)

**Teoría.** El **schema** es el contrato de tu API: define qué tipos existen y qué operaciones se pueden hacer. Se puede escribir en **SDL** (Schema Definition Language) o generarse desde código (módulo 6). En SDL:

```graphql
scalar DateTime               # scalars custom además de Int, Float, String, Boolean, ID

type User {
  id: ID!                     # ! = non-null (siempre viene)
  name: String!
  email: String!
  projects: [Project!]!       # lista non-null de Projects non-null
}

type Project {
  id: ID!
  title: String!
  status: ProjectStatus!
  owner: User!
}

enum ProjectStatus { ACTIVE ARCHIVED }

input CreateProjectInput {     # los inputs son tipos APARTE de los outputs
  title: String!
  ownerId: ID!
}

type Query {                   # punto de entrada de lecturas
  user(id: ID!): User
  projects: [Project!]!
}

type Mutation {                # punto de entrada de escrituras
  createProject(input: CreateProjectInput!): Project!
}
```

Si venís de React, pensá el schema como los **tipos de props de toda tu API**: un contrato que declara qué forma tiene cada cosa y qué es obligatorio. Dos sutilezas que importan:

- **Non-null (`!`) es parte del contrato.** Elegir bien los `!` es diseño de API, no decoración.
- **Input types ≠ Object types.** No podés usar un `type` como argumento de entrada; para eso existen los `input`. Es a propósito: lo que entra y lo que sale tienen formas distintas (un input no tiene `id`, igual que un DTO de creación).

> **Modelo mental: los dos `!` de una lista.** `[Project!]!` tiene un `!` afuera y otro adentro, y hablan de cosas distintas. El de **afuera** es sobre la lista (¿puede ser `null` la lista entera?); el de **adentro**, sobre cada item (¿puede haber un `null` adentro?).
> - `[Project!]!` → la lista existe siempre y no tiene nulls. El caso normal (vacío es `[]`, no `null`).
> - `[Project!]` → la lista puede ser `null`, pero si existe no tiene nulls. Distingue "todavía no se cargó" de "está vacía".
> - `[Project]` → la lista y sus items pueden ser `null`. Raro; útil en una operación batch donde cada posición es "resultado o null".

**Avanzado — interfaces y unions** (podés saltearlo en una primera pasada y volver cuando lo necesites): modelan polimorfismo. Una `interface` comparte campos entre tipos; una `union` dice "esto es A o B" sin campos comunes.

**Ejercicios 2**
2.1 ✍️ Escribí en SDL un type `Task` con `id` (ID!), `title` (String!), `done` (Boolean!) y `assignee` (User, opcional). Sumá la query `tasks: [Task!]!`.
2.2 🧠 ¿Qué diferencia hay entre `tasks: [Task!]!`, `tasks: [Task!]` y `tasks: [Task]`? Dá un caso donde elegirías cada uno.
2.3 🧠 ¿Por qué GraphQL te obliga a declarar `input CreateTaskInput` en vez de reusar el `type Task` como argumento?

---

## Módulo 3 — Queries, mutations y subscriptions

**Teoría.** Hay tres tipos de operación raíz:

- **Query** — lecturas. Se ejecutan en paralelo (los campos no dependen entre sí).
- **Mutation** — escrituras. Se ejecutan en **serie**, en orden, para que `crear` y después `borrar` no compitan.
- **Subscription** — un stream en tiempo real (sobre WebSocket): el server empuja datos cuando pasa algo.

Del lado del cliente, una operación puede tener **variables** (parámetros tipados) en vez de valores hardcodeados:

```graphql
query GetUser($id: ID!) {        # $id es una variable tipada
  user(id: $id) {
    name
    projects { title }
  }
}
```

```jsonc
// variables que mandás aparte de la query
{ "id": "1" }
```

Otras piezas del lenguaje: **alias** (renombrar un campo en la respuesta), **fragments** (reutilizar un set de campos), y **directivas** como `@include(if:)` / `@skip(if:)` para campos condicionales.

Una subscription se declara en el schema como cualquier operación raíz:

```graphql
type Subscription {
  taskAdded: Task!        # el server empuja un Task cada vez que se crea uno
}
```

Las **subscriptions** en 2026 usan la librería **`graphql-ws`** sobre WebSocket (el viejo `subscriptions-transport-ws` quedó deprecado y sin mantenimiento — si ves tutoriales con ese, están viejos). Ojo con un detalle que confunde al configurar: la librería es `graphql-ws`, pero el **sub-protocolo** WebSocket que negocian cliente y server se llama **`graphql-transport-ws`** (vas a tener que nombrarlo en la config de ambos lados). También existe transporte sobre **HTTP/SSE** (`graphql-sse`) para cuando no querés mantener un socket abierto, pero WebSocket sigue siendo lo estándar.

**Ejercicios 3**
3.1 🔁 ¿Por qué las mutations se ejecutan en serie y las queries en paralelo?
3.2 ✍️ Escribí una query `GetProject($id: ID!)` que traiga el `title`, el `status` y el `name` del `owner`.
3.3 🧠 ¿Para qué sirve un *fragment* y qué problema de mantenimiento evita cuando tenés 5 componentes que muestran un `User`?
3.4 🧠 ¿Cuándo usarías una `subscription` y cuándo te alcanza con polling/refetch desde el cliente? Dá un caso de cada uno.

---

## Módulo 4 — Resolvers: la unidad de ejecución

**Teoría.** Un **resolver** es la función que produce el valor de un campo. GraphQL resuelve una query recorriendo el árbol: ejecuta el resolver de `user`, y con ese resultado ejecuta el resolver de `user.projects`, y así. Cada resolver recibe cuatro argumentos (firma **conceptual**, para fijar el modelo mental — el tipo real exportado por `graphql` es `GraphQLFieldResolver<TSource, TContext, TArgs>` y NestJS te abstrae todo esto con decoradores, módulo 6):

```ts
type Resolver<TParent, TArgs, TContext, TResult> = (
  parent: TParent,    // el resultado del resolver padre (el User, para resolver projects)
  args: TArgs,        // los argumentos del campo (ej. { id })
  context: TContext,  // estado compartido del request (user autenticado, dataloaders, db)
  info: GraphQLResolveInfo, // metadata de la query (qué campos se pidieron, etc.)
) => TResult | Promise<TResult>;
```

La clave mental: **pensá cada resolver como un componente de React.** Recibe `parent` (como las props que le bajan de arriba) y decide qué resolver; el árbol de resolvers es el árbol de componentes. De ahí lo importante: **`projects` no es un dato guardado en el `User`; es un campo con su propio resolver.** Por eso GraphQL puede componer datos de fuentes distintas (el `User` viene de Postgres, sus `projects` de otro servicio) sin que el cliente se entere. Esa **resolución por campo** es la fuente del poder de GraphQL... y también del problema N+1 (módulo siguiente).

Un resolver que no escribís usa el **default resolver**: si el `parent` ya tiene una propiedad con el nombre del campo, la devuelve. Por eso `user.name` no necesita resolver propio: ya viene en el objeto.

**Ejercicios 4**
4.1 🔁 ¿Cuáles son los cuatro argumentos de un resolver y para qué sirve `parent`?
4.2 🧠 ¿Por qué `User.projects` necesita un resolver propio pero `User.name` no?
4.3 🧠 Si una query pide `user { projects { owner { name } } }`, ¿en qué orden se ejecutan los resolvers y qué es el `parent` de `owner`?

---

## Módulo 5 — El problema N+1 y DataLoader

**Teoría.** Este es **el** tema que separa a quien "usó GraphQL" de quien lo entiende. Mirá esta query:

```graphql
query { projects { title owner { name } } }
```

Si tenés 100 proyectos, el resolver de `projects` hace 1 query a la base (`SELECT * FROM projects`). Después, GraphQL ejecuta el resolver de `owner` **una vez por cada proyecto** → 100 queries más (`SELECT * FROM users WHERE id = ?`). Total: **1 + N = 101 queries**. Eso es el **N+1**, y mata la performance.

La solución estándar es **DataLoader** (de la librería `dataloader`). Hace dos cosas:

1. **Batching:** junta todos los `load(id)` de un mismo *tick* del event loop y los resuelve en **una sola** llamada batch (`SELECT * FROM users WHERE id IN (...)`).
2. **Caching por request:** si pedís el mismo `id` dos veces en la misma query, lo trae una sola vez.

```ts
import DataLoader from "dataloader";

// La batch function recibe TODOS los ids juntos y devuelve los resultados EN EL MISMO ORDEN
function createUserLoader(repo: UserRepo): DataLoader<string, User> {
  return new DataLoader<string, User>(async (ids: readonly string[]) => {
    const users = await repo.findByIds([...ids]);          // 1 sola query
    const byId = new Map(users.map((u) => [u.id, u]));
    return ids.map((id) => byId.get(id) ?? new Error(`User ${id} no existe`));
  });
}
```

El resolver de `owner` pasa a ser `context.userLoader.load(project.ownerId)` en vez de `repo.findById(...)`. Con 100 proyectos: **1 + 1 = 2 queries.**

> **Regla de oro:** un DataLoader vive **por request**, no global. Si lo compartís entre requests, el caching te sirve datos viejos a otro usuario. En NestJS se crea en el `context` de cada request (módulo 7).

**Ejercicios 5**
5.1 🔁 Explicá el problema N+1 con tus palabras, usando un ejemplo de proyectos y owners.
5.2 🧠 ¿Por qué la batch function de DataLoader DEBE devolver los resultados en el mismo orden que los ids que recibió?
5.3 ✍️ Escribí una batch function `batchProjectsByOwner(ownerIds)` que, dada una lista de `ownerId`, devuelva por cada uno **su array de proyectos** (pista: agrupás los proyectos por `ownerId` y mapeás en orden).
5.4 🧠 ¿Por qué un DataLoader nunca debe ser un singleton compartido entre requests?

---

## Módulo 6 — GraphQL en NestJS: code-first

**Teoría.** NestJS integra GraphQL vía `@nestjs/graphql` + `@nestjs/apollo`, usando el `ApolloDriver`. Hoy corre sobre **Apollo Server 5**: `@nestjs/apollo` v13+ ya pide `@apollo/server` v5 (Apollo Server 4 quedó **EOL en enero de 2026**, y ya no se instala el viejo `apollo-server-express`). Hay dos enfoques:

> ⚠️ **Gotcha al migrar a AS5:** el viejo plugin del Playground (`@apollo/server-plugin-landing-page-graphql-playground`) quedó deprecado y da warnings de peer-dep con Apollo Server 5. En AS5 usás la landing page nativa (`ApolloServerPluginLandingPageLocalDefault` en dev, o la apagás en prod). Si seguís un tutorial viejo que instala el plugin de Playground, ese es el ruido que vas a ver.

- **Schema-first:** escribís el SDL a mano y Nest genera los tipos TS.
- **Code-first** (el recomendado en proyectos TS): escribís **clases decoradas** y Nest **genera el SDL** automáticamente. Una sola fuente de verdad, todo tipado.

```ts
import { ObjectType, Field, ID, Resolver, Query, Mutation, Args, InputType } from "@nestjs/graphql";

@ObjectType()
export class Project {
  @Field(() => ID) id!: string;
  @Field() title!: string;
  @Field(() => User) owner!: User;   // el ! de TS; el non-null de GraphQL es el default
}

@InputType()
export class CreateProjectInput {
  @Field() title!: string;
  @Field(() => ID) ownerId!: string;
}

@Resolver(() => Project)
export class ProjectsResolver {
  constructor(private readonly projects: ProjectsService) {}

  @Query(() => [Project])
  allProjects(): Promise<Project[]> {
    return this.projects.findAll();
  }

  @Mutation(() => Project)
  createProject(@Args("input") input: CreateProjectInput): Promise<Project> {
    return this.projects.create(input);
  }
}
```

Para el campo `owner` (que necesita resolverse aparte, módulo 4) usás un **`@ResolveField`**:

```ts
@ResolveField(() => User)
owner(@Parent() project: Project, @Context() ctx: GqlContext): Promise<User> {
  return ctx.userLoader.load(project.ownerId);   // ← DataLoader, no N+1
}
```

> **Gotcha de nullability:** en code-first, los campos son **non-null por defecto**. Para hacer uno opcional, marcás `@Field(() => User, { nullable: true })`. No alcanza con poner `?` en TS — el decorador es el que define el schema.

**Ejercicios 6**
6.1 🔁 ¿Qué genera Nest en code-first a partir de tus clases decoradas?
6.2 ✍️ Escribí un `@ObjectType() Task` con `id`, `title`, `done` y un `@Resolver` con una query `tasks(): Task[]`.
6.3 🧠 En code-first, ¿por qué `title?: string` en TS no alcanza para que el campo sea opcional en el schema GraphQL? ¿Qué tenés que hacer?

---

## Módulo 7 — Context, autenticación y autorización

**Teoría.** El **context** es un objeto que se construye **una vez por request** y llega a todos los resolvers. Es donde viven: el usuario autenticado, los DataLoaders y cualquier cosa con scope de request.

```ts
// en GraphQLModule.forRoot
context: ({ req }) => ({
  req,
  userLoader: createUserLoader(userRepo),   // un loader nuevo por request (módulo 5)
}),
```

**Auth.** GraphQL no tiene "rutas" donde colgar un middleware por endpoint, así que la autenticación se hace distinto que en REST:

- **Autenticación** (¿quién sos?): un **guard** que lee el token del `req` y pone el `user` en el context. En Nest usás un guard adaptado a GraphQL (`GqlExecutionContext.create(ctx).getContext().req`).
- **Autorización** (¿podés hacer esto?): puede ser a nivel **operación** (un guard sobre la query/mutation completa) o a nivel **campo** (un resolver que chequea antes de devolver un campo sensible como `email`).

```ts
@Injectable()
export class GqlAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context).getContext();
    return Boolean(ctx.req.user);   // poblado por la capa de autenticación
  }
}
```

> **Diferencia clave con REST:** en REST protegés *endpoints*; en GraphQL, como una sola query puede tocar muchos tipos, la autorización suele vivir **más cerca del dato** (en el resolver o el ResolveField), no en una sola puerta de entrada.

**Ejercicios 7**
7.1 🔁 ¿Cuántas veces se construye el context y por qué eso importa para los DataLoaders?
7.2 🧠 ¿Por qué la autorización en GraphQL tiende a estar a nivel campo/resolver y no en una sola "puerta" como en REST?
7.3 ✍️ Escribí un guard `GqlAuthGuard` que rechace si no hay `user` en el context (usá `GqlExecutionContext`).

---

## Módulo 8 — Manejo de errores

**Teoría.** GraphQL casi siempre responde **HTTP 200**, incluso con errores: el status va dentro del body, en un array `errors` junto a `data`. Esto sorprende a quien viene de REST.

```jsonc
{
  "data": { "user": null },
  "errors": [
    {
      "message": "No autorizado",
      "path": ["user"],
      "extensions": { "code": "UNAUTHENTICATED" }   // tu código de error va acá
    }
  ]
}
```

Conceptos:

- **Errores parciales:** una query puede devolver `data` para los campos que salieron bien y `errors` para los que fallaron. Si un campo non-null falla, el error "burbujea" hasta el primer ancestro nullable.
- **`extensions.code`:** acá va tu taxonomía (`UNAUTHENTICATED`, `FORBIDDEN`, `NOT_FOUND`, `BAD_USER_INPUT`). El cliente discrimina por `code`, no por el `message` (que es para humanos).
- En NestJS tirás un `GraphQLError` (o una excepción que un *filter* mapea), no un `HttpException` con status.

> **Regla:** no filtres detalles internos (stack traces, SQL) en producción. Apollo lo hace por defecto en prod, pero verificalo. Los errores esperables (validación, permisos) llevan `code`; los inesperados, un `INTERNAL_SERVER_ERROR` genérico.

**Ejercicios 8**
8.1 🔁 ¿Por qué una respuesta GraphQL con errores suele venir con HTTP 200?
8.2 🧠 ¿Por qué el cliente debería ramificar por `extensions.code` y no por el `message`?
8.3 🧠 Una query pide `user { name email }` donde `user: User` es **nullable** (como en el módulo 2) y `email: String!` es non-null. `email` falla por permisos. ¿Qué le llega al cliente en `data`, y por qué? Razoná el burbujeo del null y qué pasa con `name` y con `user`. Como extensión: ¿qué cambiaría si `user` también fuera non-null?

---

## Módulo 9 — Paginación: offset vs cursor (Relay)

**Teoría.** Devolver `[Project!]!` entero no escala. Hay dos estrategias:

- **Offset/limit** (`page`, `pageSize`): simple, pero se rompe si insertan/borran filas mientras paginás (te saltás o repetís items), y es lenta en offsets grandes.
- **Cursor-based** (la recomendada): cada item tiene un **cursor** opaco (típicamente su id o un campo ordenable codificado). Pedís "los N después de este cursor". Estable ante inserciones y eficiente.

El estándar de facto es el de **Relay Connections**:

```graphql
type ProjectConnection {
  edges: [ProjectEdge!]!
  pageInfo: PageInfo!
}
type ProjectEdge {
  node: Project!
  cursor: String!
}
type PageInfo {
  hasNextPage: Boolean!
  endCursor: String
}

type Query {
  projects(first: Int!, after: String): ProjectConnection!
}
```

Parece verboso, pero te da un formato consistente que los clientes (Apollo, Relay) saben paginar solos con caché incremental.

> **El cursor es opaco a propósito.** Para el cliente es una cadena sin significado (típicamente el `id` o la sort key codificados en base64); lo recibe en `endCursor` y lo pasa tal cual en el próximo `after`, **nunca lo interpreta ni lo construye**. Esa opacidad es justo lo que te deja cambiar la implementación interna (de id a un campo compuesto, por ejemplo) sin romper a nadie — y evita el antipatrón de tratar el cursor como un offset disfrazado.

**Ejercicios 9**
9.1 🧠 ¿Por qué la paginación por offset se "rompe" si se insertan filas mientras un usuario pagina? Dá el escenario concreto.
9.2 🔁 En el patrón Relay, ¿qué es un `edge`, qué es un `node` y qué es el `cursor`?
9.3 🧠 ¿Cuándo aceptarías offset/limit en vez de cursor a pesar de sus problemas?

---

## Módulo 10 — Performance y seguridad

**Teoría.** Acá GraphQL paga el precio de su flexibilidad: como el cliente arma queries arbitrarias, **un atacante también**. Te conviene ordenar las defensas en dos grupos.

**1) Limitar lo que ENTRA** (controlar la query *antes* de ejecutarla):

- **Profundidad (depth limiting):** una query maliciosa anida `user { projects { owner { projects { owner { ... } } } } }` para colgar el server. Cortás a una profundidad máxima. El paquete clásico es `graphql-depth-limit`, pero ojo: está sin mantener desde 2022 y necesita `@types/graphql-depth-limit` aparte. Hoy, para *solo* limitar profundidad, el sucesor mantenido es **`@graphile/depth-limit`**; y si querés algo más completo, `graphql-query-complexity`, reglas de validación propias o **GraphQL Armor** (que empaqueta depth + complexity + más defensas).
- **Complejidad (query complexity):** asignás un costo a cada campo y rechazás queries que superan un presupuesto (`graphql-query-complexity`). Es más fino que la profundidad sola.
- **Persisted queries / allow-list:** el cliente solo manda un **hash** de queries pre-aprobadas; el server rechaza cualquier query ad-hoc. Mata la superficie de ataque.
- **Introspección apagada en prod** (o restringida): la introspección expone tu schema entero. Útil en dev, riesgosa abierta al público.

**2) Limitar lo que CUESTA en runtime** (controlar la *ejecución*):

- **DataLoader** (módulo 5): evita que una query con muchos campos relacionados tumbe la base con N+1.
- **Timeouts y rate limiting** por operación, como en cualquier API.

> **Criterio:** en REST, cada endpoint tiene un costo conocido. En GraphQL, el costo lo define el cliente en runtime. Por eso depth + complexity limiting **no son opcionales** en una API pública. Es la primera pregunta de seguridad que te van a hacer en una entrevista senior sobre GraphQL.

**Ejercicios 10**
10.1 🔁 Nombrá dos ataques específicos de GraphQL que no existen igual en REST.
10.2 🧠 ¿Por qué limitar solo la profundidad no alcanza, y qué agrega el límite de complejidad? Dá un ejemplo de query poco profunda pero costosa.
10.3 🧠 ¿Qué ganás y qué perdés al apagar la introspección en producción?

---

## Módulo 11 — El cliente: Apollo Client

**Teoría.** Acá juega tu experiencia de React. **Apollo Client** es el cliente más usado; su superpoder es el **caché normalizado**: guarda cada objeto por su `__typename` + `id` en un store plano, así dos queries que traen el mismo `User` comparten una sola copia. Actualizar ese `User` refresca toda la UI que lo usa.

> **No es la única opción** (mismo espíritu "cuándo NO" del módulo 12): **urql** es más liviano y modular; y si NO necesitás caché normalizado, `graphql-request` + **TanStack Query** te da fetching y caché por-query con mucho menos peso. Apollo gana cuando el caché normalizado (entidades compartidas que se actualizan solas en toda la UI) te paga su tamaño.

> **Si normalizaste estado en Redux, ya lo conocés.** Es la misma idea que predica la doc de Redux/RTK: en vez de guardar el árbol anidado, guardás cada entidad **una sola vez** indexada por su id (`User:1`) y el resto referencia. Frase mental: **el caché de Apollo es una mini base de datos en memoria, indexada por `__typename:id`.** Por eso necesita el `id`: es la clave primaria de esa base.

```tsx
const GET_PROJECTS = gql`
  query GetProjects {
    allProjects { id title owner { id name } }
  }
`;

function Projects() {
  const { data, loading, error } = useQuery(GET_PROJECTS);
  if (loading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <List items={data.allProjects} />;
}
```

Conceptos que importan:

- **Normalización por `id`:** por eso tus types necesitan `id`. Sin `id`, Apollo no puede normalizar y termina cacheando por query (peor). Configurás `typePolicies` para casos especiales.
- **Fetch policies:** `cache-first` (default), `cache-and-network`, `network-only`, etc. Controlan cuándo confiar en el caché vs ir al server.
- **Mutations y actualización de caché:** tras una mutation, o refetcheás queries, o actualizás el caché a mano (`update`), o devolvés el objeto modificado **con su `id`** y Apollo lo mergea solo.

> **Conexión con REST:** acá se ve por qué el caché de GraphQL es "más difícil" que el de REST. REST cachea por URL (HTTP caching gratis). GraphQL, con un solo endpoint POST, no puede usar ese caché HTTP por defecto, así que la caché es **trabajo del cliente** (Apollo). Pero hay una vuelta: con **APQ (Automatic Persisted Queries) vía GET**, el query viaja como un hash en la URL y **sí** se vuelve cacheable por CDN — por eso las persisted queries no son solo seguridad (módulo 10), también recuperan parte de la caché HTTP que el POST tira a la basura. Igual: es un trade-off, no una ventaja pura.

**Ejercicios 11**
11.1 🔁 ¿Por qué Apollo Client necesita el campo `id` (o una key) para cachear bien?
11.2 🧠 Hacés una mutation que cambia el `title` de un proyecto. ¿Cuáles son las tres formas de que la UI se entere, y cuál preferís cuándo?
11.3 🧠 ¿Por qué el HTTP caching "gratis" de REST no aplica en GraphQL, y qué lo reemplaza?

---

## Módulo 12 — Federation y el criterio del "cuándo NO"

**Teoría.** **Federation** (Apollo Federation 2) resuelve un problema de escala organizacional: en vez de un schema monolítico gigante, varios equipos exponen **subgraphs** (cada uno su pedazo del grafo) y un **gateway** los compone en un supergraph. Un type `User` puede tener campos definidos en el subgraph de Identidad y campos extendidos en el de Facturación. Es la respuesta de GraphQL a los microservicios.

No lo necesitás hasta que tengas varios equipos/servicios; meterlo antes es **over-engineering**. (Federation es un tema avanzado: si recién arrancás, alcanza con saber que existe y para qué.)

Dicho eso, llegamos a **lo más importante de todo el módulo** — el criterio que de verdad te van a evaluar:

### ¿Cuándo NO usar GraphQL?

| Contexto | Mejor opción | Por qué |
|---|---|---|
| Un solo cliente, CRUD interno | **REST** | GraphQL agrega complejidad (N+1, caché, seguridad) sin pagar su beneficio |
| Frontend y backend TS en un monorepo, un equipo | **tRPC** | Type-safety end-to-end sin escribir schema ni resolvers; más liviano |
| Subir/bajar archivos, streaming binario | **REST** | GraphQL es malo para binarios; lo resolvés con endpoints aparte |
| Caché HTTP/CDN central a tu performance | **REST** | El caché por URL es gratis y potente |
| Muchos clientes con necesidades distintas, grafo muy relacionado | **GraphQL** | Acá sí brilla: cada cliente pide lo suyo |
| Muchos servicios/equipos, un grafo unificado | **GraphQL Federation** | Compone el grafo sin un monolito |

> **La marca de criterio senior** no es saber GraphQL: es saber que GraphQL es una herramienta con costos concretos y elegir REST o tRPC sin complejo de inferioridad cuando son lo correcto. Coleccionar tecnología no es arquitectura.

**Ejercicios 12**
12.1 🔁 ¿Qué problema resuelve Federation y cuándo es prematuro meterlo?
12.2 🧠 Backend y frontend en TypeScript, un equipo, un monorepo. Te piden type-safety end-to-end. ¿GraphQL o tRPC? Justificá.
12.3 🧠 Dá dos escenarios concretos donde elegirías REST sobre GraphQL y explicá el porqué de cada uno.

---

# Soluciones

> Mirá esto solo después de intentarlo. Si tu respuesta difiere pero es correcta y razonada, probablemente también esté bien — varias de estas son de criterio, no de una sola respuesta.

### Módulo 1
**1.1** GraphQL resuelve el over-fetching y el under-fetching (el cliente pide exactamente los campos que necesita, en una sola query que arma el árbol de datos), a costa de mayor complejidad en el server (N+1, caché y seguridad de queries arbitrarias).

**1.2** REST. Con un solo cliente y CRUD clásico, no hay variedad de necesidades de datos que justifique GraphQL; REST es más simple, te da caché HTTP gratis y costo por endpoint conocido. GraphQL ahí solo agrega problemas.

**1.3** Más simple en la **superficie de red** (un endpoint, un contrato tipado, el cliente arma su query). Más complejo en el **server y la operación**: tenés que resolver N+1 con DataLoaders, la caché es trabajo tuyo (no HTTP gratis), y la seguridad exige limitar profundidad/complejidad porque el cliente arma queries arbitrarias.

### Módulo 2
```graphql
# 2.1
type Task {
  id: ID!
  title: String!
  done: Boolean!
  assignee: User
}

type Query {
  tasks: [Task!]!
}
```
**2.2** `[Task!]!`: la lista siempre existe y nunca tiene nulls → el caso normal de "traeme las tareas" (vacío es `[]`, no null). `[Task!]`: la lista puede ser null pero sus elementos no → "todavía no se cargó" vs "está vacía". `[Task]`: la lista puede ser null y contener nulls → raro; útil si una operación batch devuelve "resultado o null por posición" (ej. un loader que falla en algunos ids).

**2.3** Porque los object types pueden tener campos que no tienen sentido como entrada (un `id` que genera el server, campos calculados, relaciones). Separar `input` de `type` es lo mismo que separar un DTO de creación de la entidad: lo que entra y lo que sale tienen formas distintas, y mezclarlos te obliga a aceptar campos inválidos o nullables de más.

### Módulo 3
**3.1** Las queries no se modifican entre sí: leer A y leer B en cualquier orden da lo mismo, así que se paralelizan. Las mutations escriben estado; si `crearProyecto` y `borrarProyecto` corrieran en paralelo, el resultado dependería del timing (race condition). Ejecutarlas en serie y en orden hace el resultado determinístico.
```graphql
# 3.2
query GetProject($id: ID!) {
  project(id: $id) {
    title
    status
    owner { name }
  }
}
```
**3.3** Un fragment es un set de campos reutilizable (`fragment UserCard on User { id name email }`). Si 5 componentes muestran un `User` y mañana agregás `avatarUrl`, sin fragment tocás 5 queries; con fragment, tocás una y los 5 se actualizan. Evita la divergencia entre componentes que muestran lo mismo.

**3.4** **Subscription** cuando necesitás *push* del server con baja latencia y eventos que no controlás: un chat, notificaciones en vivo, el estado de un pedido que cambia desde otro actor, un dashboard de cotizaciones. **Polling/refetch** cuando el dato cambia poco o tolerás algo de retraso, o cuando no querés el costo de mantener un WebSocket abierto por cliente: un listado que refrescás cada 30s, un "tirá para actualizar". Regla práctica: la subscription paga su complejidad (infra de sockets, reconexión, escalado) solo si el tiempo real es parte del producto; si no, el refetch es más simple y barato.

### Módulo 4
**4.1** `parent` (el resultado del resolver padre), `args` (los argumentos del campo), `context` (estado compartido del request) e `info` (metadata de la query). `parent` es lo que permite la resolución encadenada: el resolver de `projects` recibe el `User` ya resuelto y con su `id` busca los proyectos.

**4.2** `name` ya viene como propiedad del objeto `User` que devolvió el resolver padre, así que el **default resolver** lo lee directo. `projects` no es un dato del `User`: es una relación que hay que ir a buscar (otra tabla, otro servicio), por eso necesita un resolver propio que use el `id` del parent.

**4.3** Orden: `user` → `projects` → `owner`. Primero se resuelve `user`; con ese resultado se resuelve `user.projects` (devuelve N proyectos); por cada proyecto se resuelve `owner`, y el `parent` de `owner` es **cada Project individual** (de donde sale el `ownerId`). Justamente acá aparece el N+1.

### Módulo 5
**5.1** Resolvés `projects` con 1 query que trae 100 proyectos. Después, GraphQL ejecuta el resolver de `owner` una vez por proyecto: 100 queries `SELECT ... WHERE id = ?`. Total 101 (1 + N) para algo que deberían ser 2. Cuantos más proyectos, peor.

**5.2** Porque DataLoader devuelve los resultados **por posición**: el `load(idX)` que hiciste se resuelve con el elemento que está en la misma posición que `idX` en el array de entrada. Si la batch function devuelve en otro orden (ej. el que vino de la base), cada `load` recibe el dato equivocado. Por eso se mapea contra los ids originales con un `Map`.
La versión **canónica** (la que usarías en producción): la batch function va a la base **una sola vez** con todos los `ownerId`, y después agrupa el resultado por `ownerId` para devolver, en orden, el array de cada uno.

```ts
// 5.3 — la que usás de verdad: 1 query para todos los owners del batch
async function batchProjectsByOwner(
  ownerIds: readonly string[],
  repo: ProjectRepo,
): Promise<Project[][]> {
  const projects = await repo.findByOwnerIds([...ownerIds]);   // 1 sola query (WHERE owner_id IN (...))
  const byOwner = new Map<string, Project[]>();
  for (const p of projects) {
    const arr = byOwner.get(p.ownerId) ?? [];
    arr.push(p);
    byOwner.set(p.ownerId, arr);
  }
  return ownerIds.map((id) => byOwner.get(id) ?? []);   // EN ORDEN; [] si el owner no tiene proyectos
}

// se enchufa al loader así:
// new DataLoader<string, Project[]>((ownerIds) => batchProjectsByOwner(ownerIds, repo))
```

Lo importante es el **agrupado en orden**: devolvés un array por cada `ownerId` recibido, en la misma posición (igual que con un loader de a uno, módulo 5.2), y `[]` —no `null`— para el owner sin proyectos. (Si los proyectos ya estuvieran cargados en memoria, el mismo agrupado aplica sin el `await`; pero el sentido de DataLoader es justamente reemplazar las N queries por **una**, así que la versión real va a la base.)
**5.4** Porque el caching de DataLoader es por-request a propósito: cachea el `User 1` que cargaste. Si el loader fuera global, ese `User 1` quedaría cacheado entre requests y le servirías datos potencialmente viejos —o de otro tenant— a un usuario distinto. El loader se crea fresco en el context de cada request (módulo 7).

### Módulo 6
**6.1** Genera el **schema GraphQL (SDL)** completo a partir de tus clases decoradas (`@ObjectType`, `@Field`, `@Query`, etc.), y conecta esos campos con tus resolvers. Una sola fuente de verdad en TS; no escribís SDL a mano.
```ts
// 6.2
@ObjectType()
export class Task {
  @Field(() => ID) id!: string;
  @Field() title!: string;
  @Field() done!: boolean;
}

@Resolver(() => Task)
export class TasksResolver {
  constructor(private readonly tasks: TasksService) {}

  @Query(() => [Task])
  tasks(): Promise<Task[]> {
    return this.tasks.findAll();
  }
}
```
**6.3** Porque el schema GraphQL no se genera del tipo TS, sino del **decorador** `@Field`. El `?` de TS solo afecta el chequeo de tipos de TypeScript; el SDL lo define `@Field`. Para hacer el campo opcional en GraphQL tenés que pasar `@Field(() => String, { nullable: true })` (y ahí sí, además, marcar `title?` en TS para que sean coherentes).

### Módulo 7
**7.1** Una vez por request. Importa porque ahí creás los DataLoaders: un loader nuevo por request garantiza que el caché del loader no cruce datos entre usuarios distintos (módulo 5).

**7.2** Porque una sola query GraphQL puede tocar muchos tipos y campos en un árbol, no un único "endpoint". No hay una puerta donde colgar el chequeo. Como cada campo se resuelve aparte, el lugar natural para preguntar "¿este usuario puede ver este dato?" es el resolver del dato (o el ResolveField), cerca de donde el dato realmente se produce.
```ts
// 7.3
@Injectable()
export class GqlAuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const ctx = GqlExecutionContext.create(context).getContext();
    if (!ctx.req?.user) {
      throw new GraphQLError("No autenticado", {
        extensions: { code: "UNAUTHENTICATED" },
      });
    }
    return true;
  }
}
```

### Módulo 8
**8.1** Porque a nivel transporte la request HTTP se procesó bien (el server entendió y respondió); los errores de la operación GraphQL son parte del *body* (el array `errors`), no del status HTTP. El status se reserva para fallos de transporte (400 por query malformada, 500 por crash del server). Esto permite **resultados parciales**: `data` para lo que salió bien y `errors` para lo que falló, en la misma respuesta 200.

**8.2** Porque el `message` es texto para humanos: puede cambiar, traducirse o reformularse sin aviso. El `extensions.code` es un contrato estable (`UNAUTHENTICATED`, `FORBIDDEN`, etc.) pensado para que el cliente ramifique lógica (redirigir al login, mostrar "sin permiso", reintentar). Acoplar lógica al `message` es frágil.

**8.3** `email` es non-null y falla → su valor no puede ser null, así que el error **burbujea** al primer ancestro nullable. `name` se resolvió bien, pero como `user` (si es nullable) termina siendo `null` por el burbujeo del `email` fallido, en la respuesta `user` viene `null` y el `errors` describe el fallo en `path: ["user","email"]`. Si `user` también fuera non-null, el null seguiría burbujeando hacia arriba hasta el primer campo nullable (eventualmente la raíz).

### Módulo 9
**9.1** Pedís página 1 (items 1–10, offset 0). Antes de pedir página 2 (offset 10) se insertan 3 filas nuevas al principio. Ahora el "offset 10" apunta a items que en tu página 1 estaban en posiciones 7–10: los **volvés a ver** (duplicados). Si en cambio se borran filas, te **saltás** items. El offset es una posición absoluta en una lista que se movió.

**9.2** Un `edge` es el "envoltorio" de un item en la conexión: contiene el `node` (el dato real, ej. el `Project`) y el `cursor` (la posición opaca de ese item, para pedir "lo que sigue después de acá"). La conexión agrupa los edges + un `pageInfo` con `hasNextPage`/`endCursor`.

**9.3** Cuando necesitás saltar a una página arbitraria por número ("ir a la página 7"), mostrar un total de páginas, o el dataset es chico y estable (un admin interno). El offset es más simple y esos casos toleran sus problemas; en feeds grandes, públicos y que cambian, cursor.

### Módulo 10
**10.1** (1) **Queries profundamente anidadas** que explotan relaciones cíclicas (`user → projects → owner → projects → ...`) para agotar CPU/memoria. (2) **Queries de alta complejidad** que piden miles de campos relacionados en una sola request (amplificación). En REST cada endpoint tiene forma y costo fijos, así que no hay una query arbitraria que el cliente pueda inflar.

**10.2** Porque una query puede ser **poco profunda pero carísima**: `{ projects(first: 10000) { owner { name } } }` tiene profundidad 3 pero pide 10.000 nodos con su join. El límite de complejidad asigna un costo a cada campo (y lo multiplica por los argumentos tipo `first`), y rechaza si se pasa de un presupuesto, atrapando casos que la profundidad sola no ve.

**10.3** Ganás: ocultás tu schema (menos info para un atacante, menos superficie). Perdés: las herramientas de dev (GraphiQL, Apollo Sandbox, codegen) dependen de la introspección, y los clientes legítimos pierden el autodescubrimiento. Por eso se suele dejar habilitada en dev/staging y apagada (o detrás de auth) en prod pública.

### Módulo 11
**11.1** Porque el caché de Apollo es **normalizado**: guarda cada objeto una sola vez bajo una key `__typename:id`. Sin `id`, no puede identificar que el `User` de la query A es el mismo de la query B, así que no los unifica ni los actualiza en conjunto: cachea peor (por query) y la UI se desincroniza.

**11.2** (1) **Refetch** de las queries afectadas (`refetchQueries`): simple, pero hace round-trips extra. (2) **Update manual del caché** (`update`): preciso y sin red extra, pero más código. (3) **Devolver el objeto modificado con su `id`** en la respuesta de la mutation: Apollo lo mergea solo en el caché normalizado — la más limpia para cambios de campos de un objeto existente. Preferís la (3) para updates de campos; la (1) cuando la mutation cambia listas/relaciones difíciles de actualizar a mano; la (2) para control fino.

**11.3** Porque el HTTP caching de REST funciona por **URL + método GET** (el navegador, CDNs y proxies cachean por esa key). GraphQL usa típicamente un solo endpoint `POST /graphql`, y POST no es cacheable por defecto: la key no distingue una query de otra. Lo reemplaza el **caché normalizado del cliente** (Apollo) y, del lado server, las **persisted queries** (que al usar GET con un hash recuperan algo de caché HTTP/CDN).

### Módulo 12
**12.1** Federation compone un grafo unificado a partir de varios **subgraphs** (cada equipo/servicio expone su parte) vía un gateway, evitando un schema monolítico. Es prematuro si tenés un solo servicio o un solo equipo: agregás un gateway, contratos entre subgraphs y complejidad operativa para resolver un problema (escala organizacional) que todavía no tenés.

**12.2** **tRPC.** En un monorepo TS con un equipo, tRPC te da type-safety end-to-end **sin** escribir un schema GraphQL, resolvers, ni un cliente con caché normalizado: los tipos del server fluyen al cliente directo por inferencia de TS. GraphQL recién gana cuando tenés clientes heterogéneos (mobile, terceros) o equipos separados que necesitan un contrato explícito e independiente del lenguaje.

**12.3** Ejemplos: (1) **Upload/descarga de archivos o streaming**: GraphQL maneja mal los binarios; un endpoint REST con `multipart`/stream es más simple y eficiente. (2) **API pública que depende de caché HTTP/CDN** para performance (ej. un catálogo muy leído): REST cachea por URL gratis en el CDN, mientras que en GraphQL tendrías que armar persisted queries y caché propia para acercarte. En ambos, REST paga menos complejidad por el mismo resultado.

---

## Siguientes pasos

Cuando estos 12 módulos te salgan fluidos, sabés GraphQL **con criterio**: no solo escribir un resolver, sino evitar el N+1, asegurar la API, paginar bien y —sobre todo— decidir cuándo NO usarlo. Recomendado a continuación:

- **Armá un subgraph real:** un `@nestjs/graphql` code-first con DataLoaders, auth por context y cursor pagination sobre tu "Task API". Es un proyecto de portfolio fuerte.
- **Conectá el cliente:** una pantalla en React con Apollo Client, caché normalizado y una mutation que actualice la UI sin refetch. Ahí cerrás el full stack.
- **Sumá las defensas:** depth + complexity limiting y persisted queries; documentá en el README por qué cada una.
- Si tu contexto es un monorepo TS de un solo equipo, hacé el ejercicio honesto de comparar tu solución GraphQL contra **tRPC** y justificá la elección. Ese criterio vale oro en una entrevista.
