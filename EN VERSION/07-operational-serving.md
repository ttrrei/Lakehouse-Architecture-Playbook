# 07 · Operational Serving: Don't Use Snowflake as an Application API

## 1. Purpose of This Document

This document discusses operational serving design in lakehouse architecture, focusing on **why applications should not query Snowflake directly for high-concurrency point lookups, why a serving store should be treated as a projection rather than a source of truth, and how a data platform can support operational data activation with low complexity**.

This is not a usage guide for DynamoDB, DocumentDB, Redis, API Gateway, or reverse ETL tools. The focus here is on more general system design questions:

> When Snowflake or a similar platform has already computed derived results with business value, how do we make those results available to application systems and business processes in a safe, stable, and auditable way — without turning the analytics platform into an OLTP API, and without having the data platform write directly to business OLTP?

My fundamental position is:

> Snowflake is well-suited as an analytical source of truth, but should not be designed as a high-concurrency application point-lookup backend. Operational serving should be a rebuildable projection published from Gold, designed to serve key-based, low-latency, application-side read scenarios.

This does not mean all operational activation must use a specific database. Document databases, key-value stores, search indexes, cache-backed services, application-owned APIs, message queues, and reverse ETL tools may all be appropriate. The key is to define clear boundaries:

- Snowflake / Gold owns derived results and business semantics;
- Serving projections own the application read shape;
- Business OLTP systems continue to own transactional facts;
- The data platform does not pollute the business transaction source of truth.

---

## 2. Why Operational Serving Is Needed

In a lakehouse architecture, the Gold layer typically produces a great deal of data with business value:

- Customer status;
- User tags;
- Risk signals;
- Recommendation results;
- Eligibility determinations;
- Operational prompts;
- Account summaries;
- Financial estimates;
- Product usage status;
- Lifecycle stage.

This data is not only for BI and analysts. In many cases, application systems, operations tools, customer service platforms, marketing systems, and risk control systems also need to read it.

But this type of read is not necessarily an analytical query.

### 2.1 Analytical Consumption

Analytical consumption typically includes:

- BI dashboards;
- Management reporting;
- Ad hoc SQL;
- Historical analysis;
- Metric exploration;
- Reconciliation;
- Finance and operations reporting.

These queries are generally scans, aggregations, joins, filters, sorts, and multi-dimensional analysis. Snowflake is well-suited to this type of workload.

### 2.2 Operational Read

Operational reads look more like application point lookups.

Typical questions are:

- What tags does a specific user currently have?
- Does a specific account meet a particular rule?
- What is the current state of a specific business object?
- Should a specific customer see a particular prompt?
- What is the latest derived result for a specific entity?

These requests typically have the following characteristics:

- Key-based;
- Single-record or small-batch reads;
- High concurrency;
- Low latency;
- Stable response shape;
- Usually triggered by an application backend;
- Repeated access patterns.

This type of workload is not suited to being sent directly to Snowflake Gold.

---

## 3. Why Applications Should Not Query Snowflake Directly

Having applications query Snowflake directly looks simple: the data is already in Gold, and the application just needs to look it up by key.

But from a system design perspective, this is typically a dangerous boundary to cross.

### 3.1 Workload Pattern Mismatch

Snowflake is an analytical platform, optimized for large-scale scans, aggregations, joins, and complex SQL.

Application point lookups care more about:

- Millisecond to low-hundreds-of-milliseconds response times;
- High-concurrency stability;
- Predictable latency;
- Fixed access patterns;
- Key-value or bounded queries;
- API-level availability.

These two workload types have different optimization goals.

Placing operational point lookups inside Snowflake makes it easy for analytical workloads and application serving workloads to interfere with each other.

### 3.2 Unpredictable Cost and Latency

Application requests typically involve large numbers of small queries, high-frequency access, and user-behavior-driven patterns.

If these requests are sent directly to Snowflake, several issues may arise:

- Warehouses are frequently woken up by small queries;
- Query queues grow;
- BI and transformation workloads are affected;
- Result cache hit rates become unstable;
- Cost attribution becomes more complex;
- Latency fails to meet application SLAs.

This is not necessarily a Snowflake problem — it is a workload placed in the wrong location.

### 3.3 Unclear Ownership Boundaries

If applications depend directly on Snowflake Gold, many ownership questions become blurred:

- Who is responsible for application API SLAs?
- Who guarantees Snowflake warehouse availability?
- How are Gold schema changes communicated to applications?
- Will application retries amplify query pressure?
- Do data model owners need to take responsibility for application production incidents?
- Do application teams need to understand Snowflake permissions and query performance?

These questions cause analytical platform and application platform ownership to become entangled.

### 3.4 Expanded Security and Permission Surface

Applications querying Snowflake directly typically means application service accounts need access to analytical platform objects.

If boundary controls are insufficient, this can lead to:

- Service accounts with excessive permissions;
- Applications accessing fields they should not;
- BI and application permission models being mixed together;
- Unclear data export boundaries;
- Expanded audit scope;
- Analytical platform exposure in the event of credential compromise.

An operational serving projection can constrain the application read surface to a narrower, more explicitly defined data structure.

---

## 4. The Basic Pattern for Operational Serving Projection

The core pattern for operational serving projection is:

```
Gold model
  -> incremental refresh / change detection
  -> publisher
  -> serving store
  -> application point lookup
```

In this pattern, Gold remains the analytical source of truth. The serving store is simply a projection published for application reads.

### 4.1 What Projection Means

A projection is not a new source of facts.

It has several defining characteristics:

- Generated from Gold or an approved source model;
- Can be republished;
- Can be cleared and rebuilt;
- Has a clearly defined key;
- Has a clearly defined payload schema;
- Has a clearly defined freshness;
- Has a clearly defined consumer;
- Has a publish log and error handling.

If the data in a serving store cannot be rebuilt from upstream, it is no longer a projection — it has become a new source of truth. This significantly increases governance and recovery complexity.

### 4.2 A Typical Data Flow

A typical serving flow might look like:

```
Silver business mirror
  -> Gold serving source model
  -> detect changed keys
  -> publish changed payloads
  -> serving store
  -> application backend reads by key
```

The key concept here is **changed keys**.

If a table only needs to update a small number of changed objects, the serving store should not be fully rewritten on every run. Full refresh is simple, but in near real-time scenarios it can introduce unnecessary compute, write capacity, latency, and failure risk.

### 4.3 Serving Store Options

Serving stores can take several forms:

| Type | Suitable For | Considerations |
|---|---|---|
| Key-value store | High-concurrency point lookup by key | Not suited for complex filters and joins |
| Document database | JSON payloads, object state, flexible fields | Schema evolution governance is important |
| Search index | Search, filtering, text queries | Not a source of truth; requires rebuild capability |
| Cache-backed service | Low-latency reads, short TTL | Requires a cache invalidation strategy |
| Application-owned API | Business system controls access and write rules | Data platform should not bypass business system ownership |
| Message queue | Async activation and event notification | Consumers need idempotency and retry |
| Reverse ETL tool | SaaS / CRM / marketing activation | Write rules and audit boundaries must be clearly defined |

There is no single correct answer here. The choice depends on access pattern, SLA, data structure, consistency requirements, team capabilities, and governance requirements.

---

## 5. Serving Contract

Operational serving cannot rely on verbal agreements alone. Every serving projection should have a contract.

### 5.1 What a Contract Should Include

A serving contract should define at minimum:

- Business purpose;
- Source Gold model;
- Target serving store;
- Owner;
- Consumer team;
- Primary key;
- Optional sort key or secondary index;
- Payload schema;
- Field meanings;
- Nullable rules;
- Freshness target;
- Availability expectation;
- Compatibility rules;
- Security classification;
- Publish pattern;
- Reconciliation requirements;
- Failure handling;
- Deprecation policy.

A serving projection without a contract easily becomes an invisible API.

### 5.2 Key and Grain Must Be Explicit

The most important elements in operational serving are key and grain.

These must be explicitly defined:

- What does one record represent?
- Is it the current state of a user?
- The current snapshot of an account?
- The latest summary of an order?
- A set of tags for a business entity?
- An aggregation within a time window?

If the grain is unclear, applications are likely to misuse the projection.

### 5.3 Payload Should Be Designed for the Consumption Scenario

Serving payloads should not simply copy a Gold table.

They should be designed based on the application read scenario:

- As few fields as possible;
- Stable structure;
- Avoiding complex joins on the application side;
- Including a `last_updated` timestamp;
- Including a snapshot version;
- Including necessary status and explanatory fields;
- Excluding unnecessary PII.

Gold models can be rich, but serving payloads should be restrained.

---

## 6. Ownership Boundaries

The success of operational serving depends heavily on whether ownership is clearly defined.

### 6.1 What the Data Team Is Responsible For

The data team is typically responsible for:

- Gold source model;
- Serving source model;
- Change detection;
- Publisher;
- Publish audit log;
- Publish error handling;
- Data quality;
- Freshness monitoring;
- Reconciliation;
- Data definitions within the serving contract.

### 6.2 What the Application Team Is Responsible For

The application team is typically responsible for:

- API layer;
- UI behavior;
- Cache strategy;
- Reader access patterns;
- Application retry;
- Fallback behavior;
- User-facing SLA;
- How data is presented to users.

### 6.3 What the Platform / Security Team Is Responsible For

The platform or security team is typically responsible for:

- Serving store provisioning;
- IAM and access boundaries;
- Encryption;
- Network access;
- Logging;
- Infrastructure monitoring;
- Backup and restore capability when required.

### 6.4 What the Business Owner Is Responsible For

The business owner is responsible for:

- Field meanings;
- Business rules;
- Whether data can be displayed to users;
- Whether freshness is sufficient;
- How to explain anomalies;
- Whether high-risk fields require manual confirmation.

If these owners are not clearly defined, serving projections will become a source of cross-team contention.

---

## 7. Consistency and Freshness

Operational serving is generally not a strongly consistent system.

It is more commonly eventual consistency: after a source system changes, the data is processed by the data platform and then published to the serving store.

### 7.1 Freshness Should Be Defined End-to-End

It is not sufficient to say "the publisher runs every 5 minutes."

The full chain should be defined:

```
source change
  -> Bronze arrival
  -> Silver update
  -> Gold refresh
  -> projection publish
  -> application read
```

Each segment can introduce latency.

### 7.2 Last Updated and Snapshot Version

Serving payloads should generally include:

- `last_updated_at`;
- `source_event_time`;
- `snapshot_version`;
- `model_version`;
- `publish_time`;
- Optional freshness status.

This allows applications and business users to understand: what point in time does the result they are reading reflect?

### 7.3 Eventual Consistency Requires Business Acceptance

If a business scenario cannot tolerate delay or brief inconsistency, a serving projection should not be used without careful consideration.

Examples of scenarios that should not rely on serving projections:

- Writing transactional state;
- Strongly consistent account balance updates;
- Real-time authorization;
- Millisecond-level risk control;
- User-interactive transactions.

These scenarios should be handled by business OLTP or dedicated real-time systems.

---

## 8. Reconciliation and Explainability

If a serving projection affects customer experience, financial judgments, risk signals, or operational actions, reconciliation is required.

### 8.1 Which Scenarios Require Reconciliation

Typically includes:

- Finance-related values;
- Customer-visible amounts;
- Risk labels;
- Eligibility determinations;
- Marketing segments;
- Automated operational actions;
- Compliance status;
- Derived results related to authoritative states in business systems.

### 8.2 What Reconciliation Can Compare

Options include:

- Gold source model row count vs. serving store item count;
- Changed key count vs. published key count;
- Published payload hash;
- Source timestamp vs. serving timestamp;
- Aggregate totals;
- Authoritative business system results;
- Consumer read result sampling.

### 8.3 Explainability Matters

Derived fields in a serving projection should be explainable:

- Which model did this value come from?
- When was it computed?
- Which version of rules was applied?
- Is this an estimate?
- Has it been manually confirmed?
- Can it be used in automated decisions?

If an application displays an estimated or advisory value, it must not let users believe it is a final authoritative fact.

---

## 9. Error Handling and Recovery

Operational serving failure modes differ from typical BI failures.

A BI report failing usually means users cannot see data. A serving projection failure may affect application functionality, user experience, or business processes.

### 9.1 Common Failure Modes

| Failure Type | Example |
|---|---|
| Publisher failure | Publish job failure, permission error, network error |
| Partial publish | Some keys succeed, others fail |
| Stale projection | Gold has updated, but serving has not |
| Schema mismatch | Field the application expects does not match the payload |
| High write volume | Sudden large volume of changes causes write pressure |
| Reader error | Application read failure or permission error |
| Reconciliation breach | Serving store diverges from Gold |
| Bad data publish | Incorrect business logic published to applications |

### 9.2 Publishing Must Be Idempotent

Publishers should be idempotent wherever possible.

Publishing the same record multiple times should have no side effects.

Common approaches include:

- Overwriting by primary key;
- Using `snapshot_version`;
- Using payload hash;
- Using conditional updates;
- Recording a publish log;
- Supporting retry for failed keys.

### 9.3 Recovery Paths

Recovery options include:

- Retrying failed keys;
- Re-publishing changed keys;
- Full rebuild of the serving store;
- Rolling back to a previous snapshot;
- Disabling a consumer feature flag;
- Falling back to the last known good snapshot;
- Marking the projection as stale.

If the serving store cannot be rebuilt from Gold, recovery becomes extremely complex.

---

## 10. Security and Privacy

Operational serving is often closer to user experience and business processes than BI, so security boundaries must be clearly defined.

### 10.1 Minimum Payload

Serving payloads should include only the fields the application needs.

Do not publish all fields from Gold to the serving store just because they exist.

Fields to avoid in particular:

- Unnecessary PII;
- Raw sensitive fields;
- Internal debug fields;
- Intermediate model features;
- Explanatory fields that should not be displayed by applications.

### 10.2 Separate Reader and Writer Access

It is recommended to explicitly separate:

- Publisher write access;
- Application read access;
- Admin and debug access;
- Platform maintenance access.

Application teams should generally not write to the serving store. Data teams should also not use the serving store to bypass business system write rules.

### 10.3 Derived Data Can Also Be Sensitive

Serving stores often contain derived data such as tags, scores, status, risk signals, and eligibility determinations.

These fields may not be raw PII, but they can still be sensitive. They may influence user experience, operational decisions, or compliance judgments.

Derived data therefore also requires classification, an owner, and an access policy.

---

## 11. Criteria for Choosing a Serving Store

When selecting a serving store, the question should not simply be "which database is fastest."

The question should be about access pattern.

### 11.1 Key Questions to Answer First

- Is this a single-key lookup or a bounded query?
- Is a range query needed?
- Is search needed?
- Is complex filtering needed?
- Are joins needed?
- Are transactional writes needed?
- What is the approximate QPS?
- What is the latency target?
- Is the payload a fixed schema or flexible JSON?
- Can the data be rebuilt from Gold?
- Is PII involved?
- Is there one consumer or multiple?
- Is cross-region read required?

### 11.2 Different Serving Shapes

| Shape | Strengths | Limitations |
|---|---|---|
| Key-value store | Simple, high-concurrency, low latency | Query patterns are constrained |
| Document database | Suited to object state and JSON payloads | Schema governance is more important |
| Relational serving DB | Strong SQL capability, suited for complex queries | More operational and connection management overhead |
| Search index | Strong for search and filtering | Should not be used as a source of truth |
| Cache | Extremely low latency | Invalidation and consistency are complex |
| Application API | Clear business ownership | Data platform depends on the application's interface capabilities |
| Reverse ETL tool | Suited for SaaS activation | Write rules and auditing must be clearly defined |

### 11.3 Don't Serve the Tool

If an application simply needs a user state point lookup, complex SQL is not required.

If the business needs complex filtering and joins, a key-value store is not the right answer.

If results need to be written back as business transactional state, this should not be routed through a serving projection to bypass the business system.

The serving store should serve the access pattern — not the other way around.

---

## 12. Relationship to Reverse ETL

Operational serving and reverse ETL are related, but not the same.

### 12.1 What Reverse ETL Typically Means

Reverse ETL typically refers to syncing results from a data warehouse or lakehouse back to business tools or SaaS systems, such as CRMs, marketing platforms, support systems, and sales tools.

This pattern is common for:

- Customer segment syncing;
- Marketing audiences;
- Sales lead scoring;
- Customer service prompts;
- Product usage tags.

### 12.2 How Operational Serving Differs

Operational serving places more emphasis on the application read path:

- The data platform publishes a projection;
- Applications read via key lookup;
- Data is typically not written directly to business OLTP;
- Projections are rebuildable;
- Latency and availability expectations are closer to application requirements.

### 12.3 Choosing the Right Pattern

| Scenario | Better Fit |
|---|---|
| A SaaS tool needs user segments | Reverse ETL tool |
| An application backend needs to read state by key | Serving projection |
| A business system must update transactional state | Application-owned API |
| Async notification to a business process | Message queue |
| A high-risk decision requires human confirmation | Manual approval workflow |

Not all data activation should be treated as a single pattern.

---

## 13. Cost of Operational Serving

Serving projections protect Snowflake workloads, but they are not free.

Costs include:

- Gold source model refresh;
- Change detection;
- Publisher compute;
- Serving store storage;
- Write capacity;
- Read capacity;
- Monitoring;
- Reconciliation;
- Retry;
- Schema compatibility management;
- Incident response.

Therefore, not every Gold model should be published as a serving projection.

A serving projection should only be introduced when the business genuinely requires low-latency, key-based operational reads.

---

## 14. Core Trade-offs

| Design Choice | Benefits | Costs |
|---|---|---|
| Applications do not query Snowflake directly | Protects analytical workloads; reduces application latency uncertainty | Requires an additional serving path |
| Using serving projections | Clear application read boundaries; rebuildable from Gold | Adds a layer of storage, publishing, and monitoring |
| Using key-value or document stores | Low-latency point lookups; suited for stable payloads | Not suited for complex joins or ad hoc queries |
| Using application-owned APIs | Clear business system ownership | Data platform depends on application interface and write rules |
| Using reverse ETL tools | Suited for SaaS activation | Write boundaries and auditing must be clearly defined |
| Delta publish | Lower cost; suited for near real-time | Requires change detection and idempotent handling |
| Full rebuild | Simple and recoverable | High cost and latency for large datasets |
| Eventual consistency | Lower system complexity | Not suited for strongly consistent transactional scenarios |
| Reconciliation | Increases trustworthiness | Adds monitoring and processing cost |
| Minimum payload | Reduces privacy and coupling risk | Applications may require additional iteration |

---

## 15. Common Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|---|---|---|
| Applications querying Snowflake directly for high-concurrency point lookups | Cost, latency, and workload boundaries become uncontrollable | Use an operational serving projection |
| Treating the serving store as a source of truth | Cannot be rebuilt; ownership boundaries become confused | Keep projections rebuildable |
| Data team writing directly to business OLTP | Large blast radius; ownership is confused | Use serving, API, queue, or reverse ETL patterns |
| Publishing all Gold models to the serving store | Cost and governance complexity increase | Only publish clearly defined key-based use cases |
| Overly large serving payloads | Privacy risk and application coupling increase | Use only the minimum necessary fields |
| No serving contract | Schema, freshness, and owner are undefined | Define a contract for every projection |
| Non-idempotent publisher | Retries may produce side effects | Use key overwrite, version, or hash-based mechanisms |
| No reconciliation | Errors may go silently undetected | Reconcile critical business fields |
| Confused ownership between application and data teams | Unclear responsibility during incidents | Explicitly define publisher, reader, and business owner |
| Using key-value stores for complex analysis | Query patterns do not match | Complex analysis should go back to Snowflake or the analytical platform |

---

## 16. Success Criteria

A well-designed operational serving architecture should satisfy:

- Application point lookups do not directly access Snowflake analytical models;
- Every serving projection has a clearly defined business purpose;
- Gold source models are the upstream source for projections;
- The serving store can be rebuilt from Gold;
- Key, grain, and payload schema are explicitly defined;
- Freshness and consistency expectations are explicitly defined;
- Publishers are idempotent;
- Publish logs and error logs are auditable;
- Reader access and writer access are separated;
- PII and derived sensitive data are classified and controlled;
- Critical business values have reconciliation;
- Application owner, data owner, and business owner boundaries are clear;
- Serving costs are included in FinOps;
- Projections have a deprecation policy;
- Scenarios unsuitable for serving are explicitly excluded.

---

## 17. Summary

The core question in operational serving is not "which database to use" — it is how to establish a clear boundary between the analytics platform and application systems.

Snowflake or a similar lakehouse platform is well-suited to computing and governing derived results, but should not be directly accessed by high-concurrency applications for point lookups, nor should it take on the responsibility of writing to business OLTP.

A more robust pattern is:

- Gold produces stable, governable business results;
- A publisher publishes the necessary results as serving projections;
- Applications read projections by key;
- The serving store is rebuildable, auditable, and monitorable;
- Business OLTP continues to own transactional facts;
- Reverse ETL, APIs, message queues, and serving stores are selected based on the specific scenario.

This allows the value of the data platform to enter business processes, without turning the data platform into another invisible application backend.
