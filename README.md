# pouchdb-http-router

`pouchdb-http-router` exposes the HTTP surface required by the PouchDB replicator to connect and synchronize two or more PouchDB nodes. It is not a general-purpose remote database API nor a CouchDB-compatible server.

## Why

PouchDB replication depends on a precise HTTP dialogue for database setup, changes feeds, revision diffing, bulk revision transfer, attachments, and local checkpoint documents. The router provides that synchronization surface while leaving unrelated database operations local to each PouchDB node.

## Design

The contract is synchronization-first and deliberately narrow. The design uses a native `node:http` handler that can also be mounted directly in Express, an opaque PouchDB storage boundary supplied by the host application, explicit lifecycle management for PouchDB database handles, and semantic `before` and `after` hooks.

Fidelity to the PouchDB replicator, bounded complexity, and end-to-end synchronization evidence take precedence over broad CouchDB API coverage. The synchronization surface is not an access-control boundary: authentication and application policy remain responsibilities of the host environment.

## Status

The reference design is published, but implementation and verification are still ongoing. The project remains under active development.

## Documentation

The authoritative design corpus is published in [`docs/design/`](docs/design/). It separates the contract, architecture, execution environment, and design decisions.

## License

Licensed under the [MIT License](LICENSE.md).
