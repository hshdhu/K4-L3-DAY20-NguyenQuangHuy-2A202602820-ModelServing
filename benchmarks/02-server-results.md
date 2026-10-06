# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` � llama.cpp `b10488` �
`--parallel 4` � `ctx=2048` � `threads=4` �
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 38 | 0.65 | 13000 | 24000 | 25000 | 8.6 | 0.0% |
| 50 | 59 | 1.02 | 33000 | 47000 | 49000 | 31.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.57x** (31% of linear) |
| P95 latency | **1.96x** |
| Effective concurrency at 50 users | 31.4 vs `--parallel 4` slots (occupancy/slot ratio 7.86) |

**Saturated.** Throughput delivered only 1.57x for 5x the offered load, and effective concurrency (31.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.57x while P95 moved 1.96x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Số user tăng 5x, RPS tăng 1.57x (0.6528 -> 1.0218), đạt 31% tuyến tính; P95 tăng 1.96x (24 -> 47 giây). Closed-loop có think time nên số user tăng 5x không chứng minh tốc độ đến tăng đúng 5x. Effective concurrency là 8.6 ở 10 user và 31.4 ở 50 user, vượt 4 slot. Metrics dưới tải 50 user cho peak busy_slots=3.96/4, processing=4, deferred=46: bằng chứng trực tiếp các slot kín và có hàng đợi. Hai mức tải chưa xác định ngưỡng bão hòa chính xác. Latency tăng phù hợp queue time tăng nhưng chưa tách được queue và compute; batching và workload mix cũng ảnh hưởng.

SLO minh họa: ít nhất 95% request có E2E <=25 giây. 10 user có max 24.63 giây, tất cả 38 completion trong CSV đạt ngưỡng, goodput khoảng 0.653 req/s. 50 user có P50=33 giây, P95=47 giây nên không đạt SLO; percentile gợi ý dưới khoảng một nửa completion đạt 25 giây (goodput dưới khoảng 0.511 req/s). Không thể tính chính xác từ CSV tổng hợp: cần latency từng request. Request chưa hoàn thành khi test kết thúc không được tính, nên cửa sổ ngắn có thể đánh giá thấp tail latency.

Knob đầu tiên tôi thử là giới hạn request in flight/hàng đợi bằng admission control để giảm queue time. Cần đo lại goodput và tính cả request bị từ chối; đây không phải khẳng định đã tăng năng lực server. Chưa tăng parallel ngay vì context được chia giữa slot và GPU chỉ có 2 GB VRAM, có thể tăng áp lực KV/cache.

CSV ghi 38/59 completion còn screenshot terminal cuối ghi 39/61. Khác biệt phù hợp với snapshot CSV trước các completion cuối lúc shutdown, nhưng chưa kiểm chứng thời điểm chính xác. Tôi giữ nguyên nguồn và dùng CSV nhất quán cho report/REFLECTION.