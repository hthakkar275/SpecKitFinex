# HTTP API Contract

Normative companion to [openapi.json](openapi.json), [spec.md](../spec.md), and
[data-model.md](../data-model.md). Scope: the planned Spring Boot 4.1.0 / JDK 21 in-memory service.

## Addresses, Identity, and Headers

Local base addresses: internal `http://127.0.0.1:8080/internal`, external
`http://127.0.0.1:8080`, public root `http://127.0.0.1:8080`.
Network deployments use TLS at the same logical bases. Internal paths require Administrator;
external paths require an active Broker. Participants cannot read or mutate internal resources.
There is no implicit administrator right to submit orders as a Broker. Security matching processes
`/internal/**` first with Administrator-only access; exact `GET /status` is public; the explicitly
listed root participant resources require an active Broker. Deny all other resource/method matches
with the appropriate JSON error. An unprefixed participant-management path does not become public
or mirror the internal API. Root-level product writes are not administrator aliases.

Authentication uses stateless HTTP Basic. Administrator bootstrap requires
`FINEX_ADMIN_USERNAME` and `FINEX_ADMIN_PASSWORD`; missing/invalid bootstrap configuration fails
startup rather than creating a default credential. Administrator Broker creation binds unique
username and password to the generated Broker ID. Passwords are write-only, BCrypt-hashed, and
never returned. Credential rotation and deletion revoke the old generation immediately.

Every application HTTP response has `X-Correlation-ID` (generated UUID, untrusted supplied values
replaced), JSON Content-Type, and the specified envelope. GET requests need no body, Content-Type,
or Accept header. JSON-body requests require `Content-Type: application/json`. Every mutation,
including administrator changes and DELETE, requires `Idempotency-Key` (1–128 printable ASCII).
A replay gets a new correlation header but the original status/code/business body.

## Operations

Paths below are relative to the indicated base. IDs are generated UUID strings. Type values and
enums are uppercase. Missing explicit product/participant type is a validation failure; unsupported
types use 9000. Product symbol lookup is a query, so it cannot collide with the ID path.

| Base | Method/path | Result and access |
| --- | --- | --- |
| Internal | POST `/product/equity` | Create Equity; 201 / 1001. |
| Internal, External | GET `/product/equity` | Bounded catalog or `?symbol=ABC` lookup; 200 / 1003; missing symbol 200 / 1004 (`success=false`). |
| Internal, External | GET `/product/equity/{id}` | Product by ID; 200 / 1003 or 200 / 1004 (`success=false`). |
| Internal | PATCH `/product/equity/{id}` | Description/symbol/tick update; 200 / 1005. |
| Internal | DELETE `/product/equity/{id}` | Logical deletion; 200 / 1006, data identifies target. |
| Internal | POST `/participant/broker` | Broker and identity creation; 201 / 1007. |
| Internal | GET `/participant/broker` or `/{id}` | Broker catalog or individual; 200 / 1009; missing ID 200 / 1010 (`success=false`). |
| Internal | PATCH `/participant/broker/{id}` | Name and/or password rotation; 200 / 1011. |
| Internal | DELETE `/participant/broker/{id}` | Disable identity/logical delete; 200 / 1012. |
| External | POST `/order` | Submit for authenticated Broker; 201 / 1013, or 1018 if fully filled. |
| External | GET `/order/{id}` | Owned report; 200 / 1015, or 200 / 1016 (`success=false`). |
| External | PATCH `/order/{id}` | Price/total amendment with expectedVersion; 200 / 1019, or 1018 if filled. |
| External | DELETE `/order/{id}` | Cancel latest remainder, no expected version; 200 / 1017. |
| External | GET `/order/trades` | Own trade page, optional productId/orderId filters; 200 / 0. |
| External | GET `/order/trades/{id}` | Own trade by ID; 200 / 0, or 200 / 999999 + TRADE_NOT_FOUND (`success=false`). |
| External | GET `/order/{id}/trades` | Complete owned order trade references via pages; 200 / 0. |
| External | GET `/orderBook/{productId}` | Depth + metrics from one revision; 200 / 1020, including empty book. |
| Public | GET `/status` | `{running:true}`; 200 / 0, no authentication. |

`/order/trades` is a static subresource and takes precedence over order-ID matching. Order submission
omits participantId; an explicitly supplied matching participantId may be accepted for compatibility,
but a different ID is 403 / 999999 + FORBIDDEN. It never chooses the authenticated identity.
Query parameters `side=BUY|SELL&price=<decimal-string>` on a book request add the exact level-volume
calculation; both must be supplied together. All other metrics are included in the same snapshot.

## Envelope, Values, and Outcome Rules

Handled business decisions return HTTP 200, including missing entities, unsupported entities,
semantic field validation, duplicates, version conflicts, invalid cancellation/amendment,
idempotency validation, and stale book cursors. A rejection has `success=false` and its existing
numeric code plus `data.reasonCode`. Successful creations retain HTTP 201. Request/protocol,
security, throttling/admission, and execution/system failures retain documented HTTP 4xx/5xx.
A syntactically valid JSON body with invalid business fields MUST reach the business error mapper;
Spring/Jackson/Bean Validation defaults MUST NOT emit 400/409/422 for these handled outcomes.
Every response has `Cache-Control: no-store`; monitoring classifies outcome from the envelope.

Exactly five mandatory top-level fields: boolean `success`, integer `responseCode`, string
`infoMessage`, `warningMessage`, `errorMessage`; `data` when relevant. Unused messages are empty
strings. Errors contain `data.reasonCode`, no private/partial data. Decimal prices/ticks/spreads/
midpoints are strings; quantities, versions, and codes are integers. UTC times are English
`dd-MMM-yyyy HH:mm:ss.SSS`. Absent market price and unavailable book price metrics are omitted.

Use the exact numeric catalog and precedence in the feature specification. Critical examples:

| Outcome | HTTP / numeric code / reason |
| --- | --- |
| Invalid product, duplicate symbol, product in use | 200 / 1002 / INVALID_PRODUCT, PRODUCT_SYMBOL_EXISTS, PRODUCT_IN_USE |
| Invalid participant, username duplicate, or participant in use | 200 / 1008 / INVALID_PARTICIPANT, USERNAME_EXISTS, PARTICIPANT_IN_USE |
| Unsupported product/participant type | 200 / 9000 / UNSUPPORTED_PRODUCT_TYPE or UNSUPPORTED_PARTICIPANT_TYPE |
| Invalid order quantity/side/type/price | 200 / 1014 / specific reason from spec |
| Order references absent product/participant | 200 / 1014 / PRODUCT_NOT_FOUND or PARTICIPANT_NOT_FOUND |
| Stale amendment | 200 / 1014 / ORDER_VERSION_CONFLICT |
| Missing/invalid expectedVersion | 200 / 1014 / MISSING_ORDER_VERSION or INVALID_ORDER_VERSION |
| Cancel already filled | 200 / 1018 / ORDER_ALREADY_FILLED |
| Cancel already cancelled | 200 / 1014 / ORDER_ALREADY_CANCELLED |
| Missing target order | 200 / 1016 (`success=false`) / ORDER_NOT_FOUND |
| Unknown product book | 200 / 999999 / PRODUCT_NOT_FOUND |
| Authentication/authorization | 401 or 403 / 999999 / UNAUTHENTICATED or FORBIDDEN |
| Wrong media type, route, HTTP method | 415, 404, 405 / 999999 / UNSUPPORTED_MEDIA_TYPE, ROUTE_NOT_FOUND, METHOD_NOT_ALLOWED |
| Invalid payload syntax / payload too large / excessive limit | 400, 413, 200 / 999999 / MALFORMED_REQUEST, REQUEST_TOO_LARGE, REQUEST_LIMIT_EXCEEDED |
| Duplicate pending command wait expires | 503 / 999999 / COMMAND_IN_PROGRESS |
| Admission/lock/history/memory capacity | 503 / 999999 / SERVICE_OVERLOADED |
| Per-caller throttle | 429 / 999999 / RATE_LIMITED |
| Stale book cursor | 200 / 999999 / BOOK_CURSOR_STALE |
| Storage unavailable | 503 / 999999 / STORAGE_UNAVAILABLE |
| Unexpected failure | 500 / 999999 / INTERNAL_ERROR |

Missing idempotency identity uses HTTP 200 with `success=false` and the applicable entity-rejected numeric code;
conflicting reuse uses HTTP 200 with `success=false` and that same rejected code. No business code is silently reused
for a neighboring entity. Unexpected error text is exactly `An unexpected error occurred.`
Retryable overload responses include `Retry-After: 1`; a pending duplicate retries the same key.
A version conflict requires a fresh read and new logical command/key, never blind replay with a
changed expectedVersion. A filled report GET remains 1015, not 1018. No deletion uses HTTP 204.

## Finite Limits and Admission

Defaults are part of this plan; changes require configuration documentation and rerunning acceptance.
Limits are enforced before business effects and never silently reduce requested matching quantities.

| Limit | Default |
| --- | --- |
| Request body / URI / headers | 16 KiB / 8 KiB / 16 KiB; request/read timeout 2 seconds |
| Normalized symbol / description / Broker name | 32 / 512 / 128 characters; raw symbol input ≤96 characters; blank values invalid |
| Username / password | 1–64 restricted ASCII username; 16–64 ASCII password; never log Basic Authorization |
| Quantity | 1–1,000,000,000 shares per order, checked long aggregates |
| Input price / tick | 0 < value ≤1,000,000,000; ≤6 fractional digits; ≤24 characters |
| Product / Broker count | 1,000 / 10,000 including retained identities and tombstones |
| Orders / trades / audit / completed keys | 1,000,000 / 2,000,000 / 2,000,000 / 1,000,000; stricter byte budget wins |
| Open orders / product | 10,000; lifecycle cancellation/expiry capacity reserved on acceptance |
| Matches / command | 2,000; dry-run overflow rejects whole command with SERVICE_OVERLOADED |
| List page / book levels per side | Default 100, maximum 500; no unbounded collection replies |
| Concurrent business admission / connector workers | 128 permits / 160 platform threads; bounded connector backlog 256 |
| Total lock-wait deadline / duplicate wait | 250 ms shared across acquired locks / 250 ms; no nested indefinite waits |
| Per authenticated principal throttle | 200 requests/s with burst 400; load baseline uses 100 principals |
| Memory | -Xmx2560m, soft retained-state admission budget 1,536 MiB, target peak RSS ≤4 GiB |
| Authentication verification | Max 4 simultaneous BCrypt cache misses; successful-cache policy below |

Application filters enforce the soft body/header limits and a configured container error fallback
uses the common JSON writer for connector failures that can produce an application-formatted
response. Oversized headers use HTTP 431/999999/REQUEST_TOO_LARGE. Malformed transport that never
reaches the application or closes without a response is outside the application envelope guarantee.

The match cap is a work-admission limit: calculate the entire prospective result before accepting;
never commit the first 2,000 fills and drop otherwise eligible quantity. Large requests can therefore
be retried after liquidity changes, or divided by the client into independently identified orders.
Use weighted reservations for records, strings, trade references, outcomes, and staged journal entries;
measure estimator accuracy under the baseline. The budget is conservative admission, not a claim
that counting map entries proves actual RSS. Stop admitting growth before reaching the budget;
continue using reserved capacity for cancel/expiry. Do not silently drop required history to recover.

## Authentication Cost and Revocation

BCrypt runs when validating a previously unseen/expired valid credential, not on every hot-path
request. A process-local successful-authentication cache holds at most 10,000 entries for at most
60 seconds, keyed by a per-process HMAC of the Authorization value. Store principal ID and credential
generation, never the raw password/header. Coalesce simultaneous misses for the same credential. Cache misses have four permits; do not
cache failures. Generate the HMAC secret from a cryptographically secure random source at process
startup, enforce the header size limit before hashing, and jitter successful-entry TTL over
45–60 seconds to avoid synchronized expiry storms.
Every request still checks active identity, role, and current credential generation before access;
commands, private reads, and replay recheck under the same metadata gate that protects the
operation; private reads recheck immediately before copying their response. Rotation/delete invalidates immediately,
including requests authenticated before a concurrent administrator change. No stale cache entry
can authorize work. Apply bounded unauthenticated verification admission before BCrypt; never
allow random password attempts to saturate the trading worker pool. A warm-cache benchmark and
cold/revocation tests are separate mandatory evidence, not a reason to omit auth cost from reports.

## Pagination and Consistency

Catalog/trade lists use opaque signed seek cursors with filter identity and initial insertion
high-water mark; stable generated sequence orders pages, excluding later inserts. Logical deletes
may remove active catalog entries on later pages; each individual response is a committed view.
Trade insertion sequences are process-wide monotonic counters assigned only while holding the
trade's product lock; they are never reused or decremented on rollback. Capture a trade-list
watermark only after holding the metadata read gate and all relevant product locks together.
For an unfiltered own-trade list this means all retained product slots, including deleted products,
not just current membership-index entries. This excludes hidden lower-sequence commits; the
metadata gate also excludes new product registration during capture. Catalog registration
sequences are captured under the metadata gate. Order-specific continuation pins the committed
per-order trade ordinal. Trade history is immutable. Cursors are bound to the caller, expire in 5 minutes, and are invalid
after process restart; malformed/expired tokens fail HTTP 400 / 999999 / MALFORMED_REQUEST.

A book cursor pins product revision plus per-side positions. If book revision changed between
pages, fail BOOK_CURSOR_STALE and request a fresh snapshot; never combine levels from revisions.
Metrics describe the full pinned book, not just the displayed page. An order report returns a
bounded initial tradeIds page with `tradeIdsComplete` and continuation metadata; following
`/order/{id}/trades` yields all IDs/details, preserving the complete logical report without
unbounded JSON. Report quantities/version describe the instant of that response; continuation
uses an immutable trade high-water mark so later fills do not silently enter an earlier report.

## Behavioral Verification Obligations

Validate every operation/code and all failure-handler layers; decimal-number coercion must fail.
Race duplicate keys across two products; inject recoverable faults at every commit journal write;
check metadata/open-order counters and priority restoration; cancel with an outdated report;
retry an old successful amendment after further fills; advance an injected UTC clock while a
multi-fill delta is staged; attempt deletion/password rotation during an authenticated command.
Verify book metrics and paged trade history beyond 500 trades, stale cursors, reserved-capacity
cancellation/expiry, byte/record admission, restart loss, and the exact SC-007/SC-008 workload.
