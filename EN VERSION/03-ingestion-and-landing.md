# 03 · Ingestion and Landing: A Unified Entry Pattern Is Not About Adding a Layer, but Reducing Long-Term Complexity

## 1. Purpose

This document discusses the ingestion layer in a lakehouse architecture. It focuses on **why a Landing layer exists, how it differs from Bronze, how CDC relates to near real-time, and how to design an ingestion pattern that is replayable, governable, and extensible**.

It is not a connector guide, SQL guide, Terraform guide, or cloud service configuration manual. The goal is to establish a set of public, general, and reusable system design principles.

My basic view is:

> The goal of the ingestion layer is not to move data into Snowflake as quickly as possible. The goal is to hand off source data to the analytical platform in a way that is governable, replayable, auditable, and scalable from an operating model perspective.

The value of a unified Landing layer is also not that it “stores one more copy of the data.” Its value is that it creates a clear system boundary:

* source systems deliver data;
* Landing receives, records, and preserves handoff evidence;
* Bronze turns the data into a queryable raw layer inside the analytical platform;
* Silver and Gold handle business semantics and consumption models.

---

## 2. What the Ingestion Layer Needs to Solve

Data ingestion may look like “moving data from A to B,” but in an enterprise platform it solves many long-term problems.

### 2.1 Consistency Across Multiple Sources

Enterprise data sources often include:

* operational databases;
* batch files;
* SaaS exports;
* third-party feeds;
* application events;
* APIs;
* manually supplied files;
* legacy system extracts.

If every source is designed with its own ingestion pattern, the platform quickly accumulates many special paths. Each path has its own scheduling, naming, error handling, backfill mechanism, monitoring model, and owner.

The purpose of a unified ingestion pattern is to make different source systems follow the same handoff rules once they enter the platform.

### 2.2 Replay and Backfill

The ingestion layer must answer: if downstream loading fails, model logic is wrong, a data quality rule is incorrect, or historical data needs to be recalculated, can the platform replay the original input?

Without stable landing evidence, many backfills depend on source systems re-exporting data or on manual patching. This increases source system burden and makes platform recovery unreliable.

### 2.3 Audit and Data Evidence

Enterprise data platforms often need to answer:

* What did the source system deliver at a given time?
* Which files or change records entered the platform?
* Which inputs were loaded successfully?
* Which inputs failed or were quarantined?
* Which data was later transformed into business models?

These questions cannot be answered only by looking at the final Gold model. They require evidence preserved at the ingestion layer.

### 2.4 Governance and Privacy Boundaries

Not all source data should directly enter the common analytical path.

Some data may contain PII, secrets, sensitive business fields, restricted-region data, or data that requires special authorization. The ingestion layer must support isolation, redaction, tokenization, or rejection before data enters a common Landing area or Bronze.

If governance logic is pushed only to Silver or Gold, sensitive data may already have expanded its exposure surface.

### 2.5 The Reality of Near Real-Time

Many teams mix CDC, streaming, and near real-time in the same conversation.

The ingestion layer must separate the questions:

* Has CDC captured the change?
* Has the change entered Landing?
* Has it been loaded into Bronze?
* Have Silver and Gold refreshed?
* Is BI or the serving projection readable?
* Has the business actually consumed the new result?

Only when the full path meets the target freshness does the business truly have near real-time data.

---

## 3. Landing and Bronze: Responsibilities Should Be Separated; Physical Implementation May Be Combined

In this reference architecture, Landing and Bronze are modeled as two different responsibilities:

* **Landing** is the handoff and evidence layer for source data entering the platform;
* **Bronze** is the raw query layer after data enters the analytical platform.

In most enterprise scenarios, physically separating the two gives clearer replay, audit, governance, and cost boundaries. However, in small-scale, low-risk, or specific table-format architectures, the two responsibilities can also be implemented together.

### 3.1 Landing Is the Handoff Boundary

Landing is the handoff point between source systems and the data platform.

It answers questions such as:

* What did the source system deliver?
* When did the data arrive?
* Can the original input still be replayed?
* Can the platform recover after a loading failure?
* Which data is allowed to enter the common analytical path?
* Which data needs to be isolated or processed first?

The keywords for Landing are:

```text
handoff / replay / audit / archive / decoupling / source evidence
```

### 3.2 Bronze Is the Raw Query Layer

Bronze is the raw layer inside Snowflake or a similar analytical platform.

It answers questions such as:

* How can the data be queried with SQL?
* How does it support downstream incremental Silver processing?
* How is source metadata preserved?
* How are ingestion timestamp, source offset, operation type, and file reference recorded?
* How do downstream models avoid scanning raw object storage files directly?

The keywords for Bronze are:

```text
queryable raw layer / source-aligned tables / technical metadata / incremental processing
```

### 3.3 Why Separating Responsibilities Matters

Separating responsibilities makes troubleshooting and governance boundaries clearer.

If data does not appear in Gold, the team can inspect the path layer by layer:

1. Did the source system produce the data?
2. Did the data arrive in Landing?
3. Was the data loaded into Bronze?
4. Did Silver process it successfully?
5. Did Gold refresh?
6. Did the consumption layer read the latest result?

If Landing and Bronze responsibilities are mixed together, many problems become a vague statement: “the data did not arrive.”

### 3.4 When the Two Can Be Implemented Together

Landing and Bronze do not always need to be physically separated into two systems.

They can be combined or the Landing layer can be weakened in scenarios such as:

* small-scale PoCs;
* low-value or disposable data;
* source systems or connectors already provide reliable historical replay;
* Iceberg, Delta, external tables, or other table-format architectures where object storage itself acts as the raw table storage;
* extremely latency-sensitive paths with low replay or audit requirements.

However, even if the physical implementation is combined, the logical responsibilities should remain clear: what is source handoff evidence, and what is the analytical raw query layer?

---

## 4. Design Principles for Unified Landing

### 4.1 Unified Does Not Mean Every Source Uses the Same Connector

A unified Landing pattern does not mean every source system must use the exact same ingestion tool.

Databases may use CDC. Third-party systems may push files. SaaS platforms may be pulled through APIs. Application events may land as micro-batches.

The important point is that after data enters the platform, it should follow a consistent handoff model.

This includes:

* standard paths or naming conventions;
* clear source identifiers;
* clear event time and ingestion time;
* clear batch or change boundaries;
* clear schema version;
* clear file or message identity;
* clear replay and DLQ rules;
* clear PII and sensitive data boundaries.

### 4.2 Landing Should Remain Business-Logic Neutral

Landing should not contain complex business logic.

It may handle:

* format standardization;
* basic structure validation;
* file integrity checks;
* PII detection or isolation;
* duplicate file detection;
* metadata enrichment;
* rejection of clearly invalid data.

It should not handle:

* core business metric calculation;
* customer segmentation;
* financial definitions;
* risk rules;
* large business joins;
* BI-oriented modeling.

Those business rules belong in Silver or Gold, not hidden inside ingestion scripts.

### 4.3 Landing Should Support Idempotency

Data ingestion must assume duplicate delivery.

A source system may resend a file. A CDC connector may replay a range of offsets. An API pull may retrieve the same records again. A manual backfill may overlap with existing data.

Landing therefore needs some form of identity or fingerprint to decide:

* Is this a new file or a duplicate file?
* Is this a repeated delivery of the same source offset?
* Is this a corrected redelivery?
* Should the platform skip, overwrite, quarantine, or reload it?

Without idempotency, backfill and replay become dangerous.

### 4.4 Landing Should Support Partitioning and Lifecycle

Landing is not infinite hot storage.

It usually needs lifecycle management:

* recent data remains in a hot access tier;
* older data moves to lower-cost storage;
* data requiring long-term retention moves to archive;
* temporary or sensitive staging is cleaned on a shorter schedule.

This is one reason separating Landing and Bronze is useful:

* Landing can be managed based on storage cost and retention policies;
* Bronze can be managed based on query performance and modeling needs.

---

## 5. Common Ingestion Patterns

### 5.1 Batch File Ingestion

Batch file ingestion is suitable for:

* daily or hourly files;
* third-party feeds;
* SaaS exports;
* financial or operational reconciliation files;
* low-frequency business data.

A typical path is:

```text
Source export
  -> Landing
  -> validation
  -> Bronze load
  -> Silver / Gold refresh
```

Benefits:

* simple;
* easy to audit;
* easy to replay;
* cost-controllable.

Costs:

* freshness depends on batch frequency;
* late-arriving files must be handled;
* file schema drift needs governance.

### 5.2 CDC-Based Ingestion

CDC is suitable for:

* operational database changes;
* incremental loading;
* capturing inserts, updates, and deletes;
* tracking changes between batch runs;
* minute-level freshness.

A typical path is:

```text
OLTP change log
  -> CDC capture
  -> Landing change files / batches
  -> Bronze raw change table
  -> Silver merge / business mirror
```

Key considerations:

* CDC captures changes; it does not equal real-time consumption;
* source offset or log position must be tracked;
* deletes must be handled;
* out-of-order changes must be considered;
* schema evolution must be handled;
* replay windows must be defined.

### 5.3 API Ingestion

API ingestion is suitable for:

* SaaS systems;
* third-party platforms;
* external business interfaces;
* systems without database-level access.

A typical path is:

```text
API pull / webhook
  -> normalized payload
  -> Landing
  -> Bronze
```

Considerations include:

* rate limits;
* pagination;
* retries;
* partial failures;
* API schema version;
* token or credential rotation;
* whether source-side historical replay is reliable.

### 5.4 Event Micro-Batch Ingestion

Event micro-batch ingestion is suitable for:

* application events;
* user behavior;
* state changes;
* lightweight near real-time analytics.

It sits between batch and streaming.

A typical path is:

```text
Application events
  -> micro-batch buffer
  -> Landing
  -> Bronze
  -> incremental processing
```

This pattern is simpler than true streaming but more timely than daily batch updates.

### 5.5 Streaming Ingestion

Streaming ingestion is suitable for scenarios that truly require event-by-event processing.

Examples include:

* millisecond-level SLAs;
* real-time risk control;
* high-frequency event streams;
* complex event processing;
* online recommendations;
* transaction-level real-time decisions.

If the business truly needs these capabilities, the streaming platform should be designed as a first-class architectural component.

But if the requirement is only minute-level freshness, streaming-first often introduces unnecessary complexity.

---

## 6. CDC Is Not Real Time

CDC is one of the most commonly misunderstood concepts in ingestion design.

The value of CDC is that it captures source system changes and lets the platform know what happened between two loading windows. It is very useful for incremental loading, reconciliation, compensation, updating business mirrors, and reducing full reload cost.

But CDC only solves the first segment of the path.

From source change to business consumption, the data still needs to pass through:

```text
Change capture
  -> Landing
  -> Bronze load
  -> Silver processing
  -> Gold refresh
  -> BI refresh or serving publish
  -> downstream consumption
```

Therefore, near real-time should not be judged only by whether CDC exists. It should be judged by end-to-end freshness.

### 6.1 When CDC Is Needed but Streaming Is Not

Many scenarios only need to know what happened between two batch loading windows:

* which orders were updated;
* which customer states changed;
* which accounts need recalculation;
* which aggregates need refresh;
* which records need compensation processing.

These requirements need CDC, but they do not necessarily need a full streaming stack.

If the business can accept minute-level refresh, CDC + micro-batch + Snowflake-native incremental processing may be sufficient.

### 6.2 When CDC Is Not Enough

CDC is not enough when:

* the business needs millisecond-level response;
* every event must trigger an immediate decision;
* complex window computation is required;
* cross-event state machines are required;
* high-throughput event processing is required;
* downstream applications depend on event streams rather than refreshed tables.

These scenarios should move into streaming or event-driven architecture, rather than only strengthening CDC.

---

## 7. Schema Drift and Data Contracts

The ingestion layer must assume that schemas will change.

Source systems may add fields, delete fields, change types, change enum values, adjust primary keys, change file formats, or upgrade API versions.

### 7.1 Basic Strategy for Schema Drift

Common strategies include:

| Change Type                  | Suggested Handling                                                 |
| ---------------------------- | ------------------------------------------------------------------ |
| New nullable field           | Accept into raw payload; decide later whether to promote to Silver |
| New important business field | Review schema and then promote into Silver / Gold                  |
| Deleted field                | Trigger compatibility checks and downstream impact analysis        |
| Narrower type                | Handle carefully; may need DLQ                                     |
| Wider type                   | Usually acceptable, but schema version should be recorded          |
| Primary key change           | Requires migration plan and backfill plan                          |
| Enum value change            | Requires accepted values or business rules to be updated           |

### 7.2 Why Bronze Should Not Be Over-Structured

Bronze needs to preserve enough raw information to support schema drift handling.

If Bronze enforces a schema too narrowly too early, source system changes can cause data loss or loading failures.

A better pattern is:

* Bronze preserves raw payload;
* key technical fields are structured;
* Silver decides which fields become stable business fields.

### 7.3 Contracts Should Strengthen Layer by Layer

Data contracts should not exist only at Gold.

Different layers need different contracts:

* Landing: file format, source, batch identity, schema version;
* Bronze: technical metadata, raw payload, source offset;
* Silver: business key, deduplication rules, entity definition;
* Gold: metric definitions, freshness, quality, owner, consumer contract.

Contracts should strengthen layer by layer so that all complexity is not pushed to the final layer.

---

## 8. DLQ and Error Handling

The ingestion layer needs an explicit error handling mechanism.

Not all bad data should block the entire pipeline, and not all errors should be silently ignored.

### 8.1 Common Error Types

| Error Type        | Example                                                      |
| ----------------- | ------------------------------------------------------------ |
| Format error      | Unparseable file, malformed JSON, incorrect CSV column count |
| Schema drift      | Type change, missing required field, unknown structure       |
| Duplicate input   | Duplicate file, offset replay, repeated API retry            |
| PII violation     | Plaintext sensitive field appears where it should not        |
| Referential issue | Missing upstream key, parent-child order issue               |
| Load failure      | Snowflake load failure, permission issue, format issue       |
| Late arrival      | Data arrives outside the expected window                     |

### 8.2 Purpose of DLQ

The purpose of DLQ is not to hide problems. It is to isolate problems while preserving diagnostic information.

DLQ should answer:

* which record or file failed;
* why it failed;
* which source, batch, and schema version it belongs to;
* whether it can be retried;
* whether it requires manual intervention;
* whether it involves PII or a security issue.

### 8.3 Error Handling Principles

Recommended principles:

* bad records that can be isolated should not block the entire source;
* PII or security issues should fail closed;
* schema drift should trigger review, not silent field loss;
* duplicate input should be handled through idempotency;
* DLQ should be monitored, not allowed to accumulate indefinitely;
* after fixing issues, a replay or reprocess path should exist.

---

## 9. Replay, Backfill, and Reprocessing

Replay is one of the most important values of the Landing layer.

Without replay capability, the data platform becomes overly dependent on source systems re-exporting data, manual backfills, or manual downstream table patches.

### 9.1 Common Replay Scenarios

* Snowflake load failure;
* Bronze schema mapping error;
* Silver logic fix;
* Gold metric definition change;
* historical recomputation;
* late-arriving data;
* duplicate correction;
* PII handling rule update.

### 9.2 Levels of Replay

Different problems require different replay sources:

| Problem Location       | Replay Source                                                                |
| ---------------------- | ---------------------------------------------------------------------------- |
| Bronze loading failure | Reload from Landing                                                          |
| Silver logic error     | Rebuild Silver from Bronze                                                   |
| Gold metric error      | Rebuild from Silver or Gold source model                                     |
| Source data error      | Source system must redeliver or correct the feed                             |
| PII rule change        | May require reprocessing from secure staging or controlled historical source |

### 9.3 Backfill Should Not Be a Special Incident Procedure

Backfill is a normal capability of a data platform. It should not depend on one-off manual scripts every time.

A mature ingestion pattern should make backfill a controlled process:

* specify source;
* specify time range;
* specify schema version;
* specify target layer;
* record execution result;
* validate data quality;
* avoid duplicate writes.

---

## 10. PII and Sensitive Data Handling

The ingestion layer is the first boundary for privacy control.

If sensitive data has already entered common Landing, Bronze, BI, or general-purpose analytical schemas, controlling exposure becomes much harder.

### 10.1 Common Landing Should Not Receive Ungoverned Sensitive Data

Recommended principle:

> Common Landing should receive data that is allowed to enter the analytical platform. Ungoverned sensitive data should enter isolated staging first, then be processed before it enters common Landing or Bronze.

Common handling patterns include:

* hash;
* tokenize;
* redact;
* mask;
* drop;
* isolate;
* reject.

### 10.2 Fail Closed

If the ingestion layer detects sensitive fields where they should not appear, it should fail closed.

That means:

* do not load into the common path;
* do not load into Bronze;
* write to a controlled error area;
* trigger security or data governance review;
* reprocess after fixing the issue.

### 10.3 Privacy Rules Should Be Replayable

If hashing or tokenization rules are upgraded, the platform needs to consider how historical data will be reprocessed.

This requires ingestion metadata to preserve:

* privacy rule version;
* schema version;
* source batch;
* processing timestamp;
* reprocessing path.

---

## 11. Freshness, Completeness, and Observability

Successful ingestion is not just “the job succeeded.”

An ingestion pipeline should be observed across at least the following dimensions.

### 11.1 Freshness

The platform should know:

* lag from source event time to now;
* file arrival time;
* Bronze load time;
* Silver / Gold refresh time;
* serving projection update time.

Freshness should be measured end to end, not only by whether the connector is running.

### 11.2 Completeness

The platform should know:

* whether all expected batches arrived;
* whether row counts are abnormal;
* whether file sizes are abnormal;
* whether CDC offsets are continuous;
* whether there are duplicates or gaps;
* whether a source has been silent for too long.

### 11.3 Correctness

The platform should know:

* whether schema matches expectations;
* whether types are correct;
* whether required fields are null;
* whether enum values are within the accepted range;
* whether business keys can be resolved;
* whether PII appears where it should not.

### 11.4 Cost and Operational Signals

The platform should also observe:

* small file count;
* load cost;
* failed file count;
* DLQ growth;
* replay frequency;
* source-specific latency;
* ingestion compute consumption.

These metrics are useful not only for troubleshooting but also for FinOps and TCO management.

---

## 12. Core Trade-Offs in the Ingestion Pattern

| Design Choice                              | Benefit                                                                          | Cost                                                                       |
| ------------------------------------------ | -------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Use unified Landing                        | Decouples source systems from the analytical platform; supports replay and audit | Adds object storage, metadata, and lifecycle management                    |
| Separate Landing / Bronze responsibilities | Clearer boundaries for governance and troubleshooting                            | May feel heavy for small-scale scenarios                                   |
| Use CDC                                    | Captures changes between batches and supports incremental processing             | Does not equal end-to-end real time; requires offset and schema management |
| Use micro-batch                            | Reduces streaming complexity while supporting minute-level refresh               | Not suitable for millisecond-level event processing                        |
| Preserve raw payload in Bronze             | Supports schema drift and later field promotion                                  | Requires governance for storage and queries                                |
| Keep business logic out of ingestion       | Clearer responsibility and easier maintenance                                    | Some business processing must wait until Silver / Gold                     |
| Handle PII before common paths             | Reduces privacy risk                                                             | Adds ingestion complexity and reprocessing requirements                    |
| Use DLQ to isolate bad data                | Prevents localized bad data from blocking the whole pipeline                     | Requires monitoring and handling process                                   |
| Support replay / backfill                  | Makes the platform recoverable and rebuildable                                   | Requires idempotency, metadata, and execution discipline                   |

---

## 13. Common Anti-Patterns

| Anti-Pattern                                                      | Problem                                                  | Better Approach                                              |
| ----------------------------------------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| Designing a custom ingestion architecture for every source        | Long-term maintenance and troubleshooting become complex | Use a unified Landing and ingestion contract                 |
| Writing source data directly into Gold                            | Lacks raw evidence and replay boundaries                 | Land first, then load into Bronze                            |
| Putting heavy business logic in ingestion adapters                | Hard to test, reuse, and govern                          | Move business logic into Silver / Gold                       |
| Building a full streaming stack just because CDC exists           | Confuses change capture with real-time consumption       | Design based on end-to-end freshness SLA                     |
| Keeping Landing as hot storage indefinitely                       | Cost becomes uncontrolled                                | Use lifecycle and archive policies                           |
| Over-structuring Bronze                                           | Schema drift can cause data loss or blockage             | Preserve raw payload and strengthen contracts layer by layer |
| Letting plaintext sensitive data enter common Landing             | Expands privacy risk                                     | Use isolated staging and ingest-time handling                |
| Writing to DLQ but never reviewing it                             | Problems become hidden data debt                         | Monitor DLQ and design a handling process                    |
| Relying on one-off scripts for backfill                           | Not auditable or repeatable                              | Establish controlled replay / backfill mechanisms            |
| Looking only at connector success instead of end-to-end freshness | Misjudges data availability                              | Monitor the full path from source event to consumption       |

---

## 14. Success Criteria

A well-designed ingestion and landing architecture should satisfy the following criteria:

* new data sources can reuse a standard ingestion pattern;
* Landing and Bronze responsibilities are clear;
* most enterprise data sources have a replay path;
* Bronze preserves enough technical metadata;
* CDC paths can explain source offsets and change semantics;
* near real-time freshness is defined end to end;
* PII and sensitive data are handled or isolated before entering common paths;
* schema drift has a handling strategy;
* DLQ has owners, monitoring, and handling processes;
* backfill is not treated as a one-off incident procedure;
* cost and small file issues are monitored;
* ingestion does not hide core business logic;
* downstream Silver / Gold can be rebuilt reliably from Bronze.

---

## 15. Closing Summary

Ingestion and Landing are often underestimated in lakehouse architecture.

If they are treated only as “loading data into Snowflake,” the platform will pay the cost later: tight coupling to source systems, weak replay capability, unclear audit boundaries, blurry PII boundaries, uncontrolled schema drift, CDC being mistaken for real-time consumption, and backfills depending on manual scripts.

A more robust design is:

* Landing acts as the handoff and evidence layer for source data entering the platform;
* Bronze acts as the raw query layer inside the analytical platform;
* CDC supports change awareness and incremental capture, but does not automatically mean real time;
* PII and sensitive data are handled before entering common paths;
* ingestion remains business-logic neutral;
* replay, DLQ, schema drift handling, and freshness are designed as platform capabilities from the beginning.

This design adds some upfront structure compared with direct ingestion, but it provides lower long-term complexity, clearer governa
