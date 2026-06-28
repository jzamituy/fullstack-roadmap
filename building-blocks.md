# Building blocks y diseño geoespacial a fondo

**CDN, service discovery, estructuras probabilísticas (Bloom/Count-Min/HLL), leaderboards y "cerca de mí" · Node + Go · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el **M5** —y cierre— del roadmap profundo de system design (M1 [Consenso](consenso.md), M2 [Replicación](replicacion.md), M3 [Streams](streams.md), M4 [Datos a escala](datos-escala.md)). Es la caja de **piezas reutilizables** que aparecen una y otra vez en los deep-dives: cómo sirve la CDN, cómo se encuentran los servicios, las **estructuras probabilísticas** que cambian exactitud por memoria (Bloom, Count-Min, HyperLogLog), cómo se hace un **leaderboard** a escala, y cómo se resuelve **"¿qué hay cerca de mí?"** (geohash, quadtree). Su complemento es el [laboratorio práctico](building-blocks-practica.md): acá entendés, allá **construís** (un Bloom filter, un Count-Min, un HyperLogLog, un leaderboard y un geohash).

**Lo que asumimos.** El método de [system-design](system-design.md), y los M1-M4 de este roadmap (reusamos consistent hashing, quórums, caché, sharding sin re-explicarlos). [Redis](redis.md) para sorted sets y GEO. No asumimos Go: los ejemplos se explican línea por línea.

> ⚠️ **Sobre los lenguajes.** Estas piezas son **estructuras de datos y patrones**. Los ejemplos van en **Node/TS** (tu stack) y en **Go** donde ayuda. La idea importa más que el runtime.

### El marco que usamos en todo el módulo

Igual que en M1-M4: **cuatro capas** ante fallas y **las tres cosas** al diseñar (happy path / modelo de falla / recuperación).

> **El ejemplo que fija el tono — "el contador de visitas únicas se comió toda la RAM".** Respuesta débil: *"guardá los user ids en un Set"*. Respuesta profunda: **para contar usuarios únicos exactos guardás cada id en un Set → con 100M de usuarios son gigabytes de RAM por cada métrica, por cada ventana de tiempo → no escala**. Si te alcanza con ~1% de error, un **HyperLogLog** te da la cardinalidad en **~12 KB** sin importar si son mil o mil millones. Cambiar exactitud por memoria —cuando el negocio tolera el error— es la jugada de las estructuras probabilísticas (módulos 3-7).

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (el grueso va al [laboratorio](building-blocks-practica.md)).

**Índice de módulos**

*Etapa 1 — Borde y descubrimiento*
1. CDN: push vs pull
2. Service discovery

*Etapa 2 — Estructuras probabilísticas*
3. Por qué aproximar a escala
4. Bloom filter (¿está en el conjunto?)
5. Count-Min Sketch (¿cuántas veces?)
6. HyperLogLog (¿cuántos únicos?)
7. Cuándo usar cuál

*Etapa 3 — Rankings a escala*
8. Leaderboard con sorted sets

*Etapa 4 — Geoespacial*
9. El problema de "cerca de mí"
10. Geohash
11. Quadtree
12. Proximity search en producción

*Etapa 5 — Cierre*
13. Cómo hablar de building blocks en una entrevista
14. Cierre del roadmap (M1-M5)

Las soluciones de **todos** los ejercicios están al final.

---

## Etapa 1 — Borde y descubrimiento

## Módulo 1 — CDN: push vs pull

**Teoría.** Una **CDN** (Content Delivery Network) es una red de servidores **cerca del usuario** (en cientos de ubicaciones / *PoPs*) que cachean contenido para servirlo con **baja latencia** y **sacarle carga al origen**. La pregunta de diseño: **¿cómo llega el contenido a la CDN?**

- **Pull (origin pull, el default):** la CDN **no** tiene el contenido hasta que alguien lo pide. En el primer request (*cache miss*), la CDN va al **origen**, trae el objeto, lo **cachea** y lo sirve; los siguientes requests (*cache hit*) salen de la CDN. El objeto vive según su **TTL** (de los headers `Cache-Control`/`Expires`). **Pro:** simple, la CDN se llena sola con lo que se pide. **Contra:** el **primer** usuario de cada objeto (o tras expirar el TTL) paga el miss (latencia + carga al origen).
- **Push:** vos **subís** el contenido a la CDN **proactivamente** (antes de que nadie lo pida), normalmente en el deploy. **Pro:** cero cache miss (todo ya está), control total de cuándo se actualiza. **Contra:** vos gestionás qué subir y cuándo invalidar; tiene sentido para contenido **grande y predecible** (videos, releases de assets) que no querés que el primer usuario espere.

El otro tema de la CDN, que se pregunta siempre: **invalidación**. Cambiar un objeto cacheado es difícil ("una de las dos cosas difíciles en CS"). Dos estrategias: **TTL** (esperás a que expire — simple pero servís contenido viejo un rato) y **versioning/cache-busting** (le cambiás la URL al objeto: `app.a1b2c3.js` — el contenido nuevo tiene URL nueva, así que nunca servís el viejo, y podés cachear "para siempre"). El cache-busting es lo que se usa para assets de frontend.

> La frase mental: **pull = la CDN se llena con lo que se pide (lazy); push = vos la llenás antes (eager).** Y para actualizar, **no invalides — versioná la URL.** Esto conecta con [Service Workers](service-workers.md) (caché del lado del browser, misma lógica de estrategias) y con la capa de borde de [system-design](system-design.md).

**Ejercicios 1**
1.1 🔁 Diferencia entre CDN pull y push en una frase.
1.2 🧠 ¿Quién paga el costo en una CDN pull y cuándo? ¿Cómo lo evita el push?
1.3 🧠 ¿Por qué "versioná la URL" (cache-busting) es mejor que "invalidá el objeto" para assets de frontend?

---

## Módulo 2 — Service discovery

**Teoría.** En una arquitectura de microservicios, las instancias **aparecen y desaparecen** todo el tiempo (autoscaling, deploys, caídas) y sus IPs cambian. ¿Cómo encuentra el servicio A a una instancia sana del servicio B sin IPs hardcodeadas? Con **service discovery**: un **registry** (registro) donde las instancias se **registran** al arrancar (y se **des-registran**/expiran al morir, vía health checks), y los clientes **consultan** para obtener direcciones vivas. Dos patrones:

- **Client-side discovery:** el cliente consulta el registry (p. ej. Consul, etcd, Eureka), obtiene la lista de instancias sanas de B, y **él mismo** elige una (balanceo del lado del cliente). **Pro:** sin salto extra, balanceo inteligente. **Contra:** cada cliente tiene que implementar la lógica de discovery/balanceo.
- **Server-side discovery:** el cliente le pega a un **balanceador/proxy** (un load balancer, un API gateway, o el Service de Kubernetes) que **él** consulta el registry y enruta. **Pro:** el cliente no sabe nada (solo conoce el LB). **Contra:** un salto más y el LB es una pieza a operar. **Es lo más común hoy** (Kubernetes Services + kube-proxy/DNS hacen esto por vos).

El registry necesita **health checks** (activos: el registry pinguea; o pasivos: la instancia manda heartbeats — [Consenso módulo 3](consenso.md)) para sacar instancias muertas, y suele apoyarse en un sistema de consenso (etcd/ZooKeeper, [Consenso módulo 13](consenso.md)) o gossip (Consul/SWIM, [Consenso módulo 14](consenso.md)) para mantener la membresía.

> En Kubernetes esto es transparente: registrás un **Service**, y el cluster te da un nombre DNS estable (`mi-servicio.namespace`) que resuelve a las instancias sanas (los Pods con ese label), balanceando solo. No "montás" service discovery; lo usás. La pieza que conviene saber nombrar: **el registry + health checks es lo que hace que las direcciones efímeras no rompan la comunicación.**

**Ejercicios 2**
2.1 🔁 ¿Qué problema resuelve el service discovery y qué es el registry?
2.2 🧠 Diferencia entre client-side y server-side discovery; ¿cuál es más común hoy y por qué?
2.3 🧠 ¿Por qué el registry necesita health checks, y con qué pieza de M1 (consenso/gossip) se suele implementar?

---

## Etapa 2 — Estructuras probabilísticas

## Módulo 3 — Por qué aproximar a escala

**Teoría.** A escala, las respuestas **exactas** a veces cuestan **demasiada memoria o CPU**, y resulta que el negocio **no necesita** exactitud. Las **estructuras de datos probabilísticas** hacen un trato explícito: **te dan una respuesta aproximada (con un error acotado y conocido) usando una fracción ínfima de la memoria** de la versión exacta. Tres preguntas clásicas y su estructura:

| Pregunta | Exacto cuesta | Probabilístico | Error |
|---|---|---|---|
| ¿`x` está en el conjunto? | guardar todos los elementos | **Bloom filter** | falsos positivos, **cero** falsos negativos |
| ¿cuántas veces apareció `x`? | un contador por elemento | **Count-Min Sketch** | **sobre**estima, nunca subestima |
| ¿cuántos elementos **únicos** hay? | un Set de todos los ids | **HyperLogLog** | ~0.8% con ~12 KB (configurable) |

El patrón común: todas usan **funciones de hash** para mapear elementos a un espacio chico y fijo, y aceptan **colisiones** (de ahí el error). El tamaño de la estructura **no crece** (o crece logarítmicamente) con la cantidad de datos — esa es la magia: un HyperLogLog ocupa ~12 KB cuente mil o mil millones de únicos.

> La pregunta que tenés que hacerte: **¿el negocio tolera un error acotado a cambio de memoria/velocidad?** Si la respuesta es sí (un contador de "vistas únicas" con ±2% está perfecto; "¿este email ya lo vimos?" con algún falso positivo que mandás a una verificación exacta, también), las probabilísticas son la herramienta. Si necesitás exactitud (saldos, facturación), no. Es un trade-off **deliberado**, no un atajo sucio.

**Ejercicios 3**
3.1 🔁 ¿Qué intercambian las estructuras probabilísticas y a cambio de qué?
3.2 🧠 ¿Por qué el tamaño de un HyperLogLog no crece con la cantidad de elementos?
3.3 🧠 Dá un caso donde un error acotado es aceptable y uno donde no.

---

## Módulo 4 — Bloom filter (¿está en el conjunto?)

**Teoría.** Un **Bloom filter** responde "**¿`x` está en el conjunto?**" con una garantía asimétrica clave: puede decir **"quizás sí"** (con chance de falso positivo) o **"definitivamente no"** (sin error). **Nunca** hay falsos negativos. Cómo funciona:

- Un **array de bits** de tamaño `m`, todo en 0, y **`k` funciones de hash**.
- **Agregar `x`:** calculás los `k` hashes de `x`, cada uno apunta a una posición del array → ponés esos `k` bits en **1**.
- **Consultar `x`:** calculás los mismos `k` hashes. Si **todos** esos bits están en 1 → "**quizás** está" (pudieron prenderse por *otros* elementos → falso positivo). Si **alguno** está en 0 → "**seguro no** está" (si se hubiera agregado, ese bit estaría en 1).

Esa asimetría —cero falsos negativos— es lo que lo hace útil como **filtro previo barato**: "¿vale la pena ir al disco/red a buscar esto?". Ejemplos: las SSTables del LSM ([Datos a escala módulo 11](datos-escala.md)) usan un Bloom por SSTable para saltear las que **seguro** no tienen la key; un CDN/cache lo usa para "¿este objeto existe?" antes de pegarle al origen; un sitio chequea "¿este password está en la lista de filtrados?".

El error (tasa de falsos positivos) **sube** a medida que agregás elementos (más bits en 1), y **baja** con más memoria (`m` mayor) y un `k` óptimo. Hay una fórmula para dimensionarlo dado cuántos elementos esperás y qué tasa de falso positivo tolerás. **No se puede borrar** un elemento (poner bits en 0 podría romper a otros) → para borrados se usan variantes (counting Bloom filter).

> 🔥 **Falla en 4 capas — Bloom filter saturado.** (1) Qué se rompe: metiste muchos más elementos de los que dimensionaste → casi todos los bits en 1 → la tasa de falsos positivos se dispara y el filtro "dice que sí a todo" (deja de filtrar). (2) Por qué a esta escala: dimensionaste para `n` y llegaron `10n`. (3) Corto plazo: monitorear el fill ratio. (4) Largo plazo: dimensionar `m` y `k` para el `n` real esperado, o usar un **scalable Bloom filter** que crece. Lo dimensionás y lo construís en el [laboratorio](building-blocks-practica.md).

**Ejercicios 4**
4.1 🔁 ¿Qué dos respuestas da un Bloom filter y cuál de las dos es exacta?
4.2 🧠 ¿Por qué un Bloom filter no tiene falsos negativos pero sí falsos positivos?
4.3 🧠 ¿Por qué no se puede borrar un elemento de un Bloom filter clásico?

---

## Módulo 5 — Count-Min Sketch (¿cuántas veces?)

**Teoría.** El **Count-Min Sketch** responde "**¿cuántas veces apareció `x`?**" (frecuencia) en un stream enorme, sin guardar un contador por elemento. Es el primo del Bloom para conteos:

- Una **matriz** de `d` filas × `w` columnas de contadores (en 0), y **`d` funciones de hash** (una por fila).
- **Incrementar `x`:** en cada fila `i`, hasheás `x` a una columna y **sumás 1** a ese contador (`d` incrementos, uno por fila).
- **Consultar `x`:** tomás los `d` contadores (uno por fila) y devolvés el **mínimo** de ellos.

¿Por qué el **mínimo**? Porque las colisiones solo pueden **sumar** cuentas de otros elementos a un contador → cada contador es `≥` la cuenta real. El mínimo es el menos contaminado → la mejor estimación. Por eso el Count-Min **sobreestima, nunca subestima** (el error es siempre hacia arriba).

Sirve para **heavy hitters** (los elementos más frecuentes): "¿cuáles son las URLs/IPs/búsquedas más populares?" en un stream de miles de millones de eventos, sin un contador por URL. Lo usan rate limiters, detección de elephant flows en redes, trending topics.

> La conexión con el Bloom: ambos son **una matriz de bits/contadores + `k` hashes + aceptar colisiones**. El Bloom responde sí/no (membresía), el Count-Min responde cuánto (frecuencia). Y la asimetría del error es la misma idea: el Bloom nunca da falso negativo, el Count-Min nunca subestima. Lo construís en el [laboratorio](building-blocks-practica.md).

**Ejercicios 5**
5.1 🔁 ¿Qué pregunta responde un Count-Min Sketch y qué devuelve al consultar (¿por qué el mínimo)?
5.2 🧠 ¿Por qué sobreestima y nunca subestima?
5.3 🧠 ¿Para qué problema concreto (nombralo) es la herramienta y por qué no usás un contador por elemento?

---

## Módulo 6 — HyperLogLog (¿cuántos únicos?)

**Teoría.** El **HyperLogLog (HLL)** responde "**¿cuántos elementos únicos hay?**" (cardinalidad) usando memoria **fija y diminuta** (~12 KB para ~0.8% de error estándar — el HLL de Redis, que empaqueta cada registro en 6 bits; el [laboratorio](building-blocks-practica.md) usa 16 KB con 1 byte/registro por simplicidad). El error es **configurable**: sigue `≈ 1.04/√m` (más registros `m` = menos error, más memoria), así que un ±2% sería una config más chica (~1.5 KB). Cuente mil o **mil millones** de únicos, la memoria no cambia. La intuición (no la matemática completa): La intuición (no la matemática completa):

- Hasheás cada elemento a una secuencia de bits "aleatoria". En una secuencia aleatoria, ver un patrón **raro** (p. ej. muchos ceros al principio: `0000…1`) es **improbable** — cuantos **más elementos distintos** veas, **más probable** es que hayas visto **algún** patrón muy raro. Concretamente: si el máximo de "ceros a la izquierda" que viste es `p`, probablemente viste del orden de `2^p` elementos distintos.
- Para bajar la varianza, HLL no usa un solo contador: divide el hash en **`m` buckets** (registros), cada uno trackea el máximo de ceros a la izquierda que vio, y combina los `m` con una **media armónica** (más robusta a outliers). El error baja con más buckets (`error ≈ 1.04/√m`).

Lo notable: los **duplicados no cambian nada** (re-ver un elemento ya visto da el mismo hash → el mismo bucket, el mismo máximo) → cuenta **únicos** automáticamente, y es **mergeable** (unís dos HLL tomando el máximo por bucket → cardinalidad de la unión, sin doble-contar) — perfecto para agregaciones distribuidas (cada shard su HLL, los unís).

> El caso de uso canónico: **"usuarios únicos" / "visitantes distintos"** por día/hora/campaña, a escala. Un `Set` exacto serían GB por métrica; un HLL son ~12 KB y se **mergea** entre shards. Redis lo trae built-in (`PFADD`/`PFCOUNT`/`PFMERGE`). Lo construís (versión simplificada) en el [laboratorio](building-blocks-practica.md).

**Ejercicios 6**
6.1 🔁 ¿Qué cuenta un HyperLogLog y con cuánta memoria respecto de un Set exacto?
6.2 🧠 ¿Por qué los elementos duplicados no afectan el conteo de un HLL?
6.3 🧠 ¿Por qué la propiedad de ser "mergeable" (unir dos HLL por el máximo) es clave para conteos distribuidos?

---

## Módulo 7 — Cuándo usar cuál

**Teoría.** El reflejo de entrevista: ante "contá/chequeá X a escala", reconocé cuál de las tres aplica por la **pregunta** que responde:

- **"¿Está/lo vi antes?"** (membresía, sí/no) → **Bloom filter**. Ej: "¿este email ya está registrado?" como filtro previo al disco; "¿esta key está en la SSTable?"; "¿este password está filtrado?".
- **"¿Cuántas veces / cuáles son los top?"** (frecuencia, heavy hitters) → **Count-Min Sketch**. Ej: "URLs/IPs/búsquedas más frecuentes"; rate limiting aproximado por clave.
- **"¿Cuántos distintos?"** (cardinalidad) → **HyperLogLog**. Ej: "usuarios únicos", "IPs distintas", "productos vistos distintos".

La regla común y la trampa: **todas valen solo si tolerás el error** y querés ahorrar memoria/CPU **a escala**. Si los datos entran cómodos en memoria exacta (un `Set`, un `Map` de contadores), **no metas una probabilística** — agregás complejidad y error sin necesidad. Y si necesitás exactitud (cobros, conteos legales), tampoco. Son para **mucho volumen + tolerancia al error**.

> La muletilla que suena senior: *"¿membresía, frecuencia o cardinalidad? ¿y el negocio tolera ±error a cambio de pasar de GB a KB? Si sí, Bloom / Count-Min / HLL; si los datos entran exactos o necesito exactitud, no."* Esa frase sola contesta media docena de preguntas de diseño.

**Ejercicios 7**
7.1 🔁 Asociá cada pregunta (membresía / frecuencia / cardinalidad) con su estructura.
7.2 🧠 ¿Cuándo NO usar una estructura probabilística aunque "suene cool"?
7.3 🧠 "Detectar las 100 búsquedas más populares de hoy sobre miles de millones de queries." ¿Qué estructura y por qué?

---

## Etapa 3 — Rankings a escala

## Módulo 8 — Leaderboard con sorted sets

**Teoría.** Un **leaderboard** (ranking de puntajes: top jugadores, trending, posiciones) parece trivial hasta que lo pensás a escala: con millones de jugadores, "¿cuál es el top 10?" y "¿en qué puesto está el jugador X?" tienen que ser **rápidos** y actualizarse en vivo. Un `ORDER BY score DESC LIMIT 10` sobre una tabla es caro si se consulta seguido y cambia constantemente, y "¿qué rank tengo?" (contar cuántos tienen más puntaje) es aún peor.

La estructura correcta es un **sorted set** (conjunto ordenado), que Redis implementa con una **skip list** + un hash. Da, todas en `O(log n)`:
- **Agregar/actualizar** el puntaje de un jugador (`ZADD`).
- **Top-K** (los K de mayor puntaje, `ZREVRANGE 0 K`).
- **Rank de un jugador** (su posición, `ZREVRANK`) — la operación que una tabla SQL hace fatal.
- **Rango por puntaje** ("todos entre 1000 y 2000 puntos", `ZRANGEBYSCORE`).

La skip list (probabilística, [Concurrencia](concurrencia.md) la roza) mantiene los elementos **ordenados** con niveles de "atajos" que permiten saltar, dando búsqueda/inserción/rank en `O(log n)` sin el costo de rebalanceo de un árbol.

> 🔥 **Falla en 4 capas — el rank por `COUNT(*)` en SQL.** (1) Qué se rompe: "¿en qué puesto estoy?" implementado como `SELECT COUNT(*) WHERE score > miScore` escanea/cuenta una porción enorme en cada consulta → lento y carga la DB. (2) Por qué a esta escala: con millones de jugadores y muchas consultas de rank por segundo (cada jugador ve su puesto), ese count repetido satura. (3) Corto plazo: cachear el top-N (la mayoría mira el top). (4) Largo plazo: **sorted set** (Redis ZSET) que mantiene el rank en `O(log n)` — el rank es una operación nativa, no un count. Lo construís en el [laboratorio](building-blocks-practica.md).

**Ejercicios 8**
8.1 🔁 ¿Qué cuatro operaciones da un sorted set y en qué complejidad?
8.2 🧠 ¿Por qué "¿en qué puesto estoy?" es caro en SQL y barato en un sorted set?
8.3 🧠 ¿Qué estructura usa Redis por debajo para el sorted set y qué le da el `O(log n)`?

---

## Etapa 4 — Geoespacial

## Módulo 9 — El problema de "cerca de mí"

**Teoría.** "Mostrame los restaurantes/conductores/usuarios **cerca de mí**" parece un `WHERE`... hasta que lo pensás. Cada punto tiene **dos** coordenadas (lat, long), y "cerca" es una distancia en **dos dimensiones**. El problema:

- Un **índice B-tree** ([Datos a escala módulo 10](datos-escala.md)) ordena por **una** dimensión. Podés indexar `lat` y `long` por separado, pero "cerca de mí" (`lat BETWEEN ... AND long BETWEEN ...`) te obliga a intersecar dos rangos enormes — un cuadrado de "candidatos por latitud" que incluye puntos a miles de km al este/oeste. **No** captura proximidad 2D: dos puntos con latitud parecida pueden estar lejísimos.
- Calcular la distancia real a **todos** los puntos y ordenar es `O(n)` — imposible a escala (millones de puntos, consultas constantes).

La clave: necesitás un índice que **preserve la proximidad espacial** — que puntos cercanos en el mapa queden cercanos en el índice, para poder buscar "los de alrededor" sin escanear todo. El truco general se llama **space-filling curve / jerarquía espacial**: convertir 2D en 1D (o en un árbol) de forma que la cercanía se conserve. Las dos técnicas canónicas: **geohash** (módulo 10) y **quadtree** (módulo 11).

> La frase mental: **el problema geoespacial es "cómo indexo dos dimensiones de forma que 'cerca en el mapa' sea 'cerca en el índice'".** Un B-tree por columna no lo logra (ordena una dimensión y rompe la otra). De ahí nacen geohash y quadtree.

**Ejercicios 9**
9.1 🔁 ¿Por qué indexar `lat` y `long` con B-trees separados no resuelve "cerca de mí"?
9.2 🧠 ¿Por qué calcular la distancia a todos los puntos no escala?
9.3 🧠 ¿Qué propiedad tiene que tener un índice geoespacial (en una frase)?

---

## Módulo 10 — Geohash

**Teoría.** El **geohash** convierte un par (lat, long) en un **string corto** (p. ej. `9q8yyk`) con una propiedad mágica: **puntos cercanos comparten un prefijo**. Cómo:

- Subdividís el mundo en una grilla, recursivamente: partís el rango de longitud en dos (¿izquierda o derecha? → un bit), después el de latitud (¿abajo o arriba? → otro bit), y repetís, **interleaving** los bits. Cada subdivisión agrega bits → más precisión.
- Esos bits se codifican en **base32** → el string. **Cuanto más largo el geohash, más preciso** (más chica la celda): 5 caracteres ≈ ±2.4 km, 6 ≈ ±0.3 km, 7 ≈ ±76 m (en precisión 6 la celda no es cuadrada: ~0.6 km de ancho × ~0.3 km de alto).
- **La propiedad clave:** como los bits van de lo general a lo específico, **dos puntos cercanos comparten un prefijo largo** (`9q8yyk` y `9q8yym` están en la misma zona). Entonces "buscar cerca" = **buscar por prefijo** (`LIKE '9q8yy%'`) → ¡y eso un **B-tree** lo hace genial! El geohash convirtió proximidad 2D en un **prefijo 1D** que cualquier índice ordenado resuelve.

> ⚠️ **El borde (edge case) que tenés que mencionar:** dos puntos pueden estar **pegadísimos** físicamente pero caer en celdas distintas (a un lado y otro de un borde de la grilla) → **prefijos distintos**. Ej: justo en el borde entre dos celdas. Por eso una búsqueda de proximidad seria consulta la celda **y sus 8 vecinas** (las 8 celdas adyacentes en la grilla), no solo el prefijo propio. Si no, te perdés los puntos del otro lado del borde. Lo ves en el [laboratorio](building-blocks-practica.md).

> La belleza del geohash: **reduce 2D a un string 1D ordenable** → lo metés en cualquier índice B-tree o key-value y buscás por prefijo. Por eso es tan popular (Redis GEO lo usa por debajo, junto con otras técnicas).

**Ejercicios 10**
10.1 🔁 ¿Qué propiedad tienen los geohashes de dos puntos cercanos, y cómo se traduce eso en una búsqueda?
10.2 🧠 ¿Por qué un geohash más largo es más preciso?
10.3 🧠 Explicá el edge case del borde y por qué se consultan las 8 celdas vecinas.

---

## Módulo 11 — Quadtree

**Teoría.** El **quadtree** es la otra técnica: un **árbol** donde cada nodo representa una región rectangular y, si esa región tiene **demasiados puntos**, se **subdivide en 4 cuadrantes** (NE, NO, SE, SO), cada uno un hijo. Recursivo: las zonas **densas** se subdividen mucho (árbol profundo ahí), las **vacías** quedan como una hoja grande. Buscar "cerca de mí" = bajar por el árbol hasta la región que te contiene y mirar esa hoja + las vecinas.

La diferencia clave con el geohash:

- **Geohash:** grilla de celdas de **tamaño fijo** (según el largo del hash). Simple, indexable por prefijo en un B-tree, pero **no se adapta** a la densidad: una celda en medio del océano y una en el centro de Manhattan tienen el mismo tamaño (la de Manhattan tiene millones de puntos, la del océano cero).
- **Quadtree:** subdivisión **adaptativa**. Cada hoja tiene a lo sumo `K` puntos (capacidad): donde hay densidad, se subdivide hasta cumplirlo → las celdas son **chicas donde hay muchos puntos y grandes donde hay pocos**. Mejor para datos con densidad **muy variable** (que es lo real: la gente se concentra en ciudades).

> El trade-off: **geohash** es más simple y encaja en un índice ordenado existente (prefijos en un B-tree, fácil de shardear y cachear); **quadtree** se adapta a la densidad (no malgasta resolución en zonas vacías ni se satura en zonas densas) pero es un árbol que mantenés en memoria/aparte y rebalanceás al insertar/borrar. Variantes relacionadas que conviene nombrar: **R-tree** (rectángulos, en PostGIS) y **S2** (de Google, celdas jerárquicas sobre una curva de Hilbert, lo usa parte de la industria de mapas/ride-hailing).

**Ejercicios 11**
11.1 🔁 ¿Cuándo subdivide un quadtree una región y en cuántos cuadrantes?
11.2 🧠 ¿Cuál es la diferencia central entre geohash (grilla fija) y quadtree (adaptativo), y para qué dato importa?
11.3 🧠 ¿Qué ventaja operativa tiene el geohash sobre el quadtree a la hora de indexar/shardear?

---

## Módulo 12 — Proximity search en producción

**Teoría.** Cómo se arma de verdad un "buscar cerca" a escala (el corazón de Uber/Tinder/Yelp/mapas):

1. **Indexás** cada punto por su celda espacial (geohash de longitud `L`, o celda S2/quadtree). En una DB: una columna `geohash` indexada (B-tree); en Redis: `GEOADD` (que internamente usa geohash como score de un sorted set, el del módulo 8).
2. **Consultás** "cerca de (lat, long) en radio R": calculás el geohash del punto de consulta, y traés los puntos de **esa celda + las vecinas** (cubriendo el radio R con el nivel de celda adecuado). Eso reduce de "millones" a "los de unas pocas celdas" — un conjunto chico de **candidatos**.
3. **Refinás:** sobre ese conjunto chico, calculás la **distancia real** (Haversine) y filtrás/ordenás por `≤ R`. El cálculo `O(n)` ahora es sobre **decenas/cientos** de candidatos, no millones.

El patrón es **"filtro grueso espacial → refinamiento exacto"**: la celda espacial **acota** los candidatos baratísimo (índice por prefijo), y la distancia exacta solo se calcula sobre ese puñado. Redis lo da llave en mano: `GEOADD`/`GEOSEARCH ... BYRADIUS`.

> 🔥 **El caso ride-hailing en 4 capas (Uber).** (1) Qué se rompe: "conductores cerca del pasajero" calculando distancia a **todos** los conductores → no escala con millones de ubicaciones actualizándose por segundo. (2) Por qué a esta escala: las ubicaciones cambian **constantemente** (cada conductor reporta cada pocos segundos) y las consultas son masivas. (3) Corto plazo: índice geoespacial (geohash/S2) + consultar solo celdas cercanas. (4) Largo plazo: **shardear por región/celda** (los conductores de una zona viven juntos → consultas locales), TTL en las ubicaciones (un conductor que dejó de reportar expira), y el patrón filtro-grueso→refinamiento. Conecta con [Datos a escala](datos-escala.md) (sharding por celda) y [Streams](streams.md) (las ubicaciones son un stream).

**Ejercicios 12**
12.1 🔁 ¿Cuáles son los tres pasos de un proximity search (indexar / consultar / refinar)?
12.2 🧠 Explicá el patrón "filtro grueso espacial → refinamiento exacto" y por qué hace escalable el cálculo de distancia.
12.3 🧠 ¿Por qué shardear por celda/región tiene sentido para un sistema tipo Uber?

---

## Etapa 5 — Cierre

## Módulo 13 — Cómo hablar de building blocks en una entrevista

**Teoría.** Estas piezas casi nunca son **el** problema de la entrevista — son lo que **encajás** dentro de un diseño más grande, y nombrarlas en el momento justo es lo que suena senior. El reflejo: cuando aparezca el sub-problema, sacá la pieza.

- *"Hay que servir assets/estáticos con baja latencia global"* → **CDN** (pull + cache-busting por versión de URL).
- *"Los servicios tienen que encontrarse y las instancias cambian"* → **service discovery** (registry + health checks; en K8s, un Service).
- *"¿Existe esto? (filtro previo barato antes de ir al disco/red)"* → **Bloom filter**.
- *"Los más frecuentes / heavy hitters sobre un stream enorme"* → **Count-Min Sketch**.
- *"Cuántos únicos a escala con poca memoria"* → **HyperLogLog**.
- *"Ranking / top-K / mi posición en vivo"* → **sorted set** (Redis ZSET).
- *"Qué hay cerca de mí"* → **geohash/quadtree** + filtro-grueso→refinamiento (en producción: **Redis GEO**, **PostGIS/R-tree**, o **S2** de Google).

Y la pregunta transversal de las probabilísticas: *"¿el negocio tolera un error acotado a cambio de pasar de GB a KB?"*.

> **Las tres cosas** aplicadas a, por ejemplo, "nearby drivers": **(a) happy path** = ubicaciones indexadas por celda (geohash) en Redis GEO, la consulta trae celdas vecinas y refina por Haversine; **(b) modelo de falla** = una celda hot (centro de la ciudad) se satura → shardear por región y cachear; ubicaciones viejas → TTL; **(c) recuperación** = si la celda hot cae, el resto sigue (blast radius por celda).

**Ejercicios 13**
13.1 🔁 Asociá tres sub-problemas con su building block.
13.2 🧠 ¿Cuál es la pregunta transversal que decide si usás una estructura probabilística?
13.3 🧠 Aplicá "las tres cosas" a un buscador de "usuarios cerca" (tipo Tinder).

---

## Módulo 14 — Cierre del roadmap (M1-M5)

**Teoría.** Cerramos los cinco módulos profundos de system design. Vale la pena ver el mapa completo, porque en una entrevista de diseño **se cruzan todos**:

- **M1 [Consenso](consenso.md):** cómo varias máquinas **se ponen de acuerdo** cuando la red miente (Raft, quórums, relojes lógicos, fencing, consistent hashing). La base de la coordinación.
- **M2 [Replicación](replicacion.md):** cómo mantener **copias** que mienten un rato (sync/async, lag, multi-región, LWW/CRDTs, 2PC vs saga). La base de la disponibilidad y la consistencia.
- **M3 [Streams](streams.md):** **datos en movimiento** (Kafka, event-time/watermarks, exactly-once, OLTP vs OLAP, Lambda/Kappa). La base del procesamiento en vivo y la analítica.
- **M4 [Datos a escala](datos-escala.md):** cuando **no entran en una máquina** (sharding, skew, multi-tenancy, migraciones sin downtime, B-tree vs LSM, IDs). La base de operar datos grandes.
- **M5 (este):** las **piezas reutilizables** (CDN, discovery, probabilísticas, leaderboard, geoespacial) que encajás en todo lo anterior.

> El hilo conductor de los cinco es **uno solo**: a escala, **no podés tener todo** —exactitud, disponibilidad, consistencia fuerte, latencia baja, simplicidad— **a la vez**, así que el oficio es **elegir conscientemente qué ceder, dónde, y tener un plan para cuando falle**. Eso es lo que las **4 capas** (qué se rompe → por qué a esta escala → control de corto plazo → rediseño) y **las 3 cosas** (happy path → modelo de falla → recuperación) entrenan a decir en voz alta. Un junior recita definiciones; un senior razona trade-offs y fallas. Si terminaste los cinco módulos y sus laboratorios, ya tenés con qué.

> **Practicá en voz alta** combinando módulos: "diseñá Uber" toca M5 (geoespacial) + M4 (sharding por celda) + M3 (stream de ubicaciones) + M2 (réplicas multi-región) + M1 (coordinación). "Diseñá un feed" toca todos también. El [banco de casos](system-design-casos.md) y el [checklist](system-design-checklist.md) son tu campo de entrenamiento.

**Ejercicios 14**
14.1 🔁 En una frase por módulo, ¿qué problema resuelve cada uno de M1-M5?
14.2 🧠 ¿Cuál es el hilo conductor único de los cinco módulos?
14.3 🧠 Elegí un sistema (Uber, feed, chat) y nombrá qué módulo del roadmap toca cada parte de su diseño.

---

## Soluciones

**1.1** Pull: la CDN trae el contenido del origen **cuando alguien lo pide** (lazy, se llena sola con cache misses). Push: vos **subís** el contenido a la CDN **proactivamente** (eager, antes de que lo pidan).

**1.2** En pull, el **primer** usuario de cada objeto (o el primero tras expirar el TTL) paga el **cache miss**: latencia extra + carga al origen. Push lo evita porque el contenido ya está en la CDN antes de cualquier request → cero miss (a costa de que vos gestionás qué subir y cuándo).

**1.3** Porque invalidar un objeto cacheado en cientos de PoPs es lento e inconsistente (hay que propagar la purga); con cache-busting el contenido nuevo tiene una **URL nueva** (`app.a1b2c3.js`), así que nunca servís el viejo (las URLs viejas dejan de referenciarse) y podés cachear cada versión **para siempre** (TTL infinito) sin riesgo de servir stale.

**2.1** Resuelve cómo un servicio **encuentra instancias sanas** de otro cuando las IPs cambian constantemente (autoscaling, deploys, caídas). El **registry** es el registro donde las instancias se anotan al arrancar (y se sacan al morir) y que los clientes consultan para obtener direcciones vivas.

**2.2** Client-side: el cliente consulta el registry y **él** elige la instancia (balanceo del lado del cliente). Server-side: el cliente le pega a un **balanceador/proxy** que consulta el registry y enruta. Server-side es más común hoy porque el cliente no necesita lógica de discovery (solo conoce el LB), y Kubernetes lo da gratis (Services + DNS).

**2.3** Porque las instancias mueren sin avisar; los **health checks** (activos o por heartbeat) sacan del registry las instancias muertas para no enrutar tráfico a ellas. Se suele implementar sobre **consenso** (etcd/ZooKeeper, [Consenso módulo 13](consenso.md)) o **gossip** (Consul/SWIM, [Consenso módulo 14](consenso.md)) para mantener la membresía consistente/escalable.

**3.1** Intercambian **exactitud** (dan una respuesta aproximada, con error acotado y conocido) a cambio de **memoria y/o velocidad** (usan una fracción ínfima de la versión exacta, y el tamaño no crece —o crece muy lento— con la cantidad de datos).

**3.2** Porque no guarda los elementos: mantiene `m` **registros de tamaño fijo** que trackean un estadístico (el máximo de ceros a la izquierda visto) sobre hashes. Agregar más elementos solo puede **actualizar** esos registros, no agregar nuevos → la memoria es constante (~12 KB) cuente mil o mil millones.

**3.3** Aceptable: contador de "vistas/usuarios únicos" con ±2% (a nadie le importa si dice 1.02M o 1.00M); "¿este email ya lo vimos?" como filtro previo (un falso positivo se resuelve con una verificación exacta). NO aceptable: saldos de cuenta, facturación, conteos legales/financieros — ahí necesitás exactitud.

**4.1** "**Quizás está**" (puede ser falso positivo) y "**definitivamente no está**" (exacta, sin error). La respuesta **exacta** es el "no".

**4.2** No hay falsos negativos porque agregar `x` **prende** sus `k` bits; si `x` está, esos bits están sí o sí en 1, así que nunca dirá "no está" para algo que está. Sí hay falsos positivos porque esos `k` bits pueden haberse prendido por **otros** elementos → todos en 1 sin que `x` se haya agregado → "quizás está" equivocado.

**4.3** Porque un bit en 1 puede estar **compartido** por varios elementos (sus hashes coincidieron en esa posición). Si pusieras ese bit en 0 para "borrar" `x`, romperías la consulta de **todos** los otros elementos que dependían de ese bit (aparecerían como "no están" → falso negativo, que el Bloom no puede tener). Para borrar se usa un counting Bloom filter (contadores en vez de bits).

**5.1** Responde **cuántas veces apareció** un elemento (frecuencia). Al consultar devuelve el **mínimo** de los `d` contadores (uno por fila) porque las colisiones solo **suman** cuentas de otros elementos → cada contador es ≥ la cuenta real → el mínimo es el menos contaminado, la mejor estimación.

**5.2** Porque una colisión hace que el contador de `x` también sume los incrementos de **otro** elemento que cayó en la misma celda → el contador queda **≥** la cuenta real. Nunca puede quedar por debajo (los incrementos de `x` siempre se cuentan). Por eso el error es siempre hacia arriba (sobreestima).

**5.3** **Heavy hitters / top-K frecuentes** (URLs/IPs/búsquedas más populares) sobre un stream enorme. No usás un contador por elemento porque la cantidad de elementos distintos es gigantesca (miles de millones de URLs/IPs) → un `Map` exacto no entra en memoria; el Count-Min usa una matriz fija chica.

**6.1** Cuenta **elementos únicos (cardinalidad)**. Usa memoria **fija y diminuta** (~12 KB para ~0.8% de error estándar), frente a un `Set` exacto que crece linealmente con la cantidad de únicos (GB para cientos de millones).

**6.2** Porque re-ver un elemento ya visto produce el **mismo hash** → cae en el **mismo bucket** y actualiza el mismo máximo de ceros a la izquierda, que ya estaba en ese valor → no cambia nada. El HLL estima en función de los **patrones distintos** vistos, no de cuántas veces.

**6.3** Porque para contar únicos **distribuido** (cada shard ve una parte de los datos), unir dos HLL tomando el **máximo por bucket** da la cardinalidad de la **unión** sin doble-contar los elementos que aparecieron en ambos shards. Un conteo exacto distribuido requeriría compartir los Sets completos (caro); el HLL se mergea en KB. (Redis: `PFMERGE`.)

**7.1** Membresía ("¿está?") → **Bloom filter**. Frecuencia ("¿cuántas veces? / top-K") → **Count-Min Sketch**. Cardinalidad ("¿cuántos distintos?") → **HyperLogLog**.

**7.2** Cuando los datos **entran cómodos en memoria exacta** (un `Set`/`Map` chico): meter una probabilística agrega complejidad y error sin ganar nada. O cuando necesitás **exactitud** (cobros, conteos legales). Son para **mucho volumen + tolerancia al error**, no un default.

**7.3** **Count-Min Sketch**: "las más populares" es un problema de **frecuencia/heavy hitters**, y "miles de millones de queries" hace inviable un contador por query distinta (no entra en memoria). El Count-Min estima frecuencias con una matriz fija y te deja sacar el top-K (combinándolo con un heap de los más vistos).

**8.1** Agregar/actualizar puntaje (`ZADD`), top-K (`ZREVRANGE`), rank de un elemento (`ZREVRANK`), y rango por puntaje (`ZRANGEBYSCORE`) — todas en **`O(log n)`**.

**8.2** En SQL, "¿en qué puesto estoy?" = `COUNT(*) WHERE score > miScore`, que tiene que **contar** cuántos te superan en cada consulta (escaneo de una porción grande, caro y repetido). El sorted set mantiene los elementos **ordenados con su posición** (skip list), así que el rank es una operación nativa `O(log n)`, no un count sobre toda la tabla.

**8.3** Una **skip list** (+ un hash para el lookup por miembro). La skip list mantiene los elementos ordenados con niveles de "atajos" probabilísticos que permiten saltar grandes tramos, dando búsqueda/inserción/rank en `O(log n)` sin el costo de rebalanceo de un árbol balanceado.

**9.1** Porque un B-tree ordena **una** dimensión: indexar `lat` y `long` por separado y hacer `lat BETWEEN ... AND long BETWEEN ...` interseca dos rangos enormes (una franja por latitud que da la vuelta al mundo) y no captura la **proximidad 2D** — dos puntos con latitud parecida pueden estar a miles de km de distancia.

**9.2** Porque calcular la distancia real a **todos** los puntos y ordenar es `O(n)` por consulta; con millones de puntos y consultas constantes (cada usuario pregunta "cerca de mí" todo el tiempo), es demasiado cómputo. Necesitás acotar los candidatos **antes** de calcular distancias.

**9.3** Tiene que **preservar la proximidad espacial**: que puntos cercanos en el mapa queden cercanos en el índice, para poder traer "los de alrededor" sin escanear todo (convertir 2D en un orden/árbol que conserve la cercanía).

**10.1** Dos puntos cercanos **comparten un prefijo** (largo) en su geohash. Eso traduce "buscar cerca" en **buscar por prefijo** (`LIKE '9q8yy%'`), que un índice B-tree resuelve eficientísimo — proximidad 2D convertida en un prefijo 1D ordenable.

**10.2** Porque cada carácter adicional corresponde a más subdivisiones de la grilla (más bits de precisión) → la celda que representa es **más chica** → ubica el punto con mayor exactitud (5 chars ≈ ±2.4 km, 7 ≈ ±76 m).

**10.3** Dos puntos físicamente pegados pueden caer a **distinto lado de un borde** de la grilla → celdas distintas → **prefijos distintos**, y una búsqueda solo por prefijo propio se los pierde. Por eso se consulta la celda **y sus 8 vecinas** (las adyacentes en la grilla), garantizando cubrir los puntos cercanos del otro lado de cualquier borde.

**11.1** Cuando una región acumula **más puntos que su capacidad** `K`, se subdivide en **4** cuadrantes (NE, NO, SE, SO), cada uno un nodo hijo; recursivo hasta que cada hoja tenga ≤ `K` puntos.

**11.2** Geohash usa una grilla de celdas de **tamaño fijo** (no se adapta); quadtree **subdivide adaptativamente** según la densidad (celdas chicas donde hay muchos puntos, grandes donde hay pocos). Importa para datos con **densidad muy variable** (la gente se concentra en ciudades): el quadtree no malgasta resolución en zonas vacías ni se satura en densas.

**11.3** El geohash produce un **string 1D ordenable** que entra directo en un índice B-tree/key-value existente (búsqueda por prefijo) y es trivial de **shardear y cachear**; el quadtree es un árbol que mantenés aparte (en memoria), rebalanceás al insertar/borrar y es más difícil de distribuir. El geohash se integra a la infraestructura de datos que ya tenés.

**12.1** (1) **Indexar** cada punto por su celda espacial (geohash/S2) en un índice ordenado o Redis GEO; (2) **consultar** trayendo los puntos de la celda del query **+ las vecinas** (acotando candidatos); (3) **refinar** calculando la distancia real (Haversine) solo sobre ese conjunto chico y filtrando por radio.

**12.2** El "filtro grueso espacial" (la celda + vecinas) reduce de millones a unas decenas/cientos de **candidatos** usando solo un índice por prefijo (baratísimo); el "refinamiento exacto" calcula la distancia real **solo** sobre ese puñado. Así el cálculo `O(n)` de distancias se aplica a `n` = candidatos (pocos), no a todos los puntos → escalable.

**12.3** Porque los conductores de una **zona** se consultan juntos (un pasajero busca conductores de **su** área), así que shardear por celda/región mantiene los datos de cada zona en el mismo shard → las consultas son **locales** (un shard) en vez de cruzar todos, y el blast radius de una zona hot queda contenido en su shard.

**13.1** Tres de: "servir estáticos global con baja latencia" → CDN; "los servicios se encuentran con instancias cambiantes" → service discovery; "¿existe esto? filtro previo barato" → Bloom filter; "heavy hitters de un stream" → Count-Min; "cuántos únicos con poca memoria" → HyperLogLog; "ranking/mi posición en vivo" → sorted set; "qué hay cerca" → geohash/quadtree.

**13.2** *"¿El negocio tolera un error acotado a cambio de pasar de GB a KB (memoria/velocidad)?"*. Si sí y hay mucho volumen → la probabilística que corresponda. Si los datos entran exactos o se necesita exactitud → no.

**13.3** (a) Happy path: ubicaciones de usuarios indexadas por geohash (Redis GEO), la consulta trae las celdas cercanas y refina por distancia real, mostrando "usuarios a ≤ X km". (b) Modelo de falla: una celda muy densa (centro de una ciudad) se satura → shardear por región + cachear las zonas calientes; ubicaciones viejas expiran por TTL. (c) Recuperación: si el shard de una zona cae, las demás zonas siguen (blast radius por celda/región).

**14.1** M1 Consenso: cómo varias máquinas **se ponen de acuerdo** pese a fallas/red. M2 Replicación: cómo mantener **copias consistentes/disponibles**. M3 Streams: cómo procesar **datos en movimiento** y analítica. M4 Datos a escala: cómo operar datos que **no entran en una máquina** (sharding/multi-tenancy/migraciones/engines). M5 Building blocks: las **piezas reutilizables** (CDN, discovery, probabilísticas, leaderboard, geoespacial).

**14.2** Que a escala **no podés tener todo a la vez** (exactitud, disponibilidad, consistencia fuerte, latencia baja, simplicidad): el oficio es **elegir conscientemente qué ceder, dónde, y tener un plan para la falla**. Las 4 capas y las 3 cosas son la forma de decir ese razonamiento en voz alta.

**14.3** Ejemplo Uber: geoespacial/proximity (M5) para "conductores cerca"; sharding por celda/región (M4) para las ubicaciones; stream de ubicaciones en vivo (M3); réplicas multi-región para disponibilidad (M2); coordinación/asignación de viajes y discovery de servicios (M1/M5). Cada parte del diseño cae en un módulo del roadmap — y combinarlos es lo que se evalúa.
