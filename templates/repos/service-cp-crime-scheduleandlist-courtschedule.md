## Repo: service-cp-crime-scheduleandlist-courtschedule

Stateless proxy service that retrieves allocated and unallocated court hearing schedules from the CP Listing backend by resolving a Case URN to a Case ID via a sidecar URN-mapper service.

**Pattern**: Stateless proxy
**Spring Boot version**: 4.0.1
**Implements**: `api-cp-crime-schedulingandlisting-courtschedule`

## Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| HTTP backend (listing) | `CP_BACKEND_URL` | CP Listing query API (`/listing-query-api/query/api/rest/listing/hearings/allocated-and-unallocated`) |
| HTTP backend (URN mapper) | `AMP_BACKEND_URL` | Internal URN-mapper service (`/urnmapper`) — translates Case URN to Case ID |
| Test doubles | WireMock 3.6.0 | API test stubs for both backends |

## Source Structure

**controllers/**
- `CourtScheduleController` — receives `caseUrn`, sanitises with OWASP `Encode.forJava()`, resolves to caseId String via `CaseUrnMapperService`, delegates to `CourtScheduleService`
- `GlobalExceptionHandler` — `@RestControllerAdvice`; maps exceptions to HTTP responses
- `RootController` — health/root endpoint

**services/**
- `CourtScheduleService` — calls `CourtScheduleClient` with the resolved caseId; maps the Listing backend response to the API response shape
- `CaseUrnMapperService` — `RestTemplate`-based service that calls the AMP URN-mapper; returns a `String` caseId (not a UUID — note difference from `service-cp-crime-prosecution-case-details`)

**clients/**
- `CourtScheduleClient` — `RestTemplate` client to CP Listing backend; sets `CJSCPPUID` header; GETs allocated-and-unallocated hearings

**mappers/**
- `HearingsMapper` — MapStruct mapper; transforms Listing backend response → API response

**domain/**
- `CaseMapperResponse` — internal DTO for URN-mapper response (`caseId` as String)
- `HearingResponse` — internal DTO for Listing backend response; nested `HearingSchedule` with `allocated` flag and `weekCommencingDurationInWeeks`

**filters/**
- `HearingResponseFilter` — post-processing filter; removes hearings where `allocated == false` AND `weekCommencingDurationInWeeks == null`; only hearings that are either allocated or have a week-commencing duration are returned to the caller
- `TracingFilter` — propagates correlation IDs via MDC (in `filters/tracing/`)

**config/**
- `AppConfig` — Spring `@Configuration`; `RestTemplate` bean
- `AppPropertiesBackend` — binds `case-mapper-client.url`, `case-mapper-client.path`, `court-schedule-client.url`, `court-schedule-client.path`, `court-schedule-client.cjscppuid`

**exceptions/**
- `GlobalExceptionHandler` — `@RestControllerAdvice`

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `CP_BACKEND_URL` | Base URL of CP Listing query backend | `http://localhost:8081` |
| `AMP_BACKEND_URL` | Base URL of the internal URN-mapper service | `http://localhost:8081` |
| `CJSCPPUID` | User UUID sent as `CJSCPPUID` header to Listing backend | `00000000-0000-0000-0000-000000000000` |
| `rpe.AppInsightsInstrumentationKey` | Azure Application Insights key (loaded from `/mnt/secrets/rpe/`) | `00000000-0000-0000-0000-000000000000` |

## Repo-Specific Architecture Rules

**Two-hop resolution pattern**
Every court-schedule request requires two sequential HTTP calls:
1. `CaseUrnMapperService` → AMP URN-mapper (`AMP_BACKEND_URL/urnmapper/{caseUrn}`) to get a String caseId
2. `CourtScheduleClient` → CP Listing backend (`CP_BACKEND_URL/listing-query-api/.../hearings/allocated-and-unallocated/{caseId}`) to get hearings

Both backends must be reachable. A non-2xx response from the URN-mapper propagates as an error without calling the Listing backend.

**`HearingResponseFilter` — client-side result filtering**
After the backend response is received, `CourtScheduleService` must apply `HearingResponseFilter.filterHearingResponse()` before returning data. The filter retains only hearings satisfying:
```java
hearingSchedule.isAllocated() || Objects.nonNull(hearingSchedule.getWeekCommencingDurationInWeeks())
```
Do not remove or bypass this filter — it enforces the data-sharing contract.

**caseId type is String (not UUID)**
Unlike `service-cp-crime-prosecution-case-details` where `CaseUrnMapperService` returns a `UUID`, in this service `CaseUrnMapperService.getCaseId()` returns a `String`. Do not change this to UUID without verifying the Listing backend's path parameter expectations.

**CJSCPPUID header is mandatory on Listing backend calls**
`CourtScheduleClient` always sends `CJSCPPUID` as a header. The CP backend returns 401/403 without it.

**Listing backend path is fixed**
Configured in `application.yaml` under `court-schedule-client.path`:
`/listing-query-api/query/api/rest/listing/hearings/allocated-and-unallocated`
The caseId is appended by `CourtScheduleClient` at call time.

**Springdoc enabled**
`application.yaml` sets `springdoc.packagesToScan: uk.gov.hmcts.cp.controllers` — Swagger UI is available at `/swagger-ui.html` locally. This is absent from most other services in this workspace.

## Debugging

| Symptom | Cause / Fix |
|---|---|
| Won't start | Check port 4550 is free; `CP_BACKEND_URL` and `AMP_BACKEND_URL` do not need to be reachable at startup |
| Empty hearings list in response | All hearings filtered by `HearingResponseFilter` — backend returned hearings where `allocated=false` and `weekCommencingDurationInWeeks=null`; verify backend data |
| HTTP 401/403 from Listing backend | `CJSCPPUID` not set or expired — set a valid UUID |
| HTTP 404 on case URN | URN-mapper returned no match — check `AMP_BACKEND_URL` is correct and the URN exists |
| WireMock not matching requests in API tests | Check `src/apiTest/resources/mappings/` stubs; both `/urnmapper/*` and listing backend paths must be stubbed |

## Repo-Specific Notes

`application.yaml` sets `management.endpoints.web.base-path: /` (root), so health is at `/health` and prometheus at `/prometheus` — not the standard `/actuator/*` paths. This is the same pattern as `service-cp-caseadmin-case-urn-mapper`.

The `spring.config.import: "optional:configtree:/mnt/secrets/rpe/"` pattern reads Azure Key Vault–mounted secrets from the pod filesystem in AKS; locally this path is absent and the import is silently skipped.

In the docker-compose API test stack, both `AMP_BACKEND_URL` and `CP_BACKEND_URL` are set to `http://wiremock:8080` so a single WireMock instance serves both client paths.
