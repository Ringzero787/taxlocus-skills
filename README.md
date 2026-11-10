# TaxLocus MCP & Agent Skills

[![Taxlocus MCP server](https://mcprush.com/badge/taxlocus-mcp/card.svg)](https://mcprush.com/to/taxlocus-mcp?ref=badge)

Official [Model Context Protocol (MCP)](https://modelcontextprotocol.io) server reference and [Agent Skills](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills) for working with **US sales tax and payroll tax** via [TaxLocus](https://taxlocus.com):

- **Remote MCP Server (Streamable HTTP):** `https://api.taxlocus.com/mcp`
- **REST API:** `https://api.taxlocus.com` ([OpenAPI docs](https://api.taxlocus.com/docs))
- **Registry Manifest:** [`server.json`](./server.json) (`com.taxlocus/taxlocus`)

---

## Connecting to the Remote MCP Server

TaxLocus exposes a stateless Streamable HTTP MCP server at `https://api.taxlocus.com/mcp`. It supports both **OAuth 2.1** (RFC 9728 Protected Resource Metadata at `https://api.taxlocus.com/.well-known/oauth-protected-resource`) and **Bearer API keys** (`tlx_live_...` or `tlx_test_...` from [taxlocus.com](https://taxlocus.com)).

### Claude Code / Claude Desktop / Cursor (Bearer API Key)

```json
{
  "mcpServers": {
    "taxlocus": {
      "type": "http",
      "url": "https://api.taxlocus.com/mcp",
      "headers": {
        "Authorization": "Bearer tlx_live_YOUR_KEY"
      }
    }
  }
}
```

### Interactive OAuth 2.1 (Claude Connectors / MCP Clients with OAuth Discovery)

Point your OAuth-capable MCP client directly at:

```text
https://api.taxlocus.com/mcp
```

The server challenges unauthenticated requests with `401 Unauthorized` and `WWW-Authenticate: Bearer resource_metadata="https://api.taxlocus.com/.well-known/oauth-protected-resource"`, guiding the client through PKCE sign-in.

---

## Available MCP Tools (16)

| Tool | Read-Only | Metered | Description |
|---|---|---|---|
| `lookup_rate` | Yes | Yes | Resolve the combined sales-tax rate, jurisdiction breakdown, taxability flag, sourcing rule, warnings, and primary-source citations for a US address or lat/lng and product category. |
| `check_nexus` | Yes | Yes | Given year-to-date sales and transaction counts, report whether a seller has crossed a state's economic-nexus threshold. |
| `filing_calendar` | Yes | Yes | Return a state's filing frequency and upcoming sales-tax return due dates based on annual tax liability or revenue. |
| `taxability_check` | Yes | Yes | Report whether one or more product categories are taxable in a state, with treatment, confidence, and source citation. |
| `lookup_payroll_rates` | Yes | Yes | Return federal payroll-tax constants (FICA + FUTA), state PIT top-line, state SUTA wage base, local income-tax jurisdictions, and state reciprocity agreements. |
| `get_workpaper` | Yes | No | Fetch the stored request, response, and SHA-256 verification hashes for an earlier `lookup_rate` or cart call (`call_id`). |
| `replay_workpaper` | Yes | No | Re-run a stored `lookup_rate` or cart call against today's corpus and return any differences and dated citations since the original `corpus_version`. |
| `list_entities` | Yes | No | List the account's tracked seller entities (`id`, `name`, `home_state`, `timezone`, and stored business `location`). |
| `create_entity` | No | No | Create a tracked seller entity with optional `home_state`, IANA `timezone`, and business `location`. |
| `update_entity` | No | No | Update a tracked entity's `name`, `home_state`, `timezone`, or business `location`. |
| `record_transaction` | No | No | Append a sale to a tracked entity's ledger (idempotent on `idempotency_key`) and return updated state nexus status. |
| `record_refund` | No | No | Book a full or partial refund against a tracked entity's ledger and return updated state nexus status. |
| `nexus_status` | Yes | No | Return a tracked entity's running economic-nexus status across every state where it has recorded sales, with distance to threshold. |
| `set_registration` | No | No | Record or clear whether a tracked entity is registered to collect sales tax in a state, and return its filing obligation. |
| `set_filing_frequency` | No | No | Record the filing frequency (`monthly`, `quarterly`, `annual`) assigned by a state for a tracked entity. |
| `mark_period_filed` | No | No | Mark or unmark a sales-tax return period (`YYYY-MM`, `YYYY-Qn`, or `YYYY`) as filed for a tracked entity. |

---

## Agent Skills

Each skill lives in its own folder with a `SKILL.md`:

| Skill | Folder | What it does |
|---|---|---|
| **taxlocus-sales-tax** | [`taxlocus-sales-tax/`](./taxlocus-sales-tax/SKILL.md) | Guides Claude in using the **TaxLocus MCP connector** to answer US sales tax questions — rate lookups, economic nexus, product taxability, and filing calendars. Picks the right tool, applies correct sourcing/nexus rules, treats unknown taxability as "verify" (not exempt), surfaces source citations, and enforces the US-only / not-tax-advice guardrails. |
| **taxlocus-payroll-tax** | [`taxlocus-payroll-tax/`](./taxlocus-payroll-tax/SKILL.md) | Guides Claude in answering US **payroll-tax** questions via TaxLocus — federal FICA/FUTA, state PIT and SUTA wage bases, state reciprocity, and local income tax in 8 states (PA, OH, IN, MD, MI, NY, MO, DE). Surfaces coverage gaps honestly for non-covered states, refuses to compute paycheck withholding, and enforces the not-tax-advice / not-withholding-tables guardrail. |
| **taxlocus-api** | [`taxlocus-api/`](./taxlocus-api/SKILL.md) | Guides AI coding assistants in **integrating the TaxLocus REST API / MCP** into a codebase — rate lookups, cart/checkout tax calculation, economic-nexus tracking, taxability, exemption certificates, and state-return worksheets (local schedules are not filled in). |

## Links

- Website: https://taxlocus.com
- MCP server: `https://api.taxlocus.com/mcp` ([docs](https://taxlocus.com/mcp))
- REST API docs: https://api.taxlocus.com/docs
- Open dataset (CC-BY-4.0): https://github.com/Ringzero787/taxlocus-data

## License

Apache-2.0. TaxLocus output is informational only — not tax or legal advice;
verify against the cited source before relying on a value for compliance.
