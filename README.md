# TaxLocus Skills

[Agent Skills](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills)
for working with **US sales tax** via [TaxLocus](https://taxlocus.com) — the
MCP connector (`https://api.taxlocus.com/mcp`) and the REST API
(`https://api.taxlocus.com`).

Each skill lives in its own folder with a `SKILL.md`.

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
