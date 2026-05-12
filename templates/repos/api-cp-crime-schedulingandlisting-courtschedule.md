## Repo: api-cp-crime-schedulingandlisting-courtschedule

OpenAPI-first API spec library that defines the Court Schedule API contract for the Scheduling and Listing domain, generating Spring interfaces and DTOs for consumers to retrieve hearing schedules by case URN.

**Pattern**: Pure spec-only
**OpenAPI spec version**: 3.0.x
**OpenAPI Generator version**: 7.19.0 (current — target 7.22.0 per upgrade cycle)
**Spring Boot version**: 4.0.2 (current — target 4.0.6+ per upgrade cycle)

## API Endpoint(s)

```
GET /case/{case_urn}/courtschedule
  → 200 CourtScheduleResponse
  → 400 ErrorResponse
```

## Generated Interfaces & Schema

- Schema file: `src/main/resources/openapi/schema/courtSchedule.schema.json`
- Generated API interface(s): `uk.gov.hmcts.cp.openapi.api.CourtScheduleApi`
- Generated models:
  - `CourtScheduleResponse` — top-level response wrapper
  - `CourtSchedule` — container for a list of hearings
  - `Hearing` — hearing details (ID, type, description, listNote, optional weekCommencing)
  - `CourtSitting` — scheduled sitting with start/end times, judge, courthouse, courtroom UUIDs
  - `HearingWeekCommencing` — week-based scheduling alternative (introduced per AMP-203)
  - `ErrorResponse` — machine-readable error with traceId

## Domain Models

| Model | Purpose |
|---|---|
| `CourtScheduleResponse` | Top-level response wrapper containing the court schedule |
| `CourtSchedule` | Container for a list of `Hearing` objects |
| `Hearing` | Hearing details: ID, type, description, listNote, and optional `weekCommencing` |
| `CourtSitting` | Scheduled court sitting with start/end times, judge UUID, courthouse UUID, courtroom UUID |
| `HearingWeekCommencing` | Week-based scheduling alternative for hearings (AMP-203) |
| `ErrorResponse` | Machine-readable error with traceId for error correlation |

## Test Structure

| Class | What it validates |
|---|---|
| `uk.gov.hmcts.cp.config.OpenApiObjectsTest` | Reflection-based contract test verifying generated model fields and API interface method signatures match the spec; must be updated when new fields are added to the spec |

## CI/CD Deviations

`publish-openapi-spec.yml` triggers on both PR and release (rather than only on release in some sibling repos). Standard 6-workflow set present (`ci-draft.yml`, `ci-released.yml`, `lint-openapi.yml`, `code-analysis.yml`, `codeql.yml`, `publish-openapi-spec.yml`). `secrets-scanner.yml` is absent from this repo's workflow set.

## Repo-Specific Notes

- The `gradle/openapi.gradle` `additionalModelTypeAnnotations` includes `@JsonInclude(NON_NULL)` explicitly injected as a full annotation path (`@com.fasterxml.jackson.annotation.JsonInclude(com.fasterxml.jackson.annotation.JsonInclude.Include.NON_NULL)`), unlike some sibling repos which omit it from the generator config.
- `HearingWeekCommencing` model was introduced specifically for AMP-203 week-based scheduling; it is a scheduling alternative to `CourtSitting` and is optional on a `Hearing`.
- Spring Boot and OpenAPI Generator dependencies are `implementation` scope — transitive deps included in published JAR.
- Run `/openapi-spec-reviewer` skill when authoring or reviewing the OpenAPI spec.
