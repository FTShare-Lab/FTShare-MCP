# 批量股票K线（MCP 工具 `ft_stock_candlesticks_batch`）

> **MCP 工具**：`ft_stock_candlesticks_batch`（category: `股票数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：批量获取多个标的历史 K 线（股票/ETF/指数/债券），返回每个标的的 K 线数组。提示：`symbols`、`interval_unit`、`until_ts_millis` 必填；分钟 K 按 Asia/Shanghai 自然日期计，首尾日期均计入，单次最多覆盖 3 天；日 K、周 K、月 K、年 K 不受该限制。
- 数据范围：以服务端返回为准（参考单标的日K：1991 至今）
- 单次限量：分钟 K 按 Asia/Shanghai 自然日期计，首尾日期均计入，单次最多覆盖 3 天；日 K、周 K、月 K、年 K 不受该限制；无分页，由 limit 控制每标的返回条数
- 提示：
  - `symbols`、`interval_unit`、`until_ts_millis` 必填。
  - since/until 是毫秒时间戳；分钟 K 的首尾时间戳转换为 Asia/Shanghai 日期后，最多涉及 3 个不同日期。
  - 需要长周期分钟数据时按自然日期分段调用，或对每个标的并发单接口请求。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbols | array[SymbolKey] | Y | 标的代码列表，如 ["000001.SZ","600000.SH"] |
| interval_unit | enum | Y | 周期单位：Minute/Day/Week/Month/Year |
| interval_value | int | N | 间隔数值（默认1） |
| adjust_kind | enum | N | 复权：None(默认)/Forward(前复权)/Backward(后复权) |
| since_ts_millis | DateTime(ms) | N | 开始时间戳（毫秒）；分钟 K 按 Asia/Shanghai 自然日期计，首尾均计入，最多覆盖 3 天 |
| until_ts_millis | DateTime(ms) | Y | 结束时间戳（毫秒） |
| limit | int | N | 每标的返回条数上限 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

数组，每项为标的 + 其 K 线列表。K 线字段（StockCandlestick）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码（外层） |
| open | decimal | Y | 开盘价，单位元 |
| high | decimal | Y | 最高价，单位元 |
| low | decimal | Y | 最低价，单位元 |
| close | decimal | Y | 收盘价（或最新价），单位元 |
| ts_millis | DateTime(ms) | Y | 收盘时间戳，单位毫秒 |
| ts_millis_open | DateTime(ms) | Y | 开盘时间戳，单位毫秒 |
| turnover | decimal | Y | 成交额，单位元 |
| volume | int64 | Y | 成交量 |

## 调用方法（MCP）

> MCP 工具名 `ft_stock_candlesticks_batch`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_candlesticks_batch","arguments":{"symbols": ["600519.SH"], "interval_unit": "day", "until_ts_millis": 1782370800000, "limit": 3}}}'
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
            res = await s.call_tool('ft_stock_candlesticks_batch', {"symbols": ["600519.SH"], "interval_unit": "day", "until_ts_millis": 1782370800000, "limit": 3})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

外层数组每项为 `[symbol, [K线数组]]`，节选 000001.XSHE 平安银行 日K：

| symbol | open | high | low | close | ts_millis | turnover | volume |
|--------|------|------|-----|-------|-----------|----------|--------|
| 000001.XSHE | 11.21 | 11.21 | 10.98 | 11.06 | 1781506800000 | 1711561286.57 | 154130495 |
| 000001.XSHE | 11.05 | 11.12 | 10.91 | 10.94 | 1781593200000 | 2069108498.76 | 188541608 |
| ... | ... | ... | ... | ... | ... | ... | ... |
