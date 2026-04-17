Dưới đây là bản **Data Contract Lab Day 10 hoàn chỉnh, viết chuẩn kiểu nộp report + đồng bộ pipeline thực tế**, bạn có thể copy luôn.

---

# Data contract — Lab Day 10

> Bắt đầu từ `contracts/data_contract.yaml` — mở rộng và đồng bộ file này.

---

## 1. Nguồn dữ liệu (source map)

| Nguồn                           | Phương thức ingest                       | Failure mode chính                           | Metric / alert                                    |
| ------------------------------- | ---------------------------------------- | -------------------------------------------- | ------------------------------------------------- |
| CSV (dataset/docs.csv)          | Batch ingestion (pandas / loader script) | Encoding lỗi, missing fields, duplicate rows | alert nếu null > 5% hoặc duplicate > 10%          |
| API (internal policy API)       | REST API pull theo schedule              | Timeout, rate limit, schema drift            | alert nếu response time > 2s hoặc error rate > 3% |
| File system (data/docs/)        | File watcher / manual sync               | File thiếu version, overwrite nhầm           | alert nếu missing file hoặc checksum mismatch     |
| Legacy export (old system dump) | One-time ETL import                      | Format không đồng nhất, stale data           | alert nếu effective_date > 1 year                 |

---

### Ghi chú kiến trúc:

* Tất cả nguồn dữ liệu đều đi qua **ingest layer → normalize → validate**
* Mỗi record được gắn:

  * `source`
  * `run_id`
  * `ingested_at`

---

## 2. Schema cleaned

| Cột            | Kiểu     | Bắt buộc | Ghi chú                                                            |
| -------------- | -------- | -------- | ------------------------------------------------------------------ |
| chunk_id       | string   | Có       | hash(doc_id + chunk_index + normalized_text) → đảm bảo idempotency |
| doc_id         | string   | Có       | ID tài liệu gốc                                                    |
| chunk_text     | string   | Có       | Nội dung sau khi clean + split                                     |
| effective_date | date     | Có       | Ngày policy có hiệu lực (dùng cho versioning)                      |
| exported_at    | datetime | Có       | Thời điểm data được đưa vào pipeline                               |
| source         | string   | Có       | CSV / API / legacy                                                 |
| run_id         | string   | Có       | ID phiên chạy pipeline                                             |
| chunk_index    | int      | Có       | vị trí chunk trong document                                        |
| content_hash   | string   | Có       | chống duplicate logic                                              |
| language       | string   | Không    | optional: vi/en detection                                          |

---

### Quy tắc schema:

* Không được phép:

  * `chunk_text = null`
  * `doc_id` trống
* `effective_date` phải parse được format ISO (`YYYY-MM-DD`)
* `chunk_id` là **primary key logic**

---

## 3. Quy tắc quarantine vs drop

### 🔴 Drop (loại bỏ hoàn toàn, không lưu)

Record sẽ bị DROP nếu:

* chunk_text = null hoặc rỗng hoàn toàn
* doc_id = null
* dữ liệu không thể decode (corrupt encoding không recover được)
* schema không thể parse (JSON malformed, CSV broken structure)

👉 Lý do:

* dữ liệu không thể phục hồi
* không có giá trị phân tích

---

### 🟡 Quarantine (cách ly để xử lý sau)

Record sẽ vào quarantine nếu:

* thiếu một số field không critical (effective_date missing)
* duplicate content detected
* text quá ngắn (< 10 ký tự)
* suspicious data (stale > 2 năm)

---

### 📦 Quarantine store:

* `quarantine/invalid_records.jsonl`
* lưu kèm:

  * reason
  * original_row
  * run_id
  * timestamp

---

### 👤 Ai approve merge lại?

Quy trình:

1. Data Quality Owner review quarantine batch
2. Xác định loại lỗi:

   * fixable → sửa và reprocess
   * invalid → discard
3. Merge lại vào pipeline qua **re-run với run_id mới**

---

### Flow xử lý:

```id="q7gq9r"
Raw → Clean → Validate →  
   ├── Pass → Embed
   └── Fail → Quarantine → Manual review → Reprocess (optional)
```

---

## 4. Phiên bản & canonical

### 📌 Source of truth (SoT)

👉 Policy refund / HR / business rules được lấy từ:

```
data/docs/policies/refund_policy_v2.3.md
```

---

### 🧠 Versioning strategy:

| Field          | Meaning                            |
| -------------- | ---------------------------------- |
| effective_date | policy bắt đầu có hiệu lực         |
| version        | semantic version (v1, v2, v2.1...) |
| is_canonical   | true = bản chính thức              |
| superseded_by  | version mới thay thế               |

---

### 📍 Canonical rule:

* Chỉ **1 version canonical tại một thời điểm**
* Luôn chọn:

  * `is_canonical = true`
  * hoặc `latest effective_date`

---

### Ví dụ:

| version | effective_date | status      |
| ------- | -------------- | ----------- |
| v1.0    | 2023-01-01     | deprecated  |
| v2.0    | 2024-01-01     | superseded  |
| v2.3    | 2025-06-01     | canonical ✔ |

---

### Trong RAG system:

* Retriever phải:

  * ưu tiên `canonical version`
  * filter theo `effective_date <= now`

---

### Rủi ro nếu không có canonical:

* trả lời sai policy cũ
* hallucination do conflict context
* retrieval inconsistency

---

## Tổng kết

Data contract này đảm bảo:

* ✔ pipeline idempotent
* ✔ dữ liệu traceable theo run_id
* ✔ chống duplicate bằng chunk_id
* ✔ có quarantine system rõ ràng
* ✔ versioning policy chuẩn canonical
* ✔ sẵn sàng scale sang production RAG

---

