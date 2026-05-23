# 06 · FinOps 与 TCO：不要只看账单，要看整体运营成本

## 1. 文档目的

这篇文档讨论 lakehouse 架构中的 FinOps 与 TCO 设计，重点是 **如何理解 Snowflake-centric 架构下的成本结构，如何让成本可以归因，如何避免 near real-time、BI、ad hoc 和 serving 场景带来不可控开销，以及为什么 TCO 比单项工具成本更重要**。

它不是一份 Snowflake 成本优化参数手册，也不会讨论具体价格、合同、折扣、采购或内部预算。这里关注的是更高层的系统设计问题：

> 当 Snowflake 或类似平台成为分析型数据平台核心时，如何从架构设计阶段就让成本可解释、可归因、可治理，并且让技术选择服务于长期 TCO，而不是只优化局部账单？

我的基本观点是：

> 数据平台的真实成本不是某一个 warehouse 或某一个云服务的账单，而是整个系统的总拥有成本：工具数量、系统集成、调度复杂度、人员技能、故障排查、治理、安全、迁移和长期维护成本。

因此，FinOps 不应该只是月底看账单。FinOps 应该是一套贯穿架构、建模、调度、消费和迁移的 operating model。

---

## 2. 为什么 TCO 比单项成本更重要

很多平台成本讨论会集中在单项问题上：

* 某个 warehouse 是否太贵；
* 某个 query 是否跑太久；
* 某个 Dynamic Table 是否刷新太频繁；
* 某个 BI dashboard 是否消耗太多；
* 某个对象存储层是否多存了一份数据。

这些问题都重要，但它们不是全部。

### 2.1 单项成本低，不代表整体成本低

一个架构可能在某个组件上看起来便宜，但整体 TCO 很高。

例如：

* 使用多个低成本服务，但需要复杂 glue code；
* 使用自建调度器，但需要更多维护和 on-call；
* 使用多个数据存储，但需要额外同步和一致性检查；
* 将业务逻辑分散在脚本、BI 和应用中，导致排障时间增加；
* 省掉标准 Landing，但 backfill 和 audit 变得困难；
* 省掉治理 metadata，但后续权限和 lineage 失控。

从账单看，这些成本可能不在同一个服务里；从组织看，它们都是真实成本。

### 2.2 单一核心平台可能降低整体复杂度成本

选择 Snowflake 作为分析核心，可能会让一部分成本集中在 Snowflake 账单上。

但如果它减少了：

* 外部调度器；
* 自定义脚本；
* 多套权限模型；
* 多个数据存储之间的同步；
* 多套监控和告警；
* 跨系统排障；
* 重复的工程技能要求；
* 旧平台长期运行；

那么整体 TCO 可能反而更低。

这也是为什么 Snowflake-centric architecture 的经济性，不能只用 Snowflake 账单判断。它应该和被减少的系统复杂度一起评估。

### 2.3 TCO 的几个组成部分

可以把数据平台 TCO 拆成以下几类：

| 成本类型               | 示例                                                          |
| ------------------ | ----------------------------------------------------------- |
| Compute cost       | warehouse、serverless compute、refresh、query processing       |
| Storage cost       | hot storage、archive、clone、intermediate tables、serving store |
| Integration cost   | connector、API、CDC、file movement、schema handling             |
| Orchestration cost | scheduler、task runner、retry、dependency management           |
| Observability cost | monitoring、logging、alerting、incident response               |
| Governance cost    | RBAC、PII、audit、lineage、data contracts                       |
| Engineering cost   | development、maintenance、debugging、on-call、training          |
| Migration cost     | dual-run、parity、cutover、shadow、decommission                 |
| Opportunity cost   | 团队花在维护复杂系统上，而不是业务数据产品上                                      |

FinOps 不是只管理第一项，而是要帮助团队理解整个结构。

---

## 3. Snowflake 成本为什么需要设计阶段介入

Snowflake 的一个特点是使用门槛低。SQL、BI、Tasks、Dynamic Tables、Snowpark、ad hoc analysis 都可以很快开始产生价值。

但也正因为如此，成本也很容易在无形中增长。

### 3.1 成本来自 workload，不是只来自平台

Snowflake 成本通常不是一个单一来源。

它可能来自：

* ingestion；
* transformation；
* BI dashboard refresh；
* ad hoc queries；
* data quality checks；
* Dynamic Table refresh；
* serverless features；
* operational serving publish；
* observability queries；
* backfill；
* migration dual-run；
* storage and retention。

如果没有 workload 维度，团队只会看到“Snowflake 花了多少钱”，但不知道为什么。

### 3.2 没有归因，就没有管理

不能归因的成本，很难被管理。

如果一个 query 没有 owner，一个 warehouse 没有用途，一个 model 没有 domain，一个 BI dataset 没有 refresh owner，一个 Dynamic Table 没有业务 freshness 理由，那么成本优化只能变成猜测。

好的 FinOps 设计应该让团队能回答：

* 哪个 domain 消耗最多？
* 哪个 use case 增长最快？
* 哪些 BI refresh 成本高但业务价值低？
* 哪些 near real-time 模型没有必要持续刷新？
* 哪些 ad hoc query 影响生产 workload？
* 哪些 backfill 是一次性迁移成本？
* 哪些成本来自 legacy dual-run？

### 3.3 成本控制不能只靠事后优化

事后优化当然必要，但不够。

如果架构本身没有：

* workload separation；
* query attribution；
* materialization discipline；
* ownership metadata；
* resource guardrails；
* monthly review；
* decommission process；

那么成本会不断重新膨胀。

FinOps 应该从设计阶段进入，而不是等账单失控后再补。

---

## 4. 成本归因模型

FinOps 的第一步是建立成本归因模型。

### 4.1 按 workload 归因

建议至少区分以下 workload：

| Workload                    | 说明                                                         |
| --------------------------- | ---------------------------------------------------------- |
| Ingestion                   | 数据加载、格式处理、metadata 记录                                      |
| Transformation              | Bronze / Silver / Gold 模型构建                                |
| BI                          | dashboard query、semantic model refresh、scheduled reporting |
| Ad hoc                      | 分析师、工程师和业务用户临时查询                                           |
| Data quality                | tests、reconciliation、validation                            |
| Near real-time refresh      | Dynamic Tables、Streams、Tasks、incremental refresh           |
| Operational serving publish | 将 Gold 结果发布到 serving projection                            |
| Observability               | audit、monitoring、metadata、cost attribution query           |
| Backfill / migration        | 历史重建、dual-run、迁移验证                                         |

这种分类的价值在于：不同 workload 的优化方式不同。

BI 成本高，不一定通过调低 transformation warehouse 解决。Dynamic Table 成本高，不一定通过减少 ad hoc query 解决。

### 4.2 按 domain 归因

仅按 workload 不够，还需要按业务 domain 归因。

例如：

* customer；
* product；
* finance；
* operations；
* marketing；
* risk；
* platform。

Domain attribution 的目的不是做内部收费，而是让业务和技术对成本有共同语言。

当一个 domain 要求更高 freshness、更复杂模型或更多 BI refresh 时，成本影响应该可见。

### 4.3 按 data product 归因

最终，成本最好能落到 data product 或 use case。

例如：

* 一个核心 Gold mart；
* 一个 BI semantic model；
* 一个 serving projection；
* 一个 reconciliation pipeline；
* 一个 high-frequency dashboard；
* 一个 migration backfill job。

这能帮助团队判断：这个数据产品产生的业务价值是否匹配其计算、存储和运营成本。

### 4.4 Query Tag 和 metadata

在 Snowflake 语境下，query tagging 是成本归因的重要机制。

一个成熟的 query tag 或等价 metadata 应该尽量包含：

* app 或 runtime；
* domain；
* model；
* environment；
* owner；
* use case；
* pipeline；
* job id；
* git sha 或 deployment version；
* cost attribution key。

不一定所有场景一开始都能做到完整，但应该有逐步提高归因率的目标。

---

## 5. Warehouse 与 Workload Strategy

在 Snowflake-centric 架构中，warehouse strategy 是 FinOps 的核心设计之一。

### 5.1 Warehouse 不是越多越好

每个 workload 都创建独立 warehouse，看起来隔离清楚，但会带来管理复杂度：

* owner 难管理；
* 配额难管理；
* idle cost 容易增加；
* right-sizing 更复杂；
* 成本碎片化；
* 命名和权限更难维护。

### 5.2 Warehouse 也不是越少越好

所有 workload 共用一个 warehouse 也有问题：

* BI 和 batch transformation 相互影响；
* ad hoc query 影响生产模型；
* cost attribution 模糊；
* queue time 无法定位；
* workload tuning 失去针对性。

### 5.3 合理的中间状态

建议按主要 workload 边界设计一组有限 warehouse，例如：

* ingestion；
* transformation；
* BI；
* ad hoc；
* observability；
* serving publish。

这不是固定标准，而是设计思路：

> warehouse 应该足够少，以降低管理复杂度；也应该足够分离，以避免不同 workload 相互污染。

### 5.4 Auto-suspend 和 right-sizing

常见原则：

* interactive / ad hoc workload 应该快速 auto-suspend；
* scheduled transformation 应按运行窗口调整；
* BI workload 需要关注 queue time 和 concurrency；
* backfill 可以临时扩容，但应有结束时间；
* warehouse size 调整应基于 query profile、spill、queue 和 SLA，而不是直觉。

Right-sizing 不是单纯降配，也包括合理升配。

如果一个查询用小 warehouse 跑很久，可能比用较大 warehouse 快速完成更贵。

---

## 6. Materialization Discipline

物化策略直接影响成本。

### 6.1 View 不一定便宜

View 没有存储成本，但查询时会重复计算。

如果一个复杂 view 被高频 BI dashboard 反复查询，它可能比物化 table 更贵。

适合 view 的场景：

* 轻量逻辑；
* 低频查询；
* 权限 projection；
* 快速实验；
* 不需要稳定性能。

不适合 view 的场景：

* 复杂 join；
* 高频 BI；
* 大量用户并发；
* 重复计算代价高的模型。

### 6.2 Table 不一定落后

很多 daily 或 hourly mart 使用 table 是合理的。

如果业务不需要分钟级 freshness，定时 table materialization 通常更简单、更可控、更容易归因。

不要因为平台支持 near real-time，就把所有模型都做成持续刷新。

### 6.3 Dynamic Table 需要业务理由

Dynamic Table 或类似持续刷新机制，适合：

* 明确 near real-time 需求；
* 数据变化频繁；
* 消费者确实需要较低延迟；
* 刷新成本可以被业务价值证明；
* 模型逻辑适合自动增量维护。

不适合：

* 每日报表；
* 低频查询；
* 业务不关心分钟级延迟；
* 模型逻辑复杂但收益不清；
* 只是为了“看起来更实时”。

### 6.4 Serving projection 也有成本

Operational serving 不只是数据平台外部的成本。

它会带来：

* Gold source model refresh；
* publish job；
* change detection；
* serving store storage；
* read capacity；
* monitoring；
* reconciliation；
* retry and error handling。

因此，serving projection 应该只用于明确的 key-based operational read，而不是把所有 Gold 都发布出去。

---

## 7. Near Real-Time 的成本边界

Near real-time 是成本放大的常见来源。

因为它通常意味着：

* 更频繁刷新；
* 更多增量检查；
* 更复杂 orchestration；
* 更高 observability 要求；
* 更严格 SLA；
* 更多失败重试；
* 更多下游一致性问题。

### 7.1 Freshness 应该被定价

任何 near real-time 需求都应该回答：

* 需要多新鲜？
* 是分钟级、小时级，还是日级？
* 延迟降低是否改变业务决策？
* 谁消费这个数据？
* 消费频率多高？
* freshness 失败的业务影响是什么？
* 是否有更简单的 batch 替代方案？

如果这些问题回答不清，就不应该默认建设 near real-time。

### 7.2 CDC 不等于低成本

CDC 避免了 full reload，但不代表成本一定低。

CDC 可能增加：

* offset management；
* delete handling；
* merge cost；
* late-arriving handling；
* schema drift complexity；
* replay complexity；
* duplicate handling；
* small file problem。

CDC 的价值是 change awareness 和 incremental capture，不是自动降低所有成本。

### 7.3 分层定义 freshness

不要只说“这个 pipeline 是 near real-time”。

应该拆成：

| 层       | Freshness 定义                    |
| ------- | ------------------------------- |
| Landing | 源变化到达 Landing 的延迟               |
| Bronze  | 源变化进入 raw query layer 的延迟       |
| Silver  | 业务镜像更新延迟                        |
| Gold    | 业务指标刷新延迟                        |
| Serving | 应用可读取 projection 的延迟            |
| BI      | dashboard 或 semantic model 可见延迟 |

这样才能找到真正的瓶颈，也才能判断成本花在哪里。

---

## 8. BI 与 Ad Hoc 成本

BI 和 ad hoc 分析是 Snowflake 平台中最容易被低估的成本来源。

### 8.1 BI refresh 需要 owner

BI dashboard 或 semantic model 的 refresh 不应该是无 owner 的后台任务。

每个重要 BI refresh 应该有：

* owner；
* business purpose；
* refresh frequency；
* source Gold model；
* expected query cost；
* user base；
* retirement rule。

如果一个 dashboard 没有人使用，或使用价值不清，它不应该长期高频刷新。

### 8.2 BI 不应该重复实现核心指标

如果 BI 报表中重复实现复杂指标，成本和语义都会失控。

更好的方式是：

* 核心指标在 Gold 定义；
* BI 做展示和轻量 measure；
* 高频报表使用 aggregate mart；
* 重复昂贵计算物化到合适层。

### 8.3 Ad hoc workload 应该隔离

Ad hoc 查询有价值，但不应影响生产 transformation 或 BI。

常见做法：

* 使用独立 warehouse；
* 设置 auto-suspend；
* 设置资源限制；
* 对高成本查询进行 review；
* 提供 approved Gold / semantic models；
* 清理长期 sandbox 和 clone。

Ad hoc 自由度和平台稳定性需要平衡。

---

## 9. Storage、Clone 与 Retention 成本

Snowflake 成本不只是 compute。

存储和保留策略也会影响 TCO。

### 9.1 Raw、intermediate、Gold 的存储价值不同

不同数据层的存储价值不同：

* Landing：source evidence、replay、archive handoff；
* Bronze：raw query layer、rebuild foundation；
* Silver：business mirror、reusable entity layer；
* Gold：business consumption、metrics、data products；
* temporary / sandbox：短期实验；
* serving projection：应用点查副本。

不是所有中间结果都需要长期保留。

### 9.2 Clone 和 sandbox 需要 TTL

Zero-copy clone 或类似能力很有价值，但如果长期存在，会增加隐性成本和治理复杂度。

建议：

* clone 有 owner；
* clone 有 purpose；
* clone 有 TTL；
* sensitive clone 有更严格权限；
* 过期 clone 自动提醒或清理；
* 长期保留需要明确理由。

### 9.3 Retention 是 FinOps 和 Governance 的交叉点

保留策略同时影响成本和合规。

需要区分：

* 哪些数据是 record of origin；
* 哪些数据可以从上游重建；
* 哪些数据必须长期保留；
* 哪些数据只是临时计算；
* 哪些 serving projection 可以重新发布；
* 哪些数据因隐私要求必须删除或缩短保留。

---

## 10. Observability 与 FinOps

没有 observability，就没有有效 FinOps。

### 10.1 需要观察什么

FinOps dashboard 或 operating review 至少需要关注：

* 总 compute consumption；
* 按 workload 的消耗；
* 按 domain 的消耗；
* 按 warehouse 的消耗；
* 按 model / pipeline 的消耗；
* BI refresh cost；
* Dynamic Table refresh cost；
* ad hoc cost；
* serving publish cost；
* storage growth；
* clone / sandbox inventory；
* unattributed cost；
* cost anomalies。

### 10.2 成本异常需要 context

成本突然上升不一定是坏事。

可能是：

* 业务活动增加；
* 新模型上线；
* backfill；
* migration dual-run；
* BI 使用量增长；
* query regression；
* warehouse size 调整；
* Dynamic Table 频繁刷新；
* schema drift 导致处理异常。

FinOps 的作用不是简单压低成本，而是解释成本变化，并判断它是否有业务价值。

### 10.3 Unattributed cost 是流程问题

不能归因的成本，应视为流程问题。

如果一部分查询、warehouse 或任务无法归因到 owner、domain、model 或 use case，说明平台 metadata 或工程规范不足。

目标不是一开始做到 100%，但应该持续降低 unattributed cost。

---

## 11. FinOps Operating Model

FinOps 不是一次性优化项目，而是持续运营机制。

### 11.1 月度 review

建议建立固定节奏，例如 monthly FinOps review。

讨论内容包括：

* 上月成本趋势；
* top cost movers；
* cost anomalies；
* unattributed workloads；
* BI refresh review；
* near real-time model review；
* warehouse right-sizing；
* storage growth；
* clone cleanup；
* new workload requests；
* migration / dual-run cost；
* decommission progress。

### 11.2 需要哪些角色

不需要复杂委员会，但需要覆盖几类视角：

* platform / data engineering；
* analytics engineering；
* BI owner；
* business domain owner；
* finance / cost owner；
* governance / security when relevant。

FinOps 不是财务团队单独能完成的，也不是工程团队单独能完成的。

### 11.3 Review 的输出

每次 review 应输出：

* action list；
* workload owner；
* cost anomaly explanation；
* approved or rejected new workload；
* right-sizing decision；
* models to optimize；
* dashboards to retire；
* clones to clean；
* decommission next steps。

如果 review 没有 action，只是看 dashboard，那么它不是 operating model。

---

## 12. Migration 与 Dual-Run 成本

迁移期间，成本通常会短期上升。

这是正常的，因为新旧平台同时运行。

### 12.1 Dual-run premium

Dual-run 成本包括：

* 新平台 ingestion；
* 新平台 transformation；
* 新旧数据对账；
* parity checks；
* BI 双路径验证；
* historical backfill；
* shadow period；
* legacy 平台继续运行。

这些成本不应被误判为新平台 steady-state 成本。

### 12.2 迁移成本需要单独归因

建议把 migration / backfill / dual-run 相关 workload 单独标记。

否则新平台上线初期，成本会看起来异常高，导致管理层误判。

### 12.3 没有 decommission，TCO 不会下降

迁移的经济性依赖旧复杂度被移除。

如果旧 pipeline、旧报表、旧调度、旧数据库、旧同步脚本继续长期运行，新平台只会增加总成本，而不会降低 TCO。

因此 FinOps 必须和 migration decommission 绑定。

---

## 13. Guardrails：成本控制不是阻止创新

FinOps guardrails 的目标不是让团队不敢使用平台，而是避免无意识的浪费和风险。

### 13.1 常见 guardrails

可以考虑：

* warehouse auto-suspend；
* resource monitor；
* query timeout；
* ad hoc warehouse isolation；
* tagging requirement；
* backfill approval；
* near real-time model review；
* high-frequency BI refresh review；
* clone TTL；
* sandbox cleanup；
* storage lifecycle；
* serving projection approval。

### 13.2 Guardrails 应该有例外流程

没有例外流程的 guardrail 会阻碍业务。

合理的例外流程应该要求：

* 业务理由；
* owner；
* 时间范围；
* 预期成本影响；
* rollback 或 cleanup plan；
* review date。

例外可以存在，但不能变成永久默认。

---

## 14. 核心 trade-off

| 设计选择                    | 好处                | 代价                                 |
| ----------------------- | ----------------- | ---------------------------------- |
| 使用 Snowflake 作为分析核心     | 降低系统拼装复杂度，统一建模和消费 | 架构更依赖 Snowflake 原生能力边界；非适配场景需要额外系统 |
| 统一 workload attribution | 成本可解释、可优化         | 需要 query tag、metadata 和工程规范        |
| 限定 warehouse 集合         | 降低管理复杂度           | 某些 workload 隔离度可能不足                |
| 拆分关键 workload warehouse | 隔离性能和成本           | warehouse 管理复杂度增加                  |
| 使用 Dynamic Table        | 支持 near real-time | 持续刷新成本需要业务价值支撑                     |
| 使用 batch table          | 简单、成本可控           | freshness 较低                       |
| 使用 view                 | 灵活、少存储            | 高频查询可能重复计算                         |
| 使用 serving projection   | 应用读取稳定            | 多一层发布和存储成本                         |
| 月度 FinOps review        | 持续优化和问责           | 需要跨团队参与                            |
| 强 guardrails            | 降低失控风险            | 需要合理例外流程                           |

---

## 15. 常见反模式

| 反模式                                  | 问题           | 更好的做法                                 |
| ------------------------------------ | ------------ | ------------------------------------- |
| 只看 Snowflake 账单，不看整体 TCO             | 低估其他系统和运维成本  | 按平台 operating model 评估成本              |
| 所有 workload 共用一个 warehouse           | 成本和性能无法归因    | 按主要 workload 分离                       |
| 每个小场景都建独立 warehouse                  | 管理复杂度上升      | 保持有限 warehouse 集合                     |
| Query 没有 tag 或 owner                 | 成本无法解释       | 使用 query tag 和 metadata               |
| 所有 Gold 都用 Dynamic Table             | 刷新成本可能高于业务价值 | 根据 freshness 选择物化方式                   |
| BI 高频刷新无人负责                          | 成本持续增长       | 每个 refresh 有 owner 和 business purpose |
| Ad hoc 影响生产 workload                 | 生产稳定性下降      | 隔离 ad hoc warehouse                   |
| Backfill 不单独标记                       | 迁移成本被误判为稳态成本 | 标记 migration / backfill workload      |
| Clone 和 sandbox 长期不清理                | 隐性存储和治理成本上升  | 设置 TTL 和 cleanup 流程                   |
| FinOps review 只有 dashboard 没有 action | 没有形成运营机制     | 输出 owner、action 和 decision            |
| 只压低成本，不看业务价值                         | 可能伤害平台交付     | 用 value / cost / risk 综合判断            |

---

## 16. 成功标准

一套设计良好的 FinOps 和 TCO 机制，应满足：

* 成本可以按 workload、domain、model、pipeline 和 use case 归因；
* 主要 warehouse 有明确用途和 owner；
* BI refresh 有 owner、频率和业务目的；
* near real-time 模型有明确 freshness 理由；
* Dynamic Table 或持续刷新机制有成本 review；
* ad hoc workload 与生产 workload 隔离；
* serving projection 成本被纳入平台视角；
* storage、clone、sandbox 有生命周期策略；
* migration / backfill / dual-run 成本被单独识别；
* unattributed cost 持续下降；
* 月度 review 产生实际 action；
* 成本优化不会破坏业务 SLA；
* TCO 评估包括工具、人员、排障、治理和迁移成本。

---

## 17. 小结

FinOps 不是月底看账单，也不是单纯压低 Snowflake 成本。

在 lakehouse 架构中，FinOps 的真正目标是让平台成本可解释、可归因、可优化，并且能和业务价值、系统复杂度、治理边界和迁移目标放在一起讨论。

Snowflake-centric architecture 可能让一部分成本集中在 Snowflake 账单上，但如果它减少了外围系统、外部调度、重复存储、自定义脚本、复杂排障和旧平台长期运行，那么整体 TCO 可能更低。

因此，判断一套架构是否经济，不应该只看某个服务的单项成本，而应该看：

* 它是否减少系统边界；
* 它是否降低长期维护成本；
* 它是否让成本可归因；
* 它是否帮助旧系统下线；
* 它是否让团队把更多时间投入到数据产品，而不是 glue code 和救火。

好的 FinOps，不是限制数据平台创造价值，而是让平台价值和成本都能被清楚地理解和管理。
