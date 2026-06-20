# TDD como disciplina: diseñar con tests, no solo testear

**Test-Driven Development, red-green-refactor, las dos escuelas y qué hace bueno a un test · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya hiciste [Testing práctico](testing.md) —ahí está la **mecánica** (pirámide, AAA, mocks, testcontainers, Supertest, cobertura)—. **Este módulo es otra cosa**: no es *cómo se escribe un test*, es **TDD como disciplina de diseño**. La diferencia que tenés que internalizar desde la primera línea: **TDD no es "escribir tests"; es usar el test como herramienta para diseñar el código antes de escribirlo.** Los tests son un efecto secundario valioso; el producto principal es un diseño mejor. El objetivo es doble: dominar el **ciclo y las escuelas** (red-green-refactor, classicist vs mockist, qué hace bueno a un test según Khorikov), y —tan importante— **el criterio**: cuándo TDD paga y cuándo es dogma que estorba.

**Lo que asumimos.** El módulo de [Testing](testing.md) entero (Jest/Vitest, dobles de prueba, `TestingModule` de Nest, e2e), TypeScript, NestJS con DI, y el dominio de ejemplo del roadmap: la **Task API** (usuarios ↔ proyectos ↔ tareas).

**Para practicar.** Lo mismo que en Testing —Vitest o Jest— más la disciplina de **no escribir una línea de producción sin un test que falle pidiéndola**.

```bash
npm i -D vitest   # o jest; la mecánica ya la viste en el módulo de Testing
```

> Nota: las herramientas (Vitest/Jest) y versiones cambian; lo de este módulo es **metodología**, que es estable. Los nombres de autores y libros (Beck, Khorikov, Khononov, Fowler) son referencias para profundizar.

**Índice de módulos**
1. Qué es TDD (y qué no es): una disciplina de diseño
2. Red-green-refactor: el ciclo y por qué cada fase
3. Un ciclo completo en vivo: una regla de negocio paso a paso
4. Las dos escuelas: classicist (Chicago) vs mockist (London)
5. El problema de sobre-mockear: tests acoplados a la implementación
6. Qué hace bueno a un test: los cuatro atributos de Khorikov
7. TDD empuja el diseño: por qué el código nace testeable
8. Bugs y specs: el test que falla primero
9. Cuándo NO hacer TDD: spikes, exploración y código trivial
10. TDD como disciplina de equipo (Tech Lead, CI, DORA)
11. El criterio: TDD es diseño, no cobertura

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es TDD (y qué no es): una disciplina de diseño

**Teoría.** La confusión más común, y la que tenés que desarmar para entender todo lo demás: **TDD no significa "tener muchos tests" ni "testear bien".** Podés tener un proyecto con 95% de cobertura y cero TDD, y podés hacer TDD impecable y terminar con cobertura moderada. Son cosas distintas.

**TDD (Test-Driven Development) es escribir el test _antes_ que el código de producción, y dejar que el test _maneje_ (drive) el diseño de ese código.** El test no es algo que agregás después para verificar; es la **primera expresión de lo que el código tiene que hacer**, escrita desde el punto de vista de quien lo va a usar. Esa inversión de orden —test primero, implementación después— es todo el truco, y tiene tres consecuencias profundas:

1. **Te obliga a definir el _qué_ antes del _cómo_.** Para escribir el test primero tenés que decidir la interfaz (cómo se llama el método, qué recibe, qué devuelve) *antes* de tener una sola línea de implementación. Estás **diseñando la API de tu código desde el lado del consumidor**, que es exactamente quien sufre una mala API.

2. **Garantiza que el código es testeable** —por construcción—. Un código escrito test-first no puede ser intesteable: nació de un test. El "esto no se puede testear" desaparece, porque el test existía antes que el código.

3. **El test que escribís primero tiene que _fallar_.** Y eso, contraintuitivamente, es lo más valioso: un test que nunca viste fallar no probó nada. Si lo escribís después del código y pasa de una, no sabés si pasa porque el código está bien o porque el test está mal escrito y pasaría con cualquier cosa.

Las **tres leyes de TDD** (Robert C. Martin), que formalizan la disciplina:

1. No escribís código de producción salvo para hacer pasar un test que **está fallando**.
2. No escribís más de un test de lo necesario para **fallar** (y no compilar es fallar).
3. No escribís más código de producción del necesario para hacer **pasar** ese test.

Suena rígido y al principio lo es; el punto es que te fuerza a trabajar en **pasos chiquitos** (baby steps) con feedback constante, en vez de escribir 200 líneas y rezar. La frase mental: **en TDD, el test es el cliente de tu código y vos sos su primer usuario —si es incómodo de testear, es incómodo de usar—.** Conecta directo con el [eval-driven development del módulo 6 de Evaluations](evals.md): escribir las evals antes de optimizar el sistema de IA es exactamente la misma idea —dejar que la medición maneje el desarrollo— aplicada a algo no determinista.

**Ejercicios 1**
1.1 Explicá por qué "tener 95% de cobertura" no implica "hacer TDD". ¿Qué es lo que define a TDD?
1.2 Enumerá las tres consecuencias de escribir el test antes que el código. ¿Cuál de ellas hace desaparecer el problema "esto no se puede testear"?
1.3 ¿Por qué es importante **ver el test fallar** antes de implementar? ¿Qué riesgo corrés si escribís el test después y pasa de una?
1.4 ¿Con qué práctica del track de IA es análogo TDD y por qué?

---

## Módulo 2 — Red-green-refactor: el ciclo y por qué cada fase

**Teoría.** El motor de TDD es un ciclo de tres fases que repetís decenas de veces por día: **Red → Green → Refactor**. Cada vuelta agrega una pizca de comportamiento.

- **🔴 Red — escribí un test que falla.** Definís el siguiente comportamiento más chico que querés, lo expresás como test, y lo corrés para verlo **fallar** (rojo). Ver el rojo confirma dos cosas: que el test efectivamente prueba algo, y que ese algo todavía no existe. Si el test pasa en rojo, el test está mal.

- **🟢 Green — hacelo pasar lo más rápido posible.** Escribís **el mínimo código** para que el test pase (verde). Acá *se permite* hacer trampa: hardcodear el valor, devolver una constante. El objetivo de esta fase **no es código elegante, es volver a verde rápido**. ¿Por qué permitir trampa? Porque te mantiene en pasos chicos y te obliga a escribir *otro* test para forzar el caso general (si hardcodeaste `return 5`, el siguiente test con otra entrada te obliga a generalizar). Es la técnica de "triangulación".

- **🔵 Refactor — limpiá, con la red de seguridad puesta.** Ahora que está verde, mejorás el diseño **sin cambiar el comportamiento**: renombrás, extraés funciones, eliminás duplicación, mejorás nombres. Los tests en verde son tu **red de seguridad**: si un refactor rompe algo, un test se pone rojo al instante. Esta es **la fase que casi todo el mundo se saltea**, y saltársela es la causa #1 de que TDD "no funcione" para la gente: sin refactor, acumulás los parches feos de la fase green y el diseño se pudre. **El refactor no es opcional; es donde TDD paga el diseño.**

Y volvés a Red con el siguiente comportamiento. El ritmo es **rápido**: cada ciclo dura de segundos a pocos minutos. Si un ciclo te lleva media hora, tus pasos son demasiado grandes.

```
   ┌──────────────────────────────────────────┐
   │                                            │
   ▼                                            │
🔴 RED ───────► 🟢 GREEN ───────► 🔵 REFACTOR ──┘
test que falla   mínimo p/pasar    limpiar sin
(ver el rojo)    (vale hardcodear) romper (red de
                                   seguridad: tests verdes)
```

La regla de oro del ciclo: **nunca refactorices en rojo.** Refactor y agregar comportamiento son dos sombreros distintos (Kent Beck): o estás haciendo pasar un test (green, podés ser feo) o estás limpiando con todo en verde (refactor, no agregás comportamiento). Mezclarlos —refactorizar mientras tenés tests rojos— es cómo te perdés y no sabés si rompiste algo por el refactor o porque el comportamiento todavía no estaba.

**Ejercicios 2**
2.1 Nombrá las tres fases del ciclo y la regla principal de cada una.
2.2 En la fase green se permite "hacer trampa" (hardcodear). ¿Por qué es válido y qué técnica te obliga después a generalizar?
2.3 ¿Cuál es la fase que la gente se saltea y por qué saltársela hace que "TDD no funcione"?
2.4 ¿Por qué nunca hay que refactorizar con un test en rojo?

---

## Módulo 3 — Un ciclo completo en vivo: una regla de negocio paso a paso

**Teoría.** Hagamos TDD de verdad sobre una regla de la Task API: **"una tarea no puede asignarse a un usuario que no es miembro del proyecto"**. Vamos ciclo por ciclo, en baby steps.

**Ciclo 1 — 🔴** El test más chico: asignar a un miembro válido funciona.

```ts
// task.service.spec.ts
it('asigna la tarea si el usuario es miembro del proyecto', () => {
  const project = new Project({ id: 7, memberIds: [4521] });
  const task = new Task({ id: 99, projectId: 7 });
  task.assignTo(4521, project);          // ← este método NO existe todavía
  expect(task.assigneeId).toBe(4521);
});
```

No compila → es rojo (ley 2: no compilar es fallar). **🟢** Mínimo para pasar:

```ts
class Task {
  assigneeId?: number;
  assignTo(userId: number, _project: Project) {
    this.assigneeId = userId;            // lo mínimo: ignoramos el proyecto por ahora
  }
}
```

Verde. **🔵** Nada que refactorizar aún. Siguiente ciclo.

**Ciclo 2 — 🔴** El caso que importa: asignar a un no-miembro debe fallar.

```ts
it('lanza si el usuario NO es miembro del proyecto', () => {
  const project = new Project({ id: 7, memberIds: [4521] });
  const task = new Task({ id: 99, projectId: 7 });
  expect(() => task.assignTo(9999, project))   // 9999 no está en memberIds
    .toThrow(NotProjectMemberError);
});
```

Rojo (hoy `assignTo` asigna a cualquiera). **🟢** El mínimo que hace pasar *ambos* tests:

```ts
assignTo(userId: number, project: Project) {
  if (!project.memberIds.includes(userId)) {
    throw new NotProjectMemberError(userId, project.id);
  }
  this.assigneeId = userId;
}
```

Verde, los dos tests pasan. **🔵 Refactor:** el chequeo de pertenencia es lógica del `Project`, no del `Task` —que `Task` meta mano en `project.memberIds` viola encapsulamiento—. Lo movemos, con los tests cuidándonos:

```ts
class Project {
  isMember(userId: number): boolean {     // el Project responde por sus miembros
    return this.memberIds.includes(userId);
  }
}
class Task {
  assignTo(userId: number, project: Project) {
    if (!project.isMember(userId)) {
      throw new NotProjectMemberError(userId, project.id);
    }
    this.assigneeId = userId;
  }
}
```

Corro los tests: siguen verdes. El refactor mejoró el diseño (cada objeto responde por lo suyo) **sin tocar el comportamiento**, y los tests me lo garantizaron. Fijate lo que pasó: **el diseño emergió del ciclo**. No diseñé `Project.isMember()` de antemano; apareció en el refactor cuando el código pidió a gritos que esa responsabilidad viviera en otro lado. Eso es TDD haciendo diseño.

**Ejercicios 3**
3.1 En el ciclo 1, ¿por qué está bien que `assignTo` ignore el parámetro `project` y asigne a cualquiera? ¿Qué fuerza a corregirlo?
3.2 En el ciclo 2, fase green, ¿por qué la implementación quedó "completa" sin trampa esta vez? (pista: ya había dos tests)
3.3 El refactor movió `isMember` de `Task` a `Project`. ¿Qué principio de diseño mejoró y cómo supiste que no rompiste nada?
3.4 ¿Qué quiere decir que "el diseño emergió del ciclo" en este ejemplo?

---

## Módulo 4 — Las dos escuelas: classicist (Chicago) vs mockist (London)

**Teoría.** Acá está el debate que define el estilo de TDD que practicás, y que aparece seguro en una entrevista senior. Hay **dos escuelas** sobre cómo testear una unidad que depende de otras:

- **Classicist** (también "Detroit", "Chicago" o "clásica" — la de Kent Beck): la **unidad bajo test es un comportamiento**, y usás las **dependencias reales** siempre que puedas. Solo reemplazás por un doble las dependencias **incómodas**: las que salen del proceso y son difíciles de manejar en un test (la base de datos, una API externa, el reloj, la red). Un `TaskService` que usa un `Project` y un validador de dominio se testea **con el `Project` y el validador reales**. Mockeás Postgres, no tus propios objetos.

- **Mockist** (también "London" — Freeman & Pryce, *Growing Object-Oriented Software*): la **unidad bajo test es una clase**, y mockeás **todas** sus colaboradoras. Testeás el `TaskService` en total aislamiento, con dobles para *todas* sus dependencias, verificando las **interacciones** (que `taskService` llamó a `project.isMember()` con tal argumento). Esto lleva al estilo **outside-in**: empezás por el test del borde (el controller), descubrís qué colaboradores necesita mockeando, y vas implementando hacia adentro.

La diferencia práctica más visible: **qué consideran "una unidad"** (un comportamiento vs. una clase) y **cuánto mockean** (lo mínimo incómodo vs. todo lo que colabora).

Cada escuela tiene su fortaleza:

- El **mockist/London** es bueno para **diseñar la colaboración** entre objetos antes de que existan (outside-in): los mocks te hacen explícitas las interfaces que cada colaborador necesita. Da feedback de diseño muy temprano.
- El **classicist/Chicago** produce tests que **sobreviven al refactor** (módulo 6): como prueban comportamiento observable y no interacciones internas, podés reorganizar las clases por dentro sin romper los tests. Da más libertad para refactorizar.

El consenso pragmático de la industria (y la posición de Vladimir Khorikov en *Unit Testing*, que es la referencia moderna): **inclinarse hacia el classicist por defecto.** Usar dependencias reales dentro del proceso y mockear solo lo que sale del proceso y *no controlás* (más en módulo 7). El mockist tiene su lugar —outside-in para descubrir diseño—, pero como dieta principal produce tests frágiles. Fowler tiene el ensayo canónico ("Mocks Aren't Stubs") si querés la versión larga.

**Ejercicios 4**
4.1 Para la escuela classicist y la mockist, decí: ¿qué consideran "una unidad" y cuánto mockean?
4.2 ¿Qué estilo de trabajo (outside-in / inside-out) se asocia con cada escuela y por qué?
4.3 Nombrá la fortaleza principal de cada escuela (qué hace bien la mockist, qué hace bien la classicist).
4.4 ¿Hacia qué escuela se inclina el consenso pragmático moderno y cuál es la regla de qué mockear?

---

## Módulo 5 — El problema de sobre-mockear: tests acoplados a la implementación

**Teoría.** El error más caro en TDD —y el que hace que mucha gente termine odiando sus tests— es **sobre-mockear**: reemplazar por dobles cosas que no hacía falta, hasta que el test deja de verificar *comportamiento* y pasa a verificar *implementación*. Cuando eso pasa, tu suite se vuelve un ancla en vez de una red.

Mirá el síntoma. Test sobre-mockeado del `TaskService`:

```ts
// ❌ Sobre-mockeado: verifica CÓMO trabaja por dentro, no QUÉ logra
it('asigna la tarea', () => {
  const project = { isMember: vi.fn().mockReturnValue(true) };  // colaborador propio, mockeado
  const repo = { save: vi.fn() };
  service.assign(task, 4521, project as any);

  expect(project.isMember).toHaveBeenCalledWith(4521);    // verifica una interacción interna
  expect(repo.save).toHaveBeenCalledTimes(1);             // verifica otra interacción interna
});
```

Este test pasa hoy. Pero el día que refactorizás —digamos, `assign` ahora chequea pertenencia de otra forma, o agrupa los saves— el comportamiento observable **es idéntico** (la tarea queda asignada y persistida), pero el test **se rompe**, porque estaba atado a las llamadas internas. Tuviste que tocar el test sin que el comportamiento cambiara. Eso es un **test frágil**, y es exactamente el anti-objetivo: un test que te frena el refactor en vez de habilitarlo. Es la violación del pilar "resistencia al refactor" del módulo 6.

La causa raíz: **mockeaste un colaborador que era parte de la unidad de comportamiento** (el `Project`, que está dentro de tu proceso y controlás). El `Project` real era barato de usar; mockearlo no aportó nada y acopló el test a *cómo* el service lo usa.

La versión classicist usa el `Project` real y verifica el **resultado**, no las llamadas:

```ts
// ✅ Verifica el resultado observable; sobrevive a refactors internos
it('asigna la tarea a un miembro y la persiste', async () => {
  const project = new Project({ id: 7, memberIds: [4521] });  // real
  await service.assign(task, 4521, project);
  expect(task.assigneeId).toBe(4521);                          // el QUÉ, no el CÓMO
  expect(await repo.findById(99)).toMatchObject({ assigneeId: 4521 });
});
```

La regla, simple y poderosa: **verificá el resultado observable (el estado final, el valor devuelto, el efecto visible), no las interacciones internas.** Las únicas interacciones que vale la pena verificar son las que cruzan un borde que te importa y *no podés observar de otra forma* —ej. que efectivamente mandaste el email, que publicaste el evento a la cola—. Verificar que tu objeto llamó a su propio colaborador interno es testear la implementación, no el comportamiento.

**Ejercicios 5**
5.1 ¿Qué significa que un test "verifica implementación en vez de comportamiento"? Dá el síntoma concreto que aparece al refactorizar.
5.2 En el ejemplo sobre-mockeado, ¿cuál fue el error de raíz (qué se mockeó que no hacía falta)?
5.3 Reescribí la idea de la regla: ¿qué deberías verificar y qué casi nunca?
5.4 ¿Cuándo *sí* vale la pena verificar una interacción y no solo el resultado?

---

## Módulo 6 — Qué hace bueno a un test: los cuatro atributos de Khorikov

**Teoría.** Si TDD produce tests, ¿cómo sabés si son buenos? Vladimir Khorikov (*Unit Testing: Principles, Practices, and Patterns*) da el marco más usado: **todo test tiene cuatro atributos**, y un buen test maximiza los cuatro —pero hay una tensión que te obliga a balancear—.

1. **Protección contra regresiones** (*protection against regressions*): ¿cuántos bugs reales atrapa el test? Un test que ejercita mucho código significativo protege mucho; un test trivial (un getter) protege poco. Sube cuanto más comportamiento real ejecuta.

2. **Resistencia al refactor** (*resistance to refactoring*): ¿el test sigue verde cuando refactorizás sin cambiar el comportamiento? Este es **el pilar rey** y el que el módulo 5 atacó: un test acoplado a la implementación tiene resistencia baja (se rompe ante cualquier refactor → falsos positivos que entrenan al equipo a ignorar la suite). Un test que verifica solo comportamiento observable tiene resistencia alta. **Se maximiza testeando el qué, no el cómo.**

3. **Feedback rápido** (*fast feedback*): ¿qué tan rápido corre? Tests rápidos los corrés a cada cambio (parte del ciclo red-green-refactor); tests lentos los corrés poco y pierden el sentido. Los unit tests de dominio son rapidísimos; los de integración contra Postgres real (testcontainers, [módulo 6 de Testing](testing.md)) son más lentos —de ahí la pirámide—.

4. **Mantenibilidad** (*maintainability*): ¿qué tan fácil es leer y mantener el test? Tests con setup enorme, mocks por todos lados y asserts crípticos cuestan más de lo que valen.

La idea más importante de Khorikov es que **estos cuatro están en tensión y no podés tener los cuatro al máximo** —tenés que elegir—. En particular: **protección contra regresiones y resistencia al refactor son lo que define un test valioso, y feedback rápido es lo que sacrificás cuando subís la otra**. El cuadrante que se desprende:

- **Mucha protección + poca resistencia** = los **tests frágiles** del módulo 5 (sobre-mockeados): atrapan bugs pero también saltan ante refactors legítimos. Peor de lo que parece, porque entrenan a ignorar la suite.
- **Mucha resistencia + poca protección** = tests **triviales** (testear un getter): nunca se rompen pero tampoco atrapan nada. Inútiles.
- **El objetivo**: tests que ejercitan comportamiento real (protección) verificando resultado observable (resistencia), tan rápidos como su tipo permita.

La conclusión accionable: **priorizá resistencia al refactor.** Un falso positivo (test que se rompe sin que haya bug) erosiona la confianza en la suite más rápido que un falso negativo ocasional. Una suite en la que el equipo confía es la que te deja refactorizar sin miedo —que es, recordá el módulo 1, el punto entero de TDD—.

**Ejercicios 6**
6.1 Nombrá los cuatro atributos de un buen test según Khorikov.
6.2 ¿Cuál es "el pilar rey" y cómo se maximiza? (conectá con el módulo 5)
6.3 Ubicá en el cuadrante: (a) un test sobre-mockeado del `TaskService`; (b) un test de un getter `task.getId()`. ¿Qué le sobra y qué le falta a cada uno?
6.4 ¿Por qué un falso positivo (test que salta sin bug) es más dañino que un falso negativo ocasional?

---

## Módulo 7 — TDD empuja el diseño: por qué el código nace testeable

**Teoría.** La afirmación central de TDD como disciplina —y la razón por la que un Tech Lead la defiende— es que **el test-first no solo te da tests, te da mejor diseño**. No por magia: porque escribir el test primero te enfrenta al dolor del mal diseño *antes* de pagarlo, cuando todavía es barato cambiarlo.

Cómo funciona el mecanismo, concretamente:

- **Acoplamiento.** Si para testear tu `TaskService` tenés que instanciar media aplicación (la DB, el mailer, tres servicios más), el test es un infierno de setup. Ese dolor en el test es el síntoma de que tu clase **depende de demasiadas cosas concretas**. TDD te hace sentir ese acoplamiento de inmediato, y la salida natural es **inyectar dependencias por interfaz** (la DI de Nest que viste en [Testing módulo 5](testing.md)): pasás un `ProjectRepository` abstracto en vez de instanciar Postgres adentro. El test mejoró → el diseño se desacopló.

- **Tamaño de las interfaces.** Cuando escribís el test primero, sentís en carne propia si un método pide 9 parámetros o devuelve un objeto enorme: es tedioso de armar en el `arrange`. Ese tedio te empuja a interfaces más chicas y cohesivas. **El test es el primer consumidor de tu API y se queja de las APIs feas.**

- **Cohesión y responsabilidad única.** Como en el ejemplo del módulo 3 (mover `isMember` de `Task` a `Project`), el refactor del ciclo va separando responsabilidades. Una clase que hace demasiado es difícil de testear sin un setup gigante; el dolor te empuja a partirla.

El puente con [NestJS senior](nestjs-senior.md): el dominio rico (entidades con comportamiento, no anémicas), las dependencias inyectadas y los puertos/adaptadores que viste ahí **no son un accidente**: son el tipo de diseño hacia el que TDD te empuja naturalmente. Un código pensado test-first tiende a tener bordes claros (qué es dominio puro, qué sale del proceso), que es justo lo que hace falta para la arquitectura hexagonal/limpia.

La matización honesta (Khononov, *Learning Domain-Driven Design*, y el pragmatismo general): **TDD empuja hacia buen diseño, pero no lo garantiza solo.** Podés hacer TDD y producir un diseño mediocre si ignorás las señales (si en vez de desacoplar, mockeás todo para callar el dolor del test —volvé al módulo 5—). TDD te *muestra* el problema; vos tenés que *escuchar* la señal y refactorizar. El feedback es la herramienta; el criterio sigue siendo tuyo.

**Ejercicios 7**
7.1 Tu test necesita instanciar Postgres, un mailer y tres servicios para probar `TaskService`. ¿Qué te está diciendo el test sobre el diseño y cuál es la salida?
7.2 ¿Por qué se dice que "el test es el primer consumidor de tu API"? Dá un ejemplo de una señal de diseño que el test te da temprano.
7.3 ¿Cómo conecta el diseño que empuja TDD con el dominio rico y las dependencias inyectadas de NestJS senior?
7.4 ¿Por qué TDD "no garantiza" buen diseño por sí solo? ¿Qué error de reacción ante el dolor del test lleva a un diseño peor?

---

## Módulo 8 — Bugs y specs: el test que falla primero

**Teoría.** Dos usos de TDD que van más allá del feature nuevo y que un equipo senior aplica por reflejo.

**1. Reproducir un bug con un test que falla (regression-first).** Llega un reporte: "asignar una tarea a un usuario eliminado del proyecto no tira error, la asigna igual". El reflejo amateur es ir al código a arreglarlo. **El reflejo TDD es escribir primero un test que reproduzca el bug** —y verlo fallar (rojo)—:

```ts
it('no asigna a un usuario que fue removido del proyecto (bug #412)', () => {
  const project = new Project({ id: 7, memberIds: [] });   // el user 4521 fue removido
  const task = new Task({ id: 99, projectId: 7 });
  expect(() => task.assignTo(4521, project)).toThrow(NotProjectMemberError);
});
```

Tres ventajas enormes: (a) el rojo **confirma que reprodujiste el bug** —si pasa de una, no entendiste el bug—; (b) cuando lo arreglás y se pone verde, **probaste que lo arreglaste** de verdad, no que creés que sí; (c) el test queda para siempre como **protección contra esa regresión** —si alguien reintroduce el bug, salta—. Por eso los bugs son la mejor materia prima del golden set... y acá conecta literal con el [módulo 2 de Evaluations](evals.md): un fallo de producción se convierte en un caso de test permanente, en código y en evals.

**2. El test como especificación ejecutable.** Un buen nombre de test es una frase de spec: `it('lanza si el usuario no es miembro del proyecto')`. Leídos en conjunto, los nombres de tus tests **son la documentación viva** de qué hace el sistema —y, a diferencia de un doc en Confluence, no se desactualizan, porque si el comportamiento cambia, el test rojo te obliga a actualizarlos—. Por eso en TDD se cuida tanto el nombre del test: no es burocracia, es la especificación. La práctica relacionada es **BDD** (Behavior-Driven Development), que lleva esto al extremo escribiendo los tests en lenguaje de negocio (`given/when/then`) para que los lea gente no técnica.

La frase mental que une los dos usos: **en TDD, antes de escribir el código que arregla o implementa algo, escribís la prueba de que hace falta. El rojo es la evidencia de que el trabajo todavía no está hecho; el verde, de que sí.**

**Ejercicios 8**
8.1 Llega un bug. ¿Cuál es el primer paso en TDD y qué tres ventajas te da escribir el test del bug antes de arreglarlo?
8.2 Si escribís el test del bug y pasa en verde de una, ¿qué aprendiste?
8.3 ¿En qué sentido los nombres de los tests son "documentación viva" y por qué no se desactualiza como un doc tradicional?
8.4 ¿Cómo conecta el "test del bug" con el golden set del módulo de Evaluations?

---

## Módulo 9 — Cuándo NO hacer TDD: spikes, exploración y código trivial

**Teoría.** El módulo que separa al que entiende TDD del que lo aplica como dogma. TDD es una herramienta poderosa **para cierto tipo de problema**, y forzarla donde no encaja es perder tiempo y generar resistencia en el equipo. Saber cuándo *no* hacerlo es tan senior como saber hacerlo.

Dónde TDD **paga claramente**:

- **Lógica de dominio y reglas de negocio** —el caso estrella—. Reglas con ramas, validaciones, cálculos, máquinas de estado (la asignación de tareas, el cálculo de un presupuesto, las transiciones de un pedido). Acá el diseño importa, los casos borde abundan y el test-first brilla.
- **Bugs** (módulo 8): siempre, regression-first.
- **Código con muchos caminos** que querés acorralar antes de escribir.

Dónde TDD **estorba o no aporta**:

- **Spikes y exploración** (acá conecta con el [módulo de spike del flujo de trabajo, si lo usás]): cuando estás *aprendiendo* cómo funciona una librería o explorando si una idea es viable, todavía **no sabés qué querés** —y TDD necesita que sepas el *qué* para escribir el test—. Hacé el spike sin tests, aprendé, **tirá el código del spike**, y *después* implementá con TDD lo que decidiste. Forzar TDD en un spike es ponerle test a algo que vas a borrar.
- **Código trivial sin lógica**: un getter, un DTO, un mapeo directo de campos, configuración. Testearlo da el cuadrante "mucha resistencia, cero protección" del módulo 6 —no atrapa nada—. No es que esté prohibido; es que no rinde.
- **Glue code / wiring de infraestructura**: el código que solo cablea piezas (un controller que delega directo a un service, un módulo de Nest). Se cubre mejor con un test e2e/integración ([módulos 6-7 de Testing](testing.md)) que con unit tests test-first.
- **UI muy visual / exploratoria**: el feedback ahí es mirar la pantalla, no un assert.

La regla de criterio: **TDD donde el diseño y la corrección importan y son conocibles de antemano (dominio); test-after o tests de mayor nivel donde el valor está en que funcione integrado (glue, infra); sin tests donde estás explorando (spike, tirás el código).** Y el antipatrón a evitar: el dogmático que hace TDD del getter y del mapeo de un DTO "porque hay que tener cobertura" —ese gasta esfuerzo en tests del peor cuadrante mientras a veces deja sin testear la regla de negocio jugosa—.

**Ejercicios 9**
9.1 Nombrá dos tipos de código donde TDD paga claramente y dos donde estorba o no aporta.
9.2 ¿Por qué TDD y un spike de exploración no se llevan bien? ¿Qué hacés en su lugar y qué hacés con el código del spike?
9.3 Testear un getter con TDD: ¿en qué cuadrante de Khorikov cae y por qué no rinde?
9.4 Un dev del equipo presume "100% de cobertura, TDD en todo, hasta los DTOs". ¿Qué le señalarías como antipatrón?

---

## Módulo 10 — TDD como disciplina de equipo (Tech Lead, CI, DORA)

**Teoría.** TDD individual es una habilidad; TDD de equipo es una **decisión de liderazgo técnico**, y como tal tiene matices que exceden la mecánica.

**No es negociable como capacidad, sí es pragmático como práctica.** Un Tech Lead no impone "TDD estricto en cada línea o no se mergea" —eso genera el dogmatismo del módulo 9 y resentimiento—. Lo que sostiene es: **todo lo que se mergea tiene tests que dan confianza, y el equipo sabe hacer TDD para cuando rinde (dominio, bugs).** El *cómo* llegaste al test (test-first puro o test-after disciplinado) importa menos que el resultado: una suite confiable que cumple los cuatro atributos del módulo 6.

**La conexión con CI/CD** (del [módulo de Docker/deploy](docker-deploy.md)): los tests de TDD son la primera barrera del pipeline. Un commit que rompe un test no se mergea. Esto solo funciona si la suite es **rápida y confiable** —si está llena de tests frágiles (módulo 5) que saltan por refactors legítimos, el equipo empieza a re-correr el CI "a ver si pasa esta vez" o a marcarlos como `skip`, y la barrera se vuelve teatro—. Por eso la calidad de los tests (resistencia al refactor) es un problema de Tech Lead, no solo de cada dev.

**La conexión con DORA** (las cuatro métricas de rendimiento de entrega: frecuencia de deploy, lead time, tasa de fallo de cambios, tiempo de recuperación). TDD impacta directo en dos: **change failure rate** baja (los bugs se atrapan antes de prod) y **deployment frequency / lead time** suben (porque podés refactorizar y entregar con confianza, sin la parálisis del "no toco eso que anda"). Una suite confiable es lo que hace posible el deploy frecuente sin miedo —que es la marca de un equipo de alto rendimiento—. Esto es material directo del próximo módulo del track, **liderazgo técnico**.

**En code review**, el lead mira los tests con el mismo rigor que el código de producción: ¿verifican comportamiento o implementación (módulo 5)? ¿el nombre es una spec (módulo 8)? ¿se testeó la regla de negocio o solo los getters (módulo 9)? Un PR con 100% de cobertura y tests frágiles es *peor* que uno con 70% y tests sólidos —y saber distinguirlos es el valor que aporta el lead—.

**Ejercicios 10**
10.1 ¿Por qué un Tech Lead no debería imponer "TDD estricto en cada línea"? ¿Qué sostiene en su lugar?
10.2 ¿Por qué los tests frágiles (módulo 5) sabotean el rol de los tests en CI/CD? ¿Qué empieza a hacer el equipo?
10.3 ¿Sobre qué dos métricas DORA impacta TDD y en qué dirección cada una?
10.4 En un code review, ¿por qué un PR con 100% de cobertura y tests frágiles puede ser peor que uno con 70% y tests sólidos?

---

## Módulo 11 — El criterio: TDD es diseño, no cobertura

**Teoría.** El módulo que cierra y el más de Tech Lead. Tenés el arsenal —el ciclo, las escuelas, los cuatro atributos, cuándo sí y cuándo no, el rol en el equipo—. El error de madurez ahora no es no hacer TDD; es **confundir TDD con sus subproductos** y perseguir la métrica equivocada.

Los principios de criterio que tenés que poder defender:

1. **TDD es una disciplina de diseño, no una de cobertura.** El producto valioso de TDD es el *diseño* (acoplamiento bajo, interfaces chicas, código que nace testeable) y la *confianza para refactorar*. Los tests son el subproducto. Quien mide TDD por % de cobertura entendió el medio y se perdió el fin —y termina en el peor cuadrante del módulo 6, mucha cobertura de tests que no protegen ni resisten—.

2. **El objetivo de todo esto es poder cambiar el código sin miedo.** Un sistema con una suite confiable es un sistema que podés evolucionar; uno sin ella se congela ("no toques eso que anda y nadie entiende"). Esa capacidad de cambiar con confianza es lo que TDD compra, y es lo que más vale en un sistema que vive años. Conecta con el "medí antes de optimizar" de [NestJS senior](nestjs-senior.md) y [Redis](redis.md): primero la red de seguridad y la medición, después el cambio.

3. **Resistencia al refactor antes que cobertura total.** Si tenés que elegir, una suite chica de tests sólidos que verifican comportamiento vale más que una suite enorme de tests frágiles. La confianza del equipo en la suite es el activo; los falsos positivos la destruyen.

4. **Escalá la disciplina al problema** —el mismo eje de criterio de todo el temario—. TDD estricto en el dominio rico; test-after en el glue; sin tests en el spike. El dogmático que hace TDD de los DTOs y el vago que no testea la regla de negocio son **el mismo error** visto desde dos lados: no haber pensado *dónde* el test aporta valor. Es la misma decisión que "¿[K8s o Fargate](kubernetes.md)?", "¿[llamada, workflow o agente](agentes.md)?", "¿cuánta [observabilidad](observabilidad.md)?": **ajustá la herramienta al problema, no apliques la máxima en todos lados.**

5. **TDD es una habilidad que se entrena, no un interruptor.** Los primeros intentos se sienten lentos y artificiales; los baby steps parecen exagerados. Es normal: estás reentrenando el reflejo de "código primero". El ritmo (red-green-refactor en minutos) llega con práctica, y el payoff —diseño y confianza— aparece en el mediano plazo, no en el primer feature.

El cierre y el puente con el rol: TDD es la **disciplina que conecta testear con diseñar**. Recorriste el ciclo red-green-refactor, las dos escuelas (classicist por defecto, mockist para descubrir diseño), los cuatro atributos de Khorikov (con la resistencia al refactor como rey), cómo el test-first empuja el diseño, los usos de regression-first y test-como-spec, cuándo *no* hacer TDD, y su rol como disciplina de equipo ligada a CI y a DORA. Junto con el [módulo de Testing](testing.md) (la mecánica) tenés el cuadro completo: *cómo* se escribe un test y *cómo* se diseña con tests. El próximo y último tramo del track Tech Lead es **liderazgo técnico** (ADRs para decidir, DORA para medir al equipo, Team Topologies para organizarlo), donde TDD/CI/DORA dejan de ser prácticas individuales y se vuelven la forma en que el equipo entrega.

**Ejercicios 11**
11.1 Completá la frase y justificá: "TDD es una disciplina de ___, no de ___".
11.2 ¿Cuál es, en una línea, el beneficio final que compra TDD y por qué importa tanto en un sistema que vive años?
11.3 ¿Por qué el dogmático que hace TDD de los DTOs y el vago que no testea la regla de negocio cometen "el mismo error"?
11.4 ¿Por qué TDD se siente lento y artificial al principio, y eso es señal de qué?
11.5 Si tuvieras que elegir entre "100% de cobertura con tests frágiles" y "60% con tests sólidos y resistentes al refactor", ¿qué elegís y por qué?

---

## Soluciones

### Módulo 1
1.1 Porque la cobertura mide *cuánto código ejecutan tus tests*, sin importar cuándo ni cómo los escribiste; podés alcanzarla escribiendo todos los tests después del código. Lo que define a TDD es **el orden y el propósito**: escribir el test *antes* que el código y dejar que **maneje el diseño** de ese código.
1.2 (1) Te obliga a definir el *qué* (la interfaz, desde el lado del consumidor) antes del *cómo*; (2) garantiza que el código es testeable por construcción; (3) el test escrito primero tiene que verse fallar. La **(2)** hace desaparecer el "esto no se puede testear": un código que nació de un test no puede ser intesteable.
1.3 Porque un test que nunca viste fallar no probó nada: no sabés si está verificando algo real. Si lo escribís después y pasa de una, podría estar mal escrito y pasaría con *cualquier* implementación (falso sentido de seguridad). Ver el rojo confirma que el test efectivamente prueba el comportamiento que falta.
1.4 Con el **eval-driven development** (módulo 6 de Evaluations): escribir las evals/medición antes de optimizar el sistema de IA es la misma idea —dejar que la medición maneje el desarrollo— aplicada a un sistema no determinista.

### Módulo 2
2.1 **Red**: escribir un test que falla y verlo fallar (confirma que prueba algo real). **Green**: escribir el mínimo código para pasar lo más rápido posible (vale hardcodear). **Refactor**: limpiar el diseño sin cambiar el comportamiento, con los tests verdes como red de seguridad.
2.2 Es válido porque mantiene los pasos chicos y el feedback rápido; no buscás elegancia, buscás verde. La técnica que te obliga a generalizar es la **triangulación**: escribís *otro* test con datos distintos que el hardcode no satisface, forzando la implementación general.
2.3 La fase de **refactor**. Saltársela hace que se acumulen los parches feos de la fase green y el diseño se pudra, así que la gente concluye que "TDD ensucia el código" —cuando en realidad nunca hicieron la parte donde TDD paga el diseño—.
2.4 Porque con un test en rojo no sabés si una falla nueva viene del refactor o del comportamiento que todavía no implementaste. Refactor (no cambia comportamiento, todo verde) y agregar comportamiento (green) son sombreros distintos; mezclarlos te hace perder de vista qué rompió qué.

### Módulo 3
3.1 Está bien porque en la fase green solo buscás pasar el único test que existe, y ese test solo prueba el caso del miembro válido (ley 3: mínimo código para pasar *ese* test). Lo que fuerza a corregirlo es el **ciclo 2**: el test del no-miembro falla con esa implementación tramposa y te obliga a agregar el chequeo real.
3.2 Porque ya había **dos tests** que acorralaban el comportamiento desde dos lados (miembro → asigna; no-miembro → lanza). Una trampa (hardcodear) no puede satisfacer ambos a la vez, así que el mínimo código que pasa los dos es ya la implementación general —la triangulación cerró el caso—.
3.3 Mejoró el **encapsulamiento / responsabilidad única**: que `Project` responda por su propia pertenencia (`isMember`) en vez de que `Task` meta mano en `project.memberIds`. Supiste que no rompiste nada porque **los tests siguieron verdes** después del movimiento (la red de seguridad de la fase refactor).
3.4 Que no diseñaste `Project.isMember()` por adelantado: apareció en el refactor, cuando el código ejercitado por los tests mostró que esa responsabilidad estaba mal ubicada. El diseño fue una *consecuencia* del ciclo, no un plan previo.

### Módulo 4
4.1 **Classicist**: la unidad es **un comportamiento**; mockea **solo** las dependencias incómodas/out-of-process (DB, red, reloj), usando reales las demás. **Mockist**: la unidad es **una clase**; mockea **todas** sus colaboradoras y verifica interacciones.
4.2 **Mockist → outside-in** (London): empezás por el borde (controller), y los mocks te revelan qué colaboradores e interfaces necesitás, implementando hacia adentro. **Classicist → inside-out** (Detroit): tendés a construir desde el dominio hacia afuera con objetos reales.
4.3 **Mockist**: buena para **descubrir/diseñar la colaboración** entre objetos antes de que existan (feedback de diseño temprano vía las interfaces que exigen los mocks). **Classicist**: produce tests que **sobreviven al refactor** (prueban comportamiento observable, no interacciones), dando libertad para reorganizar por dentro.
4.4 Hacia el **classicist** (posición de Khorikov). La regla: usar dependencias reales dentro del proceso y **mockear solo lo que sale del proceso y no controlás** (DB, APIs externas, reloj, red).

### Módulo 5
5.1 Significa que el test verifica *cómo* el código trabaja por dentro (qué métodos llama, en qué orden) en vez de *qué* logra (el resultado observable). El síntoma: al refactorizar sin cambiar el comportamiento, el test **se rompe igual** —tenés que tocar el test sin que haya cambiado nada observable—.
5.2 Mockear el **`Project`** (y verificar `project.isMember` fue llamado): es un colaborador propio, dentro del proceso y barato de usar real. Mockearlo no aportó nada y acopló el test a las llamadas internas del service.
5.3 Deberías verificar el **resultado observable** (estado final, valor devuelto, efecto visible: la tarea quedó asignada y persistida). Casi nunca deberías verificar **interacciones internas** (que tu objeto llamó a su propio colaborador).
5.4 Cuando la interacción cruza un **borde que te importa y no podés observar de otra forma**: que mandaste el email, que publicaste el evento a la cola, que escribiste en un sistema externo. Ahí el "que se haya llamado" *es* el comportamiento.

### Módulo 6
6.1 (1) Protección contra regresiones, (2) resistencia al refactor, (3) feedback rápido, (4) mantenibilidad.
6.2 La **resistencia al refactor**. Se maximiza **testeando el comportamiento observable (el qué), no la implementación (el cómo)** —exactamente lo que el módulo 5 enseñó: verificar resultado, no interacciones internas—.
6.3 (a) Test sobre-mockeado: **mucha protección, poca resistencia** (atrapa bugs pero salta ante refactors legítimos → frágil). (b) Test de un getter: **mucha resistencia, poca protección** (nunca se rompe pero no atrapa nada → trivial/inútil).
6.4 Porque un falso positivo erosiona la **confianza del equipo en la suite**: si los tests saltan sin que haya bugs, el equipo aprende a ignorarlos o re-correrlos hasta que pasen, y entonces la suite deja de proteger. Un falso negativo ocasional deja pasar un bug puntual, pero no destruye la confianza en todo el sistema de tests.

### Módulo 7
7.1 Te está diciendo que `TaskService` tiene **demasiado acoplamiento a dependencias concretas**. La salida es **inyectar dependencias por interfaz** (DI): pasar abstracciones (`ProjectRepository`, mailer) en vez de instanciar Postgres/servicios concretos adentro. El test fácil de armar = el diseño desacoplado.
7.2 Porque al escribir el test primero, vos sos el primero en *usar* la API del código (armar la llamada, los parámetros, leer el resultado), y sentís sus incomodidades antes que nadie. Ejemplo de señal temprana: un método que pide 9 parámetros o devuelve un objeto gigante es tedioso en el `arrange` → te empuja a una interfaz más chica.
7.3 El diseño que TDD empuja —bordes claros, dependencias inyectadas por interfaz, entidades con comportamiento, responsabilidad única— es exactamente el **dominio rico + puertos/adaptadores** de NestJS senior. Test-first tiende naturalmente a separar dominio puro de lo que sale del proceso, que es lo que pide la arquitectura hexagonal/limpia.
7.4 Porque TDD *muestra* el dolor del mal diseño, pero vos tenés que **escuchar la señal y refactorizar**. El error es reaccionar al dolor del test **mockeando todo para callarlo** (módulo 5) en vez de desacoplar: así silenciás la señal y terminás con un diseño peor *y* tests frágiles.

### Módulo 8
8.1 El primer paso es **escribir un test que reproduzca el bug y verlo fallar (rojo)**, antes de tocar el código. Ventajas: (a) el rojo confirma que reprodujiste el bug de verdad; (b) cuando se pone verde, probaste que lo arreglaste (no que creés que sí); (c) queda como protección permanente contra esa regresión.
8.2 Que **no reprodujiste el bug** —no entendiste la causa real—. Si el test pasa con el código que supuestamente tiene el bug, estás testeando otra cosa; tenés que volver a entender qué falla antes de seguir.
8.3 Porque cada nombre de test (`it('lanza si el usuario no es miembro')`) es una frase de especificación del comportamiento, y leídos juntos describen qué hace el sistema. No se desactualiza como un doc en Confluence porque **si el comportamiento cambia, el test se pone rojo y te obliga a actualizarlo** —la spec y el código están atados—.
8.4 Un fallo de producción convertido en test es exactamente lo que alimenta el **golden set** (módulo 2 de Evaluations): el caso real que falló se vuelve un caso de prueba permanente. La misma disciplina —"el fallo se convierte en caso de regresión"— en código (TDD) y en evals (IA).

### Módulo 9
9.1 **Paga**: lógica de dominio / reglas de negocio (validaciones, cálculos, máquinas de estado) y bugs (regression-first). **Estorba/no aporta** (dos de): spikes/exploración, código trivial sin lógica (getters, DTOs, mapeos), glue code/wiring de infra, UI muy visual.
9.2 Porque TDD necesita que sepas *qué* querés (para escribir el test), y en un spike justamente **todavía no lo sabés** —estás aprendiendo/explorando—. En su lugar hacés el spike sin tests, aprendés, y **tirás el código del spike**; después implementás con TDD lo que decidiste.
9.3 Cae en **mucha resistencia, cero protección** (el cuadrante trivial): nunca se rompe pero tampoco atrapa ningún bug, porque un getter no tiene lógica que pueda fallar. El esfuerzo no rinde.
9.4 Que está gastando esfuerzo de TDD en el **peor cuadrante** (tests de DTOs/getters que no protegen) persiguiendo la métrica equivocada (cobertura) —y muy probablemente, mientras tanto, dejando reglas de negocio jugosas con tests pobres—. La cobertura total no es señal de buen testing.

### Módulo 10
10.1 Porque imponer TDD estricto en cada línea genera el **dogmatismo** del módulo 9 (TDD de los DTOs) y resentimiento en el equipo. En su lugar sostiene: **todo lo que se mergea tiene tests confiables, y el equipo sabe hacer TDD donde rinde** (dominio, bugs); el *cómo* (test-first puro vs test-after disciplinado) importa menos que el resultado.
10.2 Porque un test frágil salta ante refactors legítimos (falsos positivos), así que el equipo empieza a **re-correr el CI "a ver si pasa"** o a marcar tests como `skip` —y la barrera del pipeline se vuelve teatro—. La calidad de los tests (resistencia al refactor) es lo que hace que CI sea una barrera real.
10.3 Sobre **change failure rate** (baja: los bugs se atrapan antes de prod) y **deployment frequency / lead time** (suben: podés refactorizar y entregar con confianza, sin la parálisis del "no toco lo que anda").
10.4 Porque la cobertura no mide la calidad de los tests: el PR con 100% y tests frágiles te da una suite que salta sin bugs y entrena al equipo a ignorarla (peor cuadrante), mientras que el de 70% con tests sólidos protege de verdad y te deja refactorizar. El valor del lead es saber distinguirlos en el review.

### Módulo 11
11.1 "TDD es una disciplina de **diseño**, no de **cobertura** (ni de testing a secas)". Porque su producto valioso es el diseño (bajo acoplamiento, interfaces chicas, código testeable) y la confianza para refactorizar; los tests y la cobertura son subproductos. Medirlo por % de cobertura confunde el medio con el fin.
11.2 El beneficio final es **poder cambiar el código sin miedo** (refactorizar y evolucionar con una red de seguridad). Importa tanto en sistemas longevos porque sin esa red el código se congela ("no toques eso que anda y nadie entiende") y el costo de cada cambio crece hasta paralizar el producto.
11.3 Porque ambos fallaron en **pensar dónde el test aporta valor**: el dogmático pone tests donde no rinden (DTOs, peor cuadrante) y el vago no los pone donde más rinden (la regla de negocio). Es el mismo error de criterio —no ajustar la herramienta al problema— visto desde los dos extremos.
11.4 Porque estás reentrenando el reflejo de "código primero" y los baby steps se sienten exagerados al principio. Es señal de que estás **aprendiendo la disciplina** (es una habilidad entrenable, no un interruptor): el ritmo y el payoff —diseño y confianza— llegan con práctica, en el mediano plazo, no en el primer feature.
11.5 **60% con tests sólidos y resistentes al refactor.** Porque el valor de una suite es la **confianza** que da para cambiar el código, y los tests frágiles la destruyen con falsos positivos (el equipo los ignora). Una suite chica y confiable habilita el refactor; una grande y frágil lo frena —exactamente lo contrario al objetivo de TDD—.

---

Con este módulo cerrás el par que define la calidad del código: [Testing práctico](testing.md) te dio la **mecánica** (cómo se escribe un test que da confianza) y este te dio la **disciplina** (cómo se diseña *con* tests). Recorriste el ciclo red-green-refactor, las dos escuelas (classicist por defecto, mockist para descubrir diseño), los cuatro atributos de Khorikov con la resistencia al refactor como pilar rey, cómo el test-first empuja el diseño hacia bajo acoplamiento, los usos de regression-first y test-como-spec, el criterio de cuándo *no* hacer TDD, y su rol como disciplina de equipo ligada a CI/CD y a las métricas DORA. El hilo con el resto del temario queda explícito: el mismo "escalá la herramienta al problema" del criterio de [Kubernetes](kubernetes.md), [Agentes](agentes.md) y [Observabilidad](observabilidad.md), y la conexión con el eval-driven development del [track de IA](evals.md). El último tramo del track Tech Lead es **liderazgo técnico** (ADRs, DORA, Team Topologies): cómo se decide, se mide y se organiza un equipo que entrega con esta disciplina.
