Yes, this is possible, and there are **two genuinely different ways** to do it depending on what you mean by "wrap." Let me clarify the distinction first because people conflate them, then show working code for both.

## The key distinction

A LangGraph ReAct agent (from `create_react_agent`) is really just **a model + a system prompt + a list of tools**, compiled into a graph. So "wrapping" it in a deep agent can mean:

1. **Lift-and-shift**: Take the ReAct agent's *ingredients* (prompt + tools) and rebuild it as a deep agent. The deep agent itself becomes your ReAct agent, but now with planning + subagent powers. This is what you almost always want.
2. **Nest as a subagent**: Keep the existing compiled ReAct graph as-is and expose it as a *callable specialist* that a new deep agent (the "lead") can delegate to via the `task` tool.

Option 1 upgrades your agent. Option 2 keeps your agent intact and puts a planner *on top* of it.

---

## Starting point: a typical existing ReAct agent

```python
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def web_search(query: str) -> str:
    """Search the web."""
    return f"[results for {query}]"

@tool
def fetch_url(url: str) -> str:
    """Fetch a URL's text."""
    return f"[content of {url}]"

model = ChatOpenAI(model="gpt-4o")

react_agent = create_react_agent(
    model=model,
    tools=[web_search, fetch_url],
    prompt="You are a helpful research assistant. Use the tools to answer questions.",
)
```

This works but has no planning loop and no delegation. Now let's upgrade it.

---

## Option 1 — Lift-and-shift (recommended)

You keep the same model, same tools, same prompt — but pass them to `create_deep_agent` instead. You get `write_todos`, the virtual filesystem, and the `task` tool for free.

```python
from deepagents import create_deep_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

# --- same tools your ReAct agent already had ---
@tool
def web_search(query: str) -> str:
    """Search the web."""
    return f"[results for {query}]"

@tool
def fetch_url(url: str) -> str:
    """Fetch a URL's text."""
    return f"[content of {url}]"

model = ChatOpenAI(model="gpt-4o")

# --- the upgrade: deep agent with the SAME ingredients ---
deep_agent = create_deep_agent(
    tools=[web_search, fetch_url],
    model=model,
    instructions=(
        "You are a helpful research assistant. "
        "For non-trivial requests, first call write_todos to plan. "
        "Work through todos using your tools, marking each complete as you finish. "
        "If a subtask would generate a lot of intermediate output, delegate it via the task tool."
    ),
)

inputs = {"messages": [("user",
    "Compare the architectures of Llama 3 and Mistral, then summarize the tradeoffs.")]}
config = {"configurable": {"thread_id": "session_001"}}

for chunk in deep_agent.stream(inputs, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

That's the entire migration. Your old `react_agent` variable is now obsolete — `deep_agent` is a strict superset. It still does ReAct-style tool calling under the hood (deep agents *are* ReAct agents internally; the "deep" part is the planning prompt + extra built-in tools + subagent machinery layered on top).

**Why this is the right default**: `create_react_agent` and `create_deep_agent` both produce a `CompiledStateGraph` with the same invocation interface (`.invoke`, `.stream`). There's no behavior you had in the ReAct agent that you lose. You just gain capabilities.

---

## Option 2 — Nest the existing compiled ReAct graph as a subagent

Use this when:
- The ReAct agent is complex and you don't want to touch it (custom state, checkpointer, middleware, etc.).
- You want a *new* lead agent whose job is purely planning and delegation, and the old ReAct agent becomes one of its "employees."
- You want context quarantine — the lead never sees the ReAct agent's tool-call churn, only its final answer.

```python
from deepagents import create_deep_agent
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

model = ChatOpenAI(model="gpt-4o")

# --- your existing ReAct agent, untouched ---
@tool
def web_search(query: str) -> str:
    """Search the web."""
    return f"[results for {query}]"

@tool
def fetch_url(url: str) -> str:
    """Fetch a URL's text."""
    return f"[content of {url}]"

existing_react_agent = create_react_agent(
    model=model,
    tools=[web_search, fetch_url],
    prompt="You are a research specialist. Gather facts and summarize.",
)

# --- register it as a subagent of a new deep agent ---
researcher_subagent = {
    "name": "researcher",
    "description": (
        "Delegate focused research questions here. "
        "Provide a clear question; returns a summary with sources."
    ),
    "graph": existing_react_agent,   # <-- the pre-built CompiledStateGraph
}

lead = create_deep_agent(
    tools=[],   # the lead itself has no direct tools; it only plans + delegates
    model=model,
    instructions=(
        "You are a project lead. Always start by calling write_todos to plan. "
        "Delegate every research subtask to the 'researcher' subagent via the task tool. "
        "Synthesize the researcher's outputs into a final answer."
    ),
    subagents=[researcher_subagent],
)

inputs = {"messages": [("user",
    "Compare the architectures of Llama 3 and Mistral, then summarize the tradeoffs.")]}
config = {"configurable": {"thread_id": "session_002"}}

for chunk in lead.stream(inputs, config, stream_mode="values"):
    chunk["messages"][-1].pretty_print()
```

What happens at runtime:

1. Lead calls `write_todos` → writes a plan: `["research Llama 3 architecture", "research Mistral architecture", "compare", "summarize"]`.
2. Lead calls `task(description="...Llama 3...", subagent_type="researcher")` → spins up your existing ReAct graph in isolation, runs it to completion, returns one summary message.
3. Lead calls `task(...)` again for Mistral.
4. Lead synthesizes both summaries into the final answer, marking todos complete along the way.

> **Version caveat (important)**: the field name for plugging in a pre-built graph has been the most volatile part of the `deepagents` API. Recent versions accept `"graph"`, but earlier ones used `"runnable"`, and the very latest may have renamed it again. If `graph` errors, run this to check:
> ```python
> import deepagents, inspect
> print(inspect.signature(deepagents.create_deep_agent))
> # and inspect the SubAgent TypedDict in deepagents/sub_agent.py
> ```
> If neither `graph` nor `runnable` is supported in your version, fall back to **Option 1** (lift-and-shift) — it always works because it doesn't depend on the nesting API at all.

---

## Which should you pick?

| Your situation | Pick |
|---|---|
| ReAct agent is just `model + tools + prompt` | **Option 1** — lift-and-shift |
| ReAct agent has custom state, middleware, checkpointer, or you literally cannot modify it | **Option 2** — nest as subagent |
| You want one planner coordinating *several* existing agents | **Option 2** — register each as a subagent |
| You're not sure | **Option 1** — it's simpler and always works |

## Mental model

A ReAct agent is a worker who picks up tools and does tasks. A deep agent is **the same worker, now with a notepad and a phone** — they can plan with the notepad (`write_todos`) and call employees with the phone (`task`). Option 1 hands the notepad and phone to your existing worker. Option 2 hires a new manager and puts your existing worker on speed dial.

Confidence: ~98% on Option 1 (it's just normal `create_deep_agent` usage). ~80% on Option 2's exact field name (`graph` vs `runnable`) — the *concept* of nesting a compiled graph as a subagent is officially supported, but verify the key against your installed version as noted above.