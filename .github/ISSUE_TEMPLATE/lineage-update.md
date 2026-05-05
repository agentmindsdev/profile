---
name: Lineage / upstream standard update
about: An upstream standard ARP defers to (Sentry, OTel GenAI, MCP, Skills, OASF) released a new version
title: '[LINEAGE] '
labels: lineage-update
---

## Which upstream

- [ ] [Sentry data schemas](https://github.com/getsentry/sentry-data-schemas)
- [ ] [OpenTelemetry GenAI semconv](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [ ] [Model Context Protocol](https://modelcontextprotocol.io/specification)
- [ ] [Anthropic Claude Skills](https://github.com/anthropics/skills)
- [ ] [AGNTCY OASF](https://github.com/agntcy/oasf)

## Old pinned version

<!-- From the Lineage table in README. -->

## New version / release notes

<!-- Link to the upstream release. Date matters more than version
string for moving-target specs. -->

## Does ARP need a MAJOR bump?

Per the Reorientation clause (README + spec §7): if the upstream
absorbed a primitive ARP currently owns (today: §4.1 Pattern
lifecycle), ARP MUST publish MAJOR+1 within 30 days deferring to the
upstream definition.

- [ ] Upstream change is additive — ARP MINOR bump enough
- [ ] Upstream absorbed §4.1 — MAJOR bump required
- [ ] Upstream changed semantics of an inherited primitive — MAJOR bump
- [ ] Cosmetic change only — PATCH or no bump

## Migration impact

<!-- Brief: what do existing ARP senders need to do? Or "nothing,
collectors accept both shapes during the transition window." -->
