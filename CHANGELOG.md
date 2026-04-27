# Changelog

All notable changes to the AgentMinds Reporting Profile (ARP) are
documented here.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this profile adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.2.0] — 2026-04-27

Phase 2 of the v1.1 deepdive plan. 11 additive primitives covering
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
