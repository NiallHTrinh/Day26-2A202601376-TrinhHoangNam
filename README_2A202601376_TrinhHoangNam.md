# README (nháp) — MCP Weather Assistant

## 1. Use case đã chọn

**Trợ lý tra cứu & dự báo thời tiết (Weather Assistant).**

Lý do chọn use case này: dữ liệu thời tiết có API công khai miễn phí (WeatherAPI.com), input/output đơn giản (tên thành phố → số liệu thời tiết) nên dễ minh hoạ rõ cơ chế MCP (schema tự sinh, discovery, tool execution) mà không bị rối bởi domain logic phức tạp; đồng thời đủ phong phú để mở rộng dần qua từng giai đoạn của bài lab: từ 1 tool đơn giản → server độc lập qua MCP → thêm Auth/Registry/Versioning cho production → agent thực tế (Google ADK) dùng tool qua Streamable HTTP để trả lời câu hỏi thời tiết bằng ngôn ngữ tự nhiên.

Use case được xây dựng tiến hoá qua 4 giai đoạn trong repo, mỗi giai đoạn là một server MCP riêng phục vụ đúng mục đích minh hoạ của giai đoạn đó:

| Giai đoạn | Server | Vai trò |
|---|---|---|
| Cơ bản | `02-mcp-basics/weather_server.py` | 1 tool `get_weather`, transport stdio, không auth |
| Auth | `03-production/auth_server.py` | Cùng tool `get_weather`, transport HTTP + Bearer token |
| Versioning | `03-production/versioned_server.py` | `get_weather` (v1, deprecated) + `get_weather_v2` (v2), resource metadata version |
| Production (lab chính) | `04-lab/mcp-server/weather.py` | 3 tool thật, gọi WeatherAPI.com, transport Streamable HTTP, đóng gói Docker |

## 2. Các MCP tools đã xây

### Server chính — `04-lab/mcp-server/weather.py` (dữ liệu thời tiết thật từ WeatherAPI.com)

| Tool | Mô tả |
|---|---|
| `get_current_weather` | Lấy thời tiết hiện tại của một thành phố |
| `get_forecast` | Lấy dự báo thời tiết 1–3 ngày tới của một thành phố |
| `health_check` | Kiểm tra server đang hoạt động (dùng để verify deployment) |

### Server minh hoạ khác (giai đoạn cơ bản / auth / versioning, dữ liệu mock)

| Tool | Server | Mô tả |
|---|---|---|
| `get_weather` | `02-mcp-basics/weather_server.py` | Thời tiết hiện tại, trả chuỗi đơn giản (dữ liệu mock nội bộ) |
| `get_weather` | `03-production/auth_server.py` | Giống trên, chạy qua HTTP + yêu cầu bearer token |
| `get_weather` (v1) | `03-production/versioned_server.py` | Tool cũ, deprecated, vẫn hoạt động cho client legacy |
| `get_weather_v2` (v2) | `03-production/versioned_server.py` | Bản nâng cấp — trả JSON chi tiết, hỗ trợ forecast + đơn vị đo |

## 3. Input/Output của từng tool

### `get_current_weather(city: str) -> str`
*(server: `04-lab/mcp-server/weather.py`)*

- **Input:** `city` — tên thành phố, ví dụ `"Hanoi"`, `"Danang"`, `"Brisbane"`.
- **Output:** chuỗi text định dạng sẵn, gồm nhiệt độ (°C/°F), cảm giác như, tình trạng thời tiết, độ ẩm, gió (tốc độ + hướng), áp suất, chỉ số UV, tầm nhìn, thời điểm cập nhật. Ví dụ thực tế đã kiểm thử:
  ```
  Current Weather for Hanoi, , Vietnam:

  Temperature: 27.0°C (80.6°F)
  Feels like: 31.0°C (87.8°F)
  Condition: Patchy rain nearby
  Humidity: 90%
  Wind: 10.8 km/h (6.7 mph) S
  Pressure: 1002.0 mb
  UV Index: 1.0
  Visibility: 10.0 km

  Last updated: 2026-08-28 22:00
  ```
- Nếu thiếu `WEATHERAPI_KEY` hoặc gọi API thất bại → trả chuỗi thông báo lỗi thân thiện thay vì raise exception (để agent vẫn tổng hợp được câu trả lời).

### `get_forecast(city: str, days: int = 3) -> str`
*(server: `04-lab/mcp-server/weather.py`)*

- **Input:** `city` — tên thành phố; `days` — số ngày dự báo (1–3, tự động giới hạn tối đa 3 cho free tier WeatherAPI.com).
- **Output:** chuỗi text liệt kê từng ngày: ngày, nhiệt độ cao/thấp (°C/°F), tình trạng, khả năng mưa (%), tốc độ gió tối đa, chỉ số UV — các ngày cách nhau bằng `---`.

### `health_check() -> str`
*(server: `04-lab/mcp-server/weather.py`)*

- **Input:** không có tham số.
- **Output:** chuỗi cố định xác nhận server đang chạy, ví dụ: `"✅ Weather MCP Server is running! Ready to provide weather data for Australian cities and worldwide."`

### `get_weather(city: str) -> str`
*(server: `02-mcp-basics/weather_server.py`, `03-production/auth_server.py`)*

- **Input:** `city` — tên thành phố (dữ liệu mock có sẵn cho `Hanoi`, `Haiphong`, `Danang`; thành phố khác trả giá trị mặc định).
- **Output:** chuỗi đơn giản dạng `"{city}: {nhiệt độ}°C, {tình trạng}"`, ví dụ `"Hanoi: 29°C, trời mưa"`.

### `get_weather` (v1, deprecated) và `get_weather_v2`
*(server: `03-production/versioned_server.py`)*

| Tool | Input | Output |
|---|---|---|
| `get_weather(city)` | `city: str` | Chuỗi đơn giản, giống bản 02-mcp-basics — giữ để không break client cũ |
| `get_weather_v2(city, include_forecast=False, units="celsius")` | `city: str`; `include_forecast: bool` (mặc định `False`) — có kèm dự báo 2 ngày tới không; `units: str` — `"celsius"` hoặc `"fahrenheit"` | Chuỗi JSON: `api_version`, `city`, `temp`, `units`, `condition`, `humidity`, `wind_speed_kmh`, `timestamp`, và `forecast` (mảng 2 ngày) nếu `include_forecast=True` |

## 4. Cách chạy server

### Server chính (production, dữ liệu thật) — `04-lab/mcp-server`

```bash
cd 04-lab/mcp-server
uv sync
# Tạo file .env cùng thư mục với nội dung: WEATHERAPI_KEY=<key của bạn>
uv run python weather.py
```
Server lắng nghe tại `http://localhost:8085/mcp` (transport Streamable HTTP). Có thể đổi cổng bằng biến môi trường `PORT`.

**Chạy bằng Docker (đóng gói production):**
```bash
cd 04-lab/mcp-server
docker build -t weather-mcp .
docker run --rm -p 8085:8085 --env-file .env -e PORT=8085 weather-mcp
```
Lưu ý: cần set `PORT` khi chạy container để server tự nhận diện môi trường non-interactive và chạy chế độ HTTP thay vì stdio.

### Server minh hoạ cơ bản (mock, không cần API key) — `02-mcp-basics`

```bash
cd 02-mcp-basics
python weather_server.py   # chạy độc lập, mặc định transport stdio
# hoặc để client tự spawn:
python weather_client.py
```

### Server có Auth — `03-production/auth_server.py`

```bash
cd 03-production
python auth_server.py      # lắng nghe http://localhost:8000/mcp
```

### Server có Versioning — `03-production/versioned_server.py`

```bash
cd 03-production
python versioned_server.py   # chạy qua stdio, client tự spawn (versioned_client.py)
```

## 5. Cách đăng ký với Claude Code

**Server chạy qua stdio** (ví dụ `02-mcp-basics/weather_server.py`) — đăng ký 1 lần, dùng mãi:
```bash
claude mcp add weather -- python /đường/dẫn/tới/02-mcp-basics/weather_server.py
```

**Server chạy qua Streamable HTTP** (ví dụ `04-lab/mcp-server/weather.py`, đang chạy sẵn tại `localhost:8085`):
```bash
claude mcp add --transport http weather-mcp http://localhost:8085/mcp
```

**Server yêu cầu authentication** (ví dụ `03-production/auth_server.py`, đang chạy sẵn tại `localhost:8000`):
```bash
claude mcp add --transport http weather-secure http://localhost:8000/mcp \
  --header "Authorization: Bearer dev-token-abc123"
```

Kiểm tra sau khi đăng ký:
```bash
claude mcp get weather-mcp
```
hoặc gõ `/mcp` ngay trong phiên Claude Code để xem danh sách server đã kết nối và tool khả dụng.

## 6. Authentication

Áp dụng ở `03-production/auth_server.py` (server `get_weather` chạy qua HTTP thay vì stdio):

- Cơ chế: **Bearer token**, xác thực ở tầng transport thông qua `TokenVerifier` của MCP SDK (`AuthSettings` + `StaticTokenVerifier`) — logic tool (`get_weather`) hoàn toàn không biết gì về auth.
- Token hợp lệ (demo): `dev-token-abc123` (mặc định, có thể override bằng biến môi trường `MCP_AUTH_TOKEN`) hoặc `prod-key-xyz789`.
- Luồng: client gửi header `Authorization: Bearer <token>` → SDK gọi `verify_token()` → token đúng → trả về `AccessToken` (kèm `scopes=["weather:read"]`) → cho phép gọi tool. Token sai/thiếu → server trả `401`/`403`.
- Server chính của lab (`04-lab/mcp-server/weather.py`) **hiện chưa bật authentication** — mọi client biết URL đều gọi được. Đây là điểm có thể bổ sung nếu muốn đưa lên môi trường thật (áp dụng lại đúng cơ chế `TokenVerifier` như `auth_server.py`).

## 7. Versioning

Áp dụng ở `03-production/versioned_server.py`, minh hoạ 3 kỹ thuật giữ backward compatibility:

1. **Tool mới song song** — `get_weather_v2` tồn tại cạnh `get_weather` (v1), không xoá tool cũ nên client cũ không bị break.
2. **Tham số optional có default** — `get_weather_v2(city, include_forecast=False, units="celsius")`: client cũ gọi `get_weather_v2(city="Hanoi")` vẫn chạy đúng như trước khi có 2 tham số mới.
3. **Server metadata qua resource** — resource `server://info` công bố `version` (`"2.0.0"`), danh sách `deprecated_tools` (`["get_weather"]`), và `migration_guide` bằng văn bản để client tự biết nên chuyển sang tool nào.

Server chính của lab (`04-lab/mcp-server/weather.py`) hiện chỉ có 1 phiên bản tool (`get_current_weather`, `get_forecast`, `health_check`), chưa áp dụng versioning — có thể áp dụng lại đúng mẫu trên nếu sau này cần thay đổi schema mà vẫn giữ tương thích ngược.
