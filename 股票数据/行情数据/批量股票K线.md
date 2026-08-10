# 批量股票K线（MCP 工具 `ft_stock_candlesticks_batch`）

> **MCP 工具**：`ft_stock_candlesticks_batch`（category: `股票数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：批量查询股票、ETF、可转债和指数等多个标的的历史 K 线，支持在一次查询中混合不同证券类别。
- 数据范围：1991 至今（不同标的的实际上线时间不同；批量结果分别按各标的可用历史数据返回）
- 单次限量：无分页，由 `limit` 控制每个标的的返回条数；分钟 K 线同时传入 since/until 时，时间范围不超过 3 天，其他周期不受 3 天限制
- 提示：
  - `symbols`、`interval_unit`、`until_ts_millis` 必填。
  - `interval_value` 必须是大于 0 的整数，且仅在 `interval_unit=Minute` 时生效；不传默认为 1 分钟，传 5 表示 5 分钟。选择 `Day`/`Week`/`Month`/`Year` 时，分别固定按 1 日、1 周、1 月、1 年周期处理；所有 `symbols` 使用相同周期。
  - 市场后缀支持 `.XSHG`/`.SH`、`.XSHE`/`.SZ`、`.BJSE`/`.BJ`；输入短后缀时，响应中的 symbol 会规范化为长后缀。
  - 未传 `since_ts_millis` 和 `limit` 时，每个标的默认最多返回 50 根 K 线。
  - 分钟 K 线超过 3 天时，当前外部接口返回系统错误。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbols | string[] | Y | 标的代码列表，允许股票、ETF、可转债和指数混合，例如 `["600519.SH","510300.SH","113027.SH","000300.SH"]` |
| interval_unit | enum | Y | 周期单位：Minute/Day/Week/Month/Year |
| interval_value | int | N | 分钟周期数值，必须大于 0；当前仅在 `interval_unit=Minute` 时生效，不传等同于 `1` |
| adjust_kind | enum | N | 复权类型：None（默认）/Forward（前复权）/Backward（后复权） |
| since_ts_millis | int(ms) | N | 开始时间戳；分钟 K 线与 until 的跨度不超过 3 天 |
| until_ts_millis | int(ms) | Y | 结束时间戳，单位毫秒 |
| limit | int | N | 每个标的的返回条数上限；未传 since 和 limit 时默认 50 |

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
| close | decimal | Y | 收盘价或最新价，单位元，JSON 中为字符串 |
| high | decimal | Y | 最高价，单位元，JSON 中为字符串 |
| low | decimal | Y | 最低价，单位元，JSON 中为字符串 |
| open | decimal | Y | 开盘价，单位元，JSON 中为字符串 |
| symbol | string | Y | 标的代码；响应统一序列化为 `.XSHG`、`.XSHE`、`.BJSE` 等长市场后缀 |
| ts_millis | int(ms) | Y | 收盘时间戳，单位毫秒 |
| ts_millis_open | int(ms) | Y | 开盘时间戳，单位毫秒 |
| turnover | decimal | Y | 成交额，单位元，JSON 中为字符串 |
| volume | int64 | Y | 成交量 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_candlesticks_batch","arguments":{"symbols":["600519.SH"],"interval_unit":"day","interval_value":1,"since_ts_millis":1781506800000,"until_ts_millis":1782370800000,"limit":2}}}')

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
                'ft_stock_candlesticks_batch',
                {'symbols': ['600519.SH'], 'interval_unit': 'day', 'interval_value': 1, 'since_ts_millis': 1781506800000, 'until_ts_millis': 1782370800000, 'limit': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| symbol | open | high | low | close | ts_millis | ts_millis_open | turnover | volume |
|------|------|------|------|------|------|------|------|------|
| 600519.SH | 1222.6500 | 1241.8700 | 1207.5100 | 1207.6800 | 1782284400000 | 1782264600000 | 5516574459.5500 | 4533528 |
| 600519.SH | 1207.0000 | 1227.0000 | 1200.0000 | 1212.1000 | 1782370800000 | 1782351000000 | 5860481755.2100 | 4844649 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
