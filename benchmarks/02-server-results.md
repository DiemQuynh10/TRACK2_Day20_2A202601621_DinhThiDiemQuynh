# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 9 | 0.16 | 38000 | 55000 | 55000 | 5.1 | 0.0% |
| 50 | 15 | 0.27 | 30000 | 56000 | 56000 | 8.5 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.65x** (33% of linear) |
| P95 latency | **1.02x** |
| Effective concurrency at 50 users | 8.5 vs `--parallel 4` slots (occupancy/slot ratio 2.11) |

**Saturated.** Throughput delivered only 1.65x for 5x the offered load, and effective concurrency (8.5) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 grew no faster than throughput (1.02x vs 1.65x), so this server still has headroom at 50 users.

> **Small sample.** Only 9 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

The server is already queuing at 10 users, but 50 users doesn't make things
dramatically worse — throughput actually scaled up 1.65x (not down) and P95 grew only
1.02x (55000 -> 56000ms). The convincing number here is effective concurrency vs slots:
5.1 in-flight at 10 users and 8.5 at 50 users, both already above `--parallel 4` — the
server was queuing requests even at the lighter load, so there wasn't a sharp "before
vs after" saturation cliff between these two runs; it was already past the knee at 10
users. The `make metrics` run corroborates this: peak `n_busy_slots_per_decode` =
3.81 of 4 (95%) during the 50-user run — essentially every decode step used all 4
slots, confirming the server was running at its compute ceiling, not idle.

The samples here are small (9 and 15 completed requests in 60s), so I trust the
slot-occupancy gauge (3.81/4, direct evidence from the server) more than the exact
P95/RPS numbers from only 9-15 requests, which are indicative rather than statistically
solid. Both readings agree on the same conclusion though: this CPU-only box is
compute/bandwidth-bound at roughly 4 concurrent decodes regardless of how many more
users pile on.

To raise goodput at a fixed SLO, the first knob I'd change is **not** `--parallel` —
more slots would only split the same fixed compute/bandwidth budget into smaller
pieces, since the busy-slots gauge already shows the ceiling is compute, not slot
count. I'd cut per-request decode cost instead: lower `max_tokens` for the SLO-critical
path, or move to a smaller/faster model — the `make tune` sweep already found this
CPU's per-thread ceiling (13.0 tok/s at 6 threads), so there's no thread-count win left
to spend; the only lever that moves goodput at a fixed SLO is making each request
cheaper.
