# ETF成份列表（MCP 工具 `ft_etf_components_all`）

> **MCP 工具**：`ft_etf_components_all`（category: `ETF专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询全部 ETF 的成份股清单，每只 ETF 返回成份代码、成份名称和标的代码。提示：无入参，返回数组。
- 数据范围：无时间维度，最新快照（成份股当前构成）
- 单次限量：无入参、无分页，一次返回全部 ETF 的成份清单
- 提示：
  - 无入参，直接 GET 即可；建议带 `If-Modified-Since` 头走协商缓存。
  - `components` 与 `components_name` 两个数组按相同下标对齐，下标 i 对应同一只成份股。
  - 单条记录的 `components` 可能较长（成份股数十至上千只），整包体积较大。
  - 当前 MCP 示例调用已验证通过；返回体较大时会按服务端展示策略截断。

## 输入参数

无入参。

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码，如 510300.XSHG |
| components | array[string] | Y | 成份代码列表，与 components_name 同序对齐 |
| components_name | array[string] | Y | 成份名称列表（中文），与 components 同序对齐 |

## 调用方法（MCP）

> MCP 工具名 `ft_etf_components_all`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

每只 ETF 一条记录（全量返回，这里仅示意 2 只）：

| symbol | components | components_name |
|--------|------------|-----------------|
| 510300.XSHG | [300001.XSHE, 300014.XSHE, 300750.XSHE, ...] | [特锐德, 亿纬锂能, 宁德时代, ...] |
| 159915.XSHE | [000001.XSHE, 000002.XSHE, ...] | [平安银行, 万科A, ...] |
| ... | ... | ... |
