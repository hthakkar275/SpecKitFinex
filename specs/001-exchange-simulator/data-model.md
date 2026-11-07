# Data Model: Electronic Financial Exchange Simulator

This is the Phase 1 model for the in-memory adapter. Wire schemas are in
[OpenAPI](contracts/openapi.json); precise HTTP limits and errors are in [api.md](contracts/api.md).
The model preserves all feature statuses and the two accepted clarifications.

## Value Objects and Representation

| Value | Domain representation | Rules / JSON representation |
| --- | --- | --- |
| Entity/command IDs | UUID entity IDs; bounded string command identity | Generated opaque entity IDs; command key 1–128 printable ASCII characters. |
| Quantity | Checked Java `long` | Whole shares; order total 1–1,000,000,000; JSON integer. Aggregates use checked `long`, never wrap. |
| Price/tick | Canonical `BigDecimal` | Plain decimal string, 1–24 characters, at most 6 fractional digits; value >0 and ≤1,000,000,000. Rounding to a tick must remain positive and within range. |
| Spread/midpoint | Exact `BigDecimal` | Plain decimal string; spread ≥0; midpoint may have 7 fractional digits and need not be tick aligned. No formatting rounding. |
| Timestamp | `Instant` + injected UTC `Clock` | `dd-MMM-yyyy HH:mm:ss.SSS`, English months, UTC; never used alone to break priority ties. |
| Version/priority/revision | Positive checked `long` | Version starts 1. Numeric JSON integer where exposed; counter exhaustion rejects before changes. |
| Symbol | Normalized string | Trim/uppercase with `Locale.ROOT`; 1–32 ASCII letters/digits, dot or hyphen, begins alphanumeric. Unique including retained tombstones. |

Decimal input grammar is digits with an optional dot and 1–6 fractional digits, no sign/grouping/
exponent. Positive-value/range checks follow grammar. Domain examples in spec prose are values;
actual JSON decimals are quoted strings. Output uses plain canonical form (`0` for zero, no
unnecessary trailing zeros). The canonical form represents the same exact value, not a fixed scale.

## Administrator and Broker Identity

- `PrincipalId`: immutable generated identity; `role`: ADMINISTRATOR or PARTICIPANT.
- `username`: 1–64 ASCII letters/digits/dot/underscore/hyphen; unique, case-sensitive.
- `passwordHash`: BCrypt hash, never included in API/report/audit. Password input is write-only,
  16–64 ASCII characters, never logged or retained in DTOs beyond processing.
- `brokerId`: required for PARTICIPANT, absent for ADMINISTRATOR; server-owned mapping.
- `credentialGeneration`, `active`: revoke/delete/rotation boundaries checked on every request.
- Administrator bootstrap comes from environment configuration. Broker create requires a name,
  `type=BROKER`, username, password; update may change name and rotate password. Username,
  principal ID, Broker ID, and type are immutable. Creating credentials and Broker is one transaction.

Identity lookup never treats a client-supplied Broker ID as proof of identity. Deleting a Broker
disables its identity atomically and retains tombstones/history. Authentication and cached
outcome replay recheck active/generation state; responses never return hashes or passwords.

## Product

Fields: `id`, `type=EQUITY`, normalized `symbol`, `description` (1–512 characters), `tickSize`
(default `0.01`), `active`, and internal metadata revision. ID/type are immutable. Symbol/tick
updates require zero booked orders; description can change without altering order priority.
Deletion requires zero open orders and prevents active lookup/new orders. Historical references
still resolve internally. A stable `ProductSlot` holds the synchronization object even after deletion;
its optional `OrderBook` is allocated only when needed. Before a deletion or symbol/tick in-use
check, administration expires affected books under the metadata write gate and sorted product locks.

## Participant

Fields: `id`, `type=BROKER`, `name` (trimmed, 1–128 characters), `principalId`, `active`, and
transactional `openOrderCount`. Names need not be unique. Open count counts orders with positive
booked quantity, not shares, and changes together with the matching/cancel/expiry transaction.
Administrator deletion inspects the count under the exclusive metadata gate. Counter/identity
changes must be undone if a commit fails. Deletion does not erase prior orders/trades.

## Order

| Field | Meaning |
| --- | --- |
| `id`, `productId`, `participantId` | Immutable order identity and references. |
| `side`, `type`, `longevity` | BUY/SELL, LIMIT/MARKET, DAY; immutable. |
| `price` | Current normalized limit price; omitted on market orders. |
| `entryTime`, `entryDay` | Original successful acceptance time and UTC trading day; immutable. |
| `prioritySequence` | Current product-local queue priority; price change/increase resets it. |
| `originalQuantity` | Original submitted whole-share total; immutable even after amendment. |
| `effectiveTotalQuantity` | Latest accepted total including prior executions, not an amendment delta. |
| `tradedQuantity`, `bookedQuantity`, `removedQuantity` | Cumulative executions, remaining eligible shares, dropped/cancelled/expired shares. |
| `status`, `removalReason` | Status below; reason absent unless removed quantity is positive. |
| `version` | Current optimistic amendment version; positive integer. |
| `tradeIds` | Execution-order references; response pages use the bounded continuation policy. |

Invariant: `traded + booked + removed = effectiveTotal`; every value is nonnegative, traded
quantity equals all referenced execution quantities, and only booked quantity can trade.
A valid amendment total is strictly greater than traded quantity. Original quantity does not
participate in that equality once the total has changed.

### Status transitions

| Event | Result |
| --- | --- |
| Limit accepted, no fills | BOOKED |
| Limit partially matched, remainder open | PARTIALLY_BOOKED_FILLED |
| Entire effective total matched, any order type | FILLED |
| Market partially matched, remainder dropped | PARTIALLY_FILLED + MARKET_REMAINDER_DROPPED |
| Market finds no liquidity | NOT_FILLED + MARKET_REMAINDER_DROPPED |
| Cancel/expiry, no prior fills | CANCELLED + PARTICIPANT_CANCELLED or DAY_EXPIRED |
| Cancel/expiry, some fills | PARTIALLY_CANCELLED + PARTICIPANT_CANCELLED or DAY_EXPIRED |
| Invalid entry | No stored order; REJECTED remains reserved |

Terminal orders cannot return to the book. A successful new order starts at version 1 including
all immediate matches; each subsequent state-changing command increments every changed order
once, regardless of fill count. Effective no-ops, rejections, and replay do not increment.
Amend requires current `expectedVersion`; stale versions fail before terminal-state evaluation.
Cancellation requires no version and removes the remainder present when it runs.

## OrderBook and PriceLevel

One logical book per product. `bids` descend and `offers` ascend by numeric price. Each level
has exact total quantity and insertion-ordered order nodes; an order-ID index permits direct
cancellation/amendment removal without scanning all orders. `revision` changes whenever open
book contents change. A quantity reduction retains the existing node/priority; price changes
or increases append with new sequence after immediate matching. Empty levels disappear.

Snapshots copy one committed revision and include product ID/symbol, price levels, exact best/
worst prices, spread, midpoint, per-side volume, and continuation metadata. Missing sides have
zero volume and omitted price metrics; spread/midpoint are omitted unless both sides exist.
A missing exact price level has zero quantity. Explicit query-side validation still applies.

## Trade

Immutable `id`, `time`, `buyOrderId`, `sellOrderId`, `productId`, price, positive quantity, and
internal product sequence and process-wide insertion sequence. Assign insertion sequence while
holding its product lock; never reuse/decrement it on rollback. A trade executes at the resting price for the minimum remaining
quantity. Ownership derives from its orders; both Brokers see the same trade but cannot use its
counterparty order ID to fetch a private order report. Trade IDs are generated/staged before
publication and may have harmless gaps after a failed transaction.

## CommandOutcome and AuditEvent

`CommandOutcome`: `(principalId, key)`, normalized operation/target/payload fingerprint,
IN_FLIGHT/COMPLETED reservation, immutable HTTP status and business envelope, completion time,
expiry at least 24h later. Replay is authorized again and happens before version validation.
Outstanding commands never expire. Infrastructure failure before commit releases the reservation;
business rejection uses HTTP 200/`success=false` and can be replayed. Temporary wait timeout does not overwrite the original.

`AuditEvent`: immutable ID, command/system identity, actor ID, operation, accepted sequence,
time, affected product/order/trade IDs, and committed outcome. Contains no credentials/full HTTP
payloads. One logical mutation has one audit event; matching trade references are recorded within
it. Expiry is a SYSTEM mutation with lifecycle-reserved capacity. Replays do not duplicate audit.

## Adapter Indexes and Atomic Boundary

Indexes: product IDs/symbols/tombstones; participant IDs/usernames/open counts; order/trade ID
locators to product slots; per-product orders/levels/trades; participant trade membership;
idempotency reservations; append-only segmented audit/history. Mutable structures never escape
adapter methods. Queries take the same product locks and copy immutable DTOs; global trade pages
take relevant product locks in sorted ID order while holding the metadata read gate. An unfiltered
own-trade query locks all retained product slots simultaneously before capturing its insertion
watermark; a pending lower-sequence commit therefore cannot leak into a later page. Private reads
recheck active identity/credential generation under that gate before copying data.

Lock order: admission → command reservation → metadata read/write gate → sorted product locks.
Duplicate command waiting never holds domain locks. Administration takes the exclusive metadata
gate, so it cannot race a commit that references a deleted entity. State deltas and undo journals
cover every changed index, counter, node, audit entry, and outcome. Shared counters undo their
own atomic deltas, never overwrite an absolute old count. Shared history/audit rollback removes
only entries owned by that transaction, never truncates another product's append. Recovery tests inject failures
between writes and require exact restoration before unlocking.

Per-order lifecycle reservations guarantee room to cancel/expire retained open orders even when
normal growth admission is exhausted. Neither a count cap nor memory pressure silently deletes
retained history, book levels, or unexpired command outcomes. Fatal process loss erases everything;
future persistence must replace the entire atomic port contract, not individual map calls.
