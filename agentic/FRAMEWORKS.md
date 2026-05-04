# Agentic Frameworks & Tools Reference

## LangChain Agent

```python
from langchain.agents import create_react_agent, AgentExecutor
from langchain.tools  import tool
from langchain_openai import ChatOpenAI

@tool
def search_web(query: str) -> str:
    """Search the internet for current information."""
    # implement with serpapi, tavily, etc.
    return f"Results for: {query}"

@tool
def run_python(code: str) -> str:
    """Execute Python code and return output."""
    import io, sys
    buf = io.StringIO()
    sys.stdout = buf
    try:
        exec(code, {})
        return buf.getvalue()
    except Exception as e:
        return f"Error: {e}"
    finally:
        sys.stdout = sys.__stdout__

llm   = ChatOpenAI(model="gpt-4o", temperature=0)
tools = [search_web, run_python]
agent = create_react_agent(llm, tools, prompt=hub.pull("hwchase17/react"))
executor = AgentExecutor(agent=agent, tools=tools, verbose=True,
                          max_iterations=10, handle_parsing_errors=True)
result = executor.invoke({"input": "What is the square root of the current AAPL stock price?"})
```

---

## CrewAI Multi-Agent

```python
from crewai import Agent, Task, Crew, Process
from crewai_tools import SerperDevTool, FileWriterTool

search_tool = SerperDevTool()
write_tool  = FileWriterTool()

researcher = Agent(
    role="Senior Research Analyst",
    goal="Find accurate, up-to-date information on the topic",
    backstory="Expert at finding and synthesizing information from multiple sources",
    tools=[search_tool],
    verbose=True,
    max_iter=5,
)

writer = Agent(
    role="Technical Writer",
    goal="Write clear, well-structured reports",
    backstory="Specialist in turning research into readable documents",
    tools=[write_tool],
    verbose=True,
)

research_task = Task(
    description="Research the top 5 Python frameworks for web APIs in 2025",
    expected_output="A detailed comparison with pros, cons, performance benchmarks",
    agent=researcher,
)

write_task = Task(
    description="Write a structured report based on the research",
    expected_output="A markdown report with sections: Overview, Comparison Table, Recommendations",
    agent=writer,
    output_file="api_frameworks_report.md",
    context=[research_task],
)

crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, write_task],
    process=Process.sequential,
    verbose=True,
)

result = crew.kickoff()
```

---

## LangGraph Stateful Agent

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages:  Annotated[list, operator.add]
    tool_calls: list
    final_answer: str | None

def should_continue(state: AgentState) -> str:
    if state.get("final_answer"):
        return END
    if len(state["messages"]) > 20:
        return END
    return "call_tool"

def agent_node(state: AgentState) -> AgentState:
    # Call LLM, get next action
    ...

def tool_node(state: AgentState) -> AgentState:
    # Execute tool calls
    ...

graph = StateGraph(AgentState)
graph.add_node("agent", agent_node)
graph.add_node("call_tool", tool_node)
graph.set_entry_point("agent")
graph.add_conditional_edges("agent", should_continue, {END: END, "call_tool": "call_tool"})
graph.add_edge("call_tool", "agent")

app = graph.compile()
result = app.invoke({"messages": [("user", "Analyze this data...")]})
```

---

## AutoGen Code Agent

```python
import autogen

llm_config = {
    "model": "gpt-4o",
    "api_key": "YOUR_KEY",
    "temperature": 0,
}

assistant = autogen.AssistantAgent(
    name="Coder",
    llm_config=llm_config,
    system_message="You are an expert Python programmer. Write clean, tested code.",
)

user_proxy = autogen.UserProxyAgent(
    name="User",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=10,
    is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
    code_execution_config={
        "work_dir": "coding_workspace",
        "use_docker": False,
    },
)

user_proxy.initiate_chat(
    assistant,
    message="Write a Python script that downloads the top 10 HackerNews stories and saves them to a CSV file.",
)
```

---

## Tool Calling (Anthropic / OpenAI)

### Anthropic tool use
```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a city",
        "input_schema": {
            "type": "object",
            "properties": {
                "city":    {"type": "string", "description": "City name"},
                "country": {"type": "string", "description": "Country code (ISO 2)"},
            },
            "required": ["city"],
        },
    }
]

def get_weather(city: str, country: str = "DZ") -> str:
    # Real implementation would call a weather API
    return f"Weather in {city}, {country}: 28°C, Sunny"

def run_agent(user_message: str) -> str:
    messages = [{"role": "user", "content": user_message}]
    while True:
        resp = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            tools=tools,
            messages=messages,
        )
        if resp.stop_reason == "end_turn":
            return next(b.text for b in resp.content if b.type == "text")
        # Process tool calls
        tool_results = []
        for block in resp.content:
            if block.type == "tool_use":
                fn   = {"get_weather": get_weather}[block.name]
                out  = fn(**block.input)
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": out,
                })
        messages.append({"role": "assistant", "content": resp.content})
        messages.append({"role": "user",      "content": tool_results})
```

---

## Vector Memory (RAG for Agents)

```python
from sentence_transformers import SentenceTransformer
import numpy as np
from dataclasses import dataclass, field

@dataclass
class VectorStore:
    model_name: str = "all-MiniLM-L6-v2"
    docs:       list[str]  = field(default_factory=list)
    embeddings: np.ndarray = None

    def __post_init__(self):
        self.model = SentenceTransformer(self.model_name)

    def add(self, text: str):
        self.docs.append(text)
        emb = self.model.encode([text])
        self.embeddings = emb if self.embeddings is None else np.vstack([self.embeddings, emb])

    def search(self, query: str, top_k: int = 3) -> list[tuple[str, float]]:
        if not self.docs: return []
        q_emb  = self.model.encode([query])
        scores = np.dot(self.embeddings, q_emb.T).flatten()
        idxs   = np.argsort(scores)[::-1][:top_k]
        return [(self.docs[i], float(scores[i])) for i in idxs]

# Usage:
# memory = VectorStore()
# memory.add("The user prefers Python over JavaScript")
# memory.add("The user is building a FastAPI application")
# results = memory.search("What language does the user prefer?")
```

---

## Frameworks Comparison

| Framework | Best For | Complexity | Multi-Agent | Streaming |
|-----------|----------|------------|-------------|-----------|
| LangChain | General agents + RAG | Medium | Via agents | ✅ |
| LangGraph | Complex stateful flows | High | ✅ | ✅ |
| CrewAI | Role-based collaboration | Low | ✅ Native | ✅ |
| AutoGen | Code generation/execution | Medium | ✅ | ✅ |
| Anthropic API | Direct tool use | Low | Manual | ✅ |
| Custom loop | Full control | Low | Manual | Manual |

---

## Prompt Templates for Agents

### Zero-shot agent
```
You are an AI assistant with access to tools.
Think step by step. Use tools when you need current information.
Be concise. When you have the answer, say "Final Answer: <answer>".
```

### Role-based specialist
```
You are a {role} with {years} years of experience in {domain}.
Your communication style is {style}.
You always {behavior_1} and never {behavior_2}.
When solving problems, you follow this process:
1. {step_1}
2. {step_2}
3. {step_3}
```

### Chain-of-thought
```
Before answering, think through this step by step:
<thinking>
1. What exactly is being asked?
2. What information do I have?
3. What information is missing?
4. What is the best approach?
</thinking>

Then provide your answer.
```
