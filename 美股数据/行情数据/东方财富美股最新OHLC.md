# 东方财富美股最新OHLC（MCP 工具 `ft_eastmoney_us_stock_latest_kline`）

> **MCP 工具**：`ft_eastmoney_us_stock_latest_kline`（category: `美股数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取单只东方财富美股的最新一根日 K 线。`stock_code` 在 MCP 层必填，用于避免无代码全量查询超时。
- 数据范围：最新快照（无时间维度，每标的仅最新一个交易日）
- 单次限量：分页返回，默认 `page=1`、`page_size` 由后端定
- 提示：
  - `stock_code` 为 MCP 必填参数，使用不带市场前缀的纯代码（如 `AAPL`）。
  - 公开 v1 技术上允许省略该参数，但实测可能长时间无响应；MCP 因此主动拒绝无代码调用。
  - 响应结构为 `PaginatedResponse`（items/total_pages/total_items），无 code/message 信封。
  - 字段 `date` 为 ISO 日期，OHLC/amount 为 Decimal 字符串，volume 为整数。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | Y | 股票代码，如 `AAPL`；MCP 必填 |
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

EastmoneyUsStockLatestKline：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| secid | string | Y | 证券 ID，格式 {market}.{code}，如 105.ADV |
| code | string | Y | 股票代码 |
| name | string | Y | 股票名称 |
| market | string | Y | 市场编号 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘价（Decimal 字符串） |
| close | string | Y | 收盘价（Decimal 字符串） |
| high | string | Y | 最高价（Decimal 字符串） |
| low | string | Y | 最低价（Decimal 字符串） |
| volume | int | Y | 成交量 |
| amount | string | Y | 成交额（Decimal 字符串） |
| amplitude | number | Y | 振幅 |
| klt | int | Y | K 线类型（101=日 K） |
| fqt | int | Y | 复权类型（1=前复权） |

## 调用方法（MCP）

> MCP 工具名 `ft_eastmoney_us_stock_latest_kline`。MCP Streamable HTTP 要求先 `initialize` 获取 `Mcp-Session-Id`，再发送 `notifications/initialized`，最后调用 `tools/call`。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_us_stock_latest_kline","arguments":{"stock_code":"AAPL","page":1,"page_size":2}}}')

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
                'ft_eastmoney_us_stock_latest_kline',
                {'stock_code': 'AAPL', 'page': 1, 'page_size': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

```json
{"secid":"105.AAPL","code":"AAPL","name":"苹果","market":"105","date":"2026-08-03","open":"309.58","close":"303.42","high":"311.8","low":"302.56","volume":73762121,"amount":"22508802048","amplitude":2.99,"klt":101,"fqt":1}
```
