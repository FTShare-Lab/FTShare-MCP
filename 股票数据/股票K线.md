# 股票K线（MCP 工具 `ft_stock_candlesticks`）

> **MCP 工具**：`ft_stock_candlesticks`（category: `股票数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取标的（股票/ETF/指数/债券）历史 K 线，返回数组（非分页）。支持分/日/周/月/年周期与前/后复权。提示：`symbol`、`interval_unit`、`until_ts_millis` 必填；分钟 K 按 Asia/Shanghai 自然日期计，首尾日期均计入，单次最多覆盖 3 天；日 K、周 K、月 K、年 K 不受该限制。
- 数据范围：1991 至今（跨深发展/浦发/茅台/万科等老股探测，1991 年起稳定有数据；1990 年开市初期窗口接口会报错，故下限取 1991）
- 单次限量：分钟 K 按 Asia/Shanghai 自然日期计，首尾日期均计入，单次最多覆盖 **3 天**；日 K、周 K、月 K、年 K 不受该限制；无分页，由 limit 控制返回条数
- 提示：
  - `symbol`、`interval_unit`、`until_ts_millis` 必填。
  - since/until 是**毫秒时间戳**；分钟 K 的首尾时间戳转换为 Asia/Shanghai 日期后，最多涉及 3 个不同日期。
  - 分钟 K 长周期数据需按自然日期分段调用。
  - 默认不复权（`none`），可设 `forward` 前复权或 `backward` 后复权。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | SymbolKey | Y | 标的代码，如 000001.SZ |
| interval_unit | enum | Y | 周期单位：`minute` / `day` / `week` / `month` / `year` |
| interval_value | int | N | 间隔数值（默认1，如 Day+1=日K，Minute+5=5分钟） |
| adjust_kind | enum | N | 复权：`none`（默认）/ `forward`（前复权）/ `backward`（后复权） |
| since_ts_millis | DateTime(ms) | N | 开始时间戳（毫秒）；分钟 K 按 Asia/Shanghai 自然日期计，首尾均计入，最多覆盖 3 天 |
| until_ts_millis | DateTime(ms) | Y | 结束时间戳（毫秒） |
| limit | int | N | 返回条数上限 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

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

> MCP 工具名 `ft_stock_candlesticks`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
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
