# 02 - Continuous batching under load (u50)

Host `Windows-AMD64`   `--parallel 4`   14 samples over
60s at 2.0s intervals   raw CSV: `02-server-metrics-u50.csv`

| Gauge                                    |                              Peak observed |
| :--------------------------------------- | -----------------------------------------: |
| `n_busy_slots_per_decode` (avg/decode) |                      3.92 of 4 slots (98%) |
| `requests_processing`                  |                                          0 |
| `requests_deferred`                    |                                          0 |
| `kv_cache_usage_ratio`                 | n/a   not exported by llama.cpp`b10488` |
| `tokens_predicted_total` (final)       |                                      17026 |

Highest sampled value was **3.92 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` stayed at zero: every request found a free slot on arrival.

## My observation

_What was the peak batch width, and does it match the effective concurrency in
`02-server-results.md`? If the two disagree, which do you trust and why?_

* Độ rộng batch đạt đỉnh là 3.92 / 4 slots, lệch lớn so với mức effective concurrency (38.3). Hai con số này khác nhau vì đo lường hai thực thể khác nhau:
  - **n_busy_slots_per_decode (3.92)**: Đo thực tế số lượng slot đang được xử lý song song trong lõi tính toán của engine (bị giới hạn cứng bởi `--parallel 4`).
  - **Effective concurrency (38.3)**: Đo tổng lượng request đang nằm trong hệ thống (gồm 4 request đang xử lý và ~34 request còn lại đang nằm chờ ở hàng đợi).
* Để đánh giá hiệu năng batching của mô hình, chỉ số `n_busy_slots_per_decode` đáng tin hơn vì nó phản ánh chính xác trạng thái hoạt động bên trong của engine, trong khi số còn lại đo  lường mức độ ùn tắc hàng đợi.
