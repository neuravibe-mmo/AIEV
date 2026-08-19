# Hướng Dẫn Build & Triển Khai AI Edit Video (AIEV)

Tài liệu hướng dẫn chi tiết quy trình cài đặt, kiểm tra môi trường, build mã nguồn và khởi chạy dự án **AI Edit Video**.

---

## 1. Cấu trúc Monorepo & Công nghệ

Dự án được tổ chức theo mô hình **npm workspaces**:

```
AIEV/
├── apps/
│   ├── web/               # Next.js 16 (React 19, Tailwind CSS v4, Lucide Icons) - Port 6868
│   └── server/            # Node.js + Express + TypeScript + SQLite (better-sqlite3) - Port 6869
├── engines/
│   └── remotion/          # Remotion Engine (Composition, Assembler video)
├── video-projects/        # HyperFrames scene projects (HTML + GSAP)
├── start/                 # Bộ script tự động kiểm tra, build & khởi chạy
└── update/                # Bộ script cập nhật mã nguồn & migrations
```

---

## 2. Yêu cầu hệ thống (Prerequisites)

Trước khi build, hệ thống cần đáp ứng các điều kiện sau:

| Thành phần | Yêu cầu tối thiểu | Ghi chú |
|---|---|---|
| **Node.js** | `>= 22.0.0` | Bắt buộc (HyperFrames & Next.js 16 yêu cầu Node 22+) |
| **npm** | `>= 10.0.0` | Hỗ trợ quản lý workspace |
| **FFmpeg** | Có sẵn trên `PATH` | Xử lý video, audio, convert format |
| **Google Chrome** | Phiên bản mới nhất | Render headless cho HyperFrames & Remotion |
| **Python** *(Tùy chọn)* | `>= 3.10` | Dùng cho `faster-whisper` (transcribe) và `VieNeu-TTS` |

---

## 3. Kiểm tra môi trường (Environment Doctor)

Dự án tích hợp công cụ kiểm tra môi trường độc lập:

```bash
# Kiểm tra tình trạng môi trường hiện tại
node start/doctor.mjs

# Tự động phát hiện và hỏi cài đặt các thành phần còn thiếu
node start/doctor.mjs --fix

# Tự động cài đặt toàn bộ không cần hỏi lại
node start/doctor.mjs --fix --yes
```

---

## 4. Cài đặt Dependencies

Tại thư mục gốc của dự án:

```bash
# Cài đặt toàn bộ dependencies cho root và tất cả workspaces
npm install
```

> **Lưu ý:** Nếu gặp lỗi liên quan đến `better-sqlite3` khi đổi phiên bản Node.js, hãy chạy:
> ```bash
> npm rebuild better-sqlite3 -w apps/server
> ```

---

## 5. Cấu hình biến môi trường (`.env`)

Tạo file `.env` tại thư mục gốc (nếu chưa có):

```bash
cp .env.example .env
```

Các biến cấu hình quan trọng:

```env
# Port cấu hình
PORT=6869
WEB_PORT=6868

# Anthropic / Claude API (hoặc dùng đăng nhập Claude Code CLI)
ANTHROPIC_API_KEY=sk-ant-...

# Google Gemini API (dùng cho tạo ảnh AI, Gemini TTS)
GEMINI_API_KEY=AIzaSy...

# Cấu hình Whisper (nếu dùng local)
WHISPER_DEVICE=auto
```

---

## 6. Quy trình Build

### 6.1. Kiểm tra Typecheck (TypeScript)

Chạy kiểm tra kiểu trên toàn bộ workspaces (`apps/server`, `apps/web`, `engines/remotion`):

```bash
npm run typecheck
```

Hoặc typecheck từng workspace riêng:

```bash
npm run typecheck -w apps/server
npm run typecheck -w apps/web
npm run typecheck -w engines/remotion
```

### 6.2. Build toàn bộ dự án

Chạy lệnh build ở thư mục gốc:

```bash
npm run build
```

Lệnh này sẽ thực thi tuần tự:
1. `npm run build -w apps/server`: Biên dịch TypeScript (`tsc`) sang `apps/server/dist/`
2. `npm run build -w apps/web`: Build Next.js tối ưu cho Production sang `apps/web/.next/`

### 6.3. Quản lý Build Stamp

Dự án sử dụng cơ chế hash vân tay mã nguồn để phát hiện thay đổi giữa bản build và source:

```bash
# In mã hash stamp hiện tại
node start/build-stamp.mjs --print

# Lưu stamp sau khi build thành công
node start/build-stamp.mjs --save

# Kiểm tra bản build có khớp với source không (exit 0 nếu khớp, 1 nếu lệch)
node start/build-stamp.mjs --check
```

---

## 7. Khởi chạy ứng dụng

### 7.1. Chế độ Development (Hot-reload)

Dành cho lập trình viên cần chỉnh sửa code trực tiếp:

```bash
npm run dev
```

- **Web UI:** [http://localhost:6868](http://localhost:6868)
- **Backend API:** [http://localhost:6869](http://localhost:6869)
- Web UI tự động proxy các request `/api/*` và `/media/*` sang Backend.

### 7.2. Chế độ Production (Đã build)

Chạy bản build production bằng npm:

```bash
npm run start
```

Hoặc sử dụng các launcher tự động trong thư mục `start/`:

- **Windows:** Nhấp đúp vào `start/start.bat`
- **macOS:** Nhấp đúp vào `start/start.command` hoặc `./start/start.sh`
- **Linux:** `./start/start.sh`

Các script này sẽ tự động:
1. Kiểm tra Node.js >= 22
2. Chạy `doctor.mjs` kiểm tra thư viện/công cụ ngoài
3. Tự build lại nếu mã nguồn có thay đổi (dựa trên build-stamp)
4. Khởi chạy ngầm Server & Web và tự động mở trình duyệt.

### 7.3. Dừng ứng dụng

- **Windows:** Chạy `start/stop.bat`
- **macOS / Linux:** Chạy `start/stop.command` hoặc `./start/stop.sh`

---

## 8. Xử lý sự cố thường gặp (Troubleshooting)

| Vấn đề | Nguyên nhân | Cách khắc phục |
|---|---|---|
| `Node.js version mismatch` | Phiên bản Node < 22 | Cài đặt Node.js 22 LTS trở lên (qua `nvm`, `brew` hoặc `nodejs.org`). |
| `better-sqlite3 compile error` | Thiếu build tool C++ | Cài đặt `python` & `build-essential` (Linux) hoặc Visual Studio C++ Build Tools (Windows). |
| `Port 6868 / 6869 in use` | Tiến trình cũ chưa tắt | Chạy `./start/stop.sh` hoặc tìm kill PID qua `lsof -i :6868,6869`. |
| `Permission denied (start.sh)` | Chưa cấp quyền thực thi | Chạy `chmod +x start/*.sh start/*.command`. |
| `Tailwind / Next.js Turbopack error` | Cache build bị hỏng | Xóa thư mục `.next` trong `apps/web/` và chạy lại `npm run build`. |
