# AgentMinds Reporting Profile (ARP)

**Version:** 1.0.0
**Status:** Internal profile — open for public comment
**Maintainer:** AgentMinds (`api.agentminds.dev`)
**License:** CC-BY-4.0

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

## 1. Glossary

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

---

## 2. The Envelope

Every report sent to AgentMinds (or any ARP-compliant collector) is a
single JSON object with this top-level shape:

```jsonc
{
  "arp_version": "1.0",                      // optional but recommended
  "agent": "<string>",                       // required
  "site_id": "<string>",                     // required (set by transport if omitted)
  "report": { ... },                         // required — see §3
  "memory": { ... },                         // optional — see §4
  "telemetry": { ... },                      // optional — see §5
  "project_info": { ... },                   // optional — see §6
  "schema": {                                // optional — see §7
    "name": "agentminds.arp",
    "version": "1.0"
  }
}
```

For backwards compatibility, collectors MUST also accept the legacy
key `ars_version` as an alias for `arp_version` and `agentminds.ars`
as the schema name; both decay to the same v1.0 envelope.

The transport (HTTP POST `/sync/ingest`, MCP tool call, etc.) MAY add
`api_key`, `received_at`, and `correlation_id` siblings; these are not
part of ARP itself.

---

## 3. `report` — Current state

```jsonc
{
  "severity": "fatal" | "error" | "warning" | "info" | "debug",
  "summary": "<2-3 sentence human-readable status>",
  "score": 0..100,                           // optional health score
  "metrics": { ... },                        // free-form key/value, see §3.2
  "warnings":        [Warning],              // see §3.3
  "recommendations": [Recommendation]        // see §3.4
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

- **`fingerprint`** — Stable identity across reports and sites. SHOULD be
  computed as `sha256(agent_name + "::" + normalize(message))` where
  `normalize` lowercases, strips whitespace, and removes volatile
  numerics (timestamps, request IDs, percentages). If the SDK omits it,
  the collector MUST compute and persist a server-side fingerprint
  before storing the row.
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

### 6.1 `SkillManifest` (Anthropic Claude Skills + OASF aligned)

A single skill is described by parsing its `SKILL.md` file's YAML
frontmatter and the file body. The on-disk format is
[Claude Skills](https://github.com/anthropics/skills); the wire format
in ARP is:

```jsonc
{
  "name":         "<string, required>",
  "description":  "<string, required>",
  "version":      "<semver, optional>",
  "license":      "<SPDX id, optional>",
  "allowed_tools": ["Read", "Edit", ...],    // Claude Skills field
  "triggers":     ["...", "..."],            // ARP extension
  "compatibility": {
    "agent_runtimes": ["claude-code>=1.0", "agentminds-cli>=0.1"]
  }
}
```

Only `name` and `description` are required, matching Anthropic's
SKILL.md spec. AgentMinds publishes its own skills (`agentminds-status`,
`agentminds-agents`, `agentminds-pool`, `agentminds-connect`) in this
format.

---

## 7. Versioning & evolution

- ARP uses **semantic versioning**: `MAJOR.MINOR.PATCH`.
- **MINOR** bumps add OPTIONAL fields. Senders SHOULD continue to work
  against older collectors; collectors SHOULD ignore unknown fields.
- **MAJOR** bumps remove or rename REQUIRED fields. Migration notes
  ship in `docs/arp-migration-<old>-to-<new>.md`.
- Senders SHOULD include `arp_version` at the envelope root. Collectors
  MUST default to the latest minor of the same major when omitted.
- The canonical JSON Schema is at
  [`docs/schemas/agent_report.schema.json`](schemas/agent_report.schema.json).
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
  "arp_version": "1.0",
  "agent": "uptime_agent",
  "site_id": "site_a1b2c3",
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
    ]
  }
}
```

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
[github.com/UzunGridera/Agentminds](https://github.com/UzunGridera/Agentminds)
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
