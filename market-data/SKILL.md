# Market Data Skills

Two skills for accessing financial market data from the Rebyte runtime.

## Skills

| Skill | Purpose | Triggers |
|-------|---------|----------|
| [xyznot-historical-data](./xyznot-historical-data.md) | Bulk historical queries, backtesting, analysis | "historical data", "price history", "backtesting" |
| [xyznot-data-points](./xyznot-data-points.md) | Specific lookups: latest price, news, fundamentals | "current price", "latest news on", "fundamentals for" |

## Data Available

| Dataset | Rows | Content |
|---------|------|---------|
| `bars_1m` | ~2B | 1-min OHLCV, 5 years |
| `fundamentals` | ~492K | SEC XBRL filings |
| `news` | ~803K | Financial news articles |

## Architecture

| Interface | URL | Use |
|-----------|-----|-----|
| MCP (schema) | `mcp.xyznot.com/v1/mcp` | `list_datasets`, `table_schema` |
| HTTP SQL (data) | `mcp.xyznot.com/v1/sql` | All SQL queries |

**Rule**: Always use HTTP SQL for data. MCP `sql` tool times out.
