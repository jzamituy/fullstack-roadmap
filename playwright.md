# Playwright: tests E2E de browser que no son flaky

**Automatización de UI end-to-end · Playwright + pytest (Python) · locators, auto-waiting y CI · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este es el primer módulo del **track QA / Automation**, apuntado al perfil **AI QA / SDET**. Asume [Python](python.md) (pytest, fixtures, async) y el módulo de [Testing práctico](testing.md) —de ahí viene la **estrategia**: la pirámide de tests, qué mockear, por qué el E2E va arriba y se usa con cuidado—. **No repetimos esa teoría; la usamos.** El **E2E de browser** es la prueba que más se parece a un usuario real: abre un navegador, hace clic, escribe, y verifica lo que ve en pantalla. Es la más valiosa (prueba el sistema entero, de verdad) y la más peligrosa (lenta, cara, y propensa a ser **flaky** —fallar sin que nada se rompió—). El objetivo del módulo es doble: las **técnicas** (locators robustos, auto-waiting, mockeo de red, CI) y el **criterio** (cuánto E2E, qué probar con él y qué no, y cómo no terminar con una suite que nadie confía).
>
> **Por qué Python acá.** Playwright es multi-lenguaje; elegimos **Python + pytest** para que el track QA enganche con [Python](python.md) y con el perfil AI QA. Toda la teoría (locators, auto-waiting, flakiness, POM) es idéntica en la versión TypeScript —cambia la sintaxis, no las ideas—.

**Lo que asumimos.** Python y pytest ([Python](python.md)), la pirámide y los conceptos de [Testing](testing.md), y una app web con backend (la "Task API" del roadmap servida con un front, o cualquier app con login + un CRUD).

**Para practicar.**

```bash
uv add --dev pytest-playwright
playwright install            # baja los navegadores (Chromium, Firefox, WebKit)
# correr:  pytest         |  ver navegador:  pytest --headed         |  un browser:  pytest --browser firefox
```

> Nota sobre datos volátiles: la API de Playwright es estable, pero flags de CLI, la integración con CI y los nombres de imágenes Docker cambian entre versiones. Verificá contra `playwright.dev/python` al implementar.

**Índice de módulos**
1. El E2E en su lugar: la cima de la pirámide y el criterio
2. Por qué Playwright (vs Selenium/Cypress)
3. Tu primer test: el fixture `page`, navegar, interactuar, afirmar
4. Locators: el corazón de un E2E confiable (`get_by_role` y la estrategia user-facing)
5. Auto-waiting y web-first assertions: el fin de los `sleep()`
6. Flakiness: causas y curas
7. Page Object Model: estructurar para mantener (y cuándo es over-engineering)
8. Autenticación y estado: `storage_state`, no loguear por UI cada vez
9. Interceptar y mockear la red: `route`/`fulfill`
10. Debugging y CI: trace viewer, codegen, sharding y contenedores
11. Testear features de IA y el criterio de cierre

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El E2E en su lugar: la cima de la pirámide y el criterio

**Teoría.** Antes de escribir un solo test, el criterio, porque el error #1 en automatización de UI es **escribir demasiados E2E**. En la pirámide de [Testing](testing.md), el E2E está **arriba**: pocos, porque cada uno es **lento** (arranca un navegador, navega, espera renders), **caro** (de correr y de mantener) y **frágil** (un cambio de UI o un timing lo rompe). Abajo están los unitarios (muchos, rápidos, baratos) y en el medio los de integración.

La regla de oro del E2E: **probá los pocos *flujos de usuario críticos* de punta a punta, no cada detalle.** Un E2E debe responder "¿el usuario puede lograr lo que vino a hacer?" —registrarse, loguearse, crear y completar una tarea, pagar—. Los casos borde, las validaciones, las ramas raras: eso se prueba **más abajo** (unit/integración), que es más rápido y preciso. Si tenés 200 tests E2E, casi seguro estás probando con E2E cosas que un test más barato cubría mejor.

Por qué importa esta disciplina: una suite E2E grande y lenta que falla seguido (flaky, módulo 6) genera lo peor que le puede pasar a un equipo de testing: **que nadie confíe en los tests.** Cuando "siempre falla algo" y el reflejo es "dale retry y mergeá", los tests dejaron de aportar señal —son ruido caro—. Pocos E2E sólidos sobre los flujos que importan dan más confianza que cientos de tests frágiles.

El E2E es **caja negra**: no le importa cómo está hecho el código por dentro, solo lo que el usuario ve y hace. Eso es su fuerza (prueba el sistema **entero** —front, back, base de datos, integración real—, lo que ningún test más abajo hace) y su debilidad (cuando falla, te dice *que* algo anda mal, no *dónde*). La frase mental: **el E2E es la cima de la pirámide —el más realista y el más caro/frágil—; usalo para los pocos flujos de usuario críticos de punta a punta, y dejá los detalles y casos borde para tests más abajo, porque una suite E2E enorme y flaky destruye la confianza que un test debería dar—.**

**Ejercicios 1**
1.1 ¿Por qué el E2E va arriba de la pirámide y se usa con moderación? Nombrá sus tres costos.
1.2 ¿Cuál es la regla de oro de qué probar con E2E y qué dejar para tests más abajo?
1.3 ¿Qué es lo peor que le pasa a un equipo cuando la suite E2E es grande y flaky?
1.4 ¿Por qué se dice que el E2E es "caja negra", y cuál es la fuerza y la debilidad de eso?

---

## Módulo 2 — Por qué Playwright (vs Selenium/Cypress)

**Teoría.** Hay varias herramientas para automatizar un navegador; Playwright (Microsoft) se volvió el estándar moderno, y entender **por qué** te dice qué problemas resuelve. Contra los predecesores:

- **vs Selenium** (el clásico): Selenium controla el navegador por un protocolo externo (WebDriver) y **no espera solo** —tenías que poner *waits* manuales por todos lados, la fuente histórica de la flakiness—. Playwright tiene **auto-waiting** integrado (módulo 5): espera automáticamente a que los elementos estén listos antes de actuar. Es la diferencia más importante.
- **vs Cypress**: Cypress es excelente pero corre **dentro del navegador** (lo que limita multi-tab, multi-origen, y solo soportó bien Chromium por mucho tiempo) y es **JS/TS only**. Playwright corre **por fuera** controlando el browser, soporta **múltiples lenguajes** (Python, TS, Java, .NET) y maneja escenarios complejos (varias pestañas, varios orígenes, descargas) sin problema.

Las capacidades que definen a Playwright:

- **Cross-browser de verdad**: corre los mismos tests en **Chromium, Firefox y WebKit** (el motor de Safari) —cubrís los tres motores reales del mercado—.
- **Auto-waiting y web-first assertions** (módulos 5): el antídoto a la flakiness, integrado.
- **Browser contexts**: cada test puede correr en un **contexto aislado** —como una sesión de incógnito fresca, con sus propias cookies/storage—, livianos de crear. Esto da **aislamiento barato** (módulo 6) y **paralelización** real.
- **Herramientas de primera**: **trace viewer** (una repetición visual del test paso a paso), **codegen** (graba tus clics y genera el test), inspector (módulo 10).

El modelo de objetos, de afuera hacia adentro: **Browser** (el navegador) → **BrowserContext** (una sesión aislada) → **Page** (una pestaña). En pytest-playwright trabajás casi siempre con la **`page`**, que el plugin te da ya lista por test, en su propio contexto. La frase mental: **Playwright ganó por el auto-waiting integrado (mata la flakiness que Selenium dejaba a tu cargo), por correr por fuera del navegador con soporte multi-lenguaje y cross-browser real (Chromium/Firefox/WebKit), y por los browser contexts aislados que dan paralelización y aislamiento baratos—.**

**Ejercicios 2**
2.1 ¿Cuál es la diferencia más importante entre Playwright y Selenium, y qué problema histórico resuelve?
2.2 ¿En qué se diferencia Playwright de Cypress (dónde corre cada uno, lenguajes, escenarios)?
2.3 ¿Qué es un browser context y qué dos cosas habilita?
2.4 Nombrá el modelo de objetos Browser → Context → Page y con cuál trabajás en cada test.

---

## Módulo 3 — Tu primer test: el fixture `page`, navegar, interactuar, afirmar

**Teoría.** `pytest-playwright` integra Playwright con pytest ([Python](python.md)): te da fixtures listas —sobre todo **`page`**, una pestaña en un contexto aislado, una por test—. Un test E2E es la estructura **AAA** de [Testing](testing.md) llevada al navegador: navegás (arrange), interactuás (act), afirmás lo que se ve (assert).

```python
from playwright.sync_api import Page, expect

def test_crear_una_tarea(page: Page):
    # Arrange: ir a la app
    page.goto("http://localhost:5173")

    # Act: interactuar como un usuario
    page.get_by_role("button", name="Nueva tarea").click()
    page.get_by_label("Título").fill("Comprar pan")
    page.get_by_role("button", name="Guardar").click()

    # Assert: verificar lo que el usuario ve (web-first assertion, módulo 5)
    expect(page.get_by_text("Comprar pan")).to_be_visible()
```

Lo que hay que ver:

- **`page.goto(url)`**: abre la URL. Playwright **espera** a que la página cargue (no necesitás un wait manual).
- **Interacciones**: `.click()`, `.fill(texto)`, `.check()`, `.select_option(...)`, `.press("Enter")`. Operan sobre **locators** (módulo 4), no sobre selectores crudos.
- **`expect(locator).to_be_visible()`**: la **web-first assertion** —reintenta automáticamente hasta que se cumpla o venza el timeout (módulo 5)—. Es distinta del `assert` de pytest (que evalúa una vez): `expect` de Playwright **espera**.

Nota de API: usamos la **API síncrona** (`sync_api`), la más simple y la recomendada para empezar. Playwright también tiene API **async** (`async_api`, con `async def`/`await`) —útil si tu suite ya es async o necesitás concurrencia fina—, pero la sync alcanza para la enorme mayoría de los tests y es más legible. La frase mental: **un test E2E es AAA en el navegador: `page.goto` (arrange), interacciones sobre locators como `.click()`/`.fill()` (act), y `expect(locator).to_be_visible()` (assert), donde el fixture `page` te da una pestaña aislada por test y `expect` reintenta en vez de evaluar una sola vez—.**

**Ejercicios 3**
3.1 ¿Qué te da el fixture `page` de pytest-playwright y qué garantiza respecto al aislamiento?
3.2 Mapeá las tres partes de AAA a un test E2E con un ejemplo de cada una.
3.3 ¿Qué hace `page.goto` además de navegar, y por qué eso evita un wait manual?
3.4 ¿En qué se diferencia `expect(locator).to_be_visible()` de un `assert` normal de pytest?

---

## Módulo 4 — Locators: el corazón de un E2E confiable (`get_by_role` y la estrategia user-facing)

**Teoría.** Si te llevás una sola cosa del módulo, que sea esta: **la elección de locators es lo que más determina si tu suite E2E es robusta o un infierno de mantenimiento.** Un *locator* es cómo le decís a Playwright qué elemento querés. Y hay una jerarquía clara de mejor a peor, según **cuánto se parece a cómo un usuario (o un lector de pantalla) percibe la página**.

Los locators **user-facing** (los que Playwright recomienda, en orden de preferencia):

- **`get_by_role`**: por rol de accesibilidad y nombre accesible. `page.get_by_role("button", name="Guardar")`, `get_by_role("heading", name="Mis tareas")`. **Es el preferido**: es como un usuario y un lector de pantalla ven la página, y resiste cambios de estilo/estructura.
- **`get_by_label`**: un input por su `<label>`. `page.get_by_label("Email")`. Ideal para formularios.
- **`get_by_text`**: por el texto visible. `page.get_by_text("Bienvenido")`.
- **`get_by_placeholder`**, **`get_by_alt_text`**, **`get_by_title`**: para placeholders, imágenes, etc.
- **`get_by_test_id`**: por un atributo dedicado (`data-testid`). El *escape hatch* cuando nada de lo anterior identifica bien el elemento; es **estable** (no cambia con el contenido) pero requiere que el equipo de front ponga los `data-testid`.

Y los que **evitás** salvo último recurso: **selectores CSS y XPath** crudos (`page.locator(".btn-primary > div:nth-child(2)")`). Son **frágiles**: atados a la estructura del DOM y a clases de estilo que cambian con cualquier refactor de CSS, y **no expresan intención** (`.btn-primary` no dice "el botón de guardar"). Un test lleno de selectores CSS se rompe cada vez que el front toca el markup, aunque la funcionalidad no haya cambiado —flakiness de mantenimiento—.

Por qué los user-facing ganan: prueban la app **como la usa la gente**, sobreviven a refactors internos, y como premio **te empujan hacia una UI accesible** (si no podés ubicar un botón por su rol/nombre, probablemente tampoco lo pueda un lector de pantalla —el test detecta un problema de accesibilidad real—). La frase mental: **preferí locators user-facing (`get_by_role` primero, después label/text), usá `get_by_test_id` como escape hatch estable, y huí de CSS/XPath crudos —porque los selectores atados al DOM se rompen con cada refactor de markup, mientras los basados en rol/texto prueban la app como la ve el usuario y resisten el cambio—.**

**Ejercicios 4**
4.1 ¿Por qué la elección de locators es lo que más impacta en la robustez de una suite E2E?
4.2 Ordená de preferido a último recurso: `get_by_test_id`, CSS/XPath, `get_by_role`, `get_by_label`. ¿Por qué `get_by_role` es el preferido?
4.3 ¿Por qué los selectores CSS/XPath crudos son frágiles? Dá un ejemplo de qué los rompe sin que cambie la funcionalidad.
4.4 ¿Cómo es que preferir locators user-facing además mejora la accesibilidad de la app?

---

## Módulo 5 — Auto-waiting y web-first assertions: el fin de los `sleep()`

**Teoría.** El pecado capital del E2E viejo (Selenium) era el **`sleep()`**: "esperá 2 segundos a que cargue y después hacé clic". Es horrible por los dos lados: si ponés poco, el test falla porque el elemento todavía no estaba (flaky); si ponés mucho, la suite se vuelve lentísima. Playwright **elimina** los sleeps con dos mecanismos.

**1. Auto-waiting (espera automática) en las acciones.** Antes de actuar sobre un elemento (`.click()`, `.fill()`), Playwright corre **actionability checks**: espera a que el elemento esté **adjunto al DOM, visible, estable (no animándose), habilitado y sin nada encima**. Recién cuando está accionable, actúa. No esperás "2 segundos"; esperás **a la condición correcta**, el tiempo que haga falta (hasta un timeout). Esto solo elimina la mayoría de los sleeps.

**2. Web-first assertions (`expect`) que reintentan.** `expect(locator).to_be_visible()` no evalúa una vez y falla: **reintenta** hasta que la condición se cumpla o venza el timeout. Las hay para todo: `to_have_text(...)`, `to_have_count(n)`, `to_be_enabled()`, `to_have_value(...)`, `to_have_url(...)`. La diferencia con un `assert` de pytest es exactamente esa: el `assert` es una foto (evalúa el estado **ahora**); `expect` es una espera (aguanta a que el estado **llegue** a lo esperado). El **timeout** de esa espera es configurable —global (vía `expect.set_options(timeout=...)` o la config de pytest) y por aserción (`expect(locator).to_have_text(..., timeout=10_000)`)—; subilo para una operación legítimamente lenta, nunca para tapar una condición mal elegida.

```python
# ❌ Frágil: foto inmediata + sleep arbitrario
import time                                       # (solo para mostrar el anti-patrón; no lo uses)
page.get_by_role("button", name="Cargar").click()
time.sleep(2)                                   # ¿2s alcanzan? ¿sobran? nadie sabe
assert page.get_by_role("listitem").count() == 3  # evalúa una vez; si tardó 2.1s, falla

# ✅ Robusto: la aserción espera a la condición
page.get_by_role("button", name="Cargar").click()  # auto-wait a que el botón sea accionable
expect(page.get_by_role("listitem")).to_have_count(3)  # reintenta hasta que haya 3 (o timeout)
```

La regla de hierro: **nunca un `sleep()` en un test de Playwright.** Si sentís que necesitás uno, es que estás esperando lo que no debés (un tiempo) en vez de lo que sí (una condición observable): un texto que aparece, una URL que cambia, un request que termina. Casi siempre hay una web-first assertion que expresa esa condición. La frase mental: **Playwright mata los `sleep()` con auto-waiting (espera a que el elemento sea accionable antes de actuar) y web-first assertions (`expect` reintenta hasta que la condición se cumpla); esperás a la *condición correcta*, no a un tiempo arbitrario —y si te dan ganas de poner un sleep, te falta identificar qué condición observable estás esperando—.**

**Ejercicios 5**
5.1 ¿Por qué `sleep()` es malo por los dos lados (poco y mucho)?
5.2 ¿Qué chequea el auto-waiting antes de actuar sobre un elemento? (los actionability checks)
5.3 ¿Cuál es la diferencia entre un `assert` de pytest y un `expect` de Playwright? (foto vs espera)
5.4 "Si te dan ganas de poner un sleep, te falta algo." ¿Qué te falta identificar, y con qué lo reemplazás?

---

## Módulo 6 — Flakiness: causas y curas

**Teoría.** Un test **flaky** es el que **falla a veces sin que nada se haya roto** —pasa, lo corrés de nuevo, pasa, vuelve a fallar—. Es el cáncer del E2E: erosiona la confianza (módulo 1) hasta que el equipo ignora los rojos. Conocer las causas y sus curas es la habilidad central de un QA de automatización.

Las causas y qué las cura:

- **Timing / esperas mal hechas** → la cura es el módulo 5: **nunca `sleep()`**, siempre auto-waiting y web-first assertions. Es, lejos, la causa #1 de flakiness, y Playwright la resuelve si no te peleás con él.
- **Falta de aislamiento entre tests** → un test deja estado (un usuario creado, un registro en la DB) que hace fallar a otro, o el resultado depende del **orden** de ejecución. Cura: cada test **arranca de un estado conocido** y **no depende de otros**. Los browser contexts aislados (módulo 2) te dan el aislamiento del navegador; el aislamiento de **datos** lo ponés vos (siguiente punto).
- **Datos de prueba no deterministas** → tests que dependen de data preexistente que cambia, o que chocan entre sí (dos tests creando el "mismo" usuario). Cura: **datos deterministas y únicos** por test (un email con timestamp/uuid), seteo y limpieza vía API o fixtures (módulo 8), no a través de la UI.
- **Red / servicios externos** → un test que pega a un servicio real lento o caído falla por algo ajeno. Cura: **mockear la red** cuando lo que probás no es esa integración (módulo 9).
- **Animaciones, focus, condiciones de carrera del front** → auto-waiting cubre casi todo; el resto, esperar la condición observable correcta.

Y el punto de criterio más importante: **el retry automático es un olor, no una cura.** Playwright (y pytest) te dejan reintentar tests fallidos, y en CI un par de retries amortiguan flakiness residual —pero si tu estrategia para la flakiness es "subí los retries", estás **escondiendo** el problema, no resolviéndolo—. Un test que necesita 3 intentos para pasar no te da confianza; te da una falsa sensación de verde. El retry es una red de seguridad para lo inevitable, no la solución. La frase mental: **un test flaky falla sin que nada se rompa y destruye la confianza; las curas son esperar condiciones (no `sleep`), aislar cada test (estado conocido, sin depender del orden), datos deterministas y únicos, y mockear lo externo irrelevante —y el retry automático esconde la flakiness, no la cura—.**

**Ejercicios 6**
6.1 Definí "test flaky" y por qué es tan dañino para un equipo.
6.2 Nombrá tres causas de flakiness y la cura de cada una.
6.3 ¿Por qué los datos de prueba deben ser deterministas y únicos por test? Dá un ejemplo de cómo lo lográs.
6.4 ¿Por qué subir los retries automáticos es un olor y no una solución?

---

## Módulo 7 — Page Object Model: estructurar para mantener (y cuándo es over-engineering)

**Teoría.** A medida que la suite crece, los tests con locators y pasos repetidos por todos lados se vuelven imposibles de mantener: si cambia el botón "Guardar", lo tenés que arreglar en 40 tests. El patrón clásico para esto es el **Page Object Model (POM)**: encapsulás cada página (o componente) en una clase que expone sus **locators** y **acciones** de alto nivel, y los tests usan esa clase en vez de tocar la página directo.

```python
from playwright.sync_api import Page, expect

class TaskPage:
    def __init__(self, page: Page):
        self.page = page
        self.boton_nueva = page.get_by_role("button", name="Nueva tarea")
        self.input_titulo = page.get_by_label("Título")
        self.boton_guardar = page.get_by_role("button", name="Guardar")

    def ir(self):
        self.page.goto("http://localhost:5173/tareas")

    def crear_tarea(self, titulo: str):          # acción de alto nivel, en el lenguaje del usuario
        self.boton_nueva.click()
        self.input_titulo.fill(titulo)
        self.boton_guardar.click()

# El test queda legible y desacoplado del markup:
def test_crear_tarea(page):
    tareas = TaskPage(page)
    tareas.ir()
    tareas.crear_tarea("Comprar pan")
    expect(page.get_by_text("Comprar pan")).to_be_visible()
```

Las ventajas: **un solo lugar que tocar** cuando cambia la UI (DRY: el locator del botón vive en la clase, no en 40 tests), tests que se leen en el **lenguaje del dominio** ("crear tarea") y no en clics sueltos, y reuso entre tests.

Pero —y acá está el criterio que separa al senior— **el POM se sobre-aplica con facilidad**. Señales de over-engineering: clases POM para páginas con un solo test, jerarquías profundas de herencia entre page objects, o métodos que envuelven una sola línea sin agregar significado. La alternativa moderna y más liviana en pytest son las **fixtures** (módulo 8 y [Python](python.md)): una fixture que te arma el `TaskPage` ya logueado y en la página, sin ceremonia. La regla: **introducí POM (o fixtures) cuando la repetición empieza a doler, no de entrada por dogma.** Para una suite chica, tests directos y legibles son mejores que una arquitectura de page objects que nadie pidió. La frase mental: **el Page Object Model encapsula locators y acciones de cada página en una clase, así un cambio de UI se arregla en un solo lugar y los tests hablan el lenguaje del dominio; pero se sobre-aplica fácil —usalo (o usá fixtures de pytest) cuando la repetición duele, no como ceremonia desde el primer test—.**

**Ejercicios 7**
7.1 ¿Qué problema de mantenimiento resuelve el Page Object Model y cómo (qué encapsula)?
7.2 Nombrá dos ventajas concretas del POM más allá de "es más prolijo".
7.3 Dá dos señales de que estás sobre-aplicando el POM.
7.4 ¿Cuál es la regla para decidir cuándo introducir POM o fixtures? Conectá con el criterio del temario.

---

## Módulo 8 — Autenticación y estado: `storage_state`, no loguear por UI cada vez

**Teoría.** Casi toda app real tiene **login**, y casi todo test corre sobre páginas que requieren estar autenticado. El error de principiante: **hacer login a través de la UI al principio de cada test** (ir a `/login`, escribir email y password, clic en entrar). Si tenés 50 tests, son 50 logins por la UI —lentísimo, y además le agregás a cada test un punto de falla (el flujo de login) que no es lo que ese test quería probar—.

La solución de Playwright: **`storage_state`** —guardar el estado de sesión (cookies y `localStorage`) **una vez**, y **reusarlo** en todos los tests—. Te logueás una sola vez (idealmente por API, aún más rápido que por UI), serializás el estado a un archivo, y cada test arranca un contexto **ya autenticado** cargando ese archivo.

```python
# 1) Una vez (un script o una fixture de sesión): logueás y guardás el estado
context = browser.new_context()
page = context.new_page()
page.goto("http://localhost:5173/login")
page.get_by_label("Email").fill("qa@test.com")
page.get_by_label("Password").fill("secreto")
page.get_by_role("button", name="Entrar").click()
expect(page).to_have_url("**/dashboard")        # esperá a estar logueado (glob; o re.compile(r"/dashboard"))
context.storage_state(path="auth.json")          # guardás cookies + localStorage

# 2) En los tests: arrancás un contexto YA autenticado, sin pasar por el login
#    En pytest-playwright se setea vía la fixture browser_context_args:
import pytest

@pytest.fixture(scope="session")
def browser_context_args(browser_context_args):
    return {**browser_context_args, "storage_state": "auth.json"}
```

> **Ojo con la caducidad.** El `storage_state` guarda cookies/JWT que **expiran**: si el token tiene vida corta, regenerá el estado al inicio de cada corrida (una fixture `scope="session"` que rehace el login si el archivo no existe o está vencido). Mejor todavía: logueate **por API** en esa fixture (un POST al endpoint de login) y serializá el estado —más rápido y sin depender del flujo de UI—.

Las ventajas: **velocidad** (no repetís el login 50 veces), y **foco** (cada test prueba lo suyo, no el login una y otra vez). El login **sí** se prueba —pero en **un** test dedicado a ese flujo, no como prólogo de todos—. Variante para roles: guardás varios `storage_state` (admin, usuario común) y cada test usa el que necesita. Esto conecta directo con el aislamiento del módulo 6: además del estado de sesión, asegurate de que los **datos** de cada test estén en un estado conocido (seteados/limpiados por API o fixtures, no por UI). La frase mental: **no te loguees por la UI en cada test —logueate una vez (mejor por API), guardá cookies+localStorage con `storage_state`, y arrancá cada test en un contexto ya autenticado—; es más rápido y mantiene cada test enfocado en lo suyo, dejando el login probado en su propio test dedicado—.**

**Ejercicios 8**
8.1 ¿Por qué hacer login por la UI al inicio de cada test es un mal patrón? Nombrá los dos problemas.
8.2 ¿Qué guarda `storage_state` y cómo se reusa en los tests?
8.3 Si no logueás por UI en cada test, ¿dónde se prueba el login?
8.4 ¿Cómo manejarías tests que necesitan distintos roles (admin vs usuario común)?

---

## Módulo 9 — Interceptar y mockear la red: `route`/`fulfill`

**Teoría.** Playwright puede **interceptar las requests de red** que hace el front y responder lo que vos quieras, con `page.route(patrón, handler)`. El handler decide: dejar pasar la request real (`route.continue_()`), responder un mock (`route.fulfill(...)`), o abortarla (`route.abort()`). Es una herramienta poderosa con un trade-off importante de criterio.

```python
def test_lista_vacia_muestra_mensaje(page):
    # interceptá la API de tareas y respondé una lista vacía
    page.route("**/api/tasks", lambda route: route.fulfill(
        status=200,
        json={"tasks": []},
    ))
    page.goto("http://localhost:5173/tareas")
    expect(page.get_by_text("No tenés tareas todavía")).to_be_visible()
```

Para qué sirve mockear la red en un E2E:

- **Aislar el front del back**: probás que la UI reacciona bien a una respuesta dada, sin depender de que el backend tenga esos datos.
- **Forzar estados difíciles de reproducir**: una lista vacía, un error `500`, un `403`, una respuesta lentísima (para probar el spinner), un caso borde que en el back real es difícil de armar. Esto es oro: probar el **estado de error** de la UI sin tener que romper el backend.
- **Estabilidad**: sacás de la ecuación servicios externos lentos o flaky (módulo 6).

El trade-off, y acá está el criterio: **mockear la red en un E2E te quita justamente lo que hace valioso al E2E —la integración real—.** Si mockeás todas las APIs, ya no estás probando "el sistema entero de punta a punta"; estás probando el front contra un backend de mentira (algo más cercano a un test de integración de front). La regla: **mockeá lo que NO es el objeto del test** —servicios de terceros, o para forzar un estado de error puntual— y dejá **real** el flujo que el E2E existe para validar. Un E2E del "happy path" de crear una tarea debería pegarle al backend de verdad; un E2E que prueba "qué muestra la UI cuando la API devuelve 500" obviamente mockea ese 500. La frase mental: **`page.route` intercepta requests y te deja responder mocks (`fulfill`), dejar pasar (`continue_`) o abortar —ideal para forzar estados de error/borde y aislar terceros—; pero mockear de más le saca al E2E su razón de ser (la integración real), así que mockeá lo que no es el objeto del test y dejá real el flujo que vinís a validar—.**

**Ejercicios 9**
9.1 ¿Qué tres cosas puede hacer un handler de `page.route` con una request interceptada?
9.2 Nombrá dos usos legítimos de mockear la red en un E2E.
9.3 ¿Cuál es el trade-off de mockear la red en un E2E? ¿Qué se pierde?
9.4 Dá un caso donde mockear está bien y uno donde mockear traicionaría el propósito del E2E.

---

## Módulo 10 — Debugging y CI: trace viewer, codegen, sharding y contenedores

**Teoría.** Dos cosas separan una suite E2E que se mantiene de una que se abandona: **poder debuggear un fallo rápido** y **correr confiable en CI**. Playwright es fuerte en ambas.

**Debugging y herramientas:**
- **Trace viewer**: el arma principal. Playwright graba un **trace** (línea de tiempo con snapshots del DOM, cada acción, requests de red, consola) que abrís y **recorrés paso a paso** —ves exactamente qué había en pantalla cuando falló—. En CI lo configurás para guardarlo **solo en fallo** (`--tracing=retain-on-failure`) y lo inspeccionás con `playwright show-trace trace.zip`. Para un fallo en CI que no reproducís local, el trace es lo que te salva.
- **Codegen**: `playwright codegen <url>` abre el navegador, **grabás** tus clics y te **genera el código** del test (con buenos locators). Excelente para arrancar un test o descubrir el locator correcto —no para dejar el código generado tal cual, que conviene limpiar—.
- **Modo headed y debug**: `pytest --headed` (ver el navegador), `--slowmo` (cámara lenta), y `PWDEBUG=1` abre el **inspector** para ir paso a paso. Y los mismos artefactos guardados solo en fallo con `--video=retain-on-failure` y `--screenshot=only-on-failure`, junto al `--tracing=retain-on-failure`.

**Correr en CI** (conecta con [Docker y CI/CD](docker-deploy.md)):
- **Imagen oficial**: Playwright provee una imagen Docker (`mcr.microsoft.com/playwright/python`) con los navegadores y dependencias del SO ya instalados —evitás pelearte con libs faltantes en el runner—. Si no, `playwright install --with-deps` baja navegadores + deps del sistema.
- **Headless por defecto**: en CI corre sin pantalla.
- **Paralelización y sharding**: corrés tests en paralelo dentro de una máquina con `pytest-xdist` (`pytest -n auto`), y repartís la suite entre **varias** máquinas con **sharding** (`--shard=1/3`) para bajar el tiempo total. Ojo: el `--shard` no es magia de Playwright —lo combinás con una **matriz de CI** que levanta N jobs, cada uno corriendo su shard—. Los browser contexts aislados (módulo 2) hacen que el paralelismo sea seguro.
- **Artefactos en fallo**: configurás CI para subir trace, screenshots y video de los tests que fallaron —así debuggeás un rojo de CI sin reproducirlo localmente—.

La frase mental: **el trace viewer (repetición paso a paso con DOM, red y consola, guardado en fallo) es tu herramienta #1 de debugging, y codegen te arranca tests; en CI corrés headless con la imagen oficial, en paralelo (xdist) y con sharding entre máquinas, subiendo trace/video como artefactos para diagnosticar los rojos sin reproducirlos a mano—.**

**Ejercicios 10**
10.1 ¿Qué es el trace viewer y por qué es la herramienta clave para un fallo de CI que no reproducís local? ¿Cómo lo configurás para no inflar cada corrida?
10.2 ¿Para qué sirve codegen y cuál es su límite (qué no hay que hacer con su salida)?
10.3 ¿Qué da la imagen Docker oficial de Playwright que te ahorra problemas en CI?
10.4 ¿Cómo bajás el tiempo total de la suite en CI, y por qué es seguro paralelizar en Playwright?

---

## Módulo 11 — Testear features de IA y el criterio de cierre

**Teoría.** El cierre, con el ángulo que define el perfil **AI QA**: **¿cómo testeás una feature impulsada por un LLM** —un chatbot, un resumen, una respuesta de RAG ([Vector Databases](vector-dbs.md))— **si su salida es no determinista?** Un E2E clásico afirma texto exacto (`to_have_text("Bienvenido")`); pero un LLM **no devuelve el mismo texto dos veces**, así que `to_have_text("La respuesta es 42")` va a ser flaky por diseño. Esto rompe el modelo de aserción exacta, y manejarlo es lo que distingue a un AI QA.

Las estrategias, de la más simple a la más rica:

1. **Mockear el LLM para los tests de UI.** Lo más importante: **separá dos preguntas distintas**. "¿La UI está bien cableada?" (¿el mensaje aparece, se muestra el spinner, se renderiza el markdown, se maneja el error?) se prueba **mockeando la respuesta del LLM** (módulo 9: `route`/`fulfill` con una respuesta fija) → test **determinista y rápido**. "¿La respuesta del LLM es **buena**?" es **otra cosa** y no se prueba con un E2E: se prueba con **[evals](evals.md)** (golden sets, LLM-as-judge, las online evals de [Deploy de IA](deploy-ai.md)). No mezcles las dos.
2. **Aserciones sobre estructura/contrato, no texto exacto.** Cuando sí corrés contra el LLM real, afirmá **propiedades** que se cumplen siempre: que llegó una respuesta no vacía, que tiene la **forma** esperada (un JSON con tales campos), que **contiene** entidades clave (`to_contain_text("42")` en vez del texto completo), que no disparó un guardarraíl.
3. **Aserciones semánticas / LLM-as-judge.** Para validar el *sentido* de una respuesta no determinista, usás otro LLM como juez ("¿esta respuesta contesta la pregunta, sí/no?") —la misma técnica de [evals](evals.md)—, sabiendo que el juez también se equivoca y hay que calibrarlo.

```python
# Estrategia 1: la UI cableada, con el LLM mockeado → determinista y rápido.
def test_chat_renderiza_la_respuesta(page):
    page.route("**/api/chat", lambda route: route.fulfill(
        status=200, json={"answer": "La capital de Francia es París."},
    ))
    page.goto("http://localhost:5173/chat")
    page.get_by_label("Mensaje").fill("¿Capital de Francia?")
    page.get_by_role("button", name="Enviar").click()
    expect(page.get_by_test_id("chat-respuesta")).to_be_visible()   # plomería, no calidad

# Estrategia 2: contra el LLM real, afirmá estructura/contrato, nunca texto exacto.
def test_respuesta_real_tiene_forma(page):
    page.goto("http://localhost:5173/chat")
    page.get_by_label("Mensaje").fill("¿Cuánto es 40 + 2?")
    page.get_by_role("button", name="Enviar").click()
    respuesta = page.get_by_test_id("chat-respuesta")
    expect(respuesta).not_to_be_empty()              # llegó algo
    expect(respuesta).to_contain_text("42")          # contiene la entidad clave, no el texto completo
```

La regla de oro del AI QA: **separá lo determinista de lo no determinista.** El E2E (mockeando el LLM) prueba que el producto **funciona** —la plomería—; las evals prueban que la IA **es buena** —la calidad—. Confundirlas te lleva a E2E flaky que reintentás para siempre, o a creer que "pasó el E2E" significa "la IA responde bien" (no lo significa).

**Y el criterio de cierre del módulo**, el mismo de todo el temario: **poco E2E y del bueno.** Pocos flujos críticos de punta a punta (módulo 1), con locators user-facing (módulo 4), sin `sleep` (módulo 5), aislados y deterministas (módulo 6), estructurados cuando duele (módulo 7), autenticados por estado (módulo 8), mockeando solo lo que no es el objeto del test (módulo 9), debuggeables por trace y rápidos en CI (módulo 10). Una suite así da **confianza**; una suite enorme y flaky da **ruido**. La frase mental: **para features de IA, separá lo determinista (¿la UI funciona? → mockeá el LLM, E2E estable) de lo no determinista (¿la IA es buena? → evals, no E2E), y cuando corras contra el LLM real afirmá estructura/contrato o usá un juez, nunca texto exacto; y el criterio que cierra todo es poco E2E y del bueno —confianza, no ruido—.**

**Ejercicios 11**
11.1 ¿Por qué `to_have_text("...")` sobre la salida de un LLM es flaky por diseño?
11.2 ¿Cuáles son las dos preguntas distintas que hay que separar al testear una feature de IA, y con qué se prueba cada una?
11.3 Cuando corrés contra el LLM real, ¿qué tipo de aserciones usás en vez de texto exacto? Dá dos.
11.4 Enunciá la regla de oro del AI QA y el criterio de cierre del E2E ("poco y del bueno").

---

## Proyecto integrador (capstone): una suite E2E confiable para la Task App

El ejercicio que cierra el módulo y que mostrás en una entrevista de QA/SDET. No te damos la solución; te damos los **criterios de aceptación**. Construilo con Playwright + pytest sobre una app con login y un CRUD (la Task App del roadmap, o cualquier app tuya).

**Qué construir.** Una suite E2E de **pocos flujos críticos**, no de todo:

1. **Setup de auth con `storage_state`** (módulo 8): un login (por API o UI) una vez, reusado por los demás tests.
2. **El happy path crítico de punta a punta, contra el backend real**: loguearse → crear una tarea → verla en la lista → completarla → verla completada. Con **locators user-facing** (módulo 4) y **web-first assertions** (módulo 5), **sin un solo `sleep`**.
3. **Un test de estado de error mockeando la red** (módulo 9): la API de tareas devuelve `500` → la UI muestra el mensaje de error correcto.
4. **Datos deterministas y aislamiento** (módulo 6): cada test arranca de un estado conocido y no depende del orden ni de otros tests.

**Criterios de aceptación.**
- [ ] Ningún `sleep()` / espera por tiempo: todo es auto-waiting + `expect`.
- [ ] Los locators son user-facing (`get_by_role`/`get_by_label`/`get_by_text`); cero CSS/XPath crudos.
- [ ] El login se hace una vez (vía `storage_state`), no en cada test.
- [ ] La suite **pasa 10 veces seguidas** sin un solo fallo intermitente (prueba de no-flakiness).
- [ ] Corre en **CI** (headless), en paralelo, y sube el **trace** de los tests que fallan.
- [ ] Hay al menos un test que mockea la red para forzar un estado de error.

**Extensiones (suben el nivel).**
- Agregá **Page Objects** (o fixtures) cuando la repetición empiece a doler (módulo 7) —no antes—.
- **Cross-browser**: corré la suite en Chromium, Firefox y WebKit y resolvé las diferencias.
- **Visual regression**: agregá `expect(page).to_have_screenshot()` a una pantalla estable y manejá los baselines (ojo: las imágenes base dependen del SO/CI y son una trampa de flakiness de píxeles → corré las baselines en el **mismo entorno** que CI, actualizalas con `--update-snapshots` de forma controlada, y tolerá ruido con `max_diff_pixels`/máscaras de zonas dinámicas).
- **Feature de IA** (módulo 11): testeá una pantalla con un chatbot/resumen **mockeando el LLM** para la plomería, y dejá la calidad de la respuesta para una eval aparte ([evals](evals.md)).

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 Porque es el más realista pero el más caro: es lento (arranca navegador, navega, espera renders),
    caro de correr y mantener, y frágil (un cambio de UI o un timing lo rompe). Por eso pocos.
1.2 Probar los pocos FLUJOS DE USUARIO CRÍTICOS de punta a punta (¿el usuario logra lo que vino a
    hacer?), y dejar los casos borde, validaciones y ramas raras para tests más abajo (unit/integración),
    que son más rápidos y precisos.
1.3 Que nadie confía en los tests: cuando "siempre falla algo", el reflejo es darle retry y mergear, y
    los tests dejan de aportar señal (son ruido caro).
1.4 Porque no le importa cómo está hecho el código por dentro, solo lo que el usuario ve y hace. Fuerza:
    prueba el sistema entero (front+back+DB+integración real). Debilidad: cuando falla te dice QUE algo
    anda mal, no DÓNDE.
```

### Módulo 2
```
2.1 El auto-waiting integrado: Playwright espera solo a que los elementos estén listos antes de actuar,
    mientras Selenium te obligaba a poner waits manuales (la fuente histórica de la flakiness).
2.2 Cypress corre DENTRO del navegador (limita multi-tab/multi-origen, históricamente solo Chromium) y
    es JS/TS only. Playwright corre POR FUERA controlando el browser, soporta varios lenguajes (Python,
    TS, Java, .NET) y maneja escenarios complejos (varias pestañas/orígenes, descargas).
2.3 Una sesión de navegador aislada (como incógnito fresco, con sus cookies/storage propios), liviana de
    crear. Habilita aislamiento barato entre tests y paralelización real.
2.4 Browser (el navegador) → BrowserContext (sesión aislada) → Page (una pestaña). En cada test trabajás
    con la Page (el fixture `page`, en su propio contexto).
```

### Módulo 3
```
3.1 Una pestaña (Page) lista para usar, en un browser context aislado, una por test → cada test arranca
    con cookies/storage limpios, sin contaminarse con otros.
3.2 Arrange: page.goto(url) (ir a la app). Act: interacciones como get_by_role(...).click() /
    .fill(...). Assert: expect(locator).to_be_visible() / to_have_text(...).
3.3 Espera a que la página cargue antes de seguir, así que no necesitás un wait manual después de navegar.
3.4 El assert de pytest evalúa el estado UNA vez (una foto); expect de Playwright REINTENTA hasta que la
    condición se cumpla o venza el timeout (espera).
```

### Módulo 4
```
4.1 Porque los locators son el acoplamiento entre el test y la página: locators frágiles (atados al DOM)
    se rompen con cada refactor de markup, y mantener eso en una suite grande es un infierno; locators
    robustos sobreviven a los cambios.
4.2 Preferido → último recurso: get_by_role, get_by_label, get_by_test_id, CSS/XPath. get_by_role es el
    preferido porque identifica el elemento como lo ven un usuario y un lector de pantalla (por rol +
    nombre accesible), resistiendo cambios de estilo/estructura.
4.3 Porque están atados a la estructura del DOM y a clases de estilo que cambian con cualquier refactor
    de CSS/markup, y no expresan intención. Ej.: renombrar .btn-primary o mover un div rompe el test
    aunque el botón siga funcionando igual.
4.4 Porque si no podés ubicar un elemento por su rol/nombre accesible, probablemente un lector de
    pantalla tampoco pueda: el test que insiste en locators user-facing termina detectando (y empujando
    a arreglar) problemas reales de accesibilidad.
```

### Módulo 5
```
5.1 Si ponés poco tiempo, el elemento todavía no está listo y el test falla (flaky); si ponés mucho, la
    suite se vuelve lentísima. Nunca sabés el número correcto porque depende de la máquina/red.
5.2 Los actionability checks: que el elemento esté adjunto al DOM, visible, estable (sin animarse),
    habilitado y sin nada encima. Recién cuando es accionable, actúa.
5.3 El assert de pytest evalúa el estado actual una sola vez (foto); el expect de Playwright reintenta
    hasta que la condición se cumpla o venza el timeout (espera a que el estado llegue).
5.4 Te falta identificar la CONDICIÓN OBSERVABLE que estás esperando (un texto que aparece, una URL que
    cambia, un conteo de elementos), y la reemplazás por la web-first assertion que la expresa.
```

### Módulo 6
```
6.1 Un test flaky falla a veces sin que nada se haya roto (pasa, reintentás, vuelve a fallar). Es dañino
    porque erosiona la confianza: el equipo empieza a ignorar los rojos y los tests dejan de dar señal.
6.2 (Tres de) timing/esperas → auto-waiting + web-first assertions, nunca sleep; falta de aislamiento →
    cada test arranca de estado conocido y no depende del orden; datos no deterministas → datos únicos
    por test, seteo/limpieza por API/fixtures; red/servicios externos → mockear lo irrelevante.
6.3 Para que los tests no choquen entre sí ni dependan de data preexistente que cambia. Ej.: generar un
    email único por test con un uuid/timestamp (qa+<uuid>@test.com) en vez de un email fijo compartido.
6.4 Porque esconde el problema en vez de resolverlo: un test que necesita 3 intentos para pasar no da
    confianza, da un falso verde. El retry es una red de seguridad para lo inevitable, no la cura de la
    flakiness.
```

### Módulo 7
```
7.1 La repetición de locators y pasos en muchos tests (si cambia un botón, lo arreglás en N tests). El
    POM encapsula los locators y las acciones de alto nivel de cada página en una clase, y los tests
    usan esa clase.
7.2 (Dos de) un solo lugar que tocar cuando cambia la UI (DRY); tests legibles en el lenguaje del
    dominio ("crear_tarea") en vez de clics sueltos; reuso entre tests.
7.3 (Dos de) clases POM para páginas con un solo test; jerarquías profundas de herencia entre page
    objects; métodos que envuelven una sola línea sin agregar significado.
7.4 Introducir POM (o fixtures de pytest) cuando la repetición empieza a doler, no por dogma desde el
    primer test. Es el "la solución más simple que resuelve el problema" del temario: no metas
    arquitectura que nadie pidió.
```

### Módulo 8
```
8.1 (Dos problemas) Es lentísimo (repetís el login por UI en cada uno de los N tests), y le agrega a
    cada test un punto de falla ajeno (el flujo de login) que no es lo que ese test quería probar.
8.2 Guarda el estado de sesión: cookies y localStorage. Se reusa cargándolo al crear el contexto
    (storage_state="auth.json"), así cada test arranca ya autenticado sin pasar por el login.
8.3 En UN test dedicado al flujo de login, no como prólogo de todos los demás.
8.4 Guardando varios storage_state (uno por rol: admin, usuario común) y haciendo que cada test cargue
    el que necesita.
```

### Módulo 9
```
9.1 Dejar pasar la request real (route.continue_()), responder un mock (route.fulfill(...)), o abortarla
    (route.abort()).
9.2 (Dos de) aislar el front del back (probar que la UI reacciona a una respuesta dada sin depender de
    los datos reales); forzar estados difíciles de reproducir (lista vacía, 500, 403, respuesta lenta);
    sacar de la ecuación servicios externos lentos/flaky.
9.3 Que perdés la integración real, que es lo que hace valioso al E2E: si mockeás todas las APIs, ya no
    probás "el sistema entero de punta a punta", sino el front contra un backend de mentira.
9.4 Bien: probar qué muestra la UI cuando la API devuelve 500 (mockeás ese 500). Traiciona el propósito:
    mockear la API de tareas en el test del happy path de "crear una tarea" —ahí justamente querés la
    integración real con el backend—.
```

### Módulo 10
```
10.1 Una repetición visual del test paso a paso (snapshots del DOM, acciones, red, consola) que recorrés
     para ver qué había en pantalla al fallar. Es clave para un rojo de CI que no reproducís local. Lo
     configurás con retain-on-failure (solo guarda el trace cuando el test falla) para no inflar cada corrida.
10.2 codegen abre el navegador, grabás tus clics y genera el test con buenos locators. Límite: no dejar
     el código generado tal cual —conviene limpiarlo/refactorizarlo—; sirve para arrancar o descubrir el
     locator correcto.
10.3 Los navegadores y las dependencias del SO ya instalados (mcr.microsoft.com/playwright/python), así
     no te peleás con librerías faltantes en el runner de CI.
10.4 Corriendo en paralelo (pytest-xdist, -n auto) y repartiendo la suite entre máquinas con sharding
     (--shard=1/3). Es seguro porque cada test corre en un browser context aislado, sin compartir estado.
```

### Módulo 11
```
11.1 Porque un LLM no devuelve el mismo texto dos veces (es no determinista): afirmar texto exacto va a
     fallar de forma intermitente aunque la feature funcione bien.
11.2 (1) ¿La UI está bien cableada? (aparece el mensaje, el spinner, se renderiza, se maneja el error) →
     se prueba con E2E MOCKEANDO el LLM (determinista). (2) ¿La respuesta del LLM es buena? → se prueba
     con evals (golden sets, LLM-as-judge, online evals), NO con E2E.
11.3 (Dos de) que la respuesta no esté vacía; que tenga la forma/estructura esperada (JSON con tales
     campos); que CONTENGA entidades clave (to_contain_text en vez del texto completo); que no haya
     disparado un guardarraíl; aserciones semánticas con un LLM-as-judge.
11.4 Regla de oro del AI QA: separar lo determinista (¿la UI funciona? → mock + E2E estable) de lo no
     determinista (¿la IA es buena? → evals). Criterio de cierre: poco E2E y del bueno —pocos flujos
     críticos, locators user-facing, sin sleep, aislados/deterministas, mockeando solo lo que no es el
     objeto del test, debuggeables y rápidos en CI— que dé confianza en vez de ruido.
```

---

## Siguientes pasos

Con este módulo arrancás el track QA / Automation con criterio de producción: el E2E en su lugar (la cima de la pirámide, poco y crítico), por qué Playwright (auto-waiting, cross-browser, contexts aislados), cómo se escribe un test (el fixture `page`, AAA en el navegador), los locators user-facing que hacen la diferencia entre una suite robusta y un infierno, el auto-waiting y las web-first assertions que matan los `sleep`, las causas y curas de la flakiness, el Page Object Model (y cuándo es over-engineering), el manejo de auth con `storage_state`, el mockeo de red y su trade-off, el debugging con trace viewer y el CI con sharding, y —el ángulo AI QA— cómo testear features de IA separando lo determinista (E2E con LLM mockeado) de lo no determinista (evals). Con la regla de siempre: **poco E2E y del bueno —confianza, no ruido—.** Lo que sigue en el track QA es [API testing y contract testing](api-testing.md): probar las APIs por debajo de la UI —más rápido, más estable y más preciso que el E2E— y garantizar que servicios que se hablan entre sí no se rompan el contrato. Y todo se conecta con lo que ya sabés: la pirámide de [Testing](testing.md), el CI/CD de [Docker](docker-deploy.md), y las [evals](evals.md) para la parte de IA que el E2E no debe probar.
