---
id: github.com/MeteorsLiu/skillhub-publish-skill
name: SkillHub Publish Skill
description: Submit a user-provided AI agent skill to the SkillHub discovery center for review and inclusion. Use when the user wants to publish, submit, register, list, or add a skill to SkillHub so other agents can discover it.
tags:
  - skillhub
  - publish
  - registry
  - discovery
  - submission
dependencies:
  tools: []
  skills: []
---

Use this workflow to submit a skill to the SkillHub discovery center.

## Required Inputs

Ask for any missing required input:

- Skill ID: a Git-based SkillHub ID, for example `github.com/owner/repo` or `github.com/owner/repo/path/to/skill`
- Version: optional. Use `latest` if the user does not provide one.

Do not invent the skill ID, repository, subdirectory, or version.

## Validate Before Registering

Before registering, verify that the skill can be fetched and parsed:

```bash
tmp=$(mktemp -d)
skillhub fetch "<skill-id>@<version>" "$tmp"
rm -rf "$tmp"
```

If `skillhub` is not on PATH but a local binary is known, use that binary.

Stop and report the error if fetch fails. Common causes:

- The repository or subdirectory does not exist
- The version or tag does not exist
- `SKILL.md` is missing or invalid
- The repository is private or requires credentials

## Register

Submit the skill to the discovery center:

```bash
curl -fsS -X POST "${SKILLHUB_DISCOVERY_HOST:-http://218.11.5.155:8399}/v1/register" \
  -H 'Content-Type: application/json' \
  -d '{"id":"<skill-id>","version":"<version>"}'
```

Use `latest` for version when the user did not provide one.

## Response To User

Explain the result precisely:

- If registration succeeds, say the skill was submitted to SkillHub for review and discovery indexing.
- Do not say it is approved unless the API response or later search result proves it.
- If registration fails, include the API error and the validated skill ID/version.

## Optional Verification

If the user asks whether the skill is already discoverable, search for it after registration:

```bash
curl -fsS -X POST "${SKILLHUB_DISCOVERY_HOST:-http://218.11.5.155:8399}/v1/search" \
  -H 'Content-Type: application/json' \
  -d '{"id":"<skill-id>","limit":1}'
```

If it is not returned, say it may still be pending review or indexing.
