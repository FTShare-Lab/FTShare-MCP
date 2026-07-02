# GDP（MCP 工具 `ft_consumer_gdp_quarterly`）

> **MCP 工具**：`ft_consumer_gdp_quarterly`（category: `宏观经济/国内宏观`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询中国 GDP 季度汇总结果。按季度（按 `field_date` 的季度）返回 GDP 总量、同比，以及第一/二/三产业增加值与各产业累计同比。季度标签按累积口径输出，如「YYYY年第1-3季度」。底层从宏观经济指标库加载后按季度聚合，按年份（2000 年起）降序排列。响应为数组直出，元素为 `GdpComputed`，含货币单位与币种。无入参。
- 数据范围：宏观经济季度数据，以服务端返回为准
- 单次限量：全量季度序列一次返回，无分页上限
- 提示：
  - `period` 为季度累积口径标签（如「2025年第1-3季度」），非单季度。
  - 金额单位见 `unit`（通常为亿元），币种见 `currency`。
  - 经 5s 瞬时失败策略缓存。

## 输入参数

无。

## 输出参数

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

> MCP 工具名 `ft_consumer_gdp_quarterly`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
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
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

全量季度序列按季度降序返回，节选最新一期（2026年第1季度）核心字段：

| period | gdp | gdp_yoy | primary | secondary |
|--------|-----|---------|---------|-----------|
| 2026年第1季度 | 334193 | 5 | 11941 | 116135 |
| ... | ... | ... | ... | ... |
