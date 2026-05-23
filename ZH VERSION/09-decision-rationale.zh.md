# 09 · Decision Rationale：关键架构取舍

## 1. 文档目的

这篇文档汇总整套 Lakehouse Architecture Playbook 中最重要的架构取舍。

前面的章节分别讨论了设计目标、参考架构、ingestion、data modeling、governance、FinOps、operational serving 和 migration。这一篇不再重复具体设计，而是回答更本质的问题：

> 为什么这套 playbook 推荐这些设计选择？这些选择解决了什么问题？牺牲了什么？什么时候不应该这样做？什么情况下需要重新评估？

我的基本观点是：

> 成熟的数据平台设计，不只是列出“推荐架构”，更重要的是解释“为什么推荐它”，以及“什么情况下它不再成立”。

这篇文档的目标不是把 Snowflake 或任何一个 vendor 写成唯一答案。Snowflake 在这套 playbook 中只是 reference platform。真正要讨论的是：在构建低复杂度 lakehouse 架构时，如何在系统复杂度、数据新鲜度、治理、安全、TCO、迁移风险和团队能力之间做取舍。

---

## 2. 总体取舍原则

整套 playbook 的决策逻辑可以归纳为六条。

### 2.1 优先降低系统复杂度

如果一个能力可以在现有核心平台内稳定实现，就不要默认引入新的系统边界。

每增加一个系统，就增加一组：

* 权限模型；
* 部署流程；
* 监控和告警；
* 故障模式；
* on-call 责任；
* 成本归因；
* 技能要求；
* 迁移和下线问题。

因此，低复杂度不是“少用工具”的口号，而是长期可运营性的前提。

### 2.2 以业务 freshness 决定实时架构

不要因为 CDC、streaming 或 event-driven 这些词本身而建设实时平台。

应该先问：

* 业务到底需要多新鲜的数据？
* 分钟级是否足够？
* 小时级是否足够？
* 延迟降低是否改变业务决策？
* 业务是否愿意为更低延迟承担更高复杂度和成本？

只有业务 SLA 证明需要 true streaming，才应该引入完整 streaming architecture。

### 2.3 解耦 OLTP 结构和业务语义

源系统 schema 服务于应用事务，不服务于长期分析语义。

因此，数据平台需要通过 Medallion 建模，把源系统结构、业务镜像和业务消费分开。

### 2.4 分清 analytical、operational read 和 transactional workload

Snowflake 或类似 lakehouse 平台适合分析型 workload，但不应被当作应用 OLTP API。

应用点查、事务写入、实时决策和复杂事件处理，都需要根据访问模式单独判断。

### 2.5 治理和 FinOps 必须前置

治理和成本不是平台上线后的运营问题，而是架构边界问题。

如果 PII、权限、审计、owner、freshness、data contract 和 cost attribution 没有设计进去，后续一定会以更高复杂度的形式补回来。

### 2.6 迁移以 decommission 为终点

新平台上线不等于迁移完成。

只有旧链路被下线、权限被清理、成本被释放、消费者完成切换，迁移才真正产生复杂度下降。

---

## 3. Decision 1：为什么使用 Snowflake 作为 reference analytical core

### 推荐选择

在这套 playbook 中，Snowflake 被用作 reference analytical core，用来承载：

* analytical storage；
* SQL transformation；
* Bronze / Silver / Gold modeling；
* BI-serving models；
* near real-time incremental processing；
* 部分 orchestration；
* governance metadata；
* workload attribution。

### 为什么这样设计

主要理由不是“Snowflake 什么都能做”，而是它能在很多企业场景下降低系统拼装复杂度。

如果团队把 raw data、transformation、metrics、BI models、incremental processing、access control 和 cost attribution 分散到多个系统中，长期会产生大量 glue code 和跨系统排障成本。

Snowflake-centric 设计的价值在于：

* 减少系统边界；
* 统一建模和消费路径；
* 减少外部调度和临时脚本；
* 让 SQL-first data engineering 更集中；
* 让 governance 和 cost attribution 更容易落地；
* 降低团队需要同时掌握的运行时数量。

### 代价是什么

代价不是简单的“成本更高”或“治理更难”。

真正的代价是：

> 架构会更依赖 Snowflake 原生能力和设计边界。对 Snowflake 不擅长的 workload，需要 companion systems 或 separate architecture。

例如：

* 毫秒级事件处理；
* 强实时风控；
* 高频 streaming；
* 复杂事件处理；
* 大规模在线特征服务；
* 应用事务写入。

这些不应该强行放入 Snowflake 主链路。

### 什么时候不适用

如果企业有以下条件，可能需要选择其他核心平台或混合架构：

* 强开源可移植性要求；
* streaming-first 数据平台；
* 大规模 ML / feature engineering 优先；
* 已有深度 Databricks、Fabric、BigQuery、Redshift 或其他生态投资；
* 极复杂多云或 active-active 需求；
* 团队的主要能力栈不在 SQL-first data engineering。

### 重新评估信号

如果出现以下情况，应重新评估 Snowflake-centric 边界：

* 大量 workload 必须绕开 Snowflake 才能满足 SLA；
* 外围 companion systems 数量快速增长；
* Snowflake-native orchestration 无法满足依赖复杂度；
* true streaming 成为主 workload，而不是例外；
* 成本集中度无法通过 TCO 解释；
* 团队反而因为平台集中而降低交付效率。

---

## 4. Decision 2：为什么需要一条主数据链路

### 推荐选择

使用一条默认主链路：

```text
Sources
  -> Landing
  -> Bronze
  -> Silver
  -> Gold
  -> BI / Analytics / Operational Serving
```

### 为什么这样设计

主链路的价值是降低心智负担。

如果每个数据源、每个报表、每个应用消费都采用不同路径，平台会变成无法解释的集合。团队遇到数据问题时，不知道该从哪个系统、哪层、哪个 owner 开始排查。

主链路不是为了限制创新，而是为大多数场景提供默认解释框架。

### 代价是什么

统一主链路可能会让某些简单场景显得“多了一层”。例如，小型 PoC 或低价值临时数据，可能不需要完整 Landing / Bronze / Silver / Gold 流程。

### 什么时候不适用

不适合强行套用的场景包括：

* 一次性探索；
* 临时数据分析；
* 低风险 PoC；
* 真正 streaming-first 场景；
* 应用事务写入；
* 极低延迟 serving。

### 重新评估信号

如果越来越多场景都需要绕开主链路，说明主链路可能过重、过慢，或者未覆盖真实业务模式。

---

## 5. Decision 3：为什么 Landing 和 Bronze 要职责分离

### 推荐选择

Landing 和 Bronze 被建模为两个不同职责：

* Landing 是源数据进入平台的 handoff 和 evidence layer；
* Bronze 是进入分析平台后的 raw query layer。

多数企业级场景下，两者物理分离更清晰；但在小规模、低风险或特定 table-format 架构中，两者可以合并实现。

### 为什么这样设计

Landing 解决的是：

* 源系统交付了什么；
* 原始输入是否可回放；
* 加载失败后如何恢复；
* 数据进入平台前是否需要治理；
* 审计和归档边界在哪里。

Bronze 解决的是：

* 数据如何进入分析平台后被查询；
* 如何支持 Silver 增量处理；
* 如何保留 source metadata；
* 如何避免下游直接扫描对象存储；
* 如何提供 raw analytical layer。

这两个职责不同。

### 代价是什么

物理分离会增加：

* 对象存储管理；
* metadata；
* 生命周期策略；
* 加载链路；
* 初始设计复杂度。

### 什么时候可以合并

可以合并的情况包括：

* 小规模 PoC；
* 低价值数据；
* 源系统可可靠重放；
* 使用 Iceberg / Delta / external table 等 table-format；
* replay 和 audit 要求较低；
* 延迟极度敏感且风险可接受。

### 重新评估信号

如果 Landing 只是复制数据、没有 replay、audit、lifecycle 或 governance 价值，就应重新评估它的设计。

如果 Bronze 被大量业务用户直接消费，也应重新评估 Silver / Gold 是否缺失。

---

## 6. Decision 4：为什么 CDC 不等于 real time

### 推荐选择

CDC 应被理解为 change awareness 和 incremental capture，而不是端到端 real-time consumption。

### 为什么这样设计

CDC 只说明平台知道源系统发生了哪些变化。

但业务是否获得 near real-time 数据，还取决于：

```text
Change capture
  -> Landing
  -> Bronze load
  -> Silver processing
  -> Gold refresh
  -> BI refresh / serving publish
  -> downstream consumption
```

任何一个环节都可能决定最终 freshness。

### 代价是什么

如果不把 CDC 当作 real time，就需要更明确地定义端到端 freshness。这会增加度量和沟通成本，但能避免误解。

### 什么时候 CDC 足够

当业务只是需要知道 batch loading 之间发生了哪些 insert、update、delete，并基于这些变化做增量加载、对账、补偿或重算时，CDC 可能已经足够。

### 什么时候需要 true streaming

如果业务需要：

* event-by-event processing；
* 毫秒级响应；
* complex event windows；
* 实时决策；
* 下游直接消费事件流；
* 高吞吐低延迟事件处理；

就应该设计 streaming / event-driven architecture。

### 重新评估信号

如果 near real-time pipeline 不断被要求降低到秒级或毫秒级，或者业务开始依赖事件级决策，那么 CDC + micro-batch 模式可能不再适合。

---

## 7. Decision 5：为什么 Medallion 的核心是语义解耦

### 推荐选择

使用 Bronze / Silver / Gold，但重点不是命名，而是职责：

* Bronze 保留 raw evidence；
* Silver 构建 business mirror；
* Gold 提供 business consumption。

### 为什么这样设计

源系统 schema 服务于 OLTP，不服务于长期分析语义。

如果 BI、指标和应用消费直接依赖源系统表，源系统变化会不断冲击业务消费。

Medallion 的真正价值是解耦两类操作：

1. 把 OLTP 存储结构转换成稳定业务镜像；
2. 从业务镜像中提取指标、报表、数据产品和 serving source model。

### 代价是什么

分层会增加建模工作量。

团队需要定义：

* 每层职责；
* grain；
* owner；
* data contract；
* quality checks；
* change policy。

如果团队没有建模纪律，Medallion 可能只是多了几层名字。

### 什么时候可以简化

对于临时分析、低风险 PoC、单一小数据源，可以简化层级。

但一旦数据进入长期消费、BI 或应用场景，就需要清晰的语义层。

### 重新评估信号

如果 Silver 只是字段重命名，没有形成 business mirror，应重新设计 Silver。

如果 Gold 只是一个大宽表集合，指标仍然散落在 BI 中，应重新设计 Gold。

---

## 8. Decision 6：为什么不默认所有模型 near real-time

### 推荐选择

Near real-time 只应用于有明确业务价值的模型，不作为所有模型的默认刷新方式。

### 为什么这样设计

Near real-time 会增加：

* 刷新频率；
* 增量状态管理；
* orchestration；
* observability；
* error handling；
* cost；
* consistency 问题。

如果业务只是每天或每小时查看数据，持续刷新并不增加价值。

### 代价是什么

不是所有数据都最新。团队需要对不同层和不同模型定义不同 freshness。

这要求业务和技术共同接受：不同数据产品可以有不同 freshness，而不是所有东西都追求实时。

### 什么时候应该 near real-time

适合场景：

* 运营指标；
* 业务状态同步；
* 风险提示；
* 客户标签；
* high-frequency dashboard；
* operational serving source；
* 对延迟敏感但不需要毫秒级的场景。

### 重新评估信号

如果 near real-time 模型使用率低、刷新成本高、业务不关注分钟级差异，应降级为 batch。

如果 batch 模型频繁被业务抱怨延迟，应评估 incremental 或 near real-time。

---

## 9. Decision 7：为什么 Dynamic Tables 不是默认答案

### 推荐选择

Dynamic Tables 或类似持续刷新机制，应作为 near real-time 建模工具，而不是所有 Gold 模型的默认物化方式。

### 为什么这样设计

Dynamic Tables 可以降低增量刷新复杂度，但持续刷新本身有成本。

如果模型逻辑复杂、数据变化频繁、依赖链很深，持续刷新可能带来高成本和难排障的依赖。

### 代价是什么

不默认使用 Dynamic Tables，意味着某些模型需要明确设计 batch、incremental table、stream + task 或 materialized view。

这增加了设计判断，但降低了无意识成本。

### 什么时候适合

适合：

* 明确分钟级 freshness；
* 依赖逻辑相对清楚；
* 消费者频繁使用；
* 成本可归因；
* 业务价值可证明。

### 重新评估信号

如果 Dynamic Table refresh 成本增长快、消费者很少、freshness 无业务价值，就应重新评估物化方式。

---

## 10. Decision 8：为什么应用不应该直接查 Snowflake

### 推荐选择

BI 和分析读取 Snowflake Gold；应用点查读取 operational serving projection。

### 为什么这样设计

Snowflake 适合 analytical workload：扫描、聚合、join、历史分析、BI 和 ad hoc exploration。

应用点查通常是 key-based、高并发、低延迟、小结果集。这更像 serving workload，不是 analytical workload。

应用直接查 Snowflake 会带来：

* latency 不可预测；
* warehouse 被小查询唤醒；
* BI 和应用 workload 混用；
* service account 权限扩大；
* ownership 模糊；
* 应用 SLA 依赖分析平台运行时。

### 代价是什么

引入 serving projection 会增加：

* serving store；
* publisher；
* consistency monitoring；
* reconciliation；
* serving contract；
* additional cost。

### 什么时候可以直接查 Snowflake

低频内部工具、后台运营查询、非关键低并发场景，可能可以直接查 Snowflake。

但这应该是有边界的例外，而不是应用架构默认模式。

### 重新评估信号

如果应用查询 Snowflake 的 QPS、延迟要求或业务重要性不断上升，应改为 serving projection 或应用专属服务。

---

## 11. Decision 9：为什么 serving store 是 projection，不是事实源

### 推荐选择

Operational serving store 应该是从 Gold 发布出来的可重建 projection。

### 为什么这样设计

Projection 的好处是：

* 可以从上游重建；
* 不承载交易事实；
* 数据定义来自 Gold；
* 应用读取边界清楚；
* 错误可以通过重新发布修复；
* 数据平台不直接污染 OLTP。

如果 serving store 成为事实源，平台就需要承担新的写入一致性、冲突处理、恢复、审计和 ownership 问题。

### 代价是什么

Projection 通常是 eventual consistency。

业务必须接受：source change 到 serving readable 之间存在延迟。

### 什么时候不适用

不适合：

* 强一致交易状态；
* 用户写入事务；
* 账户余额权威状态；
* 实时授权；
* 必须同步生效的业务规则。

这些应由 OLTP 或专门实时系统处理。

### 重新评估信号

如果应用开始向 serving store 写入状态，或者业务把 serving projection 当成权威事实，应立即重新设计 ownership 边界。

---

## 12. Decision 10：为什么数据平台不直接写业务 OLTP

### 推荐选择

数据平台可以发布结果，但不直接写业务 OLTP 的 transaction source of truth。

### 为什么这样设计

数据平台直接写 OLTP 会带来：

* blast radius 扩大；
* ownership 混乱；
* rollback 困难；
* 凭证风险；
* 审计复杂；
* 应用团队和数据团队责任混合。

因此，推荐使用受控 activation 模式：

* serving store；
* application-owned API；
* message queue；
* reverse ETL tool；
* manual approval workflow。

### 代价是什么

需要额外设计 activation pattern，而不是数据平台直接 update 业务表。

这增加了一些前期设计，但降低生产系统风险。

### 什么时候可以例外

如果业务系统明确提供受控 API，且 owner、audit、validation、rollback 都由业务系统管理，数据平台可以作为调用方之一。

但这仍然不是“直接写 OLTP 表”。

### 重新评估信号

如果数据 pipeline 开始拥有业务状态写入规则，说明边界已经偏离，应重新设计。

---

## 13. Decision 11：为什么治理要前置

### 推荐选择

PII、RBAC、audit、data quality、ownership、contract 和 retention 在设计阶段进入架构。

### 为什么这样设计

治理后置会导致：

* PII 扩散；
* raw data 被广泛访问；
* 权限难收回；
* 指标无人负责；
* 审计证据缺失；
* quality failure 无 owner；
* 临时数据产品无法下线。

这些问题后续补救的复杂度远高于前置设计。

### 代价是什么

前置治理会增加初始设计和流程成本。

但好的治理不应该是重审批，而应是清晰边界、metadata 和 owner 机制。

### 什么时候可以轻量化

PoC、sandbox、低敏数据可以轻量治理，但需要明确它们不能直接升级为生产数据产品。

### 重新评估信号

如果用户开始直接依赖未治理模型，或者 PII 出现在非预期层级，应立即加强治理边界。

---

## 14. Decision 12：为什么 FinOps 要前置

### 推荐选择

从架构设计阶段引入 workload attribution、warehouse strategy、query tag、materialization discipline 和 monthly review。

### 为什么这样设计

Snowflake 或类似平台容易快速产生价值，也容易快速产生不可解释成本。

如果没有成本归因，团队只能看到总账单，却无法回答：

* 哪个 workload 花费最多；
* 哪个 domain 成本增长最快；
* 哪些 BI refresh 没有价值；
* 哪些 near real-time 模型过度设计；
* 哪些迁移成本只是临时 dual-run；
* 哪些旧系统没有下线。

### 代价是什么

FinOps 前置需要工程纪律：query tag、owner、metadata、review、guardrails。

### 什么时候可以简化

早期 PoC 可以简化 FinOps，但进入生产后必须建立成本归因。

### 重新评估信号

如果 Snowflake 成本增长但无法解释来源，或者平台价值无法和业务场景关联，说明 FinOps 机制不足。

---

## 15. Decision 13：为什么迁移不能只以上线为目标

### 推荐选择

迁移按以下生命周期管理：

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

### 为什么这样设计

数据平台迁移的难点不在新平台能不能建出来，而在消费者能否安全切换，以及旧平台能否真正下线。

如果没有 decommission，新平台只是增加复杂度。

### 代价是什么

渐进式迁移比 Big Bang 更长，需要 dual-run、parity、沟通和治理。

但它降低了业务风险。

### 什么时候可以简化

低风险、无依赖、低价值对象可以快速迁移并下线。

核心业务、财务、合规、应用依赖对象必须谨慎迁移。

### 重新评估信号

如果 dual-run 长期没有退出条件，或者旧平台在新平台上线后继续增长，说明迁移治理失效。

---

## 16. 决策矩阵

| 决策               | 推荐方向                                                     | 核心理由                         | 主要代价                  | 重新评估信号                     |
| ---------------- | -------------------------------------------------------- | ---------------------------- | --------------------- | -------------------------- |
| Analytical core  | Snowflake as reference core                              | 降低系统拼装复杂度                    | 依赖 Snowflake 能力边界     | 大量 workload 需要绕开 Snowflake |
| Main data path   | Sources → Landing → Bronze → Silver → Gold → Consumption | 降低心智负担                       | 简单场景可能偏重              | 大量例外路径出现                   |
| Landing / Bronze | 职责分离，物理可合并                                               | replay、audit、governance 边界清楚 | 多一层管理                 | Landing 没有实际价值             |
| CDC              | change awareness，不等于 real time                           | 避免过度 streaming               | 需要端到端 freshness 定义    | 业务要求事件级 SLA                |
| Medallion        | semantic decoupling                                      | OLTP 可变，业务语义稳定               | 需要建模纪律                | Silver / Gold 职责混乱         |
| Near real-time   | 按业务价值选择                                                  | 降低过度刷新成本                     | freshness 不统一         | 业务频繁抱怨延迟或成本过高              |
| Dynamic Tables   | 用于明确 near real-time                                      | 简化增量刷新                       | 持续刷新成本                | 使用率低、成本高                   |
| Operational read | Serving projection                                       | 避免应用直连 Snowflake             | 增加 serving path       | 应用 QPS / SLA 上升            |
| Serving store    | projection，不是 source of truth                            | 可重建、可审计                      | eventual consistency  | serving 被当成事实源             |
| Reverse ETL      | 不直接写 OLTP                                                | 降低 blast radius              | 需要 activation pattern | 数据平台拥有业务写入逻辑               |
| Governance       | 前置设计                                                     | 降低权限和 PII 债务                 | 初始流程成本                | raw / PII 扩散               |
| FinOps           | 前置归因                                                     | 成本可解释                        | 需要工程纪律                | 成本增长无法解释                   |
| Migration        | decommission 为终点                                         | 真正降低复杂度                      | 周期更长                  | permanent dual-run         |

---

## 17. 常见错误决策模式

### 17.1 工具驱动，而不是问题驱动

错误模式：因为某个工具流行，所以引入。

更好的方式：先定义 workload、SLA、ownership、governance 和 TCO，再选择工具。

### 17.2 把局部最优当成全局最优

一个局部方案可能很快，但会增加整体复杂度。

例如，一个临时 serverless job 解决了一个需求，但增加了新的权限、监控、失败模式和 owner。

### 17.3 用实时解决语义问题

有些问题不是数据不够实时，而是业务定义不清、数据模型不稳定、quality 不可信。

加速刷新不会解决语义混乱。

### 17.4 用治理补丁解决架构混乱

如果模型层级混乱，后面包再多 view、masking 和权限，也只是增加复杂度。

治理应该和架构一起设计。

### 17.5 迁移只建设新系统，不移除旧系统

这是最常见的现代化失败模式。

新平台上线后，旧平台继续运行，团队获得的是双倍复杂度。

---

## 18. 如何使用这篇 decision rationale

这篇文档可以用于：

* 架构评审；
* 新数据平台设计；
* 迁移路线讨论；
* vendor / platform 选型比较；
* near real-time 需求评估；
* governance 和 FinOps 设计；
* operational serving 方案评审；
* 判断某个需求是否应该进入 Snowflake 主链路。

使用方式不是机械套用结论，而是用每个 decision 的结构进行讨论：

```text
Recommendation
  -> Why
  -> Cost
  -> When not to use
  -> Re-evaluation signals
```

如果某个项目的条件不同，结论可以不同。但应该明确解释为什么不同。

---

## 19. 小结

这套 playbook 的核心不是 Snowflake，也不是 Medallion、CDC、Dynamic Tables 或 serving store。

核心是架构判断：

* 什么时候应该收敛系统边界；
* 什么时候应该引入额外系统；
* 什么时候 near real-time 足够；
* 什么时候必须 true streaming；
* 什么时候 Snowflake 是合适核心；
* 什么时候 Snowflake 应该只是下游 analytical sink；
* 什么时候 serving projection 能解决问题；
* 什么时候必须回到业务 OLTP 或专门实时系统；
* 什么时候治理应该前置；
* 什么时候迁移才算真正完成。

我的最终判断是：

> 好的数据平台架构，不是把所有能力都塞进一个系统，也不是把所有工具都接起来，而是在每个边界上做清楚的选择，并能解释这些选择的业务价值、技术代价和重新评估条件。

这也是这套 Lakehouse Architecture Playbook 想要表达的核心：用低复杂度的系统设计，支撑足够及时、可治理、可迁移、TCO 可解释的数据平台。
