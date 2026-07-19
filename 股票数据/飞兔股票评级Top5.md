# 飞兔股票评级Top5（MCP 工具 `ft_stock_rating_top5`）

> **MCP 工具**：`ft_stock_rating_top5`（category: `股票数据/特色数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询指定日期券单评级分市场 Top5（飞兔数据来源）。按 `date` 返回该日期各分市场（all/xshg/xshe/bjse）评级 Top5 列表，含标的 symbolid、交易所 exid、等级、年化收益率、胜率等。档位 `variant` 默认 `300001`（30w01），可传 `300000`（30w）等；`type` 控制市场筛选，默认 `all`。提示：`date` 必填，格式 YYYYMMDD；`variant` 默认 `300001`；`type` 默认 `all`，可选 xshg / xshe / bjse。
- 数据范围：最新一期（按请求 date）
- 单次限量：每个市场 Top5
- 提示：
  - 数据来自飞兔（来源标识 feitu）。
  - `date` 必填，格式 YYYYMMDD。
  - `variant` 默认 `300001`；`type` 默认 `all`，可选 xshg / xshe / bjse。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| date | string | Y | 日期 YYYYMMDD |
| variant | string | N | 档位，如 300001（30w01）、300000（30w），默认 300001 |
| type | StockRatingMarketFilter | N | 市场：all / xshg / xshe / bjse，默认 all |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

StockRatingTop5Response：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| by_market | array[StockRatingMarket] | Y | 按市场分组的 Top5 列表 |

StockRatingMarket：

StockRatingRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| 日期 | string | Y | 日期 |
| symbolid | int | Y | 标的 ID |
| exid | int | Y | 交易所 ID |
| 等级 | float | Y | 评级等级 |
| 年化收益率 | float | Y | 年化收益率 |
| 胜率 | float | Y | 胜率 |

## 调用方法（MCP）

> MCP 工具名 `ft_stock_rating_top5`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_rating_top5","arguments":{"date": "20260624"}}}'
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
            res = await s.call_tool('ft_stock_rating_top5', {"date": "20260624"})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

`date=20260617&variant=300001&type=all`，by_market 结构示例（market=all）：

| date | symbol_id | rank | annual_return | win_rate |
|------|-----------|------|---------------|----------|
| 20260617 | 920126 | 5.0 | 4.9799 | 1.0 |
| 20260617 | 688146 | 5.0 | 2.2075 | 0.8095 |
| ... | ... | ... | ... | ... |
