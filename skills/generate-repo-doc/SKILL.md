# Skill: Generate Repo Doc

## Trigger

Invoke this skill when a user asks to:
- Generate the repo-specific Claude template
- Auto-generate Claude documentation for this repo
- Run `/generate-repo-doc`
- Create or refresh the `templates/repos/<repo-name>.md` file

Invocation command: `/generate-repo-doc`

---

## Process

### Step 1 — Identify the repo

```bash
REPO_NAME=$(basename "$PWD")
REPO_PATH="$PWD"
TEMPLATE_PATH="../../apim-claude-template/templates/repos/${REPO_NAME}.md"
echo "Repo: $REPO_NAME"
echo "Output: $TEMPLATE_PATH"
```

### Step 2 — Detect repo type

- `api-cp-*` → **API spec repo** → follow API discovery steps
- `service-cp-*` → **Service repo** → follow Service discovery steps
- Otherwise → stop: *"This skill only supports `api-cp-*` and `service-cp-*` repos."*

---

## API Spec Repo Discovery (`api-cp-*`)

### Step 3A — Read build metadata

```bash
# Spring Boot version
grep -m1 "springBootVersion\|spring-boot.*version\|id.*org.springframework.boot" build.gradle

# OpenAPI Generator version
grep -m1 "openApiGeneratorVersion\|openapi-generator" build.gradle gradle/openapi.gradle

# Lombok version
grep -m1 "lombokVersion\|lombok" build.gradle
```

### Step 4A — Read OpenAPI spec

```bash
# First 100 lines for title, version, openapi version, paths overview
head -100 src/main/resources/openapi/openapi-spec.yml

# All path definitions
grep -A 20 "^paths:" src/main/resources/openapi/openapi-spec.yml | head -80

# Security schemes
grep -A 10 "securitySchemes:" src/main/resources/openapi/openapi-spec.yml
```

### Step 5A — Read generator config

```bash
cat gradle/openapi.gradle
```

Check for:
- `interfaceOnly` setting
- `additionalModelTypeAnnotations` — flag if `@JsonInclude(NON_NULL)` is missing
- `inputSpec` syntax — flag if using deprecated `=` assignment instead of `.set()`
- Generated package names

### Step 6A — Enumerate test classes

```bash
find src/test/java -name "*.java" | sort
# For each, read first 20 lines to understand what it tests
```

### Step 7A — Check for special features

```bash
# Docs directory
ls docs/ 2>/dev/null

# OpenSpec change workflow
ls openspec/ 2>/dev/null

# Application.java with controller implementations (hybrid repo detection)
find src/main/java -name "Application.java" | head -1
find src/main/java -name "*Controller*.java" -not -path "*/openapi/*" | head -5

# CI workflows
ls .github/workflows/
```

### Step 8A — Generate the template file

Write to `../../apim-claude-template/templates/repos/${REPO_NAME}.md` with this structure:

```markdown
## Repo: <repo-name>

<One sentence: what this repo is and what business capability it serves>

**Pattern**: Pure spec-only | Hybrid (spec + implementation)
**OpenAPI spec version**: x.x.x
**OpenAPI Generator version**: x.x.x (current — target 7.22.0 per upgrade cycle)
**Spring Boot version**: x.x.x (current — target 4.0.6+ per upgrade cycle)

## API Endpoint(s)

[Exact endpoint definitions from openapi-spec.yml: method, path, response codes]

## Generated Interfaces & Schema

- Schema file(s): `src/main/resources/openapi/schema/<name>.schema.json`
- Generated API interface(s): `uk.gov.hmcts.cp.openapi.api.<InterfaceName>`
- Generated models: [list with one-line purpose]

## Domain Models

| Model | Purpose |
|---|---|
[from openapi-spec.yml schemas or build/generated if accessible]

## Test Structure

| Class | What it validates |
|---|---|
[from src/test/java scan]

## Generator Config Notes

[Flag any deviations from standard: missing @JsonInclude, deprecated inputSpec syntax, non-standard packages]

## CI/CD Deviations

[Only list workflows that differ from standard set, or "Standard workflow set — no deviations."]

## Repo-Specific Notes

[HMAC security, openspec workflow, compileOnly scope, hybrid layer, multiple interfaces, key docs, etc. Or "None."]
```

---

## Service Repo Discovery (`service-cp-*`)

### Step 3B — Read build metadata

```bash
grep -m1 "springBootVersion\|org.springframework.boot" build.gradle
```

### Step 4B — Read application config

```bash
cat src/main/resources/application.yaml 2>/dev/null || cat src/main/resources/application.properties 2>/dev/null
```

Extract all `${VAR:default}` patterns — these are the environment variables.

### Step 5B — Read infrastructure config

```bash
cat docker-compose.yml 2>/dev/null
cat docker/docker-compose.yml 2>/dev/null
```

Identify: databases, message brokers, WireMock, other containers.

### Step 6B — Enumerate source classes

```bash
find src/main/java -name "*.java" | sort
# Read each file's first 10 lines to identify class type and purpose
```

Categorise by: controllers, services, clients, filters, config, mappers, entities.

### Step 7B — Read existing CLAUDE.md if present

```bash
cat CLAUDE.md 2>/dev/null
```

Use as a source of verified facts — do not reinvent content that is already documented.

### Step 8B — Check for special features

```bash
# Auth filters
find src/main/java -name "*Filter*.java" | head -10

# Env var docs
cat .envrc.example 2>/dev/null || cat .env 2>/dev/null

# CI workflows
ls .github/workflows/

# Test structure
find src/test/java -name "*.java" | sort
find src/apiTest/java -name "*.java" 2>/dev/null | sort
```

### Step 9B — Generate the template file

Write to `../../apim-claude-template/templates/repos/${REPO_NAME}.md` with this structure:

```markdown
## Repo: <repo-name>

<One sentence: what this service does and which api-cp-* interface it implements>

**Pattern**: Stateless proxy | DB-backed
**Spring Boot version**: x.x.x (current — target 4.0.6+ per upgrade cycle)
**Implements**: `api-cp-<name>` generated interface(s)

## Infrastructure

| Component | Technology | Purpose |
|---|---|---|

## Source Structure

[Key classes by package — one line each: `ClassName` — what it does]

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|

## Repo-Specific Architecture Rules

[Rules unique to this service: client ID requirement, event pipeline, service bus toggle, auth filter, validation patterns]

## Debugging

| Symptom | Cause / Fix |
|---|---|

## Repo-Specific Notes

[Test deviations, special CI workflows, docs, etc. Or "None."]
```

---

## Step N — Commit to apim-claude-template

After generating the file:

```bash
cd ../../apim-claude-template
git add templates/repos/${REPO_NAME}.md
git commit -m "feat: add repo-specific CLAUDE.md template for ${REPO_NAME}"
cd -
```

Tell the user:
> ✓ Generated `apim-claude-template/templates/repos/${REPO_NAME}.md` and committed.
>
> Review the file, then run `/setup-claude-md` in this repo (and in any other repo that needs the same template).

---

## Rules

- **Read actual source files** — never invent class names, endpoint paths, or env var names.
- **Idempotent** — re-running overwrites the template file and creates a new commit; this is safe.
- **Do not include shared content** — commands, layering rules, TracingFilter standard, Docker standard, CI standard set, and publishing details are all in the shared templates. Only capture what is unique to this repo.
- **Flag technical debt** — if you discover missing `@JsonInclude`, deprecated inputSpec syntax, or outdated Spring Boot / OpenAPI Generator versions, add a note under "Generator Config Notes" or "Repo-Specific Notes" — but do not change the source code.
- **Commit to apim-claude-template** — the generated file is owned by the template repo, not the source repo.
