# ENTSO-E API & Source Data Analysis

**Dataset:** Day-Ahead Prices (documentType: A44)
**Author:** Aylin Akbari
**Date:** September 2026

# 0001: API Structure Findings

## Status

Accepted

## Context

I have decided to collect energy-related data from ENTSO-E and make it
available for downstream processing and analysis.

The data is provided through ENTSO-E's APIs and may include
different datasets, time resolutions, geographical areas, and
data types.

The ingestion system should:

* Retrieve data reliably from ENTSO-E.
* Support multiple data types.
* Handle API failures and temporary unavailability.
* Avoid unnecessary duplicate requests.
* Preserve the original/raw data.
* Allow the ingestion process to be extended as new datasets
  are required.
* Support reproducible data processing.

Before defining the ingestion and storage architecture, the
structure and semantics of the ENTSO-E API response need to be
understood.

This ADR records findings from the investigation of the ENTSO-E
Day-Ahead Prices dataset (`documentType: A44`). Architectural
decisions based on these findings are documented in separate ADRs.

## Findings

### 1. What does each `TimeSeries` represent?

In a request for a single bidding zone, fields such as
`in_Domain`, `out_Domain`, and `currency_Unit` are typically
constant across the `TimeSeries` elements in the response.
Therefore, these fields do not explain why multiple `TimeSeries`
elements appear within the same response; they describe the
context of the returned data.

Based on the observed responses, changes in `resolution` across
different parts of the requested time range can result in multiple
`TimeSeries` elements appearing in a single response.

The `mRID` at the `TimeSeries` level is a sequential identifier
assigned by the sender. It should not be used as part of the
natural key because it is not guaranteed to remain stable across
different request executions.

### 2. What is resolution and how does it affect ingestion?

`resolution` defines the temporal granularity of the data.

Currently observed resolutions include:

* `PT15M`: Quarter-hourly
* `PT60M`: Hourly

Resolution affects the number of observations generated for a given
time interval and therefore needs to be preserved during ingestion.

### 3. How is `position` indexed?

`position` is 1-indexed.

The timestamp of an individual observation can therefore be
calculated as:

```text
timestamp = PeriodStart + ((position - 1) × resolution)
```

This calculation provides the basis for deriving an explicit
timestamp for each observation.

Based on observed responses, each TimeSeries element contains exactly one Period. Resolution changes within a requested time range result in separate TimeSeries elements rather than multiple Period elements within a single TimeSeries.


### 4. Timezone Standard in `timeInterval` (UTC vs Local)

The time interval is represented using UTC.

The ingestion process should therefore preserve UTC semantics when
deriving timestamps from the interval start, resolution, and
position.

### 5. If a country/zone has a different resolution rate, does XML show it?

Yes.

The XML response contains the resolution information associated
with the returned data.

Therefore, the ingestion process should not assume a single
resolution globally and should read the resolution from the
source data.

### 6. `bidding_zone` as an explicit data attribute

The bidding zone should be stored as a dedicated column rather
than being retained only inside the original request parameters.

This makes the semantic identity of the dataset explicit and
allows the column to be used directly for filtering and
partition-related operations.

The same principle applies to other dataset-level attributes that
are required for querying or identifying the data.

### 7. `document_type` as an explicit data attribute

`document_type` should be stored as a dedicated column rather than
being represented only through the API request parameters.

For the current dataset, the value is:

```text
document_type = A44
```

Keeping this attribute explicit preserves the source dataset
identity independently of the request that produced the record.

The decision to use separate tables for different document types
is intentionally outside the scope of this ADR and is documented
separately.

### 8. Natural identity of an observation

The investigation indicates that an individual time-series
observation can be identified using its bidding zone and temporal
position.

The candidate natural key for idempotent ingestion is:

```text
bidding_zone
+ timeInterval_start
+ resolution
+ position
```

This combination will be used as the natural key for identifying
duplicate observations in the ingestion process.

The implementation of idempotency and the staging strategy are
documented in ADR 0002.

### 9. Derived `timestamp_utc`

The source data provides the components required to calculate an
explicit UTC timestamp:

* `timeInterval_start`
* `resolution`
* `position`

The derived field will be:

```text
timestamp_utc =
    timeInterval_start + ((position - 1) × resolution)
```

`timestamp_utc` will be calculated during the staging step rather
than in the final layer.

The reason for calculating this field in staging is to establish a
consistent temporal representation before the data reaches the
final data model.

The staging/final strategy is documented in ADR 0002.

## Scope of This ADR

This ADR documents findings about the structure and semantics of
the ENTSO-E source data.

The following architectural decisions are intentionally documented
separately:

* Staging and final data layers → ADR 0002
* Idempotent ingestion and natural-key enforcement → ADR 0002
* Separate tables per `document_type` → ADR 0003
* Detailed partitioning strategy → separate ADR if required

## Related Decisions

* ADR 0002: Staging Strategy
* ADR 0003: Per-Document-Type Tables
