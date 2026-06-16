---
name: xyznot-historical-data
description: Access historical financial data (price bars, news, fundamentals) via MCP + HTTP SQL. Use when user needs bulk historical queries, backtesting, statistical analysis, or multi-year data. Triggers: "historical data", "price history", "backtesting", "all news for", "fundamentals over time", "SQL query on market data".
---

# xyznot Historical Data Skill

Full historical financial data access — 5 years of 1-min OHLCV bars, SEC fundamentals, and news.

## Architecture

| Layer | URL | Purpose |
|-------|-----|---------|
| **MCP** (schema) | `https://mcp.xyznot.com/v1/mcp` | Discover datasets, get table schemas |
| **HTTP SQL** (data) | `https://mcp.xyznot.com/v1/sql` | Run SQL against full datasets |

Auth: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

**Rule**: NEVER use MCP `sql` tool — it times out. Always use the HTTP SQL endpoint for data queries.

## MCP Client Configuration

### Claude Code
```bash
claude mcp add --transport http xyznot-financial \
  https://mcp.xyznot.com/v1/mcp \
  --header "X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e"
```

### Cursor (`~/.cursor/mcp.json`)
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

### Claude Desktop
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

## Step 1: Discover Schema via MCP

```bash
# Initialize → get session-id
curl -sS -D /tmp/hdr -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"client","version":"1.0"}}}'
SID=$(grep -i "mcp-session-id" /tmp/hdr | tr -d '\r' | awk '{print $2}')

# Complete handshake
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}'

# Get schema — use rebyte.public prefix in MCP calls
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"table_schema","arguments":{"tables":["rebyte.public.bars_1m"]}}}'
```

SSE response: extract `data: {"jsonrpc":...}` lines. `result.content[0].text` = markdown schema.

## Step 2: Query Data via HTTP SQL

```bash
curl -s --max-time 60 -X POST https://mcp.xyznot.com/v1/sql \
  -H "Content-Type: text/plain" -H "Accept: application/json" \
  -H "X-API-Key: $KEY" \
  --data "SELECT ticker, t, c, v FROM bars_1m ORDER BY t DESC LIMIT 10"
```

Table names: `bars_1m` or `rebyte.public.bars_1m`.

## Datasets

### bars_1m (~2B rows) — 1-min OHLCV
| Column | Type | Description |
|--------|------|-------------|
| ticker | VARCHAR | Stock symbol |
| t | TIMESTAMP | Bar time (UTC) |
| o | DOUBLE | Open |
| h | DOUBLE | High |
| l | DOUBLE | Low |
| c | DOUBLE | Close |
| v | BIGINT | Volume |
| n | BIGINT | Trade count |
| year | VARCHAR | Partition key |
| month | VARCHAR | Partition key |

### fundamentals (~492K rows) — SEC XBRL
Columns: `tickers`(ARRAY), `company_name`, `fiscal_year`, `fiscal_period`, `filing_date`, `acceptance_datetime`, `end_date`
Financials: `is_*`(~45 income stmt), `bs_*`(~25 balance sheet), `cf_*`(~10 cash flow), `ci_*`(~5 comp income)

### news (~803K rows) — Financial News
| Column | Type | Description |
|--------|------|-------------|
| tickers | ARRAY(VARCHAR) | Mentioned symbols |
| published_utc | TIMESTAMP | Publication time |
| title | VARCHAR | Headline |
| content | VARCHAR | Full text |
| content_embedding | FixedSizeList(1536) | Vector embedding |

## Query Tips

- `year`/`month` are VARCHAR: `WHERE year = '2026'`
- `tickers` is ARRAY: `WHERE ARRAY_CONTAINS(tickers, 'AAPL')`
- Always add `year`/`month` filter for partition pruning
- `ORDER BY t LIMIT N`: instant (TopK)
- `COUNT(*)`: instant (metadata)
- Avoid `WHERE ticker=` without date range (full scan → timeout)

## Limitations

- No column descriptions (Metadata empty in MCP table_schema)
- Column meaning: o=open, is_*=income statement, bs_*=balance sheet, cf_*=cash flow
