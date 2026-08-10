# AH股对比（MCP 工具 `ft_get_stk_ah_comparison`）

> **MCP 工具**：`ft_get_stk_ah_comparison`（category: `股票数据/特色数据`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：查询 A/H 股历史比价，支持按港股代码 `hk_code`、A 股代码 `ts_code`、单日或日期区间筛选。信息包括双方名称、收盘价、涨跌幅、A/H 比价（A 股收盘价 ÷ 港股收盘价）和 A/H 溢价率（（A/H 比价 − 1）× 100）。
- 数据范围：按交易日查询
- 单次限量：分页，默认 `page=1` / `page_size=50`，`page_size` 上限 1000
- 提示：
  - 可单按 `hk_code` 或 `ts_code` 过滤，也可配合 `trade_date` 或 `start_date`/`end_date` 区间。
  - `hk_code` 支持 `700` 或 `00700.HK`；`ts_code` 需 `xxxxxx.SH/SZ/BJ` 后缀格式。
  - `trade_date` 与 `start_date`/`end_date` 二选一；未提供 `trade_date` 时，`start_date` 和 `end_date` 必须同时提供。
  - `page_size` 上限 1000。

## 输入参数

| 名称 | 类型 | 必选 | 描述 |
|------|------|------|------|
| hk_code | string | N | 港股股票代码，支持 `700` 或 `00700.HK` |
| ts_code | string | N | A 股股票代码，格式 `xxxxxx.SH/SZ/BJ` |
| trade_date | int32 | N | 交易日期 YYYYMMDD |
| start_date | int32 | N | 起始日期 YYYYMMDD |
| end_date | int32 | N | 结束日期 YYYYMMDD |
| page | int | N | 页码，默认 1 |
| page_size | int | N | 每页条数，默认 50，上限 1000 |

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| ah_comparison | decimal | Y | A/H 比价 = A 股收盘价 / 港股收盘价 |
| ah_premium | decimal | Y | A/H 溢价率 =（A 股收盘价 / 港股收盘价 − 1）× 100 |
| close | decimal | Y | A 股收盘价 |
| hk_close | decimal | Y | 港股收盘价 |
| hk_code | string | Y | 港股股票代码 |
| hk_name | string | Y | 港股股票名称 |
| hk_pct_chg | decimal | Y | 港股涨跌幅（百分数） |
| name | string | Y | A 股股票名称 |
| pct_chg | decimal | Y | A 股涨跌幅（百分数） |
| trade_date | int32 | Y | 交易日期 YYYYMMDD |
| ts_code | string | Y | A 股股票代码 |

## 调用方法（MCP）

> MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

**curl**：

```bash
set -euo pipefail

MCP_BASE_URL="<MCP_BASE_URL>"

check_mcp_response() {
  local response=$1
  printf '%s\n' "$response"
  if printf '%s\n' "$response" | grep -Eq '"isError"[[:space:]]*:[[:space:]]*true|"error"[[:space:]]*:[[:space:]]*\{'; then
    return 1
  fi
}

SID=$(curl -fsS -m 60 -D - -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"ftshare-doc-example","version":"1.0.0"}}}' \
  | awk 'tolower($1)=="mcp-session-id:" {print $2}' \
  | tr -d '\r')

if [ -z "$SID" ]; then
  printf '%s\n' 'initialize 未返回 Mcp-Session-Id' >&2
  exit 1
fi

curl -fsS -m 60 -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

CALL_RESPONSE=$(curl -fsS -m 60 -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $SID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_get_stk_ah_comparison","arguments":{"trade_date":20260714,"page":1,"page_size":2}}}')

check_mcp_response "$CALL_RESPONSE"
```

**Python（`mcp` SDK，自动握手管理 session）**：

```python
import asyncio

from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client


MCP_BASE_URL = "<MCP_BASE_URL>"


async def main():
    async with streamable_http_client(MCP_BASE_URL) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            result = await session.call_tool(
                'ft_get_stk_ah_comparison',
                {'trade_date': 20260714, 'page': 1, 'page_size': 2},
            )
            if result.is_error:
                raise RuntimeError(result.content[0].text)
            print(result.structured_content)
            print(result.content[0].text)


asyncio.run(main())
```

## 数据样例

| hk_code | ts_code | hk_name | name | hk_close | close | ah_comparison | ah_premium |
|------|------|------|------|------|------|------|------|
| 00038.HK | 601038.SH | 第一拖拉机股份 | 一拖股份 | 7.24 | 11.57 | 1.5980662983425414364640883978 | 59.806629834254143646408839780 |
| 00107.HK | 601107.SH | 四川成渝高速公路 | 四川成渝 | 4.65 | 5.35 | 1.1505376344086021505376344086 | 15.053763440860215053763440860 |
| ... | ... | ... | ... | ... | ... | ... | ... |
