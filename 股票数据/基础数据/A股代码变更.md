# A股代码变更（MCP 工具 `ft_get_stk_code_change`）

> **MCP 工具**：`ft_get_stk_code_change`（category: `股票数据/基础数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询 A 股股票的代码变更历史及每个代码的起止使用区间。每条记录的 `code` 为该时段实际使用的代码，`trade_code` 为该股票的最新代码。已上市股票的最早区间起始优先使用上市日期；最早一次变更前的原代码若与变更后代码不同，则补充该原代码的使用区间，其 `start_date` 为空、`end_date` 为最早一次变更日期前一日。支持以逗号分隔多个 `trade_code` 查询。
- 数据范围：按代码变更日期查询（覆盖 1993 年以来 A 股代码变更记录，含沪深主板代码迁移及新三板→北交所代码升板）
- 单次限量：无分页，按 `trade_code` 返回该股票全部代码变更记录；多 `trade_code`（逗号分隔）时合并返回
- 提示：
  - `trade_code` 必填，为空或仅空白字符时返回 `code: 400, message: "trade_code is required"`。
  - `trade_code` 支持逗号分隔多个代码，如 `600848.SH` 或 `600848.SH,000001.SZ`。
  - `start_date` 与 `end_date` 同时提供时，须 `start_date` ≤ `end_date`，否则返回 400 错误。
  - `start_date`/`end_date` 按区间过滤，当单只股票记录较多时注意响应大小。
  - 日期参数格式为 `YYYYMMDD`（如 `20200101`），非时间戳。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| trade_code | string | Y | 股票代码（带 .SZ/.SH 后缀），支持逗号分隔多个，如 `600848.SH` 或 `600848.SH,000001.SZ` |
| start_date | string | N | 过滤区间起始日期，`YYYYMMDD` 格式 |
| end_date | string | N | 过滤区间结束日期，`YYYYMMDD` 格式；与 `start_date` 同时提供时须 `start_date` ≤ `end_date` |

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
| code | string | Y | 该时段内实际使用的股票代码（带 .SZ/.SH 后缀） |
| end_date | string | N | 该代码结束使用的日期（`YYYYMMDD`）；`null` 表示当前仍在使用 |
| name | string | Y | 股票名称 |
| start_date | string | Y | 该代码开始使用的起始日期（`YYYYMMDD`；最早原代码之前可能为空字符串） |
| trade_code | string | Y | 该股票的最新代码（带 .SZ/.SH 后缀） |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_stk_code_change","arguments":{"trade_code":"001872.SZ"}}}')

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
                'ft_get_stk_code_change',
                {'trade_code': '001872.SZ'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| 001872.SZ | 1872 | 招商港口 | 20181226 | null |
|------|------|------|------|------|
| null | null | null | null | null |
| null | null | null | null | null |
| ... | ... | ... | ... | ... |
