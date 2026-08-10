# 分时与分钟 K 线（MCP 工具 `intraday_kline`）

> **MCP 工具**：`intraday_kline`（category: `股票数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：分时与分钟 K 线统一查询入口，通过 `type` 参数选择数据口径，支持分钟 K 线、分时价格线、批量 K 线和期货分钟 K 线。
- 数据范围：分钟级 / 分时行情
- 单次限量：`limit` 1-500；`batch` 的 `symbols` 最多 500 个标的
- 提示：
  - `type` 默认 `minute_kline`；各 type 有不同必填字段（见输入参数表"必选"列）。
  - `minute_kline` 传 `date`（`YYYYMMDD`）按天查询完整交易时段（09:30-15:00 CST），按 `ts_millis` 去重；`date` 优先级高于 `since_ts_millis`/`until_ts_millis`。
  - `batch` 必填 `symbols`（1-500 个标的）和 `until_ts_millis`；结果展开为逐标的记录。
  - `futures` 用 `symbol`（WIND 合约码如 `A2605.DCE`）和 `interval`（默认 `1min`）。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| type | string | N | 数据口径，默认 `minute_kline`；可选 `minute_kline`/`intraday_price`/`batch`/`futures` |
| symbol | string | 条件 | 单标的代码（外部码，如 `600584.SH`）；`minute_kline`/`intraday_price` 必填 |
| symbols | string[] | 条件 | 批量标的列表，1-500 个；`batch` 必填 |
| interval_unit | string | N | K 线区间单位，默认 `Minute`；可选 `Minute`/`Day`/`Week`/`Month`/`Year` |
| interval_value | int | N | 分钟周期数值，仅 `batch` 的 `interval_unit=Minute` 时生效，≥1 |
| limit | int | N | 返回条数，1-500；GET 默认 40 |
| date | string | N | 按天查询，`YYYYMMDD` 如 `20260624`；仅 `minute_kline` |
| adjust_kind | string | N | 复权方式，默认 `None`；可选 `None`/`Forward`/`Backward` |
| until_ts_millis | int64 | 条件 | 截止毫秒时间戳；`batch` 必填 |
| since_ts_millis | int64 | N | 起始毫秒时间戳；`batch`/单票可选 |
| interval | string | N | 期货周期，默认 `1min`；仅 `futures` |
| start | int64 | N | 期货开始时间戳（毫秒）；仅 `futures` |
| end | int64 | N | 期货结束时间戳（毫秒，闭区间）；仅 `futures` |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项 |

### data 业务字段

`minute_kline` 口径（默认）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| close | string | Y | 收盘价 |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| open | string | Y | 开盘价 |
| ts_millis | int64 | Y | K 线结束时间戳（毫秒） |
| ts_millis_open | int64 | Y | K 线开始时间戳（毫秒） |
| turnover | string | Y | 成交额 |
| volume | int64 | Y | 成交量 |

> 其他口径返回字段不同：
> - `intraday_price`：`price`/`ts_millis`/`turnover`/`volume`
> - `batch`：结果展开为逐标的记录，字段与 `minute_kline` 一致
> - `futures`：`symbol`/`trade_date`/`datetime`/`open`/`high`/`low`/`close`/`volume`/`amount`/`open_interest`/`vwap`

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"intraday_kline","arguments":{"type":"minute_kline","symbol":"600584.SH","date":"20260624"}}}')

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
                'intraday_kline',
                {'type': 'minute_kline', 'symbol': '600584.SH', 'date': '20260624'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

`minute_kline` 口径（`type=minute_kline, symbol=600584.SH, date=20260624`）：

| close | high | low | open | ts_millis | ts_millis_open | turnover | volume |
|------|------|------|------|------|------|------|------|
| 84.0000 | 84.0000 | 84.0000 | 84.0000 | 1782264600000 | 1782264540000 | 215178348.0000 | 2561647 |
| 86.0500 | 86.0800 | 84.1000 | 84.3700 | 1782264660000 | 1782264600000 | 319907084.3800 | 3769394 |
| 86.9800 | 87.0000 | 86.0800 | 86.0800 | 1782264720000 | 1782264660000 | 370503841.9000 | 4280336 |
| ... | ... | ... | ... | ... | ... | ... | ... |
