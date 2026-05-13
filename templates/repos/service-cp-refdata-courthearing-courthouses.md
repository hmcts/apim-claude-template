## Repo: service-cp-refdata-courthearing-courthouses

Stateless proxy service that retrieves courthouse and courtroom reference data from the CP Reference Data backend by court ID; it implements the `api-cp-refdata-courthearing-courthouses` generated `CourtHouseApi` and `CourtRoomApi` interfaces.

**Pattern**: Stateless proxy
**Spring Boot version**: 4.0.1
**Implements**: `api-cp-refdata-courthearing-courthouses:1.0.8` — `CourtHouseApi`, `CourtRoomApi`

## Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| HTTP backend | `CP_BACKEND_URL` | CP Reference Data service (`/referencedata-service/query/api/rest/referencedata/courtrooms`) |
| Test doubles | None (no WireMock in docker-compose) | docker-compose.yml uses `.env` only; API tests require external stub or real backend |

## Source Structure

**controllers/**
- `CourtHousesController` — implements `CourtHouseApi`; receives `courtId` (UUID), delegates to `CourtHousesService.getCourtHouseByCourtId()`; returns `CourtHouseResponse`
- `CourtRoomsController` — implements `CourtRoomApi`; receives `courtId` and `courtRoomId` (both UUID), delegates to `CourtHousesService.getCourthouseByCourtIdAndCourtRoomId()`; returns `CourtHouseResponse`
- `GlobalExceptionHandler` — `@RestControllerAdvice`; maps `HttpClientErrorException(NOT_FOUND)` and `HttpServerErrorException` to HTTP responses
- `RootController` — health/root endpoint

**services/**
- `CourtHousesService` — calls `CourtHousesClient` with courtId UUID; returns mapped `CourtHouseResponse`

**clients/**
- `CourtHousesClient` — `RestTemplate` client to CP Reference Data backend; sets `Accept: application/vnd.referencedata.ou-courtroom+json` and `CJSCPPUID` header; GETs `{CP_BACKEND_URL}{path}/{courtId}`; returns HTTP 404 (`HttpClientErrorException`) if backend returns `{}` (empty JSON object); throws `HttpServerErrorException` for non-2xx responses

**mappers/**
- `CourtHouseMapper` — MapStruct mapper; transforms raw JSON `String` response body → `CourtResponse` → `CourtHouseResponse`

**domain/**
- `CourtResponse` — internal DTO for backend response

**config/**
- `AppConfig` — Spring `@Configuration`; `RestTemplate` bean
- `AppPropertiesBackend` — binds `service.court-house-client.url`, `service.court-house-client.path`, `service.court-house-client.cjscppuid`

**filters/tracing/**
- `TracingFilter` — propagates correlation IDs via MDC

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `CP_BACKEND_URL` | Base URL of CP Reference Data backend | `http://localhost` |
| `CJSCPPUID` | User UUID sent as `CJSCPPUID` header to backend | `00000000-0000-0000-0000-000000000000` |
| `rpe.AppInsightsInstrumentationKey` | Azure Application Insights key (loaded from `/mnt/secrets/rpe/`) | `00000000-0000-0000-0000-000000000000` |

## Repo-Specific Architecture Rules

**Vendor media type on Accept is mandatory**
`CourtHousesClient.getRequestEntity()` sets `Accept: application/vnd.referencedata.ou-courtroom+json`. The CP backend returns 406 if this header is missing or set to `application/json`. Do not change this to a generic media type.

**Empty JSON object `{}` is treated as 404**
The CP backend returns HTTP 200 with body `{}` when no courthouse is found for the given `courtId`. `CourtHousesClient` explicitly checks for this and throws `HttpClientErrorException(HttpStatus.NOT_FOUND)`:
```java
if (EMPTY_JSON_OBJECT.equals(response.getBody())) {
    throw new HttpClientErrorException(HttpStatus.NOT_FOUND);
}
```
`GlobalExceptionHandler` maps this to an HTTP 404 response. Do not remove this check.

**CJSCPPUID header is mandatory on every backend call**
`CourtHousesClient.getRequestEntity()` always adds the `CJSCPPUID` header. The CP backend returns 401/403 without it.

**Backend URL path nesting**
`application.yaml` nests the client config under `service.court-house-client.*` (not the flat `court-house-client.*` pattern used in other services). `AppPropertiesBackend` binds `@Value("${service.court-house-client.url}")` etc. Match this nesting exactly when adding new config.

**Backend path is fixed**
Configured in `application.yaml`:
`/referencedata-service/query/api/rest/referencedata/courtrooms`
The `courtId` UUID is appended as a path segment by `CourtHousesClient.buildUrl()`.

**Both controllers share `CourtHousesService`**
`CourtHousesController` (`GET /courthouses/{courtId}`) and `CourtRoomsController` (`GET /courthouses/{courtId}/courtrooms/{courtRoomId}`) both delegate to `CourtHousesService`. The service passes courtId (and courtRoomId where relevant) to the same `CourtHousesClient`.

**`spring.application.name` mismatch**
`application.yaml` sets `spring.application.name: service-cp-refdata-courthearing-judges` — note this does not match the repo name. This is a pre-existing configuration artefact. Do not change it without verifying downstream observability dashboards.

## Debugging

| Symptom | Cause / Fix |
|---|---|
| Won't start | Check port 4550 is free; `CP_BACKEND_URL` does not need to be reachable at startup |
| HTTP 404 for a known court ID | Backend returned `{}` — verify the courtId exists in the reference data backend; check `CP_BACKEND_URL` is correct |
| HTTP 406 from backend | `Accept` header missing or wrong — verify `CourtHousesClient.getRequestEntity()` sets `application/vnd.referencedata.ou-courtroom+json` |
| HTTP 401/403 from backend | `CJSCPPUID` not set — set to a valid UUID |
| `HttpServerErrorException` propagated as 500 | Backend returned a non-2xx, non-404 response — check backend health and logs |

## Repo-Specific Notes

The `docker-compose.yml` in this repo is minimal — it only runs the app container using an `.env` file and does not include a WireMock service. API tests require either a real backend or a separately started stub. This is a deviation from the other services in this workspace which include WireMock in their docker-compose stacks.

`management.endpoints.web.base-path: /` is set, so health is at `/health` and prometheus at `/prometheus` — not the standard `/actuator/*` paths.

The `spring.config.import: "optional:configtree:/mnt/secrets/rpe/"` pattern reads Azure Key Vault–mounted secrets from the pod filesystem in AKS; locally this path is absent and the import is silently skipped.

The `gradle/` directory layout uses a sub-divided structure (`gradle/dependencies/`, `gradle/tasks/`, `gradle/github/`, `gradle/github/docker.gradle`) rather than the flat structure found in the other services. This affects where shared Gradle config lives — check these subdirectories when troubleshooting build issues specific to this repo.
