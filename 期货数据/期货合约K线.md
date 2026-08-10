# 期货合约K线（MCP 工具 `ft_futures_contract_kline`）

> **MCP 工具**：`ft_futures_contract_kline`（category: `期货数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询指定 WIND 合约的期货 K 线，支持 1、5、15、30、60 分钟和日、周、月、季、年周期；周线以东八区周五为周期末，月、季、年线以末自然日为周期末，未满周期覆盖周期首日至最后一根可用日线。可按毫秒时间戳闭区间筛选，同时指定起止时间时跨度不超过 3 天，结束时间不能单独使用。
- 数据范围：按交易日查询
- 单次限量：`limit` 默认 500，控制返回条数；`start`/`end` 跨度 ≤3 天
- 提示：
  - `symbol` 必填。
  - `interval` 默认 `1min`；别名：`1m/5m/15m/30m/1h/1d/1w/1mo/1q/1y`。
  - `start`/`end` 为毫秒时间戳；同时传入时跨度硬限制 ≤3 天；只传 `end` 会被拒。
  - 不按时间过滤（省略 start/end）时全历史返回，条数受 `limit` 约束，需全量请显式增大 `limit`。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | WIND 合约全码，如 A2605.DCE |
| interval | string | N | 周期，默认 1min；可选 1min/5min/15min/30min/60min/daily/weekly/monthly/quarterly/yearly |
| start | int64 | N | 开始时间戳（毫秒）；与 end 跨度 ≤3 天；仅 start 表示 [start,+∞) |
| end | int64 | N | 结束时间戳（毫秒，闭区间）；须与 start 同时传入，禁止只传 end |
| limit | int | N | 最大返回条数，默认 500 |

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
| amount | f64 | Y | 成交额 |
| close | f64 | Y | 收盘价 |
| datetime | int64 | Y | K 线时间戳（毫秒） |
| high | f64 | Y | 最高价 |
| low | f64 | Y | 最低价 |
| open | f64 | Y | 开盘价 |
| open_interest | f64 | Y | 持仓量 |
| symbol | string | Y | 合约代码（规范为小写段） |
| trade_date | int | Y | 交易日 YYYYMMDD |
| volume | int64 | Y | 成交量 |
| vwap | f64 | Y | 成交均价 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_futures_contract_kline","arguments":{"symbol":"A2605.DCE","interval":"daily","limit":3}}}')

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
                'ft_futures_contract_kline',
                {'symbol': 'A2605.DCE', 'interval': 'daily', 'limit': 3},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| symbol | trade_date | datetime | open | high | low | close | volume | open_interest |
|------|------|------|------|------|------|------|------|------|
| a2605 | 20260514 | 1778742000000 | 0.0 | 0.0 | 0.0 | 4749.0 | 0 | 593.0 |
| a2605 | 20260518 | 1779087600000 | 0.0 | 0.0 | 0.0 | 4749.0 | 0 | 0.0 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
