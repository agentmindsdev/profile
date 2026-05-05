# ARP Eval Set v0 — Strawman

**Status:** strawman, open for delta. Not normative until v1.0.
**Maintainer:** AgentMinds (`agentminds.dev`).
**Spec:** companion to ARP v1.2.1 (`§4.1 Pattern`, `§3 Report`).
**License:** CC-BY-4.0.
**Discussion:** [agentmindsdev/profile#2](https://github.com/agentmindsdev/profile/issues/2)

## Why this exists

Cross-site pattern pools claim to surface "patterns worth applying."
That claim cannot be tested without a benchmark of known incidents
+ expected retrieval behavior. Without it, pilots are wiring
recommendation engines into production remediation loops blind.

This eval set is the open contract: a row schema + worked examples
that any conformant collector (ARP §9 Conformance L1+) can run
against and report recall/noise ratios on.

## Row schema (v0)

```yaml
id: <stable-uuid-or-slug>
input_state:
  site_type: <web|saas|api|cli|other>
  detected_tech: [<lowercase tokens>]
  observation: <human-readable description of what the agent observed>
  signals:                # optional structured signals
    p95_latency_ms: <int>
    error_rate_pct: <float>
    # ... domain-specific
expected:
  top_n: <int>            # how many results to evaluate
  required_fingerprints:  # patterns that MUST appear in top-N
    - <pattern_fingerprint_1>
    - <pattern_fingerprint_2>
  forbidden_fingerprints: # patterns that MUST NOT appear in top-N
    - <pattern_fingerprint_3>
  recall_min: <0.0-1.0>   # min recall threshold for this row
  noise_max: <0.0-1.0>    # max false-positive ratio in top-N

policy_version: "v1.2.1"  # ARP version this row was authored against
notes: <free-form rationale>
```

## Field rationales

**input_state** — captures everything a recommendation engine has
when invoked. Mirrors ARP §3 Report shape so evaluation rows can
be derived from real reports.

**required_fingerprints / forbidden_fingerprints** — both directions
matter. A retrieval system that returns the right pattern but also
returns three irrelevant ones in top-N is failing differently than
one that returns no relevant patterns.

**recall_min / noise_max** — explicit thresholds per row, because
"acceptable recall" varies by stakes. A `security-critical` pattern
needs recall ≥0.95; a `performance-tip` pattern can tolerate recall
0.7. Single global threshold flattens this distinction.

**policy_version** — without it, a row scored under v1 scoring
weights is incomparable to v2. Pinning the policy version makes
temporal drift explicit.

## Worked examples

### Row 001: Missing HSTS header on web app

```yaml
id: web-hygiene-001-missing-hsts
input_state:
  site_type: web
  detected_tech: [nginx, react, postgresql]
  observation: "GET / returns 200 but no Strict-Transport-Security header"
  signals:
    has_https: true
    hsts_present: false
expected:
  top_n: 5
  required_fingerprints:
    - "tier1::security::strict-transport-security-header"
  forbidden_fingerprints:
    - "tier1::performance::compress-images"  # off-domain
    - "tier1::seo::add-meta-description"     # off-category
  recall_min: 0.95
  noise_max: 0.40
policy_version: "v1.2.1"
notes: "Tier-1 hygiene check, must be top-of-list when flagged."
```

### Row 002: Agent retry storm on rate-limited API

```yaml
id: agent-failure-002-retry-storm
input_state:
  site_type: api
  detected_tech: [fastapi, openai, redis]
  observation: "OpenAI 429 rate limit errors, agent retried 47 times in 30s"
  signals:
    error_rate_pct: 23.0
    p95_latency_ms: 45000
expected:
  top_n: 5
  required_fingerprints:
    - "production_observed::ai_agents::exponential-backoff-on-429"
  forbidden_fingerprints:
    - "tier1::seo::*"  # wildcard not in v0 schema, illustrative
  recall_min: 0.85
  noise_max: 0.30
policy_version: "v1.2.1"
notes: "Production-observed pattern from real harvest; should rank above any
documented variant of the same advice."
```

### Row 003: Soft overlap with site's own solved pattern

```yaml
id: filter-003-soft-overlap-keep
input_state:
  site_type: web
  detected_tech: [postgresql, fastapi]
  observation: "DB connection leak warning"
  solved_pattern_words: [rollback, memory]
expected:
  top_n: 5
  required_fingerprints: []          # no recommendation expected to surface
  forbidden_fingerprints: []
  recall_min: 0.0
  noise_max: 0.50
  flag_solved_expected: true         # extension: filter behavior assertion
policy_version: "v1.2.1"
notes: "Tests Soft tier of decide_warning. Hits ≥2 content-word overlap with
own solved set, Jaccard <0.5. Should keep + flag, not drop."
```

## Open questions for delta discussion

1. **Granularity of `forbidden_fingerprints`** — wildcards (`tier1::seo::*`)
   useful but break stable identity. Defer to v0.1?

2. **Cross-tenant rows** — should the eval set include rows that test
   visibility tier compliance? E.g., "this Tier-3 pattern MUST NOT appear
   in any top-N response except for the originating tenant." Probably
   needs a separate eval-set namespace.

3. **AgentMart parallel** — workflow assets need a comparable schema.
   Likely fields: `asset_class`, `compatibility_constraints`,
   `determinism_required`. Same `recall_min/noise_max` mechanic. Schema
   delta welcome.

4. **Scoring policy versioning** — `policy_version: v1.2.1` pinned per
   row, but how do we mark "this row's expected behavior changes under
   policy v1.3"? Possible: `expected_by_policy: { v1.2.1: {...}, v1.3: {...} }`.

## How to contribute

This is a strawman, not a spec. To propose changes:

1. Open an issue at [agentmindsdev/profile/issues] referencing this file
2. Or fork → edit → PR with rationale in commit message
3. Or comment on [issue #2](https://github.com/agentmindsdev/profile/issues/2)

---

*Drafted 2026-05-05 by AgentMinds. Open-spec, CC-BY-4.0. No single owner.*
