# Skill: Create PR

## Trigger

Invoke this skill when a user asks to create a pull request, raise a PR, or open a PR.

Invocation command: `/create-pr`

---

## Process

### Step 1 — Gather git context

Run these commands and collect all output before proceeding:

```bash
git branch --show-current
git log origin/main..HEAD --oneline
git diff origin/main..HEAD --stat
git diff origin/main..HEAD
git log origin/main..HEAD --format="%B" | head -100
```

If `origin/main` does not exist, try `origin/master` instead.

### Step 2 — Extract the JIRA ticket

From the branch name, extract the JIRA ticket reference using these rules:

- Branch patterns: `dev/AMP-433`, `dev/AMP433`, `feature/AMP-123-description`, `AMP-456`
- Normalise to uppercase with hyphen: `AMP-433` (not `AMP433`)
- If no ticket is found in the branch name, scan the most recent commit message for a pattern like `AMP-NNN`
- If still not found, prompt the user: *"I couldn't find a JIRA ticket in the branch name. What's the ticket reference (e.g. AMP-433)?"*

Construct the JIRA URL:
```
https://tools.hmcts.net/jira/browse/<TICKET>
```

### Step 3 — Understand the changes

From the git diff and commit log, determine:

**What changed** — concrete list of files/components modified:
- Group by type: e.g. build files, test infrastructure, application code, config, docker
- Be specific: "Removed TestContainers dependencies from build.gradle" not "Updated build file"

**Why it's needed** — the business or technical rationale:
- Look for clues in commit messages, branch name, and the nature of the diff
- If the rationale isn't clear from the code, ask the user: *"Can you give me a one-line summary of why this change is needed, for the PR description?"*

### Step 4 — Draft the PR

**Title format:**
```
<TICKET> <concise description of the change in plain English>
```
Example: `AMP-433 Replace TestContainers with Docker Compose for integration tests`

Keep the title under 72 characters. Do not include "changes" as the only description — be specific.

**Body format:**

```markdown
## JIRA
[<TICKET>](https://tools.hmcts.net/jira/browse/<TICKET>)

## What changed
- <specific change 1>
- <specific change 2>
- <specific change 3>
...

## Why it's needed
<1-3 sentences explaining the motivation — technical debt, bug fix, new requirement, etc.>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

### Step 5 — Check for uncommitted changes

Before creating the PR, run:
```bash
git status
git log origin/main..HEAD --oneline
```

- If there are **uncommitted changes**, warn the user: *"You have uncommitted changes. Do you want me to commit them first, or create the PR with only the pushed commits?"*
- If the **branch hasn't been pushed**, run `git push -u origin <branch>` first
- If there are **no commits ahead of main**, stop and tell the user: *"There are no commits ahead of main on this branch. Nothing to PR."*

### Step 6 — Create the PR and update JIRA

**Create the PR** and capture the returned URL:

```bash
PR_URL=$(gh pr create \
  --title "<title>" \
  --body "$(cat <<'EOF'
<body>
EOF
)" \
  --base main)
echo "$PR_URL"
```

**Post the PR link as a comment on the JIRA ticket** using the JIRA REST API:

```bash
curl -s -X POST \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{\"body\": \"GitHub PR: ${PR_URL}\"}" \
  "https://tools.hmcts.net/jira/rest/api/2/issue/<TICKET>/comment"
```

`JIRA_TOKEN` is a JIRA Personal Access Token. If it is not set in the environment, prompt the user:
*"I need your JIRA Personal Access Token to post the PR link to the ticket. Set it with: `export JIRA_TOKEN=<your-token>`. You can generate one at https://tools.hmcts.net/jira/secure/ViewProfile.jspa under Personal Access Tokens."*

Use a heredoc for the PR body to preserve formatting. Output the PR URL to the user after creation.

### Step 7 — (Optional) Create Confluence release page

Invoke this step when the user indicates the ticket is ready for SIT (e.g. "ready to release", "create release note", "ship to SIT").

**Query JIRA for all tickets in the release:**

```bash
curl -s \
  -H "Authorization: Bearer ${JIRA_TOKEN}" \
  "https://tools.hmcts.net/jira/rest/api/2/search?jql=fixVersion=<VERSION>+AND+project=AMP" \
  | jq '[.issues[] | {key: .key, summary: .fields.summary, status: .fields.status.name, type: .fields.issuetype.name}]'
```

**Validate readiness** — if any ticket status is not `QA Done` or `Done`, warn the user before proceeding:
*"The following tickets are not yet QA signed off: AMP-XXX, AMP-YYY. Do you want to proceed anyway?"*

**Create the Confluence release page** under the AMP releases parent page:

```bash
curl -s -X POST \
  -H "Authorization: Bearer ${CONFLUENCE_TOKEN}" \
  -H "Content-Type: application/json" \
  -d "{
    \"type\": \"page\",
    \"title\": \"Release <VERSION> — SIT Deployment — $(date +'%d %b %Y')\",
    \"ancestors\": [{\"id\": \"<PARENT_PAGE_ID>\"}],
    \"space\": {\"key\": \"AMP\"},
    \"body\": {
      \"storage\": {
        \"value\": \"<confluence-wiki-markup>\",
        \"representation\": \"storage\"
      }
    }
  }" \
  "https://tools.hmcts.net/confluence/rest/api/content"
```

**Release page format:**

```
<h2>Environment Sign-off</h2>
<table>
  <tr><th>Environment</th><th>Status</th><th>Date</th></tr>
  <tr><td>Dev</td><td>✅ Done</td><td><date></td></tr>
  <tr><td>QA</td><td>✅ Done</td><td><date></td></tr>
  <tr><td>SIT</td><td>⏳ Pending</td><td>-</td></tr>
</table>

<h2>Tickets in this release</h2>
<table>
  <tr><th>Ticket</th><th>Summary</th><th>Type</th><th>Status</th></tr>
  <!-- one row per JIRA ticket -->
</table>

<h2>Deployment steps</h2>
<ul>
  <li>Raise deployment request</li>
  <li>Notify SIT team</li>
  <li>Deploy to SIT</li>
  <li>Run smoke tests</li>
</ul>

<h2>Rollback plan</h2>
Redeploy previous version: <PREVIOUS_VERSION>

<h2>Sign-off</h2>
<table>
  <tr><th>Role</th><th>Name</th><th>Date</th></tr>
  <tr><td>Dev Lead</td><td></td><td></td></tr>
  <tr><td>QA Lead</td><td></td><td></td></tr>
  <tr><td>SIT Lead</td><td></td><td></td></tr>
</table>
```

**Post the Confluence page link as a comment on each JIRA ticket:**

```bash
for TICKET in <ticket-list>; do
  curl -s -X POST \
    -H "Authorization: Bearer ${JIRA_TOKEN}" \
    -H "Content-Type: application/json" \
    -d "{\"body\": \"Release note: <CONFLUENCE_PAGE_URL>\"}" \
    "https://tools.hmcts.net/jira/rest/api/2/issue/${TICKET}/comment"
done
```

`CONFLUENCE_TOKEN` is a Confluence Personal Access Token. If not set, prompt the user:
*"I need your Confluence Personal Access Token. Set it with: `export CONFLUENCE_TOKEN=<your-token>`. Generate one at https://tools.hmcts.net/confluence/plugins/servlet/pat — it may be the same as your JIRA_TOKEN."*

`PARENT_PAGE_ID` is the Confluence page ID of the releases folder. If not known, ask the user:
*"What is the Confluence parent page ID for release pages? You can find it in the page URL: `/pages/<ID>/`."*

---

## Rules

- **Never use "changes" as the sole PR title descriptor.** Always describe what the change actually does.
- **Always include the JIRA link** — if the ticket can't be found, ask before creating the PR.
- **Never force-push** or amend existing commits without explicit user instruction.
- **Always confirm** before pushing the branch if it hasn't been pushed yet.
- If `gh` CLI is not authenticated, tell the user to run `gh auth login` first.

---

## Example output

**Branch:** `dev/claude-automated-pr-jira-update`
**Extracted ticket:** `AMP-440`

**Title:**
```
AMP-440 Add README verification section for Claude automated PR generation
```

**Body:**
```markdown
## JIRA
[AMP-440](https://tools.hmcts.net/jira/browse/AMP-440)

## What changed
- Added `## This is to verify Claude automated PR generation with JIRA update` section to `README.md` as a test marker to confirm the Claude Code `/create-pr` skill correctly picks up changes, extracts the JIRA ticket from the branch name, and posts the PR link back to JIRA as a comment

## Why it's needed
Verification that the Claude Code `/create-pr` skill end-to-end flow works correctly — branch name extraction, JIRA link construction, and posting the GitHub PR link as a JIRA comment — before rolling out the skill to the wider team.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
```

**JIRA comment posted:**
```
GitHub PR: https://github.com/hmcts/service-cp-crime-hearing-case-event-subscription/pull/210
```