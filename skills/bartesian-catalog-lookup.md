---
name: Read the Bartesian catalog without transacting
description: Search, batch-look-up and read Bartesian products and their real-time
  availability, either over the store's UCP Shopping MCP endpoint or over the
  unauthenticated Shopify storefront JSON endpoints Bartesian documents itself.
api: mcp/bartesian-ucp-shopping-tools.json
endpoint: https://bartesian.com/api/ucp/mcp
operations:
- search_catalog
- lookup_catalog
- get_product
generated: '2026-08-06'
method: generated
source: https://bartesian.com/agents.md
---

# Read the Bartesian catalog

Two routes, with different requirements. Pick the cheaper one that answers the question.

## Route A — no credentials at all (storefront JSON)

Bartesian documents these itself in `/agents.md`. No auth, no UCP profile, no MCP.

| Call | Returns |
|---|---|
| `GET https://bartesian.com/products.json` | every published product |
| `GET https://bartesian.com/collections/{handle}/products.json` | one collection's products |
| `GET https://bartesian.com/products/{handle}.json` | one product |
| `GET https://bartesian.com/collections/all` | browse page |
| `GET https://bartesian.com/search?q={query}&type=product` | storefront search |
| `GET https://bartesian.com/sitemap.xml` | sitemap index (US, en-CA, fr-CA, en-GB, en-EU) |

Each product object carries `id, title, handle, body_html, published_at, created_at,
updated_at, vendor, product_type, tags, variants, images, options`.

**Limitation:** this is the published catalog, not a pricing or availability oracle. It
does not take buyer context, so it will not reflect market-specific pricing or
per-destination availability. For that, use Route B.

## Route B — UCP Shopping MCP (needs a UCP agent profile)

Endpoint: `POST https://bartesian.com/api/ucp/mcp`,
`Content-Type: application/json`, `Accept: application/json, text/event-stream`.

You may call `initialize` and `tools/list` anonymously — both return 200 and
`tools/list` returns all 13 tool schemas. **Tool invocation is gated:** every tool
requires `meta.ucp-agent.profile`, the URI of your platform's own UCP profile document.
Without it you get HTTP 422 / JSON-RPC `-32001` `invalid_profile_url`.

1. **`search_catalog`** — natural-language `catalog.query` plus:
   - `catalog.filters.categories[]` (OR logic)
   - `catalog.filters.price.min` / `.max` — **integers in minor currency units**
   - `catalog.filters.available` — boolean, defaults `true` (sale-ready items only)
   - `catalog.pagination.cursor` and `catalog.pagination.limit` (default 10, minimum 1)
   - `catalog.context` — `address_country` (ISO 3166-1 alpha-2), `address_region`,
     `postal_code`, `language` (BCP 47), `currency` (ISO 4217), `intent`
   - `catalog.signals` — `dev.ucp.buyer_ip`, `dev.ucp.user_agent`
2. **`lookup_catalog`** — resolve known identifiers in one round trip.
   `catalog.ids` is **required**, `minItems: 1`, `maxItems: 10`. Chunk anything larger.
3. **`get_product`** — one product in full: variants, exact pricing, real-time
   availability. Use this, not the storefront JSON, when the answer has to be current.

## Notes

- Paginate with the returned cursor; do not guess offsets.
- Always pass `context.address_country` and `context.currency`. Bartesian runs
  market-scoped storefronts, so an unqualified price is the wrong price for most buyers.
- None of these three tools accept `meta.idempotency-key` — they are reads, and on this
  store only `complete_checkout` declares an idempotency key.
- Stop here if you are only answering a question. Cart and checkout live in
  `bartesian-shop-and-checkout.md`, and completing a checkout requires contemporaneous
  buyer approval.
