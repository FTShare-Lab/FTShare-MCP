# A股现金流量表（MCP 工具 `ft_cashflow`）

> **MCP 工具**：`ft_cashflow`（category: `股票数据/财务数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询 A 股上市公司现金流量表数据。返回宽表结构（每行一个报告期 + 多个财务科目列），数值字段可为空，优先取「合并调整」报表类型，没有则取「合并未调整」。同一工具支持两种模式：模式A 传 `stock_code` 查单票所有报告期；模式B 不传 `stock_code` 时必填 `year + report_type + page + page_size`，按单个报告期分页查所有票。提示：`stock_code` 格式为 6 位数字 + 交易所后缀（如 `000001.SZ`），模式A 与模式B 二选一；模式A 单票查询不分页，返回该票全部报告期；模式B 才按 `page/page_size` 分页。
- 数据范围：以服务端返回为准
- 单次限量：模式B 分页默认 `page=1`、`page_size=50`，最大 500（`validate_page` 校验）
- 提示：
  - `stock_code` 格式为 6 位数字 + 交易所后缀（如 `000001.SZ`），模式A 与模式B 二选一。
  - 模式A 单票查询不分页，返回该票全部报告期；模式B 才按 `page/page_size` 分页。
  - 数值字段每对（原值 + `_yoy` 同比）成组返回，同比字段无值时为 null。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | N | A 股代码，6 位数字 + 后缀（如 000001.SZ）；存在则进入模式A 单票查询 |
| year | int | N | 年份（模式B 必填），如 2024 |
| report_type | string | N | 报告期类型（模式B 必填）：q1 / q2 / q3 / annual |
| page | int | N | 页码（模式B 必填），从 1 开始 |
| page_size | int | N | 每页条数（模式B 必填），默认 50，最大 500 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

CashflowItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| stock_code | string | Y | A 股代码 |
| stock_name | string | Y | 股票名称 |
| year | int | Y | 年份 |
| report_type | string | Y | 报告期类型：q1 / q2 / q3 / annual |
| report_type_cn | string | Y | 报告期中文名 |
| publish_date | string | Y | 发布日期 YYYY-MM-DD |
| detail_report_type | string | Y | 报表明细类型（优先取「合并调整」） |
| cash_equ_inc | decimal | Y | 期末现金及现金等价物余额 |
| cash_equ_inc_yoy | decimal | Y | 期末现金及现金等价物余额同比 |
| net_oper_cash_flow | decimal | Y | 经营活动产生的现金流量净额 |
| net_oper_cash_flow_ratio | decimal | Y | 经营活动净额同比 |
| goods_sale_render_service_cash | decimal | Y | 销售商品、提供劳务收到的现金 |
| goods_sale_render_service_cash_ratio | decimal | Y | 销售商品提供劳务收到现金同比 |
| net_invest_cash_flow | decimal | Y | 投资活动产生的现金流量净额 |
| net_invest_cash_flow_ratio | decimal | Y | 投资活动净额同比 |
| invest_proceeds | decimal | Y | 投资活动收到的现金 |
| invest_proceeds_ratio | decimal | Y | 投资活动收到现金同比 |
| fix_intan_long_pay_cash | decimal | Y | 购建固定无形长期资产支付的现金 |
| fix_intan_long_pay_cash_ratio | decimal | Y | 购建固定无形长期资产支付现金同比 |
| net_fin_cash_flow | decimal | Y | 筹资活动产生的现金流量净额 |
| net_fin_cash_flow_ratio | decimal | Y | 筹资活动净额同比 |

## 调用方法（MCP）

> MCP 工具名 `ft_cashflow`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '\r')

curl -fsS -m 10 -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_cashflow","arguments":{"stock_code": "600519.SH", "page": 1, "page_size": 3}}}'
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
            res = await s.call_tool('ft_cashflow', {"stock_code": "600519.SH", "page": 1, "page_size": 3})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

> 示例调用已验证通过，真实样例可从 `structuredContent.data` 查看。

| stock_code | ind_name | ind_value | end_date | report_type |
|------------|----------|-----------|----------|-------------|
| ... | ... | ... | ... | ... |
