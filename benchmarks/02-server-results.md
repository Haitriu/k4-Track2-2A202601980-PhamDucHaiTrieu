# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=0` (CUDA offload hangs indefinitely on this Quadro M2200; forced CPU-only via `LAB_N_GPU_LAYERS=0`)

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 20 | 0.12 | 69000 | 104000 | 104000 | 7.9 | 0.0% |
| 50 | 58 | 0.34 | 122000 | 122000 | 122000 | 35.4 | 65.5% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **2.81x** (56% of linear) |
| P95 latency | **1.17x** |
| Effective concurrency at 50 users | 35.4 vs `--parallel 4` slots (occupancy/slot ratio 8.85) |

**At capacity, still scaling.** All 4 decode slots are busy (effective concurrency 35.4) but throughput still rose 2.81x. You are at the knee -- the next increment of load is where P95 starts to run away.

P95 grew no faster than throughput (1.17x vs 2.81x), so this server still has headroom at 50 users.

## Your reading

The server is saturated well before 50 users. The number that convinced me is the
**65.5% failure rate at 50 users** (38 of 58 requests hit the 120s client read-timeout),
versus **0% failures at 10 users**. Effective concurrency at 50 users is 35.4 — nearly
9x the `--parallel 4` slot count — meaning the vast majority of in-flight requests were
sitting in queue, not being decoded. `n_busy_slots_per_decode` (from `make metrics`,
peak 3.77/4) confirms all 4 decode slots were continuously busy, so throughput was
already at its ceiling; adding more users only grew the queue, not the output rate.

Throughput did rise 2.81x going from 10→50 users (56% of the 5x offered-load increase),
which is real continuous-batching gain from 1 busy slot's worth of idle capacity being
soaked up — but P50/P95/P99 latency all collapsed to the same ~122s value, i.e. nearly
every successful request waited almost the full client timeout. That flat, bunched
percentile spread (P50 = P95 = P99) is itself saturation evidence: it means requests
aren't finishing at varying points along a normal service-time curve, they're all
getting released together near the timeout ceiling — a queueing signature, not a
compute one.

**What I'd change first**: raise `--parallel` (more decode slots) rather than threads.
This machine is CPU-only (GPU offload hangs — see REFLECTION §1) with only 4 physical
cores, so `--parallel` slots and OS/decode threads are already competing for the same
4-8 hardware contexts; adding slots without adding real compute would just make each
slot's per-token cost worse. The actual first lever here is the one `make tune` already
found: `-t 8` instead of the physical-core default `-t 4` was a free 2.62x on raw decode
throughput (benchmarks/01-tuning-tg128.md) with zero added contention, before touching
`--parallel` at all. Only after that would I look at trimming `LAB_LOAD_SHORT_TOKENS`/
`ctx` or adding real slots to attack the queueing bottleneck directly.
