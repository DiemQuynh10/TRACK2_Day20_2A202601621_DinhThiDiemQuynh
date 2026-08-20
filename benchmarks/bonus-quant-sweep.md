# Bonus - Quantization sweep (Gemma 4 E2B, Unsloth Dynamic ladder)

Host `Windows-AMD64` · llama.cpp `b10488` ·
`threads=6` `ngl=0` · metric `tg128`

| Quantization | Size (GB) | tg128 (tok/s) | vs UD-Q4_K_XL | tok/s per GB |
|:--|--:|--:|--:|--:|
| UD-Q2_K_XL | 2.24 | 11.3 | 0.90x | 5.1 |
| UD-Q4_K_XL | 2.97 | 12.6 | 1.00x | 4.3 |
| UD-Q6_K_XL | 4.39 | 10.1 | 0.80x | 2.3 |

Decode is memory-bandwidth-bound, so fewer bytes per weight usually means more
tokens per second -- the "tok/s per GB" column shows how much of that you are
actually getting back per gigabyte spent.

Speed is only half the trade. The other half is quality, and no benchmark here
measures it. Serve two of these (`make serve` and
`.venv/bin/python labs/02-serve/serve.py --compare`) and ask each the same three questions
before you claim a winner.

## Your finding

The curve is not monotonic: `UD-Q4_K_XL` is the **fastest** point (12.6 tok/s), not the
endpoints. Both `UD-Q2_K_XL` (11.3) and `UD-Q6_K_XL` (10.1) are slower — this confirms
the base-track finding (`01-quickstart-results.md`) from a second angle. My CPU is
compute-limited, so there's a sweet spot in the middle: `UD-Q2_K_XL` pays a
dequantization-work tax for its aggressive packing, while `UD-Q6_K_XL` pays a
raw-bytes-moved tax for its higher precision. `UD-Q4_K_XL` is the point where neither
tax dominates, which is also visible in "tok/s per GB" bottoming out hardest at Q6
(2.3) — that extra precision costs the most per byte for the least speed return.

I would ship **`UD-Q4_K_XL`** on this machine — it's both the fastest and the default,
so there's no trade-off to make here at all: unlike a typical memory-bandwidth-bound
machine where dropping bits buys real speed at a quality cost, on this compute-bound
CPU the "efficient" choice and the "high quality" choice are the same point. I tested
the same arithmetic-reasoning prompt ("60km in 45min -> km/h, show work") on all three:
`UD-Q2_K_XL`, `UD-Q4_K_XL`, and `UD-Q6_K_XL` all produced essentially identical, correct
step-by-step reasoning at `temperature=0` — I did not observe quality breaking down
anywhere on this ladder for a simple math/factual prompt. That doesn't mean quality is
identical in general (harder reasoning or long-context tasks could separate them), but
for this machine and this workload, `UD-Q4_K_XL` dominates on every axis I measured.
