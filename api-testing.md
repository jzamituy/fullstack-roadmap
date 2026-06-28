# API Testing y Contract Testing: probar por debajo de la UI

**Tests de API automatizados · httpx + pytest, OpenAPI, Pact y carga · stack Python · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Es el segundo módulo del **track QA / Automation** (perfil **AI QA / SDET**) y asume [Python](python.md) (pytest, httpx, Pydantic), [Testing](testing.md) (la pirámide, qué mockear, testcontainers), y [Playwright](playwright.md) (el E2E que está **arriba** de esto). También conviene tener presente [FastAPI](fastapi.md) (la API que vas a probar, con sus contratos Pydantic y su OpenAPI). El **API testing** prueba el sistema **por debajo de la UI**: le pegás directo a los endpoints HTTP, sin navegador. Es el punto dulce de la automatización —**más rápido, más estable y más preciso** que el E2E, y más realista que un unit test—. El objetivo del módulo es doble: las **técnicas** (escribir tests de API, validar el contrato, contract testing entre servicios, carga) y el **criterio** (qué probar en esta capa y qué no, cuándo contract testing en vez de E2E, cómo testear una API de IA no determinista).

**Lo que asumimos.** Python + pytest, httpx, Pydantic ([Python](python.md), [FastAPI](fastapi.md)), la pirámide y testcontainers de [Testing](testing.md), y una API real para probar (la "Task API" del roadmap, o cualquier API con auth + CRUD).

**Para practicar.**

```bash
uv add --dev pytest httpx
# contract testing y spec-driven (módulos 7-8):  uv add --dev pact-python schemathesis
# carga (módulo 9):  uv add --dev locust
```

> Nota sobre datos volátiles: httpx y pytest son estables; las APIs de pact-python, schemathesis y Locust, y el ecosistema Postman/Newman/Pact Broker, cambian entre versiones. Los ejemplos muestran la **forma**; verificá la doc de cada herramienta al implementar.

**Índice de módulos**
1. El API testing en su lugar: bajo el E2E, rápido y preciso
2. El panorama de herramientas: Postman/Newman vs código (httpx + pytest)
3. Anatomía de un test de API: status, body, headers
4. Qué afirmar: el contrato, no solo el `200`
5. Probar las capas: funcional, auth/authz, validación, casos borde
6. Datos, aislamiento y entornos (setup/teardown, nunca contra prod)
7. Contract testing: el problema N×M entre servicios y Pact
8. OpenAPI como contrato: validación de schema y testing dirigido por spec
9. Performance y load testing en la capa de API
10. API testing en CI: testcontainers, Newman y verificación de contratos
11. Testear APIs de IA y el criterio de cierre

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El API testing en su lugar: bajo el E2E, rápido y preciso

**Teoría.** En la pirámide de [Testing](testing.md), el API testing vive en la capa de **integración / servicio**: por debajo del [E2E de browser](playwright.md) (que maneja la UI con un navegador real) y por encima de los unitarios (que prueban una función aislada). Le pegás a los **endpoints HTTP** directamente —un `POST /tasks`, un `GET /tasks/1`— y verificás la respuesta, **sin navegador y sin frontend**.

Por qué es el punto dulce de la automatización, comparado con el E2E:

- **Más rápido**: no arranca un navegador ni renderiza páginas; es un request HTTP. Una suite de API corre en segundos donde una E2E tarda minutos.
- **Más estable (menos flaky)**: no hay timing de UI, ni elementos que tardan en aparecer, ni locators que se rompen con un cambio de CSS ([Playwright](playwright.md) módulo 6). La API es un contrato más estable que el DOM.
- **Más preciso**: cuando falla, sabés que el problema está en el backend (o en el contrato), no en "algo de la UI". El fallo apunta más cerca de la causa.

Por eso la regla de la pirámide: **mové hacia abajo todo lo que puedas.** Lo que se puede verificar pegándole a la API —la lógica de negocio, las validaciones, los códigos de estado, los permisos, los casos borde— se prueba **acá**, no con un E2E. El E2E queda reservado para los pocos flujos de usuario críticos que necesitan la UI real ([Playwright](playwright.md) módulo 1). Un equipo maduro tiene **muchos** tests de API y **pocos** E2E.

La conexión con lo que ya viste: en [Testing](testing.md) probaste endpoints con **Supertest** (que levanta tu app Nest y le pega). El API testing "de caja negra" de este módulo es lo mismo llevado un paso más afuera: le pegás a la API **corriendo** (en un entorno de test) por HTTP real, como lo haría cualquier cliente —tu front, otro servicio, una integración—. La frase mental: **el API testing le pega a los endpoints HTTP sin UI: más rápido, más estable y más preciso que el E2E, y más realista que un unit test —es el punto dulce de la automatización, donde se prueba todo lo que no necesita el navegador, dejando al E2E solo los pocos flujos críticos de UI—.**

**Ejercicios 1**
1.1 ¿Dónde vive el API testing en la pirámide y qué prueba exactamente?
1.2 Nombrá las tres ventajas del API testing sobre el E2E y por qué cada una.
1.3 ¿Qué dice la "regla de la pirámide" sobre qué probar en esta capa vs con E2E?
1.4 ¿En qué se parece y en qué se diferencia del testing con Supertest de [Testing](testing.md)?

---

## Módulo 2 — El panorama de herramientas: Postman/Newman vs código (httpx + pytest)

**Teoría.** Hay dos grandes formas de hacer API testing, y entender cuándo usar cada una es criterio, no preferencia.

- **Postman (y Newman).** Postman es una herramienta **gráfica**: armás requests, los agrupás en **colecciones**, definís entornos (dev/staging) con variables, y le agregás aserciones en scripts. **Newman** es su runner de línea de comandos: corre una colección en **CI**. Fortalezas: **exploración** (probar un endpoint a mano al instante), documentar y compartir colecciones, y onboarding rápido de gente no-dev. Debilidades: los tests viven en archivos JSON de colección (difíciles de versionar y revisar en Git como código), la lógica compleja en scripts se vuelve incómoda, y se aleja de tu base de código.

- **Código (en Python: `httpx` + `pytest`).** Escribís los tests como código en tu repo, con un cliente HTTP (`httpx`, o `requests`) y el framework de testing que ya usás (`pytest`, [Python](python.md)). Fortalezas: **versionado y revisado en Git** como cualquier código, reuso real (fixtures, helpers), lógica compleja natural (loops, datos generados, setup/teardown programático), e integración directa con tu CI y tus otros tests. Es lo que sostiene una **suite automatizada seria**.

El criterio —y no es "uno reemplaza al otro"—: **Postman para explorar y para checks manuales/compartibles; código en el repo para la suite automatizada que corre en CI y es la fuente de verdad.** Muchos equipos usan los dos: Postman para tantear un endpoint nuevo o documentar, y la suite en código (pytest + httpx) como el conjunto de regresión real. El antipatrón es construir **toda** la automatización en colecciones de Postman: termina siendo difícil de mantener, revisar y razonar comparado con tests en código. La frase mental: **Postman/Newman brilla para explorar, documentar y compartir colecciones (y correrlas en CI con Newman); el código (httpx + pytest) es lo que sostiene la suite automatizada seria —versionada, reutilizable, integrada al repo—; usá Postman para tantear y código para la regresión, no toda la automatización en colecciones—.**

**Ejercicios 2**
2.1 ¿Qué es Postman y qué es Newman, y para qué brilla cada uno?
2.2 ¿Qué ventajas da escribir los tests de API como código en el repo?
2.3 ¿Cuál es el criterio para elegir entre Postman y código, y por qué no es "uno reemplaza al otro"?
2.4 ¿Cuál es el antipatrón respecto a Postman?

---

## Módulo 3 — Anatomía de un test de API: status, body, headers

**Teoría.** Un test de API es la estructura **AAA** de [Testing](testing.md) sobre HTTP: armás el request (arrange), lo enviás (act), y verificás la respuesta (assert). En Python lo hacés con **`httpx`** (un cliente HTTP moderno, sync y async) y **`pytest`**. El cliente se arma una vez como **fixture** ([Python](python.md)) y se reusa.

```python
import httpx
import pytest

@pytest.fixture
def client():
    # base_url y timeout en un solo lugar; se cierra solo al terminar
    with httpx.Client(base_url="http://localhost:8000", timeout=10) as c:
        yield c

def test_crear_tarea(client):
    # Act: enviar el request
    r = client.post("/tasks", json={"titulo": "Comprar pan"})

    # Assert: las tres dimensiones de una respuesta HTTP
    assert r.status_code == 201                      # 1) el status code
    body = r.json()
    assert body["titulo"] == "Comprar pan"           # 2) el body
    assert "id" in body
    assert r.headers["content-type"].startswith("application/json")  # 3) los headers
```

Las tres dimensiones que verifica un test de API:

- **Status code**: ¿el endpoint devolvió el código correcto? `201` al crear, `200` al leer, `204` al borrar, `400`/`422` ante input inválido, `401`/`403` sin permiso, `404` si no existe. El status es lo primero que afirmás —y un error común es chequear solo el body y olvidar que el status también es parte del contrato—.
- **Body**: ¿el contenido es el esperado? Los campos, sus valores, su **forma** (módulo 4).
- **Headers**: a veces parte del contrato —`Content-Type`, `Location` en un `201`, headers de paginación, de rate limit, de caché—.

`httpx` te da todo eso: `r.status_code`, `r.json()` / `r.text`, `r.headers`. Y como cualquier test ([Testing](testing.md)), un test de API es **AAA**, **determinista** y **aislado** (módulo 6). La nota de API vs Supertest: acá `httpx` le pega a la API **por la red** (a un servidor corriendo), no la levanta in-process —es testing de caja negra de la API tal como la ve un cliente real—. La frase mental: **un test de API es AAA sobre HTTP con httpx + pytest: enviás el request y afirmás las tres dimensiones de la respuesta —status code (parte del contrato, no lo olvides), body y headers—, con el cliente como fixture reutilizable—.**

**Ejercicios 3**
3.1 Mapeá las tres partes de AAA a un test de API con httpx.
3.2 ¿Cuáles son las tres dimensiones de una respuesta HTTP que verificás, y cuál es el error común?
3.3 ¿Por qué conviene armar el cliente httpx como una fixture de pytest?
3.4 ¿En qué se diferencia este testing "por la red" del de Supertest in-process?

---

## Módulo 4 — Qué afirmar: el contrato, no solo el `200`

**Teoría.** El error más común del API testing junior: afirmar solo `assert r.status_code == 200` y darse por satisfecho. Un `200` te dice que el endpoint **no explotó**, no que **devolvió lo correcto**. Testear de verdad es verificar el **contrato**: la **forma** y las **reglas** de la respuesta, no solo que respondió.

Qué incluye el contrato de una respuesta:

- **La estructura (schema)**: qué campos tiene, de qué tipo, cuáles son obligatorios. En Python la forma idiomática es validar con **Pydantic** ([FastAPI](fastapi.md)): definís el modelo esperado y lo validás contra la respuesta. Si falta un campo o cambió de tipo, el test falla con un error claro —cazás un cambio de contrato accidental—.

```python
from pydantic import BaseModel

class TareaOut(BaseModel):
    id: int
    titulo: str
    completada: bool

def test_contrato_de_tarea(client):
    r = client.get("/tasks/1")
    assert r.status_code == 200
    tarea = TareaOut.model_validate(r.json())   # valida estructura y tipos; lanza si no cumple
    assert tarea.completada is False             # + aserciones de valor/regla de negocio
```

- **Los valores y las reglas de negocio**: que `total = suma de items`, que `completada` arranca en `False`, que la fecha de creación está seteada. El schema valida la *forma*; vos validás el *significado*.
- **Las respuestas de error** (lo que más se olvida): un input inválido debe dar `422` **con un body de error útil**; un recurso inexistente, `404`; sin permiso, `403`. Y el **body de error también es contrato**: su forma se afirma igual que la del happy path —cada vez más APIs serias lo estandarizan con **`problem+json` (RFC 9457)**, así que el test verifica que el error trae los campos esperados (`type`, `title`, `detail`, `status`), no solo el código—. **El camino de error es parte del contrato tanto como el happy path** —y un test que solo prueba el caso feliz miente sobre la robustez de la API—.

La disciplina: **afirmá el contrato completo —status + estructura + valores + errores—, no un `200` solitario.** Validar el schema con Pydantic conecta directo con lo que [FastAPI](fastapi.md) hace en el borde de entrada: ahí Pydantic valida lo que **entra**; acá lo usás para validar lo que **sale**. La frase mental: **un `200` dice "no explotó", no "devolvió lo correcto"; testeá el contrato —estructura (valida el schema con Pydantic), valores y reglas de negocio, y los caminos de error (4xx con su body)—, porque el camino de error es parte del contrato tanto como el happy path—.**

**Ejercicios 4**
4.1 ¿Por qué `assert status_code == 200` solo es insuficiente? ¿Qué te dice y qué no?
4.2 ¿Qué incluye "el contrato" de una respuesta? Nombrá las tres cosas.
4.3 ¿Cómo validás la estructura de una respuesta en Python, y con qué concepto de FastAPI conecta?
4.4 ¿Por qué hay que testear las respuestas de error y no solo el happy path?

---

## Módulo 5 — Probar las capas: funcional, auth/authz, validación, casos borde

**Teoría.** Un endpoint no se prueba con un test; se prueba con un **conjunto** que cubre sus distintas dimensiones. Las capas a cubrir, de la más obvia a la más olvidada:

- **Funcional (happy path)**: ¿el endpoint hace lo que dice? Crear devuelve el recurso creado, listar devuelve la lista, etc. El **ciclo CRUD completo** es un patrón clave: crear → leerlo → actualizarlo → borrarlo → verificar que ya no está. Prueba la coherencia del recurso de punta a punta.
- **Validación (inputs inválidos → 4xx)**: campos faltantes, tipos equivocados, valores fuera de rango, formatos inválidos. Cada uno debe dar el error correcto (`400`/`422`), no un `500` ni un `200` que guarda basura. Esto es lo que [FastAPI](fastapi.md)/Pydantic valida en el borde —y vos verificás que efectivamente lo rechaza—.
- **Autenticación y autorización**: dos cosas distintas que se confunden. **Autenticación** (¿quién sos?): sin token o con token inválido → `401`. **Autorización** (¿podés hacer esto?): un usuario que intenta tocar un recurso de **otro** → `403`. Este último —el control de acceso entre usuarios— es de lo más crítico y de lo que más se escapa: es la base de [seguridad](autenticacion.md) y conecta con el testing detrás de auth de [Testing](testing.md) módulo 8. **Un test que verifica que el usuario A NO puede ver el recurso del usuario B vale oro** (previene fugas de datos, el mismo riesgo multi-tenant de [RAG](rag.md)/[Vector Databases](vector-dbs.md)). Esto tiene nombre de mercado: **BOLA** (Broken Object Level Authorization), también llamado **IDOR**, y es el **#1 del OWASP API Security Top 10** —el vocabulario exacto que esperan en una entrevista de SDET—.
- **Casos borde**: listas vacías, paginación (página fuera de rango, límites), filtros, ordenamientos, valores límite, idempotencia (¿qué pasa si mando el mismo `POST` dos veces?), concurrencia.

La regla: **para cada endpoint, pensá en las cuatro capas, no solo en el happy path.** El happy path es donde menos bugs hay (es lo que el dev probó a mano); los bugs viven en la validación floja, en los permisos mal chequeados y en los casos borde. Por eso un buen test de API es, sobre todo, un buen **catálogo de lo que puede salir mal**. La frase mental: **cada endpoint se prueba por capas —funcional (CRUD completo), validación (inputs inválidos → 4xx), auth (401 sin token) y authz (403 sobre recursos ajenos), y casos borde (vacíos, paginación, idempotencia)—; el happy path es donde menos bugs hay, así que el valor está en probar lo que puede salir mal, y el test de "A no puede ver lo de B" es de los más valiosos—.**

**Ejercicios 5**
5.1 ¿Qué es el "ciclo CRUD completo" como test y qué prueba?
5.2 Diferenciá autenticación de autorización con el status code que devuelve cada falla.
5.3 ¿Por qué el test de "el usuario A no puede ver el recurso de B" es tan valioso? Con qué riesgo conecta.
5.4 ¿Por qué se dice que los bugs no viven en el happy path? ¿Dónde viven?

---

## Módulo 6 — Datos, aislamiento y entornos (setup/teardown, nunca contra prod)

**Teoría.** Las mismas reglas de tests confiables de [Testing](testing.md) y [Playwright](playwright.md) (deterministas, aislados, repetibles) aplican al API testing, con sus particularidades de manejo de **estado** y **datos**.

- **Aislamiento y datos deterministas**: cada test arranca de un **estado conocido** y no depende de otros ni del orden. El problema típico: un test que crea una tarea y otro que cuenta tareas y espera "3" —corren juntos y se pisan—. Curas: **datos únicos por test** (un título/email con uuid), y **setup/teardown** que deja el estado limpio. El `POST` de setup y el `DELETE` de teardown pueden hacerse por la **propia API** (más realista) o directo contra la **base de datos** (más rápido); las fixtures de pytest (`yield` + limpieza) son el lugar natural para esto. Y este aislamiento es justo lo que te habilita a **correr la suite en paralelo** (`pytest -n`, xdist) sin que los tests se pisen —ese paralelismo, que acelera el CI, lo retomamos en el módulo 10—.

```python
import uuid, pytest

@pytest.fixture
def tarea_creada(client):
    # setup: crear un recurso único
    titulo = f"tarea-{uuid.uuid4()}"
    r = client.post("/tasks", json={"titulo": titulo})
    tarea = r.json()
    yield tarea
    # teardown: limpiar, pase lo que pase en el test
    client.delete(f"/tasks/{tarea['id']}")
```

- **Entornos**: corrés contra un entorno de **test/staging dedicado**, **nunca contra producción** —un test que crea/borra datos contra prod es un desastre esperando a pasar—. La URL base y las credenciales salen de **variables de entorno** (no hardcodeadas, no en el repo: [Docker](docker-deploy.md) módulo 5, secretos). Una colección de Postman maneja esto con "environments"; en código, con env vars / un `.env.test`.
- **Bases de datos reales y efímeras**: para tests de API que tocan persistencia, la opción robusta es **testcontainers** ([Testing](testing.md) módulo 6): levantás un Postgres real en un contenedor descartable, corrés los tests, lo tirás. Te da una DB real (no un mock que miente) y aislada (cada corrida arranca limpia).

La regla, la misma de siempre: **un test que falla un día y pasa al otro sin que cambie el código es inútil.** El aislamiento y los datos deterministas no son prolijidad; son lo que hace que un rojo signifique "hay un bug" y no "mala suerte". La frase mental: **cada test de API arranca de un estado conocido con datos únicos (uuid) y setup/teardown vía fixtures (por API o DB), corre contra un entorno de test dedicado —jamás prod— con credenciales en variables de entorno, e idealmente sobre una DB real efímera (testcontainers); el aislamiento es lo que hace que un rojo signifique un bug y no mala suerte—.**

**Ejercicios 6**
6.1 Dá un ejemplo de dos tests de API que se pisan por falta de aislamiento y cómo lo arreglás.
6.2 ¿Dónde ponés el setup/teardown en pytest y qué dos formas hay de crear/limpiar datos?
6.3 ¿Por qué nunca corrés tests contra producción, y de dónde salen la URL y las credenciales?
6.4 ¿Qué aporta testcontainers a un test de API que toca la base de datos?

---

## Módulo 7 — Contract testing: el problema N×M entre servicios y Pact

**Teoría.** Acá entra el tema más senior del módulo. En una arquitectura de **varios servicios** (microservicios, o tu backend + el front, o tu API + la de un tercero), un servicio **consumidor** llama a un servicio **proveedor**. El problema: el proveedor puede **cambiar su API** (renombrar un campo, cambiar un tipo, sacar un endpoint) y **romper al consumidor sin enterarse** —los tests de cada servicio pasan en aislamiento, pero juntos explotan en producción—.

La solución obvia parece ser tests de **integración E2E entre servicios** (levantar todos y probarlos juntos), pero eso es **lento, frágil y difícil de armar** (necesitás todos los servicios corriendo, con sus datos, a la vez). Para eso existe el **contract testing**, y su forma más usada es el **consumer-driven contract testing** con **Pact**:

1. **El consumidor define sus expectativas.** En su propio test, el consumidor declara: "cuando le pido `GET /tasks/1` al proveedor, espero un `200` con un body de esta forma". Ese test corre contra un **mock** del proveedor y, como subproducto, **genera un "pact"**: un archivo (JSON) que es el **contrato** —exactamente lo que el consumidor necesita—.

```python
# Lado CONSUMIDOR (pact-python v3) — forma conceptual, verificá la API exacta de tu versión.
# La v3 (core en Rust, default Pact Spec V4) es la API actual; la v2 (Consumer/has_pact_with) quedó legacy.
from pact import Pact
from pact.v3 import match

pact = Pact("frontend", "task-api")
(pact
 .upon_receiving("un pedido de la tarea 1")
 .given("existe la tarea 1")                          # estado del provider
 .with_request("GET", "/tasks/1")
 .will_respond_with(200)
 .with_body({                                          # MATCHERS: el pact afirma la FORMA/tipo, no valores fijos
     "id": match.int(),                                # (valores literales → contratos frágiles que rompen ante
     "titulo": match.str(),                            #  cualquier cambio de dato; un contrato describe estructura)
     "completada": match.bool(),
 }))

with pact.serve() as mock:                             # levanta el mock; al cerrar escribe el pact (el contrato)
    r = httpx.get(f"{mock.url}/tasks/1")
    assert r.status_code == 200
# el test del consumidor corre contra el mock y, como subproducto, genera el pact
```

2. **El proveedor verifica el contrato.** El proveedor toma ese pact y corre una **verificación**: ¿mi API real cumple lo que el consumidor espera? Si el proveedor cambió algo que rompe el contrato, **su** build falla —antes de deployar, no en producción—. Es la mitad que más cuesta en sistemas reales (hay que saber dejar al provider en el **estado** que el pact declara con `given`).

```python
# Lado PROVEEDOR (pact-python v3) — forma conceptual, verificá la API de tu versión
from pact import Verifier

# levantás tu API real (en localhost:8000) y la verificás contra los pacts del broker
verifier = (Verifier("task-api")
            .add_transport(url="http://localhost:8000")
            .broker_source("https://mi-broker.pactflow.io"))   # o .add_source("pacts/") para archivos locales
verifier.verify()                                               # falla si la API real no cumple algún pact
```

3. **El Pact Broker** es el intermediario: almacena los pacts y las verificaciones, y habilita el gate **`can-i-deploy`** —"¿es seguro deployar esta versión del proveedor sin romper a ningún consumidor?"— en el CI (módulo 10). El gate es un comando concreto:

```bash
# ¿puedo deployar esta versión del proveedor a producción sin romper consumidores?
pact-broker can-i-deploy --pacticipant task-api --version "$GIT_SHA" --to-environment production
# sale 0 (verde, deploy seguro) o ≠0 (rojo, frena el deploy) según las verificaciones registradas en el broker
```

> Nota de empaquetado: el paquete se instala como `pact-python` pero se **importa como `pact`**; la v3 trae el **core nativo en Rust** (rueda con FFI, sin Java/Ruby), y el **Pact Broker corre como servicio aparte** (Docker / PactFlow), no es parte de la librería.

La diferencia clave con un test de integración E2E: en contract testing **los servicios nunca corren juntos**. El consumidor prueba contra un mock; el proveedor verifica contra un contrato. Cada uno en su pipeline, rápido y aislado, pero la combinación **garantiza la compatibilidad**. "Consumer-driven" significa que **el consumidor define qué necesita** (no el proveedor qué ofrece) —así el proveedor sabe exactamente qué partes de su API importan y cuáles puede cambiar libremente—. La frase mental: **el contract testing (Pact) evita que un proveedor rompa a sus consumidores sin levantar todos los servicios juntos: el consumidor declara qué espera y genera un contrato (pact), el proveedor verifica que su API real lo cumple, y el broker habilita un gate can-i-deploy —compatibilidad garantizada con tests rápidos y aislados, no con un E2E frágil entre servicios—.**

**Ejercicios 7**
7.1 ¿Qué problema resuelve el contract testing en una arquitectura de varios servicios?
7.2 ¿Por qué los tests de integración E2E entre servicios no son una buena solución a ese problema?
7.3 Describí el flujo consumer-driven con Pact: ¿qué hace el consumidor, qué hace el proveedor, y qué genera cada uno?
7.4 ¿Qué significa "consumer-driven" y qué rol cumple el Pact Broker (incluido `can-i-deploy`)?

---

## Módulo 8 — OpenAPI como contrato: validación de schema y testing dirigido por spec

**Teoría.** Hay otro "contrato" que ya tenés casi gratis: la **especificación OpenAPI** de tu API. [FastAPI](fastapi.md) **genera el OpenAPI automáticamente** desde tus modelos Pydantic y tus path operations —es la descripción formal, machine-readable, de todos tus endpoints: rutas, parámetros, schemas de request y response, códigos de estado—. Esa spec es una fuente de verdad que el testing puede explotar de dos formas:

- **Validar respuestas contra el schema OpenAPI.** En vez de escribir a mano el modelo esperado (módulo 4), validás que cada respuesta **cumpla el schema que la propia spec declara**. Si tu API devuelve algo que no matchea su propio OpenAPI, es un bug de contrato —la implementación se separó de la spec—.

- **Spec-driven / property-based testing con Schemathesis.** Acá está lo potente: **Schemathesis** lee tu OpenAPI y **genera automáticamente cientos de casos de prueba** —incluyendo entradas raras, límite y malformadas (*fuzzing*)— contra cada endpoint, verificando propiedades generales: que la API no tire `500` ante input válido según el schema, que las respuestas cumplan el schema declarado, que respete los códigos de estado. Es **property-based testing** (probar propiedades que deben valer para *toda* entrada) aplicado a APIs, y encuentra bugs que vos no pensaste en escribir como caso.

```bash
# Schemathesis: genera y corre cientos de casos contra tu API desde su OpenAPI
schemathesis run http://localhost:8000/openapi.json --checks all
# Si la spec está en archivo (Schemathesis 4.x), el target va aparte con --url:
#   schemathesis run openapi.yaml --url http://localhost:8000
# fases: examples, coverage, fuzzing, stateful — inputs raros/malformados, conformidad al schema y 500 inesperados
```

La relación con los otros "contratos" del módulo: el **schema OpenAPI** describe la API de **un** servicio (su forma pública); **Pact** (módulo 7) captura lo que un **consumidor específico** necesita de un proveedor. No compiten: OpenAPI valida que la implementación cumple su spec publicada; Pact valida que dos servicios concretos siguen siendo compatibles. Y ambos conectan con la idea central del módulo: **el testing serio verifica contratos, no anécdotas.** La frase mental: **el OpenAPI que FastAPI genera de tus modelos Pydantic es un contrato machine-readable que podés validar (las respuestas cumplen su propio schema) y explotar con Schemathesis (genera cientos de casos/fuzzing desde la spec, property-based) —complementa a Pact: OpenAPI valida que un servicio cumple su spec, Pact que dos servicios siguen compatibles—.**

**Ejercicios 8**
8.1 ¿De dónde sale el OpenAPI de una API FastAPI y qué describe?
8.2 ¿Qué dos formas de testing habilita tener la spec OpenAPI?
8.3 ¿Qué hace Schemathesis y por qué "property-based / fuzzing" encuentra bugs que un test escrito a mano no?
8.4 ¿En qué se diferencia el contrato OpenAPI del contrato Pact, y por qué no compiten?

---

## Módulo 9 — Performance y load testing en la capa de API

**Teoría.** Hasta acá probamos **correctitud** (¿hace lo correcto?); la capa de API es también donde se prueba el **rendimiento** (¿aguanta la carga?, ¿responde rápido?). Es una dimensión distinta y se confunde con la funcional. Los tipos:

- **Load testing**: simular **muchos usuarios concurrentes** haciendo requests, para ver cómo se comporta la API bajo carga esperada. ¿La latencia se mantiene? ¿La throughput escala? ¿Algo se cae?
- **Stress testing**: subir la carga **más allá** de lo esperado hasta encontrar el punto de quiebre (¿a cuántos usuarios empieza a fallar?).
- **Spike testing**: un pico súbito de tráfico (¿aguanta un golpe repentino?).

La herramienta idiomática en Python es **Locust**: definís el comportamiento de un "usuario" como código y Locust simula miles de ellos, reportando latencias y throughput.

```python
from locust import HttpUser, task, between

class UsuarioApi(HttpUser):
    wait_time = between(1, 3)          # cada usuario espera 1-3s entre acciones

    @task
    def listar_tareas(self):
        self.client.get("/tasks")

    @task
    def crear_tarea(self):
        self.client.post("/tasks", json={"titulo": "carga"})
# correr:  locust -f load.py --host http://localhost:8000   (UI web para fijar usuarios y rampa)
```

(Otra herramienta muy usada, fuera de Python, es **k6** —scripts en JavaScript, muy popular en CI y con buen soporte de SLOs/thresholds—; acá elegimos **Locust** por seguir el stack Python del track, pero saber que k6 existe es criterio de herramienta esperable en un SDET.) Las métricas que mirás son las de [observabilidad](observabilidad.md), ahora del lado del test: **latencia en percentiles (p95/p99, no el promedio** —el promedio esconde la cola que sufren los usuarios reales—), **throughput** (requests/seg), y **tasa de errores** bajo carga. El objetivo suele ser validar un **SLO** ("p95 < 300 ms con 500 usuarios concurrentes").

El criterio: **el load testing es para cuando el rendimiento es un requisito**, no para toda API. Una API interna de bajo tráfico no lo necesita; una que va a recibir picos o tiene SLAs de latencia, sí. Y conecta fuerte con [IA](deploy-ai.md): una API que sirve un LLM tiene un perfil de carga particular (requests largos, caros, I/O-bound) y su load testing mira el costo y la latencia de inferencia, no solo requests/seg. La frase mental: **el load testing (con Locust) simula usuarios concurrentes para validar que la API aguanta la carga y cumple su SLO de latencia —mirás percentiles p95/p99, throughput y tasa de error, no el promedio—; es una dimensión distinta de la correctitud, y se hace cuando el rendimiento es un requisito, no para toda API—.**

**Ejercicios 9**
9.1 Diferenciá load, stress y spike testing.
9.2 ¿Qué hace Locust y cómo definís el comportamiento de un usuario?
9.3 ¿Por qué mirás percentiles (p95/p99) y no el promedio de latencia? (conectá con observabilidad)
9.4 ¿Cuándo hacés load testing y cuándo no? ¿Qué tiene de particular el de una API de IA?

---

## Módulo 10 — API testing en CI: testcontainers, Newman y verificación de contratos

**Teoría.** Una suite de API que no corre en **CI** automáticamente en cada cambio no sirve para prevenir regresiones —es un script que alguien corre cuando se acuerda—. Integrarla al pipeline ([Docker y CI/CD](docker-deploy.md)) es lo que la vuelve una red de seguridad real. Las piezas:

- **La API tiene que estar disponible para los tests.** Dos enfoques: levantar la app + sus dependencias en el pipeline (con **docker compose** o **testcontainers**, [Testing](testing.md) módulo 6 —un Postgres efímero, la app, y los tests pegándole), o correr contra un entorno de **staging** desplegado. El primero da aislamiento total por corrida; el segundo prueba algo más cercano a producción.
- **Correr la suite pytest** como un step del pipeline, en paralelo si conviene (`pytest -n` con xdist). Un test rojo **frena el merge/deploy**.
- **Newman para las colecciones Postman** (si las usás): `newman run coleccion.json -e entorno.ci.json` corre la colección en CI y reporta. Es cómo las colecciones de Postman entran al pipeline.
- **Verificación de contratos en el pipeline** (módulo 7): el build del **consumidor** publica su pact al broker; el build del **proveedor** verifica los pacts; y el gate **`can-i-deploy`** consulta al broker "¿esta versión es compatible con todo lo desplegado?" **antes** de permitir el deploy. Así el contract testing cumple su promesa: cazar incompatibilidades en CI, no en producción.
- **Schemathesis en CI** (módulo 8): correrlo contra la API levantada como un step más, para fuzzing automático en cada cambio.

El principio, el de [Docker/CI](docker-deploy.md): **lo que no corre automáticamente, no existe como garantía.** Tests de API en el repo que solo corren a mano se desactualizan y se saltean bajo presión; los mismos tests en CI, bloqueando el merge, son los que de verdad previenen regresiones. La frase mental: **la suite de API vale como red de seguridad solo si corre en CI en cada cambio: levantás la API con docker compose/testcontainers (o staging), corrés pytest (y Newman para colecciones), verificás los pacts con el gate can-i-deploy antes de deployar, y sumás Schemathesis para fuzzing —porque lo que no corre automáticamente no existe como garantía—.**

**Ejercicios 10**
10.1 ¿Cuáles son las dos formas de tener la API disponible para los tests en CI, y qué da cada una?
10.2 ¿Cómo entran las colecciones de Postman al pipeline?
10.3 ¿Cómo se cierra el ciclo de contract testing en CI? (publicar, verificar, can-i-deploy)
10.4 ¿Por qué "lo que no corre automáticamente no existe como garantía"?

---

## Módulo 11 — Testear APIs de IA y el criterio de cierre

**Teoría.** El cierre, con el ángulo **AI QA**: ¿cómo testeás un **endpoint que devuelve la salida de un LLM** —un `/chat`, un `/resumen`, un `/rag/query`— si la respuesta es **no determinista**? Es el mismo problema que en [Playwright](playwright.md) módulo 11, ahora en la capa de API, y la respuesta es la misma idea: **separá lo determinista de lo no determinista.**

Qué **sí** testeás en la API de una feature de IA (determinista, en tu suite normal):

- **El contrato y la plomería**: status `200`, el body tiene la **forma** esperada (un JSON con `respuesta`, `fuentes`, etc.), el **streaming** funciona (si es SSE, que lleguen los chunks y se ensamble bien), los **errores** se manejan (`429` del proveedor, timeout → un error limpio, no un `500` crudo).
- **Aserciones de estructura/contrato, no de texto exacto**: `assert "respuesta" in body` y que sea no vacía, que `fuentes` sea una lista, que la respuesta **contenga** una entidad clave —nunca `assert body["respuesta"] == "texto exacto"`, que sería flaky por diseño—.
- **Guardarraíles y límites**: que un input que debe ser rechazado devuelva el error correcto, que se respeten límites de longitud, que no se filtren datos de otro tenant (el test de authz del módulo 5, **crítico** en RAG). El **red-teaming del endpoint de IA** —prompt injection, jailbreaks, fuga de prompt del sistema— es **otra capa** y vive en [Seguridad de IA](seguridad-ia.md): acá testeás el contrato y la plomería; allá, los ataques específicos de LLM.
- **SLAs operacionales**: latencia (p95) y, si aplica, costo por request dentro de presupuesto ([Deploy de IA](deploy-ai.md)) —la API de IA tiene SLAs de latencia/costo que un test puede verificar—.

Qué **NO** testeás con un test de API: **si la respuesta del LLM es *buena*.** La calidad de la salida no determinista es trabajo de **[evals](evals.md)** (golden sets, LLM-as-judge) y de las **online evals** en producción ([Deploy de IA](deploy-ai.md)), no de la suite de API. Confundir las dos lleva a tests flaky o a una falsa sensación de cobertura ("pasó el test de API" ≠ "la IA responde bien"). Para tests deterministas de la plomería, **mockeás la llamada al LLM** (módulo 6) y verificás que tu endpoint arma bien el request, parsea bien la respuesta y maneja los errores.

**Y el criterio de cierre del módulo y del track QA**: el API testing es el **punto dulce** —probá acá todo lo que no necesita la UI (módulo 1), verificá el **contrato** y no un `200` (módulo 4), cubrí las capas incluida la **autorización** (módulo 5), aislá y usá datos deterministas (módulo 6), garantizá compatibilidad entre servicios con **contract testing** (módulo 7) y conformidad con **OpenAPI/Schemathesis** (módulo 8), medí el rendimiento cuando es requisito (módulo 9), y corré **todo en CI** (módulo 10)—. Sobre esa base de muchos tests de API rápidos y estables, ponés **pocos E2E** ([Playwright](playwright.md)) para los flujos críticos de UI, y **evals** ([evals](evals.md)) para la calidad de la IA. La frase mental: **una API de IA se testea separando lo determinista (contrato, plomería, streaming, errores, authz, SLAs → suite de API, mockeando el LLM) de lo no determinista (¿la respuesta es buena? → evals, no API testing); y el criterio que cierra el track QA es la pirámide bien armada —muchos tests de API que verifican contratos, pocos E2E para la UI crítica, y evals para la IA—.**

**Ejercicios 11**
11.1 ¿Por qué afirmar el texto exacto de la respuesta de un endpoint de IA es flaky por diseño?
11.2 Nombrá cuatro cosas deterministas que SÍ testeás en la API de una feature de IA.
11.3 ¿Qué NO se prueba con un test de API en una feature de IA, y con qué se prueba?
11.4 Resumí el criterio de cierre del track QA: ¿qué se prueba con API testing, qué con E2E y qué con evals?

---

## Proyecto integrador (capstone): la suite de API de la Task API

El ejercicio que cierra el módulo y el track QA, y que mostrás en una entrevista de QA/SDET. No te damos la solución; te damos los **criterios de aceptación**. Construilo con httpx + pytest contra la Task API (o tu API con auth + CRUD).

**Qué construir.** Una suite de API que cubra las capas, no solo el happy path:

1. **CRUD completo** (módulo 5): crear → leer → actualizar → borrar → verificar que no está, afirmando el **contrato** con Pydantic (módulo 4), no solo el status.
2. **Validación**: inputs inválidos (campo faltante, tipo equivocado) → `422` con body de error.
3. **Auth/authz** (módulo 5): sin token → `401`; el usuario A intentando ver/modificar el recurso de B → `403`. (El test que previene fugas.)
4. **Aislamiento** (módulo 6): datos únicos por test (uuid), setup/teardown vía fixtures, corriendo contra un entorno de test (idealmente DB efímera con testcontainers).
5. **Todo en CI** (módulo 10): la suite corre en el pipeline y un rojo frena el merge.

**Criterios de aceptación.**
- [ ] Cada test afirma el **contrato** (estructura + valores), no solo `status_code == 200`.
- [ ] Hay tests de **error** (4xx) además del happy path, con su body verificado.
- [ ] Existe el test de **autorización** "A no puede acceder al recurso de B" (`403`).
- [ ] Los tests **pasan 10 veces seguidas** sin fallos intermitentes (aislamiento + datos únicos).
- [ ] La suite corre **en CI** (con la API levantada vía compose/testcontainers o staging) y bloquea el merge si falla.
- [ ] Las credenciales/URL salen de **variables de entorno**, no hardcodeadas.

**Extensiones (suben el nivel).**
- **Schemathesis** (módulo 8): corré fuzzing contra el OpenAPI de la API y resolvé lo que encuentre.
- **Contract testing** (módulo 7): si tenés front + back (o dos servicios), escribí un pact del consumidor y verificalo en el proveedor.
- **Load testing** (módulo 9): un script de Locust para un endpoint clave; validá un SLO de p95.
- **Feature de IA** (módulo 11): testeá un endpoint `/chat` o `/rag` **mockeando el LLM** —contrato, streaming, manejo de `429`/timeout y authz— y dejá la calidad para una eval aparte ([evals](evals.md)).

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 En la capa de integración/servicio: por debajo del E2E de browser y por encima de los unitarios.
    Prueba los endpoints HTTP directamente (request → respuesta), sin navegador ni frontend.
1.2 (1) Más rápido: es un request HTTP, no arranca navegador ni renderiza. (2) Más estable: no hay
    timing de UI ni locators que se rompen, la API es un contrato más estable que el DOM. (3) Más
    preciso: un fallo apunta al backend/contrato, no a "algo de la UI".
1.3 Mover hacia abajo todo lo que se pueda: lo que se verifica pegándole a la API (lógica, validaciones,
    status, permisos, casos borde) se prueba acá; el E2E queda solo para los pocos flujos de UI críticos.
1.4 Se parece en que ambos prueban endpoints. Se diferencia en que Supertest levanta la app in-process;
    el API testing de caja negra le pega a la API CORRIENDO por HTTP real, como cualquier cliente externo.
```

### Módulo 2
```
2.1 Postman es una herramienta gráfica para armar requests, agruparlos en colecciones y agregar
    aserciones; brilla para explorar, documentar y compartir. Newman es su runner CLI: corre una
    colección en CI.
2.2 Versionado y revisión en Git como cualquier código, reuso real (fixtures/helpers), lógica compleja
    natural (loops, datos generados, setup/teardown), e integración directa con el CI y los demás tests.
2.3 Postman para explorar y checks manuales/compartibles; código (httpx+pytest) para la suite
    automatizada de regresión que corre en CI. No se reemplazan: muchos equipos usan los dos para cosas
    distintas (tantear vs regresión).
2.4 Construir TODA la automatización en colecciones de Postman: termina difícil de mantener, versionar y
    revisar comparado con tests en código.
```

### Módulo 3
```
3.1 Arrange: armar el request (cliente, URL, body). Act: enviarlo (client.post/get). Assert: verificar
    la respuesta (status, body, headers).
3.2 Status code, body y headers. El error común es chequear solo el body y olvidar que el status code
    (y a veces los headers) también son parte del contrato.
3.3 Para centralizar base_url/timeout/auth en un solo lugar, reusarlo en todos los tests, y que se cierre
    solo al terminar (yield). Evita repetir la configuración del cliente en cada test.
3.4 httpx le pega a la API por la red (a un servidor corriendo), testing de caja negra como un cliente
    real; Supertest la levanta in-process. El primero prueba la API tal como la ve un cliente externo.
```

### Módulo 4
```
4.1 Insuficiente porque un 200 dice "no explotó", no "devolvió lo correcto". Te dice que el endpoint
    respondió sin error, pero no si el contenido, la forma y las reglas son las esperadas.
4.2 (1) La estructura/schema (qué campos, de qué tipo, cuáles obligatorios). (2) Los valores y reglas de
    negocio. (3) Las respuestas de error (4xx con su body).
4.3 Validando la respuesta contra un modelo Pydantic (Modelo.model_validate(r.json())), que lanza si la
    forma/tipos no cumplen. Conecta con la validación de Pydantic en FastAPI: ahí valida lo que ENTRA,
    acá lo usás para validar lo que SALE.
4.4 Porque el camino de error es parte del contrato tanto como el happy path: un input inválido debe dar
    422 con body útil, un inexistente 404, sin permiso 403. Un test que solo prueba el caso feliz miente
    sobre la robustez de la API.
```

### Módulo 5
```
5.1 Crear un recurso → leerlo → actualizarlo → borrarlo → verificar que ya no está. Prueba la coherencia
    del recurso de punta a punta (que cada operación deja el estado correcto para la siguiente).
5.2 Autenticación (¿quién sos?): sin token o token inválido → 401. Autorización (¿podés hacer esto?): un
    usuario tocando un recurso de otro → 403.
5.3 Porque previene fugas de datos entre usuarios/tenants: verifica que el control de acceso funciona.
    Es el BOLA/IDOR, #1 del OWASP API Security Top 10. Conecta con el riesgo multi-tenant de
    RAG/Vector Databases (que A reciba data de B) y con seguridad.
5.4 Porque el happy path es lo que el dev ya probó a mano; los bugs viven en la validación floja, los
    permisos mal chequeados y los casos borde (vacíos, paginación, idempotencia, concurrencia).
```

### Módulo 6
```
6.1 Ej.: un test crea una tarea y otro cuenta tareas esperando "3"; corriendo juntos se pisan. Se
    arregla con datos únicos por test (uuid) y setup/teardown que limpia, de modo que cada test no
    dependa del estado dejado por otro ni del orden.
6.2 En fixtures de pytest (setup antes del yield, teardown después). Las dos formas de crear/limpiar
    datos: por la propia API (POST/DELETE, más realista) o directo contra la base de datos (más rápido).
6.3 Porque un test que crea/borra datos contra prod es un desastre (corrompe datos reales). La URL base
    y las credenciales salen de variables de entorno (no hardcodeadas, no en el repo).
6.4 Una base de datos real y efímera: levanta un Postgres en un contenedor descartable, los tests corren
    contra él (no contra un mock que miente) y arranca limpio en cada corrida (aislamiento).
```

### Módulo 7
```
7.1 Que un servicio proveedor cambie su API y rompa a un consumidor sin enterarse: los tests de cada
    servicio pasan en aislamiento pero juntos explotan en producción.
7.2 Porque levantar todos los servicios juntos para probarlos es lento, frágil y difícil de armar
    (necesitás todos corriendo con sus datos a la vez); es justo lo que el contract testing evita.
7.3 El consumidor declara en su test qué espera del proveedor (request → respuesta esperada), corre
    contra un mock y GENERA un pact (el contrato). El proveedor toma ese pact y VERIFICA que su API real
    lo cumple; si lo rompe, su build falla.
7.4 "Consumer-driven" = el consumidor define qué necesita (no el proveedor qué ofrece), así el proveedor
    sabe qué partes importan. El Pact Broker almacena pacts y verificaciones y habilita el gate
    can-i-deploy ("¿es seguro deployar esta versión sin romper consumidores?").
```

### Módulo 8
```
8.1 FastAPI lo genera automáticamente desde los modelos Pydantic y las path operations. Describe
    formalmente (machine-readable) todos los endpoints: rutas, parámetros, schemas de request/response,
    códigos de estado.
8.2 (1) Validar que las respuestas cumplan el schema que la propia spec declara. (2) Spec-driven /
    property-based testing con Schemathesis (generar casos automáticamente desde la spec).
8.3 Lee el OpenAPI y genera cientos de casos (incluyendo entradas raras/límite/malformadas, fuzzing)
    verificando propiedades generales (no 500 ante input válido, respuestas conformes al schema). Encuentra
    bugs que no escribiste como caso porque prueba propiedades para TODA entrada, no ejemplos elegidos.
8.4 OpenAPI describe la API pública de UN servicio (que la implementación cumpla su spec); Pact captura
    lo que UN consumidor específico necesita de un proveedor (que dos servicios sigan compatibles). No
    compiten: cubren cosas distintas.
```

### Módulo 9
```
9.1 Load: muchos usuarios concurrentes con la carga esperada (¿se mantiene la latencia/throughput?).
    Stress: subir más allá de lo esperado hasta el punto de quiebre. Spike: un pico súbito de tráfico.
9.2 Locust simula muchos usuarios; definís el comportamiento de un usuario como una clase HttpUser con
    métodos @task que hacen los requests (y wait_time entre acciones). Corre miles de esos usuarios.
9.3 Porque el promedio esconde la cola: unos pocos requests muy lentos no mueven el promedio pero sí
    arruinan la experiencia de esos usuarios. El p95/p99 captura esa cola, que es lo que se siente.
9.4 Cuando el rendimiento es un requisito (picos esperados, SLAs de latencia), no para toda API. Una API
    de IA tiene un perfil particular (requests largos, caros, I/O-bound): su load testing mira latencia y
    costo de inferencia, no solo requests/seg.
```

### Módulo 10
```
10.1 (1) Levantar la app + dependencias en el pipeline (docker compose / testcontainers): aislamiento
     total por corrida. (2) Correr contra un entorno de staging desplegado: más cercano a producción.
10.2 Con Newman: newman run coleccion.json -e entorno.ci.json corre la colección como un step del
     pipeline y reporta.
10.3 El build del consumidor publica su pact al broker; el build del proveedor verifica los pacts; y el
     gate can-i-deploy consulta al broker si la versión es compatible con todo lo desplegado, ANTES de
     permitir el deploy.
10.4 Porque los tests que solo corren a mano se desactualizan y se saltean bajo presión; solo los que
     corren en CI bloqueando el merge previenen regresiones de verdad. La garantía es el automatismo.
```

### Módulo 11
```
11.1 Porque un LLM no devuelve el mismo texto dos veces (no determinista): afirmar el texto exacto va a
     fallar intermitentemente aunque el endpoint funcione bien.
11.2 (Cuatro de) el contrato/forma del body (200, JSON con los campos esperados); que el streaming/SSE
     funcione (lleguen y se ensamblen los chunks); el manejo de errores (429/timeout → error limpio, no
     500); aserciones de estructura/contains (no texto exacto); guardarraíles y authz (no filtrar data de
     otro tenant); SLAs de latencia p95 y costo.
11.3 NO se prueba si la respuesta del LLM es BUENA (la calidad de la salida no determinista). Eso se
     prueba con evals (golden sets, LLM-as-judge) y online evals en producción, no con tests de API. Para
     la plomería determinista se mockea la llamada al LLM.
11.4 API testing: todo lo que no necesita UI —contrato, capas, authz, casos borde— (muchos, rápidos,
     estables). E2E: los pocos flujos de UI críticos (Playwright). Evals: la calidad de la salida de IA.
     La pirámide bien armada: muchos API tests, pocos E2E, y evals para lo no determinista.
```

---

## Siguientes pasos

Con este módulo cerrás el track QA / Automation: el API testing en su lugar (el punto dulce, bajo el E2E), el panorama de herramientas (Postman/Newman para explorar, httpx+pytest para la suite seria), la anatomía de un test (status, body, headers), la disciplina de afirmar el **contrato** y no un `200`, las capas a cubrir (funcional, validación, auth/authz, casos borde), el aislamiento y los entornos, el **contract testing** con Pact (compatibilidad entre servicios sin E2E frágil), OpenAPI y Schemathesis (testing dirigido por spec y fuzzing), el load testing con Locust, la integración en CI, y —el ángulo AI QA— cómo testear APIs de IA separando el contrato determinista (suite de API) de la calidad no determinista ([evals](evals.md)). Con la regla de siempre: **muchos tests de API rápidos y estables que verifican contratos, pocos E2E para la UI crítica, y evals para la IA.** Con esto, el track QA queda completo —[Playwright](playwright.md) para la UI y API testing para lo de abajo— y se enlaza con todo el temario: la pirámide de [Testing](testing.md), el CI/CD de [Docker](docker-deploy.md), los percentiles de [Observabilidad](observabilidad.md), y las [evals](evals.md) y el [deploy de IA](deploy-ai.md) para la parte de IA. El perfil **AI QA / SDET** queda cubierto: sabés probar una app de punta a punta, sus APIs, los contratos entre servicios, el rendimiento, y —lo que distingue al rol— qué de un sistema de IA se prueba con tests deterministas y qué con evaluaciones.
