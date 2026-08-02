---
name: Buy from Brooklinen over UCP/MCP
description: >-
  Search, cart and check out on Brooklinen's Universal Commerce Protocol endpoint at /api/ucp/mcp. Use when
  the user actually wants to purchase. Payment completion requires explicit, contemporaneous human approval —
  this skill will not finalize a payment on its own.
api: mcp/brooklinen-mcp.yml
operations:
- search_catalog
- lookup_catalog
- get_product
- create_cart
- update_cart
- get_cart
- create_checkout
- update_checkout
- complete_checkout
- get_order
generated: '2026-08-02'
method: generated
source: https://www.brooklinen.com/agents.md
---

# Buy from Brooklinen over UCP/MCP

Brooklinen implements the [Universal Commerce Protocol](https://ucp.dev) for agent-driven commerce. The
transactional surface is an MCP JSON-RPC endpoint:

```
POST https://www.brooklinen.com/api/ucp/mcp
Content-Type: application/json
Accept: application/json, text/event-stream
```

## The hard prerequisite: you need a UCP agent profile

**This endpoint will refuse you until you have one.** Every request carries a `meta` object, and
`meta.ucp-agent.profile` is required — a URI pointing at *your own* published UCP profile document, which
Brooklinen fetches and validates. It maps to the HTTP `UCP-Agent` header.

Without a resolvable profile, even `tools/list` fails:

```json
{"jsonrpc":"2.0","id":1,"error":{"code":-32001,"message":"UCP discovery failed",
 "data":{"code":"invalid_profile_url","content":"Unable to fetch agent profile: Missing profile uri",
 "continue_url":"https://brooklinen2.myshopify.com/"}}}
```

HTTP 422. This was verified against the live endpoint on 2026-08-02. If you cannot publish a profile, you
have two honest options: fall back to `brooklinen-browse-catalog.md` for read-only work, or install the
Shopify Shop skill at `https://shop.app/SKILL.md`, which Brooklinen's own `/agents.md` recommends for
personal shopping assistants and which handles buyer-approved checkout through Shop Pay.

## Step 0 — Discover

```
GET https://www.brooklinen.com/.well-known/ucp
```

Confirm the protocol version you intend to speak is in `supported_versions` (currently `2026-04-08` latest
stable, and `2026-01-23`) and read `services["dev.ucp.shopping"]` for the MCP `endpoint`. Do not hardcode the
endpoint — resolve it from this document. Check `capabilities` before relying on discounts or fulfillment
options; both require protocol `>= 2026-04-08` on this store.

## Step 1 — Search

Call `search_catalog` with the buyer's intent. Always pass buyer context — `context.address_country` and
`context.currency` — or you will get default-market pricing and availability and quote the user the wrong
number.

Use `lookup_catalog` when you already hold product or variant identifiers (it is the batch form), and
`get_product` for a single product's full detail.

## Step 2 — Cart

`create_cart` with the chosen items, then `update_cart` to adjust quantities and `get_cart` to re-read state.
`cancel_cart` abandons it.

Send `meta.idempotency-key` — a UUID, mapped to the HTTP `Idempotency-Key` header — on every cart and
checkout mutation. It is the only retry-safety mechanism on this surface. Generate one per logical operation
and reuse it verbatim on retries; generating a fresh key on retry defeats the purpose and can duplicate the
operation.

## Step 3 — Checkout

`create_checkout` opens the session. `update_checkout` sets the shipping address and fulfillment method.

Note this store's fulfillment config: `allows_multi_destination.shipping` is **false** and
`allows_method_combinations` is `[["shipping"]]`. One shipping destination, one method. Do not attempt to
split an order across addresses.

`get_checkout` re-reads state; `cancel_checkout` abandons.

## Step 4 — Payment, with a human

**Stop here and ask the user.** Brooklinen states it in two places — `/agents.md` and `/robots.txt`:

> Checkouts are for humans. Do NOT complete checkout, payment, or order placement automatically — no
> scripted form fills, browser automation, or end-to-end agent flows that finalize payment without an
> explicit, contemporaneous human approval step.

Only call `complete_checkout` after the user has seen the final total, shipping address and method, and has
approved *this specific payment, now*. Approval given earlier in the conversation for a different total is
not approval for this one. If you cannot get contemporaneous approval, route through Shop Pay via
`https://shop.app/SKILL.md` instead.

Available payment handlers on this store, from the discovery profile: Google Pay (`gpay`), Shopify card
(`shopify.card`, accepting visa/master/amex/discover/diners_club) and Shop Pay (`shop_pay`).

## Step 5 — Confirm

`get_order` returns the placed order. Report the order identifier back to the user.

## Error handling

- Transport failures come back as JSON-RPC error objects (`error.code`, `error.message`, `error.data`).
- Business-logic failures use the UCP shopping error response shape: `ucp` (status `error`), `messages[]`
  (at least one), and an optional `continue_url` for buyer handoff or session recovery. **When you get a
  `continue_url`, hand the user that link** — it is the designed escape hatch back to a human flow.
- 429 means you are rate-limited per IP. Back off exponentially; no `Retry-After` is published.

See `errors/brooklinen-problem-types.yml`, `conventions/brooklinen-conventions.yml` and
`mcp/brooklinen-tool-crosswalk.yml`.
