# Thuyết Minh Kỹ Thuật (Technical Defense) — Lab 19

**Học viên:** Nguyễn Đình Hoàng  
**Mã số học viên (MSSV):** 2A202601436  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  

---

### 1. Coreference Resolution (Phân giải đại từ)
- **Ví dụ từ dữ liệu HackerNoon:** `art_003::c000` (Bài báo về Google Ventures đầu tư vào Cohere và Anthropic):
  > *"Google Ventures invested $300 million in Cohere, an AI startup founded by Aidan Gomez and former Microsoft researchers. Cohere developed enterprise LLM models for search... Additionally, Google invested $2 billion into Anthropic... They developed Claude, a competitive conversational assistant."*
- **Hiện tượng:** Đại từ *"They"* trong câu *"They developed Claude..."* đứng sau hai công ty được đề cập gần nhau (Cohere và Anthropic). Trong một số trường hợp không dùng Conservative Coreference Prompt, LLM phân giải nhầm *"They"* thành *"Cohere and Microsoft researchers"* thay vì *"Anthropic (Dario & Daniela Amodei)"*.
- **Hậu quả đối với Graph:** Tạo ra quan hệ sai `(Cohere)-[DEVELOPED]->(Claude)` thay vì `(Anthropic)-[DEVELOPED]->(Claude)`. Trong GraphRAG, thông tin sai lệch này làm câu trả lời của mô hình suy luận sai nguồn gốc sản phẩm Claude.

---

### 2. Entity Resolution Threshold & Lexical Guard
- **Ngưỡng cosine similarity:** `threshold = 0.90` (sử dụng `sentence-transformers/all-MiniLM-L6-v2` kết hợp thuật toán Union-Find).
- **Cặp thực thể bị Guard chặn:** `Google` vs `Google Ventures` (Cosine similarity ~ 0.88).
- **Lý do chặn:** Mặc dù hai chuỗi có độ tương đồng embedding cao và trùng tiền tố, cơ chế **Lexical Guard** (`merge_guard`) kiểm tra sau khi loại bỏ hậu tố doanh nghiệp (`inc`, `corp`, `llc`) nhận diện `ventures` là tên đơn vị đầu tư mạo hiểm độc lập. Việc gộp `Google Ventures` vào `Google` làm mất đi ngữ cảnh phân biệt giữa các khoản đầu tư trực tiếp của Google và danh mục đầu tư mạo hiểm của Google Ventures.

---

### 3. Đồ thị & Super-node Mitigation
- **Top 3 Super-nodes trong Đồ thị Lab 19:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|------|--------------|---------------------|----------------------|
| 1 | **Apple Inc.** | Company | 8 |
| 2 | **Nvidia Corporation** | Company | 8 |
| 3 | **Meta Platforms Inc.** | Company | 7 |

- **Ưu điểm & Rủi ro của Temporal Mitigation (`ORDER BY published_date DESC LIMIT 50`):**
  - *Ưu điểm:* Ngăn chặn hiện tượng bùng nổ ngữ cảnh khi thực thể trung tâm có hàng ngàn kết nối. Đảm bảo thông tin truyền vào LLM Generator luôn chứa các sự kiện M&A, đầu tư, ra mắt sản phẩm mới nhất.
  - *Rủi ro:* Nếu truy vấn của người dùng tập trung vào các mốc lịch sử quá khứ (ví dụ: *"Microsoft đã đầu tư bao nhiêu vào OpenAI trong giai đoạn 2019?"*), chiến lược cắt tỉa 50 cạnh mới nhất có thể vô tình xóa bỏ các quan hệ lịch sử quan trọng đó.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):

| Loại câu hỏi | Metric | Flat RAG | GraphRAG | Nhận xét phân tích |
|--------------|--------|----------|----------|-------------------|
| **Factoid** | Comprehensiveness | 5.00 | 5.00 | Hai phương pháp tương đương nhau khi thông tin nằm gói gọn trong 1 chunk. |
| **Factoid** | Faithfulness | 5.00 | 5.00 | Cả hai đều đạt độ tin cậy tuyệt đối từ bằng chứng chunk. |
| **Factoid** | Multi-hop reasoning | 5.00 | 5.00 | Factoid đơn giản không đòi hỏi suy luận nhiều chặng. |
| **Factoid** | Latency (s) | 8.168s | 6.096s | GraphRAG truy vấn đúng seed node mang lại context ngắn gọn, giảm thời gian suy luận. |
| **Factoid** | Token usage | 871 tokens | 651 tokens | Graph Context tập trung hơn so với 6 vector chunks ngẫu nhiên. |
| **Multi-hop** | Comprehensiveness | 5.00 | 5.00 | Cả hai hệ thống đều tìm đủ câu trả lời cho các câu hỏi 2-hop. |
| **Multi-hop** | Faithfulness | 5.00 | 5.00 | Không phát sinh hiện tượng ảo giác. |
| **Multi-hop** | Multi-hop reasoning | 5.00 | 5.00 | GraphRAG kết nối đường đi Microsoft -> Person -> Company -> Google rõ ràng. |
| **Multi-hop** | Latency (s) | 18.154s | 22.840s | GraphRAG tốn thêm gian trao đổi ngữ cảnh đồ thị với LLM. |
| **Multi-hop** | Token usage | 1087 tokens | 1766 tokens | Graph Context bổ sung thông tin thuộc tính các cạnh làm tăng lượng token. |
| **Cross-doc** | Comprehensiveness | 5.00 | 4.50 | Flat RAG lấy toàn bộ text văn bản 2 tập đoàn Meta & Apple đầy đủ hơn. |
| **Cross-doc** | Faithfulness | 5.00 | 4.50 | Cả hai đều tuân thủ dữ liệu. |
| **Cross-doc** | Multi-hop reasoning | 5.00 | 4.50 | Flat RAG so sánh diện rộng rất tốt khi vector retrieve đúng bài báo chiến lược. |
| **Cross-doc** | Latency (s) | 1.902s | 9.122s | Flat RAG phản hồi nhanh hơn đáng kể. |
| **Cross-doc** | Token usage | 1223 tokens | 1926 tokens | GraphRAG kết hợp cả Vector + Graph Context dẫn đến prompt kích thước lớn. |

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
- **Đánh đổi Quality vs Cost vs Latency:**
  - *Flat RAG:* Chi phí xây dựng cực kỳ rẻ, thời gian lập chỉ mục nhanh, Latency truy vấn thấp. Rất mạnh cho câu hỏi Factoid hoặc tóm tắt tài liệu đơn lẻ. Tuy nhiên thất bại khi cần suy luận các mối quan hệ ẩn qua 2-3 tài liệu khác nhau.
  - *GraphRAG:* Đòi hỏi chi phí và thời gian trích xuất NER/RE ban đầu cao. Latency và Token usage truy vấn cao hơn 1.5 - 3 lần. Tuy nhiên mang lại khả năng truy vết nguồn gốc 100% và giải quyết các bài toán Multi-hop phức tạp.
- **Quyết định từ chối AI Coding Agent:**
  - AI Coding Agent đã đề xuất thực hiện thuật toán so sánh cặp toàn bộ ($O(N^2)$ Pairwise Vector Similarity) trên tất cả các thực thể để làm Entity Resolution.
  - *Lý do từ chối:* Phương pháp có độ phức tạp $O(N^2)$ gây tràn bộ nhớ RAM. Tôi đã thay đổi sang **FAISS Index Flat IP** với partition theo `Entity Type` kết hợp **Union-Find** và **Lexical Guard**, giảm độ phức tạp xuống $O(N \log N)$.
- **Giải pháp scale 350MB (~100,000 bài báo):**
  - *Bottleneck:* Tốc độ gọi API trích xuất NER/RE và nạp vào Neo4j (API Rate-limit & I/O Bottleneck).
  - *Giải pháp:* Sử dụng Celery/Redis async task queue, Neo4j `apoc.periodic.iterate` bulk streaming, và Louvain/Modularity Community Detection để sinh bản tóm tắt phân cấp.
