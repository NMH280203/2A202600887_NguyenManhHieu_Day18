# Individual Reflection — Lab 18

**Tên:** Nguyễn Mạnh Hiếu  
**MSSV:** 2A202600887

---

## Phần 1: Mapping bài giảng → code

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Threshold 0.85 tạo ~ít chunk hơn basic vì gộp câu cùng chủ đề; threshold 0.5 tạo nhiều chunk hơn cho test |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Parent 2048 / child 256 → 100 children từ 26 docs; child nhỏ giúp precision, parent giữ context |
| Structure-aware chunking | M1 | `chunk_structure_aware()` | Parse `#{1,3}` headers → mỗi section 1 chunk, metadata có `section` |
| BM25 + Vietnamese segmentation | M2 | `segment_vietnamese()`, `BM25Search` | underthesea + replace `_` → BM25 khớp "nghỉ phép" với "nghỉ_phép" |
| Dense + RRF fusion | M2 | `DenseSearch`, `reciprocal_rank_fusion()` | bge-m3 + Qdrant; RRF merge rank lists, doc xuất hiện cả 2 list được boost |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | bge-reranker-v2-m3 ~ vài giây lần đầu load; doc "nghỉ phép" rank cao hơn "VPN" |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Faithfulness thấp nhất (0.65) vì nhiều câu trả "Không tìm thấy"; context_precision cao nhất (0.89) |
| Failure analysis | M4 | `failure_analysis()` | Diagnostic tree map worst_metric → diagnosis + suggested_fix |
| Contextual enrichment | M5 | `_enrich_single_call()` | Combined mode 1 API call/chunk; 100 chunks ~7 phút; 1 chunk JSON parse fail → fallback |

---

## Phần 2: Khó khăn & giải quyết

### Lỗi 1: `command not found: python`
- **Cách debug:** Kiểm tra `.python-version` → dùng `python3.11 -m venv .venv`
- **Giải pháp:** Tạo virtualenv, `pip install -r requirements.txt pytest`

### Lỗi 2: CrossEncoder download chậm (~28 phút cho pytest)
- **Error:** Test `test_rerank_returns` treo khi download `BAAI/bge-reranker-v2-m3`
- **Cách debug:** Chạy pytest từng module, thấy M3 block
- **Giải pháp:** Pre-download model theo README; lần sau model đã cache → tests ~ vài phút

### Lỗi 3: `Enrichment API failed: Expecting value: line 7 column 3`
- **Cách debug:** LLM trả JSON wrapped trong markdown code block
- **Giải pháp:** Strip ` ```json ` trước `json.loads()`; fallback extractive khi parse fail

### Kiến thức thiếu → bổ sung
- Qdrant API mới dùng `query_points()` thay vì `search()` — đọc doc qdrant-client 2.x
- RAGAS cần Python 3.11+ và OPENAI_API_KEY cho metric computation

---

## Phần 3: Action Plan cho project

## Project: Chatbot nội bộ công ty

### Hiện tại
- RAG pipeline hiện tại: basic paragraph chunking + dense search + GPT answer
- Known issues: trả lời sai khi có nhiều version policy; multi-hop queries fail

### Plan áp dụng
1. [x] Chunking strategy: Hierarchical (parent/child) — precision cao, có thể return parent context
2. [x] Search: BM25 + Dense + RRF — tốt cho tiếng Việt và synonym gap
3. [x] Reranking: Có — `bge-reranker-v2-m3` qua CrossEncoder
4. [x] Evaluation: RAGAS 4 metrics + failure analysis diagnostic tree
5. [x] Enrichment: Combined single-call — tiết kiệm API cost nhưng cần validate JSON output

### Timeline
- Tuần 1: Deploy hierarchical chunking + hybrid search lên staging
- Tuần 2: Thêm version metadata cho policy docs; A/B test với/không enrichment
- Tuần 3: Parent retrieval cho multi-hop; chạy RAGAS regression trên test set 30 câu
- Tuần 4: Tune prompt (bỏ "Không tìm thấy" quá aggressive); target faithfulness ≥ 0.80

---

## Tự đánh giá

| Tiêu chí | Tự chấm (1-5) |
|----------|---------------|
| Hiểu bài giảng | 4 |
| Code quality | 4 |
| Problem solving | 4 |
| Hoàn thành lab | 5 |
