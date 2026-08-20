# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 1.6 | 38% |
| 2 | 2.8 | 67% |
| 4 | 4.0 | 96% |
| 8 | 4.1 | 100% |
| 16 | 3.3 | 81% |

**Best**: `-t 8` at 4.1 tok/s
**Slowest tested**: `-t 1` at 1.6 tok/s (2.62x spread)
**Against the physical-core default** (`-t 4`, 4.0 tok/s): 1.04x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

The curve does **not** peak at the physical core count (4) as the deck's default
expectation suggests — it keeps climbing to `-t 8`, i.e. the full logical (hyperthreaded)
core count, before falling off at `-t 16`. `-t 4` only reaches 96% of the best result.

Decode (`tg128`) on this CPU-only run is heavily compute-bound: dequantizing Q4_K
blocks and running the matmuls is enough per-token work that hyperthreading's
extra hardware thread on each physical core still finds useful instruction-level
parallelism to fill, instead of just fighting the sibling thread for the same
execution ports and cache lines. That is different from the classic
"hyperthreading doesn't help decode because it's memory-bandwidth-bound" story —
on this Skylake-family mobile part (i7-6820HQ) the extra scalar/dequant work
apparently keeps both threads per core busy rather than starved.

Above 8 threads (`-t 16` — oversubscribing 8 logical cores 2x), throughput drops
to 81% of peak: the OS scheduler is now context-switching more threads than there
are hardware contexts, so cache residency and thread-pool synchronization overhead
(barriers between decode steps) start costing more than the extra parallelism buys.

**Speedup used for REFLECTION §5**: `-t 8` vs `-t 1`, the spread reported above = **2.62x**.
