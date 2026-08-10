# 全球指数日K线（MCP 工具 `ft_global_index_daily_kline`）

> **MCP 工具**：`ft_global_index_daily_kline`（category: `指数专题/指数行情`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取东方财富全球指数及部分 A 股、港股宽基指数的历史日 K 线，包含开盘、最高、最低、收盘、成交量、成交额、振幅、涨跌幅、涨跌额和换手率；指数使用东方财富全球指数编码，部分指数早期成交量、成交额和换手率可能为零。
- 数据范围：标普500和道琼斯指数可查询 1990-01-01 至今的数据；纳斯达克、恒生指数和日经225可查询 2000 年至今的数据
- 单次限量：无分页，按 start_date/end_date 时间区间返回全部日K；建议区间控制在合理范围避免单次返回过大
- 提示：
  - `secid` 必填，东方财富全球指数编码格式，如 `100.NDX`、`100.DJIA`、`100.SPX`、`100.HSI`、`100.N225`。
  - `symbol` 使用本接口定义的全球指数代码，如 `IXIC`、`DJI`、`HSI`。
  - `start_date`/`end_date` 均为 `YYYY-MM-DD` 字符串；时间下限因标的而异（见数据范围）。
  - 价格字段为字符串，负涨跌额带 `-`。
  - 指数无成交额/换手率概念，`amount`/`turnover` 常为 `"0.00"`/`"0.0000"`。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| secid | string | Y | 东方财富全球指数编码，如 100.NDX、100.DJIA、100.SPX、100.HSI、100.N225 |
| start_date | string | N | 开始日期 YYYY-MM-DD（含） |
| end_date | string | N | 结束日期 YYYY-MM-DD（含） |

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
| amount | string | Y | 成交额（指数常为 "0.00"） |
| amplitude | string | Y | 振幅 % |
| change_amount | string | Y | 涨跌额 |
| change_pct | string | Y | 涨跌幅 % |
| close | string | Y | 收盘价 |
| code | string | Y | 指数代码，如 DJIA |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| name | string | Y | 指数中文名称，如「道琼斯」 |
| open | string | Y | 开盘价 |
| secid | string | Y | 东方财富指数编码，如 100.DJIA |
| trade_date | string | Y | 交易日 YYYY-MM-DD |
| turnover | string | Y | 换手率 %（指数常为 "0.0000"） |
| volume | int64 | Y | 成交量（早期常为 0） |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_global_index_daily_kline","arguments":{"secid":"100.N225","start_date":"2026-06-24","end_date":"2026-06-24"}}}')

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
                'ft_global_index_daily_kline',
                {'secid': '100.N225', 'start_date': '2026-06-24', 'end_date': '2026-06-24'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| DJIA | 道琼斯 | 2024-01-10 | 37552.91 | 37695.73 | 37740.77 | 37524.40 | 279540000 | 0.45 |
|------|------|------|------|------|------|------|------|------|
| null | null | null | null | null | null | null | null | null |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
