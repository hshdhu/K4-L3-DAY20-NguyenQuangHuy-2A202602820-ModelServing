# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` � `--parallel 4` � 15 samples over
60s at 2.0s intervals � raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a � not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5572 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Trong lúc load-50 chạy, peak n_busy_slots_per_decode đạt 3.96 trên 4 slot (99%), rõ ràng lớn hơn 1 và gần số slot cấu hình. Đây là giá trị trung bình số slot hoạt động mỗi bước decode cao nhất trong các mẫu đã lấy, không phải độ rộng batch tức thời tối đa. Kết quả hỗ trợ bằng chứng continuous batching: nhiều request cùng tham gia các bước decode. requests_processing đạt 4, requests_deferred đạt 46, và tokens_predicted_total tăng trong khoảng lấy mẫu, cho thấy server tiếp tục sinh token trong khi toàn bộ slot bận và nhiều request xếp hàng.

[Báo cáo load](02-server-results.md) tính effective concurrency ở 50 user là 31.4 bằng RPS nhân latency trung bình. Con số này không bằng 3.96 và không mâu thuẫn: effective concurrency tính cả request chờ và đang xử lý ở phía client, còn gauge busy slots mô tả công việc decode trong server. Ngoài ra, một số là ước lượng trung bình trên cửa sổ load hữu hạn, số còn lại là peak được lấy mẫu; không thể đối chiếu như hai giá trị cùng định nghĩa và cùng thời điểm. Tôi dùng gauge của server để đánh giá batching, dùng Little's Law để mô tả số request in flight, và requests_deferred làm bằng chứng trực tiếp về hàng đợi.

Report có 15 mẫu trong lần ghi 60 giây, dù cấu hình interval là 2 giây. Không giả định các mẫu thực tế luôn cách nhau đúng 2 giây hoặc quan sát được mọi peak giữa các lần scrape. kv_cache_usage_ratio là n/a vì build không xuất metric này; không diễn giải nó thành mức sử dụng bằng 0. tokens_predicted_total=5572 là counter tích lũy của server, không phải riêng số token của load-50.