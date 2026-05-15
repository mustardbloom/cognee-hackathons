# Cognee × Redis Hackathon — 2026-05-16

🧠 **AI-Memory Hackathon: Building your own Agent LLM Wiki** 🚀

> "Instead of just retrieving from raw documents at query time, the LLM
> incrementally builds and maintains a persistent wiki." — Andrej Karpathy

Karpathy recently popularised the idea of an LLM Wiki. In this hackathon you
will extend that idea and build one together with us, using Cognee's
open-source memory engine and Redis.

Karpathy's note for reference:
https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f

## What You Build

An **LLM Knowledge Wiki** powered by Cognee memory. Your wiki must support
three base operations:

1. **Ingest** — pull raw documents, conversations, or runs into the wiki.
2. **Query + Self-improve** — answer questions, and use feedback from each
   query to grow and refine the wiki.
3. **Lint** — keep the wiki coherent: deduplicate, resolve conflicts, and
   prune stale entries.

The goal is not just retrieval — it is a wiki that gets smarter the more it
is used.

## Memory Architecture — Where Redis Fits

Your wiki runs on a **two-tier memory** model:

```text
                    [ agent / user ]
                          │
                          ▼
          ┌─────────────────────────────────┐
          │  Redis — session memory          │   fast, ephemeral
          │  (working scratchpad, recent     │   per-conversation
          │   conversations, raw events)     │
          └────────────────┬─────────────────┘
                           │  distillation
                           ▼
          ┌─────────────────────────────────┐
          │  Cognee — permanent memory       │   structured, durable
          │  (knowledge graph, embeddings,   │   cross-session
          │   skills, summaries)             │
          └─────────────────────────────────┘
```

- **Redis is the agent's session memory.** The agent loads recent data into
  Redis as it works: raw events, user turns, intermediate observations. This
  is the hot, fast scratchpad.
- **Cognee is the permanent memory.** Session content is distilled into the
  knowledge graph — entities, relationships, summaries, and skills — so it
  can be recalled across sessions and refined over time.
- **The self-improvement loop lives in this distillation step.** What gets
  promoted from Redis into the graph, how it's structured, and how feedback
  rewrites it, is the core of your wiki design.

In code, this maps directly onto cognee's `session_id` parameter:

```python
# Goes to Redis session memory — fast cache, syncs to graph in background
await cognee.remember("user just asked about retention", session_id="chat_1")

# Goes straight to the permanent knowledge graph
await cognee.remember("Retention is calculated as ...")

# Recall queries session memory first, falls through to the graph
await cognee.recall("what did the user ask?", session_id="chat_1")
```

**This Redis-as-session-memory pattern is the core piece of the hackathon.**
Judges will want to see how you use it: what your agent puts into session
memory, how it decides what to distill into the graph, and how distillation
quality improves run over run.

## Prizes — $1,500+ Cash Pool

| Place | Prize |
|-------|-------|
| 🥇 1st | $800 cash |
| 🥈 2nd | $500 cash |
| 🥉 3rd | $200 cash |

## Demo Format

You will have **3 minutes** to stand out:

- Present your idea and explain how you leverage agent self-improvement.
- Run a live demo that showcases your agent in action.

## Schedule

| Time | What |
|------|------|
| 12:00 PM | Doors open + networking |
| 12:30 PM | Opening remarks + partner demos |
| 1:00 PM | Hacking begins |
| 4:30 PM | Project submission deadline — finalists selected |
| 5:00 PM | Finalist presentations & judging |
| 5:30 PM | Awards ceremony |
| 6:00 PM | Event wrap-up & doors close |

## Setup

> **You do not need to bring API keys or accounts.** We provide the LLM API
> key (OpenAI) and any other event-specific credentials at kickoff. Everything
> below is so you know how to wire what we hand you into your project — not
> a list of things to sign up for in advance.

### Prerequisites

- Python 3.10 – 3.14
- Docker (for Redis)
- An LLM API key — **provided by us at kickoff** (you can also bring your own
  from any [supported provider](https://docs.cognee.ai/setup-configuration/llm-providers))

### 1. Install Cognee

```bash
uv venv && source .venv/bin/activate
uv pip install "cognee[redis]"
```

### 2. Configure the LLM

We hand out an `LLM_API_KEY` at kickoff — export it in your shell:

```bash
export LLM_API_KEY="<key-we-give-you-at-the-event>"
```

Or drop it into a local `.env` based on cognee's [`.env.template`](https://github.com/topoteretes/cognee/blob/main/.env.template).
Prefer your own provider? Set `LLM_PROVIDER` / `LLM_MODEL` per the
[provider docs](https://docs.cognee.ai/setup-configuration/llm-providers).

### 3. Start Redis (session memory)

Redis is the **session-memory layer** — the fast scratchpad your agent writes
into during a conversation, before content is distilled into the permanent
graph. Cognee picks it up automatically when `REDIS_URL` is set and any
`cognee.remember(..., session_id=...)` call routes there.

```bash
docker run -p 6379:6379 redis:latest
export REDIS_URL=redis://localhost:6379
```

### 4. Run the Pipeline

Cognee's API exposes four operations — `remember`, `recall`, `forget`, and
`improve`:

```python
import asyncio
import cognee


async def main():
    # Store permanently in the knowledge graph (runs add + cognify + improve)
    await cognee.remember("Cognee turns documents into AI memory.")

    # Store in session memory (fast cache, syncs to graph in background)
    await cognee.remember(
        "User prefers detailed explanations.",
        session_id="chat_1",
    )

    # Query with auto-routing (picks best search strategy automatically)
    results = await cognee.recall("What does Cognee do?")
    for result in results:
        print(result)

    # Query session memory first, fall through to graph if needed
    results = await cognee.recall(
        "What does the user prefer?",
        session_id="chat_1",
    )
    for result in results:
        print(result)

    # Delete when done
    await cognee.forget(dataset="main_dataset")


if __name__ == "__main__":
    asyncio.run(main())
```

### CLI

The same operations are exposed as a CLI:

```bash
cognee-cli remember "Cognee turns documents into AI memory."
cognee-cli recall "What does Cognee do?"
cognee-cli forget --all
```

Launch the local UI with `cognee-cli -ui` (web app at http://localhost:3000).

### Cognee Cloud (optional)

Skip local infra and point at a managed instance:

```python
import cognee

await cognee.serve(
    url="https://your-instance.cognee.ai",
    api_key="ck_...",
)
await cognee.remember("important context")
```

## Skills

A **skill** is a small Markdown file that tells the agent how to behave for a
specific task. Skills live in a folder, get ingested via `cognee.remember(...)`
with `content_type="skills"`, and can be selected at query time. Feedback from
each run is recorded as a `SkillRunEntry`, which lets the skill **self-improve**
over time — the core loop you are building for this hackathon.

### Folder layout

```text
my_skills/
  code-review/
    SKILL.md
```

`my_skills/code-review/SKILL.md`:

```markdown
---
description: Review code changes for bugs, regressions, and missing tests.
allowed-tools: memory_search
---

# Instructions

Read the diff carefully. Report concrete issues first, with file paths and
line references when available.
```

A starter version of this skill is included at
[`my_skills/code-review/SKILL.md`](./my_skills/code-review/SKILL.md) — copy it,
fork it, or replace it with your own.

### Run a skill from Python

```python
import asyncio
import cognee
from cognee import SearchType
from cognee.memory import SkillRunEntry


async def main():
    await cognee.remember(
        "./my_skills",
        dataset_name="project",
        content_type="skills",
    )

    answer = await cognee.search(
        "Review the current auth changes",
        query_type=SearchType.AGENTIC_COMPLETION,
        datasets="project",
        skills=["code-review"],
    )
    print(answer)

    proposal_result = await cognee.remember(
        SkillRunEntry(
            selected_skill_id="code-review",
            task_text="Review the current auth changes",
            result_summary="The review missed a dataset boundary bug.",
            success_score=0.2,
            feedback=-0.6,
            error_type="missed_regression",
        ),
        dataset_name="project",
        skill_improvement={
            "skill_name": "code-review",
            "apply": False,
        },
    )
    print(proposal_result.items)


asyncio.run(main())
```

`skill_improvement={"apply": False}` returns the proposed `SKILL.md` diff
without writing it. Flip to `True` once you trust the loop.

## Other Ways to Run Cognee

| What | How | When to use |
|------|-----|-------------|
| Python SDK | `import cognee` | Building your wiki / agent logic |
| CLI | `cognee-cli remember / recall / forget` | Smoke-tests, ad-hoc ingestion |
| Local UI | `cognee-cli -ui` → http://localhost:3000 | Inspecting the graph visually |
| Cognee Cloud | `await cognee.serve(url=..., api_key=...)` | Skipping infra during the event |
| Claude Code plugin | [`cognee-integrations`](https://github.com/topoteretes/cognee-integrations/tree/main/integrations/claude-code) | Giving Claude Code persistent memory |
| Examples | [`examples/`](https://github.com/topoteretes/cognee/tree/main/examples) in the cognee repo | Reference pipelines & demos |

## Submission

Each team submits:

- a short writeup of the idea and self-improvement loop
- the wiki implementation (code + skills, if any)
- before/after evidence that the wiki improved from feedback
- a 3-minute demo

Use [`templates/SUBMISSION.md`](./templates/SUBMISSION.md) — copy it into your
team folder or PR description and fill it in.

## Resources

- [Cognee Documentation](https://docs.cognee.ai/)
- [Karpathy on LLM Wikis](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- [Redis Documentation](https://redis.io/docs/)
- [Discord](https://discord.gg/NQPKmU5CCg)

---

*Full challenge brief, starter skills, and submission template will land in
this folder ahead of the event.*
