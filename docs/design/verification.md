# Verification

This document defines the evidence required to show that an implementation satisfies the [product and protocol contract](contract.md), preserves the necessary [architectural invariants](architecture.md), and operates under the conditions in the [environment](environment.md). Tests prove the published design; implementation behavior does not create requirements.

## Verification objective

The acceptance objective is 100% contract coverage. Every material public guarantee, boundary, qualification, and lifecycle property in `contract.md` must be linked to adequate automated evidence.

A traceability record accompanies the test suite. For each contract statement, it identifies:

- the authoritative section and exact property being proved;
- the use case, preconditions, and relevant variants;
- the observable result or supporting invariant;
- the automated test or tests that provide the evidence;
- the PouchDB, Node.js, adapter, host, and configuration under which the evidence was produced.

One test may support several properties, and one property may require several tests. A reference to a test without an assertion that distinguishes the required behavior is not coverage. A contract change leaves the affected traceability entries incomplete until their proof obligations and evidence are reviewed.

Architectural and environmental tests are included when an internal invariant or integration condition is necessary to sustain a contract property. They do not create additional product requirements.

Line, branch, and statement coverage may reveal untested implementation paths. These metrics are diagnostic only; no percentage of implementation coverage substitutes for contract coverage.

## Evidence model

Verification has two complementary layers.

The primary layer is an end-to-end synchronization suite using the real PouchDB replicator against the router. Fixtures are created and inspected through direct local PouchDB handles. Only the synchronization dialogue under test traverses HTTP, so preparation and cleanup do not depend on routes outside the V1 surface.

The second layer consists of targeted HTTP, integration, and lifecycle tests. These tests prove statuses, protocol paths, ordering, resource ownership, and failure behavior that a successful synchronization could hide. In particular, a green replication does not prove that `_bulk_get` was used, that its fallback works, or that a lifecycle boundary was respected.

Tests from upstream PouchDB are inputs for identifying scenarios and risks. They are not an additional conformance gate and are not copied wholesale: their fixtures and the `minimumForPouchDB` profile exercise an HTTP surface broader than this router's contract.

## End-to-end synchronization proof

End-to-end scenarios use equivalent PouchDB versions, adapters, options, and initial states on the HTTP and direct paths. Where PouchDB fidelity is the property under test, the router result is compared with direct PouchDB synchronization for replicated content, revisions, conflicts, checkpoints, and relevant completion or error outcomes. The comparison is made within the contract's request-size and application-policy qualifications.

The suite covers at least these use cases:

| Use case | Required evidence |
| --- | --- |
| initial push | the target reaches the same relevant state and replication outcome as direct PouchDB replication |
| initial pull | the local node reaches the same relevant state and replication outcome as direct PouchDB replication |
| bidirectional synchronization | both directions preserve the expected revisions, conflicts, and checkpoints |
| topology beyond two nodes | pairwise synchronization composes without requiring protocol capabilities beyond the published surface |
| live replication | changes made after startup propagate through repeated longpoll requests, with heartbeat, timeout, and cancellation behavior exercised |
| conflicts | conflicting revision branches and their reported outcomes match the equivalent direct synchronization |
| attachments | document and design-document attachments retain their binary content, including attachment identifiers containing `/` |
| filters | `doc_ids`, selector, named, and `_view` filters select the expected changes. Named and `_view` cases must demonstrate source-side evaluation through the local design document without a remote `_view` request; function filters remain a client-side PouchDB behavior |
| checkpoints | source, target, and disabled-checkpoint configurations use the expected local-document behavior and resume consistently |

The suite also observes the HTTP dialogue required to establish which protocol path produced the result. Assertions made only against final database state are insufficient when another route or fallback could have produced the same state.

## Targeted proof obligations

Targeted evidence complements the end-to-end suite as follows.

| Proof area | Required evidence |
| --- | --- |
| V1 route surface | Exercise all thirteen route forms, both database trailing-slash forms, reserved `_local` and `_design` precedence, method distinctions, and the PouchDB adapter's database-URL fallback when `GET /` is unavailable. Representative excluded routes must not be classified as V1 synchronization operations; no response status is implied where the contract specifies none. |
| `_bulk_get` and fallback | Demonstrate that a functional bulk-read path actually issues `_bulk_get` without entering the document shim. Separately force its failure and demonstrate fallback through both ordinary and design-document reads. The PouchDB client's per-base-URL memory of `_bulk_get` support must be isolated or made explicit. |
| Path encoding | Exercise encoded database, document, design-document, local-document, and attachment identifiers. Assert that encoded identifier content does not alter route structure and that attachment identifiers preserve all `/` segments. |
| Database existence and creation | Verify absent, pre-existing, newly created, and indeterminate databases; the `GET`/404 → `PUT`/201 setup; repeated creation yielding 412; non-creation routes returning 404 for known absence; storage errors not being converted to absence; and `skip_setup` not creating a database. Direct HTTP assertions are required because a client API may mask a 404. |
| Concurrent creation | Show that two same-name creations handled by one router instance cannot both report 201 and that a dependent read waits for the in-progress result. Separate router instances must not be treated as sharing this coordination. |
| Hooks | Cover declared order, asynchronous hooks, parsing before `before`, short-circuiting before PouchDB access, shared request state, transformed final targets, stable operation identities and parameter vocabulary, phase-dependent context fields, normalized expected PouchDB errors, and application policy applied to the final transformed target. |
| Semantic responses | Exercise every public body form—JSON object, array, `null`, `Buffer`, string, and `undefined`—and both direct response mutation and complete replacement across ordered `after` hooks. |
| HTTP commitment | Prove that `committed` is read-only, that transformation is unrestricted before commitment, and that status, header, and payload restrictions apply after commitment. An incompatible late transformation must be rejected. A representative failure after commitment must terminate the transport rather than emit a replacement status; tests must not require `after` after an unrecoverable transport failure or client disconnection. |
| Request-size boundary | Verify the 64 MiB default, a valid configured limit, rejection of invalid configuration, successful parsing above any lower parser default, and HTTP 413 on overflow. Overflow must occur before hooks and PouchDB access. |
| Handle lifecycle | With an instrumented PouchDB constructor, prove per-database handle reuse, invalidation on `closed` and `destroyed`, reopening after invalidation, preservation of a replacement against stale events, and no invalidation for an ordinary operation error. |
| Long-lived `_changes` work | Verify longpoll heartbeat and timeout behavior, cancellation on client disconnection, and release of the feed and associated timers. |
| `router.close()` | Verify refusal of new work after shutdown begins, cancellation of active `_changes`, completion of already-engaged non-`_changes` operations, closure of router-owned handles, and resource release before the returned promise resolves. Also verify that databases, the HTTP server, and application-owned handles remain outside this lifecycle. |
| Host integration | Run the handler under native `node:http` and by direct Express mounting, including a non-root mount prefix. Verify operation with native request and response behavior and an unconsumed request body stream. |

Negative tests are required where an accidental success would broaden the surface, create an absent database, bypass a lifecycle boundary, or conceal a transport failure. They prove the published boundary; they do not establish statuses or compatibility that the contract deliberately leaves unspecified.

## Qualified evidence

Encoded database names remain an explicit compatibility qualification. Automated cases must use pre-existing databases with encoded names and record the correspondence among the request URL, decoded logical name passed to `databaseExists`, and PouchDB handle opened. These cases characterize and verify supported mappings but cannot establish a universal re-encoding rule that the contract does not define.

Each evidence run declares its Node.js, PouchDB, adapter, and host versions. Results support the declared matrix only; they must not be generalized into runtime or storage compatibility claims absent from the authoritative design.

## Functional evidence production

The complete functional suite is the correctness gate. Test runs must preserve failures rather than retry them invisibly and must retain enough configuration and version information to reproduce the result.

A local quick run may validate the harness, but it is not conformance evidence. Upstream audits, experiments, and isolated prototypes may explain or refine a test case; they do not replace a passing test against the implementation under review.

## Benchmark verification

Benchmarking is separate from functional conformance. It produces comparative performance evidence and establishes no latency, throughput, or resource guarantee.

The benchmark uses the real PouchDB replicator and three targets:

| Target | Configuration |
| --- | --- |
| A | `pouchdb-express-router` under Express |
| B | the new router under Express |
| C | the same new handler under native `node:http` |

The initial workloads are initial pull replication, initial push replication, and incremental bidirectional synchronization with checkpoints. Live replication, filters, conflicts, and attachments remain functional cases unless a specific performance question justifies adding them. Workload volumes are calibrated for stable measurement without excessive duration and are published with the results; the design does not prescribe fixed volumes.

### Comparative interpretation

| Comparison | Interpretation |
| --- | --- |
| A→B | change from the previous router to the new router in an Express environment; for revision-reading workloads this may include the protocol benefit of `_bulk_get` rather than only handler overhead |
| B→C | cost of the Express envelope around the same new handler |
| A→C | end-to-end system difference between the previous Express router and the new native handler |

HTTP call counts and volumes accompany durations, especially `_bulk_get` calls and document fallback reads. A→B must not be attributed solely to implementation overhead when the protocol paths differ.

### Benchmark conditions

For each workload:

- use the same PouchDB client, versions, options, seed, and equivalent initial database states for A, B, and C;
- prepare fixtures directly through local PouchDB handles;
- time from replication engagement to completion, excluding fixture preparation, assertions, and cleanup;
- run only one target at a time and prevent contamination between targets;
- reset PouchDB state between measurements;
- warm all targets symmetrically and declare the cache regime;
- state whether the PouchDB client's initial `_bulk_get` capability detection is measured or already warmed, and never mix those regimes in one comparison;
- run the workload once in each order `ABC`, `ACB`, `BAC`, `BCA`, `CAB`, and `CBA`, yielding eighteen target measurements per workload;
- repeat the complete balanced block when noise requires more observations, without retries that hide failures;
- retain raw measurements and publish the median and dispersion;
- record the implementation SHAs, dependency versions, seed, environment, workload volumes, and relevant HTTP call counts.

A functional difference that prevents the same workload from running on all three targets is resolved or reported before performance is interpreted; it is not hidden behind target-specific workloads. The reference campaign is a dedicated reproducible action. Functional CI establishes correctness independently of it.
