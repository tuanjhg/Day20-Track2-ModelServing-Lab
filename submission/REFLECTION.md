# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Khổng Mạnh Tuấn 
**ID:** 2A202600086
**Cohort:** _A20-K1_
**Ngày submit:** _2026-05-06_

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Windows 11 (WSL2 - Linux 6.6.87.2-microsoft-standard-WSL2)
- **CPU:** AMD Ryzen 7 6800H with Radeon Graphics
- **Cores:** 16 physical / 16 logical (detect-hardware reported)
- **CPU extensions:** AVX2
- **RAM:** 9.6 GB (Allocated to WSL)
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU 4GB
- **llama.cpp backend đã chọn:** CUDA
- **Recommended model tier:** Qwen2.5-1.5B-Instruct (Q4_K_M)

**Setup story** (≤ 80 chữ): Chạy trên WSL2 Windows 11. Cần cài đặt CUDA Toolkit cho WSL và build llama.cpp từ source với flag `GGML_CUDA=on` để tận dụng GPU RTX 3050. Phải cấu hình lại environment path để nhận diện đúng thư viện CUDA.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|---:|---:|---:|---:|---:|
| qwen2.5-1.5b-instruct-q4_k_m.gguf | 1656 | 74 / 120 | 28.8 / 33.8 | 1888 / 2185 / 2221 | 34.7 |
| qwen2.5-1.5b-instruct-q2_k.gguf | 6857 | 98 / 193 | 26.3 / 36.0 | 1750 / 2378 / 2645 | 38.1 |


**Một quan sát** (≤ 50 chữ): Q4_K_M chậm hơn Q2_K khoảng 35% nhưng chất lượng phản hồi tốt hơn rõ rệt. Với GPU 4GB, tốc độ 32 tok/s của Q4_K_M là quá đủ cho trải nghiệm thời gian thực. Q2_K chỉ nên dùng khi RAM cực kỳ hạn chế.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | ~0.1 | 6800 | 41000 | 41000 | 0 |
| 50 | ~0.05 | 25000 | 45000 | 45000 | 0 |

**KV-cache observation** (từ `record-metrics.py`): peak `llamacpp:kv_cache_usage_ratio` ở concurrency 50 = _0.45_ (ước tính), nghĩa là server vẫn còn dư bộ nhớ cache nhưng latency tăng vọt do GPU tính toán bị nghẽn (compute-bound) khi xử lý nhiều request song song trên 4GB VRAM.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** docker-compose
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: ~0.1 ms
- retrieve: 0.1 ms
- llama-server: 1944 - 8189 ms

**Reflection** (≤ 60 chữ): Bottleneck nằm hoàn toàn ở llama-server (chiếm >99% thời gian). Retrieve cực nhanh do tập dữ liệu nhỏ. LLM generation tốn nhiều thời gian nhất, đặc biệt khi prompt dài (RAG), cho thấy việc tối ưu serving là then chốt.

---

## 5. Bonus — The single change that mattered most

**Change:** Build llama.cpp với hỗ trợ CUDA (`-DGGML_CUDA=on`) và offload toàn bộ layers lên GPU RTX 3050 (`-ngl 99`).

**Before vs after** (paste 2-3 dòng từ sweep output):

```
before: (CPU only) ~8.5 tok/s 
after:  (CUDA offload) 32.3 tok/s
speedup: ~3.8×
```

**Tại sao nó work** (1–2 đoạn ngắn — đây là phần grader đọc kỹ nhất):

Việc di chuyển toàn bộ tính toán tensor từ CPU sang GPU NVIDIA RTX 3050 tạo ra sự khác biệt lớn nhất. Mặc dù CPU Ryzen 7 6800H khá mạnh, nhưng kiến trúc của nó không được tối ưu cho các phép nhân ma trận mật độ cao của LLM như hàng nghìn nhân CUDA trên GPU.

GPU có băng thông bộ nhớ (memory bandwidth) cao hơn nhiều so với RAM hệ thống, giúp việc truy xuất trọng số mô hình nhanh hơn. Với model Qwen 1.5B, toàn bộ trọng số nằm gọn trong 4GB VRAM, cho phép tận dụng tối đa sức mạnh tính toán song song, giảm thiểu độ trễ decode xuống mức cực thấp.

---

## 6. (Optional) Điều ngạc nhiên nhất

Khá ngạc nhiên khi thấy model 1.5B (Q4_K_M) chạy rất "thông minh" và trả lời đúng trọng tâm các câu hỏi về PagedAttention hay Disaggregated Serving mặc dù kích thước mô hình rất nhỏ. Tốc độ vượt mức 30 tok/s trên một GPU laptop tầm trung cũng là một điểm cộng lớn.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [x] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
