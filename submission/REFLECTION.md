# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Nguyễn Quang Huy
**MSSV:** 2A202602820
**Cohort:** A20-K4
**Ngày submit:** 10/6/2026

---

## 1. Hardware & runtime

- **OS:** Windows 11 AMD64; Python 3.12.10.
- **CPU:** Intel Core i7-1165G7 @ 2.80GHz; 4 physical / 8 logical cores.
- **CPU extensions:** probe chưa ghi nhận, không suy đoán AVX từ tên CPU.
- **RAM:** 15.8 GB.
- **Accelerator:** NVIDIA GeForce MX450, 2048 MiB VRAM; probe phát hiện CUDA và Vulkan.
- **Runtime thực tế:** llama.cpp b10488, asset `llama-b10488-bin-win-vulkan-x64.zip` (Vulkan, không phải CUDA binary).
- **Model:** Gemma 4 E2B (`gemma4-e2b`), `UD-Q4_K_XL` và `UD-Q2_K_XL`.
- **Chạy ở đâu:** laptop local của tôi.

**Setup story:** Windows PowerShell 5.1 đọc sai dấu em dash trong lab.ps1. Đổi sang ASCII và đặt Python xuất UTF-8 giúp probe chạy được. Sau đó bootstrap tạo virtualenv, cài package và tải runtime/models. hardware.json, models/active.json và ảnh 01 là bằng chứng.

## 2. Đo lường

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 11347 | 431 / 1282 | 27.2 / 27.8 | 2126 / 2985 / 2985 | 36.8 |
| UD-Q2_K_XL | 2.24 | 6696 | 466 / 2049 | 27.6 / 30.2 | 2210 / 3734 / 3734 | 36.2 |

**Quan sát:** 2-bit nhỏ hơn 24.6% nhưng decode chậm hơn 1.6%. Cùng prompt TTFT/TPOT, temperature=0, max_tokens=384, hai bản trả lời đúng định nghĩa, gần tương đương nhưng nguyên nhân TPOT còn chung chung. Tôi giữ 4-bit vì đủ RAM và latency thấp hơn nhẹ. Một prompt chưa đại diện chất lượng tổng thể.

Benchmark: threads=4, ngl=99, ctx=2048, max_tokens=64; warm-up bỏ, 10/10 request mỗi bản. P95/P99 nearest-rank với 10 mẫu là mẫu lớn nhất. Không suy ra cơ chế bottleneck từ chênh lệch decode nhỏ trong một lần chạy.

## 3. Serving under load

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.65 | 13000 | 24000 | 25000 | 8.6 | 0.0% |
| 50 | 1.02 | 33000 | 47000 | 49000 | 31.4 | 0.0% |

- Số user tăng 5x; throughput tăng **1.57x**, P95 tăng **1.96x**.
- Effective concurrency 50 user: **31.4**, so với **4 slot**. L=RPS x mean E2E, gồm cả queue, không phải utilization.
- Peak busy slots/decode: **3.96/4**; processing **4**, deferred **46**.

**Saturation reading:** RPS chỉ tăng 1.57x, P95 tăng 1.96x và deferred=46 cho thấy hàng đợi ở 50 user. Chưa biết ngưỡng user chính xác hay tách queue/compute định lượng. Tôi thử giới hạn in-flight/hàng đợi trước để giảm queue time; tăng slot có thể tạo áp lực KV cache/VRAM. Đề xuất cần đo lại, không phải speedup đã chứng minh.

**SLO:** ít nhất 95% completion E2E <=25 giây. CSV 10 user có max=24.63 giây nên goodput khoảng 0.653 req/s. 50 user có P50=33 giây, P95=47 giây, không đạt; percentile gợi ý goodput dưới khoảng 0.511 req/s, không phải giá trị chính xác. Cần latency từng request để tính chính xác và accounting cho request chưa hoàn thành. TTFT/TPOT dưới tải chưa được đo riêng.

CSV ghi 38/59 request còn terminal cuối ghi 39/61; tôi dùng CSV làm nguồn cho bảng, giữ nguyên screenshot. Khác biệt có thể do thời điểm xuất snapshot. Closed-loop/think time và cửa sổ 60 giây giới hạn áp dụng Little's Law như một ước lượng steady-state.

## 4. Integration

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Local laptop, không tích hợp IaC | stub / chưa tích hợp |
| N17 Data pipeline | TOY_DOCS viết sẵn | stub |
| N18 Lakehouse | Danh sách Python trong RAM | stub |
| N19 Vector + features | Keyword overlap top-3, không embedding/index | stub |
| N20 Serving | llama-server HTTP localhost:8080 | real |

Mean của 3 query: embed **0.0 ms**, retrieve **0.1 ms**, llm **5018.0 ms**, total **5018.1 ms**. Dominant stage **llm**, khoảng **100%** theo báo cáo làm tròn.

**Reflection:** LLM HTTP call chiếm gần toàn bộ latency, đúng kỳ vọng với corpus đồ chơi. Tôi ưu tiên giữ connection và giảm ngân sách output/context có kiểm tra chất lượng. Server compute thấp hơn client call, cần trace tìm overhead. Tối ưu retrieve 0.1 ms không giúp giảm tổng 2x. Chưa đo speedup các đề xuất này.

## 5. The single change that mattered most

**Change:** Bật cấu hình offload Vulkan (`ngl=0 -> 99`), giữ **4 thread**, cùng model Q4, llama-bench tg128 và hai lần lặp mỗi điểm.

```text
before:  13.65 tok/s (CPU-only, t=4, ngl=0)
after:   38.54 tok/s (offload requested, t=4, ngl=99)
speedup: 2.823x
```

Offload chuyển phần xử lý layer được hỗ trợ sang GPU, dùng khả năng xử lý song song và hệ thống bộ nhớ GPU thay vì toàn bộ decode trên CPU. Đây là cơ chế phù hợp với throughput tăng và đường cong offload gần như phẳng từ 1 đến 16 CPU thread. ngl=99 là yêu cầu offload, không chứng minh 99 layer thực tế chạy trên GPU; chưa có counters để tách compute và bandwidth. Runtime là Vulkan. Hai sweep chạy tuần tự, chưa kiểm soát nhiệt độ/power state hoặc lặp xen kẽ, nên tôi báo speedup quan sát được trên workload tg128, không khẳng định HTTP server cũng tăng 2.823x.

Một thay đổi được kiểm chứng thêm trong CPU-only là 4 -> 8 thread: 13.65 -> 16.41 tok/s (**1.202x**). Đỉnh đã đo ở 8 logical thread, còn 16 thread chỉ đạt 10.63 tok/s, giảm 35.2%. Điều này khác kỳ vọng knee tại 4 physical core. SMT có thể tận dụng tài nguyên core khi luồng khác chờ dữ liệu/dependency và giúp dequantization/tính toán; 16 thread oversubscribe gây scheduling overhead và cạnh tranh cache/execution/memory. Các cơ chế này là giả thuyết phù hợp số đo, chưa được xác nhận bằng hardware counters. Tôi giữ 4 thread khi offload, chỉ dùng 8 nếu CPU-only.

Bằng chứng: `benchmarks/01-tuning-tg128-gpu.md` và `01-tuning-tg128-cpu.md`, cùng JSON chưa sửa số đo.

## 6. Bonus

Không thực hiện bonus.

## 7. Điều làm tôi ngạc nhiên nhất

CPU-only vẫn tăng throughput khi dùng 8 logical thread; cấu hình offload lại gần như không nhạy với CPU thread. Số thread tối ưu phụ thuộc backend, không thể áp dụng chung.

## 8. Self-check trước khi push

- Bằng chứng setup: hardware.json, models/active.json, ảnh 01.
- Baseline: report/JSON, câu trả lời 4-bit/2-bit, ảnh 02.
- Serving: ảnh 03a và 03b; load test CSV và ảnh 04/05.
- Batching: report và metrics CSV; integration: report và JSON.
- Origin đúng tên repo nộp. Commit, push và xác nhận public/LMS do tôi thực hiện sau khi đọc lại báo cáo.
- Không commit weights, runtime, .venv hoặc .env.

## 9. Khai báo sử dụng AI

Tôi dùng Codex để kiểm tra/sửa lỗi PowerShell và encoding, hướng dẫn chạy lab, đọc số đo, chạy pipeline, hỗ trợ soạn nhận xét/REFLECTION và kiểm tra file. Số đo lấy từ laptop khai báo ở §1; screenshot do tôi chụp. Các giải thích cơ chế được AI hỗ trợ đề xuất, có ghi rõ giới hạn bằng chứng.