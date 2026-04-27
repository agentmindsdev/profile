---
name: agentminds-status
description: Reports current AgentMinds health by combining the public /health endpoint with the local data/central_agents/*.json snapshots. Trigger on phrases like "agentminds status", "sistem durumu", "health check", "sağlık kontrolü", "agentminds çalışıyor mu", "system health".
allowed-tools: Read, Glob, Bash, WebFetch
---

# AgentMinds — system status

Single command system status. Combines:
1. The public health endpoint (`api.agentminds.dev/health`)
2. The local self-monitoring agent JSON snapshots
   (`master-agent-system/data/central_agents/*.json`)
3. The A2A AgentCard
   (`api.agentminds.dev/.well-known/agent-card.json`)

Returns ONE table with everything that matters, prioritising
critical signals.

## Trigger phrases

English or Turkish:

- "agentminds status" / "system health" / "agentminds çalışıyor mu"
- "sistem durumu" / "sağlık kontrolü" / "health check"
- "is agentminds up" / "agentminds yaşıyor mu"

## Steps

### 1. Hit the health endpoint

```bash
curl -s https://api.agentminds.dev/health | jq '.'
```

Capture: `status`, `sites`, `last_pipeline`, `open_circuits`,
`recent_alerts_1h`, `uptime`.

### 2. Check the A2A AgentCard

```bash
curl -s https://api.agentminds.dev/.well-known/agent-card.json | jq '.skills | length, .extensions["agentminds.profile"]'
```

This proves the AgentCard is live + ARP profile pointer is set.

### 3. Read local self-monitoring snapshots

If running in the AgentMinds repo, also read:

```bash
ls master-agent-system/data/central_agents/*.json
```

For each agent (`uptime_agent.json`, `freshness_agent.json`,
`content_agent.json`, `pipeline_agent.json`, `eval_agent.json`,
`growth_agent.json`):

- Read the JSON
- Extract: `severity`, `ran_at`, `summary`, `warnings` count,
  `metrics` highlights

Note any agent whose `ran_at` is more than its expected interval
ago (uptime: 15min; freshness/pipeline: 6h; content/eval: 24h;
growth: 7d). Stale = the cron is broken; flag it.

### 4. Render single status table

Output format (Turkish-friendly, monospace-aligned):

```
AgentMinds — Sistem Durumu                       (UTC <now>)

PUBLIC ENDPOINTS
  https://agentminds.dev          UP    245ms
  https://api.agentminds.dev      UP    120ms
  /health                         OK    sites=42  pipeline=success
  /.well-known/agent-card.json    OK    skills=3 profile=ARP/1.0
  /sync/ingest/otel               OK    auth-required (expected)

CENTRAL AGENTS (last run)
  uptime_agent      info     2 min ago      web/api/scan all UP
  freshness_agent   warning  4h ago         pool 8h old (>6h budget)
  content_agent     info     12h ago        18 posts, 0 stale
  pipeline_agent    info     3h ago         pipeline=success in 22s
  eval_agent        info     11h ago        67% effectiveness
  growth_agent      info     2d ago         42 sites (+3 wk)

CRITICAL SIGNALS                              0
WARNINGS (last 1h)                            1
OPEN CIRCUITS                                 0
RECENT ALERTS (1h)                            0

OVERALL: HEALTHY  (1 warning — freshness_agent pool age)
```

### 5. Report any criticals first

If any agent reports `severity: critical` or `fatal`, surface it AT
THE TOP before the table, in a red callout box format.

If `/health` returns non-200 or times out, that's the only thing
the user needs to see — show "AgentMinds API DOWN — last known
status from local snapshots: ..." and skip everything else.

## Hard rules

1. **Never invent metrics.** If an agent JSON is missing or its
   `ran_at` is stale, report exactly that — don't fill in numbers.
2. **Per CLAUDE.md ABSOLUTE RULES**: if the API is unreachable, say
   "bağlanılamadı" — never hallucinate uptime or pipeline status.
3. **Stale agents are a real signal.** A `freshness_agent` that
   hasn't run in 12h is louder than any single warning.
4. **Show the ARP version.** The user trusts that the system
   speaks ARP; surface `extensions["agentminds.profile"].version`
   from the AgentCard.

## Reference

- A2A AgentCard: https://api.agentminds.dev/.well-known/agent-card.json
- ARP spec: https://github.com/agentmindsdev/profile
- Companion skills: `agentminds-agents` (manual run + history),
  `agentminds-pool` (knowledge pool inspection).
