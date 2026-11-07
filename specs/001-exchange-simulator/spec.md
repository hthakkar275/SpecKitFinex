# Feature Specification: Electronic Financial Exchange Simulator

**Feature Branch**: None created; branch creation is optional and no branch hook is registered.

**Feature Directory**: `specs/001-exchange-simulator`

**Created**: 2026-11-06

**Last Updated**: 2026-11-07

**Status**: Draft

**Input**: Provide a small electronic exchange with separate administrator and participant HTTP
REST APIs; Equity products; Broker participants; order entry, amendment, cancellation, reports,
books, health, and automatic price-time matching using the resting order's price. Preserve the
specified order statuses, day longevity, half-up tick rounding, and trade records. Apply the
project constitution's concurrency, error, performance, and replaceable-storage requirements.
Enhancement: use the supplied resource paths, JSON envelope, bodyless reads, date/time format,
and entity-specific numeric result codes. Use the user-confirmed code 0 for successful status
and separate trade reads, and 999999 with a specific reason for otherwise unmapped failures.

## Clarifications

### Session 2026-11-06

- Q: When an order changes after a broker reads its report, should an amendment based on that older report be rejected? → A: Require the current order version for amendments; cancellations remove the latest remaining quantity without requiring a version.
- Q: How should prices and other decimal values be represented in JSON? → A: Use decimal strings for prices, tick sizes, spreads, and midpoints; whole-share quantities remain JSON integers.

### Session 2026-11-07

- Decision: Completed application-level decisions, including rejected orders, missing entities,
  validation failures, and business conflicts, return HTTP 200 with `success=false` and the
  existing application result/reason codes. Request/security/infrastructure failures retain
  their appropriate HTTP 4xx/5xx statuses. This supersedes the earlier business-error statuses.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Administer the Trading Universe (Priority: P1)

An Administrator manages the Equity products and Broker participants that can trade, while
participants remain unable to use the internal administration API.

**Why this priority**: Trading requires identifiable instruments, registered brokers, and a clear
separation between administration and participation.

**Independent Test**: Using administrator and participant identities, create, find, update, and
delete products and participants without submitting orders; confirm access restrictions.

**Acceptance Scenarios**:

1. **Given** an Administrator, **When** they create an Equity with symbol `ABC` and a description,
   **Then** they receive a generated ID, and lookup by ID or symbol returns that product.
2. **Given** an existing product and Broker, **When** an Administrator changes their editable
   attributes, **Then** subsequent lookups show the changes and their generated IDs remain stable.
3. **Given** a request for a non-Equity product or non-Broker participant, **When** it is submitted,
   **Then** the request fails with the specific unsupported-type result and nothing is created.
4. **Given** an authenticated participant, **When** they invoke any internal operation, including
   read operations, **Then** it fails with forbidden access and changes nothing.
5. **Given** an unused product or participant, **When** an Administrator deletes it, **Then** it
   disappears from active lookup and cannot be referenced by a new order.
6. **Given** a product or participant with an open order, **When** deletion is requested, **Then**
   it fails as a conflict and neither the entity nor the order is changed.
7. **Given** a product with symbol `ABC`, **When** another is created with symbol ` abc `,
   **Then** the duplicate normalized symbol is rejected.

### User Story 2 - Submit and Match Orders Fairly (Priority: P1)

A registered Broker submits a buy or sell order and receives its resulting report after all
immediately available matches have been applied.

**Why this priority**: Fair matching and correct trade recording are the exchange's core value.

**Independent Test**: Use preconfigured products and brokers to submit controlled order sequences
and compare resulting trades, reports, and remaining book quantities with the examples below.

**Acceptance Scenarios**:

1. **Given** an empty book, **When** a Broker submits a limit buy of 10 at 100.00, **Then** it is
   assigned an ID and entry time, becomes `BOOKED`, and contributes a bid of 10 at 100.00.
2. **Given** offers of 5 at 99.00 and 8 at 100.00, **When** a limit buy of 10 at 100.00 arrives,
   **Then** trades execute for 5 at 99.00 and 5 at 100.00, the buy is `FILLED`, and 3 remain offered.
3. **Given** two offers at 100.00 accepted in order A then B, **When** a buy can consume only one,
   **Then** A trades first, including when their displayed entry timestamps are equal.
4. **Given** a resting bid of 4 at 101.00, **When** a limit sell of 7 at 100.00 arrives, **Then**
   4 trade at 101.00 and 3 rest as an offer at 100.00 with status `PARTIALLY_BOOKED_FILLED`.
5. **Given** only 3 opposite shares are available, **When** a market order for 5 arrives,
   **Then** 3 trade, the other 2 are dropped, and its status is `PARTIALLY_FILLED` with no book entry.
6. **Given** no opposite orders, **When** a market order arrives, **Then** its report is
   `NOT_FILLED`, no trade is created, and none of its quantity rests.
7. **Given** a default tick of 0.01, **When** prices 10.004 and 10.005 are submitted,
   **Then** their accepted limit prices are 10.00 and 10.01 respectively.
8. **Given** any listed order-validation failure, **When** an order is submitted, **Then** the
   specified failure is returned and no order, trade, book entry, or financial effect is stored.

### User Story 3 - Trade Reliably Under Concurrent Requests (Priority: P1)

Participants can submit concurrent requests and retry after an uncertain response without
overselling an order's remaining quantity, losing a cancellation, or creating duplicate trades.

**Why this priority**: A plausible serial result and correct quantities are essential even under load.

**Independent Test**: Run simultaneous submissions, amendments, cancellations, duplicate retries,
and reads against a preconfigured product; inject a failed state change and reconcile all effects.

**Acceptance Scenarios**:

1. **Given** one resting offer of 10, **When** two buys of 10 arrive concurrently, **Then** total
   executed quantity against that offer is at most 10 and the outcome follows one acceptance order.
2. **Given** an open order, **When** cancellation and an opposing order race, **Then** cancellation
   removes only the quantity left at its turn; no execution occurs after successful cancellation.
3. **Given** simultaneous retries of the same identified command and payload, **When** they finish,
   **Then** exactly one logical operation takes effect and retries return its original outcome.
4. **Given** an existing command identity, **When** its owner reuses it with different content,
   **Then** `IDEMPOTENCY_CONFLICT` is returned without additional effects.
5. **Given** a failure before an operation commits, **When** the caller receives its failure,
   **Then** orders, books, trades, and audit records expose none of that operation's partial effects.
6. **Given** a committed operation whose response is lost, **When** it is retried with the same
   command identity, **Then** its original outcome is returned without re-execution.
7. **Given** matching is underway, **When** a report or snapshot is read, **Then** it reflects one
   complete state before or after a committed operation, never an intermediate update.
8. **Given** a Broker read order version 1 and a later amendment or fill advances it to version 2,
   **When** a new amendment supplies `expectedVersion=1`, **Then** HTTP 200, `success=false`, code 1014, and reason
   `ORDER_VERSION_CONFLICT` are returned with no change to the order, trades, or priority.
9. **Given** a Broker's report is outdated but its order still has booked quantity, **When** it
   cancels without an expected version, **Then** the latest remaining quantity is cancelled at
   that command's turn. Replaying an earlier successful command retains its original outcome
   and does not advance the version again.

### User Story 4 - Amend, Cancel, and Expire Open Orders (Priority: P2)

A Broker controls its remaining exposure by changing or cancelling an order still on the book;
open day orders are removed when their trading day ends.

**Why this priority**: Participants need predictable control over orders and their place in the queue.

**Independent Test**: Start with seeded open orders and verify amendments, cancellation, expiry,
and their races with matching, without requiring administration workflows.

**Acceptance Scenarios**:

1. **Given** a booked buy at 99.00 and an offer at 100.00, **When** its owner changes its price
   to 100.00, **Then** it matches immediately at the resting offer's price before returning.
2. **Given** two same-price orders A then B, **When** A increases quantity or changes its effective
   price, **Then** its remaining quantity gets new time priority; a quantity-only reduction retains
   A's priority, and an effective no-op changes no priority.
3. **Given** an order originally for 10 with 4 traded, **When** its total quantity is amended to 8,
   **Then** 4 remain booked, original quantity remains 10, and effective total becomes 8.
4. **Given** that order, **When** a total quantity of 4 or less is requested, **Then** the amendment
   fails without change; cancellation is used to remove the entire remainder.
5. **Given** an untouched open order or a partially traded open order, **When** its owner cancels,
   **Then** its booked quantity becomes zero and status is `CANCELLED` or `PARTIALLY_CANCELLED`.
6. **Given** an unknown, filled, or previously cancelled order, **When** a new cancellation is attempted,
   **Then** respectively `ORDER_NOT_FOUND`, `ORDER_ALREADY_FILLED`, or `ORDER_ALREADY_CANCELLED`
   is returned, without changing prior trades.
7. **Given** an open day order at the UTC day boundary, **When** the new trading day starts,
   **Then** its remainder is cancelled with reason `DAY_EXPIRED` before any new-day match or read
   can include it; prior trades and reports remain available.

### User Story 5 - Inspect Orders, Trades, and Market Depth (Priority: P2)

A Broker monitors its own orders and trades and sees aggregated depth for any active product.

**Why this priority**: Participants must reconcile executions and understand available liquidity.

**Independent Test**: Query a fixed dataset containing each status, multiple price levels, and
trades between different brokers; compare reports, ownership restrictions, and book calculations.

**Acceptance Scenarios**:

1. **Given** an owned order, **When** its report is requested, **Then** it includes every required
   identity, order attribute, quantity, status, and trade ID, including amendments and removal reason.
2. **Given** an order or trade belonging only to another Broker, **When** it is requested,
   **Then** access fails without disclosing that record's contents or existence.
3. **Given** bids at 98.00 and 99.00 and offers at 101.00 and 102.00, **When** a snapshot is read,
   **Then** bids descend, offers ascend, equal prices aggregate, and only open quantities appear.
4. **Given** those levels, **When** best/worst prices, spread, and midpoint are requested,
   **Then** best/worst bids are 99.00/98.00, best/worst offers are 101.00/102.00,
   spread is 2.00, and midpoint is 100.00; volume agrees with the snapshot.
5. **Given** an active product with no orders, **When** its book is read, **Then** it is an empty
   snapshot with that product's ID and symbol, zero volumes, and absent price metrics.
6. **Given** a trade involving the requesting Broker, **When** it is read, **Then** its ID, time,
   buy order ID, sell order ID, price, and quantity agree with both affected order reports.

### User Story 6 - Integrate Through a Consistent HTTP Contract (Priority: P2)

An administrator or participant integrates a client once with predictable JSON responses,
resource-specific result codes, readable dates, and reads that need no request payload.

**Why this priority**: Stable response conventions let clients interpret business outcomes without
special-case parsing, unnecessary read headers, or exposure of service internals.

**Independent Test**: Exercise the five published resource families with authorized fixtures,
validate every response against the envelope and result-code tables, and inject one unexpected error.

**Acceptance Scenarios**:

1. **Given** an authorized caller, **When** an applicable read of `/product/equity`,
   `/participant/broker`, `/order`, `/orderBook/{productId}`, or public `/status` is issued using
   GET with neither a request body nor `Content-Type`, **Then** it succeeds for valid resources
   and returns JSON in the common envelope. Lookup inputs are path or query parameters.
2. **Given** a successful or failed operation, **When** its JSON response arrives, **Then**
   `success`, integer `responseCode`, `infoMessage`, `warningMessage`, and `errorMessage` are
   present, with `data` carrying relevant results or a machine-readable failure reason.
3. **Given** product and participant lifecycle operations, **When** each succeeds, is rejected,
   or finds no matching entity, **Then** it returns that entity's corresponding supplied code,
   never an order code or another entity's superficially similar outcome code.
4. **Given** a valid order submission, **When** it finishes fully filled, **Then** code 1018 is
   returned; otherwise an accepted order returns 1013. A read of either report returns 1015,
   because reading an already filled order is an order-found outcome.
5. **Given** a request to cancel a filled order, **When** it fails, **Then** HTTP 200,
   `success=false`, code 1018, and `data.reasonCode=ORDER_ALREADY_FILLED` distinguish the failure
   from a successful immediate fill. A new cancellation of a cancelled order uses 1014 and
   `ORDER_ALREADY_CANCELLED`; a replay of its successful cancellation retains 1017.
6. **Given** an order referring to a nonexistent product, **When** it is rejected, **Then** its
   envelope uses order-rejected code 1014 and reason `PRODUCT_NOT_FOUND`; a product lookup for
   that ID uses 1004. Both return HTTP 200 with `success=false`.
7. **Given** an active product with an empty book, **When** it is read, **Then** code 1020 and an
   empty snapshot are returned. An unknown product's book instead fails with HTTP 200, `success=false`, code
   999999, and reason `PRODUCT_NOT_FOUND`.
8. **Given** a returned date or time, **When** serialized, **Then** it has the exact shape
   `06-Nov-2026 14:07:03.009`, using UTC, English month abbreviations, and three millisecond digits.
9. **Given** an unexpected internal failure, **When** it is handled, **Then** HTTP 500, code
   999999, a safe general error message, and a correlation identifier are returned without
   exception text, stack traces, storage details, credentials, or sensitive payloads.
10. **Given** a successful status check or separately retrieved trade, **When** it is returned,
    **Then** `success=true` and code 0 are used. A missing private trade returns HTTP 200 with `success=false`,
    code 999999, and reason `TRADE_NOT_FOUND` without disclosing another Broker's data.
11. **Given** a limit price `"100.01"` and product tick `"0.01"`, **When** valid orders and book
    metrics are returned, **Then** all prices, ticks, spreads, and midpoints are JSON strings
    preserving their exact decimal values, while share quantities remain JSON integers.
    Submitting a JSON number instead of a required decimal string fails with HTTP 200, `success=false`, and the
    applicable entity-rejected code and reason, without creating business effects.

### User Story 7 - Confirm Service Availability (Priority: P3)

An operator or client checks whether the service process is running.

**Why this priority**: A minimal liveness check supports operation and automated monitoring.

**Independent Test**: Invoke the status endpoint without trading data or credentials.

**Acceptance Scenarios**:

1. **Given** a running service, **When** status is requested, **Then** it returns HTTP 200 with
   an explicit running indicator and no sensitive configuration or trading data.
2. **Given** a service under trading load, **When** status is requested within the declared load
   envelope, **Then** the liveness response remains available within the stated response targets.

### Edge Cases

- An unknown product differs from a valid product whose book has never been used: the former
  fails; the latter returns an empty book. Books never mix products.
- Missing, zero, negative, fractional, non-finite, malformed, or out-of-range quantities fail.
  Missing or non-positive limit prices fail; rounding a positive price to zero also fails.
- Tick rounding uses the nearest multiple of the tick, not merely the same decimal-place count:
  with tick 0.05, 10.025 becomes 10.05 and 10.024 becomes 10.00.
- Orders with equal timestamps still have an unambiguous priority. Client-provided timestamps
  cannot improve priority; concurrent commands are ordered by service acceptance, not response time.
- A fully filled market order is `FILLED`; a market remainder is dropped, never cancelled or booked.
- A price change whose rounded value is unchanged is not an effective price change. An amendment
  that would be invalid changes neither the order nor its priority.
- An amendment cannot change side, type, product, participant, or longevity, and cannot revive a
  filled, cancelled, dropped, or expired order.
- Matching, cancellation, amendment, deletion, and day expiry that race follow the same accepted
  ordering; successful deletion cannot leave an active order for a deleted entity.
- Empty sides have absent best/worst prices; spread and midpoint are absent unless both sides
  have prices. Missing price-level volume is zero; midpoint is exact and need not be tick-aligned.
- Deleted entities' prior orders and trades remain referentially valid and historically queryable
  by authorized users; deletion prevents new use and never cascades into historical executions.
- Self-matching is allowed in this simulator; no self-trade prevention rule overrides the supplied
  price-time rules. Rejected commands may leave diagnostic records, but not rejected order records.
- A request that loses its connection after commit has an unknown client-visible outcome until
  retried or queried; loss of a response never justifies creating a second logical command.
- Retrying a successful cancellation with its original command identity returns its original
  success; a new cancellation identity for that order returns `ORDER_ALREADY_CANCELLED`.
- A GET without a body or `Content-Type` is valid even if corresponding write operations require
  JSON bodies. Missing resources still produce their ordinary not-found envelope, not a media error.
- Millisecond display precision does not break timestamp ties; matching uses the separate priority.
- A JSON numeric price such as `100.01` is a field-validation failure; the valid JSON price is
  the string `"100.01"`. Decimal formatting does not change tick rounding or positive-price rules.
- A stale amendment is rejected even if its proposed values would now be a no-op. After a version
  conflict the Broker must read the current report and use a new command identity to amend again;
  cancellation needs no expected version and always targets the latest remainder.
- Deletion returns a JSON envelope rather than an empty response, so the deleted result code is
  always available. Partial or wholly unfilled market orders remain accepted business outcomes.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The service MUST provide distinct internal administrator and external participant
  HTTP REST interfaces and the liveness endpoint. There is no user interface in this feature.
- **FR-002**: Every operation except liveness MUST authenticate its caller and enforce its role.
  Participants MUST be forbidden from all internal operations. Administrator authority alone
  MUST NOT grant Broker order-entry rights.
- **FR-003**: A participant identity MUST be bound to one active Broker. Submission, amendment,
  cancellation, and private reports MUST use that identity, never trust an arbitrary supplied
  participant ID, and MUST reject attempts to act for another Broker.
- **FR-004**: Participants MUST access only their own order reports and trades involving one of
  their orders. Active product lookup and aggregated books MUST be available to every active
  participant; participant administration and participant lists remain internal.
- **FR-005**: Administrators MUST create, retrieve by ID, list, update, and delete products.
  Products MUST have a generated immutable ID, a required unique normalized symbol, a nonempty
  description, type Equity, and a positive tick size defaulting to 0.01; lookup by symbol is required.
- **FR-006**: Product type MUST be explicit on creation; any value other than Equity MUST fail
  with `UNSUPPORTED_PRODUCT_TYPE`. Symbols MUST be trimmed and normalized to uppercase.
  Blank or duplicate symbols and invalid tick sizes MUST fail without changing existing products.
- **FR-007**: Administrators MUST create, retrieve by ID, list, update, and delete participants.
  Participants MUST have a generated immutable ID, a nonempty name, and explicit type Broker;
  other types MUST fail with `UNSUPPORTED_PARTICIPANT_TYPE`. Names need not be unique.
- **FR-008**: Deleting a product or participant with booked quantity MUST fail as a conflict.
  Otherwise deletion MUST prevent active lookup and new orders while retaining historical
  references. Deleted participant identities MUST lose access. A product symbol or tick change
  with booked orders MUST fail; description and participant-name changes MAY proceed.
- **FR-009**: An active Broker MUST be able to submit an order identifying an active product,
  side buy or sell, type limit or market, and positive whole-share quantity. A limit order
  MUST supply a positive price. Every accepted order MUST have day longevity, a generated
  immutable order ID, an immutable original entry time, and a deterministic acceptance priority.
- **FR-010**: An unknown or inactive product or participant, missing or unsupported side/type,
  invalid quantity, or missing/non-positive limit price MUST reject the order before storage.
  Failed validation MUST create no order report, trade, or book change; `REJECTED` is reserved.
- **FR-011**: Limit prices MUST be normalized to the nearest multiple of the product's tick using
  half-up rounding before matching, queue placement, or reporting. The normalized price MUST
  remain positive. Financial values MUST use exact arithmetic; permitted ranges and maximum
  precision MUST be published and excess values rejected rather than truncated or overflowed.
  Prices, tick sizes, spreads, and midpoints MUST be represented as decimal strings in JSON
  requests and responses, including order reports, trade records, and book price levels.
  Strings MUST express the exact base-10 value using a decimal point where needed, without
  locale-specific grouping or exponent notation. Share quantities, order versions, and numeric
  response codes MUST remain JSON integers. A numeric or malformed JSON price value MUST fail
  with HTTP 200, `success=false`, and `INVALID_LIMIT_PRICE`; an invalid tick representation MUST fail with
  HTTP 200, `success=false`, and `INVALID_PRODUCT`, using the applicable entity-rejected numeric code.
- **FR-012**: Market orders MUST carry no effective price and MUST report price as absent.
  Supplying a price with a market order or unsupported longevity MUST fail explicitly.
- **FR-013**: A product MUST have one logical order book, created only when needed. An active
  product's unused book MUST be observable as empty without requiring a prior order.
- **FR-014**: Incoming buy and sell orders MUST match only opposite-side resting orders for
  the same product, with best price first and earliest current priority at equal prices.
  Lowest offers are best for buys; highest bids are best for sells.
- **FR-015**: A limit buy MUST match only prices at or below its normalized limit; a limit sell
  MUST match only prices at or above its limit. A market order MUST consume available opposite
  prices in priority order until filled or liquidity is exhausted.
- **FR-016**: Each match MUST execute at the resting order's price for the smaller of the two
  remaining quantities, create exactly one immutable trade, and update both orders together.
  An order MUST NOT execute more than its effective total minus already traded quantity.
- **FR-017**: Matching MUST finish all immediately eligible matches before the command outcome
  is returned. Remaining limit quantity MUST rest; remaining market quantity MUST be dropped.
  Fully filled, cancelled, and expired orders MUST immediately leave eligibility to trade.
- **FR-018**: An owner MUST be able to amend price, total quantity, or both for a limit order
  with booked quantity. The new total MUST be a positive integer strictly greater than the
  already traded quantity. An effective price change or total increase MUST reset time priority;
  a quantity-only reduction or effective no-op MUST retain priority. Original ID and entry time
  MUST remain unchanged, and a marketable amendment MUST match immediately as the incoming order.
  Every amendment MUST supply a positive integer `expectedVersion` equal to the order report's
  current `version`. After access and input validation, this comparison MUST occur atomically
  before checking amendability or changing state. A mismatch MUST return HTTP 200, `success=false`, code 1014,
  and reason `ORDER_VERSION_CONFLICT` with no business effects.
- **FR-019**: Original quantity MUST remain the submitted quantity. Each report MUST additionally
  expose the current effective total quantity, so quantity increases and reductions remain
  reconcilable. The latest accepted amended total replaces the prior effective total; prior
  executions MUST remain immutable. Invalid amendments MUST leave all state and priority unchanged.
- **FR-020**: An owner MUST be able to cancel the entire remaining booked quantity. Successful
  cancellation MUST remove that quantity immediately, preserve executions, record the removal
  reason `PARTICIPANT_CANCELLED`, and select `CANCELLED` or `PARTIALLY_CANCELLED` according to
  traded quantity. Cancellation MUST NOT require an expected version and MUST act on the latest
  remaining quantity at its turn. Dropping a market remainder MUST record reason
  `MARKET_REMAINDER_DROPPED`.
- **FR-021**: Cancellation MUST distinguish an unknown/inaccessible order, a filled order,
  an already-cancelled order (including day expiry), and a market order with no booked remainder.
  An amendment with a matching expected version MUST reject a terminal order with the specific
  non-amendable result; a stale expected version MUST return the version-conflict result first.
- **FR-022**: Day orders MUST expire at the end of their UTC calendar day. Before any new-day
  operation reads or matches a book, prior-day remainders MUST be removed atomically, with reason
  `DAY_EXPIRED` and the applicable cancellation status. Terminal reports and trades MUST remain.
- **FR-023**: A Broker MUST retrieve an owned order report with order/product/participant IDs;
  side, type, day longevity, normalized price if applicable; original entry time; original,
  effective-total, traded, booked, and removed quantities; status; removal reason if applicable;
  its current positive integer `version`; and IDs of all its trades. Traded + booked + removed
  MUST equal effective total. Amendment reductions are reflected in effective total, not added
  to removed quantity.
- **FR-024**: Status MUST follow the Order Status Rules below for all submission, matching,
  amendment, cancellation, and expiry outcomes. Rejected orders MUST NOT become stored reports.
- **FR-025**: A Broker MUST retrieve a trade by ID and list its own trades. Each trade MUST record
  generated ID, execution time, buy order ID, sell order ID, exact price, and positive quantity.
  Trade data exposed to both counterparties MUST agree and MUST NOT expose unrelated private reports.
- **FR-026**: A book snapshot MUST identify the active product by ID and current symbol, aggregate
  only remaining booked quantities at each normalized price, sort bids descending and offers
  ascending, and omit zero-volume levels. It MUST expose no order or participant identities.
- **FR-027**: A Broker MUST obtain best/worst bid and offer prices, spread (best offer minus best
  bid), midpoint (their arithmetic mean), volume at a specified side and exact price, and total
  volume per side. An absent side MUST have zero volume and absent price metrics; a missing price
  MUST have zero level volume, and a one-sided book MUST have absent spread and midpoint.
- **FR-028**: Liveness MUST return HTTP 200 with an explicit running indication without credentials
  or trading data. It confirms the running service, not persistent durability or market readiness.
- **FR-029**: Every completed business decision MUST use HTTP 200 when rejected, with
  `success=false`, a numeric `responseCode` selected by the Result Code Selection rules,
  a specific symbolic `data.reasonCode`, a safe `errorMessage`, and `X-Correlation-ID`.
  This includes semantic field validation, missing entities, unsupported entity types,
  idempotency validation, and business conflicts. Request parsing/protocol, authentication,
  authorization, routing, request-size/media, throttling/admission, and execution/infrastructure
  failures MUST retain the HTTP 4xx/5xx statuses in the Failure Outcomes table, with the same
  error envelope. Framework defaults MUST NOT convert a handled business rejection to 4xx.
  Symbolic failure names are reason codes, not substitutes for numeric `responseCode`.
- **FR-030**: Mutating commands MUST accept a caller-scoped idempotency identity. Within its
  retention period, simultaneous or later retries of the same identity, operation, and payload
  MUST return the original command outcome without repeated effects. Conflicting payload or
  operation reuse MUST fail with `IDEMPOTENCY_CONFLICT`; different callers' identities MUST not
  collide. Authentication and authorization MUST still be checked on every retry.
- **FR-031**: A failed command before commit MUST leave no partial business effects. Once committed,
  its outcome MUST remain discoverable despite response loss. A retriable failure before commit
  MUST permit a subsequent attempt with the same identity; stored rejection outcomes MUST be
  replayed consistently. Requests lacking a valid command identity MUST fail before mutation.
- **FR-032**: The complete effects of a command, including each match, both order updates, book
  changes, trades, idempotency outcome, and attributable audit record, MUST become visible
  together. Conflicting commands MUST have one service-defined acceptance order with no
  duplicate fills, negative quantities, stale amendments, or trades after successful cancellation.
  An accepted order MUST start at version 1. Each subsequent committed command that changes an
  order, including matching, amendment, cancellation, or expiry, MUST advance that order's version
  once, even if that command creates several fills. Failed commands, effective no-ops, and
  idempotent replays MUST NOT advance versions. Replay MUST return the original outcome before
  comparing the original amendment's expected version against the now-current version.
- **FR-033**: Each read MUST reflect a complete committed state. Separate reads MAY observe
  intervening commands; a report's quantities and trade IDs and an individual book snapshot
  MUST be internally consistent. Accepted priority MUST resolve equal displayed timestamps.
- **FR-034**: The proof of concept MAY retain state in memory and lose it on restart. Its supported
  deployment and lifetime MUST be explicit. Replacing storage MUST preserve observable matching,
  concurrency, error, and API behavior without changing business rules. Persistent durability,
  migration, and recovery MUST be defined before a persistent mode is adopted.
- **FR-035**: Mutation history MUST identify actor, command, ordered outcome, and related orders
  and trades, without leaking credentials. Correlated operational records MUST support measuring
  latency, completed business work, error results, contention, and saturation. Audit retention
  MUST match the advertised storage lifetime.
- **FR-036**: Requests, numeric ranges, collection pages, waits, and overload behavior MUST have
  documented finite limits before implementation. Lists MUST use bounded pages and deterministic
  ordering, with continuation behavior documented for concurrent changes. Inputs exceeding a
  published limit MUST fail explicitly, never silently truncate requested financial operations.
- **FR-037**: Within the acceptance workload the service MUST meet SC-007. Beyond capacity it MUST
  bound resource use, return explicit overload outcomes, preserve invariants, and recover as
  specified by SC-008. Error replies MUST NOT be counted as successfully completed business work.
- **FR-038**: The service MUST exchange JSON over HTTP using `/product/equity`,
  `/participant/broker`, `/order`, `/orderBook/{productId}`, and `/status` as the resource
  families described under API Resource Requirements. Existing role and ownership rules MUST
  apply to every operation; choosing a resource path MUST NOT grant access to the internal API.
- **FR-039**: Every resource read MUST be available using GET without a request body or a request
  `Content-Type` header. IDs, symbols, filters, and pagination inputs for reads MUST use path
  or query parameters. A client MUST NOT need to send an empty JSON object to perform a read.
- **FR-040**: Every JSON response, including middleware errors, liveness, empty collections,
  deletions, and unexpected failures, MUST use the JSON Response Envelope below. Responses
  MUST declare `Content-Type: application/json`; numeric codes MUST be JSON integers, not strings.
- **FR-041**: Unexpected failures MUST return HTTP 500, `success=false`, `responseCode=999999`,
  `errorMessage="An unexpected error occurred."`, and `data.reasonCode=INTERNAL_ERROR`.
  Responses MUST NOT expose exception text, stack traces, implementation/storage details,
  credentials, or private payloads; correlated diagnostics remain internal.
- **FR-042**: All API date/time values MUST use `dd-MMM-yyyy HH:mm:ss.SSS` with English month
  abbreviations, a four-digit year, 24-hour clock, and exactly three millisecond digits. UTC is
  the documented timezone for this offset-free representation; timestamps MUST NOT vary with
  host locale or timezone. Display formatting MUST NOT determine matching priority.
- **FR-043**: Every operation MUST return its own entity-and-outcome numeric code from the
  Application Result Codes and Result Code Selection rules. HTTP status reflects whether request
  handling/execution completed; `success` reflects the business outcome independently of both
  HTTP status and an order's stored trading status. HTTP 200 MUST permit `success=false`.
  Idempotent replay MUST retain the original HTTP status, numeric code, and business payload.
- **FR-044**: Every application-generated HTTP response MUST carry `X-Correlation-ID` without
  adding an extra top-level envelope field. Creation MUST return HTTP 201; successful reads,
  amendments, updates, deletions, cancellations, and liveness MUST return HTTP 200 with an
  envelope. Handled business rejections also MUST return HTTP 200 with `success=false`.
  Every response MUST carry `Cache-Control: no-store`; entity-not-found or rejection envelopes
  MUST NOT be cached as successful GET results. A successful deletion MUST NOT use HTTP 204.

### API Resource Requirements

Paths are relative to the applicable internal or external API base address. Administrators use
internal reference-data operations. Participants use only the external operations permitted
below, including an external read-only product catalog; they cannot invoke the internal catalog
or administration API. The internal base path is `/internal`; the external API uses the root
base with no additional prefix. Planning defines the host addresses and precise ID/filter selectors.

| Resource | Required capabilities and access |
| --- | --- |
| `/product/equity` | Internal administrator create, read/list, update, delete; external participant read/list by ID or symbol only. |
| `/participant/broker` | Internal administrator create, read/list, update, delete; no participant access. |
| `/order` | External Broker submit, read owned reports, amend, cancel, and read owned trade information. |
| `/orderBook/{productId}` | External participant read of aggregate depth and calculated metrics for that product. |
| `/status` | Public GET liveness returning the running indication. |

Trade-by-ID and trade-list reads required by FR-025 remain available within the `/order` resource
family; planning defines their selectors or subresources. This enhancement introduces no separate
top-level trade resource. All read selectors are path/query inputs, never request-body inputs.
Operations carrying a JSON request body must identify it as JSON; an unsupported or missing media
type for such a body returns HTTP 415 with reason `UNSUPPORTED_MEDIA_TYPE`. This requirement
MUST NOT be applied to bodyless GET requests. Reads do not require an `Accept` header either.

### JSON Response Envelope

Only the following top-level fields form the envelope. Relevant entity results, collection data,
trade details, and failure details belong inside `data`.

| Field | Required type and meaning |
| --- | --- |
| `success` | Required boolean: true for business success; false for every business rejection or request/execution failure, including HTTP 200 rejections. |
| `responseCode` | Required integer from the numeric code catalog and selection rules. |
| `infoMessage` | Required string: safe description of success, or empty string on failure. |
| `warningMessage` | Required string: safe warning when applicable, otherwise empty string. |
| `errorMessage` | Required string: empty on success; nonempty safe error description on failure. |
| `data` | Present when relevant: entity result, list, running indication, or failure object; every failure contains `data.reasonCode`. |

Messages are for people; clients inspect `success`, `responseCode`, and `data.reasonCode` even
when HTTP status is 200. HTTP status alone does not establish business success. A response with a
partial or dropped market remainder is still a
successful order acceptance; `warningMessage` identifies that unexecuted remainder. Empty lists
use an empty collection in `data`. Unused message fields are empty strings, not omitted or null.
A failure's `data` MUST NOT include partial business results or private records.
Decimal-valued fields within `data` use strings such as `"100.01"` and `"0.01"`; whole-share
quantities, order versions, and `responseCode` remain integers. Decimal fields that do not apply
remain absent as specified for market orders and empty or one-sided books.

### Application Result Codes

The supplied meanings and values are preserved. Code 0 is the user-confirmed success convention
for operations not assigned a success code in the supplied catalog.

| Area | Code | Meaning |
| --- | --- | --- |
| Product | 1001 | Added |
| Product | 1002 | Rejected |
| Product | 1003 | Found |
| Product | 1004 | Not found |
| Product | 1005 | Updated |
| Product | 1006 | Deleted |
| Participant | 1007 | Added |
| Participant | 1008 | Rejected |
| Participant | 1009 | Found |
| Participant | 1010 | Not found |
| Participant | 1011 | Updated |
| Participant | 1012 | Deleted |
| Order | 1013 | Accepted |
| Order | 1014 | Rejected |
| Order | 1015 | Found |
| Order | 1016 | Not found |
| Order | 1017 | Cancelled |
| Order | 1018 | Filled |
| Order | 1019 | Updated |
| Order book | 1020 | Found |
| General | 9000 | Unsupported entity |
| General | 999999 | General error |
| General | 0 | Success for status and separate trade reads |

### Result Code Selection

- Successful product create, read/list, update, and delete MUST use 1001, 1003, 1005, and 1006;
  their participant equivalents MUST use 1007, 1009, 1011, and 1012. An empty list is a successful
  found result; a missing individual target, including on update/delete, uses 1004 or 1010.
- A successful order submission MUST use 1018 if its resulting status is `FILLED`, otherwise
  1013, including `NOT_FILLED` and `PARTIALLY_FILLED` market orders. A successful amendment
  MUST use 1018 if it fills the order, otherwise 1019. Successful cancellation uses 1017.
- Reading an existing order report MUST use 1015 regardless of that order's status. A missing
  or inaccessible target order on read, amend, or cancel MUST use 1016. A new cancellation
  attempted on an already filled order MUST return HTTP 200, `success=false`, and 1018 with
  reason `ORDER_ALREADY_FILLED`; it MUST NOT imply a new execution or successful cancellation.
- Successful book reads and metrics MUST use 1020, including an empty active product's book.
  Successful liveness and separate trade reads MUST use 0. A successful order report that
  includes trades still uses 1015; a separate trade read does not use an order-found code.
- Unsupported product or participant entity types MUST use 9000 with their specific
  `UNSUPPORTED_PRODUCT_TYPE` or `UNSUPPORTED_PARTICIPANT_TYPE` reason and HTTP 200 with `success=false`.
  Unsupported order side, order type, or longevity MUST use order-rejected code 1014, not 9000.
- Other business rejections on recognized product, participant, or order mutations MUST use
  1002, 1008, or 1014 respectively, with HTTP 200, `success=false`, and the specific reason.
  An order rejected because its referenced product/participant is absent MUST use 1014, not 1004/1010.
  A new cancellation of an already cancelled order therefore uses 1014, not successful code 1017.
- Authentication, authorization, parsing, unknown routes, unsupported methods/media, request
  limits, throttling, capacity, storage availability, and unexpected failures MUST use 999999
  with the precise reason and the HTTP status in the Failure Outcomes table. Unmapped business
  failures, including an unknown product's book or missing separate trade, MUST also use 999999
  but return HTTP 200 with `success=false`; they MUST NOT invent a neighboring entity's numeric
  code. The reason distinguishes these from infrastructure failures carrying the same code.
- These general failure rules take precedence over entity outcomes for failures before business
  processing. Unsupported entity type follows next, then the applicable target-not-found,
  filled-order cancellation, or entity-specific business outcome. Idempotency replay returns
  the original result; conflicting identity reuse on a recognized mutation uses its entity's
  rejected code and `IDEMPOTENCY_CONFLICT`. Missing command identity likewise uses the entity's
  rejected code and `IDEMPOTENCY_REQUIRED`.

### Order Status Rules

A report uses the current effective total, cumulative traded quantity, and current booked quantity.
Removed quantity is unexecuted quantity dropped, cancelled, or expired. Reducing an order's total
through amendment does not rewrite its original quantity or past trades.

| Status | Required meaning |
| --- | --- |
| `BOOKED` | Limit order with zero traded and positive booked quantity. |
| `PARTIALLY_BOOKED_FILLED` | Limit order with positive traded and booked quantities. |
| `FILLED` | Entire current effective total traded; zero booked and removed quantities. |
| `PARTIALLY_FILLED` | Market order partly traded; its positive unfilled remainder dropped. |
| `NOT_FILLED` | Market order with zero traded; its entire quantity dropped. |
| `CANCELLED` | Limit remainder cancelled or day-expired with zero traded. |
| `PARTIALLY_CANCELLED` | Limit remainder cancelled or day-expired after some trading. |
| `REJECTED` | Reserved only; invalid orders are not stored or reportable. |

### Failure Outcomes

The following are required distinguishable failure reasons in `data.reasonCode`. The numeric
`responseCode` is selected separately by the preceding rules; symbolic reasons MUST NOT be
serialized in its place. Planning may add reasons for additional validation cases without
reassigning the supplied numeric codes or the stable reasons below. Malformed input is rejected
without business effects; parseable requests are checked for authentication and role before
business rules.
Ownership gates private record visibility; remaining checks follow request validity, entity
availability, then current order state. A private record not accessible to the caller returns the
same not-found response as a nonexistent record. Replay of an already completed command returns
its original outcome after current authentication and authorization checks, rather than attempting
the state change again. For a new amendment to an accessible existing order, version comparison
follows input validation and precedes terminal-state checks; an outdated version returns
`ORDER_VERSION_CONFLICT`, while a matching version on a terminal order returns
`ORDER_NOT_AMENDABLE`.

| Condition | HTTP status | `data.reasonCode` |
| --- | --- | --- |
| Missing or invalid caller identity | 401 | `UNAUTHENTICATED` |
| Participant invokes internal API; unauthorized participant identity supplied | 403 | `FORBIDDEN` |
| Malformed JSON/request syntax | 400 | `MALFORMED_REQUEST` |
| Non-JSON or unidentified request-body media type | 415 | `UNSUPPORTED_MEDIA_TYPE` |
| Unknown route / unsupported HTTP method | 404 / 405 | `ROUTE_NOT_FOUND` / `METHOD_NOT_ALLOWED` |
| Product or active participant absent | 200 | `PRODUCT_NOT_FOUND` / `PARTICIPANT_NOT_FOUND` |
| Private order or trade absent/inaccessible | 200 | `ORDER_NOT_FOUND` / `TRADE_NOT_FOUND` |
| Unsupported product / participant type | 200 | `UNSUPPORTED_PRODUCT_TYPE` / `UNSUPPORTED_PARTICIPANT_TYPE` |
| Missing / unsupported side | 200 | `MISSING_ORDER_SIDE` / `UNSUPPORTED_ORDER_SIDE` |
| Missing / unsupported order type | 200 | `MISSING_ORDER_TYPE` / `UNSUPPORTED_ORDER_TYPE` |
| Non-positive, fractional, malformed, or out-of-range quantity | 200 | `INVALID_QUANTITY` |
| Missing limit price / invalid or zero-rounded limit price | 200 | `MISSING_LIMIT_PRICE` / `INVALID_LIMIT_PRICE` |
| Price supplied on market order / unsupported longevity | 200 | `MARKET_PRICE_NOT_ALLOWED` / `UNSUPPORTED_LONGEVITY` |
| Invalid product/participant attributes, tick, or missing entity type | 200 | `INVALID_PRODUCT` / `INVALID_PARTICIPANT` |
| Duplicate normalized symbol | 200 | `PRODUCT_SYMBOL_EXISTS` |
| Entity deletion or product symbol/tick change conflicts with open orders | 200 | `PRODUCT_IN_USE` / `PARTICIPANT_IN_USE` |
| Cancel filled / previously cancelled or expired order | 200 | `ORDER_ALREADY_FILLED` / `ORDER_ALREADY_CANCELLED` |
| Cancel a market order with no booked remainder | 200 | `ORDER_NOT_ON_BOOK` |
| Amend terminal order / forbidden field / invalid amended total | 200 | `ORDER_NOT_AMENDABLE` / `INVALID_AMENDMENT` / `INVALID_QUANTITY` |
| Amendment missing expected version / invalid expected version | 200 | `MISSING_ORDER_VERSION` / `INVALID_ORDER_VERSION` |
| Amendment expected version differs from current order version | 200 | `ORDER_VERSION_CONFLICT` |
| Missing command identity / conflicting reuse | 200 | `IDEMPOTENCY_REQUIRED` / `IDEMPOTENCY_CONFLICT` |
| Payload too large / excessive collection limit | 413 / 200 | `REQUEST_TOO_LARGE` / `REQUEST_LIMIT_EXCEEDED` |
| Caller throttled / service at capacity | 429 / 503 | `RATE_LIMITED` / `SERVICE_OVERLOADED` |
| Storage temporarily unavailable / unexpected failure | 503 / 500 | `STORAGE_UNAVAILABLE` / `INTERNAL_ERROR` |

All HTTP 200 failures in this table have `success=false`. Syntactically valid JSON with invalid
business fields is a business rejection even when a framework binder would normally return 4xx.
Unknown route is a request failure (404); missing entity on a recognized route is a business
outcome (200). Current authentication and authorization checks still apply before all decisions.

### Key Entities *(include if feature involves data)*

- **User identity**: Authenticated Administrator or Participant; a Participant is bound to one
  Broker. Role membership and ownership determine permitted operations.
- **Product**: Immutable ID, unique normalized symbol, description, Equity type, tick size,
  and active/deleted state; owns one logical order book and is referenced by orders.
- **Participant**: Immutable ID, name, Broker type, and active/deleted state; owns orders.
- **Order**: Identity and immutable entry time; product and participant; side, type, day longevity,
  current normalized price and priority; original and effective quantities, traded/booked/removed
  quantities, status, removal reason, current version, and ordered trade references. Amendments
  must identify the version they were based on; cancellations do not require a version.
- **Order book / price level**: One product's open bids and offers; quantities aggregate by price,
  while matching retains individual order priority. Snapshots and metrics are derived views.
- **Trade**: Immutable execution ID and time, buy and sell order references, price, and quantity.
- **Command outcome**: Caller-scoped identity, operation and payload identity, original result,
  and retention scope, enabling retry without repeated effects.
- **Audit event**: Attributable, ordered record of a mutation outcome and its related entities;
  committed with the financial effects it describes.
- **Trading day**: UTC calendar-day interval governing day-order expiry.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Administrators can complete every product and participant lifecycle in Story 1;
  100% of attempted participant access to internal operations is refused without changes.
- **SC-002**: All deterministic matching examples and at least 10,000 generated command sequences
  produce the expected price-time executions; zero orders overfill, cross products, or execute
  at a price other than the eligible resting price.
- **SC-003**: All eight supplied status definitions are preserved; the seven stored statuses are
  demonstrated, and 100% of rejected orders have no stored report or business effects.
- **SC-004**: For every stored order in acceptance runs, traded + booked + removed equals effective
  total, referenced trades reconcile exactly to traded quantity, and book volumes equal the sum
  of eligible order remainders. All required metrics agree with the same snapshot state.
- **SC-005**: In at least 10,000 conflicting concurrent-command groups, every observed result
  agrees with a valid acceptance order; zero duplicate effects, partial states, or ownership
  breaches occur. A group includes competing submissions, amend/cancel races, or retries.
  All stale-version amendments MUST fail without business effects, and every affected order's
  version MUST advance exactly once per committed change, with no advancement on no-ops or replay.
- **SC-006**: All enumerated failures return the specified HTTP status, numeric `responseCode`,
  symbolic `data.reasonCode`, and correlation identifier in the common response envelope. All successful logical mutations have an attributable audit record;
  retries produce zero extra trades, orders, or mutation audit records. HTTP 200 responses with
  `success=false` are application failures and MUST NOT count as successfully completed work.
- **SC-007**: Under the baseline below, participants complete at least 1,000 valid business requests
  per second for 15 continuous minutes with 100 concurrent clients; 95% finish within 100 ms and
  99% within 250 ms, measured from submission through receipt of the complete outcome. Unexpected
  failures remain below 0.1%, and peak service resident memory remains at or below 4 GiB. Expected rejections
  are reported separately and do not satisfy the completed-business-request target.
- **SC-008**: During a separate 60-second burst offering 2,000 requests per second with up to
  1,000 concurrent clients, every capacity rejection has an explicit overload result, completed
  responses have p99 at most 2 seconds, peak service resident memory stays within 4 GiB, and invariants remain intact.
  Baseline response and completion targets resume within 30 seconds after offered load returns
  to baseline. Missing responses and intentional rejections are reported separately.
- **SC-009**: Every prior-day open remainder is ineligible at the UTC boundary and absent from
  all new-day reads; 100% of prior trades and permitted historical reports remain unchanged.
- **SC-010**: The published behavioral acceptance suite is independent of storage mode. Every
  supported storage mode passes it with the same business results; only the declared restart
  retention and durability guarantees may differ.
- **SC-011**: Every published read succeeds for valid accessible data without a body, `Content-Type`,
  or `Accept` header. All JSON responses have the five mandatory envelope fields and relevant
  `data`; 100% of returned date/time values match the exact English/UTC millisecond format.
  All returned prices, ticks, spreads, and midpoints MUST be decimal strings preserving exact
  values, and all share quantities MUST remain integers. Numeric values supplied where decimal
  strings are required MUST be rejected without business effects in every acceptance case.
- **SC-012**: Acceptance examples demonstrate every numeric code in the supplied catalog and
  confirmed code 0, including distinct product/participant/order failures, filled-order reads
  versus fills, and book-not-found behavior. All injected unexpected failures use HTTP 500,
  code 999999, and a safe general message; zero responses expose internal exception details.

**Baseline workload**: Synthetic data; 10 Equity products, 100 Brokers, and 10,000 initially open
limit orders spread across 100 populated price levels per product. Requests comprise 40% order
submissions (half limit and half market), 20% amendments, 10% cancellations, 20% owned reports/trade
reads, and 10% book/metric reads; 50% target one hot product and the rest spread across the others.
The workload supplies eligible orders for amendments/cancellations and liquidity for successful
market submissions, with prices selected to exercise both crossing and resting outcomes. Use a
service allocation of 4 CPU cores and 8 GiB memory, a separate load generator, and the in-memory
mode for the initial acceptance run. Record dataset generation, exact request schedule, resource
allocation, storage mode, and request and trade counts so the run can be reproduced. Persistent
mode results must be measured and reported separately before that mode is advertised as supported.

## Assumptions

- **A-001 — Simulation boundary**: Synthetic securities and participants only. Funds, positions,
  credit checks, custody, settlement, fees, real venues, streaming feeds, auctions, and real-asset
  handling are outside scope. All eligible opposite orders, including a Broker's own orders,
  follow the stated matching rules; self-trade prevention is not part of this simulation.
- **A-002 — Identity**: Administrator provisioning and credential issuance are deployment
  dependencies, not participant self-registration features. An administrator creates each Broker
  and associates an authenticated participant identity before it may trade. The authentication
  mechanism is chosen during planning; credentials never derive merely from a supplied Broker ID.
- **A-003 — Visibility**: The word client means an authenticated active participant except for
  public liveness. Participants see their own orders/trades, active products, and aggregate books.
  Administrators manage reference data; this feature does not grant them private trading-report
  access or order-submission rights solely by administrator role.
- **A-004 — Day boundary**: Trading is continuous, every calendar day, with no exchange holiday
  calendar. A day ends at 00:00 UTC; a command accepted at that boundary belongs to the new day.
  Expiry uses existing cancellation statuses with `DAY_EXPIRED`; it does not introduce `EXPIRED`.
- **A-005 — Amendment meaning**: Quantity amendment specifies the new total including executions,
  not a delta or a new remaining quantity. A valid total is strictly greater than traded quantity;
  cancellation removes the remainder. Original submitted quantity and entry time remain immutable.
  Current priority can change independently of original entry time as defined in FR-018.
  Amendments require the current order version; cancellation removes the latest remainder without
  requiring a version, as confirmed during clarification.
- **A-006 — Reference data changes**: Deletion is logical from the user's perspective and retains
  referenced history. Deleted symbols remain reserved during the retained data lifetime so old
  product history cannot become confused with a different product. Types and IDs cannot change;
  product symbols/ticks cannot change while orders are open. Broker names may be duplicated.
- **A-007 — Numbers and times**: Quantity means whole shares; fractional shares are unsupported.
  Prices are exact finite decimals and ticks are positive exact decimals. Prices, ticks, spreads,
  and midpoints use decimal strings in JSON; quantities remain integers, as confirmed during
  clarification. Times are service-assigned
  UTC instants formatted as `dd-MMM-yyyy HH:mm:ss.SSS` with English months; a separate ordering
  resolves timestamp ties. Values in documented ranges are accepted, and range/precision limits
  must be fixed during planning before implementation.
- **A-008 — Retention and retries**: The initial in-memory service has one active application
  instance and retains orders, trades, deletions, and audit records for its running lifetime.
  Command identities and completed outcomes are retained for at least 24 hours from completion
  while the process survives; commands still in progress do not expire. On restart all of this
  state may be lost, which clients must be told explicitly. Persistent mode must retain its
  declared acknowledged state across restart. Expired command identities may be treated as new.
- **A-009 — Initial capacity target**: SC-007/SC-008 and their workload are explicit initial
  acceptance defaults because no numeric performance target was supplied. Planning must confirm
  a reproducible environment and finite request/page/wait limits before implementation; a proposed
  change to the business performance target requires an explicit specification amendment.
- **A-010 — Dependencies**: Planning must supply administrator bootstrap, participant identity
  mapping, exact numeric and request limits, and reproducible workload fixtures. No live market,
  payments provider, or external pricing source is required. Request limits and persistent-mode
  recovery guarantees are published as part of the later API and operational contracts.
- **A-011 — Envelope defaults**: The five mandatory envelope fields are always present, unused
  messages are empty strings, and `data` carries relevant results or a failure reason. The
  correlation identifier is in `X-Correlation-ID` to preserve the requested top-level shape.
- **A-012 — Confirmed catalog gaps**: The user selected 0 for successful status checks and
  separate trade reads, and 999999 plus a specific reason for otherwise unmapped failures.
  No successful status check is mislabeled as a found product, order, or book.
- **A-013 — Outcome precedence**: Immediate complete fills on submission/amendment use 1018;
  fetching a filled order uses 1015. Cancelling an already filled order fails with 1018 plus
  `ORDER_ALREADY_FILLED`; other invalid cancellations use 1014 unless the order is absent (1016).
  HTTP status, success flag, and reason disambiguate shared outcome codes. These are documented
  defaults where the supplied catalog names an outcome without specifying operation precedence.
- **A-014 — Resource boundaries**: Internal resources use the `/internal` base path; external
  resources are rooted directly at the service base URL, as confirmed on 2026-11-07. These paths
  identify resource families under their appropriate internal/external base addresses; they do not collapse the role boundary. Trade reads remain
  in the `/order` family, and planning supplies exact ID, symbol, trade, and pagination selectors.
  UTC and English month names make the requested offset-free time format deterministic.
- **A-015 — HTTP business-result policy**: As directed on 2026-11-07, handled application
  rejections return HTTP 200 with `success=false` and existing numeric/symbolic codes. Request,
  security, admission, and execution/system failures retain their documented 4xx/5xx statuses.
  Accepted creation remains HTTP 201. Clients and monitoring inspect the envelope, not status alone.
