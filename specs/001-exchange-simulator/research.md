# Research: Electronic Financial Exchange Simulator

Date: 2026-11-07. Scope: Phase 0 planning, not application implementation.
All design unknowns are resolved below; the performance targets remain acceptance targets,
not claims of measured capacity. The current workspace defaults to JDK 11; JDK 21 must be
selected for future build/test execution. No JDK or dependencies are installed by this command.

## R1. Runtime and dependency baseline

**Decision**: Pin `org.springframework.boot:spring-boot-starter-parent:4.1.0`, JDK 21
(`maven.compiler.release=21`), and Maven Wrapper 3.9.11. Use executable JAR packaging.
Boot 4.1.0 is the explicitly requested version, even though 4.1.2 was released on 2026-10-22.
A future patch upgrade is a separate dependency change, not an implicit change to this plan.

**Rationale**: The exact 4.1.0 requirements support Java 17–26 and Maven 3.6.3+.
The project can therefore use JDK 21 without a framework downgrade. Maven 3.9.11 is released
and can be pinned by wrapper distribution URL and checksum.

**Alternatives considered**: Boot 4.1.2 (newer, not the requested pin); Boot 3.x (unnecessary);
Gradle (supported, but a second build system adds no value here).

Sources: [4.1.0 announcement](https://spring.io/blog/2026/06/10/spring-boot-4/),
[version-tagged requirements](https://raw.githubusercontent.com/spring-projects/spring-boot/v4.1.0/documentation/spring-boot-docs/src/docs/antora/modules/ROOT/pages/system-requirements.adoc),
[4.1.2 release](https://github.com/spring-projects/spring-boot/releases/tag/v4.1.2),
[Maven 3.9.11](https://maven.apache.org/docs/3.9.11/release-notes.html).

## R2. HTTP, JSON, and verification dependencies

**Decision**: Use Boot-managed `spring-boot-starter-webmvc`, `spring-boot-starter-validation`,
`spring-boot-starter-security`, and `spring-boot-starter-actuator`. Tests use
`spring-boot-starter-webmvc-test` plus Boot-managed `spring-security-test`. Let the Boot parent
manage Jackson 3, Spring Security, JUnit Jupiter 6, Mockito, and embedded Tomcat versions.
Do not add JDBC/JPA, H2, Redis, or a message broker.

**Rationale**: MVC fits synchronous HTTP commands and Java locks. The exact Boot 4.1.0 source
uses Jackson 3.1.4 and JUnit Jupiter 6.0.3; copying Boot 3/Jackson 2 test configuration is unsafe.
Keep the domain free of Spring/Jackson annotations. DTOs carry decimal fields as Java strings;
map to `BigDecimal` explicitly and format `Instant` with an immutable UTC/English formatter.
Reject JSON-number-to-string coercion and unknown request fields deliberately.

**Alternatives considered**: WebFlux (adds reactive coordination without nonblocking storage);
manual dependency versions (defeat the tested Boot BOM); runtime OpenAPI UI (not required).

Sources: [MVC starter](https://raw.githubusercontent.com/spring-projects/spring-boot/v4.1.0/starter/spring-boot-starter-webmvc/build.gradle),
[version-tagged dependency properties](https://raw.githubusercontent.com/spring-projects/spring-boot/v4.1.0/gradle.properties),
[MVC test starter](https://raw.githubusercontent.com/spring-projects/spring-boot/v4.1.0/starter/spring-boot-starter-webmvc-test/build.gradle).

## R3. Storage and transaction boundary

**Decision**: Application-owned `ExchangeTransactionPort`, `ExchangeQueryPort`, and
`IdentityPort` express complete business operations. The in-memory adapter implements the
transaction protocol; pure domain matching computes deltas rather than calling collection APIs.
A fair metadata read/write gate protects reference-data changes; a fair per-product lock
serializes each product's matching and consistent queries. Different products can trade in parallel.

**Rationale**: Metadata changes exclude trading, preventing deletion/identity races across books.
Stable product slots exist at product registration; the book itself is allocated lazily.
Concurrent maps are indexes, not a substitute for an atomic business transaction. Participant
open-order counters are updated transactionally and inspected by administration under the
metadata write gate. Deletion/symbol/tick changes first expire affected books under that write
gate and sorted product locks, then evaluate live open-order counts. No read-to-write lock upgrade is permitted.

**Commit protocol**: Admit/reserve command identity, take metadata read then product lock,
check expiry/entities/version, compute immutable order replacements and a touched-entry delta,
reserve capacity and allocate an undo journal, then apply all indexes/book/trades/audit/outcome
changes while readers are excluded. Recoverable faults roll back every touched write before
unlocking. Shared participant/capacity counters undo this transaction's atomic delta, never restore
an old absolute value; shared audit/history rollback removes only transaction-owned entries, never
truncates a global tail that another product may have extended. Publish the immutable completed outcome last; notify duplicate waiters after release.
Never execute arbitrary completion callbacks, network I/O, JSON serialization, or password hashing
inside the commit section. Work is proportional to touched state, not all retained history.
Fatal JVM termination/OOM is process loss under the declared non-durable mode, not recoverable commit.

**Alternatives considered**: One global matching lock (unnecessary cross-product contention);
copying the complete exchange on each command (history-dependent cost); unrelated CRUD writes
(partial matches and missing audits); persistent collections (extra dependency for this scope).

Source: [JDK 21 read/write lock semantics](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/locks/ReentrantReadWriteLock.html).

## R4. Idempotency, versions, and time

**Decision**: Reserve `(principalId, Idempotency-Key)` in a process-wide registry before domain
locks. Fingerprint HTTP business operation, target, and parsed business fields, including
`expectedVersion`; normalize decimal strings by exact numeric value before tick rounding.
Thus `"100.010"` equals `"100.01"`, but `"100.014"` is different even if rounded prices coincide.
Omission versus an explicit documented default is normalized identically; unknown fields fail.
Passwords in reference-data fingerprints use a keyed digest, never retained plaintext.
Same-payload duplicates wait at most 250 ms without domain locks; pending responses are HTTP 503
with reason `COMMAND_IN_PROGRESS` and do not replace the eventual original result. Different
payloads receive the entity-rejected code and `IDEMPOTENCY_CONFLICT` immediately.
Completed business rejections are retained; retryable pre-commit infrastructure failures release
the reservation. Completed outcomes remain for 24 hours while the process survives.

**Rationale**: Per-product registries would permit reuse of one caller/key against two products.
Replay checks current authorization, then returns the original immutable status/code/body before
checking the now-current amendment version; each HTTP response gets a fresh correlation header.
Order version 1 includes its initial fills. Later commands increment each changed order once;
failed/no-op/replayed commands do not. Amendments require `expectedVersion`; cancellations do not.

**UTC policy**: Inject `Clock`; expire prior-day remainders as a SYSTEM transaction before each
book-dependent read/write. A periodic sweep accelerates cleanup but does not establish correctness.
Check the date before apply and again at the final prepublication linearization point; undo any
unpublished applied delta if the date changed. If staging crossed midnight,
discard the unpublished delta, expire, and restage. New orders get the acceptance time of the
successful attempt; existing orders retain entry time and encounter expiry/version rules.

**Alternatives considered**: Client clocks; scheduler-only expiry; retrying with a new key;
checking version before replay; timestamp precision as the sole priority.

## R5. Exact values and bounded input

**Decision**: `BigDecimal` from validated strings; normalize book keys for numerical comparison.
Tick rounding is `price.divide(tick, 0, HALF_UP).multiply(tick)`. Do not use binary floating point,
`BigDecimal(double)`, or scale-only rounding for non-power-of-ten ticks. Midpoint divides an exact
sum by 2 without rounding to a tick. String serialization uses plain notation and no grouping.
Whole-share quantities use `long` with checked arithmetic and published limits.

**Rationale**: Numerical equality, tick normalization, and midpoint precision must be independent
of input string scale and client number parsing. Data-model and contract documents give concrete
bounds, zero/absence semantics, and response formats.

**Alternatives considered**: `double`; JSON numbers for decimals (user chose strings);
integer cents (cannot express arbitrary ticks and half-tick midpoints).

Source: [JDK 21 BigDecimal](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/math/BigDecimal.html).

## R6. Access control and common errors

**Decision**: One MVC application serves `/internal` administration, root-level participant resources,
and public `/status`; there is no participant API path prefix.
Use stateless HTTP Basic over TLS, with independently generated administrator bootstrap secret
and per-Broker credentials created through the administrator API. Hash passwords with Spring
Security's BCrypt encoder, cost 10; require 16–64 ASCII password characters, unique usernames,
and no generated/default production credentials. Username-to-Broker binding is server-owned.
The quickstart is loopback-only; TLS termination is mandatory for network deployment.

**Rationale**: This supplies a complete identity path without an external identity service or
JWT issuance subsystem. Recheck role, active identity, credential generation, and ownership under the same metadata gate
before a command commits, a private query copies its result, or a cached outcome is replayed. Disable browser sessions, form login,
CORS by default, and CSRF only for these stateless non-browser API chains; reject browser cross-origin
use. No authentication-result cache may keep a deleted Broker authorized. The bounded successful-auth
cache and same-boundary generation checks are specified in `contracts/api.md`; this is custom
authentication-result caching, not merely Spring UserDetails caching.

Authentication entry point, access-denied handler, routing/media/parser errors, controller advice,
and the default error route share one response writer. Handled business rejections return HTTP 200 with `success=false`; request/protocol, security,
admission, and execution failures retain their documented 4xx/5xx. Every error retains numeric
`responseCode`, precise `data.reasonCode`, and `X-Correlation-ID`; set `Cache-Control: no-store`. Generic HTTP Basic
challenges retain `WWW-Authenticate`. Restrict Actuator to an operator/loopback management interface;
its endpoints do not replace public `/status` or expose a second public response format.

**Alternatives considered**: Trusting participant IDs (not authentication); credentials in source;
OIDC/JWT (valid later, unnecessary external dependency for this simulation).

Sources: [Spring Security request authorization](https://docs.spring.io/spring-security/reference/servlet/authorization/authorize-http-requests.html),
[adaptive password verification guidance](https://docs.spring.io/spring-security/reference/features/authentication/password-storage.html).

## R7. Capacity, performance, and validation

**Decision**: Fixed bounded MVC/admission concurrency, timed locks, count and weighted-memory
limits, record reservations, and explicit overload responses. Reserve lifecycle capacity at
order acceptance for eventual cancellation/expiry; never evict promised lifetime history or
unexpired outcomes to make room. Read paths copy bounded immutable DTOs under locks, then encode
outside. Cross-product trade lists acquire relevant product locks in sorted ID order for one
committed view. Detailed defaults are in `contracts/api.md`.

**Rationale**: Unbounded queues and copy-all-history storage would defeat the high-invocation
requirement. Limit breaches reject a complete command before effects; never truncate matching.
JDK Flight Recorder and Micrometer measure contention/allocation; JUnit seeded reference-model
and barrier-based race tests verify behavior. The specified 1,000 requests/s target requires
measurement; the locking design alone is not performance evidence.

**Alternatives considered**: Unlimited virtual threads (do not remove contention/capacity limits);
request-only latency benchmarks; rejecting quickly and counting those responses as throughput.

Source: [Boot metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html).

## Research disposition

All planning decisions are fixed for this phase. Future persistent storage, horizontal replicas,
real assets, external identity providers, and dependency upgrades are outside this implementation.
A persistent adapter must pass the same port/HTTP contract suite before being advertised.

## R8. User-directed HTTP business-result convention

**Decision**: Completed business failures use HTTP 200 with `success=false`, the unchanged
application numeric code and explicit reason. Request syntax, media, routing, authentication,
authorization, rate/admission, and system failures retain 4xx/5xx; accepted creation retains 201.
This user direction supersedes the initial design's 404/409/422 business mappings and the earlier
constitution policy (now amended to 2.0.0). OpenAPI models both business success and rejection at 200.

**Rationale**: The application's client contract treats business decisions as completed HTTP
handling. Typed clients must inspect the envelope, and performance/observability count rejected
business work separately even at HTTP 200. Entity misses on known routes differ from unknown routes.

**Alternatives considered**: HTTP 4xx business outcomes (initial design, superseded by explicit
user instruction); converting authentication, malformed requests, or internal errors to 200
(not requested; these remain request/execution failures).
