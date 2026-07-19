# 期货合约K线（MCP 工具 `ft_futures_contract_kline`）

> **MCP 工具**：`ft_futures_contract_kline`（category: `期货数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询单个期货合约的历史 K 线，返回开盘、收盘、最高、最低、成交量和成交额等指标。提示：`symbol` 必填，使用 WIND 合约全码，例如 `A2605.DCE`；`interval` 默认 `1min`，支持分钟、日、周、月、季和年周期；`start` 与 `end` 为毫秒时间戳，仅分钟周期同时传入时最多覆盖 3 个自然日，且不能只传 `end`。
- 数据范围：以服务端返回为准
- 单次限量：`limit` 默认 500；仅分钟周期最多覆盖 3 个自然日，日/周/月/季/年周期不受该限制
- 提示：
  - `symbol` 必填。
  - `interval` 默认 `1min`；别名：`1m/5m/15m/30m/1h/1d/1w/1mo/1q/1y`。
  - `start`/`end` 为毫秒时间戳；分钟周期按 Asia/Shanghai 自然日期计，首尾均计入，最多覆盖 3 天；只传 `end` 会被拒。
  - 不按时间过滤（省略 start/end）时全历史返回，条数受 `limit` 约束，需全量请显式增大 `limit`。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | WIND 合约全码，如 A2605.DCE |
| interval | string | N | 周期，默认 1min；可选 1min/5min/15min/30min/60min/daily/weekly/monthly/quarterly/yearly |
| start | int64 | N | 开始时间戳（毫秒）；仅分钟周期与 end 同传时最多覆盖 3 个自然日；仅 start 表示 [start,+∞) |
| end | int64 | N | 结束时间戳（毫秒，闭区间）；须与 start 同时传入，禁止只传 end |
| limit | int | N | 最大返回条数，默认 500 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

KlineItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 合约代码（规范为小写段） |
| datetime | int64 | Y | K 线时间戳（毫秒） |
| trade_date | int | Y | 交易日 YYYYMMDD |
| open | f64 | Y | 开盘价 |
| high | f64 | Y | 最高价 |
| low | f64 | Y | 最低价 |
| close | f64 | Y | 收盘价 |
| volume | int64 | Y | 成交量 |
| amount | f64 | Y | 成交额 |
| vwap | f64 | Y | 成交均价 |
| open_interest | f64 | Y | 持仓量 |
| dominant_contract | string | N | 主力合约（合约 K 线场景不返回） |
| forward_factor | f64 | N | 前复权因子（合约 K 线场景不返回） |
| backward_factor | f64 | N | 后复权因子（合约 K 线场景不返回） |

## 调用方法（MCP）

> MCP 工具名 `ft_futures_contract_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_futures_contract_kline","arguments":{"symbol": "A2605.DCE", "limit": 3}}}'
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
            res = await s.call_tool('ft_futures_contract_kline', {"symbol": "A2605.DCE", "limit": 3})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

参考股票 K 线结构，`symbol` 为 WIND 合约全码（如 `A2605.DCE`）；仅分钟周期的 `start`/`end` 最多覆盖 3 个自然日：

| date | open | high | low | close | turnover | volume |
|------|------|------|-----|-------|----------|--------|
| ... | ... | ... | ... | ... | ... | ... |
