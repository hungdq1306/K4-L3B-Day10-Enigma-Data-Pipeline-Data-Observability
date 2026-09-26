# Group Report — Day 10: Data Pipeline & Data Observability

- **Khóa/Lớp:** K4 - Lớp B
- **Tên nhóm:** Enigma
- **Repository:** https://github.com/hungdq1306/K4-L3B-Day10-Enigma-Data-Pipeline-Data-Observability
- **Ngày hoàn thành:** 2026-09-26

---

## 1. Thành Viên & Phân Công Nhiệm Vụ

| STT | Họ và tên | MSSV | Vai trò chính | Module / Deliverable sở hữu |
| --: | --- | --- | --- | --- |
| 1 | Nguyễn Anh Tuấn | 2A202602700 | Trưởng nhóm & Pipeline Integrator | `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py`, `src/core/` |
| 2 | Đặng Quang Hưng | 2A202602719 | Data Foundation, Cleaning & Repair | `src/ingestion/crossref.py`, `src/ingestion/cleaning.py`, raw & clean datasets |
| 3 | Nguyễn Hữu Thành | 2A202602813 | RAG, Vector Database & Benchmark Evaluation | `src/retrieval/index.py`, `src/retrieval/embeddings.py`, `src/evaluation/testset.py` |
| 4 | Hà Thị Mỹ Linh | 2A202602619 | Data Observability, Corruption Suite & Reporting | `src/observability/quality.py`, `src/ingestion/corruption.py`, `src/observability/reporting.py` |

---

## 2. Tóm Tắt Kết Quả Dự Án
Nhóm Enigma đã hoàn thành 100% các tiêu chí bắt buộc theo chuẩn Rubric của Day 10:
- **Xây dựng Data Pipeline hoàn chỉnh:** Thu thập 24 bản ghi metadata từ Crossref REST API với cơ chế offline fallback đọc snapshot local `crossref_response.json`. Xử lý sạch văn bản (loại bỏ toàn bộ JATS XML tags), tính toán `age_days` và sinh trường `text_for_embedding` theo cấu trúc 5 phần giàu ngữ cảnh.
- **Data Observability:** Cấu hình **Great Expectations 1.x ephemeral mode** với 6 expectations (row count 5–5000; `paper_id`, `title`, `text_for_embedding` not null; `paper_id` unique; `summary` dài tối thiểu 30 ký tự) và kiểm soát **Freshness SLA** (bản ghi stale khi `age_days > 180`; dataset báo STALE khi tỷ lệ stale > 25%). Pha Baseline đạt `success = True` và `is_fresh = True` (stale 1/24 = 4.2%).
- **Mô phỏng Silent Failure:** Tiêm 6 kịch bản lỗi dữ liệu tổng hợp theo `corruption_log.json` (xóa 5 bài mới nhất = 20%, xóa trắng 2 summary, chèn chuỗi rác vào 2 summary, cắt 2 tiêu đề còn 5 ký tự, lùi ngày xuất bản 365 ngày cho 8 bài, nhân bản 2 dòng). Retrieval Hit Rate giảm từ **100.0% xuống 50.0%**, Mean Token F1 từ **1.0000 xuống 0.8506**, Judge Accuracy từ **100% xuống 90%**. Quality Gate chuyển sang **FAILED** và Freshness SLA báo **STALE**.
- **Idempotent Self-Repair:** Chạy lại cleaning pipeline từ raw snapshot nguyên bản (`crossref_records.json`), đưa toàn bộ chỉ số hiệu năng (Hit Rate 100%, F1 1.0, Judge 100%) và chất lượng dữ liệu (Quality PASSED, Freshness FRESH) về đúng mức baseline.

---

## 3. Kiến Trúc & Luồng Dữ Liệu End-to-End

```text
[Crossref API / Raw Snapshot]
       │
       ▼
 [Data Ingestion] ──────────► data/raw/crossref_records.json
       │
       ▼
 [Data Cleaning] ───────────► data/clean/papers_clean.csv / json
       │
       ├─────────────────────────────────┐
       ▼                                 ▼
[GX 1.x Quality Gate]          [MiniLM Vector Indexing]
data/quality/baseline.json     ChromaDB: papers-baseline
       │                                 │
       └────────────────►┌───────────────┘
                         ▼
             [Benchmark Evaluation] ◄─── data/eval/test_set.json (10 questions)
             baseline_metrics.json (Hit Rate: 100%)
                         │
                         ▼
        [Synthetic Data Corruption Suite]
        data/clean/papers_clean_corrupted.csv (Tiêm 6 lỗi)
                         │
                         ▼
         [Silent Failure Measurement]
         corrupted_metrics.json (Hit Rate: 50%, GX: FAILED, SLA: STALE)
                         │
                         ▼
            [Idempotent Self-Repair]
            Tái tạo từ data/raw/crossref_records.json
                         │
                         ▼
         [Repaired State Verification]
         repaired_metrics.json (Hit Rate: 100%, GX: PASSED, SLA: FRESH)
                         │
                         ▼
         [3-State Comparison Report]
         data/reports/corruption_report.md
```

---

## 4. Bảng Đối Chiếu Định Lượng 3 Trạng Thái (Thực Tế Chạy Pipeline)

Số liệu trích trực tiếp từ `data/results/{baseline,corrupted,repaired}_metrics.json`, `*_answers.json`, `data/quality/*_quality_report.json` và `corruption_log.json` (lần chạy `run_phase1.py` + `run_corruption_flow.py` ngày 2026-09-26, cùng test set 10 câu):

| Chỉ số / Tín hiệu Observability | Baseline (Pha 1) | Corrupted (Pha 2) | Repaired (Pha 3) | Đánh giá tác động |
| :--- | :---: | :---: | :---: | :--- |
| **Retrieval Hit Rate** | **100.0%** | **50.0%** | **100.0%** | 5 câu trượt (`test-01/02/03/04/06`) đúng là 5 câu có ground-truth thuộc 5 bài bị xóa ở kịch bản `drop_latest_records` |
| **Mean Token F1** | **1.0000** | **0.8506** | **1.0000** | Chỉ 3/10 câu giảm F1 (`test-01` = 0.000, `test-02` = 0.786, `test-03` = 0.720) |
| **Judge Accuracy** | **100.0%** | **90.0%** | **100.0%** | Chỉ `test-01` bị chấm sai; judge dùng heuristic fallback theo token F1 (không có LLM key) |
| **Mean Judge Score** | **5.0** | **4.2** | **5.0** | Phục hồi về mức baseline sau repair |
| **Data Quality Gate (GX 1.x)** | **PASSED** | **FAILED** | **PASSED** | Fail 2/6 expectations: `paper_id` unique (4 dòng, 19.0%) và độ dài `summary` ≥ 30 (4 dòng, 19.0%) |
| **Freshness SLA Status** | **FRESH** (1/24 = 4.2%) | **STALE** (8/21 = 38.1%) | **FRESH** (1/24 = 4.2%) | Tỷ lệ stale vượt ngưỡng 25% do kịch bản lùi ngày 365 ngày |
| **Số lượng bản ghi** | 24 | 21 | 24 | 24 − 5 (drop) + 2 (duplicate) = 21; repair khôi phục đủ 24 và khử trùng lặp |

**Nhận xét quan trọng:**
- **Hit rate phản ánh lỗi rõ hơn F1 và judge.** Hai câu hỏi về tác giả (`test-04`, `test-06`) trượt retrieval nhưng vẫn đạt F1 = 1.0, vì bài được retrieve đầu tiên là một bài khác có cùng danh sách tác giả. Do đó F1 và judge đang đánh giá chất lượng cao hơn thực tế.
- **Các kịch bản làm hỏng nội dung văn bản gần như không ảnh hưởng đến retrieval.** Blank summary, chèn nhiễu và cắt tiêu đề không làm trượt câu nào: mọi câu trượt đều do bài bị xóa. Ngược lại, 3 kịch bản này lại bị Quality Gate và Freshness SLA bắt được. Điều đó cho thấy tầng observability phát hiện được những lỗi mà metrics RAG không thấy.
- **Có kịch bản lỗi không bị expectation nào bắt.** Tiêu đề bị cắt còn 5 ký tự vẫn qua kiểm tra `title` not null; nhóm chưa có expectation về độ dài tiêu đề.

---

## 5. Danh Sách Deliverables & Artifacts Minh Chứng

1. **Mã nguồn thực thi:**
   - `src/core/config.py`, `src/core/utils.py`
   - `src/ingestion/crossref.py`, `src/ingestion/cleaning.py`, `src/ingestion/corruption.py`
   - `src/observability/quality.py`, `src/observability/reporting.py`
   - `src/retrieval/index.py`, `src/retrieval/embeddings.py`, `src/retrieval/qa.py`, `src/retrieval/agent.py`
   - `src/evaluation/testset.py`, `src/evaluation/metrics.py`
   - `src/pipelines/phase1.py`, `src/pipelines/corruption_flow.py`
2. **Kịch bản chạy One-Click:**
   - `python script/run_phase1.py` (Exit code: 0)
   - `python script/run_corruption_flow.py` (Exit code: 0)
3. **Artifacts dữ liệu:**
   - `data/raw/crossref_records.json`
   - `data/clean/papers_clean.csv`, `data/clean/papers_clean.json`
   - `data/eval/test_set.json` (10 câu test chuẩn 4 nhóm)
   - `data/results/baseline_metrics.json`, `data/results/corrupted_metrics.json`, `data/results/repaired_metrics.json`
   - `data/results/corruption_log.json` (Ghi chi tiết 6 kịch bản lỗi)
   - `data/reports/phase1_report.md`
   - `data/reports/corruption_report.md` (Bảng đối chiếu 3 trạng thái)
4. **Báo cáo nhóm & cá nhân:**
   - `docs/TEAM.md`
   - `report/group_report.md`
   - 4 báo cáo cá nhân: `report/2A202602700_NguyenAnhTuan.md`, `report/2A202602719_DangQuangHung.md`, `report/2A202602813_NguyenHuuThanh.md`, `report/2A202602619_HaThiMyLinh.md`

---

## 6. Bài Học Kinh Nghiệm Của Nhóm
1. **Phòng bệnh hơn chữa bệnh:** Data Quality Gate đặt ngay sau bước Ingestion/Cleaning giúp phát hiện dị thường dữ liệu trước khi vector store bị ô nhiễm.
2. **Nhận thức về Silent Failure:** Trong các hệ thống RAG, lỗi dữ liệu thường không gây crash code mà làm giảm sút âm thầm độ tin cậy của mô hình AI. Việc quan trắc metrics định kỳ là bắt buộc.
3. **Bảo tồn Raw Lineage là điều kiện tiên quyết của Self-Healing:** Cơ chế Idempotent Repair chỉ có thể hoạt động khi dữ liệu thô ban đầu được lưu giữ nguyên trạng và có khả năng tái lập độc lập.
