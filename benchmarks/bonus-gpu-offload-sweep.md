# Bonus - GPU offload sweep

Host `Windows-AMD64` · backend(s) `nvidia_cuda, vulkan` ·
llama.cpp `b10488` · `threads=6` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 7.1 | 1.00x | 5% |
| 8 | 16.7 | 2.36x | 13% |
| 16 | 30.3 | 4.28x | 23% |
| 24 | 86.7 | 12.23x | 67% |
| 32 | 124.7 | 17.59x | 96% |
| 99 | 130.1 | 18.35x | 100% |

Best: `-ngl 99` at 130.1 tok/s
-- 18.35x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Trên máy của tôi, chế độ offload hoàn toàn (`-ngl 99`) đạt hiệu năng tốt nhất (130.1 tok/s, nhanh gấp 18.35 lần so với CPU-only). Mô hình Qwen3.5 0.8B rất nhỏ (~500MB) nên toàn bộ các lớp (layers) và KV cache hoàn toàn nằm gọn trong 4GB VRAM của GPU GTX 1650 Ti mà không gặp bất kỳ giới hạn nào về bộ nhớ.

Tốc độ sinh token tăng trưởng gần như tuyến tính khi tăng số lượng layer được đưa lên GPU. Đồ thị bắt đầu đi ngang từ mức `-ngl 32` đến `-ngl 99`, điều này chứng tỏ mô hình đã hết layer để chuyển giao (mô hình Qwen3.5 0.8B chỉ có 28 layers). Chạy full offload là tối ưu nhất vì nó tránh hoàn toàn việc truyền tải dữ liệu ma trận qua băng thông PCIe chậm chạp giữa CPU và GPU trong bước decode.
