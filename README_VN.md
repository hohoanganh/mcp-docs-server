<div align="center">

![Repo Traffic](https://komarev.com/ghpvc/?username=mcp-docs-server&label=Repo+Traffic&color=blue&style=flat-square)

</div>

# AK Active Kernel documentation MCP server

| [EN](README.md) | VN |

**AK-MCP** được xây dựng dựa trên **[Model Context Protocol](https://modelcontextprotocol.io)** , cung cấp cho các AI Agent lập trình - khả năng truy cập và truy vấn tri thức chính xác về **AK (Active Kernel) Framework**.

MCP Server này giúp AI agent của bạn:

- Hiểu rõ cấu trúc của AK kernel (scheduler, message pools, timers, FSM/TSM),
- Tra cứu **cấu trúc API** một cách nhanh chóng (sử dụng trực tiếp các file header để tránh trôi API),
- Tuân thủ các **quy luật** tạo task, driver và màn hình,
- Thiết kế các task/driver mới mà **không cần** chạm vào kernel, boot, sys, network hay common code.

**(*) Hiểu rõ sản phẩm là cốt lõi của thành công !**

[<img src="images/ak-active-kernel-documentation-mcp-server-1280-640-px.png" width="960"/>](<https://epcb.vn/products/ak-embedded-base-kit-lap-trinh-nhung-vi-dieu-khien-mcu>)

## Cách hoạt động

Repo này là một repo **độc lập**. Các file header được chứa trong `vendor/ak-inc/` nên không cần tải thêm gì để sử dụng.

```
vendor/ak-inc/*.h  ──────────────► scripts/extract.mjs ─┐   (snapshot of the kernel
  ▲ refreshed by                                          │    headers; refresh with
  scripts/fetch-headers.mjs (GitHub)                      ├─►  npm run fetch-headers)
corpus/ (hand-written guides,        scripts/build-corpus ┘─► generated/corpus.json
         guardrails, enrichment) ───────────────────────────►      (docs + BM25 index)
                                                                      │
                                          src/core (resources + tools + prompts)
                                          ├── src/worker  ->  Cloudflare Worker (remote HTTP)
                                          └── src/cli     ->  npx ak-mcp (stdio, local)
```

Những file header của kernel chứa các định nghĩa hàm cần dùng. Cách sử dụng được lưu trong `/corpus/enrichment`. CI sẽ kiểm tra xem định nghĩa hàm có bị lệch khỏi quy chuẩn ban đầu hay không.

## Bộ lệnh

**Tools**

| Tool | Mục Đích |
| --- | --- |
| `start_ak_project(project_name?, ref?)` | tải bản mới nhất của AK Base Kit và trả về một thiết kế theo yêu cầu |
| `search_ak_docs(query, section?, limit?)` | tra cứu toàn bộ dự án dựa trên thuật toán BM25 |
| `get_ak_api(symbol)` | trả về chĩnh xác định nghĩa hàm, tham số, kết quả trả về và mã lỗi |
| `list_ak_api(module?)` | tra cứu API theo module (task/message/timer/fsm/tsm/ak/port) |
| `get_ak_guide(topic)` | trả về hướng dẫn xây dựng dự án như: start-project, create-task, create-driver, create-screen, use-timer, isr-bridge, tune-pools, **debug-uart-shell**, **kernel-task-log**, **agent-workflow** |
| `get_ak_guardrails()` | những vùng agent không được chạm đến |
| `analyze_ak_log(log, context?)` | phân tích lỗi UART |
| `decode_ak_lcd(dump, scale?, invert?)` | lấy hình ảnh hiện tại trên màn hình LCD |

**Prompts:** `ak-new-project`, `ak-new-task`, `ak-new-driver`, `ak-debug` - Hướng dẫn xây dựng/debug theo quy tắc AK.

**Debugging loop:** Debug chương trình thông qua cổng UART 115200 baud thông qua
[`examples/ak-console.py`](examples/ak-console.py) (cho phép agent thực hiện những tác vụ có tính thay đổi lớn bằng cách thêm `--allow-destructive`), rồi đưa cho `analyze_ak_log`.

**Resources:** `ak://index` và `ak://{section}/{id}` chứa tất cả cơ sở tri thức MCP cần để hoạt động.

## Kernel headers

Đọc từ header của AK kernel ở trong `vendor/ak-inc/`
(snapshot của release tag của firmware nằm trong `vendor/ak-inc/SOURCE.txt`). **Không cần checkout firmware**

Khi kernel thay đổi, hãy refresh lại snapshot:

```sh
npm run fetch-headers            # pinned default tag (v1.3)
npm run fetch-headers v1.4       # a specific release tag
```

sau đó, chạy `npm run build:corpus` và commit `vendor/ak-inc/`. Nếu muốn checkout firmware:

1. `$AK_INC_DIR` - biến môi trường tới thẳng `.../application/sources/ak/inc`
2. `$AK_FIRMWARE_DIR/application/sources/ak/inc` - root folder của firmware
3. `vendor/ak-inc/` - snapshot mặc định

Sau khi `generated/corpus.json` được tạo ra, corpus đã hoàn thành.

## Develop

```sh
npm install
npm run build:corpus     # tạo /generated/corpus.json từ header + corpus/
npm run drift            # kiểm tra lệch
npm test                 # kiểm tra extraactor, corpus
npm run typecheck        # core + cli
```

Pipeline của corpus hoàn toàn độc lập và chạy trên NodeJS v20 trở lên.

## Run locally (stdio)

```sh
npm run build            # build:corpus + tsc -> dist/
node dist/cli/bin.js     # hoặc chạy trực tiếp
```

Kiểm tra với MCP Inspector:

```sh
npx @modelcontextprotocol/inspector node dist/cli/bin.js
```

Client config (Claude Desktop / Cursor):

```json
{ "mcpServers": { "ak": { "command": "npx", "args": ["-y", "ak-mcp"] } } }
```

## Triển khai

Worker gói `corpus.json` tại build time, nên không cần database.

```sh
npm run dev              # local Streamable HTTP at http://localhost:8787/mcp
npm run deploy           # build:corpus + wrangler deploy
```

Endpoints: `/mcp` (Streamable HTTP), `/sse` (legacy), `/` (landing page).

Remote client config:

```json
{ "mcpServers": { "ak": { "url": "https://ak-mcp.<your-account>.workers.dev/mcp" } } }
```

**Sử dụng VSCode (vibe coding):** xem [docs/vscode-vibe-coding.md](docs/vscode-vibe-coding.md)
để xem chi tiết từng bước setup AI (Copilot Agent mode, Cursor, Cline, Claude Code) với một template có sẵn
[`.vscode/mcp.json`](examples/vscode-mcp.json) và một file quản lý dự án
([`examples/copilot-instructions.md`](examples/copilot-instructions.md)).

CI (`.github/workflows/ak-mcp.yml`) build từ header, không cần checkout:
`verify` chạy build, check lệch, test và typecheck sau đó `deploy` từ `main` khi `CLOUDFLARE_API_TOKEN` và `CLOUDFLARE_ACCOUNT_ID` được thêm vào.  Job `refresh-headers`
(chạy bằng tay với `tag` tùy chọn, hoặc với option `repository-dispatch` với định nghĩa `firmware-updated` từ repo firmware) sẽ lấy từ `vendor/ak-inc/`, kiếm chứng, và cập nhật nếu có thay đổi.

## Thêm cơ sở tri thức

- **Kernel ra phiên bản mới** Chạy `npm run fetch-headers [<tag>]` để refresh `vendor/ak-inc/`, sau đó chạy `npm run build:corpus` và lưu snapshot. Các định nghĩa mới sẽ được cập nhật.
- **Bổ sung hướng dẫn cho agent** Thêm `corpus/enrichment/<symbol>.md` để đưa chỉ dẩn cho agent.
- **Thêm công thức hoặc khái niệm mới** Thêm markdown vào trong `corpus/guides/` hoặc `corpus/concepts/` với chi tiết cần thêm (`id`, `title`, `tags`, `summary`, optional `apis`).
- Chạy `npm run drift` để kiểm chứng.

Format Markdown:

```markdown
---
symbol: timer_set            # enrichment only
summary: One-line summary.
fatal_codes: MT:0x30
see_also: timer_remove_attr, timer_tick
tags: timer, periodic
---
Markdown body (semantics, examples) ...
```
