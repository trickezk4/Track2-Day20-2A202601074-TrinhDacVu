# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64`   llama.cpp `b10488`  
retrieval backend: **keyword overlap**   3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 3708.3 | 3709.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2960.5 | 2960.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3946.0 | 3946.1 |

Mean per stage (ms): embed **0.0**   retrieve **0.1**  
llm **3538.3**   total **3538.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on **SLOs (Service Level Objects)** by counting only requests that met their targets, whereas **Raw Throughput** ignores SLOs and only considers requests per second at saturation.

This distinction highlights that Goodput is designed to be more efficient and reliable for applications where the goal is 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it prevents the wasted space that would otherwise be occupied by the internal fragmentation of contiguous memory blocks.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**, specifically to avoid the overhead of maintaining a large shared KV cache.

# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64`   llama.cpp `b10488`  
retrieval backend: **keyword overlap**   3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 3708.3 | 3709.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 2960.5 | 2960.7 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 3946.0 | 3946.1 |

Mean per stage (ms): embed **0.0**   retrieve **0.1**  
llm **3538.3**   total **3538.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it focuses on **SLOs (Service Level Objects)** by counting only requests that met their targets, whereas **Raw Throughput** ignores SLOs and only considers requests per second at saturation.

This distinction highlights that Goodput is designed to be more efficient and reliable for applications where the goal is 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the KV cache in non-contiguous pages, it prevents the wasted space that would otherwise be occupied by the internal fragmentation of contiguous memory blocks.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bound**, specifically to avoid the overhead of maintaining a large shared KV cache.

According to the context, this optimization occurs because:
1.  **Prefill** is compute-bound (requires significant processing power).
2.  **Decode** is memory-bound (requires significant bandwidth).
3.  **RadixAttention** uses 


## Which N16-N19 pieces are real

Các phần N16 (Cloud/IaC), N17 (Data pipeline), N18 (Lakehouse), và N19 (Vector index) đều là **stub** (sử dụng dữ liệu giả lập từ TOY_DOCS và tìm kiếm từ khóa cơ bản thay vì gọi dịch vụ thực tế). Chỉ có phần N20 (Serving với `llama-server`) là **real**.

Giai đoạn chiếm ưu thế hoàn toàn là **llm** (độ trễ trung bình 3538.3 ms trên tổng số 3538.6 ms, chiếm ~100%), đúng như kỳ vọng của tôi vì mô hình deep learning yêu cầu tính toán sinh token nặng nề, trong khi việc tìm kiếm từ khóa trên 6 tài liệu mẫu diễn ra cực kỳ nhanh (0.1 ms).

Nếu phải giảm một nửa độ trễ của pipeline này, tôi sẽ tập trung tối ưu hóa giai đoạn **llm**. Cụ thể, tôi có thể giảm số lượng `max_tokens` cần sinh ra, bật tính năng prompt caching (`--prompt-cache`) của llama.cpp để tái sử dụng KV cache của câu hỏi và ngữ cảnh, hoặc sử dụng cơ chế speculative decoding (draft model) để đẩy nhanh tốc độ giải mã.
