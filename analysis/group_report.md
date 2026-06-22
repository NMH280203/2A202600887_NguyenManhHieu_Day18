# Group Report — Lab 18: Production RAG

**Sinh viên:** Nguyễn Mạnh Hiếu (2A202600887)  
**Ngày:** 22/06/2026

## Thành viên & Phân công

| Tên | Module | Hoàn thành | Tests pass |
|-----|--------|-----------|-----------|
| Nguyễn Mạnh Hiếu | M1: Chunking | ☑ | 13/13 |
| Nguyễn Mạnh Hiếu | M2: Hybrid Search | ☑ | 5/5 |
| Nguyễn Mạnh Hiếu | M3: Reranking | ☑ | 5/5 |
| Nguyễn Mạnh Hiếu | M4: Evaluation | ☑ | 4/4 |
| Nguyễn Mạnh Hiếu | M5: Enrichment | ☑ | 10/10 |

*Bài tập cá nhân — một người implement toàn bộ 5 modules.*

## Kết quả RAGAS

| Metric | Naive | Production | Δ |
|--------|-------|-----------|---|
| Faithfulness | 0.8267 | 0.6458 | -0.1808 |
| Answer Relevancy | 0.7576 | 0.7242 | -0.0334 |
| Context Precision | 0.9250 | 0.8875 | -0.0375 |
| Context Recall | 0.9250 | 0.7917 | -0.1333 |

## Key Findings

1. **Biggest improvement:** Hybrid search (BM25 + Dense + RRF) + CrossEncoder rerank giữ context_precision cao (~0.89) trên corpus tiếng Việt có từ ghép.
2. **Biggest challenge:** Production pipeline có faithfulness thấp hơn naive do prompt strict + version conflict (v2023 vs v2024) + enrichment làm nhiễu số liệu.
3. **Surprise finding:** Thêm nhiều bước (enrichment, rerank) không luôn cải thiện end-to-end scores — cần đo từng bước riêng.

## Presentation Notes (5 phút)

1. RAGAS scores (naive vs production): Naive thắng faithfulness; Production vẫn đạt context_precision ≥ 0.75.
2. Biggest win — module nào, tại sao: M2 Hybrid Search — underthesea segmentation + RRF giải quyết vocabulary gap tiếng Việt.
3. Case study — 1 failure, Error Tree walkthrough: Câu "nghỉ phép năm" — context đúng nhưng LLM trả "Không tìm thấy" vì 2 version conflict.
4. Next optimization nếu có thêm 1 giờ: Version metadata filter + parent retrieval cho multi-hop queries.
