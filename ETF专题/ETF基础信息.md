# ETF基础信息（MCP 工具 `ft_etf_description_all`）

> **MCP 工具**：`ft_etf_description_all`（category: `ETF专题`）。返回统一 MCP 输出：`structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是同值的序列化 JSON，不额外返回 Markdown。输入参数 / 输出参数 / 数据样例见下文。
> 文中 `Response`、`items`、`records`、`code`、`message` 等名称仅为字段说明；MCP 对外固定为上述 `metadata/data`。

- 描述：获取全部 ETF 的基础信息（基金类型、托管人、流通份额、成立日期、管理人、是否两融、是否支持 T+0、跟踪指数信息等）。返回数组（非分页），按 symbol 排序输出。每项含 `EtfDescription` + `symbol`（标的代码）。提示：无入参，直接 GET 即可；`float_shares`、`tracking_index` 等可为空，可能为 null。
- 数据范围：无时间维度，最新快照（ETF 静态档案）
- 单次限量：无入参、无分页，一次返回全部 ETF 基础信息
- 提示：
  - 无入参，直接 GET 即可。
  - `float_shares`、`tracking_index` 等为 `Option`，可能为 null。
  - `asset_class` 取值：stock（股票型）/ bond（债券型）/ commodity（商品型）/ currency（货币型）。

## 输入参数

无入参。

## 输出参数

> MCP 固定输出信封为 `structuredContent.metadata` + `structuredContent.data`；`content[0].text` 是与其同值的序列化 JSON，不是 Markdown。
>
> `items` / `records` / `code` / `message` 等传输字段不会直接出现在 MCP 结果中；分页与截断信息统一归入 `metadata`。

| MCP 字段 | 类型 | 必填 | 描述 |
|----------|------|------|------|
| metadata | object | Y | 契约版本、数据来源、工具名、业务口径、总量、分页、返回条数、截断状态及 warnings |
| data | array | Y | 归一化后的业务数据项；元素字段见下方 |

### data 业务字段

| 名称 | 类型 | 默认显示 | 描述 |
|------|------|---------|------|
| symbol | string | Y | 标的代码，如 510300.XSHG |
| asset_class | string | Y | 基金类型：stock / bond / commodity / currency |
| custodian | string | Y | 基金托管人（银行） |
| float_shares | int64 | Y | 流通份额 |
| inception_date | date | Y | 成立日期 YYYY-MM-DD |
| management_company | string | Y | 基金管理人（公司） |
| marginable | bool | Y | 是否可融资融券 |
| name | string | Y | 标的名称（中文） |
| supports_t0 | bool | Y | 是否支持 T+0 交易 |
| tracking_index | string | N | 跟踪指数名称 |
| tracking_index_id | string | N | 跟踪指数 ID |
| tracking_index_symbol | string | N | 跟踪指数代码 |

## 调用方法（MCP）

> MCP 工具名 `ft_etf_description_all`。MCP Streamable HTTP 要求**先 initialize 拿 `Mcp-Session-Id`，发送 `notifications/initialized`，再 `tools/call`**，后续请求同时带该 Session ID 和协商后的 `MCP-Protocol-Version`。返回统一 `metadata/data` 结构化输出；`content[0].text` 为同值 JSON 文本，不额外返回 Markdown。

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
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"ft_etf_description_all","arguments":{}}}'
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
            res = await s.call_tool('ft_etf_description_all', {})
            print(res.structuredContent)   # 推荐：统一 metadata/data 结构化输出
            print(res.content[0].text)   # 兼容：与 structuredContent 同值的 JSON 文本
asyncio.run(main())
```

## 数据样例

全量 ETF 基础信息（节选）：

| symbol | asset_class | name | inception_date | management_company | custodian |
|--------|-------------|------|----------------|--------------------|-----------| 
| 159001.XSHE | currency | 货币ETF易方达 | 2013-03-29 | 易方达基金 | 交通银行 |
| 159003.XSHE | currency | 招商快线ETF | 2013-05-17 | 招商基金 | 平安银行 |
| ... | ... | ... | ... | ... | ... |
