# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 14 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.81 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1520 |

Highest sampled value was **3.81 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.81 of 4 slots (95%)** — essentially every slot busy on
essentially every sampled decode step. `02-server-results.md` reports effective
concurrency of 8.5 at 50 users, i.e. Little's Law says ~8.5 requests were "in flight"
on average. These two numbers *disagree* (8.5 > 4), and I trust the slot gauge
(3.81/4) over the Little's-Law figure, for a reason the report itself flags: Little's
Law (`RPS x avg latency`) counts a request as "in flight" from the moment it's queued,
not from the moment it starts decoding, so it necessarily overcounts against the number
of slots actually doing work — queued-but-not-yet-served requests inflate the
concurrency estimate without touching a slot. The server-side gauge is ground truth for
what the scheduler is actually doing; 3.81/4 slots busy means the server was running
essentially at its `--parallel 4` ceiling, with the extra ~4.7 "concurrent" requests
from Little's Law sitting in `requests_deferred` (46, non-zero) rather than being
served in parallel. So: batch width confirms the server is compute-saturated at its
slot limit; the concurrency figure's excess over 4 is queue depth, not real
parallelism.
