# ARP Examples

Concrete payloads at every conformance level. Each file is the smallest
report that satisfies its level so you can see what's required at each
step.

| File | Level | What it demonstrates |
|---|---|---|
| [L0-minimal.json](L0-minimal.json) | L0 | Smallest valid report — just `agent` + `report.summary` + a single `warning.message` |
| [L2-with-fingerprint.json](L2-with-fingerprint.json) | L2 | Adds Sentry-style issue lifecycle (`fingerprint`, `first_seen`, `last_seen`, `count`, `status`). This is the level AgentMinds recommends for production senders. |
| [L3-with-patterns.json](L3-with-patterns.json) | L3 | Adds §4.1 Pattern objects under `memory.learned_patterns` — the only primitive ARP owns. Required to participate in cross-site recommendation filtering. |
| [L4-with-otel.json](L4-with-otel.json) | L4 | Adds OpenTelemetry GenAI `telemetry.spans[]` and `telemetry.metrics[]`. |

## Validate locally

```bash
npx ajv-cli@5 validate \
  -s ../schemas/agent_report.schema.json \
  -d 'examples/*.json' \
  --spec=draft2020 --strict=false
```

(or with Python)

```bash
pip install jsonschema
python -c "import json,jsonschema,sys; \
  schema=json.load(open('schemas/agent_report.schema.json')); \
  [jsonschema.validate(json.load(open(f)), schema) for f in sys.argv[1:]]" \
  examples/*.json
```

CI runs this validation on every push (see
[.github/workflows/validate-examples.yml](../.github/workflows/validate-examples.yml)).

## Notes

- All examples set `arp_version: "1.2.0"` to pin the profile version
  the payload was authored against. Collectors use this to apply the
  correct lifecycle/migration rules.
- Top-level `_comment` fields are documentation only; they're allowed
  because the schema sets `additionalProperties: true` at the root.
  Strip them in production payloads.
- Timestamps are RFC3339 UTC.
