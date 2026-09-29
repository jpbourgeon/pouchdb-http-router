# Editorial standard

This standard governs repository documentation.

## Document responsibility

Each document must have a clear responsibility and contain the information necessary and sufficient to fulfil it.

Do not duplicate material owned by another authoritative document. Link to the authoritative source instead.

Choose structure from the subject matter. Do not impose a common section template unless it improves comprehension.

## Target state

Authoritative documentation describes the intended repository state after the change that introduces it is merged.

Do not describe temporary branch state, pull-request mechanics, drafting history, or discarded alternatives unless that transient state is relevant to the document's purpose.

Working, capture, evidence, and rationale material may preserve history. Published documentation should present the resulting design directly.

## Precision

Prefer direct, concrete statements over commentary, narration, or rhetorical framing.

Preserve necessary constraints, qualifications, and uncertainty. Do not shorten text by removing information required to understand the contract or design.

Avoid filler, repetition, redundant summaries, and conclusions that merely restate preceding content.

## Terminology

Use stable domain terminology consistently.

Prefer PouchDB-native concepts when describing PouchDB behavior, including replication, synchronization, changes feeds, revisions, checkpoints, attachments, and database handles.

Do not replace a precise domain term with a broader architectural synonym for stylistic variety.

Use one term for one concept unless a distinction is intentional.

## Design and rationale

State behavior, constraints, invariants, and boundaries unambiguously.

Keep implementation detail out of design descriptions unless it affects the contract.

Include rationale when it is necessary to understand a constraint, trade-off, or non-obvious design choice. Otherwise state the resulting design directly.

Use normative keywords such as `MUST`, `SHOULD`, and `MAY` only when their formal strength is intentional.

## Structure

Prefer short paragraphs organized around one idea.

Use lists for genuinely parallel items.

Use tables when comparison across dimensions is clearer than prose.

Use diagrams only when they communicate structure or interaction more clearly than text.

Do not add sections merely to satisfy a template.

## Markdown

Use standard Markdown without presentation-specific constructs unless they materially improve the document.

Use one level-one heading for the document title and ATX headings for structure.

Use sentence case for headings.

Use fenced code blocks with a language identifier when applicable.

Use relative repository links for repository-local documents.

Prefer descriptive link text when it improves readability.

## Authority

Follow the repository authority boundaries defined in `AGENTS.md`.

Do not silently reconcile conflicting authoritative sources. Resolve the applicable authority before editing documentation.
