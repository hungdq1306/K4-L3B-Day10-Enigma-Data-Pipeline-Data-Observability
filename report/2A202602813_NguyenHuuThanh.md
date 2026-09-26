# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Họ và tên       | Nguyễn Hữu Thành             |
| MSSV               | 2A202602813                     |
| Khóa/Lớp         | K4 - Lớp B              |
| Tên nhóm         | Enigma     |
| Vai trò chính    | RAG, Vector Database & Evaluation Benchmark (`retrieval/`, `testset.py`) |
| Repository         | https://github.com/hungdq1306/K4-L3B-Day10-Enigma-Data-Pipeline-Data-Observability |
| Ngày hoàn thành | 2026-09-26               |

---

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao | Trạng thái |
| ------------------ | --------------------- | ---------------- | ----------------- | -------------------------------------------- |
| Benchmark Test Set | `src/evaluation/testset.py`<br>• `build_test_set` | Cleaned DataFrame 24 dòng | `data/eval/test_set.json` (10 câu test chuẩn phủ 4 nhóm nghiệp vụ) | Hoàn thành |
| Vector Indexing & Embeddings | `src/retrieval/index.py`<br>• `LocalEmbeddingIndex.build`<br>• `LocalEmbeddingIndex.search`<br>• `LocalEmbeddingIndex.lookup`<br>`src/retrieval/embeddings.py` | Cleaned DataFrame, mô hình `all-MiniLM-L6-v2` | 3 ChromaDB collections độc lập (`papers-baseline`, `papers-corrupted`, `papers-repaired`) | Hoàn thành |
| Multi-Provider RAG QA & Agent | `src/retrieval/qa.py`<br>• `answer_question`<br>• `_extract_answer`<br>`src/retrieval/llm.py`<br>`src/retrieval/agent.py` | Câu hỏi người dùng, Vector Index, Search Results | Trích xuất câu trả lời chuẩn xác theo metadata, hỗ trợ LLM Provider chuyển đổi linh hoạt | Hoàn thành |
| Pipeline Evaluation & Metrics | `src/evaluation/metrics.py`<br>• `evaluate_pipeline`<br>• `_token_f1`<br>• `_judge_answer` | Vector Index, Test Set, Settings | `baseline_metrics.json`, `corrupted_metrics.json`, `repaired_metrics.json` | Hoàn thành |

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động | Thành viên/module được hỗ trợ | Kết quả |
| ------------------------------------ | ------------------------------------ | ---------------------------- |
| Thống nhất Document Contract cho ChromaDB | Đặng Quang Hưng (`cleaning.py`) | Thống nhất chuẩn trường `text_for_embedding` gồm 5 phần và metadata dict đầy đủ (`paper_id`, `title`, `published`, `authors_joined`, `categories_joined`, `summary`, `abs_url`, `pdf_url`) |
| Cung cấp dữ liệu chỉ số đánh giá cho báo cáo | Hà Thị Mỹ Linh (`reporting.py`) | Chuyển giao các chỉ số định lượng `retrieval_hit_rate`, `mean_token_f1`, `judge_accuracy`, `mean_judge_score` để điền vào bảng đối chiếu 3 trạng thái |
| Tích hợp luồng Index và Eval vào Pipeline | Nguyễn Anh Tuấn (`phase1.py`, `corruption_flow.py`) | Hoàn thiện lời gọi `LocalEmbeddingIndex.build` và `evaluate_pipeline` trên cả 3 pha chạy tự động |

---

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao | Cách xác minh |
| --------------------------- | ----------------------------- | ------------------------- | ----------------------- |
| Sinh 10 câu hỏi benchmark chuẩn | `src/evaluation/testset.py` | `data/eval/test_set.json` | Kiểm tra độ dài `len(ts) == 10` phủ đủ 4 nhóm: `summary`, `authors`, `date`, `categories` |
| Quản lý ChromaDB Vector Store | `src/retrieval/index.py` | 3 collection trong `data/chroma/`: `papers-baseline`, `papers-corrupted`, `papers-repaired` | Query vector cosine tương đồng và manifest JSON trong `data/embeddings/` |
| Đo lường hiệu năng RAG 3 trạng thái | `src/evaluation/metrics.py` | `data/results/*_metrics.json` | Hit Rate: 100.0% (Baseline) → 40.0% (Corrupted) → 100.0% (Repaired) |

**Output cụ thể tạo ra:**
- File `data/eval/test_set.json` chứa 10 câu hỏi kiểm thử có đầy đủ `question`, `ground_truth`, `ground_truth_doc_ids` làm thước đo chuẩn mực khách quan xuyên suốt bài lab.
- Thư mục `data/chroma/` chứa cơ sở dữ liệu vector đa phân vùng, lưu trữ embeddings 384 chiều của toàn bộ các tài liệu sạch, tài liệu lỗi và tài liệu sau khôi phục.

---

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết
1. **Thiếu thước đo định lượng khách quan (Ground Truth Benchmark):** Để đánh giá một hệ thống RAG có bị suy thoái khi dữ liệu bẩn hay không, cần một bộ "đề thi chuẩn" với các câu hỏi đại diện cho nhiều khía cạnh nghiệp vụ khác nhau thay vì chỉ hỏi vài câu ngẫu nhiên.
2. **Nguy cơ rò rỉ vector (Ghost Vectors):** Nếu dùng chung một collection vector store cho cả dữ liệu sạch, dữ liệu bẩn và dữ liệu sửa chữa, các vector cũ bị xóa mềm (soft delete) có thể vẫn còn nằm trong index HNSW, dẫn đến sai lệch kết quả đánh giá (bị lẫn dữ liệu sạch vào pha lỗi hoặc ngược lại).
3. **Trích xuất câu trả lời chính xác từ ngữ cảnh:** Cần cơ chế Question-Answering thông minh có khả năng nhận diện ý định câu hỏi để bóc tách chính xác tác giả, ngày xuất bản hay tóm tắt nội dung từ metadata và ngữ cảnh được tìm thấy.

### Cách triển khai
1. **Module `testset.py` (`build_test_set`)**:
   - Chọn lọc các bài báo tiêu biểu trong DataFrame sạch.
   - Sinh 10 câu hỏi chia đều vào 4 nhóm nghiệp vụ:
     - `summary`: 4 câu hỏi yêu cầu tóm tắt ý chính của bài báo.
     - `authors`: 2 câu hỏi hỏi về danh sách tác giả nghiên cứu.
     - `date`: 2 câu hỏi hỏi về ngày xuất bản chính xác (`YYYY-MM-DD`).
     - `categories`: 2 câu hỏi hỏi về lĩnh vực chuyên môn.
   - Quy ước cấu trúc câu hỏi có chứa tiêu đề trong nháy đơn `'<Title>'` để hỗ trợ cơ chế đối sánh chính xác kết hợp tìm kiếm ngữ nghĩa, và gán `ground_truth_doc_ids` bằng DOI của bài báo tương ứng.
2. **Module `index.py` (`LocalEmbeddingIndex`)**:
   - Hàm `_build_documents`: Chuyển đổi từng dòng DataFrame thành đối tượng Document có `content = row["text_for_embedding"]` và `metadata` chứa toàn bộ các thuộc tính gốc.
   - Hàm `build`: Khởi tạo `chromadb.PersistentClient`, cấu hình không gian vector cosine (`{"hnsw": {"space": "cosine"}}`), tự động derive tên collection dựa vào đường dẫn đầu ra (`papers-baseline`, `papers-corrupted`, `papers-repaired`).
   - Sử dụng mô hình `all-MiniLM-L6-v2` thông qua wrapper `MiniLMEmbeddings` (hỗ trợ cả SentenceTransformer trực tiếp và fallback dense embedding đảm bảo không bao giờ crash).
   - Hàm `search`: Nhúng câu hỏi query thành vector, gọi `collection.query` lấy top-k kết quả có khoảng cách cosine nhỏ nhất, tính score tương đồng `max(0.0, 1.0 - distance)`.
3. **Module `qa.py` (`answer_question`, `_extract_answer`)**:
   - Nhận diện câu hỏi qua regex để tìm tiêu đề bài báo, kết hợp giữa `lookup` chính xác và `search` ngữ nghĩa.
   - Hàm `_extract_answer`: Dựa vào từ khóa trong câu hỏi (`who authored`, `when was`, `what categories`) để trích xuất trường metadata tương ứng (`authors_joined`, `published`, `categories_joined`, hoặc câu đầu tiên của `summary`).

### Input, output và contract

| Thành phần | Mô tả |
| ------------------------------ | ------------------------------------------- |
| Input | `pd.DataFrame` sạch (hoặc lỗi/sửa), cấu hình `Settings`, `top_k=4` |
| Output | `data/eval/test_set.json`, ChromaDB collections, `SearchResult`, `AnswerResult` |
| Module phụ thuộc | `ingestion.cleaning` (cung cấp DataFrame), `retrieval.embeddings` (sinh vector) |
| Module sử dụng output | `evaluation.metrics` (đo lường kết quả), `pipelines.phase1`, `pipelines.corruption_flow` |
| Điều kiện lỗi cần xử lý | Tải mô hình embedding offline, xử lý khi không tìm thấy tài liệu liên quan (`I don't know from the indexed corpus`), rỗng metadata |

### Cách xác minh

```bash
# 1. Kiểm tra sinh bộ test set 10 câu
.venv\Scripts\python.exe -c "from core.config import load_settings; from evaluation.testset import build_test_set; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); ts=build_test_set(df, s.paths.eval_testset); print(f'Tín hiệu hoàn thành: Sinh được {len(ts)} câu hỏi test')"

# 2. Kiểm tra Indexing ChromaDB
.venv\Scripts\python.exe -c "from core.config import load_settings; from retrieval.index import LocalEmbeddingIndex; import pandas as pd; s=load_settings(); df=pd.read_json(s.paths.clean_json); idx=LocalEmbeddingIndex.build(df, s); res=idx.search('agentic RAG', top_k=2); print(f'Index thanh cong, tim thay {len(res)} ket qua, top 1: {res[0].title[:30]}...')"
```

- **Kết quả mong đợi:** In ra `Sinh được 10 câu hỏi test`, ChromaDB index thành công 24 tài liệu và truy vấn trả về kết quả chính xác.
- **Kết quả thực tế:**
  ```text
  Tín hiệu hoàn thành: Sinh được 10 câu hỏi test
  Index thanh cong, tim thay 2 ket qua, top 1: Agentic Retrieval-Augmented Ge...
  ```
- **Artifact:** `data/eval/test_set.json`, `data/chroma/chroma.sqlite3`, `data/embeddings/papers_embeddings.json`.

---

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** Lựa chọn kiến trúc lưu trữ Vector Database trên ChromaDB khi chạy quy trình đối chiếu 3 trạng thái (Baseline, Corrupted, Repaired).
- **Các phương án đã cân nhắc:**
  - *Phương án 1 (Single Collection - In-place Update):* Dùng duy nhất một collection `papers` và thực hiện xóa/chèn đè dữ liệu ở mỗi pha.
  - *Phương án 2 (Multi-Collection Isolation):* Tạo 3 collection hoàn toàn biệt lập theo tên: `papers-baseline`, `papers-corrupted`, `papers-repaired`.
- **Phương án đã chọn:** **Phương án 2 (Multi-Collection Isolation)**.
- **Lý do:**
  1. *Ngăn ngừa Ghost Vectors:* Các vector database sử dụng chỉ mục HNSW thường chỉ đánh dấu xóa logic (tombstone) khi xóa vector. Khi thực hiện update dồn dập giữa các pha, các vector cũ có thể vẫn xuất hiện trong kết quả truy vấn xấp xỉ (Approximate Nearest Neighbors), làm ô nhiễm kết quả kiểm thử.
  2. *Khả năng tái hiện (Reproducibility) và Audit:* Việc lưu trữ đồng thời cả 3 collection vật lý trong thư mục `data/chroma/` cho phép nhóm có thể inspect và query độc lập bất kỳ lúc nào để phục vụ chấm điểm và Live Demo trước Giảng viên mà không cần chạy lại toàn bộ pipeline.
- **Bằng chứng:** Trong `src/retrieval/index.py`, hàm `_derive_collection_name` ánh xạ chính xác từng file manifest sang collection name riêng biệt. Báo cáo đối chiếu phản ánh sự sụt giảm và phục hồi rõ ràng giữa các phân vùng độc lập.

---

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:**
  ```text
  KeyError: 'authors_joined'
  ChromaError: Collection 'papers-baseline' does not exist
  ```
- **Lệnh hoặc bước tái hiện:** Chạy thử nghiệm index dữ liệu ban đầu khi chưa tích hợp xong phần cleaning.
- **Nguyên nhân gốc:**
  1. Hàm `_build_documents` trong `LocalEmbeddingIndex` yêu cầu các trường metadata bắt buộc: `authors_joined`, `categories_joined`, `summary`, `published`. Khi DataFrame đầu vào từ `cleaning.py` chưa tạo các cột helper này thì hàm bị văng `KeyError`.
  2. Khi gọi `LocalEmbeddingIndex.load` trước khi gọi `LocalEmbeddingIndex.build`, ChromaDB client chưa tạo collection dẫn đến lỗi `Collection does not exist`.
- **Cách xử lý:**
  1. Thiết lập Document Contract chặt chẽ với bạn Đặng Quang Hưng (`cleaning.py`), đảm bảo hàm `build_clean_dataframe` luôn bổ sung đầy đủ các cột helper string trước khi xuất xưởng.
  2. Trong `LocalEmbeddingIndex.build`, bổ sung cơ chế xóa collection cũ nếu tồn tại (`client.delete_collection`) rồi mới tạo mới (`client.create_collection`), đảm bảo index luôn sạch sẽ và có tính Idempotent.
- **Cách xác minh sau khi sửa:** Chạy lại `python script/run_phase1.py` trơn tru từ đầu đến cuối với mã thoát 0.
- **Điều học được:** Tầng Vector Store và Data Pipeline luôn cần một Data Contract rõ ràng về Schema Metadata. Không bao giờ được giả định metadata sẽ luôn có sẵn mà phải có kiểm soát chặt chẽ ở khâu tiền xử lý.

---

## 7. Hiểu biết về luồng end-to-end

1. **Dữ liệu đi từ Crossref đến vector index như thế nào?**
   - Dữ liệu thô từ Crossref API được tải về dưới dạng JSON thô và lưu tại `data/raw/crossref_response.json` để bảo tồn dòng dõi (Data Lineage).
   - Module `cleaning.py` bóc tách các thẻ JATS XML, chuẩn hóa khoảng trắng, tính toán trường `age_days` và ghép nối thành chuỗi văn bản ngữ cảnh `text_for_embedding` gồm 5 phần.
   - Module `index.py` nhận DataFrame sạch, chuyển thành Document và đưa qua mô hình `all-MiniLM-L6-v2` để biến đổi các đoạn văn bản thành vector nhúng 384 chiều, sau đó lưu trữ bền vững vào ChromaDB.

2. **Evaluation set và ground-truth document IDs dùng để đo retrieval/answer quality ra sao?**
   - Bộ `test_set.json` gồm 10 câu hỏi, mỗi câu gắn với `ground_truth` (câu trả lời chuẩn mực) và `ground_truth_doc_ids` (danh sách DOI của bài báo chứa thông tin).
   - Khi đánh giá Retrieval: Hệ thống tìm kiếm top-k tài liệu liên quan từ ChromaDB. Nếu DOI của tài liệu truy xuất chứa `ground_truth_doc_ids`, hệ thống ghi nhận một "Hit" (đo lường bằng `retrieval_hit_rate`).
   - Khi đánh giá Answer Quality: Câu trả lời do Agent trích xuất được so sánh với `ground_truth` để tính chỉ số `token_f1` (độ chính xác từng token) và chấm điểm định tính qua `judge_accuracy`.

3. **Quality checks khác freshness monitoring ở điểm nào trong bài lab?**
   - **Quality checks (Great Expectations 1.x):** Giám sát tính toàn vẹn về cấu trúc, cú pháp và quy tắc nghiệp vụ của dữ liệu (Schema Integrity) như: số lượng bản ghi nằm trong khoảng 5-5000, ID không được trùng lặp, các cột chính không được null, tóm tắt phải đủ dài $\ge$ 30 ký tự.
   - **Freshness monitoring (Freshness SLA):** Giám sát tính cập nhật về mặt thời gian (Temporal Validity). Dữ liệu có thể hoàn toàn sạch sẽ, không có bất kỳ giá trị null nào (Quality Check PASSED), nhưng nếu toàn bộ bài báo đều được xuất bản từ nhiều năm trước (`age_days > 180` chiếm > 25%), hệ thống sẽ báo động STALE.

4. **Vì sao phải dùng cùng test set cho baseline, corrupted và repaired?**
   - Đây là nguyên tắc cơ bản của một thực nghiệm khoa học có đối chứng (Controlled Experiment).
   - Nếu mỗi trạng thái dùng một bộ câu hỏi khác nhau, sự thay đổi của các chỉ số hiệu năng (Hit Rate, F1) sẽ bị nhiễu do độ khó của câu hỏi chứ không phản ánh đúng tác động của dữ liệu.
   - Dùng chung một bộ test set 10 câu cố định giúp cô lập biến số duy nhất là **chất lượng của dữ liệu** (Sạch vs Lỗi vs Đã sửa).

5. **Repair được xem là thành công dựa trên artifact và metric nào?**
   - **Về Artifact:** Có đầy đủ `papers_clean_repaired.csv/json`, collection ChromaDB `papers-repaired`, file `repaired_metrics.json` và bảng đối chiếu trong `corruption_report.md`.
   - **Về Metric:** Trạng thái Quality Gate phục hồi từ `FAILED` về `PASSED`, Freshness phục hồi về `FRESH`, chỉ số `retrieval_hit_rate` phục hồi từ 40.0% lên 100.0%, và `mean_token_f1` phục hồi từ 0.5720 lên 1.0000.

---

## 8. Phân tích kết quả

### Metrics chính

| Metric / Signal | Baseline | Corrupted | Repaired | Nhận xét của cá nhân |
| ---------------------- | -------: | --------: | -------: | ---------------------------------------------------- |
| `retrieval_hit_rate` | 100.00% | 40.00% | 100.00% | Rơi rớt dữ liệu mới làm tụt 60% Hit Rate; sau repair phục hồi tuyệt đối |
| `mean_token_f1` | 1.0000 | 0.5720 | 1.0000 | Nhiễu text và rỗng summary khiến F1 giảm mạnh; phục hồi 100% sau repair |
| `judge_accuracy` | 100.00% | 60.00% | 100.00% | LLM Judge phát hiện câu trả lời bị thoái hóa trên dữ liệu bẩn |
| `mean_judge_score` | 5.00 / 5.0 | 3.20 / 5.0 | 5.00 / 5.0 | Điểm trung bình chất lượng câu trả lời lấy lại mức tối đa sau phục hồi |
| `Quality Gate (GX 1.x)` | **PASSED** | **FAILED** | **PASSED** | GX bắt chính xác 2 lỗi: Uniqueness (trùng lặp) và String Length (rỗng summary) |
| `Freshness SLA` | **FRESH** | **STALE** | **FRESH** | Tỷ lệ bài quá hạn vọt lên 38.1% ở pha lỗi, sau đó trở lại 4.2% an toàn |

### Kết luận từ số liệu

1. **Chuỗi 1: [Data Corruption] → [Quality/Freshness Signal] → [Agent Metric Degradation]:**
   - Kịch bản tiêm lỗi `drop_latest_records` làm mất các bài báo mới nhất, kết hợp `blank_summary` và `inject_noise` làm suy thoái nội dung.
   - Hệ thống Observability lập tức đổi trạng thái sang `FAILED` và `STALE`.
   - Hệ quả trực tiếp: `retrieval_hit_rate` sụt giảm nghiêm trọng từ **100% xuống 40%**, và `mean_token_f1` rớt từ **1.0 xuống 0.5720**. Điều này chứng minh hiện tượng Silent Failure đã diễn ra rõ rệt nếu không có chốt chặn kiểm dịch.

2. **Chuỗi 2: [Idempotent Repair] → [Quality Signal Recovery] → [Agent Metric Full Recovery]:**
   - Kích hoạt quy trình phục hồi từ nguồn snapshot nguyên bản `data/raw/crossref_records.json`, chạy lại cleaning và tái lập vector collection `papers-repaired`.
   - Great Expectations và Freshness SLA lập tức quay về trạng thái `PASSED` và `FRESH`.
   - Hiệu năng Agent phục hồi trọn vẹn 100%: Hit Rate trở lại **100%** và F1 trở lại **1.0000**.

### Corruption nào ảnh hưởng rõ nhất và vì sao?
- Kịch bản **`drop_latest_records` (bỏ rơi 20% bài báo mới nhất)** gây ảnh hưởng thảm khốc nhất đến Retrieval Hit Rate (kéo tụt từ 100% xuống 40%). Lý do là các câu hỏi đánh giá tập trung vào các công trình mới nhất, khi tài liệu mục tiêu bị biến mất khỏi vector store thì mô hình dense retrieval hoàn toàn không thể tìm thấy context đúng (Retrieval Miss).
- Kịch bản **`blank_summary`** và **`inject_noise`** ảnh hưởng trực tiếp nhất đến Token F1, khiến câu trả lời của AI bị cụt ngủn hoặc ngập tràn ký tự vô nghĩa.

### Kết quả nào khác với kỳ vọng ban đầu?
- Ban đầu tôi giả định rằng khi dữ liệu bị lỗi, điểm số Hit Rate sẽ chỉ giảm nhẹ khoảng 10-20% do mô hình embedding có khả năng tìm kiếm tương đồng mờ (fuzzy semantic match). Tuy nhiên, kết quả thực tế cho thấy Hit Rate giảm tới **60%** (từ 100% xuống 40%). 
- *Giải thích:* Trong các hệ thống RAG phục vụ tài liệu học thuật đòi hỏi độ chính xác cao, khi tài liệu gốc bị drop hoặc tiêu đề bị cắt cụt dưới 8 ký tự, embedding của chunk bị dịch chuyển rất xa trong không gian vector đa chiều, dẫn đến việc top-k kết quả bị trật hoàn toàn khỏi vùng mục tiêu.

---

## 9. Điều học được và hướng cải thiện

### Ba điều quan trọng nhất
1. **Dữ liệu là trần chất lượng của AI (Data is the ceiling of RAG):** Dù mô hình ngôn ngữ lớn (LLM) hay thuật toán sinh có hiện đại đến đâu, nếu dữ liệu vector đầu vào bị suy thoái thì câu trả lời chắc chắn sẽ bị ảo giác hoặc sai lệch (Garbage In, Garbage Out).
2. **Giá trị sống còn của Data Observability & Quality Gates:** Không bao giờ để dữ liệu đi thẳng vào Vector Database mà không qua chốt kiểm dịch tự động. Great Expectations 1.x đóng vai trò như một hệ thống "cách ly kiểm dịch" ngăn chặn hiện tượng Silent Failure trước khi gây hại cho người dùng cuối.
3. **Thiết kế Idempotent trong Data Engineering:** Việc lưu trữ bản snapshot thô (Raw Preservation) và xây dựng các bước biến đổi dữ liệu có tính chất Idempotent (chạy lại nhiều lần vẫn cho kết quả nhất quán) là điều kiện tiên quyết để hệ thống có khả năng tự chữa lành (Self-Healing).

### Nếu có thêm thời gian
Tôi sẽ triển khai giải pháp **Hybrid Search (kết hợp Dense Retrieval MiniLM với Sparse BM25 via Reciprocal Rank Fusion - RRF)** và tích hợp thêm tầng **Re-ranking (Cross-Encoder)**. Giải pháp này giúp hệ thống vừa hiểu được ngữ nghĩa sâu sắc, vừa không bị trượt khi người dùng tìm kiếm theo mã định danh DOI hoặc từ khóa chuyên ngành chính xác, đồng thời tăng khả năng chống chịu nhiễu tốt hơn khi văn bản bị chèn ký tự rác.

---

## 10. Cam kết của thành viên

Đánh dấu sau khi tự kiểm tra:

- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Nguyễn Hữu Thành  
**Ngày xác nhận:** 2026-09-26
