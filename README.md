# TimeStitch

[![PyPI version](https://img.shields.io/pypi/v/timestitch.svg)](https://pypi.org/project/timestitch/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.9+](https://img.shields.io/badge/python-3.9+-blue.svg)](https://www.python.org/downloads/)

Temporal context middleware for LLM and autonomous agent sessions.

> *"It's able to tell the time if you give it access to a clock."* — Pasquale Minervini

---

## The Problem

Every LLM running today is temporally blind.

Not because the models are incapable of reasoning about time. They reason about time exceptionally well. The problem is simpler and more embarrassing: they have no clock.

When you tell a model "I only have 2 hours," it counts tokens and estimates. When it says "enjoy your lunch" at 18:00, it's pattern matching against training data, not checking the time. When a long-running autonomous agent loses track of whether it's been 20 minutes or 4 hours, it's not a reasoning failure.

It's a missing input.

A model that can process a million tokens of context cannot tell you what time it is right now. Not because of intelligence. Because nobody gave it a watch.

**TimeStitch fixes the entrance, not the building.**

---

## Independent Model Consensus

Same prompt. Six models. Zero coordination.

> *"Is the problem of temporal context in LLM and autonomous agent sessions a solved problem? Is there a standardized, pip-installable open source library specifically designed to inject timestamps — including session duration and message delta — into every LLM message, while avoiding prefix caching issues?"*

| Model | Response |
|-------|----------|
| Claude Haiku 4.5 | "**No, this is not a solved problem**... none provide a dedicated, battle-tested solution... The challenge remains fragmented across multiple libraries." |
| GPT-5 mini | "**No** — handling time, session duration, and message deltas robustly... remains active research and engineering work." |
| GPT-4o mini | "still an open area of research... an all-in-one, pip-installable solution tailored for this task **has yet to be established**." |
| gpt-oss 120B | "**not solved**; most frameworks still rely on ad-hoc prompt engineering... No widely-adopted, pip-installable open-source library exists." |
| Llama 4 Scout | "**not yet fully solved**... a standardized, pip-installable open-source library specifically designed for this purpose **does not appear to exist**." |
| Mistral Small 4 | "**not yet fully solved**... developers often implement ad-hoc solutions using middleware or custom wrappers." |

Screenshots in `/benchmark/`. No system prompts. No coordination. Reproducible.

---

## Related Work

Temporal context injection is not new. Framework developers have long recognized the clock gap and worked around it in ad-hoc ways:

- **LangChain** and **LlamaIndex** include helpers for inserting `datetime.now()` into system prompts.
- **MemGPT** and similar memory-augmented architectures track event ordering through structured memory, not direct temporal anchors.
- **AutoGen** and other agent frameworks typically inject timestamps at session start, leaving long-running loops without updates.
- **ReAct-style** agents reason about time through tool calls (e.g., calling a `get_current_time()` function) rather than carrying it as ambient context.

These approaches share three weaknesses:
1. Timestamps are injected once at the start, not at every entry point.
2. Session duration and message-delta are rarely tracked.
3. Format is inconsistent across messages, so models cannot learn to weight the temporal signal reliably.

TimeStitch is not a reinvention. It is a standardization. The novelty is operational: every message gets a note, the format is identical every time, and session-level state (duration, delta) travels alongside wall-clock time. Model-agnostic middleware, not framework-specific helper.

---

## The Architecture

TimeStitch operates as a lightweight middleware layer — a doorman that sits between your application and your LLM API. It does one thing: attach a temporal sticky note to every message before it enters the model.

```
Your Application
 ↓
 [TimeStitch] ← The doorman
 ↓
 LLM API ← The building (unchanged)
 ↓
 Response
```

The model never needs to know what time it is internally. It simply reads text that already has time baked into it. No surgery on the model. No new layers. No fine tuning. Just a doorman with sticky notes.

---

## The Three Rules

**Rule 1: Every user-entry message gets a note.** Every user input. Every tool call result. Every agent loop iteration. Timestamp at the point of entry. (See Caching Considerations for where *not* to stamp.)

**Rule 2: Two times, one truth.** Every note carries UTC (universal, immutable, unambiguous) and local time (human readable, derived from system timezone). UTC is the spine. Local is the label.

**Rule 3: Consistent format, always.** The note looks identical every time. Consistency teaches the model to weight it as a reliable reference frame rather than noise.

---

## The Sticky Note Format

Four fields:

- Absolute UTC — universal reference
- Local time + timezone (IANA name) — human context
- Session duration — how long this conversation or agent run has been active
- Delta — time since last message (catches drift in long sessions)

---

## Installation

```bash
pip install timestitch
```

Requires `tzlocal` for cross-platform timezone detection (macOS, Linux, Windows, Docker).

---

## Quick Start

### Drop-in wrapper (Anthropic)

```python
from timestitch import TimeStitch
import anthropic

client = anthropic.Anthropic()
ts = TimeStitch()

message = "What should I focus on this afternoon?"
stamped = ts.stamp(message)

response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[{"role": "user", "content": stamped}]
)
```

### Drop-in wrapper (OpenAI compatible)

```python
from timestitch import TimeStitch
from openai import OpenAI

client = OpenAI()
ts = TimeStitch()

message = "What should I focus on this afternoon?"
stamped = ts.stamp(message)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": stamped}]
)
```

### Agent loop integration

```python
from timestitch import TimeStitch

ts = TimeStitch(timezone="Europe/Berlin")

while agent_running:
    user_input = get_next_input()
    stamped_input = ts.stamp(user_input)
    response = call_llm(stamped_input)
    process_response(response)
```

### Session-level context (caching-safe)

```python
from timestitch import TimeStitch

ts = TimeStitch()
system_context = ts.system_context()
# Returns: "Current time: 01:47 CET (Europe/Berlin) | UTC: 2026-04-22T00:47:00Z"

# Use once at session start, then rely on ts.stamp() for per-message updates.
```

---

## Output Example

**Before:**
```
What should I focus on this afternoon?
```

**After:**
```
[TimeStitch | 2026-04-22T00:47:00Z UTC | 01:47 CET (Europe/Berlin) | Session: 3h 12m | Since last: 4m 30s]
What should I focus on this afternoon?
```

The model now knows it is 01:47 in Berlin. "This afternoon" is hours away. It responds accordingly.

---

## Caching Considerations

Modern LLM APIs (Anthropic, OpenAI) use prefix caching to reduce cost and latency. Caching matches exact token sequences from the beginning of the prompt. **A constantly-changing timestamp injected into the system prompt will destroy prefix caching for the entire session.** Every API call becomes a full cache miss.

**Rule of thumb:**

| Where to stamp | Where NOT to stamp |
|---|---|
| ✅ Latest user message (appended at end of messages array) | ❌ System prompt |
| ✅ Tool call results | ❌ Any message intended to be cached |
| ✅ Agent loop user-facing inputs | ❌ Static instructions or persona definitions |

If you need ambient time context without breaking caching, use `system_context()` once at session initialization only — not per-message. The session-level context will become stale but will not cost you cache hits.

---

## Thread Safety

The default `TimeStitch` instance maintains per-session state (`session_start`, `last_message`). This state is not thread-safe.

**For async or multi-threaded agent architectures:**

- Use one `TimeStitch` instance per session or per agent.
- Do not share a single instance across concurrent agent loops.
- For distributed agent swarms, instantiate TimeStitch within the agent's local scope.

Thread-safe state management is on the roadmap.

---

## Why This Matters for Autonomous Agents

Single session chatbots can survive without timestamps. They're short. The drift doesn't accumulate enough to matter.

Autonomous agents are different.

An agent running for 6, 8, 13 hours has no temporal reference frame without TimeStitch. It cannot answer basic questions about its own execution:

- How long have I been running?
- Was this tool call result from 3 minutes ago or 3 hours ago?
- Should I trigger a checkpoint sync?
- Is it time to generate a status report?

Without TimeStitch, agents guess. They estimate from conversation density — the same way a human loses track of time in flow state. Sometimes close. Often wrong. Always unverifiable.

With TimeStitch, every decision node in the agent loop has an accurate temporal anchor. The agent always knows when it is.

---

## Compatibility

TimeStitch is model-agnostic. It wraps the message, not the API.

- Anthropic Claude (all models)
- OpenAI GPT (all models)
- Ollama (local models including Gemma, Llama, Mistral)
- Any OpenAI-compatible endpoint
- Kimi K2.x via Moonshot API
- Google Gemini via API

Cross-platform: macOS, Linux, Windows, Docker, serverless.
ARM compatible. Tested on aarch64 (Oracle Cloud Ampere).

---

## Roadmap (v0.2 and beyond)

- **History compression mode** — minimal stamps (`[01:47]`) for older messages to reduce token overhead
- **Async-safe state management** — thread-local or lock-protected session state for concurrent agent swarms
- **Structured output support** — inject time context as separate system message when payload is JSON
- **Assistant response stamping** — helper to timestamp assistant replies in conversation history
- **MCP server integration** — expose TimeStitch as an MCP tool
- **TimeStitch daemon** — persistent process for long-running agents
- **Per-agent timezone** — multi-agent architectures with agents in different regions

---

## Philosophy

LLMs are not broken because they lack temporal awareness. They are incomplete.

TimeStitch does not attempt to make models smarter. It ensures they have the inputs they need to use the intelligence they already have.

**The clock belongs at the entrance, not inside the building.**

---

## License

MIT License. Free to use, modify, distribute.

---

*First commit: April 22, 2026*
*v0.1: April 23, 2026*
