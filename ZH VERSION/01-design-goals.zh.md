# 01 · 设计目标：降低复杂度，而不是堆叠工具

## 1. 文档目的

这篇文档定义整套 Snowflake Lakehouse Playbook 的设计目标。

它不讨论具体如何配置 Snowflake、如何编写 SQL、如何创建 pipeline，也不试图证明 Snowflake 是所有数据平台问题的唯一答案。它关注的是更前置的问题：

> 当我们设计一套基于 Snowflake 的湖仓一体数据平台时，到底想解决什么问题？什么样的复杂度值得消除？什么样的复杂度必须接受？如何在数据新鲜度、治理、安全、成本和可运营性之间做取舍？

我的基本判断是：

> 现代数据平台最难的问题，往往不是缺少工具，而是系统复杂度持续累积之后，团队已经无法稳定理解、治理、演进和下线这套系统。

因此，这套 playbook 的设计目标不是“把所有先进工具都接进来”，而是设计一套足够简单、边界清楚、可治理、可迁移、TCO 可解释的数据平台。

Snowflake 在这里是一个重要选择，但它不是信仰。选择 Snowflake-centric architecture 的理由，是希望把分析型数据平台的大部分能力收敛到一个核心系统中，从而减少跨系统编排、重复存储、重复调度、权限碎片化和故障排查复杂度。

---

## 2. 为什么系统复杂度是现代数据平台的核心问题

数据平台的复杂度不是一次性出现的。它通常是多年演进的结果。

一开始，团队可能只是为一个报表建了一个同步脚本。后来多了一个调度器、一个数据仓库、一个 BI 数据集、一个数据质量检查、一个数据导出任务、一个 reverse ETL job、一个临时 Lambda、一个手工补数流程。每一个单独看都合理，但放在一起之后，平台会变成一组难以解释的链路。

这种复杂度通常来自几个方向。

### 2.1 数据链路复杂度

不同源系统使用不同接入方式：数据库 CDC、文件、API、第三方 feed、手工导出、应用事件流。每个源系统都有自己的命名、调度、错误处理、重放方式和负责人。

结果是：新增一个数据源不是复制一个成熟模式，而是重新做一次架构设计。

### 2.2 调度和编排复杂度

数据处理可能分布在多个地方：数据库任务、Python 脚本、Airflow DAG、云函数、CI/CD workflow、BI refresh、应用后台任务。

当数据延迟或口径错误时，团队需要同时查看多个系统的日志、权限、重试和状态。这种跨系统排障本身就是高成本。

### 2.3 语义和建模复杂度

如果报表、BI、数据导出和应用消费都直接依赖源系统表，业务逻辑就会散落到多个地方。源系统一改字段，下游全部受影响；业务口径一升级，也很难判断应该改哪一层。

这也是为什么我认为 Medallion 架构的价值不在于 Bronze / Silver / Gold 这些名字，而在于解耦两类数据操作：

1. 把 OLTP 系统中的存储结构转换成相对稳定的业务镜像；
2. 从业务镜像中提取指标、报表、数据产品和 operational projection。

解耦之后，OLTP 系统可以变化，业务逻辑可以升级，下游消费也不需要直接承受源系统的每一次结构变化。

### 2.4 治理和权限复杂度

治理不是只有数据隐私问题。它还包括：谁能访问什么数据，哪些字段是 PII，哪些模型可以被 BI 使用，哪些数据可以被应用消费，哪些结果可以回写业务流程，哪些查询需要审计，哪些数据需要保留或删除。

如果治理能力后置，平台上线后再补权限、审计、PII、数据质量和 lineage，系统复杂度会快速上升。治理逻辑会散落在 SQL、BI、脚本、IAM、应用代码和人工流程中，最终很难证明平台是可控的。

### 2.5 成本和 TCO 复杂度

很多数据平台的成本问题不是“某个服务贵”，而是没人能解释成本来自哪里。

如果查询没有归因，warehouse 没有 ownership，pipeline 没有 use case，BI refresh 没有成本视角，near real-time 模型没有 materialization discipline，那么月底账单只能变成争论，而不能变成管理动作。

更重要的是，TCO 不只是云账单。它还包括人员技能、系统集成、故障排查、迁移维护、安全审计和长期演进成本。

### 2.6 迁移复杂度

很多新平台失败，不是因为新架构不可行，而是因为旧平台没有退出机制。

如果没有 dual-run、parity、cutover、shadow 和 decommission 机制，新平台上线后，旧平台会长期保留。结果不是现代化，而是双平台长期并存。

现代云平台提供了大量托管服务和集成工具，这本身是好事。它让团队可以很快把各种特异化需求接入平台：一个源系统用一个托管 CDC 服务，一个文件源用一个对象存储触发器，一个业务流程用一个 serverless function，一个报表刷新用一个外部 scheduler，一个应用消费再加一个临时 API。

这些选择在局部看都合理，也通常能快速交付。但如果缺少统一架构原则，它们会让不统一的系统设计更容易被“快速融合”进来。短期交付速度提升，长期却积累了更多技术债：更多权限模型、更多调度入口、更多日志位置、更多失败模式、更多无人愿意下线的临时链路。

因此，云平台的便利性并不会自动降低复杂度。它只是降低了创建新组件的门槛。如果没有清晰的主链路和治理边界，核心问题反而会变得更严峻。

---

## 3. 这套架构的设计目标

这套 playbook 的目标可以概括为七点。

### 3.1 用一条主链路降低心智负担

平台应该有一条容易解释的主链路：

```text
Sources
  -> Landing
  -> Bronze
  -> Silver
  -> Gold
  -> BI / Analytics / Operational Serving
```

这条链路的意义不是限制所有实现，而是提供一个默认心智模型。大多数数据源、大多数转换、大多数消费模式都应该能放进这条链路中解释。

如果每个源系统、每个报表、每个应用消费都需要一条特殊链路，平台长期一定会失控。

### 3.2 尽量使用 Snowflake 原生能力

如果一个分析型数据平台已经选择 Snowflake 作为核心，那么很多能力应优先评估 Snowflake 原生方案：

* 数据加载；
* SQL transformation；
* incremental processing；
* Dynamic Tables；
* Streams；
* Tasks；
* data quality checks；
* workload isolation；
* cost attribution；
* access control。

这不是因为 Snowflake 一定比所有其他工具更强，而是因为系统边界越少，长期运维越简单。

每引入一个额外系统，都要问：

* 它解决的问题是否 Snowflake 无法合理解决？
* 它带来的复杂度是否被业务价值证明？
* 团队是否有能力长期运营它？
* 它是否会增加权限、监控、调度、故障排查和 TCO？

### 3.3 明确 near real-time 的边界

这套架构关注的是 near real-time，而不是所有意义上的 real time。

在这里，near real-time 主要指分钟级或小批量增量刷新。它适合很多场景：运营指标、客户状态、风险提示、BI 加速、业务标签、数据产品同步。

但它不等于毫秒级 streaming，也不等于实时撮合、实时风控、高频交易或复杂事件处理。

CDC 是实现 near real-time 的一个重要能力，但 CDC 不等于 real time。CDC 让平台知道两次批处理或两次加载之间发生了什么变化，它解决的是 change awareness 和 incremental capture，而不是端到端实时消费。

有些场景确实需要 CDC，因为团队需要知道 batch loading 间隔内发生了哪些 insert、update、delete，并基于这些变化做增量加载、对账、补偿或重新计算。但这并不意味着必须把完整 Kafka / Flink / streaming stack 都引入进来。

设计目标应该是：用业务真正需要的 freshness 来决定架构，而不是用“是否有 CDC”来决定是否建设完整实时链路。

### 3.4 解耦 OLTP 结构和业务逻辑

源系统的 OLTP schema 服务于应用事务，不服务于分析平台的长期语义稳定性。

因此，数据平台不应该让所有报表、指标和下游消费直接依赖 OLTP 表结构。更合理的方式是：

* Bronze 保留源系统证据；
* Silver 构建稳定的业务镜像；
* Gold 承载业务指标和消费模型。

这样，源系统可以为了应用需求调整表结构，数据团队也可以独立演进业务逻辑、指标口径和数据产品。

### 3.5 将治理和隐私前置

治理不应该是最后才补的检查项。

设计时就需要考虑：

* PII 在哪里被识别；
* 哪些数据可以进入 common landing；
* 哪些字段需要 hash、tokenize 或隔离；
* 哪些角色可以访问哪些层；
* 哪些模型可以被 BI 使用；
* 哪些数据可以被 operational serving；
* 哪些访问需要审计；
* 哪些数据产品需要 owner 和 contract。

治理前置不是为了拖慢交付，而是为了避免平台上线后出现不可控的权限、隐私和审计债务。

### 3.6 让成本可以归因到业务和工作负载

Snowflake 的成本治理不能只依赖月底账单。

平台应该从设计阶段就考虑：

* 不同 workload 使用哪些 warehouse；
* 查询如何打 tag；
* BI refresh 如何归因；
* Dynamic Table 是否真的必要；
* near real-time 模型是否值得持续刷新；
* ad hoc query 如何隔离；
* 成本如何映射到 domain、model、pipeline、owner 和 use case。

目标不是绝对降低每一个查询的成本，而是让成本可以被解释、被管理、被优化。

### 3.7 支持渐进式迁移，而不是 Big Bang

数据平台迁移最怕一次性切换。

新的 lakehouse 应该支持：

* 新旧链路 dual-run；
* 数据一致性 parity check；
* 消费者逐步 cutover；
* shadow period；
* rollback path；
* legacy decommission。

如果没有 decommission 机制，迁移的结果往往不是新平台替代旧平台，而是新旧平台长期共存。

---

## 4. 为什么选择 Snowflake-centric Lakehouse

选择 Snowflake-centric architecture 的主要理由不是“Snowflake 什么都能做”，而是它可以减少很多中小型和中大型数据团队的系统拼装复杂度。

这并不意味着 Snowflake 是唯一合理选择。根据企业的技术栈、团队能力、数据规模、云平台偏好、治理要求和成本模型，也可以选择 Databricks、Microsoft Fabric、Amazon Redshift、Google BigQuery，或者其他 lakehouse / warehouse 架构。真正关键的问题不是选哪个品牌，而是平台是否能降低整体复杂度，是否能支撑治理和 TCO，是否能让团队长期稳定运营。

### 4.1 统一分析型存储与计算

Snowflake 可以承载 raw、cleaned、conformed、business-ready 多层数据模型。团队可以围绕 SQL、表、视图、动态表、任务和权限构建统一的数据开发和消费体验。

这比把数据分散在多个数据库、对象存储、脚本、缓存层和 BI 数据集中更容易治理。

### 4.2 减少外部调度和 glue code

很多平台复杂度来自 glue code：一段 Lambda 触发一个任务，一个脚本调用另一个系统，一个 scheduler 管理 Snowflake job，一个 CI workflow 负责生产调度。

如果主要转换、增量处理和调度可以尽量留在 Snowflake 运行时内，平台的故障面会更少，排障路径也会更清楚。

### 4.3 支持 SQL-first 数据工程

大多数分析型转换、建模、指标和质量检查都可以用 SQL 表达。Snowflake-centric architecture 可以让团队把更多逻辑放在数据平台内部，而不是散落在应用代码或外部脚本中。

这对数据团队的长期维护非常重要。

### 4.4 更容易做治理和成本归因

当 workload 集中在少数平台对象中，治理和成本归因更容易落地。

如果查询、模型、任务、warehouse 和角色都在同一个平台中可见，团队更容易建立 owner、tag、audit、quality 和 cost attribution 机制。

### 4.5 降低整体 TCO

选择单一核心系统不一定让单项账单最低，但可能降低总拥有成本。

少一个调度器，少一个 streaming cluster，少一组自定义脚本，少一套权限模型，少一组监控和 on-call 流程，都会降低长期复杂度。

这就是为什么 TCO 不能只看某个工具的价格，而要看整个平台的 operating model。

---

## 5. Snowflake-centric 不适合解决什么

这套设计并不试图把所有数据系统都收敛到 Snowflake。

以下场景通常不应该默认使用 Snowflake lakehouse 主链路解决：

* 毫秒级事件处理；
* 高频交易或实时撮合；
* 强实时风控；
* 超高吞吐 streaming；
* 复杂事件处理；
* 低延迟在线推理；
* 大规模 online feature serving；
* 应用事务写入；
* 强多区域 active-active；
* 对开源可移植性有强硬要求的架构。

这些场景需要根据自己的 SLA、吞吐、延迟、可靠性和团队能力单独设计。

架构成熟的表现，不是把所有问题都塞进一个平台，而是清楚知道哪些问题应该放进来，哪些问题应该留在外面。

---

## 6. 设计原则总结

### 原则一：默认减少系统边界

除非有明确业务理由，否则不要为每个能力引入一个新系统。

### 原则二：用主链路统一心智模型

Sources → Landing → Bronze → Silver → Gold → Consumption 应该是大多数场景的默认解释框架。

### 原则三：CDC 不等于 real time

CDC 解决变化捕获，不自动解决端到端实时消费。是否需要 true streaming，应由业务 SLA 决定。

### 原则四：Medallion 解耦 OLTP 和业务语义

Bronze 保留证据，Silver 构建业务镜像，Gold 支持业务消费。

### 原则五：Snowflake 是分析中心，不是 OLTP API

应用点查和事务写入不应直接落到 Snowflake 主分析链路上。

### 原则六：治理和 FinOps 前置

PII、权限、审计、质量、成本归因和 owner 机制都应该在平台设计阶段考虑。

### 原则七：TCO 比单项成本更重要

系统数量、人员技能、排障、监控、迁移和治理都是平台成本的一部分。

### 原则八：迁移必须有退出机制

没有 decommission 的 dual-run 会变成永久复杂度。

---

## 7. 核心 trade-off

| 目标                | 选择                                                  | 好处               | 代价                  |
| ----------------- | --------------------------------------------------- | ---------------- | ------------------- |
| 降低复杂度             | 以 Snowflake 为分析中心                                   | 减少跨系统编排和重复治理     | 平台集中度上升             |
| 支持 near real-time | 使用 CDC、micro-batch、Dynamic Tables、Streams、Tasks 等组合 | 支持分钟级 freshness  | 不适合毫秒级场景            |
| 解耦源系统             | 引入 Landing 和 Bronze                                 | 可回放、可审计、可隔离源系统变化 | 多一层存储和元数据管理         |
| 稳定业务语义            | 使用 Silver 业务镜像                                      | OLTP 可变，业务逻辑可升级  | 需要建模纪律              |
| 服务业务消费            | 使用 Gold 数据产品                                        | 指标和报表更一致         | 需要 owner 和 contract |
| 支持应用点查            | 使用 operational serving projection                   | 避免应用直连 Snowflake | 多一层 serving store   |
| 控制成本              | 强调 TCO 和归因                                          | 成本可解释、可优化        | 需要持续运营机制            |
| 降低迁移风险            | 使用 dual-run 和 cutover                               | 可验证、可回滚          | 短期成本和工作量上升          |

---

## 8. 成功标准

一套设计良好的 Snowflake-centric lakehouse，不应该只用“是否上线”来衡量。

更重要的成功标准包括：

* 新数据源可以复用标准 ingestion pattern；
* 大多数分析逻辑可以在 Snowflake 分层模型中解释；
* BI 指标主要来自 Gold，而不是散落在报表工具中；
* PII、权限和审计边界清楚；
* near real-time 场景有明确 freshness 定义；
* 不需要 true streaming 的场景没有被过度设计；
* 应用点查不直接访问 Snowflake 分析模型；
* 成本可以按 workload、domain、model 和 use case 归因；
* legacy pipeline 有明确 cutover 和 decommission 路径；
* 团队可以稳定运维，而不是依赖少数人记忆。

---

## 9. 常见反模式

| 反模式                           | 问题                                        | 更好的做法                               |
| ----------------------------- | ----------------------------------------- | ----------------------------------- |
| 因为有 CDC 就建设完整 streaming stack | 把 change capture 误解成 end-to-end real time | 根据业务 freshness 决定架构                 |
| 所有源系统都单独设计接入链路                | 长期运维和排障复杂                                 | 使用统一 landing 和 ingestion pattern    |
| 让 BI 直接依赖 OLTP 结构             | 源系统变化直接冲击报表                               | 用 Silver 构建业务镜像                     |
| 把所有业务逻辑写在 BI 工具中              | 指标口径漂移                                    | 在 Gold 层统一核心指标                      |
| 应用直接查询 Snowflake 做点查          | 成本、延迟和 workload 边界不可控                     | 使用 operational serving projection   |
| 数据平台直接写业务 OLTP                | blast radius 大，ownership 混乱               | 使用受控 reverse ETL / serving / API 模式 |
| 等上线后再补治理                      | PII、权限、审计债务扩大                             | 设计阶段前置治理                            |
| 只看工具账单，不看 TCO                 | 低估运维和复杂度成本                                | 建立 workload attribution 和 FinOps 机制 |
| 新平台上线后旧平台不下线                  | 永久双平台复杂度                                  | 设计 cutover、shadow、decommission      |

---

## 10. 小结

这套 playbook 的设计目标可以用一句话概括：

> 用尽可能少、边界尽可能清晰的系统，支撑足够及时、可治理、可解释成本、可迁移的数据平台。

Snowflake-centric lakehouse 的价值，不是把所有数据问题都交给 Snowflake，而是在适合的场景下，把分析型数据平台的核心能力收敛起来，降低长期系统复杂度。

真正重要的不是“是否实时”，而是业务需要什么级别的新鲜度；不是“是否用了更多工具”，而是这些工具是否降低了整体复杂度；不是“是否上线新平台”，而是旧复杂度是否真的被移除。

这也是后续各章要继续展开的问题：总体架构如何设计，Landing 如何解耦接入，Medallion 如何管理语义演进，治理和隐私如何前置，FinOps 如何贯穿 TCO，operational serving 如何避免误用 Snowflake，以及迁移如何真正收口。
