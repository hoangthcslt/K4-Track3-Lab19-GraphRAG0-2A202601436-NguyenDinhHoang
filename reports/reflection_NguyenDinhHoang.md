# Báo Cáo Suy Ngẫm Cá Nhân (Reflection) — Lab 19

**Học viên:** Nguyễn Đình Hoàng  
**MSSV:** 2A202601436  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  

---

## 1. Mapping Bài Giảng Vào Implementation Code

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|--------------------------|------------------|------------------------|-----------------------------|
| **Conservative Coreference** | Module 1 | `resolve_coref_batch()` | Phân giải đại từ bảo thủ giúp loại bỏ các liên kết giả mạo giữa các thực thể ở xa nhau. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giới hạn kiểu thực thể (Company, Person, Technology) giúp đồ thị chuẩn hóa và không bị rác. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Tận dụng lệnh Cypher `UNWIND` giúp ghi hàng chục node/edge vào Neo4j Aura trong vài millisecond. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp chính xác `Microsoft Corp` vào `Microsoft`, `Apple Inc.` vào `Apple` thông qua vector + guard. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `node_degree()` | Giới hạn tối đa 50 cạnh mới nhất cho các super-node như Apple, Nvidia giúp kiểm soát được kích thước prompt. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `judge_json()` | Đánh giá tự động khách quan 3 tiêu chí Comprehensiveness, Faithfulness, Multi-hop theo thang điểm 1-5. |

---

## 2. Bài Học Kinh Nghiệm Quá Trình Debugging

- **Lỗi kỹ thuật phức tạp nhất:**
  1. Xử lý định dạng đầu ra không đồng nhất từ LLM khi trích xuất JSON (lỗi `JSONDecodeError` do markdown backticks hoặc text giải thích kèm theo).
  2. Lỗi mất kết nối cơ sở dữ liệu Neo4j (`SessionExpired` / `ServiceUnavailable`) khi thực hiện hàng loạt truy vấn lặp trong bước nạp đồ thị và đánh giá.
- **Giải pháp khắc phục:**
  1. Xây dựng hàm `parse_json_object()` sử dụng biểu thức chính quy cắt chuỗi từ `{` đầu tiên đến `}` cuối cùng, loại bỏ toàn bộ phần dư thừa trước khi parse.
  2. Triển khai cơ chế Per-call Driver reconnect và Retry logic (tối đa 5 lần thử lại) trong `run_cypher()`, đảm bảo toàn bộ pipeline chạy liên tục mà không bị gián đoạn bởi sự cố mạng tạm thời.

---

## 3. Kế Hoạch Áp Dụng Thực Tế (Action Plan)

- **Dự án ứng dụng:** Hệ Thống Phân Tích Dòng Tiền & Mối Quan Hệ Doanh Nghiệp (Enterprise Corporate Intelligence Platform).
- **Lý do chọn GraphRAG:** Các câu hỏi truy vấn doanh nghiệp có độ phức tạp cao, yêu cầu truy vết mối quan hệ sở hữu chéo, đầu tư mạo hiểm và nhân sự cấp cao qua nhiều tầng công ty con. Flat RAG đơn thuần không thể đáp ứng được.
- **Cấu trúc Đồ thị:**
  - **Nodes:** `Company`, `Person`, `Technology`, `Investor`, `Product`.
  - **Relations:** `INVESTED_IN`, `ACQUIRED`, `DEVELOPED`, `FOUNDED`, `WORKED_AT`, `PARTNERED_WITH`.
- **Giải pháp Scale:** Tích hợp FAISS HNSW cho bước Entity Resolution và áp dụng Louvain Community Detection để tóm tắt các cụm tập đoàn trước khi đưa vào truy vấn.

---

## 4. Bảng Tự Đánh Giá

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | **5/5** | Nắm vững toàn bộ 5 module từ Chunking đến Evaluation. |
| Khả năng kiểm soát AI Coding Agent | **5/5** | Làm chủ kiến trúc, từ chối thuật toán kém hiệu quả O(N^2). |
| Chất lượng đồ thị tri thức xây dựng | **5/5** | Đồ thị sạch trên Neo4j, 100% edge provenance integrity. |
| Khả năng phân tích và debug hệ thống | **5/5** | Giải quyết triệt để lỗi JSON decode và connection pool. |
