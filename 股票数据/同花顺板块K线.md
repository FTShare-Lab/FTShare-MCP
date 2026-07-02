# 同花顺板块K线（MCP 工具 `ft_ths_board_kline`）

> **MCP 工具**：`ft_ths_board_kline`（category: `股票数据/打板专题数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取单个同花顺板块的历史日 K 线（开高低收 + 成交量），全部数值按**字符串**返回。返回 `PaginatedResponse`，`total_items` 为该板块全部交易日数。典型板块如行业类（881101）有 4500+ 根日 K（可追溯到 2007 年），新概念板块（886056）约 600+ 根。
- 数据范围：2007-08 至今（跨行业 881101/881102、概念 885525、地区 886001 等多板块探测，行业类最早 2007-08-01）
- 单次限量：无硬性上限，由 page/page_size 分页，默认返回全部行（不传分页参数）
- 提示：
  - `board_code` 必填，需先通过板块列表接口或内存映射拿到；csrc 模块板块无 K 线，请求会返回 400。
  - 数值字段（open/high/low/close/volume）均为字符串。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| board_code | string | Y | 板块代码，如 886056 |
| page | int | N | 页码，从 1 开始 |
| page_size | int | N | 每页数量 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| items | array | Y | 当前页 K 线列表 |
| total_pages | int | Y | 总页数 |
| total_items | int | Y | 总交易日数 |

ThsBoardKlineRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| board_code | string | Y | 板块代码 |
| board_name | string | Y | 板块名称 |
| module | string | Y | 所属模块：concept/csrc/industry/region |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘 |
| high | string | Y | 最高 |
| low | string | Y | 最低 |
| close | string | Y | 收盘 |
| volume | string | Y | 成交量 |

## 调用方法（MCP）

> MCP 工具名 `ft_ths_board_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_ths_board_kline","arguments":{"board_code": "BK0425"}}}'
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
            res = await s.call_tool('ft_ths_board_kline', {"board_code": "BK0425"})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

阿尔茨海默概念 886056：

| board_code | board_name | module | date | open | high | low | close | volume |
|------------|------------|--------|------|------|------|-----|-------|--------|
| 886056 | 阿尔茨海默概念 | concept | 2023-09-19 | 998.312 | 1024.311 | 992.78 | 997.56 | 506490480 |
| 886056 | 阿尔茨海默概念 | concept | 2023-09-20 | 976.249 | 991.568 | 975.323 | 979.195 | 256607070 |
| 886056 | 阿尔茨海默概念 | concept | 2023-09-21 | 977.94 | 993.159 | 976.141 | 982.057 | 321082480 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
