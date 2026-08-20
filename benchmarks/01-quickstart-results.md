# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 52279 | 2630 / 2716 | 114.8 / 121.0 | 9819 / 10338 / 10338 | 8.7 |
| UD-Q2_K_XL | 2.24 | 19802 | 2711 / 2770 | 157.7 / 159.6 | 12555 / 12798 / 12798 | 6.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.38x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

With GPU offload active (`ngl=99`, NVIDIA Quadro M2200), `UD-Q2_K_XL` is still **not
worth it** on this machine — same direction as the CPU-only run, now by a wider
margin. It is 0.73 GB smaller but decodes **1.38x slower** (6.3 vs 8.7 tok/s) and its
TTFT P95 (2770 ms) is worse than Q4's (2716 ms) despite being the lighter file. This
4096 MiB Maxwell-class GPU only has room to offload the full model, not extra headroom
for the heavier 2-bit unpacking work per token; since dequantizing 2-bit blocks costs
more scalar/ALU work per weight than 4-bit, and this GPU's compute (not its 4 GB of
VRAM) is the limited resource here, Q2 loses on both ends: no bandwidth win, real
compute cost. Quality-wise the two are close on simple factual questions (tested with
`--compare` on port 8090, e.g. "why is the sky blue") — both gave correct, fluent,
non-hallucinated answers. So Q2 buys smaller disk footprint only, at a real decode-speed
and worst-case-latency cost, with no quality upside to justify it.
