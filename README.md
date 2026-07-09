# 🪂 Airdrop Suite

Một project duy nhất gồm **Backend (FastAPI)** + **App 1 (List Airdrop)** +
**App 2 (Check Airdrop)**, chạy như một phần mềm desktop local — chỉ cần
`python launcher.py`.

Backend là **Single Source Of Truth**: toàn bộ dữ liệu nằm trong SQLite,
App 1 và App 2 tuyệt đối không đọc/ghi database hay file JSON trực tiếp —
mọi thao tác đều thông qua REST API của Backend.

## Cấu trúc project

```
AirdropSuite/
├── backend/              # FastAPI — Single Source of Truth
│   ├── main.py            # App, routes, mount /app1 /app2 /static
│   ├── config.py, database.py, models.py, schemas.py
│   ├── crud.py, storage.py, utils.py
│   ├── static/, templates/   # Dashboard riêng của Backend (tại "/")
│   ├── logs/, uploads/
│   └── database.db        # (tự tạo khi chạy lần đầu)
│
├── app1/                 # List Airdrop — quản lý task/checklist hàng ngày
│   └── index.html          # 1 file HTML độc lập (Vanilla JS, không build step)
│
├── app2/                 # Check Airdrop — phân tích & watchlist dự án
│   ├── index.html
│   ├── css/style.css
│   └── js/ (config, api, sidebar, dashboard, accordion, prompt-builder, search, app)
│
├── launcher.py            # Điểm khởi chạy duy nhất
├── requirements.txt
└── README.md
```

## Cài đặt & Chạy

```bash
pip install -r requirements.txt
python launcher.py
```

Launcher sẽ:
1. Khởi động Backend (FastAPI/Uvicorn) tại `http://127.0.0.1:8000`.
2. Kiểm tra Backend đã sẵn sàng (health check).
3. Mở cửa sổ **AIRDROP SUITE** với 2 nút bấm:
   - **APP 1** → mở `http://127.0.0.1:8000/app1/`
   - **APP 2** → mở `http://127.0.0.1:8000/app2/`
   - (kèm nút phụ **Backend Dashboard** → `http://127.0.0.1:8000/`)
4. Khi đóng cửa sổ Launcher → Backend tự **shutdown hoàn toàn**, không để
   lại tiến trình nền (zombie process). Nếu máy không có Tkinter (một số
   bản Python tối giản trên Linux), Launcher tự chuyển sang menu Console
   tương đương, vẫn đảm bảo tắt sạch Backend khi thoát.

> Chạy trên Windows/macOS/Linux đều dùng chung 1 lệnh `python launcher.py`.
> Không cần cài thêm gì ngoài `requirements.txt` (Tkinter đi kèm sẵn trong
> hầu hết bản cài đặt Python chính thức).

## Kiến trúc dữ liệu (Data Model chuẩn — dùng chung 3 thành phần)

```jsonc
{
  "id": "string",
  "category": "TESTNET | DEPIN",
  "name": "string",
  "logo": "string (url)",
  "links": {
    "website": "", "x": "", "discord": "", "guild": "", "galxe": "", "docs": "",
    "other": [{ "label": "", "url": "" }]
  },
  "tasks": [{ "id": "", "text": "", "done": false, "doneAt": null }],
  "favorite": false,
  "event": "ACTIVE | SNAPSHOT | TGE_SOON | CLAIM_LIVE | ENDED",
  "analysis": {
    "score": null, "last_update": null,
    "info": {}, "funding": {}, "backers": [], "roadmap": {},
    "tokenomics": null, "community": {}, "development": {},
    "news": [], "ai_analysis": {}, "notes": ""
  },
  "created_at": "ISO datetime",
  "updated_at": "ISO datetime"
}
```

`backend/utils.py::normalize_project()` là nơi DUY NHẤT chuẩn hoá dữ liệu
này — mọi API tạo/sửa/import đều đi qua hàm này để đảm bảo App 1 và App 2
luôn thấy đúng 1 cấu trúc, không lệch field.

## API (Backend)

| Method | Path | Mô tả |
|---|---|---|
| GET | `/api/health` | Health check |
| GET | `/api/projects` | Lấy toàn bộ project |
| PUT | `/api/projects` | Ghi đè toàn bộ (App 1 dùng khi save) |
| GET | `/api/project/{id}` | Lấy 1 project |
| POST | `/api/project` | Tạo mới |
| PUT | `/api/project/{id}` | Cập nhật (partial) |
| PATCH | `/api/project/{id}` | Cập nhật (partial, alias của PUT) |
| DELETE | `/api/project/{id}` | Xoá |
| POST | `/api/import` | Import/merge JSON (update theo id, tạo mới nếu chưa có) |
| GET | `/api/export` | Export toàn bộ ra JSON |
| GET | `/api/stats` | Thống kê tổng quan |
| GET | `/api/activity` | Lịch sử hoạt động |
| GET | `/api/settings` / PUT | Cấu hình lưu trong DB |
| POST | `/api/backup` | Backup SQLite ra `/backend/logs` |
| GET | `/api/logs` | Danh sách file log/backup |

Xem chi tiết & thử trực tiếp tại Swagger UI: `http://127.0.0.1:8000/docs`.

## App 1 — List Airdrop

Giữ nguyên 100% UI/UX/logic gốc (tab TESTNET/DEPIN, search, task checklist,
pagination, modal thêm/sửa, favorite, import/export...). Điểm thay đổi
DUY NHẤT: tầng `Storage` bên trong `app1/index.html` được bật
`CONFIG.backend = true` và gọi Backend qua `/api` (đường dẫn tương đối,
hoạt động đúng dù chạy ở port nào). Nếu Backend tạm thời không phản hồi,
App 1 tự động fallback về `localStorage` và hiển thị "Offline Mode" —
cơ chế này vốn đã có sẵn trong code gốc, không cần chỉnh sửa gì thêm.

## App 2 — Check Airdrop

Giữ nguyên 100% UI/UX/logic gốc (sidebar project list, filter, sort,
accordion phân tích, prompt builder cho AI check-news, import JSON,
health indicator...). App 2 chỉ đọc dữ liệu (GET) và ghi lại kết quả phân
tích AI thông qua Import (POST `/api/import` — update theo id nếu đã tồn
tại). Điểm thay đổi: `apiBase` mặc định lấy theo origin hiện tại
(`window.location.origin`) thay vì hardcode `http://localhost:8000`, để
hoạt động đúng dù Backend chạy ở host/port nào; vẫn giữ nguyên tính năng
cho phép người dùng tự đổi API Base URL trong màn hình Cài đặt.

## Các lỗi đã phát hiện & sửa trong quá trình refactor

1. **Lỗi nghiêm trọng (crash toàn bộ tính năng tạo/sửa project):**
   `utils.normalize_project()` gán `created_at`/`updated_at` dạng chuỗi ISO
   thẳng vào cột kiểu `DateTime` của SQLAlchemy — SQLite driver từ chối giá
   trị không phải `datetime` object và ném lỗi 500 ở MỌI request tạo/sửa
   project. Đã thêm `utils.parse_datetime()` để luôn convert về đúng kiểu
   `datetime` trước khi lưu.
2. Cập nhật 1 project qua `PUT/PATCH /api/project/{id}` không tự cập nhật
   `updated_at` (do field này không nằm trong `ProjectUpdate` schema nên
   không bao giờ được set) — đã sửa để backend luôn tự bump `updated_at`
   và giữ nguyên `created_at` gốc.
3. `POST /api/import` không trả về danh sách project đã lưu, khiến App 1
   sau khi import phải tự dùng lại dữ liệu thô (chưa chuẩn hoá) thay vì dữ
   liệu thật trong DB — đã bổ sung field `projects` trong response.
4. Import merge ghi đè luôn `created_at` của project đã tồn tại về thời
   điểm import — đã sửa để giữ nguyên `created_at` gốc trừ khi JSON import
   chỉ định rõ ràng.
5. Trang Dashboard riêng của Backend (`backend/static/app.js`) gọi sai
   endpoint: `/api/projects/export` và `/api/projects/import` (dùng
   `FormData` file upload) trong khi API thật là `GET /api/export` và
   `POST /api/import` (nhận JSON) — đã sửa lại đúng theo API chuẩn.
6. App 2 hardcode `apiBase: 'http://localhost:8000'` khiến nếu Backend chạy
   ở port/host khác thì App 2 luôn lỗi kết nối mặc định — đã đổi sang
   `window.location.origin`.
7. Backend chưa từng phục vụ App 1 / App 2 (chỉ có route `/` cho Dashboard
   của chính nó) dù giao diện Dashboard đã có sẵn nút "Open App 1/App 2" trỏ
   tới `/app1`, `/app2` — đã bổ sung mount `StaticFiles` cho 2 route này.
8. Bổ sung `PATCH /api/project/{id}` (alias của PUT) theo đúng danh sách
   API yêu cầu.

Toàn bộ các sửa đổi trên đều **không thay đổi UI/UX/tính năng** hiện có —
chỉ khắc phục lỗi và kết nối đúng 3 thành phần lại với nhau.
