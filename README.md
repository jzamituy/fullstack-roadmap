# 🚀 Mi Hub Full Stack

Mi espacio personal para documentar el camino de **Frontend (React / React Native) → Full Stack con Node.js**. Apuntes, teoría y ejercicios, accesibles desde cualquier lado.

## ¿Qué hay acá?

Empezá por el **[🗺️ Roadmap Full Stack 2026](roadmap.md)** — el plan completo, el stack objetivo y los patrones de arquitectura. Desde ahí (o el menú lateral) llegás a cada módulo. Cada página sigue el mismo formato: **teoría → ejercicios → soluciones al final**.

**Fundamentos y backend**
- 📘 [TypeScript](typescript.md) · 🦅 [NestJS](nestjs.md) ([patrones](nestjs-patrones.md), [senior](nestjs-senior.md)) · ⚙️ [Node.js por dentro](nodejs.md)
- 🐘 [PostgreSQL](postgresql.md) · 🍃 [NoSQL (Mongo/DynamoDB)](nosql.md) · 🔐 [Autenticación](autenticacion.md)
- 🧪 [Testing](testing.md) · 🔴 [TDD](tdd.md) · ⚡ [Redis](redis.md) · 📡 [Tiempo real y Swagger](tiempo-real.md)

**Operación, nube y arquitectura (perfil senior / Tech Lead)**
- 🐳 [Docker, deploy y CI/CD](docker-deploy.md) · ☸️ [Kubernetes](kubernetes.md) · 🔭 [Observabilidad](observabilidad.md)
- ☁️ [AWS](aws.md) ([hands-on con CDK](aws-practica.md)) · 🧱 [DDD](ddd.md) · 🔀 [Event-driven](event-driven.md)
- 📱 [Mobile System Design](mobile-system-design.md) · 🧭 [De senior a Tech Lead](liderazgo.md)

**Inteligencia Artificial (el diferenciador 2026)**
- Conceptos (stack Claude): 🤖 [LLMs](ia-llms.md) · 📚 [RAG](rag.md) · 🦾 [Agentes](agentes.md) · 📊 [Evaluations](evals.md)
- **Track Python AI** (perfil AI app developer / AI QA): 🐍 [Python](python.md) · ⚡ [FastAPI](fastapi.md) · ✍️ [Prompt Engineering](prompt-engineering.md) · 🧮 [Vector DBs y RAG](vector-dbs.md) · 🦾 [AI Agents (LangGraph/AutoGen)](ai-agents-python.md) · 🎙️ [Voice AI](voice-ai.md) · 🖼️ [Multimodal y visión](multimodal.md) · 🛡️ [Seguridad de IA y red-teaming](seguridad-ia.md) · 🔧 [Fine-tuning](fine-tuning.md) · 🤖 [Coding asistido por IA](ai-assisted-coding.md) · 🚀 [Deploy de IA](deploy-ai.md)
- **QA / Automation:** 🎭 [Playwright](playwright.md) · 🔌 [API & Contract Testing](api-testing.md)
- **ML clásico / Predictivo:** 📊 [ML clásico y recomendadores](ml-recommenders.md)

> Esto es un trabajo en progreso: voy sumando temas a medida que avanzo.

---

### Cómo agregar contenido nuevo

1. Creá un archivo `mi-tema.md` en la raíz del repo.
2. Agregá un enlace en `_sidebar.md`.
3. `git add . && git commit -m "Agrego mi-tema" && git push`.

En segundos queda online. No hay paso de build.
