# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 20 samples over
115s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.99 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1745 |

Highest sampled value was **3.99 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.99 of 4 slots (100%)** — with GPU offload active, the server
kept all 4 decode slots continuously busy for essentially the whole 115s sampling
window. `02-server-results.md` reports effective concurrency of **24.3** for the same
50-user run (via Little's Law) — about 6x the slot occupancy.

The two numbers do not disagree, they measure different things, same as the earlier
CPU-only run: `n_busy_slots_per_decode` is server-side occupancy (hard-capped at 4,
the `--parallel` limit) and effective concurrency counts every request in flight from
the client's side, including the 46 sitting in `requests_deferred` waiting for a free
slot. I trust `n_busy_slots_per_decode` for "is the GPU doing useful work" (yes, 100%
of capacity, essentially the whole run) and effective concurrency for "how deep is the
queue" (6x the slot count). Unlike the CPU-only run, this GPU run had **0% failures**
at 50 users (`02-server-results.md`) — the server stayed fully saturated but never
timed clients out, it just made every request wait its turn in a queue behind 4 GPU
decode slots that were always full.
