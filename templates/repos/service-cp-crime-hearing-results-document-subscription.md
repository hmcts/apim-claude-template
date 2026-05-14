## Repo: service-cp-crime-hearing-results-document-subscription

DB-backed event-driven service that receives hearing-results document notifications from Progression/HearingNows services, fans them out to registered subscribers via Azure Service Bus, and serves document content on demand.

**Pattern**: DB-backed
**Spring Boot version**: 4.0.6
**Implements**: `api-cp-crime-hearing-results-document-subscription`

## Infrastructure

| Component | Technology | Purpose |
|---|---|---|
| Database | PostgreSQL 15 (prod) / 18-alpine (docker) | Subscription, event, HMAC, and document-mapping persistence |
| Message broker | Azure Service Bus (or emulator) | Inbound queue `hrds.notifications.inbound`; outbound queue `hrds.notifications.outbound` |
| Key vault | Azure Key Vault | HMAC signing keys for subscriber callbacks |
| Material backend | `MATERIAL_CLIENT_URL` | Fetches material metadata and document bytes |
| Document service | `DOCUMENT_SERVICE_URL` | Stores document mapping records |
| Test doubles | WireMock 3.13.2 | Stubs for material and document service in API tests |
| Service Bus emulator | `mcr.microsoft.com/azure-messaging/servicebus-emulator` + SQL Edge | Local dev / CI substitute for Azure Service Bus |

## Source Structure

**subscription/controllers/**
- `SubscriptionController` — CRUD for client subscriptions and HMAC secret rotation; extracts `clientId` from MDC via `ClientIdResolutionFilter.MDC_CLIENT_ID` on every call
- `NotificationController` — receives inbound PCR events from Progression/HearingNows; queues to `hrds.notifications.inbound` if event type is known; serves document bytes via `getDocument()`
- `MockCallbackController` — test/dev endpoint that receives outbound callback POSTs and logs them
- `GlobalExceptionHandler` — `@RestControllerAdvice`; maps `EntityNotFoundException` → 404, `ResponseStatusException` → passthrough
- `RootController` — health/root endpoint

**subscription/managers/**
- `NotificationManager` — orchestration layer between controllers and services; `processPcrNotification()` drives the full inbound pipeline; `getDocumentContent()` retrieves document bytes

**subscription/services/**
- `SubscriptionService` — create/update/delete/get client subscriptions; wraps `ClientRepository` and `ClientEventRepository`
- `SubscriptionValidationService` — validates client and subscription existence before mutations
- `EventTypeService` — reads `EventTypeEntity` records; `eventExists()` check used in `NotificationController`
- `NotificationService` — `processInboundEvent()`: polls for material metadata via Awaitility, saves document mapping
- `MaterialService` — calls `MaterialClient` to wait for material metadata using Awaitility polling
- `DocumentService` — saves `DocumentMappingEntity` records
- `CallbackDeliveryService` — fans out outbound PCR events per subscriber; queues to `hrds.notifications.outbound` or calls callback directly if Service Bus disabled
- `JsonMapper` — `ObjectMapper` wrapper; serialise/deserialise event payloads
- `ClockService` — `Clock` abstraction for testable time operations

**subscription/clients/**
- `CallbackClient` — `RestTemplate` HTTP client; POSTs signed callback payloads to subscriber endpoints
- `MaterialClient` — fetches material metadata from `MATERIAL_CLIENT_URL`; sets `CJSCPPUID` header
- `MaterialDocumentClient` — downloads document bytes from material service

**subscription/repositories/**
- `ClientRepository` — JPA; `ClientEntity` (UUID PK)
- `ClientEventRepository` — JPA; `ClientEventEntity`
- `ClientHmacRepository` — JPA; `ClientHmacEntity`
- `DocumentMappingRepository` — JPA; `DocumentMappingEntity`
- `EventTypeRepository` — JPA; `EventTypeEntity` (seeded via Flyway)

**subscription/mappers/**
- `ClientEntityMapper`, `ClientEventEntityMapper`, `ClientHmacMapper`, `ClientSubscriptionMapper`, `DocumentMapper`, `EventTypeMapper`, `NotificationMapper` — MapStruct entity ↔ DTO mappers; never edit generated `*Impl` classes

**subscription/entities/**
- `ClientEntity`, `ClientEventEntity`, `ClientHmacEntity`, `DocumentMappingEntity`, `EventTypeEntity` — JPA entities; all use UUID PKs with `@GeneratedValue(strategy = GenerationType.UUID)`

**subscription/model/**
- `DocumentContent` — internal DTO wrapping document bytes, content type, and filename
- `EventNotificationPayloadWrapper`, `MaterialMetadata` — internal DTOs for event pipeline

**servicebus/services/**
- `ServiceBusClientService` — `queueMessage()` and `receiveMessages()` abstraction over the Azure SDK processor
- `ServiceBusProcessorService` — starts/stops `ServiceBusProcessorClient` instances for inbound and outbound queues
- `ServiceBusHandlers` — message-received and error handler callbacks wired to `ServiceBusProcessorService`
- `ServiceBusClientFactory` — creates `ServiceBusSenderClient` and `ServiceBusProcessorClient` via SDK
- `ServiceBusAdminService` — ensures queues exist via `ServiceBusAdminInterface`
- `ServiceBusRetryService` — implements exponential retry with configurable delay steps from `service-bus.retry-msecs`

**servicebus/admin/**
- `ServiceBusAdminInterface` — abstraction for queue admin operations
- `ServiceBusAdminAzureImpl` — Azure `ServiceBusAdministrationClient`-backed implementation
- `ServiceBusAdminEmulatorImpl` — emulator-backed implementation (detected via connection string containing `sb://` not `https`)
- `ServiceBusAdminBase` — shared logic

**servicebus/config/**
- `ServiceBusProperties` — binds `service-bus.admin-connection`, `service-bus.connection`, `service-bus.max-tries`; `isEmulator()` returns true when connection string lacks `https`
- `ServiceBusAdminConfiguration` — `@Bean` wiring for admin interface selection
- `RetryServiceConfig` — parses `service-bus.retry-msecs` comma-separated string into `List<Long>`

**hmac/**
- `HmacManager` — top-level HMAC signing coordinator
- `HmacKeyService` — loads HMAC keys; fetches from `SecretStoreServiceInterface`
- `HmacSigningService` — computes HMAC-SHA256 signatures for callback payloads
- `EncodingService` — Base64 encode/decode helpers
- `KeyPair` — model holding public/private key material

**vault/**
- `VaultServiceInterface` — abstraction; implementations: `SecretStoreServiceAzureImpl` (Azure Key Vault SDK), `SecretStoreServiceStubImpl` (local stub), `SecretStoreServiceDebug` (debug logging wrapper)
- `VaultServiceConfiguration` — selects implementation based on `vault.enabled`
- `VaultServiceProperties` — binds `vault.enabled`, `vault.uri`, `vault.client-id`

**filters/**
- `ClientIdResolutionFilter` — extracts client ID from JWT bearer token; stores in MDC as `MDC_CLIENT_ID`; required by every repository and subscription operation
- `TracingFilter` — propagates `X-Correlation-Id` header via MDC

**PostStartup** — `@PostConstruct` bean that registers Service Bus processors; the app health endpoint returns UP only after this completes (allows up to ~120 s)

## Environment Variables

| Variable | Purpose | Default |
|---|---|---|
| `SERVER_PORT` | HTTP server port | `4550` |
| `ENVIRONMENT_NAME` | Environment name enum (`DEV`, `NONLIVE`, `LIVE`, etc.) | `UNKNOWN` |
| `DATASOURCE_URL` | PostgreSQL JDBC URL | `jdbc:postgresql://localhost:5432/appdb` |
| `DATASOURCE_USERNAME` | PostgreSQL username | `postgres` |
| `DATASOURCE_PASSWORD` | PostgreSQL password | `postgres` |
| `HIKARI_MAX_POOL_SIZE` | HikariCP max pool size | `8` |
| `MATERIAL_CLIENT_URL` | Base URL for material metadata/document service | `http://localhost:8081` |
| `CJSCPPUID` | User UUID sent as `CJSCPPUID` header to material client | `00000000-0000-0000-0000-000000000000` |
| `AZURE_SERVICE_BUS_ADMIN_URI` | Service Bus admin connection string | emulator default (`sb://localhost:5300;...`) |
| `AZURE_SERVICE_BUS_URI` | Service Bus data-plane connection string | emulator default (`sb://localhost;...`) |
| `SERVICE_BUS_RETRY_SECONDS` | Comma-separated retry delay millis | `0,1000,2000,10000,30000,60000` |
| `SERVICE_BUS_MAX_TRIES` | Max delivery attempts | `5` |
| `AZURE_SERVICE_BUS_AUTO_START_PROCESSORS` | Whether to auto-start Service Bus processors on startup | `true` |
| `DOCUMENT_SERVICE_URL` | Base URL for document storage service | `http://localhost:8082` |
| `SUBSCRIPTION_OAUTH_ENABLED` | Whether OAuth validation is enabled for subscription endpoints | `true` |
| `AZURE_VAULT_ENABLED` | Whether Azure Key Vault is used for HMAC keys | `true` |
| `AZURE_VAULT_URI` | Azure Key Vault URI | _(empty)_ |
| `AZURE_CLIENT_ID` | Azure managed identity / service principal client ID for Key Vault | `00000000-0000-0000-0000-000000000000` |

## Repo-Specific Architecture Rules

**Client ID is mandatory in every query**
Every controller and repository call must include the client ID extracted with:
```java
UUID.fromString(MDC.get(ClientIdResolutionFilter.MDC_CLIENT_ID))
```
The `ClientIdResolutionFilter` populates this from the JWT bearer token on every request. Operations attempted without a valid client ID in MDC will throw `IllegalArgumentException` or produce incorrect data silently.

**Service layering — no layer skipping**
Controller → Manager → Service → Client/Repository. Controllers must not call Services directly when a Manager exists. Controllers must not call Repositories directly.

**Event Processing Pipeline**

```
PCR Inbound Event (from Progression/HearingNows)
    |
NotificationController.createNotification()
    |
    +--> [Service Bus enabled] queue to hrds.notifications.inbound
    |
    +--> [Service Bus disabled] synchronous path
    |
ServiceBusHandlers / NotificationManager.processPcrNotification()
    |
NotificationService.processInboundEvent()
    |
    +-> MaterialService.waitForMaterialMetadata()  [Awaitility polling]
    +-> DocumentService.saveDocumentMapping()
    |
CallbackDeliveryService.submitOutboundPcrEvents()
    |
    +--> [Service Bus enabled] queue to hrds.notifications.outbound per subscriber
    +--> [Service Bus disabled] CallbackClient.sendToSubscriber()
```

**Service Bus emulator vs Azure detection**
`ServiceBusProperties.isEmulator()` returns `true` when the connection string does not contain `https`. Azure connection strings start with `Endpoint=sb://...;...EntityPath=...` and contain `https`. The emulator uses `sb://localhost`. The admin and data-plane clients are wired accordingly in `ServiceBusAdminConfiguration`.

**Flyway migrations**
Migrations live in `src/main/resources/db/migration/` using naming `V<VERSION>__<description>.sql`. `baseline-on-migrate: true` is set. Migrations auto-run on `bootRun` and test startup (Testcontainers spins a real PostgreSQL).

**HMAC key rotation**
`SubscriptionController.rotateClientSubscriptionSecret()` rotates the HMAC key for a subscription. New keys are stored in Azure Key Vault when `vault.enabled=true`; locally the stub implementation is used.

## Debugging

| Symptom | Cause / Fix |
|---|---|
| Won't start | PostgreSQL running and reachable? `DATASOURCE_URL` correct? Port 4550 free? |
| App takes 60–120 s to become healthy | Normal — `PostStartup` polls Service Bus emulator until queues are ready before HTTP server accepts traffic; `start_period: 60s` in docker health check accounts for this |
| Material timeout / `ConditionTimeoutException` | Tune `MATERIAL_CLIENT_TIMEOUT_MSECS` / `MATERIAL_CLIENT_INTERVAL_MSECS` (if added as env vars); check material service is reachable at `MATERIAL_CLIENT_URL` |
| PMD failures | Check `.github/pmd-ruleset.xml`; most common failures are cyclomatic complexity and method naming |
| Test failures | Run with `-i`; check MDC setup in filter chain — `ClientIdResolutionFilter` must fire before any controller |
| Service Bus connection refused | Toggle `AZURE_SERVICE_BUS_AUTO_START_PROCESSORS=false` to skip processor startup; check emulator connection strings |
| `IllegalArgumentException: Invalid UUID` in controller | `ClientIdResolutionFilter` did not populate MDC — JWT token missing, malformed, or filter not registered |
| Flyway migration failure | Check `V<N>__*.sql` naming; if baseline needed, set `baseline-on-migrate: true` (already set); check PostgreSQL user has DDL privileges |

## Repo-Specific Notes

**Test structure deviation**: Unit and integration tests use Testcontainers (`@SpringBootTest` + `postgres:18-alpine`) — no local PostgreSQL required for `./gradlew test`. The full docker stack (`./gradlew dockerTest`) adds Service Bus emulator + SQL Edge.

**Apple Silicon warning**: SQL Edge (`mcr.microsoft.com/azure-sql-edge`) must be run with `platform: linux/amd64` on M-chip Macs — see `docker/docker-compose.yml` comments.

**OpenAPI generation**: Run `./gradlew openApiGenerate` after spec changes; never edit `build/generated/` directly.

**`MockCallbackController`**: A dev/test endpoint that accepts inbound callback POSTs. Only active in non-production environments. Do not remove — it is used in API tests to verify end-to-end callback delivery.

**`PostStartup`**: The `@PostConstruct` bean in `PostStartup.java` wires Service Bus processor subscriptions. If `AZURE_SERVICE_BUS_AUTO_START_PROCESSORS=false`, processors must be started manually or via the admin API. The actuator health endpoint reflects Service Bus readiness.
