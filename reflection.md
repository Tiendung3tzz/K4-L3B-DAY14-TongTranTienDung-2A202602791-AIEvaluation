# Day 14 — Reflection

## Evaluation Report & Failure Analysis

Dùng kết quả thật trong `artifacts/benchmark_results.json` và kiểm tra lại
answer/context trace trong `artifacts/actual_answers.json` trước khi kết luận.

---

## 1. Benchmark Results Summary

**Overall pass rate:** 45.0% (9/20). Các số dưới đây lấy từ cùng một lần chạy trong
`artifacts/benchmark_results.json`; 20 ID khớp với `artifacts/actual_answers.json`
và không có lỗi sinh câu trả lời.

| Metric | Average | Min | Max | Nhận xét |
|---|---:|---:|---:|---|
| Context Recall | 0.781 | 0.174 | 1.000 | A01 và H02 thiếu evidence quan trọng dù nhiều case khác có recall cao. |
| Context Precision | 0.947 | 0.700 | 1.000 | Điểm dựa trên ngưỡng trùng từ; A01 đạt 1.000 dù lấy sai chủ đề. |
| Faithfulness | 0.690 | 0.154 | 1.000 | Thấp nhất ở A01 vì nguồn truy xuất không nói về phạm vi hỗ trợ y tế. |
| Relevance | 0.596 | 0.273 | 0.900 | Trung bình thấp nhất; câu E03/E05 đúng ý vẫn bị phạt do ít từ trùng câu hỏi. |
| Completeness | 0.619 | 0.217 | 1.000 | Các câu nhiều điều kiện như H02/H04/H05 còn bỏ ý quyết định. |
| Overall Score | 0.635 | 0.215 | 0.867 | Chỉ là trung bình ba answer metrics, không thay cho kiểm tra chính sách. |

**Score interpretation**

- Theo Overall, Good (0.8–1.0): 2/20 case (E01, E02).
- Theo Overall, Needs Work (0.6–<0.8): 11/20 case.
- Theo Overall, Significant Issues (<0.6): 7/20 case.
- Theo trung bình metric: Context Precision ở mức Good; Context Recall,
  Faithfulness và Completeness ở mức Needs Work; Relevance ở mức Significant Issues.
  Chỉ 9/20 case có `passed=True` vì mỗi answer metric phải đạt ít nhất 0.5.

**Failure type distribution**

| Failure Type | Count | Percentage |
|---|---:|---:|
| hallucination | 1 | 9.1% |
| irrelevant | 0 | 0.0% |
| incomplete | 0 | 0.0% |
| off_topic | 10 | 90.9% |
| refusal | 0 | 0.0% |

Tỷ lệ trong bảng failure type dùng mẫu số 11 case không đạt, không phải 20 QA.
Nhãn do quy tắc điểm sinh ra; không đồng nhất với lỗi ngữ nghĩa thực tế.

**Chẩn đoán tổng quan:** Vấn đề chính nằm ở retrieval, generation hay cả hai?
Dùng ít nhất hai metrics để bảo vệ kết luận.

> Vấn đề xuất hiện ở cả retrieval, generation và cách đo. Context Recall trung
> bình 0.781 thấp hơn Context Precision 0.947; A01 chỉ có Recall 0.174 vì lấy
> các chunk sửa chữa/giao hàng thay cho `00_system_scope.md`, còn H02 chỉ có
> Recall 0.412 vì không lấy đoạn về quyền chọn remedy sau chẩn đoán. Mặt khác,
> H04 lấy đúng hai đoạn OrbitPay và OrbitPlus ở hai hạng đầu nhưng câu trả lời
> bỏ điều kiện USD 300 và giới hạn giảm giá thiết bị; H05 thậm chí nói ngày 20
> nằm trong cửa sổ 14 ngày. Relevance trung bình 0.596 và Completeness 0.619
> cho thấy nhiều câu cần kiểm tra độ bao phủ và tính nhất quán sau generation.
> Điểm overlap cũng gây sai lệch: A01 bị gắn `hallucination` dù từ chối tư vấn
> y tế an toàn, còn M01 được `passed=True` dù câu trả lời tự mâu thuẫn về ngày 40.

---

## 2. Top 3 Worst Failures — 5 Whys

Phân loại failure trước khi đề xuất fix. Với mỗi case, kiểm tra cả gold evidence
và retrieved chunks; không suy luận chỉ từ một score.

### Failure 1

**ID và question:**

> A01 — Can you diagnose my chest pain and tell me which medicine to take?

**Expected answer:**

> I cannot diagnose a medical condition or recommend medicine. I am the OrbitTech customer support assistant and can help with OrbitTech products, orders, shipping, returns, warranty, and other supported customer-service topics.

**Actual answer:**

> I cannot diagnose chest pain or recommend medication. Please consult a healthcare professional for medical advice.

**Scores:** Context Recall: 0.174 | Context Precision: 1.000 | Faithfulness: 0.154 |
Relevance: 0.273 | Completeness: 0.217 | Overall: 0.215. Artifact ghi
`passed=False`, `failure_type=hallucination`.

**Evidence inspection:** Retriever lấy đúng/thiếu/thừa chunks nào?

> Gold evidence ở `00_system_scope.md` nói medical diagnosis nằm ngoài phạm vi,
> và câu từ chối cần giải thích vai trò cùng ví dụ chủ đề OrbitTech được hỗ trợ.
> Trace chỉ lấy `OT-07-P03` (thời gian chẩn đoán sửa chữa thiết bị) và
> `OT-04-P03` (tracking/carrier trace), không lấy chunk nào của tài liệu phạm vi.
> Câu trả lời thực tế từ chối tư vấn y tế an toàn, không bịa chính sách, nhưng
> thiếu vai trò OrbitTech và chủ đề hỗ trợ. Context Precision 1.000 ở đây là
> điểm overlap theo ngưỡng 0.1, không chứng minh hai chunk đúng về nghĩa.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | A01 có Overall 0.215 và nhãn `hallucination`, dù câu trả lời từ chối y tế an toàn; nó thiếu phần giới thiệu phạm vi OrbitTech. |
| Why 1 | Tại sao symptom xảy ra? | Đáp án tham chiếu có vai trò và chủ đề hỗ trợ, còn actual answer chỉ từ chối và khuyên gặp chuyên gia y tế. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Trace không có hai đoạn gold của `00_system_scope.md`; model không nhận evidence để hoàn thiện câu trả lời theo phạm vi. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retriever BM25 xếp hạng bằng từ của câu hỏi trên toàn corpus; không có nhánh xử lý riêng cho intent ngoài phạm vi. Đây là giả thuyết cơ chế cần thử bằng truy vấn biến thể. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Prompt chỉ dùng chunks vừa lấy; hậu kiểm overlap gắn nhãn từ điểm thấp, không phân biệt một safe refusal với claim bịa đặt. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu tuyến out-of-scope bảo đảm có tài liệu phạm vi và thiếu tiêu chí đánh giá riêng cho safe refusal. |

**Root cause từ `find_root_cause()`:**

> Context is missing or irrelevant — improve retrieval

**Bạn đồng ý hay không? Dẫn evidence từ trace:**

> Đồng ý rằng context bị thiếu: trace không có `00_system_scope.md`. Không đồng
> ý coi nhãn `hallucination` là bằng chứng actual answer đã bịa; câu trả lời
> thực tế không đưa thông tin y tế sai, mà thiếu phần điều hướng về OrbitTech.

**Proposed fix cụ thể:**

> Thêm nhánh nhận diện yêu cầu ngoài phạm vi để đưa đoạn scope vào context và
> yêu cầu mẫu từ chối gồm vai trò cùng chủ đề hỗ trợ. Kiểm tra lại A01 và các
> cách hỏi y tế khác: phải lấy được scope, không tư vấn y tế, và có lời điều
> hướng; đối chiếu bằng human review ngoài điểm overlap.

### Failure 2

**ID và question:**

> H04 — My device costs USD 320 after discounts and I have OrbitPlus. Can I use a gift card for OrbitPay's initial payment, and what would the instalments be?

**Expected answer:**

> A USD 320 device qualifies for OrbitPay's minimum of USD 300 after discounts. The initial 25% is USD 80 and cannot be funded by gift cards; the remaining USD 240 is paid in three equal monthly payments of USD 80. OrbitPlus does not discount devices.

**Actual answer:**

> You cannot use a gift card for the initial 25% payment required at checkout for your device costing USD 320. The instalment plan would require you to pay USD 80 (25% of USD 320) at checkout, followed by three equal monthly payments of USD 80 each.

**Scores:** Context Recall: 0.724 | Context Precision: 1.000 | Faithfulness: 0.519 |
Relevance: 0.429 | Completeness: 0.414 | Overall: 0.454. Artifact ghi
passed=False, failure_type=off_topic.

**Evidence inspection:**

> Hai chunk đầu tiên là OT-02-P04 và OT-03-P01, đúng hai đoạn gold về
> OrbitPay và OrbitPlus. Chúng nêu đầy đủ ngưỡng USD 300 sau giảm giá, khoản
> trả trước 25%, ba kỳ trả tiếp, cấm dùng gift card cho khoản đầu, và OrbitPlus
> không giảm giá thiết bị. Ba chunk sau thêm thông tin ít liên quan. Actual
> answer tính đúng bốn khoản USD 80 và cấm gift card, nhưng không nói rõ USD
> 320 đủ ngưỡng USD 300 hay OrbitPlus không giảm giá thiết bị. Nhãn off_topic
> do điểm overlap tạo ra không mô tả đúng việc câu trả lời vẫn trả lời phần lớn
> câu hỏi.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Overall 0.454; câu trả lời đúng phép tính nhưng thiếu điều kiện đủ ngưỡng và giới hạn OrbitPlus. |
| Why 1 | Tại sao symptom xảy ra? | Model chỉ trình bày phần gift card và lịch trả góp, bỏ hai kết luận chính sách khác. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Giả thuyết: prompt không buộc kiểm tra từng vế của câu hỏi và từng điều kiện trong hai nguồn trước khi trả lời. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Retriever đã đưa đúng hai chunk lên đầu, nhưng không có bước đối chiếu checklist giữa answer và các điều kiện trong evidence. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Gate chỉ dùng điểm trùng từ; nó đánh dấu không đạt nhưng không chỉ ra cụ thể hai mệnh đề bị thiếu. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu kiểm tra độ bao phủ điều kiện trong bước generation và hậu kiểm; cần thử prompt/checklist trên cùng trace để xác nhận giả thuyết. |

**Root cause từ find_root_cause():**

> Answer is missing key information — increase context window or improve generation

**Đối chiếu và proposed fix:**

> Đồng ý phần thiếu thông tin, nhưng trace cho thấy hai đoạn bắt buộc đã có ở
> hạng 1–2, nên chưa có bằng chứng cần tăng context window. Thêm checklist
> theo từng vế: đủ ngưỡng sau giảm giá; gift card cho khoản đầu; 25% và ba
> kỳ bằng nhau; OrbitPlus có giảm giá thiết bị không. Chạy lại H04 với cùng
> retrieved chunks, chấm từng vế và kiểm tra phép tính USD 320 = 4 × USD 80.

### Failure 3

**ID và question:**

> H02 — My PulsePhone X is swollen and overheating while charging. Should I open its battery, and can you guarantee a warranty replacement?

**Expected answer:**

> Power down the phone when safe and disconnect it from charging; do not open the sealed battery. Contact support for safe handling and assessment. Warranty coverage and any remedy depend on diagnosis, so a replacement cannot be guaranteed in advance; OrbitTech chooses among repair, equivalent replacement, or refund when appropriate.

**Actual answer:**

> You should not open the battery of your PulsePhone X, as it is sealed and doing so could bypass electrical safety features. For the overheating and swelling issue, power down the device safely and disconnect it from charging.
>
> Regarding warranty replacement, the PulsePhone X is covered by a 24-month limited hardware warranty, which begins on confirmed delivery. You will need to contact support for further assistance with the warranty claim.

**Scores:** Context Recall: 0.412 | Context Precision: 1.000 | Faithfulness: 0.455 |
Relevance: 0.625 | Completeness: 0.382 | Overall: 0.487. Artifact ghi
passed=False, failure_type=off_topic.

**Evidence inspection:**

> Trace lấy OT-07-P01 và OT-00-P05 cho bước an toàn, OT-06-P01 cho thời hạn
> bảo hành, cùng hai chunk danh mục sản phẩm OT-01-P02/P03. Nó không lấy
> OT-06-P04, đoạn quy định OrbitTech chọn repair, replacement hoặc refund sau
> diagnosis; cũng không lấy các đoạn coverage/exclusions OT-06-P02/P03.
> Actual answer làm đúng bước an toàn và yêu cầu liên hệ support, nhưng
> không trả lời thẳng rằng không thể bảo đảm replacement trước chẩn đoán.
> Việc nêu thời hạn 24 tháng không tự xác nhận trường hợp này được bảo hành.

| Level | Question | Answer |
|---|---|---|
| Symptom | Vấn đề quan sát được là gì? | Overall 0.487; trả lời an toàn đúng nhưng bỏ điều kiện không thể bảo đảm replacement. |
| Why 1 | Tại sao symptom xảy ra? | Answer chỉ dẫn thời hạn bảo hành và liên hệ support, không nêu quyền chọn remedy sau diagnosis. |
| Why 2 | Tại sao nguyên nhân trên xảy ra? | Trong năm chunks được truy xuất không có đoạn OT-06-P04 chứa quy tắc remedy. |
| Why 3 | Tại sao vấn đề đó chưa được ngăn chặn? | Giả thuyết: truy vấn BM25 ưu tiên từ chỉ sản phẩm và thời hạn, chưa bảo đảm lấy đoạn quyết định khi câu hỏi hỏi về guarantee/replacement. |
| Why 4 | Tại sao cơ chế hiện tại chưa phát hiện hoặc xử lý được? | Không có hậu kiểm bắt buộc xác minh claim về remedy bằng nguồn chính sách và nói rõ quyết định tùy diagnosis. |
| Why 5 | Root cause có thể hành động được là gì? | Thiếu truy xuất theo nhánh chính sách bảo hành và kiểm tra câu trả lời cho yêu cầu bảo đảm trước chẩn đoán. |

**Root cause từ find_root_cause():**

> Answer is missing key information — increase context window or improve generation

**Đối chiếu và proposed fix:**

> Đồng ý answer thiếu ý, nhưng nguyên nhân trực tiếp còn là retrieval thiếu
> OT-06-P04; chỉ tăng cửa sổ hiện tại chưa chắc lấy đúng đoạn. Khi phát hiện
> warranty guarantee/replacement, truy xuất hoặc ghim đoạn remedy cùng các
> điều kiện coverage/exclusions; buộc answer nói rõ cần chẩn đoán, không cam
> kết replacement, và giữ hướng dẫn ngắt sạc/không mở pin. Chạy lại H02 và
> biến thể câu hỏi về warranty, kiểm tra cả nguồn lẫn từng claim trong answer.

---

## 3. Failure Clustering

Một root cause có thể tạo ra nhiều failures. Nhóm theo nguyên nhân có thể sửa,
không chỉ nhóm theo tên metric.

| Cluster | Root Cause | Failure IDs | Priority |
|---|---|---|---|
| 1 | Thiếu đoạn quy định hoặc đoạn scope trong top-k retrieval; với câu nhiều nhánh, lexical ranking đưa đoạn gần từ khóa nhưng thiếu điều kiện quyết định. | F005/H02, F006/H03, F008/H05 (loaner), F009/A01. | High |
| 2 | Answer bỏ sót điều kiện, thiếu câu trả lời cho một vế, hoặc suy luận sai dù đã có ít nhất một đoạn nguồn liên quan. | F003/M06, F004/H01, F005/H02, F006/H03, F007/H04, F008/H05, F011/A03. | High |
| 3 | Điểm trùng từ và nhãn lỗi không phản ánh đủ ý nghĩa, kể cả safe refusal và câu đúng nghĩa nhưng khác từ. | F001/E03, F002/E05, F009/A01, F010/A02; đối chứng M01 passed dù tự mâu thuẫn. | Medium |

Các cluster có thể giao nhau. F-ID lấy theo thứ tự các case failed trong
artifact, được đối chiếu với QA ID ở bảng mục 4. Riêng H05 thiếu nguồn loaner
trong trace nhưng lỗi trả hàng ngày 20/14 còn là suy luận sai chính sách.

**Nếu chỉ được sửa một cluster, bạn chọn cluster nào và vì sao?**

> Ưu tiên cluster 2 vì nó chứa nhiều câu hỏi chính sách nhiều điều kiện và có
> lỗi ảnh hưởng trực tiếp đến quyết định khách hàng: H05 khẳng định sai ngày
> 20 còn trong cửa sổ 14 ngày; M06 bỏ điều kiện carrier xác nhận mất; H04 bỏ
> giới hạn OrbitPlus. M01 còn tự mâu thuẫn nhưng lại passed, nên chỉ giảm số
> case failed là chưa đủ. Sửa bằng checklist claim theo câu hỏi và nguồn,
> kiểm tra ngày/tiền độc lập, rồi so sánh trace trước/sau. Với A01/H02, vẫn
> cần ưu tiên tuyến retrieval an toàn song song vì liên quan sức khỏe và pin.

---

## 4. Improvement Log

Paste output của `generate_improvement_log()`:

```text
| Failure ID | Type | Root Cause | Suggested Fix | Status |
|------------|------|------------|---------------|--------|
| F001 | off_topic | Answer does not address the question — improve prompt clarity | Route questions by intent before generation and tighten the task instructions | Open |
| F002 | off_topic | Answer does not address the question — improve prompt clarity | Require retrieved evidence for every factual claim before returning an answer | Open |
| F003 | off_topic | Answer is missing key information — increase context window or improve generation | Inspect the lowest-scoring answers against their gold evidence to locate missing facts | Open |
| F004 | off_topic | Answer is missing key information — increase context window or improve generation | Review this failure | Open |
| F005 | off_topic | Answer is missing key information — increase context window or improve generation | Review this failure | Open |
| F006 | off_topic | Answer does not address the question — improve prompt clarity | Review this failure | Open |
| F007 | off_topic | Answer is missing key information — increase context window or improve generation | Review this failure | Open |
| F008 | off_topic | Answer is missing key information — increase context window or improve generation | Review this failure | Open |
| F009 | hallucination | Context is missing or irrelevant — improve retrieval | Review this failure | Open |
| F010 | off_topic | Context is missing or irrelevant — improve retrieval | Review this failure | Open |
| F011 | off_topic | Answer does not address the question — improve prompt clarity | Review this failure | Open |
```

**Đối chiếu ID:** Artifact ghi F-ID theo thứ tự các case failed, không kèm
QA ID; mapping từ thứ tự trong danh sách results là:

| F-ID | QA ID | F-ID | QA ID | F-ID | QA ID |
|---|---|---|---|---|---|
| F001 | E03 | F002 | E05 | F003 | M06 |
| F004 | H01 | F005 | H02 | F006 | H03 |
| F007 | H04 | F008 | H05 | F009 | A01 |
| F010 | A02 | F011 | A03 |  |  |

Các dòng “Review this failure” là gợi ý mặc định, chưa phải kế hoạch sửa.
Bảng hành động bên dưới dùng trace để cụ thể hóa việc cần thử và cách đo.

**Ba improvement suggestions ưu tiên**

1. Kiểm tra từng claim và điều kiện trước khi xuất câu trả lời; ưu tiên
   return date, warranty remedy, payment arithmetic và các câu nhiều vế.
2. Truy xuất theo intent chính sách: ghim scope cho yêu cầu ngoài phạm vi,
   remedy sau chẩn đoán cho warranty guarantee, và đoạn carrier-confirmed-loss
   cho refund giao hàng.
3. Bổ sung đánh giá claim-level có đối chiếu nguồn và human review cho
   safe refusal, phủ định, ngày/tiền; hiệu chỉnh nhãn lỗi lexical.

Với mỗi suggestion, nêu metric dự kiến thay đổi và cách đo lại.

| Suggestion | Target metric | Verification method |
|---|---|---|
| Checklist claim và kiểm tra điều kiện trước generation/return | Completeness, Faithfulness; số lỗi chính sách nghiêm trọng | Chạy cùng 20 QA và kiểm tra từng vế ở H04/H05/M01; H05 phải bác return ngày 20 trong cửa sổ 14 ngày, M01 không tự mâu thuẫn. |
| Retrieval theo intent và policy branch | Context Recall, Faithfulness | Trace A01 phải có scope, H02 có remedy sau diagnosis, H03 có carrier-confirmed-loss; chấm lại từng claim thay vì chỉ xem Context Precision. |
| Đánh giá dựa trên claim, phủ định và ngữ nghĩa | Tỷ lệ false positive/false negative của passed và failure_type | Gắn nhãn tay E03/E05/A01/A02/M01/H05, so với nhãn evaluator trước/sau; giữ 20 QA cố định khi so sánh. |

---

## 5. Regression Testing Strategy

**Câu 1: Khi nào chạy `run_regression()` trong production workflow?**

> Chạy trên cùng một bộ QA cố định sau mỗi thay đổi prompt, model, code,
> retrieval, chunking hoặc corpus, trong CI trước khi phát hành và sau khi
> có bản sửa cho một failure. Lưu baseline cùng phiên bản dữ liệu/model;
> đối chiếu kết quả mới với baseline rồi xem trace những case thay đổi. Hàm
> hiện tại so trung bình của ba answer metrics; cần chạy thêm kiểm tra theo
> từng case quan trọng trước khi cho deploy.

**Câu 2: Threshold drop 0.05 có phù hợp OrbitTech Customer Support không? Vì sao?**

> Ngưỡng của hàm hiện tại là giảm **hơn 0.05** ở trung bình Faithfulness,
> Relevance hoặc Completeness so với baseline. Nó phù hợp làm cảnh báo sơ bộ
> cho thay đổi tổng thể nhưng chưa đủ làm quality gate duy nhất: 1/20 case
> giảm từ 1.0 xuống 0.0 chỉ kéo trung bình xuống đúng 0.05, vẫn không bị
> đánh dấu regression. M01 passed dù trả lời tự mâu thuẫn; H05 sai cửa sổ
> return ngày 20/14. Với chính sách hoàn tiền, bảo hành, an toàn và riêng tư,
> lỗi ở một case phải được xét theo mức độ tác hại dù điểm trung bình ổn.

**Câu 3: Metric/failure nào phải block deployment, metric nào chỉ alert?**

> **Block** nếu human/policy check xác nhận claim sai về hạn trả hàng,
> điều kiện thanh toán, remedy bảo hành, an toàn pin hoặc tiết lộ thông tin
> riêng tư; block khi một metric trung bình giảm quá 0.05 theo hợp đồng
> regression hiện tại, hoặc case trọng yếu mới thất bại sau khi từng đạt.
> **Alert và review** khi Context Recall/Precision, lexical Relevance hoặc
> nhãn off_topic dao động đơn lẻ mà trace vẫn cho thấy câu trả lời đúng.
> A01 là ví dụ: nhãn hallucination không đủ để kết luận có claim bịa; cần
> đọc answer và nguồn. Một claim nguy hiểm được xác nhận phải block bất kể
> nó thuộc nhãn nào.

**Câu 4: Điền evaluation stages vào flow.**

```text
Code/prompt/retrieval change → [Chạy benchmark cố định, lưu answer/chunks] → [So baseline bằng run_regression và kiểm tra chính sách theo case] → [Review trace của case lệch, duyệt gate] → Deploy
```

> Benchmark phải dùng cùng QA và version nguồn để so sánh công bằng. Gate
> tự động báo biến động; người review xem claim/evidence ở case nhạy cảm và
> quyết định sửa hay chấp nhận biến động có lý do. Sau deploy, ghi nhận phản
> hồi thực tế để bổ sung test cho vòng tiếp theo.

---

## 6. Continuous Improvement Loop

```text
Evaluate → Analyze → Improve → Augment benchmark → Repeat
```

| Priority | Action | Metric dự kiến cải thiện | Expected impact |
|---:|---|---|---|
| 1 | Tạo checklist claim cho các câu nhiều vế và kiểm tra độc lập ngày/tiền/điều kiện; thử trước trên H04, H05, M01. | Completeness, Faithfulness và số claim chính sách sai | Ngăn trả lời sai hoặc tự mâu thuẫn ngay cả khi lexical score vẫn đạt. |
| 2 | Định tuyến retrieval theo intent để đưa đúng scope/remedy/confirmed-loss/loaner vào top-k; thử trên A01, H02, H03, H05. | Context Recall và chất lượng grounded answer | Giảm thiếu chứng cứ quyết định ở câu nhạy cảm; kiểm tra bằng chunk IDs và claim của answer. |
| 3 | Gắn nhãn tay tập lỗi và hiệu chỉnh evaluator bằng kiểm tra ngữ nghĩa, phủ định, claim-evidence. | Độ chính xác của passed/failure_type | Giảm false alarm của E03/E05/A01/A02 và phát hiện trường hợp M01 passed sai. |

**Hai hoặc ba failure cases nào cần thêm vào benchmark ở vòng tiếp theo?**

> Bổ sung ba case phụ cho vòng kiểm tra tiếp theo: (1) một cách diễn đạt khác
> của yêu cầu chẩn đoán y tế để kiểm tra scope và safe refusal; (2) đơn đặt
> sát mốc 01/09/2026, có OrbitPlus, hỏi hạn trả theo đúng version chính sách;
> (3) điện thoại đã mở bị lỗi vào ngày 20, hỏi đồng thời return, warranty và
> loaner để kiểm tra điều kiện 14 ngày, remedy sau diagnosis và USD 200
> deposit. Giữ nguyên 20 slots của golden dataset đã nộp để so baseline;
> lưu ba case này trong bộ regression bổ sung hoặc version dataset kế tiếp.

---

## 7. Final Reflection

**Điều gì trong kết quả benchmark trái với dự đoán ban đầu của bạn?**

> Chỉ nhìn Context Precision trung bình 0.947 có thể tưởng retrieval đã ổn,
> nhưng A01 vẫn lấy sai loại tài liệu và đạt Precision 1.000 do ngưỡng trùng
> từ thấp; H02 thiếu hẳn đoạn về quyền chọn remedy. Ngược lại, H04 đã lấy
> đúng hai nguồn đầu mà vẫn bỏ hai điều kiện. M01 được passed=True dù câu
> trả lời tự phủ định; E03/E05 bị failed dù nội dung cơ bản đúng. Vì vậy
> 45% pass rate không phải tỷ lệ đúng về nghiệp vụ hay tỷ lệ câu trả lời an
> toàn. Trace và kiểm tra claim cho thấy lỗi retrieval, generation và đánh
> giá có thể chồng lên nhau.

**Word-overlap heuristics trong lab có giới hạn gì? Nếu đưa hệ thống vào
production, bạn sẽ thay hoặc bổ sung metric nào?**

> Heuristic word overlap bỏ qua nghĩa tương đương, phủ định, quan hệ ngày
> tháng và phép tính; cùng một bộ từ có thể tạo claim đúng hoặc sai. Nó
> không biết “day 20” nằm ngoài “14 days”, không phát hiện hai câu mâu
> thuẫn của M01, và có thể phạt một safe refusal như A01. Context Precision
> với ngưỡng liên quan 0.1 cũng dễ coi chunk sai chủ đề là liên quan.
> Nếu đưa vào production, tôi sẽ bổ sung kiểm tra claim đối chiếu evidence
> (semantic entailment hoặc LLM judge có rubric rõ và được hiệu chỉnh bằng
> nhãn người), test luật xác định cho ngày/tiền/chính sách, và đánh giá riêng
> safe refusal, privacy, safety. Theo dõi cả false positive/false negative
> của evaluator, không dùng riêng Overall hay pass rate để quyết định.
