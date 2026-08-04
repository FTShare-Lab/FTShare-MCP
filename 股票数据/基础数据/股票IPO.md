# 股票IPO（MCP 工具 `ft_stock_ipos`）

> **MCP 工具**：`ft_stock_ipos`（category: `股票数据/基础数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取 A 股 IPO（新股发行）信息。每项含发行价、发行数量、网上发行数量、发行市盈率、行业市盈率、申购上限、申购代码、申购日期、上市日期等，带标的代码。提示：仅支持分页，不支持筛选；需带 total 的分页请用 `stock-ipos/paginated`。
- 数据范围：以服务端返回为准
- 单次限量：分页返回，由 page/page_size 控制（按 pagination 截断，不带 total）
- 提示：
  - 仅支持分页，不支持筛选；需带 total 的分页请用 `stock-ipos/paginated`。
  - `symbol` 输出格式受请求头 `X-Data-API-Symbol-Format` 控制。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
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

StockIpo：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码 |
| price | decimal | Y | 发行价格 |
| shares | i64 | Y | 发行数量 |
| online_shares | i64 | Y | 网上发行数量 |
| pe | decimal | Y | 发行市盈率 |
| industry_pe | decimal | Y | 行业市盈率 |
| max_subscription_shares | i64 | Y | 申购上限 |
| subscription_symbol_id | string | Y | 申购代码 |
| subscription_date | date | Y | 申购日期 |
| listing_date | date | Y | 上市日期 |

## 调用方法（MCP）

> MCP 工具名 `ft_stock_ipos`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_ipos","arguments":{}}}'
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
            res = await s.call_tool('ft_stock_ipos', {})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

page=1&page_size=2 节选：

| symbol | symbol_name | subscription_date | price | pe | shares | online_shares |
|--------|-------------|-------------------|-------|-----|--------|---------------|
| 001248.SZ | 华润新能源 | 2026-06-22 | null | null | 2423343000 | 632176000 |
| 920193.BJ | ... | 2026-06-17 | 8.52 | 14.98 | 28000000 | 25200000 |
| ... | ... | ... | ... | ... | ... | ... |
