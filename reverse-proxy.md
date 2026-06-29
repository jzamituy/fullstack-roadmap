# Reverse proxy y la capa web delante de tu app

**nginx, TLS, load balancing y el desenredo proxy / gateway / ingress · configs reales · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo es lo que va **delante** de tu Node/Express en producción — la capa que casi nadie te enseña ordenada y que sí o sí aparece cuando desplegás algo real. El foco: **entender los conceptos Y escribir config de nginx que funcione.** Las configs de acá son reales y copiables; entendé cada línea, no la pegues a ciegas.

**Lo que asumimos.** Sabés qué es una app Node/[Express](express.md) escuchando en un puerto, Docker básico, y HTTP (request, response, status codes, headers, HTTPS por arriba). Ayuda haber desplegado algo alguna vez y haberte preguntado "¿y cómo le pongo HTTPS a esto?". Justamente.

> ⚠️ **Nota sobre datos volátiles.** Las configs de nginx, los tiempos de los certificados de Let's Encrypt y las versiones de Caddy/Traefik se mueven. Todo lo marcado con ⚠️ verificalo en la fuente oficial (`nginx.org`, `letsencrypt.org`, `caddyserver.com`, `doc.traefik.io`) antes de llevarlo a producción. En particular, los **tiempos de los certificados se están acortando año a año** (módulo 5): la renovación automática ya no es opcional.

**Tipos de ejercicio:** 🔁 recordar · 🧠 criterio/análisis · ✍️ implementación (escribir/corregir config).

**Índice de módulos**
1. Forward proxy vs reverse proxy: quién esconde a quién
2. Por qué un reverse proxy delante de tu Node
3. nginx como reverse proxy: la config real
4. Load balancing: repartir entre varias instancias
5. Terminación TLS/HTTPS y certificados
6. En el borde: compresión, caché, rate limiting y headers
7. El gran desenredo: proxy vs load balancer vs API gateway vs Ingress vs CDN
8. Alternativas modernas: Caddy y Traefik
9. El lado de Express: `trust proxy` y sus trampas
10. El criterio: ¿lo hacés vos o lo hace la plataforma?

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Forward proxy vs reverse proxy: quién esconde a quién

**Teoría.** Un **proxy** es un intermediario que se para en el medio de una comunicación. La pregunta que lo define todo es: **¿en el medio de qué lado?**

- **Forward proxy:** se para **delante del cliente** y actúa **en nombre del cliente**. Esconde **quién pregunta**. Ejemplo clásico: el proxy de egress de una empresa por el que salen todos los navegadores de los empleados (filtra, cachea, registra), o el típico de la variable `HTTP_PROXY`.
- **Reverse proxy:** se para **delante de los servidores** y actúa **en nombre del servidor**. Esconde **quién responde**. El cliente cree que habla directo con él, pero por detrás hay uno o varios backends. Ejemplos: nginx/Caddy/Traefik delante de tu Node, un CDN, un AWS ALB, un Ingress de Kubernetes.

La frase que lo fija: **forward proxy = delante del cliente (esconde quién pregunta); reverse proxy = delante de los servidores (esconde quién responde).**

La analogía: el reverse proxy es la **recepción de un edificio de oficinas**. El visitante (el cliente) habla con recepción; no sabe —ni le importa— en qué piso está cada equipo, ni cuántos equipos hay. Recepción **enruta** al piso correcto, **filtra** quién entra, **recibe la correspondencia** (los certificados, módulo 5) y, si un equipo se mudó de piso, el visitante ni se entera. El forward proxy, en cambio, es **el cadete que sale a hacer los mandados por vos**: está de tu lado (del cliente), sale al mundo en tu nombre.

A nosotros, como gente de backend, **el que nos importa es el reverse proxy** — todo este módulo es sobre eso. El forward proxy es más cosa de redes corporativas y privacidad.

La frase mental: **proxy = intermediario; la pregunta es de qué lado. Forward = delante del cliente, esconde quién pregunta. Reverse = delante de los servidores, esconde quién responde. Tu Node vive detrás de un reverse proxy: la recepción del edificio.**

**Ejercicios 1**
1.1 🔁 Definí forward proxy y reverse proxy en una frase cada uno. ¿Qué esconde cada uno?
1.2 🧠 nginx delante de tu API Node, ¿es forward o reverse proxy? ¿Por qué?
1.3 🧠 Un CDN, un AWS ALB y un Ingress de Kubernetes, ¿qué tienen en común con nginx en este contexto?

---

## Módulo 2 — Por qué un reverse proxy delante de tu Node

**Teoría.** ¿Por qué no exponés tu Node directo a internet y listo? Técnicamente **podés** (Node puede escuchar en el `:443`), pero casi nunca conviene. El reverse proxy te resuelve un paquete de cosas que tu app no debería estar haciendo:

- **Terminación TLS/HTTPS.** El HTTPS "muere" en el proxy; de ahí a tu Node va HTTP plano por la red privada. Un solo lugar para manejar certificados (módulo 5).
- **Servir estáticos.** nginx sirve archivos del disco **muchísimo más rápido** que Node empujándolos por el event loop.
- **Buffering de clientes lentos.** Y acá hay una razón **específica de Node**: Node es **single-thread por proceso**, un solo event loop. Si un cliente con internet de pueblo manda los bytes de a goteo, no querés que ese socket lento tenga ocupado tu event loop. nginx **bufferea** el request/response completo y le habla a Node en ráfagas rápidas — el event loop no queda rehén de la conexión lenta. (`proxy_buffering` viene **on** por defecto.)
- **Load balancing** entre varias instancias de Node (módulo 4) — porque un proceso de Node usa un solo core.
- **Seguridad.** Escondés el servidor de app, centralizás headers, no exponés el stack de Node directo.
- **Compresión, caché y rate limiting** en el borde (módulo 6).

El consejo "**no expongas Node directo a internet**" es sólido para self-hosting/VPS. Pero ojo con el matiz: **no es absoluto.** Si estás en Vercel/Render/Fly/ALB/Ingress, **la plataforma ya corre un proxy endurecido** delante tuyo — sumarle tu propio nginx es redundante (módulo 10). El consejo es sobre **quién corre el borde**, no sobre si Node técnicamente puede escuchar en el 443.

La frase mental: **el reverse proxy le saca a tu Node el trabajo que no debería hacer: TLS, estáticos, bufferear clientes lentos (clave por el event loop single-thread), balancear, comprimir y cachear. No exponés Node directo… salvo que una plataforma ya te dé ese borde.**

**Ejercicios 2**
2.1 🔁 Nombrá cuatro cosas que un reverse proxy le saca de encima a tu app Node.
2.2 🧠 ¿Por qué el "buffering de clientes lentos" es especialmente importante en Node y no tanto en un server multi-thread? (pista: event loop)
2.3 🧠 "No expongas Node directo a internet." ¿En qué caso este consejo NO aplica, y por qué?

---

## Módulo 3 — nginx como reverse proxy: la config real

**Teoría.** Vamos al hueso. El reverse proxy más usado del mundo es **nginx**, y la directiva clave es **`proxy_pass`**. La config mínima que reenvía todo a tu Node corriendo en `localhost:3000`:

```nginx
server {
    listen 80;
    server_name app.ejemplo.com;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

¿Por qué esos cuatro `proxy_set_header`? Porque **por defecto nginx NO pasa varios datos del request original** — y tu app los necesita:
- **`Host $host`** — reenvía el hostname real que pidió el cliente (por defecto nginx mandaría el del upstream, no el original).
- **`X-Real-IP $remote_addr`** — la IP del cliente (si no, tu app vería la IP del proxy).
- **`X-Forwarded-For $proxy_add_x_forwarded_for`** — agrega la IP del cliente a la cadena XFF (el valor idiomático).
- **`X-Forwarded-Proto $scheme`** — le dice a tu app si el request **original** fue `http` o `https` (crítico después de terminar TLS — módulos 5 y 9).

Esos headers son los que tu Express va a leer para saber **quién es el cliente real** y **si vino por HTTPS** (módulo 9). Sin ellos, tu app cree que todos los requests vienen del proxy por HTTP.

El bloque **`upstream`** le pone nombre a tu backend (y es la base del load balancing del módulo 4):

```nginx
upstream node_app {
    server 127.0.0.1:3000;
}
server {
    listen 80;
    location / {
        proxy_pass http://node_app;
    }
}
```

**WebSockets** (tu [tiempo real](tiempo-real.md)) necesitan un handshake especial de "upgrade". El patrón recomendado usa un `map`:

```nginx
http {
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        location /ws/ {
            proxy_pass http://node_app;
            proxy_http_version 1.1;                    # ⚠️ ver nota de versión
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
        }
    }
}
```

> ⚠️ Nota de versión: el default de `proxy_http_version` pasó a `1.1` **desde nginx 1.29.7** (antes era `1.0`). En versiones modernas ya no es estrictamente necesaria esa línea para WebSockets, pero **dejala**: la mayoría de los nginx instalados a mediados de 2026 todavía son anteriores, y ser explícito no molesta.

La frase mental: **`proxy_pass` reenvía a tu Node; los cuatro `proxy_set_header` le dicen a tu app quién es el cliente real y si vino por HTTPS — sin ellos, solo ve al proxy.** (WebSockets piden el upgrade explícito.)

**Ejercicios 3**
3.1 🔁 ¿Cuál es la directiva de nginx que reenvía requests a tu backend? ¿Para qué sirve el bloque `upstream`?
3.2 🧠 ¿Por qué hay que setear `proxy_set_header Host $host`? ¿Qué pasaría sin él?
3.3 ✍️ Escribí un bloque `server` de nginx que escuche en el 80, responda para `api.midominio.com`, y haga reverse proxy a un Node en `127.0.0.1:4000`, pasando los cuatro headers `X-Forwarded`/`Host`/`Real-IP`.

---

## Módulo 4 — Load balancing: repartir entre varias instancias

**Teoría.** Acá conectamos con algo que ya sabés de [Node por dentro](nodejs.md): **Node es single-thread por proceso**, un proceso usa **un core**. Entonces, para aprovechar una máquina de 8 cores o para aguantar más carga, corrés **N instancias** de tu app (con el módulo `cluster` de Node, con PM2, o con N contenedores), y el reverse proxy **reparte el tráfico** entre ellas. Eso es **load balancing**.

Volvé a la recepción del edificio (módulo 1): es la misma recepción que, en vez de mandar a todos los visitantes al mismo escritorio, los **reparte entre varios escritorios idénticos** del mismo equipo para que ninguno se sature. En nginx, eso es el `upstream` con varios `server` y un **método de balanceo**:

```nginx
upstream node_app {
    # round-robin es el DEFAULT (sin directiva): reparte por turnos
    # least_conn;                 # manda al que tiene menos conexiones activas
    # ip_hash;                    # "pega" cada cliente a un backend por su IP

    server 127.0.0.1:3001 weight=5;                  # pesa más: recibe más tráfico
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3003 backup;                    # solo si los demás caen

    keepalive 32;                 # reusa conexiones al backend (necesita http/1.1)
}
```

> ⚠️ Para que ese `keepalive` al upstream realmente reuse conexiones, en el `location` necesitás `proxy_http_version 1.1;` **y** `proxy_set_header Connection "";` (vaciar el header `Connection`, que por defecto sería `close`). Es el mismo HTTP/1.1-hacia-el-backend que piden los WebSockets (módulo 3): desde nginx 1.29.7 el `proxy_http_version 1.1` ya es el default, pero el `Connection ""` para keepalive lo seguís poniendo.

Los métodos principales (todos en el nginx open-source): **round-robin** (default, por turnos), **`least_conn`** (al menos cargado), **`ip_hash`** (sticky: el mismo cliente siempre al mismo backend, útil si tenés estado en memoria — aunque lo mejor es **no** tener estado, y guardarlo en [Redis](redis.md)).

> ⚠️ **Health checks — ojo con esto que confunde a todos:**
> - **Passive** (gratis, open-source): `max_fails` + `fail_timeout`. nginx marca un backend como caído **después** de que fallen requests reales.
> - **Active** (probes periódicos con la directiva `health_check`, más `slow_start`, etc.): **solo en nginx Plus (la versión paga).** Mucha gente asume que los health checks activos son gratis y **no lo son**.

Esto es la versión **a mano** de lo que un [Kubernetes Service/Ingress](kubernetes.md) o un **AWS ALB** hacen automáticamente (el ALB sí trae health checks activos de fábrica). Lo armás vos en un VPS; te lo dan hecho en una plataforma gestionada (módulo 10).

La frase mental: **un proceso de Node = un core, así que corrés N instancias y el proxy las balancea (round-robin por default, `least_conn` si querés). Health checks pasivos gratis; los activos, solo en nginx Plus. K8s y el ALB te dan esto automático.**

**Ejercicios 4**
4.1 🔁 ¿Por qué necesitás varias instancias de Node para aprovechar una máquina multi-core? ¿Qué hace el reverse proxy con ellas?
4.2 🧠 ¿Qué diferencia hay entre health checks passive y active en nginx? ¿Cuál es la trampa de costo?
4.3 ✍️ Escribí un `upstream` llamado `api` que balancee con `least_conn` entre tres instancias (`127.0.0.1:3001/3002/3003`), donde la tercera sea de `backup`.

---

## Módulo 5 — Terminación TLS/HTTPS y certificados

**Teoría.** Este módulo tiene dos partes que conviene no mezclar: **(5a)** cómo terminás TLS en nginx (la config), y **(5b)** de dónde sale el certificado y cómo lo renovás. Vamos en orden.

### 5a. Terminar TLS en nginx

"Terminar TLS en el proxy" significa: **el HTTPS se desencripta en nginx**, y de nginx a tu Node va **HTTP plano** por la red privada de confianza. Tu Node nunca ve el certificado. La analogía: la correspondencia llega **lacrada** (cifrada) a recepción; recepción la **abre** y la distribuye en sobre normal por los pisos internos.

El bloque HTTPS + el redirect de HTTP a HTTPS:

```nginx
# Todo el tráfico HTTP (80) se redirige a HTTPS (443)
server {
    listen 80;
    server_name app.ejemplo.com;
    return 301 https://$host$request_uri;
}

# El server real, en HTTPS
server {
    listen 443 ssl;                          # ⚠️ la forma con parámetro `ssl`
    server_name app.ejemplo.com;

    ssl_certificate     /etc/letsencrypt/live/app.ejemplo.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.ejemplo.com/privkey.pem;
    ssl_protocols       TLSv1.2 TLSv1.3;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

> ⚠️ **No uses `ssl on;`** (lo vas a ver en tutoriales viejos): esa directiva se **deprecó en nginx 1.15.0 y se eliminó en 1.25.1**. La forma correcta es el parámetro `ssl` en el `listen 443 ssl;`.

### 5b. De dónde sale el cert: Let's Encrypt y la renovación

El certificado lo sacás **gratis** con **Let's Encrypt**, usando el cliente **Certbot**:
- `sudo certbot --nginx` → obtiene el certificado **y te edita la config de nginx** para instalarlo.
- `certbot certonly` → solo te da el cert, vos manejás nginx.
- **Renovación automática:** Certbot ya deja configurada una tarea (systemd timer o cron) que corre `certbot renew` y renueva cuando se acerca el vencimiento. Probala con `certbot renew --dry-run`.

> ⚠️ **El dato más importante de 2026: los certificados duran cada vez menos.** La tendencia de la industria (CA/Browser Forum) es bajar el máximo de vida de un certificado año a año —de los ~398 días de antes hacia **~47 días** hacia fines de la década—, y Let's Encrypt ya ofrece certs de **6 días** e IP-certs (GA en enero de 2026). Las cifras y fechas exactas se mueven (⚠️ verificalas), pero el takeaway es el que importa y **no cambia: la renovación automática ya no es opcional.** Si tu renovación no está automatizada, el día que el cert venza tenés una **caída**. La renovación manual está muerta.

La frase mental: **terminás TLS en el proxy: HTTPS muere en nginx, HTTP plano hacia Node por la red privada. `listen 443 ssl` (nunca `ssl on;`), certs gratis de Let's Encrypt vía Certbot, y renovación automática SÍ o SÍ — los certs cada año duran menos.**

**Ejercicios 5**
5.1 🔁 ¿Qué significa "terminar TLS en el proxy"? ¿Cómo viaja el tráfico de nginx a Node?
5.2 🧠 ¿Por qué la renovación automática de certificados pasó de "buena práctica" a "obligatoria" en 2026?
5.3 ✍️ Escribí los dos bloques `server`: uno que redirija todo el `:80` a HTTPS, y otro en `:443 ssl` con certificados de Let's Encrypt que haga proxy a `127.0.0.1:3000`.

---

## Módulo 6 — En el borde: compresión, caché, rate limiting y headers

**Teoría.** Como el reverse proxy ve **todo** el tráfico, es el lugar natural para varias tareas transversales. Las cuatro que más vas a tocar:

**Compresión (gzip).** Achica las respuestas de texto antes de mandarlas:
```nginx
gzip            on;                 # default: off
gzip_comp_level 5;
gzip_min_length 1000;               # no comprimas cositas chiquitas
gzip_types      text/plain text/css application/json application/javascript;  # text/html SIEMPRE está incluido cuando gzip on; (no se lista); estos son los adicionales
```
⚠️ **Brotli no está en el nginx core** — es un módulo de terceros (`ngx_brotli`) que hay que compilar/cargar aparte. Si ves `brotli on;` en un ejemplo, asumí que tienen ese módulo.

**Rate limiting** (defensa contra abuso/DoS básico). Usa un "balde con goteo":
```nginx
http {
    limit_req_zone $binary_remote_addr zone=milimite:10m rate=10r/s;

    server {
        location /api/ {
            limit_req zone=milimite burst=20 nodelay;
            proxy_pass http://node_app;
        }
    }
}
```
Esto limita a 10 requests por segundo por IP, con un colchón ("burst") de 20.

**Security headers** (endurecé las respuestas):
```nginx
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
```
> ⚠️ **Trampa clásica de `add_header`:** los `add_header` se heredan **solo si el nivel actual NO define ninguno propio**. Si ponés un `add_header` dentro de un `location`, **se descartan en silencio todos los heredados** del `server`/`http`. Tenés que **redefinirlos** en el nivel donde agregás uno nuevo. (Las versiones más nuevas de nginx traen una opción para fusionar en vez de pisar ⚠️, pero no cuentes con que esté — aprendé a redefinir.)

**Caché** (`proxy_cache`) para no pegarle a Node por cosas que no cambian. Y un punto de criterio: **¿CORS en el proxy o en la app?** Respuesta: **en la app** ([Express](express.md) con el middleware `cors`), porque el CORS suele depender del contexto del request (orígenes dinámicos, credenciales) y la app conoce su propio ruteo. El proxy solo es razonable para CORS estático, y aun así la trampa del `add_header` te muerde.

La frase mental: **el proxy ve todo, así que ahí hacés compresión (gzip; brotli no es core), rate limiting (balde con goteo por IP), security headers (ojo: un `add_header` en un `location` pisa los heredados) y caché. El CORS, mejor en la app.**

**Ejercicios 6**
6.1 🔁 Nombrá tres tareas transversales que tiene sentido hacer en el reverse proxy.
6.2 🧠 Pusiste security headers en el bloque `http` y andan, pero al agregar un `add_header` dentro de un `location` desaparecen. ¿Qué pasó y cómo lo arreglás?
6.3 🧠 ¿Conviene poner el CORS en el proxy o en la app Express? Justificá.
6.4 ✍️ Escribí un `location /api/` con rate limiting de 5 requests por segundo por IP (burst de 10) que haga proxy a `node_app`. Incluí la directiva `limit_req_zone` donde corresponde.
6.5 ✍️ Tenés tres security headers (`Strict-Transport-Security`, `X-Content-Type-Options`, `X-Frame-Options`) en el bloque `server`. Necesitás agregar `Cache-Control "no-store"` **solo** en `location /api/`. Escribí ese `location` cuidando la trampa de herencia del `add_header` (los tres del `server` deben seguir aplicándose dentro del `location`).

---

## Módulo 7 — El gran desenredo: proxy vs load balancer vs API gateway vs Ingress vs CDN

**Teoría.** Acá está el módulo que más vale: **toda esta familia de cosas se superpone, y por eso todo el mundo las confunde.** Vamos a ordenarlas. La clave mental: **el reverse proxy es el género; los demás son especies de reverse proxy, diferenciadas por *qué trabajo extra* enfatizan.**

| Cosa | Qué es | Trabajo principal | Relación |
|---|---|---|---|
| **Reverse proxy** | Servidor delante de los backends | Reenviar, esconder servers, terminar TLS, bufferear, cachear | **La pieza base.** Todo lo de abajo es un reverse proxy con extras. |
| **Load balancer** | Reparte tráfico entre varios backends | Distribuir carga, health-check, failover | Un reverse proxy *puede* balancear (el `upstream` de nginx). Un LB dedicado (AWS ALB/NLB, HAProxy) se enfoca en eso. Acá vive la distinción L4 (TCP) vs L7 (HTTP). |
| **API gateway** | Reverse proxy L7 especializado para APIs | Routing + **auth**, rate limiting, **transformación** de request/response, API keys, agregación | Un API gateway *es* un reverse proxy + temas de API. Ej.: AWS API Gateway, Kong. |
| **Ingress (K8s)** | La forma nativa de exponer servicios en Kubernetes | Reglas de routing declarativas → ejecutadas por un **ingress controller** (¡que suele ser nginx o Traefik!) | El Ingress es la *abstracción/config*; el controller es el reverse proxy real. (Gateway API es el sucesor — lo ves en el módulo de [Kubernetes](kubernetes.md).) |
| **CDN** | Cachés distribuidas geográficamente en el borde | Cachear y servir contenido cerca del usuario, absorber tráfico, mitigar DDoS | Un nodo de CDN también es un reverse proxy, pero lo que lo define es la **distribución geográfica + caché de borde**. Ej.: Cloudflare, CloudFront. |

El modelo mental: **reverse proxy = el género; load balancer, API gateway, ingress controller y nodo de CDN son todas especies de reverse proxy**, diferenciadas por el trabajo que enfatizan (distribuir, temas de API, config declarativa de k8s, caché geográfica).

Los puentes con lo que ya tenés en el hub:
- **[Kubernetes (Ingress/Gateway)](kubernetes.md):** el Ingress es el front-end declarativo; por debajo levanta un nginx/Traefik haciendo **exactamente** lo que acá configurás a mano.
- **[AWS](aws.md):** el **ALB** es el reverse-proxy/load-balancer gestionado de AWS (con health checks activos y TLS vía ACM); el **API Gateway** es su API gateway gestionado. Le pagás a AWS para que corra la capa nginx por vos.

La frase mental: **reverse proxy es el género; load balancer, API gateway, Ingress controller y CDN son especies que enfatizan distinto (distribuir / temas de API / declarativo en k8s / caché geográfica). El Ingress de k8s y el ALB de AWS son esta misma capa, gestionada.**

**Ejercicios 7**
7.1 🔁 ¿Por qué se dice que "el reverse proxy es el género y los demás son especies"? Dá un ejemplo de cada especie.
7.2 🧠 ¿Qué hace un API gateway que un reverse proxy "pelado" típicamente no? Nombrá dos cosas.
7.3 🧠 En Kubernetes, ¿qué diferencia hay entre el "Ingress" y el "ingress controller"? ¿Cuál es el reverse proxy de verdad?

---

## Módulo 8 — Alternativas modernas: Caddy y Traefik

**Teoría.** nginx es el estándar, pero no el único. Dos alternativas que cambian el juego según el contexto:

**Caddy** — su bandera es el **HTTPS automático de fábrica**. Mirá el `Caddyfile` completo para hacer reverse proxy con TLS:
```
app.ejemplo.com

reverse_proxy localhost:3000
```
Eso es **todo**. Caddy provisiona y **renueva el certificado de Let's Encrypt solo** (necesita el DNS apuntando al server y los puertos 80/443 abiertos). Compará con el baile de Certbot del módulo 5: acá no hay nada que cablear. **Elegí Caddy cuando** querés que el HTTPS "simplemente funcione", configs simples, equipos chicos o un VPS.

**Traefik** ⚠️ (la versión actual es la **v3**) — su bandera es ser **dinámico y consciente de contenedores**: **descubre los servicios solos** leyendo las labels de Docker, sin reiniciar nada. Ejemplo en `docker-compose`:
```yaml
services:
  mi-app:
    image: miapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mi-app.rule=Host(`app.ejemplo.com`)"
      - "traefik.http.services.mi-app.loadbalancer.server.port=3000"
```
Traefik mira el socket de Docker, lee las labels y **reconfigura el routing en vivo** cuando levantás o bajás un contenedor. **Elegí Traefik cuando** estás en un entorno Docker/k8s con servicios que van y vienen y querés que el routing **siga** a tus contenedores automáticamente.

**Elegí nginx cuando** querés máximo control/performance, config battle-tested, o ya estás en él. El costo: su config es estática (recargás para cambiarla) y los certs los cableás aparte (Certbot).

La frase mental: **nginx = control y madurez (pero TLS y reconfig a mano). Caddy = HTTPS automático de fábrica, config de dos líneas. Traefik = dinámico, descubre contenedores por labels y reconfigura en vivo. Elegís por contexto, no por moda.**

**Ejercicios 8**
8.1 🔁 ¿Cuál es la "bandera" (feature distintivo) de Caddy y cuál la de Traefik?
8.2 🧠 Tenés un docker-compose con servicios que escalás y cambiás seguido. ¿Cuál de los tres elegirías y por qué?
8.3 🧠 ¿Qué ganás y qué resignás eligiendo nginx sobre Caddy para un proyecto nuevo en un VPS?
8.4 ✍️ Escribí el `Caddyfile` mínimo que haga reverse proxy con HTTPS automático del dominio `api.midominio.com` a un Node en `localhost:5000`.

---

## Módulo 9 — El lado de Express: `trust proxy` y sus trampas

**Teoría.** Poné atención porque esto cierra el círculo con [Express](express.md). Cuando tu app está **detrás de un proxy**, el socket de Node ve la IP **del proxy** (no la del cliente), y el request llega como **HTTP plano** (el TLS se terminó arriba). Resultado: sin configurar nada, `req.ip` te da la IP del proxy, `req.protocol` te da `http` y `req.secure` te da `false`. **Todo mal.**

La solución es decirle a Express **que confíe en el proxy** para leer los headers `X-Forwarded-*` (los que seteamos en el módulo 3):

```js
app.set('trust proxy', 1)   // confiá en 1 proxy (nuestro nginx)
```

Con eso, `req.ip`, `req.protocol`, `req.secure` y `req.hostname` reflejan al **cliente real**, no al proxy. Es la app confiando en el **registro de visitas** que completó la recepción (los `X-Forwarded-*`): si confiás en ese registro, sabés quién entró de verdad; pero si dejás que cualquiera escriba en él (módulo de seguridad de abajo), te mienten. Los valores que acepta `trust proxy`:
- **número de hops** — `1` = "hay un proxy delante mío" (lo más común y recomendado);
- `'loopback'` / `'uniquelocal'` — rangos de confianza;
- IPs/subredes específicas;
- `true` — confía en el primer valor de `X-Forwarded-For` (⚠️ **peligroso**, ver abajo).

> ⚠️ **La trampa de seguridad (cae en entrevistas y rompe en producción):**
> 1. **No uses `trust proxy: true` a lo bobo.** Si confiás ciegamente en `X-Forwarded-For`, **cualquier cliente puede falsificar ese header** y hacerse pasar por otra IP. La defensa: `nginx` con `X-Forwarded-For $proxy_add_x_forwarded_for` **le appendea la IP real del socket** al final de la cadena (no descarta lo que mandó el cliente, pero al combinarlo con un número exacto de hops, Express lee la posición correcta y la falsificación no sirve). Concretamente: con `N` hops, Express toma la IP que está **`N` posiciones desde el final** de la cadena XFF (la que appendeó tu proxy), **no la primera** (que el cliente controla y puede inventar). Si querés descartar del todo lo que mandó el cliente, usás `proxy_set_header X-Forwarded-For $remote_addr;`. La regla: usá un **número exacto de hops** (`1`), no `true`.
> 2. **Esto rompe el rate limiting.** Si usás `express-rate-limit` con `trust proxy` mal configurado, un atacante puede forjar un `X-Forwarded-For` distinto en cada request, conseguir una **clave de límite distinta cada vez** y **saltarse el limitador por completo**. La librería incluso tira error si detecta un `trust proxy` demasiado permisivo. La regla: poné el **número exacto** de proxies que tenés delante.

La frase mental: **detrás de un proxy, Express ve al proxy, no al cliente — hasta que `app.set('trust proxy', N)` le dice que lea los `X-Forwarded-*`. Usá el número EXACTO de proxies, nunca `true` a ciegas: confiar de más deja falsificar la IP y saltar el rate limiting.**

**Ejercicios 9**
9.1 🔁 Sin `trust proxy`, ¿qué valores incorrectos te dan `req.ip`, `req.protocol` y `req.secure` detrás de un proxy? ¿Por qué?
9.2 🧠 ¿Por qué `app.set('trust proxy', true)` puede ser un riesgo de seguridad? ¿Qué valor usás en su lugar con un solo nginx delante?
9.3 🧠 ¿Cómo se relaciona un `trust proxy` mal configurado con un rate limiter que se puede saltar?
9.4 ✍️ Tu app Express corre detrás de **un solo** nginx. Escribí la línea de `trust proxy` correcta y explicá en un comentario por qué ese valor y no `true`.

---

## Módulo 10 — El criterio: ¿lo hacés vos o lo hace la plataforma?

**Teoría.** El cierre y el criterio. Después de aprender a configurar nginx, la pregunta senior es: **¿de verdad tenés que configurar nginx?** Muchas veces **no** — la plataforma ya te da el borde hecho.

**La plataforma lo hace por vos (NO armes tu propio proxy):**
- **Vercel / Render / Railway / Fly.io** — TLS, el proxy de borde y a menudo el balanceo están gestionados. Desplegás y el HTTPS es automático. Sumar tu nginx es al pedo y agrega un punto de falla.
- **AWS ALB** — load balancer L7 gestionado + TLS vía ACM + health checks activos.
- **Kubernetes Ingress / Gateway API** — el ingress controller (nginx/Traefik) **es** el proxy gestionado; vos escribís reglas declarativas y `cert-manager` maneja el TLS. No metas un nginx a mano dentro de los pods.

**Armás tu propio nginx/Caddy/Traefik cuando:**
- **Self-hosting en un VPS** (DigitalOcean/Hetzner/EC2 pelado) — nadie más te da el borde.
- Necesitás **control fino** que la plataforma no expone (buffering custom, reglas de rate limit raras, routing exótico).
- **Contenedores sin un ingress gestionado** (Docker Compose en una máquina) — ahí Traefik o Caddy brillan.

**Cuándo NO armar el tuyo:** si tu PaaS/ALB/Ingress ya termina TLS y balancea, un segundo proxy es complejidad redundante y una cosa más para configurar mal (doble `X-Forwarded-For`, `trust proxy` con el número de hops equivocado, etc.). La regla de oro: **sé dueño del borde solo cuando vos sos el que tiene que instalar el certificado.**

La frase mental de cierre: **configurar nginx es una herramienta, no un peaje obligatorio. En un VPS, el borde es tuyo. En Vercel/ALB/Ingress, ya está hecho — sumar tu propio proxy es redundante y riesgoso. Regla: sé dueño del borde solo cuando vos instalás el cert.**

**Ejercicios 10**
10.1 🔁 Nombrá dos escenarios donde NO deberías armar tu propio reverse proxy, y por qué.
10.2 🧠 ¿Cuál es la "regla de oro" para decidir si tenés que correr tu propio proxy?
10.3 🧠 Tenés tu app en un Ingress de Kubernetes que ya termina TLS, y un compañero quiere meter un nginx adentro de cada pod "para tener más control". ¿Qué le decís?

---

## Soluciones

> Llegaste hasta acá habiendo intentado los ejercicios, ¿no? Bien. Contrastá.

**1.1** **Forward proxy:** intermediario que se para **delante del cliente** y actúa en su nombre — esconde **quién pregunta** (el cliente). **Reverse proxy:** intermediario que se para **delante de los servidores** y actúa en su nombre — esconde **quién responde** (cuántos backends hay y cuáles).

**1.2** Es un **reverse proxy**: está **delante de tu servidor** (tu API Node) y actúa en nombre del servidor — el cliente le habla a nginx creyendo que es la app, y nginx reenvía a tu Node por detrás. Esconde el/los backend(s).

**1.3** Que **todos son reverse proxies** (o lo contienen): un CDN, un ALB y un Ingress controller se paran delante de tus backends, reciben el request del cliente y lo reenvían, escondiendo los servidores reales. Cada uno enfatiza un trabajo distinto (caché geográfica, balanceo gestionado, routing declarativo de k8s), pero la pieza base es la misma (módulo 7).

**2.1** Cualquiera de: terminación TLS/HTTPS, servir estáticos eficientemente, bufferear clientes lentos, load balancing entre instancias, seguridad (esconder el server, centralizar headers), compresión, caché, rate limiting.

**2.2** Porque Node es **single-thread por proceso**: un solo event loop maneja todas las conexiones. Si un cliente lento manda los bytes de a goteo y Node lo atiende directo, ese socket lento **ocupa tiempo del event loop** que necesitan todas las demás conexiones → degrada a todos. nginx **bufferea** el request/response completo y le habla a Node en ráfagas rápidas, así el event loop nunca queda esperando a un cliente lento. En un server multi-thread, un thread bloqueado no frena a los demás, así que duele menos.

**2.3** NO aplica cuando estás en una **plataforma gestionada** (Vercel, Render, Fly, AWS ALB, k8s Ingress): esa plataforma **ya corre un proxy de borde endurecido** delante de tu app, que termina TLS y balancea. Ahí tu Node "mira a internet" solo en apariencia — hay un proxy gestionado adelante. Sumar tu propio nginx sería redundante. El consejo es sobre **quién corre el borde**, no sobre si Node puede escuchar en el 443.

**3.1** La directiva es **`proxy_pass`** (reenvía el request al backend indicado). El bloque **`upstream`** le pone **nombre a un grupo de uno o más backends**, y es la base del load balancing: en vez de apuntar `proxy_pass` a una IP fija, apuntás a un `upstream` que puede tener varias instancias.

**3.2** Porque **por defecto nginx manda como `Host` el del upstream** (`$proxy_host`), no el hostname original que pidió el cliente. Sin `proxy_set_header Host $host`, tu app (y cualquier lógica que dependa del dominio: virtual hosts, generación de URLs absolutas, multi-tenancy por subdominio) **vería el host equivocado**. Con `$host` reenviás el hostname real que pidió el cliente.

**3.3**
```nginx
server {
    listen 80;
    server_name api.midominio.com;

    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**4.1** Porque un proceso de Node es **single-thread y usa un solo core**: una sola instancia no aprovecha una máquina de varios cores ni escala más allá de lo que da un core. Corrés **N instancias** (cluster, PM2 o N contenedores) para usar todos los cores / aguantar más carga, y el reverse proxy **reparte (balancea)** los requests entre esas instancias.

**4.2** **Passive** (gratis): nginx marca un backend como caído **después** de que fallen requests reales, usando `max_fails` + `fail_timeout` — es reactivo, se entera cuando ya falló algo. **Active**: nginx hace **probes periódicos** (con la directiva `health_check`) para detectar backends caídos **antes** de mandarles tráfico real. La trampa de costo: **los health checks activos son solo de nginx Plus (la versión paga)** — en el open-source solo tenés passive.

**4.3**
```nginx
upstream api {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003 backup;
}
```

**5.1** "Terminar TLS en el proxy" significa que **el HTTPS se desencripta en nginx**: el proxy tiene el certificado y la clave, recibe el tráfico cifrado y lo descifra ahí. De nginx a Node el tráfico viaja como **HTTP plano** por la red privada/de confianza (Node nunca ve el certificado). Un solo lugar centraliza la gestión de certificados.

**5.2** Porque **los certificados duran cada vez menos**: el máximo de la industria (CA/Browser Forum) viene bajando de los ~398 días hacia ~47 días hacia fines de la década, y Let's Encrypt ya ofrece certs de 6 días. Con vencimientos tan frecuentes, renovar a mano es inviable: tarde o temprano te olvidás y el cert vence → **caída del sitio** (HTTPS roto). La renovación automática (Certbot con su timer/cron, o Caddy que lo hace solo) deja de ser "buena práctica" y pasa a ser obligatoria.

**5.3**
```nginx
server {
    listen 80;
    server_name app.ejemplo.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name app.ejemplo.com;

    ssl_certificate     /etc/letsencrypt/live/app.ejemplo.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/app.ejemplo.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
(Los cuatro headers van siempre, como en el módulo 3 — `X-Forwarded-Proto` es especialmente importante acá: es el que le dice a tu Node que el request original fue HTTPS, aunque a él le llegue HTTP plano.)

**6.1** Cualquiera de: compresión (gzip), rate limiting, security headers, caché (`proxy_cache`), terminación TLS. (Todas tienen sentido en el borde porque el proxy ve todo el tráfico.)

**6.2** Pasó la **trampa de herencia de `add_header`**: las directivas `add_header` se heredan de niveles superiores (`http`/`server`) **solo si el nivel actual no define ninguna propia**. Al agregar un `add_header` dentro del `location`, nginx **descarta en silencio** todos los heredados. Arreglo: **redefinir** los headers que querés conservar **en el mismo `location`** donde agregás el nuevo (o, en nginx 1.29.3+, usar `add_header_inherit merge;` — pero ⚠️ es muy nuevo).

**6.3** En la **app (Express con `cors`)**. Porque el CORS suele depender del **contexto del request**: orígenes permitidos dinámicos, manejo de credenciales, reglas distintas por ruta — y la app conoce su propio ruteo y lógica. El proxy solo es razonable para un CORS totalmente estático, y aun así es engorroso (manejo del preflight `OPTIONS`, y te muerde la trampa del `add_header`). Por eso: CORS en la app.

**6.4**
```nginx
http {
    limit_req_zone $binary_remote_addr zone=apilimit:10m rate=5r/s;

    server {
        location /api/ {
            limit_req zone=apilimit burst=10 nodelay;
            proxy_pass http://node_app;
        }
    }
}
```
(`limit_req_zone` va en el contexto `http`; `limit_req` en el `location`. `rate=5r/s` = 5 req/seg por IP, `burst=10` = colchón de 10.)

**6.5**
```nginx
server {
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    location /api/ {
        # Al definir un add_header acá, nginx DESCARTA los tres heredados del server.
        # Hay que REDEFINIRLOS junto al nuevo:
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header Cache-Control "no-store" always;   # el nuevo

        proxy_pass http://node_app;
    }
}
```
La trampa: si solo ponés `add_header Cache-Control ...` dentro del `location`, los tres del `server` **desaparecen en silencio** (la herencia de `add_header` es "todo o nada" por nivel). Por eso se repiten los tres heredados + el nuevo. (En nginx 1.29.3+ podrías usar `add_header_inherit merge;` para fusionar en vez de pisar, pero redefinir es lo portable.)

**7.1** Porque todos —load balancer, API gateway, ingress controller, nodo de CDN— **son reverse proxies** en su base (se paran delante de los backends y reenvían), y se diferencian por **qué trabajo extra enfatizan**. Especies: **load balancer** (AWS ALB) enfatiza distribuir carga; **API gateway** (Kong, AWS API Gateway) enfatiza temas de API (auth, rate limit, transformación); **ingress controller** (nginx/Traefik en k8s) enfatiza ejecutar config declarativa; **CDN** (Cloudflare) enfatiza caché geográfica de borde.

**7.2** Un API gateway agrega temas **específicos de APIs** que un reverse proxy pelado normalmente no hace: dos de — **autenticación/autorización** (validar tokens/API keys en el borde), **rate limiting por consumidor**, **transformación** de request/response, **agregación** de varios servicios en una respuesta. Es un reverse proxy L7 especializado para el tráfico de APIs.

**7.3** El **Ingress** es la **abstracción/configuración declarativa** de Kubernetes: un objeto donde escribís las reglas de routing (qué host/path va a qué Service). El **ingress controller** es el **programa real** (a menudo nginx o Traefik) que **lee** esas reglas y **ejecuta** el reverse proxy/load balancing. O sea: el Ingress es el "qué quiero"; el controller es el reverse proxy de verdad que lo hace. Sin un controller instalado, un objeto Ingress no hace nada.

**8.1** **Caddy:** HTTPS **automático de fábrica** (provisiona y renueva los certs de Let's Encrypt solo, sin configurar nada). **Traefik:** **dinámico y consciente de contenedores** — descubre los servicios automáticamente leyendo labels de Docker (o recursos de k8s) y reconfigura el routing en vivo, sin reiniciar.

**8.2** **Traefik.** En un entorno con servicios que escalás y cambiás seguido, Traefik **descubre los contenedores solo** (por labels) y **reconfigura el routing en vivo** cuando levantás o bajás uno — no tenés que editar y recargar config a mano cada vez. nginx te obligaría a actualizar el `upstream` y recargar; Caddy es más simple pero no tan orientado a descubrimiento dinámico de contenedores como Traefik.

**8.3** **Ganás** control fino y performance battle-tested, configuración madura y muchísima documentación/ejemplos. **Resignás** la comodidad del HTTPS automático (en nginx cableás Certbot y la renovación aparte, mientras Caddy lo hace solo con dos líneas) y la simplicidad de config (el `Caddyfile` mínimo es trivial; nginx es más verboso). Para un VPS nuevo donde querés HTTPS sin dolor, Caddy es más rápido de poner en pie; nginx conviene si necesitás el control fino o ya lo dominás.

**8.4**
```
api.midominio.com

reverse_proxy localhost:5000
```
Eso es todo: Caddy detecta el dominio, provisiona el certificado de Let's Encrypt y lo renueva solo (con el DNS apuntando al server y los puertos 80/443 abiertos). No hay bloque `listen 443 ssl`, ni `ssl_certificate`, ni Certbot: el HTTPS automático es justamente su razón de ser.

**9.1** Sin `trust proxy`, Express mira el **socket directo** y los datos del request tal como le llegan **del proxy**: `req.ip` = la **IP del proxy** (no la del cliente, porque el socket viene del proxy); `req.protocol` = **`http`** (porque el TLS se terminó en el proxy y de ahí a Node viaja HTTP plano); `req.secure` = **`false`** (deriva de `protocol`). Todos miran al proxy en vez de al cliente real, porque Express no sabe que tiene que leer los headers `X-Forwarded-*`.

**9.2** Porque `trust proxy: true` hace que Express confíe en el **primer valor de `X-Forwarded-For`**, y ese header **lo puede mandar cualquiera**: si tu último proxy no lo sobrescribe, un cliente puede **falsificar su IP** poniendo el `X-Forwarded-For` que quiera. Con un solo nginx delante, usás el **número exacto de hops**: `app.set('trust proxy', 1)` — confiás en exactamente un proxy, no en cualquier valor que llegue.

**9.3** Un rate limiter (ej. `express-rate-limit`) usa la **IP del cliente como clave** para contar requests. Si `trust proxy` está mal (demasiado permisivo, como `true`), el cliente puede **forjar un `X-Forwarded-For` distinto en cada request** → Express le asigna una **IP distinta cada vez** → el limiter cuenta cada request como si fuera de un cliente nuevo y **nunca llega al límite**: el atacante se lo saltea por completo. Por eso hay que poner el número **exacto** de proxies (y por eso la librería tira error si detecta una config demasiado abierta).

**9.4**
```js
// Hay exactamente UN proxy (nuestro nginx) entre internet y la app.
app.set('trust proxy', 1)
// Por qué 1 y no `true`: `true` confía en el primer X-Forwarded-For que llegue, así
// que un cliente podría falsificar su IP. El número exacto de hops le dice a Express
// "saltá 1 proxy y tomá la IP real del socket", que el cliente no puede falsificar.
```

**10.1** Cualquiera de dos: (a) **PaaS como Vercel/Render/Fly** — ya gestionan TLS y el proxy de borde, sumar nginx es redundante y un punto de falla extra; (b) **AWS detrás de un ALB** — el ALB ya es el LB+TLS gestionado; (c) **Kubernetes con Ingress** — el ingress controller ya es el proxy gestionado, meter nginx en los pods duplica la capa. En todos, la plataforma ya corre el borde.

**10.2** **"Sé dueño del borde solo cuando vos sos el que tiene que instalar el certificado."** Si vos instalás y renovás el cert (VPS, máquina pelada), el reverse proxy es tuyo y lo configurás. Si la plataforma termina el TLS por vos (PaaS, ALB, Ingress con cert-manager), no necesitás —ni conviene— armar otro proxy encima.

**10.3** Le digo que **no**: el Ingress controller (nginx/Traefik) ya **es** un reverse proxy gestionado que termina TLS y balancea. Meter un nginx adentro de cada pod **duplica la capa de proxy** sin ganar nada: agrega complejidad, un punto de falla más, y problemas típicos de doble proxy (doble `X-Forwarded-For`, `trust proxy` con el número de hops equivocado en la app, latencia extra). Si necesita "más control", eso se configura **en el Ingress/controller**, no metiendo un proxy redundante en cada pod. La regla de oro: el borde ya está, no lo dupliques.

---

> **Para seguir.** Las fuentes de verdad: [nginx.org/en/docs](https://nginx.org/en/docs/) (los módulos `ngx_http_proxy_module`, `ngx_http_upstream_module`, `ngx_http_ssl_module`), [letsencrypt.org](https://letsencrypt.org) y [eff-certbot.readthedocs.io](https://eff-certbot.readthedocs.io) para certificados, [caddyserver.com/docs](https://caddyserver.com/docs/) y [doc.traefik.io](https://doc.traefik.io/) para las alternativas, y [expressjs.com/en/guide/behind-proxies.html](https://expressjs.com/en/guide/behind-proxies.html) para `trust proxy`. Antes de copiar lo marcado con ⚠️ (la directiva `ssl on;` eliminada, el default de `proxy_http_version`, los tiempos de los certificados, `add_header` y su herencia, la versión de Traefik), re-verificá: esta capa se mueve. El puente natural: [Express](express.md) para la app que va detrás, [Docker, deploy y CI/CD](docker-deploy.md) para empaquetarla, [Kubernetes](kubernetes.md) para la versión orquestada (Ingress = esto, declarativo) y [AWS](aws.md) para la versión gestionada (ALB/API Gateway).
