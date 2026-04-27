# AgentMinds Skills

Reference SKILL.md files demonstrating ARP §6.1 SkillManifest in
practice. These are the skills AgentMinds itself ships — copy-paste-
adaptable for your own collector.

## Index

| Skill | Purpose | Distribution |
|---|---|---|
| [agentminds-push-report](agentminds-push-report/SKILL.md) | Walks the user's project and pushes an ARP-conformant report | Submitted to anthropics/skills#1049 |
| [agentminds-status](agentminds-status/SKILL.md) | Combined health snapshot from `/health` + AgentCard + central agent JSON | Reference |
| [agentminds-agents](agentminds-agents/SKILL.md) | List + manually trigger AgentMinds' 6 central agents | Reference |
| [agentminds-pool](agentminds-pool/SKILL.md) | Inspect the AgentMinds knowledge pool stats | Reference |

## Format

All four follow the agentskills.io / Anthropic Claude Skills format:

```yaml
---
name: <kebab-case>
description: <what + when to invoke>
allowed-tools: Read, Glob, Bash, ...
---

# <Title>
... markdown body ...
```

See [`AGENT_REPORTING_PROFILE.md` §6.1](../AGENT_REPORTING_PROFILE.md)
for the full SkillManifest spec including the v1.1.1 additions
(`metadata`, `auto_invocable`) and v1.2 additions (`oasf_skill_ids`,
`agentskills_io_canonical_name`, `execution_tier`).

## License

CC-BY-4.0 (see top-level [LICENSE](../LICENSE)).
