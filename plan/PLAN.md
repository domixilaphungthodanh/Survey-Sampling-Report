# Survey-Sampling-Report — Kế hoạch thực hiện báo cáo khảo sát lựa chọn chứng chỉ tiếng Anh

> **Trạng thái:** *Proposed* — kế hoạch xây dựng từ Issue [#1](https://github.com/ryantr-statinops/Survey-Sampling-Report/issues/1) của repo này và Issue tham khảo [#1 của VisuaSlide](https://github.com/ryantr-statinops/VisuaSlide/issues/1). Chưa phải báo cáo cuối.
> **Ngày xây dựng:** 06/11/2026 · **Nguồn chính:** Google Sheets tab `gid=1653201454` · **SDK tham khảo:** [VisuaSlide](https://github.com/ryantr-statinops/VisuaSlide)

---

## 1. Vai trò của repo này và kết quả "clone + tìm hiểu"

### 1.1. Repo `Survey-Sampling-Report` là gì?

- URL công khai: <https://github.com/ryantr-statinops/Survey-Sampling-Report>
- Thư mục làm việc hiện tại đã cấu hình sẵn remote:
  ```
  origin  https://github.com/ryantr-statinops/Survey-Sampling-Report.git (fetch/push)
  branch: main — chưa có commit nào
  ```
- Kiểm chứng clone:
  - `git fetch origin` → không có branch/commit nào được tải về.
  - `git ls-remote origin` → (trống) → xác nhận remote đang **empty**.
  - Trang GitHub hiển thị *“This repository is empty”* nhưng có **1 issue mở**.
- **Kết luận:** không có code/SDK sẵn để học trong repo này. Mọi spec phải lấy từ Issue #1 của repo + dữ liệu Google Sheets + repo SDK `VisuaSlide` (repo đã có code, được clone về thư mục `_VisuaSlide_ref/` để đọc).

### 1.2. Đã học được gì từ VisuaSlide (SDK)?

VisuaSlide là **Python SDK + CLI** để biến *một tab khảo sát trong Google Sheets* thành phân tích tái tạo được và bản trình bày Google Slides có thể refresh. Chi tiết đã đọc từ `README.md`, `AGENTS.md`, `pyproject.toml`, `docs/offline-workflow.md`, `docs/google-setup-and-recovery.md`.

**Luồng chuẩn (Accepted trong VisuaSlide):**
```
Một tab Google Sheets
  → Raw snapshot (API values + metadata)
  → Validate & prepare bằng pandas
  → Clean dataset (Parquet)
  → Aggregate analysis bằng DuckDB SQL
  → Một kết quả phân tích duy nhất (metrics, tables, reporting content)
  → Reporting spreadsheet (bảng tổng hợp + Sheets charts)
  → Google Slides (linked charts, tables, text)
```

**Công nghệ khóa (theo `pyproject.toml` / README):** CPython 3.12, `uv + pyproject.toml + uv.lock`, Hatchling, `google-api-python-client`, `google-auth`, `google-auth-oauthlib`, Pydantic + TOML, pandas, PyArrow/Parquet, DuckDB, Typer CLI, pytest + Ruff, GitHub Actions.

**Cách dùng SDK (offline, đã triển khai):**
```powershell
uv sync --all-groups
uv run python examples/offline_workflow.py
uv run visuaslide refresh --config examples/survey.example.toml --fixture-json tests/fixtures/synthetic_responses.json --dry-run
uv run visuaslide validate --config examples/survey.example.toml --snapshot-id <snapshot-id>
uv run visuaslide analyze --config examples/survey.example.toml --snapshot-id <snapshot-id>
uv run visuaslide inspect-run --config examples/survey.sandbox.local.toml --run-id <run-id>
uv run visuaslide resume --config examples/survey.sandbox.local.toml --run-id <run-id>
```

**Phần Google (chưa xác thực — *Open* / chưa qua sandbox):**
```powershell
Copy-Item examples/survey.example.toml examples/survey.sandbox.local.toml
# sửa spreadsheet_id, sheet_id (gid), schema, report text, project_dir
uv sync --locked --group dev
uv run --locked visuaslide auth --config examples/survey.sandbox.local.toml --client-secret C:\private\client_secret.json
uv run --locked visuaslide bootstrap-outputs --config examples/survey.sandbox.local.toml
uv run --locked visuaslide refresh --config examples/survey.sandbox.local.toml --dry-run
uv run --locked visuaslide refresh --config examples/survey.sandbox.local.toml
uv run --locked visuaslide review-layout --config examples/survey.sandbox.local.toml
```
→ `bootstrap-outputs` tạo/reuse reporting spreadsheet + template + presentation; ID và mapping nằm ở `<project_dir>/google_outputs.json` (git-ignored). Reporting spreadsheet là **output**, không phải nguồn khảo sát thứ hai. Refresh đọc **toàn bộ bảng** mỗi lần để phản ánh thêm/sửa/xóa. Mỗi lần chạy ghi run ID, thời gian UTC, source identity/range, hash dữ liệu/cấu hình, SDK/template version. Không báo success khi chỉ mới update một phần Sheets/Slides.

**Ý nghĩa cho repo này:** thay vì tự viết pipeline hoàn toàn mới, ta có thể **áp dụng chính pattern/SDK của VisuaSlide** cho tab khảo sát chứng chỉ tiếng Anh (một `spreadsheet_id + sheet_id`), tạo bản **tự động refresh Slides từ Sheets** như yêu cầu gốc. Vì Google adapter chưa được kiểm chứng live, kế hoạch bên dưới giữ cơ chế *manual refresh trước, scheduling sau* và yêu cầu sandbox acceptance riêng.

### 1.3. Nguồn dữ liệu khảo sát

- Google Sheets: <https://docs.google.com/spreadsheets/d/14Ic0Zl7V5fr5dOkdo7ae5pwZpzk8v3HPGKliDHEQTiw/edit?gid=1653201454>
- Chỉ dùng **tab `gid=1653201454`** (đặt tên trong TOML bằng `spreadsheet_id` + `sheet_id`, SDK tự resolve tiêu đề tab hiện tại).
- Khả dụng trực tiếp không cần auth khi sheet public qua CSV export:
  `https://docs.google.com/spreadsheets/d/14Ic0Zl7V5fr5dOkdo7ae5pwZpzk8v3HPGKliDHEQTiw/export?format=csv&gid=1653201454`
  Đã kiểm chứng: header gồm `Dấu thời gian, Câu 1…Câu 32, Địa chỉ email`; 32 cột câu hỏi đúng thứ tự Issue. Cột `Địa chỉ email` **phải loại khỏi báo cáo công khai**.
- Bản tham khảo 04/11/2026 ghi **89 lượt trả lời, 32 câu hỏi**, khoảng thời gian `05/10/2026 – 02/11/2026`. Khi thực hiện, phải **kiểm tra lại số bản ghi và ghi rõ ngày trích xuất** (số liệu có thể tăng nếu có trả lời mới).

---

## 2. Mục tiêu

- Trích xuất nguyên văn 32 câu, thống kê từng phương án, trực quan hóa bằng biểu đồ tròn, thanh, thanh chồng 100%, so sánh điểm trung bình.
- Viết nhận xét/kết luận mô tả đúng đặc điểm mẫu, không suy diễn nhân quả hoặc khái quát toàn trường.
- Tạo sản phẩm độc lập (báo cáo + Excel + ảnh biểu đồ riêng) và pipeline **tự động cập nhật Google Slides từ Google Sheets** dùng chính SDK VisuaSlide.

---

## 3. Nội dung chi tiết theo Issue #1

### 3.1. Trích xuất và kiểm tra dữ liệu

- [ ] Tổng hợp nguyên văn 32 câu, phân nhóm: chọn 1 đáp án, chọn nhiều đáp án, chấm điểm 1–5, góp ý mở (Câu 32).
- [ ] Kiểm tra thiếu dữ liệu, khoảng trắng, cách ghi tên chứng chỉ, lựa chọn mâu thuẫn; ghi rõ quy tắc chuẩn hóa, **giữ nguyên dữ liệu gốc** (từ điển chuẩn hóa riêng).
- [ ] Loại email / định danh không cần thiết khỏi dữ liệu công khai; chỉ giữ số bản ghi / mã ẩn danh (vd. PH001…) nếu cần thiết cho đối chiếu, không lộ email.
- [ ] Snapshot bất biến: lưu raw values + metadata (run ID, thời gian UTC, source identity/range, hash) theo chuẩn VisuaSlide, để replay và verify bằng independent fixtures.

### 3.2. Thống kê theo nhóm câu

| Nhóm | Câu | Chỉ tiêu |
|---|---|---|
| Đặc điểm mẫu | 1–5 | Giới tính, năm học, khoa, trình độ tiếng Anh, kinh nghiệm ôn/thi |
| Nhu cầu & hành vi | 6–10 | Chứng chỉ đã thi, dự định thi, mục đích, thời gian dự thi, nguồn thông tin |
| Thang điểm 1–5 | 11–27 | Số lượt từng mức, tỷ lệ, điểm TB từng yếu tố, TB theo nhóm yếu tố |
| Đánh giá tổng thể | 28–31 | Mức phù hợp, ý định giới thiệu, khó khăn, thông tin mong muốn từ trường |
| Góp ý mở | 32 | Số trống/không trống, tổng hợp chủ đề (định tính, không chấm tần suất giả) |

- Mỗi phương án trình bày **số lượt chọn + tỷ lệ % + mẫu số**.
- Phân biệt **số bản ghi** vs **số người duy nhất đã xác minh** (chưa có cơ chế xác minh trùng lặp).
- Câu 11–27: phân bố điểm, TB từng yếu tố và TB theo 5 nhóm:
  - *Yếu tố cá nhân* (11–13)
  - *Đặc điểm của chứng chỉ* (14–17)
  - *Chi phí* (18–20)
  - *Ý kiến/thông tin từ người khác* (21–24)
  - *Yếu tố từ nhà trường* (25–27)
- Câu 32 (tham khảo VisuaSlide #1): 51 không trống (57,3%) / 38 trống (42,7%); các câu như “Không”, “oke”, “N/A” tính là có nhập nội dung, không tự coi là góp ý thực chất; phân nhóm chủ đề định tính có thể chồng lấn.

### 3.3. Quy tắc xử lý bắt buộc (theo Issue)

- Câu chọn nhiều đáp án: mẫu số = số bản ghi trả lời; tổng tỷ lệ có thể **vượt 100%**; mỗi phương án chỉ tính **một lần mỗi bản ghi** (deduplicate trong cùng một ô).
- Giữ nguyên phương án **“So sánh IELTS, TOEIC và APTIS” ở câu 31**, không tách dấu phẩy bên trong tên phương án (parser phải dùng danh sách phương án đã biết, không split-naive).
- Ghi chú mâu thuẫn:
  - Câu 5–6: tham khảo ghi **15 bản ghi** không chọn “Đã thi ít nhất một chứng chỉ” nhưng vẫn ghi tên chứng chỉ đã thi; 0 bản ghi ngược lại → câu 6 chỉ phản ánh lựa chọn đã ghi, chưa xác nhận lịch sử thi thực tế.
  - Câu 30: **6 bản ghi** chọn đồng thời “Không gặp khó khăn” + một khó khăn khác → giữ nguyên khi thống kê, không cộng để suy số người gặp khó khăn.
  - Câu 28 có 6 lượt “Chưa chọn chứng chỉ”, câu 29 có 7 lượt → không đồng nhất hai nhóm.
- Gộp biến thể PET ở câu 6: `Pet`, `Cambridge (PET)`, `PET Cambridge` → `PET (Cambridge)`; dữ liệu gốc không sửa.
- Trim khoảng trắng, loại mục rỗng sau dấu phẩy: vd. `"APTIS, "`, `"Mạng xã hội hoặc Internet, "`.
- Chưa có chú giải 1–5 → chỉ trình bày điểm số, **không** tự gán nhãn “đồng ý” / “mức ảnh hưởng”. Tỷ lệ chọn 4–5 chỉ là mô tả.
- ทำ tròn 1 chữ số thập phân cho tỷ lệ; giữ precision cho tính TB; mọi % phải ghi rõ tử số/mẫu số/policy thiếu dữ liệu (theo VisuaSlide rules).
- Không loại bản ghi chỉ vì chọn cùng một điểm ở nhiều câu; giữ tất cả responses không trống ở V0.1 (theo VisuaSlide V0.1).

### 3.4. Biểu đồ

Yêu cầu Issue: phải có tiêu đề, nhãn, đơn vị, số lượng/tỷ lệ, chú giải phù hợp; không bị cắt nhãn.

- **Biểu đồ tròn** (single-choice): Câu 1, 2, 3, 4, 5, 7, 9, 28, 29 → 9 biểu đồ riêng. Câu 3 có 13 khoa → cần legend đọc được, có thể chia panel hoặc kéo dài kích thước.
- **Biểu đồ thanh** (multi-choice): Câu 6, 8, 10, 30, 31 → 5 biểu đồ riêng. Trục % theo mẫu số 89.
- **Thanh chồng 100%**: phân bố điểm câu 11–27 (17 yếu tố × 5 mức).
- **Thanh so sánh điểm TB**: 17 yếu tố; thêm (nếu cần) TB theo 5 nhóm.
- Xuất **PNG riêng từng biểu đồ** (300 dpi, nền trắng) để chèn báo cáo/GSlides độc lập, không chỉ bọc trong composite.

### 3.5. Nhận xét và kết luận

- Nêu đặc điểm mẫu (koncentr ở năm ba + Khoa Toán-Thống kê), phương án phổ biến, nhu cầu chứng chỉ, mục đích, nguồn thông tin, khó khăn, nhu cầu hỗ trợ — tất cả là **mô tả**.
- Đề xuất nội dung tư vấn phù hợp từ kết quả: tài liệu so sánh IELTS/TOEIC/APTIS theo mục tiêu/trình độ/cấu trúc/chi phí/thời gian; lộ trình học + tự đánh giá; truyền thông qua kênh online + dẫn về nguồn chính thức; đối chiếu quy định công nhận với thông báo hiện hành.
- Nêu giới hạn mẫu, không suy diễn nhân quả hay đại diện toàn trường khi chưa có thiết kế lấy mẫu.
- Cấu trúc Slides/báo cáo theo VisuaSlide: research question, sampling/collection method, sample composition, data quality, key results, group comparisons, conclusions with limitations.

---

## 4. Kiến trúc kỹ thuật đề xuất (tái dùng VisuaSlide)

### 4.1. Principled reuse

- Dùng trực tiếp `VisuaSlide` làm SDK (pin qua `uv`, `pyproject.toml`, Python 3.12). Nếu chưa tách package riêng, copy cấu hình + code adapter theo pattern `src/visuaslide/` nhưng **không sửa source sheet**; chỉ viết vào reporting spreadsheet + presentation do app quản lý.
- Cấu hình dạng TOML (`survey.example.toml` làm khuôn): `source.spreadsheet_id`, `source.sheet_id`, header mapping, column types, valid categories, missing rules, date/timezone, project_dir. Chỉ một tab nguồn duy nhất.
- Tách tầng rõ ràng: source access / preparation (pandas) / analysis (DuckDB SQL, không tính metric trùng 2 nơi) / storage (Parquet) / reporting / orchestration / CLI (Typer).

### 4.2. Sản phẩm trung gian (theo VisuaSlide layout)

```
data/
  raw_snapshot_{run-id}.json         # API values + metadata
  clean_english_certs.parquet        # typed dataset
output/
  stats.xlsx                         # aggregate tables mở bằng Excel
  charts/                            # PNG riêng từng chart
  report.md                          # báo cáo độc lập
  run_manifest_{run-id}.json         # run record
```

- Thư mục runtime (`.visuaslide/`, `*_local.toml`, token, `google_outputs.json`) → **git-ignored**, không commit credential/token/snapshot chứa dữ liệu thật.
- Reporting spreadsheet là output của app; chart trong Sheets được embed-linked vào Slides (Slides API + chart guide). Không làm nguồn khảo sát.

### 4.3. Mapping câu hỏi → config analysis

- Single-choice (Câu 1–5,7,9,28–29): frequency + % trên 89.
- Multi-choice (Câu 6,8,10,30,31): split theo danh sách phương án đã biết → count/% per option, dedup mỗi bản ghi, tổng >100% ghi chú.
- Numeric 1–5 (Câu 11–27): count per level, mean, median, min/max, sd (mô tả), 4–5 share, group mean.
- Open (Câu 32): non-empty/empty count + thematic synthesis, kèm appendix nguyên văn chỉ khi đã ẩn danh.

---

## 5. Danh mục bàn giao (theo Issue)

- [ ] Báo cáo tổng hợp đủ 32 câu, có bảng thống kê, nhận xét, kết luận; mở đọc độc lập (Markdown → PDF/HTML).
- [ ] Bảng thống kê chi tiết: số lượt, tỷ lệ, mẫu số (Excel workbook có thể mở).
- [ ] Bộ biểu đồ: tròn (9), thanh (5), thanh chồng 100% (1), TB 17 yếu tố (1) — có ảnh PNG riêng từng file.
- [ ] Nhãn biểu đồ không bị cắt; kiểm tra tiếng Việt, label dài, bảng overflow, order/màu ổn định.
- [ ] Đối chiếu tổng số & tỷ lệ với nguồn, kiểm tra chéo independent fixtures nhỏ.
- [ ] Ghi rõ nguồn, ngày trích xuất, quy tắc xử lý, giới hạn dữ liệu (phụ lục phương pháp).

---

## 6. Automation refresh Google Slides (remote update)

1. **Manual trước, schedule sau** (VisuaSlide principle): `auth → bootstrap-outputs → refresh --dry-run → refresh → review-layout`.
2. Mỗi lần refresh:
   - Đọc toàn bộ tab đã cấu hình → detect thêm/sửa/xóa so với snapshot trước.
   - Tái tính cùng một analysis result → update đồng bộ bảng tổng hợp, linked chart, text/tables trong presentation.
   - Dùng persisted object mapping (`google_outputs.json`), **không** tìm slide/chart bằng text hiển thị cũ; không tạo duplicate slides/charts/files khi chạy lặp.
   - Partial failure phải ghi rõ trong run record; `resume` dùng saved analysis, bỏ qua checkpoint đã xong, không đọc lại source; **không báo success khi mới update một phần**.
3. Kiểm định sandbox trước khi dùng dữ liệu thật: tạo Sheets sandbox chỉ với synthetic responses theo đúng header của `survey.example.toml`, chạy `pytest tests/sandbox` với `VISUASLIDE_SANDBOX_CONFIG`, test add/edit/delete, nhìn thumbnail Slides (checklist `google-layout-review.md`).
4. Locking + bounded retries cho GitHub/Task Scheduler; tránh overlapping refresh; credential lifecycle kiểm chứng trước khi cam kết unattended (Testing-mode refresh token có thể hết hạn 7 ngày).

---

## 7. Kiểm tra và tiêu chí hoàn thành

- **Reproducibility:** snapshot + Parquet + DuckDB logic cho cùng input ra cùng output; run ID + hash ghi đủ.
- **Statistical fixtures:** tính thử trên fixtures nhỏ independently calculated (frequency, mean, group mean, multi-select dedup, PET gộp, Q31 không tách).
- **Data quality checks:** missing/duplicate headers, unknown categories, invalid values, empty inputs, missing tabs — báo counts, không lộ response payload trong log.
- **Chart QA:** tồn tại PNG riêng cho từng chart yêu cầu; title/label/unit/legend/% đầy đủ; song ngữ/việt đúng chính tả; không bị cắt.
- **Slides QA:** template vs generated presentation tách riêng; placeholders/mappings ổn định sau refresh; vùng SDK-managed có thể bị thay đổi bởi refresh (ghi rõ); thumbnail review từng slide.
- **Acceptance (mapped từ Issue):** đủ 32 câu · số lượt/tỷ lệ/mẫu số · bộ 16 biểu đồ (9 tròn + 5 thanh + 1 chồng 100% + 1 TB) · báo cáo đọc độc lập + Excel · đối chiếu nguồn · ghi nguồn/ngày trích xuất/quy tắc/giới hạn · không lộ email.

---

## 8. Milestones đề xuất

| Giai đoạn | Nội dung | Trạng thái |
|---|---|---|
| P0 — Setup | Clone/kiểm chứng repo rỗng; clone VisuaSlide về `_VisuaSlide_ref` đọc README/AGENTS/docs; tạo `plan/`; `.gitignore` cho runtime/token | *Implemented* (bản plan này) |
| P1 — Offline pipeline | `uv sync`, viết TOML cho tab gid=1653201454, snapshot → pandas + DuckDB → aggregate tables; test trên synthetic fixture; verify số liệu vs VisuaSlide #1 (89/32) | *Proposed* |
| P2 — Charts + Excel | Sinh 16 biểu đồ PNG + workbook stats; đối chiếu với số liệu tham khảo (Nam 53,9% · IELTS dự định 37,1% · mục đích “xin việc” 65,2% · nguồn Internet 79,8% · khó khăn độ khó 47,2% · lộ trình 56,2% · TB mục tiêu nghề nghiệp 4,11 …) | *Proposed* |
| P3 — Report + Slides | Viết báo cáo 32 câu + bootstrap Sheets/Slides; link charts; review-layout; thumbnail QA | *Proposed* |
| P4 — Automation + Acceptance | Sandbox live acceptance: add/edit/delete → refresh; manual refresh ổn định; resume từ partial failure; lock + scheduled refresh (chỉ khi đã verified); ghi nhận acceptance | *Open* — phụ thuộc Google sandbox |

---

## 9. Cấu trúc thư mục đề xuất (chưa tạo)

```
plan/
  PLAN.md            # file này
  README.md          # index + link tới Issue/Sheets
config/
  survey.english_certs.toml.example   # schema 32 câu (không chứa token)
src/visuaslide/      # tái dùng SDK VisuaSlide (git submodule hoặc copy)
data/  output/  reports/              # runtime artifacts, git-ignored khi chứa dữ liệu thật
docs/
  method.md  limitations.md           # phương pháp + giới hạn khi viết báo cáo
```

---

## 10. Quyết định mở (Open) và rủi ro

- **Open:** Tên chính xác 32 header từ sheet có thay đổi không; cần freeze schema version trước coding.
- **Open:** Labels của thang 1–5 chưa có → không gán nhãn giải thích; chỉ giữ điểm số.
- **Open:** Thiết kế lấy mẫu chưa biết → không dùng suy luận quần thể, weights, CI, hypothesis tests.
- **Open:** Google Cloud project/OAuth/scope (`spreadsheets.readonly` + `drive.file`) chưa sandbox-verified; scheduling (Windows Task Scheduler vs GitHub Actions) chưa quyết; cloud scheduling cần quyết định auth/storage riêng.
- **Risk:** Sheet public CSV có thể đổi số dòng N khi có trả lời mới → phải snapshot theo ngày, không overwrite số liệu 89 cũ mà không ghi rõ ngày trích xuất mới.
- **Risk:** Biểu đồ tròn câu 3 có 13 nhóm khoa → nguy cơ legend chật/cắt nhãn nếu dùng layout Slides mặc định.
- **Risk:** Gộp PET và parser multi-select dễ sai nếu split-naive bằng dấu phẩy (đặc biệt Q31).

---

## 11. Câu hỏi cần chủ repo xác nhận trước khi code

1. Python 3.12 + `uv` có sẵn không? Có muốn tách SDK riêng hay dùng trực tiếp repo VisuaSlide như dependency/path?
2. Reporting spreadsheet + Slides presentation mới (app-managed) hay dùng deck sẵn có? Cần cung cấp file ID/template?
3. Ngày trích xuất chốt cho báo cáo là ngày chạy script (live) hay cố định 04/11/2026? 
4. Cột email có cần giữ ở raw snapshot nội bộ hay loại hẳn từ đầu?
5. Ngôn ngữ báo cáo/Slides: tiếng Việt toàn bộ (khớp Issue) hay song ngữ?
6. Lịch tự động refresh mong muốn (thủ công trước khi schedule) và kênh xác thực OAuth cá nhân?

---

### Phụ lục nhanh — Key figures tham khảo (VisuaSlide #1, 04/11/2026)

- 89 lượt trả lời · Nam 53,9% (48) · Năm ba 67,4% (60) · Toán-Thống kê 46,1% (41)
- Dự định thi: IELTS 37,1% (33) · TOEIC 20,2% · APTIS 16,9% · Chưa định 23,6%
- Mục đích: Xin việc/phát triển nghề nghiệp 65,2% (58) · Nguồn internet 79,8% (71)
- TB cao nhất: Mục tiêu nghề nghiệp 4,11/5; nhóm cá nhân 4,03/5
- Phù hợp: Phù hợp+Rất phù hợp 66,3% (59) · Giới thiệu: Có lẽ có+Chắc chắn có 73,0% (65)
- Khó khăn: Lo ngại độ khó/đạt điểm 47,2% (42) · Mong muốn: Lộ trình học/ôn thi 56,2% (50)
- Góp ý mở: 51 không trống (57,3%) · Đã thi: APTIS 66,3% (59) · Câu 5: Đã thi ít nhất một chứng chỉ 66,3% (59)

> Mọi số liệu trên chỉ là điểm tham khảo; việc triển khai phải tự trích xuất lại từ tab gid=1653201454 đúng ngày chạy.

