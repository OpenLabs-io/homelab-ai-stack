# 05 · Open WebUI tuning

[Open WebUI](https://github.com/open-webui/open-webui) (OWUI) is the chat
front-end. It runs on the always-on server and points at the GPU box's Ollama.
A few settings matter a lot, and one of them bites everyone.

## The PersistentConfig trap (read this first)

Many OWUI settings are **PersistentConfig**: the first time the app runs, it
copies the value from the environment variable into its **database**, and from
then on **the database wins** and the env var is ignored.

So you set `SOME_SETTING` in your compose file, restart, and nothing changes —
because the DB still has the old value. **Change these in the UI (or the DB), not
just the env**, or you'll chase your tail. When in doubt, assume a setting you can
see in the Admin UI is DB-backed and edit it there.

## The task model — the single highest-leverage setting

OWUI fires **up to five background generations per message** (title, tags,
follow-ups, search-query rewrite, RAG query). Point them at a dedicated small
model or they run on your big chat model and stall the GPU mid-conversation.

Set `task.model.default` to a **small model** (see [03](03-model-roster.md)). But
there's a catch that's easy to miss:

### Turn OFF reasoning on the task model

If your small model is a **hybrid reasoning** model (thinking on by default),
every one of those background chores will *think* before answering — pure waste
on a chat-title generation. Measured through OWUI's real title template:

> **1464 ms with thinking → 180 ms without. ~8× faster**, on invisible work, on
> every single message.

What **didn't** work to disable it (all tried):
- `PARAMETER think false` in the Modelfile → runtime rejects it as unknown.
- `SYSTEM "/no_think"` → ignored; the chat template only injects the switch into
  the last *user* message, so a system-block copy does nothing.
- A patched `TEMPLATE` → silently not applied.
- Appending a literal `/no_think` to each task prompt → **unreliable**; it's a
  soft trained behavior, not a hard switch. Worked in an isolated test, failed
  through the real template.

What **does** work: the runtime's **request-level `think: false`**. OWUI lifts a
model parameter named `think` into the request root, and task generation runs
through the same path that applies model params. So the fix is a **workspace
model whose params carry `think: false`**, set as the task model.

**Verify it:** fire a title-generation request and confirm the response's
reasoning content is empty. If reasoning comes back, something reset the param
(a UI save or an app update can) — re-apply it.

## Memory / parallelism settings

Set in the environment Ollama runs under (see also [02](02-inference-backend.md)):

| Setting | Value | Why |
|---|---|---|
| Parallel requests | **1** | Single user. Each slot needs its own KV cache; 1 slot keeps the whole VRAM budget for one long context. |
| Max loaded models | **3** | Primary + task model + embedder co-resident without evicting each other. |
| Context length | **32K default** | Per-request override goes higher (64K) for the primary chat. Default kept modest so casual chats don't each grab a 64K KV cache. |
| GPU overhead reserve | **~2 GB** | Holds back VRAM for the desktop so the runtime doesn't claim it and force a spill. Don't over-reserve — it comes straight out of context. |

### The parallelism ↔ context trade

`num_parallel × context_length` is roughly your total KV-cache footprint. Raising
parallelism silently multiplies KV cost — on a single-user box that's VRAM spent
on slots you'll never use at once. Keep parallel = 1 and spend the memory on
**context depth** instead of **concurrency**.

## Operational notes

- **Pin the OWUI image and exclude it from auto-updates.** A surprise major
  version bump can collide a cached frontend/service-worker with a new backend
  and leave the app "not loading" in one browser while another is fine. If that
  happens, it's almost always a **stale service worker** — clear site data for
  the domain (a hard refresh often isn't enough); the server is healthy.
- After editing models through the API, the in-process model cache needs a
  refresh (re-fetch the models list or restart the container) or you'll be
  testing the old config.
