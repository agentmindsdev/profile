# agentmindsdev/profile — Claude Code Instructions

This repo holds the public **AgentMinds Reporting Profile (ARP)** spec
under CC-BY-4.0. It is the open contract that backend collectors and
agent-side senders both implement.

Primary artifacts:

- `AGENT_REPORTING_PROFILE.md` — normative spec (sections numbered)
- `schemas/agent_report.schema.json` — JSON Schema (draft 2020-12)
- `examples/L0..L4-*.json` — conformance examples per profile level
- `eval-set/v0-strawman.md` — eval matrix for collector/sender pairs
- `CHANGELOG.md` — Keep-a-Changelog format, semver
- `skills/` — reference SKILL.md examples (ARP §6.1)

## Release Process — CRITICAL RULE

This repo uses `.github/workflows/publish-release.yml` to auto-publish
GitHub Releases when version tags are pushed.

**ANNOTATED TAGS ARE MANDATORY.** Lightweight tags will fail the
workflow with "Tag has no annotated subject."

### Correct (annotated):

```bash
git tag -a v1.3.1 -m "ARP v1.3.1 — patch release title

Description body becomes the release notes.
- Bullet 1
- Bullet 2

Backwards compatible."

git push origin v1.3.1
```

### Wrong (lightweight — will fail):

```bash
git tag v1.3.1           # ❌ no -a flag, no -m message
git push origin v1.3.1   # workflow will reject
```

### What happens after `git push origin <tag>`:

1. GitHub Actions triggers `publish-release.yml`
2. Tag's annotated subject becomes the Release title
3. Tag's annotated body becomes the Release notes
4. Pre-release detection: `vX.Y.Z-*` (with hyphen) → prerelease,
   `vX.Y.Z` (clean semver) → latest
5. Release appears in https://github.com/agentmindsdev/profile/releases
   within ~30 seconds

### Manual trigger (for retroactive publish):

GitHub Actions tab → "Publish Release" → Run workflow → enter tag name.
Useful if a tag exists but its Release is missing.

## Versioning

- Spec version is the file `AGENT_REPORTING_PROFILE.md` § header line.
- Schema version is the `$id` query string (`?v=1.3.0`).
- Both must match the tag being released.
- CHANGELOG.md must have a section for the version before tagging.
- README.md "Latest" badge must reference the tag.

## Backwards compatibility

ARP follows strict semver:

- **MAJOR**: breaking change to schema (remove field, change type, narrow
  enum). Requires migration note in CHANGELOG.
- **MINOR**: additive (new optional field, new $def, new informative §).
  v1.x senders MUST continue to validate against v1.y collectors and
  vice versa for any x ≤ y.
- **PATCH**: clarifications, informative notes, fixes that do not change
  the wire shape.

Never delete or rename a property without bumping MAJOR. Add the new
property and deprecate the old in CHANGELOG with a removal target
version.
