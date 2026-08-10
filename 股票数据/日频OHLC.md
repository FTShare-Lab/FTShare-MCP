# 日频 OHLC（MCP 工具 `daily_ohlc`）

> **MCP 工具**：`daily_ohlc`（category: `股票数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：日频 OHLC（开高低收）统一查询入口，通过 `type` 参数选择数据口径，支持 A 股及通用证券、港股、美股、东方财富板块、港股指数、全球指数和同花顺板块。不同口径分别使用 `symbol`、`stock_code`、`index_code`、`secid` 或 `board_code` 标识查询对象。
- 数据范围：按交易日查询日频 OHLC 数据
- 单次限量：`limit`/`page_size` 上限 500
- 提示：
  - `type` 默认 `stock`；各 type 有不同必填字段（见输入参数表"必选"列）。
  - `us_stock`/`eastmoney_board` 的 `start_date` 与 `end_date` 同时传入时跨度须 ≤3 天（超出返回 INVALID_ARGUMENT）。
  - 分页仅 `us_stock`/`eastmoney_board`/`eastmoney_board_latest`/`hk_index`/`ths_board`/`ths_all_board` 支持；`stock`/`hk_stock`/`global_index` 用 `limit` 控制条数。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| type | string | N | 数据口径，默认 `stock`；可选 `stock`/`hk_stock`/`us_stock`/`eastmoney_board`/`eastmoney_board_latest`/`hk_index`/`global_index`/`ths_board`/`ths_all_board` |
| symbol | string | 条件 | 证券代码（外部码，如 `600584.SH`/`00700.HK`）；`stock`/`hk_stock` 必填 |
| stock_code | string | 条件 | 美股代码（如 `AAPL`）；`us_stock` 必填 |
| index_code | string | 条件 | 港股指数代码（如 `HSI`）；`hk_index` 必填 |
| secid | string | 条件 | 全球指数 secid（如 `100.N225`）；`global_index` 必填 |
| board_code | string | 条件 | 板块代码（如 `BK0425`/`885311`）；`eastmoney_board`/`ths_board` 必填 |
| start_date | string | N | 起始日期 `YYYY-MM-DD`；`us_stock`/`hk_index`/`global_index`/`eastmoney_board` 使用 |
| end_date | string | N | 截止日期 `YYYY-MM-DD`；`us_stock`/`eastmoney_board` 与 `start_date` 跨度 ≤3 天 |
| until_date | string | N | `hk_stock` 截止日期 `YYYY-MM-DD` |
| limit | int | N | 返回条数，1-500；`stock`/`hk_stock` 使用 |
| page | int | N | 页码，≥1；分页口径使用 |
| page_size | int | N | 每页条数，1-500；分页口径使用 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项 |

### data 业务字段

`stock` 口径（默认）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| close | string | Y | 收盘价 |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| open | string | Y | 开盘价 |
| ts_millis | int64 | Y | 收盘时间戳（毫秒） |
| ts_millis_open | int64 | Y | 开盘时间戳（毫秒） |
| turnover | string | Y | 成交额 |
| volume | int64 | Y | 成交量 |

> 其他口径返回字段不同：
> - `hk_stock`：`close`/`date`/`high`/`low`/`open`/`turnover`/`volume`
> - `us_stock`：`code`/`name`/`date`/`open`/`high`/`low`/`close`/`volume`/`amount`/`amplitude`/`fqt`/`klt`/`market`/`secid`
> - `eastmoney_board`：`市场`/`开盘`/`收盘`/`最高`/`最低`/`成交量`/`成交额`/`振幅`/`换手率`/`日期`/`板块代码`/`板块名称`/`涨跌幅`/`涨跌额`
> - `eastmoney_board_latest`：`board_code`/`board_name`/`date`/`open`/`high`/`low`/`close`/`volume`/`turnover`/`turnover_rate`/`amplitude`/`change`/`change_rate`/`market`
> - `hk_index`：`index_code`/`index_name`/`trade_date`/`open`/`high`/`low`/`close`/`volume`/`turnover`/`amount`/`amplitude`/`change_amt`/`change_pct`/`secid`
> - `global_index`：`code`/`name`/`trade_date`/`open`/`high`/`low`/`close`/`volume`/`turnover`/`amount`/`amplitude`/`change_amount`/`change_pct`/`secid`
> - `ths_board`/`ths_all_board`：`board_code`/`board_name`/`module`/`date`/`open`/`high`/`low`/`close`/`volume`

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"daily_ohlc","arguments":{"type":"stock","symbol":"600584.SH","limit":2}}}')

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
                'daily_ohlc',
                {'type': 'stock', 'symbol': '600584.SH', 'limit': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

`stock` 口径（`type=stock, symbol=600584.SH, limit=2`）：

| close | high | low | open | ts_millis | ts_millis_open | turnover | volume |
|------|------|------|------|------|------|------|------|
| 77.7500 | 78.7400 | 73.9700 | 75.1500 | 1786086000000 | 1786066200000 | 17483054960.2900 | 228341425 |
| 78.5200 | 79.4900 | 76.0000 | 78.4200 | 1786345200000 | 1786325400000 | 11127773470.9100 | 143099767 |
| ... | ... | ... | ... | ... | ... | ... | ... |
