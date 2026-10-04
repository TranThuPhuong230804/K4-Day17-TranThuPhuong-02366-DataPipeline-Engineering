# K4-Track02-Day17 — Hướng dẫn nộp bài

Đây là **bài cá nhân** cho K4, Track 02, Ngày 17.
Mỗi học viên nộp repo của mình; không dùng repo chung của nhóm.

## Tên repo

- Repo đề bài: `K4-Track02-Day17-Data-Pipeline-Engineering`.
- Repo bài nộp: `K4-Track02-Day17-TranThuPhuong-2A202602366-DataPipelineEngineering`.
- Ví dụ: `K4-Track02-Day17-NguyenVanAn-20260001-DataPipelineEngineering`.

Họ tên viết không dấu, không khoảng trắng; phân cách các phần bằng dấu `-`.
Dùng MSSV thực của bạn. Có thể fork rồi đổi tên repo hoặc tạo repo mới,
giữ mã nguồn đề bài và lịch sử các thay đổi để người chấm đọc được ba cách sửa.

## Nơi nộp và deadline

Nộp **một URL repo GitHub public** vào ô bài tập **K4 / Track 02 / Day 17** trên LMS.
Không nộp bằng pull request. Giữ repo truy cập được đến khi có kết quả chấm.

Deadline mặc định là **23:59 ngày diễn ra lab, múi giờ Asia/Ho_Chi_Minh (UTC+7)**.
Nếu key coach điều chỉnh, áp dụng thông báo trên LMS được gửi trong vòng 48 giờ
sau lab theo quy ước chung. Xem ngày học thực tế trên lịch lớp;
các ngày `2026-08-10` đến `2026-08-16` trong seed chỉ dùng để mô phỏng dữ liệu.

Quy định nộp muộn và sửa sau deadline nằm trong [RULES.md](RULES.md).

## Cấu trúc và file phải nộp

Giữ các file mã nguồn, tests, dữ liệu seed và cấu hình của đề bài. Bổ sung kết quả:

```text
K4-Track02-Day17-HoVaTen-MSSV-DataPipelineEngineering/
├── README.md
├── main.py                  # Điểm chạy pipeline
├── docs/                    # Hướng dẫn đề bài và rubric
├── scripts/                 # Công cụ kiểm tra và sinh seed
├── pipeline/                 # Mã nguồn đã sửa
├── dbt_project/              # Model và cấu hình của phần dbt
├── tests/                   # Giữ nguyên bộ test đề bài
├── data/                    # Giữ nguyên dữ liệu seed
├── submission/
│   ├── REPORT.md            # Thông tin học viên, phân tích và output thực tế
│   └── checksums.txt        # Kết quả rerun, được sinh tự động
└── bonus/                   # Nếu làm bonus B2
    ├── DESIGN.md            # Nếu chọn brainstorm
    └── airflow/             # Nếu chọn Airflow: ảnh runs và log checksum
```

- Code: sửa ba lỗi trong `pipeline/`; các thay đổi phải đọc được trong Git.
- `submission/checksums.txt`: sinh bằng `make rerun3` hoặc `python -m scripts.rerun_check`, kết quả `PASS`.
- `submission/REPORT.md`: điền họ tên, MSSV, URL repo; phân tích tối đa một trang,
  không tính phần output. Nêu triệu chứng, nguyên nhân, cách sửa và khái niệm của
  từng lỗi; giải thích lựa chọn kỹ thuật và trả lời hai câu hỏi suy ngẫm.
- Dán output thực tế của verify, pytest, rerun, lateness, dbt build và parity vào REPORT.
- Nếu làm B1: nộp thay đổi `pipeline/llm_label.py` và output `BONUS PASS` trong REPORT.
- Nếu làm B2: nộp `bonus/DESIGN.md` hoặc ảnh bảy run Airflow thành công cùng log checksum.

Giới hạn một trang áp dụng cho phần phân tích (mục 1–4 của REPORT), theo định dạng
tham chiếu A4, Arial 12, giãn dòng 1,15, lề 2 cm. Không tính thông tin học viên,
output và bằng chứng bonus; vẫn nộp `REPORT.md`, không bắt buộc nộp PDF.
Nếu chọn B2 Airflow, làm theo [AIRFLOW.md](AIRFLOW.md).

Không cần commit `.venv/`, `lake/`, các database `.duckdb`, dbt `target/`, `logs/`
hay API key. Các kết quả này có thể dựng lại từ mã nguồn và seed.
`submission/checksums.txt` là bằng chứng bắt buộc và phải commit.

## Kiểm tra trước khi nộp

Trên macOS/Linux hoặc shell tương thích Makefile:

```bash
make setup
make setup-dbt
make verify
make test
make rerun3
make lateness
make dbt
make parity
```

Trên Windows PowerShell, chạy từ thư mục gốc repo:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m pip install -r requirements-dbt.txt
.\.venv\Scripts\python.exe -m scripts.verify
.\.venv\Scripts\python.exe -m pytest
.\.venv\Scripts\python.exe -m scripts.rerun_check
.\.venv\Scripts\python.exe main.py --lateness
.\.venv\Scripts\python.exe main.py --land-only
$env:DO_NOT_TRACK = '1'
Push-Location dbt_project
try {
    ..\.venv\Scripts\dbt.exe build --profiles-dir . --event-time-start 2026-08-10 --event-time-end 2026-08-17
} finally {
    Pop-Location
}
.\.venv\Scripts\python.exe -m scripts.parity
```

Khi chưa sửa đề bài, verify và một số tests fail là bình thường.
Trước khi nộp, kiểm tra:

- [ ] Verify đạt `18/18 — ALL PASS`; pytest không còn test fail.
- [ ] Rerun đạt `PASS`: checksum của fresh build và ba lần chạy lại bằng nhau.
- [ ] P99 lateness đã được đo từ Bronze và ghi trong REPORT.
- [ ] dbt build thành công; parity đạt `PARITY`.
- [ ] REPORT đã điền đầy đủ; output lấy từ lần chạy trên code bài nộp.
- [ ] `submission/checksums.txt` và các thay đổi đã commit, push lên GitHub.
- [ ] Tên repo đúng; URL mở được khi chưa đăng nhập; đã nộp URL trên LMS.
- [ ] Repo không chứa secret hay dữ liệu khách hàng thật.

Tiêu chí và điểm: [RUBRIC.md](RUBRIC.md). Các mốc thực hành: [CHECKPOINTS.md](CHECKPOINTS.md).
