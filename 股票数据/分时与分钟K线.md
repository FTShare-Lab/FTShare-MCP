# 分时与分钟 K 线（MCP 工具 `intraday_kline`）

> `intraday_kline` 是新版 MCP Server 提供的只读行情聚合工具。它把普通分钟 K、分时价格、DAEC 行情、批量 K 线和期货分钟 K 统一到一个 `type` 路由下。实时参数约束以 `tools/list` 返回的 `inputSchema` 为准。

## Tool descriptor

- `name`：`intraday_kline`
- `title`：分时与分钟 K 线
- `annotations`：`readOnlyHint=true`、`destructiveHint=false`、`openWorldHint=true`
- `_meta.securitySchemes`：`[{"type":"noauth"}]`
- 输出：统一 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON

## 支持的业务口径

| `type` | 用途 | 关键必填参数 | 主要限制 |
|--------|------|--------------|----------|
| `minute_kline`（默认） | 单标的 OHLC/K 线，按代码后缀自动路由 | `symbol` | `interval_unit=Minute` 时按 Asia/Shanghai 日期计，首尾均计入，最多覆盖 3 天 |
| `intraday_price` | 单标的分时价格线，非 OHLC | `symbol` | 不使用 K 线时间窗参数 |
| `daec_price` | DAEC 分时价格序列 | `symbol` | `days` 最大 30 个交易日；`ts_ms` 仅允许最近 31 天 |
| `daec_minute_ohlc` | DAEC OHLC/K 线 | `symbol`、`since`、`until` | `interval_unit=Minute` 时首尾日期均计入，最多覆盖 3 天 |
| `batch` | 批量标的 K 线 | `symbols`、`until_ts_millis` | 最多 500 个标的；分钟 K 按 Asia/Shanghai 日期最多覆盖 3 天 |
| `futures` | 期货分钟 K | 可选 `symbol` | `symbol` 需为具体合约代码（如 `RB2510`、`IF2510`）；连续主力代码（如 `RB0`、`AU0`、`IF0`）无数据，会返回空 `data`。`futures_interval` 默认 `1min` |

## 时间范围规则

分钟 K 的限制在三个入口保持一致：

- `type=minute_kline`：当 `interval_unit=Minute` 且同时提供起止毫秒时间戳时，先转换为 Asia/Shanghai 自然日期，首尾日期均计入，最多覆盖 3 天；即使实际时长不足 72 小时，跨 4 个日期也会被拒绝。
- `type=batch`：当 `interval_unit=Minute` 时遵循同一条自然日期限制；`until_ts_millis` 必填。
- `type=daec_minute_ohlc`：当 `interval_unit=Minute` 时首尾日期均计入。例如 `20260701` 到 `20260703` 允许，`20260701` 到 `20260704` 会被拒绝。
- `Day`、`Week`、`Month`、`Year` 不受上述“分钟 K 3 天”限制，但仍受各数据源自身范围约束。
- DAEC OHLC 支持 `Minute`、`Day`、`Week`、`Month`；非分钟 DAEC OHLC 的最大日期跨度为 40 年，`Year` 不适用于 `daec_minute_ohlc`。

`daec_price` 的 `days ≤30` 和 `ts_ms` 最近 31 天是分时价格查询的独立限制，不能与分钟 OHLC 的 3 天限制混用。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| `type` | enum | N | 默认 `minute_kline`；可选 `intraday_price`、`daec_price`、`daec_minute_ohlc`、`batch`、`futures` |
| `symbol` | string | 按 type | `minute_kline`、`intraday_price`、`daec_price`、`daec_minute_ohlc` 必填；示例 `600519.SH`。`futures` 口径传具体合约代码（如 `RB2510`），连续主力代码（`RB0` 等）无数据 |
| `symbols` | array[string] | 按 type | `batch` 必填，1～500 个标的 |
| `interval_unit` | enum | N | `Minute`、`Day`、`Week`、`Month`、`Year`，默认 `Minute`；DAEC OHLC 不支持 `Year` |
| `limit` | integer | N | 1～500；普通 GET 口径默认返回最近 40 条，按天查询会自动覆盖完整交易日 |
| `date` | string | N | 仅 `minute_kline`；`YYYYMMDD`。按该日 09:30～15:00 CST 查询，并优先于起止毫秒时间戳 |
| `adjust_kind` | enum | N | `None`、`Forward`、`Backward`，默认 `Forward`；用于普通单标的或批量 K 线 |
| `range` | string | N | 仅 `daec_price`；`Today` 或 `FiveDays`，默认 `FiveDays` |
| `days` | integer | N | 仅 `daec_price`；最多 30 个交易日 |
| `ts_ms` | integer | N | 仅 `daec_price`；起始毫秒时间戳，只允许最近 31 天 |
| `since` | string | 按 type | `daec_minute_ohlc` 必填，格式 `YYYYMMDD` |
| `until` | string | 按 type | `daec_minute_ohlc` 必填，格式 `YYYYMMDD` |
| `adjust` | string | N | 仅 DAEC OHLC；`None`、`Forward`、`Backward` |
| `since_ts_millis` | integer | N | 普通单标的或 `batch` 的开始毫秒时间戳 |
| `until_ts_millis` | integer | 按 type | `batch` 必填；`minute_kline` 不传时默认使用当前时间 |
| `futures_interval` | string | N | 仅 `futures`；如 `1min`、`5min`、`60min`，默认 `1min` |

未在 `inputSchema` 中声明的字段会被拒绝。各 `type` 不使用的可选字段不会参与该口径请求。

## 输出格式

成功结果遵循仓库 README 中的统一输出契约：

```json
{
  "metadata": {
    "schema_version": "1.0",
    "source": "ftshare",
    "tool": "intraday_kline",
    "operation": "minute_kline",
    "total": 0,
    "pagination": {
      "supported": false,
      "page": 1,
      "page_size": 1,
      "pages": 0,
      "has_more": false
    },
    "returned": 0,
    "truncated": false,
    "warnings": []
  },
  "data": []
}
```

- `metadata.operation` 与实际 `type` 对应，例如 `minute_kline`、`daec_minute_ohlc` 或 `batch`。
- `metadata.pagination.supported` 为 `false`；返回量由工具参数和服务端输出上限共同决定。
- `metadata.warnings` 可能说明 K 线去重、批量结构拍平、内外代码转换或结果截断。
- `data` 始终是数组，但元素字段随 `type` 变化：OHLC、价格点、批量拍平 K 线或期货 K 线。
- 成功业务 `data` 保持原业务字段和值，不做脱敏；对外错误信息会隐藏内部地址、路径和实现细节。

错误结果设置 `isError: true`，不返回 `structuredContent`。可结构化的业务错误会在 `content[0].text` 中返回 `{"error":{"code","message","field?","retryable","details?"}}` JSON；框架参数校验错误可能只返回脱敏文本。

## 调用示例

以下示例假设已经按 README 完成 `initialize` 和 `notifications/initialized`，并取得 `MCP_SESSION_ID`。

### 按交易日查询普通分钟 K

```bash
curl -sS -m 30 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"intraday_kline","arguments":{"type":"minute_kline","symbol":"600519.SH","date":"20260701"}}}'
```

### 查询 DAEC 分钟 OHLC

```python
import asyncio

from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client


async def main():
    async with streamablehttp_client("<MCP_BASE_URL>") as (read_stream, write_stream, _):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            result = await session.call_tool(
                "intraday_kline",
                {
                    "type": "daec_minute_ohlc",
                    "symbol": "600519.SH",
                    "interval_unit": "Minute",
                    "since": "20260701",
                    "until": "20260704",
                    "adjust": "Forward",
                },
            )
            print(result.structuredContent)


asyncio.run(main())
```

### 查询期货分钟 K

`futures` 口径的 `symbol` 必须是具体合约代码，连续主力代码无数据：

```bash
curl -sS -m 30 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"intraday_kline","arguments":{"type":"futures","symbol":"RB2510","futures_interval":"1min","limit":2}}}'
```

超过对应时间范围、缺少当前 `type` 的必填字段或传入未知字段时，工具调用会返回上述错误结果。
