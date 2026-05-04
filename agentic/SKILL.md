# Agentic AI Skill — Claude Agent Patterns

## Overview
This skill guides Claude when operating as an **autonomous agent**: planning multi-step tasks, using tools, managing state, handling errors, and knowing when to ask for human input.

Use when: building agents, multi-step workflows, tool-using systems, ReAct loops, LangChain/CrewAI-style orchestration.

---

## Core Agentic Loop

```
OBSERVE → THINK → PLAN → ACT → OBSERVE → ...
```

Every agent cycle:
1. **Observe** — what is the current state? what tools/data are available?
2. **Think** — what is needed? what is the best next step?
3. **Plan** — break the goal into atomic steps
4. **Act** — execute ONE step at a time
5. **Verify** — did it work? adjust plan if needed
6. **Report** — communicate progress and results

---

## Planning Patterns

### Hierarchical Task Decomposition
```
Goal: "Build a REST API for user management"
│
├── 1. Design schema
│   ├── 1.1 Define User model
│   ├── 1.2 Define relationships
│   └── 1.3 Generate migrations
│
├── 2. Implement endpoints
│   ├── 2.1 POST /users (create)
│   ├── 2.2 GET /users/{id} (read)
│   ├── 2.3 PATCH /users/{id} (update)
│   └── 2.4 DELETE /users/{id} (delete)
│
└── 3. Add auth + tests
```

### ReAct Pattern (Reasoning + Acting)
```
Thought: I need to find the current price of AAPL stock.
Action: web_search("AAPL stock price today")
Observation: AAPL is trading at $185.42
Thought: Now I can calculate the portfolio value.
Action: calculate(shares=100, price=185.42)
Observation: Portfolio value = $18,542
Answer: Your 100 AAPL shares are worth $18,542
```

---

## Tool Use Best Practices

### Always verify tool availability before using
```python
available_tools = get_available_tools()
if "web_search" not in available_tools:
    # Fall back to knowledge or ask user
    ...
```

### Tool call principles
- Use the **minimum** number of tool calls needed
- **Batch** related operations when possible
- **Cache** results — don't call the same tool twice with same args
- **Validate** tool outputs before using them
- Handle tool **failures** gracefully

### Error handling for tools
```python
def safe_tool_call(tool, *args, max_retries=3, **kwargs):
    for attempt in range(max_retries):
        try:
            result = tool(*args, **kwargs)
            if result is None or "error" in str(result).lower():
                raise ValueError(f"Tool returned error: {result}")
            return result
        except Exception as e:
            if attempt == max_retries - 1:
                return {"error": str(e), "fallback": True}
            time.sleep(2 ** attempt)
```

---

## State Management

### Conversation Memory
```python
class AgentMemory:
    def __init__(self, max_tokens: int = 8000):
        self.short_term: list[dict] = []   # recent messages
        self.long_term: dict = {}          # persistent facts
        self.tool_cache: dict = {}         # cached tool results
        self.max_tokens = max_tokens

    def add(self, role: str, content: str):
        self.short_term.append({"role": role, "content": content})
        self._trim()

    def remember(self, key: str, value: any):
        self.long_term[key] = value

    def recall(self, key: str):
        return self.long_term.get(key)

    def _trim(self):
        # Remove oldest messages when context is too long
        while self._estimate_tokens() > self.max_tokens:
            self.short_term.pop(0)
```

### Task State Machine
```python
from enum import Enum

class TaskState(Enum):
    PENDING   = "pending"
    PLANNING  = "planning"
    EXECUTING = "executing"
    WAITING   = "waiting_for_human"
    DONE      = "done"
    FAILED    = "failed"

class Task:
    def __init__(self, goal: str):
        self.goal   = goal
        self.state  = TaskState.PENDING
        self.steps  = []
        self.results = {}
        self.errors  = []
```

---

## Human-in-the-Loop

### When to pause and ask
An agent MUST pause and ask for human input when:
- The action is **irreversible** (delete, send email, deploy to prod)
- The cost is **high** (API calls, money, time > 5 min)
- There is **ambiguity** in the goal (>1 valid interpretation)
- The plan **diverges** from what was discussed
- A **security-sensitive** operation is needed

### Confidence thresholds
```python
def should_proceed(confidence: float, action_risk: str) -> bool:
    thresholds = {
        "low":      0.5,   # Search, read files
        "medium":   0.75,  # Write files, API calls
        "high":     0.90,  # Database writes, emails
        "critical": 0.99,  # Deployments, deletions
    }
    return confidence >= thresholds.get(action_risk, 0.9)
```

---

## Multi-Agent Patterns

### Orchestrator → Worker
```
Orchestrator (planner)
    │
    ├── ResearchAgent  → searches, summarizes
    ├── CodeAgent      → writes, executes code
    ├── ReviewAgent    → validates output quality
    └── WriterAgent    → formats final output
```

### Agent Communication Protocol
```python
class AgentMessage:
    sender: str
    receiver: str
    message_type: str  # "task" | "result" | "error" | "clarify"
    content: dict
    priority: int = 0
    requires_ack: bool = False
```

### CrewAI-style Task Definition
```python
task = Task(
    description="Research the top 5 Python web frameworks in 2025",
    expected_output="A markdown table comparing frameworks by stars, speed, and use case",
    agent=research_agent,
    tools=[web_search, web_fetch],
    context=[previous_task],   # task dependencies
    max_iterations=5,
)
```

---

## Prompt Engineering for Agents

### System prompt structure
```
You are [ROLE].
Your goal is [GOAL].
You have access to these tools: [TOOLS].

Rules:
1. Always think step by step before acting
2. Use the minimum tools necessary
3. Verify results before proceeding
4. Ask for clarification when uncertain
5. Never take irreversible actions without confirmation

Output format:
- Thought: your reasoning
- Action: tool_name(args)
- Observation: result
- Final Answer: conclusion
```

### Chain-of-thought forcing
```python
system = """
Before answering, always:
1. Restate what is being asked
2. List what information you have
3. List what information is missing
4. Outline your approach
5. Execute step by step
6. Verify your answer
"""
```

---

## Guardrails

```python
BLOCKED_ACTIONS = [
    "delete_production_database",
    "send_mass_email",
    "deploy_to_production",
    "modify_billing",
]

def validate_action(action: str, args: dict) -> tuple[bool, str]:
    if action in BLOCKED_ACTIONS:
        return False, f"Action '{action}' requires explicit human approval"
    if args.get("environment") == "production":
        return False, "Production changes require human review"
    return True, ""
```

---

## Evaluation Metrics
- **Task completion rate** — did it finish the goal?
- **Step efficiency** — how many steps vs. minimum needed?
- **Error recovery rate** — how often does it self-correct?
- **Hallucination rate** — facts stated without tool verification
- **Human interrupt rate** — how often does it need help?

---

## Common Agent Architectures

| Architecture | Best For | Tools |
|---|---|---|
| ReAct | General reasoning + tools | LangChain |
| Plan-and-Execute | Long multi-step tasks | LangGraph |
| CrewAI | Multi-agent collaboration | CrewAI |
| AutoGen | Code generation + execution | AutoGen |
| Custom loop | Full control | Any LLM API |

---

## Anti-Patterns to Avoid
- **Infinite loops** — always set `max_iterations`
- **Tool spamming** — calling same tool 10x in a row
- **Hallucinated tool calls** — calling tools that don't exist
- **Ignoring errors** — proceeding after tool failures
- **Context overflow** — not managing memory/trimming
- **Overconfidence** — acting without enough info
