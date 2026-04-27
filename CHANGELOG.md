# Changelog

All notable changes to the AgentMinds Reporting Profile (ARP) are
documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this profile adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] — 2026-04-27

### Added

- **Initial release of ARP v1.0.** Published as a profile, not a
  competing standard. See README and §"Strategic position" of the
  spec for the load-bearing positioning rationale.
- **13-section specification** covering: envelope (§2), report
  shape (§3), warning lifecycle (§3.3), recommendation object
  (§3.4), memory + learned-patterns (§4), OTel-compatible
  telemetry (§5), project-info with skills (§6), versioning
  rules (§7), transport bindings (§8), conformance levels L0–L5
  (§9), worked example (§10), implementer notes (§11), public
  comment process (§12), and a profile-vs-standard FAQ (§13).
- **JSON Schema (draft 2020-12)** at `schemas/agent_report.schema.json`
  with full coverage of every required and optional field, including
  enum constraints for status / severity / category / level and
  pattern validation for fingerprint (lowercase hex SHA-256, 64
  chars).
- **Five lineages cited** — Sentry data schemas, OpenTelemetry GenAI,
  Model Context Protocol, Anthropic Claude Skills, AGNTCY OASF.
- **One novel extension** — §4.1 Pattern object, the cross-site
  learned-pattern lifecycle. No upstream standard covers this;
  see Strategic Position in the spec for the reorientation clause
  that activates when one does.
- **Reference implementation pointers** — fingerprint normalization
  (`master-agent-system/sync/fingerprint.py` in the agentmindsdev/
  agentminds repo), Postgres lifecycle migration
  (`sync/migrations/002_warning_lifecycle.sql`), filter glue
  (`sync/recommendation_filters.py`), and idempotent backfill
  script (`sync/backfill_warning_lifecycle.py`).
- **Conformance levels L0–L5** — collectors and senders self-declare
  level. Current AgentMinds collector requires L0, recommends L2;
  cross-site recommendation filtering unlocks at L2+.

### Strategic position

This release explicitly disclaims any intent to compete with the
upstream standards. ARP is AgentMinds' documented adoption of those
standards, plus one extension. When any upstream standard absorbs
the pattern-lifecycle primitive, ARP MAJOR+1 must defer to the
upstream definition within 30 days. See README for the full clause.

### Known follow-ups (planned for 1.1)

- Add Protobuf schema export (`schemas/agent_report.proto`) for
  gRPC-over-SLIM bindings.
- Cross-reference OTel GenAI agent-spans semantic conventions when
  they exit experimental status.
- Align with the MCP Skills working group output once the
  "Skills Over MCP" SEP is approved.
- Consider OASF Category 11 (Evaluation & Monitoring) → Skill 1106
  ("Cross-Site Pattern Confidence") submission.

### Reserved (planned for 2.0)

- L5 conformance — Verifiable-Credentials identity binding per
  AGNTCY OASF Identity Service.
- Migration notes from any upstream standard that may absorb §4.1.
