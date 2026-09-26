# BỘ CÂU HỎI PHẢN BIỆN LIVE DEMO (CHẤT VẤN CÁC NHÓM KHÁC)
**Dự án:** Data Pipeline & Data Observability for RAG (Day 10)  
**Nhóm chuẩn bị:** Team Enigma (K4-L3-DAY10)  
**Mục tiêu:** Đặt những câu hỏi sâu về kiến trúc thực tế, kiểm tra độ hiểu sâu của các nhóm về **Silent Failure, Idempotency, Great Expectations 1.x, SLA Freshness và cơ chế Self-Healing**.

---

## 🎯 PHẦN 1: INGESTION & LÀM SẠCH DỮ LIỆU (DATA CLEANING)

### Câu 1: Bóc tách rác XML & Chuẩn hóa định dạng Crossref
> *"Nhóm bạn đã xử lý các thẻ XML nội tại như `<jats:p>`, `<jats:italic>` và ký tự HTML entities (`&amp;`, `&lt;`) trong abstract của Crossref bằng phương pháp gì? Nếu không bóc tách mà đưa thẳng vào embedding thì ảnh hưởng thế nào đến cosine similarity?"*
- **Ý đồ chất vấn:** Rất nhiều nhóm chỉ dùng `.strip()` hoặc regex đơn giản và bỏ sót các thẻ lồng nhau.
- **Dấu hiệu câu trả lời tốt:** Nêu rõ dùng regex chuẩn hóa nhiều bước (hoặc BeautifulSoup/lxml) để strip sạch thẻ XML, giải mã HTML entities và loại bỏ khoảng trắng thừa. Giải thích rằng thẻ rác XML làm biến dạng vector embedding, làm loãng các token ngữ nghĩa then chốt.
- **Red flag:** Nhóm trả lời "không thấy có thẻ XML trong dữ liệu" hoặc "đưa nguyên raw string vào embedding model".

### Câu 2: Cấu trúc văn bản nhúng (`text_for_embedding`)
> *"Chuỗi `text_for_embedding` của nhóm bạn gồm những trường thông tin nào? Tại sao lại chọn cấu trúc ghép nối đó thay vì chỉ nhúng duy nhất trường `abstract`?"*
- **Ý đồ chất vấn:** Kiểm tra xem nhóm có tối ưu hóa ngữ cảnh tìm kiếm hay chỉ lấy bừa 1 cột.
- **Dấu hiệu câu trả lời tốt:** Kết hợp có cấu trúc: `Title + Authors + Subject + Publication Date + Cleaned Abstract`. Lý do: Người dùng thường tìm kiếm theo tên tác giả hoặc chuyên ngành, nếu chỉ nhúng abstract thì các query dạng metadata sẽ thất bại hoàn toàn.
- **Red flag:** Chỉ nhúng `abstract`, hoặc nhúng cả các trường định danh kỹ thuật vô nghĩa như `URL`, `publisher-id`.

---

## 🎯 PHẦN 2: VECTOR STORE & TÍNH BẢO ĐẢM IDEMPOTENCY

### Câu 3: Thử nghiệm chạy lại Pipeline 5 lần liên tiếp (Idempotency)
> *"Nếu tôi nhấn chạy pipeline của nhóm bạn 5 lần liên tiếp thì số lượng vector trong ChromaDB sẽ tăng gấp 5 lần (nhân bản dữ liệu) hay giữ nguyên? Nhóm dùng phương thức `collection.add()` hay `collection.upsert()` và cơ chế sinh ID như thế nào?"*
- **Ý đồ chất vấn:** Điểm mấu chốt của Data Engineering là **Idempotency** (tính lũy đẳng). Nhiều nhóm dùng `uuid.uuid4()` kết hợp `add()` khiến mỗi lần chạy lại làm trùng lặp dữ liệu, phình to vector index và làm loãng kết quả retrieval.
- **Dấu hiệu câu trả lời tốt:** Dùng mã DOI chuẩn hóa làm khóa định danh duy nhất (`id=doi_hash`), đồng thời sử dụng `upsert` hoặc xóa partition cũ trước khi ghi lại. Chạy $N$ lần thì trạng thái trong DB vẫn chỉ có đúng $20$ bản ghi.
- **Red flag:** Nhóm thú nhận số lượng vector tăng lên, hoặc phải xóa thủ công thư mục `chroma_db/` bằng tay mỗi khi muốn chạy lại.

### Câu 4: Xử lý Vector độ dài 0 hoặc chuỗi rỗng
> *"Khi gặp một bài báo không có abstract (chuỗi rỗng sau khi làm sạch), mô hình embedding sẽ sinh ra vector như thế nào và ChromaDB tính toán Cosine Distance ra sao? Hệ thống của nhóm bạn có chặn bản ghi này lại trước khi ghi vào DB không?"*
- **Ý đồ chất vấn:** Vector của chuỗi rỗng hoặc toàn ký tự trắng sẽ là vector nhiễu hoặc gây lỗi chia cho 0 khi chuẩn hóa chuẩn L2 (norm = 0).
- **Dấu hiệu câu trả lời tốt:** Nhóm có chốt kiểm tra `min_length >= 20` ký tự tại Data Cleaning và Great Expectations, lập tức loại bỏ hoặc gán giá trị mặc định có cảnh báo thay vì nhúng chuỗi rỗng.
- **Red flag:** Không biết chuyện gì xảy ra, hoặc trả lời rằng "mô hình tự động hiểu được chuỗi rỗng".

---

## 🎯 PHẦN 3: DATA OBSERVABILITY & GREAT EXPECTATIONS 1.X

### Câu 5: Vị trí đặt Chốt kiểm định (Quality Gate)
> *"Nhóm bạn đặt bộ kiểm định Great Expectations ở bước nào trong luồng pipeline: Kiểm định trước khi nhúng vector, hay nhúng vào ChromaDB xong rồi mới kiểm định? Nếu một batch bị lỗi, nhóm chọn Fail-Fast dừng toàn bộ pipeline hay cách ly (Quarantine) bản ghi lỗi?"*
- **Ý đồ chất vấn:** Kiểm tra tư duy kiến trúc. Nhiều nhóm chỉ chạy GX như một script độc lập để "chấm điểm cho có", chứ không hề tích hợp nó làm rào chắn (Quality Gate) cho bước Embedding.
- **Dấu hiệu câu trả lời tốt:** GX phải chạy *trước* khi gọi mô hình Embedding để bảo vệ Vector Store khỏi rác. Có cơ chế cách ly (Dead-letter partition) hoặc cảnh báo dừng luồng tự động.
- **Red flag:** "Chạy model RAG xong xuôi rồi mới bật GX lên kiểm tra", hoặc script GX nằm riêng biệt không hề liên kết với pipeline chính.

### Câu 6: Great Expectations 1.x Ephemeral vs Legacy Configuration
> *"Nhóm bạn sử dụng phiên bản Great Expectations nào? Trong GX 1.x, nhóm triển khai chế độ Ephemeral in-memory hay khởi tạo qua `great_expectations.yml` truyền thống? Những Expectations cốt lõi nào được định nghĩa?"*
- **Ý đồ chất vấn:** GX 1.x có sự thay đổi kiến trúc toàn diện so với 0.18.x (`DataContext`, `ExpectationSuite`, ephemeral mode). Nhóm nào làm thật sẽ trả lời rất trôi chảy về cú pháp GX 1.x.
- **Dấu hiệu câu trả lời tốt:** Nêu rõ các kỳ vọng: `ExpectColumnValuesToNotBeNull('doi')`, `ExpectColumnValuesToMatchRegex('doi', r'^10\.')`, `ExpectColumnValueLengthsToBeBetween('text_for_embedding', min_value=20)`.
- **Red flag:** Đọc nhầm cú pháp thư viện khác, hoặc nói dùng GX nhưng trong code chỉ là các lệnh `assert` và `if-else` thông thường.

---

## 🎯 PHẦN 4: SLA FRESHNESS & ĐO ĐẠC SILENT FAILURE

### Câu 7: Định nghĩa SLA Freshness và cách đo lường
> *"Nhóm bạn định nghĩa SLA độ tươi mới (Freshness) như thế nào? Cột mốc thời gian để tính `age_days` lấy từ đâu (thời điểm chạy máy hay ngày cố định)? Khi SLA bị vi phạm (ví dụ bài báo quá 180 ngày) thì hệ thống phản ứng ra sao?"*
- **Ý đồ chất vấn:** Kiểm tra tính thực tế của SLA. Nếu tính `age_days` theo `datetime.now()` mà dữ liệu lấy từ năm 2024 thì mọi bài báo đều vi phạm.
- **Dấu hiệu câu trả lời tốt:** Nêu rõ ngày mốc tham chiếu hợp lý (ví dụ: ngày phát hành so với snapshot date), ngưỡng SLA cụ thể (ví dụ 180 ngày), và tỷ lệ bài quá hạn (ví dụ: $0\%$ ở baseline, tăng lên $>35\%$ khi bị tiêm dữ liệu cũ).
- **Red flag:** Không biết công thức tính `age_days`, hoặc không giải thích được vì sao bài báo bị coi là quá hạn.

### Câu 8: Thực chứng Lỗi âm thầm (Silent Failure) trên chỉ số RAG
> *"Tại sao gọi hiện tượng này là 'Lỗi âm thầm' (Silent Failure)? Nhóm bạn đã chứng minh bằng số liệu nào cho thấy hệ thống không báo lỗi runtime nhưng kết quả trả về của RAG bị phá hủy nghiêm trọng?"*
- **Ý đồ chất vấn:** Đánh vào trọng tâm đề tài Day 10.
- **Dấu hiệu câu trả lời tốt:** Trình bày được bảng đo đạc định lượng đối chiếu 3 trạng thái:
  - **Baseline:** Hit Rate $100\%$, Mean Token F1 $\approx 1.0$, GX PASS.
  - **Corrupted:** Vector DB vẫn trả về kết quả (Exit Code 0), nhưng Hit Rate sụt giảm nghiêm trọng (ví dụ xuống $40\%$), Mean Token F1 rơi xuống $0.57$, và LLM bắt đầu sinh câu trả lời hallucinate.
  - **Repaired:** Tự phục hồi về lại $100\%$.
- **Red flag:** Nhóm bảo "khi tiêm lỗi thì chương trình bị crash/báo exception", chứng tỏ nhóm không hiểu khái niệm *Silent Failure*.

---

## 🎯 PHẦN 5: CƠ CHẾ TỰ PHỤC HỒI (SELF-HEALING)

### Câu 9: Tính tự động của khâu Self-Healing
> *"Cơ chế tự phục hồi (Self-Healing) của nhóm bạn diễn ra hoàn toàn tự động trong mã lệnh hay cần con người can thiệp sửa file thủ công? Khi sửa xong, làm sao nhóm bảo đảm không có vector lỗi nào còn sót lại trong ChromaDB?"*
- **Ý đồ chất vấn:** Phân biệt nhóm code luồng khép kín tự động với nhóm sửa bằng tay (manual fix).
- **Dấu hiệu câu trả lời tốt:** Pipeline có luồng `repair_flow`: Khi GX hoặc SLA phát hiện batch lỗi, hàm sửa lỗi tự động rollback về snapshot raw sạch, làm sạch lại, gọi `upsert` hoặc re-index phân vùng để ghi đè vector hỏng.
- **Red flag:** "Nhóm mở file csv ra xóa dòng bị lỗi bằng tay rồi chạy lại script".

---

## 💡 BẢNG ĐỐI CHIẾU NHANH ĐỂ CHẤM ĐIỂM / GÓP Ý NHÓM BẠN

| Tiêu chí | Mức độ Yếu (Cần chất vấn) | Mức độ Xuất sắc (Chuẩn Production) |
| :--- | :--- | :--- |
| **Idempotency** | Chạy lại $N$ lần sinh ra $N \times 20$ vectors trong DB | Chạy bao nhiêu lần vẫn đúng $20$ vectors duy nhất |
| **Xử lý XML** | Để nguyên thẻ `<jats:p>`, `&amp;` trong vector text | Bóc tách sạch XML & decode HTML entities |
| **Observability Gate** | Script kiểm tra chạy rời rạc sau khi xong RAG | Chốt chặn GX 1.x & SLA chạy trước bước nhúng vector |
| **Silent Failure** | Bị crash exception hoặc không đo lường được số liệu | Chứng minh Hit Rate giảm sâu trong khi Exit Code vẫn 0 |
| **Self-Healing** | Sửa file thủ công bằng tay | Tự động hóa bằng mã lệnh thông qua snapshot sạch |
