# Proyecto: Mi Hub Full Stack

Sitio personal de estudio para documentar mi transición de **Frontend (React / React Native) → Full Stack con Node.js**. Reúne apuntes, teoría y ejercicios prácticos.

## Qué es técnicamente

- Es un sitio **Docsify**: **no hay paso de build**. GitHub Pages sirve los archivos tal cual y Docsify renderiza el Markdown en el navegador.
- Se publica desde la rama `main`, carpeta raíz `/`.
- URL en producción: https://jzamituy.github.io/fullstack-roadmap/

## Estructura de archivos

- `index.html` — cargador de Docsify y configuración (plugins, tema). Rara vez se toca.
- `_sidebar.md` — el menú lateral / navegación. **Se actualiza cada vez que se agrega contenido.**
- `README.md` — página de inicio del sitio.
- `.nojekyll` — archivo vacío **obligatorio**: evita que GitHub Pages ignore archivos que empiezan con `_` (como `_sidebar.md`). No borrar.
- `*.md` en la raíz — cada archivo es una página de contenido (ej. `roadmap.md`, `typescript.md`).

## Cómo agregar un módulo / tema nuevo (tarea más frecuente)

1. Crear un archivo `<tema>.md` en la raíz del repo (ej. `nestjs.md`, `postgresql.md`).
2. Agregar una línea con el enlace en `_sidebar.md`, bajo la sección que corresponda (o creando una sección nueva).
3. Hacer commit y push (ver abajo).

## Convenciones de contenido

- **Idioma:** español.
- **Estructura de cada página de teoría+práctica:** teoría breve por módulo → ejercicios → **soluciones agrupadas al final** (no junto a cada ejercicio, para no espiar).
- **Progresión:** de lo básico a lo avanzado; ejemplos que arrancan neutros y derivan hacia backend/Node.
- **Código:** bloques con lenguaje explícito (` ```ts `, ` ```bash `, ` ```sql `). Para TypeScript, el código de las soluciones debe **compilar en modo `--strict`**; verificar si hay dudas.
- **Formato:** Markdown limpio, encabezados jerárquicos, sin HTML salvo que sea necesario.

## Flujo de publicación

Después de editar, publicar con:

```bash
git add -A
git commit -m "<mensaje conciso en español>"
git push
```

El remoto es SSH (`git@github.com:jzamituy/fullstack-roadmap.git`). El push a `main` despliega automáticamente vía GitHub Pages en ~1-2 minutos.

## Convenciones de commits

- Mensajes concisos, en español, en modo imperativo (ej. "Agrego módulo de NestJS", "Corrijo ejercicio 3 de TypeScript").
- Un commit por unidad lógica de cambio.
- Sin trailers de coautoría ni líneas de "generado por"; los commits van a nombre del autor del repo.

## Previsualizar local (opcional)

Servir la carpeta con cualquier servidor estático y abrir en el navegador:

```bash
npx serve .
```
