# Skill: Wire Claude Context

## Trigger

Invoke this skill when a user asks to:
- Wire Claude context for this repo
- Set up shared template imports for Claude
- Run `/wire-claude-context`
- Bootstrap `.claude/CLAUDE.md`

Invocation command: `/wire-claude-context`

---

## Process

### Step 1 — Identify the repo

```bash
REPO_NAME=$(basename "$PWD")
echo "Repo: $REPO_NAME"
```

### Step 2 — Detect repo type

- If `$REPO_NAME` starts with `api-cp-` → **API spec repo** → use `api-spec-shared.md`
- If `$REPO_NAME` starts with `service-cp-` → **Service repo** → use `service-shared.md`
- Otherwise → stop and tell the user: *"This skill only supports `api-cp-*` and `service-cp-*` repos. Current directory: `$REPO_NAME`."*

### Step 3 — Create `.claude/` directory if needed

```bash
mkdir -p .claude
```

### Step 4 — Write `.claude/CLAUDE.md`

Write exactly 3 lines:

**For `api-cp-*` repos:**
```
@../../apim-claude-template/templates/shared-code-rules.md
@../../apim-claude-template/templates/api-spec-shared.md
@../../apim-claude-template/templates/claude-md-standards.md
```

**For `service-cp-*` repos:**
```
@../../apim-claude-template/templates/shared-code-rules.md
@../../apim-claude-template/templates/service-shared.md
@../../apim-claude-template/templates/claude-md-standards.md
```

Claude Code automatically reads both `CLAUDE.md` (repo root, committed) and `.claude/CLAUDE.md` (gitignored) on every session. No explicit import of `CLAUDE.md` is needed — it is loaded natively.

### Step 5 — Ensure `.claude/CLAUDE.md` is gitignored

Check if `.gitignore` already contains `.claude/CLAUDE.md`:

```bash
grep -q "\.claude/CLAUDE\.md" .gitignore 2>/dev/null && echo "already ignored" || echo "missing"
```

- If **missing**: append `.claude/CLAUDE.md` to `.gitignore`

```bash
echo ".claude/CLAUDE.md" >> .gitignore
```

- If **already ignored**: do nothing.

Note: `.claude/settings.local.json` and the root `CLAUDE.md` must remain committed — do **not** add `.claude/` (the whole directory) to `.gitignore`, only `.claude/CLAUDE.md`.

### Step 6 — Confirm

Tell the user:

> ✓ `.claude/CLAUDE.md` created for `<REPO_NAME>` (gitignored).
>
> On every Claude Code session, Claude loads:
> - `CLAUDE.md` (this repo) — repo-specific context, committed here
> - `templates/shared-code-rules.md` — team-wide code rules
> - `templates/api-spec-shared.md` (or `service-shared.md`) — repo-category standards
> - `templates/claude-md-standards.md` — HMCTS guidance for generating `CLAUDE.md`
>
> **Next step:** run `/init` to generate (or refresh) the committed `CLAUDE.md` for this repo.
> `/init` will use the HMCTS standards now in context to produce a compliant, non-duplicating file.
>
> When shared templates change in `apim-claude-template`, this repo picks them up automatically — no further action needed.

---

## Rules

- **Never commit `.claude/CLAUDE.md`** — it is a local developer file; only gitignore it, never stage it.
- **Do not touch `.claude/settings.local.json`** — that file is separate and is committed.
- **Do not touch root `CLAUDE.md`** — that file is committed and owned by this repo; use `/init` to generate or refresh it.
- **Paths are relative** from `.claude/CLAUDE.md` — `../../apim-claude-template/` navigates up to the workspace root and into the template repo. This works on any machine where repos are cloned as siblings.
- **Idempotent** — re-running this skill overwrites `.claude/CLAUDE.md` safely with the same content.