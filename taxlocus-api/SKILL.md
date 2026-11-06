---
name: taxlocus-api
description: Integrate the TaxLocus US sales-tax API: real-time rate lookups, cart/checkout tax calculation, economic-nexus tracking, product taxability, exemption certificates, and state-return worksheets. Use when asked to "add sales tax", "calculate US sales tax at checkout", "integrate TaxLocus", "track economic nexus", or "is this product taxable in <state>".
license: Apache-2.0
category: api-integration
---

# TaxLocus API — integration guide for AI coding assistants

TaxLocus is a US sales-tax API: address-accurate **rate lookups**, one-call
**cart tax calculation**, **economic-nexus** tracking, **product taxability**,
**exemption certificates**, and **state-return worksheets** (the state portion
of each return; local schedules are not filled in). REST + an MCP server,
OpenAPI 3.1.

- **Base URL:** `https://api.taxlocus.com`
- **Auth:** `Authorization: Bearer tlx_live_…` (get a key in the dashboard → API Keys; a free tier exists). Sandbox keys (`tlx_test_…`) are also issued in the dashboard.
- **OpenAPI:** `https://api.taxlocus.com/openapi.json` · **Docs:** `https://api.taxlocus.com/docs`
- **MCP server:** `https://api.taxlocus.com/mcp` (if your agent speaks MCP, connect it directly instead of writing REST code — see “MCP option”).
- **Errors:** JSON `{"error":"<code>","message":"…"}`; validation errors return `{"error":"bad_input","issues":[…]}`. Status codes: `400` bad input · `401` bad/missing key · `403` tier/feature/quota not allowed · `429` rate-limited.

---

## Phase 1: Detect

Before writing code, understand the host project:

```bash
# Language / runtime
cat package.json requirements.txt pyproject.toml go.mod Gemfile 2>/dev/null | head -20
# Where does checkout / order total / tax get computed today?
grep -rniE "tax|salestax|checkout|order total|line items" src app lib 2>/dev/null | head
# E-commerce platform already in use? (TaxLocus has turnkey Shopify + QuickBooks apps)
grep -riE "shopify|quickbooks|stripe|woocommerce" package.json 2>/dev/null | head
```

| Finding | Implication |
|---|---|
| Uses Shopify or QuickBooks | Don’t hand-code — install the TaxLocus **Shopify app** / **QuickBooks app** (auto rate + nexus). Only use the REST API for custom needs. |
| Custom checkout / cart | Use `POST /v1/cart/calculate` at checkout. |
| Just needs a rate for an address | Use `POST /v1/rate/lookup`. |
| Needs “where am I liable?” | Use the nexus endpoints + record transactions. |
| Marketplace/quotes, no key yet | Try `POST /public/lookup` (no auth, rate-limited) first. |

---

## Phase 2: Recommend

Lead with a concrete plan, not open questions. The common integration is:

1. **Cart tax at checkout** — `POST /v1/cart/calculate` with ship-to address + line items → tax to collect, per jurisdiction. This is the “one API call” path.
2. **Economic-nexus tracking** — `POST /v1/transactions` records each sale; `POST /v1/nexus/status` tells you which states you’ve crossed/are approaching (requires the `nexus_tracker` feature — Indie tier+).
3. **Product taxability** — `POST /v1/taxability/check` (is SKU taxable in a state) or `POST /v1/taxability/classify` (AI: product description → tax category).
4. **Filing**: `GET /v1/reports/summary` / `…/summary.csv` and `GET /v1/reports/filing-form` for a worksheet of the state portion of each return. Local schedules (city, county, district) are not filled in; tell the user they still prepare those.

Recommend: *“I’ll wire `cart/calculate` into checkout and `transactions` so nexus tracks automatically. Want taxability classification and filing reports too?”*

---

## Phase 3: Guide

### 1. Get a key
Sign up at `https://taxlocus.com`, then **dashboard → API Keys** → create a key (`tlx_live_…`). Store it in an env var (`TAXLOCUS_API_KEY`) — never hard-code or commit it.

### 2. Smoke test (no key needed)
```bash
curl -s https://api.taxlocus.com/public/lookup \
  -H "content-type: application/json" \
  -d '{"address":{"line1":"1600 Amphitheatre Pkwy","city":"Mountain View","state":"CA","postal_code":"94043"},
       "product_category":"goods.general"}'
```
`product_category` is required (dotted codes, e.g. `goods.general`, `clothing.general`, `software.saas`, `food.grocery`; full list via `POST /v1/taxability/classify` or the docs). You can pass `lat`+`lng` instead of `address`.

### 3. Calculate cart tax at checkout (the main call)
> Available on all plans including Free. One call does the per-line rate loop and rounding server-side. Use it for any cart with more than one line: it rounds each line's tax against that line's quantity, then sums the rounded figures, so `total` always equals `subtotal + total_tax`. Looping `rate/lookup` and summing `computed_tax` yourself is not equivalent and can land a cent off, because rounding a sum is not the same as summing rounded values.
```javascript
// Node / TypeScript
const res = await fetch("https://api.taxlocus.com/v1/cart/calculate", {
  method: "POST",
  headers: {
    "content-type": "application/json",
    authorization: `Bearer ${process.env.TAXLOCUS_API_KEY}`,
  },
  body: JSON.stringify({
    // ship-to destination (or pass lat + lng instead of address)
    address: { line1: "350 5th Ave", city: "New York", state: "NY", postal_code: "10118" },
    line_items: [
      { item_id: "sku_123", product_category: "goods.general", quantity: 2, unit_price: 49.99 },
      { item_id: "sku_456", product_category: "clothing.general", quantity: 1, unit_price: 19.0 },
    ],
  }),
});
if (!res.ok) throw new Error(`TaxLocus ${res.status}: ${await res.text()}`);
const { total_tax, total_tax_complete, warnings, line_items } = await res.json();
// total_tax_complete is false when a line's taxability is unknown (tax_status
// "unknown", tax 0). That 0 is not an exemption: decide those lines before
// charging total_tax. Log or show `warnings` ("<code>: <explanation>").
```

```python
# Python
import os, requests
r = requests.post(
    "https://api.taxlocus.com/v1/cart/calculate",
    headers={"Authorization": f"Bearer {os.environ['TAXLOCUS_API_KEY']}"},
    json={
        "address": {"line1": "350 5th Ave", "city": "New York", "state": "NY", "postal_code": "10118"},
        "line_items": [
            {"item_id": "sku_123", "product_category": "goods.general", "quantity": 2, "unit_price": 49.99},
        ],
    },
    timeout=10,
)
r.raise_for_status()
tax = r.json()["total_tax"]
```

### 4. Record the sale so nexus tracks itself (Indie tier+)
```javascript
await fetch("https://api.taxlocus.com/v1/transactions", {
  method: "POST",
  headers: { "content-type": "application/json", authorization: `Bearer ${key}` },
  body: JSON.stringify({
    entity_id: ENTITY_ID,            // from POST /v1/entities (created once)
    state: "NY",
    sale_amount: 118.98,
    transaction_date: "2026-05-22",
    external_ref: "order_1001",
    idempotency_key: "order_1001",   // retries are a no-op — safe to resend
  }),
});
```
Then `POST /v1/nexus/status` (with `entity_id`) returns each state’s status: approaching / met / registered.

### 5. Book refunds so the totals stay honest
```javascript
await fetch("https://api.taxlocus.com/v1/refunds", {
  method: "POST",
  headers: { "content-type": "application/json", authorization: `Bearer ${key}` },
  body: JSON.stringify({
    entity_id: ENTITY_ID,
    state: "NY",
    refund_amount: 59.49,            // POSITIVE — the API stores it negative
    tax_refunded: 5.28,              // POSITIVE — omit if you don't know it
    refund_date: "2026-06-02",
    refunded_ref: "order_1001",      // external_ref of the original sale
    idempotency_key: "refund_2001",
  }),
});
```
The ledger is append-only: a refund is a new negative row, never an edit of the
sale. Pass `refunded_ref` wherever you can — the refund then inherits the
original sale's **state**, exempt / marketplace / jurisdiction fields so report
totals net against the right buckets, and cumulative refunds are capped at the
original amount (`422 refund_exceeds_sale`).

`state` is optional when `refunded_ref` resolves — handy when your refund event
has no address (Shopify's `refunds/create` is exactly this shape). Send one or
the other; a `refunded_ref` that matches nothing, with no `state`, returns
`422 unknown_refund_state` rather than being guessed at.

**Using the Shopify or QuickBooks connector?** Refunds are already handled —
Shopify via the `refunds/create` webhook, QuickBooks via CreditMemo and
RefundReceipt in both the daily sync and the initial backfill. Nothing to wire.

Refunds reduce the **dollar** totals that economic-nexus thresholds measure, but
not the **transaction count** — a refund is not an extra sale. If you skip this
step your gross sales stay permanently overstated, which can both report nexus
you have not actually crossed and inflate the base you file on.

### 6. Handle errors + limits
- Always check the status code. Retry `429` (rate-limited) and `5xx` with exponential backoff; do **not** retry `400/401/403`.
- `POST /v1/transactions` and `POST /v1/refunds` are idempotent on `idempotency_key` — safe to retry.
- Per-tier rate limits (req/s): Free 5 · Indie 20 · Pro 50 · Scale 200.

---

## Verification

1. The `/public/lookup` smoke test returns a rate breakdown.
2. An authed `cart/calculate` returns a `total_tax`.
3. In the dashboard, your **API usage** counter increments.
4. If using nexus: after a few `transactions`, `nexus/status` reflects them.

---

## Reference

### Endpoints
| Endpoint | Purpose | Tier |
|---|---|---|
| `POST /public/lookup` | Rate for an address (no auth, rate-limited) | — |
| `POST /v1/rate/lookup` | Address → full rate breakdown | Free+ |
| `POST /v1/cart/calculate` | Cart/line items → tax to collect | Free+ |
| `POST /v1/taxability/check` | Is a product taxable in a state | Free+ |
| `POST /v1/taxability/classify` | AI: description → tax category | Free+ |
| `POST /v1/nexus/check` | Sales-by-state → where you have nexus | Free+ |
| `POST /v1/entities`, `GET /v1/entities` | Manage tracked entities | Indie+ |
| `POST /v1/transactions` | Record a sale (feeds nexus) | Free+ (Free: 1 entity, 50-order cap) |
| `POST /v1/refunds` | Book a full/partial refund against the ledger | Free+ |
| `POST /v1/nexus/status` | Per-state nexus status for an entity | Free+ |
| `GET /v1/reports/summary[.csv]` | Period summaries for filing | Free+ |
| `GET /v1/reports/filing-form` | State form outputs | Indie+ |
| `POST /v1/exemptions`, `…/check`, `…/verify` | Exemption certificates | Pro+ |
| `POST /v1/webhooks` | Register nexus-crossing webhooks | Pro+ |

### Tiers (monthly quota · features)
- **Free** — 1,000 calls · rate lookup, cart calculation, AI classify, taxability, and a nexus-tracking trial (record transactions for 1 entity up to 50 orders, plus nexus status, filing calendar, and a summary report). Excludes filing-form output, exemption certs, multi-entity, and webhooks.
- **Indie** — 25,000 · adds nexus tracker.
- **Pro** — 250,000 · adds multi-entity, webhooks, exemption certs.
- **Scale** — unlimited · adds bulk export, Slack alerts.

### MCP option
If the host app or your agent supports MCP, connect TaxLocus directly instead of writing REST glue:
```
https://api.taxlocus.com/mcp
```
It exposes the same capabilities (rate lookup, cart calc, taxability, nexus) as MCP tools.

### Don’t
- Don’t hard-code or commit keys — use an env var.
- Don’t recompute tax client-side from a cached rate; call `cart/calculate` at order time (rates + rules change).
- Don’t treat TaxLocus output as tax advice — it’s data to support filing; see the Terms.
