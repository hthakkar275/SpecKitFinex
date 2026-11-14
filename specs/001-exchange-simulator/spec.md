# Feature Specification: Electronic Financial Exchange Simulator

**Feature Branch**: None created; branch creation is optional and no branch hook is registered.

**Feature Directory**: `specs/001-exchange-simulator`

**Created**: 2026-11-06

**Last Updated**: 2026-11-08

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

Each numbered parent below is a topic group, not a compound requirement. IDs such as
`FR-018.8` identify individual requirements and are the units for implementation, verification,
and compliance reporting. An existing reference to `FR-018` refers to that group as a whole.

Each child states one obligation or an explicitly retained permission. Its condition can rely on
shared definitions and earlier validation steps; it does not require an independent implementation.
`MUST` and `MUST NOT` are mandatory. `MAY` preserves an existing optional permission.

Rejection statements use the existing [Failure Outcomes](#failure-outcomes) and
[Result Code Selection](#result-code-selection) profiles. These shared profiles specify the HTTP
status, envelope, numeric code, and reason, so a rejection rule does not repeat that entire contract.
Related rules can share a scenario or test fixture while receiving separate pass/fail results.
The [requirements traceability matrix](requirements-traceability.md) maps the unchanged user
stories to the atomic IDs and records the original-to-new requirement mapping.

#### FR-001 — API boundaries

This feature is a backend service with no user interface. Liveness is defined in FR-028; resource
paths are defined in FR-038.

- **FR-001.1**: The system MUST expose an internal HTTP REST interface for administration.
- **FR-001.2**: The system MUST expose an external HTTP REST interface for participants.

#### FR-002 — Authentication and roles

- **FR-002.1**: The system MUST authenticate the caller of every operation except liveness.
- **FR-002.2**: The system MUST enforce the caller role required by the requested operation.
- **FR-002.3**: The system MUST deny participants access to every internal API operation.
- **FR-002.4**: The system MUST NOT grant Broker order-entry permission solely because an identity
  has Administrator authority.

#### FR-003 — Participant identity and ownership

- **FR-003.1**: The system MUST bind each participant identity to exactly one active Broker.
- **FR-003.2**: The system MUST derive the Broker identity for an order submission from the
  authenticated participant identity.
- **FR-003.3**: The system MUST authorize an amendment against the authenticated participant's
  Broker identity.
- **FR-003.4**: The system MUST authorize a cancellation against the authenticated participant's
  Broker identity.
- **FR-003.5**: The system MUST authorize a private report against the authenticated participant's
  Broker identity.
- **FR-003.6**: The system MUST NOT use a client-supplied participant ID as proof of identity.
- **FR-003.7**: The system MUST reject an attempt by a participant to act for a different Broker
  with `FORBIDDEN`.

#### FR-004 — Private and shared information

- **FR-004.1**: The system MUST restrict a participant's order-report access to orders owned by its
  Broker.
- **FR-004.2**: The system MUST restrict a participant's trade access to trades involving its
  Broker.
- **FR-004.3**: The system MUST allow every active participant to look up active products through
  the external API.
- **FR-004.4**: The system MUST allow every active participant to read aggregate order books through
  the external API.
- **FR-004.5**: The system MUST restrict participant administration to the internal API.
- **FR-004.6**: The system MUST restrict participant lists to the internal API.

#### FR-005 — Product management and records

Symbol normalization, uniqueness, and invalid-product handling are defined in FR-006.

- **FR-005.1**: The system MUST allow an Administrator to create a product.
- **FR-005.2**: The system MUST allow an Administrator to retrieve a product by ID.
- **FR-005.3**: The system MUST allow an Administrator to retrieve a product by symbol.
- **FR-005.4**: The system MUST allow an Administrator to list products.
- **FR-005.5**: The system MUST allow an Administrator to update a product.
- **FR-005.6**: The system MUST allow an Administrator to delete a product.
- **FR-005.7**: The system MUST generate the ID of each created product.
- **FR-005.8**: Each product ID MUST be unique.
- **FR-005.9**: Each product ID MUST remain immutable.
- **FR-005.10**: Each stored product MUST have a symbol.
- **FR-005.11**: Each stored product MUST have a nonempty description.
- **FR-005.12**: Each stored product MUST have type Equity.
- **FR-005.13**: Each stored product MUST have a positive tick size.
- **FR-005.14**: The system MUST default an omitted product tick size to 0.01.

#### FR-006 — Product validation and symbol normalization

- **FR-006.1**: The system MUST require an explicit product type on product creation.
- **FR-006.2**: The system MUST reject a product type other than Equity with
  `UNSUPPORTED_PRODUCT_TYPE`.
- **FR-006.3**: The system MUST trim surrounding whitespace from a product symbol before validating
  it.
- **FR-006.4**: The system MUST normalize a product symbol to uppercase before validating or looking
  up that symbol.
- **FR-006.5**: Normalized product symbols MUST be unique.
- **FR-006.6**: The system MUST reject a blank normalized product symbol with `INVALID_PRODUCT`.
- **FR-006.7**: The system MUST reject a duplicate normalized product symbol with
  `PRODUCT_SYMBOL_EXISTS`.
- **FR-006.8**: The system MUST reject an invalid product tick size with `INVALID_PRODUCT`.
- **FR-006.9**: A rejected product change MUST leave the existing product state unchanged.

#### FR-007 — Broker management and records

- **FR-007.1**: The system MUST allow an Administrator to create a participant.
- **FR-007.2**: The system MUST allow an Administrator to retrieve a participant by ID.
- **FR-007.3**: The system MUST allow an Administrator to list participants.
- **FR-007.4**: The system MUST allow an Administrator to update a participant.
- **FR-007.5**: The system MUST allow an Administrator to delete a participant.
- **FR-007.6**: The system MUST generate the ID of each created participant.
- **FR-007.7**: Each participant ID MUST be unique.
- **FR-007.8**: Each participant ID MUST remain immutable.
- **FR-007.9**: Each stored participant MUST have a nonempty name.
- **FR-007.10**: Each stored participant MUST have type Broker.
- **FR-007.11**: The system MUST require an explicit participant type on participant creation.
- **FR-007.12**: The system MUST reject a participant type other than Broker with
  `UNSUPPORTED_PARTICIPANT_TYPE`.
- **FR-007.13**: The system MUST permit different participants to have the same name.

#### FR-008 — Deletion and reference-data changes

Booked quantity is the unfilled quantity still resting on the book. The final two statements
preserve the existing permission, rather than introducing a new mandatory update capability.

- **FR-008.1**: The system MUST reject product deletion while any order for that product has booked
  quantity.
- **FR-008.2**: The system MUST reject participant deletion while any order owned by that
  participant has booked quantity.
- **FR-008.3**: The system MUST exclude a deleted product from active lookup.
- **FR-008.4**: The system MUST exclude a deleted participant from active lookup.
- **FR-008.5**: The system MUST reject a new order referencing a deleted product.
- **FR-008.6**: The system MUST reject a new order referencing a deleted participant.
- **FR-008.7**: The system MUST retain historical references to deleted products.
- **FR-008.8**: The system MUST retain historical references to deleted participants.
- **FR-008.9**: The system MUST revoke API access for identities bound to a deleted participant.
- **FR-008.10**: The system MUST reject a product-symbol change while that product has booked
  orders.
- **FR-008.11**: The system MUST reject a product-tick change while that product has booked orders.
- **FR-008.12**: Product-description updates MAY proceed while orders are booked, subject to the
  normal update rules.
- **FR-008.13**: Participant-name updates MAY proceed while orders are booked, subject to the normal
  update rules.

#### FR-009 — Order submission and assigned attributes

- **FR-009.1**: The system MUST allow an active Broker to submit an order for a product.
- **FR-009.2**: Each accepted order MUST identify its product.
- **FR-009.3**: Each accepted order MUST identify its participant.
- **FR-009.4**: Each accepted order MUST have side buy or sell.
- **FR-009.5**: Each accepted order MUST have type limit or market.
- **FR-009.6**: Each accepted order MUST have a positive whole-share quantity.
- **FR-009.7**: Each accepted limit order MUST have a positive price.
- **FR-009.8**: Each accepted order MUST have day longevity.
- **FR-009.9**: The system MUST generate an ID for each accepted order.
- **FR-009.10**: Each order ID MUST be unique.
- **FR-009.11**: Each order ID MUST remain immutable.
- **FR-009.12**: The system MUST assign the original entry time of each accepted order.
- **FR-009.13**: An order's original entry time MUST remain immutable.
- **FR-009.14**: The system MUST assign a deterministic acceptance priority to each accepted order.

#### FR-010 — Order-entry rejection rules

Failure Outcomes supplies the specific reason for each rejection. REJECTED is a reserved status, not
a stored report status; see FR-024.

- **FR-010.1**: The system MUST reject an order whose product does not exist.
- **FR-010.2**: The system MUST reject an order whose product is inactive.
- **FR-010.3**: The system MUST reject an order whose participant does not exist.
- **FR-010.4**: The system MUST reject an order whose participant is inactive.
- **FR-010.5**: The system MUST reject an order with a missing side.
- **FR-010.6**: The system MUST reject an order with an unsupported side.
- **FR-010.7**: The system MUST reject an order with a missing type.
- **FR-010.8**: The system MUST reject an order with an unsupported type.
- **FR-010.9**: The system MUST reject an order whose quantity is not positive.
- **FR-010.10**: The system MUST reject an order whose quantity is not a whole number.
- **FR-010.11**: The system MUST reject a malformed order quantity.
- **FR-010.12**: The system MUST reject an out-of-range order quantity.
- **FR-010.13**: The system MUST reject a limit order with no price.
- **FR-010.14**: The system MUST reject a limit order whose price is not positive.
- **FR-010.15**: The system MUST NOT store an order rejected at entry.
- **FR-010.16**: The system MUST NOT create a trade for an order rejected at entry.
- **FR-010.17**: The system MUST NOT change an order book for an order rejected at entry.

#### FR-011 — Exact values and decimal encoding

Price use includes matching, queue placement, and reporting. JSON encoding applies wherever the
value appears, including reports, trades, and price levels. Response-code integer encoding is
defined in FR-040.9; all rejection responses follow FR-029.

- **FR-011.1**: The system MUST normalize each accepted limit price to the nearest multiple of the
  product tick using half-up rounding before using that price.
- **FR-011.2**: The system MUST reject a limit price whose normalized value is not positive.
- **FR-011.3**: The system MUST use exact arithmetic for financial values.
- **FR-011.4**: The system MUST publish the permitted range of each financial numeric input.
- **FR-011.5**: The system MUST publish the maximum accepted precision of each financial numeric
  input.
- **FR-011.6**: The system MUST reject a financial input outside its published range.
- **FR-011.7**: The system MUST reject a financial input exceeding its published precision.
- **FR-011.8**: Every JSON price value MUST be encoded as a decimal string.
- **FR-011.9**: Every JSON tick-size value MUST be encoded as a decimal string.
- **FR-011.10**: Every JSON spread value MUST be encoded as a decimal string.
- **FR-011.11**: Every JSON midpoint value MUST be encoded as a decimal string.
- **FR-011.12**: A decimal string MUST express its exact base-10 value.
- **FR-011.13**: A fractional decimal string MUST use a dot as its decimal separator.
- **FR-011.14**: A decimal string MUST NOT contain locale-specific digit grouping.
- **FR-011.15**: A decimal string MUST NOT use exponent notation.
- **FR-011.16**: Every JSON share quantity MUST be encoded as an integer.
- **FR-011.17**: Every JSON order-version value MUST be encoded as an integer.
- **FR-011.18**: The system MUST reject a JSON numeric price value with `INVALID_LIMIT_PRICE`.
- **FR-011.19**: The system MUST reject a malformed decimal price string with `INVALID_LIMIT_PRICE`.
- **FR-011.20**: The system MUST reject an invalid tick representation with `INVALID_PRODUCT`.

#### FR-012 — Market-order price and longevity

- **FR-012.1**: A market order MUST have no effective price.
- **FR-012.2**: A market-order report MUST omit the price.
- **FR-012.3**: The system MUST reject a price supplied with a market order with
  `MARKET_PRICE_NOT_ALLOWED`.
- **FR-012.4**: The system MUST reject unsupported order longevity with `UNSUPPORTED_LONGEVITY`.

#### FR-013 — Order-book lifecycle

- **FR-013.1**: The system MUST maintain one logical order book per product.
- **FR-013.2**: The system MUST create a product's order book only when needed.
- **FR-013.3**: The system MUST return an empty book view for an active product that has not yet
  needed a book.

#### FR-014 — Matching eligibility and priority

- **FR-014.1**: The system MUST match an incoming order only against orders for the same product.
- **FR-014.2**: The system MUST match an incoming order only against opposite-side orders.
- **FR-014.3**: The system MUST match an incoming order only against resting orders.
- **FR-014.4**: The system MUST select the lowest eligible offer price first for an incoming buy.
- **FR-014.5**: The system MUST select the highest eligible bid price first for an incoming sell.
- **FR-014.6**: The system MUST select the earliest current priority first among eligible resting
  orders at the same price.

#### FR-015 — Limit and market price eligibility

FR-014 supplies best-price selection; FR-017 supplies matching completion and remainder handling.

- **FR-015.1**: The system MUST match a limit buy only at prices at or below its normalized limit.
- **FR-015.2**: The system MUST match a limit sell only at prices at or above its normalized limit.
- **FR-015.3**: The system MUST apply no limit-price constraint when matching a market order.

#### FR-016 — Individual executions

Publication of both affected orders and the trade as one complete transaction is governed by
FR-032.1.

- **FR-016.1**: Each match MUST execute at the resting order's price.
- **FR-016.2**: Each match MUST execute the smaller of the incoming and resting orders' remaining
  quantities.
- **FR-016.3**: Each match MUST create exactly one trade.
- **FR-016.4**: A committed trade MUST remain immutable.
- **FR-016.5**: A match MUST NOT execute more than an order's effective total minus its already
  traded quantity.

#### FR-017 — Matching completion and remaining quantities

- **FR-017.1**: The system MUST finish every immediately eligible match before returning the command
  outcome.
- **FR-017.2**: The system MUST place an unfilled limit-order remainder on the book.
- **FR-017.3**: The system MUST drop an unfilled market-order remainder.
- **FR-017.4**: A fully filled order MUST immediately cease being eligible to trade.
- **FR-017.5**: A cancelled order MUST immediately cease being eligible to trade.
- **FR-017.6**: An expired order MUST immediately cease being eligible to trade.

#### FR-018 — Order amendments

An eligible target is a limit order with booked quantity after the common checks. Eligibility checks
run after version comparison. Quantity means the new overall total, including executions. Order ID
and original entry time remain immutable under FR-009.11 and FR-009.13; expectedVersion uses the
integer encoding in FR-011.17. Failure Outcomes defines the complete rejection response.

- **FR-018.1**: The system MUST allow the owner to amend the price of an eligible limit order.
- **FR-018.2**: The system MUST allow the owner to amend the total quantity of an eligible limit
  order.
- **FR-018.3**: The system MUST allow the owner to amend price and total quantity in the same
  command.
- **FR-018.4**: The system MUST reject amendment of an order that is not a limit order.
- **FR-018.5**: The system MUST reject amendment of an order that has no booked quantity.
- **FR-018.6**: The system MUST reject an amended total that is not a positive integer.
- **FR-018.7**: The system MUST reject an amended total that is less than or equal to the already
  traded quantity.
- **FR-018.8**: The system MUST reset time priority when an amendment changes the normalized price.
- **FR-018.9**: The system MUST reset time priority when an amendment increases total quantity.
- **FR-018.10**: The system MUST retain time priority for a quantity-only reduction.
- **FR-018.11**: The system MUST retain time priority for an effective no-op amendment.
- **FR-018.12**: The system MUST match a marketable amendment before returning its outcome.
- **FR-018.13**: The system MUST treat a marketable amended order as the incoming order.
- **FR-018.14**: The system MUST require `expectedVersion` on every amendment.
- **FR-018.15**: The system MUST require a positive `expectedVersion`.
- **FR-018.16**: The system MUST compare the expected version only after access and input
  validation.
- **FR-018.17**: The system MUST compare the expected version before evaluating whether the order
  remains amendable.
- **FR-018.18**: The system MUST protect version comparison and the resulting state change as one
  atomic operation.
- **FR-018.19**: The system MUST reject an expected-version mismatch with `ORDER_VERSION_CONFLICT`.

#### FR-019 — Amendment accounting and rejected changes

Business state includes affected orders, books, quantities, prices, priority, and versions. Prior
executions remain immutable under FR-016.4. Report presentation is defined by FR-023.11.

- **FR-019.1**: An order's original quantity MUST remain equal to its submitted quantity.
- **FR-019.2**: An accepted quantity amendment MUST replace the previous effective total with the
  new total.
- **FR-019.3**: A rejected amendment MUST leave all pre-request business state unchanged.

#### FR-020 — Cancellation effects

- **FR-020.1**: The system MUST allow an order's owner to cancel its entire remaining booked
  quantity.
- **FR-020.2**: A successful cancellation MUST set the order's booked quantity to zero immediately.
- **FR-020.3**: A cancellation MUST preserve the order's traded quantity.
- **FR-020.4**: A participant cancellation MUST record removal reason `PARTICIPANT_CANCELLED`.
- **FR-020.5**: A cancellation with no traded quantity MUST select status `CANCELLED`.
- **FR-020.6**: A cancellation with positive traded quantity MUST select status
  `PARTIALLY_CANCELLED`.
- **FR-020.7**: The system MUST NOT require an expected version for cancellation.
- **FR-020.8**: The system MUST cancel the remaining quantity present when that cancellation takes
  its turn.
- **FR-020.9**: A dropped market-order remainder MUST record reason `MARKET_REMAINDER_DROPPED`.

#### FR-021 — Invalid cancellation and amendment states

The filled-order cancellation rule takes precedence for a FILLED market order. Version comparison
precedes terminal-state evaluation under FR-018.17; replay is governed by FR-030 and FR-032.10.

- **FR-021.1**: The system MUST reject cancellation of an unknown or inaccessible order with
  `ORDER_NOT_FOUND`.
- **FR-021.2**: The system MUST reject cancellation of a filled order with `ORDER_ALREADY_FILLED`.
- **FR-021.3**: The system MUST reject a new cancellation of an already-cancelled order with
  `ORDER_ALREADY_CANCELLED`.
- **FR-021.4**: The system MUST reject cancellation of a day-expired order with
  `ORDER_ALREADY_CANCELLED`.
- **FR-021.5**: The system MUST reject cancellation of a market order with no booked remainder with
  `ORDER_NOT_ON_BOOK`.
- **FR-021.6**: The system MUST reject amendment of a terminal order with `ORDER_NOT_AMENDABLE` when
  its expected version matches.

#### FR-022 — UTC day expiry

- **FR-022.1**: The system MUST expire a day order at the end of its UTC calendar day.
- **FR-022.2**: The system MUST remove prior-day open remainders before a new-day operation reads
  the book.
- **FR-022.3**: The system MUST remove prior-day open remainders before a new-day operation matches
  the book.
- **FR-022.4**: The system MUST remove the affected prior-day remainders as an atomic expiry
  operation.
- **FR-022.5**: An expired remainder MUST record removal reason `DAY_EXPIRED`.
- **FR-022.6**: Expiry with no traded quantity MUST select status `CANCELLED`.
- **FR-022.7**: Expiry with positive traded quantity MUST select status `PARTIALLY_CANCELLED`.
- **FR-022.8**: The system MUST retain terminal order reports after expiry.
- **FR-022.9**: The system MUST retain prior trades after expiry.

#### FR-023 — Order reports and quantity reconciliation

Existing bounded pagination and consistency rules still apply. Market-price omission is defined in
FR-012.2; assigning the amended effective total is defined in FR-019.2.

- **FR-023.1**: The system MUST allow a Broker to retrieve a report for an order it owns.
- **FR-023.2**: An order report MUST include the order ID.
- **FR-023.3**: An order report MUST include the product ID.
- **FR-023.4**: An order report MUST include the participant ID.
- **FR-023.5**: An order report MUST include the side.
- **FR-023.6**: An order report MUST include the order type.
- **FR-023.7**: An order report MUST include day longevity.
- **FR-023.8**: A limit-order report MUST include its normalized price.
- **FR-023.9**: An order report MUST include the original entry time.
- **FR-023.10**: An order report MUST include the original quantity.
- **FR-023.11**: An order report MUST include the current effective total quantity.
- **FR-023.12**: An order report MUST include the traded quantity.
- **FR-023.13**: An order report MUST include the booked quantity.
- **FR-023.14**: An order report MUST include the removed quantity.
- **FR-023.15**: An order report MUST include the status.
- **FR-023.16**: An order report MUST include the removal reason when applicable.
- **FR-023.17**: An order report MUST include the current version.
- **FR-023.18**: An order report MUST include the IDs of all trades associated with that report's
  committed state.
- **FR-023.19**: An order report's version MUST be a positive integer.
- **FR-023.20**: For each order, traded quantity plus booked quantity plus removed quantity MUST
  equal effective total quantity.
- **FR-023.21**: A quantity reduction through amendment MUST NOT increase removed quantity.

#### FR-024 — Stored order status

Submission, matching, amendment, cancellation, and expiry are all subject to the same status table.
The no-storage entry rule is FR-010.15.

- **FR-024.1**: The system MUST assign each stored order the status defined by the Order Status
  Rules for its current outcome.
- **FR-024.2**: The system MUST NOT expose a stored report with status `REJECTED`.

#### FR-025 — Trade retrieval and records

Ownership restrictions are defined in FR-004.2. Decimal and timestamp encodings are defined in
FR-011 and FR-042.

- **FR-025.1**: The system MUST allow a Broker to retrieve an accessible trade by ID.
- **FR-025.2**: The system MUST allow a Broker to list its own trades.
- **FR-025.3**: Each trade MUST have a system-generated ID.
- **FR-025.4**: Each trade MUST record its execution time.
- **FR-025.5**: Each trade MUST record its buy order ID.
- **FR-025.6**: Each trade MUST record its sell order ID.
- **FR-025.7**: Each trade MUST record its exact execution price.
- **FR-025.8**: Each trade MUST record its executed quantity.
- **FR-025.9**: Each trade's quantity MUST be positive.
- **FR-025.10**: The system MUST return identical trade data to both counterparties.
- **FR-025.11**: The system MUST NOT disclose an unrelated private order report through trade
  retrieval.

#### FR-026 — Order-book snapshots

- **FR-026.1**: A book snapshot MUST include the active product ID.
- **FR-026.2**: A book snapshot MUST include the product's current symbol.
- **FR-026.3**: A book snapshot MUST aggregate quantity by normalized price level.
- **FR-026.4**: A book snapshot MUST count only remaining booked quantity.
- **FR-026.5**: A book snapshot MUST sort bids from highest to lowest price.
- **FR-026.6**: A book snapshot MUST sort offers from lowest to highest price.
- **FR-026.7**: A book snapshot MUST omit zero-volume price levels.
- **FR-026.8**: A book snapshot MUST NOT disclose individual order identities.
- **FR-026.9**: A book snapshot MUST NOT disclose participant identities.

#### FR-027 — Order-book calculations

- **FR-027.1**: The system MUST provide the highest available bid as the best bid.
- **FR-027.2**: The system MUST provide the lowest available bid as the worst bid.
- **FR-027.3**: The system MUST provide the lowest available offer as the best offer.
- **FR-027.4**: The system MUST provide the highest available offer as the worst offer.
- **FR-027.5**: The system MUST calculate spread as best offer minus best bid when both sides have
  prices.
- **FR-027.6**: The system MUST calculate midpoint as the exact arithmetic mean of best bid and best
  offer when both exist.
- **FR-027.7**: The system MUST provide the booked volume at a specified side and exact price.
- **FR-027.8**: The system MUST provide total booked volume for each side.
- **FR-027.9**: An empty side MUST have zero volume.
- **FR-027.10**: An empty side MUST have no best-price value.
- **FR-027.11**: An empty side MUST have no worst-price value.
- **FR-027.12**: A missing price level MUST have zero volume.
- **FR-027.13**: A book lacking either side MUST have no spread value.
- **FR-027.14**: A book lacking either side MUST have no midpoint value.

#### FR-028 — Service liveness

Liveness confirms that the service is running; it does not promise storage durability or market
readiness.

- **FR-028.1**: The liveness endpoint MUST return HTTP 200 while the service is running.
- **FR-028.2**: The liveness response MUST include an explicit running indication.
- **FR-028.3**: The liveness endpoint MUST NOT require credentials.
- **FR-028.4**: The liveness response MUST NOT expose sensitive configuration.
- **FR-028.5**: The liveness response MUST NOT expose trading data.

#### FR-029 — Business and technical failure responses

Business rejection includes semantic validation, missing entities, unsupported entity types,
idempotency validation, and business conflicts. The failure tables define request parsing, protocol,
authentication, authorization, routing, request size/media, throttling, capacity, storage, and
unexpected execution outcomes. Numeric codes are selected by FR-043.1; envelope structure is defined
by FR-040; correlation is defined by FR-044.1.

- **FR-029.1**: The system MUST return HTTP 200 for a completed business rejection.
- **FR-029.2**: Every failed outcome MUST set `success=false`, including a business rejection
  carried by HTTP 200.
- **FR-029.3**: Every failed outcome MUST include its specific symbolic `data.reasonCode`.
- **FR-029.4**: Every failed outcome MUST include a safe human-readable `errorMessage`.
- **FR-029.5**: The system MUST use the technical HTTP status in Failure Outcomes for a request,
  security, admission, or execution failure.
- **FR-029.6**: The system MUST NOT allow framework defaults to convert a handled business rejection
  into an HTTP 4xx response.

#### FR-030 — Caller-scoped idempotency

The same rules apply to concurrent duplicates and later retries. Exact HTTP/code/payload replay is
defined by FR-043.3 through FR-043.5.

- **FR-030.1**: The system MUST accept a caller-scoped idempotency identity for each mutating
  command.
- **FR-030.2**: The system MUST return the original outcome for a same-identity retry of the same
  operation and payload within its retention period.
- **FR-030.3**: The system MUST NOT repeat business effects for a same-identity retry.
- **FR-030.4**: The system MUST reject a reused identity with different payload as
  `IDEMPOTENCY_CONFLICT`.
- **FR-030.5**: The system MUST reject a reused identity for a different operation as
  `IDEMPOTENCY_CONFLICT`.
- **FR-030.6**: The system MUST keep different callers' idempotency identities in separate scopes.
- **FR-030.7**: The system MUST recheck authentication on every retry.
- **FR-030.8**: The system MUST recheck authorization on every retry.

#### FR-031 — Failed and uncertain command outcomes

- **FR-031.1**: A command that fails before commit MUST leave no partial business effects.
- **FR-031.2**: The system MUST keep a committed command outcome discoverable after response loss.
- **FR-031.3**: The system MUST permit retry with the same identity after a retriable pre-commit
  failure.
- **FR-031.4**: The system MUST replay a retained rejection outcome consistently.
- **FR-031.5**: The system MUST reject a mutating request without a valid command identity before
  any business mutation.

#### FR-032 — Atomic commands and order versions

The atomic state in FR-032.1 includes matches, both sides' order changes, book changes, trades, the
idempotency outcome, and attributable audit record. Version advancement is per changed order per
command, even for multiple fills. Stale-amendment rejection is defined by FR-018.19; cancelled-order
ineligibility is defined by FR-017.5.

- **FR-032.1**: The complete business effects of a command MUST become visible as one atomic state
  change.
- **FR-032.2**: The system MUST establish one acceptance order for conflicting commands.
- **FR-032.3**: Concurrent execution MUST NOT create duplicate fills.
- **FR-032.4**: Concurrent execution MUST NOT create negative quantities.
- **FR-032.5**: An accepted order MUST start at version 1.
- **FR-032.6**: A subsequent committed command MUST advance each order it changes by exactly one
  version.
- **FR-032.7**: A failed command MUST NOT advance an order version.
- **FR-032.8**: An effective no-op MUST NOT advance an order version.
- **FR-032.9**: An idempotent replay MUST NOT advance an order version.
- **FR-032.10**: The system MUST resolve an idempotent replay before comparing its original
  amendment version with the current order version.

#### FR-033 — Read consistency and timestamp ties

Consistency applies within each response. Separate reads can observe different committed states
because intervening commands are permitted.

- **FR-033.1**: Each read MUST reflect a complete committed state.
- **FR-033.2**: An order report's quantities MUST reconcile with its trade references for that
  response's committed state.
- **FR-033.3**: Each book snapshot MUST be internally consistent.
- **FR-033.4**: The acceptance-priority order MUST unambiguously resolve equal displayed timestamps.

#### FR-034 — Storage mode and replacement

The first proof of concept is permitted to use memory and lose process-local state on restart; the
user-selected implementation does so. This permission does not promise durability.

- **FR-034.1**: The system MUST document its supported deployment topology.
- **FR-034.2**: The system MUST document the lifetime of retained data in its selected storage mode.
- **FR-034.3**: The system MUST explicitly disclose restart data loss in the in-memory mode.
- **FR-034.4**: Replacing a storage adapter MUST preserve observable matching behavior.
- **FR-034.5**: Replacing a storage adapter MUST preserve concurrency guarantees.
- **FR-034.6**: Replacing a storage adapter MUST preserve application error behavior.
- **FR-034.7**: Replacing a storage adapter MUST preserve public API behavior.
- **FR-034.8**: Replacing a storage adapter MUST NOT require changes to business rules.
- **FR-034.9**: The project MUST define durability guarantees before adopting persistent storage.
- **FR-034.10**: The project MUST define migration behavior before adopting persistent storage.
- **FR-034.11**: The project MUST define recovery behavior before adopting persistent storage.

#### FR-035 — Audit history and operational evidence

- **FR-035.1**: Each mutation-history record MUST identify its actor.
- **FR-035.2**: Each mutation-history record MUST identify its command.
- **FR-035.3**: Each mutation-history record MUST identify its outcome.
- **FR-035.4**: Each mutation-history record MUST identify its ordering information.
- **FR-035.5**: Each mutation-history record MUST identify the orders it affects.
- **FR-035.6**: Each mutation-history record MUST identify the trades it creates.
- **FR-035.7**: Mutation history MUST NOT expose credentials.
- **FR-035.8**: Operational records MUST carry the invocation correlation identifier.
- **FR-035.9**: Operational records MUST support measuring request latency.
- **FR-035.10**: Operational records MUST support measuring completed business throughput.
- **FR-035.11**: Operational records MUST support measuring application error results.
- **FR-035.12**: Operational records MUST support measuring contention.
- **FR-035.13**: Operational records MUST support measuring saturation.
- **FR-035.14**: Audit retention MUST match the advertised storage lifetime.

#### FR-036 — Finite limits and bounded queries

- **FR-036.1**: The project MUST publish finite request-size limits before implementation.
- **FR-036.2**: The project MUST publish finite numeric-input limits before implementation.
- **FR-036.3**: The project MUST publish finite collection-page limits before implementation.
- **FR-036.4**: The project MUST publish finite wait limits before implementation.
- **FR-036.5**: The project MUST document overload behavior before implementation.
- **FR-036.6**: The system MUST bound every collection page.
- **FR-036.7**: The system MUST use deterministic ordering for collection results.
- **FR-036.8**: The project MUST document continuation behavior when collection data changes
  concurrently.
- **FR-036.9**: The system MUST explicitly reject an input that exceeds a published limit.
- **FR-036.10**: The system MUST NOT silently truncate a requested financial operation to satisfy a
  limit.

#### FR-037 — Capacity and overload acceptance

SC-007 fixes the baseline dataset, request mix, hardware, and measurement boundary. SC-008 fixes the
2,000-request/s, up-to-1,000-client, 60-second burst. These existing targets are unchanged; HTTP 200
with success=false is still a rejected outcome.

- **FR-037.1**: The system MUST complete at least 1,000 valid business requests per second under the
  SC-007 baseline.
- **FR-037.2**: The system MUST support the baseline workload with 100 concurrent clients.
- **FR-037.3**: The system MUST sustain the baseline acceptance workload for 15 continuous minutes.
- **FR-037.4**: Baseline p95 request completion latency MUST be at most 100 milliseconds.
- **FR-037.5**: Baseline p99 request completion latency MUST be at most 250 milliseconds.
- **FR-037.6**: The baseline unexpected-failure rate MUST remain below 0.1 percent.
- **FR-037.7**: Baseline peak service resident memory MUST be at most 4 GiB.
- **FR-037.8**: The system MUST return an explicit overload outcome when admission capacity is
  exceeded.
- **FR-037.9**: The system MUST preserve business invariants under the SC-008 overload workload.
- **FR-037.10**: Overload peak service resident memory MUST be at most 4 GiB.
- **FR-037.11**: Overload response p99 latency MUST be at most 2 seconds.
- **FR-037.12**: The system MUST recover baseline acceptance performance within 30 seconds after
  offered load returns to baseline.
- **FR-037.13**: Performance reports MUST exclude rejected outcomes from successfully completed
  business work.
- **FR-037.14**: The system MUST enforce its published resource bounds during overload.

#### FR-038 — JSON resources and URL bases

The API Resource Requirements table defines each family's capabilities. Role enforcement remains
FR-002; caller binding and ownership remain FR-003 and FR-004. Selecting a URL does not confer
authority.

- **FR-038.1**: The service MUST expose its application interfaces over HTTP.
- **FR-038.2**: The service MUST use JSON for application request and response representations.
- **FR-038.3**: The service MUST provide the `/product/equity` resource family.
- **FR-038.4**: The service MUST provide the `/participant/broker` resource family.
- **FR-038.5**: The service MUST provide the `/order` resource family.
- **FR-038.6**: The service MUST provide the `/orderBook/{productId}` resource family.
- **FR-038.7**: The service MUST provide the `/status` resource.
- **FR-038.8**: The service MUST use `/internal` as its administration base path.
- **FR-038.9**: The service MUST place participant resources directly at the service root.

#### FR-039 — Bodyless resource reads

Selectors include IDs, symbols, filters, and pagination. Bodyless reads do not require an empty JSON
object.

- **FR-039.1**: The system MUST make every resource read available through GET.
- **FR-039.2**: A resource read MUST NOT require a request body.
- **FR-039.3**: A resource read MUST NOT require a request Content-Type header.
- **FR-039.4**: The system MUST accept read selectors through path or query parameters.
- **FR-039.5**: A resource read MUST NOT require an Accept header.

#### FR-040 — Common JSON envelope

These rules apply to controller responses, middleware errors, liveness, empty collections,
deletions, and unexpected failures. The existing envelope table defines message values and data
contents; it remains normative interface data.

- **FR-040.1**: Every JSON response MUST use only the top-level fields defined by the JSON Response
  Envelope.
- **FR-040.2**: Every JSON response MUST include a boolean `success` field.
- **FR-040.3**: Every JSON response MUST include `responseCode`.
- **FR-040.4**: Every JSON response MUST include a string `infoMessage` field.
- **FR-040.5**: Every JSON response MUST include a string `warningMessage` field.
- **FR-040.6**: Every JSON response MUST include a string `errorMessage` field.
- **FR-040.7**: A JSON response MUST include `data` when the outcome has relevant result or failure
  details.
- **FR-040.8**: Every JSON response MUST declare Content-Type `application/json`.
- **FR-040.9**: Every numeric application response code MUST be a JSON integer.

#### FR-041 — Unexpected execution failures

The failure success flag is defined in FR-029.2; the common envelope and correlation header are
defined in FR-040 and FR-044.1.

- **FR-041.1**: The system MUST return HTTP 500 for an unexpected execution failure.
- **FR-041.2**: An unexpected-failure response MUST use numeric response code 999999.
- **FR-041.3**: An unexpected-failure response MUST use error message `An unexpected error
  occurred.`.
- **FR-041.4**: An unexpected-failure response MUST use reason `INTERNAL_ERROR`.
- **FR-041.5**: An error response MUST NOT expose exception text.
- **FR-041.6**: An error response MUST NOT expose stack traces.
- **FR-041.7**: An error response MUST NOT expose implementation or storage internals.
- **FR-041.8**: An error response MUST NOT expose credentials.
- **FR-041.9**: An error response MUST NOT expose private payloads.
- **FR-041.10**: Detailed correlated failure diagnostics MUST remain internal.

#### FR-042 — API date and time format

The format specifies four-digit years, a 24-hour clock, and exactly three millisecond digits.
Priority tie resolution is FR-033.4.

- **FR-042.1**: Every API date/time value MUST use format `dd-MMM-yyyy HH:mm:ss.SSS`.
- **FR-042.2**: API month abbreviations MUST be English.
- **FR-042.3**: API date/time values MUST represent UTC.
- **FR-042.4**: API date/time formatting MUST be independent of the host locale.
- **FR-042.5**: API date/time formatting MUST be independent of the host timezone.
- **FR-042.6**: Displayed date/time precision MUST NOT determine matching priority.

#### FR-043 — Operation-specific codes and replay fidelity

HTTP status distinguishes completed handling from request/execution failure; success distinguishes
the business outcome. HTTP 200 with success=false is required by FR-029.1 and FR-029.2.

- **FR-043.1**: Each operation MUST select the numeric response code for its own entity and outcome
  from the result-code catalog.
- **FR-043.2**: The `success` flag MUST represent the current business outcome independently of the
  order's stored trading status.
- **FR-043.3**: An idempotent replay MUST retain the original HTTP status.
- **FR-043.4**: An idempotent replay MUST retain the original numeric response code.
- **FR-043.5**: An idempotent replay MUST retain the original business payload.

#### FR-044 — Response headers and successful HTTP outcomes

Non-creation operations include reads, updates, amendments, cancellations, deletions, and liveness.
These rules exclude a bodyless HTTP 204 deletion and caching of HTTP 200 rejection/entity-miss
results. The envelope permits no extra top-level correlation field under FR-040.1.
Business-rejection HTTP status is defined by FR-029.1.

- **FR-044.1**: Every application-generated HTTP response MUST carry X-Correlation-ID.
- **FR-044.2**: A successful creation MUST return HTTP 201.
- **FR-044.3**: A successful non-creation operation MUST return HTTP 200.
- **FR-044.4**: Every application-generated HTTP response MUST carry Cache-Control `no-store`.
- **FR-044.5**: A successful deletion MUST include a JSON response body.

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

Trade-by-ID and trade-list reads required by FR-025.1 and FR-025.2 remain available within the `/order` resource
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
  Current priority can change independently of original entry time as defined in FR-018.8 through FR-018.11.
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
