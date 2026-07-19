# FTShare MCP

[中文](README.md) | [English](README_EN.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
![MCP](https://img.shields.io/badge/MCP-Streamable_HTTP-168AAD)
![Tools](https://img.shields.io/badge/tools-172-blue)

FTShare MCP 是面向 AI Agent 的金融数据 MCP 服务，让 Claude Code、Codex、WorkBuddy、OpenClaw、Hermes 等支持 MCP 的客户端，可以通过自然语言调用 FTShare 的行情、财务、宏观、基金、期货、债券、美股和港股数据。

> 本仓库提供 MCP 工具文档、参数说明和接入示例，不包含 MCP Server 源码。公共 MCP 服务由 FTShare 数据服务提供。

## 当前服务

- **服务版本**：FTShare MCP Server `0.1.1`
- **公共地址**：`https://market.ft.tech/gateway/mcp`
- **传输协议**：MCP Streamable HTTP
- **工具数量**：172
- **工具构成**：165 个 `ft_*` 数据工具 + 7 个便捷查询入口
- **服务属性**：只读金融数据服务
- **实时工具定义**：以 MCP `tools/list` 返回结果为准

各工具文档中的 `<MCP_BASE_URL>` 均应替换为公共地址。首次直接调用协议时，先通过 `initialize` 获取 `Mcp-Session-Id`，后续请求再携带该会话 ID 调用 `tools/list` 或 `tools/call`。

## 数据能力

- A 股行情、K 线、涨跌停、资金流、龙虎榜、股东和财务数据
- ETF、指数、基金净值、基金持仓、基金经理和基金公司
- 港股、美股、期货、债券和贵金属
- 宏观经济、财经日历、公告、研报和财经新闻
- 语义新闻检索及面向 Agent 的便捷查询入口

## FTShare 生态中的位置

FTShare 是底层金融数据服务，SDK、MCP 和 Skill 是不同的使用方式：

```text
FTShare 数据服务
    ├── FTShare Python SDK   开发者编程调用
    ├── FTShare MCP          Agent 标准化数据调用
    └── FTShare Skills       投研任务与业务工作流
```

MCP 负责让 Agent 获取数据，Skill 负责告诉 Agent 如何完成一项投研任务。

## 快速接入

### Claude Code

```bash
claude mcp add --transport http --scope user ftshare https://market.ft.tech/gateway/mcp
```

进入 Claude Code 后输入 `/mcp`，确认列表中存在 `ftshare`。

### Codex

在 `~/.codex/config.toml` 中加入：

```toml
[mcp_servers.ftshare]
url = "https://market.ft.tech/gateway/mcp"
```

保存后执行 `codex mcp get ftshare` 检查配置。

### WorkBuddy

在 `~/.workbuddy/mcp.json` 中加入：

```json
{
  "mcpServers": {
    "ftshare": {
      "type": "streamableHttp",
      "url": "https://market.ft.tech/gateway/mcp",
      "timeout": 30000,
      "disabled": false
    }
  }
}
```

其他支持 Streamable HTTP MCP 的客户端，也可以使用同一个公共地址接入。

## 使用示例

配置完成后，可以直接向 Agent 提问：

- 使用 FTShare 查询贵州茅台最近 5 个交易日的日 K 线
- 查询今天 A 股涨停和跌停数量
- 查询沪深 300 对应的 ETF
- 查询某只基金的净值、持仓和基金经理
- 搜索最近与机器人概念相关的财经新闻

## 返回结构

从 `0.1.1` 开始，所有工具统一返回结构化 JSON。成功结果同时提供 `structuredContent`，以及 `content[0].text` 中的同值 JSON 文本：

```json
{
  "metadata": {
    "schema_version": "1.0",
    "source": "ftshare",
    "tool": "ft_get_fund_list",
    "operation": "get_fund_list",
    "total": 1,
    "pagination": {
      "supported": true,
      "page": 1,
      "page_size": 1,
      "pages": 1,
      "has_more": false
    },
    "returned": 1,
    "truncated": false,
    "warnings": []
  },
  "data": []
}
```

Agent 和应用程序应优先读取 `structuredContent.data`；分页、截断和告警信息位于 `structuredContent.metadata`。

## 错误处理

参数或业务错误通过 `isError=true` 返回，`content[0].text` 中包含可机器解析的错误对象：

```json
{
  "error": {
    "code": "INVALID_ARGUMENT",
    "field": "page_size",
    "message": "参数值不满足约束",
    "retryable": false
  }
}
```

| 错误码 | 含义 |
|---|---|
| `MISSING_PARAMETER` | 缺少必填或条件必填参数 |
| `INVALID_TYPE` | 参数类型不正确 |
| `UNKNOWN_PARAMETER` | 传入了 Schema 未声明的参数 |
| `INVALID_ARGUMENT` | 参数值或业务条件不满足 |
| `UPSTREAM_UNAVAILABLE` | 上游数据服务暂时不可用 |

调用工具时应严格遵守 `tools/list` 返回的 `inputSchema`。成功时优先读取 `structuredContent`，失败时先判断 `isError`，再解析错误对象中的 `code` 和 `retryable`。

## 0.1.1 更新

- 工具数量由 159 调整为 172
- 移除 `echo`、`aggregate_demo` 两个示例工具
- 新增 15 个公募基金工具
- 所有工具统一提供 `title`、`inputSchema`、`outputSchema` 和只读 annotations
- 所有成功结果统一提供 `structuredContent`
- 未声明参数、缺少必填参数和超限参数在调用前被拒绝
- 业务错误统一返回 `isError=true` 和结构化错误码

## 社区交流

欢迎加入 FTShare 社区交流群，一起讨论 MCP 接入、工具调用、金融数据接口、Agent 投研工作流和项目贡献方向。

<img src="docs/assets/wechat-group-20260721.png" alt="FTShare 微信交流群" width="320" />

> **群规说明**：
> - 仅限 FTShare 项目、MCP 接入、金融数据接口和 Agent 投研工作流相关讨论
> - 禁止广告、推广、无关闲聊
> - Bug、功能需求和工具文档问题，建议优先在 GitHub Issues 中提交，群内用于快速交流和补充说明

**二维码有效期至 2026 年 7 月 21 日。** 如二维码失效，请在 Issues 中留言，维护者会更新入群方式。

## 相关项目

- [FTShare-python-sdk](https://github.com/FTShare-Lab/FTShare-python-sdk)：FTShare 金融数据 Python SDK，面向开发者编程调用
- [FTShare-skills](https://github.com/FTShare-Lab/FTShare-skills)：FTShare Agent Skill 仓库，面向数据级 Skill 和投研业务 Skill

## 按数据目录

| 大类 | 工具数 | 文档目录 |
|------|--------|----------|
| ETF专题 | 8 | [ETF专题/](./ETF专题/) |
| 债券专题 | 2 | [债券专题/](./债券专题/) |
| 公募基金 | 20 | [公募基金/](./公募基金/)；新增工具的独立文档正在同步 |
| 外汇数据 | 1 | [外汇数据/](./外汇数据/) |
| 大模型语料 | 5 | [大模型语料/](./大模型语料/) |
| 宏观经济 | 17 | [宏观经济/](./宏观经济/) |
| 指数专题 | 8 | [指数专题/](./指数专题/) |
| 期货数据 | 7 | [期货数据/](./期货数据/) |
| 港股数据 | 7 | [港股数据/](./港股数据/) |
| 现货数据 | 2 | [现货数据/](./现货数据/) |
| 美股数据 | 7 | [美股数据/](./美股数据/) |
| 股票数据 | 81 | [股票数据/](./股票数据/) |
| 便捷查询入口 | 7 | 通过 `tools/list` 查看实时 Schema |
| **合计** | **172** | 165 个数据工具 + 7 个便捷查询入口 |

## 工具索引

| MCP 工具 | 标题 | 数据目录 | 文档 |
|----------|------|----------|------|
| `capital_flow` | 资金流 | 便捷查询入口 | 实时 Schema：`tools/list` |
| `daily_ohlc` | 日频 OHLC | 便捷查询入口 | 实时 Schema：`tools/list` |
| `intraday_kline` | 分时与分钟 K 线 | 便捷查询入口 | 实时 Schema：`tools/list` |
| `margin` | 融资融券 | 便捷查询入口 | 实时 Schema：`tools/list` |
| `report_announcement_list` | 公告列表 | 便捷查询入口 | 实时 Schema：`tools/list` |
| `report_announcement_summary` | 公告摘要 | 便捷查询入口 | 实时 Schema：`tools/list` |
| `semantic_search_news` | 新闻语义检索 | 便捷查询入口 | 实时 Schema：`tools/list` |
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
| `ft_get_fund_asset_allocation` | 基金资产配置 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_classification` | 基金分类 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_company` | 基金公司 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_daily` | 基金行情日线 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_fee` | 基金费率 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_holder_structure` | 基金持有人结构 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_index_fund` | 指数跟踪基金 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_list` | 基金列表 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_manager` | 基金经理任职关系 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_net_value` | 基金净值明细 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_net_value_performance` | 基金净值收益表现 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_new_found` | 基金新发 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_portfolio` | 基金持仓明细 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_risk_level` | 基金风险等级 | 公募基金 | 实时 Schema：`tools/list` |
| `ft_get_fund_share` | 基金份额 | 公募基金 | 实时 Schema：`tools/list` |
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
| `ft_stock_rating_top5` | 非凸股票评级Top5 | 股票数据 | [股票数据/非凸股票评级Top5.md](./股票数据/非凸股票评级Top5.md) |
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
