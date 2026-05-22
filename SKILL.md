---
id: github.com/MeteorsLiu/skillhub-publish-skill
name: SkillHub Publish Skill
description: Submit a user-provided SkillHub skill to the Discovery Center for review and indexing. Use when the user wants to publish, submit, register, list, add, or make a SkillHub skill discoverable by agents.
tags:
  - skillhub
  - publish
  - registry
  - discovery
  - submission
dependencies:
  tools:
    - curl
  skills: []
skills: []
---

# SkillHub Publish Skill

Use this workflow to submit a SkillHub root skill to the Discovery Center.

The user may provide a skill id, a GitHub URL, or a raw `SKILL.md` URL. Convert it to the SkillHub root skill id, validate the package against the SkillHub design rules, then register it.

## Inputs

Required:

- Root skill id, for example `github.com/owner/repo` or `github.com/owner/repo/path/to/root-skill`

Optional:

- Version tag to publish. A concrete tag is required for release publishing.
- Discovery host. Use `SKILLHUB_DISCOVERY_HOST` if set, otherwise `http://218.11.5.155:8399`.

Do not invent the repository, subdirectory, or version. If the user gives only an ambiguous name, ask for the GitHub repository or SkillHub id.

## ID Rules

Accept only root skill ids for registration.

Valid examples:

```text
github.com/bob/rednote-skill
github.com/acme/clawhub/social/publish-post
```

Rules from the SkillHub design:

- A root skill id is the install, version, and dependency unit.
- A sub-skill id is `root skill id + "/" + relative sub-skill path`.
- Register the root skill, not an individual sub-skill.
- Large multi-skill repositories must use explicit root ids in `SKILL.md`.
- A single-root repository may omit `id`; the service may fall back to `github.com/{owner}/{repo}`.

If the user gives a raw URL like:

```text
https://raw.githubusercontent.com/owner/repo/refs/heads/main/path/to/SKILL.md
```

derive only a candidate id:

```text
github.com/owner/repo/path/to
```

Then validate it with `skillhub fetch`. Do not assume the candidate is correct until fetch and parse succeed.

## Version Rules

If the user provides a version, it must follow the SkillHub design:

```text
v1.2.3
v1.2.3-rc.1
```

For large repositories where the root skill lives in a subdirectory, tags are expected to be scoped:

```text
social/publish-post/v1.2.3
```

For publishing, do not use `latest` as the final registered version. If the user does not provide a version tag, choose the next patch version after checking remote tags, then state the chosen version before tagging.

Use `latest` only for exploratory validation before the release tag exists.

## Commit And Tag Before Registering

Publishing order:

```text
edit skill package
  -> commit changes to the skill repository
  -> create the release tag
  -> push commit and tag
  -> validate skillhub fetch id@tag
  -> register id@tag with Discovery Center
```

Do not register before the commit and tag are available remotely. Discovery workers fetch by id and version; if the tag does not exist remotely, registration cannot resolve the package.

Hard release rule:

```text
Every new publish must use a new version tag.
```

Never register an existing tag as a new publish. An existing tag may only be used to verify or inspect that old release. If the skill content changed, create and push a higher version tag first.

Version selection:

```text
first release:
  v0.1.0

patch/content update:
  increment patch, for example v0.1.0 -> v0.1.1

minor feature expansion:
  increment minor, for example v0.1.1 -> v0.2.0

breaking behavior or contract change:
  increment major, for example v0.2.0 -> v1.0.0
```

Before tagging, inspect remote tags:

```bash
git ls-remote --tags origin
```

If the selected version already exists remotely, stop and choose a higher version. Do not delete, move, or overwrite a published tag.

For a single-root repository:

```bash
git add SKILL.md skills/ references/ assets/ scripts/
git commit -m "Update SkillHub publish skill"
git tag v1.2.3
git push origin HEAD
git push origin v1.2.3
```

For a root skill in a subdirectory of a large repository, tag with the root skill path:

```bash
git tag path/to/root-skill/v1.2.3
git push origin path/to/root-skill/v1.2.3
```

Before creating a tag:

- Check existing tags with `git tag --list` or `git ls-remote --tags <repo>`.
- Do not reuse an existing tag for different content.
- If the requested tag already exists, choose or ask for a higher version.
- If the working tree contains unrelated changes, commit only the skill package files needed for the release.

## Validate Before Registering

Run `skillhub fetch` after the commit and tag are pushed, before calling the Discovery Center. If `skillhub` is not on `PATH` but a local SkillHub binary path is known, use that binary.

```bash
tmp=$(mktemp -d)
skillhub fetch "<skill-id>@<version>" "$tmp"
rm -rf "$tmp"
```

The fetch must prove that:

- The id maps to a real Git repository and optional subdirectory.
- The requested version or latest resolvable version exists.
- The root `SKILL.md` can be parsed.
- The resulting metadata includes an id, name, description, version, tags, and dependencies.

Stop if validation fails. Report the exact failing id/version and the command error.

Common failures:

- Repository does not exist.
- Subdirectory does not contain root `SKILL.md`.
- Version or scoped tag does not exist.
- `SKILL.md` frontmatter is invalid.
- A multi-skill repository is missing an explicit root `id`.
- The repository is private or requires credentials.

## Validate Skill Metadata

After fetch succeeds, inspect the returned JSON.

Required metadata:

```text
id
name
description
version
```

Recommended metadata:

```text
tags
dependencies.tools
dependencies.skills
skills
```

Check design conformance:

- `id` should be the root skill id being registered.
- `description` should describe the capability clearly enough for search.
- `dependencies.tools`, `dependencies.skills`, and `skills` should follow the detailed checks below.

If metadata is incomplete but parse succeeds, explain the issue and ask whether to continue. Do not silently publish a low-quality skill.

## Validate Tool Dependencies

`dependencies.tools` declares required command-line tools. SkillHub reports these tools; it does not install or verify them.

Check:

- Include commands the skill workflow actually needs, such as `ffmpeg`, `yt-dlp`, `rg`, `python`, or `node`.
- Use executable command names, not package names unless they are the same command.
- Do not list MCP tools, SkillHub tools, OpenClaw tools, web search, browser, or host capabilities.
- Do not list optional tools unless the `SKILL.md` clearly marks them optional in the workflow.
- Do not leave out a command if the skill tells the agent to run it.

Good:

```yaml
dependencies:
  tools:
    - ffmpeg
    - yt-dlp
```

Bad:

```yaml
dependencies:
  tools:
    - web_search
    - skillhub__load
    - "a video downloader"
```

If a required command is missing from `dependencies.tools`, ask the user to update the skill before publishing or explicitly confirm that the omission is intentional.

## Validate External Skill Dependencies

`dependencies.skills` declares external root skill dependencies only.

Check:

- Each dependency should be written as `root-skill-id@version`.
- The version is a minimum required version.
- The dependency id should point to a root skill, not a sub-skill.
- Local sub-skills listed under `skills` must not be duplicated in `dependencies.skills`.
- Do not use `latest` for dependency versions; use an explicit minimum version.
- Do not list skills that are only loosely related or optional.

Good:

```yaml
dependencies:
  skills:
    - github.com/acme/clawhub/common/image-tools@v1.2.0
    - github.com/acme/clawhub/common/uploader@v2.1.0
```

Bad:

```yaml
dependencies:
  skills:
    - github.com/acme/clawhub/social/publish-post/draft-post@v1.2.0
    - github.com/acme/optional-helper@latest
```

If a dependency points to a sub-skill, ask the user to change it to the owning root skill.

## Validate Local Sub-Skills

`skills` exposes local sub-skills under the root skill's `skills/` directory.

Check the repository layout when the root declares sub-skills:

```text
root-skill/
  SKILL.md
  skills/
    draft-post/
      SKILL.md
    publish-final/
      SKILL.md
```

Rules:

- Each entry in root `skills` should match a direct directory under `skills/`.
- Each listed sub-skill directory must contain `SKILL.md`.
- If root `skills` is omitted, direct children under `skills/` may be exposed by default.
- If root `skills` is present, only listed entries are exposed, in declaration order.
- A sub-skill usually does not need its own explicit `id`.
- A sub-skill inherits the root version.
- A sub-skill is loaded by `root-id + "/" + relative-sub-skill-path`.
- A sub-skill is not installed or registered separately.

Sub-skill frontmatter should still include useful metadata:

```yaml
---
name: Generate Draft
description: Generate a draft from user material.
tags:
  - writing
dependencies:
  tools: []
  skills: []
skills: []
---
```

Before publishing a root skill with sub-skills, inspect each exposed sub-skill `SKILL.md` for:

- Clear `name`.
- Searchable `description`.
- Correct own `dependencies.tools`.
- Correct own `dependencies.skills`.
- No explicit id unless there is a strong reason and it matches the derived id.

If the user asks to publish a sub-skill id directly, register the owning root skill instead and explain that sub-skills are exposed through `skillhub__load`, not registered as independent packages.

## Validate Resource Boundaries

Skill packages may include supporting files, scripts, assets, and references, but `resource_directory` must not be treated as another instruction source.

Check:

- Full instructions live in `SKILL.md`.
- Supporting files do not duplicate or contradict the main workflow.
- Resource files do not contain extra hidden `SKILL.md` instructions outside the declared root/sub-skill structure.
- Scripts or binaries that the skill expects the agent to run are reflected in `dependencies.tools` when they require external commands.

Do not approve a skill for publishing if it relies on hidden instructions in arbitrary resource files instead of declared root/sub-skill `SKILL.md` files.

## Register

Submit the validated root skill:

```bash
curl -fsS -X POST "${SKILLHUB_DISCOVERY_HOST:-http://218.11.5.155:8399}/v1/register" \
  -H 'Content-Type: application/json' \
  -d '{"id":"<skill-id>","version":"<version>"}'
```

Use the new concrete release tag. Do not register `latest` for a release publish. Do not register an older existing tag after making changes.

Expected response:

```json
{
  "id": "<skill-id>",
  "status": "pending"
}
```

`pending` means submitted for review/indexing. It does not mean approved or discoverable.

## Optional Discoverability Check

If the user asks whether the skill is already discoverable, search by id:

```bash
curl -fsS -X POST "${SKILLHUB_DISCOVERY_HOST:-http://218.11.5.155:8399}/v1/search" \
  -H 'Content-Type: application/json' \
  -d '{"id":"<skill-id>","limit":1}'
```

Interpretation:

- Returned with approved/searchable metadata: the skill is discoverable.
- Not returned: it may still be pending review, rejected, or not indexed yet.
- Error response: report the API error exactly.

## Response To User

Report precisely:

- Validated root skill id.
- Version used.
- Whether registration was accepted.
- Returned status, usually `pending`.
- Any metadata issues found before registration.

Do not say the skill is approved, installed, indexed, or searchable unless the API response or a later search proves it.
