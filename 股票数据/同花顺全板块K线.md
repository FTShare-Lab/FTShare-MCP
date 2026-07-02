# 同花顺全板块K线（MCP 工具 `ft_ths_all_board_kline`）

> **MCP 工具**：`ft_ths_all_board_kline`（category: `股票数据/打板专题数据`）。返回 **markdown 表格文本**（非结构化 JSON）。输入参数 / 输出参数 / 数据样例对照 [ftshare-doc](../../ftshare-doc/api-doc) 原接口。

- 描述：获取同花顺**全部板块**（概念/行业/地区，csrc 跳过）在指定日期范围内的日 K 线。结果按板块再按日期拼接成一张长表，分页返回。典型一个 3 交易日窗口约返回 900+ 行（≈ 470 个有 K 线的板块 × 3 天）。
- 数据范围：2007-08 至今（底层数据与单板块 K 线一致，行业类最早 2007-08-01）
- 单次限量：start_date 与 end_date 同时给出时跨度 **≤3 天**，否则报错；由 page/page_size 分页
- 提示：
  - 日期格式支持 `YYYY-MM-DD` 或 `YYYYMMDD`。
  - `start_date`/`end_date` 均可选；同时传时硬限 3 天跨度，超过返回错误。
  - 结果自动跳过 csrc 模块（无 K 线）。
  - 数值字段均为字符串。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| start_date | string | N | 起始日期（含），YYYY-MM-DD 或 YYYYMMDD |
| end_date | string | N | 截止日期（含），YYYY-MM-DD 或 YYYYMMDD |
| page | int | N | 页码，从 1 开始 |
| page_size | int | N | 每页数量 |

## 输出参数

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| items | array | Y | 当前页 K 线列表（多板块拼接） |
| total_pages | int | Y | 总页数 |
| total_items | int | Y | 命中总行数 |

ThsBoardKlineRow：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| board_code | string | Y | 板块代码 |
| board_name | string | Y | 板块名称 |
| module | string | Y | 所属模块 |
| date | string | Y | 日期 YYYY-MM-DD |
| open | string | Y | 开盘 |
| high | string | Y | 最高 |
| low | string | Y | 最低 |
| close | string | Y | 收盘 |
| volume | string | Y | 成交量 |

## 调用方法（MCP）

> MCP 工具名 `ft_ths_all_board_kline`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，再 `tools/call`**，后续请求带该 header。返回 **markdown 表格文本**。

**curl**：

```bash
SID=$(curl -sS -m 10 -D - -o /dev/null -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' \
  | grep -i 'mcp-session-id:' | awk '{print $2}' | tr -d '')

curl -sS -m 10 -X POST <MCP_BASE_URL> \
  -H "Accept: application/json, text/event-stream" -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_ths_all_board_kline","arguments":{}}}'
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
            res = await s.call_tool('ft_ths_all_board_kline', {})
            print(res.content[0].text)   # markdown 表格文本

asyncio.run(main())
```

## 数据样例

2026-06-13 至 2026-06-16 窗口（节选）：

| board_code | board_name | module | date | open | high | low | close | volume |
|------------|------------|--------|------|------|------|-----|-------|--------|
| 885768 | 无人零售 | concept | 2026-06-15 | 1282.071 | 1308.979 | 1277.81 | 1289.779 | 594369490 |
| 885768 | 无人零售 | concept | 2026-06-16 | 1290.717 | 1305.435 | 1268.564 | 1303.839 | 970300170 |
| 885744 | 雄安新区 | concept | 2026-06-15 | 1151.206 | 1169.115 | 1147.999 | 1156.548 | 7767410200 |
| 885744 | 雄安新区 | concept | 2026-06-16 | 1154.974 | 1164.136 | 1144.265 | 1161.025 | 6964113600 |
| 885951 | 预制菜 | concept | 2026-06-15 | 995.97 | 1007.696 | 983.701 | 990.517 | 2080070800 |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
