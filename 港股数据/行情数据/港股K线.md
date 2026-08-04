# 港股K线（MCP 工具 `ft_get_hk_candlesticks`）

> **MCP 工具**：`ft_get_hk_candlesticks`（category: `港股数据/行情数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：按港股代码和日期范围查询历史 K 线，支持日、月、季和年周期以及前复权和不复权。提示：`interval_value` 仅支持 1；港股代码支持纯数字或 `.HK` 后缀。
- 数据范围：以服务端返回为准
- 单次限量：`limit` 可选，无默认值、无上限校验；日 K 在 SQL 层 `LIMIT` 下推，其它周期聚合后内存保留最近 `limit` 根
- 提示：
  - 对外 v1 仅暴露 GET（底层 v0 同时支持 GET+POST）。
  - 时间参数为日期（`since_date` / `until_date`，YYYY-MM-DD），非时间戳；与 F3 `hk-stock-candlesticks`（毫秒戳，且仅分钟周期最多覆盖 3 个自然日）是两套独立接口。
  - 港股代码内部规范化为 5 位数字 + `.HK`。
  - 复权 `adjust_kind` 默认 forward（前复权），仅 forward / none 两档。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| trade_code | string | Y | 港股代码，如 `00700.HK` 或 `700` |
| interval_unit | string | Y | 间隔单位：day / month / quarter / year |
| until_date | date | Y | 结束日期（YYYY-MM-DD） |
| since_date | date | N | 开始日期（YYYY-MM-DD） |
| interval_value | int | N | 间隔数值（当前仅支持 1） |
| limit | int | N | 数量限制（保留最近 N 根） |
| adjust_kind | string | N | 复权类型：forward(默认/前复权) / none(不复权) |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

HkCandlestick：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| date | date | Y | 交易日期 |
| open | decimal | Y | 开盘价 |
| high | decimal | Y | 最高价 |
| low | decimal | Y | 最低价 |
| close | decimal | Y | 收盘价 |
| volume | int64 | Y | 成交量 |
| turnover | decimal | Y | 成交额 |

## 调用方法（MCP）

> MCP 工具名 `ft_get_hk_candlesticks`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '\r')

curl -fsS -m 10 -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_hk_candlesticks","arguments":{"trade_code": "00700.HK", "interval_unit": "day", "until_date": "2026-06-23"}}}'
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
            res = await s.call_tool('ft_get_hk_candlesticks', {"trade_code": "00700.HK", "interval_unit": "day", "until_date": "2026-06-23"})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

`trade_code=00700.HK`（腾讯控股）、`interval_unit=day`、`2026-06-10~17` 查询结果节选（K 线数组，按交易日倒序；核心列）：

| date | open | high | low | close | volume | turnover |
|------|------|------|-----|-------|--------|----------|
| 2026-06-16 | 462.60 | 462.60 | 445.40 | 447.40 | 24323142 | 10927700.96 |
| 2026-06-15 | 475.00 | 476.80 | 457.80 | 459.60 | 19748938 | 9161148.87 |
| ... | ... | ... | ... | ... | ... | ... |
