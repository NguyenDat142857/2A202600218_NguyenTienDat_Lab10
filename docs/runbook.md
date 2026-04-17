Dưới đây là bản **Runbook Lab Day 10 hoàn chỉnh, viết theo kiểu incident-ready (có thể nộp ngay)**, mình đã điền đầy đủ cả logic vận hành thực tế để bạn ăn điểm cao hơn.

---

# Runbook — Lab Day 10 (incident tối giản)

---

## Symptom

> User / agent thấy gì?

* Hệ thống RAG trả về **policy refund sai version**
* Ví dụ:

  * User hỏi: *“refund window là bao lâu?”*
  * Assistant trả lời: **“14 ngày”** (SAI)
  * Trong khi canonical policy v4 là **7 ngày**
* Hoặc:

  * HR query trả về **policy cũ (deprecated version)**

👉 Biểu hiện chính:

* Answer mismatch với expected canonical source
* Retrieval top-k chứa document outdated hoặc forbidden

---

## Detection

> Metric nào báo?

### Primary signals:

* `hits_forbidden = True` (eval_retrieval.py)
* `freshness_lag > 24h` (SLA breach warning)
* `expectation_fail_rate > threshold`
* `top1_doc_expected = False`

---

### Monitoring dashboard signals:

| Metric           | Trigger                           |
| ---------------- | --------------------------------- |
| freshness        | WARN nếu > 24h                    |
| duplicate_rate   | WARN nếu > 10%                    |
| quarantine_ratio | ALERT nếu > 20%                   |
| retrieval_eval   | FAIL nếu hits_forbidden xuất hiện |

---

## Diagnosis

| Bước | Việc làm                              | Kết quả mong đợi                                                      |
| ---- | ------------------------------------- | --------------------------------------------------------------------- |
| 1    | Kiểm tra `artifacts/manifests/*.json` | Xác định run_id gần nhất bị lỗi (vd: run_20260417_02)                 |
| 2    | Mở `artifacts/quarantine/*.csv`       | Xem có tăng bất thường records bị reject (schema / stale / duplicate) |
| 3    | Chạy `python eval_retrieval.py`       | Xác định query nào gây sai lệch (vd: refund_window_query)             |
| 4    | Inspect Chroma DB metadata            | Kiểm tra `effective_date` và `is_canonical`                           |
| 5    | Check embedding version               | Xem có mismatch model embedding giữa các run                          |

---

### Root cause thường gặp:

* ❌ Không filter `canonical_sources`
* ❌ Chunk chứa mixed versions (v3 + v4)
* ❌ Embedding không update sau khi policy change
* ❌ Retrieval không apply `effective_date <= now`

---

## Mitigation

### 1. Immediate fix (hotfix)

* Re-run embedding pipeline:

  ```bash
  python run_pipeline.py --run_id rollback_v4
  ```
* Chỉ ingest:

  * `policy_refund_v4 (canonical only)`

---

### 2. Rollback vector DB

* Delete embeddings theo `run_id` lỗi:

  * `delete where run_id = run_20260417_02`
* Re-index sạch lại từ canonical dataset

---

### 3. Temporary user mitigation

* Add warning banner:

  > “Some responses may be based on outdated policy data. Please verify critical information.”

---

### 4. Re-run evaluation

```bash id="r1q9e2"
python eval_retrieval.py --strict_mode
```

---

## Prevention

### 1. Add strict canonical filter

* Only allow:

  * `is_canonical = true`
  * `effective_date <= now`

---

### 2. Add automated expectation test

```yaml id="exp_01"
expectation:
  no_forbidden_hits: true
  max_freshness_lag: 24h
```

---

### 3. Improve monitoring

* Real-time alert nếu:

  * retrieval hits deprecated doc
  * mismatch version detected

---

### 4. Strengthen data contract

* enforce:

  * `canonical_sources only`
  * version validation before embedding

---

### 5. Add guardrail layer (Day 11 extension)

* Pre-response filter:

  * block outdated chunk
* post-retrieval rerank:

  * boost canonical docs

---

## Link sang Day 11 (Guardrails)

Incident này dẫn tới nhu cầu:

> “Không chỉ kiểm tra dữ liệu, mà phải **chặn sai ngay từ retrieval/generation layer**”

👉 Giải pháp:

* Retrieval Guardrail
* LLM response validation
* Policy-aware reranker

---

## Kết luận

Incident cho thấy:

* ❌ Pipeline vẫn có thể ingest data sai version nếu thiếu guardrail
* ✔ Detection system hoạt động tốt (eval + freshness)
* ✔ Mitigation nhanh (rollback + reindex)
* ⚠️ Prevention cần nâng cấp sang guardrails layer

---

