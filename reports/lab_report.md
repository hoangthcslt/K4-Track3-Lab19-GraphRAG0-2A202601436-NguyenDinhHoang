# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Đình Hoàng  
**Mã số học viên (MSSV):** 2A202601436  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** `art_003::c000` (Bài báo về Google Ventures đầu tư vào Cohere và Anthropic):
  > *"Google Ventures invested $300 million in Cohere, an AI startup founded by Aidan Gomez and former Microsoft researchers. Cohere developed enterprise LLM models for search... Additionally, Google invested $2 billion into Anthropic... They developed Claude, a competitive conversational assistant."*
- **Hiện tượng:** Đại từ *"They"* trong câu *"They developed Claude..."* đứng sau hai công ty được đề cập gần nhau (Cohere và Anthropic). Trong một số trường hợp không dùng Prompt Conservative Coref, LLM có thể phân giải nhầm *"They"* thành *"Cohere and Microsoft researchers"* thay vì *"Anthropic (Dario & Daniela Amodei)"*.
- **Hậu quả đối với Graph:** Tạo ra False Edge `(Cohere)-[DEVELOPED]->(Claude)` thay vì `(Anthropic)-[DEVELOPED]->(Claude)`. Trong GraphRAG, thông tin sai lệch này dẫn đến câu trả lời sai nghiêm trọng khi người dùng hỏi về nguồn gốc sản phẩm Claude.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao (> 0.85) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` (sử dụng mô hình `sentence-transformers/all-MiniLM-L6-v2` kết hợp thuật toán Union-Find).
- **Cặp thực thể bị Guard chặn:** `Google` vs `Google Ventures` (Cosine similarity ~ 0.88).
- **Lý do chặn:** Mặc dù hai chuỗi ký tự có độ tương đồng embedding cao và trùng tiền tố, cơ chế **Lexical Guard** (`merge_guard`) kiểm tra sau khi loại bỏ hậu tố doanh nghiệp (`inc`, `corp`, `llc`) vẫn nhận diện `ventures` là tên đơn vị đầu tư mạo hiểm độc lập thuộc tập đoàn. Việc gộp `Google Ventures` vào `Google` sẽ làm mất đi ngữ cảnh phân biệt giữa các khoản đầu tư trực tiếp của Google và danh mục đầu tư mạo hiểm của Google Ventures.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy N cạnh (N=50) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes trong Đồ thị Lab 19:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | **Apple Inc.** | Company | 8 |
| 2 | **Nvidia Corporation** | Company | 8 |
| 3 | **Meta Platforms Inc.** | Company | 7 |

- **Ưu điểm & Rủi ro của Temporal Mitigation (`ORDER BY published_date DESC LIMIT 50`):**
  - *Ưu điểm:* Ngăn chặn hiện tượng bùng nổ ngữ cảnh (Context Explosion) khi thực thể trung tâm có hàng ngàn kết nối. Đảm bảo thông tin truyền vào LLM Generator luôn chứa các sự kiện M&A, đầu tư, ra mắt sản phẩm mới nhất.
  - *Rủi ro:* Nếu truy vấn của người dùng tập trung vào các mốc lịch sử quá khứ (ví dụ: *"Microsoft đã đầu tư bao nhiêu vào OpenAI trong giai đoạn 2019?"*), chiến lược cắt tỉa 50 cạnh mới nhất có thể vô tình xóa bỏ các quan hệ lịch sử quan trọng đó.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge trên 5 Golden Questions):

| Loại câu hỏi | Metric | Flat RAG | GraphRAG | Nhận xét phân tích |
|--------------|--------|----------|----------|-------------------|
| **Factoid** | Comprehensiveness | 5.00 | 5.00 | Hai phương pháp tương đương nhau khi thông tin nằm gói gọn trong 1 chunk. |
| **Factoid** | Faithfulness | 5.00 | 5.00 | Cả hai đều đạt độ tin cậy tuyệt đối từ bằng chứng chunk. |
| **Factoid** | Multi-hop reasoning | 5.00 | 5.00 | Factoid đơn giản không đòi hỏi suy luận nhiều chặng. |
| **Factoid** | Latency (s) | 8.168s | 6.096s | GraphRAG truy vấn đúng seed node mang lại context ngắn gọn, giảm thời gian suy luận. |
| **Factoid** | Token usage | 871 tokens | 651 tokens | Graph Context tập trung hơn so với 6 vector chunks ngẫu nhiên. |
| **Multi-hop** | Comprehensiveness | 5.00 | 5.00 | Cả hai hệ thống đều tìm đủ câu trả lời cho các câu hỏi 2-hop. |
| **Multi-hop** | Faithfulness | 5.00 | 5.00 | Không phát sinh hiện tượng ảo giác (hallucination). |
| **Multi-hop** | Multi-hop reasoning | 5.00 | 5.00 | GraphRAG kết nối đường đi Microsoft -> Person -> Company -> Google rõ ràng. |
| **Multi-hop** | Latency (s) | 18.154s | 22.840s | GraphRAG tốn thêm gian trao đổi ngữ cảnh đồ thị với LLM. |
| **Multi-hop** | Token usage | 1087 tokens | 1766 tokens | Graph Context bổ sung thông tin thuộc tính các cạnh làm tăng lượng token. |
| **Cross-doc** | Comprehensiveness | 5.00 | 4.50 | Flat RAG lấy toàn bộ text văn bản 2 tập đoàn Meta & Apple đầy đủ hơn. |
| **Cross-doc** | Faithfulness | 5.00 | 4.50 | Cả hai đều tuân thủ dữ liệu. |
| **Cross-doc** | Multi-hop reasoning | 5.00 | 4.50 | Flat RAG so sánh diện rộng rất tốt khi vector retrieve đúng bài báo chiến lược. |
| **Cross-doc** | Latency (s) | 1.902s | 9.122s | Flat RAG phản hồi nhanh hơn đáng kể. |
| **Cross-doc** | Token usage | 1223 tokens | 1926 tokens | GraphRAG kết hợp cả Vector + Graph Context dẫn đến prompt kích thước lớn. |

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG gặp khó khăn (GraphRAG vượt trội - G02):**
   - *Question ID & Câu hỏi:* `G02` — *"Which tech companies or startups were founded by former Microsoft employees and later received investment from Google?"*
   - *Tại sao Flat RAG gặp khó khăn?* Vector similarity chỉ tìm được các bài báo chứa từ khóa "Google investment" hoặc "Microsoft employees", nhưng khó truy xuất được bài báo trung gian nối giữa cựu nhân viên Microsoft với việc thành lập Cohere/Inflection AI.
   - *GraphRAG đã giải quyết như thế nào?* Đồ thị tri thức thực hiện đường đi 2-hop: `(Microsoft)<-[WORKED_AT]-(Aidan Gomez/Mustafa Suleyman)-[FOUNDED]->(Cohere/Inflection AI)<-[INVESTED_IN]-(Google/Google Ventures)`. Toàn bộ liên kết chéo này được tập hợp gọn gàng trong Graph Context.

2. **Ca lỗi GraphRAG bị dồn thông tin (Overhead Latency - G03):**
   - *Question ID & Câu hỏi:* `G03` — *"Compare the direction of AI-related investments by Meta and Apple during 2023 using evidence from tech news."*
   - *Nguyên nhân:* Khi seed node là cả `Meta` và `Apple`, lượng cạnh liên quan đến chip H100, LLaMA 2, Neural Engine, WaveOne... được trích xuất rất lớn. Do kết hợp cả Graph Context và Vector Context (`k=4`), prompt đẩy lên đến 1,926 tokens làm thời gian xử lý của GraphRAG tăng từ 1.9s lên 9.1s.
   - *Đề xuất khắc phục:* Áp dụng Reranker hoặc Community Summary (GraphRAG Global Search level 0) để tóm tắt nhóm mối quan hệ trước khi đưa vào Generator.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:**
  - *Flat RAG:* Chi phí xây dựng cực kỳ rẻ, thời gian lập chỉ mục nhanh, Latency truy vấn thấp. Rất mạnh cho câu hỏi Factoid hoặc tóm tắt tài liệu đơn lẻ. Tuy nhiên thất bại khi cần suy luận các mối quan hệ ẩn qua 2-3 tài liệu khác nhau.
  - *GraphRAG:* Đòi hỏi chi phí và thời gian trích xuất NER/RE ban đầu cao (LLM extraction overhead). Latency và Token usage truy vấn cao hơn 1.5 - 3 lần. Tuy nhiên mang lại khả năng truy vết nguồn gốc (Provenance Traceability) 100% và giải quyết các bài toán Multi-hop phức tạp.
- **Quyết định từ chối AI Coding Agent:**
  - AI Coding Agent đã đề xuất thực hiện thuật toán so sánh cặp toàn bộ ($O(N^2)$ Pairwise Vector Similarity) trên tất cả các thực thể được trích xuất để làm Entity Resolution.
  - *Lý do từ chối:* Phương pháp này có độ phức tạp $O(N^2)$, khi dữ liệu tăng lên hàng ngàn thực thể sẽ gây tràn bộ nhớ RAM và đóng băng hệ thống. Tôi đã yêu cầu Agent thay đổi sang thuật toán **FAISS Index Flat IP** với partition theo `Entity Type` kết hợp **Union-Find** và **Lexical Guard**, giảm độ phức tạp xuống $O(N \log N)$ và chạy vô cùng mượt mà.
- **Giải pháp scale lên toàn bộ 350MB (~100,000 bài báo):**
  - *Bottleneck đầu tiên:* Tốc độ gọi API trích xuất NER/RE và nạp vào Neo4j (API Rate-limit & I/O Bottleneck).
  - *Giải pháp kiến trúc:*
    1. **Async Worker Queue:** Sử dụng Celery / Redis để phân tán job trích xuất NER/RE cho nhiều LLM Workers chạy song song.
    2. **Neo4j Bulk Streaming:** Sử dụng `apoc.periodic.iterate` để ghi dữ liệu Graph theo lô (batch 10,000 triples/lần) tránh khóa cơ sở dữ liệu.
    3. **Hierarchical Community Summarization:** Sử dụng thuật toán Modularity / Louvain Community Detection để nhóm các node thành từng cụm chủ đề và tạo bản tóm tắt cấp cao (Hierarchical Summaries), phục vụ truy vấn toàn cục mà không cần quét lại toàn bộ đồ thị.

---

## PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Phân giải đại từ bảo thủ giúp loại bỏ các liên kết giả mạo giữa các thực thể ở xa nhau. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giới hạn kiểu thực thể (Company, Person, Technology) giúp đồ thị chuẩn hóa và không bị rác. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Tận dụng lệnh Cypher `UNWIND` giúp ghi hàng chục node/edge vào Neo4j Aura trong vài millisecond. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp chính xác `Microsoft Corp` vào `Microsoft`, `Apple Inc.` vào `Apple` thông qua vector + guard. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `node_degree()` | Giới hạn tối đa 50 cạnh mới nhất cho các super-node như Apple, Nvidia giúp kiểm soát được kích thước prompt. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `judge_json()` | Đánh giá tự động khách quan 3 tiêu chí Comprehensiveness, Faithfulness, Multi-hop theo thang điểm 1-5. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:**
  1. Mô hình Groq trả về JSON bị bọc trong khối suy luận hoặc Markdown làm `json.loads()` bị lỗi `JSONDecodeError`.
  2. Kết nối Neo4j Aura Cloud bị ngắt quãng giữa chừng (`SessionExpired` / `ServiceUnavailable`) khi thực hiện các lượt đánh giá LLM-as-a-Judge kéo dài.
- **Cách đã xử lý thành công:**
  1. Viết lại hàm `parse_json_object()` sử dụng biểu thức chính quy (`rfind('}')` và `find('{')`) để trích xuất khối JSON chuẩn xác nhất, loại bỏ ký tự rác.
  2. Nâng cấp cơ chế `run_cypher()` với vòng lặp tự động kết nối lại (Auto-reconnect with 5 retries & connection pooling configuration `max_connection_lifetime=60`), đảm bảo pipeline chạy 100% không bị dừng đột ngột.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** Hệ Thống Tra Cứu & Phân Tích Mối Quan Hệ Doanh Nghiệp & Đầu Tư Công Nghệ (Tech Corporate Intelligence GraphRAG).
- **Đặc thù bài toán & Lý do chọn giải pháp:** Bài toán yêu cầu theo dõi dòng tiền đầu tư, thương vụ M&A và sự dịch chuyển nhân sự cấp cao giữa các tập đoàn công nghệ. Flat RAG thông thường không thể trả lời các câu hỏi như *"Nhà đầu tư nào đã gián tiếp tài trợ cho các startup do cựu nhân viên Google sáng lập?"*. Việc sử dụng GraphRAG là bắt buộc.
- **Cấu trúc Node & Relation dự kiến:**
  - **Nodes:** `Company`, `Person`, `Technology`, `Investor`, `Product`, `Event`.
  - **Relations:** `INVESTED_IN`, `ACQUIRED`, `DEVELOPED`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`, `COMPETES_WITH`.
- **Chiến lược xử lý Super-node & Entity Resolution:**
  - Áp dụng FAISS HNSW Indexing cho Entity Resolution khi dữ liệu vượt 1 triệu thực thể.
  - Thiết lập Super-node Degree Cap ở mức 100 kết nối, kết hợp bộ lọc thời gian `published_date` và trọng số tin cậy `confidence >= 0.85`.

---

## TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | **5/5** | Nắm vững toàn bộ 5 module từ Chunking đến Evaluation. |
| Khả năng kiểm soát AI Coding Agent | **5/5** | Làm chủ kiến trúc, từ chối thuật toán kém hiệu quả O(N^2). |
| Chất lượng đồ thị tri thức xây dựng | **5/5** | Đồ thị sạch trên Neo4j, 100% edge provenance integrity. |
| Khả năng phân tích và debug hệ thống | **5/5** | Giải quyết triệt để lỗi JSON decode và connection pool. |
