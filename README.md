# Bài nộp Day 26 — Võ Doanh Nhân (MSSV 2A202602770)

Repo cá nhân `K4-DAY26-VoDoanhNhan-2A202602770`, fork từ `VinUni-AI20k/Day26-MCP-Tools-Integration`.
Chạy trên Windows 11, Python 3.12.10 (Windows PowerShell 5.1 / Git Bash); MCP SDK 2.3.0 (bước 01–03), `uv` 0.13.1 (Lab 04), Google ADK 2.3.0.
Ngày chạy: 14/10/2026. Không có API key nào trong repo hay trong ảnh (`.env` đã bị gitignore).

| Bước | Ảnh | Kết quả chính |
|---|---|---|
| 1 Function Calling | `screenshots/01-function-calling.png` | 2 cặp `[model yêu cầu]` / `[app thực thi]` (Hà Nội, Đà Nẵng) và câu trả lời của Gemini |
| 2 MCP client (stdio) | `screenshots/02-mcp-client.png` | `list_tools()` thấy `get_weather` + 3 lần `call_tool`, khớp output mẫu |
| 3a Auth | `screenshots/03a-auth-ok.png`, `03a-auth-fail.png` | Token mặc định: có tool và kết quả; token sai: server trả `401 Unauthorized`, client không lấy được tool |
| 3b Registry | `screenshots/03b-registry.png` | Tìm thấy 2 tool tag `weather`, best match `get_weather_v2 v2.0.0` |
| 3c Versioning | `screenshots/03c-versioning.png` | `weather-v2 v2.0.0`, deprecated `['get_weather']`, v1 trả chuỗi, v2 trả JSON `api_version 2.0` |
| 4 Server + verify | `screenshots/04-verify-setup.png` | Server WeatherAPI ở `:8085/mcp`; `verify_setup.py`: ✅ All checks passed! |
| 5 ADK agent | `screenshots/04-adk-current.png`, `04-adk-forecast.png` | Brisbane hiện tại (cảm giác như, độ ẩm, UV, last updated) và dự báo 3 ngày Hà Nội, đều qua tool MCP |

Ghi chú trung thực về cách làm:

- **Ảnh terminal (01–03, 04-verify-setup)** là ảnh chụp thật cửa sổ Windows PowerShell (conhost) đang chạy đúng lệnh
  trên ảnh, chụp bằng Windows API `PrintWindow`. `PowerShell -Command` không tự in lệnh đang chạy, nên dòng
  `(.venv) PS ...> python ...` đầu tiên được in ra bằng `Write-Host (prompt)` ngay trước lệnh thật; dòng prompt cuối là prompt
  thật của PowerShell. Chỉ chỉnh cách hiển thị: cỡ chữ Consolas 14 cho cửa sổ đó, và bật hiển thị màu ANSI để log
  uvicorn không hiện ký tự điều khiển.
- Ảnh nhiều phần là các ảnh chụp thật của cùng một lần chạy được ghép dọc: `03a-auth-ok` (client + cửa sổ server),
  `03a-auth-fail` (client cuộn lên đầu + cuộn xuống cuối, vì traceback dài hơn màn hình + cửa sổ server có dòng
  `401 Unauthorized`), `04-verify-setup` (cửa sổ server + cửa sổ `verify_setup.py`). Trong `04-verify-setup`, phần đầu key
  Gemini mà `verify_setup.py` in ra đã được che.
- **Ảnh ADK (04-adk-current, 04-adk-forecast)** là ảnh chụp thật giao diện `adk web` (Chrome headless mở đúng phiên chat
  vừa hỏi).
- **Sửa 1 dòng code:** `03-production/registry_client.py` mở `registry.json` không chỉ định encoding, nên trên Windows
  (mặc định cp1252) báo `UnicodeDecodeError`. Đã thêm `encoding="utf-8"`; không đổi logic.
- **Server bước 4:** ảnh `04-verify-setup` chụp server chạy trong cửa sổ PowerShell thật (stdin là terminal nên server tự
  chọn Streamable HTTP, in đủ ba dòng mong đợi). Lúc chụp hai ảnh ADK, server được chạy nền với `PORT=8085`, vì khi
  không có terminal tương tác thì `weather.py` chỉ chọn Streamable HTTP nếu có biến `PORT`.
- **Câu hỏi dự báo:** câu mẫu "3-day forecast for Tokyo" bị agent từ chối, vì docstring của `get_forecast` chỉ liệt kê
  các thành phố ví dụ (Hanoi, Haiphong, Danang, Brisbane, Sydney, Melbourne) và Gemini hiểu đó là danh sách giới hạn. Giữ
  nguyên code, hỏi dự báo cho Hà Nội (đề cho phép).
- Phần tùy chọn Cloud Run (`05-cloud-run.png`) không làm.

---

# Phân biệt MCP và Function Calling

Đây là hai khái niệm hay bị nhầm lẫn nhưng thực ra ở **hai tầng khác nhau**, và **bổ sung cho nhau** chứ không thay thế.

## Cấu trúc repo

```
day26-mcp/
├── README.md                ← Bạn đang đọc file này
├── requirements.txt         ← pip install -r requirements.txt
│
├── 01-function-calling/     ← Bước 1: Function Calling thuần (Gemini SDK)
│   ├── README.md
│   └── weather_function_calling.py
│
├── 02-mcp-basics/           ← Bước 2: MCP server + client (không cần API key)
│   ├── README.md
│   ├── weather_server.py
│   └── weather_client.py
│
└── 03-production/           ← Bước 3: Auth, Tool Registry, Versioning
    ├── README.md
    ├── auth_server.py
    ├── auth_client.py
    ├── registry.json
    ├── registry_client.py
    ├── versioned_server.py
    └── versioned_client.py
│
└── 04-lab/                  ← Bước 4: Agent Google ADK + MCP server thật (WeatherAPI)
    ├── README.md
    ├── mcp-server/          ← FastMCP server (Streamable HTTP, Dockerfile cho Cloud Run)
    └── mcp-client/          ← ADK agent dùng McpToolset
```

> Bước 01–03 dùng **MCP Python SDK v2** (`mcp>=2.0`, xem `requirements.txt`).
> Lab 04 có môi trường riêng (`uv sync` trong từng thư mục).

## Quick start

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# MCP demo (không cần API key)
cd 02-mcp-basics && python weather_client.py

# Function Calling (cần Gemini API key)
export GEMINI_API_KEY=...
cd 01-function-calling && python weather_function_calling.py

# Production — Auth (2 terminal)
cd 03-production
python auth_server.py              # terminal 1
python auth_client.py              # terminal 2

# Production — Tool Registry
cd 03-production && python registry_client.py
```

---

## Định nghĩa ngắn gọn

**Function Calling** là một *khả năng của model* (capability). Model được huấn luyện để khi bạn đưa cho nó danh sách các "công cụ" (kèm schema mô tả tham số), nó có thể tự quyết định gọi công cụ nào và sinh ra JSON tham số phù hợp. Bản thân model **không chạy** function — nó chỉ nói "hãy gọi `get_weather(city='Hanoi')`". App mới là nơi chạy tool.

**MCP (Model Context Protocol)** là một *giao thức chuẩn* (protocol) — giống như USB-C hay HTTP cho thế giới AI. Nó định nghĩa cách một **MCP Client** (như Claude Code, Claude Desktop) kết nối tới các **MCP Server** để khám phá và sử dụng tools, resources, prompts một cách thống nhất.

---

## So sánh trực tiếp

| Tiêu chí | Function Calling | Model Context Protocol (MCP) |
|---|---|---|
| **Bản chất** | Tính năng của mô hình (Model capability) | Giao thức giao tiếp client–server |
| **Ai định nghĩa tool?** | Bạn hard-code trong từng app | Server tự công bố (self-describe) tool |
| **Tái sử dụng** | Phải viết lại cho mỗi app/model | Viết 1 lần, mọi MCP client dùng được |
| **Thực thi** | App của bạn tự chạy | MCP Server chạy, client điều phối |
| **Tính chuẩn hóa** | Mỗi nhà cung cấp 1 kiểu (OpenAI, Anthropic khác nhau) | Một chuẩn chung do Anthropic đề xuất |
| **Hệ sinh thái** | Khó chia sẻ dạng module đóng gói sẵn | Dễ dàng chia sẻ và tải về các "MCP Servers" mã nguồn mở |

## Quan hệ giữa chúng

Điểm quan trọng nhất: **MCP dùng Function Calling bên dưới**. Chúng không loại trừ nhau.

```
User hỏi
   │
   ▼
LLM (dùng Function Calling để quyết định gọi tool nào)
   │
   ▼
MCP Client  ──[giao thức MCP]──►  MCP Server (thực thi tool thật)
   │                                   │
   ◄───────────── kết quả ─────────────┘
   ▼
LLM tổng hợp câu trả lời
```

## Khi nào dùng cái nào?

- **Function Calling thuần**: app đơn giản, tool gắn chặt với 1 ứng dụng, không cần chia sẻ.
- **MCP**: muốn tool/tích hợp dùng lại được trên nhiều AI client, muốn tách biệt logic tool khỏi app, hoặc xây hệ sinh thái tích hợp (DB, file, API nội bộ...).

---

## Minh hoạ bằng mã nguồn

Cùng một tool `get_weather`, dưới đây là hai cách triển khai để thấy rõ sự khác biệt.

### [Cách 1 — Function Calling thuần (Google Gemini SDK)](01-function-calling/)

Tool được **định nghĩa và thực thi ngay trong app**. Model chỉ quyết định gọi tool nào, app tự chạy và đưa kết quả trở lại.

```
User hỏi → Model quyết định gọi get_weather("Hà Nội")
                    │
                    ▼
             App TỰ THỰC THI hàm get_weather
                    │
                    ▼
             Model tổng hợp câu trả lời
```

> Nhược điểm: schema viết tay, tool gắn chặt trong app — muốn dùng lại ở app khác phải copy cả schema lẫn hàm.

Chi tiết + code: xem [`01-function-calling/README.md`](01-function-calling/README.md)

### [Cách 2 — MCP (server tự công bố tool, mọi client dùng chung)](02-mcp-basics/)

Tool được tách ra **một MCP server độc lập**. Server tự "khai báo" nó có tool gì; bất kỳ MCP client nào (Claude Code, Claude Desktop, Cursor...) cũng cắm vào dùng được mà không cần biết code bên trong.

```
weather_client.py                       weather_server.py
┌─────────────┐    giao thức MCP    ┌─────────────────┐
│  list_tools │ ──────────────────▶ │ @mcp.tool()     │
│  call_tool  │ ◀────────────────── │ get_weather()   │
└─────────────┘     stdio           └─────────────────┘
```

Chi tiết + code: xem [`02-mcp-basics/README.md`](02-mcp-basics/README.md)

### Điểm khác biệt rút ra từ code

| | Function Calling thuần | MCP |
|---|---|---|
| Khai báo schema | Tự viết tay trong app | `@mcp.tool()` tự sinh từ type hints |
| Nơi thực thi tool | Trong app gọi model | Trong MCP server riêng |
| Khám phá tool | Hard-code danh sách `tools` | `session.list_tools()` tại runtime |
| Dùng lại ở app khác | Copy code | Cắm thêm client, không sửa server |
| Vai trò Function Calling | Là toàn bộ cơ chế | Là lớp model bên trong MCP |

---

## [MCP trong Production](03-production/)

Các ví dụ trên chạy tốt trên máy cá nhân, nhưng đưa vào **hệ thống production** cần giải quyết thêm ba vấn đề:

```
┌─────────────────────────────────────────────────────┐
│                  Production MCP                     │
│                                                     │
│  ┌──────────┐   ┌───────────┐   ┌───────────────┐   │
│  │ Security │   │ Registry  │   │  Versioning   │   │
│  │          │   │           │   │               │   │
│  │ • Auth   │   │ • Discover│   │ • v1 compat   │   │
│  │ • Token  │   │ • Connect │   │ • v2 features │   │
│  │ • Scopes │   │ • Health  │   │ • Deprecation │   │
│  └──────────┘   └───────────┘   └───────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 1. Security — Authentication & Authorization

MCP server phục vụ qua **HTTP** cho nhiều client → cần xác thực. MCP SDK hỗ trợ sẵn **Bearer Token** verification:

- Server: cấu hình `AuthSettings` + implement `TokenVerifier` protocol
- Client: gửi header `Authorization: Bearer <token>` qua `create_mcp_http_client(headers=...)` của MCP SDK
- Không có token → 401, token sai → 403, logic tool không biết gì về auth

| Tầng | Demo (stdio) | Production (HTTP) |
|---|---|---|
| Transport | stdio (cùng máy) | Streamable HTTP (qua mạng) |
| Auth | Không cần | Bearer token / OAuth / mTLS |
| Phạm vi truy cập | Toàn bộ | Scopes giới hạn từng client |

### 2. Tool Registry & Discovery

Agent **không hard-code** tool nào. Nó hỏi **Tool Registry** — danh mục trung tâm liệt kê tất cả tool từ mọi server — theo yêu cầu task:

```
Agent nhận task "lấy thời tiết Hà Nội"
   │
   ▼
Tool Registry: "tool nào có tag 'weather'?"
   │
   ├── get_weather v1.0 → server: weather (stdio)
   └── get_weather_v2 v2.0 → server: weather-v2 (stdio)
   │
   ▼
Agent chọn best match (v2.0, không deprecated)
   │
   ▼
Kết nối tới server weather-v2, gọi get_weather_v2(city="Hanoi")
```

Registry là **tool-centric** — đơn vị khám phá là **tool** (tag, description, parameters), không phải server.

| | Hard-code (demo) | Tool Registry (production) |
|---|---|---|
| Agent biết tool nào? | Chỉ tool được code sẵn | Tất cả tool trong registry |
| Tìm tool | Theo tên cố định | Theo tag, keyword, capability |
| Thêm tool mới | Sửa code agent | Thêm entry vào registry |
| Chọn tool | Developer quyết định | Agent tự chọn best match |

### 3. Versioning & Backward Compatibility

Server v1 có `get_weather(city)` trả chuỗi đơn giản. V2 muốn trả JSON chi tiết, thêm `include_forecast`. Nếu đổi trực tiếp → mọi client cũ break. Giải pháp — 3 kỹ thuật kết hợp:

| Kỹ thuật | Mô tả |
|---|---|
| **Tool mới song song** | `get_weather_v2` tồn tại bên cạnh `get_weather` — không xoá tool cũ |
| **Tham số optional** | `include_forecast`, `units` có default → client cũ gọi vẫn đúng |
| **Server metadata** | Resource `server://info` công bố version + deprecation notice |

Chi tiết + code cho cả 3 phần: xem [`03-production/README.md`](03-production/README.md)

### Tổng kết Production Checklist

| Khía cạnh | Dev/Demo | Production |
|---|---|---|
| **Transport** | stdio (cùng máy) | HTTP/SSE (qua mạng) |
| **Auth** | Không | Bearer token, OAuth, mTLS |
| **Discovery** | Hard-code tool/server | Tool Registry — agent tìm tool theo task |
| **Versioning** | 1 tool duy nhất | Tool v1 + v2 song song, deprecation notice |
| **Health** | Không | Health check, retry, circuit breaker |
| **Logging** | `print()` | Structured logging, tracing (OpenTelemetry) |

---

**Tóm lại:** Function Calling là *cơ chế model gọi công cụ*; MCP là *chuẩn để kết nối model với các công cụ đó* — và MCP thực chất dùng Function Calling làm nền tảng để hoạt động.
