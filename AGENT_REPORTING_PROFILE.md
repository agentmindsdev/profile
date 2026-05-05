# AgentMinds Reporting Profile (ARP)

**Version:** 1.2.1
**Status:** Internal profile — open for public comment
**Maintainer:** AgentMinds (`api.agentminds.dev`)
**License:** CC-BY-4.0

> **What's new in 1.2.0** (2026-04-27, additive over 1.1.1) — Phase 2 of the deepdive plan:
>
> - **§2.2 `lifecycle_event` envelope** — `cold_start` / `wake` / `scheduled` / `shutdown` / `running` for hibernate-aware runtimes (Cloudflare DO, Lambda)
> - **§3.7 `Exception` evidence** — `mechanism.handled` distinguishes caught vs uncaught (Sentry parity)
> - **§5.3 `dotted_order` on `TelemetrySpan`** — LangSmith-compatible single-string hierarchical sort key
> - **§6.2 `Handoff` primitive + `Report.handoffs[]`** — multi-agent delegation events (OpenAI Agents SDK lineage)
> - **§6.3 `PromptManifest` primitive + `ProjectInfo.prompts[]`** — prompt provenance (LangSmith Prompt Hub + OpenInference `prompt.*` lineage)
> - **§6.1 SkillManifest expansion:** `oasf_skill_ids[]` (LF AI & Data taxonomy refs), `agentskills_io_canonical_name`, `execution_tier` (Cloudflare 5-tier ladder + ARP `in_process`)
> - **§6.4 `tech_stack.mcp_server_exposed` + `mcp_clients_consumed[]`** — MCP role disambiguation
>
> **What was new in 1.1.1** (2026-04-27, additive over 1.1.0):
>
> - **§3.6 Breadcrumb-trail array on Warning** — Sentry-aligned, debug observability for cascade failures
> - **§5 normative reference: OpenInference v1** — adopt the upstream spec's attribute namespace (`openinference.span.kind`, `llm.*`, `retrieval.*`, `tool.*`); spec text only, Phoenix implementation excluded (ELv2 license guardrail per the deep-dive REJECT list)
> - **§5.1 TelemetrySpan typed shape** — opt-in, with `kind` (10-value OpenInference enum) and `subtype` (8-value OpenAI Agents enum)
> - **§6.1 SkillManifest expansion** — agentskills.io 6 open-spec frontmatter fields (`name`, `description`, `version`, `license`, `metadata`, `compatibility`); new `auto_invocable` flag (polarity-flipped from Anthropic Skills' `disable-model-invocation`, default false / opt-in)
>
> **What was new in 1.1.0** (2026-04-27): additive backwards-compat
> spec extension driven by [`docs/research/standards_deepdive_2026_04_27/ULTIMATE_PROFILE_RECOMMENDATION.md`](research/standards_deepdive_2026_04_27/ULTIMATE_PROFILE_RECOMMENDATION.md):
>
> - **§3.5 Score primitive** — typed evaluation outcomes (numeric / categorical / boolean) for LangSmith / Langfuse / OpenInference interop
> - **§2.1 `_meta` reverse-DNS extension envelope** — MCP-style vendor extensions, present on every object
> - **§3.3 Fingerprint union** — `string` (canonical) OR `string[]` (Sentry-aligned, with `{{ default }}` placeholder support); v1.0 senders unaffected
> - **§2 envelope additions:** `session_id`, `user_id`, `conversation_id` for multi-turn agent flow correlation (LangSmith Threads / OTel `gen_ai.conversation.id` aliases)
>
> All v1.0 senders continue to validate against the v1.1.x schema unchanged. v1.1.x senders gain the new fields without breaking v1.0 collectors (collectors MUST ignore unknown fields per §7).

---

## ⚠️ Strategic Position — read first

**This is NOT a competing standard.** It is AgentMinds' internal
**reporting profile**, built on top of:

- [Sentry data schemas](https://github.com/getsentry/sentry-data-schemas) — issue lifecycle
- [OpenTelemetry GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/) — telemetry semantic conventions
- [Model Context Protocol](https://modelcontextprotocol.io/specification) — transport binding hint
- [Anthropic Claude Skills](https://github.com/anthropics/skills) — `SKILL.md` frontmatter
- [AGNTCY OASF](https://github.com/agntcy/oasf) — descriptor envelope and observability primitives

The only novel piece is the **cross-site learned-pattern lifecycle**
in [§4.1 Pattern object](#41-pattern-object). No upstream standard
covers how agents share what they LEARNED across sites; this profile
fills that single gap.

**When a higher-level standard adopts that primitive, AgentMinds MUST
reorient this profile to it within 30 days.** We are explicitly not in
the business of owning a competing standard — we are in the business
of being the **fastest adopter** of whichever standard wins. This
positioning is load-bearing; it survives Google / Cloudflare / AWS
announcing their own agent-reporting spec at any time.

If you are reading this and wondering "should my product follow ARP
or follow standard X?" — the honest answer is **follow standard X**.
ARP is a thin convenience layer on top of those standards, optimized
for AgentMinds' delivery surface. Adopt the lineage, not the profile.

### How ARP positions AgentMinds across five layers

The five lineage standards are not competitors — they are **stacked
layers** under the Linux Foundation AI & Data umbrella (early 2026):

```
SLIM (transport, gRPC over H2/3, MLS+PQ)
   ↓
A2A / ACP (peer-to-peer agent delegation; ACP merged into A2A in Aug 2025)
   ↓
MCP (agent → tool, JSON-RPC 2.0)
   ↓
OASF (agent identity + capability schema)
   ↓
OTel GenAI (telemetry semantic conventions)
   +
Sentry-style fingerprint + issue lifecycle (cross-site dedup)
```

ARP enables AgentMinds to be **simultaneously**:

1. An **OASF record** — capability schema declaring what AgentMinds is
2. An **A2A agent** — cross-org REST endpoint at `/.well-known/agent-card.json`
3. An **MCP server** — LLM-callable tools (`agentminds_push`, `agentminds_query_pattern`)
4. **OpenInference-instrumented** — Langfuse/Arize auto-trace
5. **Sentry-pattern compatible** — fingerprint + lifecycle aggregation

No other platform ships all five layers in one product. ARP is the
contract that lets AgentMinds present consistently across them.

---

A vendor-neutral schema for agent reports, warnings, learned patterns,
recommendations, and skill manifests. ARP unifies five prior-art
lineages into one envelope so any agent — regardless of framework — can
speak the same wire format to AgentMinds and to other observability
platforms without lock-in.

## Lineage

| Source | What we adopt | Why |
|---|---|---|
| **[Sentry](https://github.com/getsentry/sentry-data-schemas)** | issue lifecycle: `status`, `first_seen`, `last_seen`, `fingerprint`, `level` | Battle-tested, used by 5M+ developers. Prevents "solved" warnings from re-surfacing. |
| **[OpenTelemetry GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)** | metric type taxonomy (`counter`/`gauge`/`histogram`), unit-of-measurement (UCUM), telemetry attribute names | CNCF-backed neutral telemetry. Datadog/New Relic/Honeycomb already parse it. |
| **[MCP](https://modelcontextprotocol.io/specification)** | transport binding hint; tool-call shape; planned "Skills Over MCP" alignment | Anthropic-led open protocol; OpenAI/Google/Microsoft/Cursor adopters. AgentMinds already exposes `mcp__agentminds__*` tools. |
| **[Anthropic Claude Skills](https://github.com/anthropics/skills)** | `SKILL.md` YAML frontmatter (`name`, `description`, `allowed-tools`) | The de-facto skill discovery format used by Claude Code agents. |
| **[AGNTCY OASF](https://github.com/agntcy/oasf)** | `descriptor` envelope (media-type, digest, size), `metric` shape, observability data referencing | Linux-Foundation-hosted multi-vendor agent framework. 75+ member orgs. |

ARP is the **report side** of agent operations (what the agent saw, what
it learned, what it recommends). Pair it with an **identity** layer (W3C
Verifiable Credentials, OASF Identity) and a **transport** layer (HTTP
POST, MCP, gRPC over SLIM) to get a full agent-to-platform pipeline.

---

## 1. Glossary and normative references

### 1.1 Glossary

- **Agent** — A named software process that observes a system and emits
  reports about it. (`uptime_agent`, `seo_agent`, `freshness_agent`)
- **Report** — A single envelope produced by one agent at one moment in
  time, carrying warnings, recommendations, learned patterns, and
  optional telemetry.
- **Warning** — An observation the agent flagged as problematic.
- **Recommendation** — A proposed action attached to one or more
  warnings, or surfaced standalone.
- **Pattern** — A reusable observation extracted from many runs of an
  agent across one or more sites; intended for cross-agent learning.
- **Fingerprint** — A stable hash that uniquely identifies a warning or
  pattern across time and across reporters, used for deduplication and
  cross-site correlation.
- **Site** — The system being observed. Identified by a stable `site_id`.

### 1.2 Normative references (v1.2.1)

The following upstream specs are cited NORMATIVELY by ARP. Implementations
MUST honour the latest version listed unless otherwise stated.

| Spec | Version | License | What ARP defers to |
|---|---|---|---|
| **[Sentry Data Schemas](https://github.com/getsentry/sentry-data-schemas)** | v7+ | BSD-3-Clause | Issue lifecycle (`status`, `first_seen`, `last_seen`, `fingerprint`, `level`, `breadcrumbs`, `mechanism.handled`) |
| **[OpenTelemetry Semantic Conventions — GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)** | 1.27+ | Apache-2.0 | `gen_ai.*` attribute namespace, span shapes, GenAI events, metric instruments + UCUM units |
| **[OpenInference v1](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md)** | v1 spec text | Apache-2.0 (spec only — Phoenix runtime explicitly excluded per §13 REJECT list) | `openinference.span.kind` 10-value enum, `llm.*` / `retrieval.*` / `tool.*` attribute namespaces |
| **[Model Context Protocol (MCP)](https://modelcontextprotocol.io/specification)** | 2025-11-25 | MIT | `_meta` reverse-DNS extension envelope, JSON-RPC 2.0 transport binding, planned Skills-Over-MCP |
| **[Anthropic Claude Skills](https://github.com/anthropics/skills)** + **[agentskills.io](https://agentskills.io)** | open spec | (community) | `SKILL.md` YAML frontmatter (`name`, `description`, `version`, `license`, `metadata`, `compatibility`, `allowed_tools`) |
| **[AGNTCY OASF](https://github.com/agntcy/oasf)** | v1 | Apache-2.0 | `descriptor` envelope, `metric` shape, observability data referencing, skill taxonomy IDs |
| **[Langfuse Score model](https://langfuse.com/docs/scores)** | (open) | MIT | Score primitive shape (numeric/categorical/boolean) |
| **[OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)** | v0.1+ | Apache-2.0 (SDK shape adopted as a vocabulary; ARP does NOT use the proprietary trace exporter) | 8-value span subtype enum, Handoff field shape |

The Sentry implementation, the OpenInference Phoenix runtime, the OpenAI
Agents SDK proprietary tracing backend, the LangSmith vendor-locked
backend, and the AGNTCY SLIM transport are each cited only at the
schema/vocabulary level — ARP does not take a runtime dependency on
any of them. See §13 (REJECT list).

---

## 2. The Envelope

Every report sent to AgentMinds (or any ARP-compliant collector) is a
single JSON object with this top-level shape:

```jsonc
{
  "arp_version": "1.2.1",                    // optional but recommended
  "agent": "<string>",                       // required
  "site_id": "<string>",                     // required (set by transport if omitted)
  "session_id":      "<string>",             // optional, v1.1 — multi-turn flow
  "user_id":         "<string>",             // optional, v1.1 — opaque user id
  "conversation_id": "<string>",             // optional, v1.1 — OTel gen_ai.conversation.id alias
  "report": { ... },                         // required — see §3
  "memory": { ... },                         // optional — see §4
  "telemetry": { ... },                      // optional — see §5
  "project_info": { ... },                   // optional — see §6
  "schema": {                                // optional — see §7
    "name": "agentminds.arp",
    "version": "1.2.1"
  },
  "_meta": { ... }                           // optional, v1.1 — see §2.1
}
```

For backwards compatibility, collectors MUST also accept the legacy
key `ars_version` as an alias for `arp_version` and `agentminds.ars`
as the schema name; both decay to the same envelope.

The transport (HTTP POST `/sync/ingest`, MCP tool call, etc.) MAY add
`api_key`, `received_at`, and `correlation_id` siblings; these are not
part of ARP itself.

### 2.1 `_meta` extension envelope (v1.1)

Every ARP object MAY carry a `_meta` field — an open object intended
for vendor-specific extensions that should not pollute the canonical
schema. Keys MUST follow reverse-DNS namespacing per the MCP
convention:

```jsonc
{
  "_meta": {
    "com.agentminds.deploy_id": "abc123",
    "io.agentminds.replay_count": 3,
    "com.othervendor.tag": "experiment-1"
  }
}
```

**Rules:**
- Reserved second-level namespaces: `agentminds.*`, `arp.*`. These
  are managed by AgentMinds; other implementers MUST NOT collide.
- All other namespaces are first-come-first-served by reverse-DNS.
- Collectors MUST ignore unknown `_meta` keys — they MUST NOT reject
  the payload solely because of an unfamiliar `_meta` field.
- Senders SHOULD NOT use `_meta` for fields that have a canonical
  ARP location. If the canonical schema gains a field that overlaps
  an existing `_meta` extension, the canonical wins; the `_meta`
  extension SHOULD be deprecated within 30 days (per §7
  reorientation clause).

`_meta` is available on the envelope and on every `$def` object
(`Warning`, `Recommendation`, `Pattern`, `Score`, `Memory`,
`SkillManifest`, `ProjectInfo`).

---

## 3. `report` — Current state

```jsonc
{
  "severity": "fatal" | "error" | "warning" | "info" | "debug",
  "summary": "<2-3 sentence human-readable status>",
  "score": 0..100,                           // optional one-shot health score
  "metrics": { ... },                        // free-form key/value, see §3.2
  "warnings":        [Warning],              // see §3.3
  "recommendations": [Recommendation],       // see §3.4
  "scores":          [Score],                // optional, v1.1 — see §3.5
  "_meta":           { ... }                 // optional, v1.1 — see §2.1
}
```

### 3.1 Severity (Sentry-aligned)

| Value     | Meaning                                                  |
|-----------|----------------------------------------------------------|
| `fatal`   | Service-impacting; user-facing outage                    |
| `error`   | Functional failure; not necessarily user-visible         |
| `warning` | Degradation or risk; should be addressed                 |
| `info`    | Informational; no action required                        |
| `debug`   | Diagnostic only; collectors MAY drop these by default    |

Severity SHOULD reflect the *worst* item in `warnings`. Collectors MAY
override it based on their own policy.

### 3.2 `metrics`

Free-form key-value object. Keys SHOULD follow OTel GenAI naming when
applicable. Examples:

```jsonc
{
  "uptime_pct_24h": 99.97,
  "response_time_ms_p95": 187,
  "gen_ai.request.model": "claude-sonnet-4-6",
  "gen_ai.usage.input_tokens": 12453,
  "gen_ai.usage.output_tokens": 856,
  "gen_ai.cost.usd": 0.041
}
```

For richer per-metric metadata (type, unit, data points), use the OASF
`metric` object form (see §5).

### 3.3 `Warning` object

A warning is the smallest atom of "something the agent saw that may
matter". It is the unit on which the AgentMinds filter pipeline operates.

```jsonc
{
  "fingerprint": "<sha256 hex, lowercased, 64 chars>",
  "message":     "<human-readable description>",
  "level":       "fatal" | "error" | "warning" | "info" | "debug",
  "severity":    "critical" | "high" | "medium" | "low" | "info",
  "status":      "unresolved" | "resolved" | "ignored" | "muted",
  "first_seen":  "<RFC 3339 UTC timestamp>",
  "last_seen":   "<RFC 3339 UTC timestamp>",
  "count":       <integer ≥ 1>,
  "category":    "performance" | "security" | "content" | "seo"
                 | "quality" | "compliance" | "infra" | "general",
  "tags":        { "<key>": "<value>", ... },
  "evidence":    { ... }
}
```

#### Required vs optional

- **REQUIRED:** `message`
- **RECOMMENDED:** `level` or `severity`, `status`, `category`,
  `last_seen`
- **OPTIONAL but stable across reports:** `fingerprint`, `first_seen`,
  `count`, `tags`, `evidence`

#### Field semantics

- **`fingerprint`** — Stable identity across reports and sites.
  Two forms accepted (v1.1):
  - **Single string** (canonical ARP): `sha256(agent_name + "::" + normalize(message))`,
    lowercase hex, 64 chars. `normalize` lowercases, strips whitespace,
    and removes volatile numerics (timestamps, request IDs, percentages).
  - **Array of strings** (Sentry-aligned, v1.1+): segments are joined
    server-side. The literal `{{ default }}` placeholder MAY appear at
    any position to inherit the SDK's default fingerprint logic.
    Example: `["myAppErrorType", "{{ default }}"]`.

  If the SDK omits `fingerprint`, the collector MUST compute and persist
  a server-side single-string form before storing the row.

  **Sentry alias (v1.2.1):** `Warning.fingerprint` is the canonical
  ARP field. For Sentry SDK ingest compatibility, collectors SHOULD
  also accept the JSON pointer alias `event.fingerprint` (Sentry's
  envelope field name) and treat it identically. Server-side
  normalization MUST emit only the canonical name on outbound payloads.
- **`level`** vs **`severity`** — `level` follows Sentry semantics (log
  severity); `severity` follows incident-management semantics
  (`critical`/`high`/`medium`/`low`). Senders MAY supply both. If only
  one is present, the collector MUST infer the other deterministically.
- **`status`** — Lifecycle state. Defaults to `unresolved` on first
  ingest. Transitions:
  - `unresolved` → `resolved` (manually or by SDK after fix verified)
  - `unresolved` → `ignored` (false-positive flag from operator)
  - `resolved` → `unresolved` (regression — same fingerprint reappeared
    after `resolved_at`)
  - `unresolved` → `muted` (acknowledged but suppressed for a window)
- **`first_seen` / `last_seen`** — UTC timestamps. The collector MUST
  update `last_seen` on every ingest where the fingerprint matches an
  existing row. `first_seen` MUST NOT be overwritten once set.
- **`count`** — Total occurrences observed for this fingerprint at this
  site. Collector increments on each ingest.
- **`evidence`** — Free-form bag carrying snippet/url/stack/screenshot
  refs. Schema-less so callers can attach what they have without
  normalizing.

#### Status filtering — normative

Any consumer that surfaces warnings to a human (dashboards,
recommendations, action lists) **MUST** apply this filter unless the
operator explicitly opts in to seeing all states:

```
include  ⇔  status ∈ {"unresolved"} AND last_seen within freshness window
exclude  ⇔  status ∈ {"resolved", "ignored", "muted"}
exclude  ⇔  last_seen older than category's stale-threshold
```

Default freshness windows:

| Category | Stale-after |
|---|---|
| `security` | 7 days |
| `performance`, `seo`, `content`, `quality` | 30 days |
| `infra`, `compliance` | 14 days |
| `general` | 21 days |

### 3.4 `Recommendation` object

```jsonc
{
  "fingerprint":  "<sha256 hex>",            // stable across reports
  "title":        "<short imperative>",
  "details":      "<longer markdown body>",
  "priority":     "critical" | "high" | "medium" | "low",
  "category":     "<same enum as Warning>",
  "warning_fingerprints": ["<fp>", ...],     // links to specific warnings
  "effort":       "trivial" | "small" | "medium" | "large",
  "confidence":   0.0..1.0,
  "code_snippet": "<diff or block>",         // optional
  "source_rule":  "<rule id, e.g. ars/seo/title-length>",
  "status":       "open" | "applied" | "dismissed"
}
```

`title` is REQUIRED; everything else is OPTIONAL but `priority`,
`fingerprint`, and `category` are STRONGLY RECOMMENDED.

**LangSmith alias (v1.2.1):** when `confidence` is a 0..1 numeric, it
is wire-compatible with LangSmith's `feedback.score` primitive.
Collectors building a `arp-langsmith-bridge` MAY map between the two
without lossy translation. For richer eval shapes (categorical /
boolean / comparative), use `Score` (§3.5) instead.

### 3.4.1 `gen_ai.evaluation.result` → §4.1 Pattern bridge (v1.2.1)

When an OTel GenAI span carries a `gen_ai.evaluation.result` attribute,
the collector MAY auto-promote that result into an ARP §4.1 Pattern
once it has been observed `≥ N` times across distinct sites (default
N=3). Informative only; reference shapes:

```jsonc
// Source — OTel GenAI span attribute
{
  "name": "evaluation.run",
  "attributes": {
    "gen_ai.system": "openai",
    "gen_ai.evaluation.name": "hallucination_check",
    "gen_ai.evaluation.result": "fail"
  }
}

// Auto-promoted — ARP §4.1 Pattern
{
  "fingerprint": "<sha256(agent + '::hallucination_check::fail')>",
  "pattern":     "hallucination_check fails on long-context prompts",
  "category":    "quality",
  "status":      "active",
  "confidence":  0.0,                  // Beta-Bernoulli updates each observation
  "first_seen":  "<first observation>",
  "last_seen":   "<latest observation>",
  "_meta": {
    "agentminds.bridge.source": "otel.gen_ai.evaluation.result",
    "agentminds.bridge.threshold_n": 3
  }
}
```

This bridge lets observability-first stacks (Langfuse, Arize,
OpenInference adopters) surface cross-site patterns without wiring
ARP push directly. Implementation in a future ARP collector; spec
text is informative as of v1.2.1.

### 3.5 `Score` object (v1.1)

Typed evaluation outcome. Lineage: Langfuse Score model + LangSmith
Feedback. Use Score when you need to attach an evaluation verdict
(numeric / categorical / boolean) to a warning, a pattern, or the
report as a whole — without overloading `Recommendation.confidence`
(which only handles numeric).

```jsonc
{
  "fingerprint":              "<sha256 hex or array>",  // optional
  "key":                      "<short stable id>",      // REQUIRED
  "data_type":                "numeric" | "categorical" | "boolean",   // REQUIRED
  "value":                    <number | string | boolean>,             // REQUIRED
  "name":                     "<human-readable label>",
  "comment":                  "<optional rationale>",
  "warning_fingerprints":     ["<fp>", ...],
  "pattern_fingerprints":     ["<fp>", ...],
  "compared_to_baseline_id":  "<run id>",
  "_meta":                    { ... }
}
```

#### Required vs optional

- **REQUIRED:** `key`, `data_type`, `value`
- **`data_type`** is a **closed enum** (`numeric` / `categorical` /
  `boolean`). Vendor extensions go through `_meta`, not new
  `data_type` values, to avoid enum fragmentation.
- **`value`** type MUST match `data_type`:
  - `numeric` → JSON number
  - `categorical` → JSON string
  - `boolean` → JSON boolean

#### When to use Score vs Recommendation.confidence

- Use **`Score`** for evaluation verdicts: "this output is toxic",
  "passed safety filter", "hallucination risk = 0.7". Multi-modal,
  attachable to many entities, designed for eval pipelines.
- Use **`Recommendation.confidence`** for "how confident are we that
  this recommendation is correct?" — single 0..1 float, baked into
  the recommendation itself.

The two coexist; they are not aliases.

### 3.6 `Breadcrumb` trail (v1.1.1)

A `Breadcrumb` is a timestamped event leading up to a Warning.
Sentry-aligned. Critical for understanding *cascade failures* —
the chain of "user clicked X → DB query Y → 500 from Z" before the
warning fired.

```jsonc
{
  "timestamp": "<RFC 3339>",
  "type":      "navigation" | "http" | "error" | "info"
              | "query" | "ui" | "user" | "default",
  "category":  "<free-form, e.g. 'console', 'fetch', 'agent.call'>",
  "message":   "<human-readable>",
  "level":     "<Severity>",
  "data":      { ... }
}
```

Attach as `Warning.breadcrumbs[]` (optional). Senders SHOULD cap
at ~50 breadcrumbs per warning to avoid payload bloat. Order
chronologically (oldest first, most-recent last).

**REQUIRED:** `timestamp`, `type`. Everything else is OPTIONAL.

### 3.7 `Exception` evidence (v1.2.0)

Structured exception payload. Lineage:
[Sentry Data Schemas](https://github.com/getsentry/sentry-data-schemas)
exception envelope, including the `mechanism.handled` field for
handled-vs-uncaught triage. Goes inside `Warning.evidence.exception`
when an agent reports a thrown/caught error.

```jsonc
{
  "type":       "<string — exception class name, e.g. 'ValueError', 'TimeoutError'>",
  "value":      "<string — stringified exception message>",
  "module":     "<string — importable module path>",
  "stacktrace": { ... },
  "mechanism": {
    "handled": <boolean>,
    "type":    "<string — exception_handler | signal | unhandled | ...>"
  }
}
```

#### Required vs optional

All fields OPTIONAL.

#### Field semantics

- **`mechanism.handled`** — `true` = caught by agent's try/except. `false` = uncaught (reached the top of the stack).
- **`mechanism.type`** — how the exception entered the report: `exception_handler` / `signal` / `unhandled` / etc.

Senders SHOULD attach this object at `Warning.evidence.exception` so
consumers can locate it deterministically.

---

## 4. `memory` — What the agent has learned

```jsonc
{
  "learned_patterns": [Pattern],             // see §4.1
  "recurring_issues": [Warning],             // same shape as §3.3
  "total_runs": <integer>,
  "schema_version": "1.0"
}
```

### 4.1 `Pattern` object

A pattern is a reusable observation extracted by an agent from many
runs. Patterns may be promoted to the cross-site pool (`patterns`
table) once they meet confidence and universality thresholds — see
[CORE_PURPOSE.md](../CORE_PURPOSE.md) for the auth boundary.

```jsonc
{
  "fingerprint":  "<sha256 hex>",            // stable across reporters
  "pattern":      "<short label>",
  "category":     "performance" | "security" | "content" | "seo"
                 | "quality" | "general",
  "detail":       "<what was learned, in 1-3 sentences>",
  "action":       "<what was done about it, if any>",
  "confidence":   0.0..1.0,
  "impact":       "critical" | "high" | "medium" | "low",
  "status":       "active" | "solved" | "obsolete",
  "first_seen":   "<RFC 3339>",
  "last_seen":    "<RFC 3339>",
  "first_seen_run": <integer>,               // run index when first observed
  "last_confirmed_run": <integer>,
  "supporting_evidence_count": <integer>,
  "applicable_site_types": ["ecommerce", "blog", ...]
}
```

#### Status semantics

- **`active`** — Pattern is currently observed; the underlying issue is
  ongoing or repeats periodically.
- **`solved`** — The pattern was observed, action was taken, and the
  pattern has not re-surfaced for at least `2 × stale_after`. AgentMinds
  MUST NOT recommend re-applying the action while status is `solved`.
- **`obsolete`** — The pattern no longer applies (tech changed, site
  type changed, vendor deprecated). Excluded from all recommendation
  surfaces.

---

## 5. `telemetry` — OpenTelemetry GenAI compatible

OPTIONAL. When present, this section MUST follow the OTel GenAI
semantic conventions. AgentMinds collectors that do not implement OTel
storage MAY ignore this section but MUST preserve it on round-trips.

```jsonc
{
  "spans":   [ /* otel span dicts */ ],
  "events":  [ /* otel event dicts */ ],
  "metrics": [
    {
      "name":  "gen_ai.client.token.usage",
      "type":  "histogram",                  // counter | gauge | histogram
      "unit":  "{token}",                    // UCUM where applicable
      "data_points": [
        { "value": 12453, "timestamp": "2026-04-27T10:15:03Z",
          "attributes": { "gen_ai.token.type": "input",
                          "gen_ai.request.model": "claude-sonnet-4-6" } }
      ]
    }
  ]
}
```

Recommended attribute names:

| Attribute | Type | Source |
|---|---|---|
| `gen_ai.system` | string | OTel |
| `gen_ai.request.model` | string | OTel |
| `gen_ai.response.model` | string | OTel |
| `gen_ai.usage.input_tokens` | int | OTel |
| `gen_ai.usage.output_tokens` | int | OTel |
| `gen_ai.cost.usd` | double | ARP extension |
| `gen_ai.recommendation.severity` | string | ARP extension |
| `gen_ai.recommendation.fingerprint` | string | ARP extension |
| `agentminds.site_id` | string | ARP extension |
| `agentminds.agent_name` | string | ARP extension |

### 5.1 Normative references (v1.1.1)

ARP §5 telemetry adopts these upstream specs **by reference**.
Senders SHOULD use upstream attribute names verbatim and namespace
ARP-specific extensions under `agentminds.*` per §2.1.

| Spec | Version | License | What ARP adopts |
|---|---|---|---|
| **OpenTelemetry GenAI semconv** | 1.27+ | Apache-2.0 | The full `gen_ai.*` attribute namespace, span shapes, and event types |
| **OpenInference v1** | spec only | Apache-2.0 | The `openinference.span.kind` 10-value enum (LLM/CHAIN/TOOL/RETRIEVER/EMBEDDING/AGENT/GUARDRAIL/EVALUATOR/RERANKER/UNKNOWN), `llm.*`/`retrieval.*`/`tool.*` attribute namespaces, prompt/response capture conventions |
| **OpenAI Agents span subtypes** | SDK 0.1+ | (vendor-defined) | The 8-value `subtype` enum: `agent_span` / `generation_span` / `function_span` / `guardrail_span` / `handoff_span` / `transcription_span` / `speech_span` / `custom_span` |

**Important — what ARP does NOT adopt from OpenInference:**

We adopt the **spec text** (Apache-2.0) only. We explicitly do NOT
take a runtime dependency on Phoenix code (Elastic License v2 + US
patent claims) — see the deep-dive REJECT list. Implementers
following ARP for telemetry should:

- ✅ Reuse OpenInference attribute names directly
- ✅ Adopt the 10-value span-kind enum
- ❌ NOT vendor-bundle Phoenix runtime as part of an ARP collector

### 5.2 `TelemetrySpan` typed shape (v1.1.1)

```jsonc
{
  "name":        "openai.chat.completions.create",
  "kind":        "LLM",                // OpenInference 10-enum
  "subtype":     "generation_span",    // OpenAI Agents 8-enum
  "trace_id":    "<hex>",
  "span_id":     "<hex>",
  "parent_span_id": "<hex>",
  "start_time":  "<RFC 3339>",
  "end_time":    "<RFC 3339>",
  "status":      { "code": "OK" },
  "attributes":  {
    "gen_ai.system": "openai",
    "openinference.span.kind": "LLM",
    "agentminds.site_id": "site_a1b2"
  },
  "events":      [ /* per-span events */ ],
  "_meta":       { /* vendor extensions */ }
}
```

Backwards-compat: v1.0 `Telemetry.spans[]` accepted free-form
`{type:object}`. v1.1.1 `TelemetrySpan` shape is OPTIONAL — every
field is OPTIONAL except for the implicit collector contract that
unknown fields are ignored. v1.0 spans pass v1.1.1 validation.

---

## 6. `project_info` — Self-description

OPTIONAL but strongly recommended. Lets cross-site recommendation
filters know what tech a site has so they don't recommend things the
site already runs.

```jsonc
{
  "skills": [SkillManifest],                 // §6.1
  "functions": [
    { "name": "...", "file": "...", "type": "core|util|api",
      "description": "..." }
  ],
  "tech_stack": {
    "framework":  "FastAPI",
    "frontend":   "Next.js",
    "database":   "PostgreSQL",
    "hosting":    "Render",
    "ai_model":   "Claude",
    "monitoring": ["Sentry", "Datadog"],
    "testing":    ["Playwright"]
  },
  "api_routes": [
    { "method": "POST", "path": "/api/...", "description": "..." }
  ]
}
```

### 6.1 `SkillManifest` (agentskills.io + Anthropic Skills aligned)

A single skill is described by parsing its `SKILL.md` file's YAML
frontmatter and the file body. The on-disk format is
[Claude Skills](https://github.com/anthropics/skills); ARP §6.1
adopts the [agentskills.io open spec](https://agentskills.io)
frontmatter fields verbatim:

```jsonc
{
  "name":           "<string, REQUIRED>",
  "description":    "<string, REQUIRED>",
  "version":        "<semver>",
  "license":        "<SPDX id, e.g. MIT, Apache-2.0, CC-BY-4.0>",
  "allowed_tools":  ["Read", "Edit", ...],
  "auto_invocable": false,                  // v1.1.1, default false (opt-in)
  "metadata":       { "<key>": "<value>" }, // agentskills.io free-form
  "triggers":       ["...", "..."],         // ARP extension
  "compatibility": {
    "agent_runtimes": ["claude-code>=1.0", "agentminds-cli>=0.1"]
  },
  "_meta":          { "<reverse-DNS>": ... }
}
```

#### v1.1.1 polarity flip — `auto_invocable`

Anthropic Skills' `disable-model-invocation: true` becomes ARP's
`auto_invocable: false` (default). Reasons:

- **Safer default.** Opt-in to LLM auto-loading is more conservative
  than opt-out.
- **Sentence reads cleaner.** "Is this skill auto-invocable?" → bool.
  "Is model invocation disabled?" → double negative.
- **Forwards-compat.** When the LLM-invocation surface grows (chains,
  guardrails, etc.), positive flags are easier to extend than
  inverted ones.

Senders MAY continue to read upstream `disable-model-invocation`
from raw `SKILL.md` files; collectors translate to the polarity-
flipped form on ingest.

Only `name` and `description` are REQUIRED, matching Anthropic's
SKILL.md spec. AgentMinds publishes its own skills (`agentminds-status`,
`agentminds-agents`, `agentminds-pool`, `agentminds-connect`,
`agentminds-push-report`) in this format.

#### v1.2.1 OASF 90000-99999 range reservation

ARP reserves OASF taxonomy IDs **90000-99999** as the
`agentminds.*` extension namespace per the §1.5 normative reference
to AGNTCY OASF. Vendors building skills against ARP MAY allocate
their own number from this range without colliding with upstream
canonical IDs (1xxx, 5xxx, etc).

To reserve a sub-range, file a PR against [`agentmindsdev/profile`](https://github.com/agentmindsdev/profile)
adding your range to a `RESERVED_RANGES.md` (forthcoming).

### 6.2 `Handoff` primitive + `Report.handoffs[]` (v1.2.0)

One multi-agent delegation event. Lineage:
[OpenAI Agents SDK](https://openai.github.io/openai-agents-python/)
`Handoff` class. Attached to a report as the `Report.handoffs[]` array
— captures multi-agent delegation events that occurred during this
report.

```jsonc
{
  "from_agent":           "<string, REQUIRED>",
  "to_agent":             "<string, REQUIRED>",
  "trigger":              "<human-readable reason for the handoff>",
  "tool_name_override":   "<string>",
  "input_filter":         "<regex/spec describing how the input is filtered before delegation>",
  "nest_handoff_history": <boolean>,
  "occurred_at":          "<RFC 3339 timestamp>",
  "_meta":                { "<reverse-DNS>": ... }
}
```

#### Required vs optional

- **REQUIRED:** `from_agent`, `to_agent`
- All other fields OPTIONAL.

### 6.3 `PromptManifest` primitive + `ProjectInfo.prompts[]` (v1.2.0)

Prompt provenance. Lineage:
[OpenInference](https://github.com/Arize-ai/openinference/blob/main/spec/semantic_conventions.md)
`prompt.*` attributes +
[LangSmith Prompt Hub](https://docs.smith.langchain.com/). Attached to
`ProjectInfo.prompts[]` — describes prompts in use by this project so
recommendation engines avoid telling you to add a prompt registry you
already have.

```jsonc
{
  "vendor":    "<string, e.g. 'langsmith', 'openai', 'self', 'anthropic-prompts'>",
  "id":        "<string>",
  "version":   "<string>",
  "url":       "<string>",
  "template":  "<inline template body>",
  "variables": { "<key>": "<value>" },
  "_meta":     { "<reverse-DNS>": ... }
}
```

#### Required vs optional

- All fields OPTIONAL.
- Senders SHOULD provide either `url` OR `template` so consumers can
  identify the prompt without ambiguity.

### 6.4 `tech_stack.mcp_server_exposed` + `mcp_clients_consumed[]` (v1.2.0)

MCP role disambiguation, attached inline to `ProjectInfo.tech_stack`.
Distinguishes whether the project publishes tools (as an MCP server),
consumes upstream MCP servers, or both.

```jsonc
{
  "tech_stack": {
    "mcp_server_exposed":   false,
    "mcp_clients_consumed": ["<upstream MCP server URL or name>", ...]
  }
}
```

#### Required vs optional

- All fields OPTIONAL.
- `mcp_server_exposed`: boolean — does this site expose tools as an
  MCP server?
- `mcp_clients_consumed`: array of strings — upstream MCP servers
  this site consumes.

---

## 7. Versioning & evolution

- ARP uses **semantic versioning**: `MAJOR.MINOR.PATCH`.
- **MINOR** bumps add OPTIONAL fields. Senders SHOULD continue to work
  against older collectors; collectors SHOULD ignore unknown fields.
  v1.1 is exactly this kind of bump (Score, `_meta`, fingerprint
  union, session/user/conversation IDs — all additive).
- **MAJOR** bumps remove or rename REQUIRED fields. Migration notes
  ship in `docs/arp-migration-<old>-to-<new>.md`.
- Senders SHOULD include `arp_version` at the envelope root. Collectors
  MUST default to the latest minor of the same major when omitted.
- The canonical JSON Schema is at
  [`docs/schemas/agent_report.schema.json`](schemas/agent_report.schema.json).

### 7.1 Version compatibility matrix

| v1.0 sender → v1.1 collector | ✅ Works unchanged. v1.0 fields are a subset of v1.1. |
| v1.1 sender → v1.0 collector | ✅ Works. Collector ignores unknown fields per §7. New fields (`scores`, `_meta`, `session_id`, etc.) are silently dropped. |
| v1.1 sender → v1.1 collector | ✅ Full feature set. Recommended path. |
| v1.0 / v1.1 sender → v1.2 collector | ✅ Works. v1.2 collector accepts older payloads unchanged; v1.2-only features unused. |
| v1.2 sender → v1.0 / v1.1 collector | ✅ Works. Collector ignores unknown fields per §7. v1.2 fields (e.g., `lifecycle_event`, `exception`, `handoffs`, `prompts`) are silently dropped. |
| v1.2 sender → v1.2 collector | ✅ Full feature set. Recommended path. |
| Pre-v1.0 (`ars_version`) sender | ✅ Works. `ars_version` accepted as alias for `arp_version`. |
- **Reorientation clause:** if any of the upstream lineage standards
  (Sentry / OTel GenAI / MCP / Claude Skills / OASF) absorbs the
  pattern-lifecycle primitive (§4.1) into their own canonical spec,
  AgentMinds MUST publish ARP `MAJOR+1` within 30 days that defers
  the primitive to the upstream definition.

---

## 8. Identity & transport (informative)

ARP is transport-agnostic. Common bindings:

- **HTTP POST** — `POST /sync/ingest` with `Authorization: Bearer
  <api_key>` and `Content-Type: application/json`. Body is the §2
  envelope.
- **MCP** — `agentminds_push` tool call with body as the
  `reports` argument.
- **gRPC over SLIM** — encode the envelope as a Protobuf message;
  AgentMinds publishes the `.proto` at
  [`docs/schemas/agent_report.proto`](schemas/agent_report.proto)
  *(planned 1.1)*.

For agent identity beyond a shared API key, AgentMinds plans to support
W3C Verifiable Credentials per AGNTCY's Identity Service in ARP 2.0.

### 8.1 `/.well-known/arp.json` self-description (v1.2.1, NORMATIVE)

Every ARP-conformant collector MUST publish a self-description at
`/.well-known/arp.json` — the canonical discovery point for any tool
or agent that wants to integrate. Think of it as the ARP equivalent
of A2A's `agent-card.json` or AGNTCY's `oasf-descriptor.json`, but
focused specifically on what ARP version and conformance level the
collector implements.

```jsonc
{
  "arp_version": "1.2.1",
  "conformance_level": "L2",   // L0..L5 per §9
  "endpoints": {
    "ingest": "https://api.agentminds.dev/api/v1/sync/report",
    "ingest_otel": "https://api.agentminds.dev/api/v1/sync/ingest/otel",
    "connect": "https://api.agentminds.dev/api/v1/connect"
  },
  "auth_schemes": ["apiKey:Authorization", "apiKey:X-AgentMinds-Key"],
  "spec_url": "https://github.com/agentmindsdev/profile",
  "schema_url": "https://github.com/agentmindsdev/profile/blob/main/schemas/agent_report.schema.json",
  "extensions_advertised": ["agentminds_io.*", "openinference.*"],
  "links": [
    { "rel": "agent_card", "href": "/.well-known/agent-card.json" },
    { "rel": "oasf_descriptor", "href": "/.well-known/oasf-descriptor.json" },
    { "rel": "mcp_server", "href": "/.well-known/mcp-server.json" }
  ]
}
```

Cross-collector tools (adapters, analytics, dashboards) SHOULD probe
`/.well-known/arp.json` first to discover the collector's surface
before issuing typed requests.

---

## 9. Conformance levels

A sender or collector is **ARP-conformant at level N** if it implements
all features at levels ≤ N.

| Level | Features                                                       |
|-------|----------------------------------------------------------------|
| **L0** | Envelope (§2) + `report.summary` + `report.warnings[].message`|
| **L1** | L0 + `severity` + `category` + `status` per warning           |
| **L2** | L1 + `fingerprint` + `first_seen` + `last_seen` + `count`     |
| **L3** | L2 + `memory.learned_patterns` with §4.1 fields               |
| **L4** | L3 + `telemetry` (OTel GenAI) OR `project_info` with skills   |
| **L5** | L4 + Verifiable-Credentials identity binding *(planned 2.0)*  |

The current AgentMinds collector requires **L0**; recommends **L2**;
unlocks cross-site recommendation filtering at **L2**+.

---

## 10. Worked example — minimal L2 report

```json
{
  "arp_version": "1.2.1",
  "agent": "uptime_agent",
  "site_id": "site_a1b2c3",
  "session_id": "run_2026-04-27T09:15Z",
  "_meta": {
    "com.agentminds.deploy_id": "render-dep-d7n7abcd",
    "com.agentminds.region": "frankfurt"
  },
  "report": {
    "severity": "warning",
    "summary": "All 3 endpoints responding; api.example.com p95 climbed 18%.",
    "metrics": {
      "uptime_pct_24h": 99.97,
      "response_time_ms_p95": 213
    },
    "warnings": [
      {
        "fingerprint": "0c5e2f1d…64hex",
        "message": "p95 response time climbed from 180ms to 213ms over 24h.",
        "level": "warning",
        "severity": "medium",
        "status": "unresolved",
        "first_seen": "2026-04-26T03:00:00Z",
        "last_seen":  "2026-04-27T09:15:00Z",
        "count": 17,
        "category": "performance",
        "tags": { "endpoint": "api.example.com", "region": "frankfurt" }
      }
    ],
    "recommendations": [
      {
        "title": "Profile the /api/v1/connect handler hot path",
        "priority": "medium",
        "category": "performance",
        "warning_fingerprints": ["0c5e2f1d…64hex"],
        "confidence": 0.7
      }
    ],
    "scores": [
      {
        "key": "p95_within_budget",
        "data_type": "boolean",
        "value": false,
        "warning_fingerprints": ["0c5e2f1d…64hex"]
      },
      {
        "key": "p95_drift_pct",
        "data_type": "numeric",
        "value": 18.3,
        "name": "p95 24h drift",
        "compared_to_baseline_id": "run_2026-04-26T03:00Z"
      }
    ]
  }
}
```

Compare with the **Sentry-style fingerprint array** form:

```json
{
  "warnings": [{
    "message": "p95 climbed",
    "fingerprint": ["uptime_agent", "p95_drift", "{{ default }}"]
  }]
}
```

Both forms are valid in v1.1; senders pick one per payload.

---

## 11. Implementer notes

- The fingerprint normalization function is reference-implemented at
  [`master-agent-system/sync/fingerprint.py`](../master-agent-system/sync/fingerprint.py).
- The Postgres lifecycle backing tables and triggers live in
  [`master-agent-system/sync/migrations/002_warning_lifecycle.sql`](../master-agent-system/sync/migrations/002_warning_lifecycle.sql).
- Reference Python SDK: [`agentminds`](https://pypi.org/project/agentminds/) ≥ 0.4.0.
- Reference Node SDK: [`@agentmindsdev/node`](https://www.npmjs.com/package/@agentmindsdev/node) ≥ 0.3.0.

## 12. Public comment

Issues, suggestions, and PRs welcome at
[github.com/agentmindsdev/profile](https://github.com/agentmindsdev/profile)
under the label `arp-spec`. We will host an `arp-spec` discussion on the
[MCP working group](https://github.com/modelcontextprotocol/specification/discussions)
and the [AGNTCY observability WG](https://github.com/agntcy) once the
1.0 profile lands.

---

## 13. Why this is a *profile* and not a *standard* (FAQ)

**Q. Aren't you just renaming "standard" to "profile"?**
No. A standard is a contract a community of vendors agrees to and
co-maintains; a profile is one vendor's documented adoption of one
or more standards plus a small extension. Different scope, different
maintenance burden, different competitive positioning.

**Q. What if Cloudflare / Google / AWS announces "Agent Reporting
Spec 1.0" tomorrow?**
We re-emit ARP as a thin wrapper over their spec, deferring the
overlapping parts to them and keeping only the §4.1 pattern-lifecycle
extension. 30-day reorientation clause (§7) is exactly for this.

**Q. Why publish it at all then?**
Because writing the contract down forces consistency across the
collector, the SDKs, and the dashboard, and lets external integrators
build against it without reverse-engineering our HTTP traffic. That
value exists with or without anyone else adopting ARP.

**Q. Why these 5 lineages and not others (LangSmith, Helicone,
Langfuse)?**
The 5 we cite have either a foundation behind them (CNCF, Linux
Foundation), a vendor commitment to keep the spec open, or a
de-facto adoption advantage that makes co-existence safer than
competition. Vendor-internal observability schemas — even good ones
— are not anchors we can build on long-term.

---

## 14. Standards landscape — research basis (2026-04-27)

This profile is grounded in a 12-target landscape analysis. The
research compared OASF / MCP / A2A / OTel GenAI / Sentry / Anthropic
Skills / LangSmith / Arize Phoenix / Langfuse / OpenAI Agents SDK /
ACP / Cloudflare Agents against AgentMinds' delivery surface.

Key findings:

- **AgentMinds' moat is the combination of cross-site pattern share
  + fingerprint dedup + lifecycle aggregation + Beta-Bernoulli
  confidence + drift detection.** No single one of the 12 standards
  combines all of these. Sentry has fingerprint + lifecycle but is
  single-org; OASF has cross-site federation but no pattern-level
  dedup; OTel has telemetry but no opinion on lifecycle.
- **ACP merged into A2A in August 2025.** Refer only to A2A from
  here forward.
- **MCP, A2A, AGNTCY are all under Linux Foundation AI & Data.**
  They stack rather than compete (SLIM ↓ A2A ↓ MCP ↓ OASF).
- **Anthropic Claude Skills is the largest distribution channel:**
  125K stars, 277K installs/skill. AgentMinds patterns can be
  re-packaged as `SKILL.md` for Claude Code reach.
- **Cloudflare Agents is a runtime layer (Durable Objects + SQLite),
  not a spec.** AgentMinds-instrumented agents can run on it;
  there is no schema to integrate.

Read the full landscape report at
[`docs/research/STANDARDS_LANDSCAPE_v1.md`](research/STANDARDS_LANDSCAPE_v1.md)
in the closed-source agentmindsdev/agentminds repo.
