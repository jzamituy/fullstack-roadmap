# Domain-Driven Design: modelar el negocio, no las tablas

**DDD estratégico y táctico, modelado colaborativo y el giro sociotécnico · TypeScript · estado del arte 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume el archivo de [NestJS nivel Senior](nestjs-senior.md) —que tiene una **intro** a DDD en su sección 2 (lenguaje ubicuo, bounded contexts, entity/value object/aggregate)—: **este módulo la profundiza y la extiende**, no la repite. También conecta con [Event-driven](event-driven.md) (eventos de dominio, CQRS), [TDD](tdd.md) (el dominio rico que TDD empuja) y, al final, con el track de [IA](rag.md). **DDD no es una arquitectura ni una receta de carpetas**: es un **enfoque de diseño** para que el software **modele el negocio** con la suficiente fidelidad como para que cambiarlo sea barato y razonarlo sea posible. El objetivo es doble: las **técnicas** (estratégicas y tácticas, con ejemplos prácticos en TypeScript) y —tanto o más importante— el **criterio**: DDD cuesta esfuerzo, y aplicarlo donde no hace falta es tan dañino como no aplicarlo donde sí.

**Lo que asumimos.** TypeScript (modo `--strict`), POO y el patrón Repository ([NestJS senior](nestjs-senior.md)), nociones de transacciones ([PostgreSQL](postgresql.md)) y de eventos ([Event-driven](event-driven.md)). El dominio de ejemplo es un **e-commerce** (Ventas, Facturación, Envíos, Soporte), con el `Pedido` como protagonista táctico.

**Para practicar.** No hace falta instalar nada: es modelado en TypeScript. Los snippets de cada módulo son **fragmentos ilustrativos** (asumen tipos de soporte como `Dinero`, `ClienteId`, `LineaPedido`, etc.); el código del **capstone** y de las **soluciones** compila en `--strict` cuando se completa con esos tipos. Lo que sí conviene tener a mano es un **dominio real** (el tuyo, o el e-commerce del módulo) para no quedarte en lo abstracto.

> Nota sobre el "estado del arte 2026": DDD no tiene *breakthroughs* anuales; evoluciona por **síntesis**. Lo moderno (vs el "blue book" de Evans de 2003) es: DDD **pragmático y liviano** (Vlad Khononov, *Learning Domain-Driven Design*, 2021), el **modelado colaborativo** (EventStorming de Brandolini; Event Modeling de Adam Dymitruk; Domain Storytelling), el **giro sociotécnico** (Team Topologies, Skelton & Pais), el diseño táctico **type-driven/funcional** (Scott Wlaschin, *Domain Modeling Made Functional*), el **modular monolith** por sobre el "microservicios primero", y la **relevancia en la era de la IA**. Todo eso está en este módulo, con sus fuentes.

**Índice de módulos**
1. Qué es DDD (y qué no): modelar el negocio, no las tablas
2. El lenguaje ubicuo: la herramienta más subestimada
3. Subdominios: core, supporting, generic — dónde invertir el esfuerzo
4. Bounded contexts: el corazón del DDD estratégico
5. Context mapping: cómo se relacionan los contextos (ACL, OHS, conformist…)
6. Modelado colaborativo: EventStorming, Event Modeling y Domain Storytelling
7. Táctico I: value objects y entities (hacer imposibles los estados inválidos)
8. Táctico II: aggregates y el límite de consistencia (las reglas de Vernon)
9. Táctico III: domain events, repositories, domain services, factories
10. Las capas: dominio, aplicación, infraestructura (el puente con hexagonal)
11. DDD y la arquitectura moderna: modular monolith, vertical slices, CQRS/ES
12. DDD y los equipos: Conway y Team Topologies (el giro sociotécnico)
13. DDD en la era de la IA y el criterio de cierre

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es DDD (y qué no): modelar el negocio, no las tablas

**Teoría.** El malentendido más caro sobre DDD es creer que es una **arquitectura** (capas, carpetas `domain/`, `application/`, `infrastructure/`) o un **catálogo de patrones** (entities, value objects, repositories) que copiás a cualquier proyecto. Eso es la **punta del iceberg táctico**, y aplicado sin lo de abajo produce **estructura vacía**: carpetas con nombres de DDD y código que sigue siendo un CRUD anémico.

DDD, en su esencia (Eric Evans, 2003), es un **enfoque para atacar la complejidad del software poniendo el foco en el dominio** —el negocio— y en construir un **modelo** de ese dominio que el código exprese fielmente. Dos ideas fundacionales:

- **El modelo es el corazón.** No el esquema de la base de datos, no el framework: el **modelo del dominio** —los conceptos del negocio, sus reglas, su lenguaje— es lo que diseñás, y todo lo demás (DB, API, UI) son detalles que sirven a ese modelo. El error clásico es diseñar **de la tabla hacia afuera** ("tengo una tabla `orders` con estas columnas") en vez de **del negocio hacia adentro** ("¿qué es un Pedido, qué puede pasarle, qué reglas lo gobiernan?").
- **La colaboración con el experto del dominio es parte del método.** DDD no es algo que el dev hace solo en su cueva: el modelo se construye **hablando con quien entiende el negocio** (el "domain expert"), destilando su conocimiento en un modelo compartido. El modelo y el código evolucionan juntos a medida que el entendimiento mejora.

Y lo que DDD **no** es: no es para todo proyecto (módulo 13 — un CRUD simple no lo necesita), no es una arquitectura específica (se combina con hexagonal, con monolito modular, con microservicios — módulos 10-11), y no es "tener una carpeta `domain`". La marca de quien entendió DDD: **distingue cuándo el dominio es lo suficientemente complejo como para justificar el esfuerzo, y cuándo no** (módulo 3).

La frase mental: **DDD no es una arquitectura ni una caja de patrones; es diseñar el software alrededor de un modelo fiel del negocio, construido con el experto del dominio —del negocio hacia adentro, no de la tabla hacia afuera—; los patrones tácticos son la punta del iceberg, y sin el trabajo de modelado debajo quedan como carpetas vacías—.**

**Ejercicios 1**
1.1 ¿Cuál es el malentendido más común sobre DDD, y por qué "tener una carpeta `domain/`" no es hacer DDD?
1.2 ¿Qué significa diseñar "del negocio hacia adentro" en vez de "de la tabla hacia afuera"? Dá un ejemplo.
1.3 ¿Por qué la colaboración con el experto del dominio es parte del método y no un extra?
1.4 ¿Qué distingue a quien "entendió" DDD respecto a quien solo copia los patrones tácticos?

---

## Módulo 2 — El lenguaje ubicuo: la herramienta más subestimada

**Teoría.** El **lenguaje ubicuo** (*ubiquitous language*) es, para mucha gente que lo subestima, lo más poderoso de DDD: **un lenguaje común y riguroso, compartido entre el negocio y el código, usado en todos lados** —en las conversaciones, en la documentación, en los nombres de clases, métodos y variables—. "Ubicuo" = está en todas partes, sin traducción entre "como habla el negocio" y "como nombra el código".

El problema que resuelve: normalmente hay una **brecha de traducción**. El negocio dice "cancelar una reserva" y el código tiene `updateStatus(2)`. Cada vez que un dev habla con el negocio, traduce mentalmente —y en cada traducción se pierde o distorsiona información, nacen bugs y malentendidos—. El lenguaje ubicuo **elimina la traducción**: si el negocio dice "una reserva se cancela", el código tiene `reserva.cancelar()`, no `updateStatus(2)`.

```ts
// ❌ El código no habla el lenguaje del negocio: hay traducción (y bugs en la traducción)
pedido.estado = 3;
pedido.flag = true;
servicio.process(pedido, 2);

// ✅ El código ES el lenguaje del negocio: cualquiera del negocio entiende qué hace
pedido.confirmar();
pedido.aplicarDescuento(cupon);
if (pedido.estaListoParaEnviar()) {
  despacho.programar(pedido);
}
```

Tres cosas que tenés que internalizar sobre el lenguaje ubicuo:

- **Es preciso y libre de ambigüedad** *dentro de su contexto* (módulo 4): cada término significa una sola cosa. Si "cliente" significa dos cosas distintas, son dos términos (o dos contextos), no uno ambiguo.
- **Se construye con el negocio, no se inventa solo.** Los términos salen de cómo habla la gente del dominio, refinados hasta quedar sin ambigüedad. Si descubrís un concepto que el negocio nombra pero tu modelo no tiene, te falta un término.
- **Cambia cuando cambia el entendimiento.** No es un glosario que se escribe una vez; evoluciona. Cuando aprendés algo nuevo del dominio, el lenguaje (y el código) se actualiza.

Por qué es la base de todo lo demás: los bounded contexts (módulo 4) son, literalmente, las fronteras donde un lenguaje ubicuo es consistente. Sin lenguaje ubicuo, el modelo táctico (módulos 7-9) modela conceptos vagos. La frase mental: **el lenguaje ubicuo es un vocabulario común, preciso y sin traducción entre el negocio y el código —si el negocio dice "cancelar reserva", el código dice `reserva.cancelar()`, no `updateStatus(2)`—; eliminar esa traducción elimina una categoría entera de bugs y es la base sobre la que se apoyan los contextos y el modelo táctico—.**

**Ejercicios 2**
2.1 ¿Qué es el lenguaje ubicuo y por qué "ubicuo"?
2.2 ¿Qué problema concreto resuelve? Dá el ejemplo de "cancelar reserva" vs `updateStatus(2)` y qué se pierde en la traducción.
2.3 ¿De dónde salen los términos del lenguaje ubicuo y qué hacés si encontrás un concepto del negocio que tu modelo no nombra?
2.4 ¿Por qué el lenguaje ubicuo es la base de los bounded contexts y del modelo táctico?

---

## Módulo 3 — Subdominios: core, supporting, generic — dónde invertir el esfuerzo

**Teoría.** Acá está la heurística que, en el DDD moderno (Khononov), **ordena todo lo demás** y separa al que aplica DDD con criterio del que lo aplica por dogma. Tu sistema no es un bloque uniforme: se divide en **subdominios**, y no todos merecen el mismo esfuerzo de modelado. Hay tres tipos:

- **Core domain (dominio central)**: lo que te hace **competitivo y distinto** —la razón por la que tu negocio gana—. Para un e-commerce de subastas, el motor de pujas; para Uber, el matching y pricing. Es **complejo, cambia seguido, y es tu ventaja**. Acá invertís el **máximo esfuerzo**: el modelado táctico rico (módulos 7-9), tu mejor gente, construido **in-house** (no lo comprás, porque ES tu diferencial).
- **Supporting subdomain (subdominio de soporte)**: necesario para que el core funcione, pero **no es tu diferencial** y es relativamente simple. Para el e-commerce, quizá la gestión de catálogo. Lo construís, pero **sin sobre-invertir**: no necesita el modelado táctico más sofisticado; a veces un CRUD bien hecho alcanza.
- **Generic subdomain (subdominio genérico)**: un problema ya resuelto, igual para todas las empresas —autenticación, facturación fiscal, envío de emails, pagos—. **No lo construís: lo comprás o usás algo off-the-shelf** (Auth0, Stripe, un proveedor de email). Construir tu propio sistema de auth "a mano" es desperdiciar esfuerzo en algo que no te diferencia y que otros ya resolvieron mejor.

La heurística de esfuerzo, que es **el** insight estratégico moderno:

| Subdominio | Qué es | Esfuerzo | Decisión |
|---|---|---|---|
| **Core** | Tu ventaja competitiva, complejo | **Máximo** (DDD táctico rico) | Construir in-house, mejor gente |
| **Supporting** | Necesario, no diferencial, simple | Medio (a veces CRUD) | Construir simple |
| **Generic** | Resuelto, igual para todos | Mínimo | **Comprar / off-the-shelf** |

Un matiz importante: la clasificación **no es estática ni siempre nítida**. Un subdominio puede ser core hoy y *commoditizarse* mañana (lo que te diferenciaba se vuelve estándar de la industria y conviene comprarlo), o ser "core" solo en una dimensión y genérico en otra. Tratá los tres tipos como una **guía de inversión que se revisa**, no como etiquetas fijas que pegás una vez.

El error caro y muy común: invertir esfuerzo de DDD táctico en un subdominio **genérico** (modelar elaboradamente tu propio sistema de auth) mientras el **core** —donde sí importa— queda como un CRUD anémico. La regla: **identificá tu core domain y volcá ahí el esfuerzo de modelado; comprá lo genérico; mantené simple lo de soporte.** Esto conecta directo con el criterio de cierre (módulo 13): DDD táctico es para el **core complejo**, no para todo. La frase mental: **dividí el sistema en subdominios —core (tu ventaja, complejo → máximo esfuerzo de DDD, in-house), supporting (necesario, simple → esfuerzo medio), generic (resuelto, igual para todos → comprar)— y volcá el modelado rico donde te diferenciás, no en un genérico que otros ya resolvieron—.**

**Ejercicios 3**
3.1 Definí core, supporting y generic subdomain con un ejemplo de cada uno.
3.2 ¿Qué decisión de "construir vs comprar" corresponde a cada tipo y por qué?
3.3 ¿Cuál es el error caro respecto a dónde se invierte el esfuerzo de DDD? Dá un ejemplo concreto.
3.4 ¿Por qué esta clasificación es "la heurística que ordena todo lo demás" en el DDD moderno?

---

## Módulo 4 — Bounded contexts: el corazón del DDD estratégico

**Teoría.** El **bounded context** (contexto delimitado) es el concepto más importante del DDD estratégico: **una frontera explícita dentro de la cual un modelo —y su lenguaje ubicuo— es consistente y significa una sola cosa**. Fuera de esa frontera, los mismos términos pueden significar algo distinto, y está bien.

El ejemplo canónico es el término **"Cliente"**:

- En **Ventas**, un Cliente tiene historial de compras, descuentos, carrito, preferencias.
- En **Facturación**, un Cliente tiene datos fiscales, dirección de facturación, condición impositiva.
- En **Soporte**, un Cliente tiene tickets, nivel de SLA, historial de contactos.
- En **Envíos**, un Cliente es básicamente una dirección de entrega y una ventana horaria.

El error que parece "lo correcto" pero es un desastre: crear **un único modelo `Cliente` gigante** que sirva a todos los contextos. Termina siendo una clase con 60 campos donde la mitad son `null` según quién la use, acoplada a todo, que nadie entiende y que cambia por razones de cuatro equipos distintos. La jugada DDD: **un modelo `Cliente` por contexto**, cada uno con **solo lo que ese contexto necesita**, y se relacionan por **identidad compartida (un `clienteId`)** y **eventos**, no por una clase compartida.

```ts
// Cada contexto es su propio módulo/namespace: el MISMO nombre `Cliente` convive
// porque vive en scopes distintos (en el repo serían ventas/cliente.ts y facturacion/cliente.ts).
namespace Ventas {
  export class Cliente {          // el Cliente que le importa a Ventas
    constructor(
      readonly id: ClienteId,
      private readonly historialCompras: Compra[],
      private readonly descuentos: Descuento[],
    ) {}
  }
}

namespace Facturacion {
  export class Cliente {          // OTRO Cliente — mismo id, modelo distinto
    constructor(
      readonly id: ClienteId,     // misma identidad
      readonly datosFiscales: DatosFiscales,
      readonly condicionImpositiva: CondicionImpositiva,
    ) {}
  }
}
```

Por qué los bounded contexts son el corazón del DDD estratégico:

- **Domestican la complejidad**: cada contexto es un modelo chico y coherente que entendés entero, en vez de un modelo gigante imposible de razonar.
- **Permiten autonomía**: cada contexto evoluciona a su ritmo, con su lenguaje, sin que un cambio en Soporte rompa Ventas.
- **Son la guía natural de los límites** de módulos (en un monolito modular) o de servicios (en microservicios) —módulo 11—. Y, como veremos, de los **equipos** (módulo 12).

La relación con los subdominios (módulo 3): un **subdominio** es del **problema** (cómo el negocio se divide); un **bounded context** es de la **solución** (cómo dividís tu software). Idealmente se alinean (un contexto por subdominio), pero no siempre es 1:1. La frase mental: **un bounded context es una frontera donde el modelo y su lenguaje son consistentes y un término significa una sola cosa; en vez de un modelo `Cliente` gigante que sirve a todos, tenés un modelo por contexto (Ventas, Facturación, Soporte…) con solo lo suyo, ligados por identidad y eventos —y esa frontera es la guía de tus módulos, servicios y equipos—.**

**Ejercicios 4**
4.1 Definí bounded context. ¿Qué es lo que es "consistente" dentro de la frontera?
4.2 Tomá "Cliente" y mostrá cómo difiere entre Ventas, Facturación y Soporte. ¿Por qué un modelo único es un error?
4.3 Si no hay una clase `Cliente` compartida, ¿cómo se relacionan los contextos entre sí?
4.4 ¿Cuál es la diferencia entre un subdominio y un bounded context? (problema vs solución)

---

## Módulo 5 — Context mapping: cómo se relacionan los contextos (ACL, OHS, conformist…)

**Teoría.** Si tenés varios bounded contexts, **se tienen que comunicar** —Ventas necesita datos de Facturación, Envíos reacciona a un pedido confirmado—. El **context mapping** es el catálogo de **patrones de relación** entre contextos, y elegir el correcto define cuán acoplado o protegido queda tu modelo. Los que importan en la práctica:

- **Anti-Corruption Layer (ACL)** — el más útil. Una capa de traducción que **protege tu modelo del modelo de otro contexto** (o de un sistema legacy/externo). Cuando consumís datos de un contexto cuyo modelo no querés que contamine el tuyo, metés un ACL en el medio que traduce **su** lenguaje al **tuyo**. Imprescindible al integrar con sistemas legacy o APIs de terceros con modelos feos.

```ts
// ACL: traduce el modelo EXTERNO (feo, ajeno) a TU lenguaje ubicuo, protegiendo tu dominio
class AclFacturacion {
  constructor(private readonly api: ApiFacturacionExterna) {}

  async obtenerCondicion(clienteId: ClienteId): Promise<CondicionImpositiva> {
    const dto = await this.api.getCustomer(clienteId.valor); // modelo externo, con sus rarezas
    // traducimos a NUESTRO concepto; el resto de nuestro dominio nunca ve el modelo externo
    return CondicionImpositiva.desdeCodigoExterno(dto.tax_status_code);
  }
}
```

- **Open Host Service (OHS) + Published Language**: al revés del ACL —cuando **vos** sos el proveedor y muchos te consumen, exponés un **API pública bien definida con un lenguaje publicado** (ej. un esquema de eventos, un contrato OpenAPI) para no acoplarte a cada consumidor. Conecta con [contract testing](api-testing.md) y [event-driven](event-driven.md).
- **Customer-Supplier**: dos contextos donde el *upstream* (proveedor) y el *downstream* (cliente) negocian; el proveedor tiene en cuenta las necesidades del consumidor. Hay colaboración y prioridades compartidas.
- **Conformist**: el downstream **se conforma** al modelo del upstream sin traducir (no puede negociar, ej. una API de un gigante que no vas a cambiar). Más barato que un ACL, pero **te acoplás** a su modelo —aceptable solo si su modelo es razonable—.
- **Shared Kernel**: dos contextos comparten **una porción pequeña de modelo** (código común). Poderoso pero **peligroso**: ese kernel compartido acopla los dos equipos (cualquier cambio los afecta a ambos), así que se reserva para un núcleo chico, estable y co-gestionado.
- **Partnership**: dos equipos/contextos con éxito o fracaso mutuo, coordinación estrecha y bidireccional.
- **Separate Ways**: la decisión de **no integrar** —duplicar es más barato que el costo de integrar—. A veces la respuesta correcta.
- **Big Ball of Mud**: no es un patrón a buscar, sino el que **identificás para defenderte**. Es el contexto legacy sin estructura ni lenguaje claro (donde todo se mezcla); al mapear sistemas heredados conviene marcarlo como tal y **aislarlo detrás de un ACL** para que su desorden no contamine los contextos sanos.

El **context map** es el diagrama que muestra tus contextos y qué patrón los relaciona —una herramienta de comunicación de arquitectura de primer nivel, porque hace visible el acoplamiento—. La regla de criterio: **cuando integrás con un modelo que no controlás o no querés que te contamine, ACL; cuando exponés a muchos, OHS + lenguaje publicado; conformist solo si el modelo ajeno es bueno y no lo podés negociar; shared kernel lo mínimo posible.** La frase mental: **el context mapping cataloga cómo se relacionan los contextos —ACL (traducí y protegé tu modelo del ajeno, el más útil), OHS+published language (exponé un contrato estable si muchos te consumen), conformist (acoplate sin traducir si no podés negociar), shared kernel (compartí lo mínimo, acopla equipos), separate ways (no integres)—; el context map hace visible el acoplamiento y es decisión de arquitectura de primer nivel—.**

**Ejercicios 5**
5.1 ¿Qué es y para qué sirve un Anti-Corruption Layer? Dá un caso donde es imprescindible.
5.2 Diferenciá Open Host Service de ACL: ¿quién protege a quién en cada uno?
5.3 ¿Qué es un Conformist y cuándo es una decisión aceptable a pesar del acoplamiento?
5.4 ¿Por qué el Shared Kernel es poderoso pero peligroso, y qué tamaño debería tener?

---

## Módulo 6 — Modelado colaborativo: EventStorming, Event Modeling y Domain Storytelling

**Teoría.** Una de las grandes evoluciones del DDD moderno (vs el blue book) son las **técnicas de modelado colaborativo**: cómo **descubrir** el dominio **junto con el negocio**, rápido y visualmente, en vez de que el dev modele solo a partir de un documento de requisitos. Son talleres con post-its (físicos o digitales) que ponen a devs y expertos a construir el modelo **en la misma sala**.

**EventStorming** (Alberto Brandolini) es la más conocida. La idea: explorar el dominio a través de los **eventos de dominio** —cosas que pasaron, en pasado: "Pedido Confirmado", "Pago Recibido", "Envío Despachado"—. Se hace con post-its de **colores con significado**:

- 🟧 **Eventos de dominio** (naranja): algo que pasó (en pasado). El centro de todo.
- 🟦 **Comandos** (azul): la acción/intención que dispara un evento ("Confirmar Pedido").
- 🟨 **Aggregates** (amarillo): el objeto que recibe el comando y emite el evento.
- 🟪 **Policies / process managers** (lila): "cuando pasa X, entonces Y" (reacciones).
- 🟩 **Read models** (verde): la info que alguien necesita ver para decidir.
- 🩷 **Sistemas externos** (rosa): otros sistemas involucrados.
- 🟥 **Hotspots** (rojo): dudas, conflictos, problemas sin resolver —oro puro, marcan dónde el dominio es confuso o nadie se pone de acuerdo—.

Hay tres niveles: **big-picture** (explorar todo el negocio, descubrir bounded contexts), **process-level** (un flujo concreto) y **design-level** (afinar aggregates y comandos, casi código). EventStorming es excepcional para **descubrir los bounded contexts** (módulo 4) y los **aggregates** (módulo 8) de forma colaborativa, y para que afloren los malentendidos (los hotspots) **antes** de escribir código.

Las otras dos técnicas modernas que conviene conocer:

- **Event Modeling** (Adam Dymitruk, ~2020): más **estructurada** que EventStorming. Modela el sistema como una **línea de tiempo** (un "blueprint") de comandos → eventos → read models a lo largo de pantallas/wireframes. Encaja muy bien con **CQRS/Event Sourcing** ([event-driven](event-driven.md)) y con especificar qué construir.
- **Domain Storytelling** (Stefan Hofer & Henning Schwentner): captura **historias** del dominio con una notación pictográfica simple (actor → actividad → objeto), ideal para entender un flujo contándolo como una narración.

Por qué esto es "lo moderno": traslada el modelado de "el dev solo con un documento" a "un taller donde el conocimiento del negocio se destila en grupo, visible y rápido". El descubrimiento del modelo se vuelve **colaborativo y barato de iterar** (mover un post-it cuesta nada; refactorizar código cuesta mucho). La frase mental: **el modelado colaborativo (EventStorming con sus post-its de colores —eventos naranja, comandos azul, aggregates amarillo, policies lila, hotspots rojo—; Event Modeling como blueprint temporal; Domain Storytelling como narrativa) descubre el dominio, los bounded contexts y los aggregates JUNTO con el negocio, haciendo aflorar los malentendidos antes de escribir una línea de código—.**

**Ejercicios 6**
6.1 ¿Qué problema del modelado "clásico" resuelven las técnicas colaborativas como EventStorming?
6.2 En EventStorming, ¿qué representan los post-its naranja, azul, amarillo y rojo? ¿Por qué los hotspots (rojo) son tan valiosos?
6.3 ¿Para qué dos cosas del DDD estratégico/táctico es especialmente bueno EventStorming?
6.4 ¿En qué se diferencia Event Modeling de EventStorming, y con qué patrón de arquitectura encaja bien?

---

## Módulo 7 — Táctico I: value objects y entities (hacer imposibles los estados inválidos)

**Teoría.** Bajamos al **DDD táctico**: los bloques con los que modelás *dentro* de un bounded context. Los dos fundamentales son **entity** y **value object**, y la distinción es por **identidad**.

**Value Object (objeto de valor).** No tiene identidad propia: se define **por sus valores** y es **inmutable**. `Dinero(100, "USD")`, `Email("a@b.com")`, un `RangoDeFechas`. Dos value objects con los mismos valores son **intercambiables**. Su superpoder, y el ángulo **moderno/type-driven** (Scott Wlaschin, *Domain Modeling Made Functional*): **hacer imposibles los estados inválidos**, poniendo la validación en el **constructor** para que un value object inválido **no pueda existir**.

```ts
class Email {
  private constructor(readonly valor: string) {}   // constructor privado: solo se crea vía `crear`

  static crear(input: string): Email {
    const v = input.trim().toLowerCase();
    if (!/^[^@\s]+@[^@\s]+\.[^@\s]+$/.test(v)) {
      throw new Error(`Email inválido: ${input}`);  // un Email inválido NO puede existir
    }
    return new Email(v);
  }

  equals(otro: Email): boolean {                    // igualdad por VALOR, no por referencia
    return this.valor === otro.valor;
  }
}
// A partir de acá, si una función recibe un `Email`, el sistema de tipos GARANTIZA que es válido.
// La validación dejó de estar desparramada por toda la app; vive en el tipo.
```

**Entity (entidad).** Tiene una **identidad que persiste en el tiempo**, más allá de sus atributos. Dos usuarios con el mismo nombre son distintos porque tienen distinto `id`; y un usuario que cambia de nombre **sigue siendo el mismo**. La igualdad se define por **identidad**, no por valores. Las entities son **mutables** (su estado cambia a lo largo de su vida), pero esos cambios pasan por **métodos que protegen las reglas**, no por setters públicos.

```ts
class Usuario {
  constructor(
    readonly id: UsuarioId,          // la identidad: lo que lo hace "el mismo" aunque cambie
    private nombre: string,
    private email: Email,            // ← un value object como atributo (composición)
  ) {}

  equals(otro: Usuario): boolean {
    return this.id.equals(otro.id);  // igualdad por IDENTIDAD, no por atributos
  }

  cambiarEmail(nuevo: Email): void { // cambio de estado vía método (no setter público)
    this.email = nuevo;
  }
}
```

La regla práctica que distingue al senior: **modelá con value objects todo lo que puedas.** En vez de pasar `string` y `number` sueltos (*primitive obsession*), envolvés los conceptos del dominio en value objects con reglas. Eso elimina una categoría enorme de bugs (no podés sumar USD + EUR, no podés crear un email inválido, no podés pasar un `clienteId` donde iba un `pedidoId` si son tipos distintos). El contra: más clases —no conviertas *cada* string en value object, reservalo para conceptos con **reglas o significado** (dinero, email, cantidades, rangos, identificadores), no para un nombre cualquiera—. La frase mental: **value object = sin identidad, inmutable, igualdad por valor, con la validación en el constructor para hacer imposibles los estados inválidos (type-driven); entity = identidad persistente, igualdad por id, mutable vía métodos que protegen reglas; y la jugada senior es modelar con value objects en vez de primitivos sueltos, reservándolos para conceptos con reglas—.**

**Ejercicios 7**
7.1 ¿Cuál es la diferencia esencial entre un value object y una entity? (la palabra clave es una sola)
7.2 ¿Qué significa "hacer imposibles los estados inválidos" y cómo lo logra un value object? Dá el ejemplo del Email.
7.3 ¿Por qué una entity es mutable pero no tiene setters públicos? ¿Cómo cambia su estado entonces?
7.4 ¿Qué es la "primitive obsession" y por qué modelar con value objects la combate? ¿Cuándo NO conviene un value object?
7.5 **Implementá.** Escribí un value object `Dinero` (monto + moneda) con igualdad por valor, validación en el constructor (monto no negativo) y un método `sumar` que **lance** si las monedas difieren. Que compile en `--strict`.

---

## Módulo 8 — Táctico II: aggregates y el límite de consistencia (las reglas de Vernon)

**Teoría.** El **aggregate** es el patrón táctico más importante y el peor entendido. Un aggregate es un **grupo de entities y value objects que se tratan como una sola unidad de consistencia**, con una **raíz (aggregate root)** que es la **única puerta de entrada**. La raíz garantiza las **invariantes** (las reglas que siempre deben cumplirse) de todo el grupo.

El ejemplo clásico: un `Pedido` (raíz) contiene `LineasDePedido`. **No modificás una línea por fuera**: le pedís al `Pedido` que lo haga, y el `Pedido` protege sus invariantes ("un pedido confirmado no acepta nuevas líneas", "el total no puede ser negativo").

```ts
class Pedido {                          // aggregate root
  readonly id!: PedidoId;               // identidad (constructor de creación/reconstitución omitido; ver módulo 9)
  private lineas: LineaPedido[] = [];
  private estado: "borrador" | "confirmado" = "borrador";
  private eventos: EventoDeDominio[] = [];

  agregarLinea(producto: ProductoId, cantidad: number, precio: Dinero): void {
    if (this.estado === "confirmado") {
      throw new Error("Un pedido confirmado no acepta nuevas líneas"); // invariante protegida
    }
    if (cantidad <= 0) throw new Error("La cantidad debe ser positiva");
    this.lineas.push(new LineaPedido(producto, cantidad, precio));
  }

  confirmar(): void {
    if (this.lineas.length === 0) throw new Error("No se puede confirmar un pedido vacío");
    this.estado = "confirmado";
    this.eventos.push(new PedidoConfirmado(this.id, this.total())); // evento de dominio (módulo 9)
  }

  private total(): Dinero {
    // suma de las líneas; la invariante "nunca negativo" se sostiene porque
    // cada LineaPedido ya validó cantidad > 0 y precio >= 0 en su constructor
    return this.lineas.reduce((acc, l) => acc.sumar(l.subtotal()), Dinero.cero("USD"));
  }
}
```

Acá están las **reglas de diseño de aggregates** (Vaughn Vernon, *Effective Aggregate Design* — el material de referencia moderno), que es lo que separa un aggregate bien diseñado de uno que rompe en producción:

1. **Protegé invariantes verdaderas dentro del límite del aggregate.** El aggregate existe para garantizar consistencia **inmediata** de sus reglas. Lo que debe ser consistente *al instante* (el total del pedido = suma de sus líneas) va **dentro** del mismo aggregate.

2. **Diseñá aggregates chicos.** El error #1 es el aggregate gigante (un `Cliente` que contiene sus pedidos, que contienen sus líneas, que contienen…). Aggregates grandes = transacciones grandes, contención, problemas de performance y de concurrencia. Preferí **muchos aggregates chicos** a pocos enormes.

3. **Referenciá otros aggregates por identidad (ID), no por objeto.** Un `Pedido` **no** tiene un objeto `Cliente` adentro; tiene un `clienteId`. Esto mantiene los aggregates chicos y desacoplados, y evita cargar medio sistema en memoria para tocar un pedido.

```ts
class Pedido {
  constructor(
    readonly id: PedidoId,
    readonly clienteId: ClienteId,   // ✅ por ID, NO un objeto Cliente — otro aggregate
  ) {}
}
```

4. **Una transacción = un aggregate.** Cada transacción modifica **un solo aggregate**. La consistencia *entre* aggregates es **eventual** (módulo 9: vía eventos de dominio), no inmediata. Si "confirmar un pedido" debe "descontar stock" (otro aggregate), eso **no** va en la misma transacción: el `Pedido` emite `PedidoConfirmado`, y un handler descuenta el stock después (consistencia eventual, [event-driven](event-driven.md)).

Esta última es la regla más contraintuitiva y la más importante: **dentro de un aggregate, consistencia inmediata; entre aggregates, consistencia eventual.** Querer consistencia inmediata entre varios aggregates te lleva a transacciones gigantes y a un diseño que no escala. La frase mental: **un aggregate es una unidad de consistencia con una raíz que es la única puerta y protege las invariantes; diseñalos CHICOS, referenciá otros aggregates por ID (no por objeto), y respetá "una transacción = un aggregate" —consistencia inmediata dentro del límite, eventual (vía eventos) entre aggregates—; el error #1 es el aggregate gigante que arrastra medio sistema en cada transacción—.**

**Ejercicios 8**
8.1 ¿Qué es un aggregate y qué rol cumple la raíz (aggregate root)? ¿Por qué es "la única puerta"?
8.2 ¿Por qué conviene diseñar aggregates chicos? ¿Qué problemas trae el aggregate gigante?
8.3 ¿Por qué se referencia a otros aggregates por ID y no por objeto? Dá el ejemplo de `Pedido`/`Cliente`.
8.4 Explicá la regla "una transacción = un aggregate" y qué implica para la consistencia entre aggregates. Dá el ejemplo de confirmar pedido / descontar stock.
8.5 **Implementá.** El negocio define que, por un **límite operativo de picking/packing del depósito**, un pedido no puede tener más de 20 líneas distintas. Agregá esa invariante de negocio al método `agregarLinea` del aggregate `Pedido` y escribí un test que la viole (espere el error). (Ojo al modelar: una invariante *de negocio* —tiene una razón del dominio— no es lo mismo que un límite técnico arbitrario; protegé en el aggregate las primeras.)

---

## Módulo 9 — Táctico III: domain events, repositories, domain services, factories

**Teoría.** Los bloques tácticos que completan el modelo, cada uno con un rol preciso:

**Domain events (eventos de dominio).** Algo **relevante para el negocio que ocurrió**, expresado en pasado: `PedidoConfirmado`, `PagoRechazado`. Son la forma de comunicar cambios **entre aggregates** y **entre bounded contexts** (módulo 8: consistencia eventual). Un aggregate los **emite**; otros reaccionan. Son inmutables (ya pasó) y llevan los datos del hecho.

```ts
class PedidoConfirmado implements EventoDeDominio {
  readonly ocurridoEn: Date;
  constructor(readonly pedidoId: PedidoId, readonly total: Dinero) {
    this.ocurridoEn = new Date();
  }
}
// El Pedido lo emite (módulo 8); un handler en Envíos reacciona ("cuando PedidoConfirmado → preparar envío").
// Es el corazón de la arquitectura event-driven (ver event-driven.md).
```

**Repository (repositorio).** La abstracción para **persistir y recuperar aggregates** como si fueran una colección en memoria, **ocultando** la base de datos. **Un repositorio por aggregate root** (no por tabla ni por entity): `PedidoRepository`, no `LineaPedidoRepository`. La **interfaz vive en el dominio** (es parte del modelo: "necesito guardar y traer pedidos"); la **implementación vive en infraestructura** (Postgres, Mongo) —el patrón puerto/adaptador de [hexagonal](nestjs-senior.md), módulo 10—.

```ts
// En el DOMINIO: el puerto, en el lenguaje del negocio
interface PedidoRepository {
  obtenerPorId(id: PedidoId): Promise<Pedido | null>;
  guardar(pedido: Pedido): Promise<void>;   // guarda el aggregate ENTERO (raíz + lo suyo)
}
// La implementación (PedidoRepositoryPostgres) vive en INFRAESTRUCTURA y el dominio no la conoce.
```

**Persistir el aggregate sin filtrar el ORM.** Acá está la tensión real que toda entrevista de arquitectura toca: tu aggregate tiene **constructor privado, campos privados e invariantes**, pero un ORM (TypeORM/Prisma) quiere entidades anémicas con setters públicos o decoradores. Si ponés decoradores de TypeORM sobre tu `Pedido`, el ORM se **filtró** al dominio (acoplamiento). La salida es un **mapper en infraestructura** que traduce entre el modelo de persistencia (filas/DTOs) y el aggregate, y un constructor de **reconstitución** separado del de creación:

```ts
class Pedido {
  // crear(): valida invariantes — para un pedido NUEVO
  static crear(id: PedidoId, clienteId: ClienteId): Pedido { /* ...reglas... */ }
  // reconstituir(): rehidrata desde la DB SIN re-validar (esos datos ya fueron válidos)
  static reconstituir(estado: PedidoEstadoPersistido): Pedido { /* ...arma sin chequear... */ }
}
```

Dos puntos que el repositorio resuelve y que hay que nombrar: (1) **`crear` vs `reconstituir`** —crear corre las invariantes, reconstituir confía en lo que ya estaba en la base, sin volver a validar ni exponer setters—; (2) **concurrencia**: como "una transacción = un aggregate" (módulo 8), dos transacciones sobre el mismo `Pedido` se resuelven con **optimistic locking** (un campo `version` que se chequea al guardar; si cambió, reintentás) —el mecanismo del módulo de [PostgreSQL](postgresql.md)—.

**Domain service (servicio de dominio).** Lógica de dominio que **no pertenece naturalmente a ninguna entity ni value object** —típicamente una operación que involucra **varios aggregates**—. Ej.: un `ServicioDeTransferencia` que coordina mover dinero entre dos cuentas (no es responsabilidad de una cuenta sola). **Ojo con la regla 4 del módulo 8**: si cada `Cuenta` es su propio aggregate, el domain service **no** debita y acredita las dos en una sola transacción —eso sería consistencia inmediata entre aggregates—; coordina la lógica, pero la transferencia real se resuelve con **consistencia eventual** (débito → evento → crédito), o es una **saga** (ver abajo). El domain service encapsula el *cálculo/decisión* de dominio, no fuerza una transacción cruzada. **Cuidado** también con el abuso: no metas en services lógica que en realidad va en un aggregate (el síntoma del *modelo anémico*: aggregates sin comportamiento y "services" con toda la lógica).

**Factory (fábrica).** Encapsula la **creación compleja** de un aggregate o value object cuando construirlo bien requiere lógica (varios pasos, invariantes a chequear al crear). Si crear un `Pedido` válido tiene reglas, una factory las concentra en vez de desparramarlas.

**Saga / Process Manager.** Cuando un flujo del negocio **cruza varios aggregates o contextos** y "una transacción = un aggregate" (módulo 8) te lo prohíbe en una sola transacción, el coordinador de ese flujo es una **saga** (o *process manager*): una secuencia de pasos disparados por eventos, **con acciones de compensación** si un paso falla. El caso de libro: confirmar pedido → cobrar → reservar stock → despachar; si el **cobro falla** después de reservar stock, la saga ejecuta la **compensación** (liberar el stock, cancelar el pedido). Es lo que cierra el flujo que el módulo 8 dejó en "un handler descuenta el stock después": ese handler, encadenado con compensaciones, **es** la saga. Hay dos estilos (lo profundiza [event-driven](event-driven.md)): **coreografía** (cada aggregate reacciona a eventos, sin un coordinador central —simple, pero el flujo queda implícito—) y **orquestación** (un process manager explícito dirige los pasos —más visible y testeable cuando el flujo es complejo—). En EventStorming, la saga es justamente la **policy** (post-it lila, módulo 6): "cuando pasó X, hacé Y". La regla: no hay transacciones distribuidas ACID entre aggregates; hay sagas con consistencia eventual y compensación.

El antipatrón que enmarca todo esto: el **modelo anémico** (*anemic domain model*). Es cuando tus "entities" son **bolsas de datos con getters/setters y cero comportamiento**, y toda la lógica vive en "services" gordos. Parece DDD (hay clases con nombres del dominio) pero **no lo es**: es procedural disfrazado. El DDD táctico de verdad pone el **comportamiento donde están los datos** —el `Pedido` sabe confirmarse, la `Cuenta` sabe debitar—; los domain services son la excepción para lo que genuinamente cruza aggregates, no la regla. La frase mental: **domain events comunican lo que pasó entre aggregates/contextos (consistencia eventual); repositories persisten aggregates como una colección (uno por aggregate root, interfaz en el dominio, impl en infra); domain services son para lógica que cruza varios aggregates; factories encapsulan creación compleja —y el enemigo a evitar es el modelo anémico, entities sin comportamiento con toda la lógica en services gordos—.**

**Ejercicios 9**
9.1 ¿Qué es un domain event y para qué se usa? ¿Por qué se expresa en pasado?
9.2 ¿Por qué hay un repositorio por aggregate root y no por tabla? ¿Dónde vive la interfaz y dónde la implementación, y con qué patrón conecta?
9.3 ¿Cuándo usás un domain service y cuál es el riesgo de abusar de ellos?
9.4 ¿Qué es el "modelo anémico", por qué es un antipatrón, y cómo se ve un modelo rico en cambio?
9.5 "Confirmar pedido → cobrar → reservar stock → despachar". Si el **cobro falla** después de reservar el stock, ¿qué patrón coordina el flujo y qué hace ante el fallo? Distinguí coreografía de orquestación.
9.6 Tu `Pedido` tiene constructor privado con invariantes. ¿Cómo lo reconstruís desde la base de datos sin saltear ni exponer esas invariantes, y cómo evitás que el ORM se filtre al dominio?

---

## Módulo 10 — Las capas: dominio, aplicación, infraestructura (el puente con hexagonal)

**Teoría.** Para que el modelo de dominio se mantenga **puro** (sin contaminarse con detalles de framework, base de datos o HTTP), DDD se combina con una arquitectura de **capas** —típicamente hexagonal / clean / puertos y adaptadores ([NestJS senior](nestjs-senior.md))—. Tres capas, con una regla de dependencia estricta:

- **Dominio** (el centro): entities, value objects, aggregates, domain events, domain services, y las **interfaces** de los repositorios (puertos). **No depende de nada** —ni de NestJS, ni de TypeORM, ni de Express—. Es TypeScript puro que modela el negocio. Esta pureza es lo que lo hace testeable (rápido, sin DB — [TDD](tdd.md)) y estable ante cambios de tecnología.

- **Aplicación** (orquestación): los **casos de uso** (*application services* / command handlers). Orquestan el flujo de un caso de uso: cargar el aggregate del repositorio, llamar a su método de dominio, guardarlo, publicar eventos. **No tiene lógica de negocio** (esa está en el dominio); tiene **coordinación**.

```ts
// CAPA DE APLICACIÓN: orquesta, no decide reglas de negocio
class ConfirmarPedidoHandler {
  constructor(
    private readonly pedidos: PedidoRepository,   // puerto del dominio (interfaz)
    private readonly bus: EventBus,
  ) {}

  async ejecutar(cmd: ConfirmarPedido): Promise<void> {
    const pedido = await this.pedidos.obtenerPorId(cmd.pedidoId);
    if (!pedido) throw new PedidoNoEncontrado(cmd.pedidoId);
    pedido.confirmar();                  // ← la REGLA vive en el dominio (el aggregate)
    await this.pedidos.guardar(pedido);  // una transacción = un aggregate (módulo 8)
    await this.bus.publicar(pedido.eventosPendientes()); // eventos → consistencia eventual
  }
}
```

- **Infraestructura** (los detalles): las **implementaciones** de los puertos —el `PedidoRepositoryPostgres`, los controllers HTTP, los clientes de APIs externas, el `EventBus` real—. Es lo intercambiable.

La **regla de dependencia** (clean/hexagonal): las dependencias apuntan **hacia adentro**. La infraestructura depende del dominio, **nunca al revés** —el dominio define la interfaz `PedidoRepository`, y la infraestructura la implementa (inversión de dependencias)—. Por eso podés cambiar Postgres por Mongo, o NestJS por Fastify, **sin tocar el dominio**.

Por qué esto importa para DDD específicamente: el modelo de dominio es lo más valioso y lo que más debe durar; **aislarlo de los detalles que cambian** (frameworks, bases de datos, que van y vienen) es lo que lo mantiene limpio y centrado en el negocio. Y la conexión con [TDD](tdd.md): un dominio puro, sin dependencias de infraestructura, es **rapidísimo de testear** (unit tests sin DB), que es justo donde vive la lógica jugosa. La frase mental: **DDD se apoya en capas (dominio puro en el centro: entities/aggregates/puertos; aplicación que orquesta casos de uso sin lógica de negocio; infraestructura con las implementaciones intercambiables) y la regla de dependencia apunta hacia adentro —la infra depende del dominio, nunca al revés—, lo que mantiene el modelo limpio, estable ante cambios de tecnología y trivial de testear—.**

**Ejercicios 10**
10.1 Nombrá las tres capas y qué vive en cada una. ¿Cuál no depende de nada y por qué importa?
10.2 ¿Qué hace la capa de aplicación y qué NO hace? Dá el ejemplo del `ConfirmarPedidoHandler`.
10.3 Enunciá la regla de dependencia. ¿Cómo permite cambiar Postgres por Mongo sin tocar el dominio?
10.4 ¿Por qué un dominio puro es especialmente fácil de testear, y con qué módulo del temario conecta?
10.5 **Implementá.** Escribí el handler de aplicación `CancelarPedido` (capa de aplicación): carga el `Pedido` del repositorio, llama a la regla de dominio (`pedido.cancelar()`), guarda y publica los eventos —sin meter lógica de negocio en el handler—.

---

## Módulo 11 — DDD y la arquitectura moderna: modular monolith, vertical slices, CQRS/ES

**Teoría.** DDD **no impone una arquitectura de deployment** —se combina con varias, y la elección moderna importa—. El cambio de criterio más relevante de los últimos años:

**Modular monolith antes que microservicios.** El reflejo de hace una década era "DDD + microservicios" (un servicio por bounded context). El consenso moderno (y el de [NestJS senior](nestjs-senior.md)) es: **empezá con un monolito modular** —un solo deployable, pero internamente dividido en **módulos que respetan los bounded contexts**, con límites claros y comunicación por interfaces/eventos—. Razones: cuando arrancás **todavía no entendés bien el dominio**, y los límites entre servicios son lo **más caro de cambiar** (mover código entre dos repos con dos DBs y dos deploys es brutal); si trazás esos límites mal —casi seguro, al principio— quedás con un **"monolito distribuido"**, lo peor de los dos mundos. El monolito modular te da los **beneficios del modelado DDD** (límites claros, contextos separados) **sin el costo operacional** de los microservicios, y si un módulo necesita escalar o desplegarse aparte, ya está aislado y lo extraés. Los bounded contexts (módulo 4) son la **guía de esos módulos**, y mañana de esos servicios.

**Vertical slice architecture.** Una organización moderna y popular: en vez de capas horizontales globales (`controllers/`, `services/`, `repositories/` para todo el sistema), organizás por **feature/caso de uso vertical** —cada slice tiene su request, su lógica de dominio y su persistencia, juntos—. Encaja con DDD porque un caso de uso suele vivir dentro de un bounded context y toca un aggregate; mejora la **cohesión** (un cambio de negocio vive en una carpeta, no desparramado por capas). Es el "organizá por dominio, no por capa técnica" de [NestJS senior](nestjs-senior.md) llevado a su forma explícita.

**CQRS y Event Sourcing** (cuando el core lo justifica, ver [event-driven](event-driven.md)):
- **CQRS** separa el **modelo de escritura** (el aggregate rico, con invariantes) del **modelo de lectura** (read models desnormalizados, optimizados para consultar). DDD y CQRS se potencian: el lado de escritura es donde vive el modelo táctico rico; el de lectura no necesita aggregates. **No es para todo el sistema** —solo para los subdominios donde lecturas y escrituras divergen mucho—.
- **Event Sourcing**: en vez de guardar el **estado actual** del aggregate, guardás la **secuencia de eventos** que lo llevaron a ese estado, y reconstruís el estado replicándolos. Da auditoría total e historia, pero agrega complejidad real. Encaja con DDD (los domain events ya son ciudadanos de primera) y con Event Modeling (módulo 6), pero es una decisión de peso —reservada para cores donde la historia/auditoría es un requisito del negocio—.

El criterio integrador: **DDD te da el modelo y los límites; la arquitectura (monolito modular, vertical slices, CQRS/ES) es cómo lo materializás —y la regla moderna es empezar simple (monolito modular) y subir complejidad solo cuando el core lo exige—.** La frase mental: **DDD se combina con arquitecturas, y la elección moderna es modular monolith antes que microservicios (los límites de contexto guían los módulos, sin el costo del "monolito distribuido"), vertical slices para cohesión por feature, y CQRS/Event Sourcing solo en los cores que lo justifican —empezá simple y extraé/complejizá cuando el dominio lo pida—.**

**Ejercicios 11**
11.1 ¿Por qué el consenso moderno es "modular monolith antes que microservicios"? ¿Qué es el "monolito distribuido" y por qué es lo peor?
11.2 ¿Cómo guían los bounded contexts la arquitectura de un monolito modular (y de futuros servicios)?
11.3 ¿Qué es la vertical slice architecture y por qué encaja con DDD? (pensá en cohesión)
11.4 ¿Cómo se potencian DDD y CQRS, y por qué CQRS/ES no es para todo el sistema?

---

## Módulo 12 — DDD y los equipos: Conway y Team Topologies (el giro sociotécnico)

**Teoría.** El **giro sociotécnico** es quizás la evolución más importante del DDD moderno: entender que **los límites del software y los límites de los equipos están entrelazados**, y que diseñar uno sin el otro fracasa. La base teórica:

- **La Ley de Conway** (1967): "las organizaciones diseñan sistemas que copian la estructura de comunicación de la organización". Es decir: **tu arquitectura va a terminar pareciéndose a tu organigrama**, lo quieras o no. Si tenés un equipo de "frontend", uno de "backend" y uno de "DBA", vas a producir un sistema partido en esas tres capas —aunque el negocio se divida en otros bounded contexts—.

- **El Inverse Conway Maneuver**: como la organización moldea la arquitectura, **organizá los equipos a propósito para obtener la arquitectura que querés**. Si querés bounded contexts autónomos (Ventas, Facturación, Envíos), armá **equipos alineados a esos contextos** —no equipos por capa técnica—. El bounded context (módulo 4) deja de ser solo una frontera de software y se vuelve también una **frontera de equipo**.

**Team Topologies** (Matthew Skelton & Manuel Pais, 2019) sistematiza esto con **cuatro tipos de equipo** —*stream-aligned* (≈ un bounded context, dueño de punta a punta), *platform* (sirve un subdominio genérico como plataforma interna), *enabling* (sube de nivel a otros, temporal) y *complicated-subsystem* (expertise profunda)— y tres modos de interacción. El **detalle de los cuatro tipos y los tres modos está en el [módulo de Liderazgo técnico](liderazgo.md)** (una sola fuente de verdad, para no duplicar); acá lo que importa es el **ángulo DDD**: el tipo que mapea a un bounded context es el **stream-aligned**, y el subdominio genérico (módulo 3) suele servirse como **platform**.

El concepto central que lo une con DDD es la **carga cognitiva (cognitive load)**: un equipo solo puede entender y mantener bien una cantidad limitada de complejidad. **Un bounded context bien dimensionado es, además, una unidad de carga cognitiva manejable para un equipo.** Si un contexto es demasiado grande, sobrecarga al equipo; si los límites están mal trazados, los equipos se pisan constantemente. Por eso el dimensionamiento de bounded contexts y la estructura de equipos **se diseñan juntos**.

Por qué esto es "lo moderno": DDD dejó de ser solo una práctica técnica para volverse **sociotécnica** —el modelo, la arquitectura y la organización son tres vistas del mismo diseño—. Ignorar la dimensión de equipos hace que los bounded contexts más prolijos del mundo se erosionen cuando la estructura organizacional empuja en otra dirección (Conway gana siempre). La frase mental: **la Ley de Conway dice que tu arquitectura copia tu organigrama, así que el Inverse Conway Maneuver organiza los equipos a propósito —un equipo stream-aligned por bounded context— y Team Topologies (stream-aligned, platform, enabling, complicated-subsystem) sistematiza eso alrededor de la carga cognitiva; el bounded context es, a la vez, frontera de software y de equipo —DDD moderno es sociotécnico—.**

**Ejercicios 12**
12.1 Enunciá la Ley de Conway y qué implica para tu arquitectura si tenés equipos por capa técnica (front/back/DBA).
12.2 ¿Qué es el Inverse Conway Maneuver y cómo se relaciona con los bounded contexts?
12.3 Nombrá los cuatro tipos de equipo de Team Topologies; ¿cuál suele mapear a un bounded context y cuál a un subdominio genérico?
12.4 ¿Qué es la "carga cognitiva" y por qué conecta el dimensionamiento de un bounded context con la estructura del equipo?

---

## Módulo 13 — DDD en la era de la IA y el criterio de cierre

**Teoría.** Cierre, con el ángulo más actual y el criterio que gobierna todo.

**DDD en la era de la IA.** Con LLMs que generan código y sistemas de IA en producción, DDD se vuelve **más** relevante, no menos —y por razones concretas que conectan con el [track de IA](rag.md)—:

- **El lenguaje ubicuo es la base de un buen contexto para el LLM.** Un modelo de dominio con nombres claros y reglas explícitas (módulos 2, 7-9) es el mejor material para que un LLM **genere código correcto** y para escribir **prompts precisos** ([Prompt Engineering](prompt-engineering.md)): si tu código habla el lenguaje del negocio, el LLM "entiende" el dominio y produce código alineado; si es un CRUD anémico con `updateStatus(2)`, el LLM no tiene de dónde inferir las reglas.
- **Los bounded contexts mapean cómo scopear sistemas de IA.** Un [RAG](rag.md) sobre el contexto de *Soporte* (tickets, SLAs) es distinto a uno sobre *Ventas*; un [agente](agentes.md) bien acotado opera dentro de un bounded context con sus reglas y sus tools. La frontera del contexto es también la frontera del conocimiento y de los permisos del sistema de IA (el multi-tenant de [Vector Databases](vector-dbs.md)).
- **El riesgo nuevo**: los LLMs, librados a su suerte, tienden a generar **código CRUD anémico** (módulo 9) —lo más común en su entrenamiento—. Sin criterio de DDD, "dejar que la IA escriba el dominio" erosiona el modelo hacia bolsas de datos sin comportamiento. La IA acelera; el **diseño del modelo sigue siendo trabajo humano** (idealmente colaborativo, módulo 6, donde la IA puede ayudar a facilitar/transcribir, pero no a decidir el modelo).

Esto es **emergente**, no dogma asentado: la práctica de combinar DDD con desarrollo asistido por IA está madurando. Pero la dirección es clara: **un buen modelo de dominio es leverage para la IA, y la IA no reemplaza el modelado.**

**El criterio de cierre — cuándo DDD paga y cuándo es over-engineering.** Es lo más importante del módulo, y la heurística viene del módulo 3 (subdominios) y de Khononov: **igualá el esfuerzo de modelado a la complejidad y al tipo de subdominio.**

- **DDD táctico rico (aggregates, value objects, todo el módulo 7-9)**: solo en el **core domain complejo** —donde la lógica de negocio es intrincada, cambia seguido, y es tu ventaja—. Ahí el modelado paga con creces.
- **DDD estratégico (lenguaje ubicuo, bounded contexts, context mapping)**: vale **casi siempre que haya más de un subdominio**, incluso en sistemas medianos —es barato y previene el modelo gigante—. El lenguaje ubicuo, en particular, **siempre** rinde.
- **Un CRUD simple, un subdominio genérico o de soporte trivial**: **no** necesita aggregates ni el aparato táctico completo. Forzar DDD táctico ahí es **over-engineering** —complejidad y ceremonia sin retorno—. Un CRUD honesto es mejor que un "DDD" de juguete.

El "DDD de juguete" (over-engineering) al lado de lo calibrado:

```ts
// ❌ Over-engineering: ceremonia sin retorno en un concepto trivial
class NombreDeEtiqueta {                 // un value object para... el nombre de una etiqueta
  private constructor(readonly valor: string) {}
  static crear(v: string): NombreDeEtiqueta { return new NombreDeEtiqueta(v); } // no valida nada real
}
class EtiquetaService { /* "domain service" para un CRUD de etiquetas sin reglas */ }

// ✅ Calibrado: lo trivial es un tipo simple; el esfuerzo de modelado va al core
type NombreDeEtiqueta = string;          // una etiqueta no tiene reglas → no necesita VO
// y los aggregates/VOs con invariantes se reservan para el Pedido del core, donde sí pagan
```

El error en los dos extremos: el junior que hace un CRUD anémico del core complejo (subdiseña lo importante), y el que mete aggregates y value objects en cada tabla de un sistema sin complejidad real (sobre-diseña lo trivial). **La marca del senior es calibrar.** Y el principio que recorre todo el temario: **la mejor solución es la más simple que resuelve el problema real** —en DDD: el modelado más rico donde la complejidad del dominio lo justifica, y nada de ceremonia donde no—. La frase mental: **DDD en la era de la IA importa más (el lenguaje ubicuo es el mejor contexto para el LLM, los bounded contexts scopean RAG/agentes, y el riesgo es que la IA erosione el modelo hacia CRUD anémico); y el criterio que cierra todo es calibrar el esfuerzo: táctico rico solo en el core complejo, estratégico casi siempre que haya varios subdominios, y nada de ceremonia en un CRUD trivial —subdiseñar el core o sobre-diseñar lo trivial son los dos errores que un senior evita—.**

**Ejercicios 13**
13.1 ¿Por qué el lenguaje ubicuo y los bounded contexts se vuelven *más* relevantes con LLMs y sistemas de IA? Dá un ejemplo de cada uno.
13.2 ¿Cuál es el riesgo nuevo de "dejar que la IA escriba el dominio", y qué sigue siendo trabajo humano?
13.3 ¿Para qué tipo de subdominio se justifica el DDD táctico rico, y para cuál es over-engineering?
13.4 ¿Cuáles son los dos errores opuestos (subdiseñar / sobre-diseñar) y cómo enuncia el criterio de cierre la regla para calibrar?

---

## Proyecto integrador (capstone): modelar un bounded context de verdad

El ejercicio que cierra el módulo —y que mostrás en una entrevista de arquitectura cuando te piden "¿sabés DDD?"—. No te damos la solución; te damos los **criterios de aceptación**. Tomá un dominio con complejidad real (el e-commerce del módulo, un sistema de reservas, o el tuyo) y modelá **un** bounded context bien.

**Qué construir.** El modelo de un bounded context (ej. *Ventas* o *Reservas*), en TypeScript `--strict`:

1. **Lenguaje ubicuo** (módulo 2): un mini-glosario de los términos del contexto, y código que los usa (métodos como `pedido.confirmar()`, no `updateStatus(2)`).
2. **Value objects** (módulo 7) para los conceptos con reglas (Dinero, Email, un identificador tipado), con validación en el constructor —imposible crear uno inválido—.
3. **Un aggregate** (módulo 8) con su raíz, sus invariantes protegidas en métodos, referencias a otros aggregates **por ID**, y que emita al menos un **domain event** (módulo 9).
4. **El puerto del repositorio** en el dominio y un caso de uso en la **capa de aplicación** (módulo 10) que cargue, ejecute la regla y guarde —respetando "una transacción = un aggregate"—.

**Criterios de aceptación.**
- [ ] El código habla el **lenguaje del negocio**: los nombres son los del dominio, no técnicos genéricos.
- [ ] Es **imposible** crear un value object inválido (la validación está en el constructor/factory).
- [ ] El aggregate **protege sus invariantes** en métodos; no hay setters públicos que las salteen.
- [ ] El aggregate referencia a otros **por ID**, no por objeto, y emite un domain event en una operación clave.
- [ ] El **dominio no importa** nada de infraestructura (ni framework, ni ORM): es TypeScript puro, testeable sin DB.
- [ ] Podés **explicar a qué tipo de subdominio** (core/supporting/generic) pertenece y por qué ese nivel de esfuerzo.

**Extensiones (suben el nivel).**
- **EventStorming en papel** (módulo 6): mapeá el flujo del contexto con eventos/comandos/aggregates/hotspots antes de codear, y mostrá cómo el modelo salió de ahí.
- **Context map** (módulo 5): dibujá cómo tu contexto se relaciona con otro (ej. un ACL hacia un sistema externo de pagos).
- **Tests del dominio** ([TDD](tdd.md)): testeá las invariantes del aggregate sin DB, rápido (es donde el dominio puro brilla).
- **El ángulo IA** (módulo 13): escribí un prompt que use tu lenguaje ubicuo para que un LLM extienda el modelo, y notá cómo un modelo rico produce mejor código que un CRUD anémico.
- **Migrá un CRUD legacy** (módulos 5, 9, 13): tomá un módulo anémico existente y sacá un aggregate rico del barro —poné el comportamiento en la entity, meté un ACL hacia lo que no controlás, reconstituí el aggregate desde la DB sin filtrar el ORM— justificando **qué NO tocás** (lo trivial que no lo amerita). Es el escenario de entrevista más frecuente.

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Creer que DDD es una arquitectura o un catálogo de patrones que copiás. Tener una carpeta domain/
    no es DDD porque DDD es el trabajo de MODELAR el negocio (lenguaje ubicuo, contextos, modelo rico);
    las carpetas sin ese trabajo quedan como estructura vacía con código CRUD anémico adentro.
1.2 "Del negocio hacia adentro" = partir de los conceptos y reglas del negocio (¿qué es un Pedido, qué
    puede pasarle?) y derivar el modelo y la persistencia de ahí. "De la tabla hacia afuera" = partir del
    esquema de la DB (tengo una tabla orders con estas columnas) y construir todo alrededor de eso. Ej.:
    modelar Pedido.confirmar() con sus reglas, no una tabla orders con un campo status que actualizás.
1.3 Porque el modelo se construye destilando el conocimiento del experto del dominio; el dev solo no
    "sabe" el negocio. El modelo y el código evolucionan con el entendimiento, que se gana hablando con
    quien entiende el dominio. Sin esa colaboración, modelás suposiciones.
1.4 Que distingue CUÁNDO el dominio es lo bastante complejo como para justificar el esfuerzo de DDD y
    cuándo no, en vez de aplicar los patrones tácticos por dogma a todo (módulo 13).
```

### Módulo 2
```
2.1 Un lenguaje común, preciso y compartido entre el negocio y el código, usado en todos lados
    (conversaciones, docs, nombres de clases/métodos/variables). "Ubicuo" porque está en todas partes,
    sin traducción entre cómo habla el negocio y cómo nombra el código.
2.2 La brecha de traducción entre el negocio y el código. Negocio: "cancelar una reserva"; código:
    updateStatus(2). En cada traducción mental se pierde/distorsiona información y nacen bugs. El
    lenguaje ubicuo elimina la traducción: el código dice reserva.cancelar().
2.3 De cómo habla la gente del dominio, refinado hasta quedar sin ambigüedad. Si encontrás un concepto
    que el negocio nombra pero tu modelo no tiene, te falta un término (y probablemente una clase/método)
    en el modelo.
2.4 Porque un bounded context es, literalmente, la frontera donde un lenguaje ubicuo es consistente
    (módulo 4); y el modelo táctico (7-9) modela los conceptos del lenguaje. Sin lenguaje ubicuo preciso,
    los contextos y el modelo modelan conceptos vagos.
```

### Módulo 3
```
3.1 Core: lo que te hace competitivo y distinto, complejo (ej. el motor de pujas de un sitio de
    subastas). Supporting: necesario pero no diferencial y simple (ej. gestión de catálogo). Generic:
    problema resuelto, igual para todos (ej. autenticación, pagos, envío de emails).
3.2 Core: construir in-house (es tu diferencial, no lo comprás). Supporting: construir simple (sin
    sobre-invertir). Generic: comprar/off-the-shelf (otros ya lo resolvieron mejor; construirlo es
    desperdiciar esfuerzo).
3.3 Invertir esfuerzo de DDD táctico en un subdominio genérico mientras el core queda anémico. Ej.:
    modelar elaboradamente tu propio sistema de auth (genérico) y dejar el motor que te diferencia como
    un CRUD sin reglas.
3.4 Porque define dónde poner el esfuerzo: el modelado rico solo donde te diferenciás (core), comprar lo
    genérico, mantener simple lo de soporte. Es la base del criterio de DDD pragmático moderno (Khononov)
    y del criterio de cierre (módulo 13).
```

### Módulo 4
```
4.1 Una frontera explícita dentro de la cual un modelo y su lenguaje ubicuo son consistentes y un
    término significa una sola cosa. Lo consistente es el MODELO y el LENGUAJE (cada término = un solo
    significado) dentro de esa frontera.
4.2 Ventas: historial de compras, descuentos, carrito. Facturación: datos fiscales, condición
    impositiva. Soporte: tickets, SLA, contactos. Un modelo único es un error porque termina en una
    clase gigante con campos null según quién la use, acoplada a todo y que cambia por razones de varios
    equipos.
4.3 Por identidad compartida (un clienteId) y por eventos de dominio, no por una clase Cliente
    compartida. Cada contexto tiene su propio modelo de Cliente con lo que necesita.
4.4 El subdominio es del PROBLEMA (cómo se divide el negocio); el bounded context es de la SOLUCIÓN
    (cómo dividís el software). Idealmente se alinean (un contexto por subdominio) pero no siempre es 1:1.
```

### Módulo 5
```
5.1 Una capa de traducción que protege tu modelo del modelo de otro contexto/sistema, traduciendo SU
    lenguaje al TUYO. Imprescindible al integrar con un sistema legacy o una API de terceros con un
    modelo feo que no querés que contamine tu dominio.
5.2 En el ACL, vos (downstream) te protegés del modelo del otro traduciéndolo. En el OHS, vos sos el
    proveedor (upstream) y te protegés de acoplarte a cada consumidor exponiendo un API/lenguaje
    publicado estable. ACL protege al consumidor; OHS protege al proveedor.
5.3 El downstream se conforma al modelo del upstream sin traducir (no puede negociar, ej. la API de un
    gigante). Aceptable cuando el modelo ajeno es razonable y no lo podés cambiar: es más barato que un
    ACL, a costa de acoplarte a ese modelo.
5.4 Porque dos contextos comparten una porción de modelo (código común), y cualquier cambio en ese
    kernel afecta a ambos equipos: acopla. Debe ser lo más chico posible, estable y co-gestionado por
    los dos equipos.
```

### Módulo 6
```
6.1 Que el dev modele solo a partir de un documento de requisitos (lento, propenso a malentendidos que
    aparecen tarde). Las técnicas colaborativas ponen a devs y expertos a construir el modelo juntos, en
    la misma sala, rápido y visual.
6.2 Naranja = eventos de dominio (algo que pasó, en pasado); azul = comandos (la acción/intención);
    amarillo = aggregates (lo que recibe el comando y emite el evento); rojo = hotspots (dudas,
    conflictos, problemas sin resolver). Los hotspots son oro porque marcan dónde el dominio es confuso
    o nadie se pone de acuerdo —justo donde hay que mirar antes de codear—.
6.3 Para descubrir los bounded contexts (estratégico, módulo 4) y los aggregates (táctico, módulo 8) de
    forma colaborativa, y para que afloren los malentendidos antes de escribir código.
6.4 Event Modeling es más estructurado: modela el sistema como una línea de tiempo (blueprint) de
    comandos → eventos → read models a lo largo de pantallas. Encaja muy bien con CQRS/Event Sourcing.
    EventStorming es más exploratorio/libre.
```

### Módulo 7
```
7.1 La identidad. El value object NO tiene identidad (se define por sus valores, es intercambiable); la
    entity SÍ tiene identidad que persiste en el tiempo (es "la misma" aunque cambien sus atributos).
7.2 Que un objeto en estado inválido no pueda construirse. El value object pone la validación en el
    constructor (privado) + factory: Email.crear() valida el formato y lanza si es inválido, así que si
    tenés un Email, el tipo garantiza que es válido —la validación vive en el tipo, no desparramada—.
7.3 Porque su estado cambia legítimamente a lo largo de su vida (un usuario cambia de email), pero esos
    cambios deben pasar por métodos que protegen las reglas (cambiarEmail(nuevo: Email)), no por setters
    públicos que permitirían dejar la entity en un estado inválido.
7.4 Primitive obsession = usar tipos primitivos sueltos (string, number) para conceptos del dominio que
    tienen reglas. Los value objects la combaten envolviendo esos conceptos con validación y
    comportamiento (no podés sumar USD+EUR, ni pasar un clienteId donde va un pedidoId). No conviene para
    cada string: reservalo para conceptos con reglas o significado (dinero, email, cantidades, ids).
```
```ts
// 7.5
class Dinero {
  private constructor(readonly monto: number, readonly moneda: string) {}
  static crear(monto: number, moneda: string): Dinero {
    if (monto < 0) throw new Error("El monto no puede ser negativo");
    return new Dinero(monto, moneda);
  }
  static cero(moneda: string): Dinero {              // semilla neutra para sumar (la usa Pedido.total(), módulo 8)
    return new Dinero(0, moneda);
  }
  sumar(otro: Dinero): Dinero {
    if (this.moneda !== otro.moneda) {
      throw new Error(`No se puede sumar ${this.moneda} + ${otro.moneda}`);
    }
    return new Dinero(this.monto + otro.monto, this.moneda);
  }
  equals(otro: Dinero): boolean {                    // igualdad por VALOR
    return this.monto === otro.monto && this.moneda === otro.moneda;
  }
}
```

### Módulo 8
```
8.1 Un grupo de entities y value objects tratados como una unidad de consistencia, con una raíz
    (aggregate root) que es la única puerta de entrada. Es "la única puerta" porque toda modificación
    pasa por la raíz, que garantiza las invariantes de todo el grupo (no se tocan las partes por fuera).
8.2 Porque aggregates chicos = transacciones chicas, menos contención y mejor concurrencia/performance.
    El aggregate gigante (Cliente que contiene pedidos que contienen líneas...) arrastra medio sistema en
    cada transacción, genera contención y no escala.
8.3 Para mantener los aggregates chicos y desacoplados y no cargar medio sistema en memoria. Ej.: Pedido
    tiene un clienteId (ID), no un objeto Cliente adentro (que es otro aggregate). Así tocar un pedido no
    arrastra al cliente y sus datos.
8.4 Cada transacción modifica un solo aggregate; la consistencia entre aggregates es eventual (vía
    eventos), no inmediata. Ej.: confirmar pedido (un aggregate) NO descuenta stock (otro aggregate) en
    la misma transacción: el Pedido emite PedidoConfirmado y un handler descuenta el stock después.
```
```ts
// 8.5 — la invariante nueva en agregarLinea:
agregarLinea(producto: ProductoId, cantidad: number, precio: Dinero): void {
  if (this.estado === "confirmado") throw new Error("Un pedido confirmado no acepta nuevas líneas");
  if (cantidad <= 0) throw new Error("La cantidad debe ser positiva");
  if (this.lineas.length >= 20) throw new Error("Un pedido no puede superar las 20 líneas (límite de picking)"); // ← invariante de negocio
  this.lineas.push(new LineaPedido(producto, cantidad, precio));
}

it("rechaza la línea número 21", () => {
  const pedido = Pedido.crear(new PedidoId("p1"), new ClienteId("c1"));
  for (let i = 0; i < 20; i++) pedido.agregarLinea(new ProductoId(`x${i}`), 1, Dinero.crear(10, "USD"));
  expect(() => pedido.agregarLinea(new ProductoId("x21"), 1, Dinero.crear(10, "USD"))).toThrow();
});
```

### Módulo 9
```
9.1 Algo relevante para el negocio que ocurrió (PedidoConfirmado). Se usa para comunicar cambios entre
    aggregates y entre bounded contexts (consistencia eventual). Se expresa en pasado porque representa
    un hecho que ya sucedió y es inmutable.
9.2 Porque el aggregate es la unidad de consistencia y persistencia: guardás/recuperás el aggregate
    entero por su raíz, no tabla por tabla. La interfaz (puerto) vive en el dominio (es parte del modelo);
    la implementación vive en infraestructura. Conecta con puertos/adaptadores (hexagonal) e inversión de
    dependencias.
9.3 Cuando la lógica de dominio no pertenece naturalmente a ninguna entity/value object, típicamente una
    operación que cruza varios aggregates (transferencia entre dos cuentas). El riesgo de abusar: meter
    en services lógica que SÍ va en un aggregate, vaciando las entities y cayendo en el modelo anémico.
9.4 Modelo anémico = entities que son bolsas de datos con getters/setters y cero comportamiento, con toda
    la lógica en services gordos. Es antipatrón porque parece DDD pero es procedural disfrazado. Un modelo
    rico pone el comportamiento donde están los datos (Pedido sabe confirmarse, Cuenta sabe debitar).
9.5 Lo coordina una saga (process manager): ejecuta los pasos vía eventos y, ante un fallo, corre
    acciones de COMPENSACIÓN (cobro falla después de reservar stock → liberar el stock, cancelar el
    pedido). No hay transacción ACID distribuida: hay consistencia eventual con compensación. Coreografía
    = cada aggregate reacciona a eventos sin coordinador central (simple, flujo implícito); orquestación
    = un process manager explícito dirige los pasos (más visible y testeable cuando el flujo es complejo).
9.6 Con dos vías de construcción: crear() valida invariantes (pedido nuevo) y reconstituir()
    rehidrata desde la DB SIN re-validar (esos datos ya fueron válidos), ambas sin setters públicos. El ORM
    no se filtra porque un mapper en infraestructura traduce entre las filas/DTOs y el aggregate; el dominio
    no lleva decoradores de TypeORM ni conoce la DB. La concurrencia sobre el mismo aggregate se maneja con
    optimistic locking (campo version).
```

### Módulo 10
```
10.1 Dominio (entities, value objects, aggregates, domain events, domain services, interfaces de
     repositorios), aplicación (casos de uso/handlers que orquestan) e infraestructura (implementaciones:
     repos concretos, controllers, clientes externos). El dominio no depende de nada: eso lo hace puro,
     estable ante cambios de tecnología y trivial de testear.
10.2 Orquesta el caso de uso: carga el aggregate del repositorio, llama a su método de dominio, lo guarda,
     publica eventos. NO tiene lógica de negocio (esa vive en el dominio). El ConfirmarPedidoHandler carga
     el Pedido, llama pedido.confirmar() (la regla está en el aggregate), guarda y publica eventos.
10.3 Las dependencias apuntan hacia adentro: la infraestructura depende del dominio, nunca al revés. El
     dominio define la interfaz PedidoRepository y la infra la implementa (inversión de dependencias), así
     que cambiar Postgres por Mongo es cambiar la implementación sin tocar el dominio.
10.4 Porque no tiene dependencias de infraestructura (DB, framework): se testea con unit tests rápidos sin
     levantar nada, y ahí vive la lógica jugosa (las invariantes). Conecta con TDD (el dominio rico que el
     test-first empuja, y la base de la pirámide).
```
```ts
// 10.5
class CancelarPedidoHandler {
  constructor(
    private readonly pedidos: PedidoRepository,
    private readonly bus: EventBus,
  ) {}

  async ejecutar(cmd: CancelarPedido): Promise<void> {
    const pedido = await this.pedidos.obtenerPorId(cmd.pedidoId);
    if (!pedido) throw new PedidoNoEncontrado(cmd.pedidoId);
    pedido.cancelar();                                    // la REGLA vive en el dominio (aggregate)
    await this.pedidos.guardar(pedido);                   // una transacción = un aggregate
    await this.bus.publicar(pedido.eventosPendientes());  // eventos → consistencia eventual
  }
}
```

### Módulo 11
```
11.1 Porque al arrancar todavía no entendés bien el dominio y los límites entre servicios son lo más caro
     de cambiar; si los trazás mal (probable al principio) quedás con un "monolito distribuido": servicios
     que tienen que llamarse entre sí para todo, con el costo operacional de microservicios y el acoplamiento
     de un monolito —lo peor de los dos mundos—.
11.2 Los bounded contexts definen los MÓDULOS internos del monolito (límites claros, comunicación por
     interfaces/eventos); y si mañana un módulo necesita escalar/desplegarse aparte, ya está aislado y se
     extrae como servicio. El contexto es la costura por donde se separa.
11.3 Organizar por feature/caso de uso vertical (request + dominio + persistencia juntos) en vez de capas
     horizontales globales. Encaja con DDD porque un caso de uso vive en un bounded context y toca un
     aggregate; mejora la cohesión (un cambio de negocio vive en una carpeta, no desparramado por capas).
11.4 CQRS separa el modelo de escritura (el aggregate rico con invariantes, donde vive el DDD táctico) del
     de lectura (read models desnormalizados, que no necesitan aggregates): se potencian. No es para todo el
     sistema, solo para subdominios donde lecturas y escrituras divergen mucho —en el resto agrega complejidad
     sin retorno—.
```

### Módulo 12
```
12.1 "Las organizaciones diseñan sistemas que copian su estructura de comunicación": tu arquitectura
     termina pareciéndose a tu organigrama. Con equipos por capa técnica (front/back/DBA) vas a producir un
     sistema partido en esas capas, aunque el negocio se divida en otros bounded contexts.
12.2 Como la organización moldea la arquitectura, organizás los equipos a propósito para OBTENER la
     arquitectura que querés: si querés bounded contexts autónomos, armás un equipo por contexto (no por capa
     técnica). El bounded context se vuelve también frontera de equipo.
12.3 Stream-aligned, platform, enabling, complicated-subsystem. El stream-aligned suele mapear a un bounded
     context (dueño de punta a punta); el platform team suele servir un subdominio genérico como plataforma
     interna.
12.4 La carga cognitiva es cuánta complejidad un equipo puede entender y mantener bien. Conecta porque un
     bounded context bien dimensionado es una unidad de carga manejable para un equipo: si es muy grande,
     sobrecarga; si los límites están mal, los equipos se pisan. Por eso se diseñan juntos el tamaño del
     contexto y la estructura del equipo.
```

### Módulo 13
```
13.1 Porque un modelo con lenguaje claro y reglas explícitas es el mejor contexto para que un LLM genere
     código correcto y para escribir prompts precisos (lenguaje ubicuo); y los bounded contexts definen cómo
     scopear un RAG/agente (un RAG sobre Soporte ≠ uno sobre Ventas; un agente opera dentro de un contexto con
     sus reglas, tools y permisos).
13.2 El riesgo: los LLMs tienden a generar código CRUD anémico (lo más común en su entrenamiento), erosionando
     el modelo hacia bolsas de datos sin comportamiento. Sigue siendo trabajo humano DISEÑAR el modelo
     (idealmente colaborativo); la IA acelera y ayuda, pero no decide el modelo.
13.3 Táctico rico (aggregates, value objects) solo en el core domain complejo (lógica intrincada, cambiante,
     que te diferencia). En un CRUD/subdominio genérico o de soporte trivial es over-engineering: ceremonia sin
     retorno. (El estratégico —lenguaje ubicuo, contextos— vale casi siempre que haya varios subdominios.)
13.4 Subdiseñar: hacer un CRUD anémico del core complejo (no modelás lo importante). Sobre-diseñar: meter
     aggregates/value objects en cada tabla de un sistema sin complejidad real. El criterio: igualá el esfuerzo
     de modelado a la complejidad y al tipo de subdominio —rico en el core, estratégico casi siempre, nada de
     ceremonia en lo trivial—. La marca del senior es calibrar; la regla es la solución más simple que resuelve
     el problema real.
```

---

## Siguientes pasos

Con este módulo tenés DDD a profundidad y con criterio: qué es (modelar el negocio, no las tablas) y qué no, el lenguaje ubicuo, la heurística de subdominios (core/supporting/generic) que decide dónde invertir, los bounded contexts y el context mapping (ACL, OHS, conformist…), el modelado colaborativo moderno (EventStorming, Event Modeling, Domain Storytelling), el táctico completo (value objects y entities type-driven, aggregates con las reglas de Vernon, domain events/repositories/services/factories y el antipatrón anémico), las capas y la regla de dependencia, la combinación con la arquitectura moderna (modular monolith, vertical slices, CQRS/ES), el giro sociotécnico (Conway y Team Topologies), y el ángulo de la era de la IA, todo cerrado con el criterio de **calibrar el esfuerzo al dominio**. Conecta con lo que ya sabés: profundiza la sección 2 de [NestJS senior](nestjs-senior.md), se apoya en los eventos y CQRS de [Event-driven](event-driven.md), en el dominio rico que empuja [TDD](tdd.md), y —en la era de la IA— en cómo un buen modelo es leverage para [RAG](rag.md), [agentes](agentes.md) y [prompting](prompt-engineering.md). Las fuentes modernas para ir más hondo: Vlad Khononov (*Learning Domain-Driven Design*), Vaughn Vernon (*Implementing DDD* y *Effective Aggregate Design*), Scott Wlaschin (*Domain Modeling Made Functional*), Alberto Brandolini (EventStorming) y Skelton & Pais (*Team Topologies*). La regla que se lleva el módulo entero: **el mejor modelo es el más simple que captura la complejidad real del negocio —ni un CRUD anémico del core, ni aggregates en cada tabla trivial—.**
