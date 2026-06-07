# Skills — Coding, Agentic, Automation

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

## 🚀 Quick Reference

### Coding
- Language rules: Python, TypeScript, React, SQL
- Debugging workflow and code review checklist
- Security checklist
- Common patterns: retry, singleton, observer, result type

### Agentic
- ReAct loop implementation
- Task planning & decomposition
- Memory system (short-term + long-term + cache)
- Multi-agent orchestrator
- Tool registry with auto-docs
- When to ask humans for input
- Frameworks: LangChain / CrewAI / LangGraph / AutoGen

### Automation
- File watching, batch processing, safe writes
- Web scraping with rate limiting
- Scheduling: Python `schedule`, cron, systemd timers
- ETL pipeline skeleton
- GitHub Actions CI/CD templates
- Docker + docker-compose recipes
- Makefile templates
- Monitoring & alerting scripts

---

## 💡 Usage with Claude

Reference these skills in your system prompt:
```
When writing code, follow the patterns in coding/SKILL.md.
When building agents, use the architecture from agentic/SKILL.md.
When automating tasks, apply principles from automation/SKILL.md.
```
