# FTShare MCP

[中文](README.md) | [English](README_EN.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

`FTShare-MCP` is the MCP tool documentation and integration guide for FTShare financial data capabilities. It is designed for MCP-compatible clients, Agent applications, and automated investment research workflows. This repository provides the public MCP endpoint, calling workflow, tool index, parameter descriptions, and examples.

For international developers, this repository can be understood as **FTShare MCP tools for financial data, market data, quantitative research, and AI Agent investment workflows**.

> This repository focuses on **MCP tool documentation and integration instructions**. It does not contain the MCP Server source code. The public MCP service is provided by the FTShare data service.

The current service exposes **172 tools**: 165 `ft_*` financial data tools and 7 convenience query tools. Together they cover market data, financial statements, macro data, funds, futures, bonds, US stocks, Hong Kong stocks, and related datasets. The live `tools/list` response is the source of truth.

## Public MCP Endpoint

This documentation uses `<MCP_BASE_URL>` as the placeholder for the MCP Streamable HTTP endpoint. The public MCP endpoint is:

```bash
MCP_BASE_URL="https://market.ft.tech/gateway/mcp"
```

When copying examples from tool documents, replace `<MCP_BASE_URL>` with the URL above.

- **MCP endpoint placeholder**: `<MCP_BASE_URL>` (Streamable HTTP)
- **Protocol flow**: call `initialize` to negotiate the version and get `Mcp-Session-Id`, send `notifications/initialized`, then call tools with the session ID and `MCP-Protocol-Version`
- **Return format**: one JSON `TextContent` block plus an identical `structuredContent`; no extra Markdown block
- **Live descriptor**: `tools/list` returns `title`, `inputSchema`, `outputSchema`, `annotations`, and `_meta`

> **Deployment status**: The output format below is the unified contract implemented by the
> matching new `ftshare-mcp-server`. Check live `tools/list` / `tools/call` responses to confirm
> whether a public environment has been upgraded. If a documented tool still lacks `outputSchema`
> or its successful result has no `structuredContent`, that environment is still running an older server.

## Use Cases

- MCP-compatible clients such as Claude Code, Codex, Cursor, and OpenClaw
- Financial data tool calls in Agent-based investment research workflows
- Automated queries for market data, financial statements, macro data, funds, futures, bonds, US stocks, and Hong Kong stocks
- Data access validation for investment research, quantitative research, and financial application development

## Calling Workflow

1. Confirm the MCP endpoint. For the public environment, use `https://market.ft.tech/gateway/mcp`.
2. Choose a tool from the tool index or from the result of `tools/list`.
3. Check parameters in the corresponding tool document or from the live `inputSchema`.
4. Initialize a session by calling MCP `initialize` and reading `Mcp-Session-Id`.
5. Complete initialization by sending `notifications/initialized`; a successful HTTP response is 202 with an empty body.
6. Call `tools/call` with `Mcp-Session-Id` and the negotiated `MCP-Protocol-Version`, where `name` is the tool name and `arguments` is the business parameter object.
7. Read `result.structuredContent` first. `result.content[0].text` is the serialized JSON form of the same value for clients that do not yet consume structured results.
8. Unknown tools and malformed request shapes are JSON-RPC protocol errors. Input validation, upstream API failures, and business execution failures are returned in `result` with `isError: true`.

## Unified Output Format

All tools use the same structured output schema. Tool-specific fields are stored in the `data` array:

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
      "full_name": "Example convertible bond",
      "stock_id": 600001,
      "exchange": 3553
    }
  ]
}
```

- `metadata.schema_version`: output contract version, currently `1.0`.
- `metadata.source`: data source, currently `ftshare`.
- `metadata.tool`: the MCP tool name that was called.
- `metadata.operation`: the executed business operation; `ft_*` tools use the corresponding SDK method.
- `metadata.total`: total matched rows, or the pre-truncation local count when the upstream omits a total.
- `metadata.pagination`: `supported`, `page`, `page_size`, `pages`, and `has_more`.
- `metadata.returned`: rows actually included in `data`.
- `metadata.truncated`: whether server item or output-size limits truncated the result.
- `metadata.warnings`: deduplication, shape, or truncation notices; empty when there are none.

The complete successful `tools/call` result contains exactly one JSON text block whose parsed value equals `structuredContent`. Legacy `items` / `records` map to `data`, while `total_items` / `total_pages` map to `metadata.total` / `metadata.pagination.pages`. Markdown is not required by MCP and is not returned by default.

Tool execution errors do not use the successful `outputSchema`. An error result sets `isError: true`, omits `structuredContent`, and returns a sanitized message in `content[0].text`. Machine-readable errors use `{"error":{"code","message","field?","retryable","details?"}}` JSON. When alternative required groups are missing, the error omits a misleading single `field` and lists the accepted groups in `details.required_any_of`. Unknown tools and malformed JSON-RPC requests remain protocol errors.

## Tool Descriptors

Each documented tool descriptor returned by `tools/list` aligns with the read-only MCP / OpenAI Apps SDK semantics:

- `title`: human-readable tool title.
- `inputSchema`: Draft 2020-12 input constraints; undeclared fields are rejected. The server removes Rust integer `format`, `$schema`, and `default:null`, then adds polymorphic conditional requirements and integer bounds.
- `outputSchema`: the shared successful `metadata/data` contract.
- `annotations`: `readOnlyHint=true`, `destructiveHint=false`, and `openWorldHint=true`.
- `_meta.securitySchemes`: `[{"type":"noauth"}]` for OpenAI client compatibility.

`structuredContent.metadata` is model-visible business result data. The descriptor sibling `_meta` carries client extension metadata; the two fields have different levels, purposes, and visibility.

Runtime validators are compiled from the final public `inputSchema`. Pagination, limits, timestamps, and polymorphic conditional requirements are rejected before any upstream request. Business rules such as calendar validity, market-code matching, and range spans remain enforced by the corresponding tool implementation.

## Generic Demo

The following examples can call any documented tool. Change only `TOOL_NAME` and `TOOL_ARGS`.

### curl

```bash
MCP_BASE_URL="https://market.ft.tech/gateway/mcp"
TOOL_NAME="ft_get_cb_lists_handler"
TOOL_ARGS='{}'

INIT_PAYLOAD='{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"curl-demo","version":"1.0.0"}}}'
CALL_PAYLOAD="{\"jsonrpc\":\"2.0\",\"id\":2,\"method\":\"tools/call\",\"params\":{\"name\":\"${TOOL_NAME}\",\"arguments\":${TOOL_ARGS}}}"

MCP_SESSION_ID=$(curl -sS -m 10 -D - -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -d "$INIT_PAYLOAD" \
  | awk 'tolower($1)=="mcp-session-id:" {print $2}' \
  | tr -d '\r')

curl -fsS -m 10 -o /dev/null -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized"}'

curl -sS -m 30 -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
  -H "MCP-Protocol-Version: 2025-11-25" \
  -d "$CALL_PAYLOAD"
```

### Python

Install dependency: `pip install mcp`

```python
import asyncio
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

MCP_BASE_URL = "https://market.ft.tech/gateway/mcp"
TOOL_NAME = "ft_get_cb_lists_handler"
TOOL_ARGS = {}

async def main():
    async with streamablehttp_client(MCP_BASE_URL) as (read_stream, write_stream, _):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            tools = await session.list_tools()
            print([tool.name for tool in tools.tools])
            result = await session.call_tool(TOOL_NAME, TOOL_ARGS)
            print(result.structuredContent)
            print(result.content[0].text)  # Compatible JSON text identical to structuredContent.

asyncio.run(main())
```

## Tool Categories

| Category | Tool Count | Documentation |
|----------|------------|---------------|
| ETF | 8 | [ETF专题/](./ETF专题/) |
| Bonds | 2 | [债券专题/](./债券专题/) |
| Public Funds | 20 | [公募基金/](./公募基金/) |
| FX Data | 1 | [外汇数据/](./外汇数据/) |
| LLM Corpus | 5 | [大模型语料/](./大模型语料/) |
| Macro Economy | 17 | [宏观经济/](./宏观经济/) |
| Index Data | 8 | [指数专题/](./指数专题/) |
| Futures Data | 7 | [期货数据/](./期货数据/) |
| Hong Kong Stocks | 7 | [港股数据/](./港股数据/) |
| Spot Data | 2 | [现货数据/](./现货数据/) |
| US Stocks | 7 | [美股数据/](./美股数据/) |
| A-share Stocks | 81 | [股票数据/](./股票数据/) |

### Convenience query tools

| MCP Tool | Title | Documentation |
|----------|-------|---------------|
| `capital_flow` | Capital Flow | Live schema: `tools/list` |
| `daily_ohlc` | Daily OHLC | [股票数据/日频OHLC.md](./股票数据/日频OHLC.md) |
| `intraday_kline` | Intraday Price and Minute K-line | [股票数据/分时与分钟K线.md](./股票数据/分时与分钟K线.md) |
| `margin` | Margin Trading | Live schema: `tools/list` |
| `report_announcement_list` | Announcement List | Live schema: `tools/list` |
| `report_announcement_summary` | Announcement Summary | [大模型语料/公告摘要.md](./大模型语料/公告摘要.md) |
| `semantic_search_news` | Semantic News Search | Live schema: `tools/list` |

For the full tool index and individual tool documentation, see the Chinese [README.md](README.md) and the category documents. The tool names, parameter names, and examples are language-independent.

## Notes

- The live `inputSchema` returned by `tools/list` defines parameter structure and preflight constraints; business rules such as calendar validity, market-code matching, and range spans remain defined by the tool documentation and runtime errors.
- Tool documents cover the officially documented `ft_*` financial data tools and the listed server-level aggregation tools.
- All documented tools use the same `metadata` / `data` structured output envelope.
- Some tools have time-window, pagination, quota, or permission constraints.
- Access quota, permissions, and commercial use of FTShare data interfaces are subject to FTShare data service terms.

## Related Projects

- [FTShare-python-sdk](https://github.com/FTShare-Lab/FTShare-python-sdk): FTShare financial data Python SDK for developer-facing data access
- [FTShare-skills](https://github.com/FTShare-Lab/FTShare-skills): FTShare Agent Skills repository for data-level Skills and investment research workflow Skills

## Community

Chinese users are welcome to join the FTShare WeChat community group to discuss MCP integration, tool calls, financial data interfaces, Agent research workflows, and contribution directions.

<img src="docs/assets/wechat-group-20260721.png" alt="FTShare WeChat community group" width="320" />

> **Community rules**:
> - Discussions should be related to FTShare, MCP integration, financial data interfaces, or Agent research workflows
> - Advertising, promotion, and unrelated off-topic chat are not allowed
> - For bugs, feature requests, and tool documentation issues, please open a GitHub Issue first. The group is for quick discussion and follow-up context

**The QR code is valid until July 21, 2026.** If it expires, please open an Issue and the maintainers will update the invitation.

## License

This project documentation is released under the MIT License. See [LICENSE](LICENSE).

The MIT License applies to the documentation and examples in this repository. It does not mean FTShare data services are available without restriction. Access quota, permissions, and commercial use of FTShare data interfaces are subject to FTShare data service terms.
