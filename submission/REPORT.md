# K4-Track02-Day17 — Report cá nhân

Phần phân tích tối đa một trang, không tính output ở phần 5.
Định dạng tham chiếu và phạm vi tính trang: [SUBMISSION.md](../docs/SUBMISSION.md).

**Họ tên / MSSV:** Trần Thu Phương / 02366

**Repo:** https://github.com/TranThuPhuong230804/K4-Day17-TranThuPhuong-02366-DataPipeline-Engineering

**Commit mã nguồn dùng để kiểm tra:** `9aa230f1bc573db90efa28905aadba981fbb1cc4`

**AI đã dùng và phạm vi hỗ trợ:** OpenAI Codex hỗ trợ đọc repo, phân tích ba lỗi, đề xuất và thực hiện thay đổi trong pipeline/REPORT, chạy và diễn giải verify, pytest, rerun, lateness, dbt build và parity; kết quả được đối chiếu bằng bộ kiểm tra có sẵn của repo.

**Nguồn tham khảo khác:** README và tài liệu trong `docs/` của repo; không dùng nguồn ngoài.

## 1. Ba lỗi

Mỗi lỗi 4 dòng. Triệu chứng = thứ bạn *thấy* đầu tiên (check nào fail, số nào lạ,
checksum nào lệch) — không phải cách sửa.

| | Lỗi Silver | Lỗi late data | Lỗi xoá (CDC) |
|---|---|---|---|
| **Triệu chứng** | `silver_tickets` trùng `ticket_id`; T-91 có cả trạng thái cũ và mới. | Feature lệch full recompute; ba event u05 của 12/08 đến ngày 15/08 bị thiếu. | T-97 đã xoá nhưng còn trong Silver, snapshot mới nhất và RAG. |
| **Nguyên nhân gốc** | Chỉ dedup trong batch theo `_lsn`, sau đó `INSERT`; không upsert theo khóa giữa batch. | `LOOKBACK_DAYS=0` không tính lại partition event date khi event đến muộn. | Delete có `after=null`, khóa nằm trong `before`; parser lấy khóa từ `after` nên loại nhầm. |
| **Cách sửa** | `silver.py`: `MERGE` theo `ticket_id`, chỉ update khi LSN nguồn lớn hơn. | `config.py`: lookback 3; `gold.py` overwrite cửa sổ nhưng vẫn group theo `event_time`. | `staging.py`: `coalesce(after.ticket_id,before.ticket_id)`; giữ delete+LSN, bỏ Kafka tombstone `_op IS NULL`. |
| **Khái niệm** | Keyed/idempotent upsert, CDC ordering. | Event time ≠ ingest time; lookback/overwrite partition. | CDC delete ≠ Kafka tombstone; delete propagation. |

## 2. Các con số

- 43 record Bronze: P50 `0`, P95 `2,9`, P99 `3`, max `3` ngày → `LOOKBACK_DAYS=ceil(P99)=3`, đủ bao phủ độ trễ theo ngày lịch.
- `submission/checksums.txt`: **PASS** — Gold checksum: `39e115c510ecdf526800eac227158a4f`
- `make parity`: **PARITY**

## 3. Lựa chọn công cụ / kỹ thuật (mỗi dòng một câu "vì sao")

- `silver_tickets` dùng MERGE (`unique_key=ticket_id`, update khi LSN mới hơn); feature dùng overwrite/microbatch (`batch_size=day`, `lookback=3`) để vừa giữ một trạng thái hiện tại vừa sửa đúng partition late data.
- Tombstone Silver giữ delete+LSN để chống hồi sinh khi replay, đồng thời xoá PII khỏi trạng thái hiện tại.
- Snapshot training dựng lại từ Bronze “as of” để tái lập đúng dữ liệu đã biết và không âm thầm đổi version đã công bố.
- DuckDB/dbt phù hợp dữ liệu nhỏ, local, dễ tái lập; Spark sẽ thêm chi phí cụm và vận hành không cần thiết.

## 4. Hai câu hỏi suy ngẫm

1. Snapshot `v2026-08-12`..`v2026-08-14` vẫn chứa văn bản của T-97 (đã bị xoá ngày
   08-15). "Snapshot bất biến" và "quyền được xoá dữ liệu" mâu thuẫn — bạn xử lý thế nào?

   Trong lab, snapshot cũ được giữ để tái lập; T-97 chỉ bị loại khỏi snapshot mới nhất và RAG. Production phải có deletion ledger, retention và quy trình erasure/crypto-shred cho mọi snapshot, cache, index và model dẫn xuất, kèm audit và dựng lại artifact. Quyền xoá là ràng buộc cao hơn tính bất biến: version cũ phải bị thu hồi hoặc redact và phát hành version/checksum mới.
2. Regex che được email và số điện thoại, nhưng tên "Nguyễn Văn An" vẫn còn. Bạn sẽ
   đặt chốt PII nào, ở tầng nào, và đo nó ra sao?

   Tôi đặt chốt trước khi dữ liệu rời Bronze sang Silver: regex cho định danh có cấu trúc kết hợp NER tiếng Việt/DLP cho tên, địa chỉ và thực thể nhạy cảm; mẫu không chắc chắn được quarantine hoặc duyệt thủ công. Chốt thứ hai quét lại training set và RAG trước khi publish. Tôi đo precision/recall/F1 trên tập PII tiếng Việt đã gán nhãn, ưu tiên false-negative rate, số leak trên một triệu record và canary PII; CI chặn phát hành nếu canary còn xuất hiện hoặc recall dưới ngưỡng.

## 5. Output (dán nguyên văn)

```text
PowerShell (Python 3.12.4 portable được dùng vì .venv cũ mất Python gốc):

$ & $py -m scripts.verify
=== verify.py — Day 17 pipeline contracts ===
  [OK ] Bronze  every daily batch landed as Parquet (7 days x 3 sources)
  [OK ] Bronze  re-landing a batch is a no-op (append-only, no duplicate file)
  [OK ] Bronze  Bronze keeps the raw truth: Kafka tombstone + redelivered events are still there
  [OK ] Silver  silver_tickets has exactly one row per ticket_id
  [OK ] Silver  T-91 shows its latest state: high / closed / bug
  [OK ] Silver  deleted ticket T-97 is a tombstone: is_deleted and no personal data left
  [OK ] Silver  no email / phone number survives past Bronze
  [OK ] Silver  silver_events has one row per event_id (Kafka redeliveries removed)
  [OK ] Silver  2 malformed events quarantined with a reason; the run did not halt
  [OK ] Gold    gold_feature_daily reconciles with a full recompute from Silver
  [OK ] Gold    u05's offline events of 08-12 (arrived 08-15) are counted on 08-12
  [OK ] Gold    LOOKBACK_DAYS covers measured P99 lateness (p99=3.00 days)
  [OK ] Gold    training set uses point-in-time priority (T-91 created as 'low')
  [OK ] Gold    late feedback creates a NEW snapshot version; the old one is untouched
  [OK ] Gold    latest training snapshot excludes the deleted ticket T-97
  [OK ] Gold    deletes propagate to the RAG index: no chunk of T-97
  [OK ] Gold    gold_doc_chunks: one row per chunk, and a re-run embeds 0 new chunks
  [OK ] Rerun   re-run 2026-08-12 three times -> Gold checksum identical to a fresh build

RESULT: 18/18 checks — ALL PASS
re-run checksums written to submission/checksums.txt

$ & $py -m pytest
..................................                                       [100%]
34 passed in 2.36s

$ & $py -m scripts.rerun_check
# Lab 17 — re-run check for 2026-08-12

run                     gold_feature_daily    gold_training_set     gold_doc_chunks       gold (combined)
fresh build             8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #1 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #2 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f
re-run #3 of 2026-08-12 8630e04a61d1          9370ca77af23          cb9ebd12fdcc          39e115c510ecdf526800eac227158a4f

RESULT: PASS — 3 re-runs, identical checksums

$ & $py main.py --lateness
event lateness over 43 Bronze records (calendar days): p50=0.00 p95=2.90 p99=3.00 max=3
-> lookback must be >= ceil(p99) = 3 day(s); config.LOOKBACK_DAYS = 3

$ Push-Location dbt_project
$ & $py -m dbt.cli.main build --profiles-dir . --event-time-start 2026-08-10 --event-time-end 2026-08-17
Running with dbt=1.12.5
Registered adapter: duckdb=1.11.0
Found 5 models, 13 data tests, 2 sources, 502 macros, 1 unit test
Finished running 3 incremental models, 13 data tests, 1 unit test, 2 view models in 0 hours 0 minutes and 2.84 seconds (2.84s).
Completed successfully
Done. PASS=19 WARN=0 ERROR=0 SKIP=0 NO-OP=0 REUSED=0 TOTAL=19
$ Pop-Location

$ & $py -m scripts.parity
=== parity: lite pipeline vs dbt ===
  [OK ] silver_tickets       lite 3c15dfd43701  dbt 3c15dfd43701
  [OK ] gold_feature_daily   lite 8630e04a61d1  dbt 8630e04a61d1
RESULT: PARITY — both implementations agree
```

Nếu dùng PowerShell, ghi lệnh tương đương và output thực tế theo [SUBMISSION.md](../docs/SUBMISSION.md).
Nếu làm bonus, thêm output B1 hoặc đường dẫn bằng chứng B2 ở cuối phần này.

## Bằng chứng bonus

- B2 — Brainstorm kiến trúc: [`bonus/DESIGN.md`](../bonus/DESIGN.md).
