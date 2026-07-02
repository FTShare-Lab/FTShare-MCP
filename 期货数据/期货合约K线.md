# 期货合约K线（MCP 工具 `ft_futures_contract_kline`）

> **MCP 工具**：`ft_futures_contract_kline`（category: `期货数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询期货合约 K 线，返回 `KlineResponse{items,total}`。`symbol` 为 WIND 合约全码（如 `A2605.DCE`），store 会规范为点号前小写段。`interval` 默认 `1min`，支持 `1min/5min/15min/30min/60min/daily` 及由日线聚合的 `weekly/monthly/quarterly/yearly`（东八区；周以周五为周期末，月/季/年以末自然日为周期末；未满一周期则聚周期首日至数据内最后一根日线）。`start`/`end` 为毫秒时间戳，按聚合后时间戳筛选（闭区间）；不传则不限时间，仅 `start` 表示 `[start,+∞)`，禁止只传 `end`。`start`/`end` 同时传入时跨度须 ≤3 天。数据来自 ClickHouse 表 `basedata.futures_contract_kline`。
- 数据范围：以服务端返回为准
- 单次限量：`limit` 默认 500，控制返回条数；`start`/`end` 跨度 ≤3 天
- 提示：
  - `symbol` 必填。
  - `interval` 默认 `1min`；别名：`1m/5m/15m/30m/1h/1d/1w/1mo/1q/1y`。
  - `start`/`end` 为毫秒时间戳；同时传入时跨度硬限制 ≤3 天；只传 `end` 会被拒。
  - 不按时间过滤（省略 start/end）时全历史返回，条数受 `limit` 约束，需全量请显式增大 `limit`。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | WIND 合约全码，如 A2605.DCE |
| interval | string | N | 周期，默认 1min；可选 1min/5min/15min/30min/60min/daily/weekly/monthly/quarterly/yearly |
| start | int64 | N | 开始时间戳（毫秒）；与 end 跨度 ≤3 天；仅 start 表示 [start,+∞) |
| end | int64 | N | 结束时间戳（毫秒，闭区间）；须与 start 同时传入，禁止只传 end |
| limit | int | N | 最大返回条数，默认 500 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| items | array[KlineItem] | Y | K 线列表 |
| total | int | Y | 总条数 |

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

> MCP 工具名 `ft_futures_contract_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_futures_contract_kline","arguments":{"symbol": "600584.SH"}}}'
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
            res = await s.call_tool('ft_futures_contract_kline', {"symbol": "600584.SH"})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

参考股票 K 线结构，数据来自 ClickHouse 表 `basedata.futures_contract_kline`，`symbol` 为 WIND 合约全码（如 `A2605.DCE`），`start`/`end` 为毫秒时间戳、跨度须 ≤3 天：

| date | open | high | low | close | turnover | volume |
|------|------|------|-----|-------|----------|--------|
| ... | ... | ... | ... | ... | ... | ... |
