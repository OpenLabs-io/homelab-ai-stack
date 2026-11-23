# 04 · VRAM budget — fitting a model *and* its context in 16 GB

The single most useful mental model in this whole stack. On a 16 GB consumer GPU,
everything is a fight for the same memory, and if you don't budget it explicitly
you'll either OOM or silently spill to the CPU (and wonder why it's slow).

## The equation

```
Total VRAM  =  model weights
            +  KV cache (grows with context length)
            +  compute / activation buffers
            +  whatever the OS/desktop already took
```

All four share the **same 16 GB**. There is no separate pool for "the model" and
"the display" — if your desktop compositor and browser are using 2 GB, your model
+ context has to fit in the remaining ~14 GB.

## Worked example (the primary model on this box)

A ~27B model at `IQ3_S`, flash attention on, KV cache `q8_0`, **64K context**:

| Component | ~Size | Notes |
|---|---|---|
| Model weights | ~10.6 GB | 27B at IQ3-class quant |
| KV cache @ 64K | ~2.3 GB | `q8_0` — would be ~2× this at `f16` |
| Compute buffers | ~0.2 GB | attention/activation scratch |
| **Subtotal (model)** | **~13.1 GB** | what the runtime reports resident |
| Desktop + browser | ~1.5–2.5 GB | compositor, the chat UI, other windows |
| **Total** | **~15–16 GB** | ...which is the whole card. It's *full*, by design. |

That's not a mistake — it's the budget **maxed on purpose** to get 64K of context
out of 16 GB. The cost is that there's no headroom, so a few habits matter:

## Consequences you have to live with

1. **The display competes with the model.** No integrated GPU here, so the same
   card draws the desktop. A heavy wallpaper, a second 4K monitor, or a
   GPU-accelerated browser tab all eat into the model's space. Keep the desktop
   lean when the budget is maxed. (A static wallpaper over an animated one is
   free VRAM.)
2. **Context length is the biggest *adjustable* cost.** Weights are fixed once you
   pick the model; the KV cache scales with how much context you allow. Halving
   the context roughly halves the KV cache. Quantizing the KV cache (`q8_0`)
   halves it again at almost no quality cost — **do that first**, before cutting
   context.
3. **Parallelism multiplies the KV cost.** Each parallel request slot needs its
   own KV cache. On a single-user box, **1 parallel slot** keeps the whole budget
   for one long context instead of splitting it across slots you'll never use
   simultaneously. (See the trade in [05](05-openwebui-tuning.md).)
4. **Keep-alive forever = that VRAM is gone until you unload.** Keeping the
   primary model resident avoids reload latency, but it means the VRAM is
   permanently spoken for. If a *second* model gets pinned resident too (easy to
   do by accident), you can OOM or spill — see the OOM story in [08](08-lessons-learned.md).
5. **A reserve for the display prevents spill.** Telling the runtime to hold back
   a GB or two of "overhead" for the OS stops it from greedily claiming VRAM the
   desktop needs and forcing an eviction. Reserve a little; don't reserve so much
   it eats into context you need.

## The decision order when you're out of VRAM

Work down this list — cheapest-quality-cost first:

1. **Quantize the KV cache** (`f16` → `q8_0`). Nearly free. Biggest single win.
2. **Trim the context length** to what you actually use. Direct KV savings.
3. **Lean out the desktop** (wallpaper, extra monitors, GPU browser).
4. **Keep parallelism at 1** on a single-user box.
5. **Drop the model quant** a notch — **last resort**, because this is the one
   that actually makes the model *dumber*. Exhaust the others first.

> The whole art is spending the 16 GB on the things you'll notice (model smarts,
> context length) and not on the things you won't (an `f16` KV cache, parallel
> slots you never use, a fancy wallpaper).
