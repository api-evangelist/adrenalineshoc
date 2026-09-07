---
name: Build and maintain an Accelerator Active Energy cart
description: Create, inspect, update and cancel a cart on the Accelerator Active Energy (A SHOC) storefront over its UCP/MCP endpoint.
api: mcp/adrenalineshoc-mcp.yml
endpoint: https://www.drinkaccelerator.com/api/ucp/mcp
operations:
  - get_product
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
consequence: write
generated: '2026-09-07'
method: generated
source: mcp/adrenalineshoc-tools-list.json
---

# Build and maintain an Accelerator Active Energy cart

These calls change server state. None of them charges anyone.

## Preconditions

Resolve a **ProductVariant id** first, with `get_product` or `search_catalog`. The cart
line item field is documented as "The Product Variant ID to add or update" — a product id
will not work. Note that the store catalog was empty at profiling time, so you may have
nothing to add.

Every call needs `meta.ucp-agent.profile` as a resolvable URI. See the browse skill.

## create_cart

`cart.line_items[]` is required on create; each entry needs `item.id` (the variant id) and
`quantity`. Optional but worth setting:

- `cart.buyer.email`, `cart.buyer.phone_number` — this is PII. Only send what the buyer
  gave you for this purchase.
- `cart.context.*` — `address_country`, `address_region`, `postal_code`, `language`,
  `currency`, `intent`, `eligibility[]`. These are *provisional hints*: authoritative data
  such as a shipping address supersedes them, and unsupported hints are ignored without
  error.
- `cart.attribution.*` — `referring_domain`, `click_id_tag`/`click_id_value`, and the five
  `utm_*` fields. Forward what the buyer's session actually carried; do not invent
  campaign values.

The response carries the **cart id**. Hold it — every later call needs it.

## get_cart / update_cart

`get_cart` takes `meta` + `id`. `update_cart` takes `meta` + `id` + a `cart` object and
diffs it, so one tool covers what the Storefront GraphQL API splits across
`cartLinesAdd`, `cartLinesUpdate`, `cartLinesRemove`, `cartBuyerIdentityUpdate`,
`cartAttributesUpdate`, `cartNoteUpdate` and `cartDiscountCodesUpdate`.

## No idempotency key

There is none — not in any tool schema, not in `/agents.md`. Safety comes from the
server-assigned id: create once, then address that id. **If a `create_cart` call times out,
do not blindly retry it.** You may have made two carts. There is no way to ask the server
"did my request land?" other than acting on an id you already hold.

## cancel_cart

`meta` + `id`. The tool description is one line: "Cancels a cart." No window, no TTL and no
cart-lifetime statement is published anywhere, so treat cancellation as best-effort and
verify with `get_cart` rather than assuming.

## Next

When the cart is right, hand off to
`skills/adrenalineshoc-checkout-with-buyer-approval.md`. Do not proceed to payment from
here.
