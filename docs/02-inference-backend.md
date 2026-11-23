# 02 · Inference backend — Ollama, and Vulkan over ROCm

## The runtime

[Ollama](https://ollama.com) serves every model. It wraps llama.cpp, handles
model loading/unloading, exposes an OpenAI-compatible API, and — importantly for
a box that sleeps — loads models lazily on first request.

The non-obvious choice is the **compute backend underneath it.**

## Vulkan, not ROCm — and why

The GPU is an AMD RDNA-class consumer card (`gfx1201`, 16 GB). AMD's "official"
compute path is **ROCm/HIP**. This stack deliberately runs Ollama on the
**Vulkan** backend instead.

### What happened (the short version)

- A ROCm **major-version bump** landed and **every GPU model started
  segfaulting** on load. GPU inference was completely dead.
- Immediate fix: **roll ROCm back** to the last known-good version and **pin it**
  so the package manager can't upgrade it out from under a working setup again.
- Then the better fix: **switch Ollama to the Vulkan backend.** It turned out to
  be *faster* for this card anyway, and it decouples inference from the ROCm
  version treadmill entirely.

### The lessons

1. **On a bleeding-edge consumer GPU, the vendor compute stack is the fragile
   part — not the models.** A toolchain upgrade broke everything while the models
   were fine.
2. **Pin the thing that broke you.** A version pin (e.g. your package manager's
   ignore/hold list) turns "silently upgraded and now broken" into "stays working
   until I choose otherwise." Keep a note of *why* each pin exists so you know
   when it's safe to lift.
3. **Vulkan is a legitimate backend, not a fallback.** For newer AMD cards it can
   match or beat ROCm and sidesteps a whole category of version-coupling pain. If
   HIP is giving you grief, measure Vulkan before fighting ROCm.
4. **Blind the thing you're not using.** When running the Vulkan backend, the
   ROCm/HIP device visibility env vars are set to hide the GPU from HIP, so
   nothing accidentally takes the ROCm path. One backend, no ambiguity.

> ROCm is still *pinned and present* — some other tools use it — but Ollama never
> touches it. The pin protects the rest of the system; Vulkan does the inference.

## KV cache & attention settings

These are set once in the environment Ollama runs under:

| Setting | Value | Why |
|---|---|---|
| Flash attention | **on** | Lower memory and faster attention; near-free on supported setups. |
| KV cache type | **`q8_0`** (quantized) | Halves KV-cache VRAM vs `f16` at negligible quality cost — this is what buys you a long context. See [04](04-vram-budget.md). |
| Keep-alive | **indefinite** | Keep the primary model resident so you don't pay a reload on every message. (Trade-off: it holds VRAM — see the OOM lesson in [08](08-lessons-learned.md).) |
| Max loaded models | **3** | Lets the chat model, the task model, and the embedder co-reside without evicting each other. |
| Parallel requests | **1** | Single-user box. More parallelism multiplies KV-cache cost — see the trade in [05](05-openwebui-tuning.md). |

### The `q8_0` KV cache is the quiet hero

Quantizing the KV cache from 16-bit to 8-bit roughly **halves** the memory each
token of context costs. On a 16 GB card that's often the difference between a
32K and a 64K context fitting alongside the weights. Quality impact is
essentially unmeasurable in practice. Quantizing the KV cache was far less
risky here than quantizing the model any further.

Next: [03 · Model roster](03-model-roster.md) for what actually runs, and
[04 · VRAM budget](04-vram-budget.md) for the arithmetic that ties all of this
together.
