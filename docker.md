# Docker a fondo: cómo funciona, mejores prácticas y casos de uso

**La arquitectura real (daemon, containerd, runc, namespaces y cgroups), imágenes por dentro (capas, union FS, digests), build con BuildKit (cache mounts, multi-stage, multi-arch), imágenes chicas y hardening de seguridad, volúmenes y redes, límites de recursos, y los casos de uso (dev, deps efímeras, CI, jobs, microservicios) · para entrevistas senior/staff y producción · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es el **deep-dive de Docker en sí**: cómo funciona por dentro, cómo construir imágenes bien, cómo endurecerlas y para qué usarlas. El **ciclo a producción** —empaquetar tu app, CI/CD, deploy con URL pública, healthchecks y graceful shutdown— vive en [Docker, deploy y CI/CD](docker-deploy.md); acá **no lo repetimos**, lo referenciamos. Pensalo así: `docker-deploy` te enseñó a *shippear* una app; este te enseña **qué es Docker de verdad y cómo usarlo con criterio**.

**Lo que asumimos.** Que ya dockerizaste algo alguna vez (sabés qué es un `Dockerfile`, `docker build`/`run`, imagen vs contenedor — todo eso está en [Docker, deploy y CI/CD](docker-deploy.md) M1-M2), que entendés procesos y memoria a nivel básico de sistema operativo, y que conocés el event loop y el heap de Node ([Node.js por dentro](nodejs.md)). No asumimos que sepas cómo Docker logra el aislamiento ni cómo se optimiza/endurece una imagen — eso es lo nuevo.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribís un Dockerfile/compose/comando).

**El marco que usamos en las fallas.** Para cada modo de falla aplicamos las **cuatro capas**: (1) **qué se rompe**, (2) **por qué** a *esta* escala, (3) **control de corto plazo**, (4) **cambio de diseño** de largo plazo.

**Índice de módulos**
1. Cómo funciona Docker por dentro: daemon, containerd, runc, namespaces y cgroups
2. Imágenes por dentro: capas, union filesystem, tags vs digests y registries
3. Construir bien: BuildKit, cache mounts, multi-stage y multi-arquitectura
4. Imágenes chicas: elegir la base y optimizar capas
5. Hardening: correr seguro (no-root, read-only, capabilities, escaneo, firma)
6. Datos y redes: volúmenes (bind/named/tmpfs) y networking
7. Runtime en producción: límites de recursos, OOM, Node y cgroups, logging, debugging
8. Casos de uso y criterio: cuándo Docker sí, cuándo no, y el checklist

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Cómo funciona Docker por dentro: daemon, containerd, runc, namespaces y cgroups

**Teoría.** "Docker" no es un programa monolítico; es una **cadena de componentes**, y entenderla te salva en entrevistas y en debugging real:

- **`docker` (CLI)** — el cliente que usás en la terminal. **No corre nada**: le manda órdenes por una API (un socket Unix, `/var/run/docker.sock`) a…
- **`dockerd` (el daemon)** — el proceso de fondo que gestiona imágenes, redes y volúmenes. Delega el ciclo de vida de los contenedores en…
- **`containerd`** — el *runtime* de alto nivel (un estándar de la industria, también lo usa Kubernetes) que gestiona la descarga de imágenes y supervisa contenedores. Para **crear** cada contenedor invoca…
- **`runc`** — el *runtime* de bajo nivel (implementa la **OCI Runtime Spec**): es quien realmente le pide al kernel de Linux crear el contenedor con sus namespaces y cgroups, y después **se retira** (un shim mantiene vivo el proceso).

Esa separación en capas estandarizadas (**OCI**: Open Container Initiative, define el formato de imagen y el runtime) es lo que hace que una imagen que construís con Docker corra en Kubernetes, Podman o cualquier runtime OCI. Docker no es dueño del formato; lo estandarizó.

**Lo que hace que un contenedor sea un contenedor** son dos features del **kernel de Linux** (no de Docker — Docker solo las orquesta):

- **Namespaces** — dan **aislamiento** (qué ve el proceso). Cada contenedor tiene sus propios namespaces: `pid` (ve sus procesos, no los del host), `net` (su propia interfaz de red e IP), `mnt` (su propio árbol de archivos), `uts` (su hostname), `ipc`, `user` (mapeo de UIDs). Por eso adentro del contenedor tu app se cree PID 1 y sola en la máquina.
- **cgroups** (*control groups*) — dan **límites** (cuánto puede usar). Acotan CPU, memoria, IO por contenedor. Es lo que evita que un contenedor se coma toda la RAM del host (módulo 7).

De acá sale la diferencia real con una **VM**: una VM virtualiza **hardware** y corre un **kernel propio** (pesada, arranca en minutos, aislamiento fuerte). Un contenedor **comparte el kernel del host** y solo aísla con namespaces/cgroups (liviano, arranca en milisegundos). El precio de esa liviandad: **el contenedor NO es una frontera de seguridad tan fuerte como una VM** — como comparten kernel, un exploit del kernel escapa del contenedor. Para cargas hostiles multi-tenant de verdad se usan runtimes con más aislamiento (gVisor, Kata, Firecracker) o VMs. Este matiz es exactamente lo que distingue una respuesta senior.

> La frase mental: **"Docker" es una cadena: CLI → dockerd → containerd → runc, sobre estándares OCI (por eso la imagen corre en cualquier runtime). Un contenedor no es magia de Docker: son dos features del kernel de Linux — namespaces (aislamiento: qué ve el proceso) + cgroups (límites: cuánto usa). Comparte el kernel del host (liviano, arranca en ms) a diferencia de una VM que corre su propio kernel — por eso el contenedor NO es una frontera de seguridad tan dura como una VM.**

**Ejercicios 1**
1.1 🔁 Nombrá la cadena de componentes de Docker desde la CLI hasta el kernel y qué hace cada uno.
1.2 🧠 ¿Qué aportan los namespaces y qué aportan los cgroups? Asigná cada uno a "aislamiento" o "límites".
1.3 🧠 ¿Por qué un contenedor arranca en milisegundos y una VM en minutos? ¿Y por qué eso implica que el contenedor es una frontera de seguridad más débil?

---

## Módulo 2 — Imágenes por dentro: capas, union filesystem, tags vs digests y registries

**Teoría.** Una **imagen** no es un archivo monolítico: es una **pila de capas de solo-lectura** más un **manifiesto** (un JSON con la config y la lista ordenada de capas). Cada instrucción del `Dockerfile` que toca el filesystem (`RUN`, `COPY`, `ADD`) produce **una capa** — un diff del filesystem respecto de la anterior.

Al **correr** un contenedor, Docker apila esas capas de solo-lectura y les pone encima una **capa de escritura fina** (copy-on-write): el contenedor puede modificar archivos, pero cada cambio **copia** el archivo a su capa superior sin tocar las de abajo. Esto lo maneja un **union filesystem** (en Linux moderno, el storage driver **overlay2**), que "funde" varias capas en una vista única. Dos consecuencias enormes:

- **Las capas se comparten entre imágenes.** Si diez imágenes usan `node:22-alpine` de base, esa capa está en disco **una sola vez**. Y al pushear/pullear, solo viajan las capas que faltan. Por eso el orden del `Dockerfile` importa tanto para la caché (módulo 3 y [docker-deploy](docker-deploy.md) M4).
- **Borrar un archivo en una capa no lo saca de la imagen.** Si en una capa copiás un secreto y en la siguiente lo `rm`, el archivo **sigue** en la capa de abajo (solo queda "tapado"). Cualquiera con la imagen lo recupera. Los secretos nunca se "borran" de una imagen; **nunca deben entrar** (módulo 5).

Una imagen es **content-addressable**: se identifica por el **digest** `sha256:...` de su contenido. Y acá está una distinción que se pregunta siempre:

- Un **tag** (`node:22`, `miapp:latest`) es un **puntero mutable** — hoy apunta a un digest, mañana el mismo tag puede apuntar a otro (cuando el mantenedor lo re-publica). `latest` es el peor: es un blanco móvil.
- Un **digest** (`node:22-alpine@sha256:abc...`) es **inmutable**: siempre es exactamente ese contenido. Para builds **reproducibles** y deploys **trazables**, pineás por digest (o al menos por un tag específico + lockeás el digest).

Las imágenes viven en **registries** (Docker Hub, GHCR, ECR). Un `docker pull node:22` resuelve el tag a un digest, baja el manifiesto y las capas que falten. Con **multi-arch** (módulo 3), el tag apunta en realidad a un *manifest list* que elige la imagen correcta según tu arquitectura (amd64/arm64).

> La frase mental: **una imagen = pila de capas read-only (una por instrucción que toca el FS) + un manifiesto; al correr se le suma una capa de escritura copy-on-write, todo fundido por un union FS (overlay2). Las capas se comparten entre imágenes y solo viajan las que faltan (por eso el orden del Dockerfile manda en la caché). Borrar un archivo NO lo saca de la capa de abajo → los secretos jamás deben entrar. Tag = puntero mutable (latest = blanco móvil); digest sha256 = inmutable → pineá por digest para reproducibilidad.**

**Ejercicios 2**
2.1 🔁 ¿Qué es una capa de imagen y qué genera una capa nueva? ¿Qué es la capa de escritura de un contenedor?
2.2 🧠 Copiás un archivo con un secreto en una capa y lo borrás con `rm` en la siguiente. ¿El secreto está en la imagen final? ¿Por qué?
2.3 🧠 ¿Cuál es la diferencia entre un tag y un digest, y por qué para reproducibilidad conviene pinear por digest?

---

## Módulo 3 — Construir bien: BuildKit, cache mounts, multi-stage y multi-arquitectura

**Teoría.** Desde Docker 23 (2023) el builder por defecto es **BuildKit** ⚠️: construye el grafo de dependencias del `Dockerfile`, paraleliza etapas independientes y habilita features que un build "clásico" no tiene. Las que importan:

**1. Cache mounts — cachear *entre builds* el cache del gestor de paquetes.** El truco de ordenar `COPY package*.json` antes del código ([docker-deploy](docker-deploy.md) M4) cachea la **capa** de `npm ci`, pero se invalida cuando cambia **cualquier** dependencia. Un **cache mount** va más fino: monta un directorio persistente para el cache de npm/pnpm que **sobrevive entre builds** aunque la capa se invalide:

```dockerfile
# syntax=docker/dockerfile:1
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

Así, cambiar una sola dependencia no re-descarga **todo** el registro: npm reusa lo que ya bajó. Es el cache de build que casi nadie configura y que acorta minutos.

**2. Build secrets — pasar un secreto al build sin que quede en la imagen.** Como vimos en [docker-deploy](docker-deploy.md) M5, un secreto en `ARG`/`--build-arg` queda en `docker history`. BuildKit lo resuelve montándolo **solo** durante ese `RUN`:

```dockerfile
RUN --mount=type=secret,id=npmtoken \
    NPM_TOKEN=$(cat /run/secrets/npmtoken) npm ci
```
```bash
docker build --secret id=npmtoken,src=./.npmtoken .
```
El secreto no queda en ninguna capa ni en el historial.

**3. Multi-stage — recap con el porqué profundo.** El multi-stage de [docker-deploy](docker-deploy.md) M3 (una etapa `build` con todo el toolchain, una etapa `runtime` limpia que copia solo el resultado) no es solo "imagen más chica": es **separar las dependencias de build de las de runtime**. Podés tener etapas intermedias (una para deps, una para compilar, una para tests) y que la final copie apenas el artefacto. Un patrón útil es una etapa `deps` que instala solo producción y otra que compila, para maximizar el reuso de caché.

**4. Multi-arquitectura — una imagen para amd64 y arm64.** Con Macs M-series y servidores ARM (AWS Graviton), tu imagen tiene que correr en **dos arquitecturas**. `docker buildx` construye ambas y publica un **manifest list** bajo un mismo tag:

```bash
docker buildx build --platform linux/amd64,linux/arm64 -t miapp:1.0 --push .
```
Quien haga `pull` recibe automáticamente la variante de su arquitectura. (Construir la arquitectura ajena se hace emulando con QEMU —más lento— o con *build nodes* nativos.)

Y la base de todo build eficiente sigue siendo el **`.dockerignore`** ([docker-deploy](docker-deploy.md) M4): reduce el **contexto de build** que la CLI le manda al daemon — sin él, mandás `node_modules`, `.git` y demás basura en cada build.

> La frase mental: **BuildKit (default desde Docker 23) habilita lo que un build clásico no: cache mounts (`RUN --mount=type=cache`) que persisten el cache de npm ENTRE builds aunque la capa se invalide; build secrets (`--mount=type=secret`) que no quedan en el historial; multi-stage para separar deps de build vs runtime; y buildx para multi-arch (amd64+arm64 en un manifest list bajo un tag). El .dockerignore recorta el contexto que viaja al daemon.**

**Ejercicios 3**
3.1 🔁 ¿Qué es BuildKit y desde cuándo es el default? Nombrá dos features que habilita.
3.2 🧠 Diferencia entre cachear la **capa** de `npm ci` (ordenando el COPY) y usar un **cache mount**. ¿Qué problema resuelve el cache mount que el orden de capas no?
3.3 🧠 ¿Por qué hoy conviene construir multi-arch, y qué herramienta lo hace? ¿Qué recibe quien hace `pull`?

---

## Módulo 4 — Imágenes chicas: elegir la base y optimizar capas

**Teoría.** Una imagen chica se transfiere más rápido (deploys y autoscaling ágiles), llena menos disco/registry y —lo más importante— tiene **menos superficie de ataque** (menos binarios instalados = menos CVEs). Dos palancas: la **base** y **cómo escribís las capas**.

**Elegir la base** (recap de [docker-deploy](docker-deploy.md) M3 con el criterio completo):

- **`node:lts-slim`** (Debian recortado, **glibc**) — la opción **segura por defecto**: máxima compatibilidad con módulos nativos, tamaño razonable.
- **`node:lts-alpine`** (musl libc; la **base** Alpine pesa ~5 MB, aunque la imagen final con Node adentro ronda las decenas de MB — el "~5 MB" es la capa base, no la imagen resultante) — mínima, pero **musl** puede romper dependencias nativas o dar sorpresas de DNS/compilación. Buena si no tenés nativos problemáticos.
- **`gcr.io/distroless/nodejs`** — solo el runtime, **sin shell ni gestor de paquetes**, no-root por defecto: superficie de ataque mínima. La más segura para runtime, pero **no tiene shell** → no podés `docker exec` a debuggear adentro (módulo 7).
- **`scratch`** — imagen **vacía**. Para binarios estáticos (Go compilado); para Node no aplica directo.

Regla transversal: **pineá la versión** y verificá que el *major* sea una **LTS vigente en 2026**; para reproducibilidad total, por **digest** (módulo 2).

**Optimizar capas:**

- **Combiná comandos relacionados en un `RUN`** y limpiá en la **misma** capa. Si instalás algo con `apt` y limpiás el cache en un `RUN` **posterior**, el cache ya quedó en la capa anterior (recordá: borrar no achica la capa de abajo, módulo 2). Correcto:
  ```dockerfile
  RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates \
      && rm -rf /var/lib/apt/lists/*
  ```
- **Ordená de lo que menos cambia a lo que más cambia** para maximizar la caché ([docker-deploy](docker-deploy.md) M4): base → deps → código.
- **Multi-stage** (módulo 3): que la imagen final no arrastre el toolchain.
- **Medí**: `docker image ls` (tamaño total) y `docker history <img>` o `dive` (peso por capa) te muestran dónde está la grasa.

> 🔥 **Falla en 4 capas — la imagen de 1.5 GB que frena los deploys.**
> (1) **Qué se rompe:** la imagen pesa 1.5 GB (base `node` completa + devDependencies + código fuente + cache de apt en capas); cada deploy y cada scale-up tardan y el registry se llena.
> (2) **Por qué a esta escala:** con autoscaling/rolling updates, cada nueva instancia baja la imagen entera; el peso se paga en **cada** arranque, no una vez.
> (3) **Corto plazo:** `.dockerignore` para no meter basura; borrar el cache de apt en la **misma** capa que la instalación.
> (4) **Largo plazo:** **multi-stage** (runtime sin toolchain) + **base slim/distroless** + `npm ci --omit=dev` + pin por digest. La imagen baja a decenas/pocos cientos de MB y los deploys se vuelven ágiles y más seguros.

> La frase mental: **imagen chica = deploy rápido + menos CVEs. Dos palancas: la base (slim glibc = default seguro; alpine mínima pero musl; distroless mínima superficie pero sin shell; scratch para binarios estáticos) y las capas (combinar RUN + limpiar en la MISMA capa porque borrar no achica la de abajo; ordenar de lo estable a lo cambiante; multi-stage). Medí con `docker history`/`dive`. Pineá por digest.**

**Ejercicios 4**
4.1 🔁 Nombrá tres imágenes base para Node y el trade-off de cada una (slim, alpine, distroless).
4.2 🧠 ¿Por qué instalar con `apt` y limpiar el cache en un `RUN` **posterior** no reduce el tamaño de la imagen? ¿Cómo se hace bien?
4.3 ✍️ Tenés este Dockerfile ingenuo. Reescribilo para que la imagen sea chica y segura (multi-stage, base adecuada, no-root, limpieza en la misma capa, orden de caché):
```dockerfile
FROM node:22
WORKDIR /app
COPY . .
RUN npm install
RUN npm run build
CMD ["node", "dist/main.js"]
```
(Solución al final.)

---

## Módulo 5 — Hardening: correr seguro (no-root, read-only, capabilities, escaneo, firma)

**Teoría.** Una imagen que funciona no es una imagen segura. El hardening es un conjunto de defensas en capas; ninguna alcanza sola. Las que se esperan de un perfil senior:

**En la imagen (build-time):**
- **No corras como `root`.** Por defecto el proceso del contenedor es root **dentro** del namespace, y si un exploit escapa, es root con más chances de dañar el host. Declará un usuario sin privilegios: `USER node` (o creá uno). Es el **mínimo privilegio** aplicado al contenedor.
- **Base mínima y actualizada** (módulo 4): menos paquetes = menos CVEs; re-buildeá para tomar parches de la base.
- **Nada de secretos en la imagen** (módulos 2 y 3): ni en capas, ni en `ARG`, ni idealmente en `ENV` (se ven con `docker inspect`). Los secretos de runtime se inyectan al correr; los de build, con BuildKit secrets.

**Al correr (runtime):**
- **Filesystem de solo lectura:** `docker run --read-only` (más un `tmpfs` para lo que necesite escribir). Si el atacante no puede escribir el filesystem, no puede dejar un binario ni modificar la app.
- **Soltá capabilities de Linux:** un contenedor arranca con un set de *capabilities* del kernel que casi ninguna app necesita. `--cap-drop=ALL` y agregás solo las imprescindibles (`--cap-add=...`). 
- **`--security-opt=no-new-privileges`:** impide que un proceso escale privilegios (p. ej. vía binarios setuid).
- **Nunca `--privileged`** salvo que sepas exactamente por qué; y **no montes `/var/run/docker.sock`** dentro de un contenedor sin necesidad: es **root en el host** de facto (quien controla el socket del daemon controla la máquina).
- **Docker rootless** (maduro en 2026): correr el **daemon** sin privilegios de root, apoyado en **user namespaces** (`userns-remap`), de modo que un escape del contenedor **no aterrice como root en el host**. Es una defensa de infraestructura que complementa el `USER` no-root de la imagen (una protege el proceso; la otra, el host).

**En el pipeline (supply chain):**
- **Escaneá la imagen** por CVEs antes de confiar en ella: **Trivy**, **Docker Scout** o **Grype**, integrado en la CI ([docker-deploy](docker-deploy.md) M9).
- **SBOM y firma:** generá un *Software Bill of Materials* (con **syft**) para saber qué contiene la imagen, y **firmala** (**cosign**/Sigstore) para que el que la despliega verifique que es la que vos publicaste y no una manipulada. Es la defensa contra ataques de **supply chain**.

> 🔥 **Falla en 4 capas — contenedor comprometido corriendo como root.**
> (1) **Qué se rompe:** una RCE en tu app; como el contenedor corre como `root`, con capabilities de más y filesystem escribible, el atacante instala herramientas, persiste y sondea el host.
> (2) **Por qué a esta escala:** el contenedor comparte kernel con el host (módulo 1); root adentro + capabilities amplias acortan el camino a un escape o a moverse lateralmente.
> (3) **Corto plazo:** parchear la vuln de la app; rotar credenciales que el contenedor podía leer.
> (4) **Largo plazo:** **defensa en capas** — `USER` no-root, `--read-only` + tmpfs, `--cap-drop=ALL`, `--security-opt=no-new-privileges`, base mínima, escaneo de CVEs en CI y firma de imágenes. Cada capa achica el daño posible.

> La frase mental: **una imagen que funciona ≠ segura. Hardening en capas: build-time (USER no-root, base mínima y parcheada, cero secretos en la imagen), runtime (--read-only + tmpfs, --cap-drop=ALL + solo las que hacen falta, --security-opt no-new-privileges, jamás --privileged ni montar docker.sock = root en el host) y supply chain (escanear CVEs con Trivy/Scout/Grype en CI, SBOM con syft, firma con cosign). Ninguna capa alcanza sola.**

**Ejercicios 5**
5.1 🔁 Nombrá tres medidas de hardening en runtime y qué ataque mitiga cada una.
5.2 🧠 ¿Por qué montar `/var/run/docker.sock` dentro de un contenedor es tan peligroso?
5.3 🧠 ¿Qué es escanear una imagen y qué agregan el SBOM y la firma (cosign) por encima del escaneo? ¿Contra qué protegen?

---

## Módulo 6 — Datos y redes: volúmenes (bind/named/tmpfs) y networking

**Teoría.** El filesystem de un contenedor es **efímero**: al recrearlo (cada deploy), la capa de escritura se pierde. Para **persistir** datos o **compartir** archivos con el host, montás almacenamiento externo. Tres tipos, y elegir mal es un error clásico:

- **Named volume** (`-v pgdata:/var/lib/postgresql/data`) — lo gestiona Docker (vive en `/var/lib/docker/volumes`). Es la opción para **datos que deben persistir** (la base de datos en dev, ver [docker-deploy](docker-deploy.md) M6). Portable, con backup, desacoplado del layout del host.
- **Bind mount** (`-v ./src:/app/src`) — monta una **ruta del host** dentro del contenedor. Ideal en **desarrollo** para hot reload (editás en tu editor, el contenedor ve el cambio). En **producción no** (acopla la imagen al filesystem del host y rompe la inmutabilidad: lo que probaste debe ser lo que corre).
- **tmpfs** (`--tmpfs /tmp`) — almacenamiento **en memoria**, se borra al parar el contenedor. Para datos **sensibles o efímeros** que no querés que toquen disco (y el compañero perfecto de `--read-only` del módulo 5: el rootfs es de solo lectura, y lo poco que se escribe va a un tmpfs).

**Networking.** Docker le da a cada contenedor su namespace de red (módulo 1) y lo conecta con un **driver**:

- **bridge** (el default) — una red virtual privada con NAT; el contenedor sale a internet y vos publicás puertos con `-p host:contenedor`. En una **red bridge definida por el usuario** (la que crea Compose), los contenedores se resuelven **por nombre** vía el DNS interno de Docker (`db:5432`, no `localhost` — [docker-deploy](docker-deploy.md) M6).
- **host** — el contenedor **comparte la pila de red del host** (sin aislamiento de red ni NAT). Más performante, pero sin aislamiento y con choques de puertos; casos puntuales.
- **none** — sin red. Para jobs que no necesitan red (aislamiento máximo).
- **overlay** — red multi-host (Swarm/orquestadores) para que contenedores en **distintas máquinas** se hablen. (En la práctica, hoy la red multi-host relevante es la **CNI de [Kubernetes](kubernetes.md)**, no Swarm.)

Publicar un puerto (`-p 8080:3000`) mapea el `8080` del host al `3000` del contenedor. Sin publicar, el contenedor es alcanzable **dentro** de su red Docker pero no desde afuera.

> La frase mental: **el FS del contenedor es efímero; para persistir/compartir montás almacenamiento: named volume (datos que persisten, gestionado por Docker — la base), bind mount (ruta del host → dev/hot-reload, nunca en prod), tmpfs (en memoria, efímero/sensible, compañero de --read-only). En red: bridge (default, NAT, y en la red definida por usuario hay DNS por nombre de servicio), host (comparte red del host, sin aislamiento), none (sin red), overlay (multi-host). `-p host:contenedor` publica hacia afuera.**

**Ejercicios 6**
6.1 🔁 Diferenciá named volume, bind mount y tmpfs, y dá el caso de uso típico de cada uno.
6.2 🧠 ¿Por qué un bind mount de código es bueno en desarrollo pero malo en producción?
6.3 🧠 En una red bridge definida por el usuario (la de Compose), ¿por qué la app conecta a `db:5432` y no a `localhost:5432`? ¿Qué lo resuelve?

---

## Módulo 7 — Runtime en producción: límites de recursos, OOM, Node y cgroups, logging, debugging

**Teoría.** Un contenedor sin límites es un vecino ruidoso: puede acaparar CPU y memoria del host y tumbar a los demás. Los **cgroups** (módulo 1) se configuran al correr:

- **`--memory=512m`** — límite de RAM. Si el contenedor lo supera, el **OOM killer** del kernel **mata el proceso** (verás `OOMKilled`). No es un warning: es muerte súbita. Ojo con el **swap**: si no acotás `--memory-swap`, Docker permite por defecto swap hasta el **doble** de `--memory`, lo que puede **enmascarar** el OOM (degrada por swapping antes de matar); en un servidor conviene igualar `--memory-swap` a `--memory` para desactivarlo.
- **`--cpus=1.5`** — límite de CPU (fracciones permitidas).

**El clásico de Node en contenedores** ⚠️: el heap de V8 tiene un tope (`--max-old-space-size`). Históricamente Node **no leía el límite de memoria del cgroup** y dimensionaba el heap según la RAM del **host** (p. ej. 32 GB), así que en un contenedor con `--memory=512m` el heap crecía tranquilo hasta que el **cgroup lo mataba por OOM** antes de que V8 hiciera GC. Node moderno detecta mejor el cgroup, pero la práctica segura sigue siendo **fijar el heap acorde al límite del contenedor**, dejando margen para lo no-heap:
```bash
# con --memory=512m, dejá el heap en ~75%
NODE_OPTIONS=--max-old-space-size=384
```
En [Kubernetes](kubernetes.md) este valor se suele **derivar del límite de memoria del pod** (con el *Downward API* o un valor calculado en el manifiesto), en vez de hardcodearlo.

**Restart policies** — qué hace Docker si el contenedor termina: `no` (default), `on-failure` (solo si sale con error), `always`, `unless-stopped` (se reinicia salvo que lo pares vos). En Compose/producción, `unless-stopped` es lo habitual. (Un orquestador como [Kubernetes](kubernetes.md) tiene su propia lógica de reinicio y a menudo ignora esto.)

**Logging** — por defecto el driver `json-file` escribe los logs de `stdout`/`stderr` a disco **sin rotación**: un contenedor charlatán **llena el disco del host**. Configurá rotación (`--log-opt max-size=10m --log-opt max-file=3`) o un driver que mande a un colector. Regla de oro: tu app **loguea a stdout/stderr** (no a archivos adentro del contenedor) y Docker/el orquestador se encarga del transporte — es el enfoque 12-factor y conecta con [Observabilidad](observabilidad.md) (logs estructurados JSON) y [docker-deploy](docker-deploy.md) M11.

**Debugging de un contenedor vivo:**
- `docker logs <c>` — ver stdout/stderr.
- `docker exec -it <c> sh` — meterse a un shell adentro (si la imagen tiene shell — **distroless/scratch no**, módulo 4).
- `docker inspect <c>` — config, mounts, red, env.
- `docker stats` — CPU/memoria/IO en vivo (útil para ver si estás cerca del límite de memoria antes del OOM).
- Para imágenes sin shell, se debuggea con un **contenedor efímero de debug** que comparte los namespaces del target (`docker debug`, o `kubectl debug` en K8s).

> 🔥 **Falla en 4 capas — el contenedor Node muere por OOM en producción.**
> (1) **Qué se rompe:** bajo carga, el contenedor Node aparece `OOMKilled` y se reinicia en loop; el heap superó el `--memory` del cgroup.
> (2) **Por qué a esta escala:** V8 dimensionó el heap sin respetar el límite del contenedor; el cgroup lo mata antes de que el GC libere.
> (3) **Corto plazo:** subir `--memory` para estabilizar y observar el uso real con `docker stats`.
> (4) **Largo plazo:** fijar `--max-old-space-size` a ~75% del límite del contenedor, alinear el límite con el uso medido, y buscar leaks ([Node.js](nodejs.md)); si es trabajo pesado, sacarlo del proceso (colas/[Redis](redis.md)).

> La frase mental: **sin límites, un contenedor es vecino ruidoso: acotá con cgroups (--memory, --cpus). Superar --memory = OOM killer mata el proceso (OOMKilled), no avisa. Gotcha de Node: V8 puede dimensionar el heap sin ver el cgroup → fijá --max-old-space-size a ~75% del límite. Logueá a stdout/stderr (12-factor) y rotá los logs (json-file sin rotación llena el disco). Debuggeás con logs/exec/inspect/stats; sin shell (distroless) usás un contenedor de debug efímero.**

**Ejercicios 7**
7.1 🔁 ¿Qué pasa si un contenedor supera su `--memory`? ¿Con qué comando ves su consumo en vivo?
7.2 🧠 Explicá el gotcha de Node y el heap en contenedores, y cómo se evita.
7.3 🧠 ¿Por qué conviene que la app loguee a stdout/stderr en vez de a archivos, y qué problema trae el driver `json-file` sin configurar?

---

## Módulo 8 — Casos de uso y criterio: cuándo Docker sí, cuándo no, y el checklist

**Teoría.** Docker es mucho más que "empaquetar para deploy". Los **casos de uso** que conviene reconocer (y nombrar en una entrevista):

- **Empaquetar y desplegar** una app de forma reproducible — el caso central, cubierto de punta a punta en [Docker, deploy y CI/CD](docker-deploy.md).
- **Entorno de desarrollo reproducible** — `docker compose up` levanta tu app + Postgres + Redis idénticos para todo el equipo, sin "instalá esta versión de Postgres a mano" ([docker-deploy](docker-deploy.md) M6). Elimina el "en mi máquina funciona" también en dev.
- **Dependencias efímeras para tests** — levantar una Postgres/Redis real, descartable, por corrida de test: es exactamente lo que hace **testcontainers** ([Testing](testing.md)), y por qué la CI puede correr tests de integración sin servicios preinstalados.
- **Agentes/runners de CI** — cada job corre en un contenedor limpio y reproducible ([docker-deploy](docker-deploy.md) M9).
- **Jobs y CLIs one-off** — correr una herramienta sin instalarla en tu máquina (`docker run --rm <img> <cmd>`): un linter, un cliente de DB, un generador. El `--rm` la borra al terminar.
- **Microservicios y sidecars** — cada servicio en su contenedor, desplegables y escalables por separado; un *sidecar* (proxy, agente de logs) junto al principal — el modelo que [Kubernetes](kubernetes.md) lleva a escala.

**Cuándo Docker NO es la respuesta** (el criterio senior de no sobre-ingeniería):
- Un **sitio estático** o una función simple: hosteás en un CDN o una plataforma serverless; el contenedor es overhead.
- Cuando el equipo/proyecto es chico y la plataforma ya te abstrae el runtime (algunas PaaS buildean sin que escribas Dockerfile).
- Como **frontera de seguridad para código hostil** (módulo 1): el contenedor comparte kernel; para aislar cargas no confiables usá VMs o runtimes tipo gVisor/Firecracker.

**El checklist de una imagen de producción** (consolidando el módulo):
1. **Multi-stage**, runtime sin toolchain ([docker-deploy](docker-deploy.md) M3).
2. **Base** slim/distroless, **pinada por digest** y LTS vigente (M2, M4).
3. **`USER` no-root**; runtime con `--read-only`+tmpfs, `--cap-drop=ALL`, `no-new-privileges` (M5).
4. **Sin secretos** en la imagen; inyectados al correr / BuildKit secrets (M3, M5).
5. **`.dockerignore`** y orden de capas para caché; cache mounts (M3, M4).
6. **Healthcheck** + **graceful shutdown** + init/PID 1 ([docker-deploy](docker-deploy.md) M7).
7. **Límites de recursos** y `--max-old-space-size` acorde (M7).
8. **Escaneo de CVEs** + SBOM/firma en CI (M5).
9. **Logs a stdout** estructurados, con rotación (M7, [Observabilidad](observabilidad.md)).

> La frase mental de cierre: **Docker no es solo "deploy": es entorno de dev reproducible, deps efímeras para tests (testcontainers), runners de CI, CLIs/jobs one-off y microservicios/sidecars. No lo uses para un sitio estático (overhead) ni como frontera de seguridad para código hostil (comparte kernel → VM/gVisor). El checklist de una imagen de prod: multi-stage + base mínima pinada por digest + no-root/read-only/cap-drop + cero secretos + caché + healthcheck/graceful shutdown + límites + escaneo/firma + logs a stdout.**

**Ejercicios 8**
8.1 🔁 Nombrá cuatro casos de uso de Docker más allá de "empaquetar para deploy".
8.2 🧠 Dá dos situaciones donde meter Docker sería sobre-ingeniería o directamente la herramienta equivocada.
8.3 🧠 Enumerá de memoria al menos seis ítems del checklist de una imagen de producción y por qué está cada uno.

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** `docker` (CLI): manda órdenes por el socket, no corre nada. `dockerd` (daemon): gestiona imágenes/redes/volúmenes y delega los contenedores. `containerd`: runtime de alto nivel que baja imágenes y supervisa contenedores. `runc`: runtime de bajo nivel (OCI) que le pide al **kernel** crear el contenedor con namespaces/cgroups y se retira. La cadena: `docker → dockerd → containerd → runc → kernel`.

**1.2** **Namespaces = aislamiento** (qué ve el proceso: sus procesos con `pid`, su red con `net`, su FS con `mnt`, su hostname con `uts`, etc.). **cgroups = límites** (cuánto puede usar: CPU, memoria, IO). Juntos: el contenedor se cree solo en la máquina (namespaces) y no puede acaparar recursos (cgroups).

**1.3** Porque el contenedor **comparte el kernel del host** y solo crea namespaces/cgroups (operación de milisegundos), mientras que la VM **arranca un kernel y un SO propios** sobre hardware virtualizado (minutos). Esa misma razón lo hace **más débil como frontera de seguridad**: al compartir kernel, un exploit del kernel desde adentro puede **escapar** al host; una VM, con su kernel aislado, no.

**2.1** Una **capa** es un diff del filesystem que genera cada instrucción que lo toca (`RUN`, `COPY`, `ADD`); son de **solo lectura** y se apilan. La **capa de escritura** del contenedor es una capa fina y efímera que Docker pone encima al correr: los cambios se escriben ahí con **copy-on-write** (se copia el archivo modificado sin tocar las capas de abajo) y se **pierden** al recrear el contenedor.

**2.2** **Sí, el secreto está en la imagen.** Borrarlo con `rm` en una capa posterior solo lo **tapa** en la vista final (union FS), pero el archivo **sigue** en la capa anterior, que es inmutable y parte de la imagen. Cualquiera puede inspeccionar esa capa y recuperarlo. Por eso un secreto no se "borra" de una imagen: **no debe entrar nunca** (usar BuildKit secrets para build, inyección en runtime para lo demás).

**2.3** Un **tag** (`miapp:latest`, `node:22`) es un **puntero mutable**: puede reapuntar a otro contenido cuando se re-publica (`latest` es el caso extremo, un blanco móvil). Un **digest** (`...@sha256:abc`) es **inmutable**: identifica exactamente ese contenido (content-addressable). Para **reproducibilidad** (que el build de hoy y el de dentro de seis meses usen la misma base) y **trazabilidad** (saber y poder volver a una imagen exacta), pineás por digest.

**3.1** **BuildKit** es el motor de build de Docker (arma el grafo de dependencias, paraleliza etapas, habilita features nuevas), **default desde Docker 23 (2023)** ⚠️. Features: **cache mounts** (`RUN --mount=type=cache`), **build secrets** (`RUN --mount=type=secret`), mejor caché y **multi-arch** con buildx. (Bastan dos.)

**3.2** Cachear la **capa** de `npm ci` (ordenando `COPY package*.json` antes del código) reusa el paso completo **solo si `package*.json` no cambió**; si cambia **una** dependencia, la capa se invalida y `npm ci` reinstala **todo** desde cero. Un **cache mount** monta el **directorio de cache de npm** de forma persistente **entre builds**, así que aunque la capa se invalide, npm **reusa lo ya descargado** y solo baja lo nuevo. Resuelve el costo de re-descargar todo el árbol ante cualquier cambio de dependencias.

**3.3** Porque hoy conviven **amd64** (servidores x86, muchos CI) y **arm64** (Macs M-series, AWS Graviton), y tu imagen tiene que correr en ambas. Lo hace **`docker buildx`** (`--platform linux/amd64,linux/arm64`), que publica un **manifest list** bajo un tag. Quien hace `pull` recibe **automáticamente** la variante de su arquitectura.

**4.1** **`node:lts-slim`** (Debian recortado, glibc): máxima compatibilidad con módulos nativos, tamaño medio — default seguro. **`node:lts-alpine`** (musl, ~5 MB base): mínima, pero musl puede romper dependencias nativas / dar sorpresas de DNS. **`distroless`**: superficie de ataque mínima (sin shell ni gestor de paquetes) y no-root, pero **no podés `exec` a debuggear** (sin shell). (Extra: `scratch`, vacía, para binarios estáticos.)

**4.2** Porque cada `RUN` es una **capa inmutable**: si instalás en una capa y el cache/artefactos quedan ahí, un `rm` en un `RUN` **posterior** solo los **tapa** en la vista final, pero el peso **sigue** en la capa anterior (módulo 2). Se hace bien **instalando y limpiando en el MISMO `RUN`**: `RUN apt-get update && apt-get install -y --no-install-recommends ... && rm -rf /var/lib/apt/lists/*`, así lo borrado nunca llega a materializarse como capa.

**4.3**
```dockerfile
# syntax=docker/dockerfile:1

# Etapa build: toolchain completo, compila el TS
FROM node:lts-slim AS build
WORKDIR /app
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci   # deps completas (incl. dev) para compilar
COPY . .
RUN npm run build

# Etapa runtime: liviana, solo prod + el compilado, no-root
FROM node:lts-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY package*.json ./
RUN --mount=type=cache,target=/root/.npm npm ci --omit=dev
COPY --from=build /app/dist ./dist
USER node                       # mínimo privilegio: no-root
EXPOSE 3000
CMD ["node", "dist/main.js"]    # forma exec: node recibe las señales
```
Qué cambió respecto del ingenuo: **multi-stage** (la imagen final no arrastra el toolchain ni las devDependencies); **base `slim`** en vez de `node:22` completa (más chica y compatible); **`npm ci`** (reproducible desde el lock) en vez de `npm install`, con **cache mount**; **orden de caché** (`package*.json` antes del código); **`USER node`** (no-root); **forma exec** de `CMD`. Para producción real se le suma el init/`tini`, el `HEALTHCHECK` y el pin por digest ([docker-deploy](docker-deploy.md) M3, M7).

**5.1** (tres cualesquiera) **`--read-only`** (+ tmpfs): el atacante no puede escribir el filesystem → no deja binarios ni modifica la app. **`--cap-drop=ALL`** (+ solo las necesarias): quita capabilities del kernel que la app no usa → menos superficie para escalar/escapar. **`--security-opt=no-new-privileges`**: impide escalar privilegios (p. ej. por setuid). **`USER` no-root**: si comprometen el proceso, no es root. **No `--privileged` / no montar `docker.sock`**: evita dar control del host.

**5.2** Porque `/var/run/docker.sock` es la **API del daemon**, que corre como **root en el host**. Quien la controla puede crear un contenedor privilegiado que monte el filesystem del host y hacer lo que quiera → montar el socket dentro de un contenedor es, de facto, **darle root en el host** a ese contenedor. Si lo comprometen, comprometen la máquina entera.

**5.3** **Escanear** una imagen (Trivy/Docker Scout/Grype) es analizar sus paquetes contra bases de **CVEs conocidas** para detectar vulnerabilidades antes de desplegar. El **SBOM** (syft) es el inventario de **todo lo que contiene** la imagen (paquetes y versiones): sirve para auditar y para reaccionar rápido cuando sale una CVE nueva ("¿tengo esta librería?"). La **firma** (cosign/Sigstore) garantiza **integridad y procedencia**: el que despliega verifica que la imagen es la que vos publicaste y **no fue manipulada**. Escaneo protege de vulnerabilidades conocidas; SBOM + firma protegen la **supply chain** (imágenes alteradas, dependencias comprometidas).

**6.1** **Named volume**: gestionado por Docker, para **datos que persisten** (la base de datos) — portable y con backup. **Bind mount**: monta una **ruta del host**, para **desarrollo** (hot reload del código). **tmpfs**: en **memoria**, para datos **efímeros/sensibles** que no deben tocar disco (y compañero de `--read-only`).

**6.2** En **desarrollo** el bind mount hace que el contenedor vea tus cambios de código al instante (hot reload) sin reconstruir la imagen — productivo. En **producción** es malo porque **acopla el contenedor al filesystem del host** y **rompe la inmutabilidad**: lo que corre ya no es la imagen que probaste y versionaste, sino lo que haya en esa ruta del host → deja de ser reproducible y trazable.

**6.3** Porque en una red bridge **definida por el usuario** (la que arma Compose) Docker provee un **DNS interno** que resuelve los servicios **por su nombre**: `db` es el hostname del contenedor Postgres en esa red. `localhost` apuntaría al **propio** contenedor de la app (su namespace de red), no al de la base. Lo resuelve el **DNS embebido de Docker** en redes definidas por el usuario (en la red bridge *default* eso no existe; por eso se usa siempre una red propia). **Compose crea esa red de usuario automáticamente** para tu proyecto — por eso ahí el `db:5432` "simplemente funciona" sin que la configures a mano.

**7.1** Si supera `--memory`, el **OOM killer del kernel mata el proceso** (el contenedor queda `OOMKilled`, y con restart policy se reinicia). Es muerte súbita, sin warning. El consumo en vivo se ve con **`docker stats`** (CPU/memoria/IO por contenedor).

**7.2** El heap de V8 tiene un tope (`--max-old-space-size`) y, históricamente, Node **no leía el límite de memoria del cgroup**: dimensionaba el heap según la RAM del **host** (enorme), así que en un contenedor con poca `--memory` el heap crecía hasta que el **cgroup lo mataba por OOM antes de que V8 hiciera GC**. Se evita **fijando `--max-old-space-size` a ~75% del límite** del contenedor (`NODE_OPTIONS=--max-old-space-size=384` con `--memory=512m`), dejando margen para memoria no-heap. Node moderno detecta mejor el cgroup ⚠️, pero fijarlo explícito sigue siendo lo seguro.

**7.3** Porque **stdout/stderr desacopla el logging del contenedor**: la app no gestiona archivos, rotación ni rutas; Docker/el orquestador captura el flujo y lo transporta a donde sea (12-factor). Loguear a **archivos adentro** del contenedor los pierde al recrearlo y complica el transporte. El problema del driver **`json-file` sin configurar** es que escribe a disco **sin rotación** → un contenedor charlatán **llena el disco del host**; se arregla con `max-size`/`max-file` o un driver que mande a un colector.

**8.1** (cuatro cualesquiera) **Entorno de dev reproducible** (compose: app+DB+Redis iguales para el equipo); **dependencias efímeras para tests** (Postgres/Redis descartable por corrida — testcontainers); **runners/agentes de CI** (job en contenedor limpio); **CLIs/jobs one-off** (`docker run --rm` para correr una herramienta sin instalarla); **microservicios y sidecars** (cada servicio/anexo en su contenedor).

**8.2** (dos cualesquiera) Un **sitio estático** o función simple: un CDN/serverless lo sirve sin el overhead de un contenedor. Un **proyecto/equipo chico** donde la PaaS ya abstrae el runtime (buildea sin Dockerfile). Como **frontera de seguridad para código hostil/no confiable**: el contenedor comparte kernel con el host, así que no aísla como una VM → usar VMs o runtimes tipo gVisor/Firecracker/Kata.

**8.3** (seis cualesquiera, con su porqué) **Multi-stage** (imagen final sin toolchain → chica y con menos superficie). **Base mínima pinada por digest y LTS** (compatibilidad + reproducibilidad + menos CVEs). **`USER` no-root** (limita el daño de una RCE). **`--read-only`+tmpfs / `--cap-drop=ALL` / `no-new-privileges`** (hardening en runtime). **Sin secretos en la imagen** (no se pueden borrar de una capa; se filtran). **`.dockerignore` + orden de capas + cache mounts** (builds rápidos y sin filtrar basura). **Healthcheck + graceful shutdown + init PID 1** (deploy sin downtime, señales bien propagadas). **Límites de recursos + `--max-old-space-size`** (evita vecino ruidoso y OOM). **Escaneo de CVEs + SBOM/firma** (supply chain). **Logs a stdout estructurados con rotación** (observabilidad sin llenar disco).

---

> **Para seguir.** Las fuentes de verdad: la **documentación oficial de Docker** (Build/BuildKit, storage, network, `docker run` reference), la **OCI** (image-spec y runtime-spec) y las guías de **seguridad** (Docker security, CIS Docker Benchmark, Trivy/Docker Scout/Sigstore). Re-verificá lo marcado con ⚠️ (BuildKit default desde Docker 23; la detección de cgroups por Node mejora entre versiones — confirmá el comportamiento de tu major LTS). Los puentes en el hub: [Docker, deploy y CI/CD](docker-deploy.md) es el **ciclo a producción** (empaquetar, CI/CD, deploy, healthcheck, graceful shutdown, PID 1) — este módulo es el **cómo funciona y las mejores prácticas** debajo de eso; [Kubernetes](kubernetes.md) es la **orquestación a escala** de estos contenedores (donde los límites, las sondas y las redes que viste acá se vuelven declarativos); [Testing](testing.md) usa Docker para las **dependencias efímeras** (testcontainers); [Observabilidad](observabilidad.md) es el destino de tus **logs a stdout**; [Node.js por dentro](nodejs.md) es el porqué del gotcha de memoria; y [Reverse proxy y capa web](reverse-proxy.md) es lo que suele ir delante de tus contenedores. **Y hacia dónde va el tema:** el ecosistema se mueve a **builds sin daemon y rootless** (BuildKit/buildah), imágenes **distroless/mínimas** por defecto, **SBOM y firma** como parte estándar del pipeline (supply chain), y **runtimes con más aislamiento** (gVisor, Kata, Firecracker) para correr contenedores con seguridad cercana a la de una VM.
