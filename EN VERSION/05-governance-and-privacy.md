# 05 · Governance & Privacy: Governance Is Not a Post-Launch Patch

## 1. Purpose of This Document

This document discusses governance and privacy design in lakehouse architecture, focusing on **PII, access control, data product ownership, auditing, data quality, operational serving boundaries, and why a data platform should not write directly to business OLTP systems**.

This is not a security configuration manual, nor will it cover specific IAM policies, Snowflake role DDL, network policies, masking policies, or audit SQL. The focus here is on higher-level system design questions:

> When a data platform becomes an enterprise's analytics hub and a source for some operational data activation, how do we keep data controllable, interpretable, auditable, and evolvable?

My fundamental position is:

> Governance is not a post-launch patch. Governance is part of data platform architecture. If PII, permissions, auditing, data quality, ownership, and cost attribution are not brought into the system boundary at the design stage, they will typically be forced back in later at far greater complexity.

In a low-complexity lakehouse architecture, the goal of governance is not to create process overhead, but to give the platform clear lines of responsibility:

- What data is allowed to enter the platform;
- What data can be accessed by whom;
- What data can be used by BI;
- What data can be point-queried by applications;
- What results can flow back into business processes;
- What access needs to be audited;
- What data products have an owner, a contract, and a quality standard.

---

## 2. Why Governance Must Come First

Many data platforms in their early stages focus more on ingestion, modeling, and dashboard delivery. Governance is typically treated as something to be added in a later phase.

This approach may look like it accelerates delivery in the short term, but it accumulates governance debt over time.

### 2.1 Permission Debt

If a platform grants broad permissions in its early days for the sake of debugging and delivery convenience, tightening them later becomes extremely difficult.

Common problems include:

- Analysts directly accessing the raw layer;
- BI service accounts accessing too many schemas;
- Developers retaining long-term access to sensitive production data;
- Unclear service account permission boundaries;
- Temporary permissions becoming permanent;
- Downstream reports depending on intermediate tables that were never meant to be public.

Once permissions are depended upon by downstream consumers, they stop being purely a security issue and become a migration and compatibility issue as well.

### 2.2 PII Exposure Debt

If PII is not identified and controlled early when it enters the platform, it will spread across multiple layers: Landing, Bronze, Silver, Gold, BI extracts, notebooks, temporary tables, exports, and serving stores.

Once it has spread, governance is no longer about "restricting a single field" — it becomes about tracking all copies, derived tables, report caches, downloaded files, and external syncs.

Therefore, the closer privacy controls are to the ingestion boundary, the lower the overall complexity.

### 2.3 Semantic Debt

Without clear data product ownership, models devolve into "only the person who wrote it knows what it does."

When a metric is questioned, a field's meaning changes, data quality fails, or a downstream report behaves unexpectedly, teams cannot quickly determine:

- Who is responsible for explaining this model;
- Which definition is the authoritative version;
- Which consumers will be affected;
- Whether a change requires notification;
- Whether historical data needs to be backfilled.

Governance is not just about security — it is also about semantic management.

### 2.4 Audit Debt

If a system has not been designed from the start to record access, changes, quality, and release history, it is very difficult to fill those gaps retroactively.

Audit typically needs to answer questions such as:

- Who accessed sensitive data;
- Which service account published data;
- Which model produced a given metric;
- Which pipeline run caused a particular change;
- Which downstream application read a specific projection;
- Which version of data was used by the business.

These questions cannot be answered by human memory alone.

---

## 3. The Basic Layering of Governance

In a lakehouse, governance should also be layered rather than governed by a single uniform rule applied across all data.

```
Landing / Bronze
  -> control source evidence and sensitive ingestion
Silver
  -> control business objects and semantic stability
Gold
  -> control business consumption and metrics
Serving Projection
  -> control operational activation
BI / Analytics
  -> control user-facing access and extracts
```

Different layers have different governance priorities.

### 3.1 Governance Focus for Landing / Bronze

Landing and Bronze are concerned with the boundary at which source data enters the platform.

Key priorities include:

- Whether data is permitted to enter the common path;
- Whether it contains PII or sensitive fields;
- Whether source metadata is complete;
- Whether schema versions are recorded;
- Whether raw evidence is replayable;
- Whether the DLQ is auditable;
- Whether replay is controlled.

The governance goal at this layer is to ensure the platform does not expand its risk surface at the point of entry.

### 3.2 Governance Focus for Silver

Silver is concerned with the stability of business objects.

Key priorities include:

- Entity definition;
- Business key;
- Deduplication rules;
- SCD rules;
- Quality baseline;
- PII classification;
- Owner;
- Schema compatibility.

The governance goal at this layer is to ensure the business mirror is reliable, reusable, and evolvable.

### 3.3 Governance Focus for Gold

Gold is concerned with business consumption.

Key priorities include:

- Metric definition;
- Data product owner;
- Freshness expectation;
- Quality SLA;
- Consumer contract;
- BI approval;
- Access policy;
- Change policy.

The governance goal at this layer is to ensure business users are consuming trusted models.

### 3.4 Governance Focus for Serving Projection

Operational serving projections are concerned with how derived results enter application consumption.

Key priorities include:

- Serving contract;
- Primary key;
- Payload schema;
- Freshness;
- Compatibility;
- Reader access;
- Publisher ownership;
- Audit log;
- Rebuild path;
- Reconciliation.

The governance goal at this layer is to prevent the serving store from becoming an ungoverned new source of truth.

---

## 4. PII and Sensitive Data Classification

PII classification should not exist only as a list in a compliance document. It should influence ingestion, modeling, access, serving, and retention.

### 4.1 A General Classification Approach

Sensitive data can be grouped into several categories:

| Category | Meaning | Common Handling |
|---|---|---|
| Sensitive identifiers | Highly sensitive identity or legally sensitive fields | Avoid entering the common path where possible; isolate and apply strict access when necessary |
| Direct identifiers | Fields that can directly identify an individual | Hash, tokenize, mask, or controlled access |
| Indirect identifiers | Fields that, in combination with others, can identify an individual or entity | RBAC, minimal exposure, aggregation |
| Behavioral attributes | Behavioral, activity, transaction, and usage records | Controlled based on sensitivity and business need |
| Derived attributes | Labels, scores, risk, segments, recommendations | Require owner, explanation, and usage boundaries |

This is a general classification only. Actual classifications should be adjusted based on industry, regional regulations, company policy, and business risk.

### 4.2 Ingest-Time Privacy Control

The closer privacy controls are to the ingestion boundary, the less risk can spread.

Common patterns include:

- Filtering or processing sensitive fields before they enter the common landing;
- Placing highly sensitive data into an isolated staging area;
- Hashing or tokenizing direct identifiers;
- Dropping fields that are not needed for analysis;
- Using a controlled schema for sensitive fields that must be retained;
- Recording the privacy rule version in metadata.

The key principle is:

> Ungoverned sensitive data should not be allowed to enter a raw layer accessible to everyone, with the hope that downstream models will simply not misuse it.

### 4.3 The Difference Between Hash, Tokenize, and Mask

These approaches should not be used interchangeably.

| Method | Suitable For | Considerations |
|---|---|---|
| Hash | Joining without needing to retrieve plaintext | Requires stable algorithm and salt management |
| Tokenize | Controlled retrieval or authorized recovery of plaintext | The token map itself is highly sensitive |
| Mask | Displaying partial information | Does not equate to removing the risk |
| Redact / Drop | When the field is not needed | Simplest and safest option |
| Isolate | Plaintext must be retained, but only for a small number of roles | Requires strong access control and auditing |

The approach should be driven by use case, not by a default of retaining everything.

### 4.4 Derived Data Can Also Be Sensitive

Many teams focus only on raw PII while overlooking derived data.

Examples include:

- Risk scores;
- Customer segments;
- Credit or fraud labels;
- Behavioral predictions;
- Revenue or value tiers;
- Churn probability;
- Compliance status.

These fields may not be direct identifiers, but they can influence customer, operational, or compliance decisions. They therefore also require an owner, explanation, access controls, and usage boundaries.

---

## 5. Access Control: Look Beyond Table-Level Permissions

Access control should not be reduced to "who can SELECT which table."

In a lakehouse, access control involves at minimum:

- Human access;
- Service account access;
- BI access;
- Operational serving access;
- Development vs. production access;
- Sensitive data access;
- Temporary access;
- Break-glass access.

### 5.1 The Principle of Least Privilege

Least privilege is not about making everything difficult — it is about ensuring each role has only the access needed to fulfill its responsibilities.

Common principles:

- Human users use functional roles rather than directly using underlying object roles;
- Service account permissions are isolated by purpose;
- BI service accounts have read-only access to approved Gold models;
- The raw and sensitive layers are not open by default;
- Production write permissions are strictly controlled;
- Temporary permissions have an expiry time;
- High-sensitivity access requires auditing.

### 5.2 Layer-Based Access Control

Default access should differ by layer.

| Layer | Default Access Principle |
|---|---|
| Landing | Typically not directly accessible to analytics users |
| Bronze | Restricted to data engineering and controlled debug purposes |
| Silver | Available to the data team and select advanced analytics users |
| Gold | The primary entry point for BI, analytics, and business consumption |
| Serving Projection | Applications read by contract |
| Sensitive / PII area | Denied by default; access granted only to explicitly authorized roles |

This layered access approach prevents raw data from becoming a de facto business consumption layer.

### 5.3 Service Accounts Should Be Held to Stricter Standards Than Human Users

Many risks originate from service accounts, not human users.

Service accounts typically:

- Exist for long periods;
- Are used by automated tasks;
- Are prone to permission creep;
- Have a large blast radius when errors occur;
- Are easily reused across multiple pipelines.

Therefore service accounts should:

- Be split by purpose;
- Avoid being shared;
- Avoid broad access;
- Be reviewed regularly;
- Have a clearly defined owner;
- Have audit records.

---

## 6. Secure Projection: Don't Wrap Every Table in a Layer

In Snowflake and similar platforms, many governance requirements can be addressed through views, secure views, masking, row filters, or a semantic layer.

However, a common anti-pattern is wrapping all Gold tables in a view layer and treating that as governance.

### 6.1 Legitimate Uses of Projection

Projections are appropriate for:

- Hiding sensitive fields;
- Exposing an approved column set;
- Providing a BI-friendly schema;
- Allowing different roles to see different fields;
- Providing an audit-friendly access point;
- Supporting a specific consumer contract.

### 6.2 Problems With Over-Projection

If all tables are wrapped in multiple layers of views, problems arise:

- Lineage becomes more complex;
- Performance troubleshooting becomes harder;
- Permission management becomes more difficult;
- Schema changes become harder to track;
- Cost attribution becomes unclear;
- Developers lose sight of which table is the real source of truth.

The goal of governance is not to add more objects — it is to make access boundaries clear.

### 6.3 Recommended Principles

Recommendations:

- Non-sensitive Gold models can serve directly as approved consumption objects;
- Use projections for PII, restricted fields, cross-domain audit purposes, or specialized consumers;
- Projections should have an owner, a stated purpose, and defined consumers;
- Do not treat projections as a universal patch for poorly structured models.

---

## 7. Data Product Ownership

Governance must be grounded in ownership.

A data product with no owner is, in effect, untrustworthy.

### 7.1 What an Owner Is Responsible For

A data product owner does not necessarily write all the code themselves, but must be accountable for:

- Business definition;
- Data quality standards;
- Freshness expectation;
- PII classification;
- Change approval;
- Downstream communication;
- Exception handling;
- Deprecation decisions.

### 7.2 Owner Is Not the Same as Technical Maintainer

A model may have:

- A business owner;
- A technical owner;
- A platform owner;
- A consumer owner.

These roles must be distinguished.

For example:

- The business owner decides what a metric means;
- The technical owner maintains the model implementation;
- The platform owner is responsible for the operating environment;
- The consumer owner is responsible for how downstream systems use the data.

Failing to distinguish owners causes problems to bounce indefinitely between teams.

### 7.3 Ownership Should Be Captured in Metadata

Ownership should not exist only in documents or chat history.

Important models should record the following in metadata:

- Owner;
- Domain;
- Sensitivity;
- Freshness;
- Materialization;
- Cost center or use case;
- Downstream consumers;
- Deprecation status.

This also provides the foundation for subsequent FinOps, governance, and migration efforts.

---

## 8. Data Quality Governance

Data quality is not a purely technical concern.

It is also a governance concern, because quality failures erode business trust, affect operational decisions, and damage platform credibility.

### 8.1 Layered Quality Governance

Different layers have different quality objectives:

| Layer | Quality Focus |
|---|---|
| Landing | File completeness, arrival time, format, source identity |
| Bronze | Load success, technical metadata, raw payload, operation type |
| Silver | Entity correctness, deduplication, business key, state validity |
| Gold | Metric correctness, freshness, consumer constraints |
| Serving | Payload completeness, publish recency, reconciliation |

### 8.2 Test Failures Require a Response Mechanism

A test failure in isolation has no value — the response is what matters.

Critical tests should define:

- Severity;
- Owner;
- Response time;
- Exception rules;
- Downstream impact;
- Escalation path.

Without these, a data quality platform becomes a dashboard of red lights rather than a governance mechanism.

### 8.3 Quality Should Be Tied to Publishing

The release or modification of critical data products should be gated on quality checks.

For example:

- A primary key test failure must block publishing;
- A PII exposure test failure must block publishing;
- A freshness failure must mark the data as unavailable;
- A reconciliation failure must notify consumers;
- A schema contract break requires consumer sign-off.

---

## 9. Auditability: The Platform Must Be Able to Explain What Happened

Auditing is not just about satisfying regulatory or compliance requirements. It is also about the explainability of a complex system.

A well-designed data platform should be able to answer:

- Where data came from;
- When it entered the platform;
- Which models it passed through;
- Who accessed it;
- Which job produced it;
- Which version was consumed;
- What permissions were granted;
- Which exceptions were handled;
- What data was published to a serving store.

### 9.1 Types of Audit Evidence

Common audit evidence includes:

- Ingestion log;
- Load history;
- Model run history;
- Data quality results;
- Access history;
- Permission change history;
- Serving publish log;
- Error and DLQ records;
- Cost attribution records;
- Lineage metadata.

### 9.2 Auditing Should Not Rely Solely on Platform Logs

Platform logs are important, but business-level auditing also requires model metadata.

For example:

- Whether a given field is PII;
- Who owns a given model;
- What a given metric's definition is;
- Which consumer a serving payload is intended for;
- Why a given model was deprecated.

These questions cannot be answered by query history alone.

---

## 10. Zero OLTP Write: Not a Ban on Activation, but a Clear Boundary

Data platforms frequently produce valuable derived results.

Examples include:

- Customer tags;
- Risk signals;
- Recommendations;
- Operational status;
- Financial estimates;
- Eligibility flags;
- Lifecycle states.

These results may need to enter business processes. This is often referred to as reverse ETL, data activation, or operational serving.

### 10.1 Why the Data Platform Should Not Write Directly to Business OLTP

A data platform writing directly to business OLTP systems creates several problems:

- **Ownership confusion**: who is ultimately responsible for the record in the business system;
- **Larger blast radius**: an error in a data job can affect a production transaction system;
- **Expanded audit scope**: the data platform becomes a business write actor;
- **Complex rollback**: a data model error becomes a business state error;
- **Permission risk**: analytics platform credentials carry business write permissions;
- **Blurred team boundaries**: responsibilities of the application and data teams become entangled.

Therefore I lean toward the following principle:

> A data platform may publish operational projections, but should not directly modify the transaction source of truth in business OLTP systems.

### 10.2 Reverse ETL Has Multiple Patterns

Zero OLTP Write does not mean a data platform's results cannot enter business processes.

Available patterns include:

| Pattern | Description | Suitable For |
|---|---|---|
| Serving store | Data platform publishes a projection; application reads it | Key-based operational read |
| Application-owned API | Data platform calls a controlled API provided by the business system | Business system must own the write rules |
| Message queue | Data platform publishes events; business system subscribes | Asynchronous activation |
| Reverse ETL tool | Dedicated tool syncs results to SaaS or application systems | CRM / marketing activation |
| Manual approval workflow | Data results enter the business only after human review | High-risk or financial scenarios |

Which pattern to use depends on business risk, latency requirements, ownership, and audit requirements.

### 10.3 Serving Projection Is a Pattern With Clear Boundaries

In many near real-time read scenarios, serving projection is a low-complexity choice.

The pattern is:

```
Gold model
  -> publish projection
  -> serving store
  -> application reads by key
```

Its advantages are:

- Applications do not query Snowflake directly;
- The data platform does not write to business OLTP;
- The serving store can be rebuilt from Gold;
- Permission boundaries are clear;
- The publish process is auditable;
- Responsibilities of the application team and data team are clearly separated.

However, it is not a universal solution. Complex transactional writes, strongly consistent state updates, user-initiated writes, and millisecond-level decisions should not be naively solved with a serving projection.

---

## 11. Governance Metadata

Governance requires metadata to sustain it.

Without metadata, governance can only rely on manual communication and individual memory.

### 11.1 Recommended Metadata to Capture

Important data objects should record:

- Domain;
- Owner;
- Layer;
- Purpose;
- Grain;
- PII classification;
- Freshness expectation;
- Materialization;
- Quality checks;
- Cost attribution;
- Downstream consumers;
- Retention;
- Change policy;
- Deprecation status.

This metadata does not need to be fully automated from day one, but it should be a design goal.

### 11.2 Metadata Should Live Close to Code and Models

If metadata exists only in separate documents, it easily becomes stale.

A better approach is to keep metadata as close as possible to:

- Model definitions;
- Schema files;
- The catalog;
- Data contracts;
- CI checks;
- The lineage system.

This way governance can be tied to the development workflow rather than being manually added after the fact.

---

## 12. Retention and Deletion

Data retention and deletion are also part of governance.

Not all data should be retained indefinitely, and not all data can be deleted arbitrarily.

### 12.1 Retention by Layer

Different layers can have different retention policies:

| Layer | Retention Approach |
|---|---|
| Landing | Retained based on replay and archive requirements |
| Bronze | Retain raw analytical evidence for rebuilding and auditing |
| Silver | Retain the business mirror and necessary history |
| Gold | Retained based on consumption and metric requirements |
| Serving | Typically only the current projection or short-term snapshot |
| Temporary / Sandbox | Should have a short TTL |

### 12.2 Deletion Is More Than DROP TABLE

A deletion request may involve:

- Raw data;
- Derived models;
- BI extracts;
- Serving projections;
- Caches;
- Exports;
- Logs;
- Archive policies.

Deletion capability must therefore be integrated with lineage, metadata, and PII classification.

### 12.3 Retention Is Also Cost Governance

Retaining all intermediate results indefinitely increases TCO.

It is necessary to distinguish between:

- What constitutes the record of origin;
- What can be rebuilt;
- What must be retained long-term;
- What is only a temporary computation result;
- Which serving projections can be republished from Gold.

---

## 13. Governance and Low Complexity

Governance is often misunderstood as a source of complexity.

It is true that over-formalized governance slows delivery. But the absence of governance creates even greater complexity.

Without governance:

- Permissions spread;
- PII spreads;
- Metrics drift;
- Owners disappear;
- Quality failures go unaddressed;
- Costs cannot be explained;
- Downstream dependencies cannot be tracked;
- Temporary data products cannot be decommissioned.

Good governance is not about adding meaningless approvals — it is about designing clear platform boundaries so that teams can deliver faster and more safely.

---

## 14. Core Trade-offs

| Design Choice | Benefits | Costs |
|---|---|---|
| Governance-first | Reduces privacy, permission, and audit debt | Higher upfront design cost |
| Ingest-time privacy control | Reduces sensitive data spread | Increases onboarding complexity |
| Layered access control | Clear boundaries between raw, business, and consumption | Requires role and permission governance |
| Least privilege | Reduces misuse and leakage risk | Requires ongoing review |
| Data product ownership | Problems have accountable owners | Requires organizational alignment |
| Projection for sensitive exposure | Data can be exposed per consumer | Overuse increases lineage complexity |
| Zero OLTP Write | Reduces blast radius on business systems | Requires reverse ETL / serving pattern design |
| Serving projection | Clear application read boundary | Requires publish, monitoring, and consistency management |
| Metadata-driven governance | Governance can be automated and audited | Requires metadata maintenance discipline |
| Retention policy | Reduces risk and cost | Requires lineage and classification support |

---

## 15. Common Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|---|---|---|
| Retrofitting governance after launch | Permission, PII, and audit debt compounds | Front-load governance at the design stage |
| Raw layer open to all analytics users by default | Sensitive data and source system internals are exposed | Layer-based access control |
| All Gold models wrapped in multiple view layers | Lineage and performance troubleshooting become complex | Use projections only for well-defined scenarios |
| Shared service accounts with long-lived broad permissions | Large blast radius | Split by purpose, least privilege, regular review |
| PII entering the platform before being handled | Sensitive data spreads | Ingest-time handling or isolated staging |
| Technical owner only, no business owner | Metric definitions go unowned | Define both business owner and technical owner |
| No response to test failures | Data quality governance collapses | Every critical test has an owner and severity |
| Data platform writes directly to business OLTP | Ownership confusion, large blast radius | Use serving, API, queue, or reverse ETL tools |
| Serving store becomes the source of truth | Cannot be rebuilt; accountability unclear | Clearly define projection and rebuild path |
| Metadata lives only in documents | Easily becomes stale | Keep metadata close to models and code |
| Temporary datasets retained indefinitely | Cost and risk increase | Apply TTL and deprecation policies |

---

## 16. Success Criteria

A well-designed governance and privacy architecture should satisfy:

- PII and sensitive data are identified at the ingestion boundary;
- Common landing does not accept ungoverned high-sensitivity data;
- Access boundaries across Bronze, Silver, Gold, and Serving are clear;
- BI primarily accesses approved Gold models;
- Sensitive projections have a clearly defined purpose and owner;
- Service account permissions are isolated by purpose;
- Critical data products have an owner, contract, freshness expectation, and quality checks;
- Data quality failures have a defined response mechanism;
- Access, publishing, permission changes, and serving publishes are auditable;
- The data platform does not write directly to business OLTP;
- Reverse ETL or operational activation follows a well-defined pattern;
- Retention and deletion have defined policies;
- Governance metadata lives close to code and models;
- Governance mechanisms reduce long-term complexity rather than creating unnecessary process.

---

## 17. Summary

Governance and Privacy are foundational capabilities in lakehouse architecture — not a checklist reviewed by the compliance team before go-live.

When governance is deferred, the platform accumulates permission debt, PII exposure debt, semantic debt, audit debt, and cost debt. Remediation later typically demands more complex permissions, more views, more chaotic exception processes, and higher operational costs.

A more sound design is to:

- Handle sensitive data at the ingestion boundary;
- Control access by layer across Bronze / Silver / Gold / Serving;
- Make Gold the primary business consumption entry point;
- Manage data products with owners, contracts, and metadata;
- Define clear projection boundaries for operational serving;
- Avoid having the data platform write directly to business OLTP;
- Treat auditability, quality, retention, and FinOps as platform capabilities to be designed in.

Low complexity does not mean no governance. Low complexity means designing governance boundaries clearly, so the platform can operate safely, stably, and explicably over the long term.
