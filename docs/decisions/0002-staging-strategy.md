# 0002: Staging and Final Data Layers

## Status

Accepted

## Context

The data retrieved from ENTSO-E needs to go through validation and
transformation before it can be used by downstream applications.

Keeping only the final, transformed data would make it difficult to
reprocess the data when transformation rules change or when a
processing step fails. It would also require fetching the data from
ENTSO-E again.

A separate staging layer provides a boundary between data ingestion
and downstream processing. This makes it possible to isolate
ingestion errors, keep track of what was received from the source,
and rebuild the final data without depending on another API request.

The pipeline should therefore support:

* Reprocessing previously ingested data.
* Isolating ingestion and transformation errors.
* Keeping enough source information to trace a record back to its
  ingestion request.
* Updating transformation logic without having to fetch the source
  data again.
* Keeping the final layer consistent and suitable for downstream use.

## Decision

Based on the API characteristics and data volatility, I decided to
adopt a **Two-Tier Data Ingestion Strategy**.

The data will be separated into two logical layers:

1. Staging
2. Final

### Staging Layer

The staging layer will contain data received from ENTSO-E, with only
the minimum processing required to store and identify the records.

For the current Day-Ahead Prices dataset, the staging table will be
defined as follows:

```text
staging_day_ahead_prices:
  - bidding_zone
  - time_interval_start
  - resolution
  - position
  - price_amount
  - timestamp_utc
  - ingestion_batch_id
  - ingested_at
  - source_endpoint
  - request_parameters
```

The `bidding_zone` column is an explicit data attribute, as
established in ADR 0001 §6.

Per ADR 0003, each document type has its own staging and final
table. The table name (`staging_day_ahead_prices`) already identifies
the document type, so a `document_type` column is not included in
this table — storing a constant `A44` value on every row would add
redundancy without providing additional query or partitioning value.

`request_parameters` is retained as raw request metadata for audit
and replay purposes. It is not used as the primary representation of
queryable dataset attributes.

The staging layer is not intended for direct consumption by
downstream applications.

It acts as the intermediate layer between the external API and the
final data model.

### Final Layer

The final layer will contain validated, transformed, and normalized
data.

Data in this layer will:

* Follow a defined schema.
* Use consistent data types.
* Have normalized timestamps and units.
* Have validated required fields.
* Be suitable for analytical queries and downstream applications.

Data will move from staging to final only after the required
validation and transformation steps have been completed.

## Data Flow

```text
ENTSO-E API
    ↓
Staging
    ↓
Validation
    ↓
Transformation / Normalization
    ↓
Final
```

## Consequences

### Positive

* Previously ingested data can be reprocessed without requesting it
  from ENTSO-E again.
* Data lineage is easier to establish.
* Transformation and ingestion failures can be isolated.
* The final layer remains clean and consistent.
* Changes to transformation logic do not require re-fetching data
  from ENTSO-E.
* The staging layer provides a stable point for replaying the
  pipeline.

### Negative

* Storage requirements increase.
* The pipeline becomes more complex.
* A lifecycle policy may be required if the project scope expands
  significantly.
* Additional processing is required to move data from staging to
  final.

## Retention Policy

No retention limit is enforced for the staging layer at this stage.

This is a deliberate decision, not an open item: at the current
data volume (a small number of bidding zones, quarter-hourly to
hourly resolution), the storage cost of keeping staging data
indefinitely is negligible compared to the cost of re-fetching it
from the ENTSO-E API.

A lifecycle policy (e.g. moving staging data older than N days to
cold storage, or deleting it) will be introduced if the project
scope expands to cover many bidding zones or several years of
history, at which point storage cost becomes a relevant factor.

## Implementation Notes

Each ingestion batch should have a unique identifier.

The staging layer should store enough metadata to identify:

* When the data was retrieved.
* Which ENTSO-E endpoint was used.
* Which query parameters were used.
* Which ingestion batch produced the record.

The staging layer will also be responsible for ingestion-level
normalization required before the data reaches the final layer.

For the current Day-Ahead Prices dataset, `timestamp_utc` will be
calculated in staging from:

```text
timestamp_utc =
    timeInterval_start + ((position - 1) × resolution)
```

The natural key used to identify an observation is:

```text
bidding_zone
+ timeInterval_start
+ resolution
+ position
```

This key will be used to support idempotent ingestion and prevent
duplicate observations when an ingestion batch is retried or
reprocessed.

The final layer should not depend on API-specific fields unless
those fields are part of the defined final data model.

## Related Decisions

* ADR 0001: ENTSO-E API Structure Findings
* ADR 0003: Per-Document-Type Tables