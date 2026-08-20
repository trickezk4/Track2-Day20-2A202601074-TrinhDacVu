# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query                                           |      Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
| :---------------------------------------------- | ----------------------: | ---------: | ------------: | -------: | ---------: |
| Why is goodput more useful than raw throughp... |   goodput, paged, radix |        0.0 |           0.1 |   4665.8 |     4666.7 |
| What problem does PagedAttention actually so... |    paged, radix, disagg |        0.0 |           0.1 |   3979.6 |     3979.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching |        0.0 |           0.1 |   3466.0 |     3466.1 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **4037.1** · total **4037.5**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the context provided, **Goodput** is more useful than raw throughput because it focuses exclusively on the **requests per second (RPS)** that meet specific targets (TTFT and TPOT).

The context explicitly states that:

1. **Goodput** counts only requests per second that met the targets.
2. **Throughput at saturation** ignores SLOs (specifically the SLOs mentioned in the context).

This i

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory**.

While the context mentions that it stores KV cache in non-contiguous pages to "remove the internal fragmentation that wasted most GPU memory," it does not explicitly state the *result* of this removal (e.g., "it allows for higher throughput" or "it reduces latency"). However, in the context of the provided text, the so

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**.

This is because the context explicitly states that prefill is compute-bound (requiring significant CPU work) and decode is memory-bound (requiring significant memory bandwidth). By separating these operations into different pools, the system can optimize for these specific constraints:

* **Prefill**

## Which N16-N19 pieces are real

* Các phần N16 (Cloud/IaC), N17 (Data pipeline), N18 (Lakehouse), và N19 (Vector index) đều là **stub** (sử dụng dữ liệu giả lập từ TOY_DOCS và tìm kiếm từ khóa cơ bản thay vì gọi dịch vụ thực tế). Chỉ có phần N20 (Serving với `llama-server`) là **real**.
* Giai đoạn chiếm ưu thế hoàn toàn là **llm** (độ trễ trung bình 4037.1 ms trên tổng số 4037.5 ms, chiếm ~100%), đúng như kỳ vọng của tôi vì mô hình deep learning yêu cầu tính toán sinh token nặng nề, trong khi việc tìm kiếm từ khóa trên 6 tài liệu mẫu diễn ra cực kỳ nhanh (0.1 ms).

* Nếu phải giảm một nửa độ trễ của pipeline này, tôi sẽ tập trung tối ưu hóa giai đoạn **llm**. Cụ thể, tôi có thể giảm số lượng `max_tokens` cần sinh ra, bật tính năng prompt caching (`--prompt-cache`) của llama.cpp để tái sử dụng KV cache của câu hỏi và ngữ cảnh, hoặc sử dụng cơ chế speculative decoding (draft model) để đẩy nhanh tốc độ giải mã.
