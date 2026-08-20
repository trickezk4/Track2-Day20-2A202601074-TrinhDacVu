# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf`   host `Windows-AMD64`   llama.cpp `b10488`
CPU: **6 physical   12 logical** cores   `ngl=99`   metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
| :----------- | ------------: | ------: |
| 1            |         135.2 |    100% |
| 3            |         125.4 |     92% |
| 6            |         132.0 |     97% |
| 12           |         135.8 |    100% |
| 24           |         133.8 |     99% |

**Best**: `-t 12` at 135.8 tok/s
**Slowest tested**: `-t 3` at 125.4 tok/s (1.08x spread)
**Against the physical-core default** (`-t 6`, 132.0 tok/s): 1.03x

Use this in your run:

```bash
LAB_N_THREADS=12 make bench
```

## My Explaination:

_Where is the knee, and why there? If the peak sits at your physical core count
and drops above it, say what the extra threads are competing for. If your curve
does something else -- flat, or still climbing at 2x logical cores -- say that
instead and reason about why. A result that contradicts the expected shape is
worth more than one that matches it, as long as you explain it._

* Kết quả đo tốc độ sinh token dao động nhẹ từ 125.4 đến 135.8 tok/s khi thay đổi số thread => không curve. Điều này xảy ra do mô hình đã được tải hoàn toàn lên GPU GTX 1650 Ti (tham số ngl=99). GPU đảm nhận 100% việc tính toán ma trận, CPU chỉ làm nhiệm vụ kích hoạt kernel. Vì vậy, số luồng CPU (-t) không ảnh hưởng tới tốc độ sinh token và sự chênh lệch nhỏ chỉ là sai số ngẫu nhiên.
