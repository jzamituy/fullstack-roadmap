# De Frontend (React / React Native) a Full Stack con Node.js

**Roadmap intensivo · 2-3 meses · Edición 2026**

> Punto de partida asumido: dominás React y React Native, entendés REST/JSON, consumís APIs y ya hiciste algún CRUD con Node/Express. El objetivo de este plan no es enseñarte "qué es un servidor", sino **profesionalizarte** para postular a roles full stack con criterio de producción.

---

## 1. Qué se está usando de verdad en backend JS en 2026

El mercado se reparte entre **Express** (lo más pedido por volumen, base de la mayoría del código existente), **NestJS** (el estándar de facto en empresas medianas/grandes por su arquitectura opinada) y un grupo emergente de alto rendimiento: **Fastify** y **Hono** (este último líder en benchmarks y pensado para edge/serverless). La constante: **TypeScript de punta a punta** ("JavaScript everywhere"), porque es lo que te permite reutilizar tu experiencia de frontend y es prácticamente requisito en ofertas profesionales hoy.

### Stack objetivo recomendado (el que te hace contratable)

| Capa | Elección principal 2026 | Por qué / alternativas |
|------|------------------------|------------------------|
| Lenguaje | **TypeScript** | No negociable a nivel laboral. Reutilizás lo que ya sabés de React. |
| Runtime | **Node.js (LTS)** | El que piden las ofertas. Conocé que existen **Bun** y **Deno** (cada vez más en serverless), pero Node sigue siendo el que paga el sueldo. |
| Framework | **NestJS** (foco principal) + **Express** (saberlo leer) | Nest te enseña arquitectura limpia y es el más valorado en empresa. Aprendé Express porque está en casi todo el código heredado. Mirá **Fastify/Hono** como bonus de performance. |
| Base de datos | **PostgreSQL** (relacional) + nociones de **MongoDB** | Postgres es el default profesional. Mongo aparece en el stack MERN/startups. |
| ORM | **Drizzle** (moderno, ligero, SQL-first) y/o **Prisma** (más maduro, schema-first) | Drizzle es la elección recomendada para proyectos nuevos en 2026; Prisma sigue dominando en ecosistema y tutoriales. Aprendé uno bien y entendé el otro. |
| Caché / colas | **Redis** | Caché, sesiones, rate limiting, colas simples. Aparece en casi toda oferta. |
| Auth | **JWT** + **OAuth 2.1 / OpenID Connect**, hashing con **bcrypt/argon2** | Base de seguridad. Suma una librería como Passport o better-auth. |
| API | **REST** (obligatorio) + nociones de **GraphQL** y **gRPC** | REST primero. GraphQL/gRPC como diferenciadores. |
| Mensajería | **Kafka** o **RabbitMQ** (conceptual) | Para arquitecturas event-driven; entender el "por qué" más que dominarlo. |
| DevOps | **Docker** (imprescindible) + nociones de **Kubernetes** y **CI/CD** | Docker es must-have hoy. K8s suma mucho. Pipeline en GitHub Actions. |
| Cloud | **AWS** (o GCP/Azure) | Saber desplegar: contenedores, una DB gestionada, variables de entorno, logs. |
| Testing | **Vitest/Jest** + **Supertest** | Tests unitarios y de integración de endpoints. |

---

## 2. Patrones de arquitectura más usados y mejor valorados

Estos son los que aparecen en entrevistas y los que separan a un junior de un mid:

1. **Monolito modular** — El "sweet spot" para la mayoría de productos en 2026: límites limpios entre módulos sin la complejidad operativa de microservicios. Es donde deberías empezar siempre.
2. **Arquitectura en capas (layered)** — Controller → Service → Repository. El patrón base que NestJS te impone naturalmente. Domínalo primero.
3. **Clean Architecture** — Separa reglas de negocio de la infraestructura. Muy valorada en dominios con reglas que cambian (precios, permisos, facturación).
4. **Hexagonal (Ports & Adapters)** — La evolución natural de Clean Architecture: tu lógica depende de interfaces, no de bases de datos o APIs concretas. Excelente para testabilidad. Considerado de los patrones más prácticos y "future-proof" para microservicios Node.
5. **Microservicios** — Servicios independientes, cada uno dueño de su data, comunicados por API o mensajes. Cambian complejidad de código por complejidad operativa: no empieces acá, pero entendé cuándo justifican el costo.

**Patrones tácticos que conviene conocer:** Repository, Dependency Injection (Nest lo trae de fábrica), DTOs + validación (Zod / class-validator), middleware/interceptors, manejo centralizado de errores, y CQRS (separar lecturas de escrituras) como concepto avanzado.

> Consejo de entrevista: sabé *cuándo NO* usar microservicios. Demostrar criterio ("empezaría con un monolito modular y extraería servicios cuando el dominio lo pida") vale más que listar tecnologías.

---

## 3. Roadmap semana a semana (plan intensivo ~10 semanas)

Asume varias horas al día. Cada bloque termina con un proyecto que va directo a tu portfolio.

### Semana 1 — TypeScript de backend + fundamentos sólidos
- TypeScript profundo: tipos, generics, utility types, narrowing, `tsconfig` estricto.
- Node por dentro: event loop, async/await real, streams, módulos ESM, variables de entorno (`dotenv`), manejo de errores.
- HTTP a fondo: métodos, status codes, headers, idempotencia, REST bien diseñado.
- **Mini-proyecto:** CLI o script en TS que consuma una API pública y procese datos (afianza TS sin UI).

### Semana 2 — API REST profesional con NestJS
- Estructura de Nest: módulos, controllers, providers, dependency injection.
- Capas Controller → Service → Repository.
- Validación con DTOs (class-validator / Zod), pipes, manejo global de excepciones.
- **Proyecto 1 — "Task API":** API REST de tareas/proyectos con CRUD completo, validación y errores bien manejados (en memoria por ahora).

### Semana 3 — PostgreSQL + ORM
- Modelado relacional: claves, relaciones, índices, normalización, migraciones.
- SQL a mano (SELECT/JOIN/agregaciones) — no dependas solo del ORM.
- Drizzle o Prisma: schema, migraciones, queries tipadas, relaciones, transacciones.
- **Proyecto 1 (continuación):** conectá la Task API a Postgres con migraciones y relaciones (usuarios ↔ proyectos ↔ tareas).

### Semana 4 — Autenticación y autorización
- Registro/login, hashing con bcrypt o argon2.
- JWT (access + refresh tokens), sesiones, cookies httpOnly.
- Roles y permisos (RBAC), guards de Nest.
- OAuth 2.1 / login con Google.
- **Proyecto 2 — "Auth Service":** servicio de autenticación reusable con refresh tokens, roles y recuperación de contraseña. Conéctalo a tu Task API.

### Semana 5 — Caché, colas y tareas en background con Redis
- Redis: caché de respuestas, invalidación, rate limiting, sesiones.
- Colas de trabajos (BullMQ): envío de emails, procesamiento diferido.
- Cron jobs / tareas programadas.
- **Proyecto 3 — "Notification/Jobs":** sistema que encola y procesa notificaciones por email con reintentos, más caché de endpoints pesados.

### Semana 6 — Testing y calidad
- Unit tests (Vitest/Jest), mocks de dependencias.
- Tests de integración de endpoints con Supertest.
- Cobertura, test de la capa de repositorio con DB de prueba (testcontainers).
- Linting, formato, commits convencionales.
- **Tarea:** llevar tus proyectos 1-3 a >70% de cobertura significativa.

### Semana 7 — Docker + despliegue + CI/CD
- Dockerfile multi-stage para Node, `docker-compose` (app + Postgres + Redis).
- Variables de entorno y secrets, logs estructurados (pino), healthchecks.
- Pipeline en GitHub Actions: lint → test → build → deploy.
- Deploy real (Railway, Render, Fly.io o AWS ECS) con DB gestionada.
- **Tarea:** dejá un proyecto corriendo en la nube con URL pública y CI verde.

### Semana 8 — Arquitectura aplicada
- Refactor de un proyecto a **Clean / Hexagonal**: separá dominio, casos de uso e infraestructura.
- Patrones: Repository, inyección de dependencias, DTOs, mapeo dominio↔persistencia.
- Documentación de API con OpenAPI/Swagger.
- **Proyecto 4 — "E-commerce / SaaS core":** catálogo + carrito + pedidos con arquitectura hexagonal, eventos de dominio y tests. Este es tu proyecto estrella.

### Semana 9 — Full Stack real + tiempo real + temas avanzados
- Conectá un frontend React (o tu app React Native) a tu backend: el círculo full stack completo.
- WebSockets / Socket.io para features en tiempo real (chat, notificaciones live).
- Nociones de GraphQL y gRPC; introducción conceptual a microservicios y mensajería (Kafka/RabbitMQ).
- **Proyecto 5 — capstone:** una app full stack end-to-end (frontend + backend + DB + auth + deploy). Ejemplos abajo.

### Semana 10 — Portfolio, preparación laboral y system design
- Pulí READMEs, diagramas de arquitectura y demos desplegadas.
- Practicá **system design** básico: "diseñá un acortador de URLs", "un sistema de reservas", explicando capas, DB, caché y escalado.
- Repasá patrones, seguridad (OWASP top 10) y preguntas típicas de Node (event loop, async, escalado horizontal).
- Actualizá CV/LinkedIn con palabras clave: *Node.js, TypeScript, NestJS, REST APIs, PostgreSQL, Docker, Redis, microservices, CI/CD*.

---

## 4. Proyectos para el portfolio (de menor a mayor impacto)

Cada proyecto debe estar **desplegado, con README, tests y diagrama**. Calidad sobre cantidad: 3 proyectos sólidos pesan más que 10 a medias.

1. **Task/Project Management API** — CRUD, relaciones, auth, validación. (Bases.)
2. **Auth Service reutilizable** — JWT + refresh + RBAC + OAuth. (Demuestra seguridad.)
3. **Sistema de colas y notificaciones** — Redis + BullMQ + emails con reintentos. (Demuestra async/escalado.)
4. **Core de E-commerce o SaaS multi-tenant** — arquitectura hexagonal, eventos de dominio, Swagger, tests. (Tu proyecto estrella; muestra arquitectura.)
5. **App full stack capstone** — elegí una:
   - Plataforma tipo **Trello/Linear** (tu frontend React + backend Nest + tiempo real).
   - **Chat en tiempo real** con WebSockets, presencia y historial.
   - **Backend para tu app React Native** (ideal: aprovechás tu experiencia mobile y cerrás el full stack).
   - **Clon funcional de un SaaS** (acortador de URLs con analytics, gestor de suscripciones, etc.).

> Tip: que al menos un proyecto reutilice tu fuerte de **React Native** con un backend propio. Es un diferenciador real frente a backenders que no saben mobile.

---

## 5. Cómo estudiar para que rinda

- **80% construyendo, 20% leyendo.** Tu ventaja como frontend es que ya sabés programar; no caigas en "tutorial hell".
- **Una herramienta por categoría, bien.** No aprendas Prisma *y* Drizzle a la vez: dominá una, conocé la otra.
- **Git desde el día 1** en todos los proyectos, con commits limpios.
- **Documentá decisiones de arquitectura** (un breve ADR por proyecto): es justo lo que evalúan en entrevistas mid.
- **Leé código ajeno** de repos NestJS bien valorados para absorber convenciones.

---

## Resumen ejecutivo

Tu camino más corto a "full stack contratable": **TypeScript + NestJS + PostgreSQL + Drizzle/Prisma + Redis + Docker**, con **JWT/OAuth** para auth y **arquitectura en capas evolucionando a hexagonal**. Empezá por un monolito modular bien hecho, no por microservicios. Construí 4-5 proyectos desplegados (uno aprovechando tu React Native) y prepará system design básico. En 2-3 meses intensivos eso te deja en condiciones reales de postular.
