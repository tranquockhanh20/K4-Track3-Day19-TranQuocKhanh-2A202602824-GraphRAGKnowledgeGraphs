# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Trần Quốc Khánh

**MSSV:** 2A202602824

**Ngày:** 05/10/2026

Các số liệu dưới đây được lấy trực tiếp từ `ket_qua_benchmark_kg.txt`, chạy với `openai:gpt-4o-mini`, embedding `openai:text-embedding-3-small`, `top_k=3`, `chunk_size=800`. Corpus sau chunking có 176 chunk; Knowledge Graph đầy đủ có 204 node và 382 relationship.

## 1. Chi phí (10 điểm)

### Indexing — chi phí một lần

```text
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     53.9
graph       196     91958     4628   0.00928    116.4
```

### Querying — trung bình mỗi câu hỏi

```text
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     3.12
graph       0.89   1.83     7554       75   0.00117     2.81
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | ---: | ---: | ---: |
| Indexing USD | 0.00112 | 0.00928 | 8.29× |
| Indexing giây | 53.9 | 116.4 | 2.16× |
| Mỗi câu: USD | 0.00013 | 0.00117 | 9.00× |
| Mỗi câu: giây | 3.12 | 2.81 | 0.90× |
| Mỗi câu: input token | 694 | 7,554 | 10.88× |

**Chi phí tăng thêm đến từ đâu?** GraphRAG phải gọi chat model thêm 20 lần để trích xuất cấu trúc từ 20 bài báo trong lúc indexing; số call tăng từ 176 lên 196 và phát sinh 4,628 output token. Khi querying, prompt GraphRAG chứa cả ba vector chunk lẫn tối đa 60 dữ kiện graph, làm input token trung bình cao hơn 10.88 lần và chi phí cao hơn 9 lần. Độ trễ truy vấn đo được lại thấp hơn nhẹ (2.81 so với 3.12 giây), nhưng đây chỉ là một lần chạy và bị ảnh hưởng bởi độ trễ API.

Với số liệu hiện tại không có điểm hòa vốn thuần về USD: GraphRAG vừa có chi phí dựng thêm `0.00928 - 0.00112 = 0.00816 USD`, vừa tốn thêm khoảng `0.00117 - 0.00013 = 0.00104 USD` cho mỗi câu. Lợi ích cần được đánh giá bằng chất lượng: recall trung bình tăng từ 0.43 lên 0.89 và judge tăng từ 1.00 lên 1.83.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao |
| --- | --- | ---: | ---: | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Một đoạn luật duy nhất đã đủ định nghĩa “tiền chất”; graph chỉ bổ sung trích dẫn Điều 2 khoản 4. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên hai bị cáo và mức án cùng nằm trong một bài báo nên vector retrieval đã đủ. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Flat chỉ trả “Không đủ thông tin”; graph nối Lê Minh Thành → vụ → tội → Điều 251 → khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nối alias “Hoàng Nato” đến tội tổ chức sử dụng trái phép chất ma túy và Điều 255, bao gồm khung cao nhất. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 1.00 / 2 | Graph | Flat nhầm “khoản b”, còn graph kết hợp tội danh, MDMA 9.6kg và Điều 250 để xác định đúng khoản 4. |
| Q6 | aggregation | 0.00 / 1 | 0.33 / 1 | Graph, nhưng chưa đạt | Graph có lợi thế tổng hợp theo node `MDMA`, nhưng câu trả lời vẫn bỏ sót một `Case` và không nêu đúng các tên bắt buộc. |

Quy luật quan sát được: Flat RAG đủ cho câu single-hop khi toàn bộ đáp án nằm trong một tài liệu (Q1, Q2). GraphRAG thắng rõ khi câu hỏi phải nối người/vụ án trong tin với tội danh, Điều luật và khoản hình phạt ở KB luật (Q3–Q5). Với câu aggregation Q6, có graph chưa đủ; bước tổng hợp của LLM và chất lượng định danh `Case` vẫn có thể làm mất kết quả.

## 3. Phân tích lỗi (20 điểm)

### Lỗi E3: Trùng thực thể `Substance`

- **Hiện tượng:** cùng một chất ngoài đời được tạo thành nhiều node do khác chữ hoa/thường hoặc dùng tên thông dụng. Ví dụ `Ketamine` và `ketamine`, `Methamphetamine` và `methamphetamine`, đồng thời `MDMA` và `thuốc lắc` chưa được nối bằng alias.
- **Bằng chứng:** truy vấn và kết quả trên graph đầy đủ sau benchmark:

```cypher
MATCH (s:Substance)
RETURN s.name AS name
ORDER BY toLower(s.name);
```

```text
Amphetamine
chất ma túy
Cocaine
côca
cần sa
etomidate
Heroine
Ketamine
ketamine
ma túy
ma túy tổng hợp
MDMA
Methamphetamine
methamphetamine
thuốc lắc
thuốc phiện
XLR-11
```

- **Nguyên nhân:** lỗi nằm ở thiết kế khóa định danh và bước trích xuất KG-2. `Substance` đang `MERGE` trực tiếp theo chuỗi `name`, trong khi Neo4j phân biệt hoa/thường. Prompt yêu cầu dùng tên chuẩn nhưng LLM không luôn tuân thủ; code cũng chưa gọi một `link_entity`/normalizer riêng cho chất và chưa có bảng alias như `thuốc lắc → MDMA`. Các tên quá chung như “ma túy” cũng được chấp nhận làm node.
- **Đề xuất sửa:** thêm `normalize_substance` và bảng alias trong `src/graph.py`; đưa mọi chất từ tin qua `link_entity(..., SUBSTANCES, normalize_substance)` trước `add_news_case`, loại các mention chung không xác định được. Có thể lưu tên báo chí trong property `aliases`. Cách này gần như không tăng token hoặc số call nhưng cần quản lý từ điển alias, và ánh xạ quá mạnh có nguy cơ gộp sai một chất mới.

### Lỗi E5: LLM tổng hợp không hết các vụ MDMA có trong graph

- **Hiện tượng:** câu trả lời GraphRAG Q6 liệt kê ba vụ nhưng graph thực tế trả về bốn node `Case` liên quan trực tiếp đến `MDMA`. Câu trả lời đã bỏ “Vụ án tại Viện Pháp y tâm thần Trung ương”; đồng thời “Vụ tổ chức sử dụng ma túy tại Sầm Sơn” có thể là một sự kiện liên quan trong cùng chuỗi vụ án, cho thấy nhu cầu định danh/gộp sự kiện rõ hơn.
- **Bằng chứng từ câu trả lời Q6 GraphRAG:** hệ thống chỉ liệt kê “Vụ góp tiền mua ma túy tại Hà Nội”, “Vụ tổ chức sử dụng ma túy tại Sầm Sơn” và “Vụ vận chuyển ma túy từ Đức về Việt Nam”. Truy vấn trực tiếp graph cho kết quả:

```cypher
MATCH (s:Substance {name:'MDMA'})<-[:INVOLVES]-(k:Case)
OPTIONAL MATCH (p:Person)-[:INVOLVED_IN]->(k)
RETURN k.name AS case_name, k.doc_id AS doc_id,
       collect(DISTINCT p.name) AS people
ORDER BY doc_id;
```

```text
Vụ vận chuyển ma túy từ Đức về Việt Nam       | news-100260917203001265
Vụ góp tiền mua ma túy tại Hà Nội             | news-100260918080821054
Vụ án tại Viện Pháp y tâm thần Trung ương     | news-100260924105118645
Vụ tổ chức sử dụng ma túy tại Sầm Sơn          | news-100260930085028036
```

- **Nguyên nhân:** dữ kiện đã tồn tại trong graph nên lỗi nằm sau retrieval, chủ yếu ở prompt trả lời/khả năng tổng hợp của LLM. Prompt hiện yêu cầu trả lời chung nhưng không bắt buộc kiểm đếm và liệt kê từng `Case`. Ngoài ra `Case.name` do LLM đặt chưa có event ID ổn định, nên hai bài liên quan Viện Pháp y tâm thần có thể bị hiểu là trùng hoặc bị gộp không nhất quán.
- **Đề xuất sửa:** với câu aggregation, dùng Cypher chuyên biệt trả `DISTINCT k.doc_id, k.name` và tạo một fact đánh số cho từng dòng; bổ sung vào prompt yêu cầu “liệt kê đủ N/N vụ, không tự bỏ hoặc gộp”. Về ontology, thêm khóa sự kiện ổn định hoặc quan hệ `SAME_CASE_AS` sau entity resolution. Đổi lại cần nhận diện loại câu hỏi, thêm nhánh truy vấn và có thể tăng độ dài prompt; entity resolution cũng có nguy cơ gộp nhầm hai sự kiện liên quan nhưng khác nhau.

## 4. Kết luận (5 điểm)

Knowledge Graph đáng dùng khi dữ liệu nằm ở nhiều nguồn và câu hỏi cần đi qua quan hệ rõ ràng, chẳng hạn người → vụ án → tội danh → Điều luật → khoản hình phạt. Trong benchmark này, GraphRAG nâng recall trung bình từ 0.43 lên 0.89 và điểm judge từ 1.00 lên 1.83; riêng Q3 và Q4 tăng từ recall 0 lên 1, còn Q5 tăng từ 0.60 lên 1.00.

Flat RAG là lựa chọn hợp lý cho câu single-hop có đáp án nằm trọn trong một tài liệu: Q1 và Q2 đều đạt recall 1.00, judge 2 ở cả hai pipeline. Flat cũng rẻ hơn khoảng 9 lần mỗi câu và chi phí indexing thấp hơn 8.29 lần. Vì vậy nên dùng GraphRAG khi giá trị của câu hỏi xuyên nguồn và yêu cầu truy vết cao hơn chi phí; với tra cứu đơn tài liệu hoặc khối lượng truy vấn lớn nhưng ngân sách thấp, Flat RAG đủ phù hợp. Với câu tổng hợp, cần Cypher chuyên biệt và entity resolution thay vì giả định LLM sẽ luôn tổng hợp đầy đủ.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.06s
```

```text
$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 14 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j cần đặt tại `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.

Người đã chọn cho `kg_my_case.png`: **Cái Quang Huy**. Đường đi đã kiểm chứng:

```text
Cái Quang Huy → Vụ vận chuyển ma túy từ Đức về Việt Nam
→ vận chuyển trái phép chất ma túy → Điều 250 BLHS
```

## Vấn đề gặp phải (không tính điểm)

Môi trường tự động hóa hiện tại không cung cấp trình duyệt có thể điều khiển, nên chưa thể tạo ba ảnh chụp Neo4j Browser đúng quy cách (phải thấy ô truy vấn và Results overview). Graph đầy đủ vẫn đang có trong Neo4j và các truy vấn đã được kiểm chứng bằng `cypher-shell`; cần mở `http://localhost:7474` và chụp ba ảnh theo Bước 8.2 trước khi nộp.
