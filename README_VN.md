<div align="center">

![Repo Traffic](https://komarev.com/ghpvc/?username=mcp-docs-server&label=Repo+Traffic&color=blue&style=flat-square)

</div>

# Tài Liệu Nghiên Cứu AK Active Kernel MCP Server

| [EN](README.md) | VN |

Một cấu hình server mã nguồn mở, miễn phí dựa trên **[Model Context Protocol](https://modelcontextprotocol.io)** dành cho tác vụ lập trình cùng AI trên **AK (Active Kernel) Framework**.

Server MCP này giúp AI agent của bạn:

- Hiểu rõ cấu trúc của AK kernel (scheduler, message pools, timers, FSM/TSM),
- Tra cứu **cấu trúc API** một cách nhanh chóng (sử dụng trục tiếp các file header để tránh trôi API),
- Tuân thủ các **quy luật** tạo task, driver và màn hình,
- Thiết kế các task/driver mới mà **không cần** chạm vào kernel, boot, sys, network hay code.

**(*) Hiểu rõ sản phẩm là cốt lõi của thành công !**

[<img src="images/ak-active-kernel-documentation-mcp-server-1280-640-px.png" width="960"/>](<https://epcb.vn/products/ak-embedded-base-kit-lap-trinh-nhung-vi-dieu-khien-mcu>)

## Cách hoạt động

Repo này là một repo **rời**. Các file header được chứa trong `vendor/ak-inc/` nên không cần clone thêm gì để sử dụng.

```
vendor/ak-inc/*.h  ──────────────► scripts/extract.mjs ─┐   (snapshot of the kernel
  ▲ refreshed by                                          │    headers; refresh with
  scripts/fetch-headers.mjs (GitHub)                      ├─►  npm run fetch-headers)
corpus/ (hand-written guides,        scripts/build-corpus ┘─► generated/corpus.json
         guardrails, enrichment) ───────────────────────────►      (docs + BM25 index)
                                                                      │
                                          src/core (resources + tools + prompts)
                                          ├── src/worker  →  Cloudflare Worker (remote HTTP)
                                          └── src/cli     →  npx ak-mcp (stdio, local)
```

Những file header của kernel chứa các định nghĩa hàm cần dùng. Cách sử dụng được lưu trong `/corpus/enrichment`. CI sẽ kiểm tra xem định nghĩa hàm có bị lệch khỏi chuẩn ban đầu hay không.

## Bộ lệnh

**Tools**

| Tool | Mục Đích |
| --- | --- |
| `start_ak_project(project_name?, ref?)` | cập nhật bản mới nhất của kit và trả về một kế hoạch |
| `search_ak_docs(query, section?, limit?)` | tra cứu toàn bộ dự án dựa trên thuật toán BM25 |
| `get_ak_api(symbol)` | trả về chĩnh xách định nghĩa hàm, tham số, kết quả trả về và mã lỗi |
| `list_ak_api(module?)` | tra cứu API theo module (task/message/timer/fsm/tsm/ak/port) |
| `get_ak_guide(topic)` | trả về một số hướng dẫn xây dựng dự án như: start-project, create-task, create-driver, create-screen, use-timer, isr-bridge, tune-pools, **debug-uart-shell**, **kernel-task-log**, **agent-workflow** |
| `get_ak_guardrails()` | những vùng agent không được chạm đến |
| `analyze_ak_log(log, context?)` | phân tích lỗi UART |
| `decode_ak_lcd(dump, scale?, invert?)` | lấy hình ảnh hiện tại trên màn hình LCD |

**Prompts:** `ak-new-project`, `ak-new-task`, `ak-new-driver`, `ak-debug` - Hướng dẫn xây dựng/debug theo chuẩn dự án.

**Debugging loop:** Debug chương trình thông qua cổng UART 115200 baud thông qua
[`examples/ak-console.py`](examples/ak-console.py) (cho phép agent thực hiện những tác vụ có tính thay đổi lớn bằng cách thêm `--allow-destructive`), rồi đưa cho `analyze_ak_log`.

`start_ak_project` tra cứu API của AK Framework trên GitHub qua trang "latest release" (sẽ sử dụng `v1.3` nếu API không thể truy cập được).

**Resources:** `ak://index` và `ak://{section}/{id}` chứa tất cả nội dung MCP cần để hoạt động.

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

## Thêm tài liệu

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

## Xây dựng cơ sở tri thức MCP cho AI Agent

Ai từng giao cho AI Agent viết code trong dự án lớn đều gặp cùng một kịch bản: Agent sinh ra vài trăm dòng, chạy được nhưng không theo style và quy tắc của team. Diff dài quá, ta lười đọc, bấm accept. Lần sau Agent viết nhiều hơn, ta lười hơn và nó lệch dần khỏi hướng mình muốn. Kết cục là mất niềm tin: test sơ qua thấy chạy đúng, nhưng trong lòng vẫn không chắc.

Nguyên nhân thường không nằm ở model, mà ở chỗ Agent không biết gì về thế giới của bạn. Những quy tắc cả team đều thuộc lòng lại chưa bao giờ được viết ra ở dạng Agent đọc được. Vậy nên việc cần làm là xây cơ sở tri thức cho Agent, bằng SKILLS, bằng MCP, hoặc cả hai.

### MCP cung cấp gì cho Agent

MCP có thể triển khai online qua một domain, hoặc offline qua CLI chạy ngay trên máy. Dù cách nào, nó mang lại cho Agent:

- **Style của codebase**: cách đặt tên, cấu trúc thư mục, quy ước comment, format.
- **Quy tắc phải tuân theo**: những gì team đã thống nhất, không cần bàn lại mỗi lần.
- **Hướng dẫn triển khai**: công thức từng bước để thêm một module, một driver, một tính năng.
- **Giới hạn và điều cấm**: thư mục read-only, API deprecated, chỗ Agent không được tự ý sửa.
- **Quy trình bắt buộc**: tự debug, nạp code chạy test thật, commit xong từng tính năng rồi mới đi tiếp.

Cấu trúc gồm ba phần: **Tools** (hành động Agent gọi được: build, nạp code, đọc log UART, chạy test), **Resources** (kho tri thức tra cứu) và **Prompts** (quy trình đóng gói sẵn).

Trong đó Tools tạo khác biệt lớn nhất. Một Agent tự build được, tự đọc được log lỗi, tự chạy được test là Agent có vòng phản hồi khép kín, thay vì viết code rồi ngồi đoán là nó đúng.
Vì sao phải hiểu thật rõ framework
Đây là phần không có đường tắt, và cũng là phần quyết định thành bại.

MCP chỉ tốt đúng bằng tri thức bạn nạp vào. Muốn viết ra được quy tắc, bạn phải biết quy tắc nào thật sự quan trọng và quy tắc nào chỉ là thói quen. Muốn đặt giới hạn, bạn phải biết chỗ nào trong framework là nhạy cảm, sửa vào là hỏng. Muốn viết hướng dẫn triển khai, bạn phải nắm được con đường đúng để thêm một tính năng, chứ không phải con đường đầu tiên nghĩ ra.

Nói cách khác, **xây MCP thực chất là quá trình biến tri thức ngầm của team thành tri thức tường minh**. Phần lớn công sức nằm ở đó, không nằm ở code của MCP server.
Một ưu điểm khác: khi tri thức đã nằm sẵn trong MCP, Agent không cần "thông minh" để đoán ra style của bạn nữa, chỉ cần đọc và làm theo. Bạn có thể hạ xuống model rẻ hơn mà vẫn yên tâm về chất lượng.

Đầu tư vào cơ sở tri thức, suy cho cùng, là đầu tư vào niềm tin. Khi đã tin được, bạn mới thật sự bắt đầu đi nhanh.
