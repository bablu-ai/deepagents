I have the full signature now. Let me give you a structured walkthrough — parameters first, then your specific questions about backends, skills, memory, middleware, cache, store, and a step-by-step upgrade path tailored to your Kubernetes constraints.

## The actual signature (current `main`)

```python
create_deep_agent(
    model:           str | BaseChatModel | None = None,
    tools:           Sequence[BaseTool | Callable | dict] | None = None,
    *,
    system_prompt:   str | SystemMessage | None = None,
    middleware:      Sequence[AgentMiddleware] = (),
    subagents:       Sequence[SubAgent | CompiledSubAgent | AsyncSubAgent] | None = None,
    skills:          list[str] | None = None,
    memory:          list[str] | None = None,
    response_format: ResponseFormat | None = None,
    context_schema:  type[Any] | None = None,
    checkpointer:    Checkpointer | None = None,
    store:           BaseStore | None = None,
    backend:         BackendProtocol | BackendFactory | None = None,
    interrupt_on:    dict[str, bool | InterruptOnConfig] | None = None,
    debug:           bool = False,
    name:            str | None = None,
    cache:           BaseCache | None = None,
) -> CompiledStateGraph
```

Note two things that have changed since your earlier code: the planning prompt parameter is now **`system_prompt`** (not `instructions`), and `tools` is now optional with a default of `[]`.

## Parameter-by-parameter

**`model`** — The LLM. String like `"openai:gpt-4o"` or a `BaseChatModel` instance. Defaults to Claude Sonnet 4. Must support tool calling.

**`tools`** — Tools the *lead* agent can call directly. The general-purpose subagent inherits these too unless you override.

**`system_prompt`** — Your strategy prompt for the lead. Gets concatenated with the built-in `BASE_AGENT_PROMPT` that teaches the lead how to use `write_todos`, the filesystem, and `task`. **You don't need to re-explain those tools** — just describe your domain strategy.

**`middleware`** — A list of `AgentMiddleware` instances appended *after* the default stack. The default stack already includes: `PlanningMiddleware` (todos), `FilesystemMiddleware`, `SubAgentMiddleware`, `SummarizationMiddleware`, and a "patch dangling tool calls" middleware. You add your own only when you want extra behavior — PII redaction, retry/fallback, custom logging, conversation summarization tuning, human-in-the-loop on specific tools, etc.

**`subagents`** — Specs for specialist subagents the lead can delegate to via the `task` tool. Each is either a dict (`{name, description, prompt, tools}`) or a pre-compiled graph.

**`skills`** — A list of *path prefixes* (e.g. `["/skills/"]`) inside the backend's filesystem that contain `SKILL.md` files. At startup, the `SkillsMiddleware` reads all `SKILL.md` files under those prefixes and injects their summaries into the system prompt. The agent learns "skill X exists, here's when to use it, full details at /skills/x/SKILL.md" and can `read_file` the full skill on demand. **This is exactly the Anthropic Skills pattern but living in the backend, not on disk.**

**`memory`** — A list of path prefixes (e.g. `["/memories/"]`) inside the backend that the `MemoryMiddleware` watches. Files matching `AGENTS.md` under these prefixes are auto-loaded into the system prompt at the start of every run. Think of it as "always-on context" — facts about the user, the org, conventions, that you want the agent to know without having to be told each time.

**`response_format`** — A Pydantic schema (or `ResponseFormat`) for structured final outputs. If set, the agent emits a final structured object instead of (or alongside) the message.

**`context_schema`** — A typed schema for the `config["configurable"]` dict. Lets you pass per-invocation context (user_id, tenant_id, locale) in a type-safe way that middleware and tools can read.

**`checkpointer`** — A LangGraph checkpointer that persists *conversation state* (messages, todos, in-state files) between invocations on the same `thread_id`. `InMemorySaver` (RAM, lost on restart), `SqliteSaver`, `PostgresSaver`, `RedisSaver`. Required for resuming conversations and for human-in-the-loop interrupts.

**`store`** — A LangGraph `BaseStore` for *cross-thread* persistence (key-value with namespaces). This is where long-term memory lives — facts you want to remember across different conversations and different users. `InMemoryStore` (RAM), or persistent stores like `PostgresStore`. **Also doubles as the storage layer for `StoreBackend`** — see backends below.

**`backend`** — Where the virtual filesystem (`ls`, `read_file`, `write_file`, `edit_file`, `glob`, `grep`) actually stores data. This is the parameter that answers your "no shell, no filesystem" question. Options below.

**`interrupt_on`** — Map of tool name → approval config. Pauses the graph before executing those tools so a human can approve/edit/reject. Requires a checkpointer.

**`debug`** — Verbose graph logging.

**`name`** — Graph name (shows up in LangSmith traces).

**`cache`** — A `BaseCache` for caching LLM responses. With identical prompts + tools, returns the cached completion instead of calling the model again. Saves money and latency on repeated identical calls.

## Backends — answering your "no shell, no filesystem" question directly

The backend is the storage layer for the agent's virtual filesystem tools. There are four built-ins:

| Backend | Storage location | Persistence | Shell access |
|---|---|---|---|
| `StateBackend` (default) | LangGraph state (RAM, attached to the thread) | Only within the thread; lost when state is cleared. Survives restarts only if you also have a persistent checkpointer. | No |
| `StoreBackend` | A LangGraph `BaseStore` (`InMemoryStore`, `PostgresStore`, etc.) | Cross-thread; persistent if the store is persistent | No |
| `FilesystemBackend` | Real disk at a configured root dir | Persistent (real files) | No |
| `LocalShellBackend` | Real disk + can run shell commands | Persistent + dangerous | **Yes — avoid** |
| `CompositeBackend` | Routes by path prefix to other backends | Mix and match | Depends on children |

**Yes, you can absolutely use memory as your backend.** The default `StateBackend` is exactly that — files live in the agent's in-memory state. No disk, no shell, nothing leaves the pod. This is the right starting point for your Kubernetes deployment.

The subtle point: **`StateBackend` is ephemeral** within a single graph invocation unless you add a checkpointer. If you want files to survive across `.invoke()` calls within the same conversation (same `thread_id`), pair `StateBackend` with `InMemorySaver` (RAM, dies on pod restart) or a persistent checkpointer (Postgres/Redis, survives pod restart). For *cross-conversation* persistence with no real disk, use `StoreBackend` + `InMemoryStore` (RAM) or a persistent store backend.

When you eventually get NAS access, you can switch to `FilesystemBackend(rootDir="/mnt/nas/agent-workspace")` with **zero code changes** elsewhere — the agent's tool calls (`read_file`, `write_file`) stay identical. That's the whole point of the backend abstraction.

## How `skills`, `memory`, `middleware`, `cache`, `store` actually work — with a stepwise upgrade path

I'll lay this out as the order you should adopt them, because trying to learn all of them at once is the wrong move.

### Step 0 — Where you are now (Pattern A from our last conversation)

```python
agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[],
    system_prompt=lead_strategy_prompt,
    subagents=[support_subagent],
)
```

This already gives you planning, the in-state filesystem (default `StateBackend`), and subagent delegation. No persistence — every `.invoke()` is a fresh start.

### Step 1 — Add a checkpointer so conversations persist within a thread

This is the smallest, highest-value upgrade. Without it, your agent forgets everything between user turns.

```python
from langgraph.checkpoint.memory import InMemorySaver

agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[],
    system_prompt=lead_strategy_prompt,
    subagents=[support_subagent],
    checkpointer=InMemorySaver(),   # RAM only — dies on pod restart
)

# Now this works across calls:
config = {"configurable": {"thread_id": "user-123-session-456"}}
agent.invoke({"messages": [("user", "Hi")]}, config)
agent.invoke({"messages": [("user", "What did I just say?")]}, config)  # remembers
```

**Kubernetes implication**: `InMemorySaver` lives inside the pod. If your pod restarts mid-conversation, history is gone. For corporate deployments you usually swap this for `PostgresSaver` or `RedisSaver` once you have a database available. Both are drop-in replacements — one line change. Until then, `InMemorySaver` is fine for stateless request/response patterns and dev work.

### Step 2 — Add a `store` for cross-thread/cross-user memory

The checkpointer remembers *one conversation*. The store remembers *facts across all conversations*. Use it when the agent should recall things from previous sessions ("last time the user said they prefer email over Slack").

```python
from langgraph.store.memory import InMemoryStore

agent = create_deep_agent(
    ...,
    checkpointer=InMemorySaver(),
    store=InMemoryStore(),
)
```

The store is also where the `StoreBackend` puts its files if you switch backends later. Same restart caveat: `InMemoryStore` is RAM. Postgres-backed store is the production upgrade.

### Step 3 — Add `memory=[...]` for always-on context (AGENTS.md)

Once you have a backend that holds files (state, store, or filesystem), you can drop `AGENTS.md` files into specific paths and the `MemoryMiddleware` will auto-load them into every run's system prompt. Think of these as "things the agent should always know."

```python
from deepagents import create_deep_agent
from deepagents.backends.utils import create_file_data

# Seed an AGENTS.md into the in-state filesystem at startup
seed_files = {
    "/memories/AGENTS.md": create_file_data(
        "Company conventions:\n"
        "- Always address customers as Mr./Ms. unless they specify otherwise.\n"
        "- Order IDs follow the pattern A### (e.g., A123).\n"
        "- Escalate refunds over $500 to a human."
    )
}

agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),
    tools=[...],
    system_prompt=lead_strategy_prompt,
    subagents=[support_subagent],
    checkpointer=InMemorySaver(),
    memory=["/memories/"],   # MemoryMiddleware watches this prefix
)

# When you invoke, seed the state's filesystem with the AGENTS.md
agent.invoke(
    {"messages": [("user", "Help with order A124")], "files": seed_files},
    {"configurable": {"thread_id": "t1"}},
)
```

The benefit over just stuffing this into `system_prompt`: the agent can also `edit_file("/memories/AGENTS.md", ...)` to update its own guidance, and the memory is *file-shaped* so it composes with skills.

### Step 4 — Add `skills=[...]` for on-demand expertise

Skills are just like memory, but **lazy-loaded**. Instead of injecting the full content into the prompt, the `SkillsMiddleware` injects only a *summary* of each skill (the first lines of its `SKILL.md`), and the agent calls `read_file` to get the full instructions when it actually needs them. This is how you give the agent 50 specialized procedures without burning 50,000 tokens of prompt.

```python
seed_files = {
    "/skills/refund_policy/SKILL.md": create_file_data(
        "# Refund Policy Skill\n"
        "Use this skill when a customer requests a refund.\n"
        "## Steps\n"
        "1. Verify order in last 30 days...\n"
        "2. Check refund eligibility...\n"
        "..."
    ),
    "/skills/escalation/SKILL.md": create_file_data(
        "# Escalation Skill\n"
        "Use this skill when issue exceeds your authority.\n"
        "..."
    ),
}

agent = create_deep_agent(
    ...,
    memory=["/memories/"],
    skills=["/skills/"],
)
```

For your situation: skills are the cleanest way to migrate your big system prompt. Take each "procedure" your existing agent knows and turn it into a `SKILL.md` file. The lead's strategy prompt stays small ("when the user asks about X, look in skills"); the procedural detail lives in skills that are read on demand. **This is exactly how Claude Code's skill system works**, and it's the deepagents team's recommended pattern for complex agents.

### Step 5 — Add `cache` to avoid paying for repeated identical LLM calls

```python
from langchain_core.caches import InMemoryCache

agent = create_deep_agent(
    ...,
    cache=InMemoryCache(),
)
```

If your agent receives the same exact subtask twice (common for the lead → subagent path when retrying), the second call returns instantly from cache. RAM-only — for distributed pods you'd use a Redis-backed cache later. Low risk to add, immediate cost savings.

### Step 6 — Add `middleware=[...]` for cross-cutting behavior

This is where you add things that should run on *every* model call regardless of which agent: PII scrubbing on outputs, custom logging, prompt-injection detection, retry/fallback to a cheaper model, conversation summarization tuning. LangChain ships several prebuilt middlewares; you can also write your own.

```python
from langchain.agents.middleware import PIIMiddleware  # example
from langchain.agents.middleware.summarization import SummarizationMiddleware

agent = create_deep_agent(
    ...,
    middleware=[
        PIIMiddleware(redact_outputs=True),
        SummarizationMiddleware(token_threshold=8000),  # tune compaction
    ],
)
```

Don't add middleware until you have a concrete need — the defaults already include the important ones.

### Step 7 (later, only if needed) — `interrupt_on` for human approval

When some tool calls are dangerous (e.g., `send_email`, `delete_record`), make them require human approval:

```python
agent = create_deep_agent(
    ...,
    checkpointer=InMemorySaver(),  # required for HITL
    interrupt_on={"send_email": True, "delete_record": True},
)
```

The graph pauses before executing those tools and surfaces the call to whatever UI is consuming the stream. The user approves/edits/rejects, and the graph resumes.

## Putting it together — the recommended Kubernetes-friendly starting point for you

Given your constraints (no shell, no real filesystem yet, pod restarts wipe RAM, may eventually get NAS), here is the configuration I'd actually deploy:

```python
from deepagents import create_deep_agent
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore
from langchain_openai import ChatOpenAI
from langchain_core.caches import InMemoryCache

# Your existing tools and existing-React-agent-as-subagent from before
from my_app.tools import lookup_account, get_orders, search_kb
from my_app.subagents import support_subagent

agent = create_deep_agent(
    model=ChatOpenAI(model="gpt-4o"),

    # Lead has no direct tools — pure planner/orchestrator
    tools=[],

    # Strategy only — execution detail lives in the subagent and in skills
    system_prompt=(
        "You are a support triage lead. For each user issue, call write_todos "
        "to plan the investigation, delegate steps to the support_agent subagent "
        "via the task tool, write findings to files under /workspace/, and "
        "synthesize a final response. Consult /skills/ for specialized procedures."
    ),

    subagents=[support_subagent],

    # In-memory storage — no shell, no disk
    # backend defaults to StateBackend (the in-state virtual filesystem)
    checkpointer=InMemorySaver(),  # per-thread conversation persistence
    store=InMemoryStore(),         # cross-thread (placeholder for future)
    cache=InMemoryCache(),         # cheap latency win

    # Always-loaded company conventions and on-demand procedures
    memory=["/memories/"],
    skills=["/skills/"],

    name="support-triage-lead",
)
```

**What happens on pod restart**: any in-flight conversation state is lost (because `InMemorySaver` and `InMemoryStore` are RAM). The agent itself is rebuilt fresh, including re-reading any seed files you inject. This is acceptable for a stateless request/response service — you'd typically have an external store of conversation history (your own DB) that you replay into the agent as the `messages` list, rather than relying on the checkpointer for durability.

**What changes when you get NAS access**: swap `StateBackend` for `FilesystemBackend(root_dir="/mnt/nas/agent-fs")` via the `backend=` parameter. Skills and memories now live as real `.md` files on the NAS that ops can edit without redeploying. Everything else stays the same. Note your caveat — "as server restart it will go away" — that wouldn't apply to NAS itself (NAS survives pod restarts), only to the in-pod RAM. NAS is exactly the right home for skills and memories long-term.

**What changes when you get a database**: swap `InMemorySaver` for `PostgresSaver` (or Redis), and `InMemoryStore` for `PostgresStore`. Now conversations and cross-thread memory survive restarts and scale across multiple pods.

## The mental model for these knobs

- **`backend`** = the agent's filing cabinet. State (RAM in this pod), Store (RAM or DB across pods), Filesystem (real disk/NAS).
- **`checkpointer`** = save button for *one* conversation. Without it, the agent has goldfish memory between turns.
- **`store`** = save button for *all* conversations. Long-term, cross-user memory.
- **`memory=[...]`** = files the agent always reads at the start (always-on context).
- **`skills=[...]`** = files the agent knows *exist* and reads only when needed (lazy expertise).
- **`middleware`** = layers that wrap every model call (logging, PII, retries).
- **`cache`** = "I've answered this exact prompt before; here's the answer."
- **`interrupt_on`** = "stop and ask a human before doing this."

## Easy real-world analogy

Imagine your existing agent is a contractor with a thick binder of procedures and a memory like a goldfish. You're putting a project manager on top.

- The **manager's strategy prompt** is the manager's job description.
- The **backend** is the office filing cabinet (RAM = whiteboard, NAS = real cabinet).
- The **checkpointer** is the manager's day planner — without it, every meeting starts from scratch.
- The **store** is the company CRM — the manager remembers facts about clients across years.
- **Memory files** are the company handbook posted on the office wall — read every morning.
- **Skill files** are the binders on the shelf — pulled down only when a specific situation arises.
- **Middleware** are office policies (every email gets PII-scrubbed, every call gets logged).
- **Cache** is "I already answered this exact question yesterday, here's my saved response."
- **interrupt_on** is the rule "manager must get my signature before sending invoices over $10k."

You don't install all of this on day one. You hire the manager (Step 0), give them a day planner (Step 1), then pin the company conventions to the wall (Step 3), then move the binders onto the shelf (Step 4), then add cost-saving and safety policies as needs emerge.

## Concrete suggestion for your *next* step

You already have Pattern A working (Option 2 from our last chat). The single most valuable next change is **Step 1 + Step 4**: add `InMemorySaver` and start moving chunks of your existing big system prompt into `SKILL.md` files under `/skills/`. That gives you (a) conversation continuity within a thread and (b) a clean migration path for the prompt-bloat problem, both with zero infrastructure dependencies. Steps 2, 3, 5, 6, 7 can wait until you actually feel the pain they solve.

Confidence: ~98% on the parameter signature and meanings (verified directly against the LangChain reference docs and the current `graph.py` source). ~95% on the backend behaviors and the skills/memory mechanics (verified against the official customization docs). ~90% on the exact pattern for seeding files into `StateBackend` via the `"files"` key in the input — that field name has been stable but worth double-checking against your installed version with a tiny smoke test before relying on it in production.