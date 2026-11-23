# 08 · Lessons learned — the cross-cutting ones

The gotchas that generalize beyond any single component. Most of these cost hours
before the cause was obvious in hindsight.

## Silent failure is the enemy

> **A model given a tool or context that fails silently will invent a plausible
> answer rather than report the failure.**

This is the single most repeated lesson in the whole stack. A dropped RAG
attachment, a pipe that became a literal argument, a tool that errored on the
wrong host — every time, the model produced a confident, wrong answer instead of
an error. **Make every partial, dropped, or refused result *visible* in what the
model receives.** An empty result and a failed result must not look the same.

## Context exhaustion looks like a crash, but isn't

When a chat fills the model's full context window and context-shifting is off, the
runtime drops the oldest tokens and **reprocesses the entire remaining context
from scratch** — a minute-plus pause that *looks* like a hang. It isn't; it's the
expensive fallback. The runtime is healthy and still returning results.

- **Diagnose before rebooting.** Check that the inference API still answers and
  slots go idle between requests. If it's returning `200`s slowly, it's
  context-grinding, not crashed — **don't restart anything.**
- **The fix is workflow, not config:** start a fresh chat before a conversation
  runs to the ceiling. Carry forward a short summary, not the full transcript.
  "Fork to fresh context" beats fighting a maxed-out window every turn.

## The display shares the model's VRAM

No integrated GPU means the same card draws your desktop *and* runs the model.
When the VRAM budget is maxed for context, a heavy wallpaper, an extra monitor, or
a GPU-accelerated browser tab comes **straight out of the model's space**. Keep
the desktop lean when you're running the budget tight. Conversely, don't blame the
model for "random slowdowns" that are really the compositor reclaiming VRAM.

## Keep-alive-forever can OOM you by accident

Pinning the primary model resident is great for latency — until a *second* model
also gets pinned (a stray `keep_alive=-1`, a workspace that loads a different base)
and now two big models fight for 16 GB. Symptom: sudden sub-30-fps in games or an
OOM-kill, with a model you forgot about sitting in VRAM.

> **First thing to check on any "why is the GPU slow" report: what's actually
> resident.** List the loaded models before theorizing. The culprit is often a
> model parked from hours ago.

## The vendor GPU stack is the fragile layer

On a bleeding-edge consumer GPU, the models are stable and the **vendor compute
toolchain** is what breaks. A major-version bump of the GPU runtime broke *all*
inference overnight. **Pin the GPU toolchain version**, keep a note of *why* each
pin exists, and treat an alternative backend (Vulkan) as a first-class option, not
a fallback.

## Config that lives in a database ignores your env file

Settings that get copied into an app's database on first run will **ignore the
environment variable** forever after — you edit the compose file, restart, and
nothing changes. Know which of your settings are DB-backed and change *those* in
the app/DB, not the env. Hours vanish into "why won't this setting take."

## Stale service workers masquerade as outages

A PWA that "won't load" after an update — blank in one browser, fine in another —
is almost always a **stale cached frontend / service worker** colliding with the
new backend, not a server fault. Confirm the server is healthy (health endpoint,
version, a websocket upgrade), then **clear site data** in the broken browser. A
plain hard-refresh often isn't enough; the service worker survives it.

## One shared secret in N places drifts

A single auth key duplicated across several files/hosts **will** silently diverge,
and the failure is confusing (one host rejects the "wrong" token with a vague
error). If you must duplicate a secret, **write down every location**, and when
one host misbehaves, **test the key first** before touching anything else. A full
day was lost to three copies of a key quietly disagreeing.

## `PATH` for a background service is not your shell's `PATH`

A user-level background service inherits a **minimal** `PATH`, not your login
shell's. A binary that works in your terminal returns "not found" to the service.
Worse, the failure can be indirect — the caller falls back to a *different* tool
and produces nonsense. **If one tool "doesn't work" but the service is healthy,
print the service's `PATH` before anything else.**

## The thread through all of it

Most of these reduce to two habits:

1. **Make failures loud.** Silent-empty and silent-wrong are the expensive ones —
   in RAG, in tools, in context handling. Surface every degraded state.
2. **Diagnose before you reboot.** "Looks stuck / looks broken" is usually a known,
   benign state (context grind, VRAM contention, stale cache) wearing a scary
   costume. Look at what's actually running first; the restart you avoid is the
   model you don't have to reload.
