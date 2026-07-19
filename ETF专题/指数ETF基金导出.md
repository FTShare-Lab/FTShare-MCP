# 指数ETF基金导出（MCP 工具 `ft_etf_fund_export`）

> **MCP 工具**：`ft_etf_fund_export`（category: `ETF专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：智投 ETF 基金导出（含主表与持仓明细）。返回单只 ETF 的完整档案：基础属性（代码/名称/类型/风险等级）、规模与净值（tot_asset/aum/nv）、各区间收益率（日/6 月/年/3 年/5 年/成立以来）、换手率系列（日/周/月/年）、申赎限额、近一年收益率、30 日月均量额，以及聚合块 `total_return`（收益率汇总）/ `top10_holdings`（前十大持仓明细）/ `fund_managers`（历任基金经理）/ `nav_growth_rate`（净值增长）。提示：`request_id` 必填，由调用方生成，原样回填响应 `request_id`；`page_size` 传 0 按默认 50 处理。
- 数据范围：以服务端返回为准
- 单次限量：始终分页，`page` 默认 1、`page_size` 默认 50（避免全量 JSON 过大被网关截断）
- 提示：
  - `request_id` 必填，由调用方生成，原样回填响应 `request_id`。
  - `page_size` 传 0 按默认 50 处理。
  - 字段众多，下面仅列核心字段；收益率/换手率系列字段名见输出参数表。
  - 真实数据样例需按公共服务返回补充。

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
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| code | int | Y | 状态码：200 成功 / 404 数据不存在 / 500 内部错误 |
| msg | string | Y | 状态信息：OK / Not Found / Internal Server Error |
| request_id | string | Y | 回填请求的 request_id |
| data | object | Y | EtfFundExportData |
EtfFundExportItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| etf_code | string | Y | `{security_code}.{exchange}` 展示用代码 |
| security_id | string | Y | 证券 ID |
| security_code | string | Y | 证券代码 |
| security_name | string | Y | 证券名称 |
| exchange | string | Y | 交易所 |
| exabbr | string | N | 交易所简称（部分标的为空） |
| operat_mode | string | Y | 运作方式 |
| fund_type | string | Y | 基金类型 |
| risk_lvl | string | Y | 风险等级 |
| tot_asset / aum | f64 | N | 基金规模 / AUM（同义） |
| nv | f64 | N | 净值 |
| nav_pershare | f64 | N | 每份净值 |
| return_d / return_6m / return_y | f64 | N | 日 / 6 月 / 近一年收益率 |
| annureturn_3y / 5y / establ | f64 | N | 3 年 / 5 年 / 成立以来年化收益 |
| best_return | f64 | N | 近一年收益率（return_y 同义，前端单独字段） |
| fund_manager | string | Y | 基金经理 |
| is_inoffice | string | Y | 是否在职 |
| accesn_date | string | Y | 任职日期 |
| discount_prem / discount_premrt | f64 | N | 折溢价额 / 折溢价率 |
| day_turnover_rt ~ turnover_rt_y | f64 | N | 各周期换手率系列 |
| turnover_vol / turnover_val 及 _w/_m/_y | f64 | N | 各周期成交量 / 成交额系列 |
| avg_30d_volume / avg_30d_amount | f64 | N | 30 日月均成交量 / 成交额 |
| nav_peritprunit / ltredemp_unit | f64 | N | 申赎单位相关 |
| purchase_uppl / redemp_uppl 等限额 | f64 | N | 申购 / 赎回等限额系列 |
| subscription_redemption_rules | string | N | 申赎规则摘要（由限额字段拼接） |
| max_chargrt | f64 | N | 最高费率（%） |
| total_return | object | Y | 收益率汇总块（return_6m/return_y/annureturn_3y/5y/establ） |
| top10_holdings | array[object] | Y | 前十大持仓明细（publish_date/end_date/invest_obj/sec_code/sec_id/sec_name/hld_shares/mkt_value/ratioin_nv） |
| fund_managers | array[object] | Y | 历任基金经理列表（fund_manager/is_inoffice/accesn_date） |
| nav_growth_rate | object | Y | 净值增长块（return_d） |

## 调用方法（MCP）

> MCP 工具名 `ft_etf_fund_export`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_fund_export","arguments":{"request_id": "demo-req-001"}}}'
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
            res = await s.call_tool('ft_etf_fund_export', {"request_id": "demo-req-001"})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

as_of_date 2026-06-11，节选（字段众多，仅列核心列）：

| etf_code | security_code | security_name | fund_type | risk_lvl | tot_asset | return_y | annureturn_3y | fund_manager |
|----------|---------------|---------------|-----------|----------|-----------|----------|---------------|--------------|
| 510290.场外交易市场 | 510290 | 南方上证380ETF | 股票ETF基金 | 中 | 156685593.06 | 36.47 | 10.66 | 孙伟 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
