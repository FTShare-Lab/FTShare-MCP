# 港股K线（MCP 工具 `ft_get_hk_candlesticks`）

> **MCP 工具**：`ft_get_hk_candlesticks`（category: `港股数据/行情数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：按 `trade_code` + 日期范围查询港股 K 线，返回 `HkCandlesticksResponse`（外层含 `trade_code` + `items` 数组）。底层来源表 `hkshareeodprices`。支持前复权/不复权，间隔单位仅支持 day/month/quarter/year（无分钟），`interval_value` 当前仅支持 1。`limit` 为可选保留最近 N 根。
- 数据范围：以服务端返回为准
- 单次限量：`limit` 可选，无默认值、无上限校验；日 K 在 SQL 层 `LIMIT` 下推，其它周期聚合后内存保留最近 `limit` 根
- 提示：
  - 对外 v1 仅暴露 GET（底层 v0 同时支持 GET+POST）。
  - 时间参数为日期（`since_date` / `until_date`，YYYY-MM-DD），非时间戳；与 F3 `hk-stock-candlesticks`（毫秒戳 + 3 天时间窗）是两套独立接口。
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

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| trade_code | string | Y | 港股代码（5 位数字 + `.HK`） |
| items | array[HkCandlestick] | Y | K 线数组 |

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

> MCP 工具名 `ft_get_hk_candlesticks`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
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
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

`trade_code=00700.HK`（腾讯控股）、`interval_unit=day`、`2026-06-10~17` 查询结果节选（K 线数组，按交易日倒序；核心列）：

| date | open | high | low | close | volume | turnover |
|------|------|------|-----|-------|--------|----------|
| 2026-06-16 | 462.60 | 462.60 | 445.40 | 447.40 | 24323142 | 10927700.96 |
| 2026-06-15 | 475.00 | 476.80 | 457.80 | 459.60 | 19748938 | 9161148.87 |
| ... | ... | ... | ... | ... | ... | ... |
