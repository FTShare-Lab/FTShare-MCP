# ETF-PCF清单列表（MCP 工具 `ft_etf_pcf_list_handler`）

> **MCP 工具**：`ft_etf_pcf_list_handler`（category: `ETF专题`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取指定交易日的 ETF PCF（申购赎回清单）文件列表，代理请求外部服务。返回 `EtfPcfListResponse`：分页元数据（total/page/page_size）+ items 列表，每项含 ETF 代码、日期、PCF 文件名。文件本身需另行调用下载接口（`/etf-pcf/etf-pcfs/{filename}`）获取 XML 内容。
- 数据范围：无时间维度（按交易日取），指定交易日 PCF 文件清单
- 单次限量：`page` 默认 1、`page_size` 默认 20、最大 100
- 提示：
  - `date` 必填，YYYYMMDD 格式，传非有效日期会被拒。
  - 列表只返回文件元信息，实际 PCF 内容（XML）需通过下载接口按 filename 获取。
  - 真实数据样例需按公共服务返回补充。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| date | int | Y | 日期 YYYYMMDD，必填 |
| page | int | N | 页码，从 1 开始，默认 1 |
| page_size | int | N | 每页条数，默认 20，最大 100 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| total | int64 | Y | 总条数 |
| page | int32 | Y | 当前页 |
| page_size | int32 | Y | 每页条数 |
| items | array[EtfPcfItem] | Y | PCF 文件列表 |

EtfPcfItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| etf_code | int | Y | ETF 代码 |
| date | int | Y | 日期 YYYYMMDD |
| filename | string | Y | PCF 文件名，如 pcf_159152_20260316.xml |

## 调用方法（MCP）

> MCP 工具名 `ft_etf_pcf_list_handler`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_pcf_list_handler","arguments":{"date": 20260624}}}'
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
            res = await s.call_tool('ft_etf_pcf_list_handler', {"date": 20260624})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

2026-06-17 当日 PCF 清单（共 1563 只，节选）：

| etf_code | date | filename |
|----------|------|----------|
| 159001 | 20260617 | pcf_159001_20260617.xml |
| 159003 | 20260617 | pcf_159003_20260617.xml |
| 159005 | 20260617 | pcf_159005_20260617.xml |
| ... | ... | ... |
