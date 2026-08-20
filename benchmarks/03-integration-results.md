# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.2 | 19260.3 | 19260.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.2 | 13043.2 | 13043.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.2 | 13332.3 | 13332.6 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **15211.9** · total **15212.2**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 (embeddings)**: **stubbed**. `embed()` in `pipeline.py` returns `None` because no
  embedding server is running (`make serve-embed` / bonus C9 was not started this run).
- **N17 (vector index / lakehouse)**: **stubbed**. `TOY_DOCS` is a hard-coded in-memory
  Python list of 6 short documents, not a real vector store.
- **N18 (retrieval)**: **stubbed, but functional**. Since no embeddings exist, `retrieve()`
  falls back to a plain keyword-overlap scorer against `TOY_DOCS` — it genuinely ranks
  and returns the top-3 docs by matching query terms, it just isn't semantic search.
- **N19 (generation / LLM call)**: **real**. Every answer above came from an actual HTTP
  call to the local `llama-server` (Gemma 4 E2B, CPU-only) — no canned responses,
  no mocking.

**Is the dominant stage what I expected?** Yes. `llm` is 100% of total latency
(mean 15211.9 ms of 15212.2 ms total) — embed and retrieve are ~0 ms because they're
just Python dict/string lookups over 6 tiny toy documents, not real vector search. On a
real N17/N18 stack with an embedding server round-trip and an actual ANN index over
thousands of documents, retrieve would no longer be free, but it would still be small
next to a multi-second LLM decode on this CPU-only setup.

**Where I'd attack to halve latency**: the `llm` stage, specifically **decode**, not
prefill. Per-query server timings show prefill costs 3–6s but decode costs 6.4–9.0s for
only 23–30 output tokens — consistent with the ~3 tok/s CPU decode rate measured in
`01-quickstart-results.md`. The retrieval/embedding stages are already free in this
run, so optimizing them buys nothing; the two levers that actually move `llm` time are
(1) the thread-count tuning already found in `01-tuning-tg128.md` (`-t 8` gives 2.62x
raw decode throughput), and (2) shortening `max_tokens` per answer, since decode cost is
linear in output length and this pipeline doesn't need long answers to prove the
context was retrieved correctly.
