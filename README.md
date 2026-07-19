# FTShare MCP

[中文](README.md) | [English](README_EN.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

`FTShare-MCP` 是 FTShare 金融数据能力的 MCP 工具文档与接入说明仓库，面向支持 MCP 的客户端、Agent 应用和自动化投研流程，提供公共 MCP 入口、调用流程、工具索引、参数说明和示例。

> 当前仓库重点是 **MCP 工具文档与接入说明**，不包含 MCP Server 源码。公共 MCP 服务由 FTShare 数据服务提供。

当前文档覆盖 FTShare 正式开放的 **150 个 `ft_*` 金融数据工具**，并补充新版 MCP Server 提供的服务端行情聚合工具，覆盖行情、财务、宏观、基金、期货、债券、美股、港股等数据；测试、示例或内部辅助工具不作为正式文档入口展示。

## MCP 公共入口

本文档统一使用 `<MCP_BASE_URL>` 表示 MCP Streamable HTTP 入口。公共 MCP 入口为：

```bash
MCP_BASE_URL="https://market.ft.tech/gateway/mcp"
```

复制各工具文档示例时，请把 `<MCP_BASE_URL>` 替换为上面的地址。

- **MCP 入口占位符**：`<MCP_BASE_URL>`（Streamable HTTP）
- **协议要点**：先 `initialize` 协商版本并获取 `Mcp-Session-Id`，发送 `notifications/initialized`，再携带 Session ID 与 `MCP-Protocol-Version` 调用工具
- **返回格式**：统一返回一个 JSON `TextContent` 和同值的 `structuredContent`；不额外返回 Markdown
- **查实时 descriptor**：`tools/list` 返回每个工具的 `title`、`inputSchema`、`outputSchema`、`annotations` 和 `_meta`

> **部署状态说明**：本文以下输出格式对应配套新版 `ftshare-mcp-server` 的统一契约。
> 公共入口是否已完成升级应以实时 `tools/list` / `tools/call` 为准；如果某个已记录工具
> 仍缺少 `outputSchema` 或成功结果中没有 `structuredContent`，表示该环境尚未部署配套新版服务。

## 适用场景

- Claude Code、Codex、Cursor、OpenClaw 等支持 MCP 的客户端
- Agent 投研工作流中的金融数据工具调用
- 自动化行情、财务、宏观、基金、期货、债券、美股、港股数据查询
- 面向投研、量化研究和金融应用开发的数据接入验证

## 调用流程

1. 确认 MCP 入口：公共环境使用 `https://market.ft.tech/gateway/mcp`，各工具文档中的 `<MCP_BASE_URL>` 都替换为这个地址。
2. 选择工具：从下方工具索引或 `tools/list` 结果中找到目标 MCP 工具名。
3. 查看参数：每个工具文档都列出输入参数、是否必填、输出字段和数据样例；`tools/list` 也会返回实时 `inputSchema`。
4. 初始化会话：第一次请求先调用 MCP `initialize`，拿到 `Mcp-Session-Id`。
5. 完成初始化：发送 `notifications/initialized` 通知，成功时 HTTP 返回 202 且响应体为空。
6. 调用工具：后续请求带上 `Mcp-Session-Id` 和协商后的 `MCP-Protocol-Version`，调用 `tools/call`，其中 `name` 是工具名，`arguments` 是业务参数。
7. 读取结果：优先读取 `result.structuredContent`；`result.content[0].text` 是与其相同的序列化 JSON，供暂不支持结构化结果的客户端兼容读取。
8. 处理错误：未知工具或请求结构错误属于 JSON-RPC 协议错误；参数校验、API 调用和业务执行错误位于 `result` 中，并设置 `isError: true`。

## 统一输出格式

所有工具使用同一结构化输出 Schema。不同工具的业务字段放在 `data` 数组中：

```json
{
  "metadata": {
    "schema_version": "1.0",
    "source": "ftshare",
    "tool": "ft_get_cb_lists_handler",
    "operation": "get_cb_lists_handler",
    "total": 1,
    "pagination": {
      "supported": false,
      "page": 1,
      "page_size": 1,
      "pages": 1,
      "has_more": false
    },
    "returned": 1,
    "truncated": false,
    "warnings": []
  },
  "data": [
    {
      "cb_id": 110001,
      "full_name": "示例可转债",
      "stock_id": 600001,
      "exchange": 3553
    }
  ]
}
```

完整的 `tools/call` 成功结果如下。`content` 只有一个 JSON 文本块，解析后的值与 `structuredContent` 完全相同：

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"metadata\":{\"schema_version\":\"1.0\",\"source\":\"ftshare\",\"tool\":\"ft_get_cb_lists_handler\",\"operation\":\"get_cb_lists_handler\",\"total\":1,\"pagination\":{\"supported\":false,\"page\":1,\"page_size\":1,\"pages\":1,\"has_more\":false},\"returned\":1,\"truncated\":false,\"warnings\":[]},\"data\":[{\"cb_id\":110001}]}"
      }
    ],
    "structuredContent": {
      "metadata": {
        "schema_version": "1.0",
        "source": "ftshare",
        "tool": "ft_get_cb_lists_handler",
        "operation": "get_cb_lists_handler",
        "total": 1,
        "pagination": {
          "supported": false,
          "page": 1,
          "page_size": 1,
          "pages": 1,
          "has_more": false
        },
        "returned": 1,
        "truncated": false,
        "warnings": []
      },
      "data": [{"cb_id": 110001}]
    },
    "isError": false
  }
}
```

- `metadata.schema_version`：输出契约版本；当前为 `1.0`。
- `metadata.source`：数据来源；当前为 `ftshare`。
- `metadata.tool`：实际调用的 MCP 工具名。
- `metadata.operation`：实际执行业务口径；`ft_*` 工具使用对应 SDK method。
- `metadata.total`：匹配总条数；总量缺失时为本地截断前条数。
- `metadata.pagination`：分页状态，包含 `supported`、`page`、`page_size`、`pages`、`has_more`。
- `metadata.returned`：本次实际放入 `data` 的条数。
- `metadata.truncated`：是否因服务端条目或输出体积限制被截断。
- `metadata.warnings`：去重、结构差异或截断等提示；无提示时为空数组。
- `data`：业务数据数组，字段以对应工具文档的“输出参数”为准。
- 原接口的 `items` / `records` 映射到 `data`；`total_items` / `total_pages` 映射到 `metadata.total` / `metadata.pagination.pages`。
- Markdown 不是 MCP 结构化工具结果的必需格式。为避免重复 payload 和读取歧义，当前默认不返回 Markdown 展示块。

工具执行错误不套用成功结果的 `outputSchema`：错误结果设置 `isError: true`，省略 `structuredContent`，并在 `content[0].text` 中返回脱敏错误文本；可结构化的错误会使用 `{"error":{"code","message","field?","retryable","details?"}}` JSON。条件多选一未满足时不指定误导性的单一 `field`，而是在 `details.required_any_of` 中列出可选参数组合。未知工具或非法 JSON-RPC 请求仍按协议错误返回。

## Tool descriptor

`tools/list` 返回的每个正式工具 descriptor 与 MCP / OpenAI Apps SDK 的只读工具语义对齐：

- `title`：面向用户的人类可读标题。
- `inputSchema`：Draft 2020-12 输入约束；未声明字段会被拒绝。服务会移除 Rust 整数 `format`、`$schema` 和 `default:null`，并补齐多口径工具的条件必填及整数上下界。
- `outputSchema`：统一的 `metadata/data` 成功结果契约。
- `annotations`：`readOnlyHint=true`、`destructiveHint=false`、`openWorldHint=true`。
- `_meta.securitySchemes`：无鉴权服务声明为 `[{"type":"noauth"}]`，用于 OpenAI 客户端兼容。

`structuredContent.metadata` 是模型可见的业务结果字段；descriptor 的 sibling `_meta` 是客户端扩展信息。两者名称相近，但层级、用途和可见性不同。

运行时参数校验器由最终公开的 `inputSchema` 编译得到；分页、limit、时间戳和多口径条件必填会在发起请求前拒绝。日期有效性、代码市场匹配和时间跨度等业务规则仍会在对应工具业务层继续校验。

## 通用 Demo

下面示例可调用任意已记录工具。只需要改 `TOOL_NAME` 和 `TOOL_ARGS`，就可以复用到各个文档里的工具。

### curl

```bash
# MCP 服务入口；等同于文档里的 <MCP_BASE_URL>。
MCP_BASE_URL="https://market.ft.tech/gateway/mcp"

# 要调用的 MCP 工具名；可在下方工具索引或 tools/list 结果中查询。
TOOL_NAME="ft_get_cb_lists_handler"

# 工具业务参数；无参数工具传 {}，有参数时按对应工具文档填写 JSON 对象。
TOOL_ARGS='{}'

# initialize 请求体；用于创建 MCP 会话。
INIT_PAYLOAD='{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"curl-demo","version":"1.0.0"}}}'

# tools/call 请求体；name 是工具名，arguments 是工具业务参数。
CALL_PAYLOAD="{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"${TOOL_NAME}\",\"arguments\":${TOOL_ARGS}}}"

# 先调用 initialize，并从响应头中提取 Mcp-Session-Id。
MCP_SESSION_ID=$(curl -sS -m 10 -D - -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d "$INIT_PAYLOAD" \
  | awk 'tolower($1)=="mcp-session-id:" {print $2}' \
  | tr -d '\r')

# 通知服务端初始化已完成；成功时返回 HTTP 202 和空响应体。
curl -fsS -m 10 -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

# 初始化完成后调用目标工具；优先读取 result.structuredContent。
curl -sS -m 30 -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d "$CALL_PAYLOAD"
```

### Python

下面示例使用 Python MCP 客户端库调用任意 `ft_*` 工具。

依赖安装（Python 3.10+）：`pip install mcp`

```python
import asyncio  # 运行异步 main 函数。
from mcp import ClientSession  # MCP 客户端会话对象，用于 initialize、tools/list、tools/call。
from mcp.client.streamable_http import streamablehttp_client  # Streamable HTTP 客户端连接工具。

MCP_BASE_URL = "https://market.ft.tech/gateway/mcp"  # 实际公共 MCP 入口；等同于文档里的 <MCP_BASE_URL>。
TOOL_NAME = "ft_get_cb_lists_handler"  # 要调用的 MCP 工具名；可在下方工具索引或 tools/list 结果中查询。
TOOL_ARGS = {}  # 工具业务参数；无参数工具传空字典，有参数时按对应工具文档填写。

async def main():  # 定义异步入口函数。
    async with streamablehttp_client(  # 建立 MCP Streamable HTTP 连接。
        MCP_BASE_URL,  # MCP 服务入口地址。
    ) as (read_stream, write_stream, _):  # read_stream 读响应，write_stream 写请求，第三个返回值此处无需使用。
        async with ClientSession(  # 创建 MCP 客户端会话。
            read_stream,  # 服务端响应读取流。
            write_stream,  # 客户端请求写入流。
        ) as session:  # session 用于调用 MCP 协议方法。
            await session.initialize()  # 初始化协议并完成会话握手。

            tools = await session.list_tools()  # 可选：获取当前服务端可用工具列表。
            print([tool.name for tool in tools.tools])  # 可选：打印工具名，确认 TOOL_NAME 是否存在。

            result = await session.call_tool(  # 调用指定 MCP 工具。
                TOOL_NAME,  # params.name：工具名。
                TOOL_ARGS,  # params.arguments：传给工具的业务参数。
            )  # result.structuredContent 是统一 metadata/data 结构化输出。

            print(result.structuredContent)  # 推荐：统一 metadata/data 结构化输出。
            print(result.content[0].text)  # 兼容：与 structuredContent 同值的 JSON 文本。

asyncio.run(main())  # 启动异步程序。
```

## 社区交流

欢迎加入 FTShare 社区交流群，一起讨论 MCP 接入、工具调用、金融数据接口、Agent 投研工作流和项目贡献方向。

<img src="docs/assets/wechat-group.png" alt="FTShare 微信交流群" width="320" />

> **群规说明**：
> - 仅限 FTShare 项目、MCP 接入、金融数据接口和 Agent 投研工作流相关讨论
> - 禁止广告、推广、无关闲聊
> - Bug、功能需求和工具文档问题，建议优先在 GitHub Issues 中提交，群内用于快速交流和补充说明

如二维码失效，请在 Issues 中留言，维护者会更新入群方式。

## 按数据目录

| 大类 | 工具数 | 文档目录 |
|------|--------|----------|
| ETF专题 | 8 | [ETF专题/](./ETF专题/) |
| 债券专题 | 2 | [债券专题/](./债券专题/) |
| 公募基金 | 20 | [公募基金/](./公募基金/) |
| 外汇数据 | 1 | [外汇数据/](./外汇数据/) |
| 大模型语料 | 6 | [大模型语料/](./大模型语料/) |
| 宏观经济 | 17 | [宏观经济/](./宏观经济/) |
| 指数专题 | 8 | [指数专题/](./指数专题/) |
| 期货数据 | 7 | [期货数据/](./期货数据/) |
| 港股数据 | 7 | [港股数据/](./港股数据/) |
| 现货数据 | 2 | [现货数据/](./现货数据/) |
| 美股数据 | 7 | [美股数据/](./美股数据/) |
| 股票数据 | 83 | [股票数据/](./股票数据/) |

## 工具索引

| MCP 工具 | 标题 | 数据目录 | 文档 |
|----------|------|----------|------|
| `intraday_kline` | 分时与分钟 K 线 | 股票数据 | [股票数据/分时与分钟K线.md](./股票数据/分时与分钟K线.md) |
| `ft_etf_adjust_factor` | ETF复权因子 | ETF专题 | [ETF专题/ETF复权因子.md](./ETF专题/ETF复权因子.md) |
| `ft_etf_components_all` | ETF成份列表 | ETF专题 | [ETF专题/ETF成份列表.md](./ETF专题/ETF成份列表.md) |
| `ft_etf_description_all` | ETF基础信息 | ETF专题 | [ETF专题/ETF基础信息.md](./ETF专题/ETF基础信息.md) |
| `ft_etf_fund_export` | 指数ETF基金导出 | ETF专题 | [ETF专题/指数ETF基金导出.md](./ETF专题/指数ETF基金导出.md) |
| `ft_etf_pcf_list_handler` | ETF-PCF清单列表 | ETF专题 | [ETF专题/ETF-PCF清单列表.md](./ETF专题/ETF-PCF清单列表.md) |
| `ft_get_etf_components_handler` | ETF成份股 | ETF专题 | [ETF专题/ETF成份股.md](./ETF专题/ETF成份股.md) |
| `ft_get_etf_pre` | ETF盘前数据 | ETF专题 | [ETF专题/ETF盘前数据.md](./ETF专题/ETF盘前数据.md) |
| `ft_get_etf_pre_single_handler` | 单只ETF盘前数据 | ETF专题 | [ETF专题/单只ETF盘前数据.md](./ETF专题/单只ETF盘前数据.md) |
| `ft_get_cb_base_data_handler` | 可转债基础数据 | 债券专题 | [债券专题/可转债基础数据.md](./债券专题/可转债基础数据.md) |
| `ft_get_cb_lists_handler` | 可转债列表 | 债券专题 | [债券专题/可转债列表.md](./债券专题/可转债列表.md) |
| `ft_get_fund_basicinfo` | 基金基础信息 | 公募基金 | [公募基金/基金基础信息.md](./公募基金/基金基础信息.md) |
| `ft_get_fund_cal_return` | 基金收益 | 公募基金 | [公募基金/基金收益.md](./公募基金/基金收益.md) |
| `ft_get_fund_nav` | 基金净值 | 公募基金 | [公募基金/基金净值.md](./公募基金/基金净值.md) |
| `ft_get_fund_overview` | 基金总览 | 公募基金 | [公募基金/基金总览.md](./公募基金/基金总览.md) |
| `ft_get_fund_support_symbols` | 基金支持标的 | 公募基金 | [公募基金/基金支持标的.md](./公募基金/基金支持标的.md) |
| `ft_consumer_forex_gold_monthly` | 外汇黄金 | 外汇数据 | [外汇数据/外汇黄金.md](./外汇数据/外汇黄金.md) |
| `ft_semantic_search_news_handler` | 新闻语义搜索 | 大模型语料 | [大模型语料/新闻语义搜索.md](./大模型语料/新闻语义搜索.md) |
| `ft_shareholders_meeting` | 股东大会 | 大模型语料 | [大模型语料/股东大会.md](./大模型语料/股东大会.md) |
| `ft_stock_announcements` | 公告列表 | 大模型语料 | [大模型语料/公告列表.md](./大模型语料/公告列表.md) |
| `ft_stock_reports` | 研报列表 | 大模型语料 | [大模型语料/研报列表.md](./大模型语料/研报列表.md) |
| `ft_type_reports` | 研报分类 | 大模型语料 | [大模型语料/研报分类.md](./大模型语料/研报分类.md) |
| `ft_baidu_financial_calendar` | 百度财经日历 | 宏观经济 | [宏观经济/百度财经日历.md](./宏观经济/百度财经日历.md) |
| `ft_consumer_credit_monthly` | 社融信贷 | 宏观经济 | [宏观经济/社融信贷.md](./宏观经济/社融信贷.md) |
| `ft_consumer_customs_trade_monthly` | 进出口 | 宏观经济 | [宏观经济/进出口.md](./宏观经济/进出口.md) |
| `ft_consumer_fiscal_revenue_monthly` | 财政收入 | 宏观经济 | [宏观经济/财政收入.md](./宏观经济/财政收入.md) |
| `ft_consumer_fixed_asset_monthly` | 固定资产投资 | 宏观经济 | [宏观经济/固定资产投资.md](./宏观经济/固定资产投资.md) |
| `ft_consumer_gdp_quarterly` | GDP | 宏观经济 | [宏观经济/GDP.md](./宏观经济/GDP.md) |
| `ft_consumer_industrial_added_value_monthly` | 工业增加值 | 宏观经济 | [宏观经济/工业增加值.md](./宏观经济/工业增加值.md) |
| `ft_consumer_money_supply_monthly` | 货币供应 | 宏观经济 | [宏观经济/货币供应.md](./宏观经济/货币供应.md) |
| `ft_consumer_pmi_monthly` | PMI | 宏观经济 | [宏观经济/PMI.md](./宏观经济/PMI.md) |
| `ft_consumer_ppi_monthly` | PPI | 宏观经济 | [宏观经济/PPI.md](./宏观经济/PPI.md) |
| `ft_consumer_price_index_monthly` | CPI | 宏观经济 | [宏观经济/CPI.md](./宏观经济/CPI.md) |
| `ft_consumer_retail_sales_monthly` | 社零 | 宏观经济 | [宏观经济/社零.md](./宏观经济/社零.md) |
| `ft_lpr_monthly` | LPR | 宏观经济 | [宏观经济/LPR.md](./宏观经济/LPR.md) |
| `ft_reserve_ratio_monthly` | 存款准备金率 | 宏观经济 | [宏观经济/存款准备金率.md](./宏观经济/存款准备金率.md) |
| `ft_tax_revenue_monthly` | 税收 | 宏观经济 | [宏观经济/税收.md](./宏观经济/税收.md) |
| `ft_us_economic` | 美国经济指标 | 宏观经济 | [宏观经济/美国经济指标.md](./宏观经济/美国经济指标.md) |
| `ft_wallstreetcn_financial_calendar` | 华尔街见闻财经日历 | 宏观经济 | [宏观经济/华尔街见闻财经日历.md](./宏观经济/华尔街见闻财经日历.md) |
| `ft_global_index_daily_kline` | 全球指数日K线 | 指数专题 | [指数专题/全球指数日K线.md](./指数专题/全球指数日K线.md) |
| `ft_index_description_all` | 指数基础信息 | 指数专题 | [指数专题/指数基础信息.md](./指数专题/指数基础信息.md) |
| `ft_index_description_list_handler` | 中证指数描述列表 | 指数专题 | [指数专题/中证指数描述列表.md](./指数专题/中证指数描述列表.md) |
| `ft_index_weight_list_handler` | 指数权重列表 | 指数专题 | [指数专题/指数权重列表.md](./指数专题/指数权重列表.md) |
| `ft_index_weight_summary_handler` | 指数权重汇总 | 指数专题 | [指数专题/指数权重汇总.md](./指数专题/指数权重汇总.md) |
| `ft_sw_industry_constituent_history` | 申万行业成份股历史 | 指数专题 | [指数专题/申万行业成份股历史.md](./指数专题/申万行业成份股历史.md) |
| `ft_sw_industry_daily_metrics` | 申万行业日度指标 | 指数专题 | [指数专题/申万行业日度指标.md](./指数专题/申万行业日度指标.md) |
| `ft_sw_industry_overview` | 申万行业总览 | 指数专题 | [指数专题/申万行业总览.md](./指数专题/申万行业总览.md) |
| `ft_futures_contract_kline` | 期货合约K线 | 期货数据 | [期货数据/期货合约K线.md](./期货数据/期货合约K线.md) |
| `ft_get_china_futures_base_data_handler` | 中国期货基础数据 | 期货数据 | [期货数据/中国期货基础数据.md](./期货数据/中国期货基础数据.md) |
| `ft_get_china_futures_lists_handler` | 中国期货列表 | 期货数据 | [期货数据/中国期货列表.md](./期货数据/中国期货列表.md) |
| `ft_get_eastmoney_futures_position` | 东方财富期货持仓 | 期货数据 | [期货数据/东方财富期货持仓.md](./期货数据/东方财富期货持仓.md) |
| `ft_major_contract` | 重大合同 | 期货数据 | [期货数据/重大合同.md](./期货数据/重大合同.md) |
| `ft_major_contract_by_symbol` | 重大合同按标的 | 期货数据 | [期货数据/重大合同按标的.md](./期货数据/重大合同按标的.md) |
| `ft_major_contract_summary` | 重大合同汇总 | 期货数据 | [期货数据/重大合同汇总.md](./期货数据/重大合同汇总.md) |
| `ft_get_company_hk` | 港股公司信息 | 港股数据 | [港股数据/港股公司信息.md](./港股数据/港股公司信息.md) |
| `ft_get_eastmoney_hk_index_daily_kline` | 东方财富港股指数日K | 港股数据 | [港股数据/东方财富港股指数日K.md](./港股数据/东方财富港股指数日K.md) |
| `ft_get_hk_basinfo` | 港股个股信息 | 港股数据 | [港股数据/港股个股信息.md](./港股数据/港股个股信息.md) |
| `ft_get_hk_candlesticks` | 港股K线 | 港股数据 | [港股数据/港股K线.md](./港股数据/港股K线.md) |
| `ft_get_hk_valuatnanalyd` | 港股估值分析 | 港股数据 | [港股数据/港股估值分析.md](./港股数据/港股估值分析.md) |
| `ft_get_market_cap_hk` | 港股市值 | 港股数据 | [港股数据/港股市值.md](./港股数据/港股市值.md) |
| `ft_hk_cashflow` | 港股现金流量表 | 港股数据 | [港股数据/港股现金流量表.md](./港股数据/港股现金流量表.md) |
| `ft_get_bullion_price` | 贵金属价格 | 现货数据 | [现货数据/贵金属价格.md](./现货数据/贵金属价格.md) |
| `ft_get_bullion_support_symbol` | 贵金属支持标的 | 现货数据 | [现货数据/贵金属支持标的.md](./现货数据/贵金属支持标的.md) |
| `ft_eastmoney_us_stock_daily_kline` | 东方财富美股日OHLC | 美股数据 | [美股数据/东方财富美股日OHLC.md](./美股数据/东方财富美股日OHLC.md) |
| `ft_eastmoney_us_stock_latest_kline` | 东方财富美股最新OHLC | 美股数据 | [美股数据/东方财富美股最新OHLC.md](./美股数据/东方财富美股最新OHLC.md) |
| `ft_eastmoney_us_stock_list` | 东方财富美股列表 | 美股数据 | [美股数据/东方财富美股列表.md](./美股数据/东方财富美股列表.md) |
| `ft_us_balance` | 美股资产负债表 | 美股数据 | [美股数据/美股资产负债表.md](./美股数据/美股资产负债表.md) |
| `ft_us_basic` | 美股基础信息 | 美股数据 | [美股数据/美股基础信息.md](./美股数据/美股基础信息.md) |
| `ft_us_cashflow` | 美股现金流 | 美股数据 | [美股数据/美股现金流.md](./美股数据/美股现金流.md) |
| `ft_us_income` | 美股利润表 | 美股数据 | [美股数据/美股利润表.md](./美股数据/美股利润表.md) |
| `ft_abnormal_trading_details` | 龙虎榜明细 | 股票数据 | [股票数据/龙虎榜明细.md](./股票数据/龙虎榜明细.md) |
| `ft_abnormal_trading_overview` | 龙虎榜总览 | 股票数据 | [股票数据/龙虎榜总览.md](./股票数据/龙虎榜总览.md) |
| `ft_auction_results` | 集合竞价结果 | 股票数据 | [股票数据/集合竞价结果.md](./股票数据/集合竞价结果.md) |
| `ft_balance` | A股资产负债表 | 股票数据 | [股票数据/A股资产负债表.md](./股票数据/A股资产负债表.md) |
| `ft_cashflow` | A股现金流量表 | 股票数据 | [股票数据/A股现金流量表.md](./股票数据/A股现金流量表.md) |
| `ft_earnings_reports_paginated` | 业绩快报 | 股票数据 | [股票数据/业绩快报.md](./股票数据/业绩快报.md) |
| `ft_eastmoney_board_constituents` | 东方财富板块成份股 | 股票数据 | [股票数据/东方财富板块成份股.md](./股票数据/东方财富板块成份股.md) |
| `ft_eastmoney_board_daily_kline` | 东方财富板块日线OHLC | 股票数据 | [股票数据/东方财富板块日线OHLC.md](./股票数据/东方财富板块日线OHLC.md) |
| `ft_eastmoney_board_latest_kline` | 东方财富板块最新OHLC | 股票数据 | [股票数据/东方财富板块最新OHLC.md](./股票数据/东方财富板块最新OHLC.md) |
| `ft_eastmoney_concept_boards` | 东方财富概念板块 | 股票数据 | [股票数据/东方财富概念板块.md](./股票数据/东方财富概念板块.md) |
| `ft_eastmoney_rank` | 东方财富股票排名 | 股票数据 | [股票数据/东方财富股票排名.md](./股票数据/东方财富股票排名.md) |
| `ft_get_bse_mapping` | 北交所映射 | 股票数据 | [股票数据/北交所映射.md](./股票数据/北交所映射.md) |
| `ft_get_cashflow_stock_code` | 现金流支持股票代码 | 股票数据 | [股票数据/现金流支持股票代码.md](./股票数据/现金流支持股票代码.md) |
| `ft_get_eastmoney_dapan_flow` | 东方财富大盘资金流 | 股票数据 | [股票数据/东方财富大盘资金流.md](./股票数据/东方财富大盘资金流.md) |
| `ft_get_eastmoney_market_valuation` | 东方财富市场估值 | 股票数据 | [股票数据/东方财富市场估值.md](./股票数据/东方财富市场估值.md) |
| `ft_get_eastmoney_sector_flow` | 东方财富板块资金流 | 股票数据 | [股票数据/东方财富板块资金流.md](./股票数据/东方财富板块资金流.md) |
| `ft_get_eastmoney_stock_flow` | 东方财富个股资金流 | 股票数据 | [股票数据/东方财富个股资金流.md](./股票数据/东方财富个股资金流.md) |
| `ft_get_eastmoney_stock_valuation` | 东方财富个股估值 | 股票数据 | [股票数据/东方财富个股估值.md](./股票数据/东方财富个股估值.md) |
| `ft_get_nth_trade_date` | 第N个交易日 | 股票数据 | [股票数据/第N个交易日.md](./股票数据/第N个交易日.md) |
| `ft_get_price_change` | 价格变动 | 股票数据 | [股票数据/价格变动.md](./股票数据/价格变动.md) |
| `ft_get_stk_ah_comparison` | AH股对比 | 股票数据 | [股票数据/AH股对比.md](./股票数据/AH股对比.md) |
| `ft_get_stock_institution_holdings` | 机构持股 | 股票数据 | [股票数据/机构持股.md](./股票数据/机构持股.md) |
| `ft_get_stock_institution_holdings_detail` | 机构持股明细 | 股票数据 | [股票数据/机构持股明细.md](./股票数据/机构持股明细.md) |
| `ft_get_stock_institution_share_holdings` | 机构股本持股 | 股票数据 | [股票数据/机构股本持股.md](./股票数据/机构股本持股.md) |
| `ft_get_stock_list` | 股票列表 | 股票数据 | [股票数据/股票列表.md](./股票数据/股票列表.md) |
| `ft_get_stock_share_handler` | 股本 | 股票数据 | [股票数据/股本.md](./股票数据/股本.md) |
| `ft_goodwill_industry` | 商誉行业 | 股票数据 | [股票数据/商誉行业.md](./股票数据/商誉行业.md) |
| `ft_goodwill_market_overview` | 商誉市场总览 | 股票数据 | [股票数据/商誉市场总览.md](./股票数据/商誉市场总览.md) |
| `ft_goodwill_predict` | 商誉预测 | 股票数据 | [股票数据/商誉预测.md](./股票数据/商誉预测.md) |
| `ft_goodwill_stock_detail` | 商誉个股明细 | 股票数据 | [股票数据/商誉个股明细.md](./股票数据/商誉个股明细.md) |
| `ft_goodwill_stock_impairment` | 商誉减值 | 股票数据 | [股票数据/商誉减值.md](./股票数据/商誉减值.md) |
| `ft_hk_sh_stock_connect_members` | 沪港通成份 | 股票数据 | [股票数据/沪港通成份.md](./股票数据/沪港通成份.md) |
| `ft_hk_sz_stock_connect_members` | 深港通成份 | 股票数据 | [股票数据/深港通成份.md](./股票数据/深港通成份.md) |
| `ft_income` | A股利润表 | 股票数据 | [股票数据/A股利润表.md](./股票数据/A股利润表.md) |
| `ft_limit_down_pool` | 跌停池 | 股票数据 | [股票数据/跌停池.md](./股票数据/跌停池.md) |
| `ft_limit_event_timeline_3s` | 涨跌停事件时间线 | 股票数据 | [股票数据/涨跌停事件时间线.md](./股票数据/涨跌停事件时间线.md) |
| `ft_limit_up_break_pool` | 炸板池 | 股票数据 | [股票数据/炸板池.md](./股票数据/炸板池.md) |
| `ft_limit_up_pool` | 涨停池 | 股票数据 | [股票数据/涨停池.md](./股票数据/涨停池.md) |
| `ft_limit_up_pool_yesterday` | 昨日涨停池 | 股票数据 | [股票数据/昨日涨停池.md](./股票数据/昨日涨停池.md) |
| `ft_margin_trading_details` | 融资融券明细 | 股票数据 | [股票数据/融资融券明细.md](./股票数据/融资融券明细.md) |
| `ft_margin_trading_details_paginated` | 融资融券明细分页 | 股票数据 | [股票数据/融资融券明细分页.md](./股票数据/融资融券明细分页.md) |
| `ft_northbound` | 北向资金交易 | 股票数据 | [股票数据/北向资金交易.md](./股票数据/北向资金交易.md) |
| `ft_performance_forecasts_paginated` | 业绩预告 | 股票数据 | [股票数据/业绩预告.md](./股票数据/业绩预告.md) |
| `ft_risk_warning_stock_quotes` | 风险警示股行情 | 股票数据 | [股票数据/风险警示股行情.md](./股票数据/风险警示股行情.md) |
| `ft_risk_warning_stocks` | 风险警示股 | 股票数据 | [股票数据/风险警示股.md](./股票数据/风险警示股.md) |
| `ft_sh_hk_stock_connect_members` | 沪股通成份 | 股票数据 | [股票数据/沪股通成份.md](./股票数据/沪股通成份.md) |
| `ft_southbound` | 南向资金交易 | 股票数据 | [股票数据/南向资金交易.md](./股票数据/南向资金交易.md) |
| `ft_stk_limit` | 涨跌停价 | 股票数据 | [股票数据/涨跌停价.md](./股票数据/涨跌停价.md) |
| `ft_stk_premarket` | 盘前数据 | 股票数据 | [股票数据/盘前数据.md](./股票数据/盘前数据.md) |
| `ft_stock_adjust_factor` | 股票复权因子 | 股票数据 | [股票数据/股票复权因子.md](./股票数据/股票复权因子.md) |
| `ft_stock_candlesticks` | 股票K线 | 股票数据 | [股票数据/股票K线.md](./股票数据/股票K线.md) |
| `ft_stock_candlesticks_batch` | 批量股票K线 | 股票数据 | [股票数据/批量股票K线.md](./股票数据/批量股票K线.md) |
| `ft_stock_capital_flows_paginated` | 股票资金流向 | 股票数据 | [股票数据/股票资金流向.md](./股票数据/股票资金流向.md) |
| `ft_stock_comment_desire_em` | 千股千评意愿度 | 股票数据 | [股票数据/千股千评意愿度.md](./股票数据/千股千评意愿度.md) |
| `ft_stock_comment_em` | 千股千评 | 股票数据 | [股票数据/千股千评.md](./股票数据/千股千评.md) |
| `ft_stock_comment_focus_em` | 千股千评关注度 | 股票数据 | [股票数据/千股千评关注度.md](./股票数据/千股千评关注度.md) |
| `ft_stock_comment_org_participate_em` | 机构参与度 | 股票数据 | [股票数据/机构参与度.md](./股票数据/机构参与度.md) |
| `ft_stock_comment_score_em` | 千股千评评分 | 股票数据 | [股票数据/千股千评评分.md](./股票数据/千股千评评分.md) |
| `ft_stock_filter` | 股票筛选 | 股票数据 | [股票数据/股票筛选.md](./股票数据/股票筛选.md) |
| `ft_stock_float_holders` | 十大流通股东 | 股票数据 | [股票数据/十大流通股东.md](./股票数据/十大流通股东.md) |
| `ft_stock_ggcg_em_handler` | 东方财富股东增减持 | 股票数据 | [股票数据/东方财富股东增减持.md](./股票数据/东方财富股东增减持.md) |
| `ft_stock_ggmx_buy_ranking_handler` | 董监高增持排名 | 股票数据 | [股票数据/董监高增持排名.md](./股票数据/董监高增持排名.md) |
| `ft_stock_ggmx_handler` | 董监高持股变动 | 股票数据 | [股票数据/董监高持股变动.md](./股票数据/董监高持股变动.md) |
| `ft_stock_ggmx_sell_ranking_handler` | 董监高减持排名 | 股票数据 | [股票数据/董监高减持排名.md](./股票数据/董监高减持排名.md) |
| `ft_stock_holders` | 十大股东 | 股票数据 | [股票数据/十大股东.md](./股票数据/十大股东.md) |
| `ft_stock_holders_number` | 股东人数 | 股票数据 | [股票数据/股东人数.md](./股票数据/股东人数.md) |
| `ft_stock_intraday_auction_volume` | 集合竞价成交量 | 股票数据 | [股票数据/集合竞价成交量.md](./股票数据/集合竞价成交量.md) |
| `ft_stock_intraday_auction_volume_symbol` | 单标的集合竞价成交量 | 股票数据 | [股票数据/单标的集合竞价成交量.md](./股票数据/单标的集合竞价成交量.md) |
| `ft_stock_ipos` | 股票IPO | 股票数据 | [股票数据/股票IPO.md](./股票数据/股票IPO.md) |
| `ft_stock_pledge_detail` | 股权质押明细 | 股票数据 | [股票数据/股权质押明细.md](./股票数据/股权质押明细.md) |
| `ft_stock_pledge_summary` | 股权质押汇总 | 股票数据 | [股票数据/股权质押汇总.md](./股票数据/股权质押汇总.md) |
| `ft_stock_rating_top5` | 飞兔股票评级Top5 | 股票数据 | [股票数据/飞兔股票评级Top5.md](./股票数据/飞兔股票评级Top5.md) |
| `ft_stock_share_chg` | 股东增减持 | 股票数据 | [股票数据/股东增减持.md](./股票数据/股东增减持.md) |
| `ft_stock_unlock_by_date_handler` | 限售解禁按日期 | 股票数据 | [股票数据/限售解禁按日期.md](./股票数据/限售解禁按日期.md) |
| `ft_stock_unlock_handler` | 限售解禁 | 股票数据 | [股票数据/限售解禁.md](./股票数据/限售解禁.md) |
| `ft_suspension_list` | 停牌列表 | 股票数据 | [股票数据/停牌列表.md](./股票数据/停牌列表.md) |
| `ft_sz_hk_stock_connect_members` | 深股通成份 | 股票数据 | [股票数据/深股通成份.md](./股票数据/深股通成份.md) |
| `ft_ths_all_board_kline` | 同花顺全板块K线 | 股票数据 | [股票数据/同花顺全板块K线.md](./股票数据/同花顺全板块K线.md) |
| `ft_ths_board_kline` | 同花顺板块K线 | 股票数据 | [股票数据/同花顺板块K线.md](./股票数据/同花顺板块K线.md) |
| `ft_ths_board_list` | 同花顺板块列表 | 股票数据 | [股票数据/同花顺板块列表.md](./股票数据/同花顺板块列表.md) |
| `ft_xueqiu_rank` | 雪球股票排名 | 股票数据 | [股票数据/雪球股票排名.md](./股票数据/雪球股票排名.md) |

## License

本项目文档采用 MIT License，详见 [LICENSE](LICENSE)。

MIT License 适用于本仓库中的文档与示例，不代表 FTShare 数据服务本身无限制开放。FTShare 数据接口的访问额度、权限和商业用途，以 FTShare 数据服务条款为准。
