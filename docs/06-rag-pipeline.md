# 06 · RAG pipeline

RAG (retrieval-augmented generation) is how the models answer questions about
*this* setup from a knowledge base instead of making things up. The interesting
decisions here are all about **not** doing heavy ML on the CPU-only server.

## Embed on the GPU box, not the server

The embedding model turns documents and queries into vectors. By default OWUI
would download a `sentence-transformers` model and embed **on its own host** — the
always-on **CPU-only server** — competing with everything else running there.

Instead, the embedder is pointed at **Ollama on the GPU box** (set the RAG
embedding engine to the Ollama backend). Now embedding happens on the machine with
the hardware for it, and the server stays light. The embedder is small (~137M), so
it co-resides cheaply with the chat model.

> **Changing the embedding model invalidates every existing vector** (dimension
> mismatch) and forces a full re-index of every document. Pick one and commit;
> don't swap it casually.

## Chunking & retrieval settings

| Setting | Value | Why |
|---|---|---|
| Chunk size | 1200 | Reasonable semantic unit for these docs |
| Chunk overlap | 200 | Keeps context across chunk boundaries |
| Top-K | 8 | Raised from 5 when the context window doubled |
| Embedding batch | 16 | Throughput without hogging the GPU |

**Why Top-K went from 5 to 8:** a single long document can be ~9 chunks. At K=5,
one long doc crowds out the shared knowledge base — the right *collection* got
retrieved but not the right *chunk*. Raising K once the context window had room
fixed it. The lesson: **Top-K has to scale with how big your individual documents
are**, not just how many documents you have.

## Hybrid search is OFF — on purpose

Enabling hybrid (keyword + vector) search pulls in a **cross-encoder reranker**
that runs on the **CPU** — i.e. on the always-on server, the exact machine the
whole design keeps ML *off* of. So it's disabled deliberately. On a split where
one box has the GPU and the other is always-on-but-weak, a CPU reranker defeats
the point. If both ran on the GPU box, the calculus would differ.

## Two silent RAG failures (both cost real time)

Neither throws an error. Both make a model **answer confidently from nothing** —
the worst failure mode for a model whose job is knowing your facts. The symptom
is identical: a plausible answer and an **empty sources list**.

### 1. An attached knowledge base needs the right `type` field

The retrieval code dispatches purely on an item's `type` (`collection`, `file`,
`note`, …). A knowledge entry attached without that field matches **no branch and
is dropped silently** — the model is left with zero context and invents an answer.
If you ever wire up knowledge through the API, don't strip the `type`.

### 2. Whether RAG runs **at all** depends on the function-calling mode

This is the subtle one:

| Mode | What happens to knowledge |
|---|---|
| **`legacy`** | Knowledge is retrieved **every turn** and injected as context. Deterministic, grounded, cited. |
| **`native`** | Retrieval is **skipped**. Knowledge is instead offered as a *tool* the model may choose to call — and often doesn't, or searches with poor queries and finds nothing. |

Measured: on `native`, asking a grounded factual question produced an invented
number with **zero sources**. On `legacy`, the same question answered correctly
**with citations**.

**So the choice is a real trade, not a default to ignore:**
- **Knowledge workspaces run `legacy`** — their job is *knowing*, and grounded
  cited answers are the point.
- **The tool-runner runs `native`** — its job is *doing*, and native gives it real
  `tool_calls`. It accepts weaker retrieval as the cost.

> **The generalizable lesson:** "function calling" and "RAG retrieval" are coupled
> in ways that aren't obvious, and getting it wrong fails *silently*. If a model
> that should be citing sources is answering from thin air, check the retrieval
> mode before you blame the model.

## Keep the knowledge base honest

These documents are **point-in-time snapshots**. They don't update themselves.
When the infrastructure changes, edit the source and re-index — or the models will
confidently tell you something that stopped being true. A stale knowledge base is
worse than none, because it's *confidently* wrong.
