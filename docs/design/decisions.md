# Design decisions

This document records the rationale behind the published [contract](contract.md), [architecture](architecture.md), [environment](environment.md), and [verification strategy](verification.md). Those documents define the current design. This document explains why its material boundaries exist and does not replace them as a source of requirements.

## Native Node HTTP boundary

The router uses native `node:http` request and response objects rather than Web `Request`/`Response` or a framework-neutral HTTP abstraction.

Server-side PouchDB already places the supported runtime in Node.js. Staying at that boundary avoids conversion layers and allows one handler to serve both native Node and Express. The tradeoff is an intentionally Node-centered execution envelope; other runtime models are not gained merely by introducing an abstraction. This choice should be revisited only if the supported PouchDB ecosystem or a demonstrated integration need changes that premise.

## Direct Express mounting

Express mounts the native handler directly; there is no Express-specific adapter.

Express already exposes the required Node request, response, and stream behavior, so a wrapper would add an integration surface without adding a capability. The host consequently retains responsibility for mount prefixes, global dispatch, body-stream preservation, and server lifecycle. A dedicated wrapper remains unjustified until a concrete Express integration requirement cannot be met by direct mounting.

## Synchronization-first scope

V1 exposes the HTTP capabilities required for PouchDB synchronization rather than a broad remote PouchDB API.

Auditing the replicator yields a materially smaller surface than exposing document management, views, maintenance, administration, and plugin APIs. Operations that do not require transport remain local to each PouchDB node, reducing the code and authority exposed over HTTP. The tradeoff is deliberate narrowness: incidental direct use of a synchronization route does not turn it into a general remote database contract, and any additional capability requires a separate decision.

## PouchDB protocol compatibility, not CouchDB emulation

The router implements the CouchDB-derived protocol elements on which PouchDB synchronization depends, not general CouchDB-server compatibility.

The product target is the PouchDB replicator and its observable synchronization behavior. Extending compatibility beyond that need would import unrelated administrative and query surfaces without strengthening the core use case. The tradeoff is that CouchDB clients or tooling cannot infer support outside the explicitly published synchronization surface.

## Fidelity within an explicit responsibility boundary

Synchronization fidelity is judged against equivalent direct PouchDB synchronization, while HTTP limits, host limits, and trusted application policy remain explicit qualifications.

This makes the relevant outcome—revisions, conflicts, checkpoints, and replication issues—the standard rather than treating plausible HTTP statuses as sufficient. Unconditional equivalence was rejected because a configured body limit or application refusal can legitimately prevent an operation that direct in-process synchronization would accept. The resulting guarantee is strong within the router's responsibility and intentionally bounded outside it.

## `_bulk_get` with the real client fallback

V1 includes `_bulk_get` and also preserves the PouchDB HTTP adapter's document-read fallback.

The bulk path avoids materially expensive revision-by-revision reads. The fallback is not merely historical compatibility: the current client records a bulk-read failure by database URL and subsequently uses ordinary and design-document reads. Supporting only the successful bulk path would therefore omit a real client behavior. The tradeoff is two narrowly scoped document-read forms that slightly widen the HTTP surface without establishing a general document API.

## Semantic operations for extensions

Hooks observe stable semantic operations rather than route patterns or legacy route names.

Several HTTP forms can represent the same operation, and transport spelling may change without changing an application's policy intent. A semantic identity keeps authorization, instrumentation, and transformation independent from those spellings. The tradeoff is that an operation intentionally carries less transport detail: direction labels such as push or pull cannot be inferred from one request, while the raw HTTP method remains available on the request when genuinely needed.

## Generic ordered hooks

V1 uses ordered `before` and `after` hooks rather than specialized hook families, route-filter DSLs, or `skip*` controls.

Two generic phases cover authorization, validation, instrumentation, and response transformation without multiplying extension APIs. A single request context allows hooks to cooperate, while semantic operation identity prevents coupling to routes. The mutable semantic response remains separate from read-only `committed` transport state because a replaceable object cannot reverse headers already emitted by a heartbeat.

The tradeoff is that hooks are trusted in-process code whose order and mutations matter. Once transport state is committed, `after` cannot offer the same transformation freedom as it can before commitment.

## Parsing before `before`

Request bodies are parsed before `before` hooks, while PouchDB access remains after them.

This ordering lets application policy inspect and transform the semantic request, then refuse it before the router resolves a database handle or runs a PouchDB operation. The cost is that semantic authorization cannot avoid body-reading and parsing work. Authentication that must reject earlier belongs at the host perimeter, and the router does not add a generic incoming-stream hook API.

## Finite configurable body limit

Parsed request bodies have a finite, configurable `bodyLimit`, with a 64 MiB default.

Materializing JSON and raw bodies before hooks requires a resource boundary. No limit would leave memory use unbounded, while small generic parser defaults can reject normal replication batches; CouchDB's much larger request ceiling addresses a different operating model. The chosen default is a revisable product compromise, not a PouchDB standard.

The tradeoff is explicit: a valid large batch or base64 attachment may require a higher setting, increasing memory exposure under concurrency, while a lower host or proxy limit may still prevail.

## Reusable per-database handles

Each router instance reuses one PouchDB handle per logical database name rather than constructing a handle for every request.

PouchDB and its LevelDB adapter maintain connections and listeners whose lifecycle extends beyond one operation. Reuse avoids repeated allocation and preserves the adapter's connection-sharing model. It also creates router-owned state that must be invalidated and closed deliberately. LRU eviction, TTLs, pools, and process-wide registries were not added because no established requirement justifies their complexity.

## Explicit handle and router lifecycle

Cached handles react to relevant PouchDB lifecycle events, and `router.close()` explicitly releases resources owned by the router instance.

Process-lifetime ownership is insufficient when a handler can be removed while its process continues: cached handles and long-lived changes feeds would otherwise outlive their owner. Explicit shutdown also distinguishes router resources from the HTTP server, databases, and application-owned handles.

The tradeoff is coordinated lifecycle work for the host. `router.close()` does not replace server shutdown, and events not observed on the router's own handle cannot be inferred reliably. The earlier position that V1 needed no public close operation is superseded.

## Application-provided database existence

The application provides `databaseExists(name)` alongside its configured PouchDB constructor or preset.

Opening PouchDB merely to test existence can create the database and falsify the HTTP distinction between lookup and creation. Conversely, a registry containing only router-opened handles cannot know about pre-existing or externally created databases. The application is the authority for the logical namespace it exposes, while the router coordinates creation only among requests handled by its own instance.

Storage inspection and a universal database catalog were rejected because the router is adapter-agnostic. A broader `acquireDatabase(name, { create })` abstraction was also rejected: constructor or preset plus an existence predicate defines the required boundary without forcing applications to supply all handles. The tradeoff is that external creation races and inconsistent namespace knowledge remain application responsibilities.

## No `multipart/related` in V1

V1 uses the JSON/base64 and binary attachment paths exercised by the current PouchDB synchronization client and does not include `multipart/related`.

No current synchronization need justifies the multipart parser, temporary-file handling, and additional event lifecycle it would introduce. This is a falsifiable boundary rather than a permanent claim about every future client: a demonstrated synchronization requirement would justify reopening it.

## Contract-driven verification

Verification starts from the published contract and uses a dedicated end-to-end suite with the real PouchDB replicator, complemented by targeted protocol and lifecycle tests.

Upstream PouchDB tests are broader than the router's synchronization contract and commonly prepare or clean fixtures through remote APIs outside V1. Treating them as the authoritative gate would either widen the product surface or require costly adaptation while still allowing important protocol paths to remain hidden. A successful replication can, for example, conceal a failed `_bulk_get` path through fallback reads.

The tradeoff is maintaining a focused suite and explicit contract traceability. Upstream tests remain useful sources of scenarios and risks, not a second authority.

## Decomposed performance comparison

Performance evidence uses three targets: the previous Express router, the new router under Express, and the same new handler under native `node:http`.

This decomposition separates the effect of replacing the router from the effect of removing the Express envelope. A single old-versus-new comparison would confound both, while revision-reading workloads may also include the protocol benefit of `_bulk_get`. The tradeoff is a more demanding benchmark campaign, but one whose comparisons have interpretable causes. It establishes comparative evidence, not a quantitative product guarantee.

## Unresolved encoded-name qualification

The mapping between decoded database names, application-provided logical existence, and pre-existing physical stores whose names contain encoded characters remains unresolved.

The previous Express router re-encoded the route parameter before opening PouchDB, while the new boundary supplies the decoded logical name to application policy and existence checks. The design establishes neither a universal re-encoding rule nor compatibility with every historical storage layout. This is an explicit qualification to verify, not a rejected alternative or a decision that may be inferred from the route design.
