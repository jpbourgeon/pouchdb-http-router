# Environment

`pouchdb-http-router` is intended to run as a long-lived Node.js server component around server-side PouchDB. This document defines the execution and integration conditions for the [product and protocol contract](contract.md) and [architecture](architecture.md).

## Runtime envelope

Node.js is the supported server runtime. The host must provide the native `node:http` request and response behavior used by the router and a PouchDB constructor or preset suitable for persistent server-side operation.

The supported deployment model is a stateful, long-lived server process. Serverless, FaaS, edge, or Worker environments without suitable persistent server-side storage are outside this envelope. Native Bun, Deno, Web `Request`/`Response`, and Hono integrations are not supported targets. Other Node frameworks may be usable when they preserve the required native HTTP behavior, but that possibility is not a compatibility guarantee.

## HTTP integration

The router handler can be passed directly to `http.createServer()` or mounted directly under Express. Express does not require a router-specific adapter layer.

A host framework or middleware stack must preserve:

- usable native request and response objects;
- an unconsumed readable request body stream when control reaches the router;
- normal connection and disconnection signaling for long-lived `_changes` requests.

Middleware that has irreversibly consumed the request body is incompatible with routes whose bodies the router must read.

The host owns global request dispatch and mount prefixes. Under Express, the router handles the URL relative to the mount point. Under native `node:http`, the application must dispatch requests to the handler with the corresponding relative URL behavior.

## Application policy

The router's synchronization surface is not an authentication boundary. The host application or surrounding infrastructure owns authentication and perimeter policy. Authentication that must reject a request before its body is read must run before the router; application hooks run within the router's request handling under the conditions defined by the contract.

CORS, security headers, rate limiting, and application-level observability also remain host or infrastructure responsibilities.

## PouchDB and storage responsibilities

The application supplies the configured `PouchDB` constructor or preset and the authoritative `databaseExists(name)` predicate defined by the contract. It therefore owns adapter selection, storage location and configuration, and accurate knowledge of the logical database namespace it exposes. The router does not inspect storage to repair incomplete or inconsistent application knowledge.

The router coordinates database creation only among requests handled by one router instance. The application must coordinate access involving other router instances, processes, or external database creators when its storage model requires it. It also owns the lifecycle of PouchDB handles that it creates and any storage changes that are not reported through the router's handles.

## Lifecycle coordination

The host owns the HTTP server and the installation or removal of the router handler. The router owns only its own PouchDB handles and active request resources.

Stopping the server or removing the handler must be coordinated with `router.close()`, especially while long-lived `_changes` requests are active. Closing the HTTP server does not replace router shutdown, and `router.close()` does not close the HTTP server. The host must await the lifecycle operations needed for both ownership domains before treating shutdown as complete.

## Request-size limits

The router enforces its configured `bodyLimit` as defined by the contract. Host frameworks, reverse proxies, and other infrastructure may impose a lower limit and reject a request before it reaches the router. To make the router's configured limit effective, the host must configure upstream limits accordingly.

Choosing a higher router or infrastructure limit is a deployment decision that must account for the memory cost of parsed request bodies and concurrent requests. The router does not override stricter limits imposed outside its boundary.
