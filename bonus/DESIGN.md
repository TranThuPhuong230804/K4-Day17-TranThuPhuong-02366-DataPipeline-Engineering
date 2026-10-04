# B2 — Pipeline dữ liệu cho trợ lý AI tra cứu hợp đồng bảo hiểm tiếng Việt

## Bài toán và ràng buộc thực tế

Tôi thiết kế một pipeline cho trợ lý AI nội bộ phục vụ tư vấn viên bảo hiểm. Người dùng hỏi các câu như “quyền lợi nội trú của hợp đồng này là bao nhiêu?”, “điều khoản loại trừ mới có áp dụng cho hợp đồng ký năm 2024 không?” hoặc “quy định A dẫn chiếu sang phụ lục nào?”. Câu trả lời phải có trích dẫn đúng trang, đúng phiên bản có hiệu lực và không làm lộ dữ liệu của khách hàng khác.

Dữ liệu gồm PDF quy tắc sản phẩm, phụ lục, biểu phí, công văn và hợp đồng khách hàng. Một phần là PDF có text; phần còn lại là bản scan lệch trang, bảng nhiều cột, dấu mộc, chữ ký và tiếng Việt có dấu. Cùng một sản phẩm có nhiều phiên bản, ngày hiệu lực và quan hệ thay thế; hợp đồng cá nhân còn chứa họ tên, số định danh, địa chỉ và thông tin sức khỏe. Quy mô ban đầu khoảng 200.000 tài liệu, tăng 5.000–10.000 tài liệu mỗi ngày. SLA cập nhật văn bản nghiệp vụ là dưới hai giờ, còn truy vấn phải trả lời trong khoảng ba giây. Sai phiên bản hoặc rò PII nghiêm trọng hơn việc chậm vài phút.

## Sơ đồ kiến trúc

```text
S3/email/DMS/API
      |
      v
[Bronze: file gốc + manifest + content hash]
      |
      +--> [Virus/type check] --bad--> [Quarantine + cảnh báo]
      |
      v
[Extract text/layout] --scan--> [OCR tiếng Việt + bảng]
      |
      v
[Silver: document/page/block + PII tags + version graph]
      |                         |
      |                         +--> [Human review queue]
      v
[Chunk + metadata + embedding cache] ---> [Vector index]
      |
      +--> [Entity/reference extraction] -> [Knowledge graph]
      |
      v
[Atomic publish manifest: index_version]
      |
      v
[Retriever hybrid + ACL filter + reranker] -> [LLM] -> [answer + citations]
      |
      v
[Trace/feedback] -> [redact] -> [eval candidates] -> [offline evaluation]
```

## 1. Nguồn và hình dạng: lưu file hay chuẩn hóa ngay?

**Quyết định.** Bronze lưu nguyên byte của tài liệu, manifest nguồn, thời điểm nhận, checksum SHA-256, tenant, MIME type và `document_id`; không OCR đè lên file gốc. Silver mới chuẩn hóa thành các bảng `document`, `page`, `block`, `table_cell` và `document_version`. Mỗi block giữ bounding box, số trang, phương pháp extract và confidence để câu trả lời quay lại được đúng vị trí nguồn. Schema có version; cột mới được thêm theo hướng tương thích, còn thay đổi phá vỡ schema phải vào quarantine.

**Đánh đổi.** Chuẩn hóa ngay khi nhận sẽ đơn giản hơn và giảm dung lượng, nhưng làm mất bằng chứng gốc khi OCR/model thay đổi. Lưu cả file gốc và layout tốn storage, đổi lại có thể chạy lại parser, audit trích dẫn và chứng minh tài liệu nào tạo ra câu trả lời. Tôi chọn khả năng tái lập vì storage object rẻ hơn chi phí xử lý sai quyền lợi bảo hiểm.

## 2. Batch hay streaming: độ tươi nào thực sự cần thiết?

**Quyết định.** Dùng microbatch 15 phút cho manifest mới và một backfill ban đêm để đối soát; tài liệu khẩn cấp có priority lane nhưng vẫn đi qua cùng validation. Event chỉ kích hoạt công việc, còn trạng thái chính nằm trong bảng keyed theo `document_id + version + content_hash`. Không yêu cầu streaming từng byte PDF vì người dùng không hưởng lợi từ độ trễ vài giây.

**Đánh đổi.** Streaming hoàn toàn cho cảm giác real-time nhưng tăng small files, orchestration và nguy cơ publish index nửa chừng. Batch hằng đêm rẻ hơn nhưng không đạt SLA hai giờ cho công văn mới. Microbatch là điểm cân bằng: đủ tươi, gom được OCR/embedding thành lô hiệu quả và dễ replay. Publish dùng manifest nguyên tử; chỉ khi mọi page, chunk, embedding và ACL hoàn tất thì `index_version` mới được chuyển sang active.

## 3. Hợp đồng dữ liệu và PII: dòng xấu đi đâu?

**Quyết định.** Trước khi vào Silver, pipeline kiểm MIME/signature thật, virus, checksum, số trang, encoding, OCR confidence, tỷ lệ ký tự lỗi, trang trống bất thường và các trường bắt buộc như sản phẩm/ngày hiệu lực. PII được nhận diện bằng regex cho số định danh, tài khoản, điện thoại kết hợp NER tiếng Việt cho tên, địa chỉ và dữ liệu sức khỏe. Dữ liệu khách hàng được tách namespace, mã hóa và gắn ACL trước khi chunk; index dùng cho tài liệu công khai không bao giờ nhận chunk có PII chưa xử lý.

**Đánh đổi.** Chặn mọi tài liệu confidence thấp giảm rủi ro nhưng có thể làm mất tài liệu scan cũ quan trọng; tự động cho qua tối đa tăng recall nhưng dễ đưa text sai hoặc PII vào model. Tôi chọn ba mức: pass, quarantine và human review. Cảnh báo gửi cho data owner khi quarantine rate vượt baseline hoặc PII leak canary xuất hiện. Chất lượng được đo bằng OCR character/word error rate, precision/recall PII, tỷ lệ trang cần review, freshness SLA và số citation không khớp trang.

## 4. RAG hay knowledge graph: một index có đủ không?

**Quyết định.** Dùng hybrid retrieval. Vector + BM25 xử lý câu hỏi tra cứu điều khoản; graph chỉ lưu các quan hệ có giá trị như `THAY_THẾ`, `ÁP_DỤNG_CHO`, `DẪN_CHIẾU`, `CÓ_PHỤ_LỤC` và khoảng hiệu lực. Retriever trước hết lọc tenant, quyền truy cập và ngày hiệu lực, sau đó hợp nhất kết quả lexical/vector với các node liên quan rồi rerank. LLM chỉ nhận các đoạn đã qua ACL và luôn phải trả citation.

**Đánh đổi.** Pure vector đơn giản, rẻ và tốt cho tương đồng ngữ nghĩa nhưng khó trả lời câu multi-hop hoặc phân biệt phiên bản. Full knowledge graph cho mọi thực thể dễ tạo “graph rác”, tốn công ontology và entity resolution. Hybrid tăng hai hệ lưu trữ và độ phức tạp vận hành, nhưng dùng graph có giới hạn cho đúng quan hệ pháp lý đem lại lợi ích rõ. Tôi chấp nhận chi phí này và đo groundedness, citation accuracy, retrieval recall@k, latency p95 và tỷ lệ “không đủ bằng chứng”.

## 5. Failure semantics, phiên bản và xoá: replay có an toàn không?

**Quyết định.** Mỗi stage ghi theo khóa ổn định và chỉ nhận revision mới hơn. OCR cache theo `content_hash + extractor_version`; embedding cache theo `text_hash + model_version`; graph edge có source block và pipeline version. Delete từ DMS tạo tombstone, gỡ tài liệu khỏi manifest active, vector index và graph; hàng tombstone giữ revision để backfill cũ không làm tài liệu hồi sinh. Backfill ghi vào `index_version` mới, đối chiếu count/checksum/eval rồi mới đổi alias; thất bại thì bỏ version chưa active, không sửa index đang phục vụ.

**Đánh đổi.** Hard delete ngay ở mọi tầng giảm storage nhưng làm mất thứ tự thay đổi và khiến replay nguy hiểm. Tombstone hỗ trợ idempotency/audit nhưng không tự đáp ứng quyền xoá PII. Vì vậy metadata tombstone được giữ, còn payload PII phải crypto-shred hoặc purge theo deletion ledger và retention; các index/model dẫn xuất được dựng lại khi cần. Side effect như gửi cảnh báo dùng outbox với idempotency key để retry không gửi trùng.

## 6. Flywheel và chi phí: feedback có đáng tin để train không?

**Quyết định.** Lưu trace đã redact gồm query, tài liệu được phép truy cập, chunk đã lấy, citation, latency, model version và feedback. Feedback không đi thẳng vào training: mẫu được dedup, kiểm PII, tách theo thời gian và review các failure quan trọng trước khi trở thành eval set. Bộ eval cố định theo product/version được chạy trước mỗi lần publish; dữ liệu tương lai không được dùng để đánh giá model quá khứ.

**Đánh đổi.** Tự động học từ mọi lượt thumbs-up/down tạo flywheel nhanh nhưng feedback thiên lệch, dễ bị prompt injection và tự củng cố câu trả lời sai. Human review toàn bộ sạch hơn nhưng không scale. Tôi chọn sampling theo rủi ro: review 100% trường hợp khiếu nại, citation sai hoặc confidence thấp; lấy mẫu phần còn lại. Chi phí lớn nhất là OCR và embedding, nên chỉ OCR trang ảnh, dedup theo hash, cache embedding và dùng model nhỏ cho routing/PII trước khi gọi model lớn. Dashboard theo dõi chi phí trên mỗi tài liệu và mỗi câu trả lời đúng, không chỉ tổng token.

## Phương án bị loại

Tôi loại phương án “đưa nguyên PDF vào LLM ở thời điểm truy vấn, không tiền xử lý và không index”. Nó hấp dẫn vì prototype rất nhanh, nhưng latency và chi phí tăng theo độ dài tài liệu, không bảo đảm chọn đúng phiên bản, khó áp ACL ở cấp đoạn, không tái lập được citation và mỗi truy vấn lại trả tiền OCR/context. Nó cũng không có nơi rõ ràng để quarantine trang lỗi hay truyền delete xuống các artifact. Với dữ liệu bảo hiểm, sự đơn giản ban đầu không bù được rủi ro vận hành và tuân thủ.

## Tiêu chí thành công và mốc triển khai

MVP bắt đầu với một sản phẩm bảo hiểm, 10.000 tài liệu và bộ eval khoảng 300 câu có đáp án/citation do nghiệp vụ duyệt. Điều kiện phát hành gồm citation accuracy ≥ 95%, không có canary PII leak, retrieval recall@10 ≥ 90%, p95 latency ≤ 3 giây và 99% tài liệu hợp lệ active trong hai giờ. Sau khi đạt các ngưỡng này mới mở rộng ontology, tenant và volume; nếu metric giảm, alias quay lại `index_version` trước thay vì sửa nóng dữ liệu đang phục vụ.
