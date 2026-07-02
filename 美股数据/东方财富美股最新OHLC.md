# 东方财富美股最新OHLC（MCP 工具 `ft_eastmoney_us_stock_latest_kline`）

> **MCP 工具**：`ft_eastmoney_us_stock_latest_kline`（category: `美股数据/行情数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取东方财富**美股最新一根日 K 线**（取每只标的 CSV 的最后一行 OHLC）。传 `stock_code` 返回单只标的最新 K 线；不传则遍历缓存目录返回全部美股最新 K 线。数据来源：东方财富，本地维护按 secid 命名的 CSV 文件。响应为 `PaginatedResponse`（非信封）。
- 数据范围：最新快照（无时间维度，每标的仅最新一个交易日）
- 单次限量：分页返回，默认 `page=1`、`page_size` 由后端定
- 提示：
  - 不传 `stock_code` 时会扫描全部美股 K 线文件，响应较慢，建议优先按 `stock_code` 精准查询。
  - `stock_code` 为不带市场前缀的纯代码（如 ADV），找不到时返回 400 错误。
  - 响应结构为 `PaginatedResponse`（items/total_pages/total_items），无 code/message 信封。
  - 字段 `date` 为 ISO 日期，OHLC/amount 为 Decimal 字符串，volume 为整数。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| stock_code | string | N | 股票代码，如 ADV；不传返回全部美股最新 K 线 |
| page | int | N | 页码，从 1 开始 |
| page_size | int | N | 每页数量 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| items | array[EastmoneyUsStockLatestKline] | Y | 最新 K 线列表 |
| total_pages | int | Y | 总页数 |
| total_items | int | Y | 总记录数 |

EastmoneyUsStockLatestKline：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| secid | string | Y | 证券 ID，格式 {market}.{code}，如 105.ADV |
| code | string | Y | 股票代码 |
| name | string | Y | 股票名称 |
| market | string | Y | 市场编号 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘价（Decimal 字符串） |
| close | string | Y | 收盘价（Decimal 字符串） |
| high | string | Y | 最高价（Decimal 字符串） |
| low | string | Y | 最低价（Decimal 字符串） |
| volume | int | Y | 成交量 |
| amount | string | Y | 成交额（Decimal 字符串） |
| amplitude | string | Y | 振幅 |
| klt | int | Y | K 线类型（101=日 K） |
| fqt | int | Y | 复权类型（1=前复权） |

## 调用方法（MCP）

> MCP 工具名 `ft_eastmoney_us_stock_latest_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_eastmoney_us_stock_latest_kline","arguments":{}}}'
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
            res = await s.call_tool('ft_eastmoney_us_stock_latest_kline', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

> 注：该接口公网返回「系统错误」，待后端修复后补真实样例。表头结构如下：

| secid | code | open | high | low | close |
|-------|------|------|------|-----|-------|
| ... | ... | ... | ... | ... | ... |
