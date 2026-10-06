# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` � llama.cpp `b10488` �
retrieval backend: **keyword overlap** � 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 4898.3 | 4898.3 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 4976.0 | 4976.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 5179.8 | 5179.9 |

Mean per stage (ms): embed **0.0** � retrieve **0.1** �
llm **5018.0** � total **5018.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece trong lần chạy | Real hay stub? |
|---|---|---|
| N16 | Laptop local, không nối cluster/IaC | stub / chưa tích hợp |
| N17 | TOY_DOCS viết sẵn, không có ingestion | stub |
| N18 | Danh sách Python trong RAM, không có lakehouse | stub |
| N19 | Keyword overlap top-3, không có embedding/vector index | stub |
| N20 | HTTP gọi llama-server localhost:8080 | real |

Cả 3 query trả về context và câu trả lời thật. Embed=0.0 ms vì không dùng embedding model; retrieve=0.1 ms là giá trị làm tròn. LLM=5018.0 ms trên total=5018.1 ms, khoảng 99.998%, là bottleneck của pipeline đồ chơi này.

Nếu phải giảm latency 2x, tôi ưu tiên stage gọi LLM: thử HTTP client giữ connection, rồi giảm ngân sách output/context với kiểm tra chất lượng. Đây là đề xuất chưa đo speedup. llm_ms bao gồm toàn bộ HTTP call, không chỉ compute: server prefill+decode thấp hơn đáng kể client latency; chưa có trace để quy phần chênh cho queue, mạng hay kết nối. Tối ưu retrieve 0.1 ms không thể giảm tổng latency 2x.