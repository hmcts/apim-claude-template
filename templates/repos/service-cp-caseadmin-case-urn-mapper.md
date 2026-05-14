## Repo: service-cp-caseadmin-case-urn-mapper

Stateless proxy service that resolves a Case URN to a Case File ID by calling the CP system-id-mapper backend, with in-memory caching of results.

**Pattern**: Stateless proxy (with in-memory cache)
**Spring Boot version**: 4.0.3
**Implements**: `api-cp-caseadmin-case-urn-mapper`

## Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| HTTP backend | `CP_BACKEND_URL` | CP system-id-mapper API (`/system-id-mapper-api/rest/systemid/mappings`) |
| In-memory cache | Spring Cache (`ConcurrentMapCacheManager`) | Cache `caseUrn → caseId` lookups; cache name `caseIdByCaseUrn` |
| Test doubles | WireMock 3.6.0 | API test stubs for backend |

## Source Structure

**controllers/**
- `CaseUrnMapperController` — validates caseUrn against regex `^[0-9a-zA-Z]{10,40}$`; accepts optional `?refresh=true` query param to bypass/refresh cache
- `GlobalExceptionHandler` — `@RestControllerAdvice`; maps exceptions to HTTP responses with `traceId`
- `RootController` — health/root endpoint

**services/**
- `CaseUrnMapperService` — delegates to `CaseUrnMapperRepository`; entry point from controller

**client/**
- `CaseUrnMapperClient` — `RestTemplate` client to CP backend; sets `Accept: application/vnd.systemid.mapping+json` and `CJSCPPUID` header; queries `?sourceId=<urn>&targetType=CASE_FILE_ID`
- `UrnMapperResponse` — internal DTO for backend response (`sourceId`, `targetId`)

**repositories/**
- `CaseUrnMapperRepository` — interface with `getCaseIdByCaseUrn(caseUrn, refresh)`
- `InMemoryCaseUrnMapperRepositoryImpl` — delegates to `CaseUrnCacheService`; routes to cache-hit or force-refresh path
- `CaseUrnCacheService` — `@Cacheable(caseIdByCaseUrn)` on `getCachedCaseId()`; `getCaseIdAndRefreshCache()` calls backend fresh then writes back to `CacheManager`

**config/**
- `AppConfig` — Spring `@Configuration`; `RestTemplate` bean
- `AppPropertiesBackend` — binds `case-urn-mapper.url`, `case-urn-mapper.path`, `case-urn-mapper.cjscppuid`
- `CachingConfig` — `@EnableCaching`; defines cache name constant `CASE_ID_BY_CASE_URN = "caseIdByCaseUrn"`

**filters/tracing/**
- `TracingFilter` — propagates correlation IDs via MDC

**utils/**
- `EncodeDecodeUtils` — OWASP encoding helpers

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `CP_BACKEND_URL` | Base URL of CP system-id-mapper backend | `http://localhost` |
| `CJSCPPUID` | User UUID sent as `CJSCPPUID` header to backend | `00000000-0000-0000-0000-000000000000` |
| `rpe.AppInsightsInstrumentationKey` | Azure Application Insights key (loaded from `/mnt/secrets/rpe/`) | `00000000-0000-0000-0000-000000000000` |

## Repo-Specific Architecture Rules

**Input validation — caseUrn format enforcement**
`CaseUrnMapperController` rejects any caseUrn that does not match `^[0-9a-zA-Z]{10,40}$` with HTTP 400 before any backend call is made.

**Cache refresh semantics**
The `?refresh=true` query parameter triggers `InMemoryCaseUrnMapperRepositoryImpl.getCaseIdByCaseUrn(caseUrn, true)`, which calls `CaseUrnCacheService.getCaseIdAndRefreshCache()`. This fetches fresh data from the backend and explicitly updates the cache entry, rather than relying on `@CacheEvict` / `@CachePut`.

**CJSCPPUID header is mandatory on every backend call**
`CaseUrnMapperClient.getRequestEntity()` always adds the `CJSCPPUID` header. If the configured value is blank, the client logs an error and passes `null` — causing the backend request to fail. Always set `CJSCPPUID` in local environments.

**Vendor media type on Accept**
Backend requests use `Accept: application/vnd.systemid.mapping+json` (not `application/json`). The CP backend returns 406 if this header is missing.

**Backend path structure**
Full URL built as: `{CP_BACKEND_URL}{case-urn-mapper.path}?sourceId={urn}&targetType=CASE_FILE_ID`
The fixed path is `/system-id-mapper-api/rest/systemid/mappings`.

## Debugging

| Symptom | Cause / Fix |
|---|---|
| HTTP 400 on valid-looking URN | URN contains non-alphanumeric chars or is shorter than 10 / longer than 40 chars — check regex `^[0-9a-zA-Z]{10,40}$` |
| HTTP 406 from backend | Missing or wrong `Accept` header — verify `CaseUrnMapperClient.getRequestEntity()` sets `application/vnd.systemid.mapping+json` |
| Stale cache being returned | Call with `?refresh=true` to force a fresh backend fetch and update the cache |
| `CJSCPPUID` null/empty in logs | `CJSCPPUID` env var not set — service starts but all backend calls will fail; set to a valid UUID |
| Won't start | Check port 4550 is free; `CP_BACKEND_URL` must be reachable at startup if any warm-up calls are configured |

## Repo-Specific Notes

The `application.yaml` sets `management.endpoints.web.base-path: /` (root), so health is at `/health` and prometheus at `/prometheus` — not the standard `/actuator/*` paths used in other services.

The `spring.config.import: "optional:configtree:/mnt/secrets/rpe/"` pattern reads Azure Key Vault–mounted secrets from the pod filesystem in AKS; locally this path is absent and the import is silently skipped.
