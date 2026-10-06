# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` � host `Windows-AMD64` � llama.cpp `b10488`
CPU: **4 physical � 8 logical** cores � `ngl=0` � metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 7.3 | 45% |
| 2 | 11.3 | 69% |
| 4 | 13.7 | 83% |
| 8 | 16.4 | 100% |
| 16 | 10.6 | 65% |

**Best**: `-t 8` at 16.4 tok/s
**Slowest tested**: `-t 1` at 7.3 tok/s (2.23x spread)
**Against the physical-core default** (`-t 4`, 13.7 tok/s): 1.20x

Use this in your run:

```bash
LAB_N_THREADS=8 make bench
```

## Your explanation

CPU-only (ngl=0) tăng throughput từ 7.35 tok/s ở 1 thread lên 11.32 ở 2 thread, 13.65 ở 4 thread và 16.41 ở 8 thread. Đỉnh trong lưới đã đo nằm tại 8 thread (số logical core), không phải 4 core physical. Chưa xác định được knee chính xác giữa các điểm, nhưng khi tăng lên 16 thread throughput giảm còn 10.63 tok/s, thấp hơn đỉnh khoảng 35.2%.

Before/after khi chỉ thay thread trong cấu hình CPU-only: 4 -> 8 thread, 13.65 -> 16.41 tok/s, speedup 1.202x (khoảng 20.2%). Model, ngl=0, metric tg128 và số lần lặp mỗi điểm giữ nguyên. Đây là speedup trong llama-bench, không phải số đo TTFT/TPOT của HTTP server.

Việc 8 logical thread vẫn tốt hơn 4 physical thread cho thấy giả định decode đạt trần bandwidth ngay tại số core physical không mô tả đầy đủ phép thử này. Một cơ chế phù hợp là SMT giúp tận dụng tài nguyên core còn rảnh khi thread khác chờ dữ liệu hoặc có dependency; decode còn phải thực hiện dequantization và tính toán. Đây là giả thuyết cơ chế, chưa có hardware counters để chứng minh phần nào chi phối. 16 thread vượt quá 8 logical CPU nên phải chia thời gian chạy, làm tăng scheduling overhead và cạnh tranh cache, execution units, cùng hệ thống bộ nhớ. Kết quả tụt ở 16 thread phù hợp với oversubscription, nhưng không xác định riêng được mức đóng góp của mỗi yếu tố.

### Đối chiếu với cấu hình GPU offload

[Bảng ngl=99](01-tuning-tg128-gpu.md) gần như phẳng: 38.50, 38.52, 38.54, 38.35 và 38.11 tok/s ở 1, 2, 4, 8 và 16 thread. Ở cùng 4 thread, ngl=0 đạt 13.65 tok/s còn ngl=99 đạt 38.54 tok/s: 2.823x. Chỉ cấu hình GPU layers được thay đổi giữa hai sweep; các lần chạy tuần tự chưa kiểm soát nhiệt độ hay power state nên đây là so sánh quan sát được, không phải kết quả nhiều lần đo xen kẽ. ngl=99 là số layer yêu cầu offload, không khẳng định 99 layer thực tế nằm trên GPU.

Tôi dùng 8 thread nếu chạy CPU-only. Với cấu hình offload hiện tại, tôi giữ 4 thread vì đó là Best của sweep ngl=99 và tăng CPU thread không tạo speedup. Không áp dụng máy móc cấu hình CPU-only tốt nhất cho GPU. Biến LAB_N_GPU_LAYERS đã được bỏ sau phép đo để các bước serving tiếp theo dùng cấu hình mặc định.