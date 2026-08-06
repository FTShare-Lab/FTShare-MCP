# 日频 OHLC（MCP 工具 `daily_ohlc`）

> `daily_ohlc` 是只读日频行情聚合工具。本文参数与 2026-08-05 公共服务 `tools/list` 返回的 `inputSchema` 对齐；实时约束仍以调用时的 `tools/list` 为准。返回统一 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON。

## 输出格式

成功结果遵循仓库 README 中的统一输出契约：`structuredContent.metadata`（schema_version/source/tool/operation/total/returned/truncated/pagination/warnings）+ `structuredContent.data`。错误结果设置 `isError: true`、不带 `structuredContent`，`content[0].text` 返回 `{"error":{"code","message","field?","retryable","details?"}}` JSON。

## 支持口径

| `type` | 用途 | 关键参数 |
|--------|------|----------|
| `stock`（默认） | A 股日 OHLC | `symbol`（如 `600000.SH`）、`limit` |
| `hk_stock` | 港股日 K | `symbol`（如 `00700.HK`）、`until_date`（YYYY-MM-DD） |
| `us_stock` | 美股日 OHLC | `stock_code`（如 `AAPL`）；可传 `start_date`、`end_date` |
| `eastmoney_board` | 东方财富板块日 OHLC | `board_code`（如 `BK0425`）、`start_date`+`end_date`（≤3 天） |
| `eastmoney_board_latest` | 东方财富板块最新 OHLC | 可选分页参数 |
| `hk_index` | 港股指数日 K | `index_code`（如 `HSI`）；可传日期区间 |
| `global_index` | 全球指数日 K | `secid`（如 `100.N225`）；可传日期区间 |
| `ths_board` | 同花顺板块日 K | `board_code`（如 `881101`） |
| `ths_all_board` | 同花顺全板块日 K | 可选分页参数 |

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| type | enum | N | 默认 `stock`；可选值见上表 |
| symbol | string | 按 type | 证券外部码，如 `600584.SH`、`00700.HK`；`stock` / `hk_stock` 必填 |
| stock_code | string | 按 type | 美股代码，如 `AAPL`；`us_stock` 必填 |
| board_code | string | 按 type | 板块代码，如 `BK0425`、`885311`；`eastmoney_board` / `ths_board` 必填 |
| index_code | string | 按 type | 港股指数代码，如 `HSI`；`hk_index` 必填 |
| secid | string | 按 type | 全球指数 secid，如 `100.N225`；`global_index` 必填 |
| limit | integer | N | 返回条数，仅 `stock` / `hk_stock` 使用；范围 1～500 |
| start_date | string | N | 区间起始日期 `YYYY-MM-DD`；用于 `us_stock` / `hk_index` / `global_index` / `eastmoney_board` |
| end_date | string | N | 区间截止日期 `YYYY-MM-DD`；`us_stock` / `eastmoney_board` 与 `start_date` 同传时跨度不能超过 3 天 |
| until_date | string | 按 type | 港股截止日期 `YYYY-MM-DD`；`hk_stock` 必填 |
| page | integer | N | 页码，最小 1 |
| page_size | integer | N | 每页条数，范围 1～500 |

`daily_ohlc` 已不再支持 `type="daec_stock"`，也不再接受 `since`、`until` 或 `adjust` 字段。DAEC 行情由独立的 `ft_daec_*` 工具提供；旧 `type` 值会因不在当前枚举中被拒绝，旧字段会因 `additionalProperties=false` 被拒绝。

### data 业务字段（type=stock）

| 名称 | 描述 |
|------|------|
| close | 收盘价 / 最新价 |
| high | 最高价 |
| low | 最低价 |
| open | 开盘价 |
| ts_millis | 收盘时间戳，毫秒 |
| ts_millis_open | 开盘时间戳，毫秒 |
| turnover | 成交额 |
| volume | 成交量 |

## 调用示例

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"daily_ohlc","arguments":{"type":"stock","symbol":"600000.SH","limit":2}}}')

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
                'daily_ohlc',
                {'type': 'stock', 'symbol': '600000.SH', 'limit': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```
