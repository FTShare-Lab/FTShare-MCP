# ETF成份列表（MCP 工具 `ft_etf_components_all`）

> **MCP 工具**：`ft_etf_components_all`（category: `ETF专题`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取全部 ETF 的成份股清单（每只 ETF 一条记录，含成份代码数组 + 成份名称数组 + 标的代码）。返回数组（非分页）。底层来自 `ETF_COMPONENTS` 内存快照，支持 `If-Modified-Since` 协商缓存，未变更时返回 304。同 `EtfComponentsAll` 协议结构（components / components_name / symbol 三字段）。
- 数据范围：无时间维度，最新快照（成份股当前构成）
- 单次限量：无入参、无分页，一次返回全部 ETF 的成份清单
- 提示：
  - 无入参，直接 GET 即可；建议带 `If-Modified-Since` 头走协商缓存。
  - `components` 与 `components_name` 两个数组按相同下标对齐，下标 i 对应同一只成份股。
  - 单条记录的 `components` 可能较长（成份股数十至上千只），整包体积较大。
  - 当前公共服务全量查询在 65 秒内未返回；示例保留调用格式，生产使用建议优先调用单只 ETF 成份股工具。

## 输入参数

无入参。

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码，如 510300.XSHG |
| components | array[string] | Y | 成份代码列表，与 components_name 同序对齐 |
| components_name | array[string] | Y | 成份名称列表（中文），与 components 同序对齐 |

## 调用方法（MCP）

> MCP 工具名 `ft_etf_components_all`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_components_all","arguments":{}}}'
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
            res = await s.call_tool('ft_etf_components_all', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

每只 ETF 一条记录（全量返回，这里仅示意 2 只）：

| symbol | components | components_name |
|--------|------------|-----------------|
| 510300.XSHG | [300001.XSHE, 300014.XSHE, 300750.XSHE, ...] | [特锐德, 亿纬锂能, 宁德时代, ...] |
| 159915.XSHE | [000001.XSHE, 000002.XSHE, ...] | [平安银行, 万科A, ...] |
| ... | ... | ... |
