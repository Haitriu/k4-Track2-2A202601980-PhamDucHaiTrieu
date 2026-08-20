# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** _<Họ Tên>_
**Cohort:** _<A20-K1 / A20-K2 / ...>_
**Ngày submit:** _<YYYY-MM-DD>_

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10 (build 10.0.19045), AMD64
- **CPU:** Intel(R) Core(TM) i7-6820HQ @ 2.70GHz
- **Cores:** 4 physical / 8 logical
- **CPU extensions:** không do `make probe` thu thập; theo spec dòng CPU (Skylake mobile) là AVX2
- **RAM:** 15.9 GB
- **Accelerator:** CPU only (thực tế). Máy có GPU NVIDIA Quadro M2200 (4096 MiB) và Vulkan,
  `make probe` đề xuất build CUDA, nhưng **CUDA offload treo vô thời hạn** lúc khởi tạo
  (xem "Setup story" bên dưới) nên toàn bộ base track chạy `ngl=0`.
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` + `cudart-llama-bin-win-cuda-12.4-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare) (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): `lab.ps1` báo lỗi parse trên Windows PowerShell 5.1 vì file
UTF-8 thiếu BOM khiến ký tự Unicode làm hỏng parser — thêm BOM là chạy được. Nghiêm
trọng hơn: `llama-server` build CUDA treo vô thời hạn khi khởi tạo GPU trên Quadro M2200
đời cũ — không crash, chỉ đứng im sau bước load model. Ép `LAB_N_GPU_LAYERS=0` (CPU-only)
thì load bình thường trong ~28s. Toàn bộ 100 điểm base không cần GPU nên không mất điểm.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 17238 | 4965 / 8749 | 319.7 / 364.7 | 24554 / 29132 / 29132 | 3.1 |
| UD-Q2_K_XL | 2.24 | 25116 | 6503 / 7516 | 385.5 / 450.7 | 30346 / 35908 / 35908 | 2.6 |

**Quan sát** (≤ 60 chữ): Q2 **không đáng dùng** trên máy này — nhẹ hơn 0.73 GB nhưng
decode **chậm hơn** 1.19× (2.6 vs 3.1 tok/s) vì máy compute-limited (4 core, không GPU),
nên chi phí dequantize 2-bit tốn hơn phần bytes tiết kiệm. Đã hỏi cùng câu ("why is
the sky blue") trên `--compare` (port 8090): chất lượng gần như tương đương, không
hallucination ở cả hai bản.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.12 | 69000 | 104000 | 104000 | 7.9 | 0.0% |
| 50 | 0.34 | 122000 | 122000 | 122000 | 35.4 | 65.5% |

- **Offered load tăng 5×, throughput thực tăng:** 2.81×
- **P95 tăng:** 1.17×
- **Effective concurrency ở 50 users:** 35.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.77 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hoà rõ ràng trước khi tới 50 users. Bằng
chứng thuyết phục nhất: **65.5% request timeout ở 50 users** (0% ở 10 users) và P50=P95=P99
đều dồn về ~122s — dấu hiệu queueing, không phải compute, vì service time thật không
lệch nhau theo phân phối bình thường. `busy_slots` đạt đỉnh 3.77/4 xác nhận engine đã
bão hoà compute. Knob đổi trước: `-t 8` thay vì `-t 4` (free 2.62× decode throughput từ
`make tune`, không tốn thêm contention) trước khi đụng tới `--parallel`.

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
- llm: 15211.9 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Đúng như kỳ vọng — decode CPU-only (~3 tok/s) áp đảo hoàn
toàn embed/retrieve vốn chỉ là tra cứu dict/list rẻ tiền trên 6 toy doc. Để giảm 2×:
tấn công decode qua `-t 8` (2.62× từ `make tune`) và giảm `max_tokens` mỗi câu trả lời,
vì cost decode tuyến tính theo số token sinh ra.

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
