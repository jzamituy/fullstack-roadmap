# Ecosistema React: cómo elegir librerías con criterio (2026)

**No "qué librería usar" sino "cómo decidir" · contendientes por categoría · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo nace de un patrón que vas a vivir como Full Stack: cada tanto aparece un post viral con "las librerías que yo recomiendo en React". A veces aciertan, a veces repiten una elección que **ya no es el default en 2026**, y a veces meten una librería que **no existe o nadie usa**. El valor no está en memorizar la lista de otro: está en tener el **criterio** para juzgarla. Eso es lo que separa al que sigue modas del que toma decisiones que aguantan dos años.

**Lo que asumimos.** React y TypeScript con comodidad, haber sentido la diferencia entre **estado del cliente** y **datos del servidor**, y haber tocado alguna vez un `package.json` con más de 30 dependencias preguntándote "¿de verdad necesito todo esto?". No asume conocer ninguna de las librerías que vamos a comparar.

> ⚠️ **Nota sobre datos volátiles (clave en este módulo).** El ecosistema React **se mueve rápido**: versiones, estado de estabilidad, quién mantiene qué y cuál es "el default" cambian cada pocos meses. Todo lo marcado con ⚠️ verificalo contra el sitio oficial y el repo de GitHub de cada proyecto antes de citarlo como definitivo. Las cifras de adopción y los estados de versión que aparecen acá son de **mediados de 2026** y van a envejecer.

**Índice de módulos**
1. El criterio: cómo evaluar una librería antes de adoptarla
2. Estado del cliente: Zustand, Redux Toolkit, Jotai
3. Datos del servidor ≠ estado del cliente: TanStack Query, SWR, Apollo, urql, Relay
4. Componentes UI: shadcn/ui, MUI, Mantine
5. Primitivas headless: Base UI, Radix, Ark UI, React Aria
6. Formularios y validación: React Hook Form + Zod, TanStack Form
7. Metaframeworks: Next.js, TanStack Start, React Router
8. Testing: Vitest, RTL, Playwright, Storybook
9. Linting y formato: oxlint, ESLint, Biome
10. Bundlers: Vite + Rolldown, rspack, esbuild
11. Veredicto: el post viral, categoría por categoría

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

**Tipos de ejercicio** (para que sepas cuándo solo pensar y cuándo abrir el editor): 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación.

---

## Módulo 1 — El criterio: cómo evaluar una librería antes de adoptarla

**Teoría.** Antes de mirar nombres, fijá la **rúbrica**. Una recomendación sin criterio detrás es una moda; con criterio, es una decisión. Estas son las dimensiones con las que se juzga cualquier dependencia que vas a meter en producción:

- **Madurez y estabilidad.** ¿Llegó a `1.0`? ¿Cuántos *breaking changes* tuvo en el último año? Una API que todavía se mueve te obliga a migrar cada release. Madurez no es lo mismo que "viejo": es "estable y predecible".
- **Salud del mantenimiento (y *bus factor*).** ¿Quién lo mantiene — una persona, una empresa, una fundación? ¿Hay commits y releases recientes? ¿Cuántas personas pueden mergear? Una librería brillante mantenida por una sola persona que se quema es un riesgo. Mirá *governance*, no solo estrellas. Acá entra también la **seguridad**: ¿el proyecto tiene política de seguridad y responde rápido a los CVEs? Una dependencia con vulnerabilidades sin parchear es un riesgo de producción directo, no un detalle.
- **Tamaño y *tree-shaking*.** El peso que termina en el bundle del usuario. No es el número de npm: es cuánto sobrevive al *tree-shaking* en tu uso real. Una diferencia de 1 KB vs 11 KB importa en una landing; importa menos en un dashboard interno.
- **Calidad de los tipos (TS).** ¿Los tipos vienen de fábrica y son buenos, o son `@types/*` de terceros que van por detrás? En 2026 esto no es opcional: una librería con tipos pobres te contamina toda la base.
- **Tamaño del ecosistema.** Plugins, integraciones, respuestas en StackOverflow, ejemplos, gente que ya pisó tus piedras. Lo aburrido-y-popular casi siempre le gana a lo elegante-y-nicho cuando hay que entregar.
- **Lock-in y costo de salida.** Si mañana querés irte, ¿cuánto duele? Un router o un metaframework te atan mucho más que una utilidad de fechas. Cuanto más al centro de tu arquitectura, más conservador conviene ser.
- **Alineación con tu stack.** La mejor librería de GraphQL es inútil si no usás GraphQL. La elección correcta depende de **tu** contexto, no del de quien escribe el post.

La regla de oro: **no elijas por novedad, elegí por el problema.** "Es nuevo y rápido" no es una razón; "resuelve este problema que tengo, con un costo de salida que acepto" sí lo es. Y el corolario: una recomendación que no dice **para qué tamaño de equipo y de app** sirve poco, porque la respuesta correcta cambia con la escala.

**Ejercicios 1**
1.1 🔁 Nombrá cuatro de las siete dimensiones del criterio.
1.2 🧠 Un post recomienda la librería X "porque es la más rápida y la más nueva". ¿Qué dos preguntas le harías antes de adoptarla?
1.3 🧠 ¿Por qué "tiene 40k estrellas en GitHub" es una señal débil de salud de mantenimiento?

---

## Módulo 2 — Estado del cliente: Zustand, Redux Toolkit, Jotai

**Teoría.** Hablamos del **estado del cliente**: lo que vive en memoria del navegador y no es un reflejo del servidor (un tema oscuro, un wizard a medio llenar, un panel abierto/cerrado). Los contendientes:

- **Zustand** — store mínima basada en hooks, ~1 KB, sin boilerplate, sin providers obligatorios. Curva plana.
- **Redux Toolkit (RTK)** — el Redux moderno: mucho menos boilerplate que el Redux clásico, DevTools excelentes, middleware, y un modelo estricto y predecible. ~11 KB.
- **Jotai** — estado **atómico**: en vez de una store global, declarás "átomos" pequeños que se componen. Encaja muy bien cuando el estado es granular y muy derivado.

**Recomendación con criterio:**
- **Default para la mayoría de apps nuevas → Zustand.** Mínimo costo, tipos buenos, sin ceremonia. Cubre el 90% de los casos de estado del cliente.
- **Apps grandes, equipos de 10+, requisitos de auditabilidad/trazabilidad estricta → Redux Toolkit.** Su rigidez es una *feature* cuando muchas manos tocan el mismo estado y necesitás un *time-travel* de DevTools y un patrón único e impuesto.
- **Estado muy granular y derivado (editores, canvases, formularios atómicos) → Jotai.**

> 🔎 **Corrección a la sabiduría viral.** Es común ver "usá **Redux** por su historia, sus mantenedores y porque escala". El argumento de escala es real, pero **Redux ya no es el *default* del ecosistema en 2026**: ⚠️ según las encuestas **State of JS / State of React**, Zustand trepó de ~28% (2023) a ~50% (2025) de adopción y ya superó a Redux en descargas de npm, mientras el uso de Redux dejó de crecer y empezó a ceder terreno. (Las cifras exactas varían según la encuesta y la métrica — verificá la fuente antes de citarlas; lo que no cambia es la *dirección*.) Recomendar Redux como elección por defecto para *cualquier* app es conservador de más: para la app promedio es más boilerplate del que necesitás. La forma correcta del consejo es **"Zustand por defecto; Redux Toolkit cuando la escala y el tamaño del equipo lo justifican"** — y, ojo, "Redux" a secas hoy significa **Redux Toolkit**, no el Redux clásico con `connect` y `mapStateToProps`.

**Ejercicios 2**
2.1 🔁 ¿Qué problema concreto resuelve un store de estado del cliente que no resuelve `useState`/`useContext`?
2.2 🧠 Un dev senior te dice "meté todo en Redux para estar seguro". ¿Por qué en 2026 ese consejo es discutible como default?
2.3 🧠 ¿En qué situación elegirías Redux Toolkit *por encima* de Zustand, a pesar del boilerplate extra?

---

## Módulo 3 — Datos del servidor ≠ estado del cliente: TanStack Query, SWR, Apollo, urql, Relay

**Teoría.** Acá está el error conceptual más caro del ecosistema, y por eso lo separamos del módulo anterior: **los datos del servidor NO son estado del cliente.** Un listado que viene de tu API no es "estado tuyo": es una **caché local de una verdad que vive en el servidor**, con su propia lógica de *staleness*, refetch, reintentos, deduplicación e invalidación. Meter eso en Zustand o Redux a mano es reimplementar mal una librería de *server-state*.

Los contendientes:

Para **REST/fetch**:

- **TanStack Query** (antes React Query) — el estándar de facto para *server-state* sobre REST/fetch. Caché, refetch en foco/reconexión, reintentos, mutaciones con *optimistic updates*, paginación e infinite queries. Tipos excelentes. Agnóstico de framework.
- **SWR** — de Vercel, misma idea (`stale-while-revalidate`), más minimalista. Encaja muy bien en proyectos Next.js chicos donde querés lo esencial sin tanta superficie de API.

Para **GraphQL** (acá la caché es *normalizada*: los objetos se guardan por su `id` y una mutación que devuelve un objeto actualiza todas las vistas que lo referencian — algo que TanStack Query no hace solo):

- **Apollo Client** — el cliente GraphQL más adoptado y con el ecosistema más grande. Caché normalizada, gestión de estado local, devtools maduras, paginación, *optimistic UI*. El costo: es **pesado** y su caché tiene una curva de aprendizaje (políticas de `type policies`, `fetchPolicy`). Es el "todo incluido".
- **urql** — alternativa **más liviana y modular**: arrancás con caché de documento simple y sumás la *Graphcache* normalizada como *exchange* (plugin) solo si la necesitás. Más fácil de entender de punta a punta; menos batteries-included que Apollo. Buen punto medio entre "simple" y "normalizado".
- **Relay** — cliente GraphQL de Meta, el más potente y opinado (fragmentos colocados, normalización agresiva, paginación por cursor de fábrica, `@defer`/`@stream`), pero con curva fuerte, compilador propio y convenciones estrictas de schema. Rinde a **escala GraphQL grande** con un equipo que banca su disciplina.

**Recomendación con criterio:**
- **Sin GraphQL → TanStack Query.** Es el caballo de batalla. **Next.js minimalista → SWR** como alternativa livianita.
- **Con GraphQL, caso general → Apollo Client** (ecosistema y madurez) o **urql** si querés algo más liviano y modular y no necesitás todo lo de Apollo.
- **Con GraphQL, a escala grande y equipo que banca la complejidad → Relay.**
- Regla práctica: **urql** si tu prioridad es entenderlo y mantenerlo simple; **Apollo** si querés el estándar con todo resuelto y devtools; **Relay** si la escala y la performance justifican su rigidez.

> 🔎 **Corrección a la sabiduría viral.** El consejo "**TanStack Query si no usás GraphQL, Relay si usás GraphQL**" es **correcto y vigente**. Pero cuidado con las listas que agregan una tercera opción con nombre llamativo "que es un nuevo contendiente": antes de adoptarla, aplicá el Módulo 1 — ⚠️ si no la encontrás en npm con descargas reales, repo activo y documentación, **probablemente no exista como librería seria** (o sea un experimento de una persona). Una recomendación responsable no mete nombres que no podés verificar. El que de verdad falta nombrar en esa categoría no es ningún recién llegado exótico, sino **SWR**.

**Ejercicios 3**
3.1 🔁 ¿Por qué se dice que los datos del servidor son "una caché", no "estado del cliente"?
3.2 🧠 Un compañero guarda la respuesta de `GET /productos` en un store de Zustand y la actualiza a mano. ¿Qué cuatro cosas vas a terminar reimplementando, y qué le proponés en su lugar?
3.3 🧠 Aparece en un post "la librería Foo, nuevo contendiente en data fetching". ¿Qué hacés antes de meterla en producción?

---

## Módulo 4 — Componentes UI: shadcn/ui, MUI, Mantine

**Teoría.** Acá decidís de dónde salen tus botones, modales y tablas. Dos filosofías opuestas:

- **Librería de componentes (MUI, Mantine)** — instalás un paquete y consumís `<Button>`, `<Dialog>`, etc. Rapidísimo para arrancar; el costo es que **personalizar a fondo** pelea contra los estilos y la API de la librería, y quedás atado a su sistema de theming.
- **shadcn/ui** — no es una dependencia, es un **catálogo de componentes que copiás a tu repo** (sobre Tailwind CSS + primitivas headless). El código es tuyo: lo editás como quieras, sin capa de theming ajena. El costo es que **vos lo mantenés**.

**Recomendación con criterio:**
- **Querés control total y un design system propio → shadcn/ui personalizado.** Es la elección dominante en 2026 para equipos que quieren poseer su UI. Con asistentes de IA, mantener y evolucionar tus propios componentes es mucho más barato que antes — la objeción histórica ("¿quién mantiene todo eso?") pesa menos hoy.
- **Querés entregar ya, un look corporativo coherente y no querés diseñar nada → MUI o Mantine.** Mantine en particular trae un set enorme de componentes y hooks listos.

> 🔎 **Veredicto sobre el post.** "**shadcn personalizado, rolando tu propio design system con Tailwind + componentes headless**" es una recomendación **sólida y muy alineada con 2026**. La clave que no hay que perder de vista: shadcn **no reemplaza** a la capa de primitivas headless del módulo siguiente — **se apoya** en ella (históricamente Radix; cada vez más Base UI). No son competidores, son capas.

**Ejercicios 4**
4.1 🔁 ¿En qué se diferencia, de fondo, instalar MUI vs. usar shadcn/ui?
4.2 🧠 ¿Cuál es el costo oculto de shadcn/ui que MUI te ahorra, y por qué en 2026 ese costo pesa menos?

---

## Módulo 5 — Primitivas headless: Base UI, Radix, Ark UI, React Aria

**Teoría.** Una primitiva **headless** te da el **comportamiento y la accesibilidad** (foco, ARIA, navegación por teclado, manejo del *portal*) de un componente complejo — un combobox, un menú, un dialog — **sin estilos**. Vos ponés el CSS. Es la capa sobre la que se construye un design system serio (incluido shadcn). Por qué importa: la accesibilidad y el manejo de foco/teclado bien hechos son **brutalmente difíciles**; nadie debería reimplementarlos.

Los contendientes:

- **Radix Primitives** — durante años *el* estándar; primitivas maduras y bien documentadas. ⚠️ Modulz (la empresa detrás de Radix) fue adquirida por **WorkOS en junio de 2022** — pero el problema no es la adquisición en sí: es que el ritmo de mantenimiento se **enfrió de forma marcada en 2024-2025** al irse varios mantenedores originales (sigue sin combobox/multi-select nativos y el soporte de novedades llegó lento). Adquisición vieja, estancamiento reciente: no confundas una cosa con la otra.
- **Base UI** — ⚠️ llegó a **v1.0 estable en diciembre de 2025**, mantenido por el equipo de **MUI** (con gente del ecosistema Radix involucrada). Filosofía de "bloques de comportamiento que ensamblás vos", y trae patrones que Radix no tenía nativos (combobox, multi-select). Es el que está **shipeando** activamente.
- **Ark UI** — mismas primitivas headless, pero **multi-framework** (React, Vue, Solid) y con lógica interna en **máquinas de estado (XState)**. Elegilo si necesitás compartir comportamiento entre frameworks.
- **React Aria** (Adobe) — el más serio en **accesibilidad** e internacionalización; un poco más verboso, pero imbatible si la a11y es un requisito duro.

**Recomendación con criterio:**
- **Design system nuevo y de largo plazo → Base UI.** Equipo activo, v1.0 estable, futuro claro.
- **Ya tenés Radix andando → no hay urgencia de migrar**; sigue siendo sólido.
- **Multi-framework → Ark UI. Accesibilidad/i18n como requisito duro → React Aria.**

> 🔎 **Veredicto sobre el post.** "**Base UI, de los creadores de Radix y MUI, en desarrollo activo**" es una recomendación **acertada**. Matiz de precisión: el **liderazgo de producto es de MUI**, pero el equipo incorpora contribuidores que vienen de **Radix** y **Floating UI** — el propio repo oficial se describe como "from the creators of Radix, Floating UI, and Material UI", así que la frase del post es **defendible**, no falsa. Lo importante — *quién shipea hoy* — está bien identificado.

**Ejercicios 5**
5.1 🔁 ¿Qué te da una primitiva "headless" y qué te deja a vos?
5.2 🧠 ¿Por qué reimplementar a mano el comportamiento accesible de un combobox es una mala idea aunque "parezca fácil"?
5.3 🧠 Arrancás un design system desde cero hoy. ¿Base UI o Radix, y por qué?

---

## Módulo 6 — Formularios y validación: React Hook Form + Zod, TanStack Form

**Teoría.** Acá hay que separar **dos responsabilidades** que la gente mezcla:

1. **Estado del formulario** — valores, campos tocados, errores, envío, *re-renders* mínimos. Eso lo da **React Hook Form (RHF)** o **TanStack Form**.
2. **Validación de esquema** — *qué* es un dato válido. Eso lo da un validador como **Zod** o **Valibot**, que además te da el **tipo TypeScript inferido** del esquema gratis.

Se conectan con un *resolver*: definís el esquema en Zod, RHF lo usa para validar. Una sola fuente de verdad para validación **y** tipos.

Los contendientes de estado de formulario:

- **React Hook Form** — el más maduro y adoptado, performante (no re-renderiza todo el form en cada tecla), ecosistema enorme de resolvers.
- **TanStack Form** — más nuevo, type-safe de punta a punta y agnóstico de framework; subiendo, pero con menos kilometraje.

**Recomendación con criterio:**
- **Default → React Hook Form + Zod.** Madurez, comunidad, resolvers para todo.
- **Apostás al ecosistema TanStack y querés type-safety uniforme → TanStack Form**, sabiendo que es más joven.

> 🔎 **Corrección a la sabiduría viral.** Llamar a esta categoría "**form validation**" y listar solo RHF vs. TanStack Form mezcla las dos capas: **RHF no valida por sí mismo**, orquesta el estado y delega la validación en un esquema. El consejo "**React Hook Form por más maduro**" es **correcto**, pero incompleto sin nombrar a **Zod/Valibot** al lado — que es lo que de verdad hace la validación y, de paso, te da los tipos.

**Ejercicios 6**
6.1 🔁 ¿Qué responsabilidad cubre React Hook Form y cuál cubre Zod?
6.2 🧠 ¿Por qué definir el esquema en Zod te ahorra escribir tipos TypeScript a mano para el formulario?
6.3 ✍️ Definí en Zod un esquema `loginSchema` con `email` (formato email) y `password` (mínimo 8 caracteres), y derivá de él el tipo `LoginForm` **sin escribirlo a mano**.

---

## Módulo 7 — Metaframeworks: Next.js, TanStack Start, React Router

**Teoría.** Un **metaframework** te da routing, SSR/streaming, data loading, *server functions* y deploy sobre React. Los contendientes:

- **Next.js** — el más adoptado, *server-first* (App Router + RSC por defecto), ecosistema y documentación enormes, deploy de primera en Vercel (y en otros lados con más trabajo).
- **TanStack Start** — *client-first*, type-safety de punta a punta sobre TanStack Router + Vite + Nitro, deploy agnóstico sin lock-in. ⚠️ Nuevo y en rápida evolución.
- **React Router (v7)** — fusionó con Remix; podés usarlo como router de SPA o como framework full-stack con SSR. Camino de actualización suave si ya venías de React Router.

**Recomendación con criterio:**
- **Default seguro → Next.js**, sobre todo si valorás ecosistema, contratación y un deploy sin fricción. Los tres convergen hacia un set de features parecido.
- **Preferís el modelo client-first y odiás el lock-in → TanStack Start** (en ascenso, vale tenerlo en el radar).
- **Venís de React Router y no querés migrar de paradigma → React Router v7.**

> 🔎 **Veredicto sobre el post.** "**Next.js; todos convergen hacia el mismo set de features; ojo con TanStack Start que sube rápido**" es **acertado y matizado**. Este hub tiene un módulo entero dedicado a esta decisión — la comparación a fondo, el contraste *server-first* vs *client-first* y el criterio de cuándo SÍ y cuándo NO, están en **[TanStack Start vs Next.js](tanstack-start.md)**.

**Ejercicios 7**
7.1 🔁 ¿Qué tres capas te resuelve un metaframework que un SPA pelado no?
7.2 🧠 ¿Por qué "todos convergen al mismo set de features" hace que la elección dependa más del contexto (equipo, lock-in, deploy) que de la lista de features?

---

## Módulo 8 — Testing: Vitest, RTL, Playwright, Storybook

**Teoría.** El stack de testing de frontend tiene piezas que **no compiten entre sí**, se complementan:

- **Vitest** — el *test runner* unitario/integración. Maduro, rapidísimo (corre sobre el pipeline de Vite), API compatible con Jest, y mejor *watch mode* y soporte de ESM/TS de fábrica.
- **Jest** — el runner clásico; sigue siendo enorme, pero en proyectos Vite, Vitest es el encaje natural.
- **React Testing Library (RTL)** — no es un runner: es la librería para testear **comportamiento desde la óptica del usuario** (buscar por rol/texto, no por detalles de implementación). Corre **sobre** Vitest o Jest.
- **Playwright** — E2E de browser real (lo cubre su propio módulo en este hub).
- **Storybook** — desarrollo y documentación de componentes en aislamiento; además sirve de base para *visual regression* y tests de interacción.

**Recomendación con criterio:**
- **Proyecto Vite → Vitest + RTL** para unit/integración; **Playwright** para E2E; **Storybook** si tenés un design system que documentar y testear visualmente.
- **Proyecto con Jest ya andando y sin Vite → no hay urgencia de migrar.**

> 🔎 **Veredicto sobre el post.** "**Vitest + RTL + Storybook; Vitest ya es maduro y muy performante**" es **sólido y vigente**. El único agregado: para la pirámide completa falta **Playwright** (o Cypress) en la capa E2E — Vitest+RTL cubren unidad e integración, no el flujo end-to-end en un browser real. Ver **[Testing práctico](testing.md)** y **[Playwright](playwright.md)**.

**Ejercicios 8**
8.1 🔁 ¿Por qué Vitest y RTL no son "lo mismo" ni compiten?
8.2 🧠 Tu suite tiene Vitest + RTL con buena cobertura, pero en producción se rompe un flujo de checkout de tres pantallas. ¿Qué capa de testing te faltaba?

---

## Módulo 9 — Linting y formato: oxlint, ESLint, Biome

**Teoría.** El *linter* atrapa errores y fuerza convenciones. Hasta hace poco era ESLint y nada más; ahora hay alternativas en Rust, mucho más rápidas:

- **ESLint** — el estándar histórico. Ecosistema de plugins gigantesco y, sobre todo, **reglas *type-aware*** (vía `typescript-eslint`) que entienden los tipos de tu código — algo que ningún linter rápido todavía iguala del todo.
- **oxlint** (proyecto OXC) — linter en Rust, **50–100× más rápido** que ESLint, con 500+ reglas y cero config para arrancar. ⚠️ Su 1.0 **todavía no hace type-aware linting completo** (cubre ~43 de 59 reglas de typescript-eslint; el modo type-aware vía `tsgolint` está en **alpha**).
- **Biome** — linter **+ formateador** en uno (también en Rust); alternativa a la dupla ESLint + Prettier.

**Recomendación con criterio:**
- **Si ya usás Vite (con Rolldown/OXC por debajo) → sumá oxlint** como primera pasada rapidísima.
- **Patrón 2026 (híbrido) → oxlint primero, ESLint para lo que falta.** oxlint corre las reglas de alta frecuencia en milisegundos; ESLint se queda solo con las **reglas type-aware** y los plugins que oxlint aún no soporta. Con el tiempo vas achicando la parte de ESLint.
- **Querés un solo binario para lint + formato → Biome** es la apuesta a considerar.

> 🔎 **Corrección a la sabiduría viral.** "**oxlint; si usás Vite 8 ya estás usando OXC vía Rolldown, así que oxlint es mejor**" tiene una **media verdad peligrosa**: oxlint es genuinamente más rápido y un gran complemento, pero en 2026 **todavía no reemplaza a ESLint** en proyectos TypeScript que dependen de reglas *type-aware*. El consejo correcto no es "reemplazá ESLint por oxlint" sino **"oxlint como pasada rápida + ESLint para las reglas type-aware, e ir cerrando la brecha"**. Decir "reemplazalo" sin el matiz te puede dejar sin reglas que estaban atrapando bugs reales. Leído con la rúbrica del Módulo 1: oxlint gana en **performance**, ESLint en **madurez** y **ecosistema** — por eso hoy conviven, y el día que oxlint cierre la brecha *type-aware* la balanza se mueve. La herramienta del año cambia; el criterio (qué dimensión priorizás) no.

**Ejercicios 9**
9.1 🔁 ¿Qué hace oxlint mucho mejor que ESLint, y qué hace ESLint que oxlint todavía no iguala en 2026?
9.2 🧠 Un compañero quiere borrar ESLint y dejar solo oxlint en un proyecto TS grande. ¿Qué riesgo le señalás y qué patrón le proponés?

---

## Módulo 10 — Bundlers: Vite + Rolldown, rspack, esbuild

**Teoría.** El *bundler* arma tu app para dev (server con HMR) y para producción (build optimizado). Los contendientes:

- **Vite** — el estándar de facto del frontend moderno; ecosistema de plugins enorme, DX excelente. ⚠️ **Vite 8 (marzo 2026)** unificó dev y producción sobre **Rolldown** (bundler en Rust sobre OXC), retirando la antigua dupla esbuild (dev) + Rollup (prod). Recortó los builds grandes del orden de **10-30×** manteniendo la compatibilidad de plugins.
- **rspack** — port de webpack en Rust; ideal si venís de una config de webpack pesada y querés la misma API con velocidad de Rust.
- **esbuild** — ultrarrápido y minimalista; Vite lo usó para dev/pre-bundling **hasta la v7** (en v8 lo reemplazó Rolldown). Sigue muy presente **dentro de** otras herramientas más que como bundler de app completo.
- **Turbopack** — el bundler de Next.js (en Rust); relevante si vivís dentro de Next.

**Recomendación con criterio:**
- **Default → Vite, sin discusión** para casi cualquier app React nueva: combina **adopción de ecosistema** y **performance** como ningún otro.
- **Migración desde webpack legacy → rspack.** **Dentro de Next.js → Turbopack** viene dado.

> 🔎 **Veredicto sobre el post.** "**Vite, lejos; pocos bundlers tienen a la vez buen ecosistema y gran performance**" es **correcto y poco polémico**. La nota de color de 2026: la línea entre "bundler" y "toolchain" se borra — Vite + Rolldown + oxlint + oxfmt son piezas del **mismo ecosistema OXC/VoidZero**, y elegir Vite cada vez más significa entrar a esa familia entera. Leído con la rúbrica: Vite gana porque combina **ecosistema** y **performance**; rspack y Turbopack solo entran cuando el contexto inclina una dimensión (webpack legacy, o el lock-in de Next.js). El pick de hoy puede cambiar; la pregunta —¿qué dimensión pesa en tu caso?— no.

**Ejercicios 10**
10.1 🔁 ¿Por qué esbuild aparece más "dentro de" otras herramientas que como bundler de app completo?
10.2 🧠 ¿Qué dos propiedades hacen de Vite el default y por qué una sola no alcanzaría?

---

## Módulo 11 — Veredicto: el post viral, categoría por categoría

**Teoría.** Cerramos aplicando el criterio del Módulo 1 al post que disparó todo esto. La conclusión general: **es mayormente sólido y actual**, con unos pocos puntos que un lector con criterio corrige. Esto es exactamente el músculo que querés entrenar — no descartar el post, sino **leerlo activamente**.

| # | Categoría | Recomendación del post | Veredicto |
|---|---|---|---|
| 1 | Estado | Redux | ⚠️ **Desactualizado como default.** En 2026 el default pragmático es **Zustand**; Redux Toolkit queda para escala/equipos grandes. |
| 2 | UI | shadcn personalizado | ✅ **Sólido.** Muy alineado con 2026. |
| 3 | Headless | Base UI | ✅ **Acertado.** Nit: lo lidera el equipo **MUI**, no "los creadores de Radix". |
| 4 | Data fetching | TanStack Query / Relay (+ un "nuevo contendiente") | ✅ El núcleo es correcto. 🚩 El "nuevo contendiente" con nombre no verificable: **no adoptar sin verificar que exista**. Falta **SWR**. |
| 5 | Forms | React Hook Form | ✅ **Correcto** pero incompleto: falta **Zod/Valibot** — RHF no valida, orquesta. |
| 6 | Metaframeworks | Next.js (ojo TanStack Start) | ✅ **Acertado y matizado.** |
| 7 | Testing | Vitest + RTL + Storybook | ✅ **Sólido.** Agregar **Playwright** para E2E. |
| 8 | Linting | oxlint (reemplazá ESLint) | ⚠️ **Media verdad.** oxlint aún no hace type-aware completo: patrón **híbrido** oxlint + ESLint. |
| 9 | Bundlers | Vite | ✅ **Correcto.** |

La moraleja del módulo entero: **una recomendación vale por el criterio que la sostiene, no por la autoridad de quien la firma.** Un post que dice "Redux porque tiene historia" sin decir "para equipos grandes" o que mete una librería que no podés encontrar en npm, no es necesariamente malo — pero te toca a vos completar el matiz. Ese es el trabajo de un Full Stack senior eligiendo su stack.

**Ejercicios 11**
11.1 🧠 De los nueve puntos del post, ¿cuáles dos corregirías con más firmeza y por qué?
11.2 🧠 ¿Por qué meter en una recomendación una librería que no se puede verificar en npm es un problema más grave que recomendar una opción "subóptima pero real"?

---

## Soluciones

### Módulo 1
```
1.1 Cualquier cuatro de: madurez/estabilidad; salud del mantenimiento (bus factor/
    governance); tamaño y tree-shaking; calidad de los tipos TS; tamaño del ecosistema;
    lock-in y costo de salida; alineación con tu stack.
1.2 (a) "¿Qué problema mío resuelve mejor que lo que ya tengo?" — novedad no es razón.
    (b) "¿Quién la mantiene y cuál es el costo de salida si la API se mueve?" — madurez
    y lock-in. Velocidad y novedad no dicen nada sobre eso.
1.3 Porque las estrellas miden popularidad/hype histórico, no salud actual: un repo con
    40k estrellas puede tener un solo mantenedor quemado, sin releases en un año y issues
    sin responder. Lo que mide salud es frecuencia de commits/releases, número de personas
    que pueden mergear (bus factor) y modelo de governance (persona vs empresa vs fundación).
```

### Módulo 2
```
2.1 Estado del cliente compartido entre componentes lejanos sin "prop drilling" ni
    re-renders masivos, con una API ergonómica. useState es local al componente; useContext
    resuelve el drilling pero re-renderiza a todos los consumidores ante cualquier cambio
    y no escala bien para estado que cambia seguido.
2.2 Porque "meté todo en Redux" trae boilerplate y peso que la app promedio no necesita,
    y porque buena parte de lo que la gente mete en Redux es en realidad *server-state*
    (va a TanStack Query, no a un store). En 2026 el default es Zustand; Redux Toolkit se
    reserva para cuando la escala o el tamaño del equipo lo justifican.
2.3 Equipo grande (10+), donde la rigidez y el patrón único impuesto son una ventaja; o
    cuando necesitás DevTools con time-travel, middleware elaborado, o auditabilidad/
    trazabilidad estricta del flujo de acciones. Ahí el boilerplate compra predecibilidad.
```

### Módulo 3
```
3.1 Porque la verdad vive en el servidor; lo que tenés en el cliente es una copia que
    puede quedar "stale". Por eso necesita políticas de caché: cuándo es fresca, cuándo
    refetchear, cómo invalidar, reintentar y deduplicar. Tratarla como "estado tuyo"
    ignora que el servidor pudo cambiarla por detrás.
3.2 Vas a reimplementar (mal): caché con invalidación, refetch en foco/reconexión,
    reintentos con backoff, y deduplicación de requests en vuelo (y probablemente loading/
    error states y paginación). Le proponés TanStack Query, que trae todo eso resuelto y
    probado, y dejás el store de Zustand solo para estado del cliente de verdad.
3.3 Aplicás el Módulo 1: la buscás en npm (¿existe? ¿descargas reales?), mirás el repo
    (¿activo? ¿quién la mantiene? ¿docs?), su madurez y su costo de salida. Si no podés
    verificar que existe y se mantiene, no entra a producción — por más que un post la
    nombre como "el nuevo contendiente".
```

### Módulo 4
```
4.1 MUI es una dependencia: consumís sus componentes y vivís con su API y su theming;
    personalizar a fondo pelea contra la librería. shadcn/ui copia el código del componente
    a tu repo: el código es tuyo, lo editás libremente y no hay capa de theming ajena —
    pero vos lo mantenés.
4.2 El costo oculto es el mantenimiento: ese código ahora es tuyo y lo tenés que cuidar y
    evolucionar (MUI te lo ahorraba al mantenerlo ellos). En 2026 pesa menos porque los
    asistentes de IA hacen mucho más barato mantener y evolucionar tu propio design system.
```

### Módulo 5
```
5.1 Te da el comportamiento y la accesibilidad (manejo de foco, ARIA, navegación por
    teclado, portales, estados). Te deja a vos el estilo/CSS — es "headless": sin look.
5.2 Porque la accesibilidad y el manejo de foco/teclado de un combobox (roles ARIA,
    anuncios a lectores de pantalla, navegación con flechas, escape, foco atrapado, i18n)
    tienen cientos de casos borde que "parecen fáciles" hasta que los pisás. Una primitiva
    headless ya los resolvió y testeó; reimplementarlos a mano casi siempre sale roto.
5.3 Base UI: v1.0 estable (dic 2025), equipo de MUI shipeando activamente, futuro claro y
    patrones que Radix no trae nativos. Radix sigue siendo sólido si ya lo tenés, pero su
    ritmo se enfrió en 2024-2025 al irse mantenedores clave (la adquisición de Modulz por
    WorkOS fue en 2022, dato aparte), así que para algo nuevo y de largo plazo, Base UI es
    la apuesta más segura.
```

### Módulo 6
```
6.1 React Hook Form cubre el estado del formulario: valores, campos tocados, errores,
    envío, y re-renders mínimos. Zod cubre la validación de esquema: define qué dato es
    válido y, de paso, infiere el tipo TypeScript. Se conectan con un resolver.
6.2 Porque el esquema de Zod ES la fuente de verdad: con z.infer<typeof schema> obtenés el
    tipo TypeScript del formulario automáticamente. Si escribieras el tipo a mano, tendrías
    dos fuentes que se pueden desincronizar; con Zod, validación y tipo salen de lo mismo.
```
```ts
// 6.3
import { z } from "zod";

const loginSchema = z.object({
  email: z.email(),            // Zod 4: validador de email top-level (antes z.string().email())
  password: z.string().min(8),
});

type LoginForm = z.infer<typeof loginSchema>;
// LoginForm === { email: string; password: string } — inferido, una sola fuente de verdad
```

### Módulo 7
```
7.1 Routing, renderizado en servidor (SSR/streaming) y data loading (más server functions
    y deploy). Un SPA pelado solo renderiza en el cliente y te deja armar routing y datos a
    mano.
7.2 Porque si las features tienden a igualarse, el desempate ya no es "cuál tiene X": pasa
    a ser quién encaja mejor con tu equipo (a quién contratás, qué saben), cuánto lock-in
    aceptás (Next+Vercel vs deploy agnóstico de TanStack Start) y tu paradigma preferido
    (server-first vs client-first). El contexto manda, no la checklist.
```

### Módulo 8
```
8.1 Vitest es el test runner (ejecuta los tests, da assertions, mocks, watch). RTL es una
    librería que se usa DENTRO del runner para renderizar componentes y testearlos como lo
    haría un usuario (por rol/texto, no por implementación). Vitest ejecuta; RTL define cómo
    consultás el DOM. Se usan juntos.
8.2 La capa end-to-end (E2E) en un browser real, con Playwright (o Cypress). Vitest + RTL
    prueban unidades y componentes en aislamiento, pero no recorren el flujo completo de
    tres pantallas con navegación, red y estado reales — ahí es donde se rompió.
```

### Módulo 9
```
9.1 oxlint es 50–100× más rápido (está en Rust) y trae 500+ reglas con cero config. ESLint
    todavía gana en reglas type-aware (typescript-eslint), que entienden los tipos del
    código, y en la amplitud de su ecosistema de plugins — cosas que oxlint 1.0 aún no
    iguala (su modo type-aware vía tsgolint está en alpha).
9.2 Riesgo: al sacar ESLint perdés las reglas type-aware (y plugins) que oxlint todavía no
    cubre — reglas que pueden estar atrapando bugs reales. Patrón propuesto: híbrido —
    oxlint como pasada rápida sobre las reglas de alta frecuencia, ESLint solo para las
    type-aware y los plugins faltantes, e ir achicando la parte de ESLint con el tiempo.
```

### Módulo 10
```
10.1 Porque esbuild es ultrarrápido pero minimalista: le falta el ecosistema de plugins y
     las features de un bundler de app completo, así que conviene como pieza interna (Vite
     lo usó para dev y pre-bundling hasta la v7) más que como la herramienta de build final.
10.2 Adopción de ecosistema (plugins, integraciones, comunidad, ejemplos) y performance.
     Una sola no alcanza: un bundler rapidísimo pero sin ecosistema te deja sin plugins ni
     soporte; uno con ecosistema enorme pero lento te frena el dev y los builds. Vite tiene
     las dos (y con Rolldown en Vite 8 ya cerró la brecha de velocidad que le quedaba).
```

### Módulo 11
```
11.1 (Respuesta razonada; las dos más firmes:) (a) "Redux como default" — en 2026 el
     default es Zustand; recomendar Redux a secas para cualquier app es boilerplate de más
     y no distingue server-state de client-state. (b) "Reemplazá ESLint por oxlint" — oxlint
     aún no hace type-aware completo, así que reemplazarlo sin matiz te saca reglas que
     atrapan bugs; el patrón correcto es híbrido. (También vale señalar el "nuevo
     contendiente" de data fetching no verificable y la falta de Zod en forms.)
11.2 Porque una opción subóptima pero real igual funciona, tiene docs, comunidad y un costo
     de salida conocido — a lo sumo no es la mejor. Una librería que no podés verificar en
     npm puede no existir, estar abandonada o ser el experimento de una persona: la metés y
     te quedás sin soporte, sin parches de seguridad y sin nadie que responda. El daño es de
     otra categoría: no es "elegí peor", es "elegí algo que no debería estar en producción".
```

---

## Siguientes pasos

Con este módulo sumás el **criterio para elegir tu stack de frontend**, que es tan importante como saber usar cada pieza. El recorrido natural desde acá: **(1)** profundizá la decisión más pesada — la de metaframework — en **[TanStack Start vs Next.js](tanstack-start.md)**, donde el contraste *server-first* vs *client-first* se trata a fondo; **(2)** bajá a la capa de runtime con **[React Server Components a fondo](rsc.md)**, que es el paradigma sobre el que se paran Next.js y compañía; **(3)** llevá las elecciones de testing de este módulo a la práctica en **[Testing práctico](testing.md)** y **[Playwright](playwright.md)**. La constante: cada vez que aparezca el próximo post viral con "las librerías que recomiendo", ya tenés la rúbrica del Módulo 1 para leerlo con criterio en vez de copiarlo.
