# 0003: Separate Tables per Document Type

## Status

Accepted

## Context

ENTSO-E exposes multiple document types (e.g. Day-Ahead Prices,
Actual Generation, Total Load), as established in ADR 0001 §1 and §7.

Different document types do not share the same response schema.
For example, Day-Ahead Prices contains a `price_amount` field, while
Actual Generation contains `quantity` and `psrType` fields that do
not exist in the price dataset.

A decision is needed on whether to store all document types in a
single table distinguished by a `document_type` column, or to use
separate tables per document type.

## Options Considered

1. **Single table with a `document_type` column.** All document
   types share one table; a `document_type` column distinguishes
   rows. Columns specific to one document type would be `NULL` for
   rows belonging to other document types.

2. **Separate tables per document type.** Each document type has its
   own staging and final tables, with a schema specific to that
   document type.

## Decision

Separate tables will be used per document type (e.g.
`staging_day_ahead_prices`, `staging_actual_generation`).

This avoids a sparse schema where a large portion of columns would
be `NULL` depending on the document type of a given row, and allows
each table's schema to accurately reflect the structure of its
source document type.

The table name itself identifies the document type, so a
`document_type` column within each table would be redundant (see
ADR 0002).

## Consequences

### Positive

* Each table's schema accurately reflects its document type; no
  columns are structurally irrelevant to a subset of rows.
* Queries do not need to filter on `document_type`, since each table
  already represents a single type.
* Adding a new document type does not require altering an existing
  table's schema.

### Negative

* The number of tables grows as new document types are added.
* Any logic that needs to query across document types (e.g. joining
  prices with generation) must explicitly union or join across
  multiple tables rather than filtering a single one.
* Shared ingestion logic (e.g. retry handling, batch tracking) must
  be designed to work generically across tables rather than assuming
  a single table.

## Related Decisions

* ADR 0001: ENTSO-E API Structure Findings
* ADR 0002: Staging and Final Data Layers