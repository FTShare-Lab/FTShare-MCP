# 日频 OHLC（MCP 工具 `daily_ohlc`）

> `daily_ohlc` 是新版 MCP Server 提供的只读行情聚合工具。实时参数约束以 `tools/list` 返回的 `inputSchema` 为准。返回统一 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON。



## 输出格式

成功结果遵循仓库 README 中的统一输出契约：`structuredContent.metadata`（schema_version/source/tool/operation/total/returned/truncated/pagination/warnings）+ `structuredContent.data`。错误结果设置 `isError: true`、不带 `structuredContent`，`content[0].text` 返回 `{"error":{"code","message","field?","retryable","details?"}}` JSON。

## 支持口径

| `type` | 用途 | 关键参数 |
|--------|------|----------|
| `stock`（默认） | A 股日 OHLC | `symbol`（如 `600000.SH`）、`limit` |
| `daec_stock` | DAEC 日 OHLC | `symbol`、`since`+`until`（YYYYMMDD，跨度最大 40 年） |
| `hk_stock` | 港股日 K | `symbol`（如 `00700.HK`）、`until_date`（YYYY-MM-DD） |
| `us_stock` | 美股日 OHLC | `stock_code`（如 `AAPL`） |
| `eastmoney_board` | 东方财富板块日 OHLC | `board_code`（如 `BK0425`）、`start_date`+`end_date`（≤3 天） |
| `eastmoney_board_latest` | 东方财富板块最新 OHLC | `board_code` |
| `hk_index` | 港股指数日 K | `index_code`（如 `HSI`） |
| `global_index` | 全球指数日 K | `secid`（如 `100.N225`） |
| `ths_board` | 同花顺板块日 K | `board_code`（如 `881101`） |
| `ths_all_board` | 同花顺全板块日 K | `start_date`+`end_date` |

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| type | enum | N | 默认 `stock`，见上表 |
| symbol | string | 按 type | 证券外部码（`600584.SH`/`00700.HK`）；stock/daec_stock/hk_stock 用 |
| stock_code | string | 按 type | 美股代码（如 `AAPL`）；us_stock 用 |
| board_code | string | 按 type | 板块代码（`BK0425`/`885311`）；eastmoney_board/ths_board 用 |
| index_code | string | 按 type | 港股指数（如 `HSI`）；hk_index 用 |
| secid | string | 按 type | 全球指数 secid（如 `100.N225`）；global_index 用 |
| limit | int | N | 返回条数（stock/hk_stock） |
| start_date / end_date | string | 按 type | YYYY-MM-DD；eastmoney_board/us_stock 等区间，≤3 天 |
| since / until | string | 按 type | YYYYMMDD；daec_stock 区间 |
| until_date | string | 按 type | YYYY-MM-DD；hk_stock |
| adjust | string | N | daec 复权 `None`/`Forward`/`Backward` |
| page / page_size | int | N | 分页 |

### data 业务字段（type=stock）

| 名称 | 描述 |
|------|------|
| close | 业务字段 |
| high | 业务字段 |
| low | 业务字段 |
| open | 业务字段 |
| ts_millis | 业务字段 |
| ts_millis_open | 业务字段 |
| turnover | 业务字段 |
| volume | 业务字段 |

## 调用示例

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '\r')

curl -fsS -m 10 -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -sS -m 30 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"daily_ohlc","arguments":{"type": "stock", "symbol": "600000.SH", "limit": 2}}}'
```

**Python（`mcp` SDK，自动握手管理 session）**：

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def main():
    async with streamablehttp_client('<MCP_BASE_URL>') as (r, w, _):
        async with ClientSession(r, w) as s:
            await s.initialize()
            res = await s.call_tool('daily_ohlc', {"type": "stock", "symbol": "600000.SH", "limit": 2})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

