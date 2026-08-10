# 东方财富板块日线OHLC（MCP 工具 `ft_eastmoney_board_daily_kline`）

> **MCP 工具**：`ft_eastmoney_board_daily_kline`（category: `股票数据/打板专题数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取单个东方财富概念板块的历史日线行情，包括开盘价、收盘价、最高价、最低价、成交量、成交额、振幅、涨跌幅、涨跌额、换手率和市场代码。支持按日期范围筛选；不同板块的可用起始日期不同，较早板块可追溯至 2013 年。
- 数据范围：2013-04-09 至今，不同板块的起始日期可能不同
- 单次限量：start_date 与 end_date 同时给出时跨度 **≤3 天**；由 page/page_size 分页
- 提示：
  - `board_code` 必填。
  - 日期同时传时硬限 3 天跨度；不传日期则返回该板块全部历史。
  - 所有字段（含 market/volume 等）均按字符串返回。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| board_code | string | Y | 板块代码，如 BK1024 |
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
| data | array | Y | 归一化后的业务数据项 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| 市场 | string | Y |  |
| 开盘 | string | Y |  |
| 成交量 | string | Y |  |
| 成交额 | string | Y |  |
| 振幅 | string | Y |  |
| 换手率 | string | Y |  |
| 收盘 | string | Y |  |
| 日期 | string | Y |  |
| 最低 | string | Y |  |
| 最高 | string | Y |  |
| 板块代码 | string | Y |  |
| 板块名称 | string | Y |  |
| 涨跌幅 | string | Y |  |
| 涨跌额 | string | Y |  |

## 调用方法（MCP）

> MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

**curl**：

```bash
set -euo pipefail

MCP_BASE_URL="<MCP_BASE_URL>"

check_mcp_response() {
  local response=$1
  printf '%s\n' "$response"
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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_board_daily_kline","arguments":{"board_code":"BK0425","start_date":"2026-06-22","end_date":"2026-06-24","page":1,"page_size":2}}}')

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
                'ft_eastmoney_board_daily_kline',
                {'board_code': 'BK0425', 'start_date': '2026-06-22', 'end_date': '2026-06-24', 'page': 1, 'page_size': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

`board_code=BK0425, start_date=2026-06-22, end_date=2026-06-24, page=1, page_size=2` 返回 2 条记录。

| 市场 | 开盘 | 成交量 | 成交额 | 振幅 | 换手率 | 收盘 | 日期 | 最低 | 最高 | 板块代码 | 板块名称 | 涨跌幅 | 涨跌额 |
|------|------|------|------|------|------|------|------|------|------|------|------|------|------|
| 90 | 26227.98 | 38155212 | 34837293936 | 2.74 | 1.64 | 26314.72 | 2026-06-22 | 25613.24 | 26332.79 | BK0425 | 工程建设 | 0.12 | 31.56 |
| 90 | 26141.51 | 30737938 | 29215529688 | 2.13 | 1.32 | 26114.72 | 2026-06-23 | 25986.76 | 26546.5 | BK0425 | 工程建设 | -0.76 | -200 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
