# 全球指数日K线（MCP 工具 `ft_global_index_daily_kline`）

> **MCP 工具**：`ft_global_index_daily_kline`（category: `指数专题/指数行情`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取全球指数（含部分 A 股/港股宽基）的**历史日K线**，返回 `{total, items}` 结构（非分页，按时间区间一次性返回）。每根 K 线含开高低收 + 成交量 + 成交额 + 振幅 + 涨跌幅 + 涨跌额 + 换手率，价格字符串形式。`secid` 采用东方财富全球指数编码（如 `100.NDX` 纳斯达克、`100.DJIA` 道琼斯、`100.SPX` 标普500、`100.HSI` 恒生、`100.N225` 日经225）。注意：早期年份（如 1990 年代）的 `volume`/`amount`/`turnover` 多为 0（指数无成交量概念），近期数据成交量字段逐步有值。
- 数据范围：1990-01-01 至今（跨道琼斯/标普500/纳斯达克/恒生/日经多标的探测，标普500、道琼斯可至 1990-01-01，纳斯达克/恒生/日经 2000 年起，更早年份无数据）
- 单次限量：无分页，按 start_date/end_date 时间区间返回全部日K；建议区间控制在合理范围避免单次返回过大
- 提示：
  - `secid` 必填，东方财富全球指数编码格式，如 `100.NDX`、`100.DJIA`、`100.SPX`、`100.HSI`、`100.N225`。
  - 与「全球指数基础信息」接口的 symbol（`IXIC`/`DJI`/`HSI`）不是同一套编码，不可直接互用。
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

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| total | int | Y | 命中日K总数 |
| items | array | Y | 日K记录列表 |

GlobalIndexKlineItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| secid | string | Y | 东方财富指数编码，如 100.DJIA |
| code | string | Y | 指数代码，如 DJIA |
| name | string | Y | 指数中文名称，如「道琼斯」 |
| trade_date | string | Y | 交易日 YYYY-MM-DD |
| open | string | Y | 开盘价 |
| close | string | Y | 收盘价 |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| volume | int64 | Y | 成交量（早期常为 0） |
| amount | string | Y | 成交额（指数常为 "0.00"） |
| amplitude | string | Y | 振幅 % |
| change_pct | string | Y | 涨跌幅 % |
| change_amount | string | Y | 涨跌额 |
| turnover | string | Y | 换手率 %（指数常为 "0.0000"） |

## 调用方法（MCP）

> MCP 工具名 `ft_global_index_daily_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_global_index_daily_kline","arguments":{"secid": "100.DJIA"}}}'
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
            res = await s.call_tool('ft_global_index_daily_kline', {"secid": "100.DJIA"})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

道琼斯 100.DJIA 2024-01-02 至 2024-01-10 日K：

| code | name | trade_date | open | close | high | low | volume | change_pct |
|------|------|------------|------|-------|------|-----|--------|------------|
| DJIA | 道琼斯 | 2024-01-10 | 37552.91 | 37695.73 | 37740.77 | 37524.40 | 279540000 | 0.45 |
| DJIA | 道琼斯 | 2024-01-09 | 37523.55 | 37525.16 | 37552.38 | 37373.30 | 293230000 | -0.42 |
| DJIA | 道琼斯 | 2024-01-08 | 37327.37 | 37683.01 | 37692.92 | 37249.24 | 362200000 | 0.58 |
| DJIA | 道琼斯 | 2024-01-05 | 37455.46 | 37466.11 | 37623.62 | 37323.82 | 299490000 | 0.07 |
| DJIA | 道琼斯 | 2024-01-04 | 37425.28 | 37440.34 | 37716.41 | 37425.28 | 380220000 | 0.03 |
| DJIA | 道琼斯 | 2024-01-03 | 37629.23 | 37430.19 | 37629.23 | 37401.85 | 329150000 | -0.76 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
