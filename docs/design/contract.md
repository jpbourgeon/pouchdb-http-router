# Product and protocol contract

`pouchdb-http-router` exposes the HTTP surface required by the PouchDB replicator to connect and synchronize two or more PouchDB nodes. Its V1 contract is synchronization only. It is neither a general-purpose remote PouchDB API nor a CouchDB-compatible server.

## Contract principles

### PouchDB fidelity

For equivalent PouchDB storage, options, and initial state, synchronization through the router must produce the same relevant outcomes as direct PouchDB synchronization, including revisions, conflicts, and checkpoints.

This guarantee is bounded by the HTTP interface. Request-size limits, earlier limits imposed by the host or a proxy, and trusted application policy may reject or change an operation that direct synchronization would accept. The router therefore does not promise unconditional equivalence with an in-process synchronization.

Differences from CouchDB outside the synchronization surface are intentional and explicit.

### Synchronization-first scope

A capability belongs to V1 only when it is required for correct or efficient replication by the PouchDB replicator. Bidirectional synchronization consists of two unidirectional replications; connecting more than two nodes does not add protocol primitives.

The surface includes `_bulk_get` because avoiding its document-by-document fallback is materially relevant to replication efficiency. No quantitative performance guarantee is established by this contract.

## V1 HTTP surface

| Capability | Method and path | Contract role |
| --- | --- | --- |
| database information and existence | `GET /:database` | PouchDB setup and source `info()`, including `update_seq` |
| database creation | `PUT /:database` | automatic setup after a missing-database response unless `skip_setup` is used |
| changes feed | `GET /:database/_changes` | one-shot and live replication through longpoll, including heartbeat and timeout behavior |
| filtered changes feed | `POST /:database/_changes` | `doc_ids` and selector filters |
| missing revisions | `POST /:database/_revs_diff` | replication revision comparison |
| bulk revision write | `POST /:database/_bulk_docs` | replicated writes with `new_edits:false` |
| bulk revision read | `POST /:database/_bulk_get` | revision reads with `revs:true` and `latest:true` |
| document fallback read | `GET /:database/:documentId` | PouchDB's fallback after `_bulk_get` fails |
| design-document fallback read | `GET /:database/_design/:designId` | fallback reads for design documents |
| attachment read | `GET /:database/:documentId/:attachmentPath+` | binary attachment retrieval, including attachment names containing `/` |
| design-document attachment read | `GET /:database/_design/:designId/:attachmentPath+` | attachment retrieval from design documents |
| local document read | `GET /:database/_local/:localId` | checkpoint document reads |
| local document write | `PUT /:database/_local/:localId` | checkpoint document writes |

These thirteen route forms are the complete V1 synchronization surface.

The forms `/:database` and `/:database/` identify the same database operations. Reserved `_local` and `_design` forms remain distinct from generic document and attachment paths. Encoded characters inside identifiers do not change the route structure, and attachment paths preserve all of their segments.

No root route is required. When PouchDB cannot obtain an instance UUID through `GET /`, its HTTP adapter falls back to the database URL.

## Replication behavior

### Database setup and creation

`GET /:database` does not request database creation:

- a database known to be absent produces `404 not_found`;
- a database known to exist returns its information.

`PUT /:database` requests database creation:

- a known existing and usable database produces `412`;
- a known absent database produces `201` only after it has become usable.

The remaining synchronization routes do not implicitly create an absent database and produce `404` when its absence is known. Body parsing and trusted application policy may complete or reject a request before this missing-database result is reached. A storage error that cannot be classified is not converted into `404` or `412`.

The normal PouchDB setup sequence may therefore be `GET` → `404` → `PUT` → `201`. If another request wins the creation race, PouchDB may receive and accept `412`. `skip_setup` does not create a database through this sequence.

The public V1 storage boundary consists of:

- `PouchDB: PouchDBConstructor`, the constructor or preset used to open databases;
- `databaseExists: (name: string) => boolean | Promise<boolean>`, the application's logical-existence predicate for the database namespace it exposes.

`databaseExists` does not create a database. It returns `false` only when absence is known. An inability to determine existence is an error and must not be converted into absence.

The router does not discover every database by inspecting storage, maintain a universal database catalog, or compensate for inconsistent application knowledge. It coordinates requests handled by one router instance, but does not establish cross-process or external-creator coordination.

### Bulk reads and fallback reads

`_bulk_get` is part of the guaranteed V1 surface. The two document-read routes preserve the PouchDB HTTP adapter's fallback after a `_bulk_get` failure, including design documents. Their presence does not establish a general document-read API.

### Changes and filters

Live replication uses repeated longpoll requests to `_changes` with heartbeat, timeout, and cancellation behavior expected by PouchDB.

`doc_ids` and selector filters use `POST _changes`. Named filters and `_view` filters are included in V1 and execute against the source database's design document on the server side. They do not add a remote `_view` endpoint. Function filters supplied directly to the replicator continue to execute on the client side.

Checkpoint placement options and disabled checkpoints do not add routes beyond the local-document operations already listed.

### Request hooks

V1 exposes ordered `before` and `after` hooks. Hooks of each kind run sequentially in configuration order and share one request context. The router parses the request body before running `before` hooks. A `before` hook can therefore refuse a parsed request before any PouchDB access for that request, while a parsing failure occurs before the hook.

The context exposes the request, the semantic `operation`, the database name, route parameters, query parameters, parsed body, request-scoped `state`, the PouchDB handle when one was used, the current `response`, and read-only `committed` state. The operation identifier describes PouchDB semantics rather than a route spelling and is one of `database.info`, `database.create`, `changes.read`, `revisions.diff`, `documents.bulkRead`, `documents.bulkWrite`, `attachment.read`, `localDocument.read`, `localDocument.write`, or `document.read`.

A `before` hook may mutate the database name, route parameters, query parameters, body, and `state`. Returning a response stops the remaining `before` hooks and the PouchDB operation; all `after` hooks still run. When a `before` hook changes the database or another target parameter, application authorization must apply to that final target.

An `after` hook observes the final context values and may mutate `state` or replace the current response for subsequent `after` hooks. `committed` reports whether HTTP headers have already been sent. It is read-only: an `after` hook cannot undo committed HTTP state, and any response change must remain compatible with that state.

### Request-size boundary

The public `bodyLimit` option sets the per-request limit for parsed JSON and raw bodies and defaults to `64 MiB`. Exceeding it produces HTTP `413` before application hooks or PouchDB access for that request. Invalid `bodyLimit` configuration is rejected. No lower implicit parser default may silently replace the configured or default contract limit.

This default is not a PouchDB limit and does not guarantee that every valid document or replication batch can pass. A large `_bulk_docs` request or a base64-encoded attachment may require a higher configured limit, at a corresponding memory cost. A lower limit in the host application or reverse proxy may prevail.

## Public lifecycle

`router.close()` begins asynchronous shutdown. Once shutdown begins, the router accepts no new request processing. It terminates or cancels active `_changes` handling as required, while allowing already-engaged non-`_changes` operations to finish.

The router then closes its cached PouchDB handles. Completion of `router.close()` means those router-owned resources have been released. Shutdown does not destroy databases, close the HTTP server, or manage PouchDB handles owned by the application.

## Security boundary

The synchronization surface is not an access-control mechanism. `_changes`, `_bulk_get`, and `_bulk_docs` provide substantial read and write capabilities. Authentication at the application boundary and trusted application policy remain necessary when the network is not trusted.

A direct HTTP call may use any exposed synchronization route, but that incidental accessibility does not expand the product contract.

## Explicitly outside V1

V1 does not guarantee:

- database destruction or remote maintenance, including `DELETE /:database` and `_compact`;
- `_all_docs` as a remote query API;
- document creation, update, or deletion outside replication operations;
- direct attachment writes or deletions;
- remote persistent-view queries, `_temp_view`, or `_view_cleanup`;
- a root welcome endpoint;
- `_session`, CouchDB administration, Fauxton, `_users`, `_replicator`, cluster management, or configuration APIs;
- generic `HEAD` behavior;
- `multipart/related`;
- a public `api` capability, a mode selecting one, or a second HTTP surface;
- CouchDB server compatibility beyond the protocol elements required by PouchDB synchronization.

Operations that do not require transport remain local to each PouchDB node. Any additional capability requires a separate contract decision.

## Unresolved compatibility qualification

The V1 route set and URL parameter semantics are established. Compatibility between decoded database names, application-provided logical existence, and pre-existing physical stores with encoded names still requires verification. No general re-encoding rule or compatibility guarantee for all historical storage layouts is established.
