## Repo: api-cp-crime-hearing-results-document-subscription

OpenAPI-first API spec library that defines the Crime Hearing Results Document Subscription contract, enabling consumers to register webhooks and receive PCR document notifications with HMAC-signed callbacks.

**Pattern**: Pure spec-only
**OpenAPI spec version**: 3.1.0
**OpenAPI Generator version**: 7.21.0 (current — target 7.22.0 per upgrade cycle)
**Spring Boot version**: 4.0.5 (current — target 4.0.6+ per upgrade cycle)

## API Endpoint(s)

```
GET /event-types
  → 200 EventTypeResponse
  → 401 (no body)
  → 403 (no body)

POST /client-subscriptions
  → 201 ClientSubscription (includes HmacCredentials — returned once only)
  → 400 ErrorResponse
  → 401 (no body)
  → 403 (no body)

GET /client-subscriptions/{clientSubscriptionId}
  → 200 ClientSubscription
  → 401 (no body)
  → 403 (no body)
  → 404 (no body)

PUT /client-subscriptions/{clientSubscriptionId}
  → 200 ClientSubscription
  → 400 ErrorResponse
  → 401 (no body)
  → 403 (no body)
  → 404 (no body)

DELETE /client-subscriptions/{clientSubscriptionId}
  → 204 (no body)
  → 401 (no body)
  → 403 (no body)
  → 404 (no body)

POST /notifications
  → 202 (no body)
  → 400 ErrorResponse
  → 401 (no body)
  → 403 (no body)

GET /client-subscriptions/{clientSubscriptionId}/documents/{documentId}
  → 200 (PDF binary)
  → 401 (no body)
  → 403 (no body)
  → 404 (no body)
```

## Generated Interfaces & Schema

- Schema file: `src/main/resources/openapi/schema/` (JSON schema files validated via AJV in CI)
- Generated API interface(s):
  - `uk.gov.hmcts.cp.openapi.api.SubscriptionApi` — subscription and event-type endpoints
  - `uk.gov.hmcts.cp.openapi.api.NotificationApi` — internal notification trigger and document retrieval (tagged "Internal")
- Generated models:
  - `ClientSubscription` — full subscription record including callback URL, event types, and HMAC credentials
  - `ClientSubscriptionRequest` — request body for creating/updating a subscription
  - `EventPayload` — notification payload sent to the subscriber webhook
  - `HmacCredentials` — HMAC secret returned once at subscription creation; not retrievable again
  - `EventTypeResponse` — list of valid event type strings

## Domain Models

| Model | Purpose |
|---|---|
| `ClientSubscription` | Registered webhook subscription record with callback URL and event type |
| `ClientSubscriptionRequest` | Request body for POST/PUT; `eventTypes` must have exactly one entry (`minItems: 1, maxItems: 1`) |
| `EventPayload` | Payload delivered to subscriber webhook on notification fanout |
| `HmacCredentials` | HMAC-SHA256 secret returned once at creation; used to verify `X-Signature` on callbacks |
| `EventTypeResponse` | Wrapper for the list of valid event type strings |

## Test Structure

| Class | What it validates |
|---|---|
| `uk.gov.hmcts.cp.subscription.OpenAPISpecTest` | Reflection-based; verifies generated model field types/names and API interface method signatures match the spec |
| `uk.gov.hmcts.cp.subscription.ValidateClientSubscriptionRequestTest` | Validates Jakarta validation annotations: HTTPS-only URL regex, `eventTypes` list size = 1, null checks |
| `uk.gov.hmcts.cp.subscription.SubscriptionKeySecurityTest` | Parses `openapi-spec.yml` at runtime with SnakeYAML; validates security schemes and that all subscription endpoints return 401/403 |

## CI/CD Deviations

- `ci-released.yml` passes `-x test` to Gradle on release builds — tests are skipped on the release publish step.
- `ci-draft.yml` injects the generated artefact version into `openapi-spec.yml` via `hmcts/update-openapi-version` before build/publish steps run.
- `lint-openapi.yml` rejects internal HMCTS URLs (`cjscp.org.uk`, `hmcts.net`, `justice.gov.uk`, `ejudiciary.net`, `service.gov.uk`) in the spec.
- No `secrets-scanner.yml` in this repo's workflow set (present in some sibling repos).

## Repo-Specific Notes

- **compileOnly dependency scope**: Spring Boot (`spring-boot-starter-web`, `spring-boot-starter-validation`) and OpenAPI Generator core (`openapi-generator-core`, `swagger-parser`) are declared `compileOnly` — they are NOT included in the published JAR's transitive dependencies. Only `io.swagger.core.v3:swagger-annotations` is `implementation`. See `docs/DEPENDENCIES.md` for consuming this JAR. This is the strictest dependency isolation of all sibling repos.
- **HMAC security**: Callbacks include `X-Key-Id` and `X-Signature` (HMAC-SHA256) headers. The HMAC secret is inside `HmacCredentials` and is returned exactly once at subscription creation — it cannot be retrieved again.
- **Webhook URL constraint**: Callback URLs must match `^https://.*$`; HTTP is rejected by Jakarta validation.
- **Event type constraint**: Each subscription registers exactly one event type; `eventTypes` has `minItems: 1, maxItems: 1`.
- **OpenSpec change workflow**: Proposed API changes are tracked as structured artifacts under `openspec/changes/<change-name>/`. Use `/opsx:propose`, `/opsx:apply`, `/opsx:verify`, and `/opsx:archive` slash commands to manage the full change lifecycle.
- **Key docs**: `docs/NOTIFICATIONS.md` (notification flow with sequence diagram), `docs/DEPENDENCIES.md` (how to consume the JAR), `docs/NOTIFICATION_REQUIREMENTS.md` (functional requirements; consumer is Remand and Sentence Service).
- **Notification fanout flow**: Progression Service generates PCR document → POST /notifications triggers this service → fetches PDF via time-limited SAS URL → resolves subscribers → fans out via Artemis Message Broker → webhooks delivered through APIM with HMAC-signed requests → failed webhooks retry with exponential backoff; exhausted retries go to Dead Letter Queue.
- **OpenAPI spec version**: 3.1.0 (all other sibling repos use 3.0.0 or 3.0.x).
- Run `/openapi-spec-reviewer` skill when authoring or reviewing the OpenAPI spec.
