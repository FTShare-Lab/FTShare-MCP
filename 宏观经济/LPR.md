# LPR（MCP 工具 `ft_lpr_monthly`）

> **MCP 工具**：`ft_lpr_monthly`（category: `宏观经济/国内宏观`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：查询中国贷款市场报价利率（LPR）月度汇总计算结果。按日期返回 LPR 1 年期利率（`file_id` = M00000553）与 5 年期利率（M00000554）的当日值。底层从宏观经济指标库加载后按 `field_date` 聚合，按年份（2000 年起）降序排列。响应为数组直出，元素为 `LprComputed`。无入参。
- 数据范围：宏观经济日频利率数据，以服务端返回为准
- 单次限量：全量序列一次返回，无分页上限
- 提示：
  - `date` 为 ISO 格式日期（如「2025-12-22」）。
  - 利率单位为百分比（%），缺失为 null。
  - 经 5s 瞬时失败策略缓存。

## 输入参数

无。

## 输出参数

数组元素（LprComputed）：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| date | string | Y | 日期，ISO 格式如「2025-12-22」（来自 `field_date`） |
| lpr_1y | decimal | Y | LPR 1 年期利率（%，M00000553） |
| lpr_5y | decimal | Y | LPR 5 年期利率（%，M00000554） |

## 调用方法（MCP）

> MCP 工具名 `ft_lpr_monthly`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_lpr_monthly","arguments":{}}}'
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
            res = await s.call_tool('ft_lpr_monthly', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

全量序列按日期降序返回，节选最近两期核心字段：

| date | lpr_1y | lpr_5y |
|------|--------|--------|
| 2026-05-20 | 3 | 3.5 |
| 2026-04-20 | 3 | 3.5 |
| ... | ... | ... |
