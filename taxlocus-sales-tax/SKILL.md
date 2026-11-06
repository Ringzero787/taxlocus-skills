---
name: taxlocus-sales-tax
description: Use the TaxLocus connector to answer US sales tax questions accurately: combined rate lookups, economic nexus, product taxability, and filing calendars. Apply when the user asks about US sales tax rates ("how much sales tax on…"), economic nexus / where they must collect, whether a product is taxable in a state, sales-tax filing due dates, or tracking nexus over time. Picks the right tool, explains the sourcing and nexus rules, and surfaces the source citation.
license: Apache-2.0
category: finance
---

# TaxLocus — US sales tax (connector usage)

TaxLocus answers US sales tax questions through six MCP tools, and every result
is citation-backed. This skill helps you choose the right tool and interpret
the results correctly.

## Tool selection

| When the user asks… | Use |
|---|---|
| "What's the sales tax rate for <address/ZIP>?" · "how much tax on this order?" | `lookup_rate` |
| "Where do I have economic nexus?" · "have I crossed <state>'s threshold?" | `check_nexus` |
| "Is <product> taxable in <state>?" | `taxability_check` |
| "When are my sales tax returns due in <state>?" | `filing_calendar` |
| "Record this sale" · ongoing nexus tracking | `record_transaction` (needs a tracked entity) |
| "What's my running nexus status?" | `nexus_status` (needs a tracked entity) |

`lookup_rate` accepts an address or ZIP (it geocodes addresses) plus a
`product_category`. It returns the combined rate broken down by
state / county / city / special district for the location you send, and
reports the state's default sourcing rule.

## Interpreting results — apply these rules

**Always surface the citation.** Each result is backed by a source rate table
or statute. Show it so the user can audit the number.

**Sourcing.** Whether a sale is origin- or destination-sourced varies by state
and by intra- vs. interstate sale. `lookup_rate` does not choose the address
for you: it rates the location it is given. The `sourcing` field is the
state's default rule, so you know which address to rate: the seller's ship-from
address for in-state sales in an origin state, the buyer's ship-to address
otherwise. Some states split the rule between the state and local layers, so
when the answer depends on it, tell the user to confirm the state's rule. Do
not tell the user the sourcing question is settled just because a rate came
back.

**Economic nexus.** Thresholds differ widely (e.g., California $500k sales-only;
many states $100k OR 200 transactions). When reading `check_nexus`:
- State whether the logic is sales-only or sales-OR-transactions.
- Note the measurement period (current or prior calendar year).
- Note whether marketplace sales count toward the threshold.
- Distinguish "approaching" from "met" — "met" means a registration/collection
  obligation, so advise the user to confirm and register.

**Taxability — never infer "exempt" from missing data.** If `taxability_check`
or a rate lookup returns no taxability (or a 0% effective rate) for a product
category, treat it as "not modeled / unknown for that state," NOT as "exempt."
Tell the user to verify against the cited source, and pass an accurate
`product_category` to get a meaningful answer in the first place.

`lookup_rate` returns `taxability`: `taxable`, `exempt` or `unknown`. When it
is `unknown` the rate reads 0 and `computed_tax` is null because TaxLocus has
no rule for that category there; say so plainly and never call it exempt.

**Warnings.** `lookup_rate` returns a `warnings` array of
`"<code>: <explanation>"` strings. Pass each one on to the user with the rate:
`zip_centroid` (rate is for the ZIP code's centre, not the street address),
`no_local_jurisdiction` (state rate only; a local tax may be missing),
`taxability_unknown`, and `holiday_provisional` (holiday dates not yet
confirmed by the state).

**Transactions.** `record_transaction` appends a sale to a tracked entity's
ledger (it needs an `entity_id`) and is idempotent on `idempotency_key`. Use it
for ongoing nexus tracking; `nexus_status` then returns the running totals.

## Guardrails

- **US only.** No VAT, GST, or non-US jurisdictions. If asked about those, say
  TaxLocus doesn't cover them.
- **Not tax or legal advice.** Results are data to support decisions. For
  compliance or filing actions, tell the user to verify against the cited
  source and, where appropriate, consult a tax professional.
- **Don't recompute from a cached rate.** Rates and rules change — call the tool
  at the moment you need the answer rather than reusing an earlier number.
