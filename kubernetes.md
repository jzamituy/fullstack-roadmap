# Kubernetes: orquestar contenedores en producción

**Fundamentos de K8s con criterio, viniendo de Docker · 2026**

> Cómo usar esta guía: leé la teoría de cada módulo, hacé los ejercicios **sin mirar las soluciones**, y contrastá al final. Este módulo asume que ya hiciste [Docker, deploy y CI/CD](docker-deploy.md) (imágenes, contenedores, Compose) y que conocés [AWS](aws.md) (ECS/Fargate). Kubernetes (K8s) es el **orquestador de contenedores** estándar de la industria: una vez que tenés una app en un contenedor, K8s la corre, la escala, la sana cuando se cae y la actualiza sin downtime, a través de muchas máquinas. El objetivo de este módulo es doble: **entender los fundamentos** (pods, deployments, services, etc.) y —tan importante— **el criterio de cuándo K8s y cuándo es overkill**. Porque el error de Tech Lead más caro acá no es no saber K8s: es **meter K8s donde un Fargate o un PaaS hacían el trabajo con una fracción de la complejidad operativa**.

**Lo que asumimos.** Docker (imágenes, contenedores, el `Dockerfile` de tu API Nest), redes básicas, YAML, AWS conceptual. No asumimos experiencia previa con clusters.

**Para practicar.** Un cluster local liviano y `kubectl`:

```bash
# kind o minikube levantan un cluster de K8s local sobre Docker
brew install kind kubectl     # (o el gestor de tu OS)
kind create cluster
kubectl get nodes             # tu cluster de un nodo, listo
```

> Nota sobre datos volátiles: las versiones de K8s, el estado de APIs (Gateway API, Ingress) y qué controladores están vigentes cambian rápido (ej. ingress-nginx está en proceso de retiro/mantenimiento reducido y Gateway API es la dirección recomendada, aunque Ingress sigue soportado). Verificá contra `kubernetes.io` al implementar.

**Índice de módulos**
1. Qué es Kubernetes y qué problema resuelve
2. La arquitectura: control plane y nodos
3. Pods: la unidad mínima (y mortal)
4. Deployments: estado deseado declarativo
5. Services: red estable sobre pods efímeros
6. Tráfico externo: Ingress y Gateway API
7. Configuración: ConfigMaps y Secrets
8. Salud, recursos y autoscaling
9. Estado: volúmenes, PV/PVC y StatefulSets
10. Lo declarativo: manifiestos, kubectl y Helm
11. El criterio: ¿necesitás Kubernetes?

Las soluciones de **todos** los ejercicios están al final, en la sección "Soluciones".

---

## Módulo 1 — Qué es Kubernetes y qué problema resuelve

**Teoría.** Con Docker aprendiste a empaquetar tu app en una **imagen** y correrla como un **contenedor**. Eso resuelve "funciona en mi máquina". Pero producción tiene preguntas que Docker solo no responde:

- ¿Cómo corro **20 copias** de mi API repartidas en 5 máquinas?
- Si un contenedor **se cae**, ¿quién lo reinicia? Si una máquina entera muere, ¿quién mueve sus contenedores a otra?
- ¿Cómo hago un **deploy sin downtime** (reemplazar la v1 por la v2 gradualmente)?
- ¿Cómo **escalo** de 3 a 30 réplicas cuando sube el tráfico, y vuelvo a bajar?
- ¿Cómo se **encuentran** entre sí los contenedores si sus IPs cambian todo el tiempo?

**Kubernetes es la respuesta a todas esas preguntas.** Es un **orquestador de contenedores**: un sistema que corre tus contenedores a través de un grupo de máquinas (un **cluster**), y se encarga de programarlos, sanarlos, escalarlos, conectarlos y actualizarlos automáticamente.

La idea central, la que tenés que internalizar y que define todo K8s: es **declarativo**. Vos no le das órdenes ("arrancá este contenedor", "reiniciá aquel"); le declarás el **estado deseado** ("quiero 20 réplicas de mi API v2, corriendo siempre"), y K8s trabaja sin parar para que la **realidad coincida** con lo que declaraste. Si una réplica muere, la realidad (19) ya no coincide con lo deseado (20), así que K8s crea una nueva. Esto es un **loop de reconciliación**: comparar estado actual vs. deseado, y actuar para cerrar la brecha. Es el mismo paradigma que viste en Infraestructura como Código (módulo de AWS práctica): declarás el qué, no el cómo.

La frase mental: **a Docker le decís cómo correr un contenedor; a Kubernetes le declarás cómo querés que se vea tu sistema, y él se ocupa de mantenerlo así.** Esa diferencia —imperativo vs. declarativo, un contenedor vs. un sistema entero auto-gestionado— es de lo que se trata el módulo.

**Ejercicios 1**
1.1 Nombrá tres problemas de producción que Docker solo no resuelve y Kubernetes sí.
1.2 ¿Qué significa que Kubernetes sea "declarativo"? Explicá el loop de reconciliación con el ejemplo de una réplica que se cae.
1.3 ¿Con qué otro tema del temario comparte K8s el paradigma declarativo?

---

## Módulo 2 — La arquitectura: control plane y nodos

**Teoría.** Un cluster de Kubernetes tiene dos tipos de componentes: el **control plane** (el cerebro que decide) y los **nodos** (las máquinas donde corren tus contenedores, los músculos).

**Control plane** (el cerebro, normalmente replicado para alta disponibilidad):

- **API server**: la puerta de entrada a todo. Todo —`kubectl`, los controladores, vos— habla con el cluster a través de esta API REST. Es el único que escribe el estado.
- **etcd**: la base de datos del cluster (un key-value distribuido). Guarda **todo el estado**: qué deberías tener corriendo, la config, todo. Es la "fuente de verdad". Si perdés etcd, perdés el cluster.
- **Scheduler**: decide **en qué nodo** poner cada pod nuevo, según recursos disponibles, reglas y restricciones.
- **Controller manager**: corre los **loops de reconciliación** (módulo 1). Es el que nota "deberían ser 20, hay 19" y actúa.

**Nodos** (las máquinas worker, donde vive tu app):

- **kubelet**: el agente en cada nodo que habla con el control plane y se asegura de que los contenedores que le tocan estén corriendo y sanos.
- **kube-proxy**: maneja la red del nodo (el ruteo hacia los Services, módulo 5).
- **container runtime**: lo que realmente corre los contenedores (containerd, etc.) — la pieza que viene de Docker.

Dos piezas más del plano de red conviene nombrarlas: el **CNI** (Container Network Interface — el plugin que le da IP a cada Pod y conecta la red del cluster; ej. Calico, Cilium) y **CoreDNS** (el DNS interno que resuelve los nombres de Service del módulo 5). En clusters modernos, los CNIs basados en eBPF como Cilium incluso reemplazan a kube-proxy.

El flujo mental cuando hacés un deploy: vos mandás "quiero 20 réplicas" al **API server** → se guarda en **etcd** → el **controller manager** ve que faltan pods → el **scheduler** elige nodos → el **kubelet** de cada nodo arranca los contenedores vía el **runtime**. Cada pieza hace una cosa y se comunican por la API.

La buena noticia para vos: en la práctica **rara vez administrás el control plane a mano.** Los servicios de **K8s gestionado** —**EKS** (AWS), **GKE** (Google), **AKS** (Azure)— corren el control plane por vos; vos te ocupás de tus nodos y tus apps. Conocer la arquitectura igual importa: explica *por qué* K8s se comporta como se comporta (todo pasa por la API, todo es un loop de reconciliación) y es pregunta de entrevista segura.

**Namespaces: dividir el cluster lógicamente.** Un mismo cluster se particiona en **namespaces**: agrupaciones lógicas que aíslan recursos (un `dev`, un `staging`, un `prod`; o un namespace por equipo). La mayoría de los objetos —Pods, Deployments, Services, ConfigMaps, Secrets— **viven dentro de un namespace** (por eso un Secret de un namespace no lo ve otro). `kubectl` opera sobre `default` salvo que le pases `-n <namespace>`. Importan porque son la unidad sobre la que se aplican el **RBAC** (quién puede qué, módulo 7), la **ResourceQuota** (cuánto CPU/memoria puede consumir el grupo) y la **NetworkPolicy** (qué se puede comunicar con qué). Es lo primero que tocás al operar un cluster real.

**Ejercicios 2**
2.1 ¿Cuál es la diferencia entre el control plane y los nodos? Da un componente de cada uno y qué hace.
2.2 ¿Qué guarda etcd y por qué es crítico?
2.3 Describí el flujo de "quiero 20 réplicas" pasando por los componentes del control plane y el nodo.
2.4 ¿Qué es un namespace y para qué sirve? Nombrá dos cosas que se apliquen "por namespace".

---

## Módulo 3 — Pods: la unidad mínima (y mortal)

**Teoría.** Acá hay una sorpresa para el que viene de Docker: **Kubernetes no corre contenedores directamente, corre Pods.** Un **Pod** es la unidad mínima desplegable de K8s: **uno o más contenedores** que comparten red (la misma IP y puertos) y almacenamiento, y que se programan juntos en el mismo nodo. En el 90% de los casos un Pod = un contenedor (tu API), pero el Pod es el envoltorio.

¿Por qué un envoltorio y no el contenedor pelado? Porque a veces necesitás contenedores **acoplados** que viven y mueren juntos: tu app + un **sidecar** (un contenedor auxiliar, ej. uno que recolecta logs o hace de proxy). Comparten localhost y disco, así que cooperan como si fueran un proceso. Ese patrón —el sidecar— solo tiene sentido con la abstracción del Pod.

Lo **crítico** que tenés que clavar: **los Pods son efímeros y mortales.** No son mascotas, son ganado. Se crean, se destruyen y se reemplazan **constantemente** —cuando escalás, cuando actualizás, cuando un nodo se cae, cuando reorganizás recursos—. Y cada vez que un Pod muere y nace otro, **el nuevo tiene una IP distinta.** Nunca te aferrás a un Pod ni a su IP. Dos consecuencias enormes:

- **No guardes estado en un Pod** (ni sesiones en memoria, ni archivos que importen). Es exactamente el principio "stateless" del módulo de Node y de Redis: el estado va afuera (Postgres, Redis, almacenamiento persistente del módulo 9), porque el Pod puede desaparecer en cualquier momento. K8s lleva ese principio al extremo: asume que tus Pods mueren todo el tiempo.
- **No te conectás a un Pod por su IP** (cambia siempre). Para eso existe el Service (módulo 5), que te da una dirección estable por delante de Pods que van y vienen.

La frase mental: **un Pod es desechable por diseño.** Toda la robustez de K8s —auto-sanado, escalado, deploys sin downtime— nace de tratar los Pods como reemplazables. Diseñar tu app para eso (stateless, tolerante a reinicios) es lo que la hace "cloud native".

**Ejercicios 3**
3.1 ¿Qué es un Pod y por qué K8s usa este envoltorio en vez de correr el contenedor directo? Da el caso del sidecar.
3.2 ¿Qué significa que los Pods sean "efímeros y mortales" y qué dos cosas NO tenés que hacer por eso?
3.3 Conectá "no guardes estado en un Pod" con un principio que ya viste en Node/Redis.

---

## Módulo 4 — Deployments: estado deseado declarativo

**Teoría.** Si los Pods son mortales y no los manejás a mano, ¿quién crea 20 réplicas y las mantiene vivas? El **Deployment** — el objeto que usás el 95% del tiempo para correr una app sin estado. Le declarás:

- **Qué** correr (la imagen del contenedor, ej. `mi-api:v2`).
- **Cuántas** réplicas querés (`replicas: 20`).
- **Cómo** actualizar cuando cambie (rolling update por defecto).

Y el Deployment se encarga del resto vía el loop de reconciliación. Por debajo crea un **ReplicaSet** (el objeto que garantiza "siempre N Pods corriendo"); vos casi siempre hablás con el Deployment, no con el ReplicaSet directamente.

Lo que un Deployment te da gratis, que es **todo lo del módulo 1** hecho concreto:

- **Auto-sanado**: si un Pod muere o un nodo cae, el Deployment crea reemplazos hasta volver a `replicas`. No hacés nada.
- **Escalado**: cambiás `replicas: 20` a `replicas: 40` (o lo hace el autoscaler, módulo 8) y aparecen 20 Pods más.
- **Rolling updates (deploy sin downtime)**: cambiás la imagen a `v2` y el Deployment **reemplaza los Pods de a poco** —levanta algunos v2, espera que estén sanos (readiness, módulo 8), baja algunos v1, y repite— hasta que todo es v2, sin que el servicio deje de responder. Esto es el "deploy sin downtime" que en el módulo de Docker/CI-CD era un objetivo; acá es una línea de YAML.
- **Rollback**: si la v2 está rota, `kubectl rollout undo` vuelve a la v1 en segundos, porque K8s recuerda el estado anterior.

El manifiesto declarativo (lo profundiza el módulo 10), para tu API Nest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: task-api
spec:
  replicas: 3                      # estado deseado: 3 réplicas siempre
  selector:
    matchLabels: { app: task-api } # qué Pods maneja (por etiqueta)
  template:                        # el molde del Pod a crear
    metadata:
      labels: { app: task-api }
    spec:
      containers:
        - name: api
          image: mi-registry/task-api:v2
          ports: [{ containerPort: 3000 }]
          resources:                 # requests/limits (módulo 8): higiene de producción
            requests: { cpu: "100m", memory: "128Mi" }
            limits: { cpu: "500m", memory: "256Mi" }
          # las probes (liveness/readiness/startup) van acá también — módulo 8
```

Detalle que rompe a los principiantes: el `selector.matchLabels` del Deployment **tiene que coincidir** con `template.metadata.labels` (acá, `app: task-api`). Si no matchean, el Deployment no "adopta" sus propios Pods y K8s rechaza el manifiesto.

La frase mental: **el Deployment es el "quiero que esto corra así, siempre" hecho objeto.** Auto-sanado, escalado, deploy gradual y rollback dejan de ser scripts que escribís y pasan a ser propiedades declarativas de tu sistema.

**Ejercicios 4**
4.1 ¿Qué tres cosas le declarás a un Deployment, y qué objeto crea por debajo para mantener N Pods?
4.2 Explicá cómo un Deployment hace un deploy sin downtime (el rolling update), paso a paso.
4.3 ¿Por qué el rollback es casi instantáneo en K8s? Conectá con lo declarativo (módulo 1).

---

## Módulo 5 — Services: red estable sobre pods efímeros

**Teoría.** Problema del módulo 3: los Pods son efímeros y sus IPs cambian. Entonces, ¿cómo le pega tu frontend a tu API si la API son 20 Pods con IPs que mutan? ¿Cómo le pega tu API a un microservicio de pagos? La respuesta es el **Service**: una abstracción que te da una **dirección de red estable** (una IP virtual y un nombre DNS fijos) por delante de un conjunto de Pods que van y vienen.

Cómo funciona: el Service selecciona Pods por **etiqueta** (`app: task-api`, las mismas del Deployment). Cuando le pegás al Service, K8s **balancea la carga** automáticamente entre los Pods sanos que matchean esa etiqueta. Si un Pod muere y nace otro con nueva IP, el Service lo incorpora solo —vos seguís usando la misma dirección estable—. El Service **desacopla** "a quién le hablo" (un nombre estable) de "qué Pods existen ahora mismo" (cambiante).

El descubrimiento de servicios (service discovery) sale gratis vía **DNS interno**: dentro del cluster, tu API le pega al servicio de pagos simplemente como `http://pagos` (el nombre del Service). No hardcodeás IPs ni mantenés un registro: el nombre del Service *es* la dirección. Esto es lo que hace viable una arquitectura de microservicios (módulo de event-driven): los servicios se encuentran por nombre, sin importar dónde corran sus Pods.

Los **tipos** de Service que tenés que conocer:

- **ClusterIP** (el default): expone el Service **solo dentro del cluster**. Es lo que usás para que tus servicios se hablen entre sí (API → pagos). No accesible desde afuera.
- **NodePort**: abre un puerto en cada nodo. Crudo, se usa poco directamente.
- **LoadBalancer**: pide un balanceador de carga del cloud (un ELB en AWS) y expone el Service **a internet**. Es la forma de exponer algo hacia afuera, aunque para HTTP normalmente preferís un Ingress/Gateway por delante (módulo 6).

El manifiesto de un Service ClusterIP para tu API:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: task-api            # el nombre DNS interno: http://task-api
spec:
  selector: { app: task-api } # enruta a los Pods con esta etiqueta (las del Deployment)
  ports:
    - port: 80              # puerto del Service
      targetPort: 3000      # puerto del contenedor (containerPort del Pod)
```

El `selector` es la pieza clave: si no coincide con las labels de los Pods, el Service **no tiene a quién rutear** (endpoints vacíos) y las requests fallan aunque los Pods estén sanos.

La frase mental: **el Pod es la cosa mortal; el Service es la dirección inmortal que la representa.** Toda comunicación estable en K8s —entre tus servicios y desde afuera— pasa por Services, justamente porque los Pods no se pueden direccionar de forma fija.

**Ejercicios 5**
5.1 ¿Qué problema resuelve un Service y cómo elige a qué Pods enruta?
5.2 ¿Cómo le pega un servicio a otro dentro del cluster, y por qué no necesitás hardcodear IPs?
5.3 ¿Cuál es la diferencia entre un Service ClusterIP y uno LoadBalancer, y para qué usás cada uno?
5.4 **Diagnosticá.** Un Service no enruta tráfico a ningún Pod (sus endpoints están vacíos) aunque los Pods corren sanos. ¿Cuál es la causa más probable y qué revisarías primero?

---

## Módulo 6 — Tráfico externo: Ingress y Gateway API

**Teoría.** Un Service `LoadBalancer` expone *una* cosa a internet, pero pedir un balanceador del cloud por cada servicio es caro y limitado. Lo que realmente querés para tráfico HTTP es **un solo punto de entrada** que rutee según el **host** y el **path** hacia distintos servicios internos:

- `api.miapp.com` → Service `task-api`
- `miapp.com/pagos` → Service `pagos`
- terminación TLS (HTTPS) centralizada en un lugar

Eso es el trabajo de un **Ingress**: define reglas de ruteo HTTP/HTTPS hacia tus Services internos (ClusterIP). Pero —detalle clave— **el objeto Ingress por sí solo no hace nada**: necesita un **Ingress Controller**, un componente que corre en el cluster, lee esas reglas y efectivamente rutea el tráfico (históricamente ingress-nginx, entre otros). El Ingress es la regla; el controller es quien la ejecuta.

El punto **actual y de criterio** (dato volátil, verificá en `kubernetes.io`): el ecosistema se está moviendo de Ingress hacia la **Gateway API**, el sucesor más expresivo y recomendado (sus tipos `Gateway`/`HTTPRoute` ya son estables/GA). Por qué el cambio: el Ingress quedó corto (poca expresividad, cada controller inventaba anotaciones propias no portables) y la Gateway API ofrece un modelo más rico, **orientado a roles** (separa la responsabilidad del operador de infraestructura de la del dev de la app) y más portable entre implementaciones. **Importante, para no exagerar el cambio**: el Ingress **NO está deprecado** y sigue soportado; migrar a Gateway API es **recomendado pero opcional**, no forzado. Y un dato a confirmar (cambia rápido): el popular controller **ingress-nginx** entró en **retiro / mantenimiento reducido** con un sucesor anunciado, lo que empuja a los proyectos nuevos hacia Gateway API. La idea conceptual es la misma (un punto de entrada que rutea por host/path con TLS); cambia la API con la que lo declarás.

Lo que tenés que llevarte, sin perderte en el detalle de cada controller:

- **Service LoadBalancer** = exponer *un* servicio a internet (TCP/cualquier protocolo).
- **Ingress / Gateway API** = un punto de entrada **HTTP inteligente** que rutea por host/path a muchos Services internos, con TLS — lo que usás para una app web real.
- Siempre hay **una regla (Ingress/Gateway) + un controller que la ejecuta**: declarás el ruteo, un componente lo aplica.

Las dos formas de declarar el mismo ruteo (`api.miapp.com` → Service `task-api`):

```yaml
# Forma clásica: Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: task-api
spec:
  rules:
    - host: api.miapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: task-api, port: { number: 80 } }
```

```yaml
# Forma recomendada hoy: Gateway API (HTTPRoute, asume un Gateway ya creado)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: task-api
spec:
  parentRefs: [{ name: mi-gateway }]
  hostnames: ["api.miapp.com"]
  rules:
    - backendRefs: [{ name: task-api, port: 80 }]
```

**Ejercicios 6**
6.1 ¿Qué hace un Ingress y por qué no alcanza con un Service LoadBalancer por cada cosa?
6.2 ¿Por qué un objeto Ingress "no hace nada solo"? ¿Qué necesita?
6.3 ¿Por qué el ecosistema se mueve de Ingress a Gateway API? Nombrá una limitación del Ingress que Gateway API resuelve.

---

## Módulo 7 — Configuración: ConfigMaps y Secrets

**Teoría.** Tu app necesita configuración: la URL de Postgres, flags, niveles de log, API keys. La regla de oro (la *Twelve-Factor App*, que viste en config y deploy): **la config va separada del código/imagen**, en el entorno. La misma imagen `task-api:v2` corre en dev, staging y prod cambiando solo la config inyectada. K8s te da dos objetos para esto:

- **ConfigMap**: para configuración **no sensible** (URLs, nombres de features, niveles de log, parámetros). Un objeto key-value que inyectás en tus Pods como **variables de entorno** o como **archivos montados**.
- **Secret**: para datos **sensibles** (contraseñas, API keys, tokens, certificados). Se usa igual que un ConfigMap (env vars o archivos), pero está pensado para secretos.

La advertencia **crítica** que separa al que entiende de seguridad del que no: **un Secret de K8s NO está encriptado por defecto, solo está codificado en base64.** Base64 no es cifrado —cualquiera con acceso lo decodifica en un segundo—. Por defecto los Secrets se guardan así en etcd. Para que sean realmente seguros necesitás:

- **Encriptación en reposo** de etcd (configurable en el cluster), y/o
- Un **gestor de secretos externo** (AWS Secrets Manager, Vault) integrado, donde el secreto real vive afuera y K8s solo lo referencia.

Tratar un Secret de K8s como "seguro porque dice Secret" es un error clásico. Es *un poco* mejor que un ConfigMap (acceso más restringido, no se loguea tan fácil), pero base64 ≠ cifrado.

El patrón en la práctica, conectando con el módulo de seguridad (las API keys nunca en el código):

```yaml
# La API Key de Anthropic (del track de IA) inyectada desde un Secret
spec:
  containers:
    - name: api
      env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef: { name: api-secrets, key: anthropic-key }
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef: { name: api-config, key: log-level }
```

La frase mental: **ConfigMap para lo que podés mostrar, Secret para lo que no — pero "Secret" no significa "cifrado".** Separar config de imagen es lo que hace que una sola imagen sea promovible entre entornos; tratar los secretos con el cuidado real (encriptación/gestor externo) es lo que lo hace seguro.

**Quién puede leer qué: RBAC.** Lo anterior cierra el círculo con el control de acceso. K8s usa **RBAC** (Role-Based Access Control) para decidir qué puede hacer cada identidad sobre cada recurso, con cuatro piezas:

- **ServiceAccount**: la identidad de un Pod (o de una persona/CI) ante el API server.
- **Role** (dentro de un namespace) / **ClusterRole** (todo el cluster): un conjunto de permisos ("puede `get`/`list` ConfigMaps en `prod`").
- **RoleBinding** / **ClusterRoleBinding**: atan un Role a una ServiceAccount/usuario.

El principio es el de siempre: **mínimo privilegio**. Un Pod que solo necesita leer un ConfigMap recibe una ServiceAccount con un Role que permite *exactamente* eso, nada más. Y acá se ve por qué los Secrets "base64" (de arriba) son tan sensibles al RBAC: si cualquier identidad puede `get secrets` en el namespace, el base64 no protege nada. RBAC restrictivo + encriptación en reposo + gestor externo es la defensa en capas real.

**Ejercicios 7**
7.1 ¿Cuál es la diferencia entre un ConfigMap y un Secret, y de qué dos formas inyectás ambos en un Pod?
7.2 ¿Por qué es peligroso creer que un Secret de K8s "está encriptado"? ¿Qué necesitás para que lo esté de verdad?
7.3 ¿Por qué separar la config de la imagen? Conectá con la Twelve-Factor App y con poder promover una imagen entre entornos.
7.4 Nombrá las piezas de RBAC y conectá: ¿por qué un RBAC laxo hace todavía más peligroso que un Secret sea "solo base64"?

---

## Módulo 8 — Salud, recursos y autoscaling

**Teoría.** Para que el auto-sanado y el escalado de K8s funcionen, K8s necesita saber dos cosas de cada Pod: **si está sano** y **cuántos recursos usa**. Estas son las piezas que hacen que la teoría de los módulos anteriores funcione de verdad en producción.

**Probes (chequeos de salud).** K8s sondea tus contenedores con HTTP/comandos para decidir qué hacer. Las **tres** —y confundirlas es un error común—:

- **Liveness probe** (¿está vivo?): si falla, K8s **reinicia el contenedor**. Para detectar un proceso colgado/deadlockeado que sigue "corriendo" pero no responde. Pregunta: "¿hay que matarlo y reiniciarlo?".
- **Readiness probe** (¿está listo para recibir tráfico?): si falla, K8s **lo saca del Service** (deja de mandarle requests) pero **no lo reinicia**. Para cuando el Pod arranca pero todavía no está listo (calentando, conectando a la base), o está sobrecargado. Pregunta: "¿le mando tráfico ahora?".
- **Startup probe** (¿terminó de arrancar?): mientras no pasa, K8s **suspende la liveness y la readiness**. Es la que protege a las apps de **arranque lento** (Node con warmup, migraciones, una JVM): sin ella, terminás poniendo un `initialDelaySeconds` enorme en liveness —o, peor, la liveness mata al Pod a mitad del boot y entrás en un ciclo de reinicios—. Una vez que pasa, la liveness toma el control. El presupuesto de arranque lo das con `failureThreshold × periodSeconds`.

La readiness es **clave para el deploy sin downtime** (módulo 4): durante un rolling update, K8s espera a que los Pods v2 pasen la readiness antes de mandarles tráfico y bajar los v1. Sin readiness, ruteás requests a un Pod que todavía no puede atender → errores en cada deploy.

Las tres, dentro del `container` (van en el Deployment del módulo 4):

```yaml
# spec.template.spec.containers[0]:
startupProbe:                       # da hasta 30×2 = 60s para arrancar
  httpGet: { path: /health, port: 3000 }
  failureThreshold: 30
  periodSeconds: 2
livenessProbe:                      # si se cuelga, reiniciar
  httpGet: { path: /health, port: 3000 }
  periodSeconds: 10
readinessProbe:                     # si no está listo, sacar del Service
  httpGet: { path: /ready, port: 3000 }
  periodSeconds: 5
```

**Recursos: requests y limits.** Por contenedor declarás:

- **requests**: lo que el contenedor **necesita garantizado**. El scheduler (módulo 2) lo usa para decidir en qué nodo cabe.
- **limits**: el **techo** que no puede pasar. Si supera el límite de memoria, lo matan (OOMKill); el de CPU, lo estrangulan (throttle).

Sin requests/limits, un Pod puede acaparar un nodo y voltear a sus vecinos ("noisy neighbor"). Ponerlos es higiene básica de producción.

**Autoscaling con el HPA (Horizontal Pod Autoscaler).** Acá se cierra el círculo del escalado: el HPA **ajusta automáticamente el número de réplicas** de un Deployment según una métrica. "Si el uso de CPU promedio pasa el 70%, sumá réplicas (hasta 30); si baja, sacá (hasta 3)":

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: { name: task-api }
spec:
  scaleTargetRef: { apiVersion: apps/v1, kind: Deployment, name: task-api }
  minReplicas: 3
  maxReplicas: 30
  metrics:
    - type: Resource
      resource: { name: cpu, target: { type: Utilization, averageUtilization: 70 } }
```

Eso convierte el escalado manual del módulo 4 en automático y reactivo al tráfico —y, combinado con requests bien puestos, en eficiente en costo (el FinOps del módulo de AWS)—. Para que el HPA pueda calcular el % de CPU necesita el **metrics-server** instalado, y el `requests` del contenedor como base del cálculo. Un **caveat** que conviene saber: escalar por **memoria** suele ser mala señal (muchos runtimes no devuelven memoria tras el GC, así que la métrica no baja y no hay scale-down); CPU o métricas custom (req/seg, vía un adapter) son lo habitual. (Hay también escalado vertical y de nodos, pero el HPA horizontal es el pan de cada día.)

**Ejercicios 8**
8.1 ¿Cuál es la diferencia entre una liveness probe y una readiness probe? ¿Qué hace K8s cuando falla cada una?
8.2 ¿Por qué la readiness probe es clave para un deploy sin downtime?
8.3 ¿Qué diferencia hay entre `requests` y `limits`, y qué hace el HPA?
8.4 **Leé el manifiesto.** Un Deployment de una API Node no tiene `readinessProbe`, ni `resources`, ni `startupProbe`. Nombrá un problema concreto que cause cada una de esas tres ausencias en producción.
8.5 ¿Por qué la startup probe existe además de la liveness, y qué pasa si una app de arranque lento solo tiene liveness?

---

## Módulo 9 — Estado: volúmenes, PV/PVC y StatefulSets

**Teoría.** Vimos que los Pods son efímeros y no guardan estado (módulo 3). Pero algunas apps **sí** necesitan persistir datos (una base de datos, una cola, un cache que sobreviva). ¿Cómo se concilia "el Pod es desechable" con "necesito que estos datos sobrevivan al Pod"? Con la capa de almacenamiento de K8s.

- **Volume**: almacenamiento que se monta en un Pod. Algunos tipos viven y mueren con el Pod (sirven para data temporal); para persistencia real necesitás los de abajo.
- **PersistentVolume (PV)**: una pieza de almacenamiento real del cluster (un disco EBS en AWS, etc.), que existe **independiente del ciclo de vida de los Pods**. El Pod muere, el PV (y sus datos) sigue.
- **PersistentVolumeClaim (PVC)**: una **solicitud** de almacenamiento que hace tu app ("dame 10 GB"). K8s la satisface con un PV (a menudo aprovisionado automáticamente vía una **StorageClass**). El PVC desacopla "necesito disco" de "qué disco físico es" —el mismo desacople declarativo de siempre—.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: datos }
spec:
  accessModes: ["ReadWriteOnce"]   # un solo nodo lo monta a la vez
  storageClassName: gp3            # qué tipo de disco (lo provee la StorageClass)
  resources: { requests: { storage: 10Gi } }
```

El Pod monta ese PVC como un volumen; K8s aprovisiona el PV real (un EBS, etc.) por detrás. El Pod muere, el PVC y sus datos siguen.

Para apps con estado que además necesitan **identidad estable** (cada réplica es distinguible, no intercambiable como en un Deployment), está el **StatefulSet**: como un Deployment, pero da a cada Pod un **nombre estable** (`db-0`, `db-1`...), **almacenamiento propio persistente** que lo sigue aunque se reinicie, y arranque/orden controlado. Es lo que usarías para correr, por ejemplo, una base de datos replicada dentro del cluster.

Pero acá viene el **criterio senior**, que es lo más importante del módulo: **correr bases de datos con estado en Kubernetes es difícil y, muchas veces, no deberías.** Gestionar la persistencia, los backups, el failover y las réplicas de una base en K8s es complejo y arriesgado. La práctica recomendada en la enorme mayoría de los casos: **dejá las bases de datos afuera del cluster, en un servicio gestionado** (RDS/Aurora para Postgres —el módulo de AWS práctica—, DynamoDB, ElastiCache para Redis). Tu cluster corre lo **sin estado** (tu API, tus workers), que es donde K8s brilla, y se conecta a las bases gestionadas por la red. Saber que K8s *puede* correr estado (StatefulSets, PV/PVC) pero que **normalmente no conviene** es exactamente el tipo de criterio que se espera de un Tech Lead —y conecta directo con el módulo siguiente y con todo el track: la herramienta puede hacerlo no significa que sea la mejor opción.

**Ejercicios 9**
9.1 ¿Qué diferencia hay entre un PersistentVolume (PV) y un PersistentVolumeClaim (PVC)? ¿Qué desacopla el PVC?
9.2 ¿Qué le da un StatefulSet a sus Pods que un Deployment no, y para qué tipo de app sirve?
9.3 ¿Por qué la recomendación senior es NO correr tu base de datos en K8s, y qué hacés en su lugar? Conectá con AWS.

---

## Módulo 10 — Lo declarativo: manifiestos, kubectl y Helm

**Teoría.** Todo lo que viste —Deployments, Services, ConfigMaps, etc.— son **objetos** que se definen en **manifiestos YAML** y se aplican al cluster. Esto cierra la idea declarativa del módulo 1 a nivel operativo: **tu infraestructura de aplicación es código (YAML), versionado en git.** No clickeás en una consola; describís el estado deseado en archivos y los aplicás.

La herramienta es **`kubectl`** (la CLI que habla con el API server):

```bash
kubectl apply -f deployment.yaml   # crea/actualiza objetos hacia el estado deseado
kubectl get pods                   # ver el estado actual
kubectl describe pod task-api-xxx  # detalle / eventos de un objeto (debug)
kubectl logs task-api-xxx          # logs de un Pod
kubectl rollout undo deploy/task-api  # rollback (módulo 4)
```

El comando que encarna todo K8s es **`kubectl apply`**: le das el YAML del estado deseado y K8s reconcilia. Aplicás el mismo archivo dos veces y no pasa nada raro (es **idempotente**) —porque no es "hacé esto" sino "que el mundo se vea así"—. Esto es **GitOps** en su forma más simple: el estado del cluster vive en git, y se aplica desde ahí; conecta directo con el CI/CD del módulo de Docker (el pipeline corre `kubectl apply` en el deploy).

**Diagnosticar un Pod que no arranca** (el reflejo que separa a quien *opera* un cluster):

```bash
kubectl describe pod <pod>     # mirá la sección Events al final: ahí está el "por qué"
kubectl logs <pod> --previous  # logs del contenedor anterior si crasheó y reinició
kubectl exec -it <pod> -- sh   # entrar al contenedor a inspeccionar
kubectl port-forward <pod> 3000:3000  # acceder al Pod desde tu máquina sin exponerlo
```

Los estados típicos y qué miran: **`ImagePullBackOff`** (no pudo bajar la imagen → nombre/tag/credenciales del registry); **`CrashLoopBackOff`** (el contenedor arranca y muere en loop → `logs --previous` para ver la excepción de boot); **`Pending`** (no hay nodo con recursos para los `requests`, o falta un PVC). La sección **Events** de `describe` casi siempre te dice cuál es.

El problema que aparece rápido: **el YAML se multiplica y se repite.** Para cada app tenés Deployment + Service + ConfigMap + Ingress..., y para cada entorno (dev/staging/prod) casi lo mismo con pequeñas diferencias. Copiar y pegar YAML no escala. Las dos soluciones estándar:

- **Helm**: el "gestor de paquetes" de K8s. Empaqueta un conjunto de manifiestos en un **chart** **plantillable**: definís el YAML con variables y le pasás distintos **values** por entorno (réplicas, imagen, recursos). Un comando (`helm install`) despliega todo el stack. Además hay charts públicos para instalar cosas comunes (un controller, una herramienta) sin escribir el YAML a mano.
- **Kustomize**: integrado en `kubectl`, parte de un YAML base y le aplica **overlays** (parches) por entorno, sin plantillas.

La frase mental: **en K8s declarás tu sistema en YAML versionado y `kubectl apply` lo reconcilia; cuando el YAML se vuelve repetitivo, Helm/Kustomize lo parametrizan por entorno.** Es la misma progresión de IaC del módulo de AWS, ahora para la capa de aplicación.

**Ejercicios 10**
10.1 ¿Por qué se dice que `kubectl apply` es idempotente, y cómo se relaciona con lo declarativo del módulo 1?
10.2 ¿Qué problema resuelven Helm/Kustomize que el YAML plano no? Da el caso de los múltiples entornos.
10.3 ¿Cómo conecta `kubectl apply` con el CI/CD del módulo de Docker (qué corre el pipeline en el deploy)?

---

## Módulo 11 — El criterio: ¿necesitás Kubernetes?

**Teoría.** El módulo más importante para un Tech Lead, y el que cierra todo. Kubernetes es **poderosísimo**, pero esa potencia viene con una **carga operativa enorme**: hay que gestionar el cluster, los upgrades, el networking, el almacenamiento, la seguridad (RBAC), el monitoreo... K8s es prácticamente un sistema operativo distribuido, y operarlo bien requiere gente y tiempo. El error caro no es "no saber K8s"; es **adoptarlo cuando no lo necesitás y ahogar a tu equipo en complejidad accidental.**

**Cuándo K8s vale la pena:**

- Tenés **muchos servicios** (microservicios reales, decenas) que necesitan orquestación, descubrimiento y escalado coordinado.
- Necesitás **portabilidad** real: correr igual en AWS, en otro cloud y on-premise (K8s es el estándar que evita el vendor lock-in del orquestador).
- Querés **control fino** sobre scheduling, networking y políticas, y tenés un equipo (o plataforma) que puede operarlo.
- Tu escala y complejidad justifican la inversión operativa.

**Cuándo K8s es overkill (la mayoría de los proyectos):**

- **Pocos servicios, equipo chico**: la carga operativa de K8s supera de lejos el beneficio.
- Tenés alternativas más simples que cubren el caso con una fracción del esfuerzo:
  - **Contenedores gestionados sin orquestador propio**: **AWS Fargate** (con ECS), **Google Cloud Run**, **AWS App Runner**, **Azure Container Apps**. Corrés tu contenedor, ellos escalan y operan; vos no tocás un cluster. Esta es la comparación clave del track ("¿K8s o Fargate?"): para muchísimos casos, **Fargate/Cloud Run hace el 80% de lo que querías de K8s con el 10% de la complejidad operativa**.
  - **PaaS** (Railway, Render, Fly.io, Heroku): "subí tu código, nosotros nos ocupamos". Aún más simple.
  - **Serverless** (Lambda, el módulo de AWS): si tu carga es por eventos/requests, ni siquiera necesitás contenedores corriendo todo el tiempo.

**El costo, parte del criterio.** K8s no es gratis ni siquiera gestionado: pagás el **control plane** (EKS/GKE cobran por hora de cluster), los **nodos encendidos aunque estén ociosos** (reservás capacidad), y el tiempo de la gente que lo opera. Fargate y serverless, en cambio, escalan a cero o casi (pagás por uso): para cargas intermitentes o equipos chicos, esa diferencia —de infraestructura **y** operativa— suele decidir sola. Un matiz útil: si **ya tenés** un cluster, las tareas batch/cron del backend (migraciones, limpiezas, reportes) entran natural como **Jobs** (corren una vez hasta completar) y **CronJobs** (en horario), sin sumar otra herramienta; pero levantar un cluster *solo* para un cron es justo el overkill que este módulo te enseña a evitar.

La pregunta-criterio: **¿de verdad necesito orquestar muchos contenedores con control fino y portabilidad, o un servicio gestionado de contenedores me da lo que quiero sin operar un cluster?** Si tenés tres servicios y un equipo de cinco, la respuesta casi siempre es "Fargate/Cloud Run, no K8s".

Y el criterio integrador de **todo el temario**, una vez más: **la solución más simple que resuelve el problema real.** Para infraestructura significa: empezá con lo gestionado y simple (Fargate, un PaaS, serverless), y subí a K8s **solo cuando** tengas un problema concreto que lo justifique —muchos servicios, necesidad de portabilidad, control que lo gestionado no te da—. Saber que K8s es el estándar y entender sus fundamentos **es** valioso (lo vas a cruzar, y es pregunta de entrevista segura); pero saber **cuándo NO usarlo** es lo que distingue a un Tech Lead que diseña sistemas que el equipo puede sostener, del que arrastra a todos a un cluster que nadie sabe operar a las 3am.

**Ejercicios 11**
11.1 ¿Cuál es el principal costo de Kubernetes más allá de aprenderlo, y cuál es el error de Tech Lead asociado?
11.2 Nombrá dos situaciones donde K8s vale la pena y dos donde es overkill (con la alternativa que usarías).
11.3 ¿Cuál es la pregunta-criterio "¿K8s o Fargate?", y cómo se relaciona con el criterio integrador del temario?
11.4 Más allá de la carga operativa, ¿qué costos concretos tiene un cluster gestionado frente a Fargate/serverless? Y si ya tenés cluster, ¿con qué objeto correrías una tarea programada (un reporte nocturno)?

---

# Soluciones

> Mirá esto solo después de intentarlo.

### Módulo 1
```
1.1 (Tres de:) correr y repartir muchas réplicas en varias máquinas; reiniciar contenedores
    caídos y reubicar los de un nodo muerto (auto-sanado); deploy sin downtime; escalar arriba
    y abajo según el tráfico; descubrimiento entre contenedores con IPs cambiantes.
1.2 Que le declarás el estado deseado (no órdenes) y K8s trabaja para que la realidad coincida.
    Loop de reconciliación: deseás 20 réplicas; si una muere, la realidad (19) ≠ lo deseado (20),
    K8s detecta la brecha y crea una nueva hasta volver a 20.
1.3 Con la Infraestructura como Código (módulo de AWS práctica): en ambos declarás el QUÉ (el
    estado deseado) y la herramienta se ocupa del CÓMO.
```

### Módulo 2
```
2.1 El control plane es el cerebro que decide (ej. el scheduler, que elige en qué nodo va cada
    Pod); los nodos son las máquinas worker donde corren los contenedores (ej. el kubelet, que
    se asegura de que los contenedores asignados estén corriendo y sanos).
2.2 Guarda todo el estado del cluster (qué debería correr, la config) como key-value
    distribuido: es la fuente de verdad. Es crítico porque si lo perdés, perdés el cluster.
2.3 Mandás "20 réplicas" al API server → se guarda en etcd → el controller manager ve que
    faltan Pods → el scheduler elige nodos → el kubelet de cada nodo arranca los contenedores
    vía el container runtime.
2.4 Un namespace es una agrupación lógica que aísla recursos dentro de un mismo cluster (dev,
    staging, prod, o por equipo); la mayoría de los objetos viven dentro de uno. Se aplican "por
    namespace" (dos de): RBAC, ResourceQuota, NetworkPolicy, y los propios Secrets/ConfigMaps
    (uno de un namespace no lo ve otro).
```

### Módulo 3
```
3.1 Un Pod es la unidad mínima desplegable: uno o más contenedores que comparten red y
    almacenamiento y se programan juntos. Se usa el envoltorio porque a veces necesitás
    contenedores acoplados que viven/mueren juntos, como un sidecar (ej. un proxy o recolector
    de logs junto a tu app, compartiendo localhost y disco).
3.2 Que se crean y destruyen constantemente y cada Pod nuevo tiene una IP distinta. Por eso NO
    guardás estado en un Pod (sesiones/archivos importantes) y NO te conectás a un Pod por su IP.
3.3 Con el principio "stateless" de Node/Redis: el estado va afuera (Postgres, Redis,
    almacenamiento persistente) porque la instancia puede desaparecer; K8s lo lleva al extremo
    asumiendo que los Pods mueren todo el tiempo.
```

### Módulo 4
```
4.1 Qué correr (la imagen), cuántas réplicas, y cómo actualizar (rolling update). Por debajo
    crea un ReplicaSet, que garantiza que siempre haya N Pods corriendo.
4.2 Rolling update: levanta algunos Pods v2, espera a que pasen la readiness (estén sanos),
    baja algunos v1, y repite gradualmente hasta que todo es v2 — el servicio nunca deja de
    responder porque siempre hay Pods sanos atendiendo.
4.3 Porque K8s recuerda el estado anterior (es declarativo): `kubectl rollout undo` vuelve a
    declarar la v1 como estado deseado y K8s reconcilia hacia ella en segundos.
```

### Módulo 5
```
5.1 Resuelve que los Pods son efímeros con IPs cambiantes: da una IP/nombre DNS estables por
    delante de ellos. Elige a qué Pods enruta por etiqueta (label selector, ej. app: task-api)
    y balancea entre los Pods sanos que matchean.
5.2 Le pega por el nombre DNS del Service (ej. http://pagos): K8s tiene DNS interno, así que el
    nombre del Service es la dirección estable. No hardcodeás IPs porque el Service incorpora
    solo los Pods nuevos cuando cambian.
5.3 ClusterIP expone el Service solo dentro del cluster (para que tus servicios se hablen entre
    sí); LoadBalancer pide un balanceador del cloud y lo expone a internet (para acceso externo).
5.4 Lo más probable: el `selector` del Service no coincide con las labels de los Pods, así que
    el Service no tiene endpoints. Revisás primero `kubectl get endpoints <svc>` (¿vacío?) y que
    el selector del Service matchee exactamente las labels del template del Deployment.
```

### Módulo 6
```
6.1 Un Ingress define reglas de ruteo HTTP/HTTPS por host/path hacia varios Services internos,
    con TLS centralizado. No alcanza un LoadBalancer por cosa porque pedir un balanceador del
    cloud por cada servicio es caro y no rutea por host/path.
6.2 Porque el objeto Ingress es solo la regla (declarativa); necesita un Ingress Controller, un
    componente corriendo en el cluster que lee esas reglas y efectivamente rutea el tráfico.
6.3 Porque el Ingress quedó corto: poca expresividad y cada controller inventaba anotaciones
    propias no portables. Gateway API lo resuelve con un modelo más rico, orientado a roles
    (separa operador de infra y dev de app) y portable entre implementaciones.
```

### Módulo 7
```
7.1 ConfigMap es para config no sensible (URLs, flags, log level); Secret para datos sensibles
    (contraseñas, API keys, tokens). Ambos se inyectan como variables de entorno o como archivos
    montados en el Pod.
7.2 Porque un Secret por defecto solo está codificado en base64, que NO es cifrado (cualquiera
    con acceso lo decodifica al instante). Para que sea seguro necesitás encriptación en reposo
    de etcd y/o un gestor de secretos externo (AWS Secrets Manager, Vault).
7.3 Para que la misma imagen sea promovible entre entornos (dev/staging/prod) cambiando solo la
    config inyectada (Twelve-Factor App: la config va en el entorno, no en el código/imagen).
7.4 Piezas: ServiceAccount (identidad del Pod), Role/ClusterRole (conjunto de permisos),
    RoleBinding/ClusterRoleBinding (atan el Role a la identidad). Conexión: si el RBAC es laxo y
    cualquier identidad puede `get secrets`, el base64 no protege nada (se decodifica al
    instante); con mínimo privilegio, solo quien debe leer el Secret puede pedirlo.
```

### Módulo 8
```
8.1 Liveness: ¿está vivo? Si falla, K8s reinicia el contenedor (para un proceso colgado).
    Readiness: ¿está listo para tráfico? Si falla, K8s lo saca del Service (deja de mandarle
    requests) pero NO lo reinicia (para un Pod que arranca o está sobrecargado).
8.2 Porque durante el rolling update K8s espera a que los Pods v2 pasen la readiness antes de
    mandarles tráfico y bajar los v1; sin readiness, ruteás requests a Pods que aún no pueden
    atender → errores en cada deploy.
8.3 requests = lo que el contenedor necesita garantizado (el scheduler lo usa para ubicarlo);
    limits = el techo que no puede pasar (lo matan por OOM o lo estrangulan en CPU). El HPA
    (Horizontal Pod Autoscaler) ajusta automáticamente el número de réplicas según una métrica
    (CPU, requests/seg), escalando arriba y abajo con el tráfico.
8.4 Sin readinessProbe: K8s rutea tráfico a Pods que aún no pueden atender → errores en cada
    deploy. Sin resources: el Pod puede acaparar CPU/memoria del nodo y voltear vecinos (noisy
    neighbor), y el scheduler no sabe dónde ubicarlo. Sin startupProbe: si la app arranca lento,
    la liveness la mata a mitad del boot → CrashLoopBackOff.
8.5 La startup probe suspende liveness/readiness hasta que la app termina de arrancar; existe
    para los arranques lentos. Si una app de boot lento solo tiene liveness, la liveness empieza
    a contar desde el inicio y la mata antes de que termine de levantar → ciclo de reinicios.
```

### Módulo 9
```
9.1 El PV es una pieza de almacenamiento real del cluster que existe independiente del ciclo de
    vida de los Pods (el Pod muere, el PV y sus datos siguen); el PVC es la solicitud de
    almacenamiento que hace la app ("dame 10 GB"). El PVC desacopla "necesito disco" de "qué
    disco físico es".
9.2 Le da identidad estable: nombre fijo por Pod (db-0, db-1), almacenamiento propio persistente
    que lo sigue aunque se reinicie, y orden de arranque controlado. Sirve para apps con estado
    donde cada réplica es distinguible (ej. una base de datos replicada).
9.3 Porque gestionar persistencia, backups, failover y réplicas de una base en K8s es complejo y
    arriesgado. En su lugar: dejás la base afuera, en un servicio gestionado (RDS/Aurora,
    DynamoDB, ElastiCache) y corrés solo lo stateless en el cluster, conectándote por red.
```

### Módulo 10
```
10.1 Porque aplicás el mismo YAML las veces que quieras y el resultado es el mismo: no es "hacé
     esto" sino "que el mundo se vea así". Es lo declarativo del módulo 1 a nivel operativo:
     describís el estado deseado y K8s reconcilia.
10.2 Que el YAML se multiplica y repite (Deployment+Service+ConfigMap+Ingress por app, casi
     igual por entorno). Helm plantilla los manifiestos en un chart con values por entorno;
     Kustomize parte de un base y aplica overlays. Resuelven el caso de múltiples entornos sin
     copiar y pegar YAML.
10.3 El pipeline de CI/CD, en el paso de deploy, corre `kubectl apply` (o `helm upgrade`) contra
     el cluster con el YAML versionado en git — es GitOps: el estado del cluster vive en git y
     se aplica desde el pipeline.
```

### Módulo 11
```
11.1 La enorme carga operativa: gestionar cluster, upgrades, networking, almacenamiento,
     seguridad, monitoreo (K8s es casi un SO distribuido). El error de Tech Lead: adoptarlo
     cuando no se necesita y ahogar al equipo en complejidad accidental.
11.2 Vale la pena con: muchos servicios que orquestar; necesidad de portabilidad multi-cloud/
     on-prem (con un equipo que pueda operarlo). Overkill con: pocos servicios y equipo chico
     → Fargate/Cloud Run/App Runner; carga por eventos → serverless (Lambda); o un PaaS
     (Railway/Render) para "subí tu código y listo".
11.3 "¿De verdad necesito orquestar muchos contenedores con control fino y portabilidad, o un
     servicio gestionado (Fargate/Cloud Run) me da lo que quiero sin operar un cluster?". Se
     relaciona con el criterio integrador del temario: la solución más simple que resuelve el
     problema real — empezá con lo gestionado y subí a K8s solo cuando se justifique.
11.4 Costos: el control plane gestionado (EKS/GKE cobran por hora de cluster) + los nodos
     encendidos aunque estén ociosos (capacidad reservada), frente al pago-por-uso y scale-to-zero
     de Fargate/serverless. Si ya tenés cluster, un reporte nocturno se corre con un CronJob (y
     una tarea única con un Job).
```

---

## Siguientes pasos

Con este módulo tenés los fundamentos de Kubernetes con criterio: el modelo declarativo y el loop de reconciliación, la arquitectura (control plane y nodos), los objetos que usás todos los días (Pods, Deployments, Services, Ingress/Gateway, ConfigMaps/Secrets), las piezas que hacen funcionar el auto-sanado y el escalado (probes, requests/limits, HPA), cómo se maneja el estado (PV/PVC, StatefulSets) y por qué las bases van afuera del cluster, el flujo declarativo con `kubectl`/Helm, y —lo más importante— **el criterio de cuándo K8s y cuándo Fargate/serverless/PaaS**. Este es el segundo módulo del **track Tech Lead**. Lo que sigue: **observabilidad práctica** (OpenTelemetry, logs/métricas/traces, tracing distribuido — que engancha directo con el tracing del módulo 8 de Evals y con la operación de un cluster), **TDD como disciplina** (red-green-refactor, profundizando el módulo de Testing) y **liderazgo técnico** (ADRs, métricas DORA, Team Topologies). Con ellos se completa la otra mitad del perfil senior: ya no solo construir y desplegar, sino **operar, medir y decidir** sistemas que el equipo pueda sostener.
