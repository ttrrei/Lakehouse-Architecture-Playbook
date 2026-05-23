# 01 · Design Goals: Reduce Complexity, Do Not Stack Tools

## 1. Purpose

This document defines the design goals of the Snowflake Lakehouse Playbook.

It does not explain how to configure Snowflake, how to write SQL, or how to build specific pipelines. It also does not try to prove that Snowflake is the only answer to every data platform problem. Instead, it focuses on a more fundamental question:

> When designing a Snowflake-based lakehouse platform, what problems are we actually trying to solve? Which types of complexity should we remove? Which types of complexity must we accept? How should we trade off data freshness, governance, security, cost, and operability?

My basic view is:

> The hardest problem in modern data platforms is often not the lack of tools. It is that after years of accumulated system complexity, the team can no longer reliably understand, govern, evolve, and decommission the platform.

Therefore, the goal of this playbook is not to connect every advanced tool into the architecture. The goal is to design a data platform that is simple enough to operate, clear enough to govern, realistic enough to migrate to, and explainable enough from a TCO perspective.

Snowflake is an important choice in this playbook, but it is not a belief system. The reason to consider a Snowflake-centric architecture is to consolidate much of the analytical data platform into one core system, reducing cross-system orchestration, duplicated storage, scattered scheduling, fragmented permissions, and troubleshooting complexity.

---

## 2. Why System Complexity Is the Core Problem

Data platform complexity does not usually appear all at once. It is usually the result of years of incremental decisions.

At first, a team may build a simple sync script for one report. Later, it adds a scheduler, a warehouse, a BI dataset, a data quality check, a data export job, a reverse ETL job, a temporary serverless function, and a manual backfill process. Each decision may be reasonable in isolation, but together they create a chain that becomes difficult to explain and operate.

This complexity typically comes from several directions.

### 2.1 Data Pipeline Complexity

Different source systems often use different ingestion patterns: database CDC, files, APIs, third-party feeds, manual exports, and application event streams. Each source may have its own naming convention, schedule, error handling, replay mechanism, and owner.

As a result, adding a new data source no longer means reusing a mature pattern. It becomes another architecture design exercise.

### 2.2 Scheduling and Orchestration Complexity

Data processing may be spread across many places: database jobs, Python scripts, Airflow DAGs, cloud functions, CI/CD workflows, BI refresh jobs, and application background tasks.

When data is late or incorrect, the team has to inspect logs, permissions, retries, and execution states across multiple systems. Cross-system troubleshooting becomes a cost in itself.

### 2.3 Semantic and Modeling Complexity

If reports, BI models, data exports, and application consumers all depend directly on source system tables, business logic becomes scattered across many layers. When a source field changes, downstream consumers are affected immediately. When a business definition evolves, it becomes unclear which layer should be changed.

This is why I believe the value of Medallion architecture is not the names Bronze / Silver / Gold themselves, but the separation of two types of data work:

1. converting OLTP storage structures into relatively stable business mirrors;
2. extracting metrics, reports, data products, and operational projections from those business mirrors.

Once these two operations are decoupled, OLTP systems can evolve, business logic can be upgraded, and downstream consumers no longer need to absorb every structural change from the source systems.

### 2.4 Governance and Access Complexity

Governance is not only about data privacy. It also includes who can access which data, which fields are PII, which models are approved for BI, which data products can be consumed by applications, which outputs can be activated back into business workflows, which queries require auditability, and which data needs to be retained or deleted.

If governance is added after the platform is already live, complexity grows quickly. Governance logic becomes scattered across SQL, BI tools, scripts, IAM policies, application code, and manual processes. Eventually, it becomes difficult to prove that the platform is controlled.

### 2.5 Cost and TCO Complexity

In many data platforms, the cost problem is not simply that one service is expensive. The real problem is that no one can explain where the cost comes from.

If queries are not attributed, warehouses have no ownership, pipelines have no business use case, BI refresh jobs have no cost visibility, and near real-time models have no materialization discipline, the monthly bill becomes a debate rather than a management signal.

More importantly, TCO is not just the cloud bill. It also includes team skills, system integration, incident response, migration maintenance, security review, auditability, and long-term evolution.

### 2.6 Migration Complexity

Many new platforms fail not because the new architecture is impossible, but because the old platform has no exit mechanism.

Without dual-run, parity checks, cutover, shadow mode, and decommissioning, the old platform remains after the new one goes live. The result is not modernization. It is permanent dual-platform complexity.

Modern cloud platforms provide many managed services and integration tools. This is a good thing. They allow teams to quickly support specialized requirements: one source system can use a managed CDC service, one file source can use an object storage trigger, one business process can use a serverless function, one report refresh can use an external scheduler, and one application consumer can be supported by a temporary API.

Each of these choices may be reasonable locally, and they often accelerate short-term delivery. But without a unifying architecture principle, they also make it easier for inconsistent designs to be quickly integrated into the platform. Short-term delivery improves, but long-term technical debt increases: more permission models, more scheduling entry points, more logging locations, more failure modes, and more temporary chains that nobody wants to decommission.

In other words, cloud convenience does not automatically reduce complexity. It reduces the effort required to create new components. Without a clear main path and governance boundaries, the core complexity problem can become even more severe.

---

## 3. Design Goals of This Architecture

The design goals of this playbook can be summarized in seven points.

### 3.1 Reduce Cognitive Load with One Main Data Path

The platform should have one main path that is easy to explain:

```text
Sources
  -> Landing
  -> Bronze
  -> Silver
  -> Gold
  -> BI / Analytics / Operational Serving
```

The purpose of this path is not to restrict every possible implementation. It is to provide a default mental model. Most data sources, most transformations, and most consumption patterns should be explainable through this path.

If every source system, every report, and every application consumer requires a special path, the platform will eventually become unmanageable.

### 3.2 Use Snowflake-Native Capabilities Where Reasonable

If an analytical data platform has chosen Snowflake as its core, many capabilities should first be evaluated through Snowflake-native options:

* data loading;
* SQL transformation;
* incremental processing;
* Dynamic Tables;
* Streams;
* Tasks;
* data quality checks;
* workload isolation;
* cost attribution;
* access control.

This is not because Snowflake is always better than every other tool. It is because fewer system boundaries usually make long-term operations simpler.

Every additional system should trigger a few questions:

* Does it solve a problem that Snowflake cannot reasonably solve?
* Is the added complexity justified by business value?
* Can the team operate it sustainably?
* Will it add permissions, monitoring, scheduling, troubleshooting, and TCO overhead?

### 3.3 Define the Boundary of Near Real-Time

This architecture focuses on near real-time, not every possible meaning of real time.

Here, near real-time mainly means minute-level or small-batch incremental refresh. This is suitable for many use cases: operational metrics, customer state, risk signals, BI acceleration, business tags, and data product synchronization.

But it does not mean millisecond-level streaming, real-time matching, hard real-time risk control, high-frequency trading, or complex event processing.

CDC is an important capability for near real-time systems, but CDC is not the same as real time. CDC lets the platform know what changed between two batch processing windows or two loading runs. It solves change awareness and incremental capture, not end-to-end real-time consumption.

Some scenarios do need CDC because the team needs to understand which inserts, updates, and deletes happened between batch loads, and then use those changes for incremental loading, reconciliation, compensation, or recomputation. But that does not automatically mean the platform needs a full Kafka / Flink / streaming stack.

The design goal should be to let the business freshness requirement determine the architecture, rather than letting the existence of CDC imply a full real-time platform.

### 3.4 Decouple OLTP Structure from Business Logic

Source system OLTP schemas are designed for application transactions, not for long-term semantic stability in an analytical platform.

A data platform should not make every report, metric, and downstream consumer depend directly on OLTP table structures. A better pattern is:

* Bronze preserves source evidence;
* Silver builds stable business mirrors;
* Gold carries business metrics and consumption models.

This allows source systems to change their schema for application needs, while the data team can independently evolve business logic, metric definitions, and data products.

### 3.5 Move Governance and Privacy Upfront

Governance should not be a final checklist item.

The design should consider upfront:

* where PII is identified;
* which data can enter the common landing zone;
* which fields need hashing, tokenization, or isolation;
* which roles can access which layers;
* which models are approved for BI;
* which data can be used for operational serving;
* which access patterns require auditability;
* which data products need owners and contracts.

Upfront governance is not about slowing delivery. It is about avoiding uncontrolled privacy, access, and audit debt after the platform is already in production.

### 3.6 Make Cost Attributable to Workloads and Business Use Cases

Snowflake cost governance should not rely only on the monthly bill.

The platform should consider from the beginning:

* which warehouses serve which workloads;
* how queries are tagged;
* how BI refresh jobs are attributed;
* whether a Dynamic Table is truly necessary;
* whether a near real-time model is worth continuous refresh;
* how ad hoc queries are isolated;
* how cost maps to domains, models, pipelines, owners, and use cases.

The goal is not to minimize the cost of every individual query. The goal is to make cost explainable, manageable, and optimizable.

### 3.7 Support Incremental Migration Instead of Big Bang

Data platform migration is risky when it is designed as a one-time cutover.

A new lakehouse should support:

* dual-run between old and new paths;
* data parity checks;
* gradual consumer cutover;
* shadow periods;
* rollback paths;
* legacy decommissioning.

Without a decommissioning mechanism, migration often does not replace the old platform. It simply adds a new one next to it.

---

## 4. Why Consider a Snowflake-Centric Lakehouse

The main reason to consider a Snowflake-centric architecture is not that “Snowflake can do everything.” It is that Snowflake can reduce system assembly complexity for many small, mid-sized, and large data teams.

This does not mean Snowflake is the only reasonable choice. Depending on enterprise requirements, technical stack, team capability, data scale, cloud preference, governance needs, and cost model, a company may also choose Databricks, Microsoft Fabric, Amazon Redshift, Google BigQuery, or another lakehouse / warehouse architecture.

The real question is not which brand is selected. The real question is whether the platform reduces overall complexity, supports governance and TCO, and can be operated sustainably by the team.

### 4.1 Unified Analytical Storage and Compute

Snowflake can support raw, cleaned, conformed, and business-ready layers. Teams can build a unified development and consumption experience around SQL, tables, views, dynamic tables, tasks, and access controls.

This is easier to govern than scattering data across multiple databases, object stores, scripts, cache layers, and BI datasets.

### 4.2 Less External Scheduling and Glue Code

A lot of platform complexity comes from glue code: a serverless function triggers one job, a script calls another system, a scheduler manages Snowflake jobs, or a CI workflow becomes part of production orchestration.

If major transformations, incremental processing, and scheduling can stay inside the Snowflake runtime, the platform has fewer failure surfaces and a clearer troubleshooting path.

### 4.3 SQL-First Data Engineering

Most analytical transformations, modeling logic, metrics, and data quality checks can be expressed in SQL. A Snowflake-centric architecture allows more logic to live inside the data platform rather than being scattered across application code or external scripts.

This matters for long-term maintainability.

### 4.4 Easier Governance and Cost Attribution

When workloads are concentrated in a smaller number of platform objects, governance and cost attribution become easier to implement.

If queries, models, tasks, warehouses, and roles are visible in the same platform, it becomes easier to establish ownership, tagging, auditability, quality checks, and cost attribution.

### 4.5 Lower Overall TCO

Choosing one core system does not always minimize every line item in the bill, but it may reduce total cost of ownership.

One fewer scheduler, one fewer streaming cluster, one fewer set of custom scripts, one fewer permission model, one fewer monitoring and on-call surface — each reduces long-term complexity.

This is why TCO should not be judged only by the price of one tool. It should be evaluated through the platform’s operating model.

---

## 5. What a Snowflake-Centric Architecture Should Not Try to Solve

This design does not attempt to collapse every data system into Snowflake.

The following scenarios should not default to the Snowflake lakehouse path:

* millisecond-level event processing;
* high-frequency trading or real-time matching;
* hard real-time risk control;
* extremely high-throughput streaming;
* complex event processing;
* low-latency online inference;
* large-scale online feature serving;
* application transactional writes;
* strong multi-region active-active requirements;
* architectures with strict open-source portability requirements.

These scenarios should be designed based on their own SLAs, throughput, latency, reliability, and team capability.

A mature architecture does not push every problem into one platform. It knows which problems belong inside the core path and which problems should stay outside.

---

## 6. Summary of Design Principles

### Principle 1: Reduce System Boundaries by Default

Do not introduce a new system for every capability unless there is a clear business reason.

### Principle 2: Use the Main Data Path as the Default Mental Model

Sources → Landing → Bronze → Silver → Gold → Consumption should explain most use cases.

### Principle 3: CDC Is Not Real Time

CDC solves change capture. It does not automatically solve end-to-end real-time consumption. True streaming should be driven by business SLA.

### Principle 4: Medallion Decouples OLTP from Business Semantics

Bronze preserves evidence, Silver builds business mirrors, and Gold supports business consumption.

### Principle 5: Snowflake Is an Analytical Center, Not an OLTP API

Application point lookups and transactional writes should not be placed directly on the Snowflake analytical path.

### Principle 6: Governance and FinOps Must Be Designed Upfront

PII, permissions, auditability, quality, cost attribution, and ownership should be considered at design time.

### Principle 7: TCO Matters More Than Individual Tool Cost

Number of systems, team skills, troubleshooting, monitoring, migration, and governance are all part of platform cost.

### Principle 8: Migration Must Have an Exit Mechanism

Dual-run without decommissioning becomes permanent complexity.

---

## 7. Core Trade-Offs

| Goal                              | Design Choice                                                                  | Benefit                                                   | Cost                                         |
| --------------------------------- | ------------------------------------------------------------------------------ | --------------------------------------------------------- | -------------------------------------------- |
| Reduce complexity                 | Use Snowflake as the analytical center                                         | Less cross-system orchestration and duplicated governance | Higher platform concentration                |
| Support near real-time            | Combine CDC, micro-batch, Dynamic Tables, Streams, and Tasks where appropriate | Supports minute-level freshness                           | Not suitable for millisecond-level use cases |
| Decouple source systems           | Introduce Landing and Bronze                                                   | Replayable, auditable, and resilient to source changes    | Additional storage and metadata management   |
| Stabilize business semantics      | Use Silver business mirrors                                                    | OLTP can change; business logic can evolve                | Requires modeling discipline                 |
| Serve business consumption        | Use Gold data products                                                         | More consistent metrics and reporting                     | Requires owners and contracts                |
| Support application point lookups | Use operational serving projections                                            | Avoids direct application access to Snowflake             | Adds a serving store                         |
| Control cost                      | Emphasize TCO and attribution                                                  | Cost becomes explainable and optimizable                  | Requires continuous operating discipline     |
| Reduce migration risk             | Use dual-run and cutover                                                       | Verifiable and reversible                                 | Higher short-term cost and effort            |

---

## 8. Success Criteria

A well-designed Snowflake-centric lakehouse should not be evaluated only by whether it goes live.

More important success criteria include:

* new data sources can reuse a standard ingestion pattern;
* most analytical logic can be explained through Snowflake layered models;
* BI metrics mainly come from Gold, not scattered report logic;
* PII, access control, and audit boundaries are clear;
* near real-time scenarios have explicit freshness definitions;
* scenarios that do not need true streaming are not over-engineered;
* application point lookups do not directly access Snowflake analytical models;
* cost can be attributed by workload, domain, model, and use case;
* legacy pipelines have clear cutover and decommission paths;
* the team can operate the platform reliably without depending on a few people’s memory.

---

## 9. Common Anti-Patterns

| Anti-Pattern                                              | Problem                                                     | Better Approach                                            |
| --------------------------------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------- |
| Building a full streaming stack just because CDC exists   | Confuses change capture with end-to-end real time           | Let business freshness requirements determine architecture |
| Designing a unique ingestion path for every source        | Long-term operations and troubleshooting become complex     | Use a unified landing and ingestion pattern                |
| Letting BI depend directly on OLTP structures             | Source changes directly break reports                       | Use Silver to build business mirrors                       |
| Putting all business logic in BI tools                    | Metrics drift across reports                                | Define core metrics in Gold                                |
| Letting applications query Snowflake for point lookups    | Cost, latency, and workload boundaries become unpredictable | Use operational serving projections                        |
| Letting the data platform directly write to business OLTP | Large blast radius and unclear ownership                    | Use controlled reverse ETL, serving, or API patterns       |
| Adding governance after go-live                           | PII, permission, and audit debt expands                     | Design governance upfront                                  |
| Looking only at tool bills, not TCO                       | Underestimates operational and complexity cost              | Establish workload attribution and FinOps practices        |
| Launching a new platform without retiring the old one     | Creates permanent dual-platform complexity                  | Design cutover, shadow, and decommissioning                |

---

## 10. Closing Summary

The design goal of this playbook can be summarized in one sentence:

> Use as few systems as reasonably possible, with boundaries as clear as possible, to build a data platform that is timely enough, governable, cost-explainable, and migratable.

The value of a Snowflake-centric lakehouse is not that every data problem is handed to Snowflake. Its value is that, in suitable scenarios, the core analytical data platform can be consolidated, reducing long-term system complexity.

What matters is not whether the platform is “real time,” but what level of freshness the business actually needs. What matters is not whether the architecture uses more tools, but whether those tools reduce or increase overall complexity. What matters is not whether a new platform goes live, but whether old complexity is actually removed.

The following chapters expand on these questions: how to design the reference architecture, how the landing layer decouples ingestion, how Medallion modeling manages semantic evolution, how governance and privacy should be designed upfront, how FinOps connects to TCO, how operational serving avoids misuse of Snowflake, and how migration can truly converge.
