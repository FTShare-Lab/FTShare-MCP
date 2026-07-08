# FTShare MCP

[中文](README.md) | [English](README_EN.md)

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

`FTShare-MCP` is the MCP tool documentation and integration guide for FTShare financial data capabilities. It is designed for MCP-compatible clients, Agent applications, and automated investment research workflows. This repository provides the public MCP endpoint, calling workflow, tool index, parameter descriptions, and examples.

For international developers, this repository can be understood as **FTShare MCP tools for financial data, market data, quantitative research, and AI Agent investment workflows**.

> This repository focuses on **MCP tool documentation and integration instructions**. It does not contain the MCP Server source code. The public MCP service is provided by the FTShare data service.

The current documentation focuses on **150 officially documented `ft_*` financial data tools**, covering market data, financial statements, macro data, funds, futures, bonds, US stocks, Hong Kong stocks, and related datasets. Testing, demo, or internal helper tools are not listed as official documentation entries.

## Public MCP Endpoint

This documentation uses `<MCP_BASE_URL>` as the placeholder for the MCP Streamable HTTP endpoint. The public MCP endpoint is:

```bash
MCP_BASE_URL="https://market.ft.tech/gateway/mcp"
```

When copying examples from tool documents, replace `<MCP_BASE_URL>` with the URL above.

- **MCP endpoint placeholder**: `<MCP_BASE_URL>` (Streamable HTTP)
- **Protocol flow**: call `initialize` first to get `Mcp-Session-Id`, then call `tools/call`
- **Return format**: markdown table text, not structured JSON
- **Live schema**: `tools/list` returns the `inputSchema` for each tool

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
5. Call `tools/call` with `Mcp-Session-Id`, where `name` is the tool name and `arguments` is the business parameter object.
6. Read the result. `ft_*` tools return markdown table text by default.
7. Handle errors on the client side, including non-2xx HTTP responses, MCP protocol errors, parameter errors, and server-side business errors.

## Generic Demo

The following examples can call any `ft_*` tool. Change only `TOOL_NAME` and `TOOL_ARGS`.

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

curl -sS -m 30 -X POST "$MCP_BASE_URL" \
  -H "Accept: application/json, text/event-stream" \
  -H "Content-Type: application/json" \
  -H "Mcp-Session-Id: $MCP_SESSION_ID" \
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
            print(result.content[0].text)

asyncio.run(main())
```

## Tool Categories

| Category | Tool Count | Documentation |
|----------|------------|---------------|
| ETF | 8 | [ETF专题/](./ETF专题/) |
| Bonds | 2 | [债券专题/](./债券专题/) |
| Public Funds | 5 | [公募基金/](./公募基金/) |
| FX Data | 1 | [外汇数据/](./外汇数据/) |
| LLM Corpus | 5 | [大模型语料/](./大模型语料/) |
| Macro Economy | 17 | [宏观经济/](./宏观经济/) |
| Index Data | 8 | [指数专题/](./指数专题/) |
| Futures Data | 7 | [期货数据/](./期货数据/) |
| Hong Kong Stocks | 7 | [港股数据/](./港股数据/) |
| Spot Data | 2 | [现货数据/](./现货数据/) |
| US Stocks | 7 | [美股数据/](./美股数据/) |
| A-share Stocks | 81 | [股票数据/](./股票数据/) |

For the full tool index and individual tool documentation, see the Chinese [README.md](README.md) and the category documents. The tool names, parameter names, and examples are language-independent.

## Notes

- The live schema returned by `tools/list` is the source of truth for runtime parameters.
- Tool documents are written for the officially documented `ft_*` financial data tools.
- Most tools return markdown table text, not structured JSON.
- Some tools have time-window, pagination, quota, or permission constraints.
- Access quota, permissions, and commercial use of FTShare data interfaces are subject to FTShare data service terms.

## Related Projects

- [FTShare-python-sdk](https://github.com/FTShare-Lab/FTShare-python-sdk): FTShare financial data Python SDK for developer-facing data access
- [FTShare-skills](https://github.com/FTShare-Lab/FTShare-skills): FTShare Agent Skills repository for data-level Skills and investment research workflow Skills

## Community

Chinese users are welcome to join the FTShare WeChat community group to discuss MCP integration, tool calls, financial data interfaces, Agent research workflows, and contribution directions.

<img src="docs/assets/wechat-group-20260715.png" alt="FTShare WeChat community group" width="320" />

> **Community rules**:
> - Discussions should be related to FTShare, MCP integration, financial data interfaces, or Agent research workflows
> - Advertising, promotion, and unrelated off-topic chat are not allowed
> - For bugs, feature requests, and tool documentation issues, please open a GitHub Issue first. The group is for quick discussion and follow-up context

**The QR code is valid until July 15, 2026.** If it expires, please open an Issue and the maintainers will update the invitation.

## License

This project documentation is released under the MIT License. See [LICENSE](LICENSE).

The MIT License applies to the documentation and examples in this repository. It does not mean FTShare data services are available without restriction. Access quota, permissions, and commercial use of FTShare data interfaces are subject to FTShare data service terms.
