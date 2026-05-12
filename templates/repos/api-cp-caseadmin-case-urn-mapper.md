## Repo: api-cp-caseadmin-case-urn-mapper

OpenAPI-first API spec library that maps a Case URN to a Case ID for the Case Administration domain.

**Pattern**: Pure spec-only
**OpenAPI spec version**: 3.0.0
**OpenAPI Generator version**: 7.22.0 (current — target 7.22.0 per upgrade cycle)
**Spring Boot version**: 4.0.6 (current — target 4.0.6+ per upgrade cycle)

## API Endpoint(s)

```
GET /urnmapper/{case_urn}?refresh={boolean}
  → 200 CaseMapperResponse
  → 400 ErrorResponse
```

## Generated Interfaces & Schema

- Schema file: `src/main/resources/openapi/schema/` (schemas inlined in spec; no separate schema file)
- Generated API interface(s): `uk.gov.hmcts.cp.openapi.api.CaseIdByCaseUrnApi`
- Generated models:
  - `CaseMapperResponse` — response body with `caseId` and `caseUrn` (caseUrn required)
  - `ErrorResponse` — machine-readable error with `error`, `message`, `details`, `timestamp` (Instant), `traceId`

## Domain Models

| Model | Purpose |
|---|---|
| `CaseMapperResponse` | Maps a case URN to a case ID; `caseUrn` is required, `caseId` is optional |
| `ErrorResponse` | Machine-readable error with traceId for error correlation |

## Test Structure

| Class | What it validates |
|---|---|
| `uk.gov.hmcts.cp.config.OpenApiObjectsTest` | Reflection-based contract test verifying generated model fields and API interface method signatures match the spec |

## CI/CD Deviations

`ci-draft.yml` publishes a draft OpenAPI spec to SwaggerHub in addition to the standard draft artefact publish. `lint-openapi.yml` additionally checks that internal HMCTS domain URLs (`cjscp.org.uk`, `service.gov.uk`, `justice.gov.uk`, `hmcts.net`, `ejudiciary.net`) are not referenced in the spec. `secrets-scanner.yml` is absent from this repo's workflow set.

## Repo-Specific Notes

- The OpenAPI spec defines schemas inline (in `components/schemas`) rather than in separate `*.schema.json` files under `src/main/resources/openapi/schema/` — there is no external schema file to validate via AJV.
- The `refresh` query parameter (boolean, optional) is included on the `GET /urnmapper/{case_urn}` endpoint for cache-busting; it is not present in most other repos.
- Spring Boot dependencies are declared as `implementation` (not `compileOnly`) — all transitive Spring Boot deps are included in the published JAR. This differs from `api-cp-crime-hearing-results-document-subscription` which uses `compileOnly` for Spring Boot to avoid transitive conflicts.
- The `jar.gradle` script includes `CHANGELOG.md` and a CycloneDX SBOM (`bom.json`) inside the published JAR.
- Run `/openapi-spec-reviewer` skill when authoring or reviewing the OpenAPI spec.
