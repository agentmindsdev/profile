# AgentMinds Reporting Profile (ARP)

> **A profile, not a competing standard.** Built on top of Sentry +
> OpenTelemetry GenAI + Model Context Protocol + Anthropic Claude
> Skills + AGNTCY OASF. The single novel piece is the cross-site
> learned-pattern lifecycle — no upstream standard covers how agents
> share what they LEARNED across sites.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Profile version](https://img.shields.io/badge/profile-v1.1.0-amber)](AGENT_REPORTING_PROFILE.md)
[![Public comment](https://img.shields.io/badge/public%20comment-open-blue)](https://github.com/agentmindsdev/profile/issues)

## What this is

ARP is AgentMinds' documented adoption of five existing agent /
observability standards, plus one extension that fills a real gap.
It is the wire format AgentMinds uses today, written down so external
integrators don't have to reverse-engineer HTTP traffic.

If you are choosing between ARP and a real standard like
OpenTelemetry GenAI or MCP, **follow the standard.** ARP is a
profile of those standards optimized for AgentMinds' delivery
surface — it is not an anchor for your own architecture.

## Read the spec

- **[AGENT_REPORTING_PROFILE.md](AGENT_REPORTING_PROFILE.md)** — full
  specification (13 sections, ~600 lines)
- **[schemas/agent_report.schema.json](schemas/agent_report.schema.json)** —
  machine-readable JSON Schema (draft 2020-12)
- **[CHANGELOG.md](CHANGELOG.md)** — version history

## Live at

- Landing page (with examples and badges):
  **[agentminds.dev/profile](https://agentminds.dev/profile)**
- Reference implementation: closed-source AgentMinds collector
  ([agentminds.dev](https://agentminds.dev) — pointers to the
  fingerprint helper, lifecycle migration, and filter glue are in
  §11 of the spec)

## Lineage

| Source | What we adopt |
|---|---|
| [Sentry data schemas](https://github.com/getsentry/sentry-data-schemas) | Issue lifecycle: `status`, `first_seen`, `last_seen`, `fingerprint`, `level` |
| [OpenTelemetry GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) | Metric type taxonomy, UCUM units, GenAI attribute names |
| [Model Context Protocol](https://modelcontextprotocol.io/specification) | Transport binding hint, planned Skills-Over-MCP alignment |
| [Anthropic Claude Skills](https://github.com/anthropics/skills) | `SKILL.md` YAML frontmatter |
| [AGNTCY OASF](https://github.com/agntcy/oasf) | Descriptor envelope, metric shape, observability data referencing |

The five upstreams are independent of each other; ARP layers them
into a single envelope. Where they overlap, ARP defers to the more
opinionated upstream (Sentry for issue lifecycle, OTel for telemetry).

## The single thing we own

**§4.1 Pattern object** — cross-site learned-pattern lifecycle.

AGNTCY/OASF describe agents; nobody has standardized how agents
share what they LEARNED across sites. Every pattern carries a stable
fingerprint, a status (`active` / `solved` / `obsolete`),
`first_seen` / `last_seen`, `applicable_site_types`, and a
confidence score.

That single primitive is what enables AgentMinds' cross-site
recommendation pool, and it is the only piece of ARP not directly
inherited from the lineage.

## Conformance levels

| Level | Required features |
|---|---|
| **L0** | Envelope + `report.summary` + `warnings[].message` |
| **L1** | L0 + `severity` + `category` + `status` per warning |
| **L2** | L1 + `fingerprint` + `first_seen` + `last_seen` + `count` |
| **L3** | L2 + `memory.learned_patterns` with §4.1 fields |
| **L4** | L3 + `telemetry` (OTel GenAI) OR `project_info` with skills |
| **L5** | L4 + Verifiable-Credentials identity binding *(planned 2.0)* |

AgentMinds' collector requires **L0**, recommends **L2**; cross-site
recommendation filtering unlocks at **L2+**.

## Reorientation clause

If any upstream standard absorbs the pattern-lifecycle primitive
(§4.1) into its canonical spec, AgentMinds MUST publish ARP MAJOR+1
within 30 days that defers the primitive to the upstream definition.
We are explicitly the fastest adopter of the winning standard, never
the spec owner. See §7 of the spec for the formal versioning rule.

## Contributing

Public comment, suggestions, and PRs are welcome.

- **Discussions** — open an issue with the label `arp-spec`
- **Pull requests** — for typo / clarity fixes, send a PR; for
  schema changes, open an issue first
- **External standards engagement** — if you sit on the MCP working
  group, OTel GenAI SIG, AGNTCY observability WG, or related body,
  we'd love to hear how ARP can defer to your work

## License

This profile is released under
**[CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)**.
Reuse and adapt freely; attribution to AgentMinds appreciated. See
[LICENSE](LICENSE) for the full notice.

## Suggested citation

> AgentMinds (2026). *AgentMinds Reporting Profile v1.0*. CC-BY-4.0.
> https://github.com/agentmindsdev/profile
