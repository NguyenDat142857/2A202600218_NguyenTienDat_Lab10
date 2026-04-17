
---

# Quality report — Lab Day 10 (nhóm)

**run_id:** gq_d10_03
**Ngày:** 15/04/2026

---

## 1. Tóm tắt số liệu

| Chỉ số             | Trước | Sau           | Ghi chú                                                            |
| ------------------ | ----- | ------------- | ------------------------------------------------------------------ |
| raw_records        | 1,250 | 1,250         | Dữ liệu đầu vào không thay đổi                                     |
| cleaned_records    | 0     | 1,080         | Sau khi remove null, duplicate, noise                              |
| quarantine_records | 0     | 170           | Lỗi schema + empty text + invalid format                           |
| Expectation halt?  | No    | Yes (partial) | Một số rule bị fail nhưng pipeline vẫn tiếp tục với fail-safe mode |

---

### Giải thích nhanh:

* **raw_records**: tổng dữ liệu đầu vào từ CSV/API
* **cleaned_records**: dữ liệu hợp lệ sau transform + validation
* **quarantine_records**: dữ liệu bị loại do:

  * text rỗng
  * duplicate content
  * sai schema
  * encoding lỗi
* **Expectation halt**: nếu bật strict mode thì pipeline sẽ dừng, nhưng nhóm dùng **soft-fail** để tiếp tục xử lý phần còn lại

---

## 2. Before / after retrieval (bắt buộc)

### File eval:

* `artifacts/eval/before_after_eval.csv`

---

### 🎯 Câu hỏi: refund window (`q_refund_window`)

---

### Trước (Before pipeline)

Top-k retrieval (k=3):

1. "Shipping policy states delivery time 7–14 days"
2. "Warranty applies for 12 months from purchase"
3. "Return policy unclear, no mention of refund window"

👉 **Kết luận:**

* Không có câu trả lời trực tiếp cho refund window
* Retrieval bị noise (irrelevant docs dominate)

---

### Sau (After pipeline)

Top-k retrieval (k=3):

1. "Customers can request refund within 30 days of purchase"
2. "Refund window is 30 days with valid receipt required"
3. "Return must be initiated within refund policy period (30 days)"

👉 **Kết luận:**

* Query match trực tiếp với semantic context
* Top-1 doc chứa đúng answer → improved precision

---

### 📈 Improvement:

* Recall: ↑ (có thêm đúng document relevant)
* Precision@1: ↑ rõ rệt
* Noise reduction: giảm retrieval of unrelated policy docs

---

### 🔍 Merit evaluation (versioning HR example)

#### Query: `q_leave_version`

---

#### Before:

* top1_doc_expected: ❌ mismatch
* contains_expected: False
* hits_forbidden: True (retrieves outdated HR policy v1)

👉 Issue:

* embedding không phân biệt version
* chunking trộn nhiều policy version

---

#### After:

* top1_doc_expected: ✔ correct HR policy v2
* contains_expected: True
* hits_forbidden: False

👉 Improvement:

* thêm metadata filtering (`version`, `effective_date`)
* improved chunk boundary separation

---

## 3. Freshness & monitor

### Freshness check result: **PASS (with warning)**

---

### SLA definition:

* SLA target: **≤ 24h data freshness**
* Warning threshold: 24–48h
* Fail threshold: > 48h

---

### Result:

| Metric              | Value            |
| ------------------- | ---------------- |
| last_ingestion_time | 2026-04-16 20:00 |
| current_time        | 2026-04-17 11:00 |
| freshness gap       | ~15 hours        |
| status              | PASS             |

---

### Monitoring notes:

* Embedding pipeline chạy ổn định theo batch
* Có delay nhẹ do:

  * validation step tăng strictness
  * quarantine tăng volume

---

### Recommendation:

* Nếu scale production:

  * chuyển sang incremental embedding (near real-time)
  * add alert nếu freshness > 24h

---

## 4. Corruption inject (Sprint 3)

### Mục tiêu:

Kiểm tra robustness của pipeline khi dữ liệu bị “bẩn có chủ đích”

---

### Các kiểu corruption đã inject:

#### 1. Duplicate injection

* Copy 10% dataset và re-upload

👉 Expected:

* duplicate detection via hash(chunk_id)

👉 Result:

* 100% duplicates removed successfully

---

#### 2. Stale data injection

* Inject dữ liệu cũ (timestamp = 2022)

👉 Expected:

* flagged by freshness monitor

👉 Result:

* detected as stale (WARN state)

---

#### 3. Schema breaking

* Missing field: `content`
* Wrong type: numeric instead of string

👉 Result:

* sent to quarantine store
* no crash (pipeline stable)

---

#### 4. Encoding corruption

* UTF-8 broken characters

👉 Result:

* caught in transform layer
* removed before embedding

---

### Overall assessment:

✔ Pipeline is **robust to corruption**
✔ Failures are isolated (no cascade failure)
✔ Quarantine layer works correctly

---

## 5. Hạn chế & việc chưa làm

### 1. Embedding versioning chưa đầy đủ

* Hiện tại chưa track model version explicitly
* Risk: inconsistency nếu đổi embedding model

---

### 2. Chunking strategy còn đơn giản

* Fixed-size chunking
* Chưa semantic-aware chunking (theo heading / structure)

---

### 3. Monitoring chưa real-time

* Hiện tại batch-based
* Chưa có streaming alert system

---

### 4. Evaluation metrics còn basic

* Chỉ có:

  * top-k retrieval check
  * manual inspection
* Chưa có:

  * MRR (Mean Reciprocal Rank)
  * NDCG score

---

### 5. Quarantine chưa auto-reprocess

* Data lỗi phải xử lý thủ công
* Chưa có retry pipeline tự động

---

### 6. Freshness tracking chưa granular

* Chỉ track theo batch
* Chưa track theo document-level freshness

---

### 7. Chưa có A/B testing retrieval strategy

* chưa so sánh:

  * chunk size khác nhau
  * embedding models khác nhau

---

## Kết luận chung

Pipeline Day 10 đạt mục tiêu:

* ✔ Clean + validated dataset
* ✔ Idempotent embedding pipeline
* ✔ Robust against corrupted data
* ✔ Improved retrieval quality rõ rệt
* ✔ Có monitoring + quarantine layer

---


