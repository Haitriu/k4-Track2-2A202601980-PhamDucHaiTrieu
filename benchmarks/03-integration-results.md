# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.2 | 11572.5 | 11572.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.2 | 8552.0 | 8552.3 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.2 | 8686.5 | 8686.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.2** ·
llm **9603.7** · total **9604.0**
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
  call to the local `llama-server` (Gemma 4 E2B, GPU offload active on the Quadro M2200) —
  no canned responses, no mocking.

**Is the dominant stage what I expected?** Yes. `llm` is 100% of total latency
(mean 9603.7 ms of 9604.0 ms total) — embed and retrieve are ~0 ms because they're
just Python dict/string lookups over 6 tiny toy documents, not real vector search. With
GPU offload this run is ~1.6x faster end-to-end than the earlier CPU-only pipeline run
(9603.7 ms vs 15211.9 ms mean), which lines up with the ~1.9x-2.8x decode speedup GPU
gave in `01-quickstart-results.md` and `01-tuning-tg128.md` — llm dominates either way,
GPU just makes that dominant stage itself faster.

**Where I'd attack to halve latency**: still the `llm` stage, specifically **decode**.
Per-query server timings show prefill now costs 2.4–3.3s but decode still costs
2.5–3.3s for only 23–30 output tokens even with GPU offload — retrieval/embedding are
already free, so there's nothing to gain there. The two levers that would actually move
`llm` time further are (1) `--parallel`/batching tuning from `02-server-results.md`
(this pipeline runs one request at a time, so it never benefits from continuous batching
the way the load tests do), and (2) shortening `max_tokens` per answer, since decode
cost is linear in output length and this pipeline doesn't need long answers to prove
context was retrieved correctly.
