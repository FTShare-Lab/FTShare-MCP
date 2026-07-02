# ETF成份股（MCP 工具 `ft_get_etf_components_handler`）

> **MCP 工具**：`ft_get_etf_components_handler`（category: `ETF专题`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：根据标的代码查询单只 ETF 的成份股清单（成份代码数组 + 成份名称数组 + 标的代码）。与全量接口 `etf-components-all` 单条结构一致，未找到时由 API 层返回错误（404 语义）。返回单个 `EtfComponentsAll` 对象。
- 数据范围：无时间维度，最新快照（成份股当前构成）
- 单次限量：无分页，单只 ETF 全部成份一次返回
- 提示：
  - `symbol` 必填，带交易所后缀（如 510300.XSHG、159915.XSHE）。
  - `components` 与 `components_name` 两个数组按相同下标对齐，下标 i 对应同一只成份股。
  - 未找到该 ETF 成份时返回错误，调用方需处理 not-found。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| symbol | string | Y | ETF 标的代码，带交易所后缀，如 510300.XSHG、159915.XSHE |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码 |
| components | array[string] | Y | 成份代码列表，与 components_name 同序对齐 |
| components_name | array[string] | Y | 成份名称列表（中文），与 components 同序对齐 |

## 调用方法（MCP）

> MCP 工具名 `ft_get_etf_components_handler`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_etf_components_handler","arguments":{"symbol": "600584.SH"}}}'
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
            res = await s.call_tool('ft_get_etf_components_handler', {"symbol": "600584.SH"})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

510300.XSHG 沪深300ETF 成份股（共 300 只，节选）：

| symbol | components | components_name |
|--------|------------|-----------------|
| 510300.XSHG | [000001.XSHE, 000002.XSHE, 000063.XSHE, 000100.XSHE, ...] | [平安银行, 万科A, 中兴通讯, TCL科技, ...] |
| ... | ... | ... |
