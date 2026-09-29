# Architecture

`pouchdb-http-router` is organized around a single native HTTP handler, a declarative synchronization route set, a semantic request pipeline, and router-owned PouchDB resources. The public guarantees and boundaries that this structure realizes are defined in the [product and protocol contract](contract.md).

## HTTP execution boundary

The central execution boundary is a handler with the native Node signature `(req, res)`. It depends only on the request and response capabilities needed from `node:http`:

- `req.method`, `req.url`, `req.headers`, and the readable request stream;
- `res.statusCode`, `res.setHeader()`, `res.headersSent`, `res.write()`, and `res.end()`.

The same handler is passed directly to `http.createServer()` or mounted under Express. It does not use Express request decoration or response helpers, and no Express-specific wrapper participates in request processing. When mounted, the handler consumes the URL relative to its mount point; global dispatch and mount-prefix ownership remain outside the router.

The router does not introduce a framework-neutral request abstraction. Native request and response objects remain the transport boundary, while routing, hook context, PouchDB operations, and semantic responses remain router concepts.

## Route model

The V1 synchronization surface is represented internally by route declarations. Each declaration contains the semantic operation, the `sync` capability, HTTP method, path pattern, body mode (`none`, `json`, or `raw`), and operation handler. Matchers are compiled once when the router is initialized.

Route matching operates on the raw pathname before identifier decoding. Only captured parameters are decoded. Reserved `_local` and `_design` forms are matched before generic document and attachment forms, and attachment identifiers retain every path segment. These invariants prevent encoded identifier content from changing the route structure.

Transport spelling and semantic operation identity are separate. Multiple method or path forms may map to the same immutable operation, so hooks and operation handlers do not depend on a particular route pattern.

## Request pipeline

Each accepted request follows one semantic pipeline:

1. parse the request URL and select a precompiled route declaration;
2. derive the semantic operation, database name, and named parameters;
3. parse the query and, when declared, read and parse the request body under the configured limit;
4. create the request context and run `before` hooks in order;
5. unless a hook short-circuits the operation, resolve the PouchDB handle for the final database target;
6. execute the route's PouchDB operation;
7. normalize the result or expected PouchDB error into a semantic response;
8. run `after` hooks in order;
9. emit the remaining status, headers, and payload through the native response.

Parsing completes before `before`; handle resolution occurs afterward. A transformed database or targeting parameter therefore affects the PouchDB operation actually selected, while a short circuit avoids PouchDB access for that request. Long-lived `_changes` handling may emit a heartbeat and commit status and headers before its final semantic response is available; the response phase preserves that irreversible transport state.

## Request context and semantic response

The router creates one request context and owns it for the full pipeline. Hooks share that context. `operation` remains immutable; `database`, `params`, `query`, and `body` may be transformed before PouchDB access; `state` carries request-scoped hook state. Values observed after the operation are the values actually used.

The PouchDB handle and semantic response enter the context only when the corresponding processing has occurred. A `before` short circuit leaves the handle absent. Expected PouchDB errors are converted to semantic responses before `after`, so extension behavior is expressed against one response model rather than internal exceptions.

The semantic response remains distinct from the native response. The router tracks HTTP commitment separately through read-only `committed` state; replacing the semantic response cannot reverse status or headers already emitted. Changes after commitment are limited to compatible payload that has not yet been written. If no faithful HTTP response remains possible after a late failure, the router terminates the transport instead of manufacturing another status.

The context exposes neither the native response object nor the internal route declaration.

## Storage and database handles

Storage is opaque to the router. The application supplies the PouchDB constructor or preset and authoritative logical-existence predicate defined by the contract. The router does not inspect adapter-specific storage, derive filesystem paths, or maintain a catalog of every database.

Database resolution uses the final database name produced by `before`. When no reusable handle exists, existence is established before constructing PouchDB. Lookup and creation remain distinct: lookup does not construct a database known to be absent, while creation waits until the new handle is usable before exposing success.

Each router instance owns a cache containing at most one reusable PouchDB handle for each logical database name. Handles created and retained by the router are reused across requests. There is no LRU, TTL, pool, or process-wide handle registry.

A cached handle is removed when that handle reports `closed` or `destroyed`. Invalidation removes only the affected handle, so a stale lifecycle event cannot discard a replacement. Ordinary operation errors do not invalidate the cache. PouchDB handles created by the application remain outside this ownership boundary, including lifecycle changes that are not reported on the router's handle.

## Creation coordination

Concurrent creation is coordinated per database name within one router instance. A single in-flight creation record serializes the attempt: competing creation requests wait rather than starting another construction, and a read whose existence result depends on that attempt waits for it to settle. The coordination record is cleared when the attempt completes.

This coordination does not provide a global lock. Other processes and external creators remain outside the router's coordination boundary.

## Long-lived changes work

The `_changes` operation owns its PouchDB changes feed, heartbeat, timeout, and cancellation resources for the lifetime of the request. Client disconnection cancels the feed, clears the associated timers, and releases the request resources.

The router tracks active `_changes` work separately from ordinary in-flight operations. Shutdown cancels the former and waits for the latter.

## Router lifecycle

The router instance owns the route matchers, request bookkeeping, cached PouchDB handles, active `_changes` resources, and shutdown coordination created for that instance.

When `router.close()` begins, a lifecycle guard prevents new requests from entering the pipeline. Shutdown then cancels active `_changes` work, waits for engaged non-`_changes` operations, and closes router-owned PouchDB handles. The close operation completes only after those resources have been released.

Database destruction, HTTP server shutdown, and application-owned PouchDB handles are not part of this lifecycle. Removing the handler or stopping its server remains separate from releasing router-owned resources.

## Ownership boundaries

The router owns synchronization route matching, request parsing, hook orchestration, database-handle resolution, PouchDB operation execution, semantic response normalization, native HTTP emission, long-lived `_changes` resources, and its own shutdown.

It does not own application-wide dispatch, the HTTP server, storage configuration beyond the supplied PouchDB constructor or preset, authentication and other perimeter policy, application-owned handles, cross-process database coordination, or protocol capabilities outside the synchronization contract.
