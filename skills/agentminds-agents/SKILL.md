---
name: agentminds-agents
description: Lists the 6 self-monitoring central agents (uptime, freshness, content, pipeline, eval, growth), shows when each last ran, and triggers a manual run for any agent on request. Trigger on phrases like "agentminds agents", "ajanları çalıştır", "run agents", "ajan durumu", "central agents".
allowed-tools: Read, Glob, Bash
---

# AgentMinds — central agents inventory + manual run

The 6 self-monitoring central agents (per the
clever-booping-lantern.md plan) live in
`master-agent-system/central_agents/`. Each is a standalone Python
script that reads its target, builds an ARP-conformant report, and
pushes it to AgentMinds Central.

## What this skill does

1. **List** all 6 agents with their last-run timestamps + severity.
2. **Trigger** a manual run for any agent the user names.
3. **Show** the freshly-produced report after a manual run.

## Trigger phrases

- "agentminds agents" / "central agents" / "list agents"
- "ajanları çalıştır" / "ajan listesi" / "ajan durumu"
- "run uptime agent" / "uptime agent çalıştır"

## The 6 agents

| Agent | Cadence | Script | What it monitors |
|---|---|---|---|
| `uptime_agent` | 15 min | `central_agents/uptime_agent.py` | agentminds.dev + api up/down + SSL + scan pipeline |
| `freshness_agent` | 6 h | `central_agents/freshness_agent.py` | Knowledge pool staleness + per-site/agent recency |
| `content_agent` | daily 10:00 | `central_agents/content_agent.py` | Blog post count + 90-day AEO freshness |
| `pipeline_agent` | 6 h (15min after pipeline) | `central_agents/pipeline_agent.py` | Pipeline status + guardian circuits + alerts |
| `eval_agent` | daily 11:00 | `central_agents/eval_agent.py` | Recommendation effectiveness + feedback quality |
| `growth_agent` | weekly Mon 07:00 | `central_agents/growth_agent.py` | Site count + pattern growth + storage |

The Task Scheduler `.bat` files at the repo root
(`run_<agent>_agent.bat`) are wired to these cadences.

## Steps

### 1. List agents + last runs

```bash
ls master-agent-system/data/central_agents/*.json
```

For each JSON, read it and extract:

```python
{
  "agent": <agent_name>,
  "ran_at": <iso timestamp>,
  "severity": <info|warning|error|fatal>,
  "warning_count": len(report["warnings"]),
  "summary": <one-line>,
}
```

Render as a sortable table by severity, then by recency.

### 2. Detect stale agents

For each agent, compare `ran_at` against the expected cadence
(table above). If the cadence is exceeded by 2x, flag the agent
as `stale`. Example: uptime expected every 15min — stale at 30+min.

Output:

```
STALE AGENTS (cron is broken or paused):
  growth_agent     last run 18d ago (cadence: 7d)
```

### 3. Manual run (when requested)

If the user names an agent ("run uptime agent" /
"uptime agent çalıştır"):

```bash
python master-agent-system/central_agents/<name>_agent.py
```

Capture stdout (the agent prints its result JSON). Show:

- Duration
- Severity
- Warning count
- 3 most-severe warnings (full text)
- ARP envelope shape sanity (presence of `arp_version`, `status`,
  `category` per warning)

### 4. Optional: dry-run all 6

If the user says "run all agents" / "tüm ajanları çalıştır":

```bash
for agent in uptime freshness content pipeline eval growth; do
  python master-agent-system/central_agents/${agent}_agent.py
done
```

Run sequentially (NOT parallel — they share the data/ directory).
Total time ~60-90s. Render a final summary table after all 6
finish.

## Hard rules

1. **Never run agents without the user asking.** Listing is safe;
   manual run is destructive (it pushes new reports to the API).
2. **Surface the cron health.** Stale agents = the bigger problem
   than any single warning.
3. **Per ARP §3.3:** every warning shown should have `category`
   and `status`. If an agent's output is missing those, that
   agent is not yet ARP-conformant — flag it.
4. **CLAUDE.md ABSOLUTE RULES apply:** never fabricate agent runs;
   if a Python file is missing, say so.

## Reference

- ARP §3 + §4: `docs/AGENT_REPORTING_PROFILE.md`
- ARP helper used by all 6 agents:
  `master-agent-system/central_agents/arp_helper.py`
- Companion: `agentminds-status` (system overview),
  `agentminds-pool` (knowledge pool inspection).
