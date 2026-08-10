# ETF基础信息（MCP 工具 `ft_etf_description_all`）

> **MCP 工具**：`ft_etf_description_all`（category: `ETF专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取全部 ETF 的基础信息，包括标的代码、基金类型、托管人、流通份额、成立日期、管理人、是否支持融资融券与 T+0，以及跟踪指数信息。
- 数据范围：无时间维度，最新快照（ETF 静态档案）
- 单次限量：无入参、无分页，一次返回全部 ETF 基础信息
- 提示：
  - 无入参，直接 GET 即可。
  - `float_shares`、`tracking_index` 等为 `Option`，可能为 null。
  - `asset_class` 取值：stock（股票型）/ bond（债券型）/ commodity（商品型）/ currency（货币型）。

## 输入参数

无入参。

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| asset_class | string | Y | 基金类型：stock / bond / commodity / currency |
| custodian | string | Y | 基金托管人（银行） |
| float_shares | int64 | Y | 流通份额 |
| inception_date | date | Y | 成立日期 YYYY-MM-DD |
| management_company | string | Y | 基金管理人（公司） |
| marginable | bool | Y | 是否可融资融券 |
| name | string | Y | 标的名称（中文） |
| supports_t0 | bool | Y | 是否支持 T+0 交易 |
| symbol | string | Y | 标的代码，如 510300.XSHG |
| tracking_index | string | N | 跟踪指数名称 |
| tracking_index_id | string | N | 跟踪指数 ID |
| tracking_index_symbol | string | N | 跟踪指数代码 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_description_all","arguments":{}}}')

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
                'ft_etf_description_all',
                {},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| symbol | asset_class | name | inception_date | management_company | custodian |
|------|------|------|------|------|------|
| 159311.XSHE | stock | 数字经济ETF易方达 | 2025-07-07 | 易方达基金 | 交通银行 |
| 159212.XSHE | stock | 深100ETF南方 | 2025-03-27 | 南方基金 | 中国农业银行 |
| 159942.XSHE | stock | 中创100(退市) | 2015-05-25 | 华润元大基金 | 招商证券 |
| ... | ... | ... | ... | ... | ... |
