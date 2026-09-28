# Legal MCP Server

![tests](https://github.com/samSignal/legal-mcp-server/actions/workflows/tests.yml/badge.svg)
![Python](https://img.shields.io/badge/python-3.11%2B-blue)
![MCP](https://img.shields.io/badge/Model%20Context%20Protocol-server-purple)
![License: MIT](https://img.shields.io/badge/license-MIT-green)

A **Model Context Protocol (MCP) server** that lets AI assistants and agents (Claude Desktop, Claude Code, IDE assistants or your own agent) **search a legal document collection, read articles and cite them**. Retrieval is handled by [legal-hybrid-search](https://github.com/samSignal/legal-hybrid-search) (BM25 + dense + fusion), so the agent gets well-ranked results with ready-made citations instead of guessing.

## Tools, resources and prompts

| Type | Name | Purpose |
|---|---|---|
| Tool | `search_legislation(query, k, act_codes, year_from, year_to, mode)` | Hybrid search with act/year filters; returns article ids, citations and highlighted snippets |
| Tool | `get_article(article_id)` | Full text, citation and previous/next article ids (for reading context) |
| Tool | `list_acts()` | Act codes, names, years and article counts (for choosing filters) |
| Tool | `cite_article(article_id, style)` | `full`: *Residential Tenancy Act (2017), Article 2 (Security deposit)*; `short`: *RTA 2017, art. 2* |
| Resource | `legal://acts/{code}` | A whole act as Markdown |
| Prompt | `answer_with_citations(question)` | Search, then read, then answer with citations, or say nothing relevant was found |

Design choices:

- **Typed, validated inputs and outputs.** Tools declare JSON Schemas (e.g. `k` between 1 and 20) and return structured results (Pydantic models), so clients get machine-readable output and bad arguments are rejected before any work runs.
- **All tools are marked read-only** (`readOnlyHint`, `idempotentHint`), so clients can allow them without asking for confirmation each time.
- **Errors teach the agent how to recover.** An unknown id returns *"No article 'XX-9'. IDs look like 'RT-2' ...; act codes: CA, CR, DP, LC, RT"* instead of a stack trace.
- **Passages are merged per article**, so long documents chunked for retrieval don't fill the results with duplicates.
- **Logs go to stderr**, because stdout carries the protocol over stdio.
- **Two transports:** stdio for desktop clients, and streamable HTTP (`/mcp`) for remote or multi-user deployments.

## Quick start

```bash
git clone https://github.com/samSignal/legal-mcp-server.git
cd legal-mcp-server
pip install -e ".[dev]"

python examples/client.py "my landlord will not return my deposit"   # scripted MCP client
npx @modelcontextprotocol/inspector legal-mcp                         # interactive MCP Inspector
legal-mcp --transport streamable-http --port 8000                     # HTTP endpoint at /mcp
```

Output of the example client (a real run):

```text
Tools: search_legislation, get_article, list_acts, cite_article

Top results for: 'my landlord will not return my deposit'
  RT-2   Residential Tenancy Act (2017), Article 2 (Security deposit)
         The security **deposit** may **not** exceed two months' rent. The **landlord** shall **return** the **deposit** ...
  RT-7   Residential Tenancy Act (2017), Article 7 (Termination by the landlord)
  RT-4   Residential Tenancy Act (2017), Article 4 (Landlord's access to the premises)

Residential Tenancy Act (2017), Article 2 (Security deposit)
The security deposit may not exceed two months' rent. The landlord shall return the deposit within 30 days ...
```

### Use it from Claude Desktop

Add this to `claude_desktop_config.json` (Settings → Developer → Edit Config), then restart Claude Desktop:

```json
{
  "mcpServers": {
    "norland-legal": {
      "command": "legal-mcp"
    }
  }
}
```

If `legal-mcp` is not on your PATH, use the full path to your virtual environment's Python with `"args": ["-m", "legal_mcp.server"]`. Then ask, for example, *"Using norland-legal, how much notice must my landlord give before visiting, and what's the citation?"*

### Use it from Claude Code

```bash
claude mcp add norland-legal -- legal-mcp
```

## Tests

`pytest -q` runs 9 tests, and GitHub Actions runs them on Python 3.11 and 3.12 plus the example client on every push. Most tests use a **real MCP client session** rather than calling the functions directly:

- tool discovery, input/output schemas and read-only annotations
- the full search → get_article → cite workflow
- filters, schema validation (`k=500` is rejected) and helpful error messages
- resources and prompts
- an **end-to-end test that launches the server as a subprocess over stdio**, the way Claude Desktop does

## Project layout

```
legal_mcp/
  library.py   domain layer: search, article lookup, citations, snippets (no MCP code)
  server.py    MCP tools, resource and prompt; stdio / streamable-HTTP entry point
  data/        the document collection (fictional Norland statutes)
examples/client.py
tests/
```

Keeping the domain logic in `library.py`, separate from the protocol layer, means the same search and citation code could also sit behind a REST API or a LangGraph tool.

## Note

The statutes are fictional, written for this project (see legal-hybrid-search). Nothing here is legal advice.

## License

MIT.
