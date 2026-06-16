---
name: xyznot-data-points
description: Look up specific financial data points — latest price, recent news, company fundamentals for a single ticker. Use when user asks for "current price of X", "latest news on X", "what is X's market cap", "how did X do today". For bulk/historical analysis use xyznot-historical-data instead.
---

# xyznot Data Points Skill

Targeted lookups — one ticker, one result.

## Architecture

| Layer | URL | Purpose |
|-------|-----|---------|
| **HTTP SQL** | `https://mcp.xyznot.com/v1/sql` | Point queries for specific data |

Auth: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

## Endpoints

All via `POST https://mcp.xyznot.com/v1/sql` with raw SQL body.

### Latest Price

```bash
curl -s --max-time 10 -X POST https://mcp.xyznot.com/v1/sql \
  -H "Content-Type: text/plain" -H "Accept: application/json" \
  -H "X-API-Key: $KEY" \
  --data "SELECT ticker, t, o, h, l, c, v FROM bars_1m WHERE ticker = 'AAPL' AND year = '2026' AND month = '6' ORDER BY t DESC LIMIT 1"
```

### Latest 10 Prices (for mini-chart)

```bash
--data "SELECT t, o, h, l, c, v FROM bars_1m WHERE ticker = 'AAPL' AND year = '2026' AND month = '6' ORDER BY t DESC LIMIT 10"
```

### Latest News

```bash
--data "SELECT title, description, published_utc, source, keywords FROM news WHERE ARRAY_CONTAINS(tickers, 'TSLA') ORDER BY published_utc DESC LIMIT 5"
```

### Company Fundamentals (latest filing)

```bash
--data "SELECT company_name, fiscal_year, fiscal_period, end_date, acceptance_datetime, is_revenues, is_net_income_loss, is_basic_earnings_per_share FROM fundamentals WHERE ARRAY_CONTAINS(tickers, 'AAPL') ORDER BY acceptance_datetime DESC LIMIT 1"
```

### Today's OHLC (daily bar from 1-min)

```bash
--data "SELECT MIN(t) as open_time, MAX(t) as close_time, FIRST_VALUE(o) OVER w as open, MAX(h) as high, MIN(l) as low, LAST_VALUE(c) OVER w as close, SUM(v) as volume FROM bars_1m WHERE ticker = 'AAPL' AND t >= '2026-06-16' AND t < '2026-06-17' GROUP BY DATE_TRUNC('day', t) WINDOW w AS (PARTITION BY DATE_TRUNC('day', t) ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)"
```

## Performance

All point queries return in < 2 seconds when year/month filter is used.
