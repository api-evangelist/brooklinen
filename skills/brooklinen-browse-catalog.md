---
name: Browse the Brooklinen catalog
description: >-
  Read Brooklinen product, variant, price and availability data from the anonymous Shopify storefront JSON
  endpoints — no credentials, no transaction. Use this when the task is research, comparison, price checking
  or building a local mirror of the catalog, not buying.
api: openapi/brooklinen-storefront-openapi.yml
operations:
- listProducts
- getProduct
- listCollectionProducts
- suggestSearch
generated: '2026-08-02'
method: generated
source: https://www.brooklinen.com/agents.md
---

# Browse the Brooklinen catalog

Brooklinen's storefront at `https://www.brooklinen.com` publishes read-only JSON for its catalog. Brooklinen
documents this itself in [`/agents.md`](https://www.brooklinen.com/agents.md), under "Read-Only Browsing (No
Authentication Required)". Nothing here requires a credential.

**If the user wants to buy something, stop and use the purchase skill instead** —
`brooklinen-agentic-purchase.md`. This skill never transacts.

## Before you start

Read `https://www.brooklinen.com/agents.md`. It is the canonical agent-facing description of the store and it
changes weekly (`/sitemap_agentic_discovery.xml` declares `changefreq: weekly`). Do not cache it
indefinitely.

## Steps

### 1. Find candidate products

For a keyword query, call `suggestSearch`:

```
GET https://www.brooklinen.com/search/suggest.json?q=percale%20sheets&resources[type]=product
```

Results arrive under `resources.results.products[]`, each with `title`, `handle`, `url`, `price`,
`available` and an HTML `body` blurb. The `handle` is what you use to address the product everywhere else.

Brooklinen's own docs also list `GET /search?q={query}&type=product`. That renders HTML — prefer the
`suggest.json` form above when you want structured data.

### 2. Or enumerate a collection

To walk a merchandising collection, call `listCollectionProducts`:

```
GET https://www.brooklinen.com/collections/all/products.json?limit=250&page=1
```

The `all` handle is the full catalog. Paging is page-number based on `limit` and `page`. There is **no**
cursor, no total count and no `Link` header — page until `products` comes back empty, then stop. `listProducts`
(`/products.json`) is the same contract without the collection scope.

### 3. Read one product in full

```
GET https://www.brooklinen.com/products/{handle}.json
```

`getProduct` returns `{ "product": { … } }`. The fields that matter:

- `variants[]` — the purchasable units. Each carries `id`, `sku`, `price` (a decimal **string**, not cents),
  `compare_at_price` (null when not discounted), `available`, and `option1`/`option2`/`option3`.
- `options[]` — the option axes (e.g. Size, Color) and their permitted `values`. Read these to understand
  what `option1`/`option2`/`option3` on a variant mean; the positions are not self-describing.
- `images[]` — each with `src`, `width`, `height` and `variant_ids[]` binding it to specific variants.
- `body_html` — the description, as HTML. Strip tags before quoting it to a user.

Availability is per-variant. A product with any `available: true` variant is buyable; do not report a product
as in stock without naming which variant.

### 4. Report to the user

Quote `price` with the currency the store returned. If the user is outside the US, say so — pricing and
availability on this surface are the store default. Buyer-specific pricing requires the UCP surface, which
takes `context.address_country` and `context.currency`.

## Rules

- **Never transact here.** `robots.txt` Disallows `/cart/`, `/checkout`, `/checkouts/`, `/orders` and
  `/account`. Do not fetch them and do not script form posts against them.
- **Back off on 429.** Rate limits are per IP and no `Retry-After` is published. Use exponential backoff.
- **Errors are bare HTTP status codes.** There is no problem+json envelope on this surface. A 404 on a
  `{handle}` means the handle is wrong or the product is unpublished. A 400 on a sub-sitemap means you built
  the URL yourself instead of following it from `/sitemap.xml` — those require their `from`/`to` parameters.
- See `conventions/brooklinen-conventions.yml` and `errors/brooklinen-problem-types.yml` for the full
  cross-cutting rules.
