# Costo y economía del diseño: cuándo la restricción es la plata

**El costo como ciudadano de primera clase del diseño: unit economics, el mapa de gastos del cloud, egress y topología, storage tiering, el precio de cada "9" de disponibilidad, build vs buy · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es corto y transversal: no introduce un sistema nuevo, sino una **dimensión** que atraviesa todos los demás — **¿cuánto cuesta este diseño, y escala el costo mejor o peor que el tráfico?**. Es una señal clara de nivel: un junior diseña algo que funciona; un senior diseña algo que funciona y no se rompe; un **Staff** diseña algo que funciona, no se rompe **y no funde a la empresa cuando crece**. La pregunta *"¿cuánto sale esto al mes?"* casi nunca aparece en un tutorial y casi siempre aparece en una revisión de arquitectura real.

**Lo que asumimos.** La **matemática de servilleta** (QPS pico, storage/mes, ancho de banda, potencias de 10) de [Diseño de sistemas](system-design.md) M2 — acá la usamos para estimar **dólares**, no solo bytes. El **multi-región y la replicación** de [Replicación](replicacion.md) (su costo es el tema del módulo 5). El **egress y la topología de red** (anycast, CDN, cross-AZ) de [Networking](networking.md). El **storage y el ciclo de vida del dato** de [Datos a escala](datos-escala.md). Lo **gestionado vs propio** que ya cerró cada módulo del hub (y explícitamente [Seguridad de sistemas](seguridad-sistemas.md) M7). No re-explicamos eso: le ponemos **precio**.

> ⚠️ **Nota sobre datos volátiles.** Los precios de cloud (AWS/GCP/Azure ⚠️) cambian seguido y varían por región, volumen y compromiso (on-demand vs reserved vs spot). **Todos los números de este módulo son órdenes de magnitud aproximados de 2026 para fijar la intuición, no para cotizar** — verificá el precio exacto en la calculadora oficial antes de decidir. Lo que **no** cambia es la **estructura**: que el egress a internet sea ~10× el storage por GB, que cross-AZ no sea gratis, que cada "9" de disponibilidad cueste más que el anterior. Memorizá las **relaciones**, no los dígitos.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís código).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo. Y toda respuesta de diseño completa tiene **tres cosas**: una forma para el **happy path**, un **modelo de falla** y un **proceso de recuperación**. Acá la "falla" suele ser un **costo que se dispara**, no una caída — pero el marco aplica igual.

**Índice de módulos**
1. El costo como restricción de diseño: unit economics
2. El mapa de costos del cloud: dónde se va la plata
3. Egress y topología: el costo de mover datos
4. Storage tiering: no todo el dato vale lo mismo
5. El precio de la disponibilidad y de la coordinación
6. La economía del build vs buy: gestionado, serverless y reservado
7. El criterio: optimizar costo sin caer en la trampa

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El costo como restricción de diseño: unit economics

**Teoría.** En la mayoría de los ejercicios de diseño tratamos el costo como algo que se ve "al final, si sobra tiempo". En un sistema real es una **restricción de primera clase**, al mismo nivel que la latencia o la disponibilidad. Y la forma madura de pensarlo no es el costo **total** (un número grande que no dice nada) sino el costo **unitario**: **¿cuánto cuesta servir un request, un usuario activo, un tenant, un GB almacenado?** Eso son las **unit economics**.

¿Por qué unitario? Porque es lo único que te dice si el negocio **cierra** y si el diseño **escala**. Si servir a un usuario te cuesta $0.50/mes en infra y le cobrás $0.40, no tenés un problema de infra: tenés un negocio que **pierde más cuanto más crece**. Y al revés: un costo total alto puede estar perfecto si el costo por unidad es bajo y el ingreso por unidad lo cubre. El número que importa es el **margen por unidad**, no el gasto absoluto.

La segunda idea Staff: **cómo escala el costo con el tráfico.** Tres formas, de mejor a peor:

- **Sublineal:** el costo crece **más lento** que el tráfico (economías de escala: cachés que amortizan, recursos fijos que se diluyen, descuentos por volumen). Lo ideal.
- **Lineal:** el costo crece **igual** que el tráfico (2× usuarios → 2× costo). Aceptable y común; el margen por unidad se mantiene.
- **Superlineal:** el costo crece **más rápido** que el tráfico (2× usuarios → 3× costo). La bomba de tiempo. Suele venir de **coordinación** (cada nodo nuevo habla con todos los demás → O(n²)), de **fan-out** que se multiplica, o de **datos que se cruzan** cada vez más. Un diseño que parece barato con 1.000 usuarios puede ser insostenible con 1.000.000 **no porque sea más grande, sino porque escala mal**.

> La pregunta de diseño que casi nadie hace y un Staff hace siempre: **"este diseño, ¿cuánto cuesta por unidad, y esa curva es sublineal, lineal o superlineal a medida que crece?"**. Identificar el término superlineal **antes** de construirlo es lo que separa una arquitectura que escala de una que hay que reescribir cuando el tráfico llega.

> 🔥 **Falla en 4 capas — el costo escala peor que el ingreso.**
> (1) **Qué se rompe:** el sistema funciona, pero cada usuario nuevo cuesta más de lo que aporta; crecer **destruye** margen.
> (2) **Por qué a esta escala:** hay un término **superlineal** escondido (coordinación O(n²), fan-out, egress que se multiplica) que con pocos usuarios era ruido y ahora domina.
> (3) **Corto plazo:** medir el costo por unidad (atribución por feature/tenant), apagar lo que no rinde, poner *quotas*.
> (4) **Largo plazo:** rediseñar el término superlineal (cachear, batch, particionar para cortar el cruce de datos), y meter el costo unitario como métrica de primer nivel junto a latencia y errores.

La frase mental: **pensá el costo por UNIDAD (request/usuario/tenant/GB), no el total — es lo único que dice si el negocio cierra y si el diseño escala. Y preguntá cómo escala la curva: sublineal (ideal), lineal (ok) o superlineal (bomba: coordinación O(n²), fan-out, egress). El término superlineal hay que cazarlo antes de construirlo.**

**Ejercicios 1**
1.1 🔁 ¿Por qué el "costo por unidad" dice más que el "costo total"? ¿Qué te revela que el total esconde?
1.2 🧠 Un diseño cuesta $200/mes con 10.000 usuarios y $6.000/mes con 100.000 (10× usuarios, 30× costo). ¿Cómo escala el costo? ¿Qué tipo de causa buscarías?
1.3 🧠 Tu costo de infra por usuario es $0.50/mes y cobrás $0.40/mes. Con **100.000 usuarios**, ¿cuánto perdés al mes? Si duplicás la base a 200.000, ¿la pérdida sube, baja o se mantiene? ¿Por qué es urgente arreglarlo aunque el total de hoy parezca bajo?

---

## Módulo 2 — El mapa de costos del cloud: dónde se va la plata

**Teoría.** Antes de optimizar hay que saber **dónde** se gasta. En casi cualquier sistema cloud el gasto cae en cuatro baldes, y la intuición de la mayoría está mal calibrada sobre cuál pesa:

- **Cómputo** (EC2/Fargate/Lambda, contenedores, funciones ⚠️). Suele ser el balde grande y el más visible. Escala con CPU/RAM × tiempo encendido. Acá viven las decisiones de *serverless vs siempre-encendido* y *reserved vs on-demand* (módulo 6).
- **Storage** (S3, discos EBS, la base de datos ⚠️). Barato **por GB** pero crece monótonamente (los datos casi nunca se borran solos). El precio por GB cambia mucho según el **tier** (módulo 4).
- **Red / transferencia de datos** (egress). **El que más sorprende en la factura.** No es por su precio por GB (ni el más caro ni el más barato), sino porque es **invisible en el diseño**: nadie dibuja las flechas entre cajas pensando "esta flecha cuesta plata", pero **cada flecha que cruza una frontera de red se factura** (módulo 3).
- **Operaciones gestionadas** (RDS, DynamoDB, SQS, API Gateway, NAT Gateway ⚠️). Pagás por la **conveniencia** de no operarlo vos. A veces el cargo por-operación o por-hora de un servicio "chico" (un NAT Gateway, un API Gateway con mucho tráfico) sorprende más que el cómputo. Ojo con el NAT Gateway en particular: cobra un **cargo por GB procesado** (~$0.045/GB ⚠️) que **se suma** al egress que el tráfico ya paga, no lo reemplaza — un byte que sale a internet por NAT puede pagar processing **más** egress (~$0.045 + ~$0.09 ≈ ~$0.135/GB). Es el costo invisible apilado sobre otro costo invisible.

La regla de magnitud que conviene tener tatuada (⚠️ números 2026 aproximados, AWS, para **intuición**):

| Recurso | Orden de magnitud (USD) | Nota |
|---|---|---|
| **Ingress** (datos que entran) | **$0** | Entrar es gratis; salir se paga |
| Transferencia **misma AZ** | ~$0 / casi gratis | Mantené lo chatty acá |
| Transferencia **cross-AZ** | ~$0.01–0.02 /GB **por sentido** | Se factura ida **y** vuelta por separado; no es gratis aunque "sea interno" |
| Egress **cross-región** | ~$0.01–0.02 /GB | Replicar entre regiones cuesta (varía por par de regiones) |
| Egress **a internet** | ~$0.09 /GB (baja por volumen) | El que infla la factura |
| Storage **S3 Standard** | ~$0.023 /GB/mes | El "caliente" |
| Storage **archivo** (Glacier Deep Archive) | ~$0.001 /GB/mes | ~20× más barato (módulo 4) |

Lo clave de esta tabla no son los dígitos (que cambian) sino las **relaciones**: **ingress gratis, egress caro; salir a internet ~10× más caro por GB que guardarlo un mes; cross-AZ no es gratis aunque parezca "tráfico interno".** Esas relaciones se mantienen entre proveedores y entre años.

**El quinto balde que casi nadie presupuesta: la telemetría.** A escala, los **logs, métricas y trazas** ([Observabilidad](observabilidad.md)) se vuelven un costo de primer orden que sorprende tanto como el egress — y por la misma razón: nadie lo dibuja. El gasto viene de tres frentes: **ingest** (cada log/métrica/span que mandás a Datadog/CloudWatch/etc. se paga por volumen ⚠️), **retención** (guardarlos cuesta como cualquier storage, y los retenés "por las dudas"), y la **cardinalidad** de las métricas (cada combinación nueva de labels/dimensiones es una serie temporal más que se cobra — una métrica con un `user_id` de alta cardinalidad explota el costo). Y como la telemetría suele cruzar AZ hacia el colector, **también** paga egress cross-AZ. Las palancas son las mismas que en el resto del módulo: **sampling** (no guardes el 100% de las trazas — [Observabilidad](observabilidad.md)), **tiering/retención** de logs (los viejos a un tier frío o se borran, módulo 4), y **controlar la cardinalidad** (no metas IDs únicos como labels). Es el ejemplo perfecto del gasto invisible-que-escala-mal que este módulo enseña a cazar.

> El antipatrón #1 de costo: **diseñar como si la red fuera gratis.** El diagrama de cajas y flechas no muestra dónde están las fronteras de facturación (AZ, región, internet), así que el costo de red es **estructural e invisible** hasta que llega la factura. El módulo 3 es justo aprender a "ver" esas fronteras en el diagrama.

La frase mental: **cuatro baldes — cómputo (el grande y visible), storage (barato/GB pero monótono), red/egress (el que sorprende, porque es invisible en el diagrama), gestionados (pagás conveniencia). Relaciones que no cambian: ingress gratis, egress a internet ~10× el storage/GB, cross-AZ no es gratis. Antes de optimizar, mirá la factura desglosada: la intuición sobre cuál balde pesa suele estar mal.**

**Ejercicios 2**
2.1 🔁 Nombrá los cuatro baldes de costo de un sistema cloud típico. ¿Cuál suele ser el que más sorprende en la factura y por qué?
2.2 🔁 ¿Cuánto cuesta el **ingress** (datos que entran) en general? ¿Y por qué el egress es distinto?
2.3 🧠 Dos servicios muy "chatty" (se hablan miles de veces por segundo) están desplegados en **AZ distintas** "para alta disponibilidad". ¿Qué costo invisible aparece y cómo lo bajarías sin perder HA?
2.4 🧠 Tu factura de observabilidad (Datadog/CloudWatch) creció más rápido que el tráfico. ¿Cuáles son los tres frentes de gasto de la telemetría y qué palanca ataca cada uno?

---

## Módulo 3 — Egress y topología: el costo de mover datos

**Teoría.** Acá profundizamos el balde que más sorprende. La idea central: **mover datos cuesta, y cuánto cuesta depende de qué frontera cruzan.** Las fronteras, de gratis a cara: misma AZ (~gratis) → cross-AZ (~$0.01–0.02/GB) → cross-región (~$0.02/GB) → internet (~$0.09/GB ⚠️). Diseñar para costo es, en buena medida, **diseñar para que los datos crucen la menor cantidad de fronteras caras posible.**

Tres patrones que dominan el costo de egress:

- **Servicios chatty cross-AZ.** Si A llama a B miles de veces por segundo y están en AZ distintas, cada llamada paga egress cross-AZ en los **dos** sentidos. Multiplicá por millones de requests/día y el "tráfico interno gratis" que imaginabas es una línea gorda en la factura. Mitigación: **locality** — mantené juntas (misma AZ) las conversaciones de alta frecuencia, y reservá el cruce de AZ para lo que **necesita** redundancia. (Tensión real con la HA del módulo 5: la solución no es "todo en una AZ", es ser **deliberado** sobre qué cruza.)
- **CDN para cortar egress de origin.** Servir un asset 1.000.000 de veces desde tu origin = 1.000.000 × tamaño de egress a internet. Servirlo desde un **CDN** ([Networking](networking.md)) lo entrega desde el borde: pagás el egress del CDN (más barato por volumen) y, sobre todo, **el origin egresa una sola vez** y el resto sale del caché. El CDN no es solo latencia: es una **palanca de costo de egress** enorme para contenido cacheable.
- **Data gravity.** Los datos "pesan": es caro moverlos, así que el **cómputo tiende a migrar hacia donde están los datos**, no al revés. Procesá cerca del dato (misma región que la base / el lake) en vez de traerte terabytes cruzando regiones para procesarlos del otro lado. Mover el código (KB) es gratis; mover el dato (TB) cuesta una fortuna.

> 🔥 **Falla en 4 capas — la factura de egress se dispara.**
> (1) **Qué se rompe:** el costo mensual sube fuerte sin que el tráfico de negocio crezca en proporción; el desglose muestra "Data Transfer" como el balde top.
> (2) **Por qué a esta escala:** un patrón chatty cross-AZ/región, o servir contenido pesado directo del origin, multiplica el egress por cada request. Era ruido con poco tráfico; ahora domina.
> (3) **Corto plazo:** meter un **CDN** delante del contenido cacheable, colocalizar (misma AZ) los servicios chatty, comprimir payloads.
> (4) **Largo plazo:** diseñar la topología por **locality** (qué cruza qué frontera es una decisión, no un accidente), procesar **cerca del dato** (data gravity), y revisar las flechas del diagrama preguntando "¿esta cruza una frontera de facturación?".

La frase mental: **mover datos cuesta según la frontera que cruzan: misma AZ ~gratis < cross-AZ < cross-región < internet. Diseñar para costo = minimizar cruces caros. Los tres grandes: servicios chatty cross-AZ (colocalizá lo de alta frecuencia), CDN para que el origin egrese una sola vez, y data gravity (llevá el cómputo al dato, no el dato —TB— al cómputo). Las flechas del diagrama cuestan plata; aprendé a ver las fronteras.**

**Ejercicios 3**
3.1 🔁 Ordená de más barato a más caro: egress a internet, transferencia misma AZ, cross-región, cross-AZ. ¿Qué principio de diseño se deriva de ese orden?
3.2 🧠 ¿Por qué un CDN es una palanca de **costo** (no solo de latencia)? ¿Qué le pasa al egress del origin cuando lo ponés?
3.3 🧠 Explicá "data gravity". ¿Por qué movés el cómputo hacia el dato y no al revés, y qué tiene que ver con el costo de egress?
3.4 ✍️ Implementá en TypeScript un `costoEgresoMensualUSD(trafico, precios)` que estime el costo de egress **a internet** dado el `qpsPromedio` y los `bytesPorRespuesta`. (La solución está al final; tiene que compilar en `--strict`. Pista: req/s → req/mes, bytes → GB de facturación con 10⁹.)

---

## Módulo 4 — Storage tiering: no todo el dato vale lo mismo

**Teoría.** El storage es barato por GB, pero crece **para siempre** (los datos casi nunca se borran), así que a escala se vuelve un balde grande igual. La palanca clave: **no todos los datos se acceden con la misma frecuencia, así que no tienen por qué costar lo mismo.** Los proveedores ofrecen **tiers** (clases de almacenamiento) que negocian **precio de storage** contra **costo y latencia de recuperación** (⚠️ nombres/precios de AWS S3, aproximados 2026):

- **Hot** (S3 Standard, ~$0.023/GB/mes): acceso inmediato y frecuente, sin cargo de recuperación. Para lo que se lee seguido.
- **Warm** (S3 Standard-IA / One Zone-IA, ~$0.0125/GB/mes): más barato de guardar, pero **pagás por recuperar** (cargo por GB leído) y conviene solo si accedés poco. Para datos que querés a mano pero rara vez tocás.
- **Cold / Archivo** (Glacier Flexible ~$0.0036, Glacier Deep Archive ~$0.001/GB/mes): **~20× más barato** que hot, pero la recuperación tarda **minutos a horas** y tiene cargo. Para retención que casi nunca leés (compliance, backups viejos, logs de hace un año).

El trade-off es siempre el mismo: **cuanto más barato el storage, más caro y lento recuperar.** El error clásico en ambas direcciones: dejar **todo** en hot (pagás de más por datos que nadie mira) o mandar a archivo algo que después necesitás seguido (pagás recuperaciones carísimas y sufrís la latencia de horas).

La herramienta para no decidirlo a mano: **lifecycle policies** (políticas de ciclo de vida). El dato **se enfría con el tiempo**: un log de hoy se consulta mucho, el de hace 90 días casi nunca, el de hace un año jamás (salvo auditoría). Una política mueve automáticamente los objetos entre tiers por edad (ej.: Standard 0–30 días → IA 30–90 → Glacier 90–365 → borrar/Deep Archive a los 365). Esto conecta con la **retención** de [Datos a escala](datos-escala.md): definir *cuánto tiempo y en qué tier* vive cada tipo de dato es una decisión de diseño con impacto directo en la factura.

> El matiz que se olvida: el tier barato **no es gratis de usar**, es gratis de **guardar**. Standard-IA y Glacier cobran por **recuperación** (y a veces un **mínimo de permanencia** — borrar antes de X días igual te cobra el período). Mandar a IA datos que resultan accederse seguido puede salir **más caro** que dejarlos en Standard. La regla: **el tier sigue al patrón de acceso real**, que hay que medir, no suponer.

La frase mental: **el storage crece para siempre; la palanca es que no todo el dato se accede igual. Tiers = precio de guardar ↔ costo+latencia de recuperar: hot (caro guardar, gratis leer) → warm/IA → cold/archivo (~20× más barato guardar, recuperación cara y lenta). Automatizá con lifecycle policies (el dato se enfría con la edad). Ojo: el tier frío cobra por recuperar y por permanencia mínima — seguí el patrón de acceso REAL, no el supuesto.**

**Ejercicios 4**
4.1 🔁 ¿Qué negocia un tier de storage más barato (tipo Glacier) a cambio de su menor precio por GB?
4.2 🧠 ¿Qué hace una *lifecycle policy* y por qué encaja con la idea de que "el dato se enfría con el tiempo"? Dá un ejemplo de política para logs.
4.3 🧠 Un equipo movió todos sus datos a Standard-IA "para ahorrar" y la factura **subió**. ¿Qué pudo pasar?

---

## Módulo 5 — El precio de la disponibilidad y de la coordinación

**Teoría.** Acá le ponemos número a algo que en los módulos de distribuido tratamos como propiedad técnica: **la disponibilidad y la consistencia fuerte cuestan dinero, y el costo crece más rápido que la garantía.** Es el lado económico del CAP/PACELC de [Replicación](replicacion.md) y del consenso de [Consenso](consenso.md).

**Cada "9" de disponibilidad cuesta más que el anterior.** Pasar de 99% (~3.65 días de caída/año) a 99.9% (~8.8 h) a 99.99% (~52 min) a 99.999% (~5 min) no es un costo lineal: cada nueve extra exige **redundancia, multi-AZ, multi-región, failover automático, más on-call** — y cada uno cuesta desproporcionadamente más que el anterior, mientras te ahorra cada vez **menos** minutos de caída. La pregunta Staff no es "¿lo querés muy disponible?" (todos dirían que sí) sino **"¿cuánto te cuesta un minuto de caída, y cuánto cuesta el nueve que lo evita?"** — si el nueve cuesta más que las caídas que previene, estás sobre-invirtiendo.

**La redundancia multiplica la infra.** Multi-AZ con réplicas activas ≈ 2× la infra de cómputo/base; multi-región activa-activa puede ser 2–3× **más** el costo de egress de sincronizar regiones (módulo 3). Es real y a veces necesario — pero es una **decisión de costo**, no un default gratis. "Lo ponemos multi-región por las dudas" puede triplicar la factura para protegerte de un evento que quizá nunca pase.

**La consistencia fuerte cuesta latencia y plata.** Un quorum, un commit sincrónico cross-región, una transacción distribuida: cada uno paga **round-trips de coordinación** (que cross-región son 50–150 ms cada uno — [Networking](networking.md)) y más infra. Por eso el hub repite que la consistencia eventual es más barata y más disponible **cuando el dominio la tolera**: elegir eventual donde se puede es también una **decisión de costo**.

**El sobre-aprovisionamiento.** Reservar capacidad para el pico 24/7 cuando el pico dura 2 horas al día significa pagar por recursos **ociosos** el 90% del tiempo. Es el costo de no tener elasticidad — y la razón por la que el autoscaling y el serverless (módulo 6) existen.

> 🔥 **Falla en 4 capas — multi-región "por las dudas" que duplica la factura.**
> (1) **Qué se rompe:** el costo se va a 2–3× por infra redundante + egress de sincronización, para una garantía de HA que el negocio no necesitaba a ese nivel.
> (2) **Por qué a esta escala:** activa-activa cross-región paga doble/triple cómputo **más** el egress de mantener las regiones en sync, y todo eso escala con el tráfico.
> (3) **Corto plazo:** medir cuánto cuesta realmente un minuto de caída vs el costo del setup multi-región; bajar a multi-AZ (mucho más barato) si el SLA lo permite.
> (4) **Largo plazo:** dimensionar la disponibilidad al **costo real de la caída** (no al "9" más alto posible), usar consistencia eventual donde el dominio la tolera, y autoscaling en vez de reservar para el pico 24/7.

La frase mental: **la disponibilidad y la consistencia fuerte cuestan, y no linealmente: cada "9" extra cuesta más y ahorra menos minutos. Redundancia multi-AZ ≈ 2× infra; multi-región activa-activa 2–3× + egress de sync. Consistencia fuerte = round-trips de coordinación (caros cross-región) + más infra. La pregunta Staff: "¿cuánto cuesta un minuto de caída vs el nueve que lo evita?" — dimensioná al costo real, no al "9" máximo.**

**Ejercicios 5**
5.1 🔁 ¿Por qué se dice que "cada nueve de disponibilidad cuesta más que el anterior"? ¿Qué exige cada nueve extra?
5.2 🧠 ¿Cuál es la pregunta que un Staff hace antes de aceptar "lo queremos con 99.999%"? ¿Por qué desactiva el "todos quieren más disponibilidad"?
5.3 🧠 Conectá la consistencia eventual con el costo: ¿por qué elegir eventual (donde el dominio la tolera) es también una decisión de plata, no solo de latencia?

---

## Módulo 6 — La economía del build vs buy: gestionado, serverless y reservado

**Teoría.** Tres decisiones de costo que aparecen en casi todo diseño, y en las tres la trampa es mirar **un solo número**.

**Gestionado vs self-hosted.** El servicio gestionado (RDS en vez de tu Postgres, SQS en vez de tu RabbitMQ ⚠️) tiene un precio por hora **visible** que parece caro comparado con "lo corro yo en una VM más barata". La comparación honesta no es precio-de-instancia vs precio-de-servicio: es el **costo total de propiedad (TCO)**, que en el self-hosted incluye lo **invisible** — el **tiempo de ingeniería** de instalarlo/parchearlo/escalarlo, el **on-call** de cuando se rompe a las 3am, el **riesgo** de operarlo mal. Para la mayoría de los equipos, el gestionado es **más barato** una vez que contás el costo de la gente. Se justifica self-hostear cuando el volumen es tan grande que el margen del proveedor supera el costo de un equipo dedicado, o cuando tenés un requisito que el gestionado no cubre. (Es el mismo criterio "no lo construyas vos" de [Seguridad](seguridad-sistemas.md) M7, ahora en dólares.)

**Serverless vs siempre-encendido.** Serverless (Lambda, Fargate, DynamoDB on-demand ⚠️) = **pagás por uso**, $0 cuando no corre. Siempre-encendido (una instancia 24/7) = **costo fijo** corra o no. Hay un **punto de equilibrio (break-even)**:

- Tráfico **bajo o spiky** (picos cortos, mucho idle) → **serverless** gana: no pagás el idle, y el pico lo absorbe la elasticidad.
- Tráfico **alto y sostenido** → la **instancia dedicada/reservada** gana: a volumen, el precio por-request del serverless supera el costo fijo amortizado de tener la máquina encendida.

El error es elegir por moda en vez de por la **curva de tráfico**: serverless para un servicio con carga alta y constante puede salir varias veces más caro; una instancia 24/7 para algo que se usa 1 hora al día es tirar plata en idle.

**On-demand vs reserved vs spot.** Para el cómputo siempre-encendido hay tres precios del **mismo recurso**: **on-demand** (flexible, el más caro), **reserved / savings plans** (te comprometés a 1–3 años → ~30–72% más barato ⚠️, llegando al techo con 3 años pagados por adelantado, para carga **base predecible**), y **spot** (capacidad sobrante con hasta ~90% de descuento ⚠️ pero **te la pueden quitar con poco aviso** → solo para trabajo **tolerante a interrupción**: batch, workers de cola, procesamiento reanudable). En producción, el spot no se usa "crudo": se **mitiga la interrupción** diversificando entre varios *pools* de capacidad (distintos tipos de instancia/AZ, para que no te queden sin todos a la vez), con **fallback automático a on-demand** cuando no hay spot, y reaccionando al **aviso de terminación (~2 min)** drenando la instancia (capacity rebalancing). La estrategia madura **combina**: reserved para la base estable, on-demand para la variación esperada, spot (diversificado, con fallback) para el batch interrumpible.

> 🔥 **Falla en 4 capas — serverless elegido por moda en carga alta y constante.**
> (1) **Qué se rompe:** un servicio con tráfico alto y sostenido en serverless cuesta varias veces lo que costaría en instancias reservadas.
> (2) **Por qué a esta escala:** el precio por-invocación del serverless es genial mientras hay idle, pero a volumen sostenido **supera** el costo fijo amortizado de una máquina encendida — la curva cruza el break-even.
> (3) **Corto plazo:** medir requests/mes reales y compararlos con el umbral de break-even; mover el servicio caliente a instancias.
> (4) **Largo plazo:** elegir el modelo de cómputo por la **curva de tráfico** (spiky→serverless, sostenido→reservado), combinar reserved+on-demand+spot, y revisar el mix cuando el patrón cambia.

La frase mental: **en build-vs-buy mirá el costo TOTAL, no un solo número. Gestionado vs propio: el TCO del propio incluye gente/on-call/riesgo (invisibles) → gestionado suele ganar. Serverless vs dedicado: hay break-even — spiky/bajo→serverless (no pagás idle), alto/sostenido→reservado (el por-request supera el fijo amortizado). On-demand/reserved/spot: combiná — reserved la base, spot el batch interrumpible. Elegí por la curva de tráfico, no por moda.**

**Ejercicios 6**
6.1 🔁 ¿Qué costos "invisibles" entran en el TCO de self-hostear algo, que el precio de la instancia no muestra?
6.2 🧠 ¿En qué forma de la curva de tráfico gana serverless y en cuál gana una instancia dedicada? ¿Por qué existe un punto de equilibrio?
6.3 🧠 ¿Para qué tipo de carga es apropiado **spot** y para cuál no? ¿Por qué?
6.4 ✍️ Implementá un `umbralReqMensual(serverless, dedicada)` en TypeScript que, dado el precio serverless por millón de requests y el costo fijo mensual de la opción dedicada, devuelva **a partir de cuántos requests/mes** conviene la dedicada. (La solución está al final; tiene que compilar en `--strict`.)

---

## Módulo 7 — El criterio: optimizar costo sin caer en la trampa

**Teoría.** El cierre, como en cada módulo del hub: el costo es un **eje a optimizar con criterio**, no a minimizar siempre. Y la disciplina que lo gobierna se parece muchísimo a la de **performance** ([Observabilidad](observabilidad.md)): **medí antes de optimizar.**

**Optimización prematura de costo = otra forma de optimización prematura.** Pasar dos sprints ahorrando $50/mes en un sistema que factura millones es regalar tiempo de ingeniería (que cuesta **mucho** más que $50/mes) por un ahorro irrelevante. El costo de **optimizar** (horas de gente, complejidad agregada, riesgo de romper algo) entra en la ecuación: optimizá donde el gasto es **material** y escala **mal**, ignorá el resto. Un Staff sabe **qué no vale la pena optimizar**.

**Medí, no supongas — el equivalente del profiling.** Igual que no optimizás performance sin un profiler, no optimizás costo sin el **desglose de la factura** (cost explorer, tags por servicio/equipo/feature). La intuición sobre dónde se va la plata casi siempre está mal (el módulo 2): el balde que creés que pesa no es el que pesa. **Atribución de costo** (taggear recursos por equipo/producto) es a FinOps lo que el profiler es a performance: te dice **dónde** mirar.

**FinOps: el costo es responsabilidad de ingeniería, no solo de finanzas.** La práctica madura (FinOps ⚠️) pone visibilidad de costo en manos de quien diseña: dashboards de gasto por servicio, alertas de anomalía (un gasto que se dispara avisa como avisa un error), y la noción de que **una decisión de arquitectura es una decisión de costo**. No es "el equipo de finanzas nos reta"; es la misma cultura de ownership que la observabilidad y el on-call.

**Y la trampa simétrica: sub-optimizar también cuesta.** Así como sobre-optimizar quema ingeniería, **ignorar** el costo hasta que la factura asusta lleva a recortes de pánico que dañan la confiabilidad. El punto medio: el costo como **métrica de primera clase** desde el diseño (unit economics del módulo 1), revisada periódicamente, optimizada cuando es material — ni obsesión ni ceguera.

> **El cierre de criterio — cuándo optimizar costo y cuándo no.** ¿El gasto es chico o escala bien (sublineal/lineal) y el ahorro posible es menor al costo de ingeniería de lograrlo? **No lo toques.** ¿El gasto es material, escala mal (superlineal), o un balde domina la factura sin razón de negocio? **Ahí sí**, y empezá por **medir** (desglose + atribución), no por suponer. La pregunta Staff: *"¿el ahorro de optimizar esto supera lo que cuesta optimizarlo — y este gasto escala con el negocio o peor?"*.

La frase mental de cierre: **el costo se optimiza con criterio, no se minimiza siempre. Medí antes de optimizar (el desglose de la factura es el profiler del costo; tu intuición sobre el balde grande está mal). Optimización prematura de costo = quemar ingeniería cara por un ahorro chico → optimizá solo lo material que escala mal. FinOps: el costo es ownership de ingeniería (una decisión de arquitectura es una decisión de costo), con visibilidad y alertas como con los errores. Ni obsesión ni ceguera.**

**Ejercicios 7**
7.1 🔁 ¿Por qué "optimización prematura de costo" es una trampa? ¿Qué costo se suele olvidar al hacer la cuenta del ahorro?
7.2 🧠 ¿Cuál es el equivalente del "profiler" en la optimización de costos, y por qué hace falta antes de tocar nada?
7.3 🧠 Resumí la pregunta que decide si un gasto vale la pena optimizar.

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** El **costo total** es un número grande que no dice si el negocio cierra ni si el diseño escala: un total alto puede estar perfecto (si el costo por unidad es bajo y el ingreso lo cubre) y un total bajo puede esconder una bomba. El **costo por unidad** (request/usuario/tenant/GB) revela el **margen por unidad** —lo único que dice si servir a un cliente más suma o resta plata— y, comparado entre escalas, revela **cómo escala la curva** (sublineal/lineal/superlineal). El total esconde justamente esas dos cosas: el margen y la forma de la curva.

**1.2** 10× usuarios → 30× costo: el costo escala **superlinealmente** (crece más rápido que el tráfico). Buscaría una causa de tipo **coordinación o cruce de datos que crece con el cuadrado** (cada nodo/usuario interactúa con todos los demás, O(n²)), **fan-out** que se multiplica por usuario, o **egress** que crece más que linealmente porque cada usuario nuevo hace cruzar más datos entre fronteras. El síntoma (costo superlineal) casi siempre apunta a un término de coordinación/fan-out/egress que con pocos usuarios era ruido y ahora domina — hay que identificarlo y rediseñarlo (cachear, batchear, particionar para cortar el cruce) antes de que el tráfico llegue.

**1.3** El margen por unidad es −$0.10/usuario/mes (cobrás $0.40, gastás $0.50). Con **100.000 usuarios** perdés 100.000 × $0.10 = **$10.000/mes**; al duplicar a **200.000** la pérdida **sube** a **$20.000/mes** — crece **linealmente con la base**, así que cada usuario nuevo te hace perder más. Es urgente porque el problema **no es el total de hoy, es la dirección de la curva**: con margen negativo por unidad, **crecer —que debería ser bueno— te funde más rápido**. El total bajo solo significa que todavía tenés pocos usuarios; el modelo está roto en la unidad y el éxito (más usuarios) **amplifica** la pérdida. Hay que arreglar las unit economics **antes** de escalar: o baja el costo por usuario (optimizar infra, cachear, tier más barato) o sube el precio. Escalar con margen negativo es acelerar hacia el precipicio.

**2.1** Los cuatro baldes: **cómputo** (instancias/contenedores/funciones — CPU/RAM × tiempo), **storage** (S3/discos/base — por GB, crece monótono), **red / transferencia de datos (egress)**, y **operaciones gestionadas** (RDS/DynamoDB/SQS/NAT Gateway — pagás conveniencia). El que **más sorprende** es el de **red/egress**, no porque su precio por GB sea el más alto, sino porque es **invisible en el diagrama**: nadie dibuja las flechas entre servicios pensando que cada una que cruza una frontera (AZ/región/internet) se factura, así que el costo aparece recién en la factura.

**2.2** El **ingress** (datos que entran a la nube) suele ser **gratis** ($0) — los proveedores no cobran por meter datos. El **egress** (datos que salen) es lo que se cobra, y mucho cuando sale **a internet** (~$0.09/GB ⚠️). La asimetría es deliberada: entrar es gratis (te incentivan a traer tus datos), salir cuesta (lock-in suave: sacar tus datos o servirlos a usuarios cuesta plata). Por eso el diseño de costo se centra en el egress, no en el ingress.

**2.3** Aparece el **egress cross-AZ**: cada una de las miles de llamadas/segundo entre A y B cruza la frontera de AZ y se factura (~$0.01–0.02/GB, y en **los dos sentidos**), así que el "tráfico interno" que creías gratis es una línea gorda. Para bajarlo **sin perder HA**: colocalizá las **conversaciones de alta frecuencia** en la **misma AZ** (locality) y mantené la redundancia cross-AZ para lo que de verdad la necesita — por ejemplo, instancias de A y B **en cada AZ** que se hablen **dentro** de su AZ (mismo-AZ ~gratis), con failover cross-AZ solo si una cae. La HA viene de tener réplicas en varias AZ, no de que **cada request** cruce AZ.

**2.4** Los **tres frentes** de gasto de la telemetría y su palanca: (1) **Ingest** — pagás por **volumen** de logs/métricas/spans enviados → palanca: **sampling** (no guardar el 100% de las trazas; muestrear lo normal y conservar lo anómalo/errores). (2) **Retención** — guardar la telemetría es storage que crece → palanca: **tiering/expiración** (logs viejos a un tier frío o borrarlos, módulo 4; retené solo lo que vas a consultar). (3) **Cardinalidad** — cada combinación nueva de labels es una serie temporal más que se cobra → palanca: **controlar la cardinalidad** (no usar IDs únicos como `user_id`/`request_id` de label; acotar las dimensiones). Bonus: si la telemetría cruza AZ hacia el colector, también paga egress cross-AZ (colocalizar el agente). La factura creció más que el tráfico casi siempre por **cardinalidad** que explotó o **retención** sin política.

**3.1** De más barato a más caro: **transferencia misma AZ** (~gratis) < **cross-AZ** (~$0.01–0.02/GB) < **cross-región** (~$0.02/GB) < **egress a internet** (~$0.09/GB) ⚠️. El principio que se deriva: **diseñá para que los datos crucen la menor cantidad de fronteras caras posible** — mantené lo chatty/de alta frecuencia dentro de la misma AZ, reservá el cruce de AZ/región para lo que necesita redundancia o alcance global, y poné un CDN para que el contenido salga del borde y no del origin.

**3.2** Porque sin CDN, servir un asset N veces = **N × egress a internet desde tu origin**. Con CDN, el origin entrega el asset **una sola vez** (o pocas, al popular el caché de cada PoP) y las siguientes N−1 entregas salen del **borde**, donde el egress es más barato por volumen. El egress del **origin** cae drásticamente (de N a ~1 por PoP), que es el caro de evitar. Por eso el CDN no es solo latencia (servir cerca del usuario): es una **palanca de costo de egress** — mueve el grueso del tráfico de salida desde tu origin caro al CDN barato, y además esconde el origin.

**3.3** **Data gravity** = los datos "pesan": son caros y lentos de mover (mover TB cruzando regiones cuesta egress y tiempo), así que **atraen** al cómputo hacia donde están, como la gravedad. Movés el **cómputo hacia el dato** porque el código pesa KB (mover una imagen de contenedor o desplegar una función a la región del dato es trivial y barato) mientras que el dato pesa TB (traértelo cruzando regiones es egress cross-región carísimo + latencia). Con el costo: procesar **en la región del dato** evita el egress de exportar terabytes; procesar del otro lado paga ese egress en cada corrida. Por eso los lakes/bases y el procesamiento conviven en la misma región.

**3.4**
```ts
interface PerfilTrafico {
  qpsPromedio: number;       // requests por segundo, promedio del mes
  bytesPorRespuesta: number; // tamaño medio de la respuesta servida
}

interface PreciosEgreso {
  precioInternetPorGB: number; // USD/GB de egress a internet (~0.09 ⚠️)
}

function costoEgresoMensualUSD(
  trafico: PerfilTrafico,
  precios: PreciosEgreso,
): number {
  const SEGUNDOS_POR_MES = 30 * 24 * 3600; // ~2.592M
  const bytesPorMes =
    trafico.qpsPromedio * trafico.bytesPorRespuesta * SEGUNDOS_POR_MES;
  // El egress se factura en GB DECIMAL (10^9 bytes), no GiB (2^30).
  const gbPorMes = bytesPorMes / 1e9;
  return gbPorMes * precios.precioInternetPorGB;
}
```
Idea: es napkin math de [system-design](system-design.md) M2 con precio. **req/s → req/mes** (× segundos del mes), **req → bytes** (× tamaño de respuesta), **bytes → GB** (÷ 10⁹, ojo: el billing usa GB decimal, no GiB binario), **GB → USD** (× precio/GB). Ejemplo, siguiendo la cadena de la función: 1.000 qps × 50 KB/respuesta = 50 MB/s ≈ 180 GB/hora ≈ 4.320 GB/día ≈ **~129.6 TB/mes** × $0.09 ≈ **~$11.700/mes** de egress (≈ $11.664 exactos, justo lo que devuelve la función) — el tipo de número que justifica un CDN. Compila en `--strict`.

**4.1** Negocia **costo y latencia de recuperación**: a cambio de un precio por GB mucho más bajo (~20× en Glacier Deep Archive), la recuperación **tarda minutos a horas** (no es acceso inmediato) y **tiene un cargo por GB recuperado** (y a veces un **mínimo de permanencia**: borrar antes de X días igual te cobra el período). O sea, el tier frío es barato de **guardar** pero caro y lento de **leer** — solo conviene para datos que casi nunca tocás.

**4.2** Una **lifecycle policy** mueve objetos automáticamente entre tiers (y eventualmente los borra) **según reglas, típicamente por edad**, sin que nadie lo haga a mano. Encaja con "el dato se enfría con el tiempo" porque el patrón de acceso cae con la edad: lo de hoy se lee mucho, lo de hace meses casi nunca. Ejemplo para **logs**: Standard los primeros 30 días (debugging/incidentes recientes) → Standard-IA de 30 a 90 (consulta ocasional) → Glacier de 90 a 365 (retención/compliance, casi nunca leídos) → borrar (o Deep Archive) pasado el año. Cada tramo paga lo justo para su frecuencia de acceso real.

**4.3** Standard-IA es más barato de **guardar** pero cobra por **recuperación** (cargo por GB leído) y tiene **mínimo de permanencia** (~30 días) y, según el caso, un **mínimo de tamaño** por objeto. Si esos datos en realidad **se accedían seguido**, los cargos de recuperación pudieron **superar** el ahorro de storage; o si eran **muchos objetos chicos** o se **borraban/sobrescribían antes** del mínimo de permanencia, los mínimos penalizaron. Moraleja del módulo: **el tier tiene que seguir el patrón de acceso real (medido)**; mover a IA datos calientes o de vida corta sale **más caro** que dejarlos en Standard.

**5.1** Porque cada nueve adicional reduce el **downtime permitido** en un orden de magnitud (99%≈3.65 días/año → 99.9%≈8.8 h → 99.99%≈52 min → 99.999%≈5 min) y para lograrlo hay que agregar mecanismos **cada vez más caros**: redundancia, multi-AZ, luego multi-región, failover automático probado, más on-call, eliminación de cada SPOF restante. Cada nueve exige **más** infra/ingeniería que el anterior y te ahorra **menos** minutos absolutos de caída → el costo marginal sube mientras el beneficio marginal baja. Por eso la disponibilidad "infinita" no existe a un costo razonable.

**5.2** La pregunta: **"¿cuánto nos cuesta realmente un minuto (o una hora) de caída, y cuánto cuesta el nueve que lo evita?"**. Desactiva el "todos quieren más disponibilidad" porque convierte un deseo abstracto ("que no se caiga nunca") en una **comparación económica concreta**: si el nueve extra cuesta más que las caídas que previene (dado el costo real de la caída para *este* negocio), estás **sobre-invirtiendo**. Un blog y un sistema de pagos tienen costos de caída distintos por varios órdenes de magnitud → merecen niveles de disponibilidad (y de gasto) distintos. El número, no la intención, decide.

**5.3** Porque la consistencia **fuerte** se paga con **round-trips de coordinación** (quórum, commit sincrónico, transacción distribuida) que cross-región cuestan 50–150 ms cada uno y exigen **más infra y réplicas sincronizadas** — más latencia *y* más plata. La consistencia **eventual** evita esa coordinación: cada réplica responde local y reconcilia después, así que es **más barata y más disponible**. Elegir eventual donde el dominio la tolera (un contador de likes, un feed) en vez de fuerte (un saldo bancario) no es solo bajar latencia: es **no pagar** la coordinación cara. Sobre-pedir consistencia fuerte donde no hace falta es gastar de más, igual que pedir un nueve de más.

**6.1** El precio de la instancia muestra solo el **hierro**; el TCO del self-hosted suma lo **invisible**: el **tiempo de ingeniería** de instalar, configurar, parchear, actualizar y escalar el servicio; el **on-call** y el costo operativo de responder cuando se rompe (incluida la madrugada); el **riesgo** de operarlo mal (una mala config de backups/seguridad que termina en incidente); y el **costo de oportunidad** (esa ingeniería no está construyendo producto). Sumado todo, la VM "más barata" suele salir **más cara** que el servicio gestionado una vez que contás la gente — por eso el gestionado gana salvo a volúmenes enormes o requisitos que no cubre.

**6.2** **Serverless** gana con tráfico **bajo o spiky** (picos cortos y mucho idle): como pagás **por uso** y $0 cuando no corre, no malgastás en idle y el pico lo absorbe la elasticidad. La **instancia dedicada/reservada** gana con tráfico **alto y sostenido**: su costo fijo se **amortiza** entre muchísimos requests, y a ese volumen el precio **por-invocación** del serverless termina **superando** el fijo de tener la máquina prendida. Existe un **punto de equilibrio** porque uno es **costo variable puro** (crece con cada request) y el otro **costo fijo** (constante): las dos rectas se cruzan en un volumen de requests; debajo conviene el variable (serverless), arriba el fijo (dedicada).

**6.3** **Spot** sirve para trabajo **tolerante a interrupción**: batch, procesamiento de colas, jobs reanudables, render, ETL — cosas que si se cortan a mitad **se reintentan** sin drama. No sirve para lo que necesita estar **siempre disponible y sin cortes**: una API de cara al usuario, una base de datos primaria, un servicio con estado en memoria que no tolera que lo maten. La razón: spot es capacidad **sobrante** del proveedor con ~70–90% de descuento ⚠️, pero **te la puede quitar con poco aviso** cuando la necesita para clientes on-demand. El descuento es el precio de aceptar esa interrupción → solo lo aceptás donde la interrupción es barata.

**6.4**
```ts
interface OpcionServerless {
  precioPorMillonReq: number; // USD por 1.000.000 de requests ⚠️
}

interface OpcionDedicada {
  costoFijoMensualUSD: number; // instancia(s) encendidas 24/7, corra o no
}

// Requests/mes a partir de los cuales la dedicada sale más barata que serverless.
function umbralReqMensual(
  serverless: OpcionServerless,
  dedicada: OpcionDedicada,
): number {
  const precioPorReqServerless = serverless.precioPorMillonReq / 1_000_000;
  // Igualamos costos:  costoFijo = reqs * precioPorReq  →  reqs = costoFijo / precioPorReq
  return dedicada.costoFijoMensualUSD / precioPorReqServerless;
}
```
Idea: el serverless es **costo variable** (`reqs × precioPorReq`) y la dedicada es **costo fijo** (`costoFijoMensualUSD`). El break-even es donde se igualan → `reqs = costoFijo / precioPorReq`. **Debajo** de ese umbral, serverless es más barato (pagás poco porque hay poco tráfico); **arriba**, la dedicada gana (su fijo ya se amortizó y el variable del serverless lo superó). Ejemplo: serverless a $0.20/millón → $0.0000002/req; dedicada a $50/mes → umbral = 50 / 0.0000002 = **250.000.000 req/mes** (~96 req/s sostenidos): por debajo de eso, serverless; por encima, instancia. Compila en `--strict`. (Simplifica: ignora egress, requests de distinto costo y el costo operativo de cada modelo — para una decisión real, sumá esos términos.)

**7.1** Es una trampa porque gastás un recurso **caro** (tiempo de ingeniería) para ahorrar uno que puede ser **irrelevante**: dos semanas de un equipo optimizando para bajar $50/mes es regalar miles de dólares de salario por un ahorro insignificante. El costo que se olvida al calcular el ahorro es justamente el **costo de optimizar**: las horas de ingeniería, la **complejidad** que agregás (que después hay que mantener) y el **riesgo** de romper algo estable. La cuenta honesta es *ahorro esperado − costo de lograrlo (y de mantenerlo)*; si da negativo o marginal, no se toca. Optimizá donde el gasto es **material y escala mal**, no en todos lados.

**7.2** El equivalente del profiler es el **desglose/atribución de la factura**: el cost explorer del proveedor + **tags** por servicio, equipo, producto o feature que dicen **dónde** se va realmente la plata. Hace falta antes de tocar nada por la misma razón que no optimizás performance sin profiler: la **intuición sobre el balde grande casi siempre está mal** (módulo 2) — vas a "optimizar" lo que creés caro y resulta que el gasto estaba en otro lado (un NAT Gateway, egress invisible, datos calientes en el tier equivocado). Medir primero evita gastar ingeniería optimizando lo que no mueve la aguja.

**7.3** **"¿El ahorro de optimizar esto supera lo que cuesta optimizarlo (ingeniería + complejidad + riesgo), y este gasto escala con el negocio o peor?"**. Si el gasto es chico o escala bien (sublineal/lineal) y el ahorro no paga el esfuerzo → **no lo toques** (sería optimización prematura). Si el gasto es **material**, **domina** la factura sin razón de negocio, o escala **superlinealmente** → vale optimizar, y se empieza por **medir** (desglose + atribución), no por suponer. Ni obsesión (sobre-optimizar quema ingeniería) ni ceguera (ignorarlo lleva a recortes de pánico que dañan la confiabilidad).

---

> **Para seguir.** Las fuentes de verdad: la **calculadora de precios** y la doc de billing de tu proveedor (**AWS Pricing Calculator / Cost Explorer**, GCP, Azure ⚠️ — los números de este módulo son intuición, no cotización), el **Well-Architected Framework** (pilar de *Cost Optimization*) y el cuerpo de prácticas de **FinOps** (FinOps Foundation ⚠️). Re-verificá todo lo marcado con ⚠️ (precios por GB, descuentos de reserved/spot, nombres y mínimos de los tiers de storage, cargos de los servicios gestionados): se mueven por región, volumen y año — lo estable son las **relaciones**, no los dígitos. Los puentes en el hub: [Diseño de sistemas](system-design.md) M2 es la matemática de servilleta que acá pasamos a dólares; [Networking](networking.md) es el egress y la topología (CDN/anycast) que dominan el módulo 3; [Datos a escala](datos-escala.md) es el storage y la retención del módulo 4; [Replicación](replicacion.md) y [Consenso](consenso.md) son el costo de la disponibilidad y la coordinación del módulo 5; [Seguridad de sistemas](seguridad-sistemas.md) M7 es el mismo criterio "gestionado vs propio", ahora en plata; y [Observabilidad](observabilidad.md) es la disciplina de "medí antes de optimizar" que el módulo 7 traslada del rendimiento al costo.
