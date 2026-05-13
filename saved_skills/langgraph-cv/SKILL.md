# /langgraph-cv

Instrument a custom LangGraph agent so its message state can be loaded into the Context Visualizer at `http://localhost:5173`.

## Background

The Context Visualizer reads a `state.json` file that describes the full message history of a LangGraph run — every system prompt, human turn, AI reasoning step, tool call, and tool result — along with the model name and context window size. This skill adds that export capability to any existing LangGraph agent by:

1. Dropping a `cv_export.py` helper into the agent's directory
2. Adding two lines to the agent file

## What to do

### Step 1 — Find the agent file

If the user has named a file, use that. Otherwise look for Python files in the current directory that contain `from langgraph` or `import langgraph`. Ask the user to confirm which file to instrument.

### Step 2 — Read the agent file

Read the full file. Identify:
- The **model name** — look for `ChatOllama(model=...)`, `ChatOpenAI(model=...)`, `ChatAnthropic(model=...)`, or a `MODEL = "..."` constant.
- The **agent variable** — what `create_react_agent(...)` is assigned to.
- The **invoke call** — `result = agent.invoke(...)` or equivalent.
- The **messages variable** — typically `result["messages"]`.
- Any **tool list** — functions decorated with `@tool` or passed to `create_react_agent`.
- The **directory** of the agent file — `cv_export.py` goes there.

### Step 3 — Create cv_export.py

Write the following file to the **same directory as the agent file**. Do not modify it — copy it exactly:

```python
"""cv_export.py — Context Visualizer export helper for LangGraph agents.

Drop this file next to any LangGraph agent, then:

    from cv_export import export_state

    result = agent.invoke({...})
    export_state(result["messages"], model="qwen2.5", output="state.json")

Load the written state.json in the Context Visualizer at http://localhost:5173
(Agent Context section → paste the absolute path → Load).
"""

import json
from datetime import datetime
from pathlib import Path

CONTEXT_WINDOWS: dict[str, int] = {
    "qwen2.5": 32_768,
    "qwen2.5-coder": 32_768,
    "llama3.2": 131_072,
    "llama3.1": 131_072,
    "llama3": 8_192,
    "mistral": 32_768,
    "mistral-nemo": 131_072,
    "phi3": 131_072,
    "phi3.5": 131_072,
    "gemma2": 8_192,
    "gemma3": 131_072,
    "deepseek-r1": 65_536,
    "command-r": 131_072,
    "gpt-4o": 128_000,
    "gpt-4o-mini": 128_000,
    "gpt-4-turbo": 128_000,
    "gpt-3.5-turbo": 16_385,
    "claude-opus-4": 200_000,
    "claude-sonnet-4": 200_000,
    "claude-haiku-4": 200_000,
    "claude-3-5-sonnet": 200_000,
    "claude-3-5-haiku": 200_000,
    "claude-3-opus": 200_000,
}


def serialize_message(msg) -> dict:
    if isinstance(msg.content, str):
        content = msg.content
    elif isinstance(msg.content, list):
        content = " ".join(
            b.get("text", "") if isinstance(b, dict) else str(b)
            for b in msg.content
        )
    else:
        content = str(msg.content)

    entry: dict = {"role": msg.type, "content": content}

    if hasattr(msg, "tool_calls") and msg.tool_calls:
        entry["tool_calls"] = [
            {"name": tc["name"], "args": tc["args"]}
            for tc in msg.tool_calls
        ]

    if hasattr(msg, "name") and msg.name:
        entry["tool_name"] = msg.name

    return entry


def resolve_context_window(model: str, override: int | None = None) -> int:
    if override:
        return override
    bare = model.split(":")[0].lower()
    return CONTEXT_WINDOWS.get(bare, 128_000)


def stream_and_export(
    agent,
    inputs: dict,
    model: str,
    output: str | Path = "state.json",
    context_window: int | None = None,
    tools: list[str] | None = None,
    extra: dict | None = None,
) -> dict:
    """Run agent with streaming and always write state.json, even on crash."""
    messages = []
    try:
        for chunk in agent.stream(inputs, stream_mode="values"):
            if "messages" in chunk:
                messages = chunk["messages"]
    finally:
        if messages:
            export_state(messages, model=model, output=output,
                         context_window=context_window, tools=tools, extra=extra)
    return {"messages": messages}


def export_state(
    messages: list,
    model: str,
    output: str | Path = "state.json",
    context_window: int | None = None,
    tools: list[str] | None = None,
    extra: dict | None = None,
) -> Path:
    """Write a Context Visualizer–compatible state.json."""
    out = Path(output)
    state = {
        "model": model,
        "context_window_tokens": resolve_context_window(model, context_window),
        "timestamp": datetime.now().isoformat(),
        "messages": [serialize_message(m) for m in messages],
    }
    if tools:
        state["tools"] = tools
    if extra:
        state.update(extra)

    out.write_text(json.dumps(state, indent=2))
    print(f"Context Visualizer state → {out.resolve()}")
    return out
```

### Step 4 — Instrument the agent file

Make two edits to the agent file:

**a) Add the import** near the top (after existing imports):
```python
from cv_export import stream_and_export
```

**b) Replace `agent.invoke(...)` with `stream_and_export(...)`**, which captures messages
incrementally and always writes state.json — even if the run crashes mid-way:
```python
result = stream_and_export(
    agent,
    {"messages": [...]},         # same inputs dict that was passed to agent.invoke()
    model=MODEL,                 # adjust to match the actual model variable/string
    output="state.json",
    tools=[t.name for t in tools_list],   # optional — omit if tools not in scope
)
messages = result["messages"]   # same shape as result["messages"] from invoke()
```

Remove the original `agent.invoke(...)` call and any separate `export_state(...)` call.

Use the actual variable names from the file — do not invent new ones.

If the agent file already has its own `serialize_message` function, remove it and use the one from `cv_export` instead.

### Step 5 — Handle output file naming

- If the agent already writes a `state.json` (e.g. it was previously instrumented), keep the same filename.
- If there are multiple agents in the same directory, use distinct names like `state_research.json`, `state_chat.json`, etc. and tell the user.

### Step 6 — Print a summary

```
Instrumented: <agent_file>
Helper:       <directory>/cv_export.py
State file:   <directory>/state.json   (written after each run)

To visualize:
  1. Run your agent normally
  2. Open http://localhost:5173
  3. Scroll to "Agent Context (LangGraph + Ollama)"
  4. Paste: <absolute path to state.json>
  5. Click Load
```

## Rules

- Never modify the agent's logic — only add the import and the export call.
- Place `cv_export.py` in the same directory as the agent file, not the project root.
- If `cv_export.py` already exists there, skip Step 3 (do not overwrite it).
- If the model name is a variable (e.g. `MODEL = os.getenv(...)`), pass the variable, not a hardcoded string.
- If you cannot determine the messages variable name with confidence, ask the user before editing.
