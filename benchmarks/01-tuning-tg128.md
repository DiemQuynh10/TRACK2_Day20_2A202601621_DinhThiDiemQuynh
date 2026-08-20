# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 4.6 | 35% |
| 3 | 9.5 | 73% |
| 6 | 13.0 | 100% |
| 12 | 12.8 | 98% |
| 24 | 9.4 | 73% |

**Best**: `-t 6` at 13.0 tok/s
**Slowest tested**: `-t 1` at 4.6 tok/s (2.84x spread)
**Against the physical-core default** (`-t 6`, 13.0 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

The knee sits exactly at `-t 6`, my physical core count, which is the textbook shape.
From 1 -> 6 threads, throughput scales almost linearly (4.6 -> 13.0 tok/s, a 2.83x
speedup for 6x the threads — sublinear because a decode step for one token is a
sequence of small per-layer matmuls with a dependency chain, so there's coordination
overhead, not pure embarrassingly-parallel work). Past 6 threads it degrades: `-t 12`
(logical core count, i.e. one thread per SMT/hyperthread) is already 2% below the
peak, and `-t 24` (4x oversubscription) drops all the way back to 9.4 tok/s, the same
level as `-t 3`.

The mechanism: decode is memory-bandwidth-bound per token (each thread is streaming
weight rows out of RAM/cache for its matmul slice), not compute-bound. The Ryzen
5500U's 6 physical cores share one memory controller and one bandwidth budget between
them. Once you have 6 threads, that budget is already saturated — a 7th+ thread doesn't
get to move more bytes/sec, it just adds scheduling and synchronization overhead
(threads finishing their slice and spinning/waiting at the barrier for the slowest
one) on top of the same bandwidth ceiling. Going to `-t 12` puts two software threads
on each physical core's two SMT lanes, which mostly contend for the same execution
units and L1/L2 cache rather than adding real parallel throughput — hence the ~flat-to-
slightly-worse result. `-t 24` makes it actively worse: with 4 OS threads scheduled
per core, the OS scheduler is context-switching and the threads are fighting each
other for cache lines, so wall-clock throughput regresses below even `-t 3`.

Practical takeaway for this machine: `--threads` should equal physical core count
(6), never logical (12) — for a bandwidth-bound workload like autoregressive decode,
SMT/hyperthreading buys nothing because the two logical threads on a core share the
same load/store ports to memory, and oversubscribing beyond that actively costs
throughput via scheduling overhead.
