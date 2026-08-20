# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trịnh Đắc Vụ
**Cohort:** A20-K4
**Ngày submit:** 20/08/2026

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 10
- **CPU:** AMD Ryzen 5 4600H with Radeon Graphics
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2
- **RAM:** 7.4 GB
- **Accelerator:** NVIDIA GeForce GTX 1650 Ti
- **llama.cpp asset đã tải:** llama-b10488-bin-win-cuda-12.4-x64.zip
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi

**Setup story** (≤ 80 chữ): Lấy prebuilt binary của llama.cpp hỗ trợ CUDA và tải model Qwen3.5 0.8B (phù hợp RAM < 8GB). Quá trình setup diễn ra suôn sẻ, sau đó tôi đổi mã hóa file `lab.ps1` sang UTF-8 với BOM để khắc phục các lỗi parser cú pháp trên PowerShell Windows 5.1.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| ------------ | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| Q4_K_M       |      0.50 |    104415 |         393 / 580 |        9.7 / 17.0 |    993 / 1649 / 1649 |          103.5 |
| UD-Q2_K_XL   |      0.39 |      6118 |         446 / 761 |       11.9 / 16.4 |   1298 / 1715 / 1715 |           84.3 |

**Quan sát** (≤ 60 chữ): Bản 2-bit (UD-Q2_K_XL) không đáng dùng. Nó chậm hơn bản 4-bit 1.23 lần (84.3 so với 103.5 tok/s). Vì model Qwen3.5 0.8B rất nhỏ nằm trọn trong VRAM nên không nghẽn băng thông bộ nhớ mà nghẽn tính toán. Chi phí giải nén ma trận của 2-bit tốn tài nguyên GPU nhiều hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| ----: | ---: | -------: | -------: | -------: | ---------------: | -------: |
|    10 | 1.67 |     4500 |     9600 |    11000 |              8.4 |    10.4% |
|    50 | 1.45 |    30000 |    38000 |    40000 |             38.3 |     0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.87×
- **P95 tăng:** 3.96×
- **Effective concurrency ở 50 users:** 38.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.92 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa tại mức khoảng 10 users. Bằng chứng là ở 10 users, Effective Concurrency (8.4) đã vượt quá 4 slots. Khi lên 50 users, RPS giảm từ 1.67 xuống 1.45 còn P95 tăng vọt lên 38s, chứng tỏ tải thêm vào trở thành queue time. Để tăng goodput, tôi sẽ đổi `--parallel` lên 8 hoặc 16 trước vì GPU dư VRAM cho mô hình Qwen3.5 0.8B.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day                   | Piece            | Real hay stub? |
| --------------------- | ---------------- | -------------- |
| N16 Cloud/IaC         | `LocalStack`   | stub           |
| N17 Data pipeline     | `dlt`          | stub           |
| N18 Lakehouse         | `DuckDB`       | stub           |
| N19 Vector + features | `SQLite`       | stub           |
| N20 Serving           | `llama-server` | real           |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 3538.3 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck hoàn toàn ở giai đoạn LLM (chiếm ~100%), đúng như kỳ vọng vì mô hình deep learning yêu cầu tính toán nặng hơn tìm kiếm từ khóa. Để giảm độ trễ pipeline đi 2 lần, tôi sẽ tối ưu hóa LLM: giảm `max_tokens`, bật prompt caching (`--prompt-cache`), hoặc áp dụng speculative decoding.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Đổi quantization từ UD-Q2_K_XL lên Q4_K_M

```
before:  84.3 tok/s
after:   103.5 tok/s
speedup: 1.23x
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

* Sự thay đổi từ bản UD-Q2_K_XL (2-bit) lên Q4_K_M (4-bit) mang lại mức cải thiện hiệu năng rõ rệt (1.23x) dù dung lượng mô hình tăng từ 0.39 GB lên 0.50 GB. Hiện tượng này trái ngược với kỳ vọng thông thường (mô hình nhẹ hơn thì chạy nhanh hơn) nhưng hoàn toàn hợp lý trong điều kiện phần cứng và kích thước mô hình này.
* Vì mô hình Qwen3.5 0.8B cực kỳ nhỏ, nó được tải và chạy hoàn toàn trên VRAM của GPU GTX 1650 Ti (ngl=99) mà không bị giới hạn bởi băng thông bộ nhớ RAM (memory-bandwidth bound). Do đó, nút thắt chuyển sang năng lực tính toán và chi phí giải nén ma trận (dequantization overhead) trên các nhân CUDA. Bản nén 2-bit yêu cầu các phép toán giải nén/dịch bit phức tạp hơn để đưa về float16, tiêu tốn nhiều chu kỳ xử lý của GPU hơn là lượng truyền tải dữ liệu mà nó tiết kiệm được. Do đó, chuyển sang bản 4-bit giúp tăng tốc độ sinh token lên đáng kể.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 (sweep-gpu) + B5 (C8 Semantic Cache)

**Numbers:**

- **B2 (GPU offload sweep):**

```
before:  7.1 tok/s (-ngl 0, CPU-only)
after:   130.1 tok/s (-ngl 99, GPU full offload)
speedup: 18.35x
```

- **B5 (C8 Semantic Cache):**

```
before:  8 LLM calls (32.4s latency)
after:   1 LLM call  (4.05s latency)
speedup: 8.0x (7/8 hits, 87% LLM calls saved)
```

**Điều này nói lên gì mà deck chưa nói:**

1. **Về B2 (GPU offload sweep):** Sự cải thiện tốc độ (18.35x) từ 7.1 lên 130.1 tok/s cho thấy GPU CUDA (GTX 1650 Ti) có sức mạnh tính toán vượt trội hơn hẳn CPU trong các phép toán giải mã (decode). Đồ thị tăng tốc tuyến tính và đi ngang hoàn toàn từ `-ngl 32` đến `-ngl 99` chứng minh rằng mô hình Qwen3.5 0.8B (chỉ có 28 layers) đã được nạp trọn vẹn lên VRAM của GPU. Khi được offload 100%, hệ thống tránh được việc trung chuyển dữ liệu ma trận qua băng thông PCIe vốn là nút thắt cổ chai lớn giữa CPU và GPU.
2. **Về B5 (C8 Semantic Cache):** Thực nghiệm cho thấy rủi ro lớn nhất của Semantic Cache là hiện tượng **False Hit** khi sử dụng các mô hình nhỏ (Qwen3.5 0.8B) ở chế độ pooling (`serve-embed`) làm embedder. Với ngưỡng threshold mặc định `0.80`, câu hỏi không liên quan như *"Explain TTFT and TPOT."* vẫn bị so khớp nhầm (similarity = 0.86) vào câu trả lời của *"What is goodput at SLO?"*. Điều này xảy ra do biểu diễn vector (sentence embeddings) của chat model rất yếu (weak embedder), khiến các câu hỏi bị co cụm trong dải độ tương đồng hẹp (0.85 - 0.89). Nếu nâng threshold lên `0.90` để tránh False Hit, toàn bộ các paraphrase đúng cũng bị coi là **False Miss** (tỉ lệ hit rate về 0%). Kết luận rút ra là semantic cache bắt buộc phải đi kèm với một embedding model chuyên dụng (như BGE-M3 hay Qwen3-Embedding) để phân tách rõ ràng ranh giới giữa các câu hỏi khác nghĩa.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Gemma 4 nặng hơn dự kiến nên không chạy được.

---

## 8. Self-check trước khi push

- [X] `hardware.json` committed
- [X] `models/active.json` committed
- [X] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [X] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [X] `benchmarks/02-server-results.md` committed (`make load-report`)
- [X] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [X] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [X] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [X] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
  đã được thay bằng nhận xét của bạn
- [X] 5 screenshots trong `submission/screenshots/`
- [X] `make verify` → **exit 0**
- [X] Repo GitHub ở chế độ **public**
- [X] Đã paste public URL vào VinUni LMS
- [X] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
