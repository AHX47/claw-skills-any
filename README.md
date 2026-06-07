# Skills — any thing ...
<img width="1024" height="572" alt="image" src="https://github.com/user-attachments/assets/626bb674-2214-4782-a4f1-90b93383326d" />


## 📁 Structure

```
claw-skills-any/
├── .gitignore
├── README.md
├── agentic/                 # Agent design patterns & frameworks
│   ├── SKILL.md             # Core agentic principles
│   ├── PATTERNS.md          # ReAct, planning, memory, multi-agent
│   ├── FRAMEWORKS.md        # LangChain, CrewAI, LangGraph, AutoGen
│   └── a
├── ai/                      # AI/ML specific skills
│   ├── dataset-generation-skill.md
│   ├── model-training-skill.md
│   ├── ocr-vision-3d-skill.md
│   └── rag-memory-skill.md
├── architecture/            # System architecture skills
│   └── strategic-architecture-skill.md
├── automation/              # Automation & scripting
│   ├── SKILL.md             # Core automation principles
│   ├── RECIPES.md           # CI/CD, Docker, Makefile, cron
│   ├── scraping-deepsearch-report-skill.md
│   ├── social-media-automation-skill.md
│   └── a
├── backend/                 # Backend development
│   ├── backend-design-skill.md
│   └── database-security-skill.md
├── coding/                  # General coding practices
│   ├── SKILL.md             # Core coding principles
│   ├── PATTERNS.md          # Language patterns & security
│   └── a
├── core/                    # Core language skills
│   ├── flet-design-skill.md
│   ├── java-skill.md
│   ├── python-skill.md
│   └── react-nodejs-skill.md
├── frontend/                # Frontend development
│   └── frameworks-skill.md
├── security/                # Security-focused skills
│   ├── security-research-skill.md
│   └── supply-chain-security-skill.md
├── systems/                 # Systems programming
│   ├── assembly-skill.md
│   └── linux-debug-skill.md
└── tools/                   # Tooling & integration
    ├── thinking-integration-agi-skill.md
    └── a
```

---
## 🚀 Quick Reference – All Skill Folders

### 🤖 Agentic
- **ReAct loop** – thought → action → observation, with fallback & max iterations  
- **Task planning** – hierarchical decomposition, dependency graph, checkpointing  
- **Memory system** – short‑term (conversation buffer), long‑term (vector store), cache (LRU)  
- **Multi‑agent orchestrator** – round‑robin, handoff, voting, supervisor pattern  
- **Tool registry** – auto‑generated OpenAPI docs, type validation, rate limiting  
- **Human‑in‑the‑loop** – escalation triggers, approval gates, clarification prompts  
- **Frameworks** – LangChain (chains), CrewAI (roles), LangGraph (state graphs), AutoGen (conversable agents)

### 🧠 AI
- **Dataset generation** – synthetic data (Faker, CTGAN), augmentation (albumentations), weak labeling, train/val/test splits  
- **Model training** – hyperparameter tuning (Optuna), checkpointing, early stopping, distributed (DeepSpeed, FSDP)  
- **OCR & vision** – Tesseract / EasyOCR, layout analysis (layoutparser), 3D point cloud (Open3D, PCL)  
- **RAG memory** – chunking (semantic / fixed), vector retrieval (FAISS, Chroma), reranking (cross‑encoders), memory summarization

### ⚙️ Automation
- **File watching** – watchdog, inotify, debouncing, safe atomic writes  
- **Web scraping** – rate limiting (tenacity), retries, rotating user‑agents, Playwright stealth  
- **Scheduling** – Python schedule, cron / systemd timers, timezone handling  
- **ETL pipeline** – extract (APIs, DBs), transform (pandas, polars), load (S3, BigQuery)  
- **CI/CD** – GitHub Actions (matrix, caching), GitLab CI, Jenkins  
- **Container recipes** – Docker multi‑stage, .dockerignore, docker‑compose with secrets  
- **Makefile templates** – phony targets, env vars, help generation  
- **Monitoring** – health checks, Prometheus metrics, alertmanager, log rotation

### 💻 Coding
- **Language rules** – Python (black, mypy), TypeScript (strict, ESLint), React (hooks, memo), SQL (parameterised, indexes)  
- **Debugging workflow** – reproducer → bisect → logs (structured) → hypothesis → fix + regression test  
- **Code review checklist** – security (no secrets), performance (O complexity), edge cases, naming  
- **Security checklist** – input validation, output encoding, auth, rate limiting, dependency scanning  
- **Common patterns** – retry (exponential backoff), singleton (thread‑safe), observer (event bus), result type (Rust‑style `Ok`/`Err`)

### 🏛️ Architecture
- **Strategic architecture** – C4 model (context, containers, components, code)  
- **Decision records** – ADR template with status, context, consequences  
- **Quality attributes** – scalability (horizontal/vertical), resilience (circuit breakers), observability (logging, metrics, tracing)  
- **Integration patterns** – event‑driven (Kafka), REST vs GraphQL, saga orchestration  

### 🔙 Backend
- **Backend design** – layered architecture (controller → service → repository), DTOs, dependency injection  
- **Database & security** – connection pooling (pgbouncer), migration tools (Alembic, Flyway), row‑level security, encryption at rest, SQL injection prevention  

### 🎨 Frontend
- **Frameworks** – React (Next.js, Vite), Vue (Nuxt), SvelteKit  
- **State management** – Zustand / Redux / Pinia, server state (TanStack Query)  
- **Performance** – code splitting, lazy loading, image optimisation (sharp), CLS / FID metrics  
- **Accessibility** – ARIA labels, keyboard nav, colour contrast (WCAG 2.1)  

### 🔷 Core
- **Flet design** – Flutter controls from Python, page routing, theme management  
- **Java** – Maven/Gradle, streams, CompletableFuture, JUnit + Mockito  
- **Python** – virtual envs (venv/poetry), async (asyncio/aiohttp), type hints + pydantic  
- **React + Node.js** – React hooks, Next.js SSR, Express middleware, JWT authentication  

### 🔒 Security
- **Security research** – threat modelling (STRIDE), CVE tracking, exploit analysis, responsible disclosure  
- **Supply chain security** – SBOM generation (syft, CycloneDX), dependency scanning (Snyk, Dependabot), signature verification (Sigstore, SLSA)  

### ⚡ Systems
- **Assembly** – x86‑64 / ARM, calling conventions, syscall interface, inline assembly in C  
- **Linux debugging** – strace (syscalls), perf (profiling), gdb (core dumps), ftrace (kernel tracing), /proc /sys exploration  

### 🛠️ Tools
- **Thinking integration (AGI)** – chain‑of‑thought, tree‑of‑thoughts, self‑consistency, reasoning trace logging  
- **Integration patterns** – function calling, tool use, structured output (JSON schema), parallel tool execution  

---

## 💡 Usage with Claude

Copy these bullet points directly into your system prompt, or reference the original `SKILL.md` files:

> *“When writing production code, follow the **Coding** patterns: retry logic, security checklist, and debugging workflow.  
> For multi‑agent systems, implement **Agentic** memory and orchestrator patterns from agentic/SKILL.md.  
> Automate backups and ETL using **Automation** recipes – watch files, schedule cron, run GitHub Actions.  
> For vision‑RAG pipelines, apply **AI** dataset generation and vector retrieval from ai/rag-memory-skill.md.”*

