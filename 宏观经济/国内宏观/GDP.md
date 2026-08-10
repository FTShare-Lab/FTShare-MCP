# GDP（MCP 工具 `ft_consumer_gdp_quarterly`）

> **MCP 工具**：`ft_consumer_gdp_quarterly`（category: `宏观经济/国内宏观`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询中国 GDP 季度累计数据，覆盖 2000 年以来各季度，包含 GDP 总量及同比、第一/二/三产业增加值及各产业累计同比，以及货币单位和币种；季度标签采用「YYYY年第1-3季度」等累计口径，按年份降序排列。
- 数据范围：宏观经济季度数据
- 单次限量：全量季度序列一次返回，无分页上限
- 提示：
  - `period` 为季度累积口径标签（如「2025年第1-3季度」），非单季度。
  - 金额单位见 `unit`（通常为亿元），币种见 `currency`。

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
| currency | string | Y | 货币种类 |
| gdp | decimal | Y | GDP 总量（unit） |
| gdp_yoy | decimal | Y | GDP 同比（%） |
| period | string | Y | 季度标签，累积口径如「YYYY年第1-3季度」 |
| primary | decimal | Y | 第一产业增加值（unit） |
| primary_cumulative_yoy | decimal | Y | 第一产业累计同比（%） |
| secondary | decimal | Y | 第二产业增加值（unit） |
| secondary_cumulative_yoy | decimal | Y | 第二产业累计同比（%） |
| tertiary | decimal | Y | 第三产业增加值（unit） |
| tertiary_cumulative_yoy | decimal | Y | 第三产业累计同比（%） |
| unit | string | Y | 货币单位 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_consumer_gdp_quarterly","arguments":{}}}')

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
                'ft_consumer_gdp_quarterly',
                {},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| period | gdp | gdp_yoy | primary | secondary |
|------|------|------|------|------|
| 2026年第1-2季度 | 695704.0000 | 4.7000 | 31522.0000 | 250473.0000 |
| 2026年第1季度 | 334192.9000 | 5.0000 | 11941.0000 | 116135.0000 |
| 2025年第1-4季度 | 1401879.2000 | 5.0000 | 93347.0000 | 499653.0000 |
| ... | ... | ... | ... | ... |
