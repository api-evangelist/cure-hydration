---
name: agentic-purchase
description: Buy Cure Hydration product on a shopper's behalf over the store's UCP/MCP endpoint, with the buyer-approval and idempotency rules the store itself publishes.
api: cure-hydration-ucp-mcp
generated: '2026-08-11'
method: generated
source: https://www.curehydration.com/agents.md + https://www.curehydration.com/robots.txt + mcp/cure-hydration-tools-list.json
operations:
  - search_catalog
  - create_cart
  - update_cart
  - get_cart
  - create_checkout
  - update_checkout
  - get_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
endpoint: https://www.curehydration.com/api/ucp/mcp
---

# Purchase from CURE over UCP/MCP

## Read this first — the store's own rule

CURE states in both `robots.txt` and `llms.txt`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent flows that
> finalize payment without an explicit, contemporaneous human approval step.

**Do not call `complete_checkout` without a contemporaneous human approval you can point to.**
If you cannot obtain one at the moment of payment, the store directs you to route the purchase
through the Shop skill at `https://shop.app/SKILL.md` instead.

## 0. Discover

```
GET https://www.curehydration.com/.well-known/ucp
```

Confirms `ucp.version` (currently `2026-04-08`, with `2026-01-23` also supported), the
`dev.ucp.shopping` MCP endpoint, the capability set, and the payment handlers
(`com.google.pay`, `dev.shopify.card`, `dev.shopify.shop_pay`).

## 1. Every call carries an agent profile

```json
{"meta": {"ucp-agent": {"profile": "https://your-agent.example/profile"}}}
```

This is an identity gate, not a credential — the server fetches the URI. It must be public
and return 200. Pass buyer context too, or pricing and availability will be wrong:

```json
{"context": {"address_country": "US", "currency": "USD"}}
```

## 2. Find the item

`search_catalog` with the buyer's intent, then `get_product` for the identifier you will add
to the cart. Prices come back as integer minor units — `{"amount": 3900, "currency": "USD"}`
is $39.00. Quote in major units.

## 3. Cart

- `create_cart` — start it.
- `update_cart` — one tool covering what GraphQL splits across seven mutations (lines add/update/remove, attributes, note, discount codes, buyer identity).
- `get_cart` — read it back and show the buyer the real total before going further.
- `cancel_cart` — abandon cleanly rather than leaving it open.

## 4. Checkout

- `create_checkout` — returns line items, totals, discounts and taxes.
- `update_checkout` — set shipping address and delivery method. The UCP fulfillment capability
  on this store declares `allows_multi_destination.shipping: false` and a single
  `["shipping"]` method combination, so do not attempt a split-destination order.
- `get_checkout` — re-read totals after every mutation and show the buyer the final number.

## 5. Complete — the irreversible step

`complete_checkout` **requires** an idempotency key:

```json
{"meta": {
  "ucp-agent": {"profile": "https://your-agent.example/profile"},
  "idempotency-key": "<stable key for this purchase attempt>"
}}
```

Both `ucp-agent` and `idempotency-key` are in the tool's `required` list. Generate the key
once per purchase attempt and reuse it verbatim on any retry — that is the only protection
against double-charging the buyer. Retention is not published, so do not assume a long
dedupe window.

Returns the order id and the Thank You Page URL. Hand the buyer that URL.

## 6. Recover

- `cancel_checkout` — abandon before completion.
- `get_order` — read the resulting order.
- On any `-32001` error, `error.data.continue_url` is a human-completable URL. When you are
  stuck, escalate the buyer to it rather than retrying blindly.
- On `429`, back off. Limits are per IP and unpublished.

## 7. Error shapes

Not RFC 9457. JSON-RPC 2.0 error objects with a vendor slug in `data.code`:

| `data.code` | HTTP | Meaning |
|---|---|---|
| `invalid_profile_url` | 200 | `meta["ucp-agent"].profile` was missing |
| `profile_unreachable` | 422 | The profile URI could not be fetched |

Full catalogue: `errors/cure-hydration-problem-types.yml`.
