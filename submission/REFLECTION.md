# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Trịnh Đắc Vũ
**Cohort:** A20
**Ngày submit:** 2026-08-20

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
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 104415 | 393 / 580 | 9.7 / 17.0 | 993 / 1649 / 1649 | 103.5 |
| UD-Q2_K_XL | 0.39 | 6118 | 446 / 761 | 11.9 / 16.4 | 1298 / 1715 / 1715 | 84.3 |

**Quan sát** (≤ 60 chữ): Bản 2-bit (UD-Q2_K_XL) không đáng dùng. Nó chậm hơn bản 4-bit 1.23 lần (84.3 so với 103.5 tok/s). Vì model Qwen3.5 0.8B rất nhỏ nằm trọn trong VRAM nên không nghẽn băng thông bộ nhớ mà nghẽn tính toán. Chi phí giải nén ma trận của 2-bit tốn tài nguyên GPU nhiều hơn.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.67 | 4500 | 9600 | 11000 | 8.4 | 10.4% |
| 50 | 1.45 | 30000 | 38000 | 40000 | 38.3 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.87×
- **P95 tăng:** 3.96×
- **Effective concurrency ở 50 users:** 38.3 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.92 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa tại mức khoảng 10 users. Bằng chứng là ở 10 users, Effective Concurrency (8.4) đã vượt quá 4 slots. Khi lên 50 users, RPS giảm từ 1.67 xuống 1.45 còn P95 tăng vọt lên 38s, chứng tỏ tải thêm vào trở thành queue time. Để tăng goodput, tôi sẽ đổi `--parallel` lên 8 hoặc 16 trước vì GPU dư VRAM cho mô hình Qwen3.5 0.8B.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | stub |
| N17 Data pipeline | stub |
| N18 Lakehouse | stub |
| N19 Vector + features | stub |
| N20 Serving | `llama-server` | real |

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

Sự thay đổi từ bản UD-Q2_K_XL (2-bit) lên Q4_K_M (4-bit) mang lại mức cải thiện hiệu năng rõ rệt (1.23x) dù dung lượng mô hình tăng từ 0.39 GB lên 0.50 GB. Hiện tượng này trái ngược với kỳ vọng thông thường (mô hình nhẹ hơn thì chạy nhanh hơn) nhưng hoàn toàn hợp lý trong điều kiện phần cứng và kích thước mô hình này.

Vì mô hình Qwen3.5 0.8B cực kỳ nhỏ, nó được tải và chạy hoàn toàn trên VRAM của GPU GTX 1650 Ti (ngl=99) mà không bị giới hạn bởi băng thông bộ nhớ RAM (memory-bandwidth bound). Do đó, nút thắt chuyển sang năng lực tính toán và chi phí giải nén ma trận (dequantization overhead) trên các nhân CUDA. Bản nén 2-bit yêu cầu các phép toán giải nén/dịch bit phức tạp hơn để đưa về float16, tiêu tốn nhiều chu kỳ xử lý của GPU hơn là lượng truyền tải dữ liệu mà nó tiết kiệm được. Do đó, chuyển sang bản 4-bit giúp tăng tốc độ sinh token lên đáng kể.

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
