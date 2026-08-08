---
name: Shop the Bartesian catalog and complete a checkout
description: Search the Bartesian storefront catalog, build a cart, open a checkout,
  set shipping, and complete the purchase with explicit buyer approval, over the store's
  UCP Shopping MCP endpoint.
api: mcp/bartesian-ucp-shopping-tools.json
endpoint: https://bartesian.com/api/ucp/mcp
operations:
- search_catalog
- get_product
- create_cart
- update_cart
- create_checkout
- update_checkout
- complete_checkout
- get_order
generated: '2026-08-06'
method: generated
source: https://bartesian.com/agents.md
---

# Shop Bartesian and complete a checkout

Bartesian sells capsule-based cocktail machines and cocktail capsules from a Shopify
storefront that implements the Universal Commerce Protocol. Every tool name and every
parameter below is taken verbatim from the live `tools/list` response the store returns
at `https://bartesian.com/api/ucp/mcp`. Bartesian publishes no REST API and issues no
API keys.

## Before you call anything

1. `GET https://bartesian.com/.well-known/ucp` and confirm the version you intend to use
   is in `supported_versions` (`2026-04-08` is current; `2026-01-23` is still served).
2. Every call is a JSON-RPC 2.0 POST to `https://bartesian.com/api/ucp/mcp` with
   `Content-Type: application/json` and `Accept: application/json, text/event-stream`.
   The endpoint is POST-only — a GET returns 404.
3. You can read the contract anonymously. `initialize` and `tools/list` both answer 200
   with no credentials, so fetch the current tool schemas rather than trusting a cached
   copy.
4. **You cannot call a tool anonymously.** Every one of the 13 tools requires
   `meta.ucp-agent.profile` — the URI of *your* platform's UCP profile document (HTTP
   header form: `UCP-Agent`). Calling without it returns HTTP 422 and JSON-RPC error
   `-32001` `UCP discovery failed` / `invalid_profile_url`, and nothing else works.
5. Pass buyer context (`context.address_country` ISO 3166-1 alpha-2,
   `context.currency` ISO 4217, `context.language` BCP 47) so prices and availability
   are correct. Bartesian serves market-scoped storefronts for US, en-CA, fr-CA, en-GB
   and en-EU.

## Steps

1. **Find products** — call `search_catalog` with the buyer's intent as `catalog.query`
   plus optional `catalog.filters` (`categories`, `price.min`/`price.max` in **minor
   currency units**, `available`) and `catalog.pagination` (`cursor`, `limit`, default
   10, minimum 1). For a specific item call `get_product`; for a batch of known
   identifiers call `lookup_catalog` — `catalog.ids` is capped at **10 per request**.
2. **Build the cart** — call `create_cart` with `cart.line_items`, then `update_cart` to
   change quantities or swap variants. `get_cart` re-reads current state. Cart IDs are
   Shopify GIDs: `gid://shopify/Cart/abc123`.
3. **Open the checkout** — call `create_checkout`. You may pass `checkout.cart_id` to
   carry the cart over. Read `messages[]` on the result; a message with severity
   `requires_buyer_input` means the merchant needs something the API cannot collect
   programmatically and the checkout is not yet completable.
4. **Set fulfillment** — call `update_checkout` with the shipping address and the chosen
   shipping method. This store ships to a **single destination only**
   (`allows_multi_destination.shipping` is false), so do not attempt to split a shipment.
5. **Get buyer approval, then complete** — call `complete_checkout` **only** after the
   buyer has explicitly approved the payment. This is a hard rule in Bartesian's own
   `agents.md`: agents must not complete payment without contemporaneous buyer consent.
   If you cannot obtain it in the moment, stop and route the purchase through Shop Pay
   using the Shopify Shop skill (`https://shop.app/SKILL.md`) instead.
6. **Confirm** — call `get_order` for the current-state snapshot of the placed order
   (`gid://shopify/Order/123`).

## Rules that will bite you

- **`complete_checkout` requires an idempotency key and it is the only tool that takes
  one.** `meta.idempotency-key` is in the `required` list of `complete_checkout` and is
  not declared on any other tool on this store — not on `cancel_checkout`, not on
  `cancel_cart`. Generate one key per logical purchase attempt and **reuse the same key
  on every retry** of that attempt. Do not try to send one elsewhere; the schema does not
  accept it, and cancels are therefore not retry-safe by key. Treat a failed cancel as
  needing a `get_cart` / `get_checkout` read-back rather than a blind retry.
- **Back off on 429.** The MCP endpoint is rate limited per IP. Retry with exponential
  backoff; do not parallelize checkout traffic.
- **Errors are not RFC 9457.** Business failures come back as
  `{ucp: {status: error}, messages: [...], continue_url}`. Branch on `messages[].severity`:
  `recoverable` (fix inputs and retry), `requires_buyer_input` / `requires_buyer_review`
  (hand off to the buyer, use `continue_url`), `unrecoverable` (start a new resource).
  `messages[].path` is an RFC 9535 JSONPath to the exact offending component.
- **This is an alcohol-adjacent catalog.** Bartesian sells cocktail machines and capsules.
  Expect `eligibility_invalid` and `requires_buyer_review` on age- or
  jurisdiction-restricted line items and destinations. Bartesian does not document an
  age-verification step, so do not assume one has happened — surface these messages to
  the buyer rather than retrying around them.
- **You never touch card data.** Payment handlers on this store are Shop Pay, Shopify
  card (visa, master, american_express, discover, diners_club) and Google Pay. Supply
  `checkout.payment.instruments[]` with a `handler_id` and a token; raw PANs are not part
  of the flow.

## If you only need to read

No authentication and no UCP profile is needed for browsing:
`GET /collections/all`, `GET /products/{handle}.json`,
`GET /collections/{handle}/products.json`, `GET /search?q={query}&type=product`,
`GET /products.json`, `GET /sitemap.xml` — all on `https://bartesian.com`.
See `bartesian-catalog-lookup.md`.
