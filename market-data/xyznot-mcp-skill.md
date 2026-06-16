# xyznot Financial Data Skill

## Two Use Cases

### Use Case 1: Historical Data (MCP + HTTP SQL)
For bulk/pull of **all** historical data — price history, news, fundamentals.

| Interface | URL | Purpose |
|-----------|-----|---------|
| **MCP** | `POST /v1/mcp` | Schema discovery: `list_datasets`, `table_schema` |
| **HTTP SQL** | `POST /v1/sql` | Run SQL queries against full datasets |

**Rule**: NEVER use MCP `sql` tool — it times out. Always use HTTP SQL for data.

### Use Case 2: Specific Data Points (API)
For targeted lookups — a single ticker's latest price, one company's fundamentals, one article.

*(API endpoint details TBD — to be added once defined)*

---

## Auth

All requests: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

---

## Use Case 1: Historical Data via MCP + HTTP SQL

### MCP Handshake (Streamable HTTP SSE)

```bash
# 1. Initialize
curl -sS -D /tmp/hdr -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" \
  -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"client","version":"1.0"}}}'
SID=$(grep -i "mcp-session-id" /tmp/hdr | tr -d '\r' | awk '{print $2}')

# 2. Send initialized
curl -sS -X POST https://mcp.xyznot.com/v1/mcp \
  -H "X-API-Key: $KEY" -H "content-type: application/json" \
  -H "accept: application/json, text/event-stream" \
  -H "mcp-session-id: $SID" \
  -d '{"jsonrpc":"2.0","method":"notifications/initialized","params":{}}'

# 3. Discover schema
curl ... -d '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{"name":"table_schema","arguments":{"tables":["rebyte.public.bars_1m"]}}}'
```

Response is SSE. Extract `data: {"jsonrpc":...}` lines. Get `result.content[0].text` for markdown schema.

### HTTP SQL Queries

```bash
curl -s --max-time 60 -X POST https://mcp.xyznot.com/v1/sql \
  -H "Content-Type: text/plain" \
  -H "Accept: application/json" \
  -H "X-API-Key: $KEY" \
  --data "SELECT ticker, t, c, v FROM bars_1m ORDER BY t DESC LIMIT 10"
```

Table names: bare (`bars_1m`) or fully-qualified (`rebyte.public.bars_1m`).

---

## Datasets

### bars_1m (~20B rows) — 1-min OHLCV Price History
| Column | Type | Description |
|--------|------|-------------|
| ticker | VARCHAR | Stock symbol |
| t | TIMESTAMP | Bar time (UTC) |
| o/h/l/c | DOUBLE | Open/High/Low/Close |
| v | BIGINT | Volume |
| n | BIGINT | Number of trades |
| year | VARCHAR | Partition (e.g. '2026') |
| month | VARCHAR | Partition (e.g. '6') |

### fundamentals (~22K rows) — SEC Filings
Columns: `tickers` (ARRAY), `company_name`, `fiscal_year`, `fiscal_period`, `filing_date`, `acceptance_datetime`, `end_date`
Financials: `is_*` (~45 income stmt), `bs_*` (~25 balance sheet), `cf_*` (~10 cash flow), `ci_*` (~5 comp income)

### news (~800K rows) — Financial News
| Column | Type | Description |
|--------|------|-------------|
| tickers | ARRAY(VARCHAR) | Mentioned symbols |
| published_utc | TIMESTAMP | Publication time |
| title | VARCHAR | Headline |
| content | VARCHAR | Full text |
| content_embedding | FixedSizeList(1536) | Vector embedding |

---

## Query Tips

- `year`/`month` are VARCHAR: `WHERE year = '2026'`
- `tickers` is ARRAY: `WHERE ARRAY_CONTAINS(tickers, 'AAPL')`
- Use `year`/`month` filter for partition pruning
- `ORDER BY t LIMIT N` is instant (TopK)
- `COUNT(*)` is instant (metadata)
- Avoid `WHERE ticker=` without date range — full scan of 20B rows

---

## Typical Workflow

1. **Discover schema**: MCP `table_schema` on `rebyte.public.{table}`
2. **Design SQL**: Map question to query using known columns
3. **Pull data**: HTTP SQL → JSON file
4. **Analyze locally**: Python with `json.load()` + statistics

---

## Limitations

- No column descriptions (Metadata empty in table_schema)
- MCP `sql` tool always times out — use HTTP SQL for all data queries
