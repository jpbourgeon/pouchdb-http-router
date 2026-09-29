# Design

This directory is the authoritative published design corpus for `pouchdb-http-router`. It defines the intended contract and system design for exposing the HTTP surface required by the PouchDB replicator to connect and synchronize two or more PouchDB nodes. It does not define a general-purpose remote database API or a CouchDB-compatible server.

## Corpus

| Document | Responsibility |
| --- | --- |
| [Contract](contract.md) | Defines the problem, the guaranteed synchronization surface, expected properties, boundaries, and non-goals. |
| [Architecture](architecture.md) | Defines the system structure and internal invariants that realize the contract. |
| [Environment](environment.md) | Defines runtime and integration conditions and the responsibilities of the host environment. |
| [Verification](verification.md) | Defines the verification strategy, functional evidence, targeted tests, proof obligations, and benchmark method and conditions. |
| [Decisions](decisions.md) | Records design decisions, rationale, trade-offs, rejected alternatives, and retained hypotheses. |

Each document owns its stated subject. Material should be linked rather than duplicated across the corpus.

## Reading the design

Start with the contract to understand what the router guarantees and excludes. Read the architecture for how the design realizes that contract, then the environment for the conditions under which it is integrated. Read the verification document for how conformance and performance are demonstrated. Consult the decisions when the reason for a constraint or trade-off matters.

Established requirements and decisions are authoritative in the document that owns them. Explicit hypotheses remain conditional, and open questions remain unresolved until the authoritative corpus records a decision. Evidence supports the design and tests demonstrate implementation behavior; neither silently redefines the contract.

## Source material

Working, evidence, capture, and rationale material under `.42p/cases/` is non-normative unless explicitly stated otherwise. The [design capture](../../.42p/cases/design/2026-09-28_pouchdb-http-router_design_capture_edit-0.13.md) preserves the source state from which this corpus was produced.
