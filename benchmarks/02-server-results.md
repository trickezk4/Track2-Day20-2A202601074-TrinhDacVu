# 02 - Serve: load test + saturation reading

Host `Windows-AMD64`   llama.cpp `b10488`  
`--parallel 4`   `ctx=2048`   `threads=6`  
`ngl=99`

| Users | Reqs |  RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
| :---- | ---: | ---: | -------: | -------: | -------: | ---------------: | -------: |
| 10    |   96 | 1.67 |     4500 |     9600 |    11000 |              8.4 |    10.4% |
| 50    |   84 | 1.45 |    30000 |    38000 |    40000 |             38.3 |     0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users         |                                                           |
| :-------------------------------- | --------------------------------------------------------: |
| Offered load                      |                                                        5x |
| Throughput actually delivered     |                           **0.87x** (17% of linear) |
| P95 latency                       |                                           **3.96x** |
| Effective concurrency at 50 users | 38.3 vs`--parallel 4` slots (occupancy/slot ratio 9.59) |

**Saturated.** Throughput delivered only 0.87x for 5x the offered load, and effective concurrency (38.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.87x while P95 moved 3.96x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

_Where does your server saturate, and what is the evidence? Name the number that
convinced you. Then say what you would change first to raise goodput at your SLO --
and why that knob and not another._

* Server của tôi saturated tại mức khoảng 10 users hoặc thấp hơn. Tại 10 users, Effective Concurrency (8.4) đã vượt quá giới hạn 4 slots xử lý của hệ thống và bắt đầu có 10.4% request bị lỗi. Khi tải tăng lên 50 users, thông lượng RPS thậm chí bị giảm đi (từ 1.67 xuống 1.45 RPS) trong khi độ trễ P95 tăng vọt gần 4 lần (từ 9.6s lên 38s). Điều này chứng tỏ lượng tải thêm vào chỉ làm tăng thời gian chờ ở hàng đợi.
* Để tăng goodput theo mục tiêu SLO, tôi sẽ tăng số lượng slot chạy song song bằng tham số `--parallel` (ví dụ lên 8 hoặc 16 slots). Do mô hình Qwen3.5 0.8B rất nhẹ (~500MB), bộ nhớ VRAM của GPU GTX 1650 Ti của tôi hoàn toàn đủ chỗ cho thêm KV cache của các slot mới, giúp server xử lý được nhiều request đồng thời hơn và giảm thiểu nghẽn hàng đợi.
