# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 50 | 0.43 | 18000 | 33000 | 34000 | 8.4 | 0.0% |
| 50 | 52 | 0.45 | 50000 | 105000 | 109000 | 24.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.07x** (21% of linear) |
| P95 latency | **3.18x** |
| Effective concurrency at 50 users | 24.3 vs `--parallel 4` slots (occupancy/slot ratio 6.07) |

**Saturated.** Throughput delivered only 1.07x for 5x the offered load, and effective concurrency (24.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.07x while P95 moved 3.18x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server is saturated at or before 50 users, and with GPU offload active this shows
up as a **pure throughput ceiling**, not timeouts. The number that convinced me is
throughput only rising **1.07x for a 5x increase in offered load** (21% of linear) —
almost all of the extra load bought nothing. `n_busy_slots_per_decode` (`make metrics`,
peak 3.99/4 = 100%) confirms why: the 4 GPU decode slots were already continuously busy
at 10 users' worth of load, so there was no spare capacity for 50 users to fill.

P95 grew **3.18x** while throughput grew only 1.07x — that gap is entirely queue time,
not compute time. I know this because per-request compute cost (TPOT) is fixed by the
model and hardware regardless of queue depth; if the extra latency were compute, P50
and throughput would move together. Instead requests just wait longer behind an
already-full set of 4 slots (`requests_deferred` peaked at 46 in the batching sample).
Unlike the CPU-only run, **0% of requests timed out** here — GPU decode is fast enough
per-slot that even a 24-deep effective queue still drains inside the 120s client
timeout, it just costs latency, not correctness.

**What I'd change first**: raise `--parallel` (add more decode slots), not threads.
Threads only govern CPU-side work (tokenization, sampling, KV-cache bookkeeping); the
actual bottleneck here is GPU decode slot count, confirmed by slots pegged at 4/4
while the GPU still had free VRAM (Quadro M2200 is 4096 MiB, the model is ~3 GB). More
`--parallel` slots let the GPU batch more sequences per decode step instead of queuing
them, which is the direct lever on the throughput ceiling this data shows — unlike the
CPU-only run, where `-t 8` from `make tune` was the free win because compute threads,
not slots, were the constraint there.
