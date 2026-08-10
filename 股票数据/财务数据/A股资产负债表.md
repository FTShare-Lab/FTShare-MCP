# A股资产负债表（MCP 工具 `ft_balance`）

> **MCP 工具**：`ft_balance`（category: `股票数据/财务数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询 A 股上市公司资产负债表。每条数据对应一个报告期，优先采用「合并调整」报表，没有时采用「合并未调整」报表。支持按 `stock_code` 查询单只股票全部报告期，或按 `year` 和 `report_type` 查询全市场指定报告期；`report_type` 支持 `q1`、`q2`、`q3` 和 `annual`，并兼容以 `h1` 表示半年报。
- 数据范围：按报告期查询
- 单次限量：模式B 分页默认 `page=1`、`page_size=50`，最大 500
- 提示：
  - `stock_code` 格式为 6 位数字 + 交易所后缀（如 `600519.SH`），模式A 与模式B 二选一。
  - 模式A 单票查询不分页，返回该票全部报告期；模式B 才按 `page/page_size` 分页。
  - 数值字段大多成对（原值 + `_yoy` 同比），同比无值时为 null；末尾含衍生字段 `asset_liability_ratio`（资产负债率）。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | N | A 股代码，6 位数字 + 后缀（如 600519.SH）；存在则进入模式A 单票查询 |
| year | int | N | 年份（模式B 必填），如 2024 |
| report_type | string | N | 报告期类型（模式B 必填）：q1 / q2 / q3 / annual（兼容 h1） |
| page | int | N | 页码（模式B 必填），从 1 开始 |
| page_size | int | N | 每页条数（模式B 必填），默认 50，最大 500 |

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
| account_receivable | decimal | Y | 应收账款 |
| accounts_payable | decimal | Y | 应付账款 |
| accounts_payable_yoy | decimal | Y | 应付账款同比 |
| accounts_receivable_yoy | decimal | Y | 应收账款同比 |
| advance_receipts | decimal | Y | 预收账款 |
| advance_receipts_yoy | decimal | Y | 预收账款同比 |
| asset_liability_ratio | decimal | Y | 资产负债率（衍生） |
| cash_equivalents | decimal | Y | 货币资金 |
| inventory | decimal | Y | 存货 |
| inventory_yoy | decimal | Y | 存货同比 |
| monetary_funds_yoy | decimal | Y | 货币资金同比 |
| publish_date | string | Y | 发布日期 YYYY-MM-DD |
| report_form_type | string | Y | 报表类型（优先取「合并调整」） |
| report_type | string | Y | 报告期类型：q1 / q2 / q3 / annual |
| report_type_cn | string | Y | 报告期中文名 |
| stock_code | string | Y | A 股代码 |
| stock_name | string | Y | 股票名称 |
| t_assets | decimal | Y | 资产总计 |
| t_equity | decimal | Y | 所有者权益合计 |
| t_fixed_assets | decimal | Y | 固定资产合计 |
| t_liability | decimal | Y | 负债合计 |
| total_assets_yoy | decimal | Y | 资产总计同比 |
| total_equity_yoy | decimal | Y | 所有者权益合计同比 |
| total_liabilities_yoy | decimal | Y | 负债合计同比 |
| year | int | Y | 年份 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_balance","arguments":{"stock_code":"600519.SH"}}}')

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
                'ft_balance',
                {'stock_code': '600519.SH'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| stock_code | ind_name | ind_value | end_date | report_type |
|------|------|------|------|------|
| 600519.SH | null | null | null | q1 |
| 600519.SH | null | null | null | annual |
| 600519.SH | null | null | null | q3 |
| ... | ... | ... | ... | ... |
