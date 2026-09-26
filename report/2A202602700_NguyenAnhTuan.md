# Member Role Report — Day 10: Data Pipeline & Data Observability

## 1. Thông tin cá nhân

| Thông tin         | Nội dung                  |
| ------------------ | -------------------------- |
| Họ và tên       | Nguyễn Anh Tuấn             |
| MSSV               | 2A202602700                     |
| Khóa/Lớp         | K4 - Lớp B              |
| Tên nhóm         | Enigma     |
| Vai trò chính    | Trưởng nhóm & Pipeline Integrator (`core/`, `phase1.py`, `corruption_flow.py`) |
| Repository         | https://github.com/hungdq1306/K4-L3B-Day10-Enigma-Data-Pipeline-Data-Observability |
| Ngày hoàn thành | 2026-09-26               |

## 2. Vai trò và phạm vi công việc

### Phần việc sở hữu

| Module/deliverable | File/hàm phụ trách | Input nhận vào | Output bàn giao  | Trạng thái                                 |
| ------------------ | --------------------- | ---------------- | ----------------- | -------------------------------------------- |
| Corruption & Repair Flow (owner chính) | `src/pipelines/corruption_flow.py`: `main`, `_load_baseline`, `_evaluate`, `_observe`, `_print_comparison` | `papers_clean.json`, `baseline_metrics.json`, `test_set.json`, `crossref_records.json`; hàm `corrupt_clean_dataframe` (Linh), quality/reporting (Linh), index (Thành) | `corrupted_metrics.json`, `repaired_metrics.json`, `corruption_report.md`, bảng 3 trạng thái trên console | Hoàn thành |
| Cải tiến `src/core/` (hạ tầng dùng chung) | `config.py`: `env_int`, `env_bool`, 3 path mới trong `Paths`; `utils.py`: `_atomic_write_text`, `configure_utf8_stdio` | Biến môi trường `.env` | Settings có validate, ghi artifact atomic, console UTF-8 | Hoàn thành |
| Baseline Orchestration (đồng phát triển) | `src/pipelines/phase1.py` | Raw records, cleaned data, Chroma index, test set | `baseline_metrics.json`, `phase1_report.md` | Hoàn thành: tôi viết bản đầu (`ea4a1c7`), Đặng Quang Hưng mở rộng và hợp nhất bản cuối |
| Tích hợp nhánh & vệ sinh repo | Merge `feat/corruption_flow` → `main`, `.gitignore` | Các nhánh của thành viên | Repo không còn conflict marker, không track Chroma DB/`.env`/cache | Hoàn thành |

> Ghi chú về phạm vi: `src/core/` ban đầu là mã starter của đề (multi-provider LLM, `Paths`, `load_settings` đã có sẵn). Tôi chỉ nhận phần cải tiến liệt kê ở trên. Trong `phase1.py`, khoảng 2/3 số dòng hiện tại do Đặng Quang Hưng viết (theo `git blame`).

### Việc hỗ trợ ngoài phạm vi chính

| Hoạt động                         | Thành viên/module được hỗ trợ | Kết quả                    |
| ------------------------------------ | ------------------------------------ | ---------------------------- |
| Sửa lỗi merge `papers_embeddings.json` bị commit kèm conflict marker | Thành (`retrieval/index.py` artifact) | Khôi phục bản JSON hợp lệ (24 documents, thứ tự khớp `papers_clean.json`) |
| Sửa `run_phase1.py` crash `UnicodeEncodeError` trên Windows | Hưng (`phase1.py` in tiếng Việt) | Sửa tại `core` (`configure_utf8_stdio`), không phải sửa file của thành viên khác; `run_phase1.py` exit 0 |
| Đối chiếu số liệu báo cáo nhóm với artifact | Toàn nhóm (`report/group_report.md`) | Sửa các số sai (Corrupted 40% / F1 0.5720 / judge 60% → 50% / 0.8506 / 90%) |

## 3. Kết quả theo vai trò

| Nhiệm vụ đã thực hiện | File/hàm/artifact liên quan | Kết quả bàn giao       | Cách xác minh         |
| --------------------------- | ----------------------------- | ------------------------- | ----------------------- |
| Orchestrate corruption → evaluate → repair → compare | `src/pipelines/corruption_flow.py` | 3 bộ metrics + 3 collection Chroma (`papers-baseline/corrupted/repaired`) | `python script/run_corruption_flow.py` → exit 0 |
| Bảng đối chiếu 3 trạng thái trên console, trạng thái lấy từ kết quả thật | `_print_comparison` | Bảng hit rate / F1 / judge / GX / freshness | Output console khớp `data/quality/*_quality_report.json` |
| Ghi artifact atomic | `src/core/utils.py`: `write_json`, `write_text` | Không còn JSON ghi dở khi pipeline bị ngắt | Ghi đè thành công, lỗi serialize giữ nguyên file cũ, không còn file `.tmp` |
| Validate cấu hình từ env | `src/core/config.py`: `env_int`, `env_bool` | `TOP_K`, `MAX_RESULTS`, `FRESHNESS_THRESHOLD_DAYS` override được; mặc định 4 / 24 / 180 không đổi | `TOP_K=0` → `RuntimeError: TOP_K must be >= 1, got 0.` |

## 4. Giải thích phần kỹ thuật đã thực hiện

### Vấn đề cần giải quyết
Chứng minh bằng số liệu rằng dữ liệu bẩn làm RAG suy giảm âm thầm (silent failure), rằng tầng observability phát hiện được lỗi đó, và rằng repair từ raw snapshot đưa hệ thống về đúng baseline. Muốn so sánh có ý nghĩa, ba trạng thái phải được đo trên cùng test set, cùng cấu hình và mỗi trạng thái có collection Chroma riêng.

### Cách triển khai
- `corruption_flow.main` chạy 8 bước: load baseline, corrupt, lưu, rebuild index và evaluate, quality/freshness, repair từ `crossref_records.json`, evaluate bản repaired, rồi report và in bảng console.
- **Fail fast:** `_load_baseline` kiểm tra đủ 4 artifact của phase 1 trước khi chạy, thiếu thì báo cần chạy `run_phase1.py` trước.
- **Không lặp code:** `_evaluate` và `_observe` dùng chung cho corrupted và repaired. Mỗi dataset build collection riêng qua `embeddings_output_path`, nên không ghi đè `papers-baseline`.
- **Quality fail không làm dừng flow:** kết quả chỉ được ghi vào report. Đây chính là tình huống silent failure cần đo.
- **Repair = chạy lại cleaning trên raw snapshot bất biến:** chạy bao nhiêu lần cũng ra cùng 24 dòng (idempotent).

### Input, output và contract

| Thành phần | Mô tả |
| --- | --- |
| Input | `papers_clean.json`, `baseline_metrics.json`, `test_set.json`, `crossref_records.json`; mọi path lấy từ `Settings.paths` |
| Output | `corrupted_*` / `repaired_*` (clean CSV/JSON, embeddings, metrics, answers), quality và freshness reports, `corruption_log.json`, `corruption_report.md` |
| Module phụ thuộc | `ingestion.corruption`, `ingestion.cleaning`, `retrieval.index`, `evaluation.metrics`, `observability.quality`, `observability.reporting` |
| Module sử dụng output | `report/group_report.md`, live demo |
| Điều kiện lỗi cần xử lý | Chưa chạy phase 1 (thiếu artifact); repaired dataframe rỗng; quality gate fail (ghi nhận, không dừng); console không phải UTF-8 trên Windows |

### Cách xác minh
```bash
python script/run_phase1.py
python script/run_corruption_flow.py
```

- **Kết quả mong đợi:** Corrupted giảm rõ so với baseline, GX FAILED và STALE; Repaired trở về bằng baseline.
- **Kết quả thực tế (2026-09-26, cả hai lệnh exit 0):**
  - Baseline: hit rate 100.0%, F1 1.0000, judge 100%, GX PASSED, FRESH (stale 1/24).
  - Corrupted: hit rate 50.0%, F1 0.8506, judge 90%, GX FAILED, STALE (8/21 = 38.1%).
  - Repaired: hit rate 100.0%, F1 1.0000, judge 100%, GX PASSED, FRESH (stale 1/24).
- **Artifact/log:** `data/results/*_metrics.json`, `data/quality/*_quality_report.json`, `data/reports/corruption_report.md`.

## 5. Một quyết định kỹ thuật quan trọng

- **Bối cảnh:** bản `corruption_flow.py` trên `main` (trước khi merge) in bảng 3 trạng thái với các cột Quality Gate và Freshness **hardcode sẵn** chữ `PASSED/FAILED/FRESH/STALE`.
- **Các phương án đã cân nhắc:**
  1. Giữ nguyên bảng hardcode để demo luôn "đẹp".
  2. Bỏ bảng console, chỉ dựa vào `corruption_report.md`.
  3. Giữ bảng nhưng lấy trạng thái từ kết quả thật: quality/freshness của corrupted và repaired, còn baseline đọc từ report của phase 1.
- **Phương án đã chọn:** phương án 3 (`_print_comparison`).
- **Lý do:** bảng hardcode vẫn in PASSED ngay cả khi gate thực sự fail. Điều này che đi đúng loại lỗi mà bài lab muốn phát hiện, và vi phạm quy định "không bịa số liệu". CP5 lại yêu cầu có bảng trên console, nên không bỏ được.
- **Bằng chứng:** bảng in ra khớp `corrupted_quality_report.json` (`success: false`) và `repaired_quality_report.json` (`success: true`).

## 6. Một lỗi hoặc blocker đã xử lý

- **Triệu chứng/lỗi nguyên văn:** `UnicodeEncodeError: 'charmap' codec can't encode character 'Ắ' in position 1: character maps to <undefined>` tại `print("BẮT ĐẦU CHẠY PHASE 1...")` trong `phase1.py`.
- **Lệnh hoặc bước tái hiện:** `python script/run_phase1.py > phase1.log` trên Windows (stdout bị redirect nên dùng cp1252).
- **Nguyên nhân gốc:** trên Windows, Python mã hóa stdout theo code page hệ thống (cp1252) khi output không đi ra terminal UTF-8. Ký tự tiếng Việt có dấu không nằm trong cp1252. Lỗi có từ trước thay đổi của tôi: đã xác minh bằng cách `git stash` `src/core` về bản gốc và vẫn thấy lỗi.
- **Cách xử lý:** thêm `configure_utf8_stdio()` trong `core/utils.py` và gọi trong `core/__init__.py`. Mọi entrypoint đều import `core`, nên chỉ cần sửa một chỗ, không phải sửa file của thành viên khác.
- **Cách xác minh sau khi sửa:** chạy cùng lệnh redirect, cả `run_phase1.py` và `run_corruption_flow.py` đều exit 0, log hiển thị đúng tiếng Việt.
- **Điều học được:** lỗi môi trường (encoding, đường dẫn tuyệt đối trong artifact) có thể làm hỏng demo dù logic đúng. Cần chạy thử pipeline ở chế độ redirect log, không chỉ chạy trong terminal của IDE.

## 7. Phân tích kết quả

- **Chuỗi 1:** `drop_latest_records` xóa 5 bài → Quality Gate FAILED (duplicate `paper_id` và summary rỗng), Freshness STALE (38.1%) → hit rate giảm từ 100% xuống 50%. 5 câu trượt (`test-01/02/03/04/06`) đúng là 5 câu có ground-truth thuộc 5 bài bị xóa.
- **Chuỗi 2:** repair từ raw snapshot → GX PASSED, FRESH (1/24) → hit rate, F1 và judge trở về đúng baseline.
- **Kết quả khác kỳ vọng:**
  - **F1 và judge giảm ít hơn hit rate nhiều.** `test-04` và `test-06` (hỏi tác giả) trượt retrieval nhưng vẫn đạt F1 = 1.0, vì bài được retrieve đầu tiên là một bài khác có cùng danh sách tác giả. Judge cũng chỉ là heuristic dựa trên F1, vì cả 30 câu đều dùng fallback do không có LLM key. Kết luận: hit rate là metric đáng tin nhất trong lần chạy này.
  - **Blank summary, chèn nhiễu và cắt tiêu đề không làm trượt câu retrieval nào,** nhưng được Quality Gate bắt. Observability phát hiện được những lỗi mà metrics RAG bỏ sót.

## 8. Điều học được và hướng cải thiện

1. **Pipeline:** tách path và cấu hình về một nơi (`Settings.paths`) giúp 4 người làm song song mà không lệch đường dẫn artifact.
2. **Observability:** gate phải phản ánh kết quả thật. Hiện vẫn còn lỗ hổng: tiêu đề bị cắt còn 5 ký tự vẫn qua kiểm tra `title` not null.
3. **RAG:** một metric tốt vẫn có thể che lỗi retrieval. Cần đọc cả số liệu từng câu, không chỉ nhìn giá trị trung bình.
4. **Nếu có thêm thời gian:**
   - Thêm expectation `ExpectColumnValueLengthsToBeBetween(title, min_value=15)` và đo xem gate có bắt được `truncate_title` không.
   - Bổ sung LLM judge thật để so sánh với heuristic.
   - Bỏ `persist_path` tuyệt đối khỏi `papers_embeddings*.json`, vì đường dẫn riêng của từng máy gây conflict mỗi lần merge.

## 9. Cam kết của thành viên
- [x] Nội dung báo cáo phản ánh đúng phần việc và mức hiểu của tôi.
- [x] Tôi có thể giải thích luồng end-to-end, không chỉ module mình phụ trách.
- [x] Mọi kết luận về kết quả đều có artifact hoặc metric để đối chiếu.
- [x] Tôi không ghi “đã chạy thành công” cho phần chưa được kiểm chứng.
- [x] Báo cáo không chứa `.env`, API key, token hoặc secret.
- [x] Báo cáo này không phải bản sao nguyên văn của báo cáo nhóm hoặc báo cáo thành viên khác.

**Họ và tên:** Nguyễn Anh Tuấn  
**Ngày xác nhận:** 2026-09-26
