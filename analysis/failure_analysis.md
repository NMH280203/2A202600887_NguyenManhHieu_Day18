# Failure Analysis — Lab 18: Production RAG

**Sinh viên:** Nguyễn Mạnh Hiếu (2A202600887)  
**Pipeline:** Hierarchical chunking + M5 Enrichment (combined) + Hybrid Search + CrossEncoder Rerank

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.8267 | 0.6458 | -0.1808 |
| Answer Relevancy | 0.7576 | 0.7242 | -0.0334 |
| Context Precision | 0.9250 | 0.8875 | -0.0375 |
| Context Recall | 0.9250 | 0.7917 | -0.1333 |

**Nhận xét:** Production pipeline cải thiện retrieval (context_precision vẫn cao ~0.89) nhưng faithfulness giảm do prompt quá strict ("Không tìm thấy") và enrichment làm nhiễu context. Naive baseline đơn giản hơn nên LLM ít từ chối trả lời hơn.

---

## Bottom-5 Failures

### #1 — Nghỉ phép năm (version conflict)
- **Question:** Nhân viên được nghỉ bao nhiêu ngày phép năm?
- **Expected:** Theo chính sách hiện hành (v2024), nhân viên được nghỉ 15 ngày phép năm có lương. Chính sách cũ (v2023) là 12 ngày nhưng đã bị thay thế.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng (recall=1.0) → Query OK → **Root cause:** Prompt yêu cầu trả lời "Không tìm thấy" khi context có cả v2023 (12 ngày) và v2024 (15 ngày) gây mâu thuẫn; LLM từ chối trả lời thay vì chọn version mới.
- **Suggested fix:** Thêm metadata `version`/`effective_date` khi chunking; filter retrieval ưu tiên document mới nhất; sửa prompt để ưu tiên chính sách hiện hành.

### #2 — Bảo hiểm thử việc
- **Question:** Nhân viên thử việc có được hưởng bảo hiểm sức khỏe PVI không?
- **Expected:** KHÔNG. Nhân viên thử việc chưa được hưởng gói bảo hiểm sức khỏe PVI. Chỉ được tham gia bảo hiểm xã hội bắt buộc.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng (precision≈1.0, recall=1.0) → Query OK → **Root cause:** Context chứa thông tin phủ định ("chưa được hưởng") nhưng prompt strict khiến LLM không diễn giải câu trả lời phủ định.
- **Suggested fix:** Cải thiện prompt: cho phép trả lời "Không" / "Chưa được" khi context có bằng chứng; thêm few-shot examples cho câu hỏi yes/no.

### #3 — Lương thử việc Junior
- **Question:** Lương thử việc của nhân viên Junior mức cao nhất là bao nhiêu?
- **Expected:** Junior cao nhất là 20.000.000 VNĐ/tháng. Lương thử việc = 85% × 20.000.000 = 17.000.000 VNĐ/tháng.
- **Got:** Không tìm thấy.
- **Worst metric:** faithfulness (0.0)
- **Error Tree:** Output sai → Context đúng → Query OK → **Root cause:** Câu hỏi multi-hop (bảng lương + chính sách thử việc 85%); context có từng phần nhưng LLM không tự tính toán.
- **Suggested fix:** Thêm chain-of-thought trong prompt; hoặc pre-compute metadata `probation_salary` khi enrichment.

### #4 — Hạn mức bảo hiểm PVI
- **Question:** Bảo hiểm sức khỏe PVI có hạn mức bao nhiêu cho nhân viên?
- **Expected:** Hạn mức bảo hiểm sức khỏe PVI cho nhân viên là 200.000.000 VNĐ/năm, bao gồm nội trú, ngoại trú và nha khoa.
- **Got:** Không tìm thấy.
- **Worst metric:** answer_relevancy (0.0)
- **Error Tree:** Output sai → Context thiếu chính xác (precision=0.33) → Query OK → **Root cause:** Reranker đưa chunk không chứa số 200 triệu lên top; enrichment summary che mất con số quan trọng.
- **Suggested fix:** Giữ nguyên số liệu trong enriched_text; boost BM25 cho query có từ "hạn mức", "PVI".

### #5 — Multi-hop: thâm niên + lương Senior
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** Theo chính sách v2024: 15 ngày cơ bản + 3 ngày thâm niên (9÷3=3) = 18 ngày phép. Lương Senior (P3-P4): 20-35 triệu VNĐ/tháng.
- **Got:** Nhân viên Senior có 9 năm thâm niên sẽ được nghỉ tổng cộng 18 ngày phép (12 ngày phép năm + 6 ngày phép cộng thêm cho 9 năm thâm niên). Tuy nhiên, thông tin về lương không được đề cập trong đoạn văn.
- **Worst metric:** context_recall (0.0)
- **Error Tree:** Output một phần đúng → Context thiếu chunk lương (recall=0.0 cho phần lương) → Query multi-hop → **Root cause:** Hierarchical child chunk chỉ chứa phần nghỉ phép; chunk bảng lương không được retrieve cùng lúc.
- **Suggested fix:** Parent retrieval (retrieve child → return parent); hoặc merge chunks từ 2 nguồn khi query có nhiều entity.

---

## Case Study (cho presentation)

**Question chọn phân tích:** Nhân viên được nghỉ bao nhiêu ngày phép năm?

**Error Tree walkthrough:**
1. Output đúng? → **Không** — trả "Không tìm thấy" thay vì 15 ngày
2. Context đúng? → **Có** — context_recall = 1.0, có cả `nghi_phep_nam_v2023.md` và `nghi_phep_nam_v2024.md`
3. Query rewrite OK? → **Có** — BM25 + Dense đều match "nghỉ phép"
4. Fix ở bước: **Generation** — thêm version filter + prompt ưu tiên policy mới nhất

**Nếu có thêm 1 giờ, sẽ optimize:**
- Thêm metadata `superseded_by` cho document cũ
- Giảm enrichment noise (không prepend summary che số liệu)
- Điều chỉnh prompt: "Nếu có nhiều version, dùng version mới nhất"
