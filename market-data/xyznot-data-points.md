# xyznot Market Data: Specific Data Points

**API**: *(TBD)*

Purpose: Targeted lookups — latest price of one ticker, one company's SEC filing, single news article.

---

## Interface

| Endpoint | Role |
|----------|------|
| `GET /v1/...` (TBD) | Lookup specific data points |

Auth: `X-API-Key: a5ff6f5f752c1f04948ba3ce27119ad4202c41f1d52120698ec045b8d25f206e`

---

## Endpoints (TBD)

### Latest Price
```
GET /v1/price/{ticker}
```
Returns: `{ticker, t, o, h, l, c, v}` for the most recent 1-min bar.

### Company Fundamentals
```
GET /v1/fundamentals/{ticker}
```
Returns: latest fiscal period data (income stmt, balance sheet, cash flow).

### News Article
```
GET /v1/news/{id}
```
Returns: single article with title, content, published_utc.

---

*(Exact endpoints and response formats to be defined.)*
