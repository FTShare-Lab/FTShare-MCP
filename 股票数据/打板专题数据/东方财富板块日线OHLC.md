# 东方财富板块日线OHLC（MCP 工具 `ft_eastmoney_board_daily_kline`）

> **MCP 工具**：`ft_eastmoney_board_daily_kline`（category: `股票数据/打板专题数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询单个东方财富概念板块的历史日线 OHLC，返回开盘、收盘、最高、最低、成交量、成交额、振幅、涨跌幅和换手率等指标。提示：`board_code` 必填；`start_date` 和 `end_date` 可选，支持 YYYY-MM-DD 或 YYYYMMDD；支持分页。
- 数据范围：2013-04 至今（跨 BK0447(2013-04-09)/BK0700(2020-06-24)/BK0733(2021-03-09)/BK1024(2021-10-18) 等多板块探测，最早 2013-04-09）
- 单次限量：由 page/page_size 分页
- 提示：
  - `board_code` 必填。
  - `start_date` / `end_date` 可选；同时传入时结束日期不能早于开始日期，且跨度不超过 3 天（按日期差值计，超出返回 `INVALID_ARGUMENT "时间范围不能超过 3 天"`）。
  - 与「最新OHLC」接口不同：本接口所有字段（含 market/volume 等）一律按字符串返回。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| board_code | string | Y | 板块代码，如 BK1024 |
| start_date | string | N | 起始日期（含），YYYY-MM-DD 或 YYYYMMDD |
| end_date | string | N | 截止日期（含），YYYY-MM-DD 或 YYYYMMDD |
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

EastmoneyBoardDailyKlineRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| board_code | string | Y | 板块代码 |
| board_name | string | Y | 板块名称 |
| market | string | Y | 东财市场代码 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘 |
| close | string | Y | 收盘 |
| high | string | Y | 最高 |
| low | string | Y | 最低 |
| volume | string | Y | 成交量 |
| turnover | string | Y | 成交额 |
| amplitude | string | Y | 振幅（%） |
| change_rate | string | Y | 涨跌幅（%） |
| change | string | Y | 涨跌额 |
| turnover_rate | string | Y | 换手率（%） |

## 调用方法（MCP）

> MCP 工具名 `ft_eastmoney_board_daily_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_board_daily_kline","arguments":{"board_code": "BK0425"}}}'
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
            res = await s.call_tool('ft_eastmoney_board_daily_kline', {"board_code": "BK0425"})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

BK1024 绿色电力 2026-06-13 至 2026-06-16（节选）：

| board_code | board_name | market | date | open | close | high | low | volume | turnover | amplitude | change_rate | change | turnover_rate |
|------------|------------|--------|------|------|-------|------|-----|--------|----------|-----------|-------------|--------|---------------|
| BK1024 | 绿色电力 | 90 | 2026-06-15 | 1339.13 | 1352.81 | 1354.47 | 1338.45 | 100155280 | 88052995344 | 1.2 | 1.29 | 17.25 | 1.94 |
| BK1024 | 绿色电力 | 90 | 2026-06-16 | 1351.14 | 1351.87 | 1358.96 | 1340.3 | 99183764 | 86752614210 | 1.38 | -0.07 | -0.94 | 1.92 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... | ... |
