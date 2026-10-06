# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **4 physical � 8 logical** cores � `ngl=99` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 38.5 | 100% |
| 2 | 38.5 | 100% |
| 4 | 38.5 | 100% |
| 8 | 38.4 | 100% |
| 16 | 38.1 | 99% |

**Best**: `-t 4` at 38.5 tok/s
**Slowest tested**: `-t 16` at 38.1 tok/s (1.01x spread)
**Against the physical-core default** (`-t 4`, 38.5 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=4 make bench
```

## Your explanation

Throughput gần như phẳng từ 1 đến 4 thread, đều khoảng 38.5 tok/s. Với 8 thread đạt 38.4 tok/s và 16 thread đạt 38.1 tok/s, chỉ giảm khoảng 1% so với mức tốt nhất. Không quan sát được knee rõ trong lưới đã đo. Script chọn 4 thread là Best, nhưng speedup so với mặc định 4 core vật lý là 1.00x: thread tuning chưa tạo speedup trong phép thử này.

Sweep dùng ngl=99, yêu cầu GPU offload. Đường cong phẳng phù hợp với khả năng công việc decode trên GPU chi phối latency và số CPU thread ít ảnh hưởng. Đây là giả thuyết, cần log số layer thực tế offload để xác nhận; bảng này không đủ để kết luận CPU đã bão hòa memory bandwidth. Mức giảm nhỏ ở 16 thread có thể do scheduling overhead hoặc dao động phép đo, chưa đủ dữ liệu để tách hai nguyên nhân.

Tôi giữ 4 thread cho cấu hình hiện tại. Một sweep bổ sung với ngl=0 sẽ đo tác động của thread khi không offload GPU, đồng thời cung cấp cấu hình CPU để so với kết quả ngl=99 bằng cùng llama-bench, model và metric tg128. Không so trực tiếp tok/s của llama-bench với baseline HTTP vì hai phép đo có workload và overhead khác nhau.