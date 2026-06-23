# De Frontend (React / React Native) a Full Stack con Node.js

**Roadmap · Edición 2026 · Fase 1 (10 semanas) + Fase 2 (senior / Tech Lead / IA Engineer)**

> Punto de partida asumido: dominás React y React Native, entendés REST/JSON, consumís APIs y ya hiciste algún CRUD con Node/Express. El objetivo de este plan no es enseñarte "qué es un servidor", sino **profesionalizarte** para postular a roles full stack con criterio de producción.

> **Cómo leer este roadmap.** Tiene dos fases. La **Fase 1** es un núcleo intensivo de ~10 semanas que te deja **contratable como full stack (junior / semi-senior)**: es el plan semana a semana de la sección 3. La **Fase 2** (sección "Más allá de las 10 semanas") es el recorrido que ya cubre este temario hacia perfil **senior / Tech Lead** e **IA Engineer** —PostgreSQL senior, Node por dentro, AWS con CDK, event-driven aplicado, Kubernetes, observabilidad y un track de IA completo—, sin plazo fijo. Si abrís solo la Fase 1, mirá igual la Fase 2: es la otra mitad del viaje.

---

## 1. Qué se está usando de verdad en backend JS en 2026

El mercado se reparte entre **Express** (lo más pedido por volumen, base de la mayoría del código existente), **NestJS** (el estándar de facto en empresas medianas/grandes por su arquitectura opinada) y un grupo emergente de alto rendimiento: **Fastify** y **Hono** (este último destaca en performance y está pensado para edge/serverless). La constante: **TypeScript de punta a punta** ("JavaScript everywhere"), porque es lo que te permite reutilizar tu experiencia de frontend y es prácticamente requisito en ofertas profesionales hoy.

### Stack objetivo recomendado (el que te hace contratable)

| Capa | Elección principal 2026 | Por qué / alternativas |
|------|------------------------|------------------------|
| Lenguaje | **TypeScript** | No negociable a nivel laboral. Reutilizás lo que ya sabés de React. |
| Runtime | **Node.js (LTS)** | El que piden las ofertas. Conocé que existen **Bun** y **Deno** (cada vez más en serverless), pero Node sigue siendo el que paga el sueldo. |
| Framework | **NestJS** (foco principal) + **Express** (saberlo leer) | Nest te enseña arquitectura limpia y es el más valorado en empresa. Aprendé Express porque está en casi todo el código heredado. Mirá **Fastify/Hono** como bonus de performance. |
| Base de datos | **PostgreSQL** (relacional) + nociones de **MongoDB** | Postgres es el default profesional. Mongo aparece en el stack MERN/startups. |
| ORM | **Drizzle** (moderno, ligero, SQL-first) y/o **Prisma** (más maduro, schema-first) | Drizzle es la elección recomendada para proyectos nuevos en 2026; Prisma sigue dominando en ecosistema y tutoriales. Aprendé uno bien y entendé el otro. |
| Caché / colas | **Redis** | Caché, sesiones, rate limiting, colas simples. Aparece en casi toda oferta. |
| Auth | **JWT** + **OAuth 2.1 / OpenID Connect**, hashing con **argon2id** (preferido) o **bcrypt** | Base de seguridad. argon2id es la recomendación actual de OWASP para proyectos nuevos; bcrypt sigue válido por compatibilidad/legacy. Suma Passport o better-auth. |
| API | **REST** (obligatorio) + nociones de **GraphQL** y **gRPC** | REST primero. GraphQL/gRPC como diferenciadores. |
| Mensajería | **Kafka** o **RabbitMQ** (conceptual) | Para arquitecturas event-driven; entender el "por qué" más que dominarlo. Se profundiza en el módulo de event-driven (Fase 2). |
| DevOps | **Docker** (imprescindible) + **Kubernetes** y **CI/CD** | Docker es must-have hoy. K8s suma mucho (módulo propio en Fase 2). Pipeline en GitHub Actions. |
| Cloud | **AWS** (o GCP/Azure) | Saber desplegar: contenedores, una DB gestionada, variables de entorno, logs. Práctica real con IaC (CDK) en Fase 2. |
| Observabilidad | **OpenTelemetry** + métricas/logs/traces, SLOs | Operar lo que desplegás: ver qué hace en producción y reducir el MTTR (módulo propio en Fase 2). |
| Testing | **Vitest/Jest** + **Supertest** | Tests unitarios y de integración de endpoints. |
| IA aplicada | **LLM APIs (Claude/SDK)** + **RAG (pgvector)** + **Agentes/MCP** + **Evaluations** | El diferenciador de empleabilidad 2026: integrar LLMs en tu backend con criterio. Track propio en Fase 2. |

---

## 2. Patrones de arquitectura más usados y mejor valorados

Estos son los que aparecen en entrevistas y los que separan a un junior de un mid:

1. **Monolito modular** — El "sweet spot" para la mayoría de productos en 2026: límites limpios entre módulos sin la complejidad operativa de microservicios. Es donde deberías empezar siempre.
2. **Arquitectura en capas (layered)** — Controller → Service → Repository. El patrón base que NestJS te impone naturalmente. Domínalo primero.
3. **Clean Architecture** — Separa reglas de negocio de la infraestructura. Muy valorada en dominios con reglas que cambian (precios, permisos, facturación).
4. **Hexagonal (Ports & Adapters)** — La evolución natural de Clean Architecture: tu lógica depende de interfaces, no de bases de datos o APIs concretas. Excelente para testabilidad. Considerado de los patrones más prácticos y "future-proof" para microservicios Node.
5. **Microservicios** — Servicios independientes, cada uno dueño de su data, comunicados por API o mensajes. Cambian complejidad de código por complejidad operativa: no empieces acá, pero entendé cuándo justifican el costo.

**Patrones tácticos que conviene conocer:** Repository, Dependency Injection (Nest lo trae de fábrica), DTOs + validación (Zod / class-validator), middleware/interceptors, manejo centralizado de errores, y CQRS (separar lecturas de escrituras) como concepto avanzado —se profundiza, aplicado, en el módulo de **event-driven** (Fase 2), junto con event sourcing, saga y el patrón outbox—.

> Consejo de entrevista: sabé *cuándo NO* usar microservicios. Demostrar criterio ("empezaría con un monolito modular y extraería servicios cuando el dominio lo pida") vale más que listar tecnologías.

---

## 3. Roadmap semana a semana — Fase 1 (plan intensivo ~10 semanas)

Esta es la **Fase 1**: el núcleo que te deja contratable como full stack. Cada bloque termina con un proyecto que va directo a tu portfolio.

> **Sé honesto con el ritmo.** El plan es **agresivo**: asumí **~15-20 h/semana** sostenidas y que ya programás (venís de frontend). Varias semanas están cargadas a propósito (S1 mete TS + Node interno + HTTP; S7 mete Docker + CI/CD + deploy). Si vas con menos horas, **estiralo sin culpa**: es mejor consolidar cada proyecto que correr. Los temas profundos (event loop/streams, infra) tienen módulos propios para volver con calma.

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
- **Documentá decisiones de arquitectura** (un breve ADR por proyecto): es justo lo que evalúan en entrevistas mid —y es la antesala del **liderazgo técnico** de la Fase 2—.
- **Leé código ajeno** de repos NestJS bien valorados para absorber convenciones.
- **Cuando domines lo anterior, dá el salto a la Fase 2** (sección 6): la disciplina de **TDD** (red-green-refactor), la mitad operativa (observabilidad, K8s, AWS práctico) y el **track de IA**. Ahí es donde el perfil pasa de "contratable" a "senior / Tech Lead".

---

## 6. Más allá de las 10 semanas — Fase 2 (de full stack a senior / Tech Lead / IA Engineer)

La Fase 1 te deja **contratable**. La Fase 2 es lo que te lleva de "sé entregar" a "sé entregar **y operar** sistemas de producción, y diseñar con criterio" —el salto a **senior / Tech Lead**— más un **track de IA** que es el mayor diferenciador de empleabilidad en 2026. No tiene plazo fijo: se hace por módulos, cuando un proyecto o un objetivo laboral lo pide. Todo esto **ya está cubierto en el temario**:

**Profundización del core (de junior a senior):**
- [PostgreSQL nivel senior](postgresql.md) — aislamiento/locks, MVCC, índices/EXPLAIN, paginación keyset, pooling, JSONB.
- [Node.js por dentro](nodejs.md) — event loop a fondo, thread pool de libuv, streams, cluster, escalado.
- [Redis: caché, colas y jobs](redis.md) — cache-aside, stampede, BullMQ, idempotencia, rate limiting atómico.
- [Testing práctico](testing.md) y [TDD como disciplina](tdd.md) (red-green-refactor).
- [Patrones de diseño en NestJS](nestjs-patrones.md) y [NestJS nivel Senior](nestjs-senior.md).

**Operación y nube (perfil Tech Lead):**
- [Docker, deploy y CI/CD](docker-deploy.md) — multi-stage, PID 1, estrategias de deploy, healthchecks.
- [AWS para backend](aws.md) (criterio) + [AWS hands-on con CDK](aws-practica.md) (infra como código, RDS/S3/Fargate, FinOps, Well-Architected).
- [Kubernetes](kubernetes.md) — cuándo K8s vs Fargate, y los fundamentos.
- [Observabilidad práctica](observabilidad.md) — OpenTelemetry, los 3 pilares, SLO/error budgets, alerting por burn rate.
- [Arquitectura event-driven aplicada](event-driven.md) — event sourcing, CQRS, saga, outbox/CDC.
- [Tiempo real (WebSockets) y Swagger](tiempo-real.md) — features en vivo + documentación de API.
- [De senior a Tech Lead](liderazgo.md) — ADRs, métricas DORA, Team Topologies, la IA en el rol.

**Track de IA — conceptos (stack Claude/TS, el diferenciador 2026):**
- [IA Generativa y LLMs](ia-llms.md) — fundamentos de integrar LLMs (Claude/SDK) en tu backend.
- [RAG: responder con tu data](rag.md) — embeddings, pgvector sobre Postgres, retrieval híbrido, reranking.
- [Agentes (loop, MCP, Claude Agent SDK)](agentes.md) — de un workflow a un sistema que decide su camino.
- [Evaluations](evals.md) — medir un sistema de IA: golden set, LLM-as-judge, tracing.
- Bases NoSQL: [MongoDB / DynamoDB](nosql.md).

**Track Python AI (perfil AI app developer / AI QA — el ecosistema Python):** una segunda dirección, abierta en 2026, para apuntar a roles de IA en stack Python. Complementa el track conceptual de arriba, no lo reemplaza.
- [Python para quien viene de TS/Node](python.md) y [FastAPI](fastapi.md) — el lenguaje y el framework de APIs del ecosistema de IA.
- [Prompt Engineering](prompt-engineering.md) — especificar el modelo con medición, no a ojo.
- [Vector Databases y RAG en Python](vector-dbs.md) — internals de ANN (FAISS/HNSW) y RAG con clientes nativos.
- [AI Agents en Python](ai-agents-python.md) — LangGraph/AutoGen, multi-agente, MCP y A2A.
- [Voice AI](voice-ai.md) — STT/TTS y agentes de voz; [IA multimodal y visión](multimodal.md) — VLMs y Document AI.
- [Seguridad de IA y red-teaming](seguridad-ia.md) — jailbreaks, prompt injection, OWASP LLM Top 10, guardrails.
- [Fine-tuning y adaptación](fine-tuning.md) — cuándo y cómo (LoRA/QLoRA, DPO/RLVR, distillation) y cuándo NO.
- [Ingeniería de software asistida por IA](ai-assisted-coding.md) — dirigir agentes de coding con criterio.
- [Deploy de aplicaciones de IA](deploy-ai.md) — servir modelos, GPU/cuantización, inferencia local, costos.

**Track QA / Automation (perfil AI QA):**
- [Playwright: E2E de browser](playwright.md) y [API Testing y Contract Testing](api-testing.md) — testing en Python, con el ángulo de **testear features de IA** (no determinismo, contrato, red-teaming).

**ML clásico / Predictivo (frontera con el rol ML Engineer):**
- [ML clásico y recomendadores](ml-recommenders.md) — cuándo NO es un LLM la respuesta: gradient boosting, recommenders de dos etapas, y la frontera AI Engineer vs ML Engineer.

> El hilo conductor del track de IA es el mismo sistema que crece: una llamada simple → un extractor → RAG sobre tus propios apuntes → un agente → evals → desplegado y asegurado. Conecta con el resto: el tracing de un sistema de IA es la misma observabilidad de microservicios; pgvector vive sobre el Postgres que ya sabés operar. El **track Python AI + QA** es la especialización hacia roles de **AI app developer / AI QA** en stack Python, validado por el panel de revisión del temario.

---

## Resumen ejecutivo

**Fase 1 — camino más corto a "full stack contratable" (junior / semi-senior):** **TypeScript + NestJS + PostgreSQL + Drizzle/Prisma + Redis + Docker**, con **JWT/OAuth** para auth y **arquitectura en capas evolucionando a hexagonal**. Empezá por un monolito modular bien hecho, no por microservicios. Construí 4-5 proyectos desplegados (uno aprovechando tu React Native) y prepará system design básico. En ~2-3 meses intensivos (a buen ritmo) eso te deja en condiciones reales de postular.

**Fase 2 — de full stack a senior / Tech Lead / IA Engineer (sin plazo fijo):** profundizás el core (Postgres/Node/Redis senior, TDD), sumás la mitad operativa (AWS+CDK, Kubernetes, observabilidad, event-driven, CI/CD real) y el **track de IA** (LLMs, RAG, agentes+MCP, evaluations) más la especialización **Python AI / AI QA** (FastAPI, prompt engineering, agentes en Python, voz, multimodal, seguridad de IA, fine-tuning, deploy y testing de IA) que hoy es el mayor diferenciador. El camino no termina en "contratable": sigue hacia "el que diseña y opera el sistema".
