# HedgehogMemory

> **Radial memory architecture for AI agents — never lose context, never delete history.**

[![PyPI version](https://badge.fury.io/py/hedgehog-memory.svg)](https://pypi.org/project/hedgehog-memory/)
[![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## The Problem

Every AI agent eventually hits the context window limit. When this happens, you must either:
- **Truncate history** — the agent forgets what happened
- **Summarize everything** — you lose precision
- **Reload all context** — burns tokens, still lossy

HedgehogMemory solves this by storing memory in a **radial structure** — one permanent origin at the center, task-domain lines radiating outward, each carrying compressed nodes ordered by recency.

HedgehogMemory proposes a fourth approach: never delete, only compress; the origin is always in the context. Most importantly, it allows loading to the corresponding time task state point at any time for continued development. When a task is completed, a node is moved along a line to add a new node or open a new line record, then the origin is returned. When loading a point, the context is cleared to enter the corresponding line or several points, maintaining alignment between different modules. Even if a node is loaded incorrectly or offset, it can still be drilled down because detailed summaries of the node are loaded, while clearing that part of the context, retaining only the origin architecture rules and necessary information, always providing maximum context space where it is most needed.

---

## Architecture

```
                         [Origin]  <-- always in context (~200 tokens)
                        /    |    \
                [auth]  [rec]  [infra]   <-- task domain lines
                  |       |       |
                pos=1    pos=1   pos=1   <-- newest node (most detail)
                pos=2    ...     pos=2   <-- older node (compressed)
                pos=3            ...     <-- oldest (most compressed)
```

**Key properties:**
- **Origin is always in context** — L0 summaries for all lines, ~200 tokens total
- **Nodes are never deleted** — only pushed further from origin (position+1)
- **Abstraction increases with distance** — L0 (80 chars) → L1 (200) → L2 (600) → L3 (1800) → L4 (full)
- **100% restoration** — `load_full_state()` returns the original `full_context` string verbatim
- **Permanent storage** — everything written to `origin.json`, survives agent restarts

---

## WARNING: LLM Summarizer Required for Full Capability

> **The default `KeywordSummarizer` uses keyword matching only.**
> It works for demos and offline use, but has **limited cross-domain search accuracy**.
> 
> **For production AI agents, you MUST plug in an LLM summarizer** to get accurate
> navigation and high-quality multi-level summaries. See [Summarizer Integration](#summarizer-integration).

---

## Installation

```bash
pip install hedgehog-memory
```

Or from source:
```bash
git clone https://github.com/vvxer/HedgehogMemory.git
cd HedgehogMemory
pip install -e .
```

---

## Quick Start

```python
from radial_memory import RadialMemory, ContextWindowManager

# Initialize
mem = RadialMemory("./my_project_memory", project_name="My AI Project")

# Optional: define task lines upfront
mem.create_line("auth",  "Authentication Module", "JWT, login, session management")
mem.create_line("infra", "Infrastructure",        "Docker, CI/CD, deployment")

# --- After completing a task ---
node = mem.complete_task(
    line_id="auth",
    task_label="Implement JWT login endpoint",
    full_context="""
        Implemented POST /api/auth/login.
        Accepts {username, password}, returns {access_token, refresh_token}.
        access_token TTL=15min, refresh_token TTL=7d. Using PyJWT.
    """,
    conversation_summary="Discussed JWT vs Session tradeoffs, chose JWT stateless approach.",
    todos=["Add rate limiting", "Write unit tests"],
    env_snapshot={"python": "3.11", "django": "4.2"},
)
print(f"Saved node at position {node.position}")
```

---

## Agent Integration Pattern

The typical agent loop with HedgehogMemory:

```python
from radial_memory import ContextWindowManager, RadialMemory

mem = RadialMemory("./memory", project_name="My Project")
cw  = ContextWindowManager(mem)

# === At the start of every agent turn ===
# 1. If context window is getting full, reset to origin
system_prompt = cw.reset()     # ~200 tokens, always safe

# 2. User asks about something specific
user_query = "How did we implement the login endpoint?"

# 3. Navigate from file (no tokens spent until needed)
result = cw.load(user_query, line_id="auth")

if result.status == "found":
    # L1 brief — let LLM verify it's the right node
    print(result.context)

    if result.can_drill_deeper:
        result_l2 = result.drill_deeper()   # more detail
        result_l3 = result_l2.drill_deeper() # even more

    # 100% restore the original working state
    full_state = result.load_full_state()

# 4. When a task completes, persist it
cw.commit(
    line_id="auth",
    task_label="Add JWT refresh endpoint",
    full_context="... complete working context ...",
    conversation_summary="Implemented /refresh endpoint with token rotation.",
    todos=["Deploy to staging"],
)
```

---

## Abstraction Levels

| Level | Name     | Max Length | Use Case                        |
|-------|----------|------------|---------------------------------|
| L0    | origin   | 80 chars   | Always in context (origin view) |
| L1    | brief    | 200 chars  | Quick scan / verify match       |
| L2    | summary  | 600 chars  | Understand what was done        |
| L3    | detailed | 1800 chars | Full technical context          |
| L4    | full     | unlimited  | 100% restore original state     |

Navigation is **progressive**: start at L1, drill deeper only if needed. This minimizes token usage.

---

## Summarizer Integration

### Default (offline, no dependencies)

```python
from radial_memory import RadialMemory
from radial_memory.summarizer import KeywordSummarizer

mem = RadialMemory("./memory", summarizer=KeywordSummarizer())
```

**Limitations:** Uses keyword overlap for matching. Struggles with cross-domain queries and semantic similarity.

### OpenAI (recommended for production)

```python
from radial_memory.summarizer import OpenAISummarizer

summarizer = OpenAISummarizer(
    api_key="sk-...",
    model="gpt-4o-mini",      # cost-effective, fast
    # model="gpt-4o",         # highest quality
)
mem = RadialMemory("./memory", summarizer=summarizer)
```

### LiteLLM (Claude, Gemini, local models, etc.)

```python
from radial_memory.summarizer import LiteLLMSummarizer

# Any provider LiteLLM supports
summarizer = LiteLLMSummarizer(
    model="claude-3-5-haiku-20241022",
    api_key="...",
)
# Or local model via Ollama
summarizer = LiteLLMSummarizer(model="ollama/llama3.2")

mem = RadialMemory("./memory", summarizer=summarizer)
```

### Custom Summarizer

```python
from radial_memory.summarizer import BaseSummarizer

class MySummarizer(BaseSummarizer):
    def summarize(self, text: str, max_chars: int) -> str:
        # Your implementation
        ...

    def verify_match(self, query: str, summary: str) -> tuple[bool, float]:
        # Return (is_match, confidence_0_to_1)
        ...

mem = RadialMemory("./memory", summarizer=MySummarizer())
```

---

## API Reference

### `RadialMemory`

```python
mem = RadialMemory(base_path, project_name, summarizer=None)

mem.create_line(line_id, display_name, description)
mem.complete_task(line_id, task_label, full_context, conversation_summary,
                  file_changes=None, todos=None, env_snapshot=None) -> Node
mem.navigate(query, line_id=None, initial_level=1, confidence_threshold=0.5) -> NavigationSession | None
mem.get_origin_context() -> str          # ~200 token overview
mem.list_lines() -> list[dict]
mem.get_line_history(line_id) -> list[dict]
```

### `ContextWindowManager`

```python
cw = ContextWindowManager(mem)

cw.reset() -> str                        # origin-only context string
cw.load(query, line_id=None, ...) -> LoadResult
cw.load_node(line_id, position, level=1) -> LoadResult
cw.commit(line_id, task_label, full_context, ...) -> Node
cw.status_report() -> str
```

### `LoadResult`

```python
result.status          # "found" | "not_found"
result.context         # formatted context string for LLM
result.confidence      # 0.0 - 1.0
result.current_level   # 1-4
result.can_drill_deeper # bool
result.node            # Node object

result.drill_deeper() -> LoadResult       # go to next level
result.load_full_state() -> str           # L4: original full_context
```

---

## Storage Format

All memory is stored in a single `origin.json` file:

```json
{
  "project_name": "My Project",
  "lines": {
    "auth": {
      "display_name": "Authentication Module",
      "nodes": [
        {
          "id": "uuid",
          "position": 1,
          "task_label": "Implement JWT login",
          "summaries": {"0": "...", "1": "...", "2": "...", "3": "..."},
          "state": { "full_context": "... original verbatim context ..." },
          "created_at": "2026-01-01T10:00:00"
        }
      ]
    }
  }
}
```

The file grows over time but is never rewritten destructively. All history is preserved.

---

## Run the Demo

```bash
git clone https://github.com/vvxer/HedgehogMemory.git
cd HedgehogMemory
pip install -e .
python -m examples.demo
```

The demo runs 4 scenarios:
1. Single line full lifecycle (auth module, 3 tasks)
2. Multi-line overview (auth + recommendation engine + infra)
3. Step navigation (agent manually walks candidate nodes)
4. ContextWindowManager full loop (reset → load → drill → restore → commit)

---

## Design Principles

1. **Never delete** — old context is compressed, not removed
2. **Origin is cheap** — ~200 tokens, always safe to include in system prompt
3. **Progressive disclosure** — L1 → L2 → L3 → L4 on demand
4. **LLM-agnostic** — plug in any summarizer, works offline with defaults
5. **Crash-safe** — every `commit()` is an atomic JSON write
6. **Zero mandatory dependencies** — stdlib only for default mode

---

## Roadmap

- [ ] Multi-file sharding for very large memory stores
- [ ] Embedding-based navigation (vector search)
- [ ] MCP server wrapper (use as a tool in any MCP-compatible agent)
- [ ] Web UI for browsing origin.json visually
- [ ] Async API

---

## License

MIT License — see [LICENSE](LICENSE)

---

## Citation

If you use HedgehogMemory in your research or product, please cite:

```
@software{hedgehogmemory2026,
  title  = {HedgehogMemory: Radial Memory Architecture for AI Agents},
  author = {vvxer},
  year   = {2026},
  url    = {https://github.com/vvxer/HedgehogMemory}
}
```
