# 第N个交易日（MCP 工具 `ft_get_nth_trade_date`）

> **MCP 工具**：`ft_get_nth_trade_date`（category: `股票数据/基础数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：根据给定 N，返回当前日期向前第 N 个交易日（N >= 1）。基于底层交易日历过滤非交易日，返回当前日期与目标第 N 个交易日。响应为 `GetNthTradeDateResponse`。
- 数据范围：实时计算（基于交易日历，无固定数据范围）
- 单次限量：单次一个 N 值
- 提示：
  - `n` 必须 >= 1。
  - 「当前日期」以服务端当日为准，目标交易日为向前（历史方向）数第 N 个。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| n | uint32 | Y | 需要获取的前 N 个交易日，N >= 1 |

## 输出参数

GetNthTradeDateResponse：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| current_date | string | Y | 当前日期（YYYY-MM-DD） |
| nth_trade_date | string | Y | 前 N 个交易日（YYYY-MM-DD） |
| n | uint32 | Y | 请求的 N 值 |

## 调用方法（MCP）

> MCP 工具名 `ft_get_nth_trade_date`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_nth_trade_date","arguments":{"n": 5}}}'
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
            res = await s.call_tool('ft_get_nth_trade_date', {"n": 5})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

`n=5`（2026-06-17 为当前日期，向前第 5 个交易日）：

| current_date | n | nth_trade_date |
|--------------|---|----------------|
| 2026-06-17 | 5 | 2026-06-10 |
