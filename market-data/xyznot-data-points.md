# xyznot Market Data: Specific Data Points

**API**: `https://api.rebyte.ai/api/data`

Purpose: Targeted lookups — latest price of one ticker, one company's SEC filing, single news article.

Auth: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

---

## Endpoints (TBD)

### Latest Price
```
POST /api/data
```
SQL: `SELECT ticker, t, o, h, l, c, v FROM bars_1m WHERE ticker = $1 ORDER BY t DESC LIMIT 1`

### Company Fundamentals
```
POST /api/data
```
SQL: `SELECT * FROM fundamentals WHERE ARRAY_CONTAINS(tickers, $1) ORDER BY acceptance_datetime DESC LIMIT 1`

### News Article
```
POST /api/data
```
SQL: `SELECT * FROM news WHERE id = $1`

---

*(Exact endpoints and response formats to be defined.)*
