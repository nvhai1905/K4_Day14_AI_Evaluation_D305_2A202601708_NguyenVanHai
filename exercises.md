# Day 14 — Exercises

## AI Evaluation & Benchmarking · Lab Worksheet

**Thời gian làm bài:** 14:15–17:00

**Domain:** OrbitTech Store Customer Support

Điền trực tiếp câu trả lời vào file này. Golden dataset 20 QA được viết một lần
duy nhất trong `golden_dataset.json`, không chép lại toàn bộ vào Markdown.

---

Từ 14:15–14:30, cài môi trường và chạy baseline tests theo `guide_lab.md`.

---

## Part 1 — Warm-up (14:30–14:45)

### Exercise 1.1 — RAGAS Metric Thresholds

Theo bài giảng:

- 0.8–1.0: Good — monitor, maintain.
- 0.6–0.8: Needs work — analyze failures, iterate.
- Dưới 0.6: Significant issues — investigate.

Với từng metric, xác định khi nào score thấp có thể chấp nhận và khi nào là
critical.

| Metric | Acceptable Low Score Scenario | Critical Low Score Scenario | Action Required |
|---|---|---|---|
| Faithfulness | Answer diễn đạt lại context bằng từ khác nghĩa giống nhau (vd "seven days" vs "1 week"), khiến heuristic word-overlap bị đếm thiếu dù mọi claim vẫn có căn cứ. | Answer nêu một con số bảo hành, số tiền hoàn trả hoặc deadline không hề xuất hiện trong context đã retrieve — model bịa ra chi tiết policy. | Kiểm tra trace thủ công. Nếu chỉ là lỗi diễn đạt/heuristic thì không cần sửa model, chỉ ghi nhận giới hạn của heuristic. Nếu là bịa thông tin thật sự thì chặn deploy và siết lại grounding/prompt (yêu cầu trích dẫn context). |
| Answer Relevance | Answer trả lời đúng ý câu hỏi nhưng có thêm ngữ cảnh liên quan (vd nhắc thêm policy gần đó), làm giảm độ trùng từ vựng với câu hỏi dù vẫn hữu ích. | Answer nói về chủ đề gần giống nhưng không giải quyết đúng điều được hỏi (vd hỏi cách hủy đơn nhưng trả lời về trễ giao hàng) — lỗi nhận diện intent. | Nếu retrieval đúng mà relevance vẫn thấp thì generation/prompt đang lệch intent, cần sửa prompt. Nếu retrieval cũng sai thì coi đây là lỗi retrieval trước. |
| Context Recall | Một chi tiết phụ (vd ví dụ minh họa) nằm trong chunk không được retrieve, nhưng chunk chứa quy định chính vẫn có mặt. | Chunk chứa evidence cốt lõi để trả lời đúng bị thiếu hoàn toàn trong `retrieved_contexts`, dù nó tồn tại trong corpus. | Kiểm tra retriever (query terms, top-k, cách chia chunk) — đây là lỗi retrieval, không phải lỗi generation, sửa prompt sẽ không giải quyết được. |
| Context Precision | Chunk liên quan vẫn được retrieve nhưng xếp hạng 3–4 trong 5 do trùng từ với tài liệu không liên quan; recall không bị ảnh hưởng, chỉ ranking bị nhiễu. | Phần lớn chunk xếp hạng cao là không liên quan hoặc sai document, còn evidence đúng bị chôn ở cuối top-k hoặc không lọt vào top-k. | Tinh chỉnh ranking của retriever (trọng số BM25, mở rộng query) hoặc thêm bước reranking (xem bonus Exercise 3.5) nếu pattern này lặp lại ở nhiều case. |
| Completeness | Answer bao quát quy trình/quy định chính nhưng bỏ sót một exception hiếm khi liên quan trực tiếp tới câu hỏi. | Answer bỏ sót một điều kiện, ngoại lệ hoặc số tiền quan trọng làm thay đổi kết quả thực tế của khách hàng (vd thiếu phí restocking hoặc deadline trả hàng). | Nếu Context Recall cao thì đây là lỗi generation — cần chỉnh prompt để buộc model trích xuất đủ mọi điều kiện. Nếu Context Recall thấp thì phải sửa retrieval trước. |

### Exercise 1.2 — Bias trong LLM-as-a-Judge

Ba bias thường gặp:

- Position bias: judge ưu tiên answer xuất hiện trước.
- Verbosity bias: judge ưu tiên answer dài hơn.
- Self-preference: judge ưu tiên output giống chính model đó.

**Câu 1: Thiết kế experiment phát hiện position bias với ít nhất hai conditions.**

> Lấy khoảng 20 cặp câu trả lời (A, B) đã được đánh giá chất lượng ngang nhau bởi một rubric/human độc lập từ trước. Cho judge chấm mỗi cặp hai lần với nội dung giữ nguyên: Condition 1 = trình bày A trước, B sau; Condition 2 = đảo vị trí, B trước A sau. Nếu judge có xu hướng chọn thắng cho bất kể answer nào đứng ở vị trí đầu (tỉ lệ thắng của "vị trí 1" lệch rõ khỏi 50%, ví dụ >65% trên toàn batch), thì có position bias. Cách xử lý thực tế: luôn chấm cả hai thứ tự rồi lấy trung bình điểm của mỗi answer.

**Câu 2: Làm thế nào giảm verbosity bias bằng rubric design?**

> Rubric phải định nghĩa từng mức điểm dựa trên nội dung cụ thể (đủ điều kiện, ngoại lệ, đúng số tiền/ngày) chứ không dựa trên độ dài hay mức độ chi tiết. Ghi rõ trong rubric: "không cộng điểm cho thông tin thừa không liên quan tới câu hỏi; câu trả lời ngắn nhưng đủ và đúng vẫn được điểm tối đa". Có thể bắt judge liệt kê cụ thể claim nào đúng/sai/thiếu trước khi cho điểm tổng, thay vì cho điểm cảm tính — buộc judge neo vào evidence thay vì độ dài câu trả lời.

**Câu 3: Tại sao cần calibrate LLM judge với human labels?**

> Vì LLM judge có thể mang bias hệ thống (position, verbosity, self-preference) và không chắc hiểu đúng policy đặc thù của domain (vd quy định hoàn tiền/bảo hành riêng của OrbitTech). Calibration bằng cách lấy một tập nhỏ (20–30 case) cho cả judge và human cùng chấm, rồi đo mức đồng thuận (Cohen's kappa hoặc correlation). Nếu lệch nhiều, phải sửa lại rubric hoặc thêm few-shot example cho judge trước khi tin dùng nó ở quy mô lớn hoặc trong CI/CD.

### Exercise 1.3 — Evaluation trong CI/CD

**Câu 1: Chọn threshold để block deployment.**

| Metric | Threshold | Lý do |
|---|---:|---|
| Faithfulness | ≥ 0.85 | Hallucination trong customer support (sai giá, sai điều kiện hoàn tiền, sai thời hạn bảo hành) gây rủi ro tin cậy và pháp lý cao nhất, nên threshold phải nghiêm ngặt nhất. |
| Answer Relevance | ≥ 0.75 | Answer cần đúng trọng tâm câu hỏi, nhưng heuristic word-overlap không hoàn hảo nên để threshold thấp hơn Faithfulness một chút để tránh false block. |
| Completeness | ≥ 0.70 | Thiếu một điều kiện phụ đôi khi vẫn chấp nhận được, nhưng thiếu nhiều điều kiện sẽ gây khiếu nại; threshold vừa phải vì heuristic completeness khá strict theo overlap với expected answer. |

**Câu 2: Khi nào dùng offline evaluation, online evaluation và human review?**

> - **Offline:** mỗi khi thay đổi prompt, retriever, chunking hoặc model, trước khi merge/deploy — chạy trên golden dataset cố định để so sánh regression một cách lặp lại được.
> - **Online:** theo dõi liên tục trên traffic thật sau khi deploy, để phát hiện drift, câu hỏi mới ngoài golden dataset, hoặc vấn đề retrieval chỉ xuất hiện với dữ liệu thực tế.
> - **Human review:** định kỳ (vd hàng tuần) lấy mẫu một phần câu trả lời thật, đặc biệt các case adversarial hoặc high-stakes (privacy, số tiền hoàn trả lớn), hoặc khi offline/online metrics có tín hiệu bất thường cần xác nhận trước khi hành động.

---

## Part 2 — Core Coding (14:45–15:40)

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

## Part 3 — Golden Dataset & Real Benchmark (15:40–16:35)

### Exercise 3.1 — Build the Golden Dataset

Thiết kế và validate dataset theo Mục 5–6 trong `guide_lab.md`. Nội dung 20 QA
được điền trực tiếp trong `golden_dataset.json`; phần dưới chỉ ghi lại kết quả
và quyết định thiết kế, không chép lại toàn bộ QA.

**Kết quả dataset**

| Hạng mục | Kết quả |
|---|---|
| Tổng số records | ____ / 20 |
| Easy | ____ / 5 |
| Medium | ____ / 7 |
| Hard | ____ / 5 |
| Adversarial | ____ / 3 |
| Source documents được sử dụng | ____ / 10 |
| Validator status | PASS / FAIL |

**Ba case đại diện cho quyết định thiết kế**

| ID | Difficulty | Source document(s) | Vì sao case phù hợp với difficulty/attack type? |
|---|---|---|---|
| | | | |
| | | | |
| | | | |

**Điểm khó nhất khi xây dựng expected answer hoặc evidence là gì?**

> *Câu trả lời:*

**Xác nhận:**

- [ ] Mọi claim trong expected answer đều có evidence hỗ trợ.
- [ ] Không có questions trùng ý và không dùng kiến thức ngoài corpus.
- [ ] `python validate_golden_dataset.py` báo `PASS`.

### Exercise 3.2 — Benchmark Run

Chạy:

```bash
python domain_assistant.py
python evaluate_answers.py
```

Copy bảng terminal vào đây hoặc điền từ `artifacts/benchmark_results.json`.

| ID | Question (short) | Ctx Recall | Ctx Precision | Faithfulness | Relevance | Completeness | Overall | Passed? | Failure Type |
|---|---|---:|---:|---:|---:|---:|---:|---|---|
| E01 | | | | | | | | | |
| E02 | | | | | | | | | |
| E03 | | | | | | | | | |
| E04 | | | | | | | | | |
| E05 | | | | | | | | | |
| M01 | | | | | | | | | |
| M02 | | | | | | | | | |
| M03 | | | | | | | | | |
| M04 | | | | | | | | | |
| M05 | | | | | | | | | |
| M06 | | | | | | | | | |
| M07 | | | | | | | | | |
| H01 | | | | | | | | | |
| H02 | | | | | | | | | |
| H03 | | | | | | | | | |
| H04 | | | | | | | | | |
| H05 | | | | | | | | | |
| A01 | | | | | | | | | |
| A02 | | | | | | | | | |
| A03 | | | | | | | | | |

**Aggregate Report**

- Overall pass rate: ____%
- Avg Context Recall: ____
- Avg Context Precision: ____
- Avg Faithfulness: ____
- Avg Relevance: ____
- Avg Completeness: ____
- Failure type distribution: ____

**Ba cases có Overall Score thấp nhất**

1. ID: ____ | Score: ____ | Failure type: ____
2. ID: ____ | Score: ____ | Failure type: ____
3. ID: ____ | Score: ____ | Failure type: ____

**Nhận xét ngắn:** Metric nào yếu nhất? Kết quả gợi ý vấn đề nằm ở retrieval
hay generation?

> *Câu trả lời:*

### Exercise 3.3 — LLM-as-a-Judge Rubric Design

Thiết kế rubric domain-specific cho OrbitTech Customer Support. Mỗi mức phải
đủ cụ thể để hai người chấm độc lập có thể hiểu giống nhau.

Chọn 3–5 dimensions:

- [ ] Correctness
- [ ] Completeness
- [ ] Relevance
- [ ] Evidence/citation
- [ ] Actionability
- [ ] Safety/privacy
- [ ] Tone/clarity
- [ ] Dimension khác: __________

| Score | Tiêu chí domain-specific | Ví dụ response |
|---:|---|---|
| 5 | | |
| 4 | | |
| 3 | | |
| 2 | | |
| 1 | | |

**Ba edge cases khó chấm**

| Edge Case | Tại sao khó chấm? | Rubric xử lý thế nào? |
|---|---|---|
| | | |
| | | |
| | | |

**Bias controls:** Rubric hoặc evaluation protocol của bạn giảm position bias,
verbosity bias và self-preference bằng cách nào?

> *Câu trả lời:*

### Exercise 3.4 — Framework Comparison (Bonus +10)

Chỉ làm sau khi hoàn thành 3.1–3.3. Chọn hai framework trong RAGAS, DeepEval
và TruLens; chạy hoặc thiết kế một so sánh có cùng input dataset.

| Tiêu chí | Framework 1: ____ | Framework 2: ____ |
|---|---|---|
| Setup complexity | | |
| Metrics available | | |
| CI/CD integration | | |
| Kết quả trên cùng dataset | | |
| Insight rút ra | | |

- Scores có nhất quán không?
- Framework nào strict hơn và vì sao?
- Hai framework có tìm ra cùng failure cases không?

> *Phân tích:*

### Exercise 3.5 — Retrieval Reranking (Bonus +5)

Mục tiêu: kiểm tra việc đổi thứ tự chunks có tăng Context Precision mà không
thay đổi Context Recall hay không.

1. Chọn ít nhất 5 cases từ `artifacts/actual_answers.json`.
2. Tính Context Recall và Context Precision trước rerank.
3. Implement `rerank_by_overlap()` hoặc một reranker khác.
4. Rerank cùng tập chunks, không thêm hoặc xóa chunk.
5. Tính lại hai metrics và giải thích kết quả.

| ID | Recall before | Recall after | Precision before | Precision after | Delta Precision |
|---|---:|---:|---:|---:|---:|
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| | | | | | |
| **Avg** | | | | | |

**Tại sao Recall dự kiến không đổi?**

> *Câu trả lời:*

**Khi nào reranking không đủ và cần sửa retriever/query/chunking?**

> *Câu trả lời:*

---

## Part 4 — Reflection (16:35–16:50)

Hoàn thành `reflection.md` bằng kết quả thật từ Exercise 3.2.

---

## Completion Checklist

Hoàn thành kiểm tra cuối trong khoảng 16:50–17:00.

- [ ] Tất cả required tests pass.
- [ ] `golden_dataset.json` validate thành công.
- [ ] Exercise 3.1 hoàn thành trong file JSON và bảng kết quả phía trên.
- [ ] Exercise 3.2 có năm metrics, aggregate report và ba cases thấp nhất.
- [ ] Exercise 3.3 có rubric 1–5 và bias controls.
- [ ] `reflection.md` có ba failure analyses và regression strategy.
- [ ] Đã copy `template.py` thành `solution/solution.py`.
- [ ] Exercise 3.4 và 3.5 chỉ làm nếu chọn bonus.
