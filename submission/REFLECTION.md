# Reflection — Lab 20 (Personal Report)

> **Đây là báo cáo cá nhân.** Mỗi học viên chạy lab trên laptop của mình, với spec của mình. Số liệu của bạn không so sánh được với bạn cùng lớp — chỉ so sánh **before vs after trên chính máy bạn**. Grade rubric tính theo độ rõ ràng của setup + tuning của bạn, không phải tốc độ tuyệt đối.

---

**Họ Tên:** Cao Việt Hoàng
**Cohort:** A20-K1
**Ngày submit:** 2026-06-24

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** macOS 14 (macOS)
- **CPU:** Apple M1 Pro
- **Cores:** 8 physical / 8 logical
- **CPU extensions:** —
- **RAM:** 16.0 GB
- **Accelerator:** Apple Metal
- **llama.cpp backend đã chọn:** Metal
- **Recommended model tier:** Llama-3.2-3B-Instruct

**Setup story** (≤ 80 chữ): những gì cần thay đổi để lab chạy được trên máy bạn:

Phải cập nhật mã trong script `00-setup/download-model.py` để tải file `IQ3_M` thay cho `Q2_K` do file `Q2_K` không còn tồn tại trên HuggingFace. Đồng thời phải kích hoạt môi trường ảo `.venv` thủ công bằng `source .venv/bin/activate` trước khi chạy server để tránh lỗi không tìm thấy `python`.

---

## 2. Track 01 — Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| Llama-3.2-3B-Instruct-Q4_K_M.gguf | 12150 | 75 / 178 | 19.1 / 19.1 | 1279 / 1381 / 1406 | 52.3 |
| Llama-3.2-3B-Instruct-IQ3_M.gguf (thay thế Q2_K) | 1138 | 78 / 109 | 19.7 / 19.7 | 1319 / 1349 / 1360 | 50.8 |

**Một quan sát** (≤ 50 chữ): Q4_K_M vs Q2_K trên máy bạn — số liệu nói gì? Quality đáng đánh đổi không?

Tốc độ sinh token (Decode rate) của Q4_K_M và bản nén sâu IQ3_M (đều ~50-52 tok/s) không chênh lệch quá nhiều. TTFT của Q4_K_M nhanh hơn một chút, có thể do chip Apple M1 Pro có băng thông bộ nhớ tốt nên Q4_K_M vẫn chạy rất mượt và ưu việt hơn về chất lượng văn bản so với IQ3_M.

---

## 3. Track 02 — llama-server load test

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | 0.53 | 13000 | 22000 | 22000 | 0 |
| 50 | 0.52 | 18000 | 35000 | 39000 | 0 |

**Batching observation** (từ `record-metrics.py`):

Khi tải tăng từ 10 lên 50 user, RPS tổng không đổi (đạt ngưỡng xử lý của phần cứng máy) nhưng độ trễ P95 tăng vọt từ 22s lên 35s. Điều này cho thấy hệ thống bắt đầu vào trạng thái hàng đợi (queueing) do đã bão hòa tài nguyên tính toán/memory.

---

## 4. Track 03 — Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only
- **N17 (Data pipeline):** stub: in-memory dict
- **N18 (Lakehouse):** stub: SQLite
- **N19 (Vector + Feature Store):** stub: TOY_DOCS

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llama-server: ~ 1016.4 ms / 3782.0 ms

**Reflection** (≤ 60 chữ): bottleneck nằm ở đâu? Có khớp với kỳ vọng không?

Khớp hoàn toàn với kỳ vọng: Phần lớn thời gian của toàn pipeline (lên tới hàng nghìn ms) nằm ở khâu gọi model (llama-server), trong khi các khâu xử lý nội bộ, embedding (mock), retrieve cực nhanh. Việc tối ưu hóa model serving là mấu chốt để cải thiện tốc độ pipeline RAG.

---

## 5. Bonus — The single change that mattered most

> **Phần này bạn hãy hoàn thành bài test Bonus nếu muốn lấy điểm thưởng, hoặc giữ nguyên mô tả ngắn về CPU threads.**

**Change:** Thay đổi tùy chọn `n_threads` để xem tác động của luồng xử lý CPU trên chip M1 Pro.

**Before vs after** (paste 2-3 dòng từ sweep output):

```
(Bạn cần chạy make sweep-thread và copy output vào đây nếu muốn lấy điểm Bonus. Phần này mình để trống mẫu.)
```

**Tại sao nó work**:

Trên chip M1 Pro, số lượng `n_threads` tối ưu thường khớp với số lượng Performance Cores thực tế. Nếu đặt số thread lớn hơn cả số core vật lý, tốc độ decode không những không tăng mà còn có nguy cơ giảm xuống do context switching và giới hạn về Memory Bandwidth của hệ thống.

---

## 6. (Optional) Điều ngạc nhiên nhất

Thật ngạc nhiên khi thấy model kích thước 3B có thể load và phục vụ trên Mac M1 Pro mượt mà với tốc độ >50 token/giây mà không cần setup phần mềm rườm rà.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit (hoặc paste path snapshot vào section 1)
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-results.md` (hoặc CSV từ `record-metrics.py`) đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (ít nhất 1 sweep)
- [x] Ít nhất 6 screenshots trong `submission/screenshots/` (xem `submission/screenshots/README.md`)
- [x] `make verify` exit 0 (chạy ngay trước khi push)
- [x] Repo trên GitHub ở chế độ **public**
- [x] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Nếu private, grader không xem được → 0 điểm.
