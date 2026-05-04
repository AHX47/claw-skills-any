# Agentic Patterns — Reference Implementations

## 1. Minimal ReAct Agent (Python)

```python
import json
from typing import Callable

class ReActAgent:
    """Minimal ReAct (Reasoning + Acting) agent loop."""

    def __init__(self, llm_call: Callable, tools: dict[str, Callable],
                 max_steps: int = 10, verbose: bool = True):
        self.llm       = llm_call
        self.tools     = tools
        self.max_steps = max_steps
        self.verbose   = verbose
        self.history   = []

    def run(self, goal: str) -> str:
        self.history = [{"role": "user", "content": goal}]
        tool_descriptions = "\n".join(
            f"- {name}: {fn.__doc__}" for name, fn in self.tools.items()
        )
        system = f"""You are a helpful agent. Available tools:
{tool_descriptions}

Respond in JSON:
{{"thought": "your reasoning", "action": "tool_name or FINISH", "action_input": "input"}}

Use action "FINISH" with action_input as your final answer when done."""

        for step in range(self.max_steps):
            response = self.llm(system=system, messages=self.history)
            if self.verbose:
                print(f"[Step {step+1}] {response}")

            try:
                parsed = json.loads(response)
            except json.JSONDecodeError:
                break

            self.history.append({"role": "assistant", "content": response})

            action = parsed.get("action", "FINISH")
            action_input = parsed.get("action_input", "")

            if action == "FINISH":
                return action_input

            if action not in self.tools:
                observation = f"Error: tool '{action}' not found"
            else:
                try:
                    observation = str(self.tools[action](action_input))
                except Exception as e:
                    observation = f"Error: {e}"

            if self.verbose:
                print(f"[Observation] {observation}")
            self.history.append({"role": "user", "content": f"Observation: {observation}"})

        return "Max steps reached without completion"


# Usage example:
# def search(q): return f"Results for: {q}"
# def calculate(expr): return str(eval(expr))
# agent = ReActAgent(llm_call=my_llm, tools={"search": search, "calculate": calculate})
# result = agent.run("What is 15% of the current AAPL stock price?")
```

---

## 2. Task Planner + Executor

```python
from dataclasses import dataclass, field
from typing import Any

@dataclass
class Step:
    id: int
    description: str
    tool: str
    args: dict
    depends_on: list[int] = field(default_factory=list)
    result: Any = None
    status: str = "pending"  # pending | running | done | failed

class PlanExecutor:
    def __init__(self, tools: dict[str, callable]):
        self.tools = tools

    def execute_plan(self, steps: list[Step]) -> dict[int, Any]:
        results = {}
        for step in steps:
            # Wait for dependencies
            for dep_id in step.depends_on:
                if results.get(dep_id) is None:
                    step.status = "failed"
                    continue
                # Inject dependency result into args if placeholder
                for k, v in step.args.items():
                    if v == f"${{step_{dep_id}}}":
                        step.args[k] = results[dep_id]

            step.status = "running"
            try:
                tool_fn = self.tools[step.tool]
                step.result = tool_fn(**step.args)
                step.status = "done"
                results[step.id] = step.result
            except Exception as e:
                step.status = "failed"
                step.result = str(e)
                results[step.id] = None

        return results
```

---

## 3. Memory System

```python
import json, time
from pathlib import Path

class AgentMemory:
    """Persistent + in-context memory for agents."""

    def __init__(self, persist_path: str = None, max_context: int = 20):
        self.context      = []           # recent messages (in-context)
        self.facts        = {}           # key facts extracted
        self.tool_cache   = {}           # cached tool results
        self.max_context  = max_context
        self.persist_path = Path(persist_path) if persist_path else None
        if self.persist_path and self.persist_path.exists():
            self._load()

    def add_message(self, role: str, content: str):
        self.context.append({"role": role, "content": content, "ts": time.time()})
        if len(self.context) > self.max_context:
            self.context.pop(0)

    def remember(self, key: str, value: any, ttl: int = None):
        self.facts[key] = {
            "value": value,
            "expires": time.time() + ttl if ttl else None
        }
        self._save()

    def recall(self, key: str) -> any:
        fact = self.facts.get(key)
        if not fact: return None
        if fact["expires"] and time.time() > fact["expires"]:
            del self.facts[key]; return None
        return fact["value"]

    def cache_tool(self, tool: str, args: str, result: any, ttl: int = 300):
        self.tool_cache[f"{tool}:{args}"] = {
            "result": result, "expires": time.time() + ttl
        }

    def get_cached(self, tool: str, args: str) -> any:
        cached = self.tool_cache.get(f"{tool}:{args}")
        if not cached: return None
        if time.time() > cached["expires"]: return None
        return cached["result"]

    def get_context(self) -> list[dict]:
        return [{"role": m["role"], "content": m["content"]} for m in self.context]

    def _save(self):
        if self.persist_path:
            self.persist_path.write_text(json.dumps({"facts": self.facts}))

    def _load(self):
        data = json.loads(self.persist_path.read_text())
        self.facts = data.get("facts", {})
```

---

## 4. Multi-Agent Orchestrator

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from queue import Queue
import threading

@dataclass
class AgentTask:
    task_id:  str
    goal:     str
    context:  dict
    priority: int = 0

@dataclass
class AgentResult:
    task_id: str
    agent:   str
    result:  str
    success: bool

class BaseAgent(ABC):
    def __init__(self, name: str, skills: list[str]):
        self.name   = name
        self.skills = skills

    @abstractmethod
    def can_handle(self, task: AgentTask) -> bool: ...

    @abstractmethod
    def execute(self, task: AgentTask) -> AgentResult: ...

class Orchestrator:
    def __init__(self, agents: list[BaseAgent]):
        self.agents  = agents
        self.queue   = Queue()
        self.results = {}

    def submit(self, task: AgentTask):
        self.queue.put(task)

    def route(self, task: AgentTask) -> BaseAgent:
        candidates = [a for a in self.agents if a.can_handle(task)]
        if not candidates:
            raise ValueError(f"No agent can handle task: {task.goal}")
        return candidates[0]  # or rank by confidence

    def run(self, tasks: list[AgentTask]) -> dict[str, AgentResult]:
        threads = []
        for task in tasks:
            agent = self.route(task)
            t = threading.Thread(target=self._execute, args=(agent, task))
            threads.append(t); t.start()
        for t in threads: t.join()
        return self.results

    def _execute(self, agent: BaseAgent, task: AgentTask):
        result = agent.execute(task)
        self.results[task.task_id] = result
```

---

## 5. Tool Registry

```python
from typing import Callable, Any
from functools import wraps
import inspect

class ToolRegistry:
    """Auto-documents and manages agent tools."""

    def __init__(self):
        self._tools: dict[str, dict] = {}

    def register(self, name: str = None, description: str = None,
                  risk_level: str = "low"):
        def decorator(fn: Callable):
            tool_name = name or fn.__name__
            sig = inspect.signature(fn)
            params = {
                k: str(v.annotation) for k, v in sig.parameters.items()
                if v.annotation != inspect.Parameter.empty
            }
            self._tools[tool_name] = {
                "fn":          fn,
                "description": description or fn.__doc__ or "",
                "parameters":  params,
                "risk_level":  risk_level,
            }
            @wraps(fn)
            def wrapper(*args, **kwargs): return fn(*args, **kwargs)
            return wrapper
        return decorator

    def call(self, name: str, **kwargs) -> Any:
        if name not in self._tools:
            raise KeyError(f"Tool '{name}' not registered")
        return self._tools[name]["fn"](**kwargs)

    def describe_all(self) -> str:
        lines = []
        for name, meta in self._tools.items():
            lines.append(f"- **{name}** [{meta['risk_level']}]: {meta['description']}")
            for p, t in meta["parameters"].items():
                lines.append(f"  - {p}: {t}")
        return "\n".join(lines)

# Usage:
# registry = ToolRegistry()
# @registry.register(description="Search the web", risk_level="low")
# def web_search(query: str) -> str:
#     ...
```

---

## 6. Streaming Agent Output

```python
import sys
from typing import Generator

def stream_agent_response(chunks: Generator[str, None, None],
                           prefix: str = "🤖 ") -> str:
    """Stream agent output token by token."""
    full = ""
    sys.stdout.write(prefix)
    for chunk in chunks:
        sys.stdout.write(chunk)
        sys.stdout.flush()
        full += chunk
    sys.stdout.write("\n")
    return full
```
