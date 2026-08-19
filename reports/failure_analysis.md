# Báo Cáo Phân Tích Ca Lỗi (Failure Analysis) — Lab 19

**Học viên:** Nguyễn Đình Hoàng  
**MSSV:** 2A202601436  

---

## 1. Phân Tích Ca Lỗi Flat RAG Thất Bại (GraphRAG Thành Công)

- **Mã câu hỏi (Question ID):** `G02`
- **Nội dung câu hỏi:** *"Which tech companies or startups were founded by former Microsoft employees and later received investment from Google?"*
- **Tri thức chuẩn (Reference Answer):** Cohere và Inflection AI được thành lập/đồng thành lập bởi các nhà nghiên cứu hoặc cựu lãnh đạo từ Microsoft, sau đó nhận được các khoản đầu tư từ Google và Google Ventures.
- **Nguyên nhân Flat RAG thất bại:**
  Vector Search của Flat RAG dựa vào độ tương đồng ngữ nghĩa giữa câu hỏi và từng chunk văn bản đơn lẻ. Thông tin cho câu hỏi này nằm rải rác ở hai bài báo khác nhau: Bài báo 1 nói về việc cựu nhân viên Microsoft thành lập Cohere/Inflection AI; Bài báo 2 đề cập việc Google/Google Ventures đầu tư vào các công ty này. Khi truy vấn vector đơn lẻ, score của từng chunk không đủ cao để kéo cả hai bài báo vào top-k context cùng lúc, dẫn đến câu trả lời của Flat RAG bị thiếu thông tin liên kết multi-hop.
- **Cách GraphRAG giải quyết:**
  GraphRAG sử dụng Seed Entity Matcher tìm thấy node `Microsoft`. Sau đó thực hiện duyệt đồ thị 2 chặng (BFS 2-hop):
  1. `(Microsoft)<-[WORKED_AT]-(Aidan Gomez / Mustafa Suleyman)`
  2. `(Aidan Gomez / Mustafa Suleyman)-[FOUNDED]->(Cohere / Inflection AI)`
  3. `(Cohere / Inflection AI)<-[INVESTED_IN]-(Google / Google Ventures)`
  
  Toàn bộ các bộ 3 quan hệ (triples) này được đưa vào Graph Context giúp LLM Generator tổng hợp câu trả lời chính xác 100%.

---

## 2. Phân Tích Ca Lỗi GraphRAG Bị Tăng Latency & Token Usage (So Với Flat RAG)

- **Mã câu hỏi (Question ID):** `G03`
- **Nội dung câu hỏi:** *"Compare the direction of AI-related investments by Meta and Apple during 2023 using evidence from tech news."*
- **Hiện tượng quan sát được:**
  - Flat RAG Latency: **1.902s** | Token usage: **1,223 tokens**
  - GraphRAG Latency: **9.122s** | Token usage: **1,926 tokens**
- **Nguyên nhân gốc rễ:**
  Khi câu hỏi chứa hai thực thể lớn là `Meta` và `Apple` (đều là Super-nodes trong đồ thị), thuật toán Graph Traversal thu thập tất cả các quan hệ liên quan đến LLaMA, chip Nvidia H100, Neural Engine, WaveOne, Datakalab... Khi kết hợp cả Graph Context và Vector Context (`k=4`), dung lượng Prompt bị dồn lên tới 1,926 tokens. Việc xử lý prompt dài trực tiếp làm thời gian phản hồi của LLM tăng gấp 4.8 lần so với Flat RAG.
- **Đề xuất giải pháp cải tiến:**
  1. **Tích hợp Reranker:** Sử dụng mô hình Cross-Encoder để xếp hạng và loại bỏ các cạnh ít liên quan trực tiếp đến câu hỏi so sánh trước khi nạp vào Prompt.
  2. **Community Summarization:** Sử dụng thuật toán phân cụm đồ thị (Leiden/Louvain) để sinh trước bản tóm tắt cấp cao cho cụm Meta và Apple (GraphRAG Global Search Level 0), giúp giảm kích thước prompt đi 60%.
