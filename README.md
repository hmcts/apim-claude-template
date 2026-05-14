# apim-claude-template

Shared [Claude Code](https://claude.ai/claude-code) skills and CLAUDE.md templates for the HMCTS APIM team.
Provides a single source of truth for AI context across all `api-cp-*` and `service-cp-*` repos — maintained once here, live-imported everywhere.

---

## How it works

Each `api-cp-*` / `service-cp-*` repo carries two context files that Claude Code loads automatically on every session:

| File | Committed? | Owned by | Contains |
|---|---|---|---|
| `CLAUDE.md` (repo root) | Yes | Each repo | Repo-specific: endpoints, env vars, infra, architecture rules |
| `.claude/CLAUDE.md` | No — gitignored | Developer local | 2 `@import` lines pointing to shared templates here |

The `.claude/CLAUDE.md` for an `api-cp-*` repo looks like:
```
@../../apim-claude-template/templates/shared-code-rules.md
@../../apim-claude-template/templates/api-spec-shared.md
```

When a shared template changes here, all repos pick it up automatically — no per-repo commits needed.
When a repo's architecture changes, update `CLAUDE.md` in that repo — no apim-claude-template PR needed.

---

## Workflow

```mermaid
flowchart TD
    subgraph INSTALL["⚙️ One-time setup (per developer machine)"]
        A[Developer installs Claude Code\nnpm install -g @anthropic-ai/claude-code] --> B[Install agentic-plugins-marketplace\n/marketplace]
        B --> C[Install apim-claude-template plugin\nfrom marketplace]
        C --> D[4 skills now available globally\n/create-pr  /setup-claude-md\n/generate-repo-doc  /openapi-spec-reviewer]
    end

    subgraph ONBOARD["📦 New repo onboarding (run once per repo)"]
        E[New api-cp-* or service-cp-* repo created] --> F[Developer runs\n/generate-repo-doc]
        F --> G[Skill reads real source files\nopenapi-spec.yml · build.gradle\ntest classes · CI workflows]
        G --> H[Writes CLAUDE.md into this repo\ncommitted here · same PR as code]
        H --> I[Developer runs\n/setup-claude-md in the repo]
        I --> K[Creates .claude/CLAUDE.md\ngitignored · 2 @import lines]
    end

    subgraph SESSION["🔄 Every Claude Code session"]
        K --> L[Claude Code starts]
        L --> M1[Reads CLAUDE.md\nrepo root · committed]
        L --> M2[Reads .claude/CLAUDE.md\nresolves @imports]
        M1 --> N1[Repo-specific context\nendpoints · env vars · rules]
        M2 --> N2[shared-code-rules.md]
        M2 --> N3[api-spec-shared.md\nor service-shared.md]
        N1 & N2 & N3 --> O[Claude has full context\nteam rules · patterns · repo specifics]
    end

    subgraph COMMAND["⚡ When a skill command is fired"]
        O --> P{Which command?}

        P -->|/create-pr| Q[Reads git branch · extracts JIRA ticket\nDrafts PR body · gh pr create\nPosts PR link to JIRA]

        P -->|/generate-repo-doc| R[Re-reads source files\nRegenerates templates/repos/repo-name.md\nCommits to apim-claude-template]

        P -->|/setup-claude-md| S[Writes .claude/CLAUDE.md\nUpdates .gitignore]

        P -->|/openapi-spec-reviewer| T[Loads 4 knowledge files\ndata-sharing-policy\ninfrastructure-sla\napi-standards · security-standards]
        T --> U[Reviews spec against\neach lens in sequence]
        U --> V[Scored report\nCritical · Warning · Info\nReadiness score /100]
    end

    subgraph PROPAGATE["🔁 When shared templates change"]
        W[Team edits apim-claude-template\nshared-code-rules.md or\napi-spec-shared.md etc.] --> X[PR raised · reviewed · merged]
        X --> Y[Every developer gets the change\non next Claude Code session\nZero per-repo action needed]
    end
```

| Phase | Who | When |
|---|---|---|
| **Install** | Each developer | Once per machine |
| **Onboard repo** | Repo creator | Once per new repo |
| **Session** | Automatic | Every Claude Code session |
| **Command** | Developer | On demand |
| **Propagate** | Team via PR | When standards change |

---

## Skills

| Command | What it does |
|---|---|
| `/create-pr` | Draft and raise a GitHub PR with JIRA integration — extracts ticket from branch, transitions JIRA status |
| `/setup-claude-md` | Bootstrap `.claude/CLAUDE.md` in the current repo — run once per repo per developer |
| `/generate-repo-doc` | Auto-generate a repo-specific template by reading OpenAPI spec, build files, source layout, and CI workflows |
| `/openapi-spec-reviewer` | Review an OpenAPI v3 spec against four lenses: data-sharing policy, infrastructure SLA, HMCTS API standards, and security standards |

---

## Getting started (existing team member, existing repo)

### 1. One-time machine setup

```bash
npm install -g @anthropic-ai/claude-code
claude          # log in

brew install gh
gh auth login
```

Add to `~/.zshrc`:

```bash
export JIRA_TOKEN=<your-jira-personal-access-token>
```

Generate at: **JIRA → Profile → Personal Access Tokens**

### 2. Clone repos as siblings

All repos must be cloned into the **same parent directory** — the `@import` paths use `../../apim-claude-template/` which requires this layout:

```
~/HMCTS/APIM/
├── apim-claude-template/     ← this repo
├── api-cp-<name>/
├── api-cp-<name>/
├── service-cp-<name>/
└── service-cp-<name>/
```

### 3. Set up Claude context in each repo you work on

```bash
cd api-cp-<repo-name>
/setup-claude-md
```

That's it. Claude Code will load shared rules + category standards + repo-specific guidance on every session.

---

## Adding a new repo

> **Required before the first developer runs `/setup-claude-md`.**
> See [CONTRIBUTING.md](CONTRIBUTING.md) for the full checklist.

```bash
# 1. Clone the new repo as a sibling of apim-claude-template
cd ~/HMCTS/APIM
git clone git@github.com:hmcts/api-cp-<new-name>.git

# 2. Generate CLAUDE.md — writes directly into the new repo
cd api-cp-<new-name>
/generate-repo-doc
# Commits CLAUDE.md to this repo — raise a PR here, not in apim-claude-template

# 3. Set up local Claude context (each developer, once per machine)
/setup-claude-md
```

---

## Structure

```
apim-claude-template/
├── templates/
│   ├── shared-code-rules.md    ← team-wide code rules (all repos)
│   ├── api-spec-shared.md      ← shared guidance for all api-cp-* repos
│   └── service-shared.md       ← shared guidance for all service-cp-* repos
└── skills/
    ├── create-pr/
    ├── setup-claude-md/
    ├── generate-repo-doc/
    └── openapi-spec-reviewer/
        └── knowledge/
            ├── data-sharing-policy.md   ← UK GDPR / DPA 2018 rules
            ├── infrastructure-sla.md    ← Azure APIM / AKS SLA targets
            ├── api-standards.md         ← HMCTS RESTful API standards
            └── security-standards.md   ← OAuth 2, TLS, input validation

Each api-cp-* / service-cp-* repo:
├── CLAUDE.md                   ← committed — repo-specific context
└── .claude/
    ├── CLAUDE.md               ← gitignored — 2 shared @import lines
    └── settings.local.json     ← committed — Claude Code settings
```

---

## Updating shared content

| Change | File to edit | Who acts | Propagation |
|---|---|---|---|
| Team-wide code rule | `templates/shared-code-rules.md` | Any developer | Automatic — all repos |
| API spec pattern | `templates/api-spec-shared.md` | Any developer | Automatic — all `api-cp-*` |
| Service pattern | `templates/service-shared.md` | Any developer | Automatic — all `service-cp-*` |
| One repo's architecture | Update `CLAUDE.md` in that repo | Repo owner | Committed in same PR as code change |
| OpenAPI review policy | `skills/openapi-spec-reviewer/knowledge/` | APIM team | Automatic — all reviewers |

All changes go through a PR against `master`. One merged PR = all repos updated.

---

## POC outcomes

Produced by running `/create-pr` on a real branch against AMP-440:

- **GitHub PR:** https://github.com/hmcts/service-cp-crime-hearing-case-event-subscription/pull/211
- **JIRA comment:** PR link posted automatically to [AMP-440](https://tools.hmcts.net/jira/browse/AMP-440)
- **Confluence release page:** https://tools.hmcts.net/confluence/spaces/AMP/pages/1958295257/Release+AMP-440+-+SIT+Deployment+-+09+Apr+2026

---

## License

MIT
