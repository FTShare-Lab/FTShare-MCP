# CPI（MCP 工具 `ft_consumer_price_index_monthly`）

> **MCP 工具**：`ft_consumer_price_index_monthly`（category: `宏观经济/国内宏观`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询中国居民消费价格指数（CPI）月度汇总计算结果。按月份返回全国（含累计同比）、城市、农村三档 CPI 同比/环比/当月值。底层从宏观经济指标库按 `file_id` 加载后按月聚合，并按 `field_date` 年份（2000 年起）降序排列。响应为数组直出（非分页信封），元素为 `ComputedCpi`。无入参（v1 路由未暴露任何 query 参数）。
- 数据范围：宏观经济月度数据，以服务端返回为准
- 单次限量：全量月度序列一次返回，无分页上限
- 提示：
  - 月份字段 `month` 已格式化为中文「YYYY年MM月份」。
  - 数值字段单位为百分比（%），部分月份字段缺失时为 null。
  - 底层经 5s 瞬时失败策略缓存。

## 输入参数

无（v1 路由 `get_china_consumer_price_index_monthly` 不接收任何 query 参数）。

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| month | string | Y | 月份，格式「YYYY年MM月份」 |
| national_cpi | decimal | Y | 全国 CPI 当月值（%） |
| national_yoy | decimal | Y | 全国 CPI 同比（%） |
| national_mom | decimal | Y | 全国 CPI 环比（%） |
| cumulative | decimal | Y | 全国 CPI 累计同比（%） |
| city_cpi | decimal | Y | 城市 CPI 当月值（%） |
| city_yoy | decimal | Y | 城市 CPI 同比（%） |
| city_mom | decimal | Y | 城市 CPI 环比（%） |
| rural_cpi | decimal | Y | 农村 CPI 当月值（%） |
| rural_yoy | decimal | Y | 农村 CPI 同比（%） |
| rural_mom | decimal | Y | 农村 CPI 环比（%） |

## 调用方法（MCP）

> MCP 工具名 `ft_consumer_price_index_monthly`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_consumer_price_index_monthly","arguments":{}}}'
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
            res = await s.call_tool('ft_consumer_price_index_monthly', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

全量月度序列按月份降序返回，节选最新一期（2026年05月份）核心字段：

| month | national_cpi | national_yoy | national_mom |
|-------|--------------|--------------|---------------|
| 2026年05月份 | 101.2 | 1.1858 | -0.1001 |
| ... | ... | ... | ... |
