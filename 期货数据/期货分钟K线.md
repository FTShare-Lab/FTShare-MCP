# 期货分钟K线（MCP 工具 `ft_futures_kline_intraday`）

> **MCP 工具**：`ft_futures_kline_intraday`（category: `期货数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询指定期货合约的实时 1 分钟 K 线，可按毫秒时间戳区间筛选；同时指定起止时间时，跨度不超过 3 天。
- 数据范围：实时表中保留的期货 1 分钟 K 线
- 单次限量：默认 600 条，最大 1000 条
- 提示：
  - `symbol` 必填，支持 WIND 合约全码或表内合约代码。
  - `interval` 当前仅支持 `1min`，兼容 `1m`，默认 `1min`。
  - 同时提供 `start` 和 `end` 时，时间跨度不得超过 3 天。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | WIND 合约全码或表内合约代码 |
| interval | string | N | 当前仅支持 `1min`，兼容 `1m`，默认 `1min` |
| start | int64 | N | 开始时间，毫秒时间戳，闭区间 |
| end | int64 | N | 结束时间，毫秒时间戳，闭区间 |
| limit | int | N | 最大返回条数，默认 600，范围 1～1000 |

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
| amount | float | Y | 成交额 |
| bar_interval | string | Y | K 线周期 |
| close | float | Y | 收盘价 |
| datetime | int64 | Y | K 线时间，毫秒时间戳 |
| exchange | string | Y | 交易所 |
| high | float | Y | 最高价 |
| high_limit_price | float | Y | 涨停价 |
| low | float | Y | 最低价 |
| low_limit_price | float | Y | 跌停价 |
| open | float | Y | 开盘价 |
| open_interest | float | Y | 持仓量 |
| pre_close_price | float | Y | 前收盘价 |
| pre_settlement_price | float | Y | 前结算价 |
| settlement_price | float | Y | 结算价 |
| symbol | string | Y | 合约代码 |
| trade_date | int | Y | 交易日 YYYYMMDD |
| updated_at | int64 | Y | 实时表写入时间，毫秒时间戳 |
| variety | string | Y | 品种代码 |
| volume | int64 | Y | 成交量 |
| vwap | float | Y | 成交量加权均价 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_futures_kline_intraday","arguments":{"symbol":"A2609.DCE","limit":5}}}')

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
                'ft_futures_kline_intraday',
                {'symbol': 'A2609.DCE', 'limit': 5},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| datetime | symbol | open | high | low | close | volume |
|------|------|------|------|------|------|------|
| 1786345140000 | a2609 | 4850.0 | 4852.0 | 4849.0 | 4850.0 | 741 |
| 1786345200000 | a2609 | 4851.0 | 4856.0 | 4850.0 | 4854.0 | 1600 |
| 1786345260000 | a2609 | 4854.0 | 4854.0 | 4854.0 | 4854.0 | 0 |
| ... | ... | ... | ... | ... | ... | ... |
