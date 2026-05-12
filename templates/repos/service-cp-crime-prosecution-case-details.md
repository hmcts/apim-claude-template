## Repo: service-cp-crime-prosecution-case-details

Stateless proxy service that retrieves prosecution case details from the CP Progression backend by resolving a Case URN to a Case ID via a sidecar URN-mapper service; it implements the `api-cp-crime-prosecution-case-details` generated `CaseDetailsApi` interface.

**Pattern**: Stateless proxy
**Spring Boot version**: 4.0.1
**Implements**: `api-cp-crime-prosecution-case-details:1.0.0` — `CaseDetailsApi`

## Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| HTTP backend (progression) | `CP_BACKEND_URL` | CP Progression query API (`/progression-query-api/query/api/rest/progression/prosecutioncases`) |
| HTTP backend (URN mapper) | `AMP_BACKEND_URL` | Internal URN-mapper service (`/urnmapper`) — translates Case URN to Case ID |
| Test doubles | WireMock 3.6.0 | API test stubs for both backends |

## Source Structure

**controllers/**
- `CaseDetailController` — implements `CaseDetailsApi`; receives `caseUrn`, sanitises with OWASP `Encode.forJava()`, resolves to UUID via `CaseUrnMapperService`, delegates to `CaseDetailService`
- `GlobalExceptionHandler` — `@RestControllerAdvice`; maps exceptions to HTTP responses with `traceId`
- `RootController` — health/root endpoint

**services/**
- `CaseDetailService` — calls `ProgressionClient` with the resolved `caseId` UUID; maps the `ProgressionResponse` to a `CaseDetailResponse`
- `CaseUrnMapperService` — thin wrapper around `CaseUrnMapperClient`; returns a `UUID` from the URN-mapper service

**clients/**
- `ProgressionClient` — `RestTemplate` client to CP Progression backend; sets `CJSCPPUID` header; GETs `/progression-query-api/query/api/rest/progression/prosecutioncases/{caseId}`
- `CaseUrnMapperClient` — `RestTemplate` client to AMP URN-mapper service (`/urnmapper/{caseUrn}`); returns `CaseMapperResponse` containing the `caseId` UUID

**mappers/**
- `CaseDetailMapper` — MapStruct mapper; transforms `ProgressionResponse` → `CaseDetailResponse`

**domain/**
- `CaseMapperResponse` — internal DTO for URN-mapper response (`caseUrn`, `caseId`)
- `ProgressionResponse` — internal DTO for Progression backend response

**config/**
- `AppConfig` — Spring `@Configuration`; `RestTemplate` bean
- `AppPropertiesBackend` — binds `case-mapper-client.url`, `case-mapper-client.path`, `progression-client.url`, `progression-client.path`, `progression-client.cjscppuid`

**exceptions/**
- `GlobalExceptionHandler` — `@RestControllerAdvice` (duplicate package path from `controllers/`; uses `exceptions/` package)

**filters/tracing/**
- `TracingFilter` — propagates correlation IDs via MDC

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `CP_BACKEND_URL` | Base URL of CP Progression query backend | `http://localhost:8081` |
| `AMP_BACKEND_URL` | Base URL of the internal URN-mapper service | `http://localhost:8081` |
| `CJSCPPUID` | User UUID sent as `CJSCPPUID` header to Progression backend | `00000000-0000-0000-0000-000000000000` |
| `rpe.AppInsightsInstrumentationKey` | Azure Application Insights key | `00000000-0000-0000-0000-000000000000` |

## Repo-Specific Architecture Rules

**Two-hop resolution pattern**
Every case-detail request requires two sequential HTTP calls:
1. `CaseUrnMapperClient` → AMP URN-mapper (`AMP_BACKEND_URL/urnmapper/{caseUrn}`) to get a `UUID` caseId
2. `ProgressionClient` → CP Progression backend (`CP_BACKEND_URL/progression-query-api/.../prosecutioncases/{caseId}`) to get case details

Both backends must be reachable. If the URN-mapper returns a non-2xx response, the controller propagates the error without calling Progression.

**CJSCPPUID header is mandatory on Progression calls**
`ProgressionClient` always sends `CJSCPPUID` as a header. The CP backend returns 401/403 without it. The URN-mapper client does not send this header.

**Input sanitisation before logging**
`CaseDetailController` calls `Encode.forJava(caseUrn)` before logging. This must be preserved on any log statements added to the controller.

**Progression path is fixed**
The backend path `/progression-query-api/query/api/rest/progression/prosecutioncases` is configured in `application.yaml` under `progression-client.path`. The `caseId` UUID is appended as a path segment by `ProgressionClient`.

## Debugging

| Symptom | Cause / Fix |
|---|---|
| Won't start | Check port 4550 is free; `CP_BACKEND_URL` and `AMP_BACKEND_URL` do not need to be reachable at startup |
| HTTP 404 on case URN | URN-mapper returned no match — check `AMP_BACKEND_URL` is correct and the URN exists in the mapper service |
| HTTP 401/403 from Progression | `CJSCPPUID` not set or expired — set a valid UUID value |
| `NullPointerException` in `CaseDetailController` | URN-mapper returned a body with null `caseId` — check URN-mapper service response shape matches `CaseMapperResponse` |
| WireMock not matching requests in API tests | Check `src/apiTest/resources/mappings/` stubs match the exact URL patterns used by both clients |

## Repo-Specific Notes

`AMP_BACKEND_URL` and `CP_BACKEND_URL` may point to different hosts. In the docker-compose API test stack both are set to `http://wiremock:8080` so a single WireMock instance serves both client paths.

The `application.yaml` uses `management.endpoints.web.base-path: /actuator` (standard paths), unlike the URN-mapper service which exposes actuator at `/`.
