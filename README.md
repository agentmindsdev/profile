# AgentMinds Reporting Profile (ARP)

> **A profile, not a competing standard.** Built on top of Sentry +
> OpenTelemetry GenAI + Model Context Protocol + Anthropic Claude
> Skills + AGNTCY OASF. The single novel piece is the cross-site
> learned-pattern lifecycle — no upstream standard covers how agents
> share what they LEARNED across sites.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Profile version](https://img.shields.io/badge/profile-v1.2.1-blue)](AGENT_REPORTING_PROFILE.md)
[![Public comment](https://img.shields.io/badge/public%20comment-open-blue)](https://github.com/agentmindsdev/profile/issues)
[![Validate examples](https://github.com/agentmindsdev/profile/actions/workflows/validate-examples.yml/badge.svg)](https://github.com/agentmindsdev/profile/actions/workflows/validate-examples.yml)

## Quick start

A minimal **L0**-conformant report (only `agent` + `report` are required):

```json
{
  "arp_version": "1.2.1",
  "agent": "context_retriever",
  "report": {
    "summary": "Agent failed to retrieve context within timeout",
    "warnings": [
      { "message": "Context window exceeded — retrieval truncated" }
    ]
  }
}
```

Validate it (no clone required — fetches the schema from this repo):

```bash
npx ajv-cli@5 validate \
  -s https://raw.githubusercontent.com/agentmindsdev/profile/main/schemas/agent_report.schema.json \
  -d your-report.json \
  --spec=draft2020 --strict=false
```

Or, if you've cloned the repo, use the local schema path
(`-s schemas/agent_report.schema.json`).

For payloads at every conformance level (L0 through L4), see
[**examples/**](examples/) — each file is the smallest report that
satisfies its level.

For the full envelope and every primitive, see the
[**13-section spec**](AGENT_REPORTING_PROFILE.md).

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
  specification (13 sections)
- **[schemas/agent_report.schema.json](schemas/agent_report.schema.json)** —
  machine-readable JSON Schema (draft 2020-12)
- **[examples/](examples/)** — concrete payloads for L0 / L2 / L3 / L4
- **[eval-set/v0-strawman.md](eval-set/v0-strawman.md)** — Eval set v0
  strawman (open for delta; not normative until v1.0)
- **[CHANGELOG.md](CHANGELOG.md)** — version history

## Latest version: v1.2.1

Phase 2 of the v1.1 deepdive plan, additive over v1.0 / v1.1.x:

- `lifecycle_event` envelope (cold_start / wake / scheduled / shutdown / running)
- `Exception` evidence shape (Sentry-aligned `mechanism.handled`)
- `dotted_order` on telemetry spans (LangSmith-compatible)
- `Handoff` primitive for multi-agent delegation events
- `PromptManifest` primitive for prompt provenance
- AGNTCY OASF skill ID cross-references + `agentskills.io` linkage
- Cloudflare 5-tier `execution_tier` enum + ARP `in_process` extension
- MCP role disambiguation (`mcp_server_exposed` vs `mcp_clients_consumed[]`)

Full notes in [CHANGELOG.md](CHANGELOG.md).

## Live at

- Landing page (with examples and badges):
  **[agentminds.dev/profile](https://agentminds.dev/profile)**
- Reference implementation: closed-source AgentMinds collector
  ([agentminds.dev](https://agentminds.dev) — pointers to the
  fingerprint helper, lifecycle migration, and filter glue are in
  §11 of the spec)

## Used by

- **[agentminds.dev](https://agentminds.dev)** — reference
  implementation. A growing pool of patterns flows through the L3
  envelope; live counters at
  [agentminds.dev/showcase](https://agentminds.dev/showcase) and
  [agentminds.dev/status](https://agentminds.dev/status).

Adopters welcome — open a PR to add your project.

## Lineage

Each row cites the upstream version ARP defers to. ARP version bumps
require re-checking each upstream; if any absorbs an ARP primitive,
see the Reorientation clause below.

| Source | Adopted from version | What we adopt |
|---|---|---|
| [Sentry data schemas](https://github.com/getsentry/sentry-data-schemas) | v7+ | Issue lifecycle (`status`, `fingerprint`, `breadcrumbs`, `mechanism.handled`) |
| [OpenTelemetry GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/) | 1.27+ (later versions compatible — namespace is additive) | `gen_ai.*` attribute namespace, span shapes, GenAI events, metric instruments + UCUM units |
| [Model Context Protocol](https://modelcontextprotocol.io/specification) | spec snapshot 2025-11-25 | `_meta` reverse-DNS extension envelope, JSON-RPC 2.0 transport binding, planned Skills-Over-MCP |
| [Anthropic Claude Skills](https://github.com/anthropics/skills) + [agentskills.io](https://agentskills.io) | open spec | `SKILL.md` YAML frontmatter |
| [AGNTCY OASF](https://github.com/agntcy/oasf) | v1+ | Descriptor envelope, metric shape, observability data referencing, skill taxonomy IDs |

The five upstreams are independent; ARP layers them into a single
envelope. Where they overlap, ARP defers to the more opinionated
upstream (Sentry for issue lifecycle, OTel for telemetry).

For the full set of normative references (including OpenInference,
Langfuse Score, and OpenAI Agents SDK), see
[§1.2 of the spec](AGENT_REPORTING_PROFILE.md#12-normative-references-v121).

When an upstream releases a new version, open a
[`[LINEAGE]` issue](https://github.com/agentmindsdev/profile/issues/new?template=lineage-update.md)
flagging which ARP primitives are affected.

## The single thing we own

**§4.1 Pattern object** — cross-site learned-pattern lifecycle.

Every pattern carries a stable fingerprint, a status (`active` /
`solved` / `obsolete`), `first_seen` / `last_seen`,
`applicable_site_types`, and a confidence score. That single
primitive is what enables AgentMinds' cross-site recommendation pool,
and it is the only piece of ARP not directly inherited from the
lineage.

### How we know nobody else covers this

A 2026-04 survey of adjacent specs found cross-site pattern lifecycle
absent in each:

| Spec | Closest primitive | Why it doesn't cover §4.1 |
|---|---|---|
| OpenTelemetry GenAI | `gen_ai.*` attributes on spans | Per-call telemetry, not durable pattern objects with cross-site lifecycle |
| Model Context Protocol | Tool / resource registries | Tools and resources, not failure-pattern catalogs |
| AGNTCY OASF | Agent descriptors + skill taxonomy | Describes agents and skills, not what those agents *learned* across sites |
| Sentry data schemas | Issue lifecycle (`fingerprint`, `status`) | Single-tenant: issues belong to one project within one organization; no cross-customer pool primitive |
| OWASP / CIS benchmarks | Human-curated static rule packs (security baselines / CWE checks) | Static rules, not agent-emitted patterns with per-fingerprint lifecycle and cross-site evidence aggregation |
| Datadog Watchdog / New Relic AI | Vendor-internal anomaly detection | Proprietary engine, organization-scoped only, no open schema for re-implementation |

If you know of a spec that does cover this, please open an
[`[ARP-SPEC]` issue](https://github.com/agentmindsdev/profile/issues/new?template=arp-spec-feedback.md)
— ARP should defer to it.

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

A worked example for each level lives in [examples/](examples/).

## Reorientation clause

If any upstream standard absorbs the pattern-lifecycle primitive
(§4.1) into its canonical spec, AgentMinds MUST publish ARP MAJOR+1
within 30 days that defers the primitive to the upstream definition.
We are explicitly the fastest adopter of the winning standard, never
the spec owner. See §7 of the spec for the formal versioning rule.

## Contributing

Public comment, suggestions, and PRs welcome.

- **General questions / use-case stories** — open a
  [Discussion](https://github.com/agentmindsdev/profile/discussions).
  This is the right kind of door for "how would I use ARP for X?" or
  "is anyone else doing Y?" — not formal enough for an issue.
- **Spec feedback** — open an
  [`[ARP-SPEC]` issue](https://github.com/agentmindsdev/profile/issues/new?template=arp-spec-feedback.md)
  for clarity / schema / new-primitive proposals.
- **Upstream version drift** — open a
  [`[LINEAGE]` issue](https://github.com/agentmindsdev/profile/issues/new?template=lineage-update.md)
  when an upstream we defer to (Sentry / OTel / MCP / Skills / OASF)
  releases a new version that may affect ARP.
- **Pull requests** — typo / clarity / example fixes go straight to
  PR; schema changes need an issue first.
- **External standards engagement** — if you sit on the MCP working
  group, OTel GenAI SIG, AGNTCY observability WG, or related body,
  we'd love to hear how ARP can defer to your work.

## License

This profile is released under
**[CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)**.
Reuse and adapt freely; attribution to AgentMinds appreciated. See
[LICENSE](LICENSE) for the full notice.

## Suggested citation

> AgentMinds (2026). *AgentMinds Reporting Profile v1.2.1*. CC-BY-4.0.
> https://github.com/agentmindsdev/profile
