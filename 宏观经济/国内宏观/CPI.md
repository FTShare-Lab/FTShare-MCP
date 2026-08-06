# CPI（MCP 工具 `ft_consumer_price_index_monthly`）

> **MCP 工具**：`ft_consumer_price_index_monthly`（category: `宏观经济/国内宏观`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询中国居民消费价格指数（CPI）月度汇总计算结果。按月份返回全国（含累计同比）、城市、农村三档 CPI 同比/环比/当月值。元素为 `ComputedCpi`。提示：月份字段 `month` 已格式化为中文「YYYY年MM月份」；数值字段单位为百分比（%），部分月份字段缺失时为 null。
- 数据范围：宏观经济月度数据，以服务端返回为准
- 单次限量：全量月度序列一次返回，无分页上限
- 提示：
  - 月份字段 `month` 已格式化为中文「YYYY年MM月份」。
  - 数值字段单位为百分比（%），部分月份字段缺失时为 null。
  - 底层经 5s 瞬时失败策略缓存。

## 输入参数

无（v1 路由 `get_china_consumer_price_index_monthly` 不接收任何 query 参数）。

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| month | string | Y | 月份，格式「YYYY年MM月份」 |
| national_cpi | decimal | Y | 全国 CPI 当月值（%） |
| national_yoy | decimal | Y | 全国 CPI 同比（%） |
| national_mom | decimal | Y | 全国 CPI 环比（%） |
| cumulative | decimal | Y | 全国 CPI 累计同比（%） |
| city_cpi | decimal | Y | 城市 CPI 当月值（%） |
| city_yoy | decimal | Y | 城市 CPI 同比（%） |
| city_mom | decimal | Y | 城市 CPI 环比（%） |
| rural_cpi | decimal | Y | 农村 CPI 当月值（%） |
| rural_yoy | decimal | Y | 农村 CPI 同比（%） |
| rural_mom | decimal | Y | 农村 CPI 环比（%） |

## 调用方法（MCP）

> MCP 工具名 `ft_consumer_price_index_monthly`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_consumer_price_index_monthly","arguments":{}}}')

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
                'ft_consumer_price_index_monthly',
                {},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

全量月度序列按月份降序返回，节选最新一期（2026年05月份）核心字段：

| month | national_cpi | national_yoy | national_mom |
|-------|--------------|--------------|---------------|
| 2026年05月份 | 101.2 | 1.1858 | -0.1001 |
| ... | ... | ... | ... |
