# 东方财富美股日OHLC（MCP 工具 `ft_eastmoney_us_stock_daily_kline`）

> **MCP 工具**：`ft_eastmoney_us_stock_daily_kline`（category: `美股数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取东方财富美股历史日 K 线（OHLC），全部字段以字符串返回，支持按日期范围过滤与分页。数据来源：东方财富。提示：`stock_code` 为必填，纯代码不带市场前缀（如 AAPL）；日期格式支持 `YYYY-MM-DD` 或 `YYYYMMDD`，输出统一为 `YYYY-MM-DD`。
- 数据范围：K 线时间序列，下限以服务端返回为准
- 单次限量：分页返回，默认 `page=1`、`page_size` 由服务端决定
- 提示：
  - `stock_code` 为必填，纯代码不带市场前缀（如 AAPL）；找不到时返回 400。
  - 日期格式支持 `YYYY-MM-DD` 或 `YYYYMMDD`，输出统一为 `YYYY-MM-DD`。
  - 区间过滤 `start_date`/`end_date` 同时提供时，结束日期不能早于开始日期，且跨度不超过 3 天（按日期差值计，超出返回 `INVALID_ARGUMENT "时间范围不能超过 3 天"`）。
  - 响应结构为 `PaginatedResponse`（items/total_pages/total_items），无 code/message 信封。
  - 所有数值字段均以字符串形式返回（含 OHLC、成交量、成交额、振幅、klt、fqt）。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | Y | 股票代码，如 AAPL |
| start_date | string | N | 起始日期（含），YYYY-MM-DD 或 YYYYMMDD；不传从最早开始 |
| end_date | string | N | 截止日期（含），YYYY-MM-DD 或 YYYYMMDD；不传到最晚 |
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

EastmoneyUsStockDailyKlineRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| secid | string | Y | 证券 ID，格式 {market}.{code} |
| code | string | Y | 股票代码 |
| name | string | Y | 股票名称 |
| market | string | Y | 市场编号 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘价 |
| close | string | Y | 收盘价 |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| volume | string | Y | 成交量 |
| amount | string | Y | 成交额 |
| amplitude | string | Y | 振幅 |
| klt | string | Y | K 线类型（101=日 K） |
| fqt | string | Y | 复权类型（1=前复权） |

## 调用方法（MCP）

> MCP 工具名 `ft_eastmoney_us_stock_daily_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_us_stock_daily_kline","arguments":{"stock_code":"AAPL"}}}')

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
                'ft_eastmoney_us_stock_daily_kline',
                {'stock_code': 'AAPL'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

> 示例调用已验证通过。表头结构如下：

| stock_code | trade_date | open | high | low | close |
|------------|------------|------|------|-----|-------|
| ... | ... | ... | ... | ... | ... |
