# Requirements Traceability: Electronic Financial Exchange Simulator

**Updated**: 2026-11-08
**Source**: [Feature specification](spec.md)

This is a structural refinement of the existing 44 FR paragraphs. Each original ID remains a
stable topic group; its decimal-suffixed children are the individual verification units.
A parent reference covers all of that group's children, including explicit shared-rule references.
The original text remains available through version history rather than being duplicated here.

User stories, priorities, acceptance scenarios, independent-test descriptions, and edge cases are
unchanged. Success criteria, API behavior, result codes, and contract schemas are also unchanged.
`US3.8` means acceptance scenario 8 of User Story 3; `SC-005` refers to the existing success criterion.

## User Story to Requirement Groups

Each group below resolves to its complete atomic-ID range in the next table. This gives both a
readable journey view and an exact inventory for task/test traceability. Shared access, response,
consistency, and storage policies apply wherever relevant even when a row identifies another
journey as their main context.

| Existing user story | Requirement groups |
| --- | --- |
| US1: Administer the Trading Universe | `FR-001`–`FR-002`, `FR-004`–`FR-008`, `FR-038`–`FR-039`, `FR-044` |
| US2: Submit and Match Orders Fairly | `FR-001`, `FR-003`, `FR-009`–`FR-017`, `FR-024`, `FR-029`, `FR-033`, `FR-038`, `FR-042` |
| US3: Trade Reliably Under Concurrent Requests | `FR-002`–`FR-003`, `FR-008`, `FR-016`, `FR-018`, `FR-020`, `FR-029`–`FR-037`, `FR-043` |
| US4: Amend, Cancel, and Expire Open Orders | `FR-017`–`FR-024`, `FR-032` |
| US5: Inspect Orders, Trades, and Market Depth | `FR-003`–`FR-004`, `FR-013`, `FR-016`, `FR-023`–`FR-027`, `FR-033`, `FR-036`, `FR-038`–`FR-039` |
| US6: Integrate Through a Consistent HTTP Contract | `FR-002`, `FR-011`, `FR-021`, `FR-029`, `FR-036`, `FR-038`–`FR-044` |
| US7: Confirm Service Availability | `FR-001`, `FR-028`, `FR-037`–`FR-040`, `FR-044` |

## Original FR to Atomic IDs and Verification Basis

Every child in a row inherits that row's story/verification context. The shared-rule column
identifies clauses consolidated under another canonical ID, avoiding duplicate obligations.
These references are acceptance/inspection anchors, not claims that executable tests already exist
or that every field assertion has its own user-story scenario.

| Original group | Atomic IDs | Shared canonical rules | Existing verification basis |
| --- | --- | --- | --- |
| `FR-001`: API boundaries | `FR-001.1`–`FR-001.2` | `FR-028`, `FR-038` | US1, US2, US7; US1.4, US2.1, US7.1 |
| `FR-002`: Authentication and roles | `FR-002.1`–`FR-002.4` | — | US1, US3, US6; US1.4, US5.2, SC-001, SC-005 |
| `FR-003`: Participant identity and ownership | `FR-003.1`–`FR-003.7` | — | US2, US3, US5; US2.1, US5.2, SC-005 |
| `FR-004`: Private and shared information | `FR-004.1`–`FR-004.6` | — | US1, US5; US1.4, US5.2, US5.3, US5.5 |
| `FR-005`: Product management and records | `FR-005.1`–`FR-005.14` | `FR-006` | US1; US1.1, US1.2, US1.5, US1.7 |
| `FR-006`: Product validation and symbol normalization | `FR-006.1`–`FR-006.9` | — | US1; US1.1, US1.3, US1.7, Failure Outcomes |
| `FR-007`: Broker management and records | `FR-007.1`–`FR-007.13` | — | US1; US1.2, US1.3, US1.5, Story 1 independent lifecycle test |
| `FR-008`: Deletion and reference-data changes | `FR-008.1`–`FR-008.13` | — | US1, US3; US1.5, US1.6, Edge Cases, SC-005 |
| `FR-009`: Order submission and assigned attributes | `FR-009.1`–`FR-009.14` | — | US2; US2.1, US2.8, SC-003 |
| `FR-010`: Order-entry rejection rules | `FR-010.1`–`FR-010.17` | `FR-024` | US2; US2.8, Failure Outcomes, SC-003 |
| `FR-011`: Exact values and decimal encoding | `FR-011.1`–`FR-011.20` | `FR-040.9`, `FR-029` | US2, US6; US2.7, US6.11, SC-011 |
| `FR-012`: Market-order price and longevity | `FR-012.1`–`FR-012.4` | — | US2; US2.5, US2.6, US2.8 |
| `FR-013`: Order-book lifecycle | `FR-013.1`–`FR-013.3` | — | US2, US5; US2.1, US5.5 |
| `FR-014`: Matching eligibility and priority | `FR-014.1`–`FR-014.6` | — | US2; US2.2, US2.3, US2.4, SC-002 |
| `FR-015`: Limit and market price eligibility | `FR-015.1`–`FR-015.3` | `FR-014`, `FR-017` | US2; US2.2, US2.4, US2.5, US2.6 |
| `FR-016`: Individual executions | `FR-016.1`–`FR-016.5` | `FR-032.1` | US2, US3, US5; US2.2, US2.4, US3.1, US5.6, SC-002, SC-004 |
| `FR-017`: Matching completion and remaining quantities | `FR-017.1`–`FR-017.6` | — | US2, US4; US2.4, US2.5, US2.6, US4.5, US4.7 |
| `FR-018`: Order amendments | `FR-018.1`–`FR-018.19` | `FR-009.11`, `FR-009.13`, `FR-011.17`, `FR-029` | US3, US4; US3.8, US4.1, US4.2, US4.3, US4.4, SC-005 |
| `FR-019`: Amendment accounting and rejected changes | `FR-019.1`–`FR-019.3` | `FR-023.11`, `FR-016.4` | US4; US4.3, US4.4, SC-004, SC-005 |
| `FR-020`: Cancellation effects | `FR-020.1`–`FR-020.9` | — | US3, US4; US3.2, US3.9, US4.5, SC-003, SC-004 |
| `FR-021`: Invalid cancellation and amendment states | `FR-021.1`–`FR-021.6` | `FR-018.17`, `FR-030`, `FR-032.10` | US4, US6; US4.6, US6.5, US3.8, Failure Outcomes |
| `FR-022`: UTC day expiry | `FR-022.1`–`FR-022.9` | — | US4; US4.7, SC-009 |
| `FR-023`: Order reports and quantity reconciliation | `FR-023.1`–`FR-023.21` | `FR-012.2`, `FR-019.2` | US4, US5; US5.1, US4.3, SC-004 |
| `FR-024`: Stored order status | `FR-024.1`–`FR-024.2` | `FR-010.15` | US2, US4, US5; SC-003, Story 5 all-status fixture |
| `FR-025`: Trade retrieval and records | `FR-025.1`–`FR-025.11` | `FR-004.2`, `FR-011`, `FR-042` | US5; US5.2, US5.6, SC-004 |
| `FR-026`: Order-book snapshots | `FR-026.1`–`FR-026.9` | — | US5; US5.3, US5.5, SC-004 |
| `FR-027`: Order-book calculations | `FR-027.1`–`FR-027.14` | — | US5; US5.4, US5.5, SC-004 |
| `FR-028`: Service liveness | `FR-028.1`–`FR-028.5` | — | US7; US7.1, US7.2 |
| `FR-029`: Business and technical failure responses | `FR-029.1`–`FR-029.6` | `FR-043.1`, `FR-040`, `FR-044.1` | US2, US3, US6; US2.8, US3.4, US6.2, US6.5, US6.6, US6.7, US6.9, SC-006, SC-012 |
| `FR-030`: Caller-scoped idempotency | `FR-030.1`–`FR-030.8` | `FR-043.3`, `FR-043.4`, `FR-043.5` | US3; US3.3, US3.4, US3.6, US3.9, SC-005 |
| `FR-031`: Failed and uncertain command outcomes | `FR-031.1`–`FR-031.5` | — | US3; US3.5, US3.6, SC-005, SC-006 |
| `FR-032`: Atomic commands and order versions | `FR-032.1`–`FR-032.10` | `FR-018.19`, `FR-017.5` | US3, US4; US3.1, US3.2, US3.3, US3.5, US3.7, US3.8, US3.9, SC-005 |
| `FR-033`: Read consistency and timestamp ties | `FR-033.1`–`FR-033.4` | — | US2, US3, US5; US2.3, US3.7, US5.3, US5.6, SC-004 |
| `FR-034`: Storage mode and replacement | `FR-034.1`–`FR-034.11` | — | US3; SC-010, A-008, Storage-mode inspection |
| `FR-035`: Audit history and operational evidence | `FR-035.1`–`FR-035.14` | — | US3; US3.3, US3.5, US3.6, SC-006, SC-007, SC-008 |
| `FR-036`: Finite limits and bounded queries | `FR-036.1`–`FR-036.10` | — | US3, US5, US6; Edge Cases, SC-007, SC-008, Published-limit inspection |
| `FR-037`: Capacity and overload acceptance | `FR-037.1`–`FR-037.14` | `FR-036` | US3, US7; US7.2, SC-007, SC-008 |
| `FR-038`: JSON resources and URL bases | `FR-038.1`–`FR-038.9` | `FR-002`, `FR-003`, `FR-004` | US1, US2, US5, US6, US7; US6.1, SC-011, API Resource Requirements |
| `FR-039`: Bodyless resource reads | `FR-039.1`–`FR-039.5` | — | US1, US5, US6, US7; US6.1, US7.1, SC-011 |
| `FR-040`: Common JSON envelope | `FR-040.1`–`FR-040.9` | — | US6, US7; US6.2, US6.9, US7.1, SC-011 |
| `FR-041`: Unexpected execution failures | `FR-041.1`–`FR-041.10` | `FR-029.2`, `FR-040`, `FR-044.1` | US6; US6.9, SC-012 |
| `FR-042`: API date and time format | `FR-042.1`–`FR-042.6` | `FR-033.4` | US2, US6; US2.3, US6.8, SC-011 |
| `FR-043`: Operation-specific codes and replay fidelity | `FR-043.1`–`FR-043.5` | `FR-029.1`, `FR-029.2` | US3, US6; US3.6, US6.3, US6.4, US6.5, US6.6, US6.7, US6.10, SC-006, SC-012 |
| `FR-044`: Response headers and successful HTTP outcomes | `FR-044.1`–`FR-044.5` | `FR-040.1`, `FR-029.1` | US1, US6, US7; US6.1, US6.2, US6.9, US7.1, SC-006, SC-011 |

## Verification Rules

- The inventory contains 383 mandatory statements and 2 explicitly retained optional permissions.
- An implementation task or verification result cites the child ID, such as `FR-018.8`.
- Parent-group completion requires a disposition for every child; partial compliance stays visible.
- Shared rules have one canonical ID, even when several journeys exercise them.
- A mandatory statement receives its own pass/fail result; scenarios can share setup and execution.
- A `MAY` statement retains the original freedom of choice; inspection records the supported
  option rather than silently upgrading the permission to a mandatory capability.
- A single atomic-visibility invariant can cover an entire command aggregate. Dividing that
  invariant into unrelated per-record writes would weaken the existing concurrency guarantee.
- Record/schema field requirements can be checked separately within the existing report or
  lifecycle scenarios; no new story or changed business workflow is required.
- Planning/operational obligations use inspection or the existing measurable acceptance profile.
- For HTTP 200 business rejections, tests evaluate `success`, `responseCode`, and `data.reasonCode`;
  HTTP status alone does not establish a successful business outcome.

## Structural Preservation Evidence

The full `User Scenarios & Testing` section, including Edge Cases, was compared before and after
this rewrite and is byte-for-byte identical. Its SHA-256 is
`63e5cb5b8d896d4bad0104639ad2c288f8e6a3a64b1a7d92d204c8645990a014`.

The OpenAPI and API companion were unchanged. The plan and quality checklist were updated only
for atomic-ID interpretation, traceability, and document review metadata. This change does not
implement application code or execute runtime acceptance tests.
