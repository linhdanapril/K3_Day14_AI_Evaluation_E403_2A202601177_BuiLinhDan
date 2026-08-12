# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 50%

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.920 | 0.480 | 1.000 | Retriever lấy được hầu hết evidence cần thiết |
| Context Precision | 0.926 | 0.417 | 1.000 | Chất lượng ranking tốt, chunks relevant xếp trên |
| Faithfulness | 0.573 | 0.000 | 1.000 | **Yếu nhất** — model thường thêm thông tin không có trong context |
| Relevance | 0.708 | 0.391 | 0.933 | Khá tốt nhưng vẫn có off_topic cases |
| Completeness | 0.723 | 0.000 | 1.000 | Trung bình — thiếu một số key conditions |
| Overall Score | 0.642 | 0.267 | 0.944 | Trung bình, có room for improvement |

**Score interpretation**

- Metrics/cases ở mức Good (0.8–1.0): 10/20 cases passed
- Metrics/cases ở mức Needs Work (0.6–0.8): 5/20 cases
- Metrics/cases ở mức Significant Issues (<0.6): 5/20 cases

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 4 | 40% |
| irrelevant | 0 | 0% |
| incomplete | 0 | 0% |
| off_topic | 6 | 60% |
| refusal | 0 | 0% |

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở **generation**, không phải retrieval. Dùng ít nhất hai metrics để bảo vệ:
- **Context Recall = 0.92** (cao) + **Context Precision = 0.93** (cao) → Retriever hoạt động tốt
- **Faithfulness = 0.57** (thấp) → Model thường tạo ra nội dung không có trong retrieved context
- **off_topic = 60%** → Model không hiểu đúng intent của câu hỏi

---

## 2. Top 3 Worst Failures — 5 Whys

### Failure 1

**ID và question:** A01 — "What is the weather forecast for Tokyo tomorrow?"

**Expected answer:**
> This question is outside the scope of the Northstar Student Services Assistant. I can help with questions about academic deadlines, course registration, tuition, scholarships, attendance, grading, leave, graduation, appeals, student support, privacy, and account security at Northstar University.

**Actual answer:**
> I'm an AI assistant focused on helping with student services at Northstar University. I can't provide weather forecasts for any location. I can help you with questions about courses, registration, financial matters, academic policies, campus life, and other university services. How can I assist you today?

**Scores:** Context Recall: n/a | Context Precision: n/a | Faithfulness: 0.000 | Relevance: 0.800 | Completeness: 0.000 | Overall: 0.267

**Evidence inspection:** Retriever lấy 0 chunks (out-of-scope nên không có relevant context).

| Level | Question | Answer |
|---|---|---|
| Symptom | Model trả lời weather forecast thay vì từ chối đúng scope | |
| Why 1 | Model không detect được out-of-scope question | |
| Why 2 | Prompt không explicitly định nghĩa rõ scope boundary | |
| Why 3 | Không có explicit out-of-scope detection logic | |
| Why 4 | Assistant được trained để be helpful, override scope check | |
| Why 5 | System prompt cần thêm explicit refusal template cho out-of-scope | |

**Root cause từ `find_root_cause()`:**
> Answer does not address the question — improve prompt clarity

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Đồng ý một phần. Model không đúng scope nhưng cũng không hallucinate hoàn toàn — nó redirect sang university services. Root cause đúng là prompt không enforce scope boundary rõ ràng.

**Proposed fix cụ thể:**
> Thêm explicit system instruction: "If the question is outside these topics [list], respond with: 'This question is outside my scope. I can help with [list of topics].'" và thêm few-shot examples cho out-of-scope cases.

---

### Failure 2

**ID và question:** H04 — "A student withdraws from all courses on September 2, 2026 (before census date September 4). What are the financial and scholarship consequences?"

**Expected answer:**
> For tuition: 50% of that course's tuition is reversed since it's before census date. For scholarship: dropping below 12 graded credits on or before census date triggers an immediate eligibility review, and scholarship funds are adjusted before any student refund is calculated.

**Actual answer:**
> Financial Consequences: Since the withdrawal occurred before the census date (September 4, 2026), the student would typically be eligible for a 50% tuition refund for the withdrawn courses. However, this assumes the student is withdrawing from some but not all courses. If withdrawing from all courses, the university may have different policies regarding refunds and financial aid. Scholarship Implications: A full withdrawal before the census date could affect the student's scholarship. The Northstar Merit Scholarship requires maintaining a full course load. If dropping below 12 credits after the withdrawal, the scholarship review would be triggered. Additionally, scholarship disbursements for future semesters may be affected.

**Scores:** Context Recall: 0.480 | Context Precision: 1.000 | Faithfulness: 0.088 | Relevance: 0.933 | Completeness: 0.280 | Overall: 0.434

**Evidence inspection:** Retriever lấy đúng chunks nhưng model thêm thông tin không có trong context (ví dụ: "different policies regarding refunds and financial aid", "scholarship disbursements for future semesters").

| Level | Question | Answer |
|---|---|---|
| Symptom | Faithfulness cực thấp (0.088) — model thêm thông tin không có trong evidence | |
| Why 1 | Model tạo ra inference quá nhiều từ limited context | |
| Why 2 | Prompt không constraint model chỉ dùng retrieved context | |
| Why 3 | Không có grounding check trước khi generate | |
| Why 4 | Model preference cho "helpful complete answer" override factual accuracy | |
| Why 5 | Cần thêm instruction "Only state facts explicitly mentioned in the context" | |

**Root cause từ `find_root_cause()`:**
> Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Không đồng ý. Context Recall = 0.48 cho thấy retriever không bỏ sót hoàn toàn, và Context Precision = 1.0 cho thấy chunks retrieved đều relevant. Vấn đề thực sự là **generation** — model thêm thông tin speculation không có trong context. Root cause đúng: "Answer contains claims not supported by context — improve generation grounding."

**Proposed fix cụ thể:**
> 1. Thêm instruction: "Only use information from the provided contexts. Do not infer or speculate." 2. Thêm post-generation hallucination checker. 3. Nếu model không tìm thấy đủ evidence, phải nói "I don't have enough information to answer fully."

---

### Failure 3

**ID và question:** H02 — "A scholarship student failed one academic renewal requirement. What happens, and what if they fail again in the next term?"

**Expected answer:**
> A first failure normally produces one term of scholarship probation rather than immediate loss. During probation, the award remains active. A second consecutive failed review ends the award beginning with the next term.

**Actual answer:**
> If a scholarship student fails to meet one academic renewal requirement, the consequences depend on the specific requirement and the severity of the shortfall. Generally, Northstar University handles this through a progressive disciplinary process. First offense typically results in a scholarship review and potential probation. Repeated failures may lead to scholarship suspension or termination.

**Scores:** Context Recall: 1.000 | Context Precision: 0.750 | Faithfulness: 0.500 | Relevance: 0.800 | Completeness: 0.409 | Overall: 0.570

**Evidence inspection:** Retriever lấy đúng context với policy rõ ràng. Model paraphrase nhưng thiếu specific details (probation period, "second consecutive" trigger).

| Level | Question | Answer |
|---|---|---|
| Symptom | Completeness thấp (0.409) — thiếu specific policy details | |
| Why 1 | Model không extract đủ key details từ context | |
| Why 2 | Context có policy rõ nhưng model summarize quá ngắn | |
| Why 3 | Không instruction yêu cầu include all conditions | |
| Why 4 | Model optimize for concise response, bỏ qua policy nuances | |
| Why 5 | Prompt cần explicit instruction về policy completeness | |

**Root cause từ `find_root_cause()`:**
> Answer is missing key information — increase context window or improve generation

**Bạn đồng ý hay không? Dẫn evidence từ trace:**
> Đồng ý. Evidence đã có đầy đủ thông tin về probation và second failure, nhưng model không include đủ. Cần instruction rõ hơn về việc include tất cả conditions.

**Proposed fix cụ thể:**
> Thêm instruction: "When answering policy questions, include all conditions, deadlines, and consequences mentioned in the context. Do not omit any qualifying factors." Hoặc thêm "Policy checklist" format trong prompt.

---

## 3. Failure Clustering

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | **Generation grounding** — model thêm claims không có trong context | A01, H04, A02, A03 | High |
| 2 | **Prompt scope enforcement** — không enforce out-of-scope boundary rõ ràng | A01, A02, A03 | High |
| 3 | **Completeness instruction** — model bỏ qua key conditions/policy details | H02, H05 | Medium |

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> **Cluster 1 (Generation grounding)** — Vì nó ảnh hưởng cả 4 adversarial cases và gây ra hallucination nghiêm trọng nhất. Fix grounding sẽ cải thiện Faithfulness đáng kể và prevent misinformation.

---

## 4. Improvement Log

```markdown
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | hallucination | Context is missing or irrelevant — improve retrieval | Implement hallucination checker to filter unsupported claims | Open |
| F002 | hallucination | Context is missing or irrelevant — improve retrieval | Add grounding guardrails to ensure answers are sourced from retrieved context | Open |
| F003 | off_topic | Answer does not address the question — improve prompt clarity | Improve prompt clarity and intent classification | Open |
| F004 | off_topic | Answer is missing key information — increase context window or improve generation | Increase chunk size in RAG pipeline to reduce context fragmentation | Open |
| F005 | off_topic | Answer does not address the question — improve prompt clarity | Add clarifying questions when user intent is ambiguous | Open |
| F006 | off_topic | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers | Open |
| F007 | off_topic | Answer does not address the question — improve prompt clarity | Improve scope detection and intent routing | Open |
| F008 | off_topic | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers | Open |
| F009 | off_topic | Answer is missing key information — increase context window or improve generation | Add few-shot examples showing complete answers | Open |
| F010 | hallucination | Context is missing or irrelevant — improve retrieval | Add grounding guardrails to ensure answers are sourced from retrieved context | Open |
```

**Ba improvement suggestions ưu tiên**

1. Thêm explicit grounding instruction: "Only use information explicitly stated in the provided contexts"
2. Implement out-of-scope detection với refusal template
3. Thêm policy completeness check instruction

| Suggestion | Target metric | Verification method |
|---|---|---|
| Thêm grounding instruction | Faithfulness | Re-run benchmark, expect Faithfulness > 0.7 |
| Out-of-scope detection | Relevance (for adversarial) | Test A01-A03, expect proper refusal |
| Policy completeness instruction | Completeness | Re-run Hard cases, expect Completeness > 0.6 |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy `run_regression()` trong các trigger points:
> - Mỗi lần merge code vào main branch
> - Mỗi khi thay đổi prompt hoặc system instruction
> - Mỗi khi thay đổi retrieval strategy (embedding, chunking)
> - Mỗi khi deploy model mới hoặc model version mới
> - Trước mỗi major release

**Câu 2: Threshold drop 0.05 có phù hợp Student Services không? Vì sao?**

> Có phù hợp. Student Services là domain quan trọng (liên quan đến tuition, scholarships, grades). Drop 0.05 (5%) là threshold hợp lý vì:
> - Đủ nhạy để phát hiện degradation có ý nghĩa
> - Không quá strict gây false positives
> - Tuy nhiên, Faithfulness nên có threshold riêng (0.02) vì hallucination nguy hiểm hơn trong student services

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> **Block deployment (threshold nghiêm ngặt):**
> - Faithfulness < 0.7 (quan trọng nhất — ngăn hallucination)
> - Relevance < 0.6 (đảm bảo đúng intent)
>
> **Alert only (monitor nhưng không block):**
> - Context Recall/Precision (đã alert team nhưng không ngăn deploy)
> - Completeness < 0.5 (cần improve nhưng không emergency)

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Unit Tests + Integration Tests] → [Offline Evaluation on Golden Dataset] → [Deploy to Staging + Smoke Tests] → [Deploy to Production + Monitor]
```

> **Giải thích:**
> 1. **Unit/Integration Tests** — Kiểm tra code không break
> 2. **Offline Evaluation** — Chạy golden dataset benchmark, regression check
> 3. **Staging + Smoke Tests** — Manual verification trên sample cases
> 4. **Production + Monitor** — Online evaluation, user feedback monitoring

---

## 6. Continuous Improvement Loop

```text
Evaluate → analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Thêm grounding instruction vào system prompt | Faithfulness: 0.57 → 0.75 | High — ngăn hallucination |
| 2 | Implement out-of-scope detection | Relevance (A01-A03): 0.3 → 0.8 | High — đúng behavior |
| 3 | Thêm policy completeness instruction | Completeness: 0.72 → 0.85 | Medium — đầy đủ thông tin |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> 1. **Multi-hop reasoning** — câu hỏi cần kết hợp 3+ documents (hiện tại chỉ có 2)
> 2. **Ambiguous question** — câu hỏi có thể hiểu nhiều cách
> 3. **Edge case policy dates** — câu hỏi về effective dates và version transitions phức tạp hơn

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Adversarial cases (A01-A03) có Overall Score thấp nhất (0.26-0.30), điều này **đúng với dự đoán** vì chúng designed để test boundaries.
>
> Tuy nhiên, **bất ngờ** là Easy/Medium cases cũng có failure (M01, M05) với off_topic. Điều này cho thấy model không chỉ struggle với adversarial inputs mà còn với regular policy questions khi cần include multiple conditions.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào production, bạn sẽ thay hoặc bổ sung metric nào?**

> **Giới hạn của word-overlap:**
> 1. Không hiểu semantic similarity — "tution" vs "tuition" có overlap thấp dù cùng nghĩa
> 2. Không đánh giá được factual accuracy thực sự
> 3. Không detect được subtle policy nuances
> 4. Sensitive với stopwords và paraphrasing
>
> **Metrics bổ sung cho production:**
> 1. **LLM-as-Judge** với domain-specific rubric (thang 1-5)
> 2. **Factuality scoring** — kiểm tra từng claim có evidence không
> 3. **User satisfaction metrics** — thumbs up/down, follow-up questions
> 4. **Retrieval hit rate** — % queries có top-k chunks relevant
> 5. **Citation accuracy** — model có cite đúng source không
