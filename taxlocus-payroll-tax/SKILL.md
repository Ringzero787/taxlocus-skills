---
name: taxlocus-payroll-tax
description: Use the TaxLocus connector to answer US payroll-tax questions — federal FICA/FUTA, state personal income tax and SUTA wage bases, state reciprocity, and local (city/county/municipal) income tax in PA, OH, IN, MD, MI, NY, MO, DE. Apply when the user asks "what's the local income tax in <city>", "what's the SUTA wage base in <state>", "is there reciprocity between <A> and <B>", "what's the FICA cap this year", or general "how much payroll tax for…" questions. Returns rate data with citations. NOT a withholding calculation engine — picks the right rate, surfaces coverage gaps honestly, and enforces the not-tax-advice guardrail.
license: Apache-2.0
category: finance
---

# TaxLocus — US payroll-tax open data (connector usage)

TaxLocus publishes US payroll-tax **rate reference data** under CC-BY-4.0,
served via the MCP tool `lookup_payroll_rates` and the REST endpoint
`POST /v1/payroll/lookup`. This skill helps you answer payroll-tax rate
questions correctly and surface the gaps honestly.

## Tool selection

| When the user asks… | Use |
|---|---|
| "What's the federal/state/local payroll tax for an employee living in X, working in Y?" | `lookup_payroll_rates` |
| "What's the FICA wage base / FUTA effective rate this year?" | `lookup_payroll_rates` (federal section) |
| "What's the SUTA wage base in <state>?" | `lookup_payroll_rates` (state section) |
| "Is there reciprocity between <A> and <B>?" | `lookup_payroll_rates` with both states |
| "What's the local income tax in <city>?" | `lookup_payroll_rates` with `work_state` (and `work_city` / `work_zip` if relevant) |

`lookup_payroll_rates` accepts `year`, `resident_state`, `work_state`,
optionally `work_city` / `work_zip`, and returns federal + state PIT + state
SUTA + reciprocity (if applicable) + local rows for the requested
jurisdictions. Every row carries `citation_url` and `as_of`.

## Coverage — read this BEFORE answering

**Local income tax is published for 8 states only.** If the user asks about a
local income tax in any other state, **do not invent a rate** — say what
exists, what TaxLocus doesn't cover, and link to the documentation.

| Covered (use the data) | Not covered (be honest) | No local income tax at all |
|---|---|---|
| PA (2,627 PSDs) | KY — ~200 cities/counties, no central registry | AK, AZ, AR, CA, CT, FL, GA, HI, ID, IL, LA, ME, MA, MN, MS, MT, NE, NV, NH, NM, NC, ND, OK, RI, SC, SD, TN, TX, UT, VT, VA, WA, WI, WY, DC |
| OH (890 munis: RITA + CCA + DOR) | IA — school district surtaxes (~280 districts) | |
| IN (92 counties) | OR — Multnomah PFA + Portland Metro SHS | |
| MD (24 counties + Baltimore City) | AL — Birmingham/Bessemer/Gadsden + Macon County | |
| MI (24 cities) | CO — Denver/Aurora/Glendale/Greenwood Village/Sheridan (flat $, not %) | |
| NY (NYC + Yonkers) | WV — Charleston/Huntington/Wheeling city service fees (flat $) | |
| MO (Kansas City + St. Louis) | KS — intangibles tax (interest income, not wages) | |
| DE (Wilmington) | NJ Newark payroll tax (employer-side) | |

Full coverage doc:
`https://github.com/Ringzero787/taxlocus-data/blob/main/PAYROLL-COVERAGE.md`

**For state PIT and state SUTA wage base, all 50 states + DC are covered**
(top-line rate or no-tax flag for PIT; the wage base for SUTA).

## Interpreting results — apply these rules

**Always surface `citation_url` and `as_of`.** Every row is backed by an
official source (IRS publication, state DOR, state UI agency, local
collector). Show the link so the user can audit the number.

**Local-income-tax row semantics — read both rate columns:**
- `resident_rate` — what an employee who LIVES in the jurisdiction owes
- `non_resident_rate` — what a non-resident who WORKS in the jurisdiction owes
- If the two are equal, the city treats it as a pure workplace tax (typical for OH muni and IN counties).
- If `non_resident_rate` is NULL, the jurisdiction does not tax non-residents at that level (typical for MD county tax).

**OH muni tax is a workplace tax.** Anyone earning income in the city owes
the rate regardless of residence. Residents who work in another OH city
typically get a credit against their resident city tax — the credit factor
and credit limit are captured in the `notes` field.

**PA EIT — there are TWO rates.** `resident_rate` is "Total Resident Income
Tax" (municipal + school-district EIT combined). `non_resident_rate` is
"Municipal Nonresident EIT" only (school-district EIT doesn't apply to
non-residents). PA also has a uniform 1% Local Services Tax (~$52/year) on
workplace — call that out if relevant; it's not in this dataset (LST is a
flat occupational tax, not a percent).

**State reciprocity.** When the user describes a work-state + resident-state
pair that has reciprocity (e.g., NJ resident working in PA), call it out and
note the required exemption form (PA REV-419, MD MW507, etc.) — without the
form the non-resident state will still withhold.

**Top-line state PIT ≠ withholding amount.** `payroll_state_pit.flat_rate`
(for flat states) or `top_marginal` (for progressive states) is the rate at
the top of the structure. For progressive states, **do not apply
`top_marginal` as the withholding rate** — that overwithholds for everyone
below the top bracket. Tell the user to consult the state's published
withholding tables (linked in `citation_url`) or use a payroll engine.

**SUTA — the rate is per-employer, not lookupable.** TaxLocus publishes the
state's taxable wage base (the cap). The actual SUTA tax rate is
experience-rated and the state UI agency mails each employer their rate
each year. If the user asks "what's my SUTA rate," tell them to check the
state's annual notice.

## Guardrails

- **NOT TAX OR LEGAL ADVICE. NOT WITHHOLDING TABLES. NOT A CALCULATION
  ENGINE.** TaxLocus payroll-tax data is informational rate reference only.
  Errors in payroll withholding create Form 941 employer penalties.
- **Always tell the user to verify against `citation_url`** before relying
  on a rate for actual withholding.
- **US only.** No foreign payroll tax.
- **Don't infer "no local tax" from an empty local row.** If the user's
  city is not in a covered state, say "TaxLocus does not yet cover local
  income tax in `<state>`; if a local tax exists there, check the city
  directly." See the coverage table above for the not-covered list.
- **Don't compute paycheck withholding from these rates.** Tell the user
  that translating these rates into a per-paycheck withholding amount
  requires their payroll engine (Symmetry Payroll Point, Vertex Payroll, an
  in-house calculator, or their CPA) applying the employee's W-4, filing
  status, and pay frequency.
- **Annual refresh window.** Federal SS wage base + state SUTA wage bases
  are confirmed in Q4 for the following year. If the user is asking about
  a year that hasn't been re-cut yet, tell them.
