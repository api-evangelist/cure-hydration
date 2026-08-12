---
name: browse-catalog
description: Read Cure Hydration's product catalog, collections and store metadata without transacting, using only anonymous endpoints the store publishes for agents.
api: cure-hydration-ucp-mcp
generated: '2026-08-11'
method: generated
source: https://www.curehydration.com/agents.md + mcp/cure-hydration-tools-list.json
operations:
  - search_catalog
  - lookup_catalog
  - get_product
endpoints:
  - GET https://www.curehydration.com/meta.json
  - GET https://www.curehydration.com/collections.json
  - GET https://www.curehydration.com/products.json
  - GET https://www.curehydration.com/collections/{handle}/products.json
  - GET https://www.curehydration.com/products/{handle}.json
  - POST https://www.curehydration.com/api/2026-04/graphql.json
---

# Browse the CURE catalog

Read-only. Nothing here creates a cart, a checkout, or an order.

## 1. Confirm the store and its currency

```
GET https://www.curehydration.com/meta.json
```

Returns the shop profile: `name`, `currency`, `money_format`, `ships_to_countries`,
`published_products_count`, `published_collections_count`. Check `ships_to_countries` before
recommending anything to a buyer outside the US.

## 2. Pick a route

There are three surfaces and they are not identical. Choose deliberately.

| Need | Use | Why |
|---|---|---|
| Simple product list, no auth, no setup | `GET /products.json` | Paged with `limit` and `page`. Prices are decimal strings. |
| Buyer-aware pricing and availability | MCP `search_catalog` | Applies `context.address_country` and `context.currency`. Prices are integer minor units. |
| Selective fields, cursors, content | `POST /api/2026-04/graphql.json` | Full Storefront schema; introspectable anonymously. |

## 3. Read-only JSON

```
GET https://www.curehydration.com/collections.json
GET https://www.curehydration.com/collections/{handle}/products.json
GET https://www.curehydration.com/products/{handle}.json
```

Collection handles on this store include `shop-all`, `hydration`, `energy`, `kids`,
`variety-packs`, `bundle-save`, `fsa-hsa-eligible`, `best-seller`.

Honor the `etag` on these responses and send `If-None-Match` on re-reads.

## 4. MCP catalog tools

```
POST https://www.curehydration.com/api/ucp/mcp
Content-Type: application/json
Accept: application/json, text/event-stream

{"jsonrpc":"2.0","id":1,"method":"tools/list"}
```

`tools/list` and `initialize` are anonymous. Every **tool call** must carry a resolvable
agent profile URI:

```json
{"meta": {"ucp-agent": {"profile": "https://your-agent.example/profile"}}}
```

Omit it and you get JSON-RPC `-32001` / `invalid_profile_url`. Supply one the server cannot
fetch and you get `-32001` / `profile_unreachable` over HTTP 422. See
`errors/cure-hydration-problem-types.yml`.

- `search_catalog` — find products by buyer intent.
- `lookup_catalog` — batch lookup by identifier.
- `get_product` — full detail for one product.

## 5. Money

MCP responses use **integer minor units with a currency code**: `{"amount": 600, "currency": "USD"}`
is $6.00. Divide by 100 for two-decimal currencies before quoting a price. `/products.json`
does **not** follow this rule — it returns `"price": "39.00"`. Normalize before comparing.

## 6. Rate limits

The MCP endpoint is rate-limited per IP with no published number. Back off on `429`. The
`shopify-complexity-score` response header and the GraphQL `extensions.cost.requestedQueryCost`
field tell you what the last request cost; neither tells you what remains.
See `rate-limits/cure-hydration-rate-limits.yml`.

## 7. What you must not do

`robots.txt` disallows `/cart.js` and `/recommendations/products` and steers agents to
UCP/MCP for anything cart-shaped. Stay read-only in this skill; use `agentic-purchase` to
transact.
