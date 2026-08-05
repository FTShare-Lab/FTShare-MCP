# 非凸股票评级Top5（MCP 工具 `ft_stock_rating_top5`）

> **MCP 工具**：`ft_stock_rating_top5`（category: `股票数据/特色数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询指定日期券单评级分市场 Top5（非凸数据来源）。按 `date` 返回该日期各分市场（all/xshg/xshe/bjse）评级 Top5 列表，含标的 symbolid、交易所 exid、等级、年化收益率、胜率等。档位 `variant` 默认 `300001`（30w01），可传 `300000`（30w）等；`type` 控制市场筛选，默认 `all`。提示：`date` 必填，格式 YYYYMMDD；`variant` 默认 `300001`；`type` 默认 `all`，可选 xshg / xshe / bjse。
- 数据范围：最新一期（按请求 date）
- 单次限量：每个市场 Top5
- 提示：
  - 数据来自非凸（来源标识 feitu）。
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

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| market | string | Y | 市场：`all` / `xshg` / `xshe` / `bjse` |
| items | array | Y | 该市场 Top5 记录 |

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

> MCP 工具名 `ft_stock_rating_top5`。MCP Streamable HTTP 要求先 `initialize` 获取 `Mcp-Session-Id`，再发送 `notifications/initialized`，最后调用 `tools/call`。

**curl**：

```bash
set -euo pipefail

MCP_BASE_URL="<MCP_BASE_URL>"

check_mcp_response() {
  local response=$1
  printf '%s\n' "$response"
  # MCP 业务与协议错误仍可能使用 HTTP 200，必须检查 JSON-RPC 响应。
  if printf '%s\n' "$response" | grep -Eq '"isError"[[:space:]]*:[[:space:]]*true|"error"[[:space:]]*:[[:space:]]*\{'; then
    return 1
  fi
}

SID=$(curl -fsS -m 60 -D - -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"ftshare-doc-example","version":"1.0.0"}}}' \
  | awk 'tolower($1)=="mcp-session-id:" {print $2}' \
  | tr -d '\r')

if [ -z "$SID" ]; then
  printf '%s\n' 'initialize 未返回 Mcp-Session-Id' >&2
  exit 1
fi

curl -fsS -m 60 -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

CALL_RESPONSE=$(curl -fsS -m 60 -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_rating_top5","arguments":{"date":"20260804","variant":"300001","type":"all"}}}')

check_mcp_response "$CALL_RESPONSE"
```

**Python（`mcp` SDK，自动握手管理 session）**：

```python
import asyncio

from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client


MCP_BASE_URL = "<MCP_BASE_URL>"


async def main():
    async with streamable_http_client(MCP_BASE_URL) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            result = await session.call_tool(
                "ft_stock_rating_top5",
                {'date': '20260804', 'variant': '300001', 'type': 'all'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

MCP 与公开 v1 的字段命名不同；MCP 保留中文业务字段。`date=20260804&variant=300001&type=all` 的真实响应节选：

```json
{"by_market":[{"market":"all","items":[{"日期":"20260804","symbolid":920258,"exid":6217,"等级":5.0,"年化收益率":5.0005,"胜率":1.0},{"日期":"20260804","symbolid":301677,"exid":3554,"等级":5.0,"年化收益率":1.7413,"胜率":1.0}]}]}
```
