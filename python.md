# Python para quien viene de TypeScript/Node

**La base del track de IA en Python · del modelo mental de JS/TS al de Python · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo abre el **track de IA en Python**, paralelo al track de IA en TS/Anthropic que ya tenés ([LLMs](ia-llms.md), [RAG](rag.md), [Agentes](agentes.md), [Evals](evals.md)). El objetivo no es enseñarte a programar —ya sabés, y bien— sino **mapear lo que ya sabés de TypeScript/Node al mundo Python** lo más rápido posible, porque el ecosistema de IA/ML (FastAPI, LangGraph, las librerías de vectores, Whisper) vive mayormente en Python. La estrategia es de **contraste**: en cada tema vas a ver "en TS hacías X, en Python se hace Y", para no aprender de cero sino traducir. El objetivo final del módulo es que en pocas horas puedas leer y escribir Python idiomático con confianza, con type hints y tests, listo para los módulos de FastAPI y agentes que vienen.

**Lo que asumimos.** TypeScript a nivel sólido (tipos, async/await, módulos), Node ([event loop](nodejs.md), npm), y testing ([Jest/Vitest](testing.md)). No asumimos **nada** de Python.

**Para practicar.** Python 3.12+ y el gestor moderno `uv` (rápido, reemplaza pip/venv/pyenv en uno):

```bash
# uv: el gestor de paquetes/proyectos de Python moderno (de los creadores de ruff)
curl -LsSf https://astral.sh/uv/install.sh | sh
uv init mi-proyecto && cd mi-proyecto
uv add fastapi pydantic          # como npm install
uv run python main.py            # corre en el venv del proyecto, sin activar nada
```

> Nota sobre datos volátiles: las versiones de Python (3.12/3.13/3.14) y el tooling (`uv`, `ruff`, `mypy`) avanzan rápido. Lo conceptual de este módulo es estable; verificá comandos y flags al usarlos.

**Índice de módulos**
1. Por qué Python y el cambio de modelo mental
2. Entorno y tooling: `uv`, venv, paquetes (vs npm/node)
3. Tipos y type hints: `typing` + `mypy` (el puente con TS strict)
4. Estructuras de datos: `list`/`dict`/`set`/`tuple` y comprehensions
5. Funciones: argumentos, `*args`/`**kwargs`, decoradores
6. Clases, `dataclass` y Pydantic (el puente a FastAPI)
7. Módulos, paquetes e imports (vs ESM/CJS)
8. Errores y context managers (`try/except`, `with`)
9. Async: `asyncio` y `async`/`await` (vs el event loop de Node)
10. Testing con `pytest` (vs Jest/Vitest)
11. El criterio: cuándo Python, cuándo Node, y qué es "pythonic"

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Por qué Python y el cambio de modelo mental

**Teoría.** Si tu stack es TS/Node y te funciona, ¿por qué aprender Python? Por una razón pragmática y específica: **el ecosistema de IA/ML vive en Python**. Los frameworks de agentes (LangGraph, AutoGen, y el propio **SDK oficial de Anthropic**, que también está en Python además de TS), las librerías de vectores (FAISS), el tooling de modelos (Whisper, transformers, vLLM), y casi todo paper o tutorial de ML asumen Python. Podés consumir LLMs desde TS perfectamente (lo hiciste en el track Anthropic), pero el día que querés FAISS local, un grafo de agentes con LangGraph, o servir un modelo propio, estás en Python. Para un rol de **AI Engineer / AI QA**, Python no es opcional: es la lingua franca.

La buena noticia: viniendo de TS, el 80% se traduce directo. Las diferencias de modelo mental que tenés que internalizar primero:

1. **Indentación significativa.** No hay llaves `{}`. Los bloques se definen por **sangría** (4 espacios por convención). El `:` abre el bloque y la indentación lo delimita. Esto no es cosmético: un espacio de más es un error de sintaxis.

   ```python
   def saludar(nombre: str) -> str:
       if nombre:                 # el ':' y la sangría reemplazan a { }
           return f"Hola {nombre}"
       return "Hola"
   ```

2. **Dinámico, pero con type hints opcionales.** Python es de tipado dinámico (como JS sin TS): los tipos no se chequean en runtime por defecto. Pero desde 3.5 tiene **type hints** (`nombre: str`) que un chequeador estático (`mypy`) verifica —exactamente el rol que cumple `tsc` sobre JS—. La diferencia clave con TS: en Python los hints son **anotaciones que no afectan la ejecución** (no hay "compilación que los borra"; directamente el intérprete los ignora salvo casos puntuales). Tu disciplina de [TS strict](typescript.md) se traslada como "Python + type hints + mypy estricto".

3. **Interpretado, sin paso de build.** No hay `tsc`. Corrés el `.py` directo con el intérprete. El "build" es, a lo sumo, el chequeo de `mypy` y el linter `ruff`, que son opcionales para correr (obligatorios para calidad).

4. **`snake_case`, no `camelCase`.** La convención de Python (PEP 8) es `snake_case` para variables y funciones, `PascalCase` para clases, `MAYUS` para constantes. Escribir `miVariable` en Python te delata como recién llegado.

5. **La filosofía "pythonic".** Python valora la legibilidad explícita ("there should be one obvious way to do it", el *Zen of Python*). Donde JS tiene tres formas de iterar, Python empuja una idiomática (módulo 4). Aprender Python no es solo sintaxis: es adoptar ese estilo.

La frase mental: **lo que sabés de programación se transfiere entero; lo que cambia es la sintaxis, las convenciones y el ecosistema. Tratá Python como "TS con otra piel y mejores librerías de IA", no como un lenguaje a aprender de cero.**

**Ejercicios 1**
1.1 ¿Por qué un AI Engineer necesita Python aunque sepa consumir LLMs desde TS? Dá dos ejemplos concretos de cuándo no te alcanza TS.
1.2 ¿Qué reemplaza en Python a las llaves `{}` de los bloques, y por qué no es un detalle cosmético?
1.3 ¿Qué rol cumple `mypy` y con qué herramienta del mundo TS lo compararías? ¿En qué se diferencia el efecto de los type hints respecto de los tipos de TS?
1.4 Traducí estos nombres a la convención de Python: `getUserById`, `MaxRetries`, `userProfile`.

---

## Módulo 2 — Entorno y tooling: `uv`, venv, paquetes (vs npm/node)

**Teoría.** El mapeo del tooling es lo primero que necesitás para no frustrarte. La tabla mental TS/Node → Python:

| Concepto | Node | Python (moderno) |
|---|---|---|
| Gestor de paquetes | `npm` / `pnpm` | `uv` (o `pip`) |
| Manifiesto | `package.json` | `pyproject.toml` |
| Lockfile | `package-lock.json` | `uv.lock` |
| Carpeta de deps | `node_modules/` | el **venv** (entorno virtual) |
| Ejecutar | `node x.js` | `python x.py` / `uv run x.py` |
| Linter/formatter | ESLint + Prettier | **`ruff`** (hace los dos, rapidísimo) |
| Type checker | `tsc` | `mypy` (o `pyright`) |
| Registro | npmjs.com | PyPI (pypi.org) |

La diferencia conceptual más importante: **el entorno virtual (venv)**. En Node, las dependencias van en `node_modules/` dentro del proyecto, aisladas por defecto. En Python, históricamente las librerías se instalaban **globales en el sistema**, lo que causaba el infierno de "el proyecto A necesita una versión y el B otra". La solución es el **venv**: un directorio aislado por proyecto con su propia versión de Python y sus paquetes —el equivalente conceptual a `node_modules`, pero que tenés que crear/activar explícitamente—.

`uv` (el gestor moderno, de los creadores de `ruff`) automatiza todo esto y es lo que te recomiendo, porque se parece muchísimo al flujo de npm:

```bash
uv init miapp           # crea pyproject.toml (como npm init)
uv add fastapi          # instala y agrega a pyproject.toml (como npm install fastapi)
uv add --dev pytest     # dependencia de desarrollo (como npm i -D)
uv run python main.py   # ejecuta en el venv del proyecto, sin activarlo a mano
uv sync                 # instala todo lo del lockfile (como npm ci)
```

Con `pip` clásico (que vas a ver en tutoriales viejos) el flujo es más manual: creás el venv (`python -m venv .venv`), lo activás (`source .venv/bin/activate`), e instalás con `pip install`. Conocelo porque lo vas a encontrar, pero usá `uv` para tus proyectos.

Regla práctica de calidad, equivalente a tu setup de TS: **`ruff` (lint + format) + `mypy` (tipos) + `pytest` (tests)** son el trío base de un proyecto Python serio, igual que ESLint+Prettier+tsc+Vitest en TS. Configurás todo en `pyproject.toml` (el `package.json` de Python).

**Ejercicios 2**
2.1 Mapeá a Python: `package.json`, `node_modules/`, `npm install -D`, `tsc`, `npm ci`.
2.2 ¿Qué problema resuelve el venv y por qué en Node no lo necesitás explícitamente?
2.3 ¿Qué hace `ruff` y qué dos herramientas del mundo JS reemplaza?
2.4 Escribí los comandos `uv` para: iniciar un proyecto, agregar `httpx` como dependencia, agregar `pytest` como dev, y correr `main.py`.

---

## Módulo 3 — Tipos y type hints: `typing` + `mypy` (el puente con TS strict)

**Teoría.** Acá tu experiencia con TS strict paga directo. Python tiene un sistema de type hints muy parecido al de TS en intención, con sintaxis distinta. Lo básico:

```python
# Anotaciones de variables y funciones
edad: int = 30
nombre: str = "Ada"

def crear_tarea(titulo: str, prioridad: int = 1) -> dict[str, str | int]:
    return {"titulo": titulo, "prioridad": prioridad}
```

El mapeo de tipos TS → Python:

| TypeScript | Python |
|---|---|
| `string` / `number` / `boolean` | `str` / `int` o `float` / `bool` |
| `string[]` | `list[str]` |
| `Record<string, number>` | `dict[str, int]` |
| `[string, number]` (tupla) | `tuple[str, int]` |
| `string \| null` | `str \| None` (u `Optional[str]`) |
| `A \| B` (union) | `A \| B` (desde 3.10) |
| `any` | `Any` (de `typing`) |
| `unknown` | `object` (lo más cercano) |
| `interface` / `type` | `TypedDict`, `Protocol`, `dataclass` (módulo 6) |
| genéricos `<T>` | `[T]` con `TypeVar` o sintaxis nueva `def f[T](...)` |

Diferencias clave que tenés que tener presentes:

- **Los hints no se chequean en runtime.** Si una función dice `-> int` y devolvés un `str`, Python **no se queja al correr** —solo `mypy` lo detecta en análisis estático—. Es como si TS no tuviera el paso de compilación obligatorio: el chequeo es opt-in vía `mypy`. Por eso, en un proyecto serio, **`mypy` en CI no es opcional** (igual que `tsc --noEmit` en tu pipeline de TS).
- **`None` es el `null`/`undefined` de Python** (hay un solo "vacío", no dos). `str | None` es tu `string | null`. `mypy` en modo estricto te obliga a chequear `None` antes de usar, igual que el `strictNullChecks` de TS.
- **`Optional[X]` es exactamente `X | None`** —vas a ver ambas formas; la moderna es `X | None`—.
- **Modo estricto.** `mypy --strict` es tu `"strict": true` del `tsconfig`: activá esto desde el día uno. Sin él, mypy deja pasar demasiado.
- **No hay un `unknown` exacto.** En la tabla puse `object` como lo más cercano, pero ojo: `object` acepta cualquier valor, pero **no te obliga a estrechar el tipo antes de usarlo** como sí hace `unknown` en TS. Es "lo más parecido", no un equivalente: la disciplina de estrechar antes de usar la ponés vos.

```python
def buscar_usuario(id: int) -> "Usuario | None": ...

u = buscar_usuario(5)
print(u.nombre)        # ❌ mypy: u podría ser None
if u is not None:
    print(u.nombre)    # ✅ narrowing, igual que en TS
```

La idea: **traés tu disciplina de TS strict tal cual; solo cambia que el chequeo es una herramienta aparte (`mypy`) que tenés que correr e integrar en CI, no parte de "ejecutar el programa".**

**Ejercicios 3**
3.1 Traducí a type hints de Python: `name: string`, `ids: number[]`, `config: Record<string, boolean>`, `user: User | null`.
3.2 Una función anotada `-> int` devuelve un string por error. ¿Falla al correr? ¿Quién lo detecta?
3.3 ¿Qué es `None` y con qué de TS/JS lo equiparás? ¿Cómo hacés narrowing para usar un `str | None`?
3.4 ¿Cuál es el equivalente de `"strict": true` del `tsconfig` y por qué hay que activarlo desde el inicio?

---

## Módulo 4 — Estructuras de datos: `list`/`dict`/`set`/`tuple` y comprehensions

**Teoría.** Las cuatro colecciones built-in de Python y su mapeo:

- **`list`** = el array de JS. Mutable, ordenada. `[1, 2, 3]`. Métodos: `.append()`, `.pop()`, slicing `lista[1:3]`.
- **`dict`** = el objeto/`Map` de JS, pero es **la** estructura central de Python (todo es dict por dentro). `{"clave": valor}`. Claves de cualquier tipo hasheable.
- **`set`** = el `Set` de JS. Sin duplicados, sin orden. `{1, 2, 3}`.
- **`tuple`** = una lista **inmutable** (no existe directo en JS; lo más cercano es un array que no mutás o `as const`). `(1, "a")`. Se usa para datos fijos y para devolver múltiples valores.

```python
tarea = {"id": 1, "titulo": "Estudiar", "tags": ["python", "ia"]}
tarea["hecha"] = False          # como obj.prop = ... en JS
titulo = tarea["titulo"]        # acceso por clave (KeyError si no existe)
titulo = tarea.get("x", "—")    # acceso seguro con default (como ?? en JS)
```

La herramienta idiomática que tenés que dominar —y que reemplaza a `.map()`/`.filter()` de JS— es la **comprehension**: una forma compacta y pythonic de construir colecciones.

```python
nums = [1, 2, 3, 4, 5]

# JS:  nums.map(n => n * 2)
dobles = [n * 2 for n in nums]                    # list comprehension

# JS:  nums.filter(n => n % 2 === 0)
pares = [n for n in nums if n % 2 == 0]           # con condición

# JS:  nums.filter(...).map(...)  encadenado
pares_x10 = [n * 10 for n in nums if n % 2 == 0]  # filtra Y transforma

# dict comprehension (no hay equivalente directo limpio en JS)
cuadrados = {n: n**2 for n in nums}               # {1:1, 2:4, 3:9, ...}
```

La comprehension es **la forma idiomática**; usar un `for` con `.append()` para construir una lista se considera menos pythonic (aunque funciona). Es el cambio de hábito más visible al venir de JS: donde tu reflejo es `.map().filter()`, en Python pensás en comprehensions.

Detalles que sorprenden viniendo de JS:

- **El `for` es `for ... in`** y siempre itera sobre elementos (como `for...of`, no `for...in` de JS). Para índices: `for i, x in enumerate(lista)`.
- **Slicing**: `lista[1:3]`, `lista[:2]`, `lista[-1]` (último), `lista[::-1]` (invertida). Mucho más expresivo que JS.
- **Verdad/falsedad**: una lista/dict/string vacíos son "falsy" (`if not lista:` = "si está vacía"). Idiomático y muy usado.
- **Desempaquetado**: `a, b = (1, 2)` y `primero, *resto = [1, 2, 3]` (como destructuring de JS).

**Ejercicios 4**
4.1 Asociá con su equivalente JS: `list`, `dict`, `set`, `tuple`. ¿Cuál no tiene equivalente directo y por qué?
4.2 Reescribí como comprehension: dada `nombres = ["ana", "leo"]`, una lista con cada nombre en mayúscula (`.upper()`).
4.3 Reescribí como comprehension con filtro: de `nums = range(10)`, los cuadrados de los números impares.
4.4 ¿Cuál es la forma idiomática de acceder a `d["x"]` cuando la clave puede no existir y querés un default? ¿Y cómo chequeás "si la lista está vacía"?

---

## Módulo 5 — Funciones: argumentos, `*args`/`**kwargs`, decoradores

**Teoría.** Las funciones en Python son más ricas en el manejo de argumentos que en JS. Lo esencial:

```python
def crear(titulo: str, prioridad: int = 1, *tags: str, **opciones: bool) -> dict:
    ...
```

- **Argumentos por defecto**: `prioridad: int = 1` (como en JS).
- **Argumentos nombrados (keyword arguments)**: podés llamar `crear("X", prioridad=3)` —pasar args por nombre, no solo por posición—. Esto es muy idiomático y hace el código legible. No tiene equivalente directo en JS (donde simulás esto con un objeto de opciones).
- **`*args`**: captura argumentos posicionales extra en una tupla (como `...rest` de JS).
- **`**kwargs`**: captura argumentos nombrados extra en un dict (no hay equivalente directo en JS).

```python
crear("Tarea", 2, "python", "ia", urgente=True)
# titulo="Tarea", prioridad=2, tags=("python","ia"), opciones={"urgente": True}
```

⚠️ **El gotcha más famoso de Python**: nunca uses un valor mutable (lista/dict) como default.

```python
def add(item, lista=[]):     # ❌ TRAMPA: la lista default se comparte entre llamadas
    lista.append(item)
    return lista
add(1)  # [1]
add(2)  # [1, 2]  ← ¡no [2]! el default se creó UNA vez y persiste

def add(item, lista=None):   # ✅ patrón correcto
    if lista is None:
        lista = []
    lista.append(item)
    return lista
```

**Decoradores.** El concepto que más vas a ver en FastAPI y que conviene entender ya. Un **decorador** es una función que envuelve a otra para agregarle comportamiento, con la sintaxis `@decorador` encima de la función. Es el equivalente conceptual a los **decoradores de NestJS** (`@Get()`, `@Injectable()`) que ya usaste —de hecho FastAPI los usa igual—:

```python
@app.get("/tareas")          # @app.get es un decorador: registra esta función como ruta GET
def listar_tareas():
    return [...]

# Un decorador propio (ej: medir tiempo), para entender qué hacen por dentro:
import functools, time
def cronometrar(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        inicio = time.perf_counter()
        resultado = func(*args, **kwargs)
        print(f"{func.__name__} tardó {time.perf_counter() - inicio:.3f}s")
        return resultado
    return wrapper

@cronometrar
def tarea_lenta(): ...
```

No necesitás escribir decoradores complejos para usar FastAPI, pero entender que `@algo` **envuelve la función de abajo** te desmitifica todo el framework. Es la misma idea que los decoradores que viste en [NestJS](nestjs.md).

**Ejercicios 5**
5.1 ¿Qué son los "keyword arguments" y qué ventaja de legibilidad dan? ¿Cómo simulás eso en JS?
5.2 ¿Qué capturan `*args` y `**kwargs` respectivamente? ¿Cuál tiene equivalente en JS y cuál no?
5.3 Explicá el gotcha del default mutable y escribí el patrón correcto.
5.4 ¿Qué es un decorador y con qué concepto de NestJS lo relacionás? ¿Qué hace `@app.get("/x")` sobre una función?

---

## Módulo 6 — Clases, `dataclass` y Pydantic (el puente a FastAPI)

**Teoría.** Las clases en Python son parecidas a las de TS con tres particularidades que chocan al principio:

```python
class Tarea:
    def __init__(self, titulo: str, prioridad: int = 1) -> None:  # el constructor
        self.titulo = titulo            # 'self' es el 'this', PERO explícito
        self.prioridad = prioridad

    def describir(self) -> str:         # 'self' va SIEMPRE como primer parámetro
        return f"{self.titulo} (p{self.prioridad})"

t = Tarea("Estudiar", 2)               # sin 'new'
print(t.describir())
```

1. **`__init__`** es el constructor (los métodos `__x__` se llaman "dunder" — double underscore — y son hooks especiales: `__init__`, `__repr__`, `__eq__`, etc.).
2. **`self` es el `this`, pero explícito**: es el primer parámetro de todo método y lo escribís siempre. No es automático como en JS.
3. **No se usa `new`**: instanciás llamando a la clase como función, `Tarea(...)`.

Para clases que son **solo datos** (lo más común en backend/IA), escribir `__init__` a mano es tedioso. Python ofrece dos atajos clave:

**`@dataclass`** — genera `__init__`, `__repr__`, `__eq__` automáticamente desde las anotaciones. Es como una clase de datos de TS sin boilerplate:

```python
from dataclasses import dataclass, field

@dataclass
class Tarea:
    titulo: str
    prioridad: int = 1
    tags: list[str] = field(default_factory=list)   # default mutable correcto

t = Tarea("Estudiar", 2)        # __init__ generado
t2 = Tarea("Estudiar", 2)
print(t == t2)                  # True: __eq__ generado compara por valor
```

**Pydantic** — el que de verdad vas a usar en IA y APIs. Es como un `dataclass` **con validación en runtime**: define un modelo con tipos, y al instanciarlo **valida y convierte** los datos, tirando error si no cumplen. Es el equivalente a **Zod** del mundo TS ([lo viste en el puente de TypeScript](typescript.md)), pero integrado al lenguaje y omnipresente en el ecosistema:

```python
from pydantic import BaseModel, Field

class CrearTarea(BaseModel):
    titulo: str = Field(min_length=1, max_length=200)
    prioridad: int = Field(default=1, ge=1, le=5)   # ge=>=, le=<=
    tags: list[str] = []   # OJO: en Pydantic el default mutable SÍ es seguro (copia por instancia)

# Valida en runtime: esto es lo que hace FastAPI con cada request body
t = CrearTarea(titulo="Estudiar", prioridad=3)       # ✅
CrearTarea(titulo="", prioridad=9)                   # ❌ ValidationError (2 errores)
```

(Detalle que parece contradecir el módulo 5: acá `tags: list[str] = []` **sí** es seguro. El gotcha del default mutable aplica a funciones y a `@dataclass`; Pydantic, en cambio, **copia el default por cada instancia** al validar, así que cada modelo arranca con su propia lista. Por eso en `@dataclass` necesitás `field(default_factory=list)` y en Pydantic no.)

Pydantic es **central** para lo que viene: FastAPI valida requests/responses con modelos Pydantic, los LLMs devuelven structured output validado con Pydantic, LangGraph tipa su estado con Pydantic. Dominar `BaseModel` es prerrequisito del resto del track. La conexión mental: **Pydantic = Zod, `dataclass` = un type/clase de datos sin validación.**

**Ejercicios 6**
6.1 ¿Qué es `self` y en qué se diferencia del `this` de JS? ¿Cómo se llama el constructor?
6.2 ¿Qué genera `@dataclass` automáticamente y para qué tipo de clases conviene?
6.3 ¿Cuál es la diferencia entre un `dataclass` y un modelo de Pydantic? ¿Con qué librería de TS equiparás Pydantic?
6.4 ¿Por qué Pydantic es prerrequisito para FastAPI y para el structured output de los LLMs?

---

## Módulo 7 — Módulos, paquetes e imports (vs ESM/CJS)

**Teoría.** El sistema de módulos de Python es más simple que el lío de ESM/CJS de Node, pero tiene sus reglas.

- **Cada archivo `.py` es un módulo.** Su nombre es el del archivo sin extensión.
- **Un paquete es una carpeta con módulos.** (Históricamente requería un `__init__.py`; hoy no siempre, pero lo vas a ver.)
- **Imports** —el mapeo con ESM—:

```python
import math                          # importa el módulo entero; usás math.sqrt(...)
from math import sqrt, pi            # importa nombres puntuales (como import { x } de ESM)
from math import sqrt as raiz        # con alias (como import { x as y })
from mi_paquete.servicios import TareaService   # import de tu propio código
```

No hay `export`: **todo lo que está a nivel de módulo es importable** por defecto (no hace falta marcarlo). La convención para señalar "esto es privado" es un guion bajo al inicio (`_funcion_interna`), pero es solo convención, no lo impide el lenguaje.

El idiom que **siempre** vas a ver y tenés que reconocer:

```python
def main() -> None:
    ...

if __name__ == "__main__":      # "si este archivo se ejecutó directamente (no fue importado)"
    main()
```

`__name__` vale `"__main__"` cuando corrés el archivo directo (`python main.py`), y vale el nombre del módulo cuando fue importado. Ese `if` es el equivalente al patrón de "código que solo corre si soy el entry point" —evita que el código de arranque se ejecute al importar el archivo desde otro—. Es idiomático y aparece en casi todos los scripts.

Gotcha viniendo de Node: los **imports son absolutos desde la raíz del proyecto** por defecto, no relativos al archivo como el `./x` de Node. Los imports relativos existen (`from .servicios import X`, con punto) pero solo dentro de paquetes y son una fuente común de confusión al principio. Regla práctica: estructurá tu proyecto como paquete y usá imports absolutos desde la raíz.

**Ejercicios 7**
7.1 ¿Cómo importás solo `sqrt` del módulo `math`? ¿Y con un alias?
7.2 ¿Cómo se marca algo como "exportable" en Python comparado con el `export` de ESM?
7.3 Explicá qué hace `if __name__ == "__main__":` y por qué se usa.
7.4 ¿Qué diferencia hay entre los imports de Python y los `./archivo` relativos de Node?

---

## Módulo 8 — Errores y context managers (`try/except`, `with`)

**Teoría.** El manejo de errores es conceptualmente igual al de JS, con otros nombres:

```python
try:
    resultado = arriesgado()
except ValueError as e:              # 'except', no 'catch'; capturás por TIPO
    print(f"valor inválido: {e}")
except (KeyError, IndexError) as e:  # múltiples tipos juntos
    print(f"no encontrado: {e}")
except Exception as e:               # el catch-all (como catch (e) genérico)
    raise                            # 're-lanza' (como throw sin argumento)
else:
    print("sin errores")             # corre si NO hubo excepción (poco común en JS)
finally:
    limpiar()                        # siempre corre (como finally de JS)
```

Diferencias con JS que importan:

- **Capturás por tipo de excepción**, no un `catch (e)` genérico que después discriminás. `except ValueError` solo atrapa ese tipo —más cercano a tu disciplina de [errores tipados en TS](typescript.md)—. Esto empuja a ser específico.
- **Las excepciones son clases** que heredan de `Exception`. Creás las tuyas heredando: `class TareaNoEncontrada(Exception): ...`.
- **`raise`** es el `throw`. `raise ValueError("mensaje")` o `raise` solo (re-lanza la actual).
- **EAFP vs LBYL**: Python prefiere el estilo "es más fácil pedir perdón que permiso" (*Easier to Ask Forgiveness than Permission*): **intentar y capturar** en vez de chequear todo antes. Donde en JS quizás harías `if (obj.x !== undefined)`, en Python a veces es más idiomático `try: obj.x except KeyError:`. No es regla absoluta, pero es un cambio de cultura.

**Context managers (`with`)** — el concepto sin equivalente directo en JS que tenés que aprender. Un context manager garantiza que un recurso se **abre y se cierra** correctamente, incluso si hay una excepción —reemplaza el patrón `try/finally` para cerrar recursos—:

```python
# Sin with: tenés que acordarte de cerrar, y manejar el finally
f = open("datos.txt")
try:
    contenido = f.read()
finally:
    f.close()

# Con with: el archivo se cierra solo al salir del bloque, pase lo que pase
with open("datos.txt") as f:
    contenido = f.read()
# acá f ya está cerrado, garantizado
```

Lo vas a usar todo el tiempo: archivos, conexiones a DB, sesiones HTTP, locks. La idea: **`with recurso as x:` = "usá `x` dentro del bloque y cerralo automáticamente al salir"**. Es el patrón idiomático para cualquier cosa que haya que liberar.

**Ejercicios 8**
8.1 Mapeá a Python: `catch`, `throw`, `finally`. ¿Cuál es la diferencia clave en *cómo* se captura respecto de JS?
8.2 ¿Cómo creás una excepción propia `TareaNoEncontrada`?
8.3 ¿Qué es el estilo EAFP y cómo contrasta con chequear antes (LBYL)? Dá un ejemplo.
8.4 ¿Qué problema resuelve `with` y qué patrón de `try/finally` reemplaza? Dá un caso de uso típico.

---

## Módulo 9 — Async: `asyncio` y `async`/`await` (vs el event loop de Node)

**Teoría.** Acá tu conocimiento del [event loop de Node](nodejs.md) se transfiere casi entero, porque el modelo es el mismo: **un solo hilo, concurrencia cooperativa vía un event loop, async para I/O**. La sintaxis es prácticamente idéntica:

```python
import asyncio
import httpx

async def traer_tarea(id: int) -> dict:        # 'async def' define una corrutina
    async with httpx.AsyncClient() as client:
        r = await client.get(f"https://api/tareas/{id}")   # await igual que en JS
        return r.json()

async def main() -> None:
    # Concurrencia: lanzar varias a la vez y esperar todas (como Promise.all)
    resultados = await asyncio.gather(
        traer_tarea(1), traer_tarea(2), traer_tarea(3)
    )
    print(resultados)

asyncio.run(main())     # arranca el event loop (en JS el runtime lo hace por vos)
```

El mapeo con JS:

| JavaScript | Python (`asyncio`) |
|---|---|
| `async function` | `async def` (define una **corrutina**) |
| `await promesa` | `await corrutina` (igual) |
| `Promise.all([...])` | `asyncio.gather(...)` |
| `Promise.race([...])` | `asyncio.wait(..., return_when=FIRST_COMPLETED)` |
| el runtime arranca el loop solo | `asyncio.run(main())` lo arrancás vos |
| `setTimeout` | `await asyncio.sleep(s)` |

(Ojo con `Promise.race`: `asyncio.wait` **no** es un drop-in. La constante es `asyncio.FIRST_COMPLETED`, espera *tasks* (no corrutinas peladas) y devuelve dos sets `(done, pending)` —vos sacás el resultado del set `done`—, no el valor directo. Para iterar resultados a medida que terminan, lo idiomático hoy es `asyncio.as_completed`.)

Las diferencias que importan:

- **El loop lo arrancás explícitamente** con `asyncio.run(...)`. En Node, el runtime ya está corriendo el loop; en Python, en un script, vos lo iniciás. (En FastAPI esto lo maneja el framework, igual que en Node.)
- **Una corrutina no corre hasta que la "await-eás" o la programás.** Llamar `traer_tarea(1)` solo *crea* la corrutina; no hace nada hasta el `await` o el `gather`. (En JS, llamar una `async function` ya arranca la promesa.) Gotcha clásico: olvidarte el `await` y que "no pase nada".
- **No mezcles sync bloqueante con async.** Igual que en Node un cómputo pesado bloquea el event loop, en asyncio una llamada **bloqueante** (un `requests.get` síncrono, un `time.sleep`) congela el loop entero. Usás librerías async (`httpx`, `asyncpg`) o mandás el trabajo bloqueante a un thread/proceso —el mismo problema y la misma solución que viste en Node con el thread pool y los workers—.
- **Python también tiene threads y multiprocessing**, pero por el **GIL** (Global Interpreter Lock — un candado que permite que solo un hilo ejecute bytecode Python a la vez), los threads **no dan paralelismo de CPU real**; sirven para I/O. Para paralelismo de CPU se usa `multiprocessing` (varios procesos). Es el análogo del "Node es single-thread, usá cluster/workers para CPU" de tu módulo de Node: misma conclusión, mecanismo distinto (GIL).

La idea: **si entendés el event loop de Node, entendés asyncio —es el mismo modelo mental—; solo cambian nombres y que el loop lo arrancás vos.**

**Ejercicios 9**
9.1 Mapeá a asyncio: `async function`, `Promise.all`, `setTimeout(fn, 1000)`. ¿Qué tenés que hacer en un script Python que en Node hace el runtime solo?
9.2 ¿Qué pasa si llamás `traer_tarea(1)` sin `await` ni `gather`? ¿En qué se diferencia de llamar una async function en JS?
9.3 ¿Por qué un `time.sleep(5)` síncrono dentro de código async es un problema, y con qué problema de Node es análogo?
9.4 ¿Qué es el GIL y qué implica para usar threads para cómputo de CPU? ¿Cuál es la salida y con qué patrón de Node se corresponde?

---

## Módulo 10 — Testing con `pytest` (vs Jest/Vitest)

**Teoría.** Para un rol de **AI QA** esto es central, y tu base de [testing](testing.md) y [TDD](tdd.md) se transfiere completa —cambia la herramienta, no la disciplina—. El estándar de Python es **`pytest`**, y es más liviano que Jest: los tests son **funciones** que empiezan con `test_`, y se asierta con el `assert` plano de Python (pytest reescribe el assert para darte mensajes ricos).

```python
# test_tareas.py  (pytest descubre archivos test_*.py y funciones test_*)
from tareas import crear_tarea
import pytest

def test_crea_tarea_con_titulo():
    t = crear_tarea("Estudiar")
    assert t.titulo == "Estudiar"          # assert plano, no expect(...).toBe(...)
    assert t.prioridad == 1

def test_titulo_vacio_falla():
    with pytest.raises(ValueError):        # como expect(...).toThrow()
        crear_tarea("")
```

El mapeo con Jest/Vitest:

| Jest/Vitest | pytest |
|---|---|
| `describe`/`it` | funciones `test_*` (o clases `Test*`) |
| `expect(x).toBe(y)` | `assert x == y` |
| `expect(fn).toThrow()` | `with pytest.raises(Error):` |
| `beforeEach` | **fixtures** (más potentes, ver abajo) |
| mocks (`vi.fn()`) | `unittest.mock` / `pytest-mock` |
| `test.each([...])` | `@pytest.mark.parametrize(...)` |

La pieza distintiva y más potente de pytest: las **fixtures**. En vez de `beforeEach`, definís funciones decoradas con `@pytest.fixture` que **proveen** lo que un test necesita, y el test las recibe **por nombre como parámetro** (inyección de dependencias en los tests):

```python
@pytest.fixture
def proyecto():                     # provee un objeto listo para varios tests
    # Project/Task son el mismo dominio del ciclo en vivo de TDD (tdd.md), ahora en Python
    return Project(id=7, member_ids=[4521])

def test_asigna_a_miembro(proyecto):    # pytest "inyecta" la fixture por nombre
    tarea = Task(id=99)
    tarea.assign_to(4521, proyecto)
    assert tarea.assignee_id == 4521

# Parametrización: un test, muchos casos (como test.each)
@pytest.mark.parametrize("entrada,esperado", [("", False), ("ok", True)])
def test_validar(entrada, esperado):
    assert validar(entrada) == esperado
```

Para el track de IA/QA, esto conecta directo:
- La **pirámide de tests**, los dobles de prueba, el criterio classicist/mockist, los cuatro atributos de un buen test ([todo lo de testing.md y tdd.md](tdd.md)) **aplican igual** —son lenguaje-agnósticos—.
- Para **API testing** de FastAPI vas a usar `pytest` + `httpx`/`TestClient` (el equivalente del Supertest que usaste con Nest).
- Para **testear sistemas de IA** (no deterministas), pytest es la base sobre la que corren las evals ([evals.md](evals.md)): los golden sets y los asserts de "faithfulness" se orquestan con pytest.

La idea: **toda tu disciplina de testing/TDD se mantiene; aprendés la sintaxis de pytest (assert plano + fixtures + parametrize) y listo.**

**Ejercicios 10**
10.1 ¿Cómo descubre pytest qué es un test? ¿Cómo se asierta una igualdad y cómo se asierta que algo lanza una excepción?
10.2 ¿Qué son las fixtures y qué reemplazan de Jest? ¿Cómo las recibe un test?
10.3 ¿Cuál es el equivalente de `test.each` y para qué sirve?
10.4 ¿Qué parte de tu conocimiento de testing/TDD NO cambia al pasar a pytest, y qué herramienta usarías para API testing de FastAPI?

---

## Módulo 11 — El criterio: cuándo Python, cuándo Node, y qué es "pythonic"

**Teoría.** El módulo de cierre. Ya podés leer y escribir Python; ahora el criterio para usarlo bien y no escribir "JavaScript con sintaxis de Python".

**Cuándo Python y cuándo Node** —porque ahora sos bilingüe y la elección es tuya—:

- **Python gana** en: IA/ML (todo el ecosistema —modelos, vectores, agentes—), data science y análisis de datos, scripting y automatización, y cualquier cosa donde la librería que necesitás solo existe en Python (FAISS, Whisper, transformers). Para un **AI Engineer**, el backend de IA suele ser Python.
- **Node/TS gana** en: backend web de alta concurrencia de I/O, full-stack con un solo lenguaje (front + back en TS), tiempo real (WebSockets), y donde ya tenés el ecosistema y el equipo en TS. Tu [track de NestJS](nestjs.md) sigue siendo la opción fuerte para una API de producto.
- **En la práctica conviven**: es muy común una API de producto en Node/TS que llama a un **servicio de IA en Python** (FastAPI) por HTTP. No es "Python *o* Node"; es "cada uno donde brilla". El criterio es el mismo "[escalá la herramienta al problema](liderazgo.md)" de todo el temario.

**Qué es "pythonic"** —escribir Python idiomático, no JS disfrazado—:

- **Comprehensions** sobre `for`+`append` (módulo 4).
- **`with`** para recursos, no `try/finally` manual (módulo 8).
- **EAFP** (intentar/capturar) donde encaja, no chequear todo antes (módulo 8).
- **Desempaquetado** (`a, b = ...`) y `enumerate`/`zip` en vez de índices manuales.
- **Nombres `snake_case`**, funciones cortas, y aprovechar la librería estándar (que es enorme: `itertools`, `collections`, `pathlib`).
- **Type hints + mypy estricto siempre** (módulo 3): tu disciplina de TS no se abandona, se traslada.
- **El Zen de Python** (`import this`): explícito mejor que implícito, simple mejor que complejo, legibilidad cuenta. Cuando dudes entre dos formas, elegí la más legible y obvia.

**El criterio de calidad para el track que viene**: un proyecto Python serio de IA tiene el mismo rigor que pedías en TS —`ruff` + `mypy --strict` + `pytest` en CI, Pydantic para validar los bordes, async para el I/O, y tests que verifican comportamiento—. La IA no es excusa para bajar la vara: los módulos de [FastAPI](fastapi.md), [vector DBs](vector-dbs.md), [agentes Python](ai-agents-python.md), [voice](voice-ai.md) y [deploy de IA](deploy-ai.md) que siguen asumen este piso.

El cierre y el puente: con esto tenés la **base del track de IA en Python**. Mapeaste todo tu conocimiento de TS/Node —tipos, async, testing, módulos, errores— al mundo Python, y sumaste lo distintivo (comprehensions, `with`, Pydantic, pytest, el GIL). Sos **bilingüe**: podés elegir la herramienta correcta y, sobre todo, entrar al ecosistema de IA donde vive (FastAPI, LangGraph, FAISS, Whisper). El próximo módulo, [FastAPI](fastapi.md), es la entrada natural: tu primer servicio Python de verdad, que es además la forma estándar de servir un modelo o un agente de IA.

**Ejercicios 11**
11.1 Dá dos escenarios donde elegirías Python sobre Node y dos donde elegirías Node sobre Python, con el porqué.
11.2 ¿Por qué "Python o Node" suele ser una falsa dicotomía en un sistema de IA real? Dá la arquitectura típica.
11.3 Nombrá cuatro hábitos que hacen a un código "pythonic" en vez de "JS con sintaxis Python".
11.4 ¿Qué piso de calidad (tooling) debería tener un proyecto Python de IA serio, y por qué la IA no es excusa para bajarlo?

---

## Soluciones

### Módulo 1
1.1 Porque el ecosistema de IA/ML vive en Python: frameworks, librerías de modelos y vectores, y casi todo el material asumen Python. Dos ejemplos donde TS no alcanza: usar **FAISS** (índice vectorial local, solo Python) y armar un **grafo de agentes con LangGraph** (framework Python); también servir/correr un modelo propio (Whisper, transformers).
1.2 La **indentación significativa** (sangría, 4 espacios) tras un `:`. No es cosmético porque la sangría *define* los bloques: un espacio de más o de menos cambia la estructura o es error de sintaxis.
1.3 `mypy` hace análisis estático de tipos sobre los type hints, el mismo rol que `tsc` sobre TS. Diferencia: los hints de Python **no afectan la ejecución** (el intérprete los ignora en runtime); en TS los tipos se borran al compilar pero hay un paso de build obligatorio. En Python el chequeo es opt-in vía mypy.
1.4 `getUserById → get_user_by_id`, `MaxRetries → MAX_RETRIES` (si es constante) o `MaxRetries`→`max_retries` (si es variable), `userProfile → user_profile`.

### Módulo 2
2.1 `package.json → pyproject.toml`; `node_modules/ → el venv`; `npm install -D → uv add --dev`; `tsc → mypy`; `npm ci → uv sync`.
2.2 El venv aísla las dependencias y la versión de Python **por proyecto**, evitando el conflicto de versiones entre proyectos (en Python las libs históricamente se instalaban globales). En Node no lo necesitás explícitamente porque `node_modules/` ya aísla por proyecto automáticamente.
2.3 `ruff` hace **linting y formateo** (rapidísimo), reemplazando a **ESLint + Prettier**.
2.4 `uv init`, `uv add httpx`, `uv add --dev pytest`, `uv run python main.py`.

### Módulo 3
3.1 `name: str`, `ids: list[int]`, `config: dict[str, bool]`, `user: User | None` (o `Optional[User]`).
3.2 **No falla al correr** (los hints se ignoran en runtime). Lo detecta **`mypy`** en análisis estático (por eso debe estar en CI).
3.3 `None` es el único valor "vacío" de Python; equivale a `null`/`undefined` de JS (unificados en uno). Narrowing: `if u is not None:` y dentro del bloque ya lo tratás como no-None (igual que el narrowing de TS).
3.4 `mypy --strict` es el equivalente de `"strict": true`. Hay que activarlo desde el inicio porque sin él mypy deja pasar demasiado (variables sin tipar, `Any` implícitos), perdiendo la red de seguridad.

### Módulo 4
4.1 `list → array`, `dict → objeto/Map`, `set → Set`. `tuple` no tiene equivalente directo: es una secuencia **inmutable** (lo más cercano sería un array que no mutás o `as const`). 
4.2 `[n.upper() for n in nombres]`.
4.3 `[n**2 for n in range(10) if n % 2 == 1]`.
4.4 `d.get("x", default)` para acceso seguro con default (como `??`). Lista vacía: `if not lista:` (las colecciones vacías son falsy).

### Módulo 5
5.1 Son argumentos pasados **por nombre** (`crear("X", prioridad=3)`); dan legibilidad (en la llamada se ve qué es cada valor) y permiten saltarse defaults intermedios. En JS se simula pasando un objeto de opciones (`crear("X", { prioridad: 3 })`).
5.2 `*args` captura argumentos **posicionales** extra en una tupla (equivale a `...rest` de JS); `**kwargs` captura argumentos **nombrados** extra en un dict (no hay equivalente directo en JS).
5.3 Un default mutable (`lista=[]`) se crea **una sola vez** al definir la función y se **comparte** entre llamadas, acumulando estado. Patrón correcto: default `None` y dentro `if lista is None: lista = []`.
5.4 Un decorador es una función que **envuelve** a otra para agregarle comportamiento, con sintaxis `@decorador`. Se relaciona con los decoradores de **NestJS** (`@Get()`, `@Injectable()`). `@app.get("/x")` registra la función de abajo como handler de la ruta GET `/x`.

### Módulo 6
6.1 `self` es la referencia a la instancia (el `this`), pero es **explícito**: va como primer parámetro de cada método y se escribe siempre. El constructor es `__init__`.
6.2 Genera `__init__`, `__repr__` y `__eq__` automáticamente desde las anotaciones. Conviene para clases que son **principalmente datos** (sin mucha lógica), evitando boilerplate.
6.3 El `dataclass` solo estructura datos (sin validación); **Pydantic** valida y convierte en runtime al instanciar (tira `ValidationError` si no cumple). Pydantic equivale a **Zod** de TS.
6.4 Porque FastAPI usa modelos Pydantic para **validar y tipar** request bodies y responses, y los LLMs devuelven **structured output** validado con Pydantic; sin dominar `BaseModel` no podés usar ninguno de los dos.

### Módulo 7
7.1 `from math import sqrt`; con alias: `from math import sqrt as raiz`.
7.2 No hay que marcar nada: **todo lo de nivel de módulo es importable** por defecto. La convención de "privado" es el guion bajo inicial (`_x`), pero es solo convención, no se aplica.
7.3 `__name__` vale `"__main__"` solo si el archivo se ejecuta directamente; el `if` hace que ese código (típicamente arrancar `main()`) corra **solo como entry point** y no cuando el archivo se importa desde otro módulo.
7.4 Los imports de Python son **absolutos desde la raíz del proyecto** por defecto (`from paquete.modulo import X`), no relativos al archivo como el `./x` de Node. Los relativos existen (`from .x import Y`) pero solo dentro de paquetes.

### Módulo 8
8.1 `catch → except`, `throw → raise`, `finally → finally`. Diferencia clave: en Python capturás **por tipo de excepción** (`except ValueError`), no un `catch (e)` genérico que discriminás después.
8.2 `class TareaNoEncontrada(Exception): pass` (hereda de `Exception`).
8.3 EAFP ("es más fácil pedir perdón que permiso") = **intentar la operación y capturar** la excepción si falla, en vez de chequear todas las precondiciones antes (LBYL). Ejemplo: `try: valor = d["x"] except KeyError: ...` en vez de `if "x" in d: ...`.
8.4 `with` garantiza que un recurso se **cierra/libera** al salir del bloque, pase lo que pase (incluso con excepción), reemplazando el `try/finally` manual de cierre. Caso típico: `with open(archivo) as f:` (también conexiones a DB, sesiones HTTP, locks).

### Módulo 9
9.1 `async function → async def`; `Promise.all → asyncio.gather`; `setTimeout(fn, 1000) → await asyncio.sleep(1)` (en una corrutina). En un script Python tenés que **arrancar el loop vos** con `asyncio.run(main())`; en Node el runtime ya lo corre.
9.2 Solo **crea la corrutina**, no la ejecuta (no pasa nada hasta el `await` o `gather`). En JS, llamar una async function **ya arranca** la promesa; en Python no. Gotcha: olvidar el `await`.
9.3 Porque `time.sleep` es **bloqueante** y congela el event loop entero (igual que un cómputo pesado en Node bloquea su loop). Es análogo al "no bloquees el event loop de Node"; la salida es usar versiones async (`asyncio.sleep`, `httpx`) o mandar el trabajo a otro hilo/proceso.
9.4 El **GIL** es un candado que permite que solo un hilo ejecute bytecode Python a la vez, así que los threads **no dan paralelismo de CPU real** (sí sirven para I/O). La salida para CPU es `multiprocessing` (varios procesos). Se corresponde con "Node es single-thread → usá cluster/workers para CPU".

### Módulo 10
10.1 pytest descubre archivos `test_*.py` y funciones `test_*`. Igualdad: `assert x == y` (assert plano). Excepción: `with pytest.raises(TipoError):`.
10.2 Las **fixtures** (`@pytest.fixture`) proveen objetos/estado preparados para los tests, reemplazando a `beforeEach` (y son más potentes: con scope, composables). El test las recibe **por nombre como parámetro** (inyección).
10.3 `@pytest.mark.parametrize("args", [casos...])` es el equivalente de `test.each`: corre el mismo test con muchos conjuntos de datos.
10.4 **No cambia** la disciplina: pirámide de tests, dobles de prueba, classicist vs mockist, los cuatro atributos de un buen test, TDD (red-green-refactor) —todo es lenguaje-agnóstico—. Para API testing de FastAPI usarías `pytest` + el `TestClient`/`httpx` (equivalente a Supertest).

### Módulo 11
11.1 **Python**: (a) un servicio de IA que necesita FAISS/Whisper/transformers; (b) scripting/análisis de datos. **Node**: (a) una API web de producto de alta concurrencia I/O o tiempo real (WebSockets); (b) full-stack en un solo lenguaje (front+back TS). El porqué: cada ecosistema brilla en su dominio.
11.2 Porque lo normal es que **convivan**: una API de producto en Node/TS que llama por HTTP a un **servicio de IA en Python (FastAPI)**. No competís lenguajes; ponés cada uno donde es más fuerte (el mismo "escalá la herramienta al problema").
11.3 (Cuatro de): comprehensions en vez de `for`+`append`; `with` en vez de `try/finally` manual; EAFP donde encaja; desempaquetado/`enumerate`/`zip`; `snake_case`; aprovechar la librería estándar; type hints + mypy estricto.
11.4 `ruff` + `mypy --strict` + `pytest` en CI, Pydantic para validar los bordes, async para I/O, tests de comportamiento. La IA no es excusa para bajar la vara porque un sistema de IA en producción tiene los mismos requisitos de correctitud y mantenibilidad que cualquier backend serio —y encima la complejidad extra del no determinismo—.

---

Con este módulo tenés la **base del track de IA en Python**: mapeaste tu conocimiento de TS/Node (tipos, async, testing, módulos, errores) al mundo Python y sumaste lo distintivo (comprehensions, `with`, Pydantic, pytest, el GIL). Ahora sos bilingüe y podés entrar al ecosistema de IA donde vive. Lo que sigue construye sobre esto: [FastAPI](fastapi.md) (tu primer servicio Python, y la forma estándar de servir IA), [Vector Databases](vector-dbs.md) (Pinecone/FAISS/Weaviate y RAG en Python), [AI Agents](ai-agents-python.md) (LangGraph/AutoGen), [Voice AI](voice-ai.md) y [Deploy de aplicaciones de IA](deploy-ai.md). El track corre en paralelo al de IA en TS/Anthropic que ya tenés —los mismos conceptos, el otro ecosistema— para que seas contratable en ambos mundos.
