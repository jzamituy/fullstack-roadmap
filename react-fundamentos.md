# React desde los fundamentos: hooks y Context API

**El modelo mental de los hooks, Context y las novedades de React 19 · ejemplos en TypeScript · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es la **planta baja** de la sección [Full Stack web (React)](tanstack-start.md): los módulos de [TanStack Start](tanstack-start.md), [RSC](rsc.md) y [Microfrontends](microfrontends.md) **usan** hooks todo el tiempo pero asumen que ya los entendés. Acá los construimos desde el modelo mental, que es lo que separa "sé escribir `useEffect`" de "sé **por qué** mi `useEffect` corre dos veces y cómo arreglarlo". Eso es lo que se pregunta en entrevistas.

**Lo que asumimos.** Sabés qué es un componente y JSX (venís de React o React Native, ya tirás componentes sin pensar), `npm`, y JavaScript moderno (arrow functions, destructuring, `const`/`let`, closures). Si los **closures** te suenan borrosos, frená y repasalos: **los hooks son closures con esteroides** y sin esa base, `useState` y `useEffect` van a parecer magia. No queremos magia, queremos entender.

> **¿Por qué empezar acá si ya hacés React?** Porque "hacer que funcione" y "entender el modelo" son dos cosas distintas. Mucha gente escribe hooks que andan **por casualidad** —hasta que un día no andan y no saben por qué—. Este módulo te da el modelo mental de react.dev: render como snapshot, efectos como sincronización, identidad referencial. Con eso, los bugs raros dejan de ser raros.

> ⚠️ **Nota sobre datos volátiles.** React 19 (estable desde dic 2024; vamos por 19.2 a mediados de 2026) cambió cosas que vas a ver marcadas con ⚠️: el **React Compiler** (que cambia cuándo necesitás `useMemo`/`useCallback`), `ref` como prop (adiós `forwardRef`), `<Context>` como provider directo y los **Actions**. Verificá lo marcado contra `react.dev` antes de copiarlo: algunas APIs están en camino de deprecación pero **todavía funcionan**.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (requiere editor).

**Índice de módulos**
1. El modelo mental: qué es un "render" y por qué lo cambia todo
2. Las reglas de los hooks (y por qué existen)
3. `useState`: el estado como una foto, no como una variable
4. `useEffect`: sincronizar con el mundo, NO un "ciclo de vida"
5. `useRef`: la referencia que no redibuja (y `ref` como prop en React 19)
6. `useMemo` / `useCallback` y el React Compiler: la era de la memoización automática
7. Context API: el problema del prop drilling (y por qué NO es un state manager)
8. Custom hooks: compartir lógica, no estado
9. React 19: `use`, Actions y los hooks concurrentes
10. El criterio: estado local vs Context vs URL vs state manager

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — El modelo mental: qué es un "render" y por qué lo cambia todo

**Teoría.** Antes de tocar un hook, tenés que entender qué es un **render**, porque **todo** lo demás se apoya en esto. Si este módulo te queda claro, los otros nueve caen solos.

React hace tres pasos: **Trigger → Render → Commit.**
- **Trigger:** algo dispara la actualización — el render inicial (`createRoot(...).render(<App/>)`) o un cambio de estado.
- **Render:** *React llama a tu función-componente.* Eso es renderizar: ejecutar tu función para ver qué JSX devuelve. **Tiene que ser una función pura** (mismos props/estado → mismo JSX, sin efectos secundarios).
- **Commit:** React compara el resultado con lo que había y toca el DOM **solo en lo que cambió**.

Y acá está la idea que lo cambia todo: **cada render es una foto (un snapshot).** Cuando React llama a tu componente, todo lo que vive adentro —los props, el estado, las variables, las funciones que definís, los objetos `{}`— **pertenece a ESE render y a ningún otro.** En el próximo render, React vuelve a llamar a tu función **desde cero**, y se crean **valores nuevos**.

Pensalo como un **flipbook** (esos cuadernitos que animás pasando las hojas): cada hoja es un render, un dibujo congelado. La animación es la **secuencia** de hojas, pero cada hoja por separado está quieta. Tu componente no "muta" entre frames: React dibuja una hoja nueva cada vez.

De acá sale la consecuencia más importante para entender hooks: la **identidad referencial**. Mirá:

```tsx
function Componente() {
  const obj = { id: 1 }            // objeto NUEVO en cada render
  const fn = () => console.log('hola') // función NUEVA en cada render
  // obj y fn del render #2 NO son === a los del render #1, aunque "se vean iguales"
  return <Hijo data={obj} onClick={fn} />
}
```

`{ id: 1 }` del render 1 y `{ id: 1 }` del render 2 tienen el **mismo contenido** pero son **objetos distintos** en memoria (`Object.is` da `false`). Esto, que parece un detalle, es la causa de **la mitad de los bugs de hooks**: por qué un `useEffect` corre de más, por qué `React.memo` no frena un re-render, por qué un Context redibuja todo. Anclalo ahora.

**La otra cara de la identidad: las `key` de las listas.** Cuando renderizás una lista, React usa la `key` de cada elemento como su **identidad estable** entre renders para decidir qué se agregó, movió o borró (la *reconciliación*). Por eso `key={index}` es una trampa: si reordenás o insertás al principio, los índices se corren y React asocia el estado/foco al elemento **equivocado** (el texto que tipeaste en un input "salta" a otra fila). Usá un **id estable del dato** (no la posición) como key.

La frase mental: **un render es una foto: React llama a tu función y todo lo de adentro nace y muere en ese render. Objetos y funciones se crean nuevos cada vez — misma forma, distinta identidad.**

**Ejercicios 1**
1.1 🔁 Nombrá los tres pasos de React y decí en cuál "React llama a tu componente".
1.2 🧠 ¿Por qué `{ id: 1 } === { id: 1 }` da `false` en dos renders distintos, si "se ven iguales"? ¿Por qué te debería importar?
1.3 🧠 ¿Qué significa que "renderizar tiene que ser una función pura"? Dá un ejemplo de algo que NO deberías hacer durante el render.
1.4 ✍️ Escribí un componente que, en cada render, cree un objeto `{ id: 1 }`, lo guarde en un ref el primer render, y logguee si el objeto de este render es `===` al del render anterior. Forzá re-renders con un botón y mirá la consola: vas a *ver* romperse la identidad referencial.
1.5 🧠 ¿Por qué `key={index}` puede corromper el estado (o el foco) de una lista al reordenarla o insertar al principio, y qué deberías usar como `key` en su lugar?

---

## Módulo 2 — Las reglas de los hooks (y por qué existen)

**Teoría.** Hay **dos reglas** de los hooks. La mayoría las memoriza; vos vas a entender el **por qué**, que es lo que importa.

1. **Solo llamá hooks en el nivel superior.** Nada de hooks dentro de `if`, loops, funciones anidadas o bloques `try/catch/finally`. Siempre arriba de todo del componente, **antes de cualquier `return` temprano**.
2. **Solo llamá hooks desde funciones de React.** Desde un componente o desde un *custom hook* — nunca desde una función JS común.

¿Por qué la regla 1, que parece arbitraria? Porque **React identifica cada hook por el ORDEN en que se llama**, no por un nombre. Internamente es como una lista: "el primer `useState` de este componente es la edad, el segundo es el nombre…". React no ve los nombres de tus variables; cuenta posiciones.

```tsx
// ⚠️ Bloque INTENCIONALMENTE inválido: no compila bien ni pasa el linter — ese es el punto.
function Form() {
  const [name, setName] = useState('')     // hook #1
  if (name === '') {
    const [error, setError] = useState(null) // 💥 hook condicional
  }
  const [age, setAge] = useState(0)         // ¿hook #2 o #3?
}
```

Si el `useState` del medio a veces se llama y a veces no, **el orden se rompe**: en un render `age` es el hook #3, en otro el #2, y React le entrega el estado equivocado. Por eso la regla. Es así de fácil: **orden estable = identidad estable.**

¿Y la regla 2? Para que **toda la lógica con estado sea visible** desde el código del componente, y para que el linter pueda chequear la regla 1. Por eso los custom hooks se llaman `use*` (módulo 8): es la señal de "esto puede llamar hooks adentro".

La herramienta: el plugin **`eslint-plugin-react-hooks`**. No es opcional — instalalo y respetalo. Te marca las dos reglas y las dependencias de los efectos (módulo 4). ⚠️ Desde su v6 (oct 2025) el preset `recommended` usa *flat config* (ESLint 9) y ya incluye **reglas potenciadas por el React Compiler** (módulo 6) —algunas siguen siendo opt-in dentro del preset—; si tu config es vieja, verificá la forma exacta en la doc.

La frase mental: **React identifica los hooks por orden de llamada, no por nombre. Si un hook se llama condicionalmente, el orden se rompe y React te da el estado equivocado. Por eso: siempre arriba, siempre todos.**

**Ejercicios 2**
2.1 🔁 Enunciá las dos reglas de los hooks.
2.2 🧠 ¿Por qué no podés llamar un hook dentro de un `if`? Explicá el mecanismo real (no "porque React lo prohíbe").
2.3 ✍️ Tenés un componente que hace `if (loading) return <Spinner/>` y DESPUÉS un `useEffect`. ¿Qué está mal y cómo lo arreglás?

---

## Módulo 3 — `useState`: el estado como una foto, no como una variable

**Teoría.** `useState` te da un valor que **sobrevive entre renders** y un setter que, al llamarlo, le pide a React **un nuevo render**.

```tsx
const [age, setAge] = useState(0)
```

Acá está la trampa #1, y conecta directo con el módulo 1: **`setAge` no cambia `age` en el código que se está ejecutando.** El valor `age` es una **foto** congelada de este render. ¿Te acordás del flipbook del módulo 1? Cada hoja del flipbook **es** una foto, y el estado es lo que está pintado en ESA hoja: no cambia mientras mirás esa hoja. Llamar al setter no muta esa foto — agenda un render futuro (la próxima hoja) con el valor nuevo.

```tsx
function handleClick() {
  setAge(age + 1)
  setAge(age + 1)
  setAge(age + 1)
  // age vale 0 en esta foto → los tres calculan 0 + 1 = 1. Resultado: 1, no 3.
}
```

Los tres leen el **mismo** `age` (0) porque están en la misma foto. ¿La solución? El **functional update**: le pasás una función que recibe el valor más reciente en la cola.

```tsx
function handleClick() {
  setAge(prev => prev + 1)
  setAge(prev => prev + 1)
  setAge(prev => prev + 1)
  // 0→1→2→3. Resultado: 3. ✅
}
```

Regla práctica: **si el nuevo estado depende del anterior, usá la forma funcional `setX(prev => ...)`.**

Otras dos cosas que tenés que saber:

- **Lazy initial state.** Si calcular el estado inicial es caro, pasá la **función**, no el resultado:
  ```tsx
  useState(() => crearTodosIniciales())  // ✅ corre solo en el primer render
  useState(crearTodosIniciales())        // ❌ corre en CADA render (y se descarta)
  ```
- **Batching (agrupado).** React agrupa las actualizaciones: ejecuta todo el handler y **recién después** redibuja una sola vez. Desde React 18 esto pasa **siempre** (también dentro de promesas, `setTimeout`, etc.), no solo en eventos de React.

Y la regla de oro de los objetos/arrays: **no mutes, reemplazá.** React compara con `Object.is`; si mutás el mismo objeto, la referencia no cambió y **React no se entera**.

```tsx
// ❌ React no redibuja: es el mismo array
todos.push(nuevo); setTodos(todos)
// ✅ array nuevo
setTodos([...todos, nuevo])
```

**Conectado: inputs controlados vs no controlados.** Un input **controlado** tiene su valor en estado de React (`value={x}` + `onChange`): React es la fuente de verdad, redibuja en cada tecla y podés validar/transformar al vuelo. Uno **no controlado** deja el valor en el DOM y lo leés cuando hace falta con un ref (`defaultValue` + `ref`, módulo 5): menos re-renders, pero React no "ve" cada cambio. El default es **controlado** (lo que querés casi siempre); el no controlado queda para casos puntuales —integrar con DOM no-React, un form simple de una sola lectura, o `<input type="file">` (que es no controlado obligatoriamente)—.

La frase mental: **el estado es una foto del render, no una variable viva. El setter no la cambia: agenda la próxima foto. Si dependés del valor anterior, `setX(prev => ...)`; y nunca mutes, reemplazá.**

**Ejercicios 3**
3.1 🔁 ¿Por qué llamar `setCount(count + 1)` tres veces seguidas suma 1 y no 3? ¿Cómo lo arreglás?
3.2 🧠 ¿Qué diferencia hay entre `useState(crearInicial())` y `useState(crearInicial)`? ¿Cuándo importa?
3.3 🧠 Tenés `const [user, setUser] = useState({ name: 'Ana', age: 30 })` y querés cambiar solo la edad. ¿Por qué `user.age = 31; setUser(user)` no redibuja, y cómo lo hacés bien?
3.4 ✍️ Escribí un componente `Contador` con un botón "+3" que sume tres de una, usando la forma funcional. Explicá en un comentario por qué sin ella sumaría 1.
3.5 🧠 Diferenciá un input **controlado** de uno **no controlado**. ¿Cuándo conviene cada uno?

---

## Módulo 4 — `useEffect`: sincronizar con el mundo, NO un "ciclo de vida"

**Teoría.** Acá está el hook que **más se usa mal**, y casi siempre por el mismo motivo: la gente que viene de clases lo trata como `componentDidMount`/`componentDidUpdate`. **Sacate eso de la cabeza.** La doc oficial es tajante: `useEffect` es para **sincronizar tu componente con un sistema EXTERNO a React**.

¿Qué es "externo a React"? El DOM no-React, una suscripción a un WebSocket, un `setInterval`, una librería de terceros (un mapa, un chart), el `localStorage`. Cosas que viven afuera del mundo de render de React y que hay que **sincronizar** con tu estado.

El modelo mental son **tres categorías** de código:
- **Render (puro):** calcular el JSX. Sin efectos.
- **Event handlers:** acciones que pasan porque el usuario hizo algo (un click). Lógica imperativa.
- **Effects:** efectos secundarios que pasan **por el hecho de renderizar**, no por un evento puntual.

```tsx
useEffect(() => {
  const conn = crearConexion(serverUrl, roomId)
  conn.connect()
  return () => conn.disconnect()  // cleanup: se ejecuta antes del próximo efecto y al desmontar
}, [serverUrl, roomId])           // dependencias: cuándo re-sincronizar
```

Tres piezas:
- **El array de dependencias** *no lo elegís vos*: son **todo lo reactivo que el efecto lee** (props, estado). El linter te las exige. `[]` = solo al montar; sin array = en cada render; `[a, b]` = al montar y cuando `a` o `b` cambian.
- **El cleanup** (la función que devolvés): React lo corre **antes** de re-ejecutar el efecto y al desmontar. Es lo que evita conexiones colgadas y listeners duplicados.
- **En desarrollo, con Strict Mode, el efecto corre DOS veces** (setup → cleanup → setup) a propósito, para que descubras si te falta cleanup. **No es un bug y no lo tapes con un `if (yaCorrió)`** — escribí el cleanup bien. En producción corre una sola vez.

> **El closure en acción (acordate del intro).** Te dije que "los hooks son closures con esteroides". Acá se ve: un efecto con `[]` **captura la foto del primer render** y la congela.
> ```tsx
> useEffect(() => {
>   const id = setInterval(() => {
>     setCount(count + 1)   // ❌ `count` quedó congelado en 0 (la foto del montaje) → siempre 0 + 1
>   }, 1000)
>   return () => clearInterval(id)
> }, [])  // [] = nunca re-sincroniza → el closure jamás ve un `count` nuevo
> ```
> El callback del intervalo hace **closure** sobre el `count` de aquel primer render y nunca lo ve cambiar: el contador se queda en 1. Esto es un **stale closure**, y es exactamente la cara concreta del snapshot del módulo 1. El arreglo es el del módulo 3: `setCount(prev => prev + 1)` — no lee la foto vieja, le pide a React el último valor de la cola.

Ahora lo más importante de todo: **probablemente NO necesitás ese efecto.** La doc tiene una página entera ("You Might Not Need an Effect") porque el abuso es epidémico. Dos casos que NO van en un efecto:

1. **Transformar datos para renderizar.** Si podés calcularlo durante el render, calculalo ahí — no lo metas en estado vía efecto.
   ```tsx
   // ❌ estado redundante + un render extra
   const [fullName, setFullName] = useState('')
   useEffect(() => { setFullName(first + ' ' + last) }, [first, last])
   // ✅ derivado, durante el render
   const fullName = first + ' ' + last
   ```
2. **Lógica de un evento del usuario.** Mostrar "agregado al carrito" va en el `onClick`, no en un efecto que observa el estado del carrito.

El test que da la doc: **usá un efecto solo para código que debe correr *porque el componente se mostró en pantalla*** (sincronizar con algo externo). Si corre por un click → handler. Si es un valor calculable → render. ¿Se entiende? Esto solo ya te ahorra la mitad de los bugs.

La frase mental: **`useEffect` no es un ciclo de vida — es sincronización con el mundo externo. Si el dato se puede calcular en el render, calculalo; si pasa por un click, va en el handler. Las dependencias no se eligen, se descubren. Y el cleanup no es opcional.**

**Ejercicios 4**
4.1 🔁 Según react.dev, ¿para qué sirve `useEffect` y para qué NO sirve?
4.2 🧠 Tenés `const [first, last] = ...` y querés mostrar el nombre completo. ¿Por qué meterlo en estado con un `useEffect` está mal, y qué hacés en su lugar?
4.3 🧠 En desarrollo tu efecto corre dos veces y te rompe (abre dos conexiones). ¿Qué te está diciendo React y cuál es el arreglo CORRECTO (no el parche)?
4.4 ✍️ Escribí un efecto que se suscriba a un evento `online`/`offline` del navegador (`window.addEventListener`) y guarde el estado, **con cleanup**. Marcá las dependencias.

---

## Módulo 5 — `useRef`: la referencia que no redibuja (y `ref` como prop en React 19)

**Teoría.** `useRef` te da una "caja" mutable que **persiste entre renders** pero cuya mutación **NO dispara un re-render**. Esa es toda la diferencia con `useState`.

```tsx
const ref = useRef(0)
ref.current = ref.current + 1  // NO redibuja
```

¿Para qué sirve algo que persiste pero no redibuja? Para dos cosas:
- **Valores que no se muestran:** un `id` de `setInterval`, el último valor de algo, una bandera interna. Datos que necesitás recordar pero que no salen en pantalla.
- **Referencias al DOM:** el uso más común. Le pasás el ref a un elemento y React te pone el nodo en `.current`.
  ```tsx
  function Buscador() {
    const inputRef = useRef<HTMLInputElement>(null)
    return (
      <>
        <input ref={inputRef} />
        <button onClick={() => inputRef.current?.focus()}>Enfocar</button>
      </>
    )
  }
  ```

La diferencia con estado, en una imagen: **el estado es lo que se dibuja; el ref es una nota adhesiva pegada al costado del monitor.** Cambiar la nota no redibuja la pantalla. Por eso: si el valor afecta lo que se ve → estado. Si es plomería interna → ref.

⚠️ **Cuidado:** no leas ni escribas `ref.current` *durante el render* (rompe la pureza del módulo 1). Solo en handlers y efectos.

⚠️ **React 19 — `ref` como prop.** Antes, para pasarle un ref a tu propio componente, necesitabas `forwardRef`. En React 19, **los componentes-función reciben `ref` como una prop normal**:

```tsx
// React 19: sin forwardRef
function MiInput({ ref, ...props }: { ref?: React.Ref<HTMLInputElement> } & React.ComponentProps<'input'>) {
  return <input ref={ref} {...props} />
}
```

(Detalle de tipado: en React 19 `React.ComponentProps<'input'>` **ya incluye `ref`**, así que la intersección con `{ ref?: ... }` de arriba es redundante; la dejamos explícita solo para que se vea de dónde sale.)

`forwardRef` **todavía funciona** en 19.2, pero está en camino de deprecación (hay un codemod para migrar). Para código nuevo: no escribas `forwardRef`. Verificá el estado exacto en react.dev antes de asumir que ya se removió — **no se removió todavía**.

La frase mental: **el estado redibuja; el ref no. Ref = recordar algo sin pintarlo (un nodo del DOM, un timer). Y en React 19, `ref` es una prop más — `forwardRef` quedó para el código viejo.**

**Ejercicios 5**
5.1 🔁 ¿Cuál es la diferencia fundamental entre `useRef` y `useState`?
5.2 🧠 Querés guardar el `id` que devuelve `setInterval` para limpiarlo después. ¿Estado o ref? ¿Por qué?
5.3 ✍️ Escribí un componente con un `<input>` y un botón que, al hacer click, lo enfoca. Tipá el ref correctamente.

---

## Módulo 6 — `useMemo` / `useCallback` y el React Compiler: la era de la memoización automática

**Teoría.** Estos dos hooks confunden mucho, así que vamos al grano: **los dos cachean cosas entre renders para no recalcular/recrear de gusto.** La diferencia es qué cachean:

- **`useMemo(fn, deps)`** cachea el **resultado** de un cálculo.
- **`useCallback(fn, deps)`** cachea la **función misma**. (Es literalmente `useMemo(() => fn, deps)`.)

```tsx
const visibles = useMemo(() => items.filter(i => i.activo), [items]) // cachea el array filtrado
const onSave = useCallback(() => guardar(id), [id])                  // cachea la función
```

¿Y por qué querrías cachear una función? **Por la identidad referencial del módulo 1.** Sin `useCallback`, `onSave` es una función nueva en cada render, y eso **rompe** un `React.memo` del hijo (lo ve como prop nueva) o dispara un `useEffect` que la tiene en deps. La memoización mantiene la **misma referencia** mientras las deps no cambien.

Pero ojo —y esto es criterio senior—: **estos hooks son SOLO optimización.** La doc lo dice claro: *"si tu código no funciona sin `useMemo`, primero encontrá y arreglá el problema de fondo."* Los únicos casos donde de verdad valen:
1. El valor/función se le pasa a un hijo envuelto en `React.memo`.
2. Es dependencia de otro hook.
3. (solo `useMemo`) el cálculo es **notablemente lento** y las deps cambian poco.

En cualquier otro caso, memoizar es ruido que ensucia el código sin ganar nada medible.

⚠️ **Y acá está la novedad que cambia el juego: el React Compiler.** Desde su **v1.0 estable (oct 2025)**, el compilador **memoiza automáticamente en tiempo de build** — y lo hace tan bien o mejor que vos a mano (puede memoizar hasta en casos donde `useMemo`/`useCallback` ni se pueden usar, como después de un `return` temprano). En la práctica: **con el compilador adoptado, escribir `useMemo`/`useCallback` a mano se vuelve mayormente innecesario.**

Dos aclaraciones importantes para no quedar mal en una entrevista:
- El compilador es **opt-in**: es un plugin de build, **no está prendido por defecto** en una app React 19 pelada. "Está disponible y estable", no "ya corre solo".
- En código nuevo con el compilador: confiá en él, y usá memo manual solo como escape hatch puntual. En código viejo: no saques la memoización existente sin testear (cambiás lo que el compilador genera).

**Regla de bolsillo para el lunes (2026):** ¿tu proyecto **no** tiene el React Compiler activado? → usá `useMemo`/`useCallback` **solo** en los 3 casos legítimos de arriba, nunca por las dudas. ¿Lo tiene activado? → **no escribas memo manual**, salvo un escape hatch puntual y medido. En los dos casos, la decisión por defecto es **menos memoización a mano, no más**.

La frase mental: **`useMemo` cachea un valor, `useCallback` una función — y existen por la identidad referencial. Pero son solo optimización: si no andás sin ellos, el bug está en otro lado. Y en 2026, el React Compiler los hace mayormente innecesarios… si lo activás.**

**Ejercicios 6**
6.1 🔁 ¿Qué cachea `useMemo` y qué cachea `useCallback`? Escribí la equivalencia entre los dos.
6.2 🧠 ¿Por qué envolver una función en `useCallback` "no sirve de nada" si el hijo NO está envuelto en `React.memo`?
6.3 🧠 En una entrevista te preguntan: "¿seguís usando `useMemo` y `useCallback` en 2026?". Respondé con criterio (mencioná el compilador y su estado).
6.4 ✍️ Tenés un padre con un `<Hijo onClick={fn} />`. Envolvé `Hijo` en `React.memo`, logueá su render, y mostrá con un `console.log` que **sin** `useCallback` el hijo re-renderiza cada vez que el padre cambia un estado no relacionado, y **con** `useCallback` no. (Sin el React Compiler activado.)

---

## Módulo 7 — Context API: el problema del prop drilling (y por qué NO es un state manager)

**Teoría.** Imaginá que el usuario logueado lo necesitás en el navbar, en el sidebar y en un botón enterrado 8 niveles abajo. Pasarlo por props por cada nivel intermedio —que ni lo usan— es el **prop drilling**: tedioso, frágil y ruidoso. **Context** resuelve eso: hace un valor disponible para todo un subárbol **sin pasarlo por props**.

Pensalo como el **sistema de altavoces de un edificio**: en vez de que cada piso le pase el mensaje al de abajo a mano (prop drilling), hay un anuncio central que **cualquier oficina puede escuchar** directamente.

Tres pasos:

```tsx
// 1) crear el contexto (con un valor por defecto)
const ThemeContext = createContext<'light' | 'dark'>('light')

// 2) proveer un valor a un subárbol
function App() {
  return (
    <ThemeContext value="dark">   {/* React 19: <Context> directo */}
      <Page />
    </ThemeContext>
  )
}

// 3) consumir el valor, sin importar cuán abajo estés
function Boton() {
  const theme = useContext(ThemeContext)  // busca el provider más cercano hacia ARRIBA
  return <button className={theme}>OK</button>
}
```

⚠️ **Dos novedades de React 19:**
- **`<Context>` como provider directo:** ya no hace falta `<ThemeContext.Provider>`, podés renderizar `<ThemeContext value=...>`. `.Provider` sigue funcionando pero está en camino de deprecación.
- **`use(Context)`:** además de `useContext`, podés leer un contexto con `const theme = use(ThemeContext)`. La gran diferencia: **`use` se puede llamar condicionalmente** (dentro de un `if`, un loop, después de un `return` temprano), mientras `useContext` debe ir siempre arriba.

Y ahora **lo más importante del módulo, lo que casi nadie te dice: Context NO es un state manager.** Es un mecanismo de **transporte** (evita el prop drilling), no de gestión de estado. ¿Por qué importa? Por cómo re-renderiza:

> **Cuando el valor de un Context cambia, React re-renderiza TODOS los consumidores de ese contexto.** Y `React.memo` **no los salva** — siguen recibiendo el valor nuevo y se redibujan igual.

Si metés todo tu estado de app en un solo Context, **cada cambio redibuja media app.** Dos patrones para sobrevivir:

1. **Memoizá el valor del provider.** Si pasás un objeto inline (`value={{ user, setUser }}`), creás un objeto nuevo en cada render (módulo 1) → todos los consumidores redibujan **siempre**, aunque el contenido no cambie. Memoizalo:
   ```tsx
   const value = useMemo(() => ({ user, setUser }), [user])
   return <UserContext value={value}>{children}</UserContext>
   ```
   (⚠️ con el React Compiler activado, esto puede ser automático — pero el patrón hay que entenderlo.)
2. **Partí los contextos por frecuencia de cambio.** El patrón clásico: `StateContext` + `DispatchContext` separados, así los que solo despachan acciones no redibujan cuando el estado cambia.

> **Nota — `useReducer` para estado con varias acciones.** Cuando el próximo estado depende del anterior y hay **varias formas de actualizarlo** (agregar / quitar / editar / reset), varios `useState` sueltos se vuelven difíciles de seguir. `useReducer(reducer, inicial)` centraliza esa lógica en una **función pura** `(estado, acción) => nuevoEstado` y te devuelve un `dispatch` de **identidad estable**. Por eso combina tan bien con el `DispatchContext` de arriba: el `dispatch` no cambia entre renders, así que los consumidores que **solo despachan** acciones no redibujan cuando el estado cambia.

La frase mental: **Context transporta un valor a un subárbol sin prop drilling — pero NO es un state manager: cuando el valor cambia, TODOS los consumidores redibujan y `memo` no los salva. Memoizá el value y partí los contextos.**

**Ejercicios 7**
7.1 🔁 ¿Qué problema resuelve Context y cuáles son los tres pasos (crear, proveer, consumir)?
7.2 🧠 ¿Por qué se dice que "Context no es un state manager"? ¿Qué pasa con los consumidores cuando el valor cambia?
7.3 🧠 Tenés `<UserContext value={{ user, setUser }}>`. ¿Por qué esto hace redibujar a TODOS los consumidores en cada render del provider, y cómo lo arreglás?
7.4 ✍️ Creá un `ThemeContext` tipado (`'light' | 'dark'`), un provider que lo ponga en `"dark"`, y un componente hijo profundo que lo consuma con `useContext`. Usá la sintaxis de React 19.
7.5 🧠 ¿Cuándo conviene `useReducer` en vez de varios `useState`, y por qué encaja con el patrón `StateContext` + `DispatchContext`?

---

## Módulo 8 — Custom hooks: compartir lógica, no estado

**Teoría.** Un **custom hook** es una función que empieza con `use` y que adentro **llama a otros hooks**. Sirve para **extraer y reutilizar lógica con estado** entre componentes, sin repetirte.

```tsx
function useOnlineStatus(): boolean {
  const [online, setOnline] = useState(navigator.onLine)
  useEffect(() => {
    const on = () => setOnline(true)
    const off = () => setOnline(false)
    window.addEventListener('online', on)
    window.addEventListener('offline', off)
    return () => {
      window.removeEventListener('online', on)
      window.removeEventListener('offline', off)
    }
  }, [])
  return online
}

// uso, en cualquier componente:
function Header() {
  const online = useOnlineStatus()
  return <span>{online ? '🟢 En línea' : '🔴 Sin conexión'}</span>
}
```

Dos cosas que tenés que tener clarísimas:

- **El nombre DEBE empezar con `use`** + mayúscula. No es estético: es lo que le permite al linter chequear las reglas de hooks adentro, y le avisa al lector que esta función puede tener estado/efectos.
- **Los custom hooks comparten LÓGICA, no ESTADO.** Esto es clave y se malentiende: *"cada llamada a un hook es completamente independiente de las demás."* Si dos componentes usan `useOnlineStatus()`, cada uno tiene su **propio** estado aislado. Si lo que querés es que **compartan el mismo estado**, un custom hook no alcanza — tenés que **levantar el estado** (lift state up) o usar Context (módulo 7).

La frase mental: **un custom hook (`use*`) extrae lógica con estado para reutilizarla — pero cada llamada tiene su estado propio. Comparte la receta, no el plato. Para compartir el plato, levantá el estado o usá Context.**

**Ejercicios 8**
8.1 🔁 ¿Por qué un custom hook tiene que empezar con `use`?
8.2 🧠 Dos componentes llaman `useCounter()`. ¿Comparten el contador o cada uno tiene el suyo? ¿Qué harías si quisieras que lo compartan?
8.3 ✍️ Escribí un custom hook `useLocalStorage<T>(key, initial)` que sincronice un estado con `localStorage` (leer al iniciar, escribir en cada cambio). Tipalo con genéricos.

---

## Módulo 9 — React 19: `use`, Actions y los hooks concurrentes

**Teoría.** React 19 sumó un set de hooks/APIs que vas a ver cada vez más, sobre todo en apps full-stack que hablan con tu API de Node. No hace falta que los domines todos hoy, pero **tenés que saber qué problema resuelve cada uno** (eso sí se pregunta).

⚠️ Todo esto es de React 19 — estable pero nuevo; confirmá firmas contra react.dev. Y para que no te abrume, los parto en **dos niveles**: lo que tenés que **entender** y lo que basta con **reconocer**.

### Nivel 1 — las que SÍ tenés que entender (caen en entrevista)

- **`use(resource)`** — la más versátil. Lee una **promesa** (se suspende hasta que resuelve, integrándose con `<Suspense>`) o un **contexto**, y —a diferencia de los demás hooks— **se puede llamar condicionalmente**. Ideal para leer datos de tu API durante el render sin el baile de `useEffect` + estado de loading.
  ```tsx
  function Comentarios({ promesa }: { promesa: Promise<Comentario[]> }) {
    const comentarios = use(promesa)  // suspende hasta resolver
    return <ul>{comentarios.map(c => <li key={c.id}>{c.texto}</li>)}</ul>
  }
  ```
  ⚠️ La promesa tiene que estar **cacheada/estable** (no la crees inline en el render); y para errores, usá un Error Boundary.

- **Actions + `useActionState`** — para **mutaciones asíncronas** (enviar un form a tu backend). Maneja por vos el estado de *pending*, los errores y el reset:
  ```tsx
  const [estado, formAction, isPending] = useActionState(
    async (prev: string | null, formData: FormData) => {
      const r = await guardarUsuario(formData)  // POST a tu API
      return r.ok ? null : 'Falló el guardado'  // lo que devolvés ES el próximo `estado`
    },
    null,
  )
  // <form action={formAction}> ... </form>   (el nombre de la 2ª posición es libre; la doc la llama formAction)
  ```
  Ojo con la tupla: la **primera posición** (`estado`) es **lo que devuelve tu action**, no "el error" por definición. Acá la usamos para el mensaje de error porque eso devolvemos, pero podría ser cualquier cosa (el dato guardado, un objeto, etc.).

- **`useTransition`** → `[isPending, startTransition]`: marca actualizaciones como **baja prioridad e interrumpibles**, para que la UI no se trabe (ej. filtrar una lista gigante mientras se tipea). ⚠️ En React 19 los callbacks dentro de `startTransition` pueden ser async (Actions); ojo que un `setState` **después de un `await`** hay que re-envolverlo en `startTransition`.

### Nivel 2 — las que basta con saber que existen

No las estudies hoy; reconocé el problema que resuelven y andá a la doc cuando las necesites.

- **`useOptimistic`** — UI optimista: mostrás el resultado esperado **al instante** mientras la mutación viaja, y React revierte solo si falla. (El "me gusta" que se pinta antes de que el server confirme.)
- **`useFormStatus`** (de `react-dom`) — deja que un botón de tu design system lea el estado del `<form>` padre (`pending`, etc.) **sin prop drilling**, como si el form fuera un Context.
- **`useDeferredValue`** — muestra una versión "vieja" de una parte lenta de la UI mientras se recalcula, para mantener la interacción fluida.
- **`useId`** — genera IDs únicos y estables (server y cliente) para atributos de accesibilidad. **No** es para keys de listas ni para cachés.

### El dúo de carga/error: `<Suspense>` + Error Boundary

Cuando un componente usa `use(promesa)` y se **suspende**, React necesita saber qué mostrar mientras carga y qué hacer si la promesa **falla**. Eso lo dan dos envoltorios:
- **`<Suspense fallback={...}>`** muestra el fallback mientras el contenido suspendido carga.
- **Un Error Boundary** (un componente de clase con `getDerivedStateFromError`/`componentDidCatch`, o la librería `react-error-boundary`) captura el error de su subárbol y muestra una UI alternativa.

```tsx
<ErrorBoundary fallback={<Error />}>
  <Suspense fallback={<Spinner />}>
    <Comentarios promesa={promesa} />   {/* hace use(promesa) adentro */}
  </Suspense>
</ErrorBoundary>
```

Es el patrón de carga/error **canónico de 2026**: en vez de manejar `isLoading`/`error` a mano en cada componente, declarás el estado de carga y el de error **una sola vez** alrededor del subárbol. (Error Boundary sigue requiriendo un componente de **clase** — es de los pocos lugares donde React todavía no tiene equivalente con hooks.)

La frase mental: **React 19 te da `use` (leer promesas/contexto, hasta condicionalmente), los Actions (`useActionState`/`useOptimistic`/`useFormStatus`, que matan el boilerplate de pending/error en formularios) y los concurrentes (`useTransition`/`useDeferredValue`) para que la UI no se trabe.**

**Ejercicios 9**
9.1 🔁 ¿Qué hace `use` que ningún otro hook puede, respecto a dónde se lo puede llamar?
9.2 🧠 ¿Qué problema concreto te resuelve `useActionState` al enviar un formulario a tu API de Node? ¿Qué tenías que hacer a mano antes?
9.3 🧠 ¿Para qué sirve `useTransition` y qué tipo de actualización marca?
9.4 ✍️ Escribí un form con `useActionState`: una action async que simule el POST, un botón deshabilitado con `isPending` mientras envía, y un mensaje de error si la action devuelve uno. Conectalo con `<form action={formAction}>`.

---

## Módulo 10 — El criterio: estado local vs Context vs URL vs state manager

**Teoría.** El cierre, y el módulo más importante: **dónde poner el estado.** Elegir mal es la causa #1 de apps React enredadas. La regla maestra: **poné el estado lo más cerca posible de donde se usa, y subilo solo cuando haga falta compartirlo.**

Una escalera, de menos a más "global" (preferí siempre el escalón más bajo que te sirva):

1. **Estado local (`useState` en el componente).** El default. Si solo este componente lo usa, acá va. No lo subas "por si acaso".
2. **Estado levantado (lift state up).** Si dos hermanos necesitan el mismo estado, subilo al **padre común más cercano** y bajalo por props. Antes de pensar en nada más global, probá esto.
3. **La URL.** Para estado que define "qué estás viendo" —filtros, página, tab, id del recurso, búsqueda—. Es compartible, bookmarkeable, sobrevive al refresh y la maneja el router. **Subestimadísima.** Mucho "estado global" en realidad es estado de URL mal puesto.
4. **Context.** Para datos **transversales** que muchos componentes leen pero que **cambian poco**: tema, idioma, usuario logueado. Recordá el módulo 7: **no es un state manager**, redibuja a todos los consumidores. Úsalo para "ambiente", no para estado de dominio que cambia seguido.
5. **State manager (Zustand, Redux, Jotai…).** El último recurso, para estado de cliente complejo, compartido y que cambia seguido, donde Context se quedaría corto por performance. Si no te duele eso, no lo metas.

⚠️ Y un apartado para 2026: con **React Server Components** y data fetching en el server (módulos [RSC](rsc.md) y [TanStack Start](tanstack-start.md)), una parte enorme de lo que antes vivía en un state manager del cliente —el **estado del servidor** (datos de tu API)— ya no necesita estar ahí. Mucho del "estado global" clásico era cache de datos remotos; eso hoy se resuelve con server components o librerías de data fetching (TanStack Query), no con Redux.

Los gotchas de entrevista, todos juntos para que los tengas frescos:
- **Stale closure en un `setInterval`:** el callback de un efecto con `[]` congela la foto del primer render → usá `setX(prev => ...)`.
- **Mentirle al array de dependencias** (silenciar el linter) es el bug clásico: las deps no se eligen, se descubren.
- **Objetos/funciones inline en deps** sin memoizar → referencia nueva cada render → el efecto/memo dispara siempre.
- **Estado derivado en un efecto** → calculalo en el render.
- **Context que redibuja todo** → memoizá el value, partí los contextos.

La frase mental de cierre: **el estado va lo más abajo y local posible; subilo solo cuando se comparte. URL para "qué ves", Context para "ambiente que cambia poco", state manager solo si duele de verdad. Y en 2026, gran parte del viejo estado global era data del servidor — eso vive en el server, no en Redux.**

**Ejercicios 10**
10.1 🔁 Enumerá la escalera del estado de menos a más global y decí cuál es el default.
10.2 🧠 Tenés filtros de un listado (categoría, orden, página) y querés que el usuario pueda compartir el link con esos filtros aplicados. ¿Dónde va ese estado y por qué?
10.3 🧠 ¿Por qué Context es buena idea para "usuario logueado" o "tema" pero mala para el estado de un formulario que cambia en cada tecla?
10.4 ✍️ Escribí un componente que sincronice un filtro con la URL: leé `?categoria=...` de los query params al renderizar y actualizalo al cambiar un `<select>`. Usá `URLSearchParams` (o, si tenés router, su API de search params).

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** Los tres pasos son **Trigger → Render → Commit**. React "llama a tu componente" en el paso **Render** (ejecuta tu función para ver qué JSX devuelve). En Commit recién toca el DOM.

**1.2** Da `false` porque `{ id: 1 }` crea un **objeto nuevo en memoria** cada vez; `Object.is` compara **identidad** (la misma referencia), no contenido. Dos objetos distintos con el mismo contenido no son `===`. Importa porque React usa `Object.is` para comparar dependencias de efectos, props de `React.memo` y valores de Context: una referencia nueva (aunque el contenido sea igual) hace que React crea que "cambió" y dispare trabajo de más.

**1.3** Que dado el mismo estado/props, debe devolver el mismo JSX y **no producir efectos secundarios** durante la ejecución. Ejemplos de lo que NO va en el render: mutar una variable externa, llamar a una API, escribir en `localStorage`, mutar props/estado, leer/escribir `ref.current`. Eso va en handlers o efectos.

**1.4**
```tsx
import { useRef, useState } from 'react'

function IdentidadReferencial() {
  const obj = { id: 1 }                       // objeto NUEVO cada render
  const anterior = useRef<{ id: number } | null>(null)
  const [, forzar] = useState(0)

  console.log('¿mismo objeto que el render anterior?', obj === anterior.current)
  anterior.current = obj                       // guardamos el de este render para comparar al próximo

  return <button onClick={() => forzar(n => n + 1)}>Re-renderizar</button>
}
// Consola: siempre `false` (salvo el primer render, que compara contra null).
// Mismo contenido { id: 1 }, distinta identidad: esa es la causa raíz de medio módulo.
```

**1.5** `key={index}` ata la identidad de cada fila a su **posición**, no al dato. Al reordenar o insertar al principio, los índices se corren: React cree que la fila 0 "sigue siendo" la misma y **reutiliza su estado/DOM** (el valor de un `<input>`, el foco, el scroll) para un dato distinto → el estado "salta" a la fila equivocada. Usá un **id estable del dato** (`key={item.id}`); el índice solo es seguro si la lista es estática (nunca se reordena ni se filtra).



**2.1** (1) Solo llamar hooks en el **nivel superior** (nada de `if`, loops, funciones anidadas, `try/catch`; antes de cualquier `return`). (2) Solo llamarlos desde **funciones de React** (componentes o custom hooks), nunca desde funciones JS comunes.

**2.2** Porque React identifica cada hook por el **orden de llamada**, no por su nombre: mantiene una lista interna ("hook #1, hook #2…") y asocia cada estado a una posición. Si un hook está dentro de un `if`, en algunos renders se llama y en otros no, y el orden se corre → React le entrega a un hook el estado que pertenecía a otro. Orden inestable = estado equivocado.

**2.3** El `useEffect` está **después de un `return` temprano** (`if (loading) return <Spinner/>`), así que en los renders con `loading === true` el hook **no se llama** → viola la regla 1 (orden inestable). Arreglo: poné **todos los hooks arriba**, antes de cualquier `return`, y recién después hacé el `return` condicional.
```tsx
function Vista({ loading }: { loading: boolean }) {
  useEffect(() => { /* ... */ }, [])  // primero los hooks
  if (loading) return <Spinner />      // después los returns
  return <Contenido />
}
```

**3.1** Porque las tres llamadas leen el **mismo `count`** (la foto de ESE render): `count` vale, digamos, 0, así que las tres calculan `0 + 1 = 1`. React no actualiza `count` en medio del handler. Se arregla con la forma funcional: `setCount(prev => prev + 1)` tres veces → cada una recibe el valor más reciente de la cola → 0→1→2→3.

**3.2** `useState(crearInicial())` **ejecuta** `crearInicial` en **cada render** y descarta el resultado después del primero (desperdicio). `useState(crearInicial)` pasa la **función** y React solo la ejecuta en el **primer render** (lazy initial state). Importa cuando ese cálculo inicial es **caro** (leer `localStorage`, procesar una lista grande).

**3.3** No redibuja porque mutás el **mismo objeto** (`user.age = 31`) y luego le pasás **la misma referencia** a `setUser`: `Object.is(user, user)` es `true`, así que React no detecta cambio. Bien hecho: creás un objeto **nuevo**:
```tsx
setUser({ ...user, age: 31 })
// o con functional update: setUser(prev => ({ ...prev, age: 31 }))
```

**3.4**
```tsx
import { useState } from 'react'

function Contador() {
  const [count, setCount] = useState(0)
  // Sin la forma funcional, los tres setCount leerían el mismo `count` (la foto del render)
  // y calcularían el mismo valor → sumaría 1. Con `prev =>`, cada uno ve el más reciente.
  const masTres = () => {
    setCount(prev => prev + 1)
    setCount(prev => prev + 1)
    setCount(prev => prev + 1)
  }
  return <button onClick={masTres}>Count: {count} (+3)</button>
}
```

**3.5** **Controlado:** el valor vive en estado de React (`value={x}` + `onChange`), React es la fuente de verdad y redibuja en cada tecla → podés validar, formatear o deshabilitar al vuelo. **No controlado:** el valor vive en el DOM y lo leés con un ref cuando hace falta (`defaultValue` + `ref`) → menos re-renders, pero React no ve cada cambio. Default: **controlado** (validación en vivo, UI que reacciona al input). No controlado: forms simples de una sola lectura, integración con DOM/librerías no-React, y `<input type="file">` (que es no controlado por obligación).

**4.1** **Sirve** para **sincronizar el componente con un sistema externo a React** (suscripciones, conexiones, APIs del navegador, librerías de terceros, DOM no-React) — efectos que ocurren *porque el componente se renderizó*. **NO sirve** como "ciclo de vida": no va para transformar datos para renderizar (eso se calcula en el render) ni para lógica que pertenece a un evento del usuario (eso va en el handler).

**4.2** Está mal porque creás **estado redundante** y un **render extra**: el efecto corre *después* de pintar, setea el estado y dispara otro render. El nombre completo es **derivable** de `first`/`last`, así que se calcula durante el render: `const fullName = first + ' ' + last`. Sin estado, sin efecto, sin render de más.

**4.3** React te está diciendo, vía el doble-render de Strict Mode en desarrollo, que **te falta cleanup**: tu efecto abre una conexión pero no la cierra, así que el setup→cleanup→setup deja dos abiertas. El arreglo **correcto** es **devolver una función de cleanup** que cierre la conexión (no un `if (yaCorrió)` que tapa el síntoma):
```tsx
useEffect(() => {
  const conn = crearConexion(url)
  conn.connect()
  return () => conn.disconnect()  // ✅ cleanup
}, [url])
```

**4.4**
```tsx
import { useEffect, useState } from 'react'

function useEstadoConexion() {
  const [online, setOnline] = useState(navigator.onLine)
  useEffect(() => {
    const on = () => setOnline(true)
    const off = () => setOnline(false)
    window.addEventListener('online', on)
    window.addEventListener('offline', off)
    return () => {                            // cleanup: saca los listeners
      window.removeEventListener('online', on)
      window.removeEventListener('offline', off)
    }
  }, [])  // [] → suscribir al montar, desuscribir al desmontar; nada reactivo se lee adentro
  return online
}
```

**5.1** `useState` guarda un valor que **se dibuja**, y cambiarlo **dispara un re-render**. `useRef` guarda un valor mutable que **persiste entre renders** pero cuya mutación **NO dispara re-render**. Estado = lo que se ve; ref = lo que recordás sin pintar.

**5.2** **Ref.** El `id` del intervalo es plomería interna: no se muestra en pantalla y no querés que cambiarlo dispare un render. `useState` redibujaría de gusto. `const idRef = useRef<number | null>(null)` y guardás ahí.

**5.3**
```tsx
import { useRef } from 'react'

function CampoEnfocable() {
  const inputRef = useRef<HTMLInputElement>(null)
  return (
    <>
      <input ref={inputRef} placeholder="Escribí algo" />
      <button onClick={() => inputRef.current?.focus()}>Enfocar</button>
    </>
  )
}
```

**6.1** `useMemo` cachea el **resultado de un cálculo**; `useCallback` cachea la **función misma**. Equivalencia: `useCallback(fn, deps)` es lo mismo que `useMemo(() => fn, deps)`.

**6.2** Porque el único efecto de `useCallback` es **mantener estable la referencia** de la función entre renders. Si el hijo no está envuelto en `React.memo`, **igual se re-renderiza** cuando el padre se renderiza (sin importar las props), así que estabilizar la referencia no evita nada. El beneficio aparece cuando el hijo es `memo` (y entonces una referencia estable evita su re-render) o cuando la función es dependencia de otro hook.

**6.3** Respuesta con criterio: "Conceptualmente sí, pero en la práctica cada vez menos. Existen por la identidad referencial —estabilizar valores/funciones para `React.memo` o para deps de otros hooks—, y son **solo optimización**: si el código no anda sin ellos, el problema está en otro lado. Desde el **React Compiler v1.0 (estable, oct 2025)**, que memoiza automáticamente en build, escribirlos a mano se vuelve mayormente innecesario — **pero el compilador es opt-in**, no está prendido por defecto, así que en un proyecto sin compilador todavía los uso para los casos legítimos." ⚠️ (Verificá el estado del compilador al momento de responder.)

**6.4**
```tsx
import { memo, useCallback, useState } from 'react'

const Hijo = memo(function Hijo({ onClick }: { onClick: () => void }) {
  console.log('Hijo renderizó')
  return <button onClick={onClick}>Hijo</button>
})

function Padre() {
  const [n, setN] = useState(0)            // estado NO relacionado con el hijo

  // ❌ Sin useCallback: `fn` es nueva cada render → memo ve "prop nueva" → Hijo renderiza siempre
  // const fn = () => console.log('click')

  // ✅ Con useCallback: misma referencia mientras deps no cambien → memo frena el re-render
  const fn = useCallback(() => console.log('click'), [])

  return (
    <>
      <button onClick={() => setN(n + 1)}>Padre: {n}</button>
      <Hijo onClick={fn} />
    </>
  )
}
// Con `fn` inline (sin useCallback), "Hijo renderizó" sale en cada click del padre.
// Con useCallback, sale UNA sola vez: memo + referencia estable evitan el re-render.
```
(Con el React Compiler activado, esto sería automático y no haría falta el `useCallback` manual.)



**7.1** Resuelve el **prop drilling**: pasar un valor por props a través de muchos niveles intermedios que no lo usan. Tres pasos: **crear** (`const Ctx = createContext(default)`), **proveer** (`<Ctx value={...}>...</Ctx>`), **consumir** (`const v = useContext(Ctx)`, que busca el provider más cercano hacia arriba).

**7.2** Porque Context es un mecanismo de **transporte**, no de gestión de estado: no optimiza re-renders. Cuando el valor del provider cambia, **React re-renderiza TODOS los componentes que consumen ese contexto**, desde el provider hacia abajo — y `React.memo` **no los frena** (reciben el valor fresco igual). Por eso meter estado que cambia seguido en un Context grande mata la performance.

**7.3** Porque `{{ user, setUser }}` es un **objeto inline**: se crea **nuevo en cada render** del provider (identidad referencial, módulo 1). Aunque `user` no haya cambiado, el `value` es una referencia distinta → todos los consumidores ven "valor nuevo" y redibujan. Arreglo: memoizar el value.
```tsx
const value = useMemo(() => ({ user, setUser }), [user])
return <UserContext value={value}>{children}</UserContext>
```

**7.4**
```tsx
import { createContext, useContext } from 'react'

const ThemeContext = createContext<'light' | 'dark'>('light')

function App() {
  return (
    <ThemeContext value="dark">   {/* React 19: provider directo */}
      <Layout />
    </ThemeContext>
  )
}

function Layout() { return <Boton /> }  // nivel intermedio: no toca el tema

function Boton() {
  const theme = useContext(ThemeContext) // lo consume directo, sin props
  return <button className={theme}>Tema: {theme}</button>
}
```

**7.5** Conviene `useReducer` cuando el estado tiene **varias transiciones** (agregar/quitar/editar/reset) y el próximo valor depende del anterior: en vez de esparcir esa lógica en muchos `setX`, la centralizás en una función pura `(estado, acción) => nuevoEstado`, más fácil de testear y razonar. Encaja con `StateContext` + `DispatchContext` porque `dispatch` tiene **identidad estable** (no cambia entre renders): podés pasarlo por el `DispatchContext` y los componentes que solo despachan acciones **no redibujan** cuando el estado cambia (solo redibujan los que consumen el `StateContext`).

**8.1** Porque el prefijo `use` + mayúscula es la **convención que React y el linter reconocen**: le permite a `eslint-plugin-react-hooks` aplicar las reglas de hooks dentro de la función, y le señala a quien lee que esa función puede llamar hooks (tener estado/efectos). Sin el `use`, el linter no lo trata como hook.

**8.2** Cada uno tiene **el suyo**: cada llamada a un hook (incluido un custom hook) es **completamente independiente**, con su propio estado aislado. Si querés que lo **compartan**, un custom hook no alcanza — tenés que **levantar el estado** al padre común y bajarlo por props, o ponerlo en un **Context**.

**8.3**
```tsx
import { useState, useEffect } from 'react'

function useLocalStorage<T>(key: string, initial: T): [T, (v: T) => void] {
  const [value, setValue] = useState<T>(() => {        // lazy init: lee storage 1 sola vez
    const raw = localStorage.getItem(key)
    return raw !== null ? (JSON.parse(raw) as T) : initial
  })
  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value))   // sincroniza en cada cambio
  }, [key, value])
  return [value, setValue]
}
```
Notá: la lectura va en *lazy initial state* (no en cada render) y la escritura en un efecto con `[key, value]` como deps.

**9.1** `use` se puede llamar **condicionalmente** — dentro de un `if`, un loop o después de un `return` temprano —, mientras que **todos los demás hooks** (incluido `useContext`) deben llamarse en el nivel superior, siempre, en el mismo orden. Además, `use` puede leer una **promesa** (suspende con `<Suspense>`), no solo un contexto.

**9.2** `useActionState` te maneja **automáticamente el estado de *pending*, los errores y el reset** de una mutación asíncrona (enviar el form, esperar el POST a tu API). Antes lo hacías **a mano**: un `useState` para `isLoading`, otro para `error`, un `try/catch/finally`, deshabilitar el botón mientras carga, resetear al terminar… todo ese boilerplate lo absorbe el hook, y se conecta con `<form action={formAction}>`.

**9.3** `useTransition` marca actualizaciones de estado como **de baja prioridad e interrumpibles**, para que una actualización pesada (re-renderizar una lista enorme al filtrar) **no bloquee** interacciones urgentes como tipear. Devuelve `[isPending, startTransition]`; envolvés la actualización no urgente en `startTransition(...)` y usás `isPending` para mostrar un indicador.

**9.4**
```tsx
import { useActionState } from 'react'

async function fakePost(nombre: string): Promise<{ ok: boolean }> {
  await new Promise((r) => setTimeout(r, 800))
  return { ok: nombre.trim().length > 0 }   // "falla" si está vacío
}

function FormNombre() {
  const [error, formAction, isPending] = useActionState(
    async (_prev: string | null, formData: FormData) => {
      const nombre = String(formData.get('nombre') ?? '')
      const r = await fakePost(nombre)
      return r.ok ? null : 'El nombre no puede estar vacío'  // null = sin error
    },
    null,
  )
  return (
    <form action={formAction}>
      <input name="nombre" />
      <button disabled={isPending}>{isPending ? 'Enviando…' : 'Guardar'}</button>
      {error && <p role="alert">{error}</p>}
    </form>
  )
}
// useActionState maneja el pending, el error y el reset; no hace falta useState para isLoading/error.
```

**10.1** De menos a más global: **(1) estado local (`useState`)** ← el default → **(2) estado levantado al padre común** → **(3) la URL** → **(4) Context** → **(5) state manager**. La regla: usá el escalón más bajo que te resuelva el problema; subí solo cuando necesites compartir.

**10.2** Va en **la URL** (query params: `?categoria=...&orden=...&page=...`). Porque es estado que define "qué estás viendo", y ponerlo en la URL lo hace **compartible por link**, bookmarkeable y resistente al refresh — justo lo que pide el requisito. Meterlo en `useState` o en un store lo perdería al recargar y no se podría compartir.

**10.3** Porque "usuario logueado" y "tema" son datos **transversales que cambian muy poco**: encajan con el modelo de Context (transporte a muchos consumidores, pocos cambios). El estado de un formulario que cambia **en cada tecla** haría que **todos los consumidores del Context redibujen en cada tecla** (módulo 7) — un desastre de performance. Ese estado va **local** al formulario (o a un state manager si es muy complejo), no en Context.

**10.4**
```tsx
// Versión con la API del navegador (sin router). La fuente de verdad es la URL, no useState.
function FiltroCategoria() {
  const params = new URLSearchParams(window.location.search)
  const categoria = params.get('categoria') ?? 'todas'

  function cambiar(e: React.ChangeEvent<HTMLSelectElement>) {
    const next = new URLSearchParams(window.location.search)
    next.set('categoria', e.target.value)
    // actualiza la URL sin recargar; en una app real usás el router (TanStack/React Router)
    window.history.pushState(null, '', `?${next.toString()}`)
  }

  return (
    <select value={categoria} onChange={cambiar}>
      <option value="todas">Todas</option>
      <option value="libros">Libros</option>
      <option value="juegos">Juegos</option>
    </select>
  )
}
// Con un router (recomendado): leés/escribís los search params con su API (useSearchParams),
// que además dispara el re-render al cambiar la URL — con history.pushState lo manejás vos.
```

---

> **Para seguir.** La fuente de verdad de este módulo es la documentación oficial: [react.dev](https://react.dev) — en particular [Rules of Hooks](https://react.dev/reference/rules/rules-of-hooks), [Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects), [You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect), [useContext](https://react.dev/reference/react/useContext), el [blog de React 19](https://react.dev/blog/2024/12/05/react-19) y el [React Compiler v1](https://react.dev/blog/2025/10/07/react-compiler-1). Antes de copiar lo marcado con ⚠️ (estado del React Compiler, `forwardRef` y `<Context.Provider>` en camino de deprecación, las firmas de `use`/Actions), re-verificá en react.dev: algunas APIs todavía funcionan pero están en transición. El puente natural desde acá: [TanStack Start](tanstack-start.md) y [React Server Components](rsc.md) para llevar estos hooks al mundo full-stack, y [Microfrontends](microfrontends.md) para la cara arquitectónica.
