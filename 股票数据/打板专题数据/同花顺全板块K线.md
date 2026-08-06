# 同花顺全板块K线（MCP 工具 `ft_ths_all_board_kline`）

> **MCP 工具**：`ft_ths_all_board_kline`（category: `股票数据/打板专题数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：按日期范围查询同花顺概念、行业和地区板块的日 K 线，并按板块和日期分页返回。提示：`start_date` 和 `end_date` 可选，支持 YYYY-MM-DD 或 YYYYMMDD；支持分页。
- 数据范围：2007-08 至今（底层数据与单板块 K 线一致，行业类最早 2007-08-01）
- 单次限量：由 page/page_size 分页
- 提示：
  - 日期格式支持 `YYYY-MM-DD` 或 `YYYYMMDD`。
  - `start_date`/`end_date` 均可选；同时传入时结束日期不能早于开始日期，且跨度不超过 3 天（按日期差值计，超出返回 `INVALID_ARGUMENT "时间范围不能超过 3 天"`）。
  - 结果自动跳过 csrc 模块（无 K 线）。
  - 数值字段均为字符串。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| start_date | string | N | 起始日期（含），YYYY-MM-DD 或 YYYYMMDD |
| end_date | string | N | 截止日期（含），YYYY-MM-DD 或 YYYYMMDD |
| page | int | N | 页码，从 1 开始 |
| page_size | int | N | 每页数量 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

ThsBoardKlineRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| board_code | string | Y | 板块代码 |
| board_name | string | Y | 板块名称 |
| module | string | Y | 所属模块 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘 |
| high | string | Y | 最高 |
| low | string | Y | 最低 |
| close | string | Y | 收盘 |
| volume | string | Y | 成交量 |

## 调用方法（MCP）

> MCP 工具名 `ft_ths_all_board_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

**curl**：

```bash
set -euo pipefail

MCP_BASE_URL="<MCP_BASE_URL>"

check_mcp_response() {
  local response=$1
  printf '%s\n' "$response"
  # MCP 业务与协议错误仍可能使用 HTTP 200，必须检查 JSON-RPC 响应。
  if printf '%s\n' "$response" | grep -Eq '"isError"[[:space:]]*:[[:space:]]*true|"error"[[:space:]]*:[[:space:]]*\{'; then
    return 1
  fi
}

SID=$(curl -fsS -m 60 -D - -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"ftshare-doc-example","version":"1.0.0"}}}' \
  | awk 'tolower($1)=="mcp-session-id:" {print $2}' \
  | tr -d '\r')

if [ -z "$SID" ]; then
  printf '%s\n' 'initialize 未返回 Mcp-Session-Id' >&2
  exit 1
fi

curl -fsS -m 60 -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

CALL_RESPONSE=$(curl -fsS -m 60 -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_ths_all_board_kline","arguments":{"start_date":"2026-07-15","end_date":"2026-07-17","page":1,"page_size":2}}}')

check_mcp_response "$CALL_RESPONSE"
```

**Python（`mcp` SDK，自动握手管理 session）**：

```python
import asyncio

from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client


MCP_BASE_URL = "<MCP_BASE_URL>"


async def main():
    async with streamable_http_client(MCP_BASE_URL) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            result = await session.call_tool(
                'ft_ths_all_board_kline',
                {'start_date': '2026-07-15',
                 'end_date': '2026-07-17',
                 'page': 1,
                 'page_size': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

2026-06-13 至 2026-06-16 窗口（节选）：

| board_code | board_name | module | date | open | high | low | close | volume |
|------------|------------|--------|------|------|------|-----|-------|--------|
| 885768 | 无人零售 | concept | 2026-06-15 | 1282.071 | 1308.979 | 1277.81 | 1289.779 | 594369490 |
| 885768 | 无人零售 | concept | 2026-06-16 | 1290.717 | 1305.435 | 1268.564 | 1303.839 | 970300170 |
| 885744 | 雄安新区 | concept | 2026-06-15 | 1151.206 | 1169.115 | 1147.999 | 1156.548 | 7767410200 |
| 885744 | 雄安新区 | concept | 2026-06-16 | 1154.974 | 1164.136 | 1144.265 | 1161.025 | 6964113600 |
| 885951 | 预制菜 | concept | 2026-06-15 | 995.97 | 1007.696 | 983.701 | 990.517 | 2080070800 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
