# Liderazgo técnico: decidir, medir y organizar un equipo que entrega

**De senior a Tech Lead · ADRs, métricas DORA, Team Topologies y el rol de la IA · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **módulo que cierra el track Tech Lead** del temario. Asume que recorriste lo técnico —[arquitectura event-driven](event-driven.md), [NestJS senior](nestjs-senior.md), [testing](testing.md) y [TDD](tdd.md), [observabilidad](observabilidad.md), [Kubernetes](kubernetes.md), [AWS](aws.md)— y ahora sube un nivel: **cómo se lidera técnicamente un equipo que construye todo eso**. El cambio de mentalidad que tenés que hacer desde la primera línea: como senior, tu output era el código que escribías; como Tech Lead, **tu output es lo que el equipo entrega, y vos lo multiplicás o lo frenás**. El objetivo es doble: las **herramientas concretas del rol** (ADRs para decidir, DORA para medir, Team Topologies para organizar) y —tan importante— **el criterio**: liderar no es mandar ni es codear más rápido que nadie, es crear las condiciones para que el equipo entregue bien y sostenido.

**Lo que asumimos.** El track técnico completo y, sobre todo, la madurez de criterio que se viene repitiendo en cada módulo de cierre ("escalá la herramienta al problema"). Este módulo es más de **decisiones, comunicación y organización** que de código —es la otra mitad del perfil senior/lead—.

> Nota: esto es metodología y práctica de equipo, no API que cambia de versión. Las referencias (Nygard para ADRs, el reporte DORA/*Accelerate* de Forsgren-Humble-Kim, *Team Topologies* de Skelton-Pais) son para profundizar; lo volátil son los benchmarks numéricos de DORA, que se recalibran cada año.

**Índice de módulos**
1. Qué es un Tech Lead (y qué no es)
2. Decisiones que perduran: los ADRs
3. Cómo se escribe un ADR
4. Medir la entrega: las cuatro métricas DORA
5. DORA en profundidad: velocidad y estabilidad no se contraponen
6. Organizar equipos: Team Topologies
7. Modos de interacción y carga cognitiva
8. El trabajo que no es código: review, mentoring, decir que no
9. Deuda técnica: gestionarla con datos, no con miedo
10. La IA en el rol del Tech Lead
11. El criterio: liderar es multiplicar, no controlar

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es un Tech Lead (y qué no es)

**Teoría.** El malentendido más caro al llegar al rol: creer que un Tech Lead es **el mejor programador del equipo que además reparte tareas**. No. Un Tech Lead es **quien es responsable de que el equipo entregue software de calidad de forma sostenida** —y eso casi nunca se logra escribiendo vos más código—.

El cambio de unidad de medida, que tenés que internalizar:

- Como **IC** (individual contributor / senior): tu impacto = lo que vos producís. Más horas tuyas, más output.
- Como **Tech Lead**: tu impacto = lo que **el equipo entero** produce. Si pasás el día con la cabeza en tu propio ticket, el equipo está sin desbloquear, sin dirección técnica y sin revisar. Tu código rinde menos que las cinco personas que dejaste esperando.

Esto se llama el paso de **"trabajo de hacedor" a "trabajo de multiplicador"**: tu tiempo más valioso es el que sube el output de otros —desbloquear, decidir una dirección técnica, revisar bien, mentorear, sacar impedimentos—. No significa que dejás de codear (un TL que no toca código pierde el pulso técnico y la credibilidad), significa que **codeás lo que solo vos podés/debés codear** (los problemas espinosos, las pruebas de concepto que destraban una decisión) y delegás el resto, incluso si lo harías "mejor".

Las responsabilidades núcleo del rol, que los módulos siguientes desarrollan:

- **Dirección técnica y decisiones**: elegir arquitecturas, dejar el porqué registrado (ADRs, módulos 2-3).
- **Salud de la entrega**: que el equipo entregue rápido *y* estable, medido (DORA, módulos 4-5).
- **Organización del trabajo y del equipo**: cómo se reparte, qué carga cognitiva soporta cada uno (Team Topologies, módulos 6-7).
- **Desarrollar a la gente**: review, mentoring, feedback (módulo 8).
- **De cara afuera**: traducir entre negocio y técnica, defender el tiempo del equipo, decir que no con fundamento.

Una distinción de vocabulario útil: **Tech Lead ≠ Engineering Manager**. El EM es responsable de las *personas* (carrera, performance, contratación, 1:1s); el TL es responsable de lo *técnico* (arquitectura, calidad, dirección). En equipos chicos a veces es la misma persona, pero son dos trabajos distintos; confundirlos —querer hacer gestión de personas *y* arquitectura *y* seguir codeando a tiempo completo— es la receta del burnout. Por eso este módulo deja **a propósito** fuera los temas puros de *people management* (carrera, 1:1s, contratación, feedback de performance): son del EM. Acá nos centramos en el liderazgo **técnico**. La frase mental: **dejaste de ser el que resuelve todos los problemas; ahora sos el que se asegura de que los problemas se resuelvan bien, por quien corresponda.**

**Ejercicios 1**
1.1 ¿Cuál es el cambio de "unidad de medida" del impacto al pasar de senior a Tech Lead?
1.2 ¿Significa eso que un Tech Lead deja de programar? Justificá y explicá *qué* tipo de código debería seguir tocando.
1.3 ¿Qué es el paso de "trabajo de hacedor" a "trabajo de multiplicador"? Dá dos ejemplos de trabajo de multiplicador.
1.4 Diferenciá Tech Lead de Engineering Manager. ¿Por qué confundirlos lleva al burnout?

---

## Módulo 2 — Decisiones que perduran: los ADRs

**Teoría.** Un equipo toma decisiones técnicas todo el tiempo: "usamos Postgres y no Mongo", "los eventos van por SNS+SQS y no Kafka", "autenticamos con JWT de acceso corto + refresh". Seis meses después llega alguien nuevo (o vos mismo, que te olvidaste) y pregunta **"¿por qué hicimos esto así?"**. Sin un registro, la respuesta es arqueología: revisar Slack viejo, preguntar al que ya se fue, o —lo peor— *revertir la decisión sin saber qué problema resolvía* y volver a chocar con el mismo muro. Ese es el problema que resuelven los **ADRs**.

Un **ADR (Architecture Decision Record)** es un documento corto que captura **una** decisión arquitectónica significativa: el contexto en que se tomó, la decisión, y sus consecuencias. La idea (Michael Nygard, 2011) es simple y poderosa: **las decisiones importantes se documentan en el momento, en archivos versionados junto al código** (típicamente `docs/adr/0001-titulo.md` en el repo), no en una wiki que nadie actualiza.

Lo que hace valioso a un ADR no es la decisión en sí —esa la ves en el código— sino **el contexto y el porqué**, que el código *no* puede contarte:

- El código te dice *que* usás Postgres. El ADR te dice *por qué* lo elegiste sobre Mongo **en ese momento, con la información que tenías** (ej: "el modelo es relacional, el equipo ya sabe SQL, y JSONB cubre los casos documentales que anticipamos").
- Eso te permite, en el futuro, **reevaluar con criterio**: si las premisas cambiaron (ahora sí necesitás escala horizontal masiva), sabés exactamente qué premisa se rompió y por qué tendría sentido cambiar. Sin el ADR, no sabés si la decisión original fue meditada o un accidente histórico.

La propiedad clave, y la que más se malentiende: **un ADR es inmutable.** No lo editás cuando la decisión cambia. Si más adelante reemplazás Postgres por otra cosa, escribís un **ADR nuevo** que dice "esta decisión *supersede* al ADR-0007" y dejás el viejo con estado `Superseded`. ¿Por qué? Porque el valor histórico es justamente ver *cómo evolucionó el pensamiento del equipo*: qué se creía en 2024, qué cambió, por qué. Editar el viejo borraría esa historia —que es el activo—.

Conecta directo con todo el track: cada decisión grande que tomaste en los módulos anteriores —[event-driven](event-driven.md) (coreografía vs orquestación), [NoSQL](nosql.md) (cuándo salir de Postgres), [Kubernetes](kubernetes.md) (¿K8s o Fargate?)— es exactamente el tipo de decisión que merece un ADR. El criterio que aprendiste a *tomar* en esos módulos, acá aprendés a *registrarlo* para que el equipo no lo pierda.

**Ejercicios 2**
2.1 ¿Qué problema concreto resuelve un ADR? Dá el escenario del "¿por qué hicimos esto?" seis meses después.
2.2 ¿Qué captura un ADR que el código por sí solo no puede contarte, y por qué eso es lo valioso?
2.3 ¿Por qué un ADR es inmutable? ¿Qué hacés cuando una decisión vieja se reemplaza, y por qué no editás el documento original?
2.4 Nombrá dos decisiones de módulos anteriores del track que merecerían un ADR y por qué.

---

## Módulo 3 — Cómo se escribe un ADR

**Teoría.** Un ADR es deliberadamente **corto** (una página) y tiene una estructura estándar. La trampa a evitar es el documento de 20 páginas que nadie escribe ni lee: si cuesta más de 20 minutos, no se hace. Hay variantes más elaboradas (**MADR**, o la de **Tyree-Akerman** que agrega las alternativas evaluadas), pero la de Nygard —cuatro secciones— es la más usada justamente por barata. La plantilla mínima (variante de Nygard):

```markdown
# ADR-0007: Usar PostgreSQL como base de datos principal

## Estado
Aceptado  (otros posibles: Propuesto · Aceptado · Rechazado · Deprecado · Superseded por ADR-00XX)

## Contexto
La Task API necesita persistir usuarios, proyectos y tareas, con relaciones
fuertes entre ellos y consultas que cruzan esas relaciones. El equipo tiene
experiencia sólida en SQL. Anticipamos algunos campos semiestructurados
(metadata de tareas) pero no un volumen que exija escala horizontal masiva.

## Decisión
Usaremos PostgreSQL como base de datos principal. Los campos semiestructurados
irán en columnas JSONB. No introducimos una base documental por ahora.

## Consecuencias
- (+) Integridad referencial, transacciones ACID, y queries relacionales que
      el modelo pide naturalmente.
- (+) El equipo es productivo desde el día uno (ya saben SQL).
- (+) JSONB cubre lo semiestructurado sin sumar otra tecnología que operar.
- (−) Si en el futuro necesitamos escala de escritura horizontal masiva,
      habrá que reevaluar (probablemente con un ADR nuevo).
- (−) Asumimos el costo operativo de un Postgres gestionado (RDS).
```

Las cuatro secciones y qué va en cada una:

- **Estado**: dónde está la decisión en su ciclo de vida. `Propuesto` mientras se discute; `Aceptado` cuando se adopta; `Superseded por ADR-XX` cuando otra la reemplaza (módulo 2).
- **Contexto**: las **fuerzas en juego** —requisitos, restricciones, lo que sabías y lo que no—. Esta es la sección más importante: es el "por qué" que el futuro necesita. Escribila de forma neutral (las fuerzas, no la conclusión).
- **Decisión**: qué se decidió, en voz activa y clara ("Usaremos X"). Corta.
- **Consecuencias**: qué pasa como resultado —**lo bueno y lo malo**—. Un ADR honesto lista los costos y los trade-offs negativos, no solo las ventajas. Acá es donde se ve la madurez: toda decisión arquitectónica tiene un precio, y nombrarlo es lo que la hace una decisión y no un acto de fe.

Reglas prácticas: numerados y secuenciales (`0001`, `0002`...), un archivo por decisión, en el repo (versionados con el código, revisados en un PR como cualquier cambio), escritos **en el momento** de decidir (no reconstruidos seis meses después, cuando ya te olvidaste de las fuerzas reales). Solo para decisiones **significativas y costosas de revertir** —no documentás "usamos Prettier", sí documentás "elegimos arquitectura de eventos coreografiada"—. El criterio de qué amerita un ADR es el mismo que el de qué amerita pensarlo dos veces: decisiones difíciles de deshacer.

**Ejercicios 3**
3.1 Nombrá las cuatro secciones de un ADR y qué va en cada una.
3.2 ¿Cuál es la sección más importante y por qué? ¿Cómo debería escribirse?
3.3 ¿Por qué la sección de consecuencias debe incluir lo *negativo*? ¿Qué señala que alguien solo liste ventajas?
3.4 ¿Qué tipo de decisiones ameritan un ADR y cuáles no? Dá un ejemplo de cada lado.
3.5 **Escribilo vos.** Elegí una decisión real del track —ej. "SNS+SQS vs Kafka para los eventos" o "¿K8s o Fargate?"— y escribí el ADR completo con las cuatro secciones (Estado, Contexto, Decisión, Consecuencias), incluyendo **al menos dos consecuencias negativas**. (Es el entregable concreto del módulo: no alcanza con saber qué es un ADR, hay que poder escribir uno.)

---

## Módulo 4 — Medir la entrega: las cuatro métricas DORA

**Teoría.** "¿Cómo sé si mi equipo entrega bien?" La respuesta vaga ("se sienten productivos", "cerramos muchos tickets") no sirve para mejorar. El programa de investigación **DORA** (DevOps Research and Assessment, sintetizado en el libro *Accelerate*, 2018, investigación **liderada por Nicole Forsgren** con Jez Humble y Gene Kim) encontró, con datos de miles de equipos, **cuatro métricas** que predicen el rendimiento de entrega de un equipo de software —y que correlacionan con el rendimiento del negocio—. Son el estándar de la industria para medir entrega.

Las cuatro, en dos grupos:

**Velocidad / throughput** (¿qué tan rápido entregamos?):
- **Deployment Frequency** (frecuencia de despliegue): ¿cada cuánto deployás a producción? Los equipos de élite deployan **on-demand, varias veces por día**; los de bajo rendimiento, una vez por mes o menos.
- **Lead Time for Changes** (tiempo de entrega de cambios): desde que se commitea un cambio hasta que está en producción. Élite: **menos de un día/una hora**; bajo: meses.

**Estabilidad** (¿qué tan bien entregamos?):
- **Change Failure Rate** (tasa de fallo de cambios): qué % de los deploys causa un fallo en producción que requiere remediación (hotfix, rollback). Élite: **0-15%**.
- **Failed Deployment Recovery Time** (tiempo de recuperación): cuánto tardás en restaurar el servicio cuando un deploy falla. Élite: **menos de una hora**. DORA **renombró** esta métrica —antes "Time to Restore" / **MTTR**— porque el MTTR resultó estadísticamente poco fiable como medida. La intuición se conecta con el MTTR del [módulo 11 de Observabilidad](observabilidad.md) (por eso la observabilidad es una inversión de liderazgo, no solo técnica: baja directamente esta métrica), pero no son idénticos —no la llames "MTTR" si querés ser preciso—.

Lo importante del *par* de grupos: DORA mide **velocidad Y estabilidad juntas**, a propósito. Medir solo velocidad incentiva deployar basura rápido; medir solo estabilidad incentiva no deployar nunca ("si no toco nada, no se rompe"). El equipo de alto rendimiento es bueno en **las cuatro a la vez** —y el hallazgo contraintuitivo de DORA (módulo 5) es que eso es posible, que no hay que elegir—.

**Más allá de las cuatro: la quinta dimensión.** Las cuatro miden *entrega*. DORA sumó después una quinta métrica de **fiabilidad / desempeño operacional** (*reliability*): si el equipo cumple sus **objetivos operativos** —disponibilidad, latencia, los SLOs del módulo de Observabilidad— y no solo si entrega rápido y estable. Es el contrapeso al incentivo de optimizar throughput a costa de que el sistema ande mal en producción: de nada sirve deployar 50 veces por día si el servicio se cae. (Como los benchmarks numéricos, la formulación exacta se recalibra entre reportes anuales.)

Para qué las usás como Tech Lead: **para mejorar el sistema de entrega, no para rankear personas.** Si tu lead time es de tres semanas, las métricas te dicen *que* hay un problema; investigás *dónde* (¿el CI tarda horas? ¿los PRs esperan días para review? ¿los deploys son manuales y aterradores?) y atacás esa restricción. Es el "medí antes de optimizar" de [NestJS senior](nestjs-senior.md) aplicado al proceso de entrega del equipo.

**Ejercicios 4**
4.1 Nombrá las cuatro métricas DORA y agrupalas en velocidad vs estabilidad.
4.2 ¿Cuál de las cuatro es el MTTR del módulo de Observabilidad, y qué dice eso sobre por qué la observabilidad es una inversión de liderazgo?
4.3 ¿Por qué DORA mide velocidad *y* estabilidad juntas? ¿Qué pasa si medís solo una?
4.4 Tu lead time es de tres semanas. ¿Qué hacés con esa métrica? ¿Para qué NO la usás?

---

## Módulo 5 — DORA en profundidad: velocidad y estabilidad no se contraponen

**Teoría.** La creencia tradicional, profundamente arraigada: **velocidad y estabilidad son un trade-off**. "Si querés que sea estable, andá despacio y con cuidado; si vas rápido, vas a romper cosas." Parece sentido común. El hallazgo central de DORA —respaldado por años de datos— es que **es falso**: los equipos de élite son **mejores en las cuatro métricas a la vez**. Van rápido *y* rompen menos. Y los equipos lentos no son más estables por ir lentos; suelen ser **peores en todo**.

¿Cómo es posible? Porque lo que parecía un trade-off era en realidad una **consecuencia de las mismas prácticas de ingeniería**. Lo que te hace ir rápido es lo mismo que te hace estable:

- **Lotes chicos.** Deployar cambios pequeños y frecuentes (alta deployment frequency) significa que cada deploy es de bajo riesgo —poco código, fácil de revisar, fácil de revertir—. El deploy gigante mensual es lo *inestable*: 300 cambios juntos, imposible saber cuál rompió qué. Ir rápido con lotes chicos *es* ir seguro.
- **Automatización y tests** (acá conecta todo el track): un pipeline de CI/CD ([Docker/deploy](docker-deploy.md)) con una suite de tests confiable ([TDD](tdd.md)) te deja deployar seguido sin miedo (sube deployment frequency y baja lead time) *y* atrapa los bugs antes de prod (baja change failure rate). La misma inversión mejora velocidad y estabilidad.
- **Observabilidad** ([módulo entero](observabilidad.md)): cuando algo se rompe, lo ves y lo diagnosticás rápido → baja el recovery time. Y al confiar en que vas a *ver* los problemas, te animás a deployar más seguido.

La conclusión de liderazgo, y el punto del módulo: **cuando alguien te plantea "¿vamos rápido o lo hacemos bien?", la pregunta está mal formulada.** La respuesta de un Tech Lead que entiende DORA es: *"las prácticas que nos hacen ir bien son las que nos hacen ir rápido —invertimos en CI, tests y observabilidad, y obtenemos las dos cosas"*. No es magia ni voluntarismo; es que el falso trade-off se disuelve con ingeniería. Por eso todo el track técnico (testing, TDD, CI/CD, observabilidad) **es** material de liderazgo: son las palancas con las que mejorás las cuatro métricas a la vez.

El cuidado al usar DORA (la trampa del que las descubre): **son métricas de sistema, no de personas, y son fáciles de gamear si las convertís en objetivo.** Si premiás "más deploys", la gente fragmenta deploys artificialmente; si castigás change failure rate, nadie reporta los fallos. (Es la **ley de Goodhart**: cuando una métrica se vuelve un objetivo, deja de ser una buena métrica.) Las usás para entender y mejorar el *sistema de entrega*, no para un ranking individual.

**La contracara cultural: postmortems sin culpa (blameless).** Si las métricas no son para señalar culpables, la práctica que lo encarna es el **postmortem blameless**: cuando algo falla en producción, el equipo reconstruye *qué pasó y por qué* para arreglar el **sistema** (procesos, alertas, guardas, tests faltantes) **sin buscar a quién culpar**. No es blandura ni "no pasa nada": es lo que hace que la gente **reporte** los fallos y los near-misses en vez de esconderlos —y reportar rápido y con honestidad es lo que de verdad baja el recovery time (la cuarta métrica)—. Es el Goodhart de recién aplicado a personas: si castigás el error, no desaparece el error, desaparece el *reporte* del error (y entonces te enterás peor y más tarde). La cultura blameless es el complemento directo del "medí el sistema, no a las personas", y la práctica que más sostiene la fiabilidad (la quinta dimensión) en el día a día. Un buen postmortem termina en *acciones sobre el sistema* con dueño y fecha, no en "fulano tiene que tener más cuidado".

**Ejercicios 5**
5.1 ¿Cuál es la creencia tradicional sobre velocidad vs estabilidad y qué encontró DORA al respecto?
5.2 Explicá con el ejemplo de "lotes chicos" por qué ir rápido puede ser ir más seguro, no menos.
5.3 ¿Por qué se dice que el track técnico entero (testing, TDD, CI/CD, observabilidad) es "material de liderazgo"?
5.4 ¿Qué es la ley de Goodhart y cómo se aplica al riesgo de usar mal las métricas DORA? Dá un ejemplo de gaming.
5.5 ¿Qué es un postmortem *blameless* y por qué *baja* el recovery time en vez de ser solo "ser amable"? Conectalo con la ley de Goodhart aplicada a personas.

---

## Módulo 6 — Organizar equipos: Team Topologies

**Teoría.** A medida que una organización crece, la pregunta deja de ser "¿cómo estructuro mi código?" y pasa a ser "¿cómo estructuro mis **equipos**?". Y las dos preguntas están atadas por la **Ley de Conway**: *las organizaciones diseñan sistemas que copian su estructura de comunicación*. Si tenés cuatro equipos, vas a terminar con cuatro grandes módulos que se comunican como se comunican los equipos. El corolario de liderazgo —el **Inverse Conway Maneuver**— es: **diseñá los equipos con la forma que querés que tenga el software**, no al revés.

**Team Topologies** (Matthew Skelton y Manuel Pais) es el marco moderno para esto. Su tesis: en vez de inventar organigramas ad hoc, hay **cuatro tipos fundamentales de equipo**, y casi todo equipo sano encaja en uno:

1. **Stream-aligned** (alineado al flujo de valor): el equipo principal y más común. Es dueño **de punta a punta** de un flujo de valor —una parte del producto que entrega valor al usuario— (ej: "el equipo de Proyectos" o "el equipo de Onboarding"). Tiene todo lo que necesita para entregar sin depender constantemente de otros. **La mayoría de los equipos deberían ser de este tipo**; los otros tres existen para *reducir la carga* de estos.

2. **Platform** (plataforma): provee servicios internos que los stream-aligned **consumen como autoservicio** para no reinventar (la infra común, el pipeline de CI/CD, el cluster de [Kubernetes](kubernetes.md), las herramientas de [observabilidad](observabilidad.md)). El equipo de proyectos no debería tener que armar su propio sistema de tracing; la plataforma se lo da listo. La plataforma es un **producto interno**: su cliente es el equipo stream-aligned, y el éxito se mide en cuánto le reduce la fricción.

3. **Enabling** (habilitador): un equipo de especialistas que **ayuda temporalmente** a los stream-aligned a adquirir una capacidad que les falta (ej: expertos en testing que enseñan TDD, o en seguridad que suben el nivel del equipo). No hacen el trabajo *por* ellos; los **suben de nivel y se van**. Su éxito es volverse innecesarios.

4. **Complicated-subsystem** (subsistema complicado): un equipo dedicado a una parte que requiere **conocimiento especializado profundo** que no es razonable pedirle a cada stream-aligned (un motor de cálculo matemático complejo, un componente de ML, un códec de video). Existe solo cuando la complejidad lo justifica.

La idea de fondo: **estos cuatro tipos, con responsabilidades claras, evitan el caos de "todos dependen de todos".** La mayoría stream-aligned (entregan valor), apoyados por una plataforma (les saca trabajo repetitivo), habilitados puntualmente (enabling) y aislados de la complejidad especializada (complicated-subsystem).

**Ejercicios 6**
6.1 Enunciá la Ley de Conway y el "Inverse Conway Maneuver". ¿Qué implica para un Tech Lead que diseña equipos?
6.2 Nombrá los cuatro tipos de equipo de Team Topologies y cuál debería ser la mayoría.
6.3 ¿Por qué se dice que un equipo de plataforma es "un producto interno" y cómo se mide su éxito?
6.4 ¿Cuál es el objetivo paradójico de un equipo enabling? Dá un ejemplo ligado a este temario.

---

## Módulo 7 — Modos de interacción y carga cognitiva

**Teoría.** Definir los tipos de equipo es la mitad; la otra mitad de Team Topologies es **cómo interactúan** y **por qué importa la carga cognitiva**.

**Los tres modos de interacción** —las únicas formas sanas en que dos equipos se relacionan—:

- **Collaboration** (colaboración): dos equipos trabajan juntos, codo a codo, por un tiempo limitado, sobre algo nuevo o incierto (ej: el stream-aligned y la plataforma co-diseñan una capacidad nueva). Es intensa y de alto ancho de banda, pero **costosa** —difumina responsabilidades—, así que es **temporal**: sirve para descubrir, no como estado permanente.
- **X-as-a-Service** (X como servicio): un equipo **consume** lo que otro provee con mínima interacción (el stream-aligned usa el pipeline de la plataforma como un servicio, sin reuniones). Es el modo de **bajo costo y claro**: límites nítidos, autoservicio. La mayoría de las relaciones maduras tienden a este modo.
- **Facilitating** (facilitación): un equipo (típicamente enabling) **ayuda a otro a mejorar**, sin hacer el trabajo por él. Es el modo del mentoring entre equipos.

El patrón sano: la colaboración es el modo *temporal* de descubrir cómo debería ser una relación, que luego **decanta** en X-as-a-Service (la plataforma maduró el servicio y ya no hace falta co-diseñar). Un equipo que vive en colaboración permanente con muchos otros es señal de límites mal puestos.

**La carga cognitiva** —el concepto que justifica todo el marco—: un equipo tiene una **capacidad finita de cosas que puede tener en la cabeza** (el dominio, el stack, las herramientas, los sistemas con los que integra). El error organizacional clásico es **sobrecargar a un equipo** dándole demasiados sistemas o dominios dispares: rinde mal en todos, nadie entiende nada a fondo, y la calidad cae. El propósito último de Team Topologies es **mantener la carga cognitiva de cada equipo dentro de su límite**:

- El stream-aligned se enfoca en *su* flujo de valor (carga acotada).
- La **plataforma le saca de encima** la carga de la infra (no tiene que pensar en cómo opera Kubernetes).
- El **complicated-subsystem le saca** la carga del conocimiento especializado.
- El **enabling le ayuda** a subir una capacidad sin que tenga que descubrir todo solo.

Para vos como Tech Lead, esto es directamente accionable: **una de tus tareas es proteger la carga cognitiva de tu equipo.** Cuando alguien quiere sumarle al equipo un quinto sistema no relacionado, o cuando el equipo está peleando con infra que debería ser autoservicio, estás viendo un problema de carga cognitiva —y la solución suele ser organizacional (mover responsabilidad, pedir plataforma), no "que se esfuercen más"—. Conecta con el módulo 1: parte de multiplicar al equipo es **no saturarlo**.

**Ejercicios 7**
7.1 Nombrá los tres modos de interacción y cuál es costoso/temporal vs cuál es de bajo costo/permanente.
7.2 Describí el patrón sano de cómo una relación entre equipos evoluciona entre modos.
7.3 ¿Qué es la carga cognitiva de un equipo y cuál es el error organizacional clásico relacionado?
7.4 ¿Cómo "descargan" cada uno de los otros tres tipos de equipo al stream-aligned? ¿Qué tarea concreta de Tech Lead se desprende de esto?

---

## Módulo 8 — El trabajo que no es código: review, mentoring, decir que no

**Teoría.** Gran parte del impacto de un Tech Lead pasa por actividades que no producen ni una línea de código propio pero que **multiplican la calidad de todo el equipo**. Las tres más importantes:

**1. Code review como herramienta de liderazgo.** El review no es solo cazar bugs; es donde se **propaga el criterio** y se eleva el nivel del equipo. Cómo lo hace un buen lead:
- Revisa **comportamiento y diseño**, no solo estilo (el estilo lo automatiza un linter). Las preguntas valiosas: ¿esto resuelve el problema correcto? ¿el diseño aguanta el cambio que viene? ¿los tests verifican comportamiento o implementación ([TDD módulo 5](tdd.md))?
- **Explica el porqué, no solo el qué.** "Cambiá esto" enseña poco; "cambiá esto *porque* acopla el test a la implementación y se va a romper en el próximo refactor" enseña criterio que el dev aplica solo la próxima vez.
- **Distingue lo bloqueante de lo opcional** (nit/sugerencia vs. must-fix), para no ahogar un PR en perfeccionismo y matar el lead time (módulo 4).
- Es **rápido**: un PR que espera tres días para review destruye el lead time y desmotiva. Revisar pronto es trabajo de multiplicador.

**2. Mentoring y crecimiento.** Tu trabajo incluye que la gente del equipo **sea mejor ingeniero en seis meses**. Eso es delegar trabajo desafiante (aunque vos lo harías más rápido), dar feedback concreto y a tiempo, y hacer pairing en los problemas difíciles. Un equipo que crece es un equipo que multiplica tu impacto sin que vos crezcas en horas.

**3. Decir que no (y traducir).** El Tech Lead es la interfaz entre la presión del negocio ("¿se puede para el viernes?") y la realidad técnica. Dos habilidades:
- **Decir que no con fundamento**: no "no se puede" a secas, sino "para el viernes podemos entregar X sin Y; Y requiere dos semanas más por Z —¿qué priorizamos?". Das **opciones y trade-offs**, no un muro ni un sí imposible.
- **Proteger el tiempo del equipo**: filtrar interrupciones, defender espacio para pagar deuda técnica (módulo 9), no aceptar cada pedido urgente como urgente real.
- **Cerrar desacuerdos técnicos sin imponer ni eternizarlos** (*disagree and commit*): cuando dos buenos ingenieros discrepan y no hay un ganador claro, el TL no deja el debate abierto para siempre ni decide por jerarquía. Escucha, decide con criterio, lo **registra en un ADR** (con las fuerzas y los trade-offs) y el equipo se compromete con la decisión aunque alguno hubiera elegido distinto. El registro es lo que vuelve sano el "commit": no es "porque lo digo yo", es "por estas razones, que quedan escritas y se pueden revisar si cambian las premisas".

El hilo común de las tres: **tu trabajo es subir el techo del equipo, no ser el techo del equipo.** Si todo pasa por vos —cada decisión, cada review crítico, cada problema difícil—, sos un cuello de botella, no un líder. Multiplicar es distribuir criterio, no centralizarlo.

**Ejercicios 8**
8.1 ¿Por qué el code review es "herramienta de liderazgo" y no solo caza-bugs? Dá dos prácticas de un buen review.
8.2 ¿Por qué "explicar el porqué" en un review multiplica más que "decir el qué"?
8.3 **Escribilo vos.** A mitad del sprint, un stakeholder pide sumar una feature grande "porque es urgente". Escribí la respuesta de un Tech Lead: ni un "no" seco ni un "sí" imposible. (Pensá en opciones, trade-offs y qué se desplaza.)
8.4 Explicá "subí el techo del equipo, no seas el techo del equipo". ¿Qué antipatrón describe?

---

## Módulo 9 — Deuda técnica: gestionarla con datos, no con miedo

**Teoría.** La **deuda técnica** es la metáfora (Ward Cunningham) de las decisiones de implementación que aceleran ahora pero **cobran intereses después** en forma de mayor costo de cambio. Como la deuda financiera, **no es intrínsecamente mala**: tomar deuda a propósito para llegar a una fecha o validar una hipótesis puede ser una buena decisión de negocio. Lo que la vuelve tóxica es **no gestionarla**: tomarla sin saberlo, no registrarla, y nunca pagarla, hasta que los intereses (cada cambio tarda el triple, cada deploy da miedo) ahogan al equipo.

El error de los dos extremos, que un Tech Lead evita:

- **El que la ignora**: "no hay tiempo para refactorizar, hay que entregar features". Acumula intereses hasta que el equipo se paraliza —el famoso "no toques eso que anda y nadie entiende", que es deuda técnica con intereses impagos—.
- **El perfeccionista**: quiere refactorizar todo, deja de entregar valor, persigue elegancia que el negocio no pidió. Paga deuda que no estaba cobrando intereses.

El criterio del medio, gestionar con **datos**:

- **Hacela visible.** La deuda invisible no se prioriza. Registrala (tickets, una sección en los ADRs, un mapa de las zonas frágiles del código). Lo que conecta con el track: la deuda se *mide* con las herramientas que ya tenés —si la [observabilidad](observabilidad.md) muestra que cierto módulo concentra los incidentes, o si las métricas [DORA](#módulo-4--medir-la-entrega-las-cuatro-métricas-dora) muestran que el lead time sube por una zona específica, ahí está la deuda cobrando intereses, con evidencia—.
- **Priorizá la que cobra intereses altos.** No toda deuda importa: la deuda en código que nadie toca puede quedarse ahí para siempre (no cobra intereses). La que importa es la del código que **cambiás seguido** o que **concentra bugs/incidentes** —esa frena al equipo cada semana—. Pagás *esa* primero.
- **Pagala de a poco, continuamente**, no en un "gran refactor" de tres meses (que el negocio nunca aprueba y que suele fallar). La regla del boy scout: dejá el código un poco mejor de como lo encontraste, cada vez que pasás por él. Y la red de seguridad para poder hacerlo sin miedo es la suite de tests ([TDD](tdd.md)): no podés pagar deuda refactorizando si no tenés tests que te avisen si rompés algo.

La frase de liderazgo: **la deuda técnica es una decisión de negocio disfrazada de decisión técnica.** Tu trabajo es traducirla —"esta zona frágil nos cuesta X velocidad por sprint; pagarla cuesta Y; ¿lo hacemos?"— para que se priorice con criterio, no que se ignore por invisible ni se persiga por perfeccionismo.

**Ejercicios 9**
9.1 ¿Por qué la deuda técnica "no es intrínsecamente mala"? ¿Qué la vuelve tóxica?
9.2 Describí los dos extremos (el que la ignora y el perfeccionista) y qué hace mal cada uno.
9.3 ¿Qué deuda hay que priorizar y cuál puede esperar para siempre? ¿Cómo usás observabilidad y DORA para detectarla?
9.4 ¿Por qué "pagar de a poco" es mejor que el "gran refactor", y qué del track técnico es la red de seguridad que lo hace posible?
9.5 **Escribilo vos.** Elegí una zona frágil concreta (real o inventada) y escribí su caso de negocio en el formato "esta zona nos cuesta X; pagarla cuesta Y; ¿lo hacemos?", apoyándote en una señal de datos (observabilidad o DORA).

---

## Módulo 10 — La IA en el rol del Tech Lead

**Teoría.** En 2026 ningún panorama del rol está completo sin esto: los asistentes de IA (Claude Code y similares) cambiaron el trabajo de ingeniería, y un Tech Lead tiene que tener **criterio** sobre cómo el equipo los usa —ni prohibirlos por miedo ni adoptarlos sin pensar—.

Qué cambia, concretamente:

- **El cuello de botella se corre de escribir a revisar y decidir.** Si la IA genera código mucho más rápido, el límite del equipo deja de ser *cuánto código escribís* y pasa a ser *cuánto código podés revisar, entender y mantener con confianza*. Esto **amplifica** la importancia de todo lo de este track: el code review riguroso (módulo 8), los tests confiables ([TDD](tdd.md)) y la observabilidad ([módulo](observabilidad.md)) son aún más críticos cuando hay más código entrando más rápido. La IA no reemplaza el criterio de ingeniería; lo hace *más* valioso por escaso.
- **El piso sube, el techo lo pone el criterio.** La IA hace que tareas mecánicas (boilerplate, tests triviales, traducir entre lenguajes) sean casi gratis. Lo que **no** delega: las decisiones de arquitectura (los ADRs siguen siendo humanos —la IA no carga las consecuencias de la decisión), entender el dominio del negocio, y el juicio sobre qué construir y por qué.

Lo que un Tech Lead debe sostener sobre el uso de IA en el equipo:

- **El que mergea es responsable, no la IA.** Código generado por IA es código del que tu equipo se hace dueño: hay que entenderlo, revisarlo y testearlo con el *mismo* rigor (o más) que el escrito a mano. "Lo generó la IA" no es una excusa para un bug en producción. Conecta con [TDD módulo 10](tdd.md): el que mergea responde por los tests y el comportamiento.
- **Cuidado con la deuda técnica acelerada.** La IA puede generar código que *parece* funcionar y acumular deuda (módulo 9) a una velocidad nueva si nadie lo revisa con criterio. Más velocidad de generación sin más rigor de review = intereses que se acumulan más rápido.
- **Como multiplicador, la IA es enorme** si el equipo tiene las prácticas de base. Si no las tiene (sin tests, sin observabilidad, sin review), la IA multiplica también el caos. Por eso el track técnico es el prerrequisito: la IA amplifica lo que ya hay.

Y para vos personalmente: el lead que usa la IA bien delega en ella lo mecánico para liberar tiempo de *multiplicador* (decidir, mentorear, desbloquear) —exactamente el cambio del módulo 1—. La frase de criterio: **la IA cambia cuánto código se produce, no quién es responsable de que sea correcto, mantenible y resuelva el problema correcto. Ese responsable seguís siendo vos y tu equipo.**

**Ejercicios 10**
10.1 ¿Hacia dónde se corre el cuello de botella del equipo cuando la IA acelera la escritura de código, y qué prácticas del track se vuelven *más* importantes por eso?
10.2 Nombrá dos cosas que la IA hace casi gratis y dos que un Tech Lead no debería delegarle.
10.3 "Lo generó la IA" como excusa de un bug en prod: ¿por qué no es válida? ¿Con qué módulo de TDD conecta?
10.4 ¿Por qué la IA puede acelerar la acumulación de deuda técnica, y por qué el track técnico es prerrequisito para que la IA sea multiplicador y no caos?

---

## Módulo 11 — El criterio: liderar es multiplicar, no controlar

**Teoría.** El módulo que cierra el módulo, el track y —junto con el roadmap— el temario. Tenés las herramientas: ADRs para decidir y dejar el porqué, DORA para medir la entrega, Team Topologies para organizar, y las prácticas blandas (review, mentoring, decir que no, gestionar deuda). El error de madurez en el rol no es no conocer las herramientas; es **confundir liderar con controlar** —querer que todo pase por vos, decidir todo, ser el que más sabe de todo—.

Los principios de criterio que cierran el track:

1. **Tu trabajo es multiplicar, no controlar.** (Módulo 1, ahora como síntesis.) El líder que centraliza cada decisión y cada review crítico es un cuello de botella que limita al equipo a *su* capacidad. El que distribuye criterio —vía ADRs que enseñan a decidir, reviews que explican el porqué, mentoring que sube el nivel— hace un equipo que escala más allá de él. **Medí tu éxito por lo que el equipo logra cuando no estás en la sala, no por cuánto dependen de vos.**

2. **Decidí con criterio explícito y registrado, no por autoridad.** "Lo hacemos así porque lo digo yo" no escala ni enseña. "Lo hacemos así por *estas* fuerzas y *estos* trade-offs" (un ADR) escala, se puede cuestionar con datos, y forma a los demás para decidir igual de bien. La autoridad técnica se gana con buen criterio visible, no con el título.

3. **Medí el sistema, mejorá el sistema** (DORA), **sin gamear ni rankear personas.** Las métricas son para entender y destrabar el proceso de entrega, no para vigilar individuos. Un lead que usa métricas para señalar culpables destruye la confianza que necesita para liderar.

4. **Escalá la solución al problema** —el eje de criterio de TODO el temario, ahora a nivel organizacional—. No metés Team Topologies de cuatro tipos de equipo en un equipo de tres personas; no escribís un ADR para elegir Prettier; no montás observabilidad de sistema crítico en un side project. La misma pregunta de "¿[K8s o Fargate](kubernetes.md)?", "¿[llamada, workflow o agente](agentes.md)?", "¿[cuánta observabilidad](observabilidad.md)?", "¿[TDD acá o no](tdd.md)?" se vuelve "¿este equipo necesita esta estructura/proceso o es overkill?". La madurez es ajustar la herramienta al contexto, no aplicar la máxima en todos lados.

5. **Liderar es un oficio aparte, no un ascenso por antigüedad técnica.** El mejor IC no es automáticamente buen lead —son habilidades distintas (multiplicar gente vs. resolver problemas)—. Se entrena: como TDD (módulo 11 de ese módulo), los primeros pasos se sienten torpes (te cuesta delegar, querés codear todo vos), y el payoff —un equipo que entrega bien y crece— aparece en el mediano plazo.

El cierre del track y del perfil: con liderazgo técnico completás **la mitad del perfil senior que no es código puro**. La otra mitad —entregar software de calidad— la tenés en todo el track técnico; **esta** es la que te deja **dirigir cómo un equipo lo entrega**: decidir y registrar (ADRs), medir y mejorar (DORA), organizar y proteger (Team Topologies, carga cognitiva), desarrollar gente (review, mentoring) y gestionar lo difícil (deuda, decir que no, la IA con criterio). El hilo conductor del temario entero queda cerrado: empezaste pasando de Frontend a backend con Node, y terminás con las dos mitades del perfil contratable senior/Tech Lead/IA Engineer —construir bien y liderar la construcción—. Desde acá el crecimiento ya no es de temario, es de práctica: liderar equipos reales, tomar decisiones reales, y registrar lo que aprendés (volvé a empezar, ahora como mentor de otros).

**Ejercicios 11**
11.1 ¿Cuál es el error de madurez central en el rol de Tech Lead, y cómo se mide el éxito de un líder que multiplica en vez de controlar?
11.2 ¿Por qué decidir "con criterio explícito y registrado" escala mejor que decidir "por autoridad"? Conectá con los ADRs.
11.3 Aplicá "escalá la solución al problema" al nivel organizacional con dos ejemplos (uno de Team Topologies, uno de ADRs).
11.4 ¿Por qué "el mejor IC no es automáticamente buen lead"? ¿Qué tienen en común liderar y aprender TDD según el cierre?
11.5 Resumí en una frase qué mitad del perfil senior aporta el liderazgo técnico y cuál aporta el resto del track.

---

## Soluciones

### Módulo 1
1.1 De "mi impacto es lo que **yo** produzco" (IC/senior) a "mi impacto es lo que produce **el equipo entero**" (Tech Lead). Tu tiempo más valioso pasa a ser el que sube el output de otros, no el que suma tu propio código.
1.2 No deja de programar: un TL que no toca código pierde el pulso técnico y la credibilidad. Pero codea **lo que solo él puede/debe** —los problemas espinosos, las pruebas de concepto que destraban una decisión— y delega el resto, aunque lo haría "mejor", porque su tiempo rinde más multiplicando.
1.3 Es pasar de producir valor con tus propias manos (hacedor) a **subir el output de otros** (multiplicador). Ejemplos: desbloquear a alguien trabado, decidir una dirección técnica, revisar bien un PR, mentorear, sacar un impedimento organizacional.
1.4 El **Engineering Manager** es responsable de las *personas* (carrera, performance, contratación, 1:1s); el **Tech Lead** de lo *técnico* (arquitectura, calidad, dirección). Confundirlos —hacer gestión de personas *y* arquitectura *y* codear full-time a la vez— lleva al burnout porque son tres trabajos distintos sumados.

### Módulo 2
2.1 Resuelve la **pérdida del "por qué" de las decisiones**. Escenario: seis meses después llega alguien (o vos olvidado) y pregunta "¿por qué usamos Postgres y no Mongo?"; sin registro, es arqueología (Slack viejo, gente que se fue) o se revierte la decisión sin saber qué problema resolvía, chocando con el mismo muro.
2.2 Captura el **contexto y el porqué** —las fuerzas, restricciones e información que había al decidir—, que el código no puede contarte (el código muestra *que* usás Postgres, no *por qué* sobre Mongo). Es lo valioso porque te deja reevaluar con criterio en el futuro: sabés qué premisa se rompió si algo cambió.
2.3 Es inmutable porque su valor es **histórico**: muestra cómo evolucionó el pensamiento del equipo. Cuando una decisión se reemplaza, escribís un **ADR nuevo** que *supersede* al viejo (y marcás el viejo como `Superseded`); no editás el original porque borrarías la historia de qué se creía y por qué cambió —que es el activo—.
2.4 (Dos de) coreografía vs orquestación en event-driven, cuándo salir de Postgres a NoSQL, ¿K8s o Fargate?, JWT corto + refresh en auth. Todas son decisiones significativas, costosas de revertir y cuyo *porqué* (las fuerzas del momento) se perdería sin registro.

### Módulo 3
3.1 **Estado** (dónde está la decisión en su ciclo: propuesto/aceptado/superseded), **Contexto** (las fuerzas, requisitos y restricciones en juego), **Decisión** (qué se decidió, en voz activa), **Consecuencias** (qué resulta, lo bueno y lo malo).
3.2 El **Contexto**, porque es el "por qué" que el futuro necesita y que el código no guarda. Debe escribirse de forma **neutral** —las fuerzas en juego, no la conclusión— para que se entienda qué problema y qué restricciones había realmente.
3.3 Porque toda decisión arquitectónica tiene un **precio/trade-off**, y nombrarlo es lo que la hace una decisión meditada y no un acto de fe; además avisa al futuro qué costos asumió. Que alguien liste solo ventajas señala inmadurez o que no pensó los trade-offs (ninguna decisión real es gratis).
3.4 Ameritan ADR las decisiones **significativas y costosas de revertir** (ej: arquitectura de eventos coreografiada, elección de base de datos principal). No ameritan las triviales o fácilmente reversibles (ej: usar Prettier, el nombre de una variable). El criterio = decisiones difíciles de deshacer.
3.5 Ejemplo de referencia (hay variantes válidas; lo que importa es que el Contexto sea neutral y las Consecuencias incluyan lo negativo):

```markdown
# ADR-0012: Eventos vía SNS+SQS en vez de Kafka
## Estado
Aceptado
## Contexto
Necesitamos comunicación asíncrona entre servicios (notificaciones, proyecciones).
El volumen es moderado (miles de eventos/día, no millones/seg). El equipo ya opera
en AWS y no tiene experiencia operando Kafka. No necesitamos reprocesar el log de
eventos desde el inicio ni retención larga por ahora.
## Decisión
Usaremos SNS+SQS (fan-out + colas) como bus de eventos. No introducimos Kafka.
## Consecuencias
- (+) Cero operación: son servicios gestionados; encaja con el stack actual.
- (+) Fan-out y reintentos/DLQ resueltos sin infra propia.
- (−) No tenemos un log de eventos reproducible (replay) como en Kafka; si más
      adelante lo necesitamos, habrá que reevaluar (probablemente con un ADR nuevo).
- (−) El ordering estricto y el throughput masivo no son el fuerte de SQS estándar.
```

### Módulo 4
4.1 **Deployment Frequency** y **Lead Time for Changes** (velocidad/throughput); **Change Failure Rate** y **Failed Deployment Recovery Time/MTTR** (estabilidad).
4.2 El **Failed Deployment Recovery Time** es el MTTR del módulo de Observabilidad. Dice que la observabilidad es inversión de liderazgo, no solo técnica: baja directamente una de las cuatro métricas DORA (recuperás más rápido porque *ves* y diagnosticás el problema rápido).
4.3 Para no incentivar un extremo perverso: medir solo velocidad incentiva deployar basura rápido; medir solo estabilidad incentiva no deployar nunca ("si no toco nada, no se rompe"). El alto rendimiento es ser bueno en las cuatro a la vez.
4.4 La usás para **investigar dónde está la restricción** (CI lento, PRs que esperan días, deploys manuales aterradores) y atacarla —mejorar el *sistema* de entrega—. NO la usás para rankear o culpar personas (es métrica de sistema).

### Módulo 5
5.1 La creencia tradicional: velocidad y estabilidad son un trade-off (rápido = rompés; estable = lento). DORA encontró que es **falso**: los equipos de élite son mejores en las cuatro métricas a la vez (rápido *y* estable), y los lentos no son más estables, suelen ser peores en todo.
5.2 Lotes chicos = deploys pequeños y frecuentes, cada uno de bajo riesgo (poco código, fácil de revisar y revertir). El deploy gigante mensual (lento) es el *inestable*: 300 cambios juntos, imposible saber cuál rompió qué. Por eso ir rápido con lotes chicos *es* ir más seguro.
5.3 Porque las prácticas que mejoran las cuatro métricas a la vez (CI/CD, tests confiables, observabilidad) son exactamente el contenido del track técnico. Esas son las palancas con las que un lead mejora velocidad *y* estabilidad: invertir en ellas no es "lujo técnico", es la herramienta de liderazgo para la entrega.
5.4 **Ley de Goodhart**: cuando una métrica se vuelve objetivo, deja de ser buena métrica (la gente la gamea). Aplicada a DORA: si premiás "más deploys", fragmentan deploys artificialmente; si castigás change failure rate, dejan de reportar fallos. Por eso son para entender el sistema, no para objetivos/rankings individuales.
5.5 Un postmortem blameless reconstruye qué falló y por qué para arreglar el **sistema** (procesos, alertas, guardas, tests), sin buscar culpable. Baja el recovery time porque hace que la gente **reporte** los fallos rápido y con honestidad en vez de esconderlos —y solo podés recuperarte rápido de lo que se reporta a tiempo—. Es el Goodhart aplicado a personas: si castigás el error, no desaparece el error, desaparece el *reporte* del error (te enterás peor y más tarde).

### Módulo 6
6.1 **Ley de Conway**: las organizaciones diseñan sistemas que copian su estructura de comunicación. **Inverse Conway Maneuver**: diseñá los *equipos* con la forma que querés para el *software*. Implica que un TL, al organizar equipos, está diseñando indirectamente la arquitectura —debe hacerlo a propósito—.
6.2 **Stream-aligned**, **platform**, **enabling**, **complicated-subsystem**. La mayoría debería ser **stream-aligned** (los otros tres existen para reducirles la carga).
6.3 Porque su cliente son los equipos stream-aligned que consumen sus servicios como **autoservicio**, igual que un producto tiene usuarios. Su éxito se mide en **cuánta fricción le reduce** al equipo cliente (no en features entregadas en abstracto).
6.4 Su objetivo paradójico es **volverse innecesario**: sube de nivel a un stream-aligned en una capacidad que le falta y se retira (no hace el trabajo por él). Ejemplo: especialistas que enseñan TDD o testing al equipo y luego se van.

### Módulo 7
7.1 **Collaboration** (costoso, temporal —difumina responsabilidades, sirve para descubrir), **X-as-a-Service** (bajo costo, límites claros, tiende a ser el permanente/maduro), **Facilitating** (un equipo ayuda a otro a mejorar). Costoso/temporal = collaboration; bajo costo/permanente = X-as-a-Service.
7.2 La **colaboración** es el modo temporal para descubrir cómo debería ser la relación (co-diseñar algo nuevo/incierto), y luego **decanta en X-as-a-Service** cuando el servicio madura y ya no hace falta co-diseñar. Vivir en colaboración permanente con muchos equipos es señal de límites mal puestos.
7.3 Es la **cantidad finita de cosas que un equipo puede tener en la cabeza** (dominio, stack, herramientas, integraciones). El error clásico es **sobrecargar** al equipo con demasiados sistemas/dominios dispares: rinde mal en todos, nadie entiende nada a fondo, cae la calidad.
7.4 La **plataforma** le saca la carga de la infra (no piensa cómo opera K8s); el **complicated-subsystem** le saca la del conocimiento especializado; el **enabling** le ayuda a subir una capacidad sin descubrir todo solo. Tarea de TL: **proteger la carga cognitiva del equipo** —no sumarle sistemas no relacionados, pedir plataforma/autoservicio cuando pelea con infra—.

### Módulo 8
8.1 Porque el review es donde se **propaga criterio y se eleva el nivel** del equipo, no solo donde se cazan bugs (eso, en parte, lo hace el linter/los tests). Dos prácticas: revisar **diseño y comportamiento** (no solo estilo), y **explicar el porqué** de cada comentario; (también: distinguir bloqueante de opcional, ser rápido).
8.2 Porque "decir el qué" ("cambiá esto") arregla un PR puntual; "explicar el porqué" ("porque acopla el test a la implementación y se romperá en el próximo refactor") enseña **criterio que el dev aplica solo la próxima vez** —multiplica, no parchea—.
8.3 Con **opciones y trade-offs**, no un muro ni un sí imposible. Ejemplo de respuesta: "Entiendo la urgencia. Esa feature es ~5 días y el sprint ya está comprometido; tengo tres caminos: (a) la metemos ahora y sale del sprint la tarea Z de menor prioridad; (b) la armamos como spike esta semana y la planificamos completa para el próximo sprint; (c) entregamos una versión mínima el viernes y el resto después. ¿Cuál prioriza el negocio?". Das el costo y las opciones con su trade-off; la decisión de prioridad la toma quien corresponde, con información.
8.4 Significa que tu rol es **elevar la capacidad del equipo** (subir el techo), no ser vos el límite de todo lo que el equipo puede hacer (ser el techo). Describe el antipatrón del lead **cuello de botella**: todo pasa por él (cada decisión, review crítico, problema difícil), así que el equipo no puede superar su capacidad individual.

### Módulo 9
9.1 Porque, como la deuda financiera, tomarla a propósito puede ser una buena decisión (llegar a una fecha, validar una hipótesis). La vuelve tóxica **no gestionarla**: tomarla sin saberlo, no registrarla y nunca pagarla, hasta que los intereses (cambios lentos, deploys con miedo) ahogan al equipo.
9.2 **El que la ignora** ("no hay tiempo para refactorizar") acumula intereses hasta paralizar al equipo ("no toques eso que anda"). **El perfeccionista** quiere refactorizar todo y deja de entregar valor, pagando deuda que no cobraba intereses (código elegante que el negocio no pidió).
9.3 Priorizás la deuda en código que **cambiás seguido** o que **concentra bugs/incidentes** (cobra intereses altos, frena al equipo cada semana); la del código que nadie toca puede esperar para siempre. La detectás con datos: la **observabilidad** muestra qué módulo concentra incidentes y **DORA** muestra si el lead time sube por una zona específica.
9.4 Porque el "gran refactor" de tres meses el negocio no lo aprueba y suele fallar; pagar de a poco (regla del boy scout: dejá el código mejor de como lo encontraste) es sostenible y continuo. La red de seguridad que lo hace posible es la **suite de tests (TDD)**: sin tests no podés refactorizar sin miedo a romper algo.
9.5 Ejemplo de referencia: "El módulo de facturación concentra el 40% de los incidentes de los últimos dos meses (lo muestra la observabilidad) y cada cambio ahí toma ~3× lo normal (se nota en el lead time de esos PRs). Estabilizarlo —tests de caracterización + extraer la lógica de impuestos— son ~6 días. Costo de no hacerlo: ~1 día/sprint de fricción y riesgo de incidentes en el flujo que factura. Propongo pagarlo el próximo sprint. ¿Lo aprobamos?". La clave: cuantificar el interés con datos (observabilidad/DORA), el costo de pagar, y dejar la decisión priorizada con evidencia, no con miedo.

### Módulo 10
10.1 Se corre de **escribir** código a **revisarlo, entenderlo y mantenerlo con confianza** (entra más código más rápido). Se vuelven *más* importantes: el code review riguroso, los tests confiables (TDD) y la observabilidad —el criterio de ingeniería se hace más valioso por escaso, no menos—.
10.2 Casi gratis (dos de): boilerplate, tests triviales, traducir entre lenguajes, mapeos mecánicos. No delegables: las decisiones de arquitectura (ADRs —la IA no carga las consecuencias), entender el dominio del negocio, el juicio sobre qué construir y por qué.
10.3 No es válida porque el que **mergea se hace dueño** del código: hay que entenderlo, revisarlo y testearlo con el mismo rigor (o más) que el escrito a mano; "lo generó la IA" no responde por un bug en prod. Conecta con **TDD módulo 10**: el que mergea responde por los tests y el comportamiento.
10.4 Porque genera código que *parece* funcionar a una velocidad nueva, y sin review con criterio acumula deuda más rápido (más generación sin más rigor = intereses que crecen más rápido). El track técnico (tests, observabilidad, review) es prerrequisito porque la IA **amplifica lo que ya hay**: con buenas prácticas multiplica valor; sin ellas multiplica el caos.

### Módulo 11
11.1 El error es **confundir liderar con controlar** (querer que todo pase por vos, decidir todo, ser el que más sabe). El éxito de un líder multiplicador se mide por **lo que el equipo logra cuando no estás en la sala** —cuánto criterio distribuiste—, no por cuánto dependen de vos.
11.2 Porque "porque lo digo yo" no escala ni enseña, mientras que decidir por las **fuerzas y trade-offs explícitos** (un ADR) se puede cuestionar con datos, forma a los demás para decidir igual de bien, y gana autoridad por criterio visible y no por título. El ADR es el vehículo de esa transparencia.
11.3 Team Topologies: no implementás cuatro tipos de equipo en un equipo de tres personas (overkill organizacional). ADRs: no escribís un ADR para elegir Prettier (decisión trivial, reversible). En ambos, ajustás la herramienta al tamaño/contexto del problema en vez de aplicar la máxima siempre.
11.4 Porque liderar (multiplicar gente: delegar, mentorear, decidir y registrar) y resolver problemas técnicos (lo del mejor IC) son **habilidades distintas** —ser excelente en una no implica la otra—. Con aprender TDD comparten que es un **oficio que se entrena**: los primeros pasos se sienten torpes (cuesta delegar, querés codear todo) y el payoff llega en el mediano plazo.
11.5 El liderazgo técnico aporta la mitad de **dirigir cómo un equipo entrega software** (decidir, medir, organizar, desarrollar gente); el resto del track aporta la mitad de **construir el software con calidad** (el cómo técnico). Juntas forman el perfil senior/Tech Lead/IA Engineer completo.

---

Con este módulo **cerrás el track Tech Lead** y, con él, la mitad del perfil senior que no es código puro. Recorriste el cambio de hacedor a multiplicador, las herramientas de decidir y dejar registro (**ADRs**), de medir y mejorar la entrega (**métricas DORA**, y el hallazgo de que velocidad y estabilidad no se contraponen), de organizar equipos y proteger su carga cognitiva (**Team Topologies**), el trabajo que no es código (review como liderazgo, mentoring, decir que no), la gestión de **deuda técnica** con datos, y el criterio sobre la **IA** en el rol. El hilo del temario entero queda completo: arrancaste pasando de Frontend a backend con Node, sumaste bases de datos, seguridad, testing, infra, cloud, arquitectura, el track de IA, y ahora las dos disciplinas que cierran el perfil senior —[testing](testing.md)/[TDD](tdd.md) (construir con calidad) y liderazgo técnico (dirigir la construcción)—. Desde acá el crecimiento ya no es de temario: es practicar el oficio, decidir en sistemas reales, y registrar lo que aprendas para multiplicar a otros.
