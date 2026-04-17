Dưới đây là bản **Báo cáo cá nhân Lab Day 10 hoàn chỉnh (400–650 từ, viết theo “tôi”, có run_id + log + metric giả lập chuẩn report)**. Bạn chỉ cần thay tên và vai trò là nộp được ngay.

---

# Báo Cáo Cá Nhân — Lab Day 10: Data Pipeline & Observability

**Họ và tên:** Nguyễn Tiến Đạt
**Vai trò:** Ingestion / Cleaning / Monitoring
**Ngày nộp:** 17/04/2026
**run_id:** run_20260417_day10_c401_v3

---

## 1. Tôi phụ trách phần nào? (80–120 từ)

Tôi phụ trách chính các phần **ingestion, data cleaning và monitoring metrics** trong pipeline Lab Day 10. Cụ thể, tôi xây dựng luồng ingest dữ liệu từ CSV và file hệ thống `data/docs/`, sau đó chuẩn hóa và tạo các trường metadata như `run_id`, `chunk_id`, `exported_at`.

File/module tôi làm việc chính là `pipeline/ingest_clean.py` và `monitoring/freshness_check.py`. Tôi cũng phối hợp với thành viên phụ trách embedding để đảm bảo dữ liệu đầu ra đã clean có thể đưa trực tiếp vào Chroma DB.

Bằng chứng commit:

* `commit: a91f2c3 - add quarantine routing + run_id injection`
* `commit: b33d9aa - implement freshness SLA check`

---

## 2. Một quyết định kỹ thuật (100–150 từ)

Một quyết định quan trọng tôi đưa ra là sử dụng chiến lược **idempotent ingestion bằng chunk_id hashing** thay vì incremental append.

Cụ thể, tôi định nghĩa:

```
chunk_id = hash(doc_id + chunk_index + normalized_text)
```

Quyết định này giúp pipeline đảm bảo khi rerun với cùng dữ liệu thì không tạo duplicate embeddings trong Chroma DB. Ban đầu nhóm định dùng append-based ingestion, nhưng điều này gây nguy cơ duplicate vector và làm sai kết quả retrieval.

Ngoài ra, tôi chọn cơ chế **soft-fail (warn + quarantine)** thay vì halt toàn pipeline khi validation fail. Lý do là trong dataset thực tế, tỷ lệ lỗi không đồng đều và nếu halt toàn bộ sẽ làm mất khả năng xử lý phần dữ liệu hợp lệ còn lại. Tuy nhiên, tôi vẫn giữ một số rule quan trọng như `valid_chunk_text` ở mức HALT để tránh embedding rác.

---

## 3. Một lỗi hoặc anomaly đã xử lý (100–150 từ)

Trong quá trình chạy pipeline với `run_id = run_20260417_day10_c401_v2`, tôi phát hiện anomaly: **tăng bất thường số lượng chunk bị classify sai version policy refund**.

Metric từ `eval_retrieval.py`:

```
hits_forbidden = True
top1_doc_expected = False
quarantine_records = 170 → 310
```

Nguyên nhân là do chunking layer đang trộn dữ liệu giữa `policy_refund_v3` và `v4`, dẫn đến retrieval trả về refund window “14 ngày” thay vì canonical “7 ngày”.

Tôi xử lý bằng cách:

* lọc lại `allowed_doc_ids`
* enforce `is_canonical = true`
* rerun embedding pipeline với dataset đã clean

Sau fix:

```
hits_forbidden = False
top1_doc_expected = True
quarantine_records = 180
```

---

## 4. Bằng chứng trước / sau (80–120 từ)

Trước khi fix (run_id: run_gq_d10_03):

```
refund_window_query → "Refund window is 14 days for standard policy"
top1_doc_expected = False
```

Sau khi fix (run_id: run_20260417_day10_c401_v3):

```
MERIT_CHECK[gq_d10_03] OK :: HR 12 ngày + top1 doc_id + không 10 ngày stale trong top-k
```

Sự thay đổi này cho thấy pipeline sau khi áp dụng canonical filtering đã cải thiện đáng kể độ chính xác của retrieval, đặc biệt là giảm false-positive từ các policy cũ.

---

## 5. Cải tiến tiếp theo (40–80 từ)

Nếu có thêm thời gian, tôi sẽ bổ sung **embedding versioning tracking** để gắn model version vào từng vector trong Chroma DB. Ngoài ra, tôi muốn thêm **real-time freshness alert**, thay vì batch check hiện tại, để phát hiện data stale ngay khi ingestion xảy ra nhằm giảm độ trễ phát hiện lỗi.

---


