# 01 - Measure: latency baseline

Model `Gemma 4 E2B` � host `Windows-AMD64` � llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` � warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 � `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 11347 | 431 / 1282 | 27.2 / 27.8 | 2126 / 2985 / 2985 | 36.8 |
| UD-Q2_K_XL | 2.24 | 6696 | 466 / 2049 | 27.6 / 30.2 | 2210 / 3734 / 3734 | 36.2 |

- **TTFT** = client time to first token, including prompt processing, queueing and other request overhead. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Your observation

Cả hai quantization hoàn thành 10/10 request, với threads=4, ngl=99, ctx=2048 và max_tokens=64; request warm-up không được tính. Đây là kết quả của lần chạy baseline đã ghi lại, không phải trung bình của nhiều lần benchmark.

Bản UD-Q2_K_XL nhỏ hơn 0.73 GB (2.24 GB so với 2.97 GB), tương đương khoảng 24.6% theo kích thước làm tròn trong bảng. Decode đạt 36.2 tok/s so với 36.8 tok/s của UD-Q4_K_XL, chậm hơn khoảng 1.6%. TTFT P50 tăng từ 430.7 ms lên 465.7 ms (khoảng 8.1%); TPOT P50 tăng từ 27.16 ms lên 27.65 ms. E2E P50 tăng từ 2126.5 ms lên 2209.8 ms (khoảng 3.9%). Vì vậy, bản 2-bit tiết kiệm dung lượng weights nhưng không tạo speedup trong lần đo này. Chênh lệch decode nhỏ và chỉ có một lần chạy, nên chưa đủ bằng chứng để kết luận máy bị giới hạn bởi compute hay memory bandwidth.

TTFT P95 là 1282.0 ms ở bản 4-bit và 2048.8 ms ở bản 2-bit, tương ứng với request đầu trong tập đo của mỗi bản. Với 10 mẫu và nearest-rank, P95 và P99 đều lấy mẫu lớn nhất của từng chỉ số. Kết quả tail latency vì vậy nhạy với một request chậm; chưa xác định nguyên nhân là page cache, khởi tạo hay yếu tố khác. Thời gian load server được báo riêng, không cộng vào latency của các request đã đo.

### So chất lượng câu trả lời

Tôi dùng cùng prompt cho hai server:

> In at most 120 words, define TTFT and TPOT in LLM serving. Give one cause of high TTFT and one cause of high TPOT. No introduction or analogy.

Cấu hình: temperature=0, max_tokens=384, stream=false. Cả hai trả về finish_reason=stop. Câu trả lời được lưu tại [bản 4-bit](01-quality-4bit.txt) và [bản 2-bit](01-quality-2bit.txt).

Cả hai định nghĩa đúng TTFT và TPOT, trả lời ngắn và có chất lượng gần tương đương trên câu hỏi này. Bản 4-bit nêu hàng đợi request là một nguyên nhân TTFT cao; bản 2-bit nêu chi phí xử lý prompt ban đầu. Cả hai nhắc model loading, nhưng yếu tố này chủ yếu liên quan cold start, không phải mỗi request khi server đã sẵn sàng. Giải thích TPOT cao còn chung chung và chưa đề cập rõ memory bandwidth. Một prompt chưa đủ để kết luận chất lượng tổng thể của hai quantization.

Trên máy có 15.8 GB RAM, tôi chọn bản 4-bit cho các bước tiếp theo vì máy đủ RAM và bản này có latency thấp hơn nhẹ trong lần đo hiện tại. Bản 2-bit đáng cân nhắc khi cần giảm dung lượng weights, nhưng chưa có lợi thế tốc độ hoặc chất lượng trong phép thử này. Giảm kích thước file không đồng nghĩa RAM/VRAM sử dụng giảm đúng 24.6%.