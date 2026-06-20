# FastAPI: APIs en Python para quien viene de NestJS

**Tu primer servicio Python · y la forma estándar de servir IA · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el segundo módulo del **track de IA en Python**: asume que ya hiciste [Python para devs TS/Node](python.md) (sobre todo type hints, Pydantic y `asyncio`) y que conocés [NestJS](nestjs.md) a fondo. FastAPI es **el framework web estándar del ecosistema Python de IA**: cuando querés exponer un modelo, un pipeline de RAG o un agente por HTTP, lo hacés con FastAPI. La buena noticia viniendo de NestJS: comparten ADN —decoradores para rutas, inyección de dependencias, validación por esquemas, OpenAPI automático—, así que vas a sentirte en casa rápido. La diferencia: FastAPI es **más liviano y explícito** (menos estructura impuesta) y está construido sobre **type hints + Pydantic** de punta a punta. El objetivo del módulo es que puedas escribir una API de producción —y, en particular, una que sirva IA con streaming— con criterio.

**Lo que asumimos.** [Python con type hints, Pydantic, asyncio](python.md); NestJS (controllers, DTOs, DI, pipes, filters, Swagger); HTTP y REST.

**Para practicar.**

```bash
uv add fastapi uvicorn[standard]     # framework + servidor ASGI
uv run uvicorn main:app --reload     # corre con autoreload, como nest start --watch
# Abrí http://localhost:8000/docs → Swagger UI generado solo
```

> Nota sobre datos volátiles: FastAPI, Pydantic (v2) y Uvicorn avanzan; verificá sintaxis al implementar. Pydantic v2 cambió bastante respecto de v1 —asegurate de estar en v2—.

**Índice de módulos**
1. Qué es FastAPI y el modelo mental (vs NestJS)
2. Tu primer endpoint: path operations
3. Parámetros de path y query (validación desde type hints)
4. Request body con Pydantic (vs DTOs de Nest)
5. Response models, serialización y status codes
6. Inyección de dependencias: `Depends` (vs el DI de Nest)
7. Manejo de errores: `HTTPException` y handlers (vs filters)
8. Organizar la app: routers, middleware, CORS
9. Async, bloqueo y acceso a datos
10. Servir IA: streaming, background tasks y el patrón "API + LLM"
11. Testing y criterio: producción y comparación final con NestJS

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es FastAPI y el modelo mental (vs NestJS)

**Teoría.** FastAPI es un framework web de Python, async-first, cuya idea distintiva es: **la API se define con type hints, y de ahí sale todo gratis** —validación, serialización, documentación—. Donde en NestJS declarás un DTO con decoradores de `class-validator` y configurás Swagger, en FastAPI el **type hint Pydantic es la única fuente de verdad**: valida el input, serializa el output y genera el OpenAPI, sin repetir.

El mapeo mental con NestJS, que es tu mejor atajo:

| Concepto | NestJS | FastAPI |
|---|---|---|
| Definir ruta | `@Get()` en un controller | `@app.get("/")` en una función |
| Runtime | Node (event loop) | ASGI (async, sobre Uvicorn) |
| Validación de input | DTO + `class-validator` + `ValidationPipe` | type hint **Pydantic** (automático) |
| Serialización de output | `ClassSerializerInterceptor` | `response_model` Pydantic |
| Inyección de dependencias | DI por constructor (decoradores) | `Depends()` (explícito por parámetro) |
| Docs | `@nestjs/swagger` configurado | OpenAPI + Swagger UI **automáticos** en `/docs` |
| Manejo de errores | Exception filters | `HTTPException` + exception handlers |
| Estructura | Modules + providers (opinado) | libre (vos elegís la estructura) |

Las dos diferencias de fondo:

1. **FastAPI es menos opinado.** NestJS te impone una arquitectura (módulos, providers, el contenedor de DI). FastAPI te da las piezas y vos armás la estructura. Para un servicio chico (servir un modelo) eso es liberador; para una app grande, **vos** tenés que poner la disciplina que Nest te imponía (organización en routers, capas de servicio). Tu criterio de arquitectura del track de Nest es lo que aplicás acá a mano.

2. **Todo gira alrededor de los type hints.** No es decoración: FastAPI **inspecciona la firma de tu función** en runtime para saber qué validar, qué inyectar y qué documentar. Un parámetro tipado `q: str` se vuelve un query param string requerido; un parámetro tipado con un modelo Pydantic se vuelve el request body validado. Esto hace el código compacto y es por qué [el módulo de Python](python.md) insistió tanto con Pydantic.

La frase mental: **en NestJS declarás la intención con decoradores y DTOs; en FastAPI la declarás con la firma tipada de la función, y el framework deduce el resto.**

**Ejercicios 1**
1.1 ¿Cuál es la idea distintiva de FastAPI sobre cómo se define una API? ¿Qué obtenés "gratis" de un type hint Pydantic?
1.2 Mapeá a FastAPI: el `ValidationPipe`+DTO de Nest, el `ClassSerializerInterceptor`, la config de `@nestjs/swagger`.
1.3 ¿Qué significa que FastAPI sea "menos opinado" que Nest y qué responsabilidad te transfiere en una app grande?
1.4 ¿Por qué se dice que FastAPI "inspecciona la firma de tu función"? Dá un ejemplo de qué deduce de un parámetro tipado.

---

## Módulo 2 — Tu primer endpoint: path operations

**Teoría.** La unidad de FastAPI es la **path operation**: una función decorada con el método HTTP y la ruta. FastAPI las llama así (no "controllers"), y son funciones sueltas, no métodos de una clase (aunque podés organizar con routers, módulo 8).

```python
from fastapi import FastAPI

app = FastAPI(title="Task API", version="1.0.0")   # instancia de la app

@app.get("/")                          # decorador: método + ruta
async def root() -> dict[str, str]:    # async def es lo idiomático
    return {"status": "ok"}            # un dict se serializa a JSON solo

@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "healthy"}
```

Puntos clave:

- **`app = FastAPI(...)`** es tu aplicación; los metadatos (`title`, `version`, `description`) alimentan el OpenAPI.
- **El decorador define método y ruta**: `@app.get`, `@app.post`, `@app.put`, `@app.patch`, `@app.delete`. Es exactamente la idea de `@Get()`/`@Post()` de Nest.
- **`async def`** es lo idiomático (FastAPI es async-first), aunque `def` normal también funciona (módulo 9 explica la diferencia, que es importante).
- **Lo que devolvés se serializa a JSON automáticamente**: un `dict`, una lista, un modelo Pydantic. No hay que llamar a un `res.json()`.
- **El servidor es Uvicorn** (un servidor ASGI), que arrancás con `uvicorn main:app` —`main` es el archivo, `app` la variable—. Es el equivalente de `nest start`.

Lo que te regala desde el minuto cero, sin configurar nada: andá a **`/docs`** y tenés la **Swagger UI** completa, interactiva, generada del código. En `/redoc` tenés otra vista. En `/openapi.json`, el esquema OpenAPI crudo. En Nest esto requería instalar y configurar `@nestjs/swagger`; en FastAPI **es gratis y siempre está al día** porque sale de los mismos type hints que ejecutan la API —nunca se desincroniza de la implementación—.

**Ejercicios 2**
2.1 ¿Qué es una "path operation" y en qué se parece a un handler de controller de Nest? ¿Es un método de clase?
2.2 Escribí una path operation `GET /ping` que devuelva `{"pong": true}`.
2.3 ¿Qué pasa con un `dict` que devolvés desde una path operation? ¿Hay que serializarlo a mano?
2.4 ¿Qué obtenés en `/docs` sin configurar nada y por qué nunca se desincroniza de la implementación (a diferencia de un Swagger mantenido a mano)?

---

## Módulo 3 — Parámetros de path y query (validación desde type hints)

**Teoría.** Acá se ve la magia de "la firma es el contrato". FastAPI deduce de dónde sale cada parámetro y lo valida, según cómo lo declarás.

**Path params** —parte de la URL—: los declarás en la ruta con `{}` y como parámetro tipado de la función. El tipo **valida y convierte**:

```python
@app.get("/tareas/{tarea_id}")
async def obtener_tarea(tarea_id: int) -> dict:    # int: valida que sea entero
    return {"id": tarea_id}
# GET /tareas/99  → tarea_id = 99 (int)
# GET /tareas/abc → 422 automático: "no es un int válido"
```

Fijate: pediste `int`, y FastAPI **rechaza `/tareas/abc` con un 422** y un mensaje claro, sin que escribas validación. En Nest necesitabas un `ParseIntPipe`; acá el type hint lo hace.

**Query params** —los que NO están en la ruta—: cualquier parámetro de la función que no sea path param se vuelve query param. Con default = opcional; sin default = requerido:

```python
@app.get("/tareas")
async def listar(
    limite: int = 10,                  # query opcional con default: ?limite=20
    completadas: bool | None = None,   # query opcional: ?completadas=true
    q: str = "",                       # ?q=texto
) -> dict:
    return {"limite": limite, "completadas": completadas, "q": q}
# GET /tareas?limite=5&completadas=true
```

Para validación más fina (rangos, longitudes), usás `Query`/`Path` con restricciones —el equivalente a los decoradores de validación de tus DTOs—:

```python
from fastapi import Query, Path
from typing import Annotated

@app.get("/tareas")
async def listar(
    limite: Annotated[int, Query(ge=1, le=100)] = 10,        # 1..100
    q: Annotated[str, Query(min_length=3, max_length=50)] = "",
) -> dict:
    ...
```

`Annotated[Tipo, Query(...)]` es la sintaxis moderna (Pydantic v2 / FastAPI actual): el tipo más sus restricciones. `ge`/`le` (>=, <=), `min_length`/`max_length`, `pattern` (regex). Si no se cumple → 422 automático con detalle. Es tu `class-validator` (`@Min`, `@MaxLength`) pero declarado inline en la firma.

La idea: **el tipo declara qué es y de dónde viene el parámetro; `Query`/`Path` agregan las reglas. La validación y el 422 son automáticos.**

**Ejercicios 3**
3.1 ¿Cómo distingue FastAPI un path param de un query param? ¿Cómo marcás un query param como requerido vs opcional?
3.2 Declarás `tarea_id: int` y llega `/tareas/abc`. ¿Qué pasa, y qué de Nest te ahorraste?
3.3 Escribí una path operation `GET /buscar` con un query `q` string requerido de mínimo 2 caracteres y un `pagina: int` opcional (default 1, mínimo 1).
3.4 ¿Qué es `Annotated[int, Query(ge=1, le=100)]` y con qué de tus DTOs de Nest lo equiparás?

---

## Módulo 4 — Request body con Pydantic (vs DTOs de Nest)

**Teoría.** Para POST/PUT/PATCH, el body se declara como un **parámetro tipado con un modelo Pydantic**. Ese modelo es el equivalente exacto de tu DTO de Nest, pero la validación viene incluida (no necesitás `ValidationPipe`):

```python
from pydantic import BaseModel, Field

class CrearTarea(BaseModel):                       # el "DTO"
    titulo: str = Field(min_length=1, max_length=200)
    prioridad: int = Field(default=1, ge=1, le=5)
    tags: list[str] = []

@app.post("/tareas", status_code=201)
async def crear(tarea: CrearTarea) -> dict:        # body validado automáticamente
    # 'tarea' ya es una instancia válida de CrearTarea
    return {"creada": tarea.titulo, "prioridad": tarea.prioridad}
```

Qué pasa con `POST /tareas` y un body JSON:

- FastAPI ve que `tarea` es un modelo Pydantic → sabe que es el **request body**.
- **Parsea el JSON, valida contra el modelo y construye la instancia.** Si falta `titulo`, si `prioridad` es 9, o si el JSON está mal → **422 con la lista exacta de errores**, sin código tuyo.
- Dentro de la función, `tarea` es un objeto tipado: `tarea.titulo`, autocompletado, chequeado por mypy.

Esto es **lo mismo** que tu DTO + `class-validator` + `ValidationPipe` de Nest, pero en una sola declaración y sin wiring. El modelo Pydantic *es* la validación.

Detalles útiles:

- **Modelos anidados**: un campo puede ser otro modelo Pydantic (`autor: Usuario`), y valida en profundidad —como DTOs anidados con `@ValidateNested`, pero automático—.
- **Validadores custom**: para reglas que no son restricciones simples, Pydantic tiene `@field_validator` y `@model_validator` (lógica de validación propia), el equivalente a un validador custom de `class-validator`.
- **Body + path + query juntos**: podés mezclar en una firma: `async def actualizar(tarea_id: int, datos: ActualizarTarea, notificar: bool = False)`. FastAPI reparte cada uno a su origen (path / body / query) por su tipo. Es el detalle que más impresiona: una sola firma describe toda la entrada del endpoint.

**Ejercicios 4**
4.1 ¿Cómo se declara el request body en FastAPI y qué hace el framework con el JSON entrante? ¿Qué de Nest reemplaza?
4.2 Si el body no cumple el modelo (falta un campo, un valor fuera de rango), ¿qué responde y quién escribe esa validación?
4.3 En una firma `async def actualizar(tarea_id: int, datos: ActualizarTarea, notificar: bool = False)`, ¿de dónde saca FastAPI cada uno de los tres parámetros?
4.4 ¿Cómo agregás una regla de validación que no sea una restricción simple (ej. "el título no puede ser solo espacios")?

---

## Módulo 5 — Response models, serialización y status codes

**Teoría.** Así como el input se modela con Pydantic, el **output** también, vía `response_model`. Esto cumple el rol de tu `ClassSerializerInterceptor` de Nest: controla **qué** sale y **cómo**.

```python
class TareaPublica(BaseModel):
    id: int
    titulo: str
    prioridad: int
    # NO incluye campos internos como owner_id o flags privados

@app.get("/tareas/{id}", response_model=TareaPublica)
async def obtener(id: int) -> Tarea:           # devuelvo una Tarea interna (más campos)
    tarea = repo.find(id)
    return tarea     # FastAPI la filtra/serializa SEGÚN TareaPublica
```

Lo que hace `response_model`:

- **Filtra el output**: aunque devuelvas un objeto con 15 campos, solo salen los del `response_model`. Es la forma de **no filtrar datos sensibles** (`password_hash`, `owner_id`) —el mismo objetivo que `@Exclude()` en Nest—.
- **Valida y serializa la respuesta**: garantiza que lo que sale cumple el contrato (y documenta ese shape en OpenAPI).
- **Documenta el response** en `/docs` con su esquema exacto.

Patrón idiomático y muy importante para seguridad: **modelos distintos para input, almacenamiento y output.** `CrearTarea` (lo que entra), `Tarea` (lo interno, con todos los campos), `TareaPublica` (lo que sale). Nunca exponés el modelo interno directo. Es la disciplina de DTOs separados de tu [módulo de Nest](nestjs-patrones.md), acá explícita.

**Status codes**:

```python
@app.post("/tareas", status_code=201, response_model=TareaPublica)   # 201 Created
async def crear(tarea: CrearTarea) -> Tarea: ...

@app.delete("/tareas/{id}", status_code=204)   # 204 No Content
async def borrar(id: int) -> None: ...
```

`status_code` en el decorador fija el código de éxito. Para errores (404, 409, etc.) usás `HTTPException` (módulo 7). Como en Nest, **respetá la semántica HTTP**: 201 al crear, 204 al borrar sin body, 200 por defecto.

Extra útil para listas: `response_model=list[TareaPublica]` para endpoints que devuelven colecciones —tipa y documenta el array entero—.

**Ejercicios 5**
5.1 ¿Qué hace `response_model` y qué rol de Nest cumple? Nombrá tres cosas que aporta.
5.2 ¿Por qué conviene tener modelos separados para crear, almacenar y exponer? ¿Qué riesgo de seguridad evita?
5.3 ¿Cómo fijás un 201 en un POST y un 204 en un DELETE?
5.4 Devolvés un objeto `Tarea` interno con `password_hash` pero el `response_model` es `TareaPublica` (sin ese campo). ¿Qué sale en la respuesta?

---

## Módulo 6 — Inyección de dependencias: `Depends` (vs el DI de Nest)

**Teoría.** FastAPI tiene un sistema de **inyección de dependencias** propio, conceptualmente igual al de Nest pero **explícito por parámetro** en vez de por constructor. Una dependencia es una función (o callable) cuyo resultado se inyecta donde la pedís con `Depends()`.

```python
from fastapi import Depends
from typing import Annotated

# Una "dependencia": provee algo que varias rutas necesitan
async def get_db() -> AsyncSession:
    async with SessionLocal() as session:
        yield session                  # 'yield' = se entrega y se limpia después (como un context manager)

# Inyectás declarándola como parámetro con Depends
@app.get("/tareas")
async def listar(db: Annotated[AsyncSession, Depends(get_db)]) -> list[TareaPublica]:
    return await db.tareas.all()
```

Cómo se mapea con Nest:

- En Nest, `constructor(private repo: TareaRepository)` y el contenedor inyecta. En FastAPI, `db: Annotated[..., Depends(get_db)]` en la firma de la ruta, y FastAPI ejecuta `get_db()` e inyecta el resultado.
- **Las dependencias se componen**: una dependencia puede depender de otras (`Depends` dentro de un `Depends`). FastAPI resuelve el árbol —como el grafo de providers de Nest—.
- **`yield` para setup/teardown**: una dependencia con `yield` entrega el valor y ejecuta lo de después del `yield` al terminar el request (cerrar la sesión de DB, liberar recursos). Es el patrón de recursos con `with` del [módulo de Python](python.md), integrado al ciclo del request.

El caso de uso estrella —y el puente con [Autenticación](autenticacion.md)—: **autenticación y autorización como dependencias**.

```python
async def usuario_actual(token: Annotated[str, Depends(oauth2_scheme)]) -> Usuario:
    usuario = decodificar_jwt(token)          # valida el JWT
    if usuario is None:
        raise HTTPException(401, "No autenticado")
    return usuario

@app.get("/perfil")
async def perfil(yo: Annotated[Usuario, Depends(usuario_actual)]) -> UsuarioPublico:
    return yo     # si el token es inválido, ni se entra acá: la dependencia ya cortó
```

Cualquier ruta que pida `Depends(usuario_actual)` queda protegida: si el token falla, la dependencia lanza el 401 y la ruta ni se ejecuta. Es el equivalente de un `AuthGuard` de Nest, pero como una función inyectable y componible. Podés aplicar dependencias a una ruta, a un router entero, o a toda la app.

La idea: **`Depends` es el DI de Nest hecho explícito en la firma —ves exactamente qué necesita cada ruta— y sirve para todo: DB, auth, config, lógica compartida—.**

**Ejercicios 6**
6.1 ¿Qué es una dependencia en FastAPI y cómo se inyecta? ¿En qué se diferencia del DI por constructor de Nest?
6.2 ¿Qué hace una dependencia con `yield` y con qué patrón del módulo de Python se relaciona?
6.3 ¿Cómo implementarías autenticación con `Depends` y qué concepto de Nest cumple ese rol?
6.4 Si la dependencia `usuario_actual` lanza un 401, ¿llega a ejecutarse el cuerpo de la ruta protegida? ¿Por qué?

---

## Módulo 7 — Manejo de errores: `HTTPException` y handlers (vs filters)

**Teoría.** Para devolver un error HTTP, lanzás una **`HTTPException`** con el status y el detalle. FastAPI la convierte en la respuesta JSON correspondiente:

```python
from fastapi import HTTPException

@app.get("/tareas/{id}", response_model=TareaPublica)
async def obtener(id: int) -> Tarea:
    tarea = repo.find(id)
    if tarea is None:
        raise HTTPException(status_code=404, detail="Tarea no encontrada")
    return tarea
# → 404 {"detail": "Tarea no encontrada"}
```

Es el equivalente de lanzar una `NotFoundException` en Nest. Los casos típicos: 404 (no existe), 403 (sin permiso), 409 (conflicto), 400 (input inválido que no captura la validación automática).

Para errores **transversales** —manejar de forma uniforme una excepción de tu dominio en toda la app— usás un **exception handler**, el equivalente de los **exception filters** de Nest:

```python
# Excepción de dominio propia (del módulo de Python: hereda de Exception)
class TareaNoEncontrada(Exception):
    def __init__(self, id: int): self.id = id

# Handler global: traduce la excepción de dominio a una respuesta HTTP
@app.exception_handler(TareaNoEncontrada)
async def manejar_no_encontrada(request: Request, exc: TareaNoEncontrada):
    return JSONResponse(status_code=404, content={"detail": f"Tarea {exc.id} no existe"})

# Ahora el dominio lanza su excepción pura, sin saber de HTTP:
@app.get("/tareas/{id}")
async def obtener(id: int) -> Tarea:
    return repo.get_or_raise(id)    # lanza TareaNoEncontrada → el handler la convierte
```

Esto te da la **separación de capas** que cuidabas en Nest: tu lógica de dominio lanza excepciones de negocio puras (`TareaNoEncontrada`), que no saben nada de HTTP, y el handler en el borde las traduce a respuestas. El dominio no se acopla al protocolo.

Sobre el **contrato de errores** (lo que viste en el [módulo de tiempo real/Swagger](tiempo-real.md)): definí un shape de error consistente para toda la API (un modelo Pydantic `ErrorResponse`), documentalo, y devolvelo siempre igual. Un cliente que consume tu API —o un agente de IA— necesita errores predecibles.

**Ejercicios 7**
7.1 ¿Cómo devolvés un 404 desde una ruta? ¿Con qué de Nest se corresponde `HTTPException`?
7.2 ¿Qué es un exception handler y qué filter de Nest cumple ese rol?
7.3 ¿Qué ventaja de arquitectura te da que el dominio lance `TareaNoEncontrada` (excepción pura) y un handler la traduzca a 404?
7.4 ¿Por qué conviene un shape de error consistente en toda la API, sobre todo si la consume otro servicio o un agente?

---

## Módulo 8 — Organizar la app: routers, middleware, CORS

**Teoría.** Como FastAPI no impone estructura (módulo 1), **vos** organizás. La herramienta es el **`APIRouter`**: agrupa rutas relacionadas en un archivo, como un controller/módulo de Nest, y se monta en la app.

```python
# routers/tareas.py
from fastapi import APIRouter
router = APIRouter(prefix="/tareas", tags=["tareas"])   # prefijo común + tag para /docs

@router.get("")
async def listar(): ...

@router.post("", status_code=201)
async def crear(tarea: CrearTarea): ...

# main.py
from routers import tareas, usuarios
app.include_router(tareas.router)
app.include_router(usuarios.router)
```

El `prefix` evita repetir la ruta base; el `tags` agrupa los endpoints en la Swagger UI. Una estructura sana para una app que crece: `routers/` (endpoints), `models/` (Pydantic), `services/` (lógica de negocio), `db/` (acceso a datos) —la separación en capas de tu [track de Nest](nestjs-senior.md), que acá aplicás por convención propia, no porque el framework te obligue—.

**Middleware** —código que corre antes/después de cada request, como los middleware de Nest—:

```python
@app.middleware("http")
async def agregar_request_id(request: Request, call_next):
    request_id = request.headers.get("x-request-id", str(uuid4()))
    response = await call_next(request)         # sigue la cadena
    response.headers["x-request-id"] = request_id
    return response
```

Usos típicos: logging, request-id de correlación (puente con [observabilidad](observabilidad.md)), timing. Para tracing serio, OpenTelemetry tiene instrumentación automática de FastAPI —los mismos conceptos del módulo de observabilidad, ecosistema Python—.

**CORS** —imprescindible si un front en otro origen consume tu API—:

```python
from fastapi.middleware.cors import CORSMiddleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://miapp.com"],     # allowlist, NO "*" en prod (lección de tiempo-real.md)
    allow_methods=["*"],
    allow_headers=["*"],
)
```

La regla que ya conocés: **allowlist explícita, nunca `"*"` en producción** —el mismo cuidado de CORS del módulo de tiempo real—.

**Ejercicios 8**
8.1 ¿Qué es un `APIRouter` y a qué de Nest se parece? ¿Qué aportan `prefix` y `tags`?
8.2 Como FastAPI no impone estructura, ¿qué tenés que aportar vos en una app grande, y de dónde sacás ese criterio?
8.3 Escribí (en prosa o código) un middleware que agregue un header `X-Process-Time` con cuánto tardó el request.
8.4 ¿Cuál es la regla de CORS en producción y de qué módulo del temario viene esa lección?

---

## Módulo 9 — Async, bloqueo y acceso a datos

**Teoría.** Este módulo es donde más se nota que entendiste el [async de Python](python.md) y el [event loop](nodejs.md). FastAPI corre sobre un **event loop** (ASGI): vale exactamente la misma regla que en Node —**nunca bloquees el loop**—.

La decisión clave: **`async def` vs `def`** en una path operation.

- **`async def`**: la función corre **en el event loop**. Usala cuando hacés I/O **async** (`await` a una DB async, a `httpx`, a un LLM). Es lo idiomático y lo más eficiente.
- **`def`** (síncrona): FastAPI la corre en un **thread pool** aparte, para no bloquear el loop. Usala cuando tu código es **bloqueante** y no tenés versión async (una librería síncrona, un cálculo pesado).

El gotcha que hunde APIs: **poner código bloqueante dentro de un `async def`.** Si dentro de una path operation `async def` llamás a algo síncrono y lento (un `requests.get`, un `time.sleep`, una query con un driver sync), **bloqueás el event loop entero** y toda la API se frena —igual que un cómputo pesado en Node—. Dos salidas:

1. Usar la versión **async** de la librería (`httpx.AsyncClient` en vez de `requests`, `asyncpg`/SQLAlchemy async en vez del driver sync).
2. Si no hay versión async, o hacés la ruta `def` (y FastAPI la manda al thread pool), o usás `run_in_threadpool` para el pedazo bloqueante.

```python
# ❌ Bloquea el loop: requests es síncrono dentro de un async def
@app.get("/malo")
async def malo():
    r = requests.get("https://lenta")     # congela TODA la API mientras espera
    return r.json()

# ✅ I/O async correcto
@app.get("/bueno")
async def bueno():
    async with httpx.AsyncClient() as c:
        r = await c.get("https://lenta")   # libera el loop mientras espera
        return r.json()
```

**Acceso a datos.** Para Postgres ([tu DB del track](postgresql.md)), el stack típico es **SQLAlchemy 2.0 async** + `asyncpg`, inyectando la sesión vía `Depends` (módulo 6). Todo tu conocimiento de SQL, transacciones, índices y pooling se transfiere; cambia el cliente. Para que la sesión sea async y no bloquee, usás el engine async de SQLAlchemy y `await` en las queries.

La idea: **FastAPI es async-first; la regla de oro es la misma de Node —I/O async, nunca bloquees el loop—. Si una librería es sync y bloqueante, mandala al thread pool (`def` o `run_in_threadpool`).**

**Ejercicios 9**
9.1 ¿Cuándo usás `async def` y cuándo `def` en una path operation? ¿Qué hace FastAPI con cada una?
9.2 ¿Qué pasa si ponés un `requests.get` (síncrono) dentro de un `async def`, y con qué problema de Node es idéntico?
9.3 Dá las dos salidas para usar una operación bloqueante sin frenar la API.
9.4 ¿Qué stack usarías para hablar con Postgres de forma async desde FastAPI, y qué de tu conocimiento previo se transfiere?

---

## Módulo 10 — Servir IA: streaming, background tasks y el patrón "API + LLM"

**Teoría.** El módulo que justifica todo el track para un AI Engineer: **FastAPI es la forma estándar de exponer un modelo, un RAG o un agente por HTTP.** El patrón base es una API delgada que recibe el input, llama al LLM/agente, y devuelve el resultado.

```python
from anthropic import AsyncAnthropic
client = AsyncAnthropic()

class Pregunta(BaseModel):
    texto: str = Field(min_length=1, max_length=2000)

@app.post("/preguntar")
async def preguntar(p: Pregunta) -> dict[str, str]:
    msg = await client.messages.create(
        model="claude-opus-4-8",
        max_tokens=1024,
        messages=[{"role": "user", "content": p.texto}],
    )
    return {"respuesta": msg.content[0].text}
```

(Usás el cliente **async** y `await` —módulo 9— porque la llamada al LLM es I/O lento: bloquear el loop acá tumbaría la API bajo carga.)

**El problema específico de IA: las respuestas son lentas y largas.** Un LLM puede tardar segundos en completar. Si el usuario espera la respuesta entera, la UX es mala. La solución es el **streaming**: ir mandando tokens a medida que el modelo los genera. FastAPI lo hace con `StreamingResponse` —tu puente con el SSE que viste en [tiempo real](tiempo-real.md)—:

```python
from fastapi.responses import StreamingResponse

@app.post("/preguntar-stream")
async def preguntar_stream(p: Pregunta):
    async def generar():
        async with client.messages.stream(
            model="claude-opus-4-8", max_tokens=1024,
            messages=[{"role": "user", "content": p.texto}],
        ) as stream:
            async for texto in stream.text_stream:
                yield texto              # cada chunk sale al cliente al instante
    return StreamingResponse(generar(), media_type="text/event-stream")
```

El `async def generar()` con `yield` es un **generador async**: produce chunks que `StreamingResponse` envía a medida que llegan, sin esperar la respuesta completa. Es exactamente el modelo de SSE/streaming del módulo de tiempo real, aplicado a tokens de un LLM. Casi toda app de chat con IA en producción usa este patrón.

**Background tasks** —para trabajo que no debe bloquear la respuesta—: si tras responder hay que hacer algo lento (indexar en un vector DB, mandar un webhook, loguear a un sistema externo), lo despachás como tarea de fondo:

```python
from fastapi import BackgroundTasks

@app.post("/documentos", status_code=202)
async def subir(doc: Documento, tareas: BackgroundTasks):
    tareas.add_task(indexar_en_vector_db, doc)   # corre DESPUÉS de responder
    return {"status": "procesando"}              # 202 Accepted, no esperás la indexación
```

Para trabajo pesado o que requiere garantías (reintentos, persistencia), esto escala a una **cola** real (Celery, o el patrón de [colas/BullMQ](redis.md) que ya conocés, acá con Celery/RQ en Python) —background tasks de FastAPI es para lo liviano—.

La idea: **una API de IA es una API normal con tres patrones extra: cliente async para el LLM, streaming para respuestas largas, y background tasks/colas para el trabajo diferido (indexar, webhooks).** Estos son la base de los módulos de [vector DBs](vector-dbs.md) y [agentes](ai-agents-python.md) que vienen.

**Ejercicios 10**
10.1 ¿Por qué la llamada al LLM en la API usa el cliente **async** y `await`? ¿Qué pasaría con el cliente sync bajo carga?
10.2 ¿Qué problema de UX resuelve el streaming en una API de IA y con qué tecnología del módulo de tiempo real se conecta?
10.3 ¿Qué es un generador async (`async def` + `yield`) y cómo lo usa `StreamingResponse`?
10.4 ¿Para qué sirven las background tasks y cuándo deberías pasar a una cola real en su lugar?

---

## Módulo 11 — Testing y criterio: producción y comparación final con NestJS

**Teoría.** Cierre del módulo: testear FastAPI, llevarlo a producción, y el criterio FastAPI vs NestJS.

**Testing** —tu base de [pytest](python.md) y [testing/TDD](tdd.md) aplicada—: FastAPI trae un `TestClient` (sobre `httpx`) que es el equivalente exacto del **Supertest** que usaste con Nest:

```python
from fastapi.testclient import TestClient
client = TestClient(app)

def test_crear_tarea():
    r = client.post("/tareas", json={"titulo": "Estudiar", "prioridad": 3})
    assert r.status_code == 201
    assert r.json()["prioridad"] == 3

def test_titulo_vacio_da_422():
    r = client.post("/tareas", json={"titulo": ""})
    assert r.status_code == 422        # la validación Pydantic automática
```

Toda tu disciplina se transfiere: tests de endpoint (e2e), `Depends` te deja **sobrescribir dependencias** en los tests (`app.dependency_overrides`) para inyectar una DB de prueba o un LLM mockeado —el equivalente de los `TestingModule` overrides de Nest—. La pirámide, el criterio classicist/mockist, todo igual.

**Producción**:
- Corrés con **Uvicorn** detrás de un proceso manager. El patrón estándar es **Gunicorn gestionando workers Uvicorn** (varios procesos para usar todos los cores —acordate del GIL del [módulo de Python](python.md): un proceso no usa más de un core para CPU—), o Uvicorn con `--workers`. Es el análogo del `cluster` de Node.
- **Contenedor** ([Docker](docker-deploy.md)): imagen Python slim, `uv sync`, exponer el puerto, healthcheck. El mismo flujo del módulo de Docker.
- Lo demás del track aplica igual: variables de entorno, healthcheck (`/health`), observabilidad (OTel), CORS, versionado de la API.

**FastAPI vs NestJS — el criterio**:

- **FastAPI gana** cuando: el backend es de IA/ML (estás en Python por las librerías de todos modos), querés algo liviano y rápido de levantar, el equipo es Python, o servís modelos/agentes. Para un **servicio de IA**, es la elección natural.
- **NestJS gana** cuando: app de producto grande y estructurada donde la arquitectura impuesta (módulos, DI, opinión fuerte) ayuda a un equipo grande, full-stack TS, o ya estás en el ecosistema Node.
- **Conviven**: el patrón del [módulo de Python](python.md) —API de producto en Nest/TS que delega la parte de IA a un servicio FastAPI— es muy común. No es competencia; es cada uno en su dominio.

El parecido de fondo no es casual: ambos abrazaron decoradores, DI, validación por esquema y OpenAPI. Si dominás Nest, **FastAPI es Nest más liviano y Python-native**; la curva es corta y el criterio de arquitectura (capas, DTOs separados, errores de dominio, no bloquear el loop, tests de comportamiento) es **el mismo** —solo que en FastAPI lo aportás vos en vez de que el framework te lo imponga—.

El cierre y el puente: con FastAPI tenés tu primer servicio Python de producción y, sobre todo, **la forma de exponer IA por HTTP** (async + streaming + background tasks). Esto es el sustrato de lo que viene: el [módulo de vector DBs](vector-dbs.md) va a servir búsquedas semánticas por una API FastAPI, y el de [agentes Python](ai-agents-python.md) va a exponer un grafo de LangGraph como endpoint. La base —Python + FastAPI— ya está; ahora viene el contenido de IA propiamente dicho.

**Ejercicios 11**
11.1 ¿Cuál es el equivalente de Supertest en FastAPI y cómo testeás un endpoint? ¿Cómo inyectás una DB de prueba o un LLM mockeado?
11.2 ¿Cómo se corre FastAPI en producción para aprovechar varios cores, y con qué del GIL y de Node se conecta?
11.3 Dá dos escenarios donde elegirías FastAPI sobre NestJS y dos al revés.
11.4 ¿Por qué, si dominás NestJS, la curva de FastAPI es corta? ¿Qué del criterio de arquitectura es idéntico y qué cambia?

---

## Soluciones

### Módulo 1
1.1 La idea: la API se define con **type hints**, y de ahí FastAPI deduce todo. De un type hint Pydantic obtenés gratis: validación del input, serialización del output y documentación OpenAPI —una sola fuente de verdad—.
1.2 `ValidationPipe`+DTO → **modelo Pydantic** como parámetro (validación automática); `ClassSerializerInterceptor` → **`response_model`**; config de `@nestjs/swagger` → **OpenAPI/Swagger automáticos** en `/docs`.
1.3 Que no impone arquitectura (módulos, providers, contenedor de DI). En una app grande te transfiere la responsabilidad de **poner vos la estructura** (routers, capas de servicio, separación) —el criterio que en Nest venía impuesto—.
1.4 Porque FastAPI **inspecciona la firma en runtime** para decidir qué validar, inyectar y documentar. Ejemplo: de `tarea_id: int` (en la ruta) deduce un path param entero validado; de un parámetro tipado con un modelo Pydantic deduce el request body.

### Módulo 2
2.1 Una path operation es una **función decorada con método+ruta** (`@app.get("/")`); cumple el rol de un handler de controller de Nest. **No** es método de clase: es una función suelta (aunque podés agrupar con routers).
2.2 `@app.get("/ping")` sobre `async def ping(): return {"pong": True}`.
2.3 Se **serializa a JSON automáticamente**; no hay que serializar a mano (ni llamar a un `res.json()`).
2.4 La **Swagger UI** completa e interactiva (más ReDoc y el `openapi.json`). Nunca se desincroniza porque sale de los **mismos type hints que ejecutan la API** —no es un doc mantenido aparte que pueda quedar viejo—.

### Módulo 3
3.1 El path param está en la ruta con `{}` y como parámetro tipado; cualquier parámetro de la función que **no** esté en la ruta es query param. Requerido = sin default; opcional = con default.
3.2 Responde **422** automático ("no es un int válido") sin código tuyo; te ahorraste el `ParseIntPipe` de Nest.
3.3 `async def buscar(q: Annotated[str, Query(min_length=2)], pagina: Annotated[int, Query(ge=1)] = 1): ...` (`q` requerido sin default, `pagina` opcional con default 1).
3.4 Es un type hint (`int`) con restricciones de validación (`Query(ge=1, le=100)`); equivale a los decoradores de validación de tus DTOs (`@Min(1)`, `@Max(100)`), declarados inline en la firma.

### Módulo 4
4.1 Como un parámetro tipado con un **modelo Pydantic**; FastAPI parsea el JSON, lo **valida** contra el modelo y construye la instancia. Reemplaza el DTO + `class-validator` + `ValidationPipe` de Nest.
4.2 Responde **422 con la lista exacta de errores**; esa validación la escribe **FastAPI/Pydantic** automáticamente (vos solo declaraste el modelo).
4.3 `tarea_id` del **path** (está en la ruta y es tipo simple), `datos` del **body** (es un modelo Pydantic), `notificar` del **query** (tipo simple con default que no está en la ruta).
4.4 Con un **validador custom** de Pydantic: `@field_validator` (o `@model_validator`) en el modelo, con la lógica propia (ej. rechazar título que es solo espacios) —el equivalente a un validador custom de class-validator—.

### Módulo 5
5.1 `response_model` **filtra, valida y documenta** el output; cumple el rol del `ClassSerializerInterceptor`. Aporta: (1) no filtrar datos sensibles, (2) garantizar el shape de la respuesta, (3) documentarla en OpenAPI.
5.2 Para no exponer el modelo interno: `CrearTarea` (entra), `Tarea` (interno, completo), `TareaPublica` (sale). Evita el riesgo de **filtrar datos sensibles** (`password_hash`, `owner_id`) al serializar el objeto interno directo.
5.3 `@app.post(..., status_code=201)` y `@app.delete(..., status_code=204)`.
5.4 Sale **solo lo de `TareaPublica`**: el `password_hash` se filtra y no aparece en la respuesta (aunque el objeto interno lo tenga).

### Módulo 6
6.1 Una dependencia es una función (callable) cuyo resultado se inyecta donde la pedís con `Depends()`. Diferencia con Nest: es **explícita por parámetro** en la firma de la ruta (ves qué necesita cada ruta), no por constructor con un contenedor implícito.
6.2 Entrega el valor en el `yield` y ejecuta el **teardown** (lo de después del `yield`) al terminar el request —cerrar la sesión de DB, liberar recursos—. Se relaciona con el patrón de recursos con `with`/context manager del módulo de Python.
6.3 Una dependencia `usuario_actual` que valida el JWT y devuelve el usuario (o lanza 401); las rutas la piden con `Depends(usuario_actual)`. Cumple el rol del **`AuthGuard`** de Nest.
6.4 **No** se ejecuta el cuerpo: FastAPI resuelve las dependencias **antes** de entrar a la ruta, así que si la dependencia lanza el 401, corta ahí y la ruta nunca corre.

### Módulo 7
7.1 `raise HTTPException(status_code=404, detail="...")`. Corresponde a lanzar una excepción HTTP de Nest (`NotFoundException`, etc.).
7.2 Un handler registrado con `@app.exception_handler(MiExcepcion)` que traduce una excepción a una respuesta HTTP uniforme en toda la app; cumple el rol de los **exception filters** de Nest.
7.3 Da **separación de capas**: el dominio lanza excepciones de negocio puras (no sabe de HTTP) y el handler en el borde las traduce a respuestas. El dominio no se acopla al protocolo HTTP.
7.4 Porque un consumidor (otro servicio o un agente de IA) necesita **errores predecibles** para manejarlos programáticamente; un shape inconsistente obliga a parsear caso por caso y rompe la automatización.

### Módulo 8
8.1 Un `APIRouter` agrupa rutas relacionadas en un archivo (como un controller/módulo de Nest) y se monta con `include_router`. `prefix` evita repetir la ruta base; `tags` agrupa los endpoints en la Swagger UI.
8.2 Tenés que aportar **la estructura/arquitectura** (routers, capas services/models/db, separación de responsabilidades); el criterio sale de tu track de Nest (separación en capas, DTOs separados), aplicado por convención propia.
8.3 Un `@app.middleware("http")` que toma el tiempo antes de `call_next`, calcula la diferencia después, y setea `response.headers["X-Process-Time"]` con ese valor.
8.4 **Allowlist explícita de orígenes, nunca `"*"` en producción**; la lección viene del módulo de **tiempo real** (CORS/CSWSH).

### Módulo 9
9.1 `async def` cuando hacés I/O **async** (corre en el event loop, lo idiomático y eficiente); `def` cuando el código es **bloqueante** sin versión async (FastAPI la corre en un **thread pool** para no bloquear el loop).
9.2 **Bloqueás el event loop entero** mientras espera (toda la API se frena); es idéntico a poner un cómputo pesado/llamada bloqueante en el event loop de Node.
9.3 (1) Usar la versión **async** de la librería (`httpx.AsyncClient`, `asyncpg`/SQLAlchemy async); (2) si no hay, hacer la ruta `def` (va al thread pool) o mandar el pedazo bloqueante con `run_in_threadpool`.
9.4 **SQLAlchemy 2.0 async + asyncpg**, con la sesión inyectada vía `Depends`. Se transfiere todo tu conocimiento de SQL, transacciones, índices y pooling (del módulo de Postgres); cambia solo el cliente/driver.

### Módulo 10
10.1 Porque la llamada al LLM es **I/O lento**; con `await` se libera el loop mientras espera. Con el cliente **sync** bajo carga, cada llamada bloquearía el event loop y la API se caería bajo concurrencia.
10.2 Resuelve la espera larga: en vez de aguardar la respuesta completa (mala UX), manda **tokens a medida que se generan**. Se conecta con **SSE / streaming** del módulo de tiempo real.
10.3 Un generador async es un `async def` con `yield` que produce chunks de a uno; `StreamingResponse` los **envía al cliente a medida que se generan**, sin esperar la respuesta completa.
10.4 Para ejecutar trabajo **después de responder** sin bloquear la respuesta (indexar, webhooks, logging). Pasás a una **cola real** (Celery/RQ, o el patrón de colas que ya conocés) cuando el trabajo es pesado o necesita garantías (reintentos, persistencia, durabilidad).

### Módulo 11
11.1 El **`TestClient`** (sobre httpx) es el equivalente de Supertest; testeás haciendo requests (`client.post(...)`) y aserciones sobre status/JSON. Inyectás DB de prueba o LLM mockeado con `app.dependency_overrides` (sobrescribir las dependencias `Depends`) —como los overrides del `TestingModule` de Nest—.
11.2 Con **Gunicorn gestionando workers Uvicorn** (o Uvicorn `--workers`): varios procesos para usar todos los cores. Se conecta con el **GIL** (un proceso no usa más de un core para CPU) y con el `cluster` de Node (misma idea: multiproceso para escalar CPU).
11.3 **FastAPI**: (a) backend de IA/ML (ya estás en Python por las librerías); (b) servicio liviano para servir un modelo/agente. **NestJS**: (a) app de producto grande donde la arquitectura impuesta ayuda a un equipo grande; (b) full-stack en TS / ya estás en el ecosistema Node.
11.4 Porque comparten ADN: decoradores para rutas, DI, validación por esquema, OpenAPI. Idéntico: el criterio de arquitectura (capas, DTOs/modelos separados, errores de dominio, no bloquear el loop, tests de comportamiento). Cambia: en FastAPI **vos** aportás la estructura, en Nest viene impuesta.

---

Con este módulo tenés tu primer servicio Python de producción y —clave para el track— **la forma estándar de servir IA por HTTP**: cliente async para el LLM, streaming de tokens (`StreamingResponse`) y background tasks/colas para el trabajo diferido. Mapeaste FastAPI contra NestJS (decoradores, `Depends` como DI, Pydantic como validación, `HTTPException`/handlers como filters, `TestClient` como Supertest) y viste que tu criterio de arquitectura se transfiere entero. Lo que sigue construye encima: [Prompt Engineering](prompt-engineering.md) (cómo hablarle al modelo), [Vector Databases](vector-dbs.md) (Pinecone/FAISS/Weaviate, servidos por una API como esta), [AI Agents](ai-agents-python.md) (LangGraph/AutoGen expuestos como endpoints), [Voice AI](voice-ai.md) y [Deploy de aplicaciones de IA](deploy-ai.md). Ya tenés el caparazón —Python + FastAPI—; ahora viene lo que va adentro.
