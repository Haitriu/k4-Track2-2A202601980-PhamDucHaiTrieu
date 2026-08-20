# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Phạm Đức Hải Triều
**Cohort:** K4 (suy từ tên repo `k4-Track2-...` — báo lại nếu không đúng)
**Mã số sinh viên:** 202601980
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (build 10.0.19045), AMD64
- **CPU:** Intel(R) Core(TM) i7-6820HQ @ 2.70GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** không do `make probe` thu thập; theo spec dòng CPU (Skylake mobile) là AVX2
- **RAM:** 15.9 GB
- **Accelerator:** NVIDIA Quadro M2200 (4096 MiB), CUDA offload **ACTIVE** (`ngl=99`).
  Ban đầu chạy tự động lần đầu bị treo vô thời hạn lúc khởi tạo CUDA (nghi cold-start
  JIT/PTX compile cho kiến trúc Maxwell đời cũ, hoặc tiến trình cũ còn giữ context GPU);
  các lần chạy sau (bench, tune-server, load test, pipeline) GPU hoạt động ổn định.
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` + `cudart-llama-bin-win-cuda-12.4-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): `lab.ps1` báo lỗi parse trên Windows PowerShell 5.1 vì file
UTF-8 thiếu BOM khiến ký tự Unicode làm hỏng parser — thêm BOM là chạy được. GPU CUDA
lúc đầu treo vô thời hạn ở lần chạy tự động đầu tiên nên tôi tạm ép CPU-only để không
mất tiến độ; các lần chạy tay sau đó (`bench`, `serve`, `load test`, `pipeline`) GPU lên
bình thường (`ACTIVE`, xem screenshot `01-hardware-probe.png`) nên toàn bộ số liệu cuối
cùng trong report này dùng GPU. Phần thread-tuning (§5) vẫn đo bằng CPU vì đó là câu
hỏi riêng về threads, không phụ thuộc GPU.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 52279 | 2630 / 2716 | 114.8 / 121.0 | 9819 / 10338 / 10338 | 8.7 |
| UD-Q2_K_XL | 2.24 | 19802 | 2711 / 2770 | 157.7 / 159.6 | 12555 / 12798 / 12798 | 6.3 |

**Quan sát** (≤ 60 chữ): Với GPU (Quadro M2200) bật, Q2 vẫn **không đáng dùng** — nhẹ
hơn 0.73 GB nhưng decode **chậm hơn 1.38×** (6.3 vs 8.7 tok/s) và TTFT P95 còn tệ hơn cả
Q4. GPU 4GB đủ chứa model nhưng dequantize 2-bit tốn compute hơn 4-bit. Hỏi cùng câu
("why is the sky blue") trên `--compare`: chất lượng gần như tương đương, không
hallucination ở cả hai bản.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.43 | 18000 | 33000 | 34000 | 8.4 | 0.0% |
| 50 | 0.45 | 50000 | 105000 | 109000 | 24.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.07×
- **P95 tăng:** 3.18×
- **Effective concurrency ở 50 users:** 24.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.99 / 4 slots

**Saturation reading** (≤ 80 chữ): Với GPU, server bão hoà ở compute chứ không phải
timeout — **0% failure cả 2 mức tải**, nhưng throughput chỉ tăng 1.07× khi tải tăng 5×
(21% linear) trong khi P95 tăng 3.18×. `busy_slots` đạt đỉnh 3.99/4 (100%) ngay từ mức
10 users — 4 slot GPU đã đầy, nên phần tải thêm chỉ biến thành queue time chứ không
sinh thêm throughput. Đổi trước: tăng `--parallel` (không phải `-t`, vì bottleneck là
slot GPU chứ không phải CPU thread) để engine batch nhiều sequence hơn mỗi decode step.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng | stub — chạy 100% local, không hạ tầng cloud/IaC nào |
| N17 Data pipeline | `TOY_DOCS` | stub — danh sách Python cứng, không có ingestion pipeline |
| N18 Lakehouse | `TOY_DOCS` (in-memory) | stub — list trong RAM thay cho lakehouse/document store |
| N19 Vector + features | keyword-overlap fallback | stub — không có embedding server chạy nên `retrieve()` fallback sang so khớp từ khoá thay vì vector search |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.2 ms
- llm: 9603.7 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Đúng như kỳ vọng — decode áp đảo hoàn toàn embed/retrieve
(chỉ là tra cứu dict/list rẻ tiền trên 6 toy doc). Với GPU, pipeline nhanh hơn ~1.6× so
với bản CPU-only (9.6s vs 15.2s). Để giảm 2× nữa: tấn công decode qua `--parallel`
(pipeline hiện chạy tuần tự, không hưởng continuous batching) và giảm `max_tokens`.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** tăng `-t` (thread count) từ 1 lên 8 (`make tune` sweep, xem `benchmarks/01-tuning-tg128.md`)

```
before:  1.6 tok/s   (-t 1)
after:   4.1 tok/s   (-t 8)
speedup: 2.62×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Curve **không** peak ở physical core count (4) như deck mặc định kỳ vọng — nó tiếp tục
leo tới `-t 8` (full logical/hyperthreaded core count) rồi mới rớt ở `-t 16`. `-t 4` chỉ
đạt 96% của kết quả tốt nhất. Lý do: decode (`tg128`) trên máy CPU-only này là
compute-bound thực sự — dequantize block Q4_K cộng matmul tốn đủ công việc mỗi token
để hardware thread thứ hai trên mỗi physical core (hyperthreading) vẫn tìm được
instruction-level parallelism hữu ích để lấp, thay vì chỉ giành execution port/cache
với sibling thread. Đây khác với câu chuyện quen thuộc "hyperthreading không giúp decode
vì nó bandwidth-bound" — trên chip mobile Skylake này phần dequant/scalar work có vẻ
giữ cả hai thread mỗi core bận thay vì đói dữ liệu.

Trên `-t 16` (oversubscribe 8 logical core 2×), throughput rớt còn 81% peak: OS
scheduler giờ context-switch nhiều thread hơn số hardware context sẵn có, nên chi phí
cache residency và đồng bộ hoá thread-pool giữa các bước decode (barrier) bắt đầu tốn
hơn phần song song thêm được. Kết quả bất ngờ so với deck (peak lệch khỏi physical
core count) chính là điểm đáng nói nhất của phép đo này, không phải con số tuyệt đối.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

_(1–2 câu. Không bắt buộc, nhưng grader đọc hết.)_

_(để trống nếu bạn không làm phần này)_

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
