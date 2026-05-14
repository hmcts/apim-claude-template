# Skill: Setup Claude MD

## Trigger

Invoke this skill when a user asks to:
- Set up Claude context for this repo
- Bootstrap `.claude/CLAUDE.md`
- Run `/setup-claude-md`
- Initialise Claude guidance for this repository

Invocation command: `/setup-claude-md`

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

### Step 3 — Check the repo-specific CLAUDE.md exists

```bash
ls CLAUDE.md 2>/dev/null
```

- If the file **does not exist**: stop and tell the user:
  *"No `CLAUDE.md` found in this repo. Run `/generate-repo-doc` first to auto-generate it, then re-run `/setup-claude-md`."*
- If the file **exists**: proceed.

### Step 4 — Create `.claude/` directory if needed

```bash
mkdir -p .claude
```

### Step 5 — Write `.claude/CLAUDE.md`

Write exactly 2 lines:

**For `api-cp-*` repos:**
```
@../../apim-claude-template/templates/shared-code-rules.md
@../../apim-claude-template/templates/api-spec-shared.md
```

**For `service-cp-*` repos:**
```
@../../apim-claude-template/templates/shared-code-rules.md
@../../apim-claude-template/templates/service-shared.md
```

Claude Code automatically reads both `CLAUDE.md` (repo root, committed) and `.claude/CLAUDE.md` (gitignored) on every session. No explicit import of `CLAUDE.md` is needed — it is loaded natively.

### Step 6 — Ensure `.claude/CLAUDE.md` is gitignored

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

### Step 7 — Confirm

Tell the user:

> ✓ `.claude/CLAUDE.md` created for `<REPO_NAME>`.
>
> On every Claude Code session, Claude loads:
> - `CLAUDE.md` (this repo) — repo-specific context, committed here
> - `templates/shared-code-rules.md` — team-wide code rules
> - `templates/api-spec-shared.md` (or `service-shared.md`) — repo-category standards
>
> When shared templates change in `apim-claude-template`, this repo picks them up automatically — no further action needed.
>
> When this repo's architecture changes, update `CLAUDE.md` and commit it in the same PR as the code change.

---

## Rules

- **Never commit `.claude/CLAUDE.md`** — it is a local developer file; only gitignore it, never stage it.
- **Do not touch `.claude/settings.local.json`** — that file is separate and is committed.
- **Do not touch root `CLAUDE.md`** — that file is committed and owned by this repo; `/generate-repo-doc` manages it.
- **Paths are relative** from `.claude/CLAUDE.md` — `../../apim-claude-template/` navigates up to the workspace root and into the template repo. This works on any machine where repos are cloned as siblings.
- **Idempotent** — re-running this skill overwrites `.claude/CLAUDE.md` safely with the same content.
