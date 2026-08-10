# 非凸股票评级Top5（MCP 工具 `ft_stock_rating_top5`）

> **MCP 工具**：`ft_stock_rating_top5`（category: `股票数据/特色数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询指定日期各市场的券单评级 Top5，数据来源为非凸。覆盖全市场、上交所、深交所和北交所，信息包括标的代码、交易所、评级等级、年化收益率和胜率等。`variant` 用于选择评级档位，默认为 `300001`（30w01）；`type` 用于筛选市场，默认为 `all`。
- 数据范围：当日数据（仅支持查询当天日期；历史日期返回空）
- 单次限量：每个市场 Top5
- 提示：
  - 数据来自非凸（来源标识 feitu）。
  - `date` 必填，格式 YYYYMMDD；仅当天有效，历史日期返回空。
  - `variant` 默认 `300001`；`type` 默认 `all`，可选 xshg / xshe / bjse。
  - 判断是否有业务数据须检查 `by_market[].items` 是否非空，而非外层 `returned`（空 `by_market` 时外层仍计 1）。

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
| data | array | Y | 归一化后的业务数据项 |

### data 业务字段

data 为单元素数组，元素含按市场分组的列表：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| by_market | array | Y | 按市场分组的 Top5 列表 |

by_market 元素及 items 字段：

| 名称 | 类型 | 描述 |
|------|------|------|
| market | string | 市场标识（all / xshg / xshe / bjse） |
| items | array | 该市场 Top5 记录 |
| exid | int | 交易所代码（items 内） |
| symbolid | int | 标的代码（items 内） |
| 日期 | string | YYYYMMDD（items 内） |
| 等级 | number | 评级等级（items 内） |
| 年化收益率 | number | 年化收益率（items 内） |
| 胜率 | number | 胜率（items 内） |

## 调用方法（MCP）

> MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

**curl**：

```bash
set -euo pipefail

MCP_BASE_URL="<MCP_BASE_URL>"

check_mcp_response() {
  local response=$1
  printf '%s\n' "$response"
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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_stock_rating_top5","arguments":{"date":"20260810","variant":"300001","type":"all"}}}')

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
                'ft_stock_rating_top5',
                {'date': '20260810', 'variant': '300001', 'type': 'all'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

`date=20260810, variant=300001, type=all` 返回 `by_market` 1 组（market=all），`items` 5 条，节选：

| exid | symbolid | 日期 | 等级 | 年化收益率 | 胜率 |
|------|------|------|------|------|------|
| 3553 | 603468 | 20260810 | 5.0 | 1.651 | 1.0 |
| 6217 | 920038 | 20260810 | 5.0 | 1.4798 | 1.0 |
| 3554 | 300795 | 20260810 | 5.0 | 1.4531 | 0.9048 |
| ... | ... | ... | ... | ... | ... |
