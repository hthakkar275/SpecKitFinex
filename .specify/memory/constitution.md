# SpecKitFinex Constitution

## Core Principles

### I. Explicit REST and Error Contracts

- The application MUST expose its exchange capabilities through documented HTTP REST API
  endpoints. Each endpoint MUST define request and response schemas, validation rules,
  HTTP methods, success statuses, and failure outcomes in a machine-readable API contract.
- A completed application-level decision MUST return HTTP 200 even when its business outcome
  is a rejection, missing entity, unsupported entity, validation failure, or business conflict.
  Its envelope MUST set `success=false` and provide the stable application-specific result
  code, explicit reason, safe message, and correlation identifier. Accepted business outcomes
  MUST set `success=true`; successful creation MAY use HTTP 201 as defined by the API contract.
- HTTP 4xx/5xx MUST be reserved for request/protocol, authentication/authorization, routing,
  request-size/media, throttling/admission, and infrastructure/unexpected execution failures.
  Their envelopes MUST also set `success=false` and provide documented codes/reasons. Semantic
  business validation MUST NOT accidentally become HTTP 400/409/422 through framework defaults.
- Application result codes MUST distinguish actionable failure reasons without requiring clients
  to parse message text; their meanings MUST be documented and MUST NOT be silently reassigned.
  Clients, monitoring, and performance reports MUST inspect `success` and the result code;
  HTTP 200 alone MUST NOT establish business success or count rejected work as useful throughput.
- Contracts MUST document retry behavior and compatibility policy. Breaking changes MUST use
  an explicit versioning and migration strategy. Deployment components that generate HTTP
  errors MUST follow the error contract where configurable; transport failures with no HTTP
  response MUST be treated as an unknown outcome by clients.

These rules separate completed business decisions from request/execution failures and let clients
handle both predictably, including failures before endpoint logic runs.

### II. Financial Correctness and Concurrency Safety

- Every state-changing operation MUST define its financial invariants and atomic boundary.
  Related changes to orders, trades, holdings, reservations, or balances, where present, MUST
  either commit together or leave no partial business effects. Prices, quantities, and monetary
  calculations MUST use exact representations with explicit precision and rounding rules;
  binary floating-point MUST NOT represent financial values.
- Conflicting operations MUST have a defined serialization order. Synchronization or
  transactional isolation MUST preserve invariants across concurrent requests, background
  work, and every supported application instance. The deployment topology MUST NOT exceed
  the guarantees of its concurrency mechanism.
- Reads MUST obey documented consistency guarantees and MUST NOT expose partially applied
  operations or mutable internal state. Matching or prioritization rules, if implemented,
  MUST define deterministic ordering for competing accepted commands.
- Operations that can create financial effects MUST define an idempotency mechanism.
  Concurrent or retried submissions with the same identity and payload MUST NOT duplicate
  those effects; reusing an identity with a different payload MUST produce an explicit error.
  The retention period and guarantees across timeouts and restarts MUST be documented.
- Automated tests MUST exercise conflicting operations, simultaneous retries, and failures
  during state changes, asserting invariants and observable outcomes rather than only the
  absence of runtime race warnings.

Financial correctness is a release requirement and MUST NOT be traded for throughput.

### III. Measurable Performance Under Load

- Every feature affecting API capacity or latency MUST specify numeric acceptance targets
  before implementation: sustained request rate, concurrent clients, p95 and p99 latency,
  acceptable error rate, and resource limits. Targets MUST identify hardware, dataset size,
  request mix, storage adapter, and test duration so results are reproducible.
- API processing MUST bound request sizes, collection responses, queues, waits, and expensive
  work. Overload MUST trigger controlled admission or backpressure and an appropriate HTTP
  status with an explicit application result code, rather than unbounded resource growth.
- Performance validation MUST include representative contention on shared exchange state,
  sustained load, and overload recovery. Reports MUST distinguish successful business work
  from rejected or failed requests so fast failures cannot inflate throughput claims.
- Changes to critical execution paths MUST be checked against the accepted baseline.
  Regressions outside the documented budget MUST be resolved or explicitly approved with
  measured impact and rationale before release. Optimization MUST preserve correctness.

### IV. Replaceable Storage with Consistent Semantics

- Business logic MUST depend on storage interfaces owned by the application or domain layer,
  rather than concrete collections, database clients, or vendor-specific models. HTTP handlers
  MUST NOT implement storage-specific business rules.
- Storage contracts MUST specify atomic operations or transaction boundaries, consistency,
  conflict handling, ordering where relevant, idempotency, and error behavior. Interfaces MUST
  support complete business operations; a generic CRUD abstraction alone MUST NOT force
  callers to assemble unsafe read-modify-write sequences.
- The first proof of concept MAY use an in-memory adapter. Its supported topology, loss of
  data on restart, and limits on idempotency retention MUST be explicit. It MUST provide the
  same required concurrency and atomicity guarantees within that supported scope.
- Persistent adapters MUST be substitutable through composition or configuration without
  changing business rules or public API schemas. Adapter-specific failures MUST map to the
  shared application error model. Each adapter MUST pass the same behavioral contract suite,
  supplemented by tests of its actual concurrency and failure mechanisms.
- Any persistent-storage adoption MUST define migrations, recovery behavior, and durability
  guarantees, including survival of acknowledged operations and idempotency records. A change
  in guarantees MUST be documented and reviewed; adapter replacement MUST NOT silently weaken
  financial invariants.

### V. Verifiable Behavior and Operational Accountability

- Automated checks MUST verify API contracts, application result codes, financial invariants,
  concurrent behavior, idempotency, and storage contracts for the affected functionality.
  Reproduced defects MUST receive regression coverage at the level that demonstrates the fault.
- Each invocation MUST be traceable through its correlation identifier. Structured logs and
  metrics MUST expose latency, throughput, application error codes, contention, and saturation
  without exposing credentials or sensitive financial payloads.
- Financial state changes MUST produce an attributable audit record with operation identity,
  ordering information, and outcome. Recording that audit event MUST share the business
  operation's atomic boundary or use an equivalent reliable mechanism that prevents committed
  effects from losing their audit record. Audit durability MUST match the declared storage mode.
- Test and benchmark reports MUST state their environment and limits. An in-memory proof of
  concept MUST NOT be represented as evidence of persistent durability or multi-instance safety.

## Security and Scope Constraints

- The system's scope is a small electronic financial exchange served through HTTP APIs.
  Architecture MUST favor the simplest design that satisfies these principles; framework,
  language, database, and deployment choices MUST be justified in implementation plans.
- Requests MUST be validated at trust boundaries. Access to account-specific data and financial
  mutations MUST enforce authentication, authorization, and ownership. An isolated proof of
  concept MAY use explicitly configured test identities; it MUST NOT expose that mode as a
  production authentication mechanism.
- Network deployments MUST protect credentials and sensitive traffic in transit. Secrets MUST
  remain outside source control; responses and logs MUST NOT disclose stack traces, database
  internals, credentials, or unnecessary account data.
- The in-memory proof of concept MUST use synthetic funds and data. Handling real assets or
  production customer data MUST require a documented production-readiness review covering
  security, durability, recovery, auditability, and applicable obligations before activation.

## Development Workflow and Quality Gates

1. Feature specifications MUST define endpoint behavior, failure outcomes, financial invariants,
   retry semantics, and acceptance scenarios for concurrency and storage failures where relevant.
2. Implementation plans MUST record storage and synchronization boundaries, supported topology,
   numeric performance targets, and a constitution compliance check before implementation starts.
3. Changes MUST include the affected contract, invariant, concurrency, adapter, and security
   checks. API and storage integration tests MUST use the actual configured adapters; critical
   performance changes MUST include reproducible load-test evidence.
4. Review MUST verify failure statuses and application result codes, atomicity, safe retries,
   bounded resource use, and independence from concrete storage. Required checks MUST pass
   before merging; exceptions MUST follow Governance rather than bypassing a failed gate.
5. Releases MUST document API compatibility changes, storage-mode limitations, operational
   settings, and migration or recovery steps when applicable.

## Governance

This constitution takes precedence over conflicting project plans, templates, and local practices.
Specifications, plans, tasks, implementation reviews, and release reviews MUST check compliance
with the principles relevant to their scope and record the evidence or outstanding violations.

Amendments MUST document the proposed text, rationale, compatibility impact, and migration needs.
A project maintainer MUST approve an amendment before adoption. The adopted document MUST update
its version and amendment date while preserving the original ratification date. Dependent artifacts
MUST be reviewed for alignment; this constitution workflow does not modify them automatically.

Versions MUST follow semantic versioning: MAJOR for incompatible principle removals or
redefinitions, MINOR for new principles or materially expanded guidance, and PATCH for
clarifications without changed obligations. Initial adoption is version 1.0.0.

Exceptions MUST identify the affected rule, justification, risk, compensating controls, owner,
and expiry or removal condition, and receive recorded maintainer approval before the affected
work is merged or released. Exceptions MUST NOT authorize corrupted financial state, duplicate
financial effects within promised idempotency guarantees, or silent weakening of public contracts.
Recurring exceptions MUST trigger a design correction or a proposed constitutional amendment.

**Version**: 2.0.0 | **Ratified**: 2026-11-06 | **Last Amended**: 2026-11-07
