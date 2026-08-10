# DAEC历史OHLC（MCP 工具 `ft_daec_ohlcs`）

> **MCP 工具**：`ft_daec_ohlcs`（category: `股票数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：按标的和日期区间查询历史 OHLC K 线，包括开盘价、最高价、最低价、收盘价；还可提供前收盘价及 MA5、MA10、MA20 均线。
- 数据范围：取决于标的历史行情数据
- 单次限量：标准模式按日期区间返回；兼容模式默认 `limit=250`
- 提示：
  - `symbol` 必填。
  - 标准模式下 `since`、`until` 必填，日期格式为 `YYYYMMDD`。
  - `compat=v2` 启用兼容版响应，此时由 `span` 指定周期、`limit` 控制返回条数。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | 标的代码，如 `600000.XSHG` |
| since | string | N | 起始日期，`YYYYMMDD` |
| until | string | N | 结束日期，`YYYYMMDD` |
| interval | string | N | 周期：`Minute`/`Day`/`Week`/`Month`，默认 `Day` |
| adjust | string | N | 复权：`None`/`Forward`/`Backward` |
| compat | string | N | 传 `v2` 启用兼容版响应 |
| span | string | N | 兼容模式周期：`DAY1`/`WEEK1`/`MONTH1`，默认 `DAY1` |
| limit | int | N | 兼容模式返回数量，默认 250 |
| until_ts_ms | int64 | N | 兼容模式结束时间戳，毫秒；优先于默认当前日期 |

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
| current_time | string | Y | 当前时间，RFC3339 带时区 |
| has_last_empty | bool | Y | 是否存在末尾空值标记 |
| ma10 | array | Y | MA10 数组，元素 `{ctm, p}` |
| ma20 | array | Y | MA20 数组，元素 `{ctm, p}` |
| ma5 | array | Y | MA5 数组，元素 `{ctm, p}` |
| ohlcs | array | Y | K 线数组，元素字段见下表 |
| prev_close | number | Y | 前收盘价 |

ohlcs 元素（兼容版缩写字段）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| o | string | Y | 开盘价 |
| h | string | Y | 最高价 |
| l | string | Y | 最低价 |
| c | string | Y | 收盘价 |
| v | int64 | Y | 成交量 |
| t | string | Y | K 线时间 |
| otm | string | Y | 成交额 |
| ctm | int64 | Y | 时间戳，毫秒 |

ma5 / ma10 / ma20 元素：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| ctm | int64 | Y | 时间戳，毫秒 |
| p | string | Y | 均价 |

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_daec_ohlcs","arguments":{"symbol":"600000.XSHG","compat":"v2","span":"DAY1","limit":3}}}')

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
                'ft_daec_ohlcs',
                {'symbol': '600000.XSHG', 'compat': 'v2', 'span': 'DAY1', 'limit': 3},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| prev_close | current_time | ohlcs（o/h/l/c） | ma5 | ... |
|------|------|------|------|------|
| 9.26 | 2026-08-10T18:19:59.910366764+08:00 | null | [{'ctm': 1786031999999, 'p': None}, {'ctm': 1786118399999, '… | null |
| ... | ... | ... | ... | ... |
