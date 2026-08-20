# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B`   host `Windows-AMD64`   llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64`   warm-up discarded
Completed requests: `Q4_K_M` 10/10   `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
| :----------- | --------: | --------: | ----------------: | ----------------: | -------------------: | -------------: |
| Q4_K_M       |      0.50 |    104415 |         393 / 580 |        9.7 / 17.0 |    993 / 1649 / 1649 |          103.5 |
| UD-Q2_K_XL   |      0.39 |      6118 |         446 / 761 |       11.9 / 16.4 |   1298 / 1715 / 1715 |           84.3 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.23x SLOWER** than `Q4_K_M` here, despite being 0.11 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead   few cores, no GPU offload   the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## My Observation:

_Is the smaller quantization worth it on your machine?_
_Compare the numbers above, then judge the answer quality yourself: run `make serve` on each and ask the same question twice. Size and speed are measurable; usefulness is your call._

* Không, nó nhẹ hơn nhưng chậm hơn về decode token. Vì mô hình Qwen3.5 0.8B rất nhỏ và nằm trọn trong VRAM của GPU, hệ thống không bị giới hạn bởi băng thông bộ nhớ (memory-bandwidth bound) mà rơi vào trạng thái nghẽn tính toán (compute-limited). Do đó, chi phí giải nén ma trận phức tạp của định dạng 2-bit tốn thời gian xử lý hơn cả lợi ích giảm dung lượng mang lại.
* Khi chào và hỏi đơn giản: "bạn biết tôi là ai không?" thì 4b ổn còn 2b bị hallucinate, 2b trả lời lan man và không liên quan đến câu hỏi.
