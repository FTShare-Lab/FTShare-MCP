# A股利润表（MCP 工具 `ft_income`）

> **MCP 工具**：`ft_income`（category: `股票数据/财务数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询 A 股上市公司利润表数据，底表 `ashare_income`。返回**宽表**结构（每行一个报告期 + 多个财务科目列），数值字段为 `Option<Decimal>`，优先取「合并调整」报表类型，没有则取「合并未调整」。同一路径支持两种模式：模式A 传 `stock_code` 查单票所有报告期；模式B 不传 `stock_code` 时必填 `year + report_type + page + page_size`，按单个报告期分页查所有票。`report_type` 取值 `q1` / `q2` / `q3` / `annual`（兼容 `h1` → `q2`）。
- 数据范围：以服务端返回为准
- 单次限量：模式B 分页默认 `page=1`、`page_size=50`，最大 500（`validate_page` 校验）
- 提示：
  - `stock_code` 格式为 6 位数字 + 交易所后缀（如 `000001.SZ`），模式A 与模式B 二选一。
  - 模式A 单票查询不分页，返回该票全部报告期；模式B 才按 `page/page_size` 分页。
  - 数值字段每对（原值 + `_yoy` 同比）成组返回，同比无值时为 null。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | N | A 股代码，6 位数字 + 后缀（如 000001.SZ）；存在则进入模式A 单票查询 |
| year | int | N | 年份（模式B 必填），如 2024 |
| report_type | string | N | 报告期类型（模式B 必填）：q1 / q2 / q3 / annual（兼容 h1） |
| page | int | N | 页码（模式B 必填），从 1 开始 |
| page_size | int | N | 每页条数（模式B 必填），默认 50，最大 500 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| items | array | Y | 当前页记录列表 |
| total_pages | int | Y | 总页数（模式A 有数据为 1，否则 0） |
| total_items | int | Y | 总记录数（模式A 为该票报告期数） |

IncomeItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| stock_code | string | Y | A 股代码 |
| stock_name | string | Y | 股票名称 |
| year | int | Y | 年份 |
| report_type | string | Y | 报告期类型：q1 / q2 / q3 / annual |
| report_type_cn | string | Y | 报告期中文名 |
| publish_date | string | Y | 发布日期 YYYY-MM-DD |
| report_form_type | string | Y | 报表类型（优先取「合并调整」） |
| t_revenue | decimal | Y | 营业总收入 |
| t_revenue_yoy | decimal | Y | 营业总收入同比 |
| cost | decimal | Y | 营业成本 |
| cost_yoy | decimal | Y | 营业成本同比 |
| t_cost | decimal | Y | 营业总成本 |
| t_cost_yoy | decimal | Y | 营业总成本同比 |
| sale_expense | decimal | Y | 销售费用 |
| sale_expense_yoy | decimal | Y | 销售费用同比 |
| manag_expense | decimal | Y | 管理费用 |
| manag_expense_yoy | decimal | Y | 管理费用同比 |
| financial_cost | decimal | Y | 财务费用 |
| financial_cost_yoy | decimal | Y | 财务费用同比 |
| profit | decimal | Y | 营业利润 |
| profit_yoy | decimal | Y | 营业利润同比 |
| t_profit | decimal | Y | 利润总额 |
| t_profit_yoy | decimal | Y | 利润总额同比 |
| n_profit | decimal | Y | 净利润（含少数股东损益） |
| n_profit_yoy | decimal | Y | 净利润同比 |
| parcomp_n_profit | decimal | Y | 归属于母公司股东的净利润 |
| parcomp_n_profit_yoy | decimal | Y | 归母净利润同比 |

## 调用方法（MCP）

> MCP 工具名 `ft_income`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_income","arguments":{"stock_code": "600519.SH", "page": 1, "page_size": 3}}}'
```

**Python（`mcp` SDK，自动握手管理 session）**：

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def main():
    async with streamablehttp_client('<MCP_BASE_URL>') as (r, w, _):
        async with ClientSession(r, w) as s:
            await s.initialize()
            res = await s.call_tool('ft_income', {"stock_code": "600519.SH", "page": 1, "page_size": 3})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

> 示例调用已验证通过，真实样例可按返回的 markdown 表格查看。

| stock_code | ind_name | ind_value | end_date | report_type |
|------------|----------|-----------|----------|-------------|
| ... | ... | ... | ... | ... |
