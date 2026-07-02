# 东方财富美股日OHLC（MCP 工具 `ft_eastmoney_us_stock_daily_kline`）

> **MCP 工具**：`ft_eastmoney_us_stock_daily_kline`（category: `美股数据/行情数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取东方财富**美股历史日 K 线**（OHLC），按 `stock_code` 读取对应 CSV 的全部行，全部字段以字符串返回，支持按日期范围过滤与分页。数据来源：东方财富，本地维护按 secid 命名的 CSV 文件。响应为 `PaginatedResponse`（非信封）。注意：日期范围（start/end）跨度上限为 **3 天**（后端强校验）。
- 数据范围：K 线时间序列，下限以服务端返回为准
- 单次限量：分页返回，默认 `page=1`、`page_size` 由后端定；区间跨度上限 3 天
- 提示：
  - `stock_code` 为必填，纯代码不带市场前缀（如 AAPL）；找不到时返回 400。
  - 日期格式支持 `YYYY-MM-DD` 或 `YYYYMMDD`，输出统一为 `YYYY-MM-DD`。
  - 区间过滤 `start_date`/`end_date` 同时提供时，跨度不得超过 3 天，否则返回 400。
  - 响应结构为 `PaginatedResponse`（items/total_pages/total_items），无 code/message 信封。
  - 所有数值字段均以字符串形式返回（含 OHLC、成交量、成交额、振幅、klt、fqt）。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | Y | 股票代码，如 AAPL |
| start_date | string | N | 起始日期（含），YYYY-MM-DD 或 YYYYMMDD；不传从最早开始 |
| end_date | string | N | 截止日期（含），YYYY-MM-DD 或 YYYYMMDD；不传到最晚 |
| page | int | N | 页码，从 1 开始 |
| page_size | int | N | 每页数量 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| items | array[EastmoneyUsStockDailyKlineRow] | Y | 日 K 线列表 |
| total_pages | int | Y | 总页数 |
| total_items | int | Y | 总记录数 |

EastmoneyUsStockDailyKlineRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| secid | string | Y | 证券 ID，格式 {market}.{code} |
| code | string | Y | 股票代码 |
| name | string | Y | 股票名称 |
| market | string | Y | 市场编号 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘价 |
| close | string | Y | 收盘价 |
| high | string | Y | 最高价 |
| low | string | Y | 最低价 |
| volume | string | Y | 成交量 |
| amount | string | Y | 成交额 |
| amplitude | string | Y | 振幅 |
| klt | string | Y | K 线类型（101=日 K） |
| fqt | string | Y | 复权类型（1=前复权） |

## 调用方法（MCP）

> MCP 工具名 `ft_eastmoney_us_stock_daily_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_us_stock_daily_kline","arguments":{"stock_code": "AAPL"}}}'
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
            res = await s.call_tool('ft_eastmoney_us_stock_daily_kline', {"stock_code": "AAPL"})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

> 示例调用已验证通过。表头结构如下：

| stock_code | trade_date | open | high | low | close |
|------------|------------|------|------|-----|-------|
| ... | ... | ... | ... | ... | ... |
