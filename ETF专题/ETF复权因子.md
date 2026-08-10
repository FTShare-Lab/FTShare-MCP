# ETF复权因子（MCP 工具 `ft_etf_adjust_factor`）

> **MCP 工具**：`ft_etf_adjust_factor`（category: `ETF专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询 ETF 当日复权因子和截至当日累计复权因子。指定 `trade_date` 可查询单日快照，未指定 `symbol` 时覆盖全部标的；指定 `start_date` 与 `end_date` 可查询区间内各交易日数据，区间查询必须同时指定 `symbol`。
- 数据范围：每个标的仅返回最新交易日的复权因子快照，无历史序列
- 单次限量：分页返回，默认 `page=1`、`page_size` 默认 50；单标的最新快照仅 1 条
- 提示：
  - 未指定 `symbol` 时仅支持单日扫描；区间扫描必须指定 `symbol`。
  - `trade_date` 为空时默认当天，非交易日回退到前一交易日。
  - `symbol` 输出统一为 `symbol.suffix` 格式（如 `510300.SH`）。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | N | ETF 代码，支持纯数字或带后缀格式；区间扫描必填 |
| trade_date | string | N | 交易日期 YYYYMMDD；空则默认当天，非交易日回退前一交易日 |
| start_date | string | N | 区间起始日期 YYYYMMDD；区间扫描必填且需配 symbol |
| end_date | string | N | 区间结束日期 YYYYMMDD；区间扫描必填且需配 symbol |
| offset | int | N | 返回结果起始偏移 |
| limit | int | N | 返回结果最大条数 |

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
| adj_factor | f64 | Y | 当日复权因子 |
| ex_adj_factor | f64 | Y | 截至当日累计复权因子 |
| symbol | string | Y | ETF 代码，统一输出 symbol.suffix 格式 |
| trade_date | string | Y | 交易日 YYYYMMDD |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_adjust_factor","arguments":{"symbol":"510300.SH","start_date":"20260623","end_date":"20260624","limit":2}}}')

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
                'ft_etf_adjust_factor',
                {'symbol': '510300.SH', 'start_date': '20260623', 'end_date': '20260624', 'limit': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| symbol | trade_date | adj_factor | ex_adj_factor |
|------|------|------|------|
| 510300.SH | 20260623 | 1.0 | 0.470035 |
| ... | ... | ... | ... |
