# 东方财富板块最新OHLC（MCP 工具 `ft_eastmoney_board_latest_kline`）

> **MCP 工具**：`ft_eastmoney_board_latest_kline`（category: `股票数据/打板专题数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取东方财富概念板块的最新一日 OHLC 行情（开/收/高/低 + 成交量/成交额 + 振幅/涨跌幅/涨跌额/换手率）。传 `board_code` 返回单板块最新一根；不传则返回全部板块各自最新一根（分页）。提示：`board_code` 可选；不传时用于扫全部板块最新行情；`market` 为东财市场代码（如 90）。
- 数据范围：最新快照（最近一个交易日）
- 单次限量：单板块默认返回 1 行；不传 board_code 时返回全部板块最新行，由 page/page_size 分页
- 提示：
  - `board_code` 可选；不传时用于扫全部板块最新行情。
  - `market` 为东财市场代码（如 90）。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| board_code | string | N | 板块代码，如 BK1024；不传则返回全部板块最新K线 |
| page | int | N | 页码，从 1 开始 |
| page_size | int | N | 每页数量 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

EastmoneyBoardLatestKline：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| board_code | string | Y | 板块代码 |
| board_name | string | Y | 板块名称 |
| market | int | Y | 东财市场代码 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | decimal | Y | 开盘 |
| close | decimal | Y | 收盘 |
| high | decimal | Y | 最高 |
| low | decimal | Y | 最低 |
| volume | int64 | Y | 成交量 |
| turnover | decimal | Y | 成交额 |
| amplitude | float | Y | 振幅（%） |
| change_rate | float | Y | 涨跌幅（%） |
| change | decimal | Y | 涨跌额 |
| turnover_rate | float | Y | 换手率（%） |

## 调用方法（MCP）

> MCP 工具名 `ft_eastmoney_board_latest_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_board_latest_kline","arguments":{}}}'
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
            res = await s.call_tool('ft_eastmoney_board_latest_kline', {})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

BK1024 绿色电力最新交易日：

| board_code | board_name | market | date | open | close | high | low | volume | turnover | amplitude | change_rate | change | turnover_rate |
|------------|------------|--------|------|------|-------|------|-----|--------|----------|-----------|-------------|--------|---------------|
| BK1024 | 绿色电力 | 90 | 2026-06-16 | 1351.14 | 1351.87 | 1358.96 | 1340.30 | 99183764 | 86752614210 | 1.38 | -0.07 | -0.94 | 1.92 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
