# 批量可转债K线（MCP 工具 `ft_convertible_bond_candlesticks_batch`）

> **MCP 工具**：`ft_convertible_bond_candlesticks_batch`（category: `债券专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：批量获取多只可转债的历史 K 线，按标的提供开盘价、最高价、最低价、收盘价、成交量和成交额，价格单位为元；支持分、日、周、月、年周期及前复权、后复权，仅适用于可转债标的。
- 数据范围：2002 年首只可转债（阳光转债）发行至今；各标的按自身实际发行和上市时间返回可用历史数据
- 单次限量：无分页，由 `limit` 控制每个标的的返回条数；仅分钟 K 线的 since/until 时间跨度 ≤3 天，其他周期不受 3 天限制
- 提示：
  - `symbols`、`interval_unit`、`until_ts_millis` 必填。
  - `interval_value` 必须是大于 0 的整数，且仅在 `interval_unit=Minute` 时生效；不传默认为 1 分钟，传 5 表示 5 分钟。选择 `Day`/`Week`/`Month`/`Year` 时，分别固定按 1 日、1 周、1 月、1 年周期处理；所有 `symbols` 使用相同周期。
  - `symbols` 中每项必须是可转债代码；沪市支持 `.XSHG`/`.SH`，深市支持 `.XSHE`/`.SZ`。
  - 输入短后缀时，响应中的 symbol 会规范化为 `.XSHG`、`.XSHE` 等长后缀。
  - 若任一 symbol 不是可转债，整个批量请求失败，不静默过滤；当前外部接口返回系统错误。
  - since/until 均为毫秒时间戳；分钟 K 线超过 3 天时当前外部接口返回系统错误，拉取更长周期需分段调用。
  - 默认不复权（None），可设 Forward 前复权或 Backward 后复权。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbols | string[] | Y | 可转债代码列表，如 `["113027.XSHG","128048.XSHE"]`；也接受 `.SH`、`.SZ` 短后缀 |
| interval_unit | enum | Y | 周期单位：Minute/Day/Week/Month/Year |
| interval_value | int | N | 分钟周期数值，必须大于 0；当前仅在 `interval_unit=Minute` 时生效，不传等同于 `1` |
| adjust_kind | enum | N | 复权：None（默认，不复权）/Forward（前复权）/Backward（后复权） |
| since_ts_millis | int(ms) | N | 开始时间戳，单位毫秒；分钟 K 线与 until 的跨度 ≤3 天 |
| until_ts_millis | int(ms) | Y | 结束时间戳，单位毫秒 |
| limit | int | N | 每个标的的返回条数上限；未传 since 和 limit 时默认最多返回 50 根 K 线 |

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
| close | decimal | Y | 收盘价（或最新价），单位元；JSON 中为字符串 |
| high | decimal | Y | 最高价，单位元；JSON 中为字符串 |
| low | decimal | Y | 最低价，单位元；JSON 中为字符串 |
| open | decimal | Y | 开盘价，单位元；JSON 中为字符串 |
| symbol | string | Y | 可转债代码；响应统一使用 `.XSHG`、`.XSHE` 等长市场后缀 |
| ts_millis | int(ms) | Y | 收盘时间戳，单位毫秒 |
| ts_millis_open | int(ms) | Y | 开盘时间戳，单位毫秒 |
| turnover | decimal | Y | 成交额，单位元；JSON 中为字符串 |
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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_convertible_bond_candlesticks_batch","arguments":{"symbols":["113027.SH"],"interval_unit":"day","until_ts_millis":1782370800000,"limit":2}}}')

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
                'ft_convertible_bond_candlesticks_batch',
                {'symbols': ['113027.SH'], 'interval_unit': 'day', 'until_ts_millis': 1782370800000, 'limit': 2},
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
| 113027.SH | 201.9500 | 206.5450 | 194.6660 | 194.8950 | 1717570800000 | 1717551000000 | 1203839872.3300 | 5973980 |
| 113027.SH | 208.0000 | 216.6000 | 196.0000 | 203.3990 | 1717657200000 | 1717637400000 | 4078942878.1300 | 19714910 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
