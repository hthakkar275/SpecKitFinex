# Implementation Plan: Electronic Financial Exchange Simulator

**Branch**: `001-exchange-simulator` (Spec Kit logical feature identifier; no Git repository or
branch was created) | **Date**: 2026-11-07 | **Spec**: [spec.md](spec.md)

**Input**: `specs/001-exchange-simulator/spec.md`; user-selected Spring Boot 4.1.0, JDK 21,
and in-memory persistence. This command ends at design; it does not implement the application.

## Summary

Build one Spring MVC REST service with an `/internal` administration base and root-level participant resources and public
`/status`. Administrators manage Equity products and Broker identities; Brokers submit, amend,
cancel, and monitor orders and trades. Pure Java domain rules implement exact-decimal tick
rounding, price-time matching at resting prices, versions, and UTC day expiry. Application-owned
transaction/query ports isolate a concurrency-safe in-memory adapter from controllers and business
rules. Product-scoped transactions stage touched changes and roll back recoverable failures;
books, both order sides, trades, audit, indexes, and idempotent results commit together.

See [research](research.md), [data model](data-model.md), [HTTP contract](contracts/api.md),
[machine-readable OpenAPI](contracts/openapi.json), and [validation guide](quickstart.md).

## Technical Context

**Language/Version**: Java 21; immutable records/value objects where appropriate. Maven Wrapper
3.9.11 and compiler release 21; Spring Boot parent/plugin pinned to 4.1.0 as requested.

**Primary Dependencies**: Boot MVC, Validation, Security, Actuator starters; Boot-managed
Jackson 3, Tomcat 11, Micrometer, JUnit Jupiter 6, Mockito, MVC test support, security test support.
Use `java.math.BigDecimal`, injected `Clock`, and JDK locks; no database or broker dependency.

**Storage**: Process-local maps and ordered price levels behind `ExchangeTransactionPort` and
`ExchangeQueryPort`. All state is lost on process restart. Exactly one active instance; no
horizontal replicas or persistent adapter are delivered in this feature.

**Testing**: JUnit unit/reference-model tests; MVC JSON/schema/error tests; real random-port HTTP
and security tests; reusable storage-port contract suite; deterministic concurrent/fault-injection
and UTC-boundary tests. Maven Surefire for unit/contract tests and Failsafe for `*IT` integration
scenarios. JDK 21 load driver/JFR and operator-restricted Micrometer metrics for capacity evidence.

**Target Platform**: JDK 21 JVM on a Linux server, reproducible Linux performance runs; local
macOS development supported. Current workspace Java is 11, so future execution requires selecting
JDK 21. Network deployments require TLS; examples use loopback HTTP with synthetic data only.

**Project Type**: Single Maven REST web-service project, executable JAR; no frontend.

**Performance Goals**: SC-007: 1,000 successful business requests/s, 100 concurrent clients,
15 minutes, p95 ≤100 ms, p99 ≤250 ms, unexpected failures <0.1%, RSS ≤4 GiB on 4 CPU/8 GiB.
SC-008: 2,000 offered requests/s, up to 1,000 clients, 60 seconds; response p99 ≤2 seconds,
controlled rejection, invariants preserved, baseline recovery within 30 seconds.

**Constraints**: Price-time correctness, atomic multi-fill effects, no duplicate retries, decimal
strings/UTC date format/common envelope, optimistic amendment versions, bodyless GETs, bounded
work and retained history. Limits and authentication rules are fixed in `contracts/api.md`.
Performance remains unmeasured until implementation; an executable failure is not a passing gate.

**Scale/Scope**: Baseline 10 products, 100 Brokers, 10,000 open orders and 100 populated levels per
product, 50% traffic on one hot product. Finite configured caps include 1,000 products, 10,000
Brokers, 1,000,000 orders, 2,000,000 trades, and 1,000,000 completed command outcomes, subject to
stricter weighted-memory admission. Retained history is never silently evicted before its promise.

## Constitution Check

Pre-research review: all five principles can be satisfied without exemptions. Post-design review:
all five are covered by the concrete design below; execution evidence remains a release gate.

| Principle | Pre-design | Post-design evidence and required check |
| --- | --- | --- |
| I. REST/error contracts | PASS | OpenAPI plus `contracts/api.md`; HTTP 200 business-result policy; envelope writer across MVC/security/error paths; operation-specific code/HTTP tests. |
| II. Correctness/concurrency | PASS | Product transaction lock, metadata gate, touched-state rollback, process-wide idempotency, exact decimals, versions, expiry; race/reference-model/fault tests. |
| III. Measured performance | PASS | Numeric SC-007/008 workload retained; admission/record/memory bounds; reports separate committed work, rejection, and timeouts; real benchmark required. |
| IV. Replaceable storage | PASS | Business-operation transaction/query ports, pure matching engine, adapter-independent acceptance tests; no collection/database types in public ports. |
| V. Verification/audit | PASS | Audit and idempotency outcome in the commit journal; correlated metrics/logs and permission tests; explicit restart-loss limitation. |

Security/scope check: synthetic assets only, authenticated role/owner checks, TLS off loopback,
redacted secrets, restricted management interface. Workflow check: every FR-001–FR-044 has design
and validation coverage below. No constitutional exception is requested. A failed future
correctness, contract, or performance check blocks release rather than being relabeled as passed.

## Project Structure

### Documentation (this feature)

```text
specs/001-exchange-simulator/
├── spec.md
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── checklists/requirements.md
└── contracts/
    ├── api.md
    └── openapi.json
```

`tasks.md` belongs to the next `$speckit-tasks` phase and is not generated here.

### Source Code (repository root)

Planned layout only; these application files are not created by this command.

```text
pom.xml
mvnw
mvnw.cmd
.mvn/wrapper/
src/main/java/com/finex/exchange/
├── ExchangeApplication.java
├── domain/                 # Products, orders, trades, exact values, pure matching delta
├── application/
│   ├── service/            # Command/query orchestration and authorization
│   └── port/               # Atomic transaction/query/identity contracts
├── adapter/
│   ├── http/               # Internal and external controllers, DTOs, envelope/error writer
│   └── memory/             # Locks, indexes, staging/journal, identity/idempotency store
└── configuration/          # Security, JSON, Clock, limits, management
src/main/resources/application.yaml
src/test/java/com/finex/exchange/
├── domain/
├── contract/
├── concurrency/
├── http/
├── integration/            # *IT real HTTP and injected adapter-failure scenarios
└── performance/            # Seeded workload/standalone runner, not ordinary unit tests
src/test/resources/fixtures/
```

**Structure Decision**: One deployable, with explicit inward dependencies. HTTP DTOs do not expose
mutable storage objects. Domain code depends only on JDK types; adapter wiring selects memory.
No separate microservices, generic repository framework, or speculative persistent schema is needed.

## Design and Execution Boundaries

1. **HTTP and identity**: Match `/internal/**` first, public `GET /status` second, and explicit
   authenticated participant resources at root, with unmatched paths denied. Authenticate Basic
   credentials,
   enforce administrator/Broker ownership, and map DTOs strictly. Administrator bootstrap uses
   environment secrets; administrator Broker creation establishes a unique username and hash.
   Read requests need no body, Content-Type, or Accept. Completed business rejections use HTTP
   200 with `success=false`; request/security/system failures retain 4xx/5xx. Serialize the exact
   envelope and set `Cache-Control: no-store`. Count business success from the envelope.
2. **Command protocol**: Bounded admission → global caller/key reservation → metadata read gate →
   stable product lock → lazy expiry → validation/version comparison → pure delta → capacity
   reservation → journaled apply → final outcome publication → unlock → waiter notification.
   Administrator mutations instead hold the metadata write gate; expire affected books under
   sorted product locks before deletion/symbol/tick in-use checks. Lock waits share one deadline.
3. **Fair matching**: Numeric price-sorted maps and insertion-ordered nodes per price with an
   order-ID node index. A product sequence assigned at acceptance resolves timestamp ties.
   Price changes/increases append with new priority; reductions keep the original queue node.
   Versions advance once per changed order per committed command, not per fill.
4. **Atomicity**: Stage immutable orders/trades/audit/outcome and reversible touched-entry updates.
   No whole-history copies or side-effectful callbacks while committing. Readers use the same
   product lock; faults restore maps, queue positions, and command state before unlock. Undo
   shared counters
   using compensating atomic deltas and remove only transaction-owned history/audit entries;
   never reset a shared absolute counter or truncate another transaction's append.
5. **Expiry**: Inject UTC clock and check under the product lock on every dependent operation.
   Recheck before apply and before final publication; cross-midnight staging is discarded, expiry
   committed, and command restaged after rolling back unpublished applied writes. Cancellation/
   expiry capacity is reserved when accepting open exposure.
6. **Queries**: Copy immutable bounded results under lock. Cross-product trade pages lock relevant
   product slots in sorted ID order, including every retained product for an unfiltered own-trade query.
   Capture committed-view high-water marks only while all those locks remain held; stale
   book cursors fail rather than claiming an inconsistent snapshot is coherent.
7. **Operate**: Metrics/logging exclude secrets and payloads; traces use correlation IDs. Retain
   history for process lifetime, idempotency for at least 24h, and reject growth at finite caps.
   Management is loopback/operator-only; root `/status` stays the public envelope endpoint.

## Verification and Requirement Coverage

| Requirements | Design location | Acceptance evidence |
| --- | --- | --- |
| FR-001–008 | HTTP access rules, identity/reference model | Administrator lifecycle, duplicate symbols, deletion races, role/ownership denial. |
| FR-009–017 | Pure matching and atomic book delta | Exact ticks, resting-price fills, priority ties, invalid-order absence, limit/market residuals. |
| FR-018–024 | Versions, cancellation, UTC expiry/report model | Stale/no-op/replay cases; all seven stored statuses; midnight and cancel/fill races. |
| FR-025–028 | Consistent query ports and status | Own trade visibility, cross-book list snapshot, exact metrics, public liveness. |
| FR-029–033 | Error writer, global idempotency, commit journal | Complete numeric/reason mappings, cross-product key reuse, faults at every journal write. |
| FR-034–037 | Adapter ports, audit/reservations/admission | Contract suite, restart loss, memory/record caps, cancellation reserve, SC-007/008 reports. |
| FR-038–044 | OpenAPI/JSON formatter/security handlers | All five resource families; bodyless reads; decimal strings; UTC English format; envelope/status/code consistency. |

Testing must include a warm-cache steady-state run and a separately reported cold-authentication
run, 10,000 seeded command sequences, and at least 10,000 concurrent conflict groups. Benchmark
fixtures use seed 20261106 and maintain valid amendment/cancel targets and both matched/resting
liquidity; expected stale-version or missing-liquidity rejections do not count as useful throughput.
A future implementation must fix its exact fixture schedule and publish JDK/OS/CPU/heap, GC,
RSS, request counts, percentiles, errors, and lock/admission metrics with the report.

## Complexity Tracking

No constitutional violations or exceptions. Product-scoped transactions, a metadata gate, a
small touched-entry journal, and a global command registry are necessary to meet atomic matching,
reference-data deletion, safe retries, and replaceable storage; they do not introduce additional
processes or a persistence framework.
