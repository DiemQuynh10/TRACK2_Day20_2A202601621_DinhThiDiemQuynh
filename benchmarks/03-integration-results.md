# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 8612.1 | 8612.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 7748.9 | 7749.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 7477.5 | 7477.6 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7946.2** · total **7946.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC**: stub — localhost only, no cluster or Compose stack.
- **N17 Data pipeline**: stub — `TOY_DOCS`, an in-memory Python list, stands in for a
  real ingestion/batch job.
- **N18 Lakehouse**: stub — the toy dict, no SQLite/Delta/Iceberg table behind it.
- **N19 Vector + features**: stub — retrieval is keyword overlap (no embedding server
  running), which is exactly why `embed` reads 0.0ms rather than a real embedding cost.
- **N20 Serving**: real — `llama-server` (llama.cpp `b10488`), the same server used for
  every other track in this lab.

The dominant stage — LLM at 100% of total, ~7.9s mean — is exactly what I expected and
matches the base-track finding: this CPU decodes at ~13 tok/s (from `make bench`/`make
tune`), so producing even a short ~20-25 token answer costs seconds, while retrieve
(keyword overlap over 3 toy docs) costs 0.1ms and embed costs 0ms because there's no
embedding call happening at all — it's a fallback, not a fast real stage. If I had to
halve this pipeline's latency, I'd attack the LLM stage, specifically prefill: prefill
here is 3.1-4.1s of the 7.5-8.6s total (well over a third), driven by the system
prompt + retrieved-context tokens being reprocessed on every call. Keeping the system
prompt byte-identical across calls (already true here) enables llama.cpp's prompt
cache to reuse that prefix — the real next step would be to actually exercise the
cache across repeated calls (this run reset it each query) and to trim retrieved
context to only what's needed, since prefill token count is the lever that scales
with what N19 retrieves, not with the LLM itself.
