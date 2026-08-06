# 指数K线（MCP 工具 `ft_index_candlesticks`）

> **MCP 工具**：`ft_index_candlesticks`（category: `指数专题/指数行情`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 为同值 JSON。

- 描述：查询单只指数的分钟、日、周、月或年 K 线，仅接受指数标的。
- 提示：分钟 K 的起止时间最多覆盖 3 个自然日；其他周期不受该限制。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | 指数代码，支持 `.XSHG/.XSHE` 和 `.SH/.SZ` |
| interval_unit | string | Y | `minute`、`day`、`week`、`month`、`year` |
| interval_value | integer | N | 分钟周期数值，最小为 1，仅分钟 K 生效；默认 1 |
| adjust_kind | string | N | `none`、`forward`、`backward`；默认 `none` |
| since_ts_millis | integer | N | 开始时间戳，毫秒 |
| until_ts_millis | integer | Y | 结束时间戳，毫秒 |
| limit | integer | N | 返回 K 线数量上限；未传 `since_ts_millis` 和 `limit` 时默认最多 50 根 |

## 输出参数

`data` 为 K 线数组：

| 字段 | 类型 | 描述 |
|------|------|------|
| open / high / low / close | string | 开盘、最高、最低、收盘点位 |
| ts_millis | integer | K 线结束时间戳（毫秒） |
| ts_millis_open | integer | K 线开始时间戳（毫秒） |
| turnover | string | 成交额 |
| volume | integer | 成交量 |

## 调用方法（MCP）

> MCP 工具名 `ft_index_candlesticks`。MCP Streamable HTTP 要求先 `initialize` 获取 `Mcp-Session-Id`，再发送 `notifications/initialized`，最后调用 `tools/call`。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_index_candlesticks","arguments":{"symbol":"000300.SH","interval_unit":"day","until_ts_millis":1782370800000,"limit":2}}}')

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
                'ft_index_candlesticks',
                {'symbol': '000300.SH',
                 'interval_unit': 'day',
                 'until_ts_millis': 1782370800000,
                 'limit': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

```json
{"open":"4950.9753","high":"5031.9994","low":"4944.8760","close":"5020.1038","ts_millis":1782370800000,"ts_millis_open":1782351000000,"turnover":"1110040892256.1000","volume":34666428900}
```
