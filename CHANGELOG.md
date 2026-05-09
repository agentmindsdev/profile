# Changelog

All notable changes to the AgentMinds Reporting Profile (ARP) are
documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this profile adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.3.0] — 2026-05-08

Minor release over v1.2.2. Single normative addition (B2: blast-
radius classification on the Pattern object) with a deliberate
null-as-conservative-default contract for collectors, plus three
informative collector-side delivery additions (B1 / A3 / F0)
documented as worked examples for implementers.

### Added

- **§4.1.1 `Pattern.reversibility` (OPTIONAL)** — closed-enum field
  classifying the blast radius of a pattern's recommended action.
  Values: `safe_config`, `reversible_code`, `risky_infra`,
  `security_critical`, `null`. Senders MAY emit; collectors MUST
  treat absent or null as `risky_infra`-equivalent for
  auto-application gating. Conservative-default-on-purpose: a
  pre-classification null state must never expand the auto-apply
  surface.
- **`$defs.Reversibility`** in JSON Schema with the closed enum +
  null. Validates all four named values, accepts null and absent,
  rejects unknown strings.
- **§7.1 v1.2.x ↔ v1.3 compatibility entries** — bidirectional
  forward/backward compat verified. Old collectors ignore the new
  field; old senders are gated as if reversibility were null
  (= risky_infra).

### Notes — no bulk backfill

Implementations may leave existing patterns with null `reversibility`
indefinitely. Bulk LLM reclassification of legacy patterns is **NOT**
recommended:

- A 5-15% mis-classification rate on a destructive pattern (DB drop
  stamped as `safe_config`) would silently expand the auto-apply
  surface across consuming sites — exactly the failure mode the
  field is meant to prevent.
- The conservative null-default makes legacy patterns safe by
  design; pilot-driven manual classification when specific use
  cases emerge is the recommended path.

The reference harvester (`agentmindsdev/agentminds`,
`master-agent-system/central_agents/external_pattern_harvester.py`)
emits `reversibility` only when the action's category is
unambiguous from the trigger and action text. Ambiguous patterns
omit the field rather than guess.

### Fixed

- **§11.1 wording** — earlier draft said
  `top_production_observed` patterns "carry `status ==
  'production_observed'`", which conflicts with the §4.1
  PatternStatus enum (`active | solved | obsolete`). Reworded to
  describe the collector-internal `production_signal_tier` field
  explicitly and note that the tagging mechanism is collector-
  specific, not normative ARP.

### Reference collector additions (informative, not normative)

These describe the `agentmindsdev/agentminds` collector's
delivery surface; documented in §11.1 as worked examples.
Other ARP-conformant collectors are free to deliver differently.

- **B1 — Split top-rules arrays.** `/personalized-rules` returns
  `top_production_observed` and `top_documented` as distinct
  arrays. The mixed `top_rules` array is kept for backwards
  compatibility, slated for v1.4 removal.
- **A3 — `negative_evidence` array.** Filtered patterns are now
  surfaced with explicit reason enum
  (`site_type_mismatch_narrow` / `already_applied` /
  `non_english_content` / `low_confidence` /
  `bundle_size_exceeded`). Consumers can render these as
  "considered but excluded" UX, instead of silently dropping
  candidates.
- **F0 — Non-English content filter.** The reference collector
  filters non-English pattern submissions before scoring;
  rejected candidates surface in `negative_evidence` with reason
  `non_english_content`.

### Changed

- `spec_version` field across `/sync/me`, `/sync/personalized-
  rules`, and `/sync/pool-stats` now reports `"ARP-1.3.0"`. Pre-
  2026-05-09 deployments returned `"ARP-1.1"` until the drift was
  reconciled in `agentminds@2a2d5f0`.

### Implementation status (as of release)

| Surface | Version | Live |
|---|---|---|
| Spec (this repo) | `v1.3.0` tag | ✓ |
| Backend collector | `agentminds@a8c23b3+` (api.agentminds.dev) | ✓ |
| MCP server | `agentminds-mcp@1.3.2+` (npm) | ✓ |
| Node SDK | `@agentmindsdev/node@0.4.0+` (npm) | ✓ |
| Python SDK | `agentminds@0.5.0+` (PyPI) | ✓ |

### Lineage

- averageuser612 r/LangChain feedback point #4 (2026-05-04).
- Conceptual parallel: blast-radius labels in deployment systems
  (Liquibase safe/risky migration tags, SemVer breaking-change
  severity, AWS CloudFormation update-replace policy).

---

## [1.2.2] — 2026-05-05

Patch release over v1.2.1. Single informative addition documenting
the reference collector's tier-segregated delivery shape. No
normative changes; v1.2.1 senders and collectors remain conformant.

### Added

- **§11.1 Reference collector — tier-segregated delivery
  (informative)** — Worked example describing how the reference
  AgentMinds collector surfaces Pattern `status` (§4.1) in its
  `/personalized-rules` response: `top_production_observed` and
  `top_documented` as separate arrays, with `top_rules` (mixed)
  retained for backward compatibility and deprecated for collector
  v1.4 removal. Documented for other implementers facing the same
  UI/API surface question.

### Notes

- This is a **profile-internal informative** addition. Collectors
  choose how to surface tiers in their delivery APIs; the Pattern
  object's `status` field remains the canonical tier indicator.
- Backend implementation: agentminds commit `96c720e` (2026-05-05).

---

## [1.2.1] — 2026-05-05

Patch release over v1.2.0. Five cross-standard alias / bridge / discovery
primitives added so ARP plays cleanly with Sentry SDKs, LangSmith
feedback, OTel GenAI evaluation spans, AGNTCY OASF taxonomies, and
cross-collector discovery tooling. All additive — v1.2.0 senders work
unchanged against v1.2.1 collectors.

### Added

- **§3.3 Sentry `event.fingerprint` alias** — Collectors SHOULD accept
  the JSON pointer alias `event.fingerprint` (Sentry's envelope field
  name) and treat it identically to the canonical
  `Warning.fingerprint`. Server-side normalization MUST emit only the
  canonical name on outbound payloads. Lets Sentry SDK ingest land
  without translation.
- **§3.4 LangSmith `confidence` ↔ `feedback.score` alias** — When
  `Recommendation.confidence` is a 0..1 numeric, it is wire-compatible
  with LangSmith's `feedback.score` primitive. Collectors building an
  `arp-langsmith-bridge` MAY map between the two without lossy
  translation. Richer eval shapes (categorical / boolean / comparative)
  continue to use `Score` (§3.5).
- **§3.4.1 `gen_ai.evaluation.result` → §4.1 Pattern bridge** —
  Collectors MAY auto-promote OTel GenAI `gen_ai.evaluation.result`
  attributes into ARP §4.1 Patterns once observed ≥ N times across
  distinct sites (default N=3). Lets observability-first stacks
  (Langfuse, Arize, OpenInference adopters) surface cross-site
  patterns without wiring ARP push directly. Spec text is informative
  as of v1.2.1; reference implementation lands in a future ARP
  collector.
- **§6.1 OASF 90000-99999 range reservation** — ARP reserves AGNTCY
  OASF taxonomy IDs **90000-99999** as the `agentminds.*` extension
  namespace per the §1.5 normative reference. Vendors building skills
  against ARP MAY allocate their own number from this range without
  colliding with upstream canonical IDs (1xxx, 5xxx, etc.). Sub-range
  reservations file a PR against `RESERVED_RANGES.md` (forthcoming).
- **§8.1 `/.well-known/arp.json` self-description (NORMATIVE)** —
  Every ARP-conformant collector MUST publish a self-description at
  `/.well-known/arp.json` carrying `arp_version`, `conformance_level`,
  `endpoints`, `auth_schemes`, `spec_url`, `extensions_advertised`,
  and `links` to companion `agent_card.json` / `oasf_descriptor.json`
  / `mcp_server.json`. Cross-collector tools (adapters, analytics,
  dashboards) SHOULD probe this URL first to discover the collector's
  surface before issuing typed requests. ARP equivalent of A2A's
  `agent-card.json` or AGNTCY's `oasf-descriptor.json`.

### Schema / Spec

- **No additions to `agent_report.schema.json`.** §8.1 introduces a
  separate `/.well-known/arp.json` document (not part of the report
  envelope). The aliases (§3.3, §3.4) are collector-side normalization
  rules; the bridge (§3.4.1) is informative; the OASF reservation
  (§6.1) is a number range.
- **Spec §1.5 Normative references** updated to include v1.2.1 line
  items.
- **Existing v1.2.0 fields unchanged.** New v1.2.1 primitives are
  alias notes, informative bridges, namespace reservations, or
  out-of-band discovery contracts — none expand the report envelope.

## [1.2.0] — 2026-04-27

Phase 2 of the v1.1 deepdive plan. Additive primitives covering
multi-agent observability, prompt provenance, hibernation lifecycle,
exception triage, hierarchical telemetry, and AGNTCY OASF skill
taxonomy alignment. All backwards-compat — v1.0/v1.1.x senders work
unchanged against v1.2 collectors.

### Added

- **§2.2 `lifecycle_event` envelope** — `cold_start | wake | scheduled
  | shutdown | running` for hibernate-aware runtimes (Cloudflare DO,
  AWS Lambda). OPTIONAL — non-hibernating agents omit.
- **§3.7 `Exception` evidence shape** — Sentry-aligned, `mechanism.handled`
  distinguishes caught vs uncaught. Goes inside `Warning.evidence.exception`.
- **§5.3 `dotted_order` on `TelemetrySpan`** — LangSmith-compatible
  single-string hierarchical sort (`<RFC3339>Z<uuid>.<RFC3339>Z<uuid>...`).
  Lets clients sort spans without parent_id JOINs.
- **§6.2 `Handoff` primitive + `Report.handoffs[]`** — multi-agent
  delegation events. Lineage: OpenAI Agents SDK Handoff class.
  Required: from_agent + to_agent. Adds `Report.handoffs[]` array.
- **§6.3 `PromptManifest` primitive + `ProjectInfo.prompts[]`** — prompt
  provenance (LangSmith Prompt Hub + OpenInference `prompt.*` lineage).
- **§6.1 `oasf_skill_ids[]`** — cross-reference to AGNTCY OASF skill
  taxonomy IDs (e.g. `[1106]` for "Cross-Site Pattern Confidence" under
  Category 11). LF AI & Data alignment.
- **§6.1 `agentskills_io_canonical_name`** — optional cross-link to
  agentskills.io marketplace.
- **§6.1 `execution_tier`** — Cloudflare Agents 5-tier execution ladder
  + ARP `in_process` extension (`workspace`, `dynamic_worker`, `npm`,
  `browser`, `sandbox`, `in_process`).
- **§6.4 `tech_stack.mcp_server_exposed` + `mcp_clients_consumed[]`** —
  MCP role disambiguation. Closes the "is this site exposing or consuming
  MCP?" gap.

### Schema

- New `$defs`:
  - `Handoff` (required: from_agent, to_agent)
  - `PromptManifest` (free-form provenance)
  - `Exception` (mechanism.handled boolean for triage)
- `lifecycle_event` top-level enum property (5 values + OPTIONAL)
- `TelemetrySpan.dotted_order` string field
- `Report.handoffs[]` array
- `ProjectInfo.prompts[]` array
- `ProjectInfo.tech_stack` typed properties (`mcp_server_exposed`,
  `mcp_clients_consumed`)
- `SkillManifest`: `oasf_skill_ids` (int array), `agentskills_io_canonical_name`,
  `execution_tier` (6-value enum)

### Tests

- 19 new schema validator cases — total: 62 → 81 schema tests
- Coverage: enum-validation paths for lifecycle_event, execution_tier;
  Handoff required-field enforcement; oasf_skill_ids type validation;
  Exception evidence shape; dotted_order on spans; v1.0/1.1.x backwards-
  compat round-trips on all new optional fields.

## [1.1.1] — 2026-04-27

Phase 1B of the v1.1 deepdive plan. Adds the remaining 3 must-ship
items: Breadcrumbs, agentskills.io frontmatter alignment,
OpenInference normative reference. All additive, all backwards-compat
with v1.0 and v1.1.0.

### Added

- **§3.6 `Breadcrumb` trail array on Warning.** Sentry-aligned
  timestamped events leading up to a warning. Required: `timestamp`
  + `type` (8-value enum: navigation/http/error/info/query/ui/user/
  default). Critical for cascade-failure debugging. Cap ~50/warning.
- **§5.1 OpenInference v1 normative reference.** ARP §5 telemetry
  now formally adopts OpenInference's attribute namespace by
  reference. Spec text only (Apache-2.0); Phoenix runtime explicitly
  excluded (ELv2 + US patent claims).
- **§5.2 `TelemetrySpan` typed shape.** OPTIONAL upgrade from v1.0
  free-form span dicts. New optional fields: `kind` (10-value
  OpenInference enum), `subtype` (8-value OpenAI Agents enum),
  `trace_id`, `span_id`, `parent_span_id`, `start_time`, `end_time`,
  `status`, `attributes`, `events`, `_meta`. v1.0 free-form spans
  still validate.
- **§6.1 SkillManifest expansion.** agentskills.io 6 open-spec
  frontmatter fields adopted verbatim: `name`, `description`,
  `version`, `license` (SPDX), `metadata` (free-form), `compatibility`.
  Plus `auto_invocable: bool` — polarity-flipped from Anthropic
  Skills' `disable-model-invocation`. Default `false` (opt-in is
  safer; collectors translate inverted upstream form on ingest).

### Schema

- New `$defs`:
  - `Breadcrumb` (required: timestamp, type)
  - `OpenInferenceSpanKind` (10-value enum)
  - `OpenAISpanSubtype` (8-value enum)
  - `TelemetrySpan` (typed shape, all fields optional)
- `Warning.breadcrumbs[]` references `Breadcrumb`
- `Telemetry.spans[]` now references `TelemetrySpan` (was free-form
  `{type:object}` — additive: TelemetrySpan also accepts
  additionalProperties)
- `SkillManifest.metadata` (object) and `SkillManifest.auto_invocable`
  (boolean) added

### Tests

- 18 new schema validator cases — total: 44 → 62 schema tests
- Coverage: breadcrumb attach + reject paths, agentskills.io fields
  acceptance + auto_invocable boolean enforcement, TelemetrySpan
  typed validation incl. enum-rejection paths and v1.0 free-form
  backwards-compat

### What this unlocks

- **Sentry SDK adapter:** ARP collectors can now translate Sentry
  breadcrumb arrays directly without lossy normalization
- **OpenInference adopters** (Arize, Langfuse, anyone using the
  attribute namespace): direct interop, no field renames
- **Anthropic / agentskills.io ecosystem**: skill metadata round-trips
  without lossy translation; the polarity flip on `auto_invocable`
  is a small clarity win

## [1.1.0] — 2026-04-27

Additive backwards-compat release. v1.0 senders work unchanged
against v1.1 collectors. v1.1 senders work against v1.0 collectors
because all new fields are OPTIONAL and unknown-key-safe.

Driven by `STANDARDS_DEEPDIVE_2026_04_27/ULTIMATE_PROFILE_RECOMMENDATION.md`
(12-target deep-dive synthesis). Phase 1A scope (4/7 of the must-ship
tier from the report).

### Added

- **§3.5 `Score` primitive.** Typed evaluation outcomes (`numeric` /
  `categorical` / `boolean`) attached to warnings, patterns, or the
  report as a whole. Closed `data_type` enum to prevent fragmentation.
  Lineage: Langfuse Score model + LangSmith Feedback. Unlocks eval
  pipeline interop.
- **§2.1 `_meta` reverse-DNS extension envelope.** MCP-aligned
  vendor extension convention. Available on the envelope and on every
  `$def` (Warning, Recommendation, Pattern, Score, Memory,
  SkillManifest, ProjectInfo). Reserved second-level namespaces:
  `agentminds.*`, `arp.*`. Other implementers use first-come-first-
  served reverse-DNS.
- **§3.3 fingerprint accepts array form.** Sentry-aligned
  `string[]` form with `{{ default }}` placeholder support coexists
  with the canonical lowercase-hex SHA-256 single-string form. Per
  payload, senders pick one; v1.0 senders unaffected.
- **§2 envelope additions:** `session_id`, `user_id`,
  `conversation_id`. Multi-turn agent flow correlation; aliases of
  LangSmith Threads + OpenInference `session.id` / `user.id` + OTel
  `gen_ai.conversation.id` semantics.
- **§7.1 version compatibility matrix.** Explicit table covering
  v1.0↔v1.1 cross-version interop guarantees.

### Spec doc updates

- §10 worked example now demonstrates `_meta`, `session_id`,
  `scores[]`, and the fingerprint array form side-by-side.
- §7 versioning text now explicitly identifies v1.1 as a MINOR
  (additive) bump and gives senders a clear upgrade path.

### Schema

- `$id` and `title` reference v1.1.
- `Fingerprint` is now a `oneOf` between the canonical string form
  and the Sentry-aligned array form.
- New `Score` `$def` with closed `data_type` enum.
- `_meta` field added to top-level + `Report` + `Warning` +
  `Recommendation` + `Pattern` + `Memory` + `ProjectInfo` +
  `SkillManifest` + `Score`.
- `session_id`, `user_id`, `conversation_id` top-level properties.

### Tests

- 24 new test cases covering each addition, including v1.0
  backwards-compat round-trips and rejection of malformed inputs
  (empty fingerprint array, invalid `Score.data_type`, empty
  `session_id` string).
- Total schema validator tests: 20 → 44.

### Reorientation watch

Per §7 reorientation clause and the deep-dive Section 6, any of the
following upstream events triggers ARP MAJOR+1 within 30 days:

- MCP Skills Over MCP WG ships `notifications/agent/learned`
- AGNTCY OASF v2 adds `learned_pattern` object class
- Sentry adds Pattern / cross-environment-fingerprint primitive
- OpenInference SemConv adds `pattern.*` attributes

A `agentminds-standards-watcher` bot (Phase 4 of the v1.1 plan)
will monitor these upstreams weekly.

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
