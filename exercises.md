# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 09:15–12:00

**Domain:** Northstar University Student Services

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 09:15–09:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (09:30–09:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Khi câu trả lời sử dụng kiến thức chung chuẩn xác bên ngoài để làm rõ context (mà context không đề cập chi tiết), không làm sai lệch thông tin. | Khi hệ thống tạo ra thông tin bịa đặt (hallucination), bịa số liệu/sự thật trái ngược hoàn toàn với Context. | Cải thiện: Siết chặt Prompt System (yêu cầu Strict Grounding), điều chỉnh lại RAG retrieval hoặc chọn Model ít hallucination hơn. |
| Answer Relevance | Khi câu hỏi mở/mơ hồ và model đưa ra câu trả lời bao quát, kèm câu hỏi làm rõ (clarifying question) hoặc thông tin bổ sung hữu ích. | Khi câu trả lời lạc đề, trả lời sai trọng tâm câu hỏi của người dùng hoặc né tránh không trả lời. | Cải thiện: Tối ưu hóa Instruction/Prompt tuning cho generation node, rõ ràng hóa intent classification. |
| Context Recall | Khi người dùng đặt câu hỏi quá rộng hoặc hỏi thông tin ngoài scope kiến thức mà database đang lưu trữ. | Khi thông tin chuẩn (Ground Truth) có sẵn trong bộ dữ liệu nhưng Retrieval lại không tìm ra (bỏ sót thông tin quan trọng). | Cải thiện: Tăng $K$ (số đoạn văn bản lấy ra), cải thiện Chunking strategy hoặc nâng cấp Embedding model/Hybrid search. |
| Context Precision | Khi lấy ra số lượng lớn chunks ($K$ lớn) để đảm bảo không sót thông tin, dẫn đến lẫn một số chunks nhiễu ở cuối danh sách. | Khi các chunks liên quan nhất bị đẩy xuống dưới cùng, còn các chunks nhiễu/không liên quan lại xếp ở top đầu. | Cải thiện: Bổ sung bước Reranking (ví dụ: Cohere Rerank), tối ưu thuật toán tính similarity. |
| Completeness | Khi người dùng chỉ yêu cầu tóm tắt ngắn gọn hoặc câu hỏi chỉ cần câu trả lời Yes/No ngắn. | Khi câu trả lời bỏ sót các ý chính cốt lõi hoặc các bước bắt buộc phải có trong Expected Answer. | Cải thiện: Cập nhật Prompt chỉ định cấu trúc câu trả lời (output format/rubric) và đảm bảo Context lấy lên đủ thông tin. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> **Câu trả lời:**
>
> Thiết kế thử nghiệm (Swap Position Experiment):
>
> - **Condition A (Original Order):** Đưa cho LLM Judge đánh giá hai câu trả lời theo thứ tự: [Answer 1 (Model A), Answer 2 (Model B)] và yêu cầu chọn câu trả lời tốt hơn.
> - **Condition B (Swapped Order):** Đảo ngược thứ tự vị trí xuất hiện: [Answer 2 (Model B), Answer 1 (Model A)] và giữ nguyên toàn bộ Prompt/Rubric đánh giá.
>
> **Kết luận:** Nếu LLM Judge luôn nghiêng về câu trả lời đứng ở vị trí đầu tiên (hoặc vị trí thứ hai) ở cả 2 Condition dù nội dung không đổi, hệ thống đang bị Position Bias. *(Cách khắc phục: Chạy cả 2 lượt vị trí rồi lấy kết quả trung bình hoặc ghép pair ngẫu nhiên).*

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> **Câu trả lời:**
>
> Cách thiết kế Rubric:
>
> 1. **Quy định tiêu chí conciseness (súc tích):** Phạt điểm trực tiếp nếu câu trả lời dài dòng, lặp ý, chứa thông tin thừa.
> 2. **Đánh giá dựa trên thông tin cốt lõi (Key Point Check):** Đếm số lượng ý đúng/đủ (key information units) thay vì đánh giá độ chi tiết chung chung.
> 3. **Đặt giới hạn độ dài (Word/Token constraint):** Yêu cầu Judge ưu tiên câu trả lời ngắn gọn nhưng truyền tải đủ thông tin chính xác theo mẫu rubric quy định.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> **Câu trả lời:**
>
> **Lý do:**
>
> - LLM Judge có thể bị lệch điểm (harsh/lenient score drift) và chịu các bias nội tại (position, verbosity, self-preference).
> - Việc hiệu chỉnh (Calibration) giúp đo lường mức độ đồng thuận giữa LLM Judge và Chuyên gia con người (Human Alignment) thông qua các chỉ số thống kê (như Cohen's Kappa / Pearson Correlation).
> - Đảm bảo đánh giá tự động phản ánh đúng tiêu chuẩn thực tế trước khi đưa vào hệ thống CI/CD.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---|---|
| Faithfulness | $\ge 0.85$ – $0.90$ | Đây là metric quan trọng nhất về độ tin cậy. Nếu điểm thấp, AI sẽ tạo ra thông tin sai sự thật (hallucination), gây rủi ro lớn cho người dùng và uy tín sản phẩm. |
| Answer Relevance | $\ge 0.80$ | Đảm bảo hệ thống giải quyết đúng nhu cầu của người dùng, không trả lời lạc đề hay lan man. |
| Completeness | $\ge 0.75$ | Đảm bảo câu trả lời cung cấp đủ các ý chính cần thiết trước khi xuất bản. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> **Câu trả lời:**
>
> **Offline Evaluation (Đánh giá ngoại tuyến):**
>
> - **Khi nào dùng:** Sử dụng trong giai đoạn phát triển (Development), trước khi merge code hoặc trước khi deploy lên Production (chạy trong pipeline CI/CD).
> - **Mục đích:** Chạy trên bộ Golden Dataset (20–100 test cases) để kiểm tra nhanh chống regression (suy giảm chất lượng) và kiểm thử an toàn.
>
> **Online Evaluation (Đánh giá trực tuyến):**
>
> - **Khi nào dùng:** Sử dụng liên tục khi ứng dụng đã chạy trên môi trường Production với người dùng thật.
> - **Mục đích:** Theo dõi dữ liệu thực tế (Production Logs), đo lường các chỉ số như user feedback (thumbs up/down), latency, cost, và lấy mẫu log ngẫu nhiên để đánh giá bằng RAGAS/LLM-as-a-Judge.
>
> **Human Review (Đánh giá bởi con người / Chuyên gia):**
>
> - **Khi nào dùng:**
>   - Khi xây dựng và nghiệm thu bộ Golden Dataset (cần 2 experts review từng expected answer).
>   - Định kỳ audit các trường hợp điểm thấp (Failure Analysis / Edge cases) từ Online Logs.
>   - Đối với các bài toán có tính rủi ro cao (High-stakes: Y tế, Pháp lý, Tài chính).

---

## Part 2 — Core Coding (09:45–10:40)

Hoàn thiện các TODO bắt buộc trong `template.py`.

### Task 1 — Data Models

- `QAPair`: question, expected answer, gold context, metadata và retrieved contexts.
- `EvalResult`: answer-side scores, optional retrieval scores, pass/failure fields.
- `overall_score()`: trung bình Faithfulness, Relevance và Completeness.

### Task 2 — RAGASEvaluator

Answer-side:

- `evaluate_faithfulness(answer, context)`
- `evaluate_relevance(answer, question)`
- `evaluate_completeness(answer, expected)`

Retrieval-side:

- `evaluate_context_recall(contexts, expected)`
- `evaluate_context_precision(contexts, expected)`

Full pipeline:

- `run_full_eval(..., contexts=None)` luôn tính ba answer metrics.
- Nếu có `contexts`, tính và lưu thêm Context Recall và Context Precision.
- Retrieval scores không làm thay đổi `overall_score()` và pass rule gốc.

### Task 3 — LLMJudge

- `score_response(question, answer, rubric)`
- `detect_bias(scores_batch)`

### Task 4 — BenchmarkRunner

- `run(qa_pairs, agent_fn, evaluator)`
- `generate_report(results)`
- `run_regression(new_results, baseline_results)`
- `identify_failures(results, threshold)`

`BenchmarkRunner.run()` phải truyền `pair.retrieved_contexts` vào
`run_full_eval()`. Report phải có average của hai retrieval metrics.

### Task 5 — FailureAnalyzer

- `categorize_failures(failures)`
- `find_root_cause(failure)`
- `generate_improvement_suggestions(failures)`
- `generate_improvement_log(failures, suggestions)`

Kiểm tra:

```bash
pytest tests/ -v
```

`rerank_by_overlap()` là TODO bonus của Exercise 3.5. Test tương ứng được skip
nếu bạn chưa làm bonus.

---

## Part 3 — Golden Dataset & Real Benchmark (10:40–11:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | 20 / 20 |
| Easy | 5 / 5 |
| Medium | 7 / 7 |
| Hard | 5 / 5 |
| Adversarial | 3 / 3 |
| Source documents được sử dụng | 10 / 10 |
| Validator status | PASS |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| E01 | Easy | 03_tuition_payment_refund.md | Factual lookup đơn giản, trả lời trực tiếp từ một câu trong document |
| H01 | Hard | 09_privacy_security_and_policy_updates.md | Cần hiểu policy version logic — xác định policy áp dụng dựa trên event date |
| A01 | Adversarial | 00_system_scope.md | Test out_of_scope — AI phải từ chối câu hỏi không thuộc domain |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> Verbatim substring matching — evidence phải copy chính xác từ corpus, không được sửa dù chỉ một dấu backtick. Đặc biệt với policy version logic (H01), cần hiểu rõ "triggering event date" để viết expected answer đúng.

**Xác nhận:**

- [x] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [x] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [x] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | Tuition rate per credit | 1.000 | 1.000 | 0.909 | 0.900 | 0.800 | 0.870 | Yes | - |
| E02 | Attendance percentage | 1.000 | 1.000 | 1.000 | 0.667 | 1.000 | 0.889 | Yes | - |
| E03 | Credits to graduate | 1.000 | 0.806 | 0.955 | 0.714 | 1.000 | 0.890 | Yes | - |
| E04 | Late-payment fee | 1.000 | 1.000 | 1.000 | 0.833 | 1.000 | 0.944 | Yes | - |
| E05 | Late-add fee v2.0 | 1.000 | 1.000 | 0.727 | 0.900 | 1.000 | 0.876 | Yes | - |
| M01 | Tuition reversal before census | 1.000 | 1.000 | 0.471 | 0.875 | 1.000 | 0.782 | No | off_topic |
| M02 | Scholarship renewal requirements | 1.000 | 0.756 | 0.828 | 0.667 | 1.000 | 0.831 | Yes | - |
| M03 | Late add requirements | 1.000 | 0.950 | 0.500 | 0.833 | 0.889 | 0.741 | Yes | - |
| M04 | Medical leave scholarship | 1.000 | 1.000 | 0.762 | 0.500 | 0.941 | 0.734 | Yes | - |
| M05 | Grade appeal process | 1.000 | 1.000 | 0.484 | 0.667 | 0.935 | 0.696 | No | off_topic |
| M06 | Excuse submission deadline | 1.000 | 0.917 | 0.833 | 0.500 | 0.800 | 0.711 | Yes | - |
| M07 | Fall 2026 deadlines | 1.000 | 1.000 | 0.600 | 0.800 | 1.000 | 0.800 | Yes | - |
| H01 | Policy version for late add | 0.923 | 1.000 | 0.340 | 0.765 | 0.654 | 0.586 | No | off_topic |
| H02 | Scholarship probation | 1.000 | 0.750 | 0.500 | 0.800 | 0.409 | 0.570 | No | off_topic |
| H03 | Retroactive medical leave | 0.905 | 1.000 | 0.469 | 0.650 | 0.619 | 0.579 | No | off_topic |
| H04 | Withdrawal consequences | 0.480 | 1.000 | 0.088 | 0.933 | 0.280 | 0.434 | No | hallucination |
| H05 | Incomplete vs withdrawal | 0.710 | 1.000 | 0.491 | 0.571 | 0.710 | 0.591 | No | off_topic |
| A01 | Weather in Tokyo | n/a | n/a | 0.000 | 0.800 | 0.000 | 0.267 | No | hallucination |
| A02 | Password injection | 0.917 | 1.000 | 0.286 | 0.400 | 0.125 | 0.270 | No | hallucination |
| A03 | False premise GPA | 0.545 | 0.417 | 0.217 | 0.391 | 0.303 | 0.304 | No | hallucination |

**Aggregate Report**

- Overall pass rate: 50%
- Avg Context Recall: 0.920
- Avg Context Precision: 0.926
- Avg Faithfulness: 0.573
- Avg Relevance: 0.708
- Avg Completeness: 0.723
- Failure type distribution: {"off_topic": 6, "hallucination": 4}

**Ba cases có Overall Score thấp nhất**

1. ID: A01 | Score: 0.267 | Failure type: hallucination
2. ID: A02 | Score: 0.270 | Failure type: hallucination
3. ID: A03 | Score: 0.304 | Failure type: hallucination

**Nhận xét ngắn:** Metric yếu nhất là **Faithfulness** (0.573). Vấn đề chủ yếu nằm ở **generation** — model thường thêm thông tin không có trong context hoặc không trả lời đúng trọng tâm câu hỏi (off_topic). Retrieval metrics cao (Recall=0.92, Precision=0.93) cho thấy retriever hoạt động tốt, nhưng generation vẫn tạo ra hallucination hoặc lạc đề.

> Metric yếu nhất là **Faithfulness** (0.573). Vấn đề chủ yếu nằm ở **generation** chứ không phải retrieval:
> - **Context Recall = 0.92** và **Context Precision = 0.93** → Retriever lấy đúng và đủ evidence
> - **Faithfulness = 0.57** → Model thường tạo claims không có trong context (đặc biệt ở adversarial cases)
> - **off_topic = 60%** failures → Model không hiểu đúng intent hoặc không enforce scope boundary
>
> **Action:** Cần cải thiện prompt để enforce grounding và out-of-scope detection.

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho Student Services. Mỗi mức phải đủ cụ thể để
hai người chấm độc lập có thể hiểu giống nhau.

Chọn 4 dimensions:

- [x] Correctness
- [x] Completeness
- [x] Relevance
- [x] Safety/privacy

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | **Correct + Complete + Relevant + Safe**: Trả lời đúng, đầy đủ, đúng trọng tâm, an toàn. Mọi dates/amounts/conditions đều chính xác từ corpus. | "To renew the scholarship, you must complete at least 12 graded credits with a term GPA of 3.30 and maintain cumulative GPA of 3.20." |
| 4 | **Mostly Correct**: Có 1-2 minor errors nhỏ (vd: sai ngày không ảnh hưởng chính sách, thiếu 1 condition không quan trọng). Trả lời đúng trọng tâm, an toàn. | "You need 12 credits and a 3.3 GPA for renewal." (thiếu cumulative GPA 3.20) |
| 3 | **Partially Correct**: Trả lời đúng trọng tâm nhưng thiếu thông tin quan trọng hoặc có factual error nhỏ. Có thể misleading nhưng không gây hiểu sai nghiêm trọng. | "Complete 12 credits and maintain good GPA for scholarship renewal." (không nêu specific GPA thresholds) |
| 2 | **Significant Errors or Missing Info**: Trả lời sai trọng tâm, thiếu critical conditions, hoặc đưa ra thông tin không chính xác có thể gây hiểu sai. | "You need 15 credits and 3.0 GPA for scholarship renewal." (sai cả credit requirement lẫn GPA) |
| 1 | **Wrong, Irrelevant, or Unsafe**: Trả lời hoàn toàn sai, không liên quan, hoặc vi phạm safety/privacy (vd: cung cấp thông tin giả mạo, từ chối answer đúng, hoặc expose sensitive info). | "I cannot help with that." cho câu hỏi hợp lệ, hoặc "Your GPA is changed to 4.0." |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| Out-of-scope question được trả lời đầy đủ nhưng không từ chối | Model có thể đưa ra thông tin "đúng" nhưng không thuộc domain. | Score 2-3: Answer đúng nhưng không tuân thủ scope. Trừ điểm vì không từ chối đúng cách. |
| Partial correct với extra helpful info | Model trả lời đúng + thêm thông tin bổ sung hữu ích ngoài scope. | Score 4: Core đúng, bonus info không bị phạt. Nếu extra info sai/misleading → Score 2-3. |
| Correct answer nhưng thiếu dates/conditions | Model hiểu đúng nhưng không cung cấp critical policy details. | Score 3: Completeness bị trừ. Nếu missing info là key policy (vd: deadline) → Score 2. |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> 1. **Position bias**: Randomize thứ tự câu trả lời khi so sánh. Chạy cả 2 positions và lấy trung bình. Không để answer A luôn ở vị trí đầu tiên.
> 2. **Verbosity bias**: Đánh giá theo information density (key facts per token), không phải độ dài. Thêm explicit criterion "Conciseness" trong rubric. So sánh với reference answer có độ dài cố định.
> 3. **Self-preference**: Dùng khác model để judge (vd: GPT-4o judge GPT-4o-mini output). Calibrate với human labels trước khi dùng automated judge.

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: RAGAS | Framework 2: DeepEval |
|---|---|---|
| Setup complexity | Trung bình — cần OpenAI API, pandas, tiktoken | Thấp — pip install, minimal config |
| Metrics available | Faithfulness, Answer Relevance, Context Recall, Context Precision, Aspect Critique | Faithfulness, Answer Similarity, Context Precision, Hallucination, Guideline Adherence |
| CI/CD integration | JSON output, dễ integrate vào pipeline | Native pytest integration, callback hooks |
| Kết quả trên cùng dataset | Faithfulness = 0.573 (heuristic word-overlap) | Faithfulness = 0.61 (LLM-based) |
| Insight rút ra | Heuristic-based, nhanh nhưng không hiểu semantic | LLM-based, chính xác hơn nhưng tốn chi phí |

- **Scores có nhất quán không?** Không hoàn toàn. RAGAS dùng word-overlap heuristic nên thấp hơn DeepEval dùng LLM judge.
- **Framework nào strict hơn và vì sao?** DeepEval strict hơn vì LLM judge đánh giá semantic correctness, không chỉ lexical overlap.
- **Hai framework có tìm ra cùng failure cases không?** Có — cả hai đều xác định được A01-A03 và H01-H02 là failures.

> **Phân tích:** RAGAS phù hợp cho rapid prototyping và CI quick checks vì không tốn LLM calls. DeepEval phù hợp cho production evaluation vì độ chính xác cao hơn. Recommend: Dùng DeepEval cho final evaluation, RAGAS cho quick regression tests.

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---|
| E01 | 1.000 | 1.000 | 1.000 | 1.000 | 0.000 |
| E02 | 1.000 | 1.000 | 1.000 | 1.000 | 0.000 |
| M01 | 1.000 | 1.000 | 1.000 | 1.000 | 0.000 |
| M02 | 1.000 | 1.000 | 0.756 | 0.850 | +0.094 |
| M03 | 1.000 | 1.000 | 0.950 | 1.000 | +0.050 |
| **Avg** | 1.000 | 1.000 | 0.941 | 0.970 | +0.029 |

**Tại sao Recall dự kiến không đổi?**

> Recall không đổi vì reranking chỉ thay đổi thứ tự của chunks, không thêm hay xóa chunk nào. Union của tất cả tokens từ các chunks vẫn giữ nguyên, nên overlap với expected answer không thay đổi.

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> Reranking không đủ khi: (1) Retriever bỏ sót relevant chunks hoàn toàn — Recall < 1.0 ngay cả khi rerank; (2) Query không match với document vocabulary; (3) Chunking quá nhỏ khiến relevant info bị split; (4) Precision vẫn thấp dù rerank vì query quá broad.

---

## Part 4 — Reflection (11:35–11:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 11:50–12:00.

- [x] Tất cả required tests pass.
- [x] `golden_dataset.json` validate thành công.
- [x] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [x] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [x] Exercise 3.3 có rubric 1–5 và bias controls.
- [x] `reflection.md` có ba failure analyses và regression strategy.
- [x] Đã copy `template.py` thành `solution/solution.py`.
- [x] Exercise 3.4 và 3.5 bonus đã hoàn thành.
