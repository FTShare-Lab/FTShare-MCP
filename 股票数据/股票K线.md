# 股票K线（MCP 工具 `ft_stock_candlesticks`）

> **MCP 工具**：`ft_stock_candlesticks`（category: `股票数据/行情数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取标的（股票/ETF/指数/债券）历史 K 线，返回**数组**（非分页）。日K典型返回每根 K 线含开高低收 + 成交量 + 成交额，价格单位元。支持分/日/周/月/年多种周期，支持前/后复权。
- 数据范围：1991 至今（跨深发展/浦发/茅台/万科等老股探测，1991 年起稳定有数据；1990 年开市初期窗口接口会报错，故下限取 1991）
- 单次限量：since/until 时间跨度 **≤3 天**，无分页，由 limit 控制返回条数
- 提示：
  - `symbol`、`interval_unit`、`until_ts_millis` 必填。
  - since/until 是**毫秒时间戳**，跨度硬限制 ≤3 天，超过会被拒。
  - 分时数据要拉长周期需分段循环调用。
  - 默认不复权（None），可设 Forward 前复权 / Backward 后复权。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | SymbolKey | Y | 标的代码，如 000001.SZ |
| interval_unit | enum | Y | 周期单位：Minute/Day/Week/Month/Year |
| interval_value | int | N | 间隔数值（默认1，如 Day+1=日K，Minute+5=5分钟） |
| adjust_kind | enum | N | 复权：None(默认,除权)/Forward(前复权)/Backward(后复权) |
| since_ts_millis | DateTime(ms) | N | 开始时间戳（毫秒）；与 until 跨度 ≤3 天 |
| until_ts_millis | DateTime(ms) | Y | 结束时间戳（毫秒） |
| limit | int | N | 返回条数上限 |

## 输出参数

数组，每根 K 线（StockCandlestick）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| open | decimal | Y | 开盘价，单位元 |
| high | decimal | Y | 最高价，单位元 |
| low | decimal | Y | 最低价，单位元 |
| close | decimal | Y | 收盘价（或最新价），单位元 |
| ts_millis | DateTime(ms) | Y | 收盘时间戳，单位毫秒 |
| ts_millis_open | DateTime(ms) | Y | 开盘时间戳，单位毫秒 |
| turnover | decimal | Y | 成交额，单位元 |
| volume | int64 | Y | 成交量 |

## 调用方法（MCP）

> MCP 工具名 `ft_stock_candlesticks`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_candlesticks","arguments":{"symbol": "600519.SH", "interval_unit": "day", "until_ts_millis": 1782370800000, "limit": 3}}}'
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
            res = await s.call_tool('ft_stock_candlesticks', {"symbol": "600519.SH", "interval_unit": "day", "until_ts_millis": 1782370800000, "limit": 3})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

平安银行 000001.SZ 近 3 个交易日日K：

| open | high | low | close | ts_millis | ts_millis_open | turnover | volume |
|------|------|-----|-------|-----------|----------------|----------|--------|
| 11.21 | 11.21 | 10.98 | 11.06 | 1781506800000 | 1781487000000 | 3423122573.14 | 308260990 |
| 11.05 | 11.12 | 10.91 | 10.94 | 1781593200000 | 1781573400000 | 2069108498.76 | 188541608 |
| 10.97 | 10.99 | 10.75 | 10.77 | 1781679240000 | 1781659800000 | 1015614977.55 | 93711016 |
| ... | ... | ... | ... | ... | ... | ... | ... |
