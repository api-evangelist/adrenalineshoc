---
name: Complete an Accelerator Active Energy checkout with buyer approval
description: Promote a cart to checkout, attach fulfillment and payment, and complete the purchase — with the mandatory human approval step the store requires.
api: mcp/adrenalineshoc-mcp.yml
endpoint: https://www.drinkaccelerator.com/api/ucp/mcp
operations:
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
consequence: financial
human_in_the_loop: required
generated: '2026-09-07'
method: generated
source: mcp/adrenalineshoc-tools-list.json
---

# Complete an Accelerator Active Energy checkout with buyer approval

**This skill spends the buyer's money, and one step of it cannot be undone.**

## The rule the store publishes, verbatim

> "Checkouts are for humans. Do NOT complete checkout, payment, or order placement
> automatically — no scripted form fills, browser automation, or end-to-end agent flows
> that finalize payment without an explicit, contemporaneous human approval step."
> — `https://www.drinkaccelerator.com/robots.txt` and `/agents.md`

Contemporaneous means *at the moment of payment*, not "the buyer told me to shop earlier".
If you cannot get that approval in the moment, stop here and route the purchase through
Shop Pay via `https://shop.app/SKILL.md` instead.

## There is no undo

Read this before you call `complete_checkout`. The tool set has **thirteen tools and no
refund, void, reverse or order-cancel among them.** `get_order` is read-only.
`cancel_checkout` only works on a checkout that has not completed. Once
`complete_checkout` succeeds, reversal is a human customer-service action against Shopify
or the merchant — you have no protocol path back. Treat the approval step as the last
reversible moment in the flow.

## Sequence

1. **create_checkout** — `meta` + `checkout`. Pass `checkout.cart_id` to convert an existing
   cart; when you do, the store uses the cart's `line_items`, `context` and `buyer` and
   ignores duplicates you resend. The response carries the **checkout id**.
2. **update_checkout** — `meta` + `id` + `checkout`. This is where the shipping address and
   the delivery method go (`checkout.fulfillment.methods`), and where a discount code goes
   (`checkout.discounts.codes`). Per the tool's own guidance, only ask about a discount
   code if the buyer mentions having one. This call carries the most sensitive payload on
   the surface — address plus payment instrument.
3. **get_checkout** — re-read totals, taxes and discounts and show the buyer the real
   number, converted from minor units. `{"amount": 2500, "currency": "USD"}` is $25.00.
4. **Ask the buyer.** Explicitly, now, with the final total, the items and the shipping
   address in front of them.
5. **complete_checkout** — `meta` + `id` + `checkout`. Only after step 4. Returns the order
   ID and the Thank You Page URL, or errors.
6. **get_order** — `meta` + `id`. Confirm what actually happened.

## Payment instruments

`checkout.payment.instruments[]` requires `id`, `handler_id` and `type`. The handler must be
one this merchant declares in its UCP profile (`well-known/adrenalineshoc-ucp.json`):

- `gpay` — Google Pay, `com.google.pay`, merchant "Accelerator Active Energy", cards VISA /
  MASTERCARD / AMEX / DISCOVER
- `shopify.card` — `dev.shopify.card`
- `shop_pay` — `dev.shopify.shop_pay`

The schema is conditional: `handler_id: apple-pay` requires `billing_address` and an
`apple_pay_token` credential; every other handler requires `credential.token` and
`credential.type`.

Fulfillment is single-destination shipping — the profile declares
`method_combinations: [["shipping"]]` and an empty `multi_destination` list.

## cancel_checkout

`meta` + `id`. Meaningful only before completion. No window is published; verify with
`get_checkout` rather than assuming it took.

## No idempotency key, no test mode

Neither exists here. There is no sandbox, no test card and no dry-run flag for this
merchant. A retried `complete_checkout` is a real second attempt at a real payment. If a
call times out, read state with `get_checkout` or `get_order` before doing anything else.

## If it fails

Errors are JSON-RPC 2.0 objects and can arrive with a non-200 status. `-32001 /
invalid_profile_url` at HTTP 422 means your `meta.ucp-agent.profile` was missing or
unresolvable. Back off on 429. Full catalogue in
`errors/adrenalineshoc-problem-types.yml`.
