# 指数ETF基金导出（MCP 工具 `ft_etf_fund_export`）

> **MCP 工具**：`ft_etf_fund_export`（category: `ETF专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：导出单只指数 ETF 的基金档案与持仓明细，包括代码、名称、类型、风险等级、规模、净值、日及中长期收益率、日/周/月/年换手率、申赎限额、30 日月均成交量额、前十大持仓、历任基金经理和净值增长情况。
- 数据范围：最新快照
- 单次限量：始终分页，`page` 默认 1、`page_size` 默认 50（避免全量 JSON 过大被网关截断）
- 提示：
  - `request_id` 必填，由调用方生成，原样回填响应 `request_id`。
  - `page_size` 传 0 按默认 50 处理。
  - 字段众多，下面仅列核心字段；收益率/换手率系列字段名见输出参数表。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| request_id | string | Y | 请求唯一标识，由调用方生成，原样写入响应 |
| page | int | N | 页码，从 1 开始，默认 1 |
| page_size | int | N | 每页条数，默认 50，传 0 按默认处理 |

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
| accesn_date | string | Y | 任职日期 |
| annureturn_3y | float64 | Y |  |
| annureturn_5y | float64 | Y |  |
| annureturn_establ | float64 | Y |  |
| aum | float64 | Y |  |
| best_return | f64 | N | 近一年收益率（return_y 同义，前端单独字段） |
| etf_code | string | Y | `{security_code}.{exchange}` 展示用代码 |
| exabbr | string | N | 交易所简称（部分标的为空） |
| exchange | string | Y | 交易所 |
| fund_manager | string | Y | 基金经理 |
| fund_managers | array | Y | 历任基金经理列表（fund_manager/is_inoffice/accesn_date） |
| fund_type | string | Y | 基金类型 |
| is_inoffice | string | Y | 是否在职 |
| max_chargrt | f64 | N | 最高费率（%） |
| nav_growth_rate | object | Y | 净值增长块（return_d） |
| operat_mode | string | Y | 运作方式 |
| return_6m | float64 | Y |  |
| return_d | float64 | Y |  |
| return_y | float64 | Y |  |
| risk_lvl | string | Y | 风险等级 |
| security_code | string | Y | 证券代码 |
| security_id | string | Y | 证券 ID |
| security_name | string | Y | 证券名称 |
| top10_holdings | array | Y | 前十大持仓明细（publish_date/end_date/invest_obj/sec_code/sec_id/sec_name/hld_shares/mkt_value/ratioin_nv） |
| tot_asset | float64 | Y |  |
| total_return | object | Y | 收益率汇总块（return_6m/return_y/annureturn_3y/5y/establ） |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_fund_export","arguments":{"request_id":"ftshare-live-check","page":1,"page_size":2}}}')

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
                'ft_etf_fund_export',
                {'request_id': 'ftshare-live-check', 'page': 1, 'page_size': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| etf_code | security_code | security_name | fund_type | risk_lvl | tot_asset | return_y | annureturn_3y | fund_manager |
|------|------|------|------|------|------|------|------|------|
| 510290.场外交易市场 | 510290 | 南方上证380ETF | 股票ETF基金 | 中 | 158001640.83 | 19.7295943916 | 8.6484013725 | 孙伟 |
| 510290.上海证券交易所 | 510290 | 380ETF | 股票ETF基金 | 中 | 158001640.83 | 19.7295943916 | 8.6484013725 | 孙伟 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
