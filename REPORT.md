# Báo cáo Lab — MCP vs Function Calling & Weather Agent (Day26)

**Họ và Tên:** Trịnh Hoàng Nam \
**MSHV:** 2A202601376 \
**Ngày thực hiện:** 28/08/2026 \
**Repo:** `Day26-2A202601376-TrinhHoangNam`

---

## 1. Mục tiêu

Repo gồm 4 phần minh hoạ tiến trình từ Function Calling thuần → MCP cơ bản → MCP production-ready → một agent thực tế (Weather Agent) dùng Google ADK làm MCP client, kết nối MCP server qua Streamable HTTP. Mục tiêu của báo cáo này là ghi lại quá trình chạy thử, kiểm chứng và các sự cố đã xử lý cho cả 4 phần, bao gồm phần mở rộng (bonus): đóng gói MCP server bằng Docker.

## 2. Kiến trúc tổng thể

```
01-function-calling   02-mcp-basics        03-production              04-lab (bài lab chính)
─────────────────    ─────────────────    ────────────────────     ─────────────────────────────
App tự định nghĩa     Server tự công bố    + Auth (Bearer token)     ADK Agent (MCP client)
schema + thực thi     tool qua MCP,        + Tool Registry           ── Streamable HTTP ──▶ MCP Server
hàm trong 1 file       client list_tools/  + Versioning (v1/v2)      (FastMCP, weather.py)
(Gemini SDK)           call_tool (stdio)                             ── REST ──▶ WeatherAPI.com
```

Kiến trúc Lab 04 (phần chính):

```
┌─────────────────┐   Streamable HTTP    ┌─────────────────┐      REST       ┌─────────────────┐
│   ADK Agent     │ ──────────────────── │   MCP Server    │ ─────────────── │  WeatherAPI.com │
│  (mcp-client)   │   localhost:8085/mcp │  (mcp-server)   │                 │                 │
└─────────────────┘                      └────────────────┘                 └─────────────────┘
```

Tool cung cấp: `get_current_weather(city)`, `get_forecast(city, days)`, `health_check()`.

## 3. Môi trường thực thi

- OS: Windows, PowerShell
- Python: 3.11 (venv riêng cho 01–03), Python 3.12 cho `04-lab` (qua `uv`)
- Package manager: `pip` (venv, cho 01–03) và `uv` (cho `04-lab`)
- Docker Desktop (cho phần bonus containerization)
- API key: Gemini API key (dùng chung tên biến `GOOGLE_API_KEY` cho cả `google-genai` SDK và Google ADK), WeatherAPI.com key

## 4. Kết quả kiểm thử

### 4.1. `01-function-calling` — Function Calling thuần (Gemini SDK)

Lệnh chạy: `python weather_function_calling.py`

Kết quả: model nhận câu hỏi "Thời tiết Hà Nội và Đà Nẵng hôm nay thế nào?", tự quyết định gọi `get_weather` 2 lần (Hà Nội, Đà Nẵng) theo đúng cơ chế Function Calling — app tự thực thi hàm và trả kết quả JSON lại cho model tổng hợp thành câu trả lời tự nhiên (kèm emoji, lời khuyên theo `SYSTEM_INSTRUCTION`). **Đạt.**

### 4.2. `02-mcp-basics` — MCP server + client qua stdio

Lệnh chạy: `python weather_client.py`

Kết quả khớp 100% với output mẫu trong README: `list_tools()` phát hiện đúng tool `get_weather`; `call_tool()` cho 3 thành phố (Hanoi, Danang, Haiphong) trả kết quả đúng định dạng. **Đạt.**

### 4.3. `03-production` — Auth, Tool Registry, Versioning

**3a. Auth (Bearer token qua HTTP):** `python auth_server.py` (terminal 1) + `python auth_client.py` (terminal 2). Client có token kết nối thành công, gọi được `get_weather` qua HTTP. **Đạt.**

**3b. Tool Registry & Discovery:** `python registry_client.py`. Registry liệt kê đúng 4 tool (`get_weather`, `get_weather_v2`, `send_email`, `query_db`); tìm theo tag `weather` ra 2 kết quả, chọn best match `get_weather_v2`; tìm theo keyword `forecast` cũng định. **Đạt** (sau khi sửa lỗi, xem mục 6).

**3c. Versioning & Backward Compatibility:** `python versioned_client.py`. Đọc đúng resource `server://info` (version, deprecated tools, migration guide); gọi được cả `get_weather` (v1, deprecated) lẫn `get_weather_v2` (v2, có forecast + units). **Đạt.**

### 4.4. `04-lab` — Weather Agent (ADK + MCP qua Streamable HTTP), chạy trực tiếp

Terminal 1: `uv run python weather.py` → server sống tại `http://0.0.0.0:8085/mcp`.
Terminal 2: `uv run adk web` → giao diện `http://localhost:8000`, chọn `weather_agent`.

Đã kiểm thử đủ cả 3 tool qua giao diện chat thật:

- `get_current_weather("Hanoi")` → trả đúng nhiệt độ, độ ẩm, gió, UV, thời gian cập nhật thực tế từ WeatherAPI.com.
- `get_forecast("Danang", 3)` → trả dự báo 3 ngày, đầy đủ nhiệt độ cao/thấp, tình trạng, khả năng mưa, UV.
- `health_check()` → trả đúng thông báo server đang hoạt động.

**Đạt** (sau khi xử lý các sự cố ở mục 6).

### 4.5. `04-lab` — Bonus: đóng gói MCP server bằng Docker

```bash
docker build -t weather-mcp .
docker run --rm -p 8085:8085 --env-file .env -e PORT=8085 weather-mcp
```

Container build thành công (5 layer, ~130s lần đầu do tải base image + cài dependency). Container chạy đúng chế độ Streamable HTTP (sau khi thêm `-e PORT=8085`, xem mục 6), lắng nghe `0.0.0.0:8085`. ADK agent (không cần đổi bất kỳ dòng code nào, vì container map ra cùng cổng host `8085`) tiếp tục gọi `get_current_weather` thành công, dữ liệu thời tiết thật trả về bình thường qua log Uvicorn (`POST /mcp` 200 OK) và qua giao diện chat. **Đạt** — chọn phương án Docker local thay vì deploy Google Cloud Run thật, theo thống nhất với người thực hiện (không cần tài khoản GCP/billing).

## 5. So sánh Function Calling vs MCP (rút ra từ thực nghiệm)

| Tiêu chí | Function Calling (01) | MCP (02–04) |
|---|---|---|
| Khai báo schema | Viết tay `FunctionDeclaration` (~15-30 dòng) | `@mcp.tool()` tự sinh từ type hints + docstring (4 dòng) |
| Nơi thực thi tool | Trong app gọi model (cùng file) | Trong MCP server độc lập, tách khỏi client |
| Khám phá tool | Hard-code danh sách tools | `list_tools()` khám phá tại runtime |
| Tái sử dụng | Copy schema + hàm sang app khác | Cắm client mới vào server có sẵn, không sửa gì |
| Vận chuyển | Trong tiến trình (in-process) | stdio (demo) hoặc HTTP (production), qua mạng được |
| Bảo mật | Không áp dụng (nội bộ) | Bearer token ở tầng transport, không đụng logic tool |
| Khạm phá tool nhiều server | Không có khái niệm | Tool Registry — tra theo tag/keyword, chọn best match |
| Tương thích ngược | Phải tự quản lý version | Tool mới song song, tham số optional, resource metadata công bố version |

Kết luận thực nghiệm: Function Calling là *cơ chế mô hình quyết định gọi tool*; MCP là *chuẩn giao thức kết nối client–server* dùng Function Calling bên dưới. ở quy mô 1 app, Function Calling đơn giản hơn; khi cần dùng lại tool ở nhiều client/nhiều server, MCP giảm hẳn chi phí bảo trì (xác nhận qua ví dụ Tool Registry — agent tự chọn `get_weather_v2` mà không cần biết trước server nào cung cấp).

## 6. Sự cố gặp phải & cách xử lý

| # | Vấn đề | Nguyên nhân | Cách xử lý |
|---|---|---|---|
| 1 | `404 NOT_FOUND: model models/gemini-2.5-flash is no longer available` khi chạy `01-function-calling` và Lab 04 | Google đã khai tử model `gemini-2.5-flash` (chuyển trạng thái "Shut down" theo tài liệu chính thức) | Đổi `MODEL`/`model=` sang `gemini-3.6-flash` (đúng theo gợi ý trực tiếp từ lỗi API) tại `01-function-calling/weather_function_calling.py` và 2 vị trí trong `04-lab/mcp-client/weather_agent/agent.py` |
| 2 | `UnicodeDecodeError: 'charmap' codec can't decode byte...` khi chạy `registry_client.py` trên Windows | `open(path)` không chỉ định `encoding`, Python trên Windows mặc định dùng codepage hệ thống (cp1252) thay vì UTF-8, không đọc được ký tự tiếng Việt có dấu trong `registry.json` | Thêm `encoding="utf-8"` vào lệnh `open()` trong `registry_client.py` |
| 3 | `401 Unauthorized — API key is invalid` khi gọi WeatherAPI.com | (a) Key ban đầu không hợp lệ trên WeatherAPI.com; (b) nhầm lẫn giữa key WeatherAPI.com và OpenWeatherMap — hai dịch vụ độc lập, không dùng chung key | Đăng ký/lấy đúng key tại `weatherapi.com` (không phải openweathermap.org), cập nhật `04-lab/mcp-server/.env` |
| 4 | Đổi key mới trong `.env` nhưng vẫn lỗi cũ | MCP server (`weather.py`) chỉ đọc biến môi trường **một lần** lúc khởi động (`API_KEY = os.getenv(...)` chạy khi import module); sửa `.env` khi server đang chạy không có tác dụng | Dừng hẳn tiến trình server (`Ctrl+C`) và khởi động lại (`uv run python weather.py`) sau mỗi lần đổi key |
| 5 | `weather_function_calling.py` và `weather.py` ban đầu không tự đọc file `.env` | Chỉ `verify_setup.py` và Google ADK CLI tự động `load_dotenv()`; 2 file kia đọc thẳng `os.environ` | Thêm `from dotenv import load_dotenv; load_dotenv()` vào đầu 2 file (dùng `python-dotenv` đã có sẵn qua dependency `mcp[cli]`, không cần cài thêm) |
| 6 | Container Docker chạy nhưng ở **chế độ stdio**, không mở cổng HTTP | Logic cuối `weather.py` tự chọn transport dựa trên `os.getenv("PORT")` và `sys.stdin.isatty()`; `docker run` không có TTY và không set `PORT` → rơi vào nhánh stdio | Thêm `-e PORT=8085` vào lệnh `docker run` để bỉc server vào chế độ Streamable HTTP — đúng với thiết kế gốc dành cho môi trường Cloud Run (luôn set `PORT`) |

## 7. Danh sách file đã tạo/sửa

**File mới (không commit — đã nằm trong `.gitignore`):**
- `01-function-calling/.env`
- `04-lab/mcp-server/.env`
- `04-lab/mcp-client/.env`
- `PLAN_TASKSPEC_LAB04.md` (kế hoạch làm việc, không thuộc phạm vi bài nộp)
- `REPORT.md` (file này)

**File code đã sửa:**
- `01-function-calling/weather_function_calling.py` — thêm `load_dotenv()`, đổi model sang `gemini-3.6-flash`
- `04-lab/mcp-server/weather.py` — thêm `load_dotenv()`
- `04-lab/mcp-client/weather_agent/agent.py` — thêm `load_dotenv()`, đổi model sang `gemini-3.6-flash` (2 vị trí)
- `03-production/registry_client.py` — thêm `encoding="utf-8"` khi đọc `registry.json`

Không có file nào ngoài phạm vi trên bị chỉnh sửa.

## 8. Kết luận

Cả 4 phần của repo đã được chạy thực tế và kiểm chứng thành công, bao gồm:
- 3 demo nền tảng (Function Calling thuần, MCP cỐ bản, MCP production với Auth/Registry/Versioning).
- Lab chính: Weather Agent dùng Google ADK làm MCP client, kết nối MCP server qua Streamable HTTP, gọi được cả 3 tool với dữ liệu thời tiết thật.
- Phần mở rộng (bonus): đóng gói MCP server bằng Docker, chạy độc lập trong container mà agent vẫn kết nối và hoạt động đúng — không cần sửa code client.

Toàn bộ sự cố phát sinh trong quá trình chạy (model bị khai tử, lỗi encoding trên Windows, nhầm lẫn nhà cung cấp API thời tiết, hành vi cache biến môi trường, và transport detection trong container) đều đã được xác định nguyên nhân gốc rễ và xử lý, có ghi log làm bằng chứng ở mục 4 và 6.

**Hướng mở rộng nếu muốn làm thêm:** deploy MCP server lên Google Cloud Run thật (Dockerfile đã sẵn sàng cho việc này), thêm CI để tự động build/test, hoặc bổ sung thêm tool MCP mới (ví dụ cảnh báo thời tiết) để minh hoạ thêm tính mở rộng của kiến trúc MCP so với Function Calling thuần.
