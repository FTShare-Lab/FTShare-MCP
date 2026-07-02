# ETF盘前数据（MCP 工具 `ft_get_etf_pre`）

> **MCP 工具**：`ft_get_etf_pre`（category: `ETF专题`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取指定交易日的全部 ETF 盘前信息（申赎单位、净值、现金差额、是否公布 IOPV、申赎允许情况、预估现金部分等）。不传 `date` 时使用当日（CST），当日数据缓存在内存。返回 `EtfPreResponse`（trade_date + items 列表），每项与 data-protocol `ETFDailyPreDataV2` 字段一致。
- 数据范围：无时间维度（按交易日取），当日 / 指定交易日快照
- 单次限量：无分页，单交易日全部 ETF 盘前信息一次返回
- 提示：
  - 不传 `date` 用当日；传 `date` 用指定交易日（YYYYMMDD）。当日数据走内存缓存，同日重复请求直接命中。
  - `creation_redemption_flag` 为位掩码：1=允许申购，2=允许赎回，用位与判断。
  - `member_market_type` 为成分股类型位掩码：1=上交所 2=深交所 4=港交所 8=北交所 16=外汇 32=其他。
  - 真实数据样例需按公共服务返回补充。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| date | int | N | 交易日 YYYYMMDD；不传则使用当日（CST） |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| trade_date | int | Y | 交易日 YYYYMMDD |
| items | array[EtfPreItem] | Y | ETF 盘前信息列表 |

EtfPreItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| etf_symbol_id | int | Y | ETF 代码，如 510300 |
| etf_market_id | int16 | Y | ETF 交易所编号 |
| creation_redemption_unit | int64 | Y | 最小申购/赎回单位（份） |
| max_cash_ratio | f64 | Y | 现金替代比例上限 |
| publish_ipov | int8 | Y | 是否公布 IOPV：1=需要公布 |
| creation_redemption_flag | int8 | Y | 申赎允许位掩码：1=允许申购，2=允许赎回 |
| record_num | int32 | Y | 成分股数量 |
| estimate_cash_component | f64 | Y | 最小申赎单位预估现金部分（元） |
| trade_date | int32 | Y | 交易日 YYYYMMDD |
| cash_component | f64 | Y | 现金差额（元） |
| nav_per_cu | f64 | Y | 最小申赎单位净值（元） |
| nav | f64 | Y | 基金份额净值（元） |
| member_market_type | int32 | Y | 成分股类型位掩码 |

## 调用方法（MCP）

> MCP 工具名 `ft_get_etf_pre`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_etf_pre","arguments":{}}}'
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
            res = await s.call_tool('ft_get_etf_pre', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

trade_date 20260617，全市场 ETF 盘前数据（节选）：

| etf_symbol_id | etf_market_id | creation_redemption_unit | max_cash_ratio | nav | cash_component |
|---------------|---------------|--------------------------|----------------|-----|----------------|
| 159001 | 3554 | 1 | 1.0 | 100.0 | 0.0 |
| ... | ... | ... | ... | ... | ... |
