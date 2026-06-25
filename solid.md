# SOLID: los cinco principios del diseño orientado a objetos

**Diseño de clases y módulos que no se pudren con el tiempo · ejemplos en TypeScript · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es la capa de **principios** que está debajo de toda la arquitectura del hub: cuando en [DDD](ddd.md), [NestJS senior](nestjs-senior.md) o [Patrones en NestJS](nestjs-patrones.md) ves "puertos y adaptadores" o "inyección de dependencias", estás viendo SOLID **aplicado**. Acá vamos a la raíz: los cinco principios que hacen que una clase o un módulo se pueda **cambiar sin miedo**. No es teoría de pizarrón — es lo que separa un código que aguanta seis meses de uno que se vuelve intocable.

**Lo que asumimos.** Programación orientada a objetos básica (clases, interfaces, herencia, polimorfismo), TypeScript (tipos, `interface`, `implements`), y haber sentido alguna vez el dolor de **tocar una clase y romper tres cosas que no tenían nada que ver**. Si ese dolor te suena, SOLID es para vos.

> **¿De dónde sale "SOLID"?** Los cinco principios los formuló **Robert C. Martin ("Uncle Bob")** alrededor del año 2000 (paper *"Design Principles and Design Patterns"*). El acrónimo **SOLID** lo acuñó **Michael Feathers** unos años después, reordenando las iniciales para que se pudiera pronunciar. La **L** (Liskov) viene de antes y de afuera: es de **Barbara Liskov** (keynote de 1987, formalizada con Jeannette Wing en 1994). No es trivia: saber que la L tiene madre propia te ayuda a entenderla distinto de las otras cuatro.

> ⚠️ **Una advertencia desde el arranque.** SOLID es una caja de herramientas, **no una religión.** Aplicado con criterio, te da código mantenible; aplicado por dogma, te da una sopa de interfaces y abstracciones que nadie entiende (over-engineering). El módulo 7 es entero sobre **cuándo NO** aplicarlo. Tené eso en la cabeza desde el principio: el objetivo es **manejar el cambio**, no coleccionar abstracciones.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere editor).

**Índice de módulos**
1. **S** — Single Responsibility: una sola razón para cambiar
2. **O** — Open/Closed: abierto a extensión, cerrado a modificación
3. **L** — Liskov: los subtipos tienen que poder reemplazar al padre
4. **I** — Interface Segregation: nadie depende de lo que no usa
5. **D** — Dependency Inversion: depender de abstracciones (y el puente con hexagonal)
6. SOLID en conjunto: cómo se refuerzan y qué problema atacan
7. El criterio: cuándo SOLID es over-engineering

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — S: Single Responsibility Principle (una sola razón para cambiar)

**Teoría.** El más famoso y el **más malentendido** de los cinco. Casi todo el mundo lo recita como *"una clase debe hacer una sola cosa"* — y está **mal**, o por lo menos incompleto. La definición original de Uncle Bob es más filosa:

> **Un módulo debe tener una, y solo una, razón para cambiar.**

Y más tarde la afiló todavía más: **un módulo debe ser responsable ante un solo *actor*** (un solo grupo de gente que pide cambios). La diferencia es enorme. "Hacer una cosa" es vago —¿qué es "una cosa"?—. "Una razón para cambiar" es operativo: **¿quién, en el negocio, podría pedirte que modifiques esta clase?** Si la respuesta es "el área de finanzas Y el área de infraestructura Y el de diseño", tu clase tiene tres razones para cambiar y viola la S.

Mirá el smell clásico:

```ts
// ❌ Tres responsabilidades, tres actores, tres razones para cambiar
class Reporte {
  calcularTotales(): number { /* lógica de negocio (la pide Finanzas) */ return 0 }
  formatearHTML(): string { /* presentación (la pide Diseño/UX) */ return '' }
  guardarEnDisco(path: string): void { /* persistencia (la pide Infra) */ }
}
```

Si Finanzas cambia una fórmula, Diseño cambia el HTML, e Infra migra de disco a S3, **los tres tocan la misma clase** — y cada cambio arriesga romper a los otros dos. Eso es **acoplamiento** de responsabilidades (acoplamiento = cuánto depende una cosa de otra; lo vemos a fondo en el módulo 6). La separás:

```ts
// ✅ Cada clase responde a un solo actor
class CalculadoraDeReporte {
  calcularTotales(): number { return 0 }       // cambia solo si cambia el negocio
}
class FormateadorHTML {
  formatear(datos: number): string { return '' } // cambia solo si cambia la presentación
}
class RepositorioDeReporte {
  guardar(contenido: string): void { }          // cambia solo si cambia la persistencia
}
```

La analogía: un empleado con **tres jefes distintos** que le dan órdenes contradictorias vive en el caos. Dale **un solo jefe** a cada clase. ¿Se entiende? Una razón para cambiar = un jefe.

⚠️ Ojo con el péndulo: esto **no** significa "una función por clase". Una clase puede tener muchos métodos mientras todos sirvan **al mismo actor**. Partir de más es el over-engineering del módulo 7.

La frase mental: **la S no es "hacé una sola cosa", es "tené una sola razón para cambiar". Preguntá quién, en el negocio, te haría tocar esta clase. Si son varios actores distintos, separá.**

**Ejercicios 1**
1.1 🔁 ¿Cuál es la definición correcta de la S y por qué "hace una sola cosa" se queda corta?
1.2 ✍️ Tenés una clase `Usuario` con `validarPassword()`, `guardarEnDB()` y `enviarEmailBienvenida()`. Decí cuántos actores/razones de cambio tiene y **refactorizala** separando cada responsabilidad en su propio colaborador; mostrá las firmas resultantes.
1.3 🧠 ¿Por qué "una clase = un método" es una mala interpretación de la S? Dá un ejemplo donde varios métodos en una clase SÍ respetan el principio.

---

## Módulo 2 — O: Open/Closed Principle (abierto a extensión, cerrado a modificación)

**Teoría.** El principio dice:

> **Las entidades de software deben estar abiertas a la extensión, pero cerradas a la modificación.**

⚠️ Nota de origen: el término OCP lo acuñó **Bertrand Meyer** (*Object-Oriented Software Construction*, 1988), con un enfoque basado en **herencia**; la versión polimórfica con interfaces que vemos acá es la reformulación que popularizó Robert C. Martin. (Igual que con la L, vale reconocer de dónde viene.)

Suena a contradicción zen, pero es concreto: deberías poder **agregar comportamiento nuevo sin tocar el código que ya funciona y está testeado.** ¿Por qué importa? Porque cada vez que modificás una clase que ya anda, **arriesgás romper lo que andaba** y tenés que re-testear todo. Si en cambio **extendés** (agregás algo nuevo), lo viejo ni se entera.

El smell que grita "violación de OCP" es el **`switch`/`if` que crece con cada caso nuevo**:

```ts
// ❌ Cada medio de pago nuevo te obliga a MODIFICAR esta función (y re-testearla entera)
function procesarPago(tipo: string, monto: number): void {
  if (tipo === 'tarjeta') { /* ... */ }
  else if (tipo === 'paypal') { /* ... */ }
  else if (tipo === 'cripto') { /* ... */ }   // ← y mañana 'transferencia', y...
}
```

La solución es **polimorfismo**: definís una abstracción y cada caso nuevo es una clase nueva que la implementa. El código que orquesta queda **cerrado** (no se toca); el sistema queda **abierto** (sumás clases).

```ts
// ✅ Abierto a extensión (nuevas clases), cerrado a modificación (esta lógica no se toca)
interface MedioDePago {
  procesar(monto: number): void
}

class Tarjeta implements MedioDePago {
  procesar(monto: number): void { /* ... */ }
}
class PayPal implements MedioDePago {
  procesar(monto: number): void { /* ... */ }
}
// Mañana agregás `class Cripto implements MedioDePago` y NO tocás nada de lo anterior.

function procesarPago(medio: MedioDePago, monto: number): void {
  medio.procesar(monto)   // no sabe ni le importa cuál es: cerrado al cambio
}
```

La analogía: la pared de tu casa tiene **zócalos (enchufes)**. Para sumar un aparato nuevo no rompés la pared ni recableás —**enchufás**—. El zócalo es la abstracción `MedioDePago`; cada aparato es una implementación.

⚠️ El matiz senior: OCP **no** significa "nunca modifiques código". Significa diseñar los **puntos de extensión** donde sabés que va a haber variación. No abstraigas todo "por las dudas" (módulo 7) — abstraé donde el cambio es **previsible y recurrente** (medios de pago, formatos de exportación, proveedores de envío).

La frase mental: **si para agregar un caso nuevo tenés que abrir y modificar una función vieja, violás OCP. Diseñá un punto de extensión (una interfaz) y que lo nuevo sea una clase nueva — el código que orquesta no se toca.**

**Ejercicios 2**
2.1 🔁 Enunciá el principio Open/Closed. ¿Qué significa "cerrado a modificación" en la práctica?
2.2 🧠 ¿Por qué un `switch` que crece con cada feature nueva es una señal de violación de OCP? ¿Qué riesgo concreto trae modificarlo cada vez?
2.3 ✍️ Tenés una función `exportar(formato: string, datos: Datos)` con un `if` para `'csv'`, `'json'` y `'xml'`. Refactorizala para que respete OCP (interfaz + una clase por formato).

---

## Módulo 3 — L: Liskov Substitution Principle (los subtipos reemplazan al padre)

**Teoría.** Esta es la que viene de afuera (Barbara Liskov) y la que más cuesta. La idea:

> **Si `S` es un subtipo de `T`, deberías poder usar un objeto `S` en cualquier lugar donde se espera un `T`, sin que el programa se rompa ni se comporte mal.**

En criollo: **una subclase tiene que cumplir el "contrato" de su superclase.** No alcanza con que herede los métodos —tiene que **comportarse** como lo que promete ser—. Si tu código funciona con la clase padre pero se rompe al pasarle una subclase, esa subclase viola LSP y la herencia está mal planteada.

El ejemplo canónico es **Cuadrado/Rectángulo**:

```ts
// ❌ Parece razonable: un cuadrado "es un" rectángulo... ¿o no?
class Rectangulo {
  constructor(protected ancho: number, protected alto: number) {}
  setAncho(w: number) { this.ancho = w }
  setAlto(h: number) { this.alto = h }
  area(): number { return this.ancho * this.alto }
}

class Cuadrado extends Rectangulo {
  setAncho(w: number) { this.ancho = w; this.alto = w }  // fuerza alto = ancho
  setAlto(h: number) { this.ancho = h; this.alto = h }   // fuerza ancho = alto
}

// Código que funciona con CUALQUIER Rectangulo:
function probar(r: Rectangulo): void {
  r.setAncho(5)
  r.setAlto(4)
  console.assert(r.area() === 20)  // ✅ con Rectangulo da 20... ❌ con Cuadrado da 16
}
```

`Cuadrado` rompe la **expectativa** del que usa `Rectangulo`: "si seteo ancho 5 y alto 4, el área es 20". Con un `Cuadrado` da 16, porque setear el alto te cambió el ancho por atrás. **La herencia "cuadrado es un rectángulo" es matemáticamente cierta pero conductualmente falsa.** Eso es violar Liskov.

Otras señales de violación de LSP:
- Una subclase que **tira `throw new Error('no soportado')`** en un método que el padre sí implementa (rompe el contrato).
- Una subclase que **endurece las precondiciones** o **debilita las postcondiciones**. (Una *precondición* es lo que un método **exige** para funcionar; una *postcondición*, lo que **garantiza** al terminar. Ejemplo: si el padre acepta cualquier número pero el hijo exige que sea positivo, *endureció* la precondición → rompe a quien le pasaba negativos. Si el padre garantiza devolver una lista **ordenada** y el hijo la devuelve sin ordenar, *debilitó* la postcondición → rompe a quien contaba con el orden.)

La analogía: si una receta dice "usá un **ave** que vuele para esta escena", y le pasás un **pingüino** (que es un ave pero no vuela), la escena se rompe. El pingüino "es un ave" en la taxonomía, pero **no cumple el contrato conductual** que el código esperaba. Y lo peligroso es lo **silencioso**: el pingüino entra perfecto en el molde de "ave" (compila, nadie sospecha), y la escena recién falla **en vivo**, cuando se esperaba que volara. Esa traición en runtime —no un error obvio en la cara— es lo que hace de Liskov una bomba de tiempo: como el `Cuadrado`, que aceptás como `Rectangulo` sin chistar y te muta el ancho por atrás.

⚠️ La lección práctica: cuando una subclase necesita "desactivar" o contradecir algo del padre, **probablemente la herencia está mal** y deberías usar **composición** en su lugar. La herencia es para "se comporta como", no para "comparte campos".

La frase mental: **Liskov no es sobre tipos, es sobre comportamiento: una subclase tiene que poder reemplazar al padre sin romper las expectativas de quien lo usa. Si tenés que tirar excepciones o anular métodos, la herencia está mal — usá composición.**

**Ejercicios 3**
3.1 🔁 Enunciá el LSP. ¿Por qué se dice que es sobre "comportamiento" y no solo sobre tipos?
3.2 🧠 ¿Por qué `Cuadrado extends Rectangulo` viola Liskov, si un cuadrado "es" un rectángulo? ¿Qué expectativa rompe?
3.3 🧠 Una subclase implementa un método del padre con `throw new Error('no implementado')`. ¿Qué principio viola y por qué? ¿Qué harías en su lugar?
3.4 ✍️ Dada la jerarquía `Cuadrado extends Rectangulo` del módulo, escribí un test (con `console.assert` o tu runner) que **pase con `Rectangulo` y falle con `Cuadrado`**, demostrando la violación. Después refactorizá para eliminarla (pista: dejá de heredar; usá una interfaz común o composición).

---

## Módulo 4 — I: Interface Segregation Principle (nadie depende de lo que no usa)

**Teoría.** El principio:

> **Ningún cliente debería verse forzado a depender de métodos que no usa.**

Traducido: **interfaces chicas y específicas, mejor que una interfaz gorda que hace de todo.** Si una clase implementa una interfaz enorme pero solo le importan 2 de sus 10 métodos, está **acoplada a 8 métodos que no le sirven** — y cada cambio en esos 8 la puede afectar o forzarla a implementar cosas vacías.

El smell: una interfaz "Dios" que obliga a implementar métodos que no aplican:

```ts
// ❌ Interfaz gorda: obliga a TODOS a implementar TODO
interface Trabajador {
  trabajar(): void
  comer(): void
  dormir(): void
}

class Humano implements Trabajador {
  trabajar(): void { }
  comer(): void { }
  dormir(): void { }
}

class Robot implements Trabajador {
  trabajar(): void { }
  comer(): void { throw new Error('un robot no come') }   // 💥 método forzado y sin sentido
  dormir(): void { throw new Error('un robot no duerme') } // 💥 (y además viola Liskov — ver más abajo)
}
```

`Robot` está obligado a implementar `comer()` y `dormir()` que no tienen sentido para él. La solución: **partir la interfaz gorda en interfaces chicas y cohesivas**, y que cada clase implemente solo las que le corresponden:

```ts
// ✅ Interfaces segregadas: cada uno implementa solo lo que le aplica
interface Trabajable { trabajar(): void }
interface Alimentable { comer(): void }
interface Descansable { dormir(): void }

class Humano implements Trabajable, Alimentable, Descansable {
  trabajar(): void { }
  comer(): void { }
  dormir(): void { }
}

class Robot implements Trabajable {   // solo lo que de verdad hace
  trabajar(): void { }
}
```

La analogía: un **menú** donde para pedir un café te **obligan a comprar el combo entero** con papas y gaseosa que no querés. ISP dice: dejá pedir solo el café. Interfaces a la carta, no menú fijo.

Fijate algo lindo: cuando partís bien las interfaces, **la violación de Liskov del módulo 3 desaparece sola** —`Robot` ya no tiene que tirar excepciones porque ni siquiera implementa lo que no hace—. Los principios se ayudan entre sí (módulo 6).

La frase mental: **muchas interfaces chicas y específicas le ganan a una interfaz gorda. Si una clase implementa métodos con `throw` o vacíos porque "no le aplican", esa interfaz hay que partirla. Café a la carta, no combo obligado.**

**Ejercicios 4**
4.1 🔁 Enunciá el ISP. ¿Qué problema trae una interfaz "gorda"?
4.2 🧠 ¿Cómo se relaciona una violación de ISP con una de Liskov? (pista: el `Robot` que tira `throw` en `comer()`).
4.3 ✍️ Tenés una interfaz `Dispositivo { imprimir(); escanear(); enviarFax() }` y una clase `ImpresoraSimple` que solo imprime. Refactorizá con interfaces segregadas.

---

## Módulo 5 — D: Dependency Inversion Principle (depender de abstracciones)

**Teoría.** El último, y el que **conecta SOLID con toda la arquitectura del hub.** El principio tiene dos partes:

> **1. Los módulos de alto nivel no deben depender de los de bajo nivel. Ambos deben depender de abstracciones.**
> **2. Las abstracciones no deben depender de los detalles; los detalles deben depender de las abstracciones.**

Desglosémoslo. **Alto nivel** = tu lógica de negocio (lo que importa, lo que da valor: "crear un pedido", "registrar un usuario"). **Bajo nivel** = los detalles de plomería (Postgres, un cliente de email, el sistema de archivos). El instinto natural es que lo de arriba dependa de lo de abajo:

```ts
// ❌ El servicio de negocio (alto nivel) depende DIRECTO de un detalle (bajo nivel)
class PostgresUsuarioDB {
  insertar(u: Usuario): void { /* SQL concreto */ }
}

class RegistrarUsuario {
  private db = new PostgresUsuarioDB()   // 🔒 atado a Postgres para siempre
  ejecutar(u: Usuario): void { this.db.insertar(u) }
}
```

El problema: tu lógica de negocio quedó **clavada a Postgres**. Para testearla necesitás una base real; para cambiar a Mongo reescribís el servicio. La dependencia apunta **hacia el detalle**, que es lo que cambia más seguido. **Invertí la dependencia**: que el alto nivel defina una **abstracción** (qué necesita) y que el detalle la implemente.

```ts
// ✅ Ambos dependen de la abstracción. El detalle "enchufa" en la interfaz que define el dominio.
interface UsuarioRepository {                 // la abstracción la define el ALTO nivel (qué necesita)
  insertar(u: Usuario): void
}

class RegistrarUsuario {
  constructor(private repo: UsuarioRepository) {}   // depende de la interfaz, no de Postgres
  ejecutar(u: Usuario): void { this.repo.insertar(u) }
}

class PostgresUsuarioRepository implements UsuarioRepository {  // el detalle IMPLEMENTA la abstracción
  insertar(u: Usuario): void { /* SQL */ }
}
// En un test: le pasás un `UsuarioRepositoryFalso` y probás la lógica sin base de datos, en milisegundos.
```

La flecha de dependencia **se invirtió**: antes `RegistrarUsuario → Postgres`; ahora `Postgres → UsuarioRepository ← RegistrarUsuario`. El detalle pasó a depender del dominio, no al revés. Eso es "inversión".

La analogía: un **control remoto universal**. No depende de la marca de la TV; depende de un **protocolo** (la interfaz infrarroja). Cualquier TV que implemente ese protocolo "enchufa". El control es el alto nivel, el protocolo la abstracción, cada TV un detalle.

**Y acá está el puente que prometimos:** este principio es **exactamente** el corazón de la **arquitectura hexagonal (puertos y adaptadores)** que ya tenés en el hub. El `UsuarioRepository` es un **puerto**; `PostgresUsuarioRepository` es un **adaptador**. La D de SOLID, aplicada a nivel arquitectónico, **es** hexagonal. No lo repetimos acá — está desarrollado en [Patrones en NestJS (módulo 8, Ports & Adapters)](nestjs-patrones.md), en [NestJS senior](nestjs-senior.md) y en [DDD (módulo 10, las capas)](ddd.md). Lo que cambia es la escala: SOLID habla de **clases**; hexagonal lleva la misma idea a la **arquitectura entera**. Mismo principio, distinto zoom.

> Nota: en frameworks como NestJS o Angular, la **inyección de dependencias (DI)** es el mecanismo que materializa la D — el contenedor te "inyecta" el adaptador concreto contra el puerto. DI es la herramienta; DIP es el principio. No los confundas: podés respetar DIP sin un framework de DI (pasando la dependencia por el constructor, como arriba).

La frase mental: **invertí la flecha: que tu negocio dependa de una interfaz que él mismo define, y que el detalle (DB, email) la implemente. Así testeás sin infra y cambiás de proveedor sin tocar la lógica. La D a nivel clase es DIP; a nivel arquitectura es hexagonal — mismo principio, distinto zoom.**

**Ejercicios 5**
5.1 🔁 Enunciá las dos partes del DIP. ¿Qué es "alto nivel" y qué es "bajo nivel"?
5.2 🧠 ¿Qué problema concreto trae que `RegistrarUsuario` haga `new PostgresUsuarioDB()` adentro? Mencioná testing y cambio de proveedor.
5.3 🧠 ¿Qué relación hay entre el DIP, los "puertos y adaptadores" de hexagonal y la "inyección de dependencias" de un framework? ¿Son lo mismo?
5.4 ✍️ Tenés `class EnviarNotificacion { private email = new SendgridClient() }`. Invertí la dependencia: definí un puerto, hacé que el servicio dependa de él, y escribí un adaptador concreto.

---

## Módulo 6 — SOLID en conjunto: cómo se refuerzan y qué problema atacan

**Teoría.** Los cinco principios no son una lista de compras independiente: **se refuerzan entre sí** y atacan **un mismo enemigo**. Entender eso es lo que te lleva de "me sé las siglas" a "diseño con criterio".

El enemigo común es el **acoplamiento rígido**: código donde un cambio se propaga en cascada y nada se puede tocar de forma aislada. Uncle Bob describe varios síntomas de un diseño que se pudre; quedate con estos tres (la lista completa suma la **viscosidad** y algún otro):
- **Rigidez:** un cambio chico obliga a tocar muchos lugares.
- **Fragilidad:** tocás algo y se rompe algo lejano que no tenía relación aparente.
- **Inmovilidad:** no podés reutilizar un pedazo porque viene pegado a todo lo demás.

SOLID es, todo junto, un **antídoto contra esos tres síntomas**. Y los principios se encadenan:

- La **S** (responsabilidad única) hace que cada pieza tenga un solo motivo de cambio → menos rigidez.
- La **D** (inversión de dependencias) habilita la **O** (open/closed): si dependés de una abstracción, agregar una implementación nueva no toca el código viejo.
- La **I** (segregación de interfaces) ayuda a la **L** (Liskov): interfaces chicas y precisas hacen que las implementaciones no tengan que "mentir" con métodos vacíos o excepciones.
- La **L** bien aplicada es lo que hace que el polimorfismo de la **O** sea **seguro** (las sustituciones no rompen).

En el fondo, las siglas son cinco caras de dos ideas viejas y eternas: **alta cohesión** (cada cosa junta con lo que le corresponde — S, I) y **bajo acoplamiento** (las cosas dependen de abstracciones, no de detalles — D, O, L). Si entendés cohesión y acoplamiento, entendés *por qué* existe SOLID.

La frase mental: **SOLID no son cinco reglas sueltas: son cinco formas de pelear contra el mismo enemigo (el acoplamiento rígido) y se habilitan entre sí. En el fondo es lo de siempre: alta cohesión, bajo acoplamiento.**

**Ejercicios 6**
6.1 🔁 Nombrá los tres síntomas de un diseño que "se pudre" (rigidez, fragilidad, inmovilidad) y explicá uno.
6.2 🧠 ¿Cómo habilita la D (inversión de dependencias) a la O (open/closed)? Dá el encadenamiento.
6.3 🧠 Se dice que SOLID se reduce a "alta cohesión, bajo acoplamiento". ¿Qué principios empujan cada una?

---

## Módulo 7 — El criterio: cuándo SOLID es over-engineering

**Teoría.** El módulo más importante, y el que casi nadie enseña. Porque **SOLID mal aplicado es peor que no aplicarlo.** Te lo dije al principio: es una caja de herramientas, no una religión. Acá está el criterio para no pasarte de rosca.

El anti-patrón estrella es la **abstracción prematura**: meter interfaces, puertos y capas "por las dudas", para un cambio que **nunca llega**. El resultado es una **sopa de indirecciones** donde para entender qué hace algo tenés que saltar por seis archivos. Eso no es código mantenible — es código **disfrazado de mantenible**, y a menudo más difícil de cambiar que el código simple que pretendía mejorar.

Las reglas de criterio:

- **YAGNI ("You Aren't Gonna Need It").** No abstraigas para un requerimiento hipotético. Abstraé cuando la variación es **real y presente** (ya tenés dos medios de pago, ya sabés que viene un tercero), no cuando es imaginaria.
- **La regla de tres.** Una primera implementación: hardcodeá. La segunda: notá la duplicación pero aguantá. **A la tercera**, ahí sí abstraé — ya tenés evidencia del patrón real, no una corazonada.
- **Duplicación antes que la abstracción equivocada.** Una abstracción mal elegida acopla cosas que parecían iguales pero no lo eran, y desacoplarlas después duele más que haber duplicado. Un poco de duplicación es más barata que una abstracción incorrecta.
- **El costo de la indirección es real.** Cada interfaz, cada capa, cada inyección agrega un salto mental. Vale la pena cuando compra flexibilidad que vas a usar; es puro lastre cuando no.

¿Cuándo NO te calientes con SOLID?
- ✅ Un **CRUD simple**, un script, un prototipo, un endpoint que lee y escribe sin lógica → código directo. Meterle hexagonal a un CRUD es matar una mosca a cañonazos.
- ✅ Cuando **todavía no sabés** dónde va a variar el sistema → esperá a que el dominio te lo muestre.
- ❌ Un dominio con **lógica de negocio real, que cambia y que varios tocan** → ahí SOLID se paga solo.

Esto conecta directo con el criterio de [DDD](ddd.md) ("no todo proyecto necesita DDD") y con el de [microfrontends](microfrontends.md) ("empezá monolito modular, extraé cuando duela"). Es el mismo principio senior repetido: **la complejidad se agrega cuando el dolor es real y medible, no por anticipado.**

La frase mental de cierre: **SOLID maneja el cambio — pero solo donde el cambio es real. Abstraer "por las dudas" te da una sopa de interfaces que duele más que el problema que evitabas. YAGNI, regla de tres, y duplicar antes que abstraer mal. La marca del senior no es aplicar SOLID en todos lados: es saber dónde NO.**

**Ejercicios 7**
7.1 🔁 ¿Qué es la "abstracción prematura" y por qué es un problema?
7.2 🧠 Explicá la "regla de tres". ¿Por qué conviene duplicar dos veces antes de abstraer?
7.3 🧠 Un compañero quiere meter puertos, adaptadores e interfaces a un endpoint que solo lee un registro de la DB y lo devuelve. ¿Qué le decís y por qué?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** La definición correcta es **"un módulo debe tener una sola razón para cambiar"** (o, afinada: ser responsable ante un solo *actor*). "Hace una sola cosa" se queda corta porque es **vaga** —"una cosa" no se puede medir— y lleva al error de partir todo en clases de un método. "Una razón para cambiar" es **operativo**: te hacés la pregunta "¿quién, en el negocio, me haría tocar esto?" y si son varios actores distintos, hay varias responsabilidades.

**1.2** Tiene **tres** razones de cambio / actores: validación (reglas de seguridad), persistencia (infraestructura/DB) y comunicación (el área que define los emails de onboarding). Cada una en su colaborador:
```ts
class Usuario {                                   // entidad de dominio, sin plomería
  constructor(public readonly email: string, private readonly hash: string) {}
}
class ValidadorDePassword {
  esValida(password: string): boolean { return password.length >= 8 }  // reglas de seguridad
}
class UsuarioRepository {
  guardar(u: Usuario): Promise<void> { return Promise.resolve() }       // persistencia
}
class ServicioDeBienvenida {
  enviar(u: Usuario): Promise<void> { return Promise.resolve() }        // comunicación
}
```
`Usuario` queda como entidad pura; cada actor (seguridad, infra, comunicación) toca **su** clase y a nadie más.

**1.3** Porque la S es sobre **razones de cambio**, no sobre cantidad de métodos. Una clase puede tener muchos métodos si **todos sirven al mismo actor**. Ejemplo: una clase `Pedido` con `agregarLinea()`, `quitarLinea()`, `calcularTotal()` y `aplicarDescuento()` tiene varios métodos pero **una sola razón para cambiar**: las reglas del pedido (un solo actor, el negocio de ventas). Respeta la S aunque no sea "un método por clase".

**2.1** "Las entidades de software deben estar **abiertas a la extensión pero cerradas a la modificación**." "Cerrado a modificación" en la práctica significa: para agregar comportamiento nuevo **no abrís ni editás el código que ya funciona y está testeado** — lo extendés con código nuevo (una clase nueva que implementa una abstracción).

**2.2** Porque cada feature nueva te obliga a **modificar** esa función (agregar un `case`), y modificar código que ya andaba significa: (1) **riesgo de romper** los casos viejos, y (2) tener que **re-testear todo** el `switch`, no solo lo nuevo. Con OCP bien hecho, el caso nuevo es una clase nueva aislada: lo viejo no se toca ni se re-testea.

**2.3**
```ts
interface Exportador {
  exportar(datos: Datos): string
}
class ExportadorCSV implements Exportador {
  exportar(datos: Datos): string { return '' /* a CSV */ }
}
class ExportadorJSON implements Exportador {
  exportar(datos: Datos): string { return JSON.stringify(datos) }
}
class ExportadorXML implements Exportador {
  exportar(datos: Datos): string { return '' /* a XML */ }
}
// El orquestador queda cerrado: no sabe qué formato es.
function exportar(exportador: Exportador, datos: Datos): string {
  return exportador.exportar(datos)
}
// Agregar 'yaml' mañana = una clase nueva, cero cambios acá.
```

**3.1** "Si `S` es subtipo de `T`, los objetos `T` deben poder reemplazarse por objetos `S` sin alterar la correctitud del programa." Es sobre **comportamiento** y no solo tipos porque heredar la firma de los métodos no alcanza: la subclase tiene que **cumplir el contrato** (las pre/postcondiciones y las expectativas) del padre. Compila ≠ se comporta bien.

**3.2** Porque rompe la **expectativa** de quien usa `Rectangulo`: "si seteo ancho 5 y alto 4, el área es 20". `Cuadrado` redefine los setters para mantener los lados iguales, así que setear el alto cambia el ancho por atrás → el área da 16. El código que funcionaba con `Rectangulo` falla con `Cuadrado`. Matemáticamente un cuadrado es un rectángulo, pero **conductualmente** no es sustituible → viola LSP. (La herencia está mal; convendría no heredar.)

**3.3** Viola **Liskov**: el padre promete que ese método funciona, y la subclase rompe la promesa tirando una excepción → no es sustituible (el código que esperaba el comportamiento del padre se rompe). En su lugar: revisar la jerarquía. Probablemente esa subclase **no debería heredar** de ese padre (no cumple su contrato); usar **composición** o **segregar la interfaz** (módulo 4) para que la subclase solo implemente lo que de verdad hace.

**3.4**
```ts
// 1) El test que demuestra la violación: pasa con Rectangulo, falla con Cuadrado.
function verificarArea(r: Rectangulo): void {
  r.setAncho(5)
  r.setAlto(4)
  console.assert(r.area() === 20, `esperaba 20, obtuve ${r.area()}`)
}
verificarArea(new Rectangulo(0, 0)) // ✅ 20
verificarArea(new Cuadrado(0, 0))   // ❌ 16 — el setAlto mutó el ancho por atrás

// 2) Refactor: no hay relación de herencia. Cada forma es independiente y, si querés
//    tratarlas en conjunto, comparten una interfaz mínima (no setters que mientan).
interface Figura {
  area(): number
}
class Rectangulo2 implements Figura {
  constructor(private readonly ancho: number, private readonly alto: number) {}
  area(): number { return this.ancho * this.alto }
}
class Cuadrado2 implements Figura {
  constructor(private readonly lado: number) {}
  area(): number { return this.lado * this.lado }
}
```
La clave: al hacer las dimensiones inmutables (`readonly`) y **no** forzar a `Cuadrado` a ser un `Rectangulo`, desaparece la sustitución traicionera. Comparten `Figura` (lo que de verdad tienen en común: calcular área), no una jerarquía falsa.

**4.1** "Ningún cliente debería verse forzado a depender de métodos que no usa." Una interfaz "gorda" trae que las clases que la implementan queden **acopladas a métodos que no les sirven**: tienen que implementarlos (a veces con `throw` o vacíos), y un cambio en esos métodos puede afectarlas aunque no los usen.

**4.2** Están conectadas: cuando una interfaz gorda **obliga** a `Robot` a implementar `comer()`, te deja como única salida razonable un `throw new Error(...)` — y eso **viola Liskov** (el `Robot` ya no es sustituible donde se espera un `Trabajador` que come). O sea: la violación de ISP (interfaz gorda) **propicia/empuja** a la de Liskov; no es una ley absoluta (en teoría podrías implementar el método de forma válida), pero en la práctica la interfaz gorda es la que te arrincona a romper el contrato. Segregar la interfaz elimina las dos de un saque: `Robot` implementa solo `Trabajable` y no tiene que mentir.

**4.3**
```ts
interface Imprimible { imprimir(): void }
interface Escaneable { escanear(): void }
interface Faxeable { enviarFax(): void }

class ImpresoraSimple implements Imprimible {
  imprimir(): void { }    // solo lo que hace
}
class MultifuncionPro implements Imprimible, Escaneable, Faxeable {
  imprimir(): void { }
  escanear(): void { }
  enviarFax(): void { }
}
```
`ImpresoraSimple` ya no está forzada a implementar `escanear()`/`enviarFax()` con métodos vacíos o excepciones.

**5.1** (1) Los módulos de **alto nivel** no deben depender de los de **bajo nivel**; ambos deben depender de **abstracciones**. (2) Las abstracciones no dependen de los detalles; los detalles dependen de las abstracciones. **Alto nivel** = la lógica de negocio (lo que da valor: registrar un usuario, crear un pedido). **Bajo nivel** = los detalles de plomería (la DB concreta, el cliente de email, el filesystem).

**5.2** Hacer `new PostgresUsuarioDB()` adentro **clava** la lógica de negocio a Postgres. Dos problemas concretos: (1) **Testing** — para probar `RegistrarUsuario` necesitás una base real (lento, frágil), en vez de inyectarle un repositorio falso y testear en milisegundos. (2) **Cambio de proveedor** — migrar a Mongo (o a otro almacenamiento) te obliga a **reescribir el servicio**, porque la dependencia apunta al detalle. Invirtiéndola, cambiás el adaptador y el servicio ni se entera.

**5.3** Son **el mismo principio en tres escalas/roles**, no cosas distintas: el **DIP** es el principio (dependé de abstracciones); **puertos y adaptadores (hexagonal)** es ese principio aplicado a nivel **arquitectura** (el puerto = la abstracción del dominio, el adaptador = el detalle que la implementa); la **inyección de dependencias** del framework es el **mecanismo** que te entrega el adaptador concreto contra el puerto en runtime. No son lo mismo pero están encadenados: DIP (principio) → hexagonal (arquitectura que lo encarna) → DI (herramienta que lo implementa). Podés respetar DIP sin framework de DI (inyectando por constructor).

**5.4**
```ts
// 1) el puerto (abstracción que define el alto nivel)
interface NotificadorPort {
  enviar(destino: string, mensaje: string): Promise<void>
}

// 2) el servicio depende del puerto, no del detalle
class EnviarNotificacion {
  constructor(private readonly notificador: NotificadorPort) {}
  async ejecutar(destino: string, mensaje: string): Promise<void> {
    await this.notificador.enviar(destino, mensaje)
  }
}

// 3) el adaptador concreto implementa el puerto
class SendgridNotificador implements NotificadorPort {
  async enviar(destino: string, mensaje: string): Promise<void> {
    /* llamada real a Sendgrid */
  }
}
// En tests: un NotificadorFalso implements NotificadorPort, sin tocar la red.
```

**6.1** **Rigidez** (un cambio chico obliga a tocar muchos lugares), **fragilidad** (tocás algo y se rompe algo lejano sin relación aparente) e **inmovilidad** (no podés reutilizar un módulo porque viene pegado a todo lo demás). Ejemplo de fragilidad: cambiás el formato de una fecha en un reporte y se rompe el login, porque ambos compartían una clase utilitaria sobrecargada. (Son tres de los síntomas que describe Uncle Bob; la lista completa suma la **viscosidad** y algún otro — pero con estos tres ya tenés el cuadro.)

**6.2** La D dice "dependé de una abstracción (interfaz), no de una implementación concreta". Una vez que tu código orquestador depende de la **interfaz** `MedioDePago` (y no de `Tarjeta` o `PayPal` concretos), agregar un medio nuevo es crear **una clase nueva** que implementa la interfaz — y el orquestador, que solo conoce la abstracción, **no se modifica**. Eso es exactamente la O (abierto a extensión, cerrado a modificación). Sin la D, no podés tener la O: estarías acoplado a las clases concretas.

**6.3** **Alta cohesión** la empujan la **S** (cada clase con un solo motivo, todo lo suyo junto) y la **I** (interfaces cohesivas, solo métodos relacionados). **Bajo acoplamiento** lo empujan la **D** (dependés de abstracciones, no de detalles), la **O** (extendés sin tocar lo existente) y la **L** (las sustituciones no acoplan al comportamiento concreto). Las cinco siglas son cinco maneras de empujar esas dos ideas.

**7.1** Es meter abstracciones (interfaces, puertos, capas) **antes** de que exista una necesidad real de variación — para un cambio hipotético que quizás nunca llega. Es un problema porque agrega **indirección y complejidad** (más archivos, más saltos mentales) sin comprar flexibilidad real: el código queda más difícil de entender y de cambiar que la versión simple que pretendía mejorar.

**7.2** La regla de tres: la **primera** vez, escribí la solución directa (hardcodeá); la **segunda** vez que aparece algo parecido, notás la duplicación pero **aguantás**; recién a la **tercera** abstraés. Conviene esperar porque con dos casos todavía no tenés evidencia de cuál es el patrón **real** de variación — abstraer temprano corre el riesgo de elegir la abstracción **equivocada**, que acopla cosas que parecían iguales y no lo eran. Con tres casos ya ves el patrón de verdad.

**7.3** Le digo que **no hace falta** y que sería over-engineering. Un endpoint que solo lee un registro y lo devuelve **no tiene lógica de negocio** ni variación previsible: meterle puertos/adaptadores/interfaces agrega indirección (más archivos, más saltos) sin comprar nada — no vas a testear lógica que no existe ni a cambiar un proveedor que no es problema. Código directo. Reservá hexagonal/SOLID pesado para el dominio con **lógica real que cambia y que varios tocan**; para un CRUD simple, es matar una mosca a cañonazos.

---

> **Para seguir.** Las fuentes de verdad de SOLID son los escritos de **Robert C. Martin ("Uncle Bob")**: el paper *"Design Principles and Design Patterns"* (2000) y el libro *Clean Architecture* (2017), donde reformula la S como "un módulo responde a un solo actor". La **L** original es de **Barbara Liskov** (*"Data Abstraction and Hierarchy"*, 1987; formalizada con Jeannette Wing, 1994). El puente natural desde acá: la **D** llevada a arquitectura en [Patrones en NestJS (Ports & Adapters)](nestjs-patrones.md), [NestJS senior](nestjs-senior.md) y [DDD](ddd.md); y el criterio de "cuándo NO" que comparten [DDD](ddd.md) y [Microfrontends](microfrontends.md). Antes que memorizar las siglas, quedate con las dos ideas que las sostienen: **alta cohesión y bajo acoplamiento.**
