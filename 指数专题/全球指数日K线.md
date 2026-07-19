# 全球指数日K线（MCP 工具 `ft_global_index_daily_kline`）

> **MCP 工具**：`ft_global_index_daily_kline`（category: `指数专题/指数行情`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询全球指数历史日 K 线，返回开盘、收盘、最高、最低、成交量、成交额和涨跌幅等指标。提示：`secid` 必填，例如 `100.NDX`、`100.DJIA`、`100.SPX`、`100.HSI` 或 `100.N225`。
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

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

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

> MCP 工具名 `ft_global_index_daily_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
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
