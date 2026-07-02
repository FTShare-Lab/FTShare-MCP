# ETF复权因子（MCP 工具 `ft_etf_adjust_factor`）

> **MCP 工具**：`ft_etf_adjust_factor`（category: `ETF专题`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询 ETF 复权因子（当日复权因子 + 截至当日累计复权因子）。支持两种模式：① 单日扫描——指定 `trade_date` 返回该日复权因子快照，未指定 `symbol` 时返回所有标的单日快照；② 多日扫描——指定 `start_date` / `end_date` 返回区间内每个交易日的复权因子，**多日扫描必须指定 `symbol`**。通过底层复权因子数据查询。响应为 `ApiResponse` 信封（code/msg/data）。
- 数据范围：每标的仅保留最新交易日复权因子快照（跨 510300/510050/159915/588000/510500 探测，均只有当日 1 条，无历史序列）
- 单次限量：分页返回，默认 `page=1`、`page_size` 由后端定；单标的最新快照仅 1 条
- 提示：
  - 未指定 `symbol` 时仅支持单日扫描；区间扫描必须指定 `symbol`。
  - `trade_date` 为空时默认当天，非交易日回退到前一交易日。
  - `symbol` 输出统一为 `symbol.suffix` 格式（如 `510300.SH`）。
  - 公网实测每个标的当前仅返回最新交易日 1 条，未见历史多日序列。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | N | ETF 代码，支持纯数字或带后缀格式；区间扫描必填 |
| trade_date | string | N | 交易日期 YYYYMMDD；空则默认当天，非交易日回退前一交易日 |
| start_date | string | N | 区间起始日期 YYYYMMDD；区间扫描必填且需配 symbol |
| end_date | string | N | 区间结束日期 YYYYMMDD；区间扫描必填且需配 symbol |
| offset | int | N | 返回结果起始偏移 |
| limit | int | N | 返回结果最大条数 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| code | int | Y | 状态码：0 成功 |
| message | string | Y | 状态信息 |
| data | object | Y | 分页对象（见下） |

data ：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| pageNum | int | Y | 当前页码 |
| pageSize | int | Y | 每页条数 |
| total | int | Y | 总记录数 |
| pages | int | Y | 总页数 |
| records | array[AdjustFactorItem] | Y | 复权因子列表 |

AdjustFactorItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | ETF 代码，统一输出 symbol.suffix 格式 |
| trade_date | string | Y | 交易日 YYYYMMDD |
| adj_factor | f64 | Y | 当日复权因子 |
| ex_adj_factor | f64 | Y | 截至当日累计复权因子 |

## 调用方法（MCP）

> MCP 工具名 `ft_etf_adjust_factor`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_adjust_factor","arguments":{}}}'
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
            res = await s.call_tool('ft_etf_adjust_factor', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

510300.XSHG 沪深300ETF 最新交易日复权因子（每标的仅当日 1 条）：

| symbol | trade_date | adj_factor | ex_adj_factor |
|--------|------------|------------|---------------|
| 510300.SH | 20260617 | 1.0 | 0.470035 |
| ... | ... | ... | ... |
