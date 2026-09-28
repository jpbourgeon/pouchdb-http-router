# pouchdb-http-router

`pouchdb-http-router` is a focused HTTP transport for connecting and synchronizing two or more PouchDB nodes. Its intended contract is the protocol surface required by PouchDB replication—not a general-purpose remote database API or a CouchDB replacement.

## Why

PouchDB synchronization depends on a precise HTTP dialogue for database setup, change discovery, revision exchange, document and attachment transfer, and checkpoints. This project aims to provide that dialogue as a thin server-side layer while leaving operations that do not require transport local to each node.

## Design

The design is synchronization-first and deliberately narrow. It targets a native `node:http` handler that can also be mounted directly in Express, an opaque PouchDB storage boundary supplied by the host application, explicit lifecycle management for long-lived database handles, and semantic `before` and `after` hooks.

Protocol fidelity, bounded complexity, and end-to-end synchronization evidence take precedence over broad CouchDB API coverage. The synchronization surface is not an access-control boundary: authentication and application policy remain responsibilities of the host environment.

## Status

The project is under active architectural redesign and development. The target synchronization surface and major design decisions have been established, but the implementation and reference documentation remain to be completed and verified.

## Documentation

The authoritative design corpus is being established in `docs/design/`, with separate documents for the contract, architecture, execution environment, and decisions. Until those documents are instituted, the [current design capture](.42p/cases/design/2026-09-28_pouchdb-http-router_design_capture_edit-0.12.md) is the working source and is explicitly non-normative.

## License

Licensed under the [MIT License](LICENSE.md).
