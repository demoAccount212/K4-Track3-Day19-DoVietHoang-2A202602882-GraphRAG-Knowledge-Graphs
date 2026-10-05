# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đỗ Việt Hoàng  **MSSV:** 2A202602882  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    157.6
graph       196     34619     6064   0.00000    265.5

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.33      696       72   0.00000     4.68
graph       0.89   1.83     4417      143   0.00000     5.28
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | 0.00000 | 0.00000 | ×1.0 (cùng miễn phí free tier) |
| Indexing giây | 157.6 | 265.5 | ×1.68 |
| Mỗi câu: USD | 0.00000 | 0.00000 | ×1.0 |
| Mỗi câu: giây | 4.68 | 5.28 | ×1.13 |
| Mỗi câu: in_tok | 696 | 4417 | ×6.3 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí indexing tăng 1.68× do GraphRAG gọi LLM 20 lần để trích xuất 20 bài báo (extract_news_cases), trong khi Flat RAG chỉ embed. Chi phí mỗi câu hỏi tăng 6.3× token input vì GraphRAG prompt dài hơn (kèm facts từ graph). Token output và USD thực tế vẫn ≈0 do dùng Gemini free tier; trên OpenAI gpt-4o-mini sẽ tốn ~$0.001–0.002/câu.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hoà | Cả 2 đều lấy được Điều 2 Luật PCMT từ vector search |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hoà | Cả 2 đều tìm thấy tên 2 bị cáo tử hình trong bài báo |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | **Graph** | Flat RAG thiếu Điều 251 & khung hình phạt; Graph đi qua cầu nối Crime lấy được khoản 1 |
| Q4 | cross-kb | 0.33 / 1 | 0.67 / 1 | **Graph** (recall) | Graph tìm thấy Điều 255 nhưng chỉ lấy khoản 1 (thiếu khoản 4 khung cao nhất) |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | **Graph** | Graph khớp MDMA 9.6kg → Điều 250 khoản 4 (tử hình); Flat thiếu clause & ngưỡng |
| Q6 | aggregation | 0.00 / 1 | 0.67 / 2 | **Graph** | Graph dùng node Substance dùng chung liệt kê 5 vụ có MDMA; Flat miss hoàn toàn |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E2: Thiếu ngữ cảnh luật — Q4 chỉ lấy khoản 1 Điều 255, thiếu khoản 4 (khung 20 năm/chung thân)

- **Hiện tượng:** Câu Q4 hỏi "phạt tù tối đa bao nhiêu", GraphRAG trả lời "khoản 1: 2-7 năm" (judge=1), trong khi gold answer là "tù 20 năm hoặc chung thân" (khoản 4 Điều 255).
- **Bằng chứng:** 
  - Câu trả lời GraphRAG: *"Theo [Điều 255 BLHS] khoản 1, hành vi này có mức phạt tù từ 02 năm đến 07 năm. (Lưu ý: Dữ liệu cung cấp chỉ liệt kê nội dung khoản 1...)"*
  - Cypher kiểm tra Điều 255:
  ```cypher
  MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
  RETURN cl.number, cl.penalty, cl.text LIMIT 10;
  ```
  ```
  kết quả:
  number  penalty                                              text
  1       phạt tù từ 02 năm đến 07 năm                         1. Người nào tổ chức sử dụng trái phép chất ma túy...
  2       phạt tù từ 07 năm đến 15 năm                         2. Phạm tội thuộc một trong các trường hợp sau đây...
  3       phạt tù từ 15 năm đến 20 năm                         3. Phạm tội thuộc một trong các trường hợp sau đây...
  4       phạt tù 20 năm hoặc tù chung thân                    4. Phạm tội thuộc một trong các trường hợp sau đây...
  ```
- **Nguyên nhân:** KG-3 `context()` lọc clause theo điều kiện `cl.number = 1 OR EXISTS { (k)-[:INVOLVES]->(s:Substance)<-[:MENTIONS]-(cl) }`. Điều 255 **không mention substance** (tổ chức sử dụng không quy định ngưỡng chất) → chỉ giữ khoản 1. Đây là lỗi thiết kế ontology: `MENTIONS` chỉ có từ Clause→Substance, mà các tội như 255, 254 không gắn chất.
- **Đề xuất sửa:** Thêm logic fallback: nếu Article không có clause nào MENTIONS substance, lấy thêm clause có penalty cao nhất (khoản cuối). Hoặc mở rộng ontology: thêm property `max_penalty` trên Crime/Article.

---

### Lỗi E3: Trùng thực thể / Liệt kê thừa — Q6 Graph liệt kê 5 vụ thay vì 3 vụ gold

- **Hiện tượng:** Q6 hỏi "vụ việc nào có liên quan đến MDMA", gold chỉ 3 vụ (Cái Quang Huy, Lê Minh Thành, Viện Pháp y). GraphRAG trả về 5 vụ, trong đó có trùng lặp "Vụ vận chuyển >10kg" và "Vụ Cái Quang Huy" (cùng 1 vụ nhưng tách 2 Case node), và thêm "Vụ án sai phạm tại Viện Pháp y" (có 0.686g MDMA nhưng gold không tính).
- **Bằng chứng:**
  - Câu trả lời GraphRAG liệt kê 5 mục (xem file kết quả dòng 67-71).
  - Cypher kiểm tra trùng Case/Substance:
  ```cypher
  MATCH (k:Case)-[:INVOLVES]->(s:Substance {name:'MDMA'})
  RETURN k.name, k.doc_id, s.name
  ORDER BY k.name;
  ```
  ```
  kết quả:
  name                                    doc_id                              s.name
  "Vụ án Cái Quang Huy..."                news-100260917203001265             MDMA
  "Vụ vận chuyển >10kg ma túy từ Đức..."  news-100260917203001265             MDMA
  "Vụ Lê Minh Thành..."                   news-100260918080821054             MDMA
  "Vụ Viện Pháp y tâm thần..."            news-100260930085028036             MDMA
  "Vụ án sai phạm Viện Pháp y..."         news-100260930085028036             MDMA
  ```
  Cùng 1 bài báo (`news-100260917203001265`) sinh ra 2 Case node; cùng 1 bài (`news-100260930085028036`) sinh ra 2 Case node.
- **Nguyên nhân:** `Case` khóa theo `name` do LLM đặt → cùng 1 bài báo, LLM trích xuất 2 "vụ" khác nhau (vd: lần 1 và lần 2 vận chuyển của Cái Quang Huy) thành 2 Case node riêng. Cũng do LLM tách "vụ án" và "vụ sai phạm" từ 1 bài báo thành 2 Case.
- **Đề xuất sửa:** 
  1. Thêm property `case_id` ổn định = `doc_id` + index, hoặc merge Case theo `doc_id` (1 bài báo = 1 Case chính, sub-events là property).
  2. Post-process: gộp Case cùng `doc_id` và cùng `Substance` khi query aggregation.

## 4. Kết luận (5 điểm)

**Nên dùng Knowledge Graph khi:**
- Câu hỏi **cross-kb** (cần nối tin tức + luật): Graph recall 0.89 vs Flat 0.51, judge 1.83 vs 1.33.
- Câu hỏi **multi-hop** (Q5: tìm clause theo substance + ngưỡng khối lượng): Graph 1.00/2 vs Flat 0.40/1.
- Câu hỏi **aggregation** (Q6: liệt kê tất cả vụ theo chất): Graph 0.67/2 vs Flat 0.00/1.
- Cần **trả lời chính xác điều khoản, ngưỡng pháp lý** — vector search không lấy được cấu trúc Điều/Khoản/Điểm.

**Flat RAG đủ khi:**
- Câu hỏi **single-hop** nằm trong 1 KB: Q1 (luật), Q2 (tin) — cả 2 pipeline recall=1.0, judge=2.
- Ngân sách token/chí phí hạn chế: Graph tốn 6.3× token input, 1.68× thời gian indexing.
- Dữ liệu ít, ít quan hệ liên kết — overhead KG không đáng.

**Điều kiện cụ thể:** Nếu >30% câu hỏi là cross-kb/multi-hop/aggregation, và ngân sách cho phép tăng 1.5–2× chi phí indexing + 6× token/câu → đầu tư KG. Dữ liệu có cấu trúc rõ (luật, quy chuẩn) + văn xuôi (tin, báo cáo) → KG hiệu quả cao.

## 5. Tự kiểm (5 điểm)

```
$ pytest tests/ -q
................................................
48 passed in 0.10s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.1-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 201 node / 388 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00000. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: **Trần Thanh Tuấn**

## Vấn đề gặp phải (không tính điểm)

1. **Gemini model deprecated**: Model mặc định `gemini-2.5-flash-lite` bị 404. Fix: thêm `GEMINI_CHAT_MODEL=gemini-3.1-flash-lite` trong `.env`.
2. **Windows encoding**: `UnicodeEncodeError` khi in tiếng Việt. Fix: `$env:PYTHONIOENCODING="utf-8"` (PowerShell).
3. **Rate limit Gemini free tier**: 5 requests/min → chờ 30s giữa các lần chạy benchmark.
4. **KG-3 lọc clause**: Logic `cl.number=1 OR MENTIONS substance` bỏ sót các Điều không mention substance (Điều 254, 255). Đã ghi lỗi E2.
5. **Trùng Case node**: LLM trích xuất 1 bài báo thành 2 Case. Đã ghi lỗi E3.