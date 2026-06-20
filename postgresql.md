# PostgreSQL + ORM: la base de datos real de tu backend

**De SQL a mano al ORM tipado · el escalón que conecta todo lo anterior · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Hasta acá tus servicios de NestJS guardaban en arrays o `Map` en memoria (`InMemoryUserRepository`). Esto sirve para aprender la mecánica, pero los datos se borran al reiniciar la app. Esta página reemplaza ese "en memoria" por una base de datos **PostgreSQL** real, primero entendiendo **SQL a mano** (no dependas solo del ORM) y después con **Drizzle**, un ORM tipado que encaja perfecto con tu experiencia de TypeScript. El cierre es el puente clave: implementar la interfaz `Repository` que ya conocés con una base real, sin tocar tu lógica de negocio.

**Por qué SQL a mano *y* ORM.** El ORM te ahorra escribir SQL repetitivo y te da tipos, pero esconde lo que pasa por debajo. Si no entendés qué query genera, no vas a poder diagnosticar el problema N+1, decidir un índice, ni leer el plan de ejecución cuando algo va lento. La regla profesional: **entendé el SQL, después dejá que el ORM lo escriba por vos.**

**Para practicar.** Levantá un Postgres local con Docker en una línea:

```bash
docker run --name pg-fullstack -e POSTGRES_PASSWORD=secret -p 5432:5432 -d postgres:16
```

Y conectate con `psql` (o cualquier cliente como TablePlus / DBeaver):

```bash
docker exec -it pg-fullstack psql -U postgres
```

**Índice de módulos**
1. Por qué relacional (y cuándo Postgres vs. Mongo)
2. Modelado: tablas, tipos y clave primaria
3. Relaciones: claves foráneas (1:N y N:M)
4. Normalización (lo justo y necesario)
5. SQL esencial: `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`
6. `JOIN`: combinar tablas
7. Agregaciones: `GROUP BY`, `COUNT`, `SUM`, `HAVING`
8. Índices y el problema de rendimiento
9. Migraciones: control de versiones de tu esquema
10. Drizzle: schema y queries tipadas
11. Relaciones en Drizzle y el problema N+1
12. Transacciones
13. El puente: conectar el ORM al patrón Repository de Nest
14. Drizzle vs. Prisma (conocé la otra)

**Nivel senior — profundidad para producción**

15. Concurrencia: aislamiento de transacciones y locks
16. MVCC, VACUUM y bloat (Postgres por dentro)
17. Índices a fondo y leer un `EXPLAIN`
18. Paginación: offset vs. keyset
19. Connection pooling y PgBouncer
20. JSONB y full-text search

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

El dominio de ejemplo es el de la "Task API" del roadmap: **usuarios** que tienen **proyectos**, y proyectos que tienen **tareas**.

---

## Módulo 1 — Por qué relacional (y cuándo Postgres vs. Mongo)

**Teoría.** Una base de datos **relacional** guarda los datos en **tablas** (filas y columnas) con un **esquema fijo**: cada columna tiene un tipo y reglas. Su superpoder es **relacionar** datos entre tablas y garantizar **integridad** (no podés tener una tarea que apunte a un proyecto que no existe) y **transacciones ACID** (varias operaciones se confirman juntas o se revierten juntas). PostgreSQL es el default profesional en 2026: maduro, robusto, con tipos avanzados (JSON, arrays, full-text search) y excelente soporte de TypeScript vía ORMs.

**MongoDB** (documental) guarda documentos tipo JSON sin esquema fijo. Es flexible para datos heterogéneos o que evolucionan rápido, pero **vos** cargás con la consistencia y las relaciones a mano. Aparece en el stack MERN y en algunas startups.

La regla práctica: **empezá con Postgres salvo que tengas una razón concreta para no hacerlo.** La mayoría de los productos (que tienen usuarios, pedidos, relaciones, dinero) quieren las garantías relacionales. Si el modelo de datos tiene relaciones y necesita consistencia, es relacional.

| | Relacional (Postgres) | Documental (Mongo) |
|---|---|---|
| Esquema | Fijo, validado por la DB | Flexible, validás vos |
| Relaciones | Nativas (foreign keys, joins) | Manuales (embebido o refs) |
| Transacciones | ACID, su fuerte | ACID multi-documento desde 4.0, pero el modelo favorece el documento único |
| Cuándo | El default; datos con relaciones | Datos heterogéneos, sin relaciones fuertes |

**Ejercicios 1**
1.1 En una frase: ¿qué garantía te da una base relacional que tendrías que implementar a mano si guardaras tareas en un array en memoria?
1.2 Para una app de e-commerce con usuarios, productos, pedidos y pagos, ¿elegirías Postgres o Mongo? Justificá en una o dos frases.
1.3 ¿Qué significa que una operación sea "ACID" en el contexto de transferir saldo entre dos cuentas? Mencioná al menos la parte de atomicidad.

---

## Módulo 2 — Modelado: tablas, tipos y clave primaria

**Teoría.** Modelar es decidir **qué tablas** existen y **qué columnas** tiene cada una. Cada tabla representa una "cosa" del dominio (una entidad): usuarios, proyectos, tareas. Cada fila es una instancia; cada columna, un atributo con un **tipo**.

La **clave primaria** (`PRIMARY KEY`) identifica de forma única cada fila. Dos enfoques comunes:

- **`serial` / `bigserial`**: entero autoincremental (1, 2, 3…). Simple y compacto.
- **`uuid`**: identificador aleatorio (`a3f1...`). No revela cuántos registros hay ni es adivinable; bueno para IDs que viajan al cliente.

> En proyectos nuevos (2026) se prefiere `GENERATED ALWAYS AS IDENTITY` sobre `serial`: es el estándar SQL y evita problemas de permisos sobre la secuencia. `serial` es la forma clásica que aún vas a ver mucho. Para `uuid`, generalos con `gen_random_uuid()` (nativo desde PG13). Ojo: un UUID **v4** como clave primaria fragmenta el índice (es aleatorio, las inserciones caen en cualquier lado del B-tree); si querés UUID, preferí **UUIDv7/ULID**, que son ordenables en el tiempo.

```sql
CREATE TABLE usuarios (
  id          SERIAL PRIMARY KEY,
  email       TEXT NOT NULL UNIQUE,        -- no se puede repetir ni dejar vacío
  nombre      TEXT NOT NULL,
  edad        INTEGER,                      -- puede ser NULL (opcional)
  creado_el   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

Tipos más usados en Postgres: `INTEGER`/`BIGINT`, `TEXT` (string sin límite; preferilo a `VARCHAR(n)` salvo que necesites el límite), `BOOLEAN`, `TIMESTAMPTZ` (fecha-hora con zona horaria — usá **siempre** la variante con `TZ`), `NUMERIC(p,s)` para dinero (nunca `FLOAT` para plata: introduce errores de redondeo), `JSONB`, y arrays.

Las **restricciones** (`constraints`) son reglas que la DB hace cumplir por vos: `NOT NULL` (obligatorio), `UNIQUE` (sin duplicados), `DEFAULT` (valor por defecto), `CHECK` (condición a medida). Esto es validación que vive en la base, la última línea de defensa además de tus DTOs.

**Ejercicios 2**
2.1 Escribí el `CREATE TABLE proyectos` con: `id` (clave primaria autoincremental), `nombre` (texto obligatorio), `archivado` (booleano, por defecto `false`) y `creado_el` (timestamp con zona, por defecto ahora).
2.2 ¿Por qué `NUMERIC(10,2)` es mejor que `FLOAT` para guardar un precio? Respondé en una frase.
2.3 Agregá a la tabla `usuarios` una restricción `CHECK` para que `edad`, si está presente, sea mayor o igual a 18.

---

## Módulo 3 — Relaciones: claves foráneas (1:N y N:M)

**Teoría.** Una **clave foránea** (`FOREIGN KEY`, FK) es una columna que apunta a la clave primaria de otra tabla. Es lo que **conecta** las tablas y garantiza **integridad referencial**: Postgres no te deja crear una tarea con un `proyecto_id` que no existe.

**Relación uno-a-muchos (1:N)** — la más común. Un proyecto tiene muchas tareas; cada tarea pertenece a un proyecto. La FK va en el lado "muchos" (la tarea):

```sql
CREATE TABLE tareas (
  id           SERIAL PRIMARY KEY,
  titulo       TEXT NOT NULL,
  completada   BOOLEAN NOT NULL DEFAULT false,
  proyecto_id  INTEGER NOT NULL REFERENCES proyectos(id) ON DELETE CASCADE
);
```

`ON DELETE CASCADE` significa: si borrás el proyecto, se borran sus tareas automáticamente. Alternativas: `ON DELETE RESTRICT` (no te deja borrar el proyecto si tiene tareas) o `ON DELETE SET NULL` (deja la tarea huérfana con `proyecto_id = NULL`). Elegir esto es una decisión de diseño, no un detalle.

**Relación muchos-a-muchos (N:M)** — una tarea puede tener muchas etiquetas y una etiqueta está en muchas tareas. No se puede expresar con una sola FK: se usa una **tabla intermedia** (join table) que tiene una FK a cada lado:

```sql
CREATE TABLE etiquetas (
  id     SERIAL PRIMARY KEY,
  nombre TEXT NOT NULL UNIQUE
);

CREATE TABLE tareas_etiquetas (
  tarea_id    INTEGER NOT NULL REFERENCES tareas(id) ON DELETE CASCADE,
  etiqueta_id INTEGER NOT NULL REFERENCES etiquetas(id) ON DELETE CASCADE,
  PRIMARY KEY (tarea_id, etiqueta_id)   -- clave primaria compuesta: evita duplicados
);
```

**Ejercicios 3**
3.1 Un usuario tiene muchos proyectos. ¿En qué tabla va la clave foránea y por qué? Escribí la columna FK que agregarías.
3.2 ¿Qué hace `ON DELETE CASCADE` y en qué caso de los proyectos/tareas tiene sentido? ¿Cuándo elegirías `RESTRICT` en su lugar?
3.3 Modelá una relación N:M entre `usuarios` y `proyectos` (un proyecto compartido por varios usuarios colaboradores). Escribí la tabla intermedia.

---

## Módulo 4 — Normalización (lo justo y necesario)

**Teoría.** **Normalizar** es organizar las tablas para que **cada dato viva en un solo lugar**, evitando duplicación. La regla práctica que cubre el 95% de los casos: si estás repitiendo el mismo valor en muchas filas (el nombre del proyecto copiado en cada tarea), eso debería ser su propia tabla, referenciada por una FK.

El problema de duplicar: si el nombre del proyecto está copiado en 200 tareas y cambia, tenés que actualizar 200 filas, y si te olvidás de una, quedan **inconsistentes** (anomalía de actualización). Con una FK, el nombre vive una vez en `proyectos` y las tareas lo referencian por `proyecto_id`.

```sql
-- MAL (desnormalizado): el nombre del proyecto se repite y se puede desincronizar
tareas(id, titulo, nombre_proyecto, dueño_email, dueño_nombre)

-- BIEN (normalizado): cada cosa en su tabla, conectadas por FK
usuarios(id, email, nombre)
proyectos(id, nombre, usuario_id → usuarios.id)
tareas(id, titulo, proyecto_id → proyectos.id)
```

**Cuándo NO normalizar (desnormalización deliberada).** A veces, por rendimiento, se duplica un dato a propósito (ej. guardar un `total` calculado en vez de sumarlo siempre, o un contador de tareas). Es una optimización válida **cuando medís** que la query normalizada es un cuello de botella — pero es una decisión consciente con su costo (mantener el dato sincronizado), no el punto de partida. **Normalizá primero; desnormalizá con datos en la mano**, igual que el "medí antes de optimizar" del archivo senior.

**Ejercicios 4**
4.1 Tenés una tabla `tareas(id, titulo, etiqueta1, etiqueta2, etiqueta3)`. ¿Qué problema tiene este diseño y cómo lo normalizás?
4.2 ¿Qué es una "anomalía de actualización"? Dá un ejemplo con el nombre de un proyecto duplicado en muchas tareas.
4.3 ¿En qué caso justificarías guardar un campo `cantidad_tareas` precalculado en la tabla `proyectos` en vez de contarlas con una query? Mencioná el costo que asumís.

---

## Módulo 5 — SQL esencial: `SELECT`, `WHERE`, `ORDER BY`, `LIMIT`

**Teoría.** `SELECT` es cómo leés datos. La forma básica:

```sql
SELECT columnas FROM tabla WHERE condición ORDER BY columna LIMIT n;
```

```sql
-- Todas las columnas de todos los usuarios
SELECT * FROM usuarios;

-- Solo algunas columnas, filtrando
SELECT id, email FROM usuarios WHERE edad >= 18;

-- Ordenar (descendente) y limitar
SELECT * FROM tareas WHERE completada = false ORDER BY creado_el DESC LIMIT 10;
```

Operadores de `WHERE`: `=`, `<>` (distinto), `<`, `>`, `<=`, `>=`, `AND`, `OR`, `NOT`, `IN (...)`, `BETWEEN a AND b`, `LIKE 'ana%'` (patrón; `%` = cualquier cosa), `IS NULL` / `IS NOT NULL`.

**Cuidado con `NULL`.** `NULL` significa "desconocido", no "cero" ni "vacío". Por eso `edad = NULL` **nunca** es verdadero; se chequea con `IS NULL`. Esto sorprende a todo el mundo al principio.

`INSERT`, `UPDATE`, `DELETE` modifican datos:

```sql
INSERT INTO usuarios (email, nombre) VALUES ('ana@test.com', 'Ana') RETURNING id;
UPDATE tareas SET completada = true WHERE id = 5;
DELETE FROM tareas WHERE completada = true;
```

`RETURNING` te devuelve las filas afectadas (útil para obtener el `id` recién generado). **Regla de oro:** un `UPDATE` o `DELETE` sin `WHERE` afecta **toda la tabla**. Pensalo dos veces.

**Ejercicios 5**
5.1 Escribí la query que traiga el `email` y el `nombre` de los usuarios mayores o iguales a 18 años, ordenados por nombre ascendente.
5.2 Escribí la query que traiga las 5 tareas no completadas más recientes (asumí que existe `creado_el`).
5.3 ¿Por qué `SELECT * FROM usuarios WHERE edad = NULL` no devuelve los usuarios sin edad? ¿Cómo lo escribís correctamente?

---

## Módulo 6 — `JOIN`: combinar tablas

**Teoría.** Un `JOIN` combina filas de dos tablas usando una condición (típicamente la FK). Es lo que te permite responder "traeme las tareas **con el nombre de su proyecto**".

```sql
SELECT tareas.titulo, proyectos.nombre AS proyecto
FROM tareas
INNER JOIN proyectos ON tareas.proyecto_id = proyectos.id;
```

Tipos de join (los dos que más vas a usar):

- **`INNER JOIN`**: solo las filas que tienen pareja en **ambas** tablas. Una tarea sin proyecto válido no aparece.
- **`LEFT JOIN`**: **todas** las filas de la tabla izquierda, con `NULL` en las columnas de la derecha si no hay pareja. Útil para "todos los proyectos, tengan o no tareas".

```sql
-- Todos los proyectos, incluso los que no tienen tareas (saldrían con tarea = NULL)
SELECT proyectos.nombre, tareas.titulo
FROM proyectos
LEFT JOIN tareas ON tareas.proyecto_id = proyectos.id;
```

Para tablas con nombres largos se usan **alias**: `FROM tareas t INNER JOIN proyectos p ON t.proyecto_id = p.id`.

Entender joins es **lo que separa** a quien depende ciegamente del ORM de quien sabe lo que el ORM hace por debajo (módulo 11). Cada vez que tu ORM trae "una tarea con su proyecto", está haciendo un join.

**Ejercicios 6**
6.1 Escribí una query que traiga el título de cada tarea junto con el email del dueño del proyecto al que pertenece (necesitás unir `tareas → proyectos → usuarios`).
6.2 ¿Cuál es la diferencia entre `INNER JOIN` y `LEFT JOIN`? Dá un caso donde necesites el `LEFT`.
6.3 Querés listar todos los usuarios y, al lado, el nombre de sus proyectos (un usuario sin proyectos igual debe aparecer). ¿Qué join usás y por qué?

---

## Módulo 7 — Agregaciones: `GROUP BY`, `COUNT`, `SUM`, `HAVING`

**Teoría.** Las **funciones de agregación** resumen muchas filas en un número: `COUNT` (cuántas), `SUM` (suma), `AVG` (promedio), `MIN`/`MAX`. Solas operan sobre toda la tabla:

```sql
SELECT COUNT(*) FROM tareas WHERE completada = false;   -- cuántas pendientes hay
```

`GROUP BY` agrupa las filas por un valor y aplica la agregación **a cada grupo**. "Cuántas tareas tiene cada proyecto":

```sql
SELECT proyecto_id, COUNT(*) AS total_tareas
FROM tareas
GROUP BY proyecto_id;
```

Regla mental: lo que ponés en `SELECT` junto a una agregación tiene que estar en el `GROUP BY` (o ser una agregación). No podés mezclar una columna "suelta" con un `COUNT(*)` sin agruparla.

**`NULL` en agregaciones (trampa clásica):** `COUNT(*)` cuenta filas, pero `COUNT(columna)` **ignora los `NULL`** (y `SUM`/`AVG` también los descartan, no los cuentan como cero). Para contar valores distintos usás `COUNT(DISTINCT columna)`. Confundir `COUNT(*)` con `COUNT(col)` es una fuente típica de números mal calculados.

`WHERE` vs `HAVING`: `WHERE` filtra filas **antes** de agrupar; `HAVING` filtra **grupos** después de agregar. "Proyectos con más de 5 tareas":

```sql
SELECT proyecto_id, COUNT(*) AS total
FROM tareas
GROUP BY proyecto_id
HAVING COUNT(*) > 5;
```

**Ejercicios 7**
7.1 Escribí la query que cuente cuántas tareas completadas hay por proyecto (`proyecto_id`, `completadas`).
7.2 Escribí la query que devuelva los `proyecto_id` que tengan **al menos** 3 tareas sin completar.
7.3 ¿Por qué `WHERE COUNT(*) > 5` da error y hay que usar `HAVING`? Explicá la diferencia en una frase.

---

## Módulo 8 — Índices y el problema de rendimiento

**Teoría.** Un **índice** es una estructura auxiliar que acelera las búsquedas, como el índice de un libro: en vez de leer toda la tabla fila por fila (`Seq Scan`), Postgres va directo. Sin índice, buscar un usuario por email en una tabla de 1 millón de filas lee el millón. Con índice, es casi instantáneo.

```sql
CREATE INDEX idx_tareas_proyecto ON tareas(proyecto_id);
```

**Qué conviene indexar:** columnas por las que filtrás (`WHERE`), unís (`JOIN`, las FK) u ordenás (`ORDER BY`) seguido. Las **claves primarias** y las columnas `UNIQUE` ya vienen indexadas automáticamente.

**Por qué no indexar todo:** cada índice ocupa espacio y, sobre todo, **ralentiza las escrituras** (`INSERT`/`UPDATE`/`DELETE` tienen que actualizar también el índice). Es un intercambio: lecturas más rápidas a cambio de escrituras un poco más lentas y más disco. Indexá lo que realmente consultás, no por las dudas — el mismo criterio de "complejidad accidental" del archivo senior.

**Cómo saber si una query usa un índice:** `EXPLAIN ANALYZE` te muestra el plan de ejecución. Si ves `Seq Scan` sobre una tabla grande en una query frecuente, probablemente falta un índice; si ves `Index Scan`, lo está usando.

```sql
EXPLAIN ANALYZE SELECT * FROM tareas WHERE proyecto_id = 42;
```

La diferencia entre una query de 5ms y una de 5s suele ser, literalmente, un índice (lo viste mencionado en el archivo senior, sección de performance).

**Ejercicios 8**
8.1 Tu endpoint `GET /usuarios/buscar?email=...` está lento con muchos usuarios. ¿Qué índice crearías? (Pensá si `email` ya tiene uno.)
8.2 Nombrá dos costos de agregar un índice. ¿Por qué no conviene indexar todas las columnas?
8.3 ¿Qué comando usás para ver si una query está haciendo `Seq Scan` o `Index Scan`?

---

## Módulo 9 — Migraciones: control de versiones de tu esquema

**Teoría.** Una **migración** es un archivo versionado que describe un cambio en el esquema (crear una tabla, agregar una columna, un índice). Son el "git de tu base de datos": en vez de tocar la DB a mano (irreproducible y peligroso), escribís migraciones que se aplican en orden, igual en tu máquina, en la de un compañero y en producción.

Por qué importan:

- **Reproducibilidad:** cualquiera levanta la DB desde cero corriendo las migraciones en orden.
- **Historial:** sabés qué cambió y cuándo, y podés revertir.
- **Equipo y deploy:** el mismo cambio se aplica idéntico en todos los entornos; el pipeline de CI/CD las corre antes de arrancar la app.

El flujo típico con un ORM como Drizzle: modificás el `schema.ts`, **generás** la migración (el ORM compara tu schema con el estado actual y escribe el SQL del diff) y la **aplicás**:

```bash
npx drizzle-kit generate    # genera el SQL de la migración a partir del schema
npx drizzle-kit migrate     # aplica las migraciones pendientes a la DB
```

Esto genera archivos `.sql` con el `CREATE TABLE` / `ALTER TABLE` correspondiente, que **sí** se commitean al repo. **Nunca** edites una migración ya aplicada en producción: creá una nueva. Y antes de aplicar una migración destructiva (borrar una columna) en prod, pensá en los datos que perdés.

**Ejercicios 9**
9.1 ¿Por qué es mala idea agregar una columna conectándote a la base de producción y escribiendo `ALTER TABLE` a mano? Dá dos razones.
9.2 Tenés que agregar una columna `prioridad` a `tareas`. Describí el flujo de pasos con un ORM (del schema al deploy), sin escribir código.
9.3 ¿Por qué las migraciones se commitean al repositorio junto con el código?

---

## Módulo 10 — Drizzle: schema y queries tipadas

**Teoría.** **Drizzle** es un ORM TypeScript "SQL-first": definís tu esquema en TS y escribís queries que se parecen al SQL pero **tipadas de punta a punta**. Si tu query pide una columna que no existe, el compilador te frena — el mismo `--strict` que venís usando, ahora también sobre la base de datos. Es ligero y la elección recomendada para proyectos nuevos en 2026.

Definís el schema en TS (esto reemplaza el `CREATE TABLE` a mano y es la fuente de las migraciones del módulo 9):

```ts
import { pgTable, serial, text, integer, boolean, timestamp } from "drizzle-orm/pg-core";

export const usuarios = pgTable("usuarios", {
  id: serial("id").primaryKey(),
  email: text("email").notNull().unique(),
  nombre: text("nombre").notNull(),
  creadoEl: timestamp("creado_el", { withTimezone: true }).notNull().defaultNow(),
});

export const proyectos = pgTable("proyectos", {
  id: serial("id").primaryKey(),
  nombre: text("nombre").notNull(),
  usuarioId: integer("usuario_id")
    .notNull()
    .references(() => usuarios.id, { onDelete: "cascade" }),
});
```

Conectás y creás el cliente `db` una vez:

```ts
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
export const db = drizzle(pool);
```

Y escribís queries tipadas con helpers (`eq`, `and`, `desc`…):

```ts
import { eq, and, desc } from "drizzle-orm";

// SELECT ... WHERE id = 1
const [usuario] = await db.select().from(usuarios).where(eq(usuarios.id, 1));

// INSERT ... RETURNING
const [nuevo] = await db
  .insert(usuarios)
  .values({ email: "ana@test.com", nombre: "Ana" })
  .returning();

// UPDATE ... WHERE
await db.update(proyectos).set({ nombre: "Nuevo nombre" }).where(eq(proyectos.id, 3));

// DELETE ... WHERE
await db.delete(proyectos).where(eq(proyectos.id, 3));
```

Fijate que `nuevo` ya está tipado como el tipo de la fila de `usuarios`: `nuevo.email` es `string`, `nuevo.id` es `number`. Ese tipo lo infiere Drizzle de tu schema, sin que lo escribas.

**Ejercicios 10**
10.1 Definí con Drizzle la tabla `tareas` con: `id` (serial PK), `titulo` (texto obligatorio), `completada` (boolean, default `false`) y `proyectoId` (FK a `proyectos.id`, con borrado en cascada).
10.2 Escribí la query Drizzle que traiga todas las tareas **no completadas** de un `proyectoId` dado, ordenadas por `id` descendente. Pista: `and`, `eq`, `desc`.
10.3 Escribí la query Drizzle que inserte un proyecto y devuelva su `id` generado.

---

## Módulo 11 — Relaciones en Drizzle y el problema N+1

**Teoría.** Para traer datos relacionados (un proyecto **con** sus tareas) con la API declarativa de Drizzle, primero declarás las relaciones y se las pasás al cliente:

```ts
import { relations } from "drizzle-orm";

export const proyectosRelations = relations(proyectos, ({ many, one }) => ({
  tareas: many(tareas),
  dueño: one(usuarios, { fields: [proyectos.usuarioId], references: [usuarios.id] }),
}));

export const tareasRelations = relations(tareas, ({ one }) => ({
  proyecto: one(proyectos, { fields: [tareas.proyectoId], references: [proyectos.id] }),
}));

// al crear el cliente, le pasás el schema con las relaciones:
// export const db = drizzle(pool, { schema });
```

Y consultás con `with` (Drizzle lo resuelve en pocas queries planificadas, **no** en N+1 — conviene verificar en los logs cuántas emite tu versión):

```ts
const proyectoConTareas = await db.query.proyectos.findFirst({
  where: eq(proyectos.id, 1),
  with: { tareas: true },
});
```

**El problema N+1** (el que aparece en el archivo senior). Es la causa #1 de lentitud con ORMs. Ocurre cuando traés una lista de N proyectos y después, en un loop, hacés **una query por cada uno** para sus tareas: 1 (la lista) + N (una por proyecto) = **N+1 queries**.

```ts
// MAL: N+1. Si hay 100 proyectos, son 101 viajes a la base.
const todos = await db.select().from(proyectos);
for (const p of todos) {
  const tareas = await db.select().from(tareas).where(eq(tareas.proyectoId, p.id));
  // ...
}

// BIEN: una sola llamada que el ORM resuelve de forma eficiente (no una query por fila).
const todos = await db.query.proyectos.findMany({ with: { tareas: true } });
```

Cómo detectarlo: si activás el log de SQL y ves la **misma query repetida muchas veces** con distinto `id`, tenés un N+1. La solución es traer todo junto (con `with`/join) en vez de iterar. Saber leer los logs de SQL para cazar esto es, textualmente, "básico Senior".

**Ejercicios 11**
11.1 Explicá el problema N+1 con tus palabras usando el caso "listar usuarios y sus proyectos". ¿Cuántas queries se hacen mal y cuántas bien?
11.2 Tenés un código que trae todos los usuarios y dentro de un `for` consulta los proyectos de cada uno. ¿Cómo lo reescribirías para evitar N+1?
11.3 ¿Cómo te darías cuenta, mirando los logs de la base, de que tu endpoint sufre un N+1?

---

## Módulo 12 — Transacciones

**Teoría.** Una **transacción** agrupa varias operaciones para que se confirmen **todas juntas o ninguna** (atomicidad). El ejemplo canónico: al confirmar un pedido querés descontar stock **y** crear el pedido; si una falla, la otra no debe quedar a medias. Es la garantía ACID del módulo 1, ahora en acción.

En Drizzle, `db.transaction` te da un objeto `tx` que usás en lugar de `db`. Si el callback lanza una excepción, Drizzle hace **rollback** automático; si termina bien, **commit**:

```ts
await db.transaction(async (tx) => {
  // mover una tarea de un proyecto a otro y marcar el destino como activo:
  await tx.update(tareas).set({ proyectoId: 2 }).where(eq(tareas.id, 10));
  await tx.update(proyectos).set({ archivado: false }).where(eq(proyectos.id, 2));
  // si cualquiera de las dos lanza, NINGUNA queda aplicada (rollback)
});
```

Esto es exactamente el **Unit of Work** del archivo senior: el caso de uso abre la transacción y le pasa `tx` al repositorio; el dominio ni se entera de que hubo transacción. La regla de oro de aquella sección sigue valiendo: **mientras todo viva en una sola base, tenés transacciones ACID gratis; aprovechalo y no repartas a la ligera datos que necesitan cambiar juntos.**

Cuándo necesitás una transacción: siempre que **dos o más escrituras** deban ser consistentes entre sí. Cuándo no: una sola operación de escritura ya es atómica por sí misma; envolverla en una transacción es ruido.

**Ejercicios 12**
12.1 ¿Qué pasa si, dentro de un `db.transaction`, la segunda operación lanza una excepción? ¿Qué queda guardado?
12.2 Escribí una transacción Drizzle que cree un proyecto y, en la misma operación, cree una primera tarea asociada a ese proyecto. Pista: necesitás el `id` del proyecto recién creado (`returning`) para la FK de la tarea.
12.3 ¿Necesitás una transacción para un único `UPDATE tareas SET completada = true WHERE id = 5`? Justificá.

---

## Módulo 13 — El puente: conectar el ORM al patrón Repository de Nest

**Teoría.** Acá se cierra el círculo con todo lo que ya sabés de NestJS. En el módulo de Repository definiste una **interfaz** (`UserRepository`) y una implementación **en memoria** (`InMemoryUserRepository`). El service dependía de la interfaz por un token, no de la implementación. Ahora escribimos una implementación **real con Drizzle** que cumple la misma interfaz — y el service **no cambia ni una línea**. Eso es la inversión de dependencias rindiendo de verdad.

La interfaz (el puerto) es la misma de antes:

```ts
export interface UserRepository {
  findById(id: number): Promise<Usuario | null>;
  save(u: NuevoUsuario): Promise<Usuario>;
}
```

La nueva implementación, con Drizzle en vez de un `Map`:

```ts
@Injectable()
export class DrizzleUserRepository implements UserRepository {
  constructor(@Inject("DB") private readonly db: NodePgDatabase) {}

  async findById(id: number): Promise<Usuario | null> {
    const [u] = await this.db.select().from(usuarios).where(eq(usuarios.id, id));
    return u ?? null;
  }

  async save(u: NuevoUsuario): Promise<Usuario> {
    const [creado] = await this.db.insert(usuarios).values(u).returning();
    return creado;
  }
}
```

Y en el módulo, cambiás **solo el provider del token** — de la implementación en memoria a la de Drizzle:

```ts
@Module({
  providers: [
    UsersService,
    { provide: "USER_REPO", useClass: DrizzleUserRepository }, // antes: InMemoryUserRepository
  ],
})
export class UsersModule {}
```

El `UsersService`, sus tests unitarios (que siguen usando el repo en memoria como mock) y los controladores quedan **intactos**. En tests unitarios usás el `Map`; en producción, Postgres. El service no se entera. Esta es la razón concreta por la que valió la pena la abstracción: cambiar de "en memoria" a "base real" es cambiar una línea de configuración.

**Detalle de producción:** cuando una inserción viola una constraint `UNIQUE` (ej. email repetido), Postgres lanza el error con código `23505`. En el repositorio conviene **capturarlo y traducirlo a una excepción de dominio** (`ConflictException` → HTTP 409) en vez de dejar que escape como un 500 genérico. Lo mismo aplica a `created_at`/`updated_at`: resolvelos con `DEFAULT now()` y, para el `updated_at`, un trigger o el `$onUpdate` de Drizzle, en vez de setearlos a mano en cada query.

**Ejercicios 13**
13.1 ¿Qué archivos cambian y cuáles **no** al pasar de `InMemoryUserRepository` a `DrizzleUserRepository`? ¿Por qué el service queda intacto?
13.2 Escribí una `DrizzleProjectRepository` que implemente `findById(id)` y `save(p)` para `proyectos`, inyectando el cliente `db` por el token `"DB"`.
13.3 En tus tests unitarios del `UsersService`, ¿seguís usando el repo en memoria o el de Drizzle? Justificá según lo que viste en el archivo de testing senior (qué mockear y qué no).

---

## Módulo 14 — Drizzle vs. Prisma (conocé la otra)

**Teoría.** El roadmap es claro: **dominá una, conocé la otra.** Ya viste Drizzle en profundidad; esto es para que sepas ubicar Prisma en una entrevista, no para aprenderlo ahora.

**Prisma** es el otro ORM dominante. Es "schema-first": definís tu modelo en un archivo propio (`schema.prisma`, no TypeScript) y Prisma **genera** un cliente tipado a partir de él. Es más maduro, con muchísima documentación y tutoriales, y una API muy ergonómica.

| | **Drizzle** | **Prisma** |
|---|---|---|
| Estilo | SQL-first; las queries se parecen al SQL | Abstracción más alta; API propia |
| Schema | En TypeScript (`schema.ts`) | DSL propio (`schema.prisma`) |
| Tipos | Inferidos del schema TS | Cliente generado en un paso de build |
| Curva | Más cerca del SQL (bueno si entendés SQL) | Más "mágica", arranque más rápido |
| Peso/runtime | Ligero, pensado para serverless/edge | Más pesado; mejoró mucho |
| Cuándo | Proyectos nuevos 2026, control del SQL | Ecosistema, equipos que ya lo usan, prototipado veloz |

La decisión Senior no es "cuál es mejor" (ambos son buenas opciones), sino **encajar con el contexto**: control fino del SQL y peso liviano → Drizzle; ergonomía, madurez y un equipo que ya lo conoce → Prisma. Lo que **no** cambia entre uno y otro es todo lo de esta guía: el modelado, el SQL, los índices, las transacciones y el patrón Repository que los aísla de tu dominio. Si mañana cambiás de ORM, cambiás la implementación del repositorio; el resto de tu app no se entera. Otra vez, la abstracción se paga sola.

**Ejercicios 14**
14.1 Nombrá dos diferencias concretas entre Drizzle y Prisma.
14.2 Un equipo arranca un proyecto nuevo en 2026, valora el control del SQL y desplegará en serverless. ¿Cuál sugerís y por qué?
14.3 Si tu app usa el patrón Repository y querés migrar de Prisma a Drizzle, ¿qué parte del código tenés que reescribir y qué parte queda igual?

---

# Nivel senior — profundidad para producción

Los módulos 1-14 te dan una base sólida. Los que siguen son los temas que separan "sé usar una base" de "entiendo Postgres en producción", y son justo los que más se preguntan en entrevistas mid/senior.

## Módulo 15 — Concurrencia: aislamiento de transacciones y locks

**Teoría.** El módulo 12 mostró la **atomicidad** ("todo o nada"). Pero una transacción tiene una segunda cara igual de importante: el **aislamiento** (la "I" de ACID), o sea qué ve una transacción de los cambios de otras que corren **al mismo tiempo**. Y acá está el error de seniority más común: creer que `db.transaction(...)` te protege de condiciones de carrera. **No lo hace por sí solo.**

El default de Postgres es **READ COMMITTED**: cada sentencia ve los datos confirmados (committed) hasta el momento en que esa sentencia empieza. Esto deja pasar el **lost update** (actualización perdida) en el patrón clásico *leer → modificar → escribir*:

```sql
-- Dos requests descuentan stock a la vez (READ COMMITTED):
-- T1: SELECT stock FROM productos WHERE id = 1;   -- lee 10
-- T2: SELECT stock FROM productos WHERE id = 1;   -- lee 10
-- T1: UPDATE productos SET stock = 9 WHERE id = 1;
-- T2: UPDATE productos SET stock = 9 WHERE id = 1;  -- ¡quedó 9; debería ser 8!
```

Tres formas de arreglarlo, de más simple a más fuerte:

```sql
-- (1) Cálculo atómico en la propia DB (lo más simple cuando se puede):
UPDATE productos SET stock = stock - 1 WHERE id = 1 AND stock > 0;

-- (2) Lock pesimista: T2 espera a que T1 confirme antes de leer
SELECT stock FROM productos WHERE id = 1 FOR UPDATE;

-- (3) Subir el aislamiento y reintentar ante conflicto (lock optimista a nivel DB)
--     BEGIN ISOLATION LEVEL SERIALIZABLE; ... ; -- si falla con error 40001, reintentás
```

Los **niveles de aislamiento** y qué anomalía previene cada uno:

| Nivel | Previene |
|---|---|
| **READ COMMITTED** (default) | dirty reads (no ves cambios sin confirmar) |
| **REPEATABLE READ** | + non-repeatable reads y phantom reads (trabajás sobre un *snapshot* fijo del inicio de la transacción) |
| **SERIALIZABLE** | + anomalías de serialización: el resultado es **como si** las transacciones hubieran corrido una tras otra |

En Postgres, `REPEATABLE READ` ya usa un snapshot que evita lecturas no repetibles y phantoms; `SERIALIZABLE` agrega la detección de anomalías y puede **abortar** una transacción con error `40001` (`serialization_failure`) que tu código **debe reintentar**.

**Locks y deadlocks.** El `SELECT ... FOR UPDATE` toma un **lock de fila** (otros que quieran esa fila esperan). Un **deadlock** ocurre cuando dos transacciones se traban mutuamente: T1 tiene la fila A y pide la B, T2 tiene la B y pide la A. Postgres lo detecta y **mata una** con error `40P01` (`deadlock_detected`). La forma de evitarlos: tomar los locks **siempre en el mismo orden** en todo el código.

**Ejercicios 15**
15.1 Dos requests hacen `SELECT saldo` → suman 100 en memoria → `UPDATE saldo`. ¿Qué anomalía aparece bajo READ COMMITTED y cómo se llama? Escribí una versión del `UPDATE` que la evite sin locks.
15.2 ¿Para qué sirve `SELECT ... FOR UPDATE` y qué problema del ejercicio anterior resuelve?
15.3 ¿Qué es un deadlock y cuál es la regla práctica para prevenirlo?

---

## Módulo 16 — MVCC, VACUUM y bloat (Postgres por dentro)

**Teoría.** ¿Cómo hace Postgres para que los lectores no bloqueen a los escritores (ni al revés)? Con **MVCC** (Multi-Version Concurrency Control). La idea: Postgres **no actualiza las filas en el lugar**. Cada `UPDATE` crea una **nueva versión** de la fila (una tupla nueva) y marca la vieja como obsoleta; cada `DELETE` marca la tupla como muerta. Cada transacción ve la versión que le corresponde según su snapshot. Por eso una lectura larga no frena una escritura.

El costo de esto son las **tuplas muertas** (dead tuples): versiones viejas que quedan ocupando espacio. Si no se limpian, la tabla y sus índices se inflan — eso es el **bloat**.

- **`VACUUM`** (y el **autovacuum**, que corre solo) recupera el espacio de las tuplas muertas para reusarlo. Es mantenimiento normal y necesario.
- **`VACUUM FULL`** reescribe la tabla entera y devuelve el espacio al SO, pero **bloquea la tabla** — se usa con cuidado, fuera de hora pico.
- **`ANALYZE`** actualiza las **estadísticas** que el planner usa para decidir planes (clave para el módulo 17).

Consecuencias prácticas que un senior conoce:

- `COUNT(*)` en Postgres recorre las tuplas vivas (no hay un contador mágico): en tablas enormes puede ser lento, sobre todo con bloat. Para un aproximado rápido se puede mirar `reltuples` en `pg_class`.
- Tablas con muchísimos `UPDATE`/`DELETE` (colas, contadores) acumulan bloat rápido; conviene vigilar el autovacuum.
- Los índices también sufren bloat; a veces se hace `REINDEX`.

**Ejercicios 16**
16.1 ¿Por qué un `UPDATE` en Postgres no "pisa" la fila vieja? ¿Qué genera eso y cómo se llama el problema si se acumula?
16.2 ¿Qué hace el autovacuum y por qué es importante en una tabla con muchas escrituras?
16.3 ¿Por qué `COUNT(*)` puede ser lento en una tabla grande, a diferencia de lo que mucha gente asume?

---

## Módulo 17 — Índices a fondo y leer un `EXPLAIN`

**Teoría.** El módulo 8 introdujo el índice B-tree. Para producción necesitás más: **tipos de índice**, **índices compuestos** y, sobre todo, **leer un plan de ejecución**.

**Tipos de índice** (no todo es B-tree):

- **B-tree**: el default; sirve para `=`, `<`, `>`, `BETWEEN`, `ORDER BY`.
- **GIN**: para columnas con múltiples valores por fila — **JSONB**, arrays y **full-text search** (módulo 20).
- **GiST**: rangos, datos geométricos/geoespaciales (PostGIS).
- **BRIN**: tablas enormes y "append-only" naturalmente ordenadas (ej. logs por fecha); ocupa poquísimo.

**Índice compuesto y la regla del prefijo izquierdo.** Un índice en `(a, b)` sirve para filtrar por `a` o por `a AND b`, y para `ORDER BY a`, pero **no** para filtrar por `b` solo. El orden de las columnas importa: poné primero la que más filtra / por la que siempre filtrás.

```sql
CREATE INDEX idx_tareas_proy_estado ON tareas (proyecto_id, completada);
-- sirve para WHERE proyecto_id = 1  y  WHERE proyecto_id = 1 AND completada = false
-- NO sirve (bien) para WHERE completada = false  solo
```

**Índice parcial** (solo las filas que te importan, más chico y rápido):

```sql
CREATE INDEX idx_tareas_pendientes ON tareas (proyecto_id) WHERE completada = false;
```

**Covering index** (`INCLUDE`): permite un *index-only scan* sin ir a la tabla, si todas las columnas que pedís están en el índice.

**Leer un `EXPLAIN ANALYZE`.** No alcanza con "¿usa índice?". Hay que leer el plan:

```
EXPLAIN ANALYZE SELECT * FROM tareas WHERE proyecto_id = 42;

Index Scan using idx_tareas_proyecto on tareas
  (cost=0.42..8.45 rows=5 width=64) (actual time=0.018..0.021 rows=4 loops=1)
  Index Cond: (proyecto_id = 42)
Planning Time: 0.10 ms
Execution Time: 0.05 ms
```

- `cost` es una **estimación** en unidades arbitrarias (no ms); `actual time` es lo medido de verdad.
- `rows=5` (estimadas) vs `rows=4` (reales): si difieren **mucho**, tus estadísticas están viejas → corré `ANALYZE`. El planner elige mal con estadísticas malas.
- Tipos de nodo de join que vas a ver: **Nested Loop** (pocas filas), **Hash Join** (sets grandes sin orden), **Merge Join** (entradas ordenadas).

**Mito a matar:** `Seq Scan` **no siempre es malo**. Si la tabla es chica, o la query trae el 80% de las filas, leer todo secuencialmente es **más rápido** que saltar por un índice. El planner lo sabe y por eso a veces ignora un índice a propósito (por **selectividad**).

**Ejercicios 17**
17.1 Tenés un índice en `(proyecto_id, completada)`. ¿Para cuáles de estas queries sirve y para cuál no? (a) `WHERE proyecto_id = 1`; (b) `WHERE proyecto_id = 1 AND completada = false`; (c) `WHERE completada = false`.
17.2 En un `EXPLAIN ANALYZE`, las `rows` estimadas son 5 y las reales 50.000. ¿Qué problema sugiere y qué comando lo corrige?
17.3 ¿Por qué a veces un `Seq Scan` es la decisión correcta del planner y no un error a "arreglar" con un índice?

---

## Módulo 18 — Paginación: offset vs. keyset

**Teoría.** Casi todo endpoint de listado pagina. La forma intuitiva es `LIMIT/OFFSET`:

```sql
SELECT * FROM tareas ORDER BY id DESC LIMIT 20 OFFSET 10000;
```

El problema: `OFFSET` **no salta** mágicamente; Postgres tiene que **leer y descartar** las 10.000 filas anteriores para devolver las 20 siguientes. Es O(n): la página 1 es instantánea, la página 500 es lenta. En tablas grandes es un anti-patrón de performance conocido (y pregunta de entrevista).

La alternativa profesional es **keyset / cursor pagination**: en vez de "saltá 10.000", decís "dame lo que viene **después del último que vi**", usando una columna ordenada e indexada:

```sql
-- primera página
SELECT * FROM tareas ORDER BY id DESC LIMIT 20;
-- siguientes páginas: pasás el id del último visto
SELECT * FROM tareas WHERE id < :ultimoId ORDER BY id DESC LIMIT 20;
```

En Drizzle:

```ts
import { lt, desc } from "drizzle-orm";

await db.select().from(tareas)
  .where(lt(tareas.id, ultimoId))
  .orderBy(desc(tareas.id))
  .limit(20);
```

Keyset es **rápido y estable** (no se "corre" si insertan filas mientras paginás), pero requiere ordenar por una columna **única e indexada** (o una combinación que desempate, ej. `(creado_el, id)`). Su límite: no podés saltar a "la página 7" arbitraria — solo avanzar/retroceder. Para feeds e *infinite scroll* es ideal; para una grilla con números de página, a veces `OFFSET` (en datasets chicos) sigue siendo aceptable.

**Ejercicios 18**
18.1 ¿Por qué `OFFSET 100000 LIMIT 20` se vuelve lento aunque solo devuelvas 20 filas?
18.2 Escribí la query keyset (SQL o Drizzle) para "las 20 tareas siguientes después de la última que mostré", ordenadas por `id` descendente.
18.3 ¿Qué requisito tiene la columna por la que ordenás para que el keyset funcione bien, y qué limitación tiene frente a offset?

---

## Módulo 19 — Connection pooling y PgBouncer

**Teoría.** Cada conexión a Postgres **no es gratis**: el servidor lanza **un proceso** por conexión, con su memoria. Abrir y cerrar una conexión por cada request es caro, y hay un límite duro (`max_connections`, ~100 por defecto). Por eso usás un **pool**: un conjunto de conexiones ya abiertas que se **reutilizan**.

Eso es lo que hace `new Pool(...)` de la librería `pg` (el del módulo 10): mantiene conexiones vivas y te las presta. Si todas están ocupadas, el request **espera** una libre (o falla por timeout) — eso es **agotar el pool**, un síntoma típico cuando hay queries lentas que retienen conexiones.

**Dimensionar el pool** es un equilibrio: muy chico = cuello de botella artificial; muy grande × muchas instancias de tu app = superás el `max_connections` de Postgres y este empieza a rechazar conexiones. Regla útil: `pool_por_instancia × número_de_instancias` debe quedar holgadamente por debajo de `max_connections`.

**PgBouncer** entra cuando tenés **muchas** instancias o serverless (cada lambda/función abre sus propias conexiones y revienta el límite). Es un **pooler externo** que multiplexa miles de conexiones de cliente sobre unas pocas reales a Postgres. Tiene dos modos clave:

- **session mode**: una conexión real queda atada a la sesión del cliente mientras dura. Compatible con todo, pero menos eficiente.
- **transaction mode**: la conexión real se devuelve al pool **al terminar cada transacción** — muchísimo más eficiente, pero **rompe** features ligadas a la sesión: *prepared statements*, `SET`, `LISTEN/NOTIFY` y advisory locks de sesión. Hay que configurar el driver para no usarlas.

**Ejercicios 19**
19.1 ¿Por qué abrir una conexión nueva a Postgres por cada request es mala idea? ¿Qué resuelve el pool?
19.2 Tu app escala a 20 instancias, cada una con un pool de 10. Postgres tiene `max_connections = 100`. ¿Qué problema hay y cómo lo encarás?
19.3 ¿Qué gana y qué pierde PgBouncer en *transaction mode* frente a *session mode*?

---

## Módulo 20 — JSONB y full-text search

**Teoría.** Postgres no es "solo tablas": dos de sus superpoderes son **JSONB** y la **búsqueda de texto**.

**JSONB** guarda JSON binario, indexable y consultable. Sirve para datos **semiestructurados, opcionales o variables** que no querés (o no podés) modelar como columnas: metadata, payloads de webhooks, configuraciones por fila.

```sql
ALTER TABLE tareas ADD COLUMN metadata JSONB;

-- -> devuelve JSONB ; ->> devuelve TEXT ; @> "contiene"
SELECT metadata->>'origen' FROM tareas WHERE metadata @> '{"origen":"import"}';

-- se indexa con GIN para que esas búsquedas sean rápidas:
CREATE INDEX idx_tareas_meta ON tareas USING GIN (metadata);
```

El criterio senior: usá JSONB para lo flexible, **no para reemplazar el modelado relacional**. Si metés todo en un JSONB perdés constraints, foreign keys, tipos y queries claras. La regla: lo que tiene relaciones y reglas → columnas y tablas; lo verdaderamente variable → JSONB.

**Full-text search (FTS).** Para buscar palabras dentro de texto con stemming y ranking (no un `LIKE '%...%'`, que no escala ni entiende idioma), Postgres trae `tsvector`/`tsquery`:

```sql
SELECT * FROM tareas
WHERE to_tsvector('spanish', titulo) @@ to_tsquery('spanish', 'informe');

-- índice GIN sobre el tsvector para que sea rápido:
CREATE INDEX idx_tareas_fts ON tareas USING GIN (to_tsvector('spanish', titulo));
```

Alcanza para búsqueda integrada sin sumar infraestructura. Si necesitás algo más potente (relevancia avanzada, typo-tolerance, facetas), ahí sí entra un motor dedicado como Elasticsearch o Meilisearch — pero para la mayoría de las apps, FTS de Postgres sobra.

**Ejercicios 20**
20.1 ¿Cuándo conviene una columna `JSONB` y cuándo es mejor modelar con columnas/tablas relacionales? Dá un criterio.
20.2 Escribí la query que traiga las tareas cuyo `metadata` contenga `{"prioridad":"alta"}` y el índice que la acelera.
20.3 ¿Por qué `to_tsvector @@ to_tsquery` es mejor que `LIKE '%palabra%'` para buscar texto, y con qué tipo de índice se acelera?

---

# Soluciones

> Mirá esto solo después de intentarlo. Si tu solución difiere pero es correcta, probablemente también esté bien — en SQL hay más de un camino.

### Módulo 1
```
1.1 La integridad referencial: la base no te deja crear una tarea que apunte a un
    proyecto inexistente. En un array en memoria eso lo tendrías que chequear a mano
    en cada inserción (y nada te garantiza que no te lo saltees).

1.2 Postgres. Tiene relaciones fuertes (usuarios↔pedidos↔pagos), necesita
    integridad referencial y, sobre todo, transacciones ACID para el dinero
    (cobrar y registrar el pedido tienen que ser atómicos).

1.3 Atomicidad: las dos operaciones (restar de una cuenta, sumar a la otra) se
    confirman juntas o se revierten juntas. Nunca queda el dinero restado de una
    sin sumarse a la otra, aunque el proceso falle en el medio.
```

### Módulo 2
```sql
-- 2.1
CREATE TABLE proyectos (
  id         SERIAL PRIMARY KEY,
  nombre     TEXT NOT NULL,
  archivado  BOOLEAN NOT NULL DEFAULT false,
  creado_el  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 2.3
ALTER TABLE usuarios ADD CONSTRAINT edad_minima CHECK (edad >= 18);
-- (el CHECK no se evalúa cuando edad es NULL, así que los usuarios sin edad pasan)
```
```
2.2 Porque NUMERIC es de precisión exacta (decimal), mientras que FLOAT es binario
    y de punto flotante: introduce errores de redondeo (0.1 + 0.2 ≠ 0.3), inaceptables
    para dinero.
```

### Módulo 3
```
3.1 La FK va en proyectos (el lado "muchos": un usuario tiene muchos proyectos,
    cada proyecto pertenece a un usuario). Columna:
        usuario_id INTEGER NOT NULL REFERENCES usuarios(id)

3.2 ON DELETE CASCADE borra automáticamente las tareas cuando se borra su proyecto.
    Tiene sentido si las tareas no existen sin su proyecto (borrar el proyecto debe
    llevarse sus tareas). Elegirías RESTRICT si querés impedir borrar un proyecto que
    todavía tiene tareas (forzar a vaciarlo primero, como red de seguridad).
```
```sql
-- 3.3 (N:M usuarios ↔ proyectos colaboradores)
CREATE TABLE proyectos_colaboradores (
  proyecto_id INTEGER NOT NULL REFERENCES proyectos(id) ON DELETE CASCADE,
  usuario_id  INTEGER NOT NULL REFERENCES usuarios(id) ON DELETE CASCADE,
  PRIMARY KEY (proyecto_id, usuario_id)
);
```

### Módulo 4
```
4.1 No escala: solo permite 3 etiquetas, y buscar "tareas con la etiqueta X" obliga
    a revisar 3 columnas. Se normaliza con una tabla `etiquetas` y una intermedia
    `tareas_etiquetas` (N:M, como en el módulo 3).

4.2 Es cuando un dato duplicado queda inconsistente tras un cambio parcial. Ejemplo:
    el nombre del proyecto está copiado en 200 tareas; lo renombrás y actualizás 199;
    la fila olvidada queda con el nombre viejo y nadie sabe cuál es el correcto.

4.3 Lo justificarías si medís que contar las tareas en cada request es un cuello de
    botella real (ej. un dashboard muy consultado). El costo: tenés que mantener
    `cantidad_tareas` sincronizado en cada alta/baja de tarea (a mano o con un
    trigger), y si te olvidás, el contador miente.
```

### Módulo 5
```sql
-- 5.1
SELECT email, nombre FROM usuarios WHERE edad >= 18 ORDER BY nombre ASC;

-- 5.2
SELECT * FROM tareas WHERE completada = false ORDER BY creado_el DESC LIMIT 5;

-- 5.3
SELECT * FROM usuarios WHERE edad IS NULL;
-- "edad = NULL" nunca es verdadero: NULL significa "desconocido", y comparar con un
-- desconocido da NULL (no true). Por eso se usa IS NULL / IS NOT NULL.
```

### Módulo 6
```sql
-- 6.1
SELECT t.titulo, u.email
FROM tareas t
INNER JOIN proyectos p ON t.proyecto_id = p.id
INNER JOIN usuarios u  ON p.usuario_id = u.id;

-- 6.3
SELECT u.nombre, p.nombre AS proyecto
FROM usuarios u
LEFT JOIN proyectos p ON p.usuario_id = u.id;
-- LEFT JOIN: queremos TODOS los usuarios, incluso los que no tienen proyectos
-- (con INNER, un usuario sin proyectos desaparecería del resultado).
```
```
6.2 INNER JOIN solo devuelve filas con pareja en ambas tablas; LEFT JOIN devuelve
    todas las de la izquierda y rellena con NULL las de la derecha cuando no hay
    pareja. Necesitás LEFT, por ejemplo, para "todos los proyectos, tengan o no
    tareas" (un proyecto vacío igual debe aparecer).
```

### Módulo 7
```sql
-- 7.1
SELECT proyecto_id, COUNT(*) AS completadas
FROM tareas
WHERE completada = true
GROUP BY proyecto_id;

-- 7.2
SELECT proyecto_id, COUNT(*) AS pendientes
FROM tareas
WHERE completada = false
GROUP BY proyecto_id
HAVING COUNT(*) >= 3;
```
```
7.3 WHERE filtra filas ANTES de agrupar, cuando todavía no existe el resultado de
    COUNT(*); por eso no puede referirse a él. HAVING filtra los GRUPOS DESPUÉS de
    calcular la agregación, así que ahí sí podés usar COUNT(*).
```

### Módulo 8
```sql
-- 8.1
-- email ya tiene índice si lo declaraste UNIQUE (UNIQUE crea uno automáticamente).
-- Si NO es UNIQUE, lo creás:
CREATE INDEX idx_usuarios_email ON usuarios(email);

-- 8.3
EXPLAIN ANALYZE SELECT * FROM tareas WHERE proyecto_id = 42;
```
```
8.2 Costos: (1) ocupa espacio en disco; (2) ralentiza las escrituras, porque cada
    INSERT/UPDATE/DELETE tiene que actualizar también el índice. No conviene indexar
    todo porque pagás esos costos en columnas que quizá nunca consultás: indexá solo
    las que filtrás/unís/ordenás de verdad.
```

### Módulo 9
```
9.1 (1) No es reproducible: tu cambio queda solo en prod; nadie más (ni otro entorno)
    lo tiene, y se pierde el historial de qué cambió. (2) Es peligroso: un error a
    mano sin revisión (un ALTER mal escrito, un DROP) impacta directo en producción
    sin posibilidad de revisarlo en un PR ni revertirlo ordenadamente.

9.2 (1) Agregás la columna `prioridad` al schema.ts. (2) Generás la migración
    (el ORM escribe el ALTER TABLE del diff). (3) Revisás el SQL generado y lo
    commiteás. (4) En el deploy, el pipeline corre las migraciones pendientes contra
    la base del entorno antes de arrancar la nueva versión de la app.

9.3 Para que el cambio de esquema viaje junto con el código que lo usa: el mismo
    commit que agrega la columna trae la migración que la crea, y cualquiera (o el
    CI/CD) reconstruye la base exacta que ese código necesita corriéndolas en orden.
```

### Módulo 10
```ts
// 10.1
export const tareas = pgTable("tareas", {
  id: serial("id").primaryKey(),
  titulo: text("titulo").notNull(),
  completada: boolean("completada").notNull().default(false),
  proyectoId: integer("proyecto_id")
    .notNull()
    .references(() => proyectos.id, { onDelete: "cascade" }),
});

// 10.2
const pendientes = await db
  .select()
  .from(tareas)
  .where(and(eq(tareas.proyectoId, proyectoId), eq(tareas.completada, false)))
  .orderBy(desc(tareas.id));

// 10.3
const [proyecto] = await db
  .insert(proyectos)
  .values({ nombre: "Nuevo proyecto", usuarioId: 1 })
  .returning({ id: proyectos.id });
// proyecto.id : number
```

### Módulo 11
```
11.1 Listás N usuarios con 1 query, y después por cada usuario hacés otra query para
     sus proyectos: 1 + N. Con 50 usuarios son 51 queries (mal). Bien: una sola llamada
     que trae usuarios y proyectos juntos (con `with` o un JOIN) → pocas queries
     planificadas, no una por fila.

11.3 Activando el log de SQL del ORM y viendo la MISMA query repetida muchas veces,
     idéntica salvo el valor del id en el WHERE. Ese patrón repetido es la firma del
     N+1.
```
```ts
// 11.2
// MAL (N+1):
//   const us = await db.select().from(usuarios);
//   for (const u of us) {
//     const ps = await db.select().from(proyectos).where(eq(proyectos.usuarioId, u.id));
//   }
// BIEN (1 query con la relación declarada):
const conProyectos = await db.query.usuarios.findMany({
  with: { proyectos: true },
});
```

### Módulo 12
```
12.1 Drizzle hace rollback automático: NADA queda guardado, ni siquiera la primera
     operación que sí había corrido bien. La transacción es todo o nada.

12.3 No. Un único UPDATE ya es atómico por sí mismo (Postgres lo aplica entero o no
     lo aplica). Envolverlo en una transacción no agrega ninguna garantía; es ruido.
     Las transacciones son para coordinar DOS o más escrituras.
```
```ts
// 12.2
await db.transaction(async (tx) => {
  const [proyecto] = await tx
    .insert(proyectos)
    .values({ nombre: "Lanzamiento", usuarioId: 1 })
    .returning({ id: proyectos.id });

  await tx.insert(tareas).values({
    titulo: "Definir alcance",
    proyectoId: proyecto.id, // usamos el id recién creado para la FK
  });
  // si el insert de la tarea falla, el proyecto tampoco queda creado (rollback)
});
```

### Módulo 13
```
13.1 Cambia SOLO el módulo (el provider del token "USER_REPO" pasa a useClass:
     DrizzleUserRepository) y se agrega el nuevo archivo del repositorio. NO cambian:
     el UsersService, los controladores, ni los tests unitarios. El service queda
     intacto porque depende de la interfaz UserRepository (por el token), no de la
     implementación concreta: le da igual si detrás hay un Map o Postgres.

13.3 En los tests unitarios seguís usando el repo en memoria (o un mock que cumpla la
     interfaz). Según el archivo senior, mockeás lo lento/externo para probar la
     lógica del service en milisegundos sin levantar una DB. La base REAL (en
     contenedor, testcontainers) se prueba en los tests de INTEGRACIÓN del propio
     DrizzleUserRepository, no en los unitarios del service.
```
```ts
// 13.2
@Injectable()
export class DrizzleProjectRepository implements ProjectRepository {
  constructor(@Inject("DB") private readonly db: NodePgDatabase) {}

  async findById(id: number): Promise<Proyecto | null> {
    const [p] = await this.db.select().from(proyectos).where(eq(proyectos.id, id));
    return p ?? null;
  }

  async save(p: NuevoProyecto): Promise<Proyecto> {
    const [creado] = await this.db.insert(proyectos).values(p).returning();
    return creado;
  }
}
```

### Módulo 14
```
14.1 (a) Schema: Drizzle lo define en TypeScript (schema.ts); Prisma en un DSL propio
     (schema.prisma) que requiere un paso de generación. (b) Estilo: Drizzle es
     SQL-first (las queries se parecen al SQL); Prisma ofrece una abstracción más alta
     con API propia. (También: Drizzle es más liviano para serverless/edge.)

14.2 Drizzle: es la recomendación para proyectos nuevos en 2026, da control fino del
     SQL (lo que el equipo valora) y es ligero, ideal para serverless/edge.

14.3 Reescribís solo las implementaciones de los repositorios (las clases que hablan
     con el ORM). La interfaz del repositorio, el service, los controladores y la
     lógica de dominio quedan igual: dependen del puerto, no del ORM.
```

### Módulo 15
```
15.1 Es un "lost update" (actualización perdida): ambas leen el mismo saldo, suman 100
     sobre ese valor y guardan; la segunda escritura pisa a la primera y se "pierde"
     una suma. Bajo READ COMMITTED no se previene solo. Versión sin locks (cálculo
     atómico en la DB):
        UPDATE cuentas SET saldo = saldo + 100 WHERE id = :id;
15.2 SELECT ... FOR UPDATE toma un lock de fila: la segunda transacción que pida esa
     fila espera a que la primera confirme, así no leen el mismo valor "viejo" en
     paralelo. Resuelve el lost update cuando el cálculo no se puede hacer atómico en
     un solo UPDATE (ej. lógica de por medio).
15.3 Un deadlock es cuando dos transacciones se bloquean mutuamente (cada una tiene un
     lock que la otra necesita). Postgres lo detecta y aborta una (error 40P01). Regla
     para prevenirlo: tomar los locks SIEMPRE en el mismo orden en todo el código.
```

### Módulo 16
```
16.1 Por MVCC: el UPDATE crea una nueva versión (tupla) de la fila y marca la vieja
     como obsoleta, para que lectores concurrentes sigan viendo su snapshot sin
     bloquearse. Eso genera "tuplas muertas"; si se acumulan, es "bloat" (la tabla y
     sus índices se inflan).
16.2 El autovacuum recupera el espacio de las tuplas muertas (y actualiza estadísticas)
     de forma automática. En tablas con muchas escrituras es clave: sin él, el bloat
     crece y degrada el rendimiento (más páginas que leer, índices más grandes).
16.3 Porque no hay un contador global: COUNT(*) recorre las tuplas vivas que la
     transacción puede ver (MVCC), y con bloat hay aún más páginas que recorrer. Para
     un aproximado rápido se puede mirar reltuples en pg_class.
```

### Módulo 17
```
17.1 (a) Sí (prefijo izquierdo: proyecto_id es la primera columna). (b) Sí (usa ambas
     columnas en orden). (c) No bien: filtrar solo por `completada` no puede aprovechar
     el índice (proyecto_id es la primera columna del compuesto).
17.2 Las estadísticas del planner están desactualizadas: estima 5 filas pero hay
     50.000, así que probablemente eligió un mal plan. Se corrige con ANALYZE (o
     esperando al autovacuum, que también analiza).
17.3 Porque si la tabla es chica o la query trae una fracción grande de las filas
     (baja selectividad), leer todo secuencialmente es más barato que saltar por el
     índice fila por fila. El planner lo decide con estadísticas; forzar un índice ahí
     empeoraría.
```

### Módulo 18
```
18.1 Porque OFFSET no "salta": Postgres lee y descarta las 100.000 filas previas para
     poder devolver las 20 siguientes. Es O(n) en el offset, así que las páginas
     profundas se degradan aunque el LIMIT sea chico.
18.3 La columna de orden debe ser única e indexada (o una combinación que desempate,
     ej. (creado_el, id)); si no, podés saltear o repetir filas. Limitación frente a
     offset: solo podés avanzar/retroceder desde el último visto, no saltar a una
     "página N" arbitraria.
```
```ts
// 18.2
import { lt, desc } from "drizzle-orm";

const pagina = await db
  .select()
  .from(tareas)
  .where(lt(tareas.id, ultimoId)) // ultimoId = id de la última fila mostrada
  .orderBy(desc(tareas.id))
  .limit(20);
```

### Módulo 19
```
19.1 Porque cada conexión Postgres es un proceso del servidor (con su memoria) y abrir/
     cerrar una por request es caro; además hay un límite (max_connections). El pool
     mantiene conexiones abiertas y las reutiliza entre requests.
19.2 20 × 10 = 200 conexiones posibles, pero max_connections = 100: Postgres rechazará
     conexiones cuando se supere. Opciones: bajar el tamaño del pool por instancia,
     subir max_connections con criterio, o (lo más escalable) poner PgBouncer delante
     para multiplexar muchas conexiones de cliente sobre pocas reales.
19.3 Gana eficiencia: en transaction mode la conexión real se libera al pool al
     terminar cada transacción, así sirve a muchos más clientes con pocas conexiones.
     Pierde compatibilidad con features de sesión: prepared statements, SET,
     LISTEN/NOTIFY y advisory locks de sesión (hay que configurar el driver).
```

### Módulo 20
```
20.1 JSONB para datos semiestructurados, opcionales o variables que no tienen
     relaciones ni reglas fuertes (metadata, payloads, config por fila). Columnas/
     tablas cuando el dato tiene relaciones, constraints, tipos o se consulta seguido:
     ahí JSONB te haría perder integridad y claridad. Criterio: relacional por defecto,
     JSONB para lo verdaderamente flexible.
20.3 Porque to_tsvector/to_tsquery entienden el idioma (stemming: "informe" matchea
     "informes"/"informar"), ignoran stopwords y permiten ranking, mientras que
     LIKE '%palabra%' es una búsqueda literal que no escala (no usa índice B-tree con
     comodín al inicio) ni entiende variantes. Se acelera con un índice GIN sobre el
     tsvector.
```
```sql
-- 20.2
SELECT * FROM tareas WHERE metadata @> '{"prioridad":"alta"}';
CREATE INDEX idx_tareas_meta ON tareas USING GIN (metadata);
```

---

## Siguientes pasos

Con este módulo ya tenés la pieza que faltaba: una base de datos **real** detrás de tus servicios Nest, entendiendo el SQL que el ORM escribe por vos. El recorrido natural a partir de acá: **(1)** conectá tu "Task API" de Nest a Postgres reemplazando los repos en memoria por implementaciones Drizzle (módulo 13); **(2)** sumá **tests de integración** del repositorio contra una DB real en contenedor (testcontainers), como pide el archivo senior; **(3)** cuando tengas datos reales, volvé a la sección de **performance del archivo senior** (N+1, índices, connection pooling, cache-aside con Redis) — ahora cada concepto tiene una base concreta donde aplicarse. El siguiente módulo del temario que más valor agrega es **autenticación práctica** (JWT + refresh + RBAC), que ya se apoya en tener usuarios persistidos en esta base.
