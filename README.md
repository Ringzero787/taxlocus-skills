# TaxLocus Skills

[Agent Skills](https://docs.anthropic.com/en/docs/agents-and-tools/agent-skills)
for working with **US sales tax** via [TaxLocus](https://taxlocus.com) — the
MCP connector (`https://api.taxlocus.com/mcp`) and the REST API
(`https://api.taxlocus.com`).

Each skill lives in its own folder with a `SKILL.md`.

| Skill | Folder | What it does |
|---|---|---|
| **taxlocus-sales-tax** | [`taxlocus-sales-tax/`](./taxlocus-sales-tax/SKILL.md) | Guides Claude in using the **TaxLocus MCP connector** to answer US sales tax questions — rate lookups, economic nexus, product taxability, and filing calendars. Picks the right tool, applies correct sourcing/nexus rules, treats unknown taxability as "verify" (not exempt), surfaces source citations, and enforces the US-only / not-tax-advice guardrails. |
| **taxlocus-api** | [`taxlocus-api/`](./taxlocus-api/SKILL.md) | Guides AI coding assistants in **integrating the TaxLocus REST API / MCP** into a codebase — rate lookups, cart/checkout tax calculation, economic-nexus tracking, taxability, exemption certificates, and filing-ready reports. |

## Links

- Website: https://taxlocus.com
- MCP server: `https://api.taxlocus.com/mcp` ([docs](https://taxlocus.com/mcp))
- REST API docs: https://api.taxlocus.com/docs
- Open dataset (CC-BY-4.0): https://github.com/Ringzero787/taxlocus-data

## License

Apache-2.0. TaxLocus output is informational only — not tax or legal advice;
verify against the cited source before relying on a value for compliance.
