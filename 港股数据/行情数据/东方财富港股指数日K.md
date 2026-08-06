# 东方财富港股指数日K（MCP 工具 `ft_get_eastmoney_hk_index_daily_kline`）

> **MCP 工具**：`ft_get_eastmoney_hk_index_daily_kline`（category: `港股数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取东方财富港股指数日 K 线数据（HSI 恒生指数、HSCEI 国企指数、HSTECH 恒生科技等），含开/收/高/低、成交量、成交额、振幅、涨跌幅/涨跌额、换手率。数据来源：东方财富。支持按指数代码、交易日或日期区间过滤，不传 `index_code` 返回全部指数。提示：日期格式为 `YYYY-MM-DD`（带横杠），与其他 eastmoney 资金流/估值接口的 `YYYYMMDD` 不同；`index_code` 取恒生体系代码（HSI / HSCEI / HSTECH 等）；`trade_date` 与 `start_date`/`end_date` 互斥。
- 数据范围：K 线时间序列，下限以服务端返回为准
- 单次限量：分页返回，默认 `page=1`、`page_size=50`、最大 `page_size=200`
- 提示：
  - 日期格式为 `YYYY-MM-DD`（带横杠），与其他 eastmoney 资金流/估值接口的 `YYYYMMDD` 不同。
  - `index_code` 取恒生体系代码（HSI / HSCEI / HSTECH 等）。
  - `trade_date` 与 `start_date`/`end_date` 互斥。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| index_code | string | N | 指数代码，如 HSI / HSCEI / HSTECH；不传返回全部指数 |
| trade_date | string | N | 交易日 YYYY-MM-DD；与 start_date/end_date 互斥 |
| start_date | string | N | 区间起始日 YYYY-MM-DD；需与 end_date 同时提供 |
| end_date | string | N | 区间结束日 YYYY-MM-DD；需与 start_date 同时提供 |
| page | int | N | 页码，从 1 开始，默认 1 |
| page_size | int | N | 每页条数，默认 50，最大 200 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

EastmoneyHkIndexDailyKlineItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| index_code | string | Y | 指数代码 |
| index_name | string | Y | 指数名称 |
| secid | string | Y | 东方财富 secid，如 124.HSI |
| trade_date | string | Y | 交易日期 YYYY-MM-DD |
| open | string | Y | 开盘价 |
| close | string | Y | 收盘价 |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| volume | string | Y | 成交量 |
| amount | string | Y | 成交额 |
| amplitude | string | Y | 振幅（%） |
| change_pct | string | Y | 涨跌幅（%） |
| change_amt | string | Y | 涨跌额 |
| turnover | string | Y | 换手率（%） |

## 调用方法（MCP）

> MCP 工具名 `ft_get_eastmoney_hk_index_daily_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_eastmoney_hk_index_daily_kline","arguments":{}}}')

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
                'ft_get_eastmoney_hk_index_daily_kline',
                {},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

港股指数日 K 共 653643 条，节选：

| index_code | index_name | trade_date | open | close | high | low |
|------------|------------|------------|------|-------|------|-----|
| CES300 | 中华沪港通300 | 2026-06-17 | 5206.81 | 5231.14 | 5233.36 | ... |
| ... | ... | ... | ... | ... | ... | ... |
