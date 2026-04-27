---
name: agentminds-pool
description: Inspects the AgentMinds knowledge pool — pattern counts by category, recent additions, staleness, network position, and tier-1/tier-2/tier-3 distribution. Trigger on phrases like "knowledge pool", "bilgi havuzu", "patterns", "pattern listesi", "pool durumu", "pool stats".
allowed-tools: Read, Glob, Grep, Bash, WebFetch
---

# AgentMinds — knowledge pool inspector

AgentMinds' moat is the cross-site pattern pool — the file at
`master-agent-system/data/knowledge_pool/pool.json` holds the
canonical aggregated patterns the system has learned across all
connected sites. This skill makes it browseable.

## What this skill does

1. Summarise the pool: total pattern count, breakdown by category
   + tier + status, freshness percentile, top sources.
2. Show recently-added patterns (last 7d).
3. Surface pool staleness (when was it last rebuilt?).
4. (Optional) compare pool stats against `/health` site count to
   spot ratio drift.

## Trigger phrases

- "knowledge pool" / "patterns" / "show pool"
- "bilgi havuzu" / "pattern listesi" / "pool durumu" / "pool stats"
- "what's in the pool" / "havuzda ne var"

## Steps

### 1. Read the pool JSON

```bash
cat master-agent-system/data/knowledge_pool/pool.json | head -50
```

Capture: file mtime (= last rebuild), total `patterns[]` length,
the categories present.

If the file doesn't exist or is older than 24h, surface that as
the loudest signal — the pipeline that builds the pool is broken.

### 2. Summarise

Render a single overview block:

```
AgentMinds Knowledge Pool                        (built <ts>, age <X>h)

TOTAL PATTERNS                <count>
TIER DISTRIBUTION
  Tier 1 (universal)          <n>
  Tier 2 (stack-specific)     <n>
  Tier 3 (one-off)            <n>

CATEGORY DISTRIBUTION
  performance                 <n>
  security                    <n>
  seo                         <n>
  content                     <n>
  quality                     <n>
  compliance                  <n>
  infra                       <n>
  general                     <n>

STATUS DISTRIBUTION (per ARP §4.1)
  active                      <n>
  solved                      <n>
  obsolete                    <n>

CROSS-SITE COVERAGE
  Patterns observed by ≥3 sites    <n>  (Tier 1 candidates)
  Patterns observed by 2 sites     <n>
  Patterns observed by 1 site      <n>
```

### 3. Recent additions (last 7d)

Filter `patterns[]` where `created_at` (or `last_confirmed_at`) is
within 7 days. Sort by date desc; show top 10:

```
RECENT PATTERNS (last 7d)
  2026-04-26  perf       site=mimari.ai     "p95 climbed after deploy"
  2026-04-25  security   site=gridera.ai    "missing CSP header"
  ...
```

### 4. Cross-reference health endpoint

```bash
curl -s https://api.agentminds.dev/health | jq '.sites'
```

Compare:
- Sites in pool data: distinct site_ids in `patterns[].source_site_id`
- Sites in /health: live count

If the gap is large (e.g., 42 sites live but only 8 contribute
patterns), flag it — most sites aren't pushing data.

### 5. Show ARP-conformance level of pool entries

For a sample of 10 random patterns, check whether they have:

- `fingerprint` (lowercase hex SHA-256, 64 chars) → ARP L2
- `status` ∈ {active, solved, obsolete} → ARP §4.1
- `first_seen` / `last_seen` ISO timestamps
- `confidence` ∈ [0.0, 1.0]

Surface as percentage. Older patterns may pre-date ARP and lack
fingerprints; that's expected — the migration script
`master-agent-system/sync/backfill_warning_lifecycle.py` has been
run for warnings, but pattern backfill is a separate follow-up.

## Hard rules

1. **Never expose patterns to unauthenticated callers.** The pool
   is the moat (per the `learned_patterns are PRIVATE` rule). If
   the user is asking about another organisation's specific
   patterns, refuse.
2. **Don't print full pattern bodies if there are 50+ recent
   additions.** Summarise; offer to drill down on category /
   site / fingerprint.
3. **CLAUDE.md ABSOLUTE RULES:** if `pool.json` is missing, say so;
   never fabricate counts.
4. **Stale pool = louder signal than any single pattern.** If
   `mtime < now() - 24h`, the freshness_agent has either failed
   or the pipeline is paused. Lead with that.

## Reference

- Pool source: `master-agent-system/data/knowledge_pool/pool.json`
- Pool schema notes: `master-agent-system/sync/migrations/001_patterns_and_events.sql`
- ARP §4.1 Pattern object: `docs/AGENT_REPORTING_PROFILE.md`
- Companion: `agentminds-status` (overall health),
  `agentminds-agents` (run the freshness_agent if pool is stale).
