# PMI（MCP 工具 `ft_consumer_pmi_monthly`）

> **MCP 工具**：`ft_consumer_pmi_monthly`（category: `宏观经济/国内宏观`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询中国采购经理指数（PMI）月度汇总计算结果，覆盖制造业与非制造业。按月份返回制造业/非制造业 PMI 当月值、同比、环比。底层从宏观经济指标库加载后按月聚合，按 `field_date` 年份（2000 年起）降序排列。响应为数组直出，元素为 `PmiComputed`。无入参。
- 数据范围：宏观经济月度数据，以服务端返回为准
- 单次限量：全量月度序列一次返回，无分页上限
- 提示：
  - `month` 格式化为「YYYY年MM月份」。
  - PMI 数值单位为百分比（%），缺失为 null。
  - 经 5s 瞬时失败策略缓存。

## 输入参数

无。

## 输出参数

数组元素（PmiComputed）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| month | string | Y | 月份，格式「YYYY年MM月份」 |
| manufacturing_pmi | decimal | Y | 制造业 PMI 当月值（%） |
| manufacturing_yoy | decimal | Y | 制造业 PMI 同比（%） |
| manufacturing_mom | decimal | Y | 制造业 PMI 环比（%） |
| non_manufacturing_pmi | decimal | Y | 非制造业 PMI 当月值（%） |
| non_manufacturing_yoy | decimal | Y | 非制造业 PMI 同比（%） |
| non_manufacturing_mom | decimal | Y | 非制造业 PMI 环比（%） |

## 调用方法（MCP）

> MCP 工具名 `ft_consumer_pmi_monthly`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_consumer_pmi_monthly","arguments":{}}}'
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
            res = await s.call_tool('ft_consumer_pmi_monthly', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

全量月度序列按月份降序返回，节选最新一期（2026年05月份）核心字段：

| month | manufacturing_pmi | non_manufacturing_pmi |
|-------|------------------|-----------------------|
| 2026年05月份 | 50 | 50.1 |
| ... | ... | ... |
