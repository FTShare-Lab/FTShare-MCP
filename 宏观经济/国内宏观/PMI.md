# PMI（MCP 工具 `ft_consumer_pmi_monthly`）

> **MCP 工具**：`ft_consumer_pmi_monthly`（category: `宏观经济/国内宏观`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询中国采购经理指数（PMI）月度数据，覆盖 2000 年以来各月，包含制造业与非制造业 PMI 当月值、同比和环比，按年份降序排列。
- 数据范围：宏观经济月度数据
- 单次限量：全量月度序列一次返回，无分页上限
- 提示：
  - `month` 格式化为「YYYY年MM月份」。
  - PMI 数值单位为百分比（%），缺失为 null。

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
| manufacturing_mom | decimal | Y | 制造业 PMI 环比（%） |
| manufacturing_pmi | decimal | Y | 制造业 PMI 当月值（%） |
| manufacturing_yoy | decimal | Y | 制造业 PMI 同比（%） |
| month | string | Y | 月份，格式「YYYY年MM月份」 |
| non_manufacturing_mom | decimal | Y | 非制造业 PMI 环比（%） |
| non_manufacturing_pmi | decimal | Y | 非制造业 PMI 当月值（%） |
| non_manufacturing_yoy | decimal | Y | 非制造业 PMI 同比（%） |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_consumer_pmi_monthly","arguments":{}}}')

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
                'ft_consumer_pmi_monthly',
                {},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| month | manufacturing_pmi | non_manufacturing_pmi |
|------|------|------|
| 2026年07月份 | 49.2000 | 49.0000 |
| 2026年06月份 | 50.3000 | 50.2000 |
| 2026年05月份 | 50.0000 | 50.1000 |
| ... | ... | ... |
