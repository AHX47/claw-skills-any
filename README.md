# Skills — Coding, Agentic, Automation

## 📁 Structure

```
skills-extra/
│
├── coding/
│   ├── SKILL.md       ← Core coding principles & best practices
│   └── PATTERNS.md    ← Python / JS / React / DB patterns with code
│
├── agentic/
│   ├── SKILL.md       ← Agent design: ReAct, planning, memory, guardrails
│   ├── PATTERNS.md    ← ReAct loop, task planner, memory, multi-agent, tools
│   └── FRAMEWORKS.md  ← LangChain, CrewAI, LangGraph, AutoGen, Anthropic API
│
└── automation/
    ├── SKILL.md       ← Automation principles, file/web/API/pipeline patterns
    └── RECIPES.md     ← CI/CD, Docker, Makefile, cron, systemd, scripts
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
