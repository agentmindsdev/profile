# AgentMinds Reporting Profile (ARP)

> **A profile, not a competing standard.** Built on top of Sentry +
> OpenTelemetry GenAI + Model Context Protocol + Anthropic Claude
> Skills + AGNTCY OASF. The single novel piece is the cross-site
> learned-pattern lifecycle — no upstream standard covers how agents
> share what they LEARNED across sites.

[![License: CC BY 4.0](https://img.shields.io/badge/License-CC_BY_4.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Profile version](https://img.shields.io/badge/profile-v1.2.0-amber)](AGENT_REPORTING_PROFILE.md)
[![Public comment](https://img.shields.io/badge/public%20comment-open-blue)](https://github.com/agentmindsdev/profile/issues)
[![Validate examples](https://github.com/agentmindsdev/profile/actions/workflows/validate-examples.yml/badge.svg)](https://github.com/agentmindsdev/profile/actions/workflows/validate-examples.yml)

## Quick start

A minimal **L0**-conformant report (only `agent` + `report` are required):

```json
{
  "arp_version": "1.2.0",
  "agent": "context_retriever",
  "report": {
    "summary": "Agent failed to retrieve context within timeout",
    "warnings": [
      { "message": "Context window exceeded — retrieval truncated" }
    ]
  }
}
```

Validate it:

```bash
npx ajv-cli@5 validate \
  -s schemas/agent_report.schema.json \
  -d your-report.json \
  --spec=draft2020 --strict=false
```

For payloads at every conformance level (L0 through L4), see
[**examples/**](examples/) — each file is the smallest report that
satisfies its level.

For the full envelope (13 sections, every primitive),
see [**§3 of the spec**](AGENT_REPORTING_PROFILE.md#3-envelope).

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
- **[CHANGELOG.md](CHANGELOG.md)** — version history

## Latest version: v1.2.0 (2026-04-27)

Phase 2 of the v1.1 deepdive plan. **11 additive primitives**,
backwards-compatible with v1.0 / v1.1.x senders:

- `lifecycle_event` envelope (cold-start / wake / scheduled / shutdown)
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
  implementation. ~1,164 patterns flowing through the L3 envelope as
  of 2026-05-04. Public stats at
  [agentminds.dev/showcase](https://agentminds.dev/showcase).

If your project also adopts ARP, open a PR adding it here. The list
is the cleanest signal that the spec is consumed, not just published.

## Lineage

Each row is **pinned** to the upstream version we tested ARP against.
Bumping ARP's version requires re-checking each upstream; if any
absorbed an ARP primitive, see the Reorientation clause below.

| Source (pinned) | What we adopt |
|---|---|
| [Sentry data schemas](https://github.com/getsentry/sentry-data-schemas) — pinned to `main` snapshot 2026-04-01 | Issue lifecycle: `status`, `first_seen`, `last_seen`, `fingerprint`, `level`; `Exception.mechanism.handled` |
| [OpenTelemetry GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — pinned to v1.31.0 | Metric type taxonomy, UCUM units, GenAI attribute names (`gen_ai.system`, `gen_ai.usage.*`, `gen_ai.response.finish_reasons`) |
| [Model Context Protocol](https://modelcontextprotocol.io/specification) — pinned to spec snapshot `2024-11-05` | Transport binding hint, planned Skills-Over-MCP alignment |
| [Anthropic Claude Skills](https://github.com/anthropics/skills) — pinned to repo `main` 2026-04-15 | `SKILL.md` YAML frontmatter |
| [AGNTCY OASF](https://github.com/agntcy/oasf) — pinned to v0.7.0 | Descriptor envelope, metric shape, observability data referencing |

The five upstreams are independent of each other; ARP layers them
into a single envelope. Where they overlap, ARP defers to the more
opinionated upstream (Sentry for issue lifecycle, OTel for telemetry).

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
| Sentry data schemas | Issue lifecycle (`fingerprint`, `status`) | Single-tenant: issues belong to one project; no cross-customer pool primitive |
| OWASP / CIS benchmarks | Community-curated rule packs | Human-curated security baselines, not agent-emitted patterns |
| Datadog / New Relic rule packs | Cross-customer alert rules | Vendor-internal, not an open spec consumers can re-implement |

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

- **Spec feedback** — open an
  [`[ARP-SPEC]` issue](https://github.com/agentmindsdev/profile/issues/new?template=arp-spec-feedback.md)
- **Upstream version drift** — open a
  [`[LINEAGE]` issue](https://github.com/agentmindsdev/profile/issues/new?template=lineage-update.md)
- **Pull requests** — typo / clarity / example fixes go straight to
  PR; schema changes need an issue first
- **External standards engagement** — if you sit on the MCP working
  group, OTel GenAI SIG, AGNTCY observability WG, or related body,
  we'd love to hear how ARP can defer to your work

## License

This profile is released under
**[CC-BY-4.0](https://creativecommons.org/licenses/by/4.0/)**.
Reuse and adapt freely; attribution to AgentMinds appreciated. See
[LICENSE](LICENSE) for the full notice.

## Suggested citation

> AgentMinds (2026). *AgentMinds Reporting Profile v1.2*. CC-BY-4.0.
> https://github.com/agentmindsdev/profile
