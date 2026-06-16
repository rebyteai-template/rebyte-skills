# xyznot Market Data: Historical / Bulk

**MCP Server**: `https://mcp.xyznot.com/v1/mcp`
**HTTP SQL**: `https://api.rebyte.ai/api/data`

Purpose: Pull **all** historical financial data — full price history, all news articles, complete SEC fundamentals.

---

## Interface

| Layer | URL | Role |
|-------|-----|------|
| **MCP** (schema) | `POST https://mcp.xyznot.com/v1/mcp` | Discover schema: `list_datasets`, `table_schema` |
| **HTTP SQL** (data) | `POST https://api.rebyte.ai/api/data` | Run SQL against full datasets (20B+ rows) |

Auth: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

**Rule**: NEVER use MCP `sql` tool — times out. Always use HTTP SQL for data queries.

---

## MCP Client Configs

### Claude Code
```bash
claude mcp add --transport http xyznot-financial https://mcp.xyznot.com/v1/mcp \
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

---

## MCP Handshake (Streamable HTTP SSE)

```bash
# 1. Initialize → get session-id from response headers
curl -sS -D /tmp/hdr -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"client","version":"1.0"}}}'
SID=$(grep -i "mcp-session-id" /tmp/hdr | tr -d '\r' | awk '{print $2}')

# 2. Complete handshake
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}'

# 3. Discover schema
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"table_schema","arguments":{"tables":["rebyte.public.bars_1m"]}}}'
```

Response is SSE. Extract `data: {"jsonrpc":...}` lines. `result.content[0].text` is a markdown schema table.

---

## HTTP SQL

```bash
curl -s --max-time 60 -X POST https://api.rebyte.ai/api/data \
  -H "Content-Type: text/plain" -H "Accept: application/json" \
  -H "X-API-Key: $KEY" \
  --data "SELECT ticker, t, c, v FROM bars_1m ORDER BY t DESC LIMIT 10"
```

Table names: bare (`bars_1m`) or `rebyte.public.bars_1m`.

---

## Datasets

### bars_1m (~20B rows) — 1-min OHLCV Price History
| Column | Type | Notes |
|--------|------|-------|
| ticker | VARCHAR | Stock symbol |
| t | TIMESTAMP | Bar time (UTC) |
| o/h/l/c | DOUBLE | Open/High/Low/Close |
| v | BIGINT | Volume |
| n | BIGINT | Trade count |
| year | VARCHAR | Partition key (e.g. '2026') |
| month | VARCHAR | Partition key (e.g. '6') |

### fundamentals (~22K rows) — SEC XBRL Filings
Columns: `tickers`(ARRAY), `company_name`, `fiscal_year`, `fiscal_period`, `filing_date`, `acceptance_datetime`, `end_date`
Financials: `is_*`(~45 income stmt), `bs_*`(~25 balance sheet), `cf_*`(~10 cash flow), `ci_*`(~5 comp income)

### news (~800K rows) — Financial News
| Column | Type | Notes |
|--------|------|-------|
| tickers | ARRAY(VARCHAR) | Mentioned symbols |
| published_utc | TIMESTAMP | Publication time |
| title | VARCHAR | Headline |
| content | VARCHAR | Full text |
| content_embedding | FixedSizeList(1536) | Vector |

---

## Query Tips

- `year`/`month` are VARCHAR: `WHERE year = '2026'`
- `tickers` is ARRAY: `WHERE ARRAY_CONTAINS(tickers, 'AAPL')`
- Always add `year`/`month` for partition pruning on bars_1m
- `ORDER BY t LIMIT N`: instant (TopK)
- `COUNT(*)`: instant (metadata)
- Avoid `WHERE ticker=` without date range (20B full scan → timeout)

---

## Limitations

- No column descriptions (Metadata empty in MCP table_schema)
- Column meaning inferred from name (o=open, is_*=income stmt, bs_*=balance sheet)
