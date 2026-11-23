# 01 · Architecture — the split-brain design

## The core idea

Two machines with opposite properties:

- **Server** — modest, CPU-only, **always on**. Cheap to run 24/7.
- **Workstation** — a real GPU (16 GB VRAM), but **powered off** most of the
  time to save electricity and wear.

Rather than compromise (run everything on one box, or leave the power-hungry one
on all day), the stack is split by *what each machine is good for*:

| On the Server (always up) | On the Workstation (on demand) |
|---|---|
| Open WebUI — the chat interface | Ollama — all model inference |
| Chat history, settings, RAG vector store | The embedding model |
| Web search (self-hosted, no API keys) | Chat / reasoning / task models |
| The **wake button** (a sleeping PC can't wake itself) | A **graceful-shutdown** endpoint |

### The consequence that makes it worth it

**The UI and all history stay reachable while the GPU box sleeps.** You can open
the chat app, read old conversations, change settings — all against the tiny
always-on server. The expensive machine only has to be awake for the seconds it's
actually generating tokens.

## The workflow

1. Open the dashboard → **Wake Workstation** (sends a Wake-on-LAN magic packet,
   then polls until the box answers).
2. Open the chat UI, pick a model or workspace, work.
3. **Shut Down Workstation** when done.

All of this works from away-from-home over a VPN. Nothing is exposed to the
public internet.

## Two design decisions worth stealing

**Wake and shutdown are *relayed through the server*, never linked directly.**
- A dashboard link opens in your *browser* — which might be a phone on cellular.
  A direct link to the workstation would arrive from an address its firewall
  drops. Relaying through the always-on server means the command originates from
  a trusted host on the LAN.
- Relaying also keeps the workstation's shutdown token out of the browser
  entirely.

**Shutdown is a two-step `GET → confirm → POST`, never a one-shot link.**
- Browsers, chat clients, and link-preview bots **prefetch** URLs. A destructive
  action behind a plain `GET` will eventually fire on a hover or a preview. The
  confirm page turns the dangerous step into an explicit `POST` that only a human
  click produces.

## Why not just one box, or always-on?

- **One box, always on:** a 16 GB GPU idling 24/7 is real money and heat for a
  machine used a few hours a day.
- **One box, wake-on-demand:** then the chat UI and history disappear every time
  the box sleeps — you lose the ability to even *look* at past chats without
  spinning up the GPU.
- **The split:** cheap persistence on the server, expensive compute on demand.
  The only cost is a few seconds of wake latency, hidden behind a "waking…"
  poll.

See [04 · VRAM budget](04-vram-budget.md) for how the workstation's single GPU is
divided, and [07 · Gated exec](07-gated-exec-pattern.md) for how models are given
*controlled* ability to act on either machine.
