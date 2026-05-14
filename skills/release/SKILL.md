---
name: release
description: Use when cutting a new GitHub release for an APIM service — finds PRs merged since the last tag, filters out dependencies, chores and docs, computes the next version, and creates the release with a functional changelog.
---

# Skill: Release

## Trigger

Invoke this skill when a user asks to:
- Create a new release
- Cut a release
- Tag and release the service
- Publish a new version

Invocation command: `/release`

---

## Process

### Step 1 — Get the last release tag

```bash
gh release list --limit 1 --json tagName,publishedAt,name
```

Extract:
- `tagName` — current version (e.g. `v1.2.4`)
- `publishedAt` — ISO timestamp used to filter PRs

Tell the user: *"Last release: `<tagName>` published on `<date>`."*

### Step 2 — Compute the next version

Default: **increment the patch number.**

| Current | Next (default) |
|---|---|
| `v1.2.4` | `v1.2.5` |
| `v2.0.0` | `v2.0.1` |

Override only if the user explicitly states:
- `minor` → `v1.2.4` → `v1.3.0`
- `major` → `v1.2.4` → `v2.0.0`

### Step 3 — Collect PRs merged since last release

```bash
gh pr list --state merged --limit 50 \
  --json number,title,mergedAt,author,body \
  --jq '.[] | select(.mergedAt > "<publishedAt>") | {number: .number, title: .title, author: .author.login, body: .body}'
```

### Step 4 — Filter and categorise

**Exclude entirely — do not mention in release notes:**
- Author is `app/dependabot` or `app/renovate`
- Title starts with `chore:`, `chore(deps):`, `docs:`, `ci:`
- Title matches pattern `bump <X> from <Y> to <Z>`

**Categorise what remains:**

| Category | Title signals |
|---|---|
| **Bug Fixes** | `fix:`, `bugfix:`, `AMP-NNN bugfix:` |
| **New Features** | `feat:`, `feature:`, `AMP-NNN Add` |
| **Improvements** | `perf:`, `refactor:`, `chore(AMP-` with JIRA ticket, `Decommission`, `Switch`, `Replace` |
| **Other Changes** | Anything else functional |

### Step 5 — Generate release notes

For each included PR:
1. Read the `## What changed` section from the PR body
2. Read the `## Why it's needed` section from the PR body
3. Write a **plain-English summary** of what the change does and why it matters — one short paragraph per PR

Do not copy raw PR titles. Do not bullet-point the `## What changed` list verbatim. Synthesise into a readable changelog entry.

**Format:**

```markdown
## What's changed in <version>

### Bug Fixes
**#<N> — <Plain English summary: what was broken and what the fix does>**

### New Features
**#<N> — <Plain English summary: what the feature does and why it was added>**

### Improvements
**#<N> — <Plain English summary: what improved and the benefit>**
```

Omit any section that has no entries.

### Step 6 — Confirm with user

Present a summary before creating anything:

```
Next version:  v1.2.5
Included PRs:  N (functional changes)
Excluded PRs:  M (dependabot: X, chore: Y, docs: Z)

Draft release notes:
---
<draft>
---

Shall I create the release?
```

**Wait for the user to confirm.** Do not create the release until they say yes.

### Step 7 — Create the release

```bash
gh release create <version> \
  --title "<version> — <one-line summary of the most significant change>" \
  --notes "$(cat <<'EOF'
<release notes>
EOF
)"
```

After creation, print the release URL.

---

## Rules

- **Always confirm** the version, PR list, and draft notes with the user before creating the release.
- **Never include** dependabot, renovate, chore, or docs PRs in the release notes — not even as a footnote.
- **Default is patch bump.** Only change minor or major if the user explicitly says so.
- **Use the PR body** (`## What changed` / `## Why it's needed`) for summaries — not just the PR title.
- **No code commits.** This skill creates a release only — it does not stage, commit, or push anything.
- If there are **no functional PRs** since the last release, tell the user: *"No functional changes found since `<tag>`. Nothing to release."*
- If `gh` CLI is not authenticated, tell the user: *"Run `gh auth login` first."*

---

## Example output

**Last release:** `v1.2.4` (2026-04-23)
**Next version:** `v1.2.5`
**Included:** 7 PRs | **Excluded:** 11 PRs (9 dependabot, 1 docs, 1 chore)

```markdown
## What's changed in v1.2.5

### Bug Fixes
**#253 — Subscriptions could only register a single event type**
A constraint introduced in API spec 2.0.8 (`@Size(max=1)`) silently rejected any subscription
with more than one event type with a 400. Subscribers registering for both
`PRISON_COURT_REGISTER_GENERATED` and `WEE_Layout5` would fail without a clear error.
Fixed in spec 2.0.9 — multiple event types per subscription are now supported again.

### New Features
**#250 — Subscribers can now rotate their HMAC signing secret on demand**
Previously, the HMAC secret issued at registration could never be changed. A new
`POST /client-subscriptions/{id}/rotate-secret` endpoint generates fresh vault secret
material under the existing key ID without altering subscription configuration.

**#249 — Hearing ID included in notification callback payload**
The `hearingId` field is now forwarded to subscribers in every callback notification,
allowing them to correlate a notification back to its originating hearing without
making a separate lookup.

**#245 — Event type included in notification callback payload**
Subscribers can now see which event type triggered a notification directly from the
callback payload, removing the need for additional API calls to determine context.

### Improvements
**#243 — All processing is now fully asynchronous**
The last remaining synchronous code paths have been removed. All PCR and HearingNows
events now flow exclusively through the Service Bus async pipeline, improving throughput
and reliability under load.

**#255 — Integration tests replaced TestContainers with Docker Compose**
TestContainers was causing port conflicts with host Postgres and added unpredictable
startup timing. Tests now run against a deterministic Docker Compose stack via
`./gradlew dockerTest`, with fast-fail if the stack is not running.

**#259 — App starts locally without Azure Key Vault**
`AZURE_VAULT_ENABLED` now defaults to `false` so developers can run the service
locally using Docker Compose only. Production deployments continue to use Key Vault
via the Dockerfile override.
```

**Release title:** `v1.2.5 — Multiple event types per subscription restored; HMAC secret rotation added`