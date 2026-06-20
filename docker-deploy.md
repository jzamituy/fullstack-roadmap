# Docker, deploy y CI/CD: llevar tu backend a producción

**De "en mi máquina funciona" a una URL pública con CI verde · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Ya tenés una "Task API" con base, auth y tests. Falta lo que la convierte en un producto: empaquetarla para que corra **igual en cualquier lado** (Docker), automatizar que cada cambio se valide solo (CI/CD) y dejarla **desplegada con una URL pública**. Esto no es opcional para tu portfolio: el roadmap exige que cada proyecto esté "desplegado, con README, tests y diagrama". Un proyecto que no se puede correr ni ver vale la mitad en una entrevista.

**Por qué importa.** Docker es **must-have** en 2026 (aparece en casi toda oferta). Y un pipeline de CI/CD que corre tus tests (el módulo anterior) antes de mergear es lo que hace que "tener tests" sirva de verdad: nadie puede romper `main` sin que el sistema lo frene.

**Para practicar.** Necesitás Docker instalado (`docker -v`, `docker compose version`) y una cuenta gratis en una plataforma de deploy (Railway, Render o Fly.io). El repo ya en GitHub (para Actions).

**Índice de módulos**
1. Por qué Docker: el fin de "en mi máquina funciona"
2. Imagen vs. contenedor; un Dockerfile básico para Node
3. Dockerfile multi-stage: imágenes livianas y seguras
4. `.dockerignore`, capas y caché de build
5. Variables de entorno y secretos en contenedores
6. `docker compose`: app + Postgres + Redis en una línea
7. Healthchecks y graceful shutdown en contenedores
8. CI/CD: qué es un pipeline y por qué cambia todo
9. GitHub Actions: un workflow real (lint → test → build)
10. Deploy real: plataforma, DB gestionada y migraciones
11. Logs y observabilidad básica en producción

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Por qué Docker: el fin de "en mi máquina funciona"

**Teoría.** El problema clásico: tu app corre en tu máquina pero falla en la del compañero o en producción, porque tienen otra versión de Node, otra librería del sistema, otra config. **Docker** lo resuelve empaquetando tu app **junto con todo lo que necesita** (runtime, dependencias, config) en una **imagen** reproducible que corre idéntica en cualquier lado.

La diferencia con una máquina virtual: una VM virtualiza un sistema operativo entero (pesado, lento de arrancar). Un **contenedor** comparte el kernel del host y aísla solo lo necesario (liviano, arranca en segundos). Por eso podés correr decenas de contenedores donde correrías pocas VMs.

Tres conceptos que no hay que confundir:

- **Imagen**: la "foto" inmutable de tu app empaquetada (el molde). Se construye una vez.
- **Contenedor**: una instancia en ejecución de una imagen (lo que sale del molde). De una imagen levantás muchos contenedores idénticos — justo lo que necesita el **escalado horizontal** del módulo de Node.
- **Dockerfile**: la receta de texto que describe cómo construir la imagen.

La ventaja para vos: el mismo artefacto que probaste localmente es el que corre en producción. Se acabó el "andá a saber qué tiene distinto el server".

**Ejercicios 1**
1.1 En una frase: ¿qué problema concreto resuelve Docker?
1.2 Diferenciá imagen y contenedor con la analogía del molde.
1.3 ¿Por qué un contenedor es más liviano que una máquina virtual?

---

## Módulo 2 — Imagen vs. contenedor; un Dockerfile básico para Node

**Teoría.** Un `Dockerfile` es una secuencia de instrucciones que construyen la imagen, capa por capa. Las más usadas:

- `FROM` — la imagen base (ej. `node:22-alpine`, una versión chica de Node).
- `WORKDIR` — el directorio de trabajo dentro del contenedor.
- `COPY` — copia archivos de tu proyecto a la imagen.
- `RUN` — ejecuta un comando al **construir** (ej. `npm ci`, `npm run build`).
- `ENV` — variables de entorno.
- `EXPOSE` — documenta el puerto que usa la app.
- `CMD` — el comando que se ejecuta al **arrancar** el contenedor.

Una primera versión (ingenua, la mejoramos en el módulo 3):

```dockerfile
FROM node:22-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

La distinción clave: `RUN` corre **durante el build** (queda "horneado" en la imagen); `CMD` corre **cuando arrancás** un contenedor de esa imagen. `RUN npm run build` compila tu TypeScript a `dist/` al construir; `CMD ["node", "dist/main.js"]` arranca la app al correr.

Para construir y correr:

```bash
docker build -t task-api .          # construye la imagen y la etiqueta "task-api"
docker run -p 3000:3000 task-api    # corre un contenedor, mapeando el puerto 3000
```

`-p 3000:3000` mapea el puerto del host al del contenedor: sin eso, la app corre aislada y no le llegás desde afuera.

**Ejercicios 2**
2.1 ¿Cuál es la diferencia entre `RUN` y `CMD` en un Dockerfile? Dá un ejemplo de cada uno para una app Node.
2.2 ¿Para qué sirve el `-p 3000:3000` al correr el contenedor? ¿Qué pasa si lo omitís?
2.3 ¿Por qué se copia `package*.json` y se hace `npm ci` **antes** de copiar el resto del código? (Pensalo; lo retomamos en el módulo 4.)

---

## Módulo 3 — Dockerfile multi-stage: imágenes livianas y seguras

**Teoría.** La imagen del módulo 2 tiene un problema: incluye **todo** lo del build (el código fuente TypeScript, las devDependencies, el compilador) que **no** necesitás para *correr* la app. Eso la hace grande, lenta de transferir y con más superficie de ataque.

La solución es el **multi-stage build**: una etapa **construye** (con todas las herramientas) y otra, limpia, **solo corre** lo necesario, copiando de la primera apenas el resultado:

```dockerfile
# Etapa 1: build (tiene devDependencies, compila el TS)
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Etapa 2: runtime (liviana: solo dependencias de producción + el dist compilado)
FROM node:22-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
RUN apk add --no-cache tini    # init mínimo: reenvía señales y cosecha zombies (módulo 7)
COPY package*.json ./
RUN npm ci --omit=dev          # solo dependencies, sin devDependencies
COPY --from=build /app/dist ./dist   # traemos SOLO el compilado de la etapa build
USER node                      # no corras como root: principio de mínimo privilegio
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:3000/health || exit 1
ENTRYPOINT ["/sbin/tini", "--"]   # tini queda como PID 1 y delega en node
CMD ["node", "dist/main.js"]      # forma "exec" (array): node recibe las señales directo
```

Lo que ganás:

- **Imagen más chica**: la final no tiene el código fuente, ni el compilador, ni las devDependencies (Jest, ESLint, etc.). Menos peso = deploys más rápidos.
- **Más segura**: menos cosas instaladas = menos vulnerabilidades posibles. Y `USER node` evita correr como `root` (si comprometen el contenedor, el atacante no es superusuario) — el **mínimo privilegio** del archivo senior, aplicado al contenedor.

**Elegir la imagen base.** `alpine` es chiquísima pero usa **musl libc** (no glibc), lo que a veces rompe dependencias nativas o da sorpresas con DNS/compilaciones. Las alternativas:

- **`node:lts-alpine`** — mínima; ideal si no tenés dependencias nativas problemáticas.
- **`node:lts-slim`** — Debian recortado (glibc); un poco más grande, pero **máxima compatibilidad** con módulos nativos.
- **`gcr.io/distroless/nodejs`** — sin shell ni gestor de paquetes (superficie de ataque mínima), no-root por defecto; la más segura, pero difícil de debuggear (no hay shell adentro).

Y **fijá la versión**: `node:22-alpine` pinea el *major* (mejor que `node:latest`, que es un blanco móvil); `node:lts-alpine` sigue la LTS vigente; y para reproducibilidad total se pinea por **digest** (`node:22-alpine@sha256:...`). Revisá que el major que elijas sea una **LTS vigente** en 2026.

Este es el patrón estándar para Node en producción. Si en una entrevista te preguntan por tu Dockerfile, "multi-stage, usuario no-root, imagen liviana pinada y un init (`tini`) para las señales" es la respuesta que demuestra criterio.

**Ejercicios 3**
3.1 ¿Qué problema del Dockerfile de una sola etapa resuelve el multi-stage?
3.2 ¿Por qué la etapa de runtime usa `npm ci --omit=dev`? ¿Qué deja afuera y por qué está bien?
3.3 ¿Por qué conviene `USER node` en vez de correr como root? Conectá con un principio del archivo senior.

---

## Módulo 4 — `.dockerignore`, capas y caché de build

**Teoría.** Docker construye la imagen en **capas**: cada instrucción del Dockerfile genera una capa, y Docker **cachea** las que no cambiaron. Si entendés esto, tus builds pasan de minutos a segundos.

La clave es el **orden**. Docker reusa la caché de una capa solo si esa instrucción **y todas las anteriores** no cambiaron. Por eso copiás primero `package*.json` y hacés `npm ci`, y **después** copiás el resto del código:

```dockerfile
COPY package*.json ./
RUN npm ci          # esta capa (lenta) se cachea: solo se re-ejecuta si cambió package.json
COPY . .            # tu código cambia seguido, pero va DESPUÉS, así no invalida el npm ci
```

Si copiaras todo el código antes del `npm ci`, **cualquier** cambio en una línea de código invalidaría la caché y reinstalaría todas las dependencias en cada build. Con el orden correcto, `npm ci` solo corre cuando cambian las dependencias.

El **`.dockerignore`** (hermano del `.gitignore`) excluye archivos del build: `node_modules` (se reinstalan dentro), `.git`, `dist`, `.env`, logs. No excluirlos hace la imagen más pesada, más lenta de construir y puede **filtrar secretos** (copiar un `.env` a la imagen es un error de seguridad clásico):

```
node_modules
dist
.git
.env
*.log
```

**Ejercicios 4**
4.1 ¿Por qué copiar `package.json` y correr `npm ci` antes de copiar el código acelera los builds siguientes?
4.2 ¿Qué pasa con la caché si copiás todo el código (`COPY . .`) antes del `npm ci`?
4.3 Nombrá dos cosas que deberían ir en `.dockerignore` y por qué (una por peso, una por seguridad).

---

## Módulo 5 — Variables de entorno y secretos en contenedores

**Teoría.** Tu app lee config de `process.env` (el módulo de Node) vía el `ConfigModule` de Nest: `DATABASE_URL`, `JWT_SECRET`, `PORT`. La regla, que ya conocés del módulo de config: **nunca hardcodees secretos ni los metas en la imagen**. La imagen es un artefacto que se comparte y versiona; un secreto adentro es un secreto filtrado.

Los secretos se inyectan **al correr el contenedor**, desde afuera:

```bash
# pasando variables sueltas
docker run -e DATABASE_URL=postgres://... -e JWT_SECRET=xxx -p 3000:3000 task-api

# o desde un archivo (que NUNCA se commitea ni se mete en la imagen)
docker run --env-file .env.production -p 3000:3000 task-api
```

La misma imagen corre en dev, staging y producción **cambiando solo las variables** — no reconstruís nada. Eso es lo bueno de separar config de código (uno de los principios de las "12-factor apps"): un único artefacto, múltiples entornos.

En producción, las plataformas (Railway, Render, ECS) tienen su propio **gestor de secretos**: cargás las variables en su panel y la plataforma las inyecta en el contenedor. Nunca viajan por el repo. Y, como viste en el módulo de config, **validá las variables al arrancar** (schema con Zod/Joi) para que la app falle rápido si falta una, en vez de explotar en runtime cuando un usuario pega a la ruta que la usa.

**No metas secretos en `ARG`/`--build-arg`.** Los build args quedan en la **metadata e historial** de la imagen: cualquiera con la imagen los ve con `docker history`. Un secreto pasado así está filtrado, igual que si lo hardcodearas. Si el *build* necesita un secreto (ej. un token para un registry npm privado), usá **BuildKit secrets**: `RUN --mount=type=secret,id=npmtoken ...` monta el secreto **solo** durante ese `RUN` y no queda en ninguna capa. La regla general sigue valiendo: los secretos de **runtime** se inyectan al correr, no al construir.

**Ejercicios 5**
5.1 ¿Por qué está mal poner `JWT_SECRET` directamente en el Dockerfile o en la imagen?
5.2 ¿Cómo corre la **misma** imagen en dev y en producción con configuraciones distintas, sin reconstruirla?
5.3 ¿Por qué conviene validar las variables de entorno al arrancar la app? (Recordá el "fail fast" del módulo de config.)

---

## Módulo 6 — `docker compose`: app + Postgres + Redis en una línea

**Teoría.** Tu backend no es solo la app: necesita **Postgres** y, pronto, **Redis**. Levantar cada uno a mano es tedioso y poco reproducible. **Docker Compose** declara todos los servicios en un `docker-compose.yml` y los levanta juntos con un comando. Es la forma estándar de tener el entorno de desarrollo completo.

```yaml
services:
  app:
    build: .
    restart: unless-stopped          # se reinicia solo si el contenedor se cae
    init: true                       # PID 1 con init (reenvía señales) — ver módulo 7
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://postgres:secret@db:5432/taskapi
      REDIS_URL: redis://redis:6379
    depends_on:
      db:
        condition: service_healthy   # espera a que Postgres esté LISTO, no solo arrancado

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret      # SOLO para dev; en prod va en el gestor de secretos
      POSTGRES_DB: taskapi
    volumes:
      - pgdata:/var/lib/postgresql/data   # persiste los datos entre reinicios
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5

  redis:
    image: redis:7

volumes:
  pgdata:
```

```bash
docker compose up -d      # levanta los tres servicios en segundo plano
docker compose down       # los baja
```

Dos detalles que evitan dolores de cabeza:

- **Red interna por nombre**: dentro de Compose, los servicios se ven por su nombre. La app conecta a `db:5432`, no a `localhost` — `db` es el hostname del servicio Postgres en la red de Compose.
- **`depends_on` con `condition: service_healthy`**: arrancar Postgres no es lo mismo que estar **listo** para aceptar conexiones. Sin el healthcheck, tu app intenta conectarse antes de tiempo y falla. El healthcheck (con `pg_isready`) hace que la app espere a que la base esté realmente operativa.
- **Volumen**: sin el `volumes`, los datos de Postgres se borran al bajar el contenedor. El volumen `pgdata` los persiste.
- **Hot reload en dev**: en desarrollo conviene montar el código como *bind mount* (`volumes: - ./src:/app/src`) y correr en modo watch, para no reconstruir la imagen en cada cambio. En **producción** es al revés: imagen inmutable, sin bind mounts (lo que probaste es lo que corre).

**Ejercicios 6**
6.1 ¿Qué hace `docker compose up` y qué ventaja tiene sobre levantar Postgres, Redis y la app por separado?
6.2 Dentro de Compose, ¿por qué la app conecta a `db:5432` y no a `localhost:5432`?
6.3 ¿Para qué sirve `depends_on` con `condition: service_healthy`? ¿Qué problema evita frente a un `depends_on` simple?

---

## Módulo 7 — Healthchecks y graceful shutdown en contenedores

**Teoría.** En producción, el orquestador (Kubernetes, ECS, o la plataforma de deploy) necesita saber dos cosas de tu contenedor: **¿está vivo y listo?** y **cómo apagarlo bien**. Dos mecanismos:

**Healthcheck endpoint.** Exponés una ruta (`GET /health`) que responde `200` si la app está sana (idealmente chequeando que la DB responde). El orquestador la consulta periódicamente: si falla, reinicia el contenedor o deja de mandarle tráfico. En Nest se hace con `@nestjs/terminus`:

```ts
@Controller("health")
export class HealthController {
  constructor(private health: HealthCheckService, private db: TypeOrmHealthIndicator) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([() => this.db.pingCheck("database")]);
  }
}
```

**Graceful shutdown.** Esto ya lo viste en el módulo de Node: cuando el orquestador manda `SIGTERM` (en cada deploy, para reemplazar tu contenedor por la versión nueva), no querés cortar de golpe. Querés **dejar de aceptar requests nuevos, terminar los en curso y cerrar conexiones**. En Nest se activa con:

```ts
app.enableShutdownHooks(); // Nest escucha SIGTERM y dispara OnApplicationShutdown en cada módulo
```

Por qué juntos importan: en un deploy, el orquestador arranca el contenedor nuevo, espera a que su `/health` dé verde, le manda tráfico, y recién entonces manda `SIGTERM` al viejo, que termina ordenadamente lo que tenía. Resultado: **deploy sin downtime**. Sin healthcheck, el orquestador manda tráfico a una app que todavía no arrancó; sin graceful shutdown, corta requests a la mitad. Los dos son marca de que operaste algo real.

**El eslabón que falta: PID 1 y la propagación de SIGTERM.** El graceful shutdown solo funciona si la señal **llega** a Node. En un contenedor, el proceso que arranca es **PID 1**, que en Linux tiene semántica especial (no hay handlers de señal por defecto y debe cosechar procesos huérfanos). Dos consecuencias prácticas:

- **Nunca uses `CMD ["npm", "start"]` en producción.** `npm` corre como PID 1 y **no reenvía SIGTERM** al proceso `node` hijo: el orquestador manda la señal, npm la ignora, y tu app muere de golpe al vencer el timeout (~10s) **sin** graceful shutdown. Es un bug silencioso clásico.
- Usá la forma **exec** de `CMD` (array, como en el módulo 3) para correr `node` directo, y mejor aún un **init liviano** como PID 1 (`tini`, o `docker run --init` / `init: true` en compose): reenvía las señales a tu app y cosecha zombies. La cadena queda: `tini → SIGTERM → node → enableShutdownHooks() → cierre ordenado`.

**`HEALTHCHECK` de Docker vs. el del orquestador.** Hay dos niveles que conviene no confundir:

- La instrucción **`HEALTHCHECK`** del Dockerfile (la del módulo 3) la usa el **runtime local** (Docker/Compose) para marcar el contenedor `healthy`/`unhealthy`.
- En producción, el **orquestador** (Kubernetes, ECS, la PaaS) usa **su propia** configuración apuntando a tu endpoint HTTP y a menudo **ignora** el `HEALTHCHECK` del Dockerfile.

Y distinguen dos sondas: **liveness** ("¿está vivo? si no, reiniciá el contenedor") y **readiness** ("¿está listo para recibir tráfico? si no, sacalo del balanceador, pero **no** lo mates"). Un endpoint que responde *not ready* mientras la app calienta (carga config, conecta a la DB) y pasa a *ready* recién cuando puede atender es justo lo que hace que el deploy sin downtime funcione de verdad.

**Ejercicios 7**
7.1 ¿Para qué usa el orquestador un endpoint `/health`? ¿Qué hace si empieza a fallar?
7.2 ¿Qué señal recibe tu contenedor en un deploy y qué debería hacer al recibirla?
7.3 ¿Cómo colaboran el healthcheck y el graceful shutdown para lograr un deploy sin downtime?

---

## Módulo 8 — CI/CD: qué es un pipeline y por qué cambia todo

**Teoría.** **CI/CD** son dos prácticas encadenadas:

- **CI (Integración Continua)**: cada vez que alguien sube código, un sistema automático lo **valida**: instala dependencias, corre el linter, los tests y el build. Si algo falla, avisa **antes** de mergear. El objetivo: que `main` esté siempre sano y que nadie rompa el trabajo de otro sin enterarse.
- **CD (Despliegue/Entrega Continua)**: si la CI pasa, el código se **despliega** automáticamente (o queda listo para desplegar con un click). El objetivo: que llegar a producción sea rutina, no un evento riesgoso.

El **pipeline** típico de un backend Node es una secuencia de etapas que corren en orden y **frenan ante el primer fallo**:

```
lint  →  test  →  build  →  deploy
 │        │        │          │
 │        │        │          └─ sube la imagen y la pone a correr (solo si todo lo previo pasó)
 │        │        └─ compila el TS / construye la imagen Docker
 │        └─ corre unitarios + integración (testcontainers del módulo anterior)
 └─ ESLint / Prettier: estilo y errores obvios
```

Acá se ve por qué el módulo de testing **se paga**: la CI corre tus tests en cada push y **bloquea el merge** si fallan. Una suite de tests sin CI depende de que cada uno se acuerde de correrla; con CI, es imposible mergear código que rompe algo. Esa es la diferencia entre "tengo tests" y "mis tests me protegen de verdad".

**Ejercicios 8**
8.1 ¿Qué diferencia hay entre CI (integración continua) y CD (despliegue continuo)?
8.2 Ordená las etapas típicas de un pipeline de backend y explicá por qué `deploy` va al final.
8.3 ¿Por qué tener tests sin CI es mucho menos efectivo que tener tests **dentro** de un pipeline?

---

## Módulo 9 — GitHub Actions: un workflow real (lint → test → build)

**Teoría.** **GitHub Actions** corre pipelines definidos en archivos YAML dentro de `.github/workflows/`. Un workflow se dispara ante eventos (un push, un pull request) y ejecuta **jobs** compuestos de **steps**.

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main]
  pull_request:            # también valida cada PR antes de mergear

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4          # baja tu código
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm                       # cachea node_modules entre corridas
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

Cómo leerlo:

- **`on`**: cuándo corre. Acá, en cada push a `main` y en **cada pull request** — esto último es lo importante: el PR muestra el check en verde/rojo y podés configurar el repo para **no permitir mergear** si la CI falla (branch protection).
- **`jobs` / `steps`**: las tareas. `checkout` trae el código, `setup-node` instala Node (con caché de dependencias), y después tus comandos: `npm ci`, lint, test, build, en orden. Si uno falla, el job se corta ahí.

Para los tests de **integración** (testcontainers del módulo anterior), Actions ya tiene Docker disponible en `ubuntu-latest`, así que tus contenedores de Postgres efímeros levantan sin config extra.

**De CI a CD: construir y publicar la imagen.** La CI valida; el **CD** construye la imagen y la **publica en un registry** (GitHub Container Registry, Docker Hub) para que la plataforma la despliegue. Dos cosas clave acá: **cachear las capas de Docker** (no solo `node_modules`) y **taggear bien** la imagen.

```yaml
  build-push:
    needs: test                      # solo si la CI pasó
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write                # para pushear a GHCR
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: |
            ghcr.io/${{ github.repository }}:${{ github.sha }}
            ghcr.io/${{ github.repository }}:latest
          cache-from: type=gha        # reusa las CAPAS cacheadas entre corridas
          cache-to: type=gha,mode=max
      # escaneo de vulnerabilidades de la imagen antes de confiar en ella:
      - run: docker scout cves ghcr.io/${{ github.repository }}:${{ github.sha }} || true
```

Por qué **taggear por SHA del commit** (`${{ github.sha }}`) y no solo `latest`: `latest` es un blanco móvil — no sabés qué versión corre ni podés hacer **rollback** a una imagen concreta. El tag por SHA (o por semver) es **inmutable y trazable**: cada deploy apunta a una imagen exacta. Y `cache-from/to: type=gha` cachea las **capas de la imagen** en el almacenamiento de Actions, que es lo más caro del build — distinto del `cache: npm`, que solo cachea las dependencias. El paso final escanea la imagen (`docker scout`, o **Trivy**) en busca de CVEs.

Como alternativa, muchas PaaS tienen **integración nativa con GitHub** y construyen/despliegan ellas mismas en cada push, sin que vos manejes el registry (módulo 10).

**Ejercicios 9**
9.1 ¿Qué dispara el workflow de ejemplo (el `on`) y por qué incluir `pull_request` es clave?
9.2 ¿Qué hace cada uno de estos steps: `checkout`, `setup-node`, `npm ci`?
9.3 ¿Cómo lográs que un PR **no se pueda mergear** si la CI falla?

---

## Módulo 10 — Deploy real: plataforma, DB gestionada y migraciones

**Teoría.** Con la imagen lista y la CI verde, falta ponerla a correr con una **URL pública**. Para un proyecto de portfolio, las plataformas **PaaS** (Platform as a Service) son la mejor relación esfuerzo/resultado: **Railway, Render, Fly.io**. Conectás tu repo de GitHub, detectan el Dockerfile, y en cada push a `main` construyen y despliegan. Te dan la URL, HTTPS y logs sin configurar servidores. (AWS ECS/EKS es más potente y lo que verás en empresas, pero para tu portfolio es overkill al principio.)

Tres cosas que un deploy real necesita y que los tutoriales suelen omitir:

- **Base de datos gestionada**: no corras tu Postgres de producción en un contenedor efímero (los datos se irían). Usás una **DB gestionada** (la plataforma ofrece Postgres con backups, o servicios como Neon/Supabase). En tu app, eso es solo otra `DATABASE_URL` apuntando a la base gestionada.
- **Migraciones en el deploy**: las migraciones (del módulo de Postgres) tienen que correr **antes** de que arranque la versión nueva, para que el esquema esté actualizado. Se hace en un paso de "release" del pipeline (`npm run migrate` antes de levantar la app). Nunca a mano contra producción.
- **Variables de entorno**: las cargás en el panel de la plataforma (módulo 5), nunca en el repo.

El flujo completo que demuestra que cerraste el círculo full stack: **push a `main` → CI corre lint/test/build → si pasa, la plataforma construye la imagen y la despliega → corre las migraciones → la nueva versión queda viva en la URL, con la vieja apagándose ordenadamente (graceful shutdown).** Eso es lo que el roadmap pide: "dejá un proyecto corriendo en la nube con URL pública y CI verde".

**Estrategias de deploy (reconocé los nombres, se preguntan).** El "deploy sin downtime" del módulo 7 tiene variantes con nombre propio:

- **Rolling update**: reemplazás las instancias **de a poco** (levantás una nueva, la verificás con el healthcheck, bajás una vieja, repetís). Es el **default** de la mayoría de orquestadores y PaaS. Simple, pero conviven las dos versiones un rato (tu esquema de DB debe ser compatible con ambas).
- **Blue-green**: mantenés **dos entornos completos** (azul = actual, verde = nuevo); cuando el verde está sano, **switcheás todo el tráfico** de golpe. Rollback instantáneo (volvés al azul), a costa del doble de infraestructura.
- **Canary**: mandás un **porcentaje chico** de tráfico a la versión nueva (5%, luego 25%…), observás métricas y errores, y si va bien escalás. Detecta problemas con impacto mínimo; requiere buen monitoreo.

Las tres se apoyan en el **healthcheck** (módulo 7): sin una señal confiable de "esta versión está sana", ninguna estrategia sirve. Y las migraciones de DB deben ser **compatibles hacia atrás** durante la transición (que las dos versiones convivan), un detalle que separa un deploy real de uno de juguete.

**Ejercicios 10**
10.1 ¿Por qué para un proyecto de portfolio conviene una PaaS (Railway/Render/Fly) antes que AWS ECS?
10.2 ¿Por qué la base de datos de producción NO debe correr en un contenedor efímero? ¿Qué usás en su lugar?
10.3 ¿En qué momento del deploy deben correr las migraciones y por qué no después de que la app ya arrancó?

---

## Módulo 11 — Logs y observabilidad básica en producción

**Teoría.** Una vez desplegado, **no podés meterte a debuggear con la consola**: lo único que tenés para entender qué pasa son los **logs** y las **métricas**. Esto conecta directo con la sección de observabilidad del archivo senior.

- **Logs estructurados**: en producción no logueás texto suelto, sino **JSON** (con `pino`, rapidísimo y el estándar en Node). ¿Por qué JSON? Porque las plataformas de logs (Datadog, CloudWatch, el visor de tu PaaS) lo **parsean y filtran**: podés buscar "todos los errores 500 del usuario X en la última hora". Con texto suelto, eso es imposible.

```ts
// en vez de: console.log("usuario " + id + " creó proyecto " + pid)
logger.info({ evento: "proyecto_creado", usuarioId: id, proyectoId: pid });
// queda como JSON: { "level":"info", "evento":"proyecto_creado", "usuarioId":42, ... }
```

- **Niveles con criterio**: `error` (algo se rompió), `warn` (algo raro pero recuperable), `info` (eventos de negocio), `debug` (detalle, normalmente apagado en prod). Loguear todo es tan inútil como no loguear nada (del senior).
- **`correlationId`**: un id único por request que atraviesa todos los logs de esa operación, para poder seguir un request entre muchos. En Nest, un interceptor o middleware lo inyecta.
- **Nunca loguees secretos ni datos sensibles**: contraseñas, tokens, datos personales fuera de los logs (los logs se guardan y se indexan; un secreto logueado es un secreto filtrado).

No hace falta montar todo el stack de observabilidad para un portfolio, pero **logs estructurados con niveles** y un healthcheck (módulo 7) ya te ponen por encima del "console.log y rezar". Si te preguntan "¿cómo debuggeás un problema en producción?", la respuesta es "logs estructurados con correlationId y métricas de latencia/errores", no "me meto al server".

**Ejercicios 11**
11.1 ¿Por qué en producción se loguea en JSON estructurado y no con `console.log` de texto suelto?
11.2 ¿Para qué sirve un `correlationId` en los logs?
11.3 ¿Por qué nunca hay que loguear contraseñas o tokens, aunque sea "temporalmente para debuggear"?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 El "en mi máquina funciona": empaqueta la app con todo lo que necesita (runtime,
    dependencias, config) en una imagen reproducible que corre idéntica en cualquier
    entorno, eliminando las diferencias entre máquinas.
1.2 La imagen es el molde inmutable (se construye una vez); el contenedor es cada pieza
    que sale de ese molde (una instancia en ejecución). De una imagen levantás muchos
    contenedores idénticos.
1.3 Porque comparte el kernel del sistema operativo del host y aísla solo lo necesario,
    en vez de virtualizar un SO entero como una VM. Por eso pesa menos y arranca en
    segundos.
```

### Módulo 2
```
2.1 RUN corre durante el BUILD y queda horneado en la imagen (ej. RUN npm run build
    compila el TS). CMD corre al ARRANCAR el contenedor (ej. CMD ["node","dist/main.js"]
    levanta la app). Hay un RUN por cada paso de construcción; un solo CMD de arranque.
2.2 Mapea el puerto del host al del contenedor (host:contenedor), para poder acceder a
    la app desde afuera. Sin -p, la app corre aislada dentro del contenedor y no le
    llegás desde el navegador/cliente.
2.3 Para aprovechar la caché de capas: si el código cambia pero las dependencias no,
    Docker reusa la capa del npm ci en vez de reinstalar todo. (Se desarrolla en el
    módulo 4.)
```

### Módulo 3
```
3.1 El de una sola etapa mete en la imagen final cosas que solo sirven para construir
    (código fuente, compilador, devDependencies), haciéndola grande y con más superficie
    de ataque. El multi-stage separa build de runtime y copia a la imagen final SOLO el
    resultado compilado.
3.2 Instala solo las dependencies de producción, omitiendo las devDependencies (Jest,
    ESLint, etc.), que no se usan para CORRER la app. Está bien porque esas herramientas
    ya cumplieron su rol en la etapa de build; la imagen de runtime no las necesita.
3.3 Para no correr como root: si comprometen el contenedor, el atacante no tiene
    privilegios de superusuario, limitando el daño. Es el principio de mínimo privilegio
    del archivo senior aplicado al contenedor.
```

### Módulo 4
```
4.1 Porque Docker cachea las capas que no cambian. Como package.json cambia poco, la
    capa del npm ci (lenta) se reusa entre builds y solo se re-ejecuta cuando cambian las
    dependencias, no en cada cambio de código.
4.2 Se invalida la caché del npm ci: como el código va antes y cambia seguido, cualquier
    modificación obliga a reinstalar TODAS las dependencias en cada build, haciéndolo
    lento.
4.3 Por peso: node_modules o dist (se generan dentro / no hacen falta). Por seguridad:
    .env (no querés filtrar secretos metiéndolos en la imagen) o .git.
```

### Módulo 5
```
5.1 Porque la imagen es un artefacto que se comparte, versiona y sube a registries; un
    secreto adentro queda expuesto a cualquiera con acceso a la imagen, y no se puede
    rotar sin reconstruir.
5.2 Inyectando las variables de entorno desde afuera al correr el contenedor (-e o
    --env-file, o el gestor de secretos de la plataforma). El código y la imagen son los
    mismos; solo cambia la config por entorno.
5.3 Para fallar rápido (fail fast): si falta o está mal una variable, la app no arranca y
    te enterás al instante en el deploy, en vez de que explote más tarde en runtime
    cuando un usuario pegue a la ruta que la usa.
```

### Módulo 6
```
6.1 Levanta todos los servicios definidos (app, Postgres, Redis) juntos con un comando,
    de forma reproducible. Evita instalar/arrancar cada uno a mano y garantiza que todo
    el equipo tiene el mismo entorno de desarrollo.
6.2 Porque dentro de la red de Compose los servicios se resuelven por su NOMBRE: "db" es
    el hostname del servicio Postgres en esa red. "localhost" apuntaría al propio
    contenedor de la app, no al de la base.
6.3 Hace que la app espere a que Postgres esté realmente LISTO para aceptar conexiones
    (no solo "arrancado"). Un depends_on simple solo espera a que el contenedor inicie,
    así que la app podría intentar conectarse antes de tiempo y fallar.
```

### Módulo 7
```
7.1 Para saber si la app está sana y lista; lo consulta periódicamente. Si /health
    empieza a fallar, el orquestador reinicia el contenedor o deja de enrutarle tráfico.
7.2 SIGTERM. Debería hacer graceful shutdown: dejar de aceptar requests nuevos, terminar
    los en curso, cerrar conexiones (DB, Redis) y recién entonces salir.
7.3 El orquestador arranca el contenedor nuevo, espera a que su /health dé verde y le
    manda tráfico; recién ahí manda SIGTERM al viejo, que termina ordenadamente lo que
    tenía. Así nunca hay un momento sin instancia sana atendiendo: deploy sin downtime.
```

### Módulo 8
```
8.1 CI (integración continua) valida automáticamente cada cambio (lint, test, build)
    para mantener main sano. CD (despliegue continuo) lleva ese código validado a
    producción de forma automática o con un click.
8.2 lint → test → build → deploy. deploy va al final porque solo tiene sentido publicar
    código que ya pasó el linter, los tests y compiló: si algo falla antes, el pipeline
    se corta y no se despliega nada roto.
8.3 Porque sin CI, correr los tests depende de que cada persona se acuerde de hacerlo;
    es fácil mergear código que rompe algo. Con CI, los tests corren en cada push/PR y
    bloquean el merge si fallan: la protección es automática y no se puede saltear.
```

### Módulo 9
```
9.1 Lo disparan los push a main y los pull requests. Incluir pull_request es clave porque
    valida el código ANTES de mergear: el PR muestra el check verde/rojo y podés impedir
    el merge si falla.
9.2 checkout baja tu código al runner; setup-node instala la versión de Node indicada (y
    cachea dependencias); npm ci instala las dependencias de forma limpia y reproducible
    desde el package-lock.
9.3 Con branch protection en el repo: marcás el check de CI como requerido para esa rama,
    así GitHub no permite mergear el PR hasta que el workflow pase en verde.
```

### Módulo 10
```
10.1 Porque una PaaS te da deploy desde el repo, URL pública, HTTPS y logs sin
     configurar servidores ni redes: máximo resultado con mínimo esfuerzo. ECS/EKS es más
     potente pero exige mucha configuración de infraestructura, innecesaria para un
     portfolio.
10.2 Porque un contenedor efímero pierde sus datos al recrearse (en cada deploy): tu base
     desaparecería. Se usa una base de datos gestionada (Postgres administrado con
     backups, o Neon/Supabase), referenciada por DATABASE_URL.
10.3 Antes de que arranque la versión nueva (en un paso de release del pipeline). Si la
     app nueva arrancara contra un esquema viejo, fallaría al usar columnas/tablas que su
     código espera y que la migración todavía no creó.
```

### Módulo 11
```
11.1 Porque las plataformas de logs parsean y filtran JSON: podés buscar y agregar (ej.
     "todos los errores 500 del usuario X en la última hora"). El texto suelto no es
     consultable de forma estructurada.
11.2 Es un id único por request que aparece en todos los logs de esa operación, para
     poder seguir/reconstruir un request específico entre miles de líneas de log
     (especialmente útil si atraviesa varios servicios).
11.3 Porque los logs se almacenan e indexan en sistemas externos; un secreto logueado
     queda persistido y expuesto a cualquiera con acceso a los logs. "Temporal" tiende a
     quedar, y rotar un secreto filtrado es costoso. Nunca se loguean.
```

---

## Siguientes pasos

Con este módulo cerrás el ciclo de vida completo: tu "Task API" empaquetada en Docker, validada por CI en cada push, y desplegada con URL pública, DB gestionada, migraciones automáticas y graceful shutdown. Eso es exactamente lo que el roadmap pide para el portfolio. El recorrido desde acá: **(1)** agregá el job de **CD** que despliega tras la CI verde, y protegé `main` para que nadie mergee con tests rojos; **(2)** sumá **Redis** (ya está en tu `docker-compose`) para el módulo de **caché y colas (BullMQ)** del roadmap — el lugar natural para el trabajo CPU-bound que aprendiste a no poner en el event loop; **(3)** un **README con diagrama de arquitectura** y la URL desplegada: es lo primero que mira quien evalúa tu portfolio. Con DB, auth, tests y deploy, ya tenés un backend de nivel profesional para mostrar.
