# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 6898 | 1121 / 1237 | 79.3 / 89.5 | 6147 / 6771 / 6771 | 12.6 |
| UD-Q2_K_XL | 2.24 | 6765 | 1662 / 1903 | 88.5 / 89.5 | 7212 / 7468 / 7468 | 11.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.12x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

On this machine (Ryzen 5 5500U, 6 physical cores, CPU-only — `ngl=0`), `UD-Q2_K_XL` is
**not** worth it. It is 0.73 GB smaller (2.24 GB vs 2.97 GB) but decodes 1.12x *slower*
(TPOT P50 88.5ms vs 79.3ms, i.e. 11.3 vs 12.6 tok/s) and its TTFT is ~48% worse (1662ms
vs 1121ms P50). This machine is compute-limited, not memory-bandwidth-limited: only 6
physical cores and no GPU offload, so the extra dequantization work UD-Q2_K_XL's more
aggressive packing requires costs more cycles than the smaller footprint saves in bytes
moved. The classic "fewer bits = faster" story only holds when decode is bottlenecked
on memory bandwidth (e.g. many cores / a GPU chewing through a huge parameter stream);
here the CPU itself is the bottleneck, so shrinking the bytes doesn't help and the extra
unpacking work actively hurts.

I asked both quantizations the same two questions with `--reasoning off`, temp=0,
directly against `/v1/chat/completions`: a factual one ("what is a hash table and why
are lookups fast") and an arithmetic one ("60 km in 45 min -> km/h, show the work").
For the factual question both answers were essentially equivalent in correctness and
clarity — no visible quality loss from UD-Q2_K_XL. So on this machine UD-Q2_K_XL buys
you *nothing*: it is both slower and smaller-but-not-meaningfully-so, with quality that
is a wash on simple prompts. I'd stick with `UD-Q4_K_XL` (the default) unless disk/RAM
were the binding constraint rather than latency.
