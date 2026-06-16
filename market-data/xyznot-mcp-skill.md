# xyznot Financial Data Skill

**MCP Server**: `https://mcp.xyznot.com/v1/mcp`

This skill provides two separate interfaces for financial data access.

---

## Use Case 1: Historical / Bulk Data

**Purpose**: Pull **all** historical data — full price history, all news articles, complete fundamentals.

**MCP Server URL**: `https://mcp.xyznot.com/v1/mcp` (Streamable HTTP SSE)

| Interface | URL | Role |
|-----------|-----|------|
| **MCP** | `POST https://mcp.xyznot.com/v1/mcp` | Discover schema: `list_datasets`, `table_schema` |
| **HTTP SQL** | `POST https://mcp.xyznot.com/v1/sql` | Run SQL against full datasets (20B+ rows) |

**Auth**: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

### MCP Client Configuration

**Claude Code**:
```bash
claude mcp add --transport http xyznot-financial \
  https://mcp.xyznot.com/v1/mcp \
  --header "X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e"
```

**Cursor** (`.cursor/mcp.json`):
```json
{
  "mcpServers": {
    "xyznot-financial": {
      "url": "https://mcp.xyznot.com/v1/mcp",
      "headers": { "X-API-Key": "a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e" }
    }
  }
}
```

**Claude Desktop** (`claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "xyznot-financial": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://mcp.xyznot.com/v1/mcp", "--header", "X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e"]
    }
  }
}
```

### Datasets

| Dataset | Rows | Content |
|---------|------|---------|
| `rebyte.public.bars_1m` | ~20B | 1-min OHLCV: ticker, t, o, h, l, c, v, n, year, month |
| `rebyte.public.fundamentals` | ~22K | SEC XBRL filings: tickers(ARRAY), company_name, fiscal_*, filing_date, acceptance_datetime, end_date, is_*(~45), bs_*(~25), cf_*(~10), ci_*(~5) |
| `rebyte.public.news` | ~800K | Articles: tickers(ARRAY), title, content, published_utc, content_embedding(FixedSizeList(1536)) |

### Workflow

**Step 1: Discover schema via MCP**

```bash
# Initialize
curl -sS -D /tmp/hdr -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" \
  -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"client","version":"1.0"}}}'
SID=$(grep -i "mcp-session-id" /tmp/hdr | tr -d '\r' | awk '{print $2}')

# Complete handshake
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}'

# Get schema (always use rebyte.public prefix in MCP calls)
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"table_schema","arguments":{"tables":["rebyte.public.bars_1m"]}}}'
```

Response is SSE. Extract `data: {"jsonrpc": ...}` lines → `result.content[0].text` is a markdown schema table.

**Step 2: Query data via HTTP SQL**

```bash
curl -s --max-time 60 -X POST https://mcp.xyznot.com/v1/sql \
  -H "Content-Type: text/plain" \
  -H "Accept: application/json" \
  -H "X-API-Key: $KEY" \
  --data "SELECT ticker, t, c, v FROM bars_1m ORDER BY t DESC LIMIT 10"
```

Table names work bare (`bars_1m`) or fully-qualified (`rebyte.public.bars_1m`).

### Query Tips

- `year`/`month` are VARCHAR: `WHERE year = '2026'`
- `tickers` is ARRAY: `WHERE ARRAY_CONTAINS(tickers, 'AAPL')`
- Always add `year`/`month` filter for partition pruning on bars_1m
- `ORDER BY t LIMIT N`: instant (TopK optimization)
- `COUNT(*)`: instant (metadata)
- Avoid `WHERE ticker=` without date range (full scan of 20B rows — will timeout)
- **NEVER use MCP `sql` tool** — always times out. Use HTTP SQL for all queries.

### Limitations

- No column descriptions (Metadata column empty in MCP table_schema)
- Column meaning must be inferred from name (o=open, is_*=income statement, bs_*=balance sheet, etc.)

---

## Use Case 2: Specific Data Points

**Purpose**: Targeted lookups — get the latest price of one ticker, fetch one company's fundamentals, retrieve a single news article.
**Endpoint**: *(API endpoint TBD — to be added)*

*(Details will be filled in once the API is defined. This will be a simple REST endpoint for point queries, separate from the bulk SQL interface above.)*
