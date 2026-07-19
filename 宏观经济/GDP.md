# GDP（MCP 工具 `ft_consumer_gdp_quarterly`）

> **MCP 工具**：`ft_consumer_gdp_quarterly`（category: `宏观经济/国内宏观`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询中国 GDP 季度汇总结果。以及第一/二/三产业增加值与各产业累计同比。季度标签按累积口径输出，如「YYYY年第1-3季度」。提示：金额单位见 `unit`（通常为亿元），币种见 `currency`。
- 数据范围：宏观经济季度数据，以服务端返回为准
- 单次限量：全量季度序列一次返回，无分页上限
- 提示：
  - `period` 为季度累积口径标签（如「2025年第1-3季度」），非单季度。
  - 金额单位见 `unit`（通常为亿元），币种见 `currency`。
  - 经 5s 瞬时失败策略缓存。

## 输入参数

无。

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

数组元素（GdpComputed）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| period | string | Y | 季度标签，累积口径如「YYYY年第1-3季度」 |
| gdp | decimal | Y | GDP 总量（unit） |
| gdp_yoy | decimal | Y | GDP 同比（%） |
| primary | decimal | Y | 第一产业增加值（unit） |
| primary_cumulative_yoy | decimal | Y | 第一产业累计同比（%） |
| secondary | decimal | Y | 第二产业增加值（unit） |
| secondary_cumulative_yoy | decimal | Y | 第二产业累计同比（%） |
| tertiary | decimal | Y | 第三产业增加值（unit） |
| tertiary_cumulative_yoy | decimal | Y | 第三产业累计同比（%） |
| unit | string | Y | 货币单位 |
| currency | string | Y | 货币种类 |

## 调用方法（MCP）

> MCP 工具名 `ft_consumer_gdp_quarterly`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_consumer_gdp_quarterly","arguments":{}}}'
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
            res = await s.call_tool('ft_consumer_gdp_quarterly', {})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

全量季度序列按季度降序返回，节选最新一期（2026年第1季度）核心字段：

| period | gdp | gdp_yoy | primary | secondary |
|--------|-----|---------|---------|-----------|
| 2026年第1季度 | 334193 | 5 | 11941 | 116135 |
| ... | ... | ... | ... | ... |
