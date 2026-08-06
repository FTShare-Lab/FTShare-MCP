# A股行情列表（MCP 工具 `ft_daec_stocks_*`）

> **MCP 工具**：`ft_daec_stocks_all` / `ft_daec_stocks_xshg` / `ft_daec_stocks_xshe` / `ft_daec_stocks_bjse`（category: `股票数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。

- 描述：分页获取 A 股实时行情（daec 全字段族，返回完整行情 + 基本面字段），支持 `filter` / `order_by`。所有板块均自动排除退市股、仅返回有实时行情的股票。
- 数据范围：实时行情快照（无历史时间维度）
- 单次限量：分页返回，默认 `page=1`、`page_size=20`，`page_size` 上限 200
- 提示：
  - 各工具对应固定板块，**内置筛选不可覆盖**。
  - `filter` 与内置筛选自动 AND 合并，`order_by` 支持 `field` / `field asc` / `field desc`。

## 工具与板块对照

| MCP 工具 | 板块 | 内置筛选 |
|----------|------|----------|
| `ft_daec_stocks_all` | 全市场 | `close != null`（排除退市） |
| `ft_daec_stocks_xshg` | 上证 A 股 | `market_id = XSHG` |
| `ft_daec_stocks_xshe` | 深证 A 股 | `market_id = XSHE` |
| `ft_daec_stocks_bjse` | 北证 A 股 | `market_id = BJSE` |

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| page | integer | N | 页码，默认 1，最小 1 |
| page_size | integer | N | 每页条数，默认 20，上限 200 |
| filter | string | N | 筛选表达式，与该板块内置筛选自动 AND 合并，如 `close > 10`、`name.contains("银行")` |
| order_by | string | N | 排序，如 `change_rate desc`（支持 `field` / `field asc` / `field desc`） |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；分页信息归入 `metadata.pagination`。

`data` 元素（同「股票详情」全字段族）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码（如 600000.XSHG） |
| name | string | Y | 标的名称 |
| open | string | Y | 开盘价，单位元 |
| high | string | Y | 最高价，单位元 |
| low | string | Y | 最低价，单位元 |
| close | string | Y | 收盘价 / 最新价，单位元 |
| prev_close | string | Y | 前收盘价，单位元 |
| change | string | Y | 涨跌额，单位元 |
| change_rate | float64 | Y | 涨跌幅 |
| volume | int64 | Y | 成交量，单位股 |
| turnover | string | Y | 成交额，单位元 |
| amplitude | float64 | Y | 振幅 |
| avg | float64 | N | 均价 |
| bid_ask_ratio | float64 | N | 委比 |
| turnover_rate | float64 | N | 换手率 |
| market_cap | string | N | 总市值，单位元 |
| tradable_a_market_cap | string | N | 流通 A 股市值，单位元 |
| pe_ttm | float64 | N | 市盈率（TTM） |
| board | string | N | 板块（如 XshgMain / XshgStar / SzChiNext / Bjse） |
| st | bool | N | 是否 ST |
| status | string | N | 状态 |
| shares | int64 | N | 总股本 |
| float_a_shares | int64 | N | 流通 A 股股本 |
| listing_date | string | N | 上市日期 |
| change_rate_day5 | float64 | N | 5 日涨跌幅 |
| change_rate_day10 | float64 | N | 10 日涨跌幅 |
| change_rate_day20 | float64 | N | 20 日涨跌幅 |
| change_rate_day60 | float64 | N | 60 日涨跌幅 |
| change_rate_day120 | float64 | N | 120 日涨跌幅 |
| change_rate_ytd | float64 | N | 年初至今涨跌幅 |
| ts_millis | int64 | N | 交易所时间戳，单位毫秒 |

## 调用方法（MCP）

> 本页 4 个 MCP 工具共用同一套 Streamable HTTP 握手流程：先 `initialize`，再发送 `notifications/initialized`，最后逐个调用 `tools/call`。

四个工具均已使用以下参数实测：

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

for payload in \
  '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_daec_stocks_all","arguments":{"page":1,"page_size":2,"order_by":"change_rate desc"}}}' \
  '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"ft_daec_stocks_xshg","arguments":{"page":1,"page_size":2,"filter":"close > 10"}}}' \
  '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"ft_daec_stocks_xshe","arguments":{"page":1,"page_size":2}}}' \
  '{"jsonrpc":"2.0","id":5,"method":"tools/call","params":{"name":"ft_daec_stocks_bjse","arguments":{"page":1,"page_size":2}}}'
do
  CALL_RESPONSE=$(curl -fsS -m 60 -X POST "$MCP_BASE_URL" \
    -H "Accept: application/json, text/event-stream" \
    -H "Content-Type: application/json" \
    -H "Mcp-Session-Id: $SID" \
    -H "MCP-Protocol-Version: 2025-11-25" \
    -d "$payload")

  check_mcp_response "$CALL_RESPONSE"
done
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
            calls = [
                ("ft_daec_stocks_all",
                    {'page': 1, 'page_size': 2, 'order_by': 'change_rate desc'}),
                ("ft_daec_stocks_xshg",
                    {'page': 1, 'page_size': 2, 'filter': 'close > 10'}),
                ("ft_daec_stocks_xshe",
                    {'page': 1, 'page_size': 2}),
                ("ft_daec_stocks_bjse",
                    {'page': 1, 'page_size': 2}),
            ]
            for tool_name, arguments in calls:
                result = await session.call_tool(tool_name, arguments)
                if result.is_error:
                    raise RuntimeError(result.content[0].text)
                print(tool_name, result.structured_content)
                print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

每个工具的首条真实记录（节选核心列）：

| MCP 工具 | symbol | name | board | close | change_rate | turnover | pe_ttm |
|----------|--------|------|-------|-------|-------------|----------|--------|
| ft_daec_stocks_all | 001232.XSHE | N嘉立创 | XsheMain | 208.01 | 1.4628226379351172 | 6883459441.93 | null |
| ft_daec_stocks_xshg | 600007.XSHG | 中国国贸 | XshgMain | 18.6 | -0.02053712480252765 | 60142671 | 15.9894 |
| ft_daec_stocks_xshe | 000001.XSHE | 平安银行 | XsheMain | 11.44 | -0.01549053356282272 | 1401213600.03 | 5.2368 |
| ft_daec_stocks_bjse | 920000.BJSE | 安徽凤凰 | Bjse | 14.39 | 0.014094432699083862 | 23286354 | 19.4046 |

## 注意事项

- 板块内置筛选不可覆盖（如上证固定 `market_id = XSHG`）。
- 全市场 / 沪深口径与板块口径存在包含关系（如深证含创业板、沪市含科创板），按需选择工具。
