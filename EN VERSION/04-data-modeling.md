# 04 · Data Modeling: Medallion Is Not Layer Naming, but Semantic Decoupling

## 1. Purpose

This document discusses data modeling in a lakehouse architecture. It focuses on **what Bronze / Silver / Gold should actually be responsible for, and why the core value of Medallion architecture is not the layer names themselves, but the decoupling of OLTP structures from business semantics**.

This is not a dimensional modeling tutorial, nor is it a template for a specific industry data model. The focus here is a more general system design question:

> After data enters the analytical platform, how should it evolve from “source system storage structures” into “business-understandable, governable, and consumable data products”?

My basic view is:

> The value of Medallion architecture is not that tables are labeled Bronze / Silver / Gold. Its value is that it separates two different types of data work: first, converting OLTP storage structures into stable business mirrors; second, extracting metrics, reports, data products, and operational projections from those business mirrors.

This decoupling creates several important outcomes:

* OLTP systems can continue evolving for application needs;
* downstream consumers do not need to depend directly on source system structures;
* business logic can evolve gradually in Silver and Gold;
* data quality, access control, cost attribution, and lineage become easier to manage;
* near real-time scenarios can define freshness layer by layer instead of being hidden inside an opaque script.

---

## 2. Why a Data Modeling Layer Is Needed

Many data platform problems are not caused by lack of data. They are caused by the lack of stable semantics.

If data is consumed directly by BI, reports, analysts, and applications immediately after being ingested from source systems, the platform will quickly encounter the following issues.

### 2.1 Source System Structures Are Not Business Semantics

OLTP tables are usually designed for application writes, transactional consistency, and query performance. Their structures often reflect system implementation details rather than how business users understand the world.

For example, one business object may be split across multiple application tables. One source table may mix several business concepts. Field names may come from historical implementation choices. Primary keys may only be meaningful inside a single system. State changes may be hidden inside update records.

If BI and metrics depend directly on these structures, every source system change propagates directly into business consumption.

### 2.2 Business Logic Becomes Scattered

Without a clear modeling layer, business logic easily becomes scattered across:

* BI tools;
* temporary SQL;
* Python scripts;
* application code;
* export jobs;
* manual spreadsheets;
* data synchronization jobs.

The result is that the same metric has multiple versions, the same entity has multiple definitions, and answering one reporting question requires tracing logic across multiple systems.

### 2.3 Data Quality Problems Become Hard to Locate

Without layer boundaries, data quality issues are hard to diagnose.

When a report number is wrong, the team may not know whether:

* the source system data is wrong;
* ingestion lost data;
* Bronze parsed data incorrectly;
* Silver deduplication logic is wrong;
* Gold metric definition is wrong;
* the BI presentation layer calculated incorrectly.

One value of layered modeling is that troubleshooting becomes a layer-by-layer process instead of a guess across the entire pipeline.

### 2.4 Governance and Cost Cannot Attach to Concrete Objects

Without stable models, it is difficult to define:

* owner;
* freshness SLA;
* quality tests;
* PII classification;
* retention;
* consumer contract;
* cost attribution;
* downstream dependency.

Governance and FinOps need to attach to concrete data products, not just to a pile of source tables and reports.

---

## 3. The Core Meaning of Medallion

Many people understand Medallion as three layers of tables: Bronze, Silver, and Gold.

I think that understanding is incomplete. More accurately, Medallion is a **semantic evolution and responsibility separation pattern**.

```text
Source storage structure
  -> Bronze: raw evidence
  -> Silver: business mirror
  -> Gold: business consumption
```

### 3.1 Operation Type One: From OLTP Structure to Business Mirror

Source system tables are usually not stable business semantics.

The first type of data work converts those table structures into relatively stable business mirrors. This usually happens between Bronze and Silver.

It includes:

* cleaning formats;
* standardizing types;
* deduplication;
* handling updates and deletes;
* interpreting source system states;
* aligning multiple source systems;
* establishing business keys;
* merging multiple sources of the same entity;
* handling historical changes;
* hiding source-specific fields.

The goal of this step is not to calculate final metrics. The goal is to let downstream consumers work with business objects instead of OLTP table structures.

### 3.2 Operation Type Two: From Business Mirror to Useful Information

The second type of data work extracts useful information from the business mirror. This usually happens between Silver and Gold.

It includes:

* metric definitions;
* fact and dimension modeling;
* aggregation;
* business marts;
* semantic-ready models;
* operational serving source models;
* reporting models;
* data products.

The goal of this step is to support business consumption, not to continue cleaning source system details.

### 3.3 Benefits of the Decoupling

Once these two types of work are decoupled, the platform gains several advantages:

* OLTP schemas can change while Silver remains relatively stable as the business mirror;
* business metrics can evolve without reinterpreting every source system detail;
* downstream consumption can depend on Gold instead of source tables;
* data quality can be tested by layer;
* lineage becomes easier to explain;
* cost can be attributed by model and use case.

---

## 4. Bronze: Raw Evidence Layer

Bronze is the raw evidence layer after data enters the analytical platform.

Its responsibility is to preserve source inputs and technical context, providing a rebuildable foundation for downstream processing.

### 4.1 What Bronze Should Preserve

Bronze should usually preserve:

* source system identifier;
* original business fields;
* source event time;
* ingestion time;
* source operation, such as insert / update / delete;
* source offset or file reference;
* pipeline identifier;
* raw payload;
* schema version;
* record hash or fingerprint.

These fields are not primarily for business users to query directly. They exist so downstream layers can understand where the data came from, when it arrived, how it changed, and whether it can be replayed.

### 4.2 What Bronze Should Not Do

Bronze should not be responsible for:

* core business metrics;
* BI-oriented semantic models;
* complex cross-source joins;
* business logic such as customer segmentation, risk scoring, or financial definitions;
* operational serving;
* final access projections.

If Bronze starts directly serving many business reports, the platform likely lacks a stable Silver / Gold layer.

### 4.3 Bronze Design Principles

Bronze design should follow these principles:

1. **Preserve fidelity first**: keep source system information as much as reasonably possible.
2. **Keep technical metadata complete**: support replay, debugging, and lineage.
3. **Be schema-drift friendly**: do not drop unknown fields too early.
4. **Be append-friendly**: especially for CDC and event data.
5. **Do not serve business consumption directly**: avoid turning the raw layer into the de facto reporting layer.

### 4.4 Relationship Between Bronze and Landing

Landing is the handoff and evidence layer for source data entering the platform. Bronze is the raw query layer inside the analytical platform.

In most enterprise scenarios, physically separating the two helps create clearer replay, audit, governance, and cost boundaries. However, in small-scale, low-risk, or specific table-format architectures, the two can also be implemented together.

The key is not that they must always be two physical systems. The key is that their logical responsibilities must be clear.

---

## 5. Silver: Business Mirror Layer

Silver is one of the most underestimated layers in Medallion architecture.

Many teams understand Silver simply as a “cleaned layer.” I think a more accurate definition is: **the business mirror layer**.

### 5.1 Core Responsibilities of Silver

Silver converts source system structures into stable business objects.

It usually handles:

* deduplication;
* type standardization;
* null and default handling;
* enum standardization;
* multi-source schema alignment;
* mapping source keys to business keys;
* entity resolution;
* update and delete semantics;
* late-arriving data;
* SCD or historical state;
* foundational business rules;
* foundational quality checks.

The goal of Silver is to let downstream consumers use business concepts instead of source system concepts.

### 5.2 What Business Mirror Means

A business mirror is not a final report, and it is not the full metric layer.

It is a stable mapping of business entities and business processes.

For example, a generic enterprise platform might have Silver objects such as:

* customer;
* account;
* product;
* order;
* transaction;
* subscription;
* payment;
* event;
* case;
* user activity.

These objects are not simple copies of OLTP tables. They are business mirrors after standardization and semantic alignment.

### 5.3 Silver Should Remain Relatively Stable

The value of Silver is that it shields downstream consumers from source system changes.

When source systems add fields, split tables, merge tables, rename fields, or upgrade APIs, Silver should try to maintain stability for downstream users.

This does not mean Silver never changes. It means Silver changes should be managed through data contracts, versioning, and compatibility policies.

### 5.4 Silver Should Not Become the Final Metric Layer Too Early

Silver can contain foundational business rules, but it should not carry too many final metrics too early.

For example:

* standardized transaction amount can belong in Silver;
* final revenue metrics should belong in Gold;
* standardized customer state can belong in Silver;
* customer lifecycle reporting metrics should belong in Gold.

The goal of Silver is to create clean, stable, reusable business objects — not to become another Gold layer.

---

## 6. Gold: Business Consumption Layer

Gold is the business consumption layer.

Its goal is to turn the business mirrors in Silver into models that can be directly consumed by BI, analytics, management reporting, business systems, or data products.

### 6.1 Common Gold Objects

Gold usually includes:

* fact tables;
* dimension tables;
* aggregate marts;
* business metrics tables;
* reporting models;
* semantic-ready models;
* operational serving source models;
* data product tables.

Gold consumers include:

* BI dashboards;
* business analysts;
* finance, operations, or product teams;
* executive reporting;
* downstream applications;
* data product consumers.

### 6.2 Core Responsibilities of Gold

Gold is responsible for:

* unifying metric definitions;
* providing business-readable models;
* supporting BI query performance;
* defining consumer contracts;
* managing freshness;
* exposing approved data products;
* providing stable sources for operational serving.

If a metric is important to the business, it should be defined in Gold whenever possible, instead of being repeatedly implemented across multiple BI reports.

### 6.3 Gold Is Not One Big Wide Table

Many platforms interpret Gold as “put every field into one wide table.” This usually creates problems:

* overly wide tables;
* tightly coupled logic;
* duplicated metrics;
* high refresh cost;
* difficult downstream dependency management;
* high risk from schema changes.

A better approach is to design Gold according to consumption patterns:

* stable analytics use facts and dimensions;
* high-frequency reports use aggregate marts;
* self-service analytics use semantic-ready models;
* application point lookups use serving source models;
* specific data products use clearly contracted data product tables.

---

## 7. Trade-Offs Among Fact, Dimension, and Wide Tables

A common debate in the Gold layer is whether to use Kimball-style facts and dimensions, or direct wide tables.

### 7.1 Value of Fact / Dimension Modeling

Fact / dimension modeling is suitable for:

* multi-dimensional analysis;
* stable metrics;
* BI reuse;
* shared reporting across multiple dashboards;
* slowly changing dimensions;
* historical analysis;
* complex slicing and dicing.

Its benefits are clear structure, stable semantics, and high reuse.

Its cost is higher modeling effort and a stronger requirement for data modeling capability.

### 7.2 Value of Wide Tables

Wide tables are suitable for:

* specific reports;
* specific data products;
* high-performance reads;
* simple downstream consumption;
* serving source models;
* low-dimensional and low-change scenarios.

Their benefit is simplicity and direct query access.

Their cost is that they can easily couple too much logic and become harder to maintain over time.

### 7.3 Suggested Decision Rule

Use the consumption pattern to decide:

| Scenario                                         | Better Fit                      |
| ------------------------------------------------ | ------------------------------- |
| Multiple reports share the same business process | Fact / dimension                |
| High-frequency fixed report                      | Aggregate mart or wide table    |
| Self-service analytics                           | Semantic-ready fact / dimension |
| Application point lookup                         | Serving-oriented wide model     |
| One-off analysis                                 | Temporary model or view         |
| Complex historical dimensions                    | Dimension with SCD              |

Do not abandon dimensional modeling simply because this is a modern lakehouse. Also do not push all logic into one table simply because a wide table is easy to query.

---

## 8. SCD and Historical Change

Not all data needs SCD, but dimensions that require historical semantics should be handled carefully.

### 8.1 When SCD Is Needed

SCD is suitable for:

* customer attribute history;
* account status history;
* product attribute changes;
* organizational structure changes;
* contract or subscription status;
* questions that require analysis based on the state at that time.

If the business question requires answering “what was the state at the time?”, historical state should be preserved.

### 8.2 When SCD Should Not Be Overused

Avoid overusing SCD in scenarios such as:

* high-volume event tables;
* append-only facts;
* temporary attributes that do not need historical semantics;
* states that can be rebuilt from events;
* low-value field changes.

SCD has costs: storage, merge logic, testing, query complexity, and comprehension cost.

### 8.3 Common Fields

A typical SCD2 dimension usually includes:

* surrogate key;
* natural key;
* valid_from;
* valid_to;
* is_current;
* record_hash;
* source_system;
* updated_at.

However, implementation should not mechanically follow a template. It should be driven by business needs.

---

## 9. Materialization Strategy: Not Every Model Needs Near Real-Time

Materialization strategy sits at the intersection of data modeling and FinOps.

The same logical model can be materialized as:

* view;
* table;
* incremental table;
* Dynamic Table;
* materialized view;
* stream-driven table;
* serving projection source.

The choice should be based on freshness, query pattern, cost, and complexity — not technical preference alone.

### 9.1 Common Materialization Options

| Materialization      | Suitable For                                 | Risk                                                      |
| -------------------- | -------------------------------------------- | --------------------------------------------------------- |
| View                 | Lightweight logic reuse, low-frequency query | May be slow or cost-unpredictable for frequent queries    |
| Table                | Stable reporting, daily or hourly refresh    | Not real-time; requires scheduling                        |
| Incremental table    | Large data volume, incremental processing    | Requires merge / dedupe logic                             |
| Dynamic Table        | Clear near real-time requirement             | Continuous refresh may increase cost                      |
| Materialized view    | Specific query acceleration                  | Limited applicability; maintenance cost must be evaluated |
| Stream + Task        | Explicit incremental logic                   | More complex than simple batch                            |
| Serving source model | Source for application point lookup          | Requires serving contract and consistency management      |

### 9.2 Dynamic Tables Are Not the Default Answer

Dynamic Tables are valuable for near real-time models, but they should not become the default materialization for all Gold models.

If a model only supports a daily report, or the business only checks it hourly, continuous refresh may add cost without adding business value.

To decide whether a model needs near real-time, ask:

* Does the consumer truly need minute-level freshness?
* Does the data change frequently enough?
* Is the query frequency high enough?
* Does lower latency change business decisions?
* Is the cost acceptable?
* Is there a simpler batch alternative?

### 9.3 Freshness Should Be Defined by Layer

Near real-time does not mean every layer in the path is equally real-time.

Freshness can be defined by layer:

* Bronze: latency from source change to raw layer;
* Silver: latency for business mirror update;
* Gold: latency for business metric refresh;
* Serving: latency before the application can read the new result.

Only then can the team identify where the freshness bottleneck actually is.

---

## 10. Data Contract: A Model Is Not Just a Table, but a Promise

A data model is more than a table structure. A mature model should be a data contract.

### 10.1 What a Contract Should Include

A Silver or Gold model should at least define:

* model purpose;
* owner;
* business definition;
* grain;
* primary key or uniqueness;
* freshness expectation;
* quality checks;
* PII classification;
* retention;
* materialization;
* cost attribution;
* downstream consumers;
* change policy.

Without this information, a model is just a table, not a reliable data product.

### 10.2 Grain Is One of the Most Important Fields

Many modeling problems come from unclear grain.

For example:

* Does one row represent one event?
* One order?
* One customer per day?
* One account current state?
* One aggregate result?

If grain is unclear, metrics can easily be double-counted or joined incorrectly.

### 10.3 Change Policy Matters

Data models will change.

Therefore, the platform needs to define:

* which changes are backward compatible;
* which changes require a new version;
* which consumers need notification;
* how field deletion is handled;
* how metric definition changes are recorded;
* whether historical backfill is required.

Without a change policy, the Gold layer gradually loses trust as the business evolves.

---

## 11. Data Quality: Test by Layer, Not as a Final Patch

Data quality should not only be checked at the Gold layer.

Different layers should focus on different tests.

### 11.1 Bronze Tests

Bronze should focus on:

* whether files or batches are loaded;
* whether technical fields exist;
* whether source offsets are continuous;
* whether operation types are valid;
* whether raw payloads are parseable;
* whether there are obvious format errors.

### 11.2 Silver Tests

Silver should focus on:

* primary key uniqueness;
* business key resolution;
* deduplication correctness;
* referential integrity;
* entity state validity;
* SCD continuity;
* not-null checks for core fields;
* accepted values.

### 11.3 Gold Tests

Gold should focus on:

* metric definitions;
* aggregation consistency;
* reconciliation with source models;
* freshness;
* completeness;
* consumer-facing constraints;
* PII exposure;
* performance expectations.

### 11.4 Quality Failures Need Owners

A failed test is only the beginning. Each critical test should have:

* owner;
* severity;
* expected response;
* exception process;
* downstream impact.

Otherwise, data quality becomes a collection of red lights that nobody acts on.

---

## 12. Modeling for Operational Serving

Gold does not only serve BI. It can also be the source for operational serving projections.

But these models need separate thinking.

### 12.1 Characteristics of Serving Source Models

A serving source model usually needs:

* key-based grain;
* explicit primary key;
* current state or latest snapshot;
* compact payload;
* freshness target;
* last_updated timestamp;
* snapshot version;
* change detection;
* reconciliation rule.

It is not a normal BI mart, and it is not a transaction table.

### 12.2 Serving Projections Should Not Become Sources of Truth

Data in an operational serving store should be rebuildable from Snowflake Gold.

This means:

* Gold remains the analytical source of truth;
* the serving store is a projection;
* applications read the projection;
* business transactional state remains owned by OLTP systems;
* the data platform does not directly write business OLTP.

### 12.3 Serving Models Need Contracts

If a Gold model is published to a serving store, it should have a clear contract:

* target consumer;
* key;
* payload schema;
* freshness;
* compatibility rule;
* reconciliation;
* failure handling;
* owner.

Otherwise, the serving projection can easily become a hidden API.

---

## 13. Naming and Organization Principles

Naming is not the most important thing, but consistent naming reduces long-term complexity.

### 13.1 Suggested Naming Thinking

Names should reflect:

* layer;
* domain;
* entity;
* grain;
* purpose.

For example, a team may use patterns such as:

```text
bronze_<source>__<entity>
silver_<domain>__<entity>
fact_<business_process>
dim_<entity>
gold_<business_output>
serving_<target>
```

This is only an example, not a mandatory standard.

The key is that the team can understand the model’s layer and purpose from its name.

### 13.2 Do Not Leak Implementation Details into Business Models

Gold model names should not overly depend on source system names, connector names, or temporary project names.

The business consumption layer should express business concepts, not source system implementation details.

### 13.3 Domain Ownership

Models are often easier to manage by business domain, such as:

* customer;
* product;
* finance;
* operations;
* risk;
* marketing;
* platform.

Domain ownership helps with ownership, quality, cost, and change management.

---

## 14. Core Trade-Offs

| Design Choice                      | Benefit                                   | Cost                                                             |
| ---------------------------------- | ----------------------------------------- | ---------------------------------------------------------------- |
| Use Bronze / Silver / Gold         | Clear responsibilities and lineage        | More layers; requires modeling discipline                        |
| Preserve raw payload in Bronze     | Supports schema drift and replay          | Storage and query access need governance                         |
| Build business mirrors in Silver   | Decouples OLTP from business semantics    | Requires understanding both source systems and business concepts |
| Unify business consumption in Gold | Consistent metrics and more stable BI     | Requires owners, contracts, and quality tests                    |
| Use fact / dimension modeling      | Strong reuse and analytical flexibility   | Higher modeling effort                                           |
| Use wide tables                    | Simple consumption and direct performance | Can become tightly coupled over time                             |
| Use Dynamic Tables                 | Supports near real-time                   | Continuous refresh may increase cost                             |
| Use batch tables                   | Simple and cost-controllable              | Lower freshness                                                  |
| Use Gold as serving source         | Reuses analytical platform outputs        | Requires serving contracts and consistency management            |
| Enforce data contracts             | Stability and governance                  | Adds development workflow overhead                               |

---

## 15. Common Anti-Patterns

| Anti-Pattern                                          | Problem                                      | Better Approach                                                                 |
| ----------------------------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------- |
| BI reads directly from Bronze                         | Business semantics are unstable              | Provide consumption models through Silver / Gold                                |
| Silver is only field renaming                         | No real business mirror is created           | Handle entities, keys, states, and quality in Silver                            |
| Gold becomes one table with every field               | Too tightly coupled and hard to maintain     | Design facts, dimensions, marts, or serving models based on consumption pattern |
| Each report implements its own metrics                | Metric definitions drift                     | Define core metrics in Gold                                                     |
| Every model is near real-time                         | High cost without clear value                | Choose materialization based on freshness and business value                    |
| Every historical field uses SCD                       | Storage and queries become complex           | Use SCD only for dimensions that need historical semantics                      |
| Model grain is unclear                                | Joins and aggregations become incorrect      | Define grain for every model                                                    |
| Model has no owner                                    | No one is responsible when issues happen     | Define an owner for each data product                                           |
| Contracts exist only in documents, not workflow       | Changes are not controlled                   | Include contracts in review and CI processes                                    |
| Serving projection has no version or freshness target | Application consumption becomes uncontrolled | Define serving contracts clearly                                                |

---

## 16. Success Criteria

A well-designed data modeling system should satisfy the following criteria:

* Bronze, Silver, and Gold responsibilities are clear;
* downstream consumers do not depend directly on OLTP schemas;
* Silver forms stable business mirrors;
* Gold carries core metrics and consumption models;
* each important model has explicit grain;
* key models have owners, freshness expectations, quality checks, and change policies;
* BI metrics mainly come from Gold, not scattered report logic;
* SCD is used only for dimensions that need historical semantics;
* near real-time models have clear business justification;
* materialization strategy matches freshness, cost, and query pattern;
* serving source models have contracts;
* data quality tests are designed by layer;
* source system changes do not directly break business consumption.

---

## 17. Closing Summary

Data modeling is the layer that connects technology and business in a lakehouse architecture.

Without a modeling layer, the platform is only moving data from source systems into Snowflake. If the modeling layer is poorly designed, complexity simply moves from source systems into BI tools, scripts, and application code.

The value of Medallion is semantic decoupling:

* Bronze preserves raw evidence;
* Silver converts OLTP storage structures into stable business mirrors;
* Gold extracts metrics, reports, data products, and serving source models from those business mirrors.

With this pattern, source systems can change, business logic can evolve, downstream consumption can remain stable, and governance and TCO become easier to manage.

A mature data model is not only queryable. It can be understood, tested, governed, migrated, decommissioned, and trusted by the business over time.
