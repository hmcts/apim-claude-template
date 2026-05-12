## Repo: api-cp-crime-prosecution-case-details

OpenAPI-first API spec library that defines the Crime Prosecution Case Details contract and — unlike pure spec-only siblings — also contains a runnable Spring Boot application implementing the generated interface.

**Pattern**: Hybrid (spec + implementation)
**OpenAPI spec version**: 3.1.0
**OpenAPI Generator version**: 7.19.0 (current — target 7.22.0 per upgrade cycle)
**Spring Boot version**: 4.0.2 (current — target 4.0.6+ per upgrade cycle)

## API Endpoint(s)

```
GET /cases/{case_urn}
  → 200 CaseDetailResponse
  → 400 ErrorResponse
  → 404 (no body)
```

## Generated Interfaces & Schema

- Schema file: `src/main/resources/openapi/schema/prosecutionCase.schema.json`
- Generated API interface(s): `uk.gov.hmcts.cp.openapi.api.CaseDetailsApi`
- Generated models:
  - `CaseDetailResponse` — response body with `caseStatus` (String) and `reportingRestrictions` (Boolean)
  - `ErrorResponse` — machine-readable error with `error`, `message`, `details`, `timestamp` (Instant), `traceId`

## Domain Models

| Model | Purpose |
|---|---|
| `CaseDetailResponse` | Top-level response: `caseStatus` (String) and `reportingRestrictions` (Boolean) |
| `ErrorResponse` | Machine-readable error with `error`, `message`, `details`, `timestamp` (Instant), `traceId` |

## Test Structure

| Class | What it validates |
|---|---|
| `uk.gov.hmcts.cp.config.OpenApiObjectsTest` | Reflection-based contract test verifying generated model fields and API interface method signatures match the spec |
| `CaseDetailsControllerTest` | Mockito unit test for the controller; mocks `CaseDetailsService` |
| `CaseDetailsServiceTest` | Plain unit test; verifies happy path and blank-URN validation |
| `GlobalExceptionHandlerTest` | Tests exception-to-response mapping for `ResponseStatusException` and general `Exception` |

## CI/CD Deviations

Full 7-workflow set present (`ci-draft.yml`, `ci-released.yml`, `lint-openapi.yml`, `code-analysis.yml`, `codeql.yml`, `secrets-scanner.yml`, `publish-openapi-spec.yml`). Standard workflow set — no deviations.

## Repo-Specific Notes

- **Hybrid repo — implementation layer exists**: Unlike all other `api-cp-*` sibling repos this repo contains a runnable Spring Boot application in addition to the generated spec artifacts:
  - `CaseDetailsController` — implements `CaseDetailsApi`; sanitizes `case_urn` input via OWASP Encoder before delegating to the service.
  - `CaseDetailsService` — validates that URN is non-blank; returns a stub `CaseDetailResponse`; throws `ResponseStatusException` for invalid input.
  - `GlobalExceptionHandler` (`@RestControllerAdvice`) — catches `ResponseStatusException` and `Exception`; builds `ErrorResponse` with `traceId` (UUID) and `timestamp` (Instant).
- **OWASP Encoder** used for input sanitization in the controller — not present in spec-only sibling repos.
- Spring Boot and OpenAPI dependencies are declared as `implementation` (not `compileOnly`) — transitive deps are included in the published JAR.
- The `gradle/openapi.gradle` `additionalModelTypeAnnotations` does NOT include `@JsonInclude(NON_NULL)` (unlike some sibling repos); `NON_NULL` filtering must be applied via other means if needed.
- `gradle/java.gradle` adds `build/generated/src/main/java` to the main source set explicitly, which is required because the application code directly instantiates generated types.
- Run `/openapi-spec-reviewer` skill when authoring or reviewing the OpenAPI spec.
