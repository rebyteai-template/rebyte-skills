# xyznot Financial Data Skill

## Overview

Two-interface access to financial market data via a Rebyte Open Source runtime at `mcp.xyznot.com`.

| Interface | URL | Use |
|-----------|-----|-----|
| **MCP** | `POST /v1/mcp` | Schema discovery only |
| **HTTP SQL** | `POST /v1/sql` | ALL data queries |

**Rule**: NEVER use MCP `sql` tool for data — it times out. Always use HTTP SQL.

## Auth

All requests: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

## MCP (Streamable HTTP SSE)

```
# 1. Initialize → get session-id
curl -sS -D /tmp/hdr -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e" \
  -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"client","version":"1.0"}}}'
# Extract SID from response header: mcp-session-id

# 2. Send initialized
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}'

# 3. Call tools
# list_datasets
curl ... -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"list_datasets","arguments":{}}}'

# table_schema (catalog is rebyte.public)
curl ... -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"table_schema","arguments":{"tables":["rebyte.public.bars_1m"]}}}'

# get_readiness
curl ... -d '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"get_readiness","arguments":{}}}'
```

Response is SSE — look for `data: {"jsonrpc":...}` lines.

## HTTP SQL

```
curl -s --max-time 60 -X POST https://mcp.xyznot.com/v1/sql \
  -H "Content-Type: text/plain" \
  -H "Accept: application/json" \
  -H "X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e" \
  --data "SELECT ticker, t, c, v FROM bars_1m ORDER BY t DESC LIMIT 10"
```

Returns JSON array. Table names work bare (`bars_1m`) or fully-qualified (`rebyte.public.bars_1m`).

## Datasets

### bars_1m (~20B rows) — 1-min OHLCV
Columns: ticker(VARCHAR), t(TIMESTAMP), o/h/l/c(DOUBLE), v(BIGINT), n(BIGINT), year(VARCHAR), month(VARCHAR)

### fundamentals (~22K rows) — SEC XBRL filings
Key: tickers(ARRAY), company_name, fiscal_year, fiscal_period, filing_date, acceptance_datetime, end_date
Metrics: is_*(~45), bs_*(~25), cf_*(~10), ci_*(~5)

### news (~800K rows) — Financial news
Key: tickers(ARRAY), title, published_utc, content, content_embedding(FixedSizeList(1536))

## Query Tips

- year/month are VARCHAR: `WHERE year = '2026'`
- tickers is ARRAY: `WHERE ARRAY_CONTAINS(tickers, 'AAPL')`
- Partition pruning: always add `year`/`month` filter for bars_1m when filtering by ticker
- `ORDER BY t LIMIT N`: instant (TopK, no full scan)
- `COUNT(*)`: instant (metadata)
- `WHERE ticker=` without date range: will timeout (20B full scan)

## Limitations

- No column descriptions (Metadata column is empty in table_schema)
- MCP `sql` tool always times out — use HTTP SQL
