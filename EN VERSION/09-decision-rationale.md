# 09 · Decision Rationale: Key Architectural Trade-Offs

## 1. Purpose

This document summarizes the most important architectural trade-offs in the Lakehouse Architecture Playbook.

The previous chapters discussed design goals, reference architecture, ingestion, data modeling, governance, FinOps, operational serving, and migration. This chapter does not repeat those designs in detail. Instead, it answers the more fundamental questions:

> Why does this playbook recommend these design choices? What problems do they solve? What do they trade off? When should they not be used? Under what conditions should they be re-evaluated?

My basic view is:

> Mature data platform design is not only about describing a recommended architecture. It is also about explaining why the architecture is recommended, and under what conditions the recommendation no longer holds.

This document does not position Snowflake or any other vendor as the only correct answer. Snowflake is used in this playbook as a reference platform. The real topic is how to make trade-offs among system complexity, data freshness, governance, security, TCO, migration risk, and team capability when building a low-complexity lakehouse architecture.

---

## 2. Overall Decision Principles

The decision logic of this playbook can be summarized in six principles.

### 2.1 Reduce System Complexity First

If a capability can be implemented reliably within the existing core platform, do not introduce a new system boundary by default.

Every additional system introduces another set of:

* permission models;
* deployment processes;
* monitoring and alerting;
* failure modes;
* on-call responsibilities;
* cost attribution problems;
* required skills;
* migration and decommissioning concerns.

Low complexity is not a slogan about using fewer tools. It is a prerequisite for long-term operability.

### 2.2 Let Business Freshness Decide Real-Time Architecture

Do not build a real-time platform simply because the words CDC, streaming, or event-driven are involved.

Ask first:

* How fresh does the business data actually need to be?
* Is minute-level freshness enough?
* Is hourly freshness enough?
* Does lower latency change business decisions?
* Is the business willing to pay the added complexity and cost of lower latency?

A full streaming architecture should be introduced only when the business SLA justifies true streaming.

### 2.3 Decouple OLTP Structures from Business Semantics

Source system schemas serve application transactions. They do not serve long-term analytical semantics.

The data platform therefore needs Medallion modeling to separate source structures, business mirrors, and business consumption.

### 2.4 Separate Analytical, Operational Read, and Transactional Workloads

Snowflake or a similar lakehouse platform is well suited to analytical workloads, but it should not be treated as an application OLTP API.

Application point lookups, transactional writes, real-time decisions, and complex event processing should be evaluated based on their own access patterns.

### 2.5 Design Governance and FinOps Upfront

Governance and cost are not merely post-launch operations. They are architectural boundary problems.

If PII, access control, auditability, ownership, freshness, data contracts, and cost attribution are not designed into the platform, they will return later in a more complex form.

### 2.6 Treat Decommissioning as the End of Migration

A new platform going live does not mean migration is complete.

Migration only reduces complexity when old pipelines are decommissioned, permissions are cleaned up, costs are released, and consumers have moved to the new path.

---

## 3. Decision 1: Why Use Snowflake as the Reference Analytical Core

### Recommendation

This playbook uses Snowflake as the reference analytical core to support:

* analytical storage;
* SQL transformation;
* Bronze / Silver / Gold modeling;
* BI-serving models;
* near real-time incremental processing;
* part of the orchestration layer;
* governance metadata;
* workload attribution.

### Why

The main reason is not that “Snowflake can do everything.” The main reason is that Snowflake can reduce system assembly complexity in many enterprise scenarios.

If raw data, transformations, metrics, BI models, incremental processing, access control, and cost attribution are spread across many systems, the platform accumulates large amounts of glue code and cross-system troubleshooting cost.

The value of a Snowflake-centric design is that it can:

* reduce system boundaries;
* unify modeling and consumption paths;
* reduce external orchestration and temporary scripts;
* centralize SQL-first data engineering;
* make governance and cost attribution easier to implement;
* reduce the number of runtimes the team must operate at the same time.

### What It Costs

The cost is not simply that “it is more expensive” or “governance is harder.”

The real cost is:

> The architecture becomes more dependent on Snowflake’s native capabilities and design boundaries. Workloads outside Snowflake’s sweet spot may require companion systems or separate architecture.

Examples include:

* millisecond-level event processing;
* hard real-time risk control;
* high-frequency streaming;
* complex event processing;
* large-scale online feature serving;
* application transactional writes.

These should not be forced into the Snowflake main path.

### When Not to Use It

An enterprise may need another core platform or a hybrid architecture if it has:

* strict open-source portability requirements;
* a streaming-first data platform strategy;
* large-scale ML or feature engineering as the primary priority;
* deep existing investment in Databricks, Fabric, BigQuery, Redshift, or another ecosystem;
* highly complex multi-cloud or active-active requirements;
* a team whose primary capability is not SQL-first data engineering.

### Re-Evaluation Signals

Re-evaluate the Snowflake-centric boundary if:

* many workloads must bypass Snowflake to meet SLA;
* the number of companion systems grows quickly;
* Snowflake-native orchestration cannot handle dependency complexity;
* true streaming becomes the primary workload rather than an exception;
* platform concentration cannot be explained through TCO;
* centralizing on Snowflake starts reducing delivery speed instead of improving it.

---

## 4. Decision 2: Why Use One Main Data Path

### Recommendation

Use one default main data path:

```text
Sources
  -> Landing
  -> Bronze
  -> Silver
  -> Gold
  -> BI / Analytics / Operational Serving
```

### Why

The value of a main path is reducing cognitive load.

If every source, report, and application consumer follows a different path, the platform becomes a collection of unexplained exceptions. When data issues occur, the team does not know which system, layer, or owner to inspect first.

The main path is not meant to restrict innovation. It provides a default explanation model for most scenarios.

### What It Costs

A unified main path may feel too heavy for some simple use cases. For example, small PoCs or low-value temporary data may not require a full Landing / Bronze / Silver / Gold flow.

### When Not to Use It

Do not force the main path onto:

* one-off exploration;
* temporary analysis;
* low-risk PoCs;
* true streaming-first scenarios;
* application transactional writes;
* extremely low-latency serving.

### Re-Evaluation Signals

If more and more scenarios need to bypass the main path, the main path may be too heavy, too slow, or misaligned with real business patterns.

---

## 5. Decision 3: Why Separate Landing and Bronze Responsibilities

### Recommendation

Landing and Bronze are modeled as two different responsibilities:

* Landing is the handoff and evidence layer for source data entering the platform;
* Bronze is the raw query layer after data enters the analytical platform.

In most enterprise scenarios, physically separating the two is clearer. However, in small-scale, low-risk, or specific table-format architectures, the two can be implemented together.

### Why

Landing answers:

* what the source system delivered;
* whether the original input can be replayed;
* how to recover from loading failures;
* whether governance is needed before data enters the platform;
* where audit and archive boundaries are.

Bronze answers:

* how data can be queried after entering the analytical platform;
* how to support incremental Silver processing;
* how source metadata is preserved;
* how downstream models avoid scanning object storage directly;
* how the platform provides a raw analytical layer.

These are different responsibilities.

### What It Costs

Physical separation adds:

* object storage management;
* metadata;
* lifecycle policies;
* loading paths;
* initial design complexity.

### When the Two Can Be Combined

They can be combined when:

* the work is a small-scale PoC;
* data value is low;
* the source system can reliably replay history;
* the architecture uses Iceberg, Delta, external tables, or another table format;
* replay and audit requirements are low;
* latency is extremely sensitive and the risk is acceptable.

### Re-Evaluation Signals

If Landing is only copying data without replay, audit, lifecycle, or governance value, its design should be re-evaluated.

If Bronze is heavily consumed directly by business users, the platform likely lacks a proper Silver / Gold layer.

---

## 6. Decision 4: Why CDC Is Not Real Time

### Recommendation

CDC should be understood as change awareness and incremental capture, not as end-to-end real-time consumption.

### Why

CDC only means the platform knows what changed in the source system.

Whether the business receives near real-time data also depends on:

```text
Change capture
  -> Landing
  -> Bronze load
  -> Silver processing
  -> Gold refresh
  -> BI refresh / serving publish
  -> downstream consumption
```

Any step in this path can define the final freshness.

### What It Costs

If CDC is not treated as real time, the platform must define end-to-end freshness explicitly. This adds measurement and communication effort, but avoids misunderstanding.

### When CDC Is Enough

CDC may be enough when the business only needs to know which inserts, updates, and deletes happened between batch loading windows, and use those changes for incremental loading, reconciliation, compensation, or recomputation.

### When True Streaming Is Needed

True streaming is needed when the business requires:

* event-by-event processing;
* millisecond-level response;
* complex event windows;
* real-time decisions;
* downstream systems consuming event streams directly;
* high-throughput, low-latency event processing.

In these cases, a streaming or event-driven architecture should be designed.

### Re-Evaluation Signals

If a near real-time pipeline is repeatedly asked to move toward second-level or millisecond-level latency, or if the business starts depending on event-level decisions, CDC + micro-batch may no longer be suitable.

---

## 7. Decision 5: Why the Core of Medallion Is Semantic Decoupling

### Recommendation

Use Bronze / Silver / Gold, but focus on responsibilities rather than names:

* Bronze preserves raw evidence;
* Silver builds business mirrors;
* Gold provides business consumption.

### Why

Source system schemas serve OLTP. They do not provide stable long-term analytical semantics.

If BI, metrics, and application consumers depend directly on source tables, source system changes continuously disrupt business consumption.

The real value of Medallion is decoupling two types of data work:

1. converting OLTP storage structures into stable business mirrors;
2. extracting metrics, reports, data products, and serving source models from those business mirrors.

### What It Costs

Layering increases modeling work.

The team needs to define:

* layer responsibilities;
* grain;
* owner;
* data contracts;
* quality checks;
* change policies.

Without modeling discipline, Medallion becomes only a set of layer names.

### When It Can Be Simplified

Layering can be simplified for temporary analysis, low-risk PoCs, or a single small data source.

But once data becomes part of long-term BI, business reporting, or application consumption, clear semantic layers are needed.

### Re-Evaluation Signals

If Silver is only field renaming and does not form business mirrors, Silver should be redesigned.

If Gold is just a collection of wide tables while metrics remain scattered in BI, Gold should be redesigned.

---

## 8. Decision 6: Why Not Every Model Should Be Near Real-Time

### Recommendation

Apply near real-time only to models with clear business value. It should not be the default refresh mode for every model.

### Why

Near real-time adds:

* higher refresh frequency;
* incremental state management;
* orchestration;
* observability;
* error handling;
* cost;
* consistency concerns.

If the business only checks the data daily or hourly, continuous refresh does not add value.

### What It Costs

Not all data will be equally fresh. The team must define different freshness expectations for different layers and models.

This requires both business and technical stakeholders to accept that different data products can have different freshness levels.

### When It Should Be Near Real-Time

Near real-time is suitable for:

* operational metrics;
* business state synchronization;
* risk signals;
* customer tags;
* high-frequency dashboards;
* operational serving sources;
* latency-sensitive scenarios that do not require millisecond-level response.

### Re-Evaluation Signals

If a near real-time model has low usage, high refresh cost, and no business concern about minute-level differences, it should be downgraded to batch.

If a batch model is repeatedly criticized for latency, it may need incremental or near real-time processing.

---

## 9. Decision 7: Why Dynamic Tables Are Not the Default Answer

### Recommendation

Dynamic Tables or similar continuous refresh mechanisms should be used as near real-time modeling tools, not as the default materialization for all Gold models.

### Why

Dynamic Tables can reduce incremental refresh complexity, but continuous refresh has a cost.

If model logic is complex, data changes frequently, and dependency chains are deep, continuous refresh may become expensive and difficult to troubleshoot.

### What It Costs

Not using Dynamic Tables by default means some models must be explicitly designed as batch tables, incremental tables, stream + task flows, or materialized views.

This adds design effort, but reduces unintentional cost.

### When It Fits

Dynamic Tables are suitable when:

* minute-level freshness is clearly required;
* dependency logic is understandable;
* consumers use the model frequently;
* cost is attributable;
* business value can be demonstrated.

### Re-Evaluation Signals

If Dynamic Table refresh cost grows quickly, consumer usage is low, and freshness has no business value, materialization should be re-evaluated.

---

## 10. Decision 8: Why Applications Should Not Query Snowflake Directly

### Recommendation

BI and analytics should read from Snowflake Gold. Application point lookups should read from operational serving projections.

### Why

Snowflake is well suited for analytical workloads: scans, aggregations, joins, historical analysis, BI, and ad hoc exploration.

Application point lookups are usually key-based, high-concurrency, low-latency, small-result-set requests. This is closer to a serving workload than an analytical workload.

Letting applications query Snowflake directly can create:

* unpredictable latency;
* small queries waking warehouses repeatedly;
* mixed BI and application workloads;
* expanded service account permissions;
* unclear ownership;
* application SLAs depending on an analytical runtime.

### What It Costs

Introducing serving projections adds:

* a serving store;
* a publisher;
* consistency monitoring;
* reconciliation;
* serving contracts;
* additional cost.

### When Direct Querying May Be Acceptable

Low-frequency internal tools, back-office operational queries, or non-critical low-concurrency scenarios may query Snowflake directly.

But this should be a bounded exception, not the default application architecture.

### Re-Evaluation Signals

If application queries to Snowflake increase in QPS, latency requirements, or business criticality, the workload should move to a serving projection or application-owned service.

---

## 11. Decision 9: Why the Serving Store Is a Projection, Not a Source of Truth

### Recommendation

An operational serving store should be a rebuildable projection published from Gold.

### Why

Projection has several benefits:

* it can be rebuilt from upstream models;
* it does not own transactional truth;
* definitions come from Gold;
* application read boundaries are clear;
* errors can be fixed by republishing;
* the data platform does not pollute OLTP systems.

If the serving store becomes a source of truth, the platform must take on new problems: write consistency, conflict handling, recovery, auditability, and ownership.

### What It Costs

Projection usually means eventual consistency.

The business must accept that there is latency between source change and serving readability.

### When It Does Not Fit

This pattern is not suitable for:

* strongly consistent transactional state;
* user transaction writes;
* authoritative account balances;
* real-time authorization;
* business rules that must take effect synchronously.

These should be handled by OLTP systems or specialized real-time systems.

### Re-Evaluation Signals

If applications begin writing state into the serving store, or if the business starts treating the serving projection as authoritative fact, ownership boundaries should be redesigned immediately.

---

## 12. Decision 10: Why the Data Platform Should Not Directly Write to Business OLTP

### Recommendation

The data platform may publish results, but it should not directly write to the transaction source of truth owned by business OLTP systems.

### Why

Directly writing from the data platform into OLTP creates:

* larger blast radius;
* unclear ownership;
* difficult rollback;
* credential risk;
* audit complexity;
* mixed responsibilities between application teams and data teams.

Recommended activation patterns include:

* serving stores;
* application-owned APIs;
* message queues;
* reverse ETL tools;
* manual approval workflows.

### What It Costs

A separate activation pattern must be designed instead of simply updating business tables from the data platform.

This adds some upfront design effort, but reduces production system risk.

### When Exceptions May Exist

If the business system provides a controlled API, and owner, audit, validation, and rollback are all managed by the business system, the data platform may act as one caller.

But this is still not “directly writing OLTP tables.”

### Re-Evaluation Signals

If data pipelines start owning business state write rules, the boundary has drifted and should be redesigned.

---

## 13. Decision 11: Why Governance Should Be Designed Upfront

### Recommendation

PII, RBAC, auditability, data quality, ownership, contracts, and retention should enter the architecture during design.

### Why

Governance added later leads to:

* PII spreading across layers;
* raw data being broadly accessed;
* permissions becoming hard to revoke;
* metrics having no owner;
* missing audit evidence;
* quality failures having no owner;
* temporary data products never being retired.

These problems are much harder to fix later than to design upfront.

### What It Costs

Upfront governance adds design and process cost.

But good governance should not mean heavy approval. It should mean clear boundaries, metadata, and ownership.

### When It Can Be Lightweight

PoCs, sandboxes, and low-sensitivity data can use lightweight governance, but they should not be silently promoted into production data products.

### Re-Evaluation Signals

If users start depending on ungoverned models, or if PII appears in unexpected layers, governance boundaries should be strengthened immediately.

---

## 14. Decision 12: Why FinOps Should Be Designed Upfront

### Recommendation

Introduce workload attribution, warehouse strategy, query tagging, materialization discipline, and monthly review from the architecture design phase.

### Why

Snowflake or similar platforms can create value quickly, but they can also create unexplained cost quickly.

Without cost attribution, the team only sees the total bill and cannot answer:

* which workload costs the most;
* which domain is growing fastest;
* which BI refresh jobs have low value;
* which near real-time models are over-engineered;
* which migration costs are temporary dual-run costs;
* which old systems have not been decommissioned.

### What It Costs

Upfront FinOps requires engineering discipline: query tags, owners, metadata, reviews, and guardrails.

### When It Can Be Lightweight

Early PoCs can use lightweight FinOps. Production platforms need cost attribution.

### Re-Evaluation Signals

If Snowflake cost grows but the source cannot be explained, or platform value cannot be tied to business use cases, FinOps is insufficient.

---

## 15. Decision 13: Why Migration Should Not Be Defined by Go-Live Alone

### Recommendation

Manage migration through the following lifecycle:

```text
Discover
  -> Prioritize
  -> Rebuild
  -> Dual-run
  -> Validate
  -> Cutover
  -> Shadow
  -> Decommission
```

### Why

The hard part of data platform migration is not building the new platform. It is safely moving consumers and actually retiring the old one.

Without decommissioning, the new platform only adds complexity.

### What It Costs

Incremental migration takes longer than Big Bang. It requires dual-run, parity, communication, and governance.

But it reduces business risk.

### When It Can Be Simplified

Low-risk, dependency-free, low-value objects can be migrated and retired quickly.

Core business, financial, compliance, and application-dependent objects require more careful migration.

### Re-Evaluation Signals

If dual-run has no exit criteria, or if the old platform continues to grow after the new platform goes live, migration governance has failed.

---

## 16. Decision Matrix

| Decision         | Recommended Direction                                              | Core Reason                                            | Main Cost                                  | Re-Evaluation Signal                                 |
| ---------------- | ------------------------------------------------------------------ | ------------------------------------------------------ | ------------------------------------------ | ---------------------------------------------------- |
| Analytical core  | Snowflake as reference core                                        | Reduces system assembly complexity                     | Depends on Snowflake capability boundaries | Many workloads need to bypass Snowflake              |
| Main data path   | Sources → Landing → Bronze → Silver → Gold → Consumption           | Reduces cognitive load                                 | May feel heavy for simple scenarios        | Too many exception paths appear                      |
| Landing / Bronze | Separate responsibilities; physical implementation may be combined | Clear replay, audit, and governance boundaries         | More management overhead                   | Landing has no actual value                          |
| CDC              | Change awareness, not real time                                    | Avoids overbuilding streaming                          | Requires end-to-end freshness definition   | Business needs event-level SLA                       |
| Medallion        | Semantic decoupling                                                | OLTP can change while business semantics remain stable | Requires modeling discipline               | Silver / Gold responsibilities are unclear           |
| Near real-time   | Use based on business value                                        | Reduces unnecessary refresh cost                       | Freshness is not uniform                   | Business complains about latency or cost is too high |
| Dynamic Tables   | Use for clear near real-time needs                                 | Simplifies incremental refresh                         | Continuous refresh cost                    | Low usage and high cost                              |
| Operational read | Serving projection                                                 | Avoids direct application access to Snowflake          | Adds serving path                          | Application QPS or SLA increases                     |
| Serving store    | Projection, not source of truth                                    | Rebuildable and auditable                              | Eventual consistency                       | Serving is treated as authoritative fact             |
| Reverse ETL      | Do not directly write OLTP                                         | Reduces blast radius                                   | Requires activation pattern                | Data platform owns business write logic              |
| Governance       | Design upfront                                                     | Reduces permission and PII debt                        | Initial process cost                       | Raw or PII data spreads unexpectedly                 |
| FinOps           | Upfront attribution                                                | Cost becomes explainable                               | Requires engineering discipline            | Cost growth cannot be explained                      |
| Migration        | Decommissioning is the finish line                                 | Actually reduces complexity                            | Longer lifecycle                           | Permanent dual-run                                   |

---

## 17. Common Bad Decision Patterns

### 17.1 Tool-Driven Instead of Problem-Driven

Bad pattern: introducing a tool because it is popular.

Better approach: first define workload, SLA, ownership, governance, and TCO, then select tools.

### 17.2 Treating Local Optimization as Global Optimization

A local solution may be fast, but it can increase overall complexity.

For example, a temporary serverless job may solve one need while adding a new permission model, monitoring path, failure mode, and owner.

### 17.3 Using Real Time to Solve Semantic Problems

Some problems are not caused by data being too slow. They are caused by unclear business definitions, unstable data models, or untrusted quality.

Faster refresh will not fix semantic confusion.

### 17.4 Using Governance Patches to Fix Architectural Confusion

If model layers are chaotic, adding more views, masking policies, and permissions later only increases complexity.

Governance should be designed together with architecture.

### 17.5 Building the New Platform Without Removing the Old One

This is one of the most common modernization failure modes.

After the new platform goes live, the old platform keeps running, and the team now has double the complexity.

---

## 18. How to Use This Decision Rationale

This document can be used for:

* architecture review;
* new data platform design;
* migration roadmap discussion;
* vendor or platform comparison;
* near real-time requirement evaluation;
* governance and FinOps design;
* operational serving review;
* deciding whether a requirement belongs in the Snowflake main path.

The point is not to apply every conclusion mechanically. Instead, use the structure of each decision:

```text
Recommendation
  -> Why
  -> Cost
  -> When not to use
  -> Re-evaluation signals
```

If a project has different conditions, the conclusion can be different. But the reason should be explicit.

---

## 19. Closing Summary

The core of this playbook is not Snowflake, Medallion, CDC, Dynamic Tables, or serving stores.

The core is architectural judgment:

* when to consolidate system boundaries;
* when to introduce additional systems;
* when near real-time is sufficient;
* when true streaming is required;
* when Snowflake is a suitable core;
* when Snowflake should only be a downstream analytical sink;
* when serving projection solves the problem;
* when the requirement must go back to OLTP or a specialized real-time system;
* when governance should be designed upfront;
* when migration is truly complete.

My final view is:

> A good data platform architecture does not put every capability into one system, nor does it connect every possible tool. It makes clear choices at each boundary and can explain the business value, technical cost, and re-evaluation conditions behind those choices.

That is the core idea of this Lakehouse Architecture Playbook: use low-complexity system design to support a data platform that is timely enough, governable, migratable, and explainable from a TCO perspective.
