# A股状态变更（MCP 工具 `ft_get_stk_status_change`）

> **MCP 工具**：`ft_get_stk_status_change`（category: `股票数据/基础数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 为同值 JSON。

- 描述：查询 A 股上市、暂停上市、终止上市等状态变更记录。
- 提示：过滤参数均可选；不传过滤条件时结果可能较大，不分页。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| trade_code | string | N | 带市场后缀的股票代码；可用逗号分隔多个代码 |
| change_date | string | N | 变更日期，`YYYYMMDD`，精确匹配 |
| change_type | string | N | 变更类型，例如上市、退市、暂停上市 |

## 输出参数

| 字段 | 类型 | 描述 |
|------|------|------|
| trade_code / name | string | 证券代码 / 名称 |
| change_date | string | 变更日期，`YYYYMMDD` |
| change_type | string | 状态变更类型 |
| change_details | string | 变更详情，可能为空字符串 |

## 调用方法（MCP）

> MCP 工具名 `ft_get_stk_status_change`。MCP Streamable HTTP 要求先 `initialize` 获取 `Mcp-Session-Id`，再发送 `notifications/initialized`，最后调用 `tools/call`。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_stk_status_change","arguments":{"trade_code":"600848.SH"}}}')

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
                "ft_get_stk_status_change",
                {'trade_code': '600848.SH'},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

```json
{"trade_code":"600848.SH","name":"上海临港","change_date":"19940324","change_type":"上市","change_details":""}
```
