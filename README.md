# Homelab AI Stack

A field guide to running a **private, local-first LLM stack** on consumer
hardware — the architecture, the tuning decisions, and the mistakes that cost
real time. Written as a reference and a running journal, not a product pitch.

Everything here runs on two machines at home. Nothing is exposed to the public
internet; remote access is over a VPN. No API keys, no cloud inference, no
telemetry leaving the house.

> **Why this exists.** The setup grew deep enough that "how is this tuned and
> *why*" stopped fitting in my head. This is the sanitized, shareable version of
> my internal notes — useful if you're building something similar, and honest
> about the dead ends.

---

## The shape of it

Two boxes with opposite strengths, deliberately split:

| | **Server** (always-on) | **Workstation** (on-demand) |
|---|---|---|
| Role | Chat UI, history, orchestration | All model inference |
| GPU | none | 16 GB consumer GPU |
| Power | sips, runs 24/7 | powered off until needed (Wake-on-LAN) |
| Hosts | Open WebUI, search, the wake button | Ollama, the models, embeddings |

The payoff: **the chat UI and all history stay up even while the GPU box is
asleep.** Only generating a response needs the expensive machine awake. Wake it
with a magic packet, use it, shut it down.

```
        ┌─────────────────────┐          Wake-on-LAN          ┌──────────────────────┐
        │   SERVER (24/7)     │ ───────── magic packet ─────► │  WORKSTATION (GPU)   │
        │                     │                                │                      │
        │  Open WebUI  ◄──────┼──── inference over LAN ───────┼──►  Ollama (Vulkan)  │
        │  chat history       │                                │     chat models      │
        │  RAG vector store   │ ───── embeddings over LAN ────┼──►  embedder          │
        │  web search         │                                │                      │
        └─────────────────────┘                                └──────────────────────┘
                  ▲
                  │ VPN only (nothing public)
             you, anywhere
```

## Contents

| Doc | What's in it |
|---|---|
| [01 · Architecture](docs/01-architecture.md) | The two-box split-brain design and why it's shaped this way |
| [02 · Inference backend](docs/02-inference-backend.md) | Ollama on **Vulkan** vs ROCm, the version-pin saga, flash attention + KV cache |
| [03 · Model roster](docs/03-model-roster.md) | A tiered roster — primary / deep / workhorse / tool-runner / task / embedder — and quant choices |
| [04 · VRAM budget](docs/04-vram-budget.md) | The equation for fitting a model **and** its context in 16 GB, with worked numbers |
| [05 · Open WebUI tuning](docs/05-openwebui-tuning.md) | The PersistentConfig trap, the task-model `think:false` win, parallelism vs context |
| [06 · RAG pipeline](docs/06-rag-pipeline.md) | Embedding on the GPU box, chunking, why hybrid search is off, two silent failure modes |
| [07 · Gated exec pattern](docs/07-gated-exec-pattern.md) | Giving a local model real shell access **safely** — approval gating, allowlists, the lessons |
| [08 · Lessons learned](docs/08-lessons-learned.md) | Cross-cutting gotchas that generalize beyond this stack |
| [JOURNAL](JOURNAL.md) | Dated log of changes, experiments, and what the stack looks like *right now* |

## The one-paragraph version

Open WebUI on the always-on server talks to Ollama on a GPU workstation that's
asleep most of the time. Ollama runs on the **Vulkan** backend (chosen after a
ROCm version bump broke GPU inference), serving a tiered roster of quantized
models sized so the model weights **and** a long context both fit in 16 GB of
VRAM at once. A tiny "task model" with reasoning turned off handles Open WebUI's
background chores cheaply. RAG embeds on the GPU box so the CPU-only server never
does heavy ML. A separate approval-gated service lets models run real commands
without being able to approve their own actions. Most of the hard-won knowledge
here is about **fitting things in 16 GB** and **failing loudly instead of
silently**.

## Describing it to someone technical

If a friend who knows the space asks what I'm actually running:

> On a 16 GB AMD card I run a quantized 27B (IQ3_S) as the primary chat model
> with a full 64K context — the weights **and** the KV cache both fit in 16 GB
> thanks to a q8_0 cache. Ollama's on the **Vulkan** backend, not ROCm. Two
> small models run on CPU to keep the GPU free: a ~2B task model with thinking
> disabled for Open WebUI's background jobs, plus a `nomic` embedder for RAG.
> **Tool calling is a separate native-function-calling model on the GPU — not
> the CPU ones.**

The detail that makes people nod is the fit: a 27B *and* a 64K window in 16 GB
on Vulkan. The common misconception to head off — the little CPU models do
background chores and embeddings, **not** tool calls.

## Not included (on purpose)

This is a sanitized public mirror of a private setup. Internal IPs, hostnames,
domains, auth tokens, and the live network surface of the exec service are **not**
here — only the designs, decisions, and lessons. If you're adapting this, the
reasoning transfers; the addresses were never the point.

## License

[MIT](LICENSE) — use any of it. No warranty; these are notes from one specific
house, not universal truth.
