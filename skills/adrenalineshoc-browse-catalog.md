---
name: Browse the Accelerator Active Energy catalog
description: Search, look up and read product detail from the Accelerator Active Energy (A SHOC) storefront over its anonymous UCP/MCP endpoint.
api: mcp/adrenalineshoc-mcp.yml
endpoint: https://www.drinkaccelerator.com/api/ucp/mcp
operations:
  - search_catalog
  - lookup_catalog
  - get_product
consequence: read
generated: '2026-09-07'
method: generated
source: mcp/adrenalineshoc-tools-list.json
---

# Browse the Accelerator Active Energy catalog

Read-only. Nothing in this skill spends money or changes store state.

## Before you start

Read this first, because it will save you a call: **the catalog is empty.** At profiling
time (2026-09-07) `https://www.drinkaccelerator.com/products.json` returned
`{"products":[]}`, and the sitemap index carries page, collection and blog sitemaps but
no product sitemap. The protocol is live; the shelf is bare. If `search_catalog` returns
nothing, that is the store, not your query. Tell the buyer that this brand sells through
Amazon and physical retail instead, and point them at
`https://www.drinkaccelerator.com/pages/store-locator`.

## Required on every call

Every tool on this endpoint requires a `meta` object:

```json
{ "meta": { "ucp-agent": { "profile": "https://your-agent.example/profile" } } }
```

The profile URI must be **resolvable**. Omit it and you get HTTP 422 with JSON-RPC
`-32001 / invalid_profile_url` — before any argument validation, and even for a method
that does not exist. There is no API key and no account.

## search_catalog

Pass `catalog.query`, `catalog.filters`, or both. Useful sub-fields, all verified in the
tool's input schema:

- `catalog.context.address_country` — ISO 3166-1 alpha-2. Pass it.
- `catalog.context.currency` — ISO 4217. Pass it. `/agents.md` explicitly asks for both.
- `catalog.context.language` — IETF BCP 47.
- `catalog.filters.price.min` / `.max` — integers in **minor units**.
- `catalog.filters.available` — defaults to `true` (sale-ready items only).
- `catalog.pagination.limit` — default 10, minimum 1.
- `catalog.pagination.cursor` — take it from the previous response.

## lookup_catalog

Resolve several product or variant identifiers in one call. Prefer this over a loop of
`get_product` calls.

## get_product

Full detail for one identifier, with a relevant set of variants. You need the
**ProductVariant id** from here before you can build a cart — `cart.line_items[].item.id`
is the variant id, not the product id.

## Money

Prices come back as integers in the currency's ISO 4217 minor units, paired with a
currency code: `{"amount": 600, "currency": "USD"}` is $6.00. Divide by 100 for
two-decimal currencies before quoting a buyer. Zero-decimal currencies such as JPY are
already whole units.

## Errors

Application errors arrive as JSON-RPC 2.0 error objects and may carry a non-200 HTTP
status. Back off on 429 — the endpoint is rate-limited per IP and no numeric limit is
published. See `errors/adrenalineshoc-problem-types.yml`.
