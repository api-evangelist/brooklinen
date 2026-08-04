# Brooklinen

<!-- API-EVANGELIST-PROVENANCE:BEGIN -->
> ### About this repository
>
> **This is not our API.** This repository is an independent, third-party profile of a company's
> **publicly available** API surface, maintained by [API Evangelist](https://apievangelist.com).
> API Evangelist does not operate, host, resell, or support this company's APIs, and is not
> affiliated with or endorsed by the company unless stated on the profile.
>
> **Where the information came from.** Everything here is assembled from material a member of the
> public can reach with a browser and no credentials — the company's own website, developer portal
> and documentation, the specifications it publishes for public use (OpenAPI, AsyncAPI, JSON Schema,
> `apis.json`, `llms.txt` and similar), its public repositories, and its public status, pricing and
> changelog pages. **Nothing here is obtained by breaching a system, defeating an access control, or
> using credentials of any kind.**
>
> **The rating is an independent assessment.** The Kin Score and Agent Readiness rating are
> independently calculated scores of a company's *public* API artifacts, produced by API Evangelist
> against a published rubric. They are not certifications, endorsements, security assessments, or
> audits, and they score published artifacts — not the quality, safety, or security of the software.
>
> **Corrections, re-scores, and removal are free.** No partnership, contract, or purchase is
> required, and you do not need to justify the request.
>
> - **Something wrong?** Open an issue on this repository, or email
>   [info@apievangelist.com](mailto:info@apievangelist.com).
> - **Published something new?** Ask for a re-score and we will re-run the rating.
> - **Want the listing taken down?** Say so and we will honor it. The profile is reduced to your
>   company name, a factual description, and a link to your own site, and the company is recorded as
>   **unrated** — never scored zero for having asked.
>
> **Response times.** Acknowledgement within **one business day**; removal or restriction within
> **two business days**; corrections and re-scores within **five business days**.
>
> **On a security or compliance team?** Email
> [info@apievangelist.com](mailto:info@apievangelist.com) with *security* in the subject line and
> you will get a person, not a form. We will tell you exactly which public URLs this profile was
> built from so your team can see the same surface we did, and we will take the listing down on
> request while you work through it.
>
> Full detail: **[Where this data comes from](https://apievangelist.com/about/where-our-data-comes-from)**
<!-- API-EVANGELIST-PROVENANCE:END -->

Brooklinen is a direct-to-consumer home essentials brand founded in Brooklyn, New York in 2014, selling
sheets, duvet covers, comforters, pillows, towels, robes and loungewear.

Brooklinen runs **no traditional developer API program** — no keys, no portal, no SDKs, no OpenAPI. What it
does have is a real, live **agentic commerce surface**, which is what this repo profiles.

- Website: https://www.brooklinen.com/
- Agent instructions: https://www.brooklinen.com/agents.md (mirrored at `/llms.txt`)
- UCP discovery: https://www.brooklinen.com/.well-known/ucp
- UCP/MCP endpoint: https://www.brooklinen.com/api/ucp/mcp
- Secondary-market listing: https://forgeglobal.com/brooklinen_stock/

## The two surfaces

| | Storefront JSON | UCP / MCP |
|---|---|---|
| Base | `https://www.brooklinen.com` | `https://www.brooklinen.com/api/ucp/mcp` |
| Auth | none (anonymous) | UCP agent profile required (`UCP-Agent` header) |
| Can do | read catalog, products, variants, search, session cart | search, cart, checkout, order |
| Cannot do | transact | be reached without a published agent profile |

They are overlapping but non-identical projections of one commerce core — see
`mcp/brooklinen-tool-crosswalk.yml`, which records the 9 MCP-only tools and the 5 REST-only operations.

## Notable findings (probed 2026-08-02)

- **UCP 2026-04-08** merchant profile, with `2026-01-23` still supported. Eight negotiated capabilities
  (cart, checkout, fulfillment, discount, order, catalog search/lookup, Shopify catalog) and three payment
  handlers (Google Pay, Shopify card, Shop Pay).
- **MCP `tools/list` is gated** on agent identity — anonymous calls return HTTP 422 / JSON-RPC `-32001`
  `invalid_profile_url`. The 13 real tool names come from the UCP shopping OpenRPC document that
  Brooklinen's own discovery profile names as its schema (saved verbatim in `mcp/`).
- **Idempotency is real**: `meta.idempotency-key` (UUID) maps to the HTTP `Idempotency-Key` header.
- **OAuth 2.0 / OIDC** customer-account metadata is served at the store origin (RFC 8414 + RFC 9728), with
  four scopes including `customer-account-mcp-api:full`.
- **Human-in-the-loop is mandatory** for payment, stated in both `/agents.md` and `/robots.txt`.
- **No A2A agent card** — `/.well-known/agent-card.json` and `/.well-known/agent.json` both 404 on
  `www.brooklinen.com` and `brooklinen2.myshopify.com`. Nothing was authored in their place.
- **No security.txt, no trust center, no status page, no changelog, no first-party SDKs.**

`openapi/brooklinen-storefront-openapi.yml` is **generated by API Evangelist**, not published by Brooklinen —
built from the endpoint list Brooklinen documents in `/agents.md` plus a live probe of each endpoint. Every
operation carries `x-evidence` with its observed HTTP status.
