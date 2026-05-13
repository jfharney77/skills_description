---
description: Add Cerebras AI API support to any Python project using LangChain or LangGraph. Cerebras is OpenAI API-compatible so langchain-openai's ChatOpenAI works with a custom base_url. Default model is gpt-oss-120b (128K context).
---

## Step 1 — Find the agent directory

Search for Python files containing `ChatOllama`, `ChatOpenAI`, `ChatAnthropic`, or `create_react_agent`. This is the directory where `llm.py` will be placed and where `.env` changes will be focused.

If there are multiple agent files, note them all — each one may need updating.

## Step 2 — Update .env

Find the `.env` file closest to the agent files (check agent dir, then project root). Add the following lines — **do not overwrite** existing values:

```
LLM_PROVIDER=ollama          # ollama | cerebras  (default keeps existing behaviour)
CEREBRAS_API_KEY=            # user fills this in
CEREBRAS_MODEL=gpt-oss-120b
```

If `LLM_PROVIDER` already exists, skip it. If `CEREBRAS_API_KEY` already has a value, skip it.

## Step 3 — Update .env.example

If a `.env.example` exists, add the same lines with placeholder comments:

```
LLM_PROVIDER=ollama   # ollama | cerebras

# Cerebras (set LLM_PROVIDER=cerebras to use)
# Get your API key at https://cloud.cerebras.ai
CEREBRAS_API_KEY=csk-...
CEREBRAS_MODEL=gpt-oss-120b
```

## Step 4 — Add langchain-openai to requirements

If a `requirements.txt` exists in the agent directory (or project root), add `langchain-openai>=0.1.0` if it is not already present.

Then install it. If the project uses a venv, activate it first or use the venv's pip directly. Look for a `venv/` directory next to `requirements.txt` and use `venv/bin/pip install langchain-openai` if found. Otherwise:

```bash
pip install langchain-openai
```

## Step 5 — Create or update llm.py

Check whether `llm.py` already exists in the agent directory.

**If it already has Cerebras support** (contains `CEREBRAS_API_KEY` or `api.cerebras.ai`), skip this step.

**If `llm.py` exists and handles multiple providers**, read it and add Cerebras as an additional `elif` branch — do not replace the file.

**Otherwise**, write the following file to the agent directory:

```python
"""llm.py — Build the LLM for the current provider (Ollama or Cerebras).

Set LLM_PROVIDER=ollama (default) or LLM_PROVIDER=cerebras in .env.
"""

import os
from pathlib import Path

from dotenv import load_dotenv

load_dotenv(Path(__file__).parent / ".env")

PROVIDER = os.getenv("LLM_PROVIDER", "ollama").lower()

if PROVIDER == "cerebras":
    MODEL = os.getenv("CEREBRAS_MODEL", "gpt-oss-120b")
else:
    MODEL = os.getenv("OLLAMA_MODEL", "qwen2.5")

OLLAMA_BASE_URL = os.getenv("OLLAMA_BASE_URL", "http://localhost:11434")
CEREBRAS_API_KEY = os.getenv("CEREBRAS_API_KEY", "")
CEREBRAS_BASE_URL = "https://api.cerebras.ai/v1"


def build_llm():
    """Return a LangChain chat model for the configured provider."""
    if PROVIDER == "cerebras":
        from langchain_openai import ChatOpenAI
        if not CEREBRAS_API_KEY:
            raise ValueError("CEREBRAS_API_KEY must be set in .env")
        return ChatOpenAI(
            model=MODEL,
            api_key=CEREBRAS_API_KEY,
            base_url=CEREBRAS_BASE_URL,
        )
    else:
        from langchain_ollama import ChatOllama
        return ChatOllama(model=MODEL, base_url=OLLAMA_BASE_URL)
```

## Step 6 — Update agent files

For each agent file that directly instantiates the LLM, make two changes:

**a) Replace the import** at the top:
```python
# Before (example — match whatever is actually there):
from langchain_ollama import ChatOllama

# After:
from llm import MODEL, PROVIDER, build_llm
```

Remove any `import os` lines that were only used for the old LLM env vars if they are no longer needed elsewhere. Keep `import os` if it is used for other things.

**b) Replace the LLM instantiation**:
```python
# Before (example — match whatever is actually there):
llm = ChatOllama(model=MODEL, base_url=OLLAMA_BASE_URL)

# After:
llm = build_llm()
```

Also update any print statements that show the model/provider so they reflect the new variables:
```python
# Before:
print(f"Model : {MODEL}")

# After:
print(f"Provider : {PROVIDER}")
print(f"Model    : {MODEL}")
```

Use the actual variable names from each file — do not invent new ones.

## Step 7 — Print summary

After all changes, print this summary to the user:

```
Cerebras setup complete
═══════════════════════════════════════════════
.env              — added LLM_PROVIDER, CEREBRAS_API_KEY, CEREBRAS_MODEL
.env.example      — updated with Cerebras placeholder
requirements.txt  — added langchain-openai
llm.py            — created build_llm() helper
<agent_file>      — updated to use build_llm()

To activate Cerebras:
  1. Open .env
  2. Set CEREBRAS_API_KEY=csk-<your-key>
  3. Set LLM_PROVIDER=cerebras
  4. Run your agent normally

To switch back to Ollama:
  Set LLM_PROVIDER=ollama  (or delete the line — ollama is the default)

Cerebras API key:  https://cloud.cerebras.ai
Default model:     gpt-oss-120b  (128K context window)
```

## Rules

- Never overwrite an existing `CEREBRAS_API_KEY` that already has a value.
- Never remove or replace existing Ollama/OpenAI/Anthropic configuration — this adds Cerebras support alongside what is already there.
- If `llm.py` already exists and already handles multiple providers, do not replace it — instead read it and add Cerebras as an additional `elif` branch.
- If the project does not use LangChain at all (no `langchain` imports found), tell the user and ask whether they still want the `.env` vars added.
- Always install `langchain-openai` using the project's own venv if one exists.
