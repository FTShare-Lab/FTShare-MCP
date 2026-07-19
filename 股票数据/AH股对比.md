# AH股对比（MCP 工具 `ft_get_stk_ah_comparison`）

> **MCP 工具**：`ft_get_stk_ah_comparison`（category: `股票数据/特色数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询 AH 股比价分页数据。支持按 `hk_code`（港股）/`ts_code`（A 股）+ 单日或日期区间过滤，返回每对 A/H 标的的历史比价记录：含双方名称/收盘价/涨跌幅、A/H 比价（= A 股收盘价 / 港股收盘价）、A/H 溢价率（=（A/H − 1）× 100）。默认 `page=1`、`page_size=50`，`page_size` 上限 500。提示：`hk_code` 支持 `700` 或 `00700.HK`；`ts_code` 需 `xxxxxx.SH/SZ/BJ` 后缀格式；`page_size` 上限 500。
- 数据范围：以服务端返回为准
- 单次限量：分页，默认 `page=1` / `page_size=50`，`page_size` 上限 500（校验层）/ 1000（store 钳位）
- 提示：
  - 对外 v1 仅暴露 GET（底层 v0 同时支持 GET+POST）。
  - 可单按 `hk_code` 或 `ts_code` 过滤，也可配合 `trade_date` 或 `start_date`/`end_date` 区间。
  - 稳定调用建议显式传入 `trade_date` 或完整日期区间；下方示例使用已验证的单日查询。
  - `hk_code` 支持 `700` 或 `00700.HK`；`ts_code` 需 `xxxxxx.SH/SZ/BJ` 后缀格式。
  - 字段注释「最大 1000」与校验层 `validate_page` 上限 500 存在冗余，实际生效以校验层 500 为准。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| hk_code | string | N | 港股股票代码，支持 `700` 或 `00700.HK` |
| ts_code | string | N | A 股股票代码，格式 `xxxxxx.SH/SZ/BJ` |
| trade_date | int32 | N | 交易日期 YYYYMMDD |
| start_date | int32 | N | 起始日期 YYYYMMDD |
| end_date | int32 | N | 结束日期 YYYYMMDD |
| page | int | N | 页码，默认 1 |
| page_size | int | N | 每页条数，默认 50，上限 500（校验层） |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

StkAhComparisonItem：

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| hk_code | string | Y | 港股股票代码 |
| ts_code | string | Y | A 股股票代码 |
| trade_date | int32 | Y | 交易日期 YYYYMMDD |
| hk_name | string | Y | 港股股票名称 |
| hk_close | decimal | Y | 港股收盘价 |
| hk_pct_chg | decimal | Y | 港股涨跌幅（百分数） |
| name | string | Y | A 股股票名称 |
| close | decimal | Y | A 股收盘价 |
| pct_chg | decimal | Y | A 股涨跌幅（百分数） |
| ah_comparison | decimal | Y | A/H 比价 = A 股收盘价 / 港股收盘价 |
| ah_premium | decimal | Y | A/H 溢价率 =（A 股收盘价 / 港股收盘价 − 1）× 100 |

## 调用方法（MCP）

> MCP 工具名 `ft_get_stk_ah_comparison`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_stk_ah_comparison","arguments":{"trade_date": 20260717, "page": 1, "page_size": 3}}}'
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
            res = await s.call_tool('ft_get_stk_ah_comparison', {"trade_date": 20260717, "page": 1, "page_size": 3})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

`trade_date=20260617` 全市场分页结果节选（共 190 对 A/H 标的）：

| hk_code | ts_code | hk_name | name | hk_close | close | ah_comparison | ah_premium |
|---------|---------|---------|------|----------|-------|----------------|------------|
| 00038.HK | 601038.SH | 第一拖拉机股份 | 一拖股份 | 7.19 | 11.5 | 1.5994 | 59.9444 |
| 00107.HK | ... | ... | ... | ... | ... | ... | ... |
| ... | ... | ... | ... | ... | ... | ... | ... |
