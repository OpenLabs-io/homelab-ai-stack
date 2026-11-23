# Journal

A running log of changes, experiments, and what the stack looks like right now.
Newest first. Dates are `YYYY-MM-DD`.

---

## Current snapshot

> Updated 2026-11-22. The tiers are stable; exact model tags evolve — this is the
> state as of the last update.

- **Inference backend:** Ollama on the **Vulkan** backend. ROCm pinned to a
  known-good version but not used by Ollama (blinded via device-visibility env).
- **GPU:** one 16 GB consumer card (RDNA-class), driving both inference and the
  desktop.
- **Primary chat model:** ~27B at IQ3-class quant, **64K** context, flash
  attention on, **`q8_0`** KV cache. Weights + context max out the 16 GB on
  purpose.
- **Task model:** ~2B, Q4, **on CPU**, reasoning **off** (`think:false`) — handles
  all of Open WebUI's background chores.
- **Embedder:** small dedicated model, embedding on the GPU box via Ollama.
- **Parallelism:** 1 slot · **max 3 models resident** · ~2 GB GPU overhead
  reserved for the desktop.
- **RAG:** `legacy` function-calling on knowledge workspaces (retrieval every
  turn, cited); `native` on the tool-runner. Hybrid search **off** (keeps the CPU
  reranker off the always-on server).
- **Exec:** per-host gated service — read path runs immediately (no shell,
  allowlisted); write path requires a human push-approval. Control plane kept out
  of the model's tool schema.

---

## Log

### 2026-11-22 — Repo created
- Started this repository: the sanitized, shareable version of the internal
  homelab notes. Covers architecture, inference backend, model roster, the VRAM
  budget, Open WebUI tuning, RAG, and the gated-exec pattern.
- Documented a live **context-exhaustion** episode: the primary chat filled its
  full 64K window (`truncated=1`), forcing a full-context reprocess (~1m40s per
  turn) that *looked* like a hang but wasn't — the backend stayed healthy and kept
  returning results. Reinforced the "fork to fresh context" workflow and added it
  to [Lessons](docs/08-lessons-learned.md).

### Earlier milestones (reconstructed, approximate dates)

> These predate the journal and are summarized from memory; treat the ordering as
> approximate.

- **GPU toolchain break → Vulkan.** A ROCm major-version bump broke all GPU
  inference (every model segfaulting on load). Rolled ROCm back and **pinned** it,
  then moved Ollama to the **Vulkan** backend — faster for this card and free of
  the ROCm version coupling. See [02](docs/02-inference-backend.md).
- **Task-model `think:false`.** Chased down why background chores were slow;
  landed on request-level `think:false` via a workspace model param after four
  other approaches failed. ~8× faster background generations. See
  [05](docs/05-openwebui-tuning.md).
- **RAG silent-failure hunt.** Diagnosed two no-error failure modes (missing
  `type` on knowledge attachments; `native` vs `legacy` deciding whether RAG runs
  at all). Standardized knowledge workspaces on `legacy`. See
  [06](docs/06-rag-pipeline.md).
- **Gated exec hardening.** A sequence of fixes after each silent-failure bug:
  no-shell read path (injection structurally impossible), control routes removed
  from the tool schema (model could approve itself), `curl` classifier that
  expands clustered flags, explicit `TRUNCATED` markers, an activity ledger. See
  [07](docs/07-gated-exec-pattern.md).

---

## Ideas / backlog

- [ ] Automate a "context is getting deep — fork now?" nudge in the chat UI.
- [ ] Revisit the primary model as better small-quant 27B-class models ship.
- [ ] Periodic check that the task-model `think:false` param survived app updates
      (it's reset-prone).
- [ ] Measure Vulkan vs ROCm head-to-head on the current card and record numbers.
