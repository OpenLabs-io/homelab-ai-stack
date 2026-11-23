# 03 · Model roster — a tiered lineup

Rather than one model for everything, the stack runs a **tier of models by job**,
sized so the important ones fit the 16 GB GPU alongside a long context. Exact
tags change as better models ship (see the [JOURNAL](../JOURNAL.md) for the
current snapshot); the **tiers** are the durable part.

## The tiers

| Tier | Job | Rough size | Lives on | Notes |
|---|---|---|---|---|
| **Primary chat** | The everyday driver — the one you actually talk to | ~27B, aggressive quant (e.g. IQ3) | **GPU** | Sized to fit **weights + 64K context** in 16 GB. The whole budget is built around this one. |
| **Deep reasoning** | Harder problems where you'll trade speed for quality | ~27B | GPU (on demand) | A reasoning-capable model; thinking **on** but at a low budget. |
| **Workhorse** | General mid-size, fast | ~12–26B (MoE helps) | GPU | A mixture-of-experts model gives big-model quality at small-model speed. |
| **Tool-runner** | The one allowed to actually *do* things | ~26B | GPU | The only roster model wired to the exec tools (see [07](07-gated-exec-pattern.md)). Kept separate so "chatting" and "acting" aren't the same model. |
| **Task model** | Open WebUI's background chores | ~1.7–2B, Q4 | **CPU** | Titles, tags, follow-up suggestions, search-query rewrites. Reasoning **off**. See [05](05-openwebui-tuning.md). |
| **Embedder** | RAG vectors | ~137M | **CPU** (or GPU) | Dedicated embedding model. See [06](06-rag-pipeline.md). |

## Why aggressive quantization on the primary model

A ~27B model at a middle quant (Q4+) would eat most of 16 GB in weights alone,
leaving no room for a usable context. Dropping the **primary** chat model to an
`IQ3`-class quant shrinks the weights enough that a **64K context** fits
alongside them. The quality hit from IQ3 on a good 27B is far smaller than the
practical win of 64K of working memory.

> **The rule that emerged:** quantize the **KV cache** and the **primary model's
> weights** to buy context — but don't quantize so far the model gets visibly
> dumber. The KV cache (`q8_0`) is nearly free to quantize; model weights have a
> real floor. Find the floor once, then leave it alone.

## Why a tiny CPU task model matters more than it looks

Open WebUI fires **up to five extra generations per message** — chat title, tags,
follow-up suggestions, a web-search query rewrite, a RAG retrieval query. If you
don't point those at a dedicated small model, **they run on whatever big model
you're chatting with**, competing for the GPU mid-conversation.

Pinning them to a ~2B model **on the CPU** gets them off the GPU entirely. They're
latency-insensitive background chores; the CPU is fine for them, and it keeps the
GPU exclusively for the model you're actually waiting on. See the `think:false`
win in [05](05-openwebui-tuning.md) — it makes that task model **8× faster**.

## Why the tool-runner is its own model

The model allowed to run shell commands is **deliberately not** the everyday chat
model. Separating "the model I brainstorm with" from "the model that can touch my
machines" means a careless chat can't accidentally fire a command, and the
tool-capable one can be held to a tighter leash (see [07](07-gated-exec-pattern.md)).

Not every model drives tools well, either — small models and non-tool-tuned ones
emit empty `tool_calls` and fail several steps into an agentic loop. The roster
guards against picking one of those for an agent task by name.
