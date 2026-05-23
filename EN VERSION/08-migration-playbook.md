# 08 · Migration Playbook: No Big Bang, No Permanent Dual-Run

## 1. Purpose of This Document

This document discusses how to migrate from a legacy data platform to a new lakehouse architecture, focusing on **how to reduce migration risk, how to establish dual-run, parity, cutover, shadow, and decommission mechanisms, and how to avoid the permanent complexity that comes from running old and new platforms side by side indefinitely**.

This is not a project plan template, nor will it define specific timelines, budgets, team structures, or internal approval processes. The focus here is on more general system design and migration governance questions:

> When an organization decides to build a new Snowflake-centric or other lakehouse architecture, how do we ensure the migration actually reduces complexity — rather than adding a new platform alongside the old one?

My fundamental position is:

> The core goal of a data platform migration is not "getting the new platform live" — it is "safely removing old complexity." Without a clear cutover and decommission mechanism, an organization that launches a new platform will most likely end up with a second platform, not a replacement.

A mature data platform migration should not pursue a Big Bang, nor should it accept indefinite dual-run. A more robust approach is:

- Start by building a clear legacy inventory;
- Prioritize by business value and risk;
- Run dual-run for critical paths;
- Build trust through parity and business sign-off;
- Cutover in a planned sequence;
- Retain a limited shadow period;
- Decommission old paths at the end.

---

## 2. Why Big Bang Is High Risk

Big Bang migration looks clean: switch all pipelines, models, reports, and consumers at once.

But in a data platform context, Big Bang is typically very high risk.

### 2.1 Data Platform Dependencies Are Complex

A data platform is not a single application. It typically connects to:

- Source systems;
- Ingestion pipelines;
- Transformation jobs;
- BI dashboards;
- Ad hoc users;
- Data exports;
- Downstream applications;
- Finance reports;
- Operational workflows;
- Compliance or audit processes.

Many of these dependencies are not always documented. A legacy table, report, or export may still be supporting a critical business process.

A one-shot cutover will expose hidden dependencies after the switch.

### 2.2 Data Definitions Require Trust to Be Built

Even if the new platform is technically correct, it still requires business trust.

Business stakeholders typically will not immediately trust new reports just because "the new architecture is more modern." They need to see:

- Whether old and new results are consistent;
- Whether differences can be explained;
- Whether new metric definitions have been confirmed;
- Whether data freshness meets requirements;
- Whether key business cycles have been validated;
- Whether there is a rollback path in case of anomalies.

All of this requires dual-run and parity — not a one-shot switch.

### 2.3 Rollback Paths Are Complex

A data platform cutover is not simply switching an endpoint.

It may involve:

- BI datasets;
- Scheduled reports;
- Downstream exports;
- Application integrations;
- User bookmarks;
- Metric definitions;
- Permission models;
- Historical backfill.

Without phased migration, rollback becomes extremely difficult.

### 2.4 Big Bang Easily Hides the True Cost

A one-shot migration often underestimates:

- Parity testing;
- Business validation;
- User communication;
- Dashboard rebuilds;
- Edge cases;
- Historical mismatches;
- Legacy shutdown;
- Training and support.

The genuinely difficult part is not building the new platform — it is migrating consumers safely and retiring the old platform.

---

## 3. Core Migration Principles

### 3.1 Business-First, Not Technology-First

Migration priority should not be decided by the technical team alone.

A more sound ordering should consider:

- Business value;
- Usage frequency;
- Risk level;
- Current pain points;
- Cost impact;
- Data quality issues;
- Platform complexity;
- Number of downstream dependencies.

Objects that are technically easy to migrate are not necessarily the ones that should be migrated first.

### 3.2 Do Not Aim for One-Shot Completion

Data platform migrations should take a progressive approach.

It is recommended to treat migration as multiple small cutovers, not one large cutover.

Every group of pipelines, models, reports, or consumers should go through:

```
Discover
  -> Rebuild
  -> Dual-run
  -> Validate
  -> Cutover
  -> Shadow
  -> Decommission
```

### 3.3 Dual-Run Must Have Exit Criteria

Dual-run exists to build trust and reduce risk — not to permanently maintain two systems.

Every dual-run object should have clearly defined exit criteria:

- Parity has reached the agreed standard;
- Differences have been explained;
- Consumers have signed off;
- Rollback path is defined;
- Shadow period has ended;
- Legacy dependencies are cleared;
- Decommission has been executed.

A dual-run without exit criteria becomes permanent complexity.

### 3.4 Migration Must Also Manage TCO

Costs typically rise during migration because old and new platforms are running simultaneously.

This is not necessarily a problem, but it must be labeled and explained.

Migration, backfill, and dual-run workloads should be distinguished from steady-state workloads — otherwise management will misread the new platform's true cost.

More importantly, TCO can only decrease once old paths are decommissioned.

---

## 4. Legacy Inventory: Know What You Have First

The first step in migration is not building new models — it is building a legacy inventory.

Without an inventory, it is impossible to assess migration scope, priority, risk, or exit paths.

### 4.1 What an Inventory Should Include

It is recommended to record at minimum:

- Pipeline name;
- Source system;
- Target table, file, or report;
- Schedule;
- Owner;
- Consumer;
- Business purpose;
- Data freshness;
- Downstream dependencies;
- Cost or runtime;
- Failure history;
- Data sensitivity;
- Migration status;
- Decommission candidacy;
- Known issues.

### 4.2 Don't Only Inventory Pipelines

Many teams only inventory ETL jobs, but the true dependencies may be elsewhere.

Also inventory:

- BI dashboards;
- Semantic models;
- Scheduled exports;
- Spreadsheets;
- API consumers;
- Manually consumed tables;
- Ad hoc analyst workflows;
- Reconciliation processes;
- Operational reports;
- Compliance or audit extracts.

If only pipelines are migrated without migrating consumer relationships, cutovers will fail.

### 4.3 Classify Legacy Objects

Legacy objects can be classified as follows:

| Type | Handling |
|---|---|
| High-value active | Prioritize for migration |
| High-risk / critical | Migrate carefully with strong parity |
| Low-use but required | Retain until a clear replacement path exists |
| Unknown owner | Begin owner discovery |
| No recent usage | Decommission candidate |
| Duplicated logic | Consolidate or replace |
| Temporary workaround | Eliminate during migration |
| Compliance-related | Review separately |

The goal of classification is not to produce documentation — it is to serve migration decisions.

---

## 5. Migration Prioritization: How to Decide What to Migrate First

Migration sequence should be based on value, risk, and complexity.

### 5.1 Objects Suitable for Early Migration

Generally suitable for early migration:

- High usage frequency;
- Obvious current quality problems;
- High maintenance cost;
- High legacy cost;
- High downstream business value;
- Can demonstrate new platform value;
- Source data and logic are relatively clear;
- Consumers are willing to participate in validation.

### 5.2 Objects Not Suitable for the First Wave

Not suitable for the first wave:

- Unclear ownership;
- Disputed business definitions;
- Too many non-transparent downstream dependencies;
- Special compliance requirements;
- Complex historical metric definitions;
- Lack of verifiable samples;
- Current system is old but stable and low-cost.

These objects may still need to be migrated, but they are not good candidates for validating the new platform.

### 5.3 Using a Value / Risk / Effort Matrix

Three dimensions can help frame decisions:

| Dimension | Question |
|---|---|
| Value | Will migration meaningfully improve business value, quality, or cost? |
| Risk | Will a migration failure affect critical business, financial, or compliance processes? |
| Effort | How complex are the source data, logic, dependencies, and validation? |

The first wave should ideally target high-value, medium-to-low risk, manageable-effort objects.

---

## 6. Rebuild: Reconstruct in the New Architecture, Don't Just Copy

Migration is not moving old pipelines verbatim to the new platform.

If old logic is simply copied into Snowflake, old complexity is carried into the new architecture.

### 6.1 First Assess Whether Old Logic Is Still Sound

Before rebuilding, ask:

- Is this data product still being used?
- Is the metric definition still correct?
- Is the source system still the authoritative source?
- Does the old pipeline contain historical workarounds?
- Are there duplicate models that can be consolidated?
- Can this be restructured using the new Medallion layering?
- Is near real-time required, or is batch sufficient?

### 6.2 Rebuild According to the New Architectural Layers

The new platform should rebuild along the standard path:

```
Source
  -> Landing
  -> Bronze
  -> Silver business mirror
  -> Gold consumption model
  -> BI / serving / export
```

Do not copy all intermediate tables from the old platform one-to-one.

Migration is an opportunity to clean up semantics, simplify models, and reduce technical debt.

### 6.3 Document Metric Differences

During rebuilding, it is very likely that the new and old logic will not be perfectly identical.

Differences may come from:

- Bugs in old logic;
- Corrections in new logic;
- Source system changes;
- Different time range handling;
- Different null and default handling;
- Different deduplication rules;
- Different late data handling;
- Business definition upgrades.

These differences must be recorded and explained — not simply resolved by chasing numeric equality.

---

## 7. Dual-Run: The Core Mechanism for Building Migration Trust

Dual-run means old and new paths run simultaneously for a period of time.

Its purpose is not to permanently maintain two systems, but to build trust, surface differences, and validate stability.

### 7.1 What Dual-Run Should Validate

Should validate:

- Row counts;
- Key coverage;
- Metric totals;
- Dimensions;
- Freshness;
- Late data handling;
- Null and default handling;
- Duplicate handling;
- Downstream BI behavior;
- Performance;
- Cost;
- Access control.

### 7.2 Dual-Run Should Not Be Extended Indefinitely

Dual-run should have a defined window and clear exit criteria.

This can be defined by business cycles, for example:

- One complete daily reporting cycle;
- One weekly reporting cycle;
- One financial month-end cycle;
- One business peak period;
- One specific campaign or operational cycle.

The key is not a fixed number of days, but coverage of enough business scenarios.

### 7.3 Dual-Run Costs Should Be Labeled

A cost increase during dual-run is normal.

But these costs should be labeled as migration and validation cost — not misread as the new platform's steady-state cost.

Otherwise management may draw incorrect conclusions about the new architecture's economics.

---

## 8. Parity Check: Consistency Is Not Simple Numeric Equality

Parity checks are at the core of dual-run.

But parity should not be understood as all numbers being identical.

### 8.1 Layers of Parity

Parity can be assessed in layers:

| Level | What to Check |
|---|---|
| Source coverage | Do old and new cover the same source data range? |
| Row count | Are record counts consistent or are differences explainable? |
| Key coverage | Is the primary key set consistent? |
| Metric parity | Are core metrics consistent? |
| Distribution | Are distributions, groupings, and top-N values consistent? |
| Freshness | Do old and new latencies meet expectations? |
| Business scenario | Are critical business use cases consistent? |
| Consumer behavior | Do reports and downstream applications behave consistently? |

### 8.2 Differences Must Be Classified

A difference does not necessarily mean the new platform is wrong.

Differences can be classified as:

| Difference Type | Meaning | Handling |
|---|---|---|
| Expected difference | Old and new definitions are known to differ | Document and obtain business confirmation |
| Legacy bug fixed | New platform has corrected an old problem | Document and explain |
| New platform bug | New logic contains an error | Fix and re-validate |
| Timing difference | Refresh times differ | Compare within freshness window |
| Source coverage gap | Data ranges are inconsistent | Fix ingestion or source mapping |
| Definition ambiguity | Business definition is unclear | Business owner decides |

### 8.3 Parity Should Be Automated, but Requires Business Explanation

Technology can automatically compare row counts, hashes, metric totals, and key coverage.

But business differences require explanation from business owners.

Especially when metric definitions have changed, the engineering team should not unilaterally decide whether a difference is acceptable.

---

## 9. Cutover: Switching Is Not a Button

Cutover is the act of moving consumers from the old path to the new path.

It is not simply "changing a connection string."

### 9.1 Pre-Conditions for Cutover

It is recommended to confirm the following before cutover:

- New models have been running stably;
- Parity has reached the agreed standard;
- Differences have been explained;
- Consumers have validated;
- Permissions have been configured;
- BI or applications have been updated;
- Rollback path is defined;
- Monitoring is live;
- Owner has confirmed;
- Decommission plan is prepared.

### 9.2 Types of Cutover

Cutovers can be classified as follows:

| Type | Description |
|---|---|
| BI cutover | Dashboard or semantic model switches to reading new Gold |
| Export cutover | Downstream file or table export switches to new model |
| API / serving cutover | Application switches to reading serving projection |
| User workflow cutover | Analysts or business users switch to new data product |
| Pipeline cutover | Downstream pipeline switches to new table or new contract |

Different cutovers require different validation approaches.

### 9.3 Cutover Requires Communication

Even when a cutover is technically seamless, communication is still needed:

- Scope of the cutover;
- Cutover timing;
- Expected changes;
- Known differences;
- Rollback method;
- Contact person;
- Old path retirement date.

A cutover without communication erodes business trust.

---

## 10. Shadow Period: Limited Retention, Not Indefinite Dual-Run

After cutover, a shadow period can be retained.

The purpose of the shadow period is to keep the old path temporarily available for comparison and rollback, while consumers are already using the new path.

### 10.1 What to Monitor During the Shadow Period

Focus on:

- New path stability;
- Business feedback;
- Data differences;
- Performance;
- Cost;
- Hidden dependencies;
- Rollback readiness.

### 10.2 The Shadow Period Must Have an End Condition

Shadow should not become a permanent dual-run.

End conditions can include:

- No significant data issues;
- No consumer objections;
- Hidden dependencies cleared;
- Rollback no longer needed;
- Decommission approved;
- Old path access volume has dropped to zero.

### 10.3 No New Dependencies Allowed During Shadow

The old path should be frozen during the shadow period.

No new reports, exports, applications, or users should be allowed to depend on the old path.

Otherwise the old platform regains momentum and decommission will fail.

---

## 11. Decommission: The True Marker of Migration Completion

Decommission is the most frequently overlooked part of migration.

But from a TCO and complexity perspective, it is the most important step.

### 11.1 What Decommission Should Include

Retiring old paths may involve:

- Stopping old pipelines;
- Deleting old schedules;
- Removing old BI data sources;
- Archiving old tables or files;
- Deleting expired permissions;
- Cleaning up service accounts;
- Stopping old monitoring;
- Updating documentation;
- Notifying consumers;
- Marking old objects as deprecated;
- Releasing infrastructure costs.

### 11.2 Confirm Dependencies Before Decommissioning

Before decommissioning, confirm:

- No active consumers;
- No scheduled reports;
- No downstream exports;
- No application reads;
- No audit or compliance retention requirements;
- Data has been archived per policy;
- Rollback window has ended;
- Owner has approved.

### 11.3 Without Decommission, Complexity Does Not Decrease

Launching the new platform is only half the migration.

Complexity only begins to decrease when the old platform is retired.

If the old platform continues to run, the team still needs to maintain old schedules, old alerts, old permissions, old models, old reports, and old institutional knowledge. The result is a rise in TCO, not a decrease.

---

## 12. Migration Governance

Migration requires governance, but governance should not become inefficient approval bureaucracy.

The goal of governance is to make migration status transparent, risk controllable, and responsibility clear.

### 12.1 Every Migration Object Should Have a Status

Recommended statuses:

```
Discovered
  -> Prioritized
  -> Rebuilding
  -> Dual-running
  -> Validating
  -> Cutover-ready
  -> Cutover-complete
  -> Shadow
  -> Decommissioned
```

These can be adjusted to fit team needs, but must be able to express where an object currently sits in the migration lifecycle.

### 12.2 Every Migration Object Should Have an Owner

At minimum:

- Technical owner;
- Business owner;
- Consumer owner;
- Platform owner when relevant.

Objects without an owner should not proceed to a production cutover.

### 12.3 Migration Decisions Should Be Traceable

Important migration decisions should record:

- Why the migration is happening;
- Migration scope;
- Old vs. new differences;
- Parity results;
- Cutover conclusion;
- Rollback plan;
- Decommission date;
- Known risks;
- Business sign-off.

This is not about formality — it is about ensuring no one has to rely on memory months later to explain why a cutover happened in a particular way.

---

## 13. Migration and FinOps

Migration and FinOps must be managed together.

### 13.1 Cost Increases During Migration Are Normal

Dual-run, backfill, parity, and shadow all increase costs.

These costs should be identified as migration cost, not steady-state cost.

### 13.2 Costs Should Be Explained by Phase

Can be broken down as:

| Phase | Cost Characteristics |
|---|---|
| Build | New platform development and testing costs rise |
| Backfill | Historical data rebuild costs rise |
| Dual-run | New and old platforms running simultaneously |
| Cutover | Support and validation costs rise |
| Shadow | Old platform retained short-term |
| Decommission | Costs begin to decrease |
| Steady-state | New platform stable operating cost |

### 13.3 Decommission Is What Realizes the TCO Benefit

If migration only builds the new platform without shutting down the old one, TCO is unlikely to decrease.

FinOps reviews should continuously track:

- How many objects have been migrated;
- How many old objects have been retired;
- Objects still in dual-run;
- Shadow periods that have run past their end date;
- Legacy objects with no owner;
- Residual costs of the old platform.

---

## 14. Data Quality and Reconciliation

Data quality during migration is not just about whether new models pass tests.

It is more about whether old and new platforms are genuinely interchangeable from a business perspective.

### 14.1 Reconciliation Dimensions

Migration validation can include:

- Row count;
- Key coverage;
- Metric totals;
- Aggregations by dimension;
- Historical trends;
- Null rate;
- Duplicate rate;
- Freshness;
- Business scenario samples;
- Consumer output comparison.

### 14.2 Historical Backfill Requires Separate Validation

Historical backfill commonly encounters:

- Missing source data;
- Historical schema changes;
- Different historical business rules;
- Time range misalignment;
- Different handling of late-arriving data;
- Timezone differences;
- Historical corrections not included.

These issues cannot be validated by testing against current data alone.

### 14.3 Don't Blindly Chase 100% Equality

If the old platform has a bug, the new platform should not replicate that bug simply for the sake of parity.

A more sound approach is to:

- Identify the difference;
- Classify the difference;
- Explain the difference;
- Have the business owner confirm whether it is acceptable;
- Document the new definition;
- Backfill history if necessary.

---

## 15. Communication and Change Management

Data platform migration is not a purely technical activity.

If users do not know when the cutover is happening, why metrics have changed, or when old reports will be retired, trust will erode.

### 15.1 What Needs to Be Communicated

At minimum, communicate:

- What is being migrated;
- Business impact;
- Where the new models are located;
- Metric changes;
- Known differences;
- Cutover timing;
- Shadow period;
- Old path retirement date;
- Support channels;
- Rollback conditions.

### 15.2 Communication Tailored to Different Audiences

Different stakeholders care about different things:

| Role | Primary Concerns |
|---|---|
| Business owner | Whether metrics are trustworthy; whether definitions have changed |
| Analyst | Where the new tables are; how to use them |
| BI owner | Whether dashboards need adjustment |
| Application owner | Whether APIs or serving are stable |
| Finance / Operations | Whether reporting cycles are affected |
| Platform team | Permissions, monitoring, running costs |
| Governance / Security | Sensitive data and access boundaries |

### 15.3 Training and Support

If the new platform changes how things are used, provide:

- A quick start guide;
- A model catalog;
- Example queries;
- Known differences;
- Office hours;
- Migration FAQ;
- Owner contacts.

Migration success is not users being forced onto the new platform — it is users understanding why the new platform is more trustworthy.

---

## 16. Risks and Controls

### 16.1 Common Risks

| Risk | Description | Control |
|---|---|---|
| Hidden dependency | Legacy objects still in use but unknown | Usage tracking, consumer discovery |
| Metric mismatch | Old and new metrics are inconsistent | Parity, business sign-off |
| Cutover failure | New path is unavailable after cutover | Rollback plan, shadow period |
| Cost spike | High dual-run and backfill costs | Migration workload tagging |
| Scope creep | New requirements keep being added during migration | Change control, phase boundaries |
| Missing owner | No one to decide on definitions or cutover | Owner assignment |
| Governance gap | New platform permissions are too broad | Access review, layered access |
| Legacy never retired | Old platform retained indefinitely | Decommission gate |
| User distrust | Users do not trust new results | Communication, reconciliation evidence |

### 16.2 Risk Control Principles

Recommended principles:

- No owner, no cutover;
- No parity, no cutover;
- No rollback, no cutover;
- No shadow end date, do not enter shadow;
- No decommission plan, migration is not complete;
- No business explanation, do not forcibly accept metric differences.

---

## 17. Core Trade-offs

| Design Choice | Benefits | Costs |
|---|---|---|
| Progressive migration | Low risk; easier to validate | Longer total cycle |
| Dual-run | Builds trust; rollback available | Short-term cost increase |
| Parity checks | Differences can be explained | Requires testing and business involvement |
| Business-first prioritization | Higher migration value | Requires cross-team coordination |
| Rebuild per new architecture | Eliminates old complexity | Not a simple copy; higher effort |
| Shadow period | Reduces cutover risk | Becomes permanent dual-run if end conditions are absent |
| Decommission gate | Actually reduces TCO | Requires dependency cleanup and organizational decisions |
| Strict owner mechanism | Clear accountability | Requires organizational alignment |
| Separate migration cost attribution | Prevents misreading new platform cost | Requires tagging and FinOps discipline |

---

## 18. Common Anti-Patterns

| Anti-Pattern | Problem | Better Approach |
|---|---|---|
| Big Bang migration | Hidden dependencies and metric differences surface all at once | Phased migration and cutover |
| Building new platform without retiring old one | TCO and complexity increase | Every object has a decommission path |
| Copying old logic verbatim to new platform | Old technical debt is carried into new architecture | Rebuild per Medallion and new data contracts |
| Starting migration without an inventory | Scope and risk are unclear | Build a legacy inventory first |
| Dual-run without exit criteria | Becomes permanent dual-run | Define exit criteria and shadow end date |
| Parity checking totals only | Hides key coverage and business differences | Multi-level parity and business scenario validation |
| Not recording old vs. new differences | Cannot be explained later | Classify, document, and sign off on differences |
| Cutover without rollback | Cutover risk is too high | Every cutover has a rollback path |
| Migration costs not separately tagged | Steady-state costs are misread | Tag migration and backfill workloads |
| Allowing new dependencies on old path during shadow | Old platform regains momentum | Freeze new consumption on old path |
| No user communication | Business does not trust new results | Explain changes, timing, and support in advance |

---

## 19. Success Criteria

A well-designed data platform migration should satisfy:

- Legacy inventory covers major pipelines, reports, exports, and consumers;
- Every migration object has an owner, status, and priority;
- High-value objects are migrated first;
- New platform is rebuilt per standard architecture, not by mechanically copying old logic;
- Dual-run has defined objectives and exit criteria;
- Parity checks cover row count, keys, metrics, freshness, and business scenarios;
- Differences are classified, explained, and signed off;
- A rollback path exists before every cutover;
- Shadow periods have a defined end date;
- Old paths are frozen from new dependencies;
- Decommission is actually executed;
- Migration, backfill, and dual-run costs are separately attributed;
- Users understand old vs. new differences and cutover timing;
- Old complexity is removed, not hidden alongside the new platform.

---

## 20. Summary

Data platform migration is not about copying an old system to a new one, nor is launching the new platform the finish line.

True migration completion means business consumers have safely switched to the new platform, critical data has been validated, old paths have been retired, permissions and schedules have been cleaned up, and costs and complexity have genuinely decreased.

A robust migration playbook should follow:

```
Discover
  -> Prioritize
  -> Rebuild
  -> Dual-run
  -> Validate
  -> Cutover
  -> Shadow
  -> Decommission
```

Big Bang is too high risk. Permanent dual-run is unacceptable.

The essence of migration is using verifiable small-step cutovers to progressively converge legacy complexity into the new lakehouse architecture — and ultimately remove the burden of the old system.

If the old platform continues to run long after the new one has launched, migration is not yet complete. All that has changed is that the complexity now has a more modern facade.
