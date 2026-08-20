# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đinh Thị Diễm Quỳnh
**Cohort:** K3
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Home Single Language (build 10.0.26200). `make probe`/`hardware.json` reports `Windows 10` — a known quirk of Python's `platform.release()`, which cannot tell 10 and 11 apart on some builds; the machine is actually Windows 11.
- **CPU:** AMD Ryzen 5 5500U with Radeon Graphics
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** not reported by `detect-hardware.py` on this platform — n/a
- **RAM:** 15.3 GB
- **Accelerator:** CPU only for the base track (`ngl=0`). Machine also has an NVIDIA GeForce GTX 1650 (4GB) detected by `hardware.json`, but the base track intentionally runs CPU-only per the lab design.
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-vulkan-x64.zip` (Windows Vulkan build, `b10488`)
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): PowerShell 5.1 failed to parse `lab.ps1` in my shell
(reserved-`<` parser error), so I ran the underlying Python scripts directly instead —
same commands, no `.ps1` wrapper. HuggingFace's `xet` download backend hung
indefinitely on the model fetch; fixed with `HF_HUB_DISABLE_XET=1` to force plain HTTP.
`llama-server.exe` hung/spun on startup even with `-ngl 0`, because probing the bundled
Vulkan backend against my old (2020, driver 457.34) NVIDIA driver never returned;
disabling `ggml-vulkan.dll` fixed it and the base track stayed CPU-only as intended.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 6898 | 1121 / 1237 | 79.3 / 89.5 | 6147 / 6771 / 6771 | 12.6 |
| UD-Q2_K_XL | 2.24 | 6765 | 1662 / 1903 | 88.5 / 89.5 | 7212 / 7468 / 7468 | 11.3 |

**Quan sát** (≤ 60 chữ): UD-Q2_K_XL không nhanh hơn — nó **chậm hơn 1.12x** (11.3 vs
12.6 tok/s) và TTFT tệ hơn ~48%, dù nhỏ hơn 0.73GB. Máy này compute-bound (6 core CPU,
không GPU), nên bit ít hơn không giúp gì; công dequant thêm còn tốn hơn. Đã hỏi cùng
2 câu (factual + arithmetic) trên cả hai qua `/v1/chat/completions` — chất lượng gần
như ngang nhau trên câu factual. Kết luận: **không đáng** dùng Q2 trên máy này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.16 | 38000 | 55000 | 55000 | 5.1 | 0.0% |
| 50 | 0.27 | 30000 | 56000 | 56000 | 8.5 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.65× (33% của tuyến tính)
- **P95 tăng:** 1.02×
- **Effective concurrency ở 50 users:** 8.5 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.81 / 4 slots

**Saturation reading** (≤ 80 chữ): Server đã bão hoà **ngay từ 10 users** — effective
concurrency (5.1) đã vượt 4 slots. Bằng chứng thuyết phục nhất: peak busy_slots=3.81/4
(95%) khi chạy `make metrics` cùng `make load-50` — server luôn dùng gần hết slot. Từ
10→50 users, P95 gần như không đổi (1.02×) trong khi throughput tăng 1.65× — nghĩa là
máy vẫn còn "room" ở mức tải này, saturation đến từ trần compute/slot chứ không phải
queue phình to thêm. Muốn nâng goodput@SLO, tôi sẽ giảm chi phí mỗi request (giảm
`max_tokens` hoặc dùng model nhỏ hơn) trước, **không phải** tăng `--parallel` — máy
compute-bound nên thêm slot chỉ chia sẻ cùng băng thông, không tạo throughput thật.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | localhost only | stub |
| N17 Data pipeline | `TOY_DOCS` in-memory list | stub |
| N18 Lakehouse | toy dict | stub |
| N19 Vector + features | keyword overlap (no embedding server) | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0ms
- retrieve: 0.1ms
- llm: 7946.2ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck là LLM decode, khớp kỳ vọng — máy này chỉ ~13
tok/s nên mỗi câu trả lời ngắn cũng mất giây. embed=0 vì đang fallback keyword overlap,
không phải embedding thật. Muốn giảm latency 2×, tôi sẽ tấn công **prefill** trong LLM
stage (chiếm >1/3 total): giữ system prompt cố định để tận dụng prompt cache thật sự
qua nhiều lần gọi, và cắt bớt context retrieve được.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** tăng `-t` (thread count) từ 1 lên 6 (= physical core count) khi chạy
`llama-bench` tg128 trên `UD-Q4_K_XL`

```
before:  4.6 tok/s   (-t 1)
after:   13.0 tok/s  (-t 6)
speedup: 2.83×
```

**Tại sao nó work** (1–2 đoạn):

Decode một token là một chuỗi matmul nhỏ theo từng layer với dependency chain, và với
mọi mức threads đây là workload bị chặn bởi **memory bandwidth**, không phải FLOPs:
mỗi thread cần liên tục kéo các hàng trọng số (weight rows) ra khỏi RAM/cache cho phần
matmul của nó. Từ `-t 1` đến `-t 6`, throughput tăng gần tuyến tính (2.83× cho 6× số
thread — dưới tuyến tính vì có overhead đồng bộ hoá giữa các layer) vì Ryzen 5500U có
6 core vật lý và băng thông bộ nhớ chưa bị bão hoà cho tới khi đủ 6 luồng cùng kéo dữ
liệu song song.

Điều thú vị (và khớp đúng lý thuyết deck) là đường cong **gãy đúng tại 6 — số core vật
lý**, không phải 12 (logical/SMT). `-t 12` chỉ bằng 98% của peak, và `-t 24` tụt xuống
9.4 tok/s — bằng với `-t 3`. Cơ chế: một khi 6 core vật lý đã bão hoà băng thông bộ
nhớ, thêm thread không mua thêm băng thông (2 SMT-lane trên cùng 1 core chia sẻ chung
load/store port tới RAM), mà chỉ thêm chi phí lập lịch (context-switch, tranh chấp
cache line) — nên `--threads` nên bằng **physical core count**, không phải logical,
cho một workload memory-bandwidth-bound như decode. Đây chính là bằng chứng thực tế
cho lý do đề bài nhấn mạnh "TPOT bị chặn bởi memory bandwidth chứ không phải FLOPs."

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (`make sweep-quant`, 3-way UD-Q2/Q4/Q6_K_XL) · B4 (challenge C5 "smallest
model that's still useful", using the same sweep) · B5 (`make embed-demo`, C9 embedding
serving regime). Skipped B1: no `cmake`/Visual Studio Build Tools on this machine, so
compiling from source wasn't feasible without installing a toolchain first.

**Numbers** (B2/B3, from `benchmarks/bonus-quant-sweep.md`):

```
before:  10.1 tok/s  (UD-Q6_K_XL, highest precision, most bytes)
after:   12.6 tok/s  (UD-Q4_K_XL, the default)
speedup: 1.25×
```

**Điều này nói lên gì mà deck chưa nói:**

The 3-point sweep (Q2=11.3, Q4=12.6, Q6=10.1 tok/s) traces a **non-monotonic** curve —
the fastest point is in the *middle* of the precision ladder, not either endpoint. The
deck's mental model ("fewer bits = faster, because memory bandwidth") assumes the
machine is memory-bandwidth-bound; on this CPU (6 cores, no GPU) it is not. Going down
in precision from Q4 to Q2 costs speed (more dequant work per byte moved) exactly as
much as going up in precision from Q4 to Q6 costs speed (more bytes to move per token).
Q4 sits at the point where neither cost dominates. This is a case where a machine's
bottleneck (compute vs. bandwidth) flips which direction on the quantization ladder is
"faster," and you cannot know which without measuring — the deck's heuristic is
directionally true only for bandwidth-bound hardware.

For **B4/C5**: across this ladder (Q2 -> Q6) I never found a quality failure point —
the same arithmetic-reasoning prompt at `temperature=0` produced essentially identical,
correct step-by-step answers on all three quantizations. On this machine the honest
finding is that quality did *not* break down anywhere I tested, and the choice between
Q2/Q4/Q6 is purely a speed question (Q4 wins on speed too, so it's not even a
trade-off here) — a genuinely different shape of "how far can I push quantization" than
the deck's framing of quality-vs-speed trade-off, because for this workload there was
no trade-off to make.

For **B5/C9** (`make embed-demo`, reusing the chat GGUF in pooling mode): embedding
serving showed the expected prefill-bound throughput curve — 0.3 texts/s at batch 1
rising to 1.3 texts/s at batch 16 (4.3x from batching alone, no extra hardware), because
each text is one forward pass with no KV cache and no decode loop, so static batching
is the only lever. This is the mirror image of the chat-serving result in §3, where
adding concurrency past the slot count bought nothing — here, adding batch size keeps
paying off because there's no shared decode-step bottleneck to saturate.

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Ngạc nhiên lớn nhất là `UD-Q2_K_XL` **chậm hơn** `UD-Q4_K_XL`, ngược với kỳ vọng "ít bit
hơn = nhanh hơn" — hoá ra máy này compute-bound chứ không memory-bandwidth-bound, nên
phần dequant thêm của quant nặng hơn thắng phần bytes tiết kiệm được. Ngạc nhiên thứ
hai (kỹ thuật hơn): `llama-server.exe` bị treo vô thời hạn ngay cả với `-ngl 0` vì nó
vẫn dò thiết bị Vulkan lúc khởi động, và driver NVIDIA cũ (2020) trên máy tôi khiến
lần dò đó không bao giờ trả về — phải tắt hẳn `ggml-vulkan.dll` mới chạy được.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
