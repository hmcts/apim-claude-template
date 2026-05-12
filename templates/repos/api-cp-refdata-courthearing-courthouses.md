## Repo: api-cp-refdata-courthearing-courthouses

OpenAPI-first API spec library that defines the Court Hearing Courthouses reference data API contract, generating two Spring interfaces and DTOs for consumers to retrieve courthouse and courtroom details.

**Pattern**: Pure spec-only
**OpenAPI spec version**: 3.0.x
**OpenAPI Generator version**: 7.18.0 (current — target 7.22.0 per upgrade cycle)
**Spring Boot version**: 4.0.1 (current — target 4.0.6+ per upgrade cycle)

## API Endpoint(s)

```
GET /courthouses/{court_id}
  → 200 CourtHouseResponse
  → 4xx ErrorResponse

GET /courthouses/{court_id}/courtrooms/{court_room_id}
  → 200 CourtRoom
  → 4xx ErrorResponse
```

## Generated Interfaces & Schema

- Schema file: `src/main/resources/openapi/schema/courtHouses.schema.json`
- Generated API interface(s):
  - `uk.gov.hmcts.cp.openapi.api.CourtHouseApi` — courthouse lookup endpoint
  - `uk.gov.hmcts.cp.openapi.api.CourtRoomApi` — courtroom lookup endpoint
- Generated models:
  - `CourtHouseResponse` — top-level response with court type, code, name, address, and list of courtrooms
  - `CourtRoom` — individual courtroom with ID and name
  - `Address` — full UK address including postcode regex validation
  - `ErrorResponse` — machine-readable error with traceId and timestamp

## Domain Models

| Model | Purpose |
|---|---|
| `CourtHouseResponse` | Top-level courthouse response: court type, code, name, address, and embedded courtrooms list |
| `CourtRoom` | Individual courtroom details: ID and name |
| `Address` | Full UK address; includes postcode regex validation constraint |
| `ErrorResponse` | Machine-readable error with traceId and timestamp for error correlation |

## Test Structure

| Class | What it validates |
|---|---|
| `uk.gov.hmcts.cp.config.OpenApiObjectsTest` | Reflection-based contract test verifying generated model fields and API interface method signatures match the spec; must be updated when new fields are added to the spec |

## CI/CD Deviations

Full 7-workflow set present (`ci-draft.yml`, `ci-released.yml`, `lint-openapi.yml`, `code-analysis.yml`, `codeql.yml`, `secrets-scanner.yml`, `publish-openapi-spec.yml`). Standard workflow set — no deviations.

## Repo-Specific Notes

- **Two generated interfaces**: This is the only repo in the sibling set that generates two distinct Spring `@RequestMapping` interfaces from a single spec — `CourtHouseApi` and `CourtRoomApi` (one per OpenAPI tag). All other repos generate a single interface.
- **Address postcode validation**: The `Address` model includes a postcode regex validation annotation — this is not present in domain models of sibling repos.
- The `gradle/openapi.gradle` `additionalModelTypeAnnotations` includes `@JsonInclude(NON_NULL)` explicitly injected as a full annotation path (`@com.fasterxml.jackson.annotation.JsonInclude(com.fasterxml.jackson.annotation.JsonInclude.Include.NON_NULL)`).
- Spring Boot and OpenAPI Generator dependencies are `implementation` scope — transitive deps included in published JAR.
- This is the oldest repo in the set by OpenAPI Generator version (7.18.0); upgrade to 7.22.0 is pending.
- Run `/openapi-spec-reviewer` skill when authoring or reviewing the OpenAPI spec.
