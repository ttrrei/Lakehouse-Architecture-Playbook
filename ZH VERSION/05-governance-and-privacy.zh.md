# 05 · Governance 与 Privacy：治理不是上线后的补丁

## 1. 文档目的

这篇文档讨论 lakehouse 架构中的治理与隐私设计，重点是 **PII、访问控制、数据产品 ownership、审计、数据质量、Operational Serving 边界，以及为什么数据平台不应该直接写业务 OLTP 系统**。

它不是一份安全配置手册，也不会展开具体 IAM policy、Snowflake role DDL、network policy、masking policy 或审计 SQL。这里关注的是更高层的系统设计问题：

> 当数据平台变成企业的分析中心和部分 operational data activation 的来源时，如何让数据仍然可控、可解释、可审计、可演进？

我的基本观点是：

> 治理不是平台上线后的补丁。治理是数据平台架构的一部分。如果 PII、权限、审计、数据质量、ownership 和成本归因没有在设计阶段进入系统边界，后面往往会以更高复杂度的形式被迫补回来。

在低复杂度 lakehouse 架构中，治理的目标不是制造流程负担，而是让平台具备清晰的责任边界：

* 哪些数据可以进入平台；
* 哪些数据可以被谁访问；
* 哪些数据可以被 BI 使用；
* 哪些数据可以被应用点查；
* 哪些结果可以回到业务流程；
* 哪些访问需要被审计；
* 哪些数据产品有 owner、contract 和质量标准。

---

## 2. 治理为什么必须前置

很多数据平台在早期更关注 ingestion、modeling 和 dashboard delivery。治理通常被认为是后续阶段再补的事情。

这种方式短期看起来能加快交付，但长期会积累治理债务。

### 2.1 权限债务

如果平台上线初期为了方便调试和交付而大量授予宽权限，后续再收紧会非常困难。

常见问题包括：

* 分析师直接访问 raw layer；
* BI service account 访问过多 schema；
* 开发人员长期保留生产敏感数据权限；
* service account 权限边界不清；
* 临时权限变成永久权限；
* 下游报表依赖本不该公开的中间表。

权限一旦被消费层依赖，就不再只是安全问题，也会变成迁移和兼容性问题。

### 2.2 PII 暴露债务

如果 PII 在进入平台早期没有被识别和控制，后续就会扩散到多个层级：Landing、Bronze、Silver、Gold、BI extracts、notebooks、temporary tables、exports、serving stores。

一旦扩散，治理就不再是“限制一个字段”，而是要追踪所有副本、派生表、报表缓存、下载文件和外部同步。

因此，隐私控制越靠近 ingestion 边界，整体复杂度越低。

### 2.3 语义债务

如果没有明确的 data product ownership，模型会变成“谁写的谁知道”。

当指标被质疑、字段含义变化、数据质量失败、下游报表异常时，团队无法快速判断：

* 谁负责解释这个模型；
* 哪个定义是权威版本；
* 哪些消费者会受影响；
* 变更是否需要通知；
* 是否需要回填历史数据。

治理不仅是安全，也是语义管理。

### 2.4 审计债务

如果系统没有在设计阶段记录访问、变更、质量和发布记录，事后很难补齐。

审计需要回答的问题通常包括：

* 谁访问了敏感数据；
* 哪个 service account 发布了数据；
* 哪个模型生成了某个指标；
* 哪次 pipeline run 导致了变化；
* 哪个下游应用读取了某个 projection；
* 哪个版本的数据被业务使用。

这些问题不能只靠人工回忆。

---

## 3. 治理的基本分层

在 lakehouse 中，治理也应该分层，而不是用一个统一规则覆盖所有数据。

```text
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

不同层的治理重点不同。

### 3.1 Landing / Bronze 的治理重点

Landing 和 Bronze 关注的是源数据进入平台的边界。

重点包括：

* 数据是否允许进入 common path；
* 是否含有 PII 或敏感字段；
* source metadata 是否完整；
* schema version 是否记录；
* raw evidence 是否可回放；
* DLQ 是否可审计；
* replay 是否受控。

这一层的治理目标是确保平台没有在入口处扩大风险。

### 3.2 Silver 的治理重点

Silver 关注业务对象的稳定性。

重点包括：

* entity definition；
* business key；
* deduplication rule；
* SCD rule；
* quality baseline；
* PII classification；
* owner；
* schema compatibility。

这一层的治理目标是确保业务镜像可靠、可复用、可演进。

### 3.3 Gold 的治理重点

Gold 关注业务消费。

重点包括：

* metric definition；
* data product owner；
* freshness expectation；
* quality SLA；
* consumer contract；
* BI approval；
* access policy；
* change policy。

这一层的治理目标是确保业务用户消费的是可信模型。

### 3.4 Serving Projection 的治理重点

Operational serving projection 关注派生结果如何进入应用消费。

重点包括：

* serving contract；
* primary key；
* payload schema；
* freshness；
* compatibility；
* reader access；
* publisher ownership；
* audit log；
* rebuild path；
* reconciliation。

这一层的治理目标是避免 serving store 变成无人治理的新事实源。

---

## 4. PII 与敏感数据分类

PII 分类不应该只是合规文档里的列表。它应该影响 ingestion、modeling、access、serving 和 retention。

### 4.1 一个通用分类方式

可以将敏感数据分为几类：

| 类别                    | 含义               | 常见处理                         |
| --------------------- | ---------------- | ---------------------------- |
| Sensitive identifiers | 高敏身份标识或法律敏感字段    | 尽量不进入 common path；必要时隔离和加严访问 |
| Direct identifiers    | 可直接识别个人的字段       | hash、tokenize、mask 或受控访问     |
| Indirect identifiers  | 与其他字段结合可识别个人或实体  | RBAC、最小暴露、aggregation        |
| Behavioral attributes | 行为、活动、交易、使用记录    | 按敏感度和业务需要控制                  |
| Derived attributes    | 标签、评分、风险、分群、推荐结果 | 需要 owner、解释和使用边界             |

这只是通用分类。实际分类应根据行业、地区法规、企业政策和业务风险调整。

### 4.2 Ingest-time privacy control

隐私控制越靠近 ingestion 边界，风险扩散越小。

常见模式包括：

* 在进入 common landing 前过滤或处理敏感字段；
* 将高敏数据放入隔离 staging；
* 对直接标识符做 hash 或 tokenization；
* 对不需要分析的字段直接 drop；
* 对必须保留的敏感字段采用受控 schema；
* 在 metadata 中记录 privacy rule version。

关键原则是：

> 不应该让未经治理的敏感数据先进入所有人都能访问的 raw layer，再寄希望于后续模型不要误用。

### 4.3 Hash、Tokenize、Mask 的区别

这些处理方式不应该混用。

| 方式            | 适合场景             | 注意点              |
| ------------- | ---------------- | ---------------- |
| Hash          | 需要 join，但不需要反查明文 | 需要稳定算法和 salt 管理  |
| Tokenize      | 需要可控反查或授权恢复      | token map 本身非常敏感 |
| Mask          | 需要展示部分信息         | 不等于删除风险          |
| Redact / Drop | 不需要该字段           | 最简单也最安全          |
| Isolate       | 必须保留明文，但只给少数角色   | 需要强访问控制和审计       |

处理方式应由 use case 驱动，而不是默认全部保留。

### 4.4 Derived data 也可能敏感

很多团队只关注原始 PII，却忽视派生数据。

例如：

* 风险评分；
* 客户分群；
* 信用或欺诈标签；
* 行为预测；
* 收入或价值分层；
* 流失概率；
* 合规状态。

这些字段可能不是直接身份标识，但可能影响客户、运营或合规决策。因此它们也需要 owner、解释、访问控制和使用边界。

---

## 5. Access Control：不要只看表权限

访问控制不应该只是“谁能 SELECT 哪张表”。

在 lakehouse 中，访问控制至少涉及：

* human access；
* service account access；
* BI access；
* operational serving access；
* development vs production access；
* sensitive data access；
* temporary access；
* break-glass access。

### 5.1 最小权限原则

最小权限不是让所有事情都很难做，而是让每个角色只拥有完成职责所需的访问。

常见原则：

* 人类用户使用功能角色，不直接使用底层对象角色；
* service account 权限按用途隔离；
* BI service account 只读 approved Gold models；
* raw 和 sensitive layer 不默认开放；
* production write 权限严格控制；
* 临时权限有到期时间；
* 高敏访问需要审计。

### 5.2 按层控制访问

不同层的默认访问应该不同。

| 层                    | 默认访问原则                            |
| -------------------- | --------------------------------- |
| Landing              | 通常不直接给分析用户访问                      |
| Bronze               | 限制给 data engineering 和受控 debug 用途 |
| Silver               | 给数据团队和部分高级分析使用                    |
| Gold                 | BI、分析和业务消费的主要入口                   |
| Serving Projection   | 应用按 contract 读取                   |
| Sensitive / PII area | 默认拒绝，只给明确授权角色                     |

这种分层访问能避免 raw data 变成事实上的业务消费层。

### 5.3 Service account 要比人类用户更严格

很多风险来自 service account，而不是人类用户。

Service account 通常：

* 长期存在；
* 被自动化任务使用；
* 权限容易被放大；
* 发生错误时影响范围大；
* 容易被多个 pipeline 复用。

因此 service account 应该：

* 按用途拆分；
* 避免共享；
* 避免 broad access；
* 定期 review；
* 有明确 owner；
* 有审计记录。

---

## 6. Secure Projection：不要把所有表都包一层

在 Snowflake 或类似平台中，很多治理需求可以通过 view、secure view、masking、row filter 或 semantic layer 来实现。

但一个常见反模式是：把所有 Gold 表都包一层 view，以为这就是治理。

### 6.1 Projection 的合理用途

Projection 适合用于：

* 隐藏敏感字段；
* 暴露 approved column set；
* 提供 BI-friendly schema；
* 支持不同角色看到不同字段；
* 提供 audit-friendly access point；
* 支持特定 consumer contract。

### 6.2 过度 projection 的问题

如果所有表都经过多层 view 包装，问题会出现：

* lineage 变复杂；
* 性能排查变困难；
* 权限管理更难；
* schema change 更难追踪；
* 成本归因不清；
* 开发者不知道真正 source of truth 是哪张表。

治理的目标不是多加一层对象，而是让访问边界清楚。

### 6.3 建议原则

建议：

* 非敏感 Gold model 可以直接作为 approved consumption object；
* 对 PII、restricted fields、cross-domain audit 或特殊 consumer 使用 projection；
* projection 应有 owner、用途和消费者；
* 不要把 projection 当作弥补混乱模型的万能补丁。

---

## 7. Data Product Ownership

治理必须落到 owner 上。

没有 owner 的数据产品，本质上是不可信的。

### 7.1 Owner 负责什么

Data product owner 不一定自己写所有代码，但必须对以下内容负责：

* 业务定义；
* 数据质量标准；
* freshness expectation；
* PII classification；
* change approval；
* downstream communication；
* exception handling；
* deprecation decision。

### 7.2 Owner 不等于技术维护者

一个模型可能有：

* business owner；
* technical owner；
* platform owner；
* consumer owner。

这些角色要区分。

例如：

* business owner 决定指标含义；
* technical owner 维护模型实现；
* platform owner 负责运行环境；
* consumer owner 负责下游使用方式。

不区分 owner，会让问题在团队之间反复转移。

### 7.3 Owner 应该进入 metadata

Ownership 不应该只存在于文档或聊天记录中。

重要模型应该在 metadata 中记录：

* owner；
* domain；
* sensitivity；
* freshness；
* materialization；
* cost center 或 use case；
* downstream consumers；
* deprecation status。

这也为后续 FinOps、governance 和 migration 提供基础。

---

## 8. Data Quality Governance

数据质量不是纯技术问题。

它也是治理问题，因为质量失败会影响业务信任、运营决策和平台 credibility。

### 8.1 分层质量治理

不同层有不同质量目标：

| 层       | 质量重点                                                  |
| ------- | ----------------------------------------------------- |
| Landing | 文件完整性、到达时间、格式、source identity                         |
| Bronze  | 加载成功、技术元数据、raw payload、operation type                 |
| Silver  | entity correctness、dedupe、business key、state validity |
| Gold    | metric correctness、freshness、consumer constraints     |
| Serving | payload completeness、publish recency、reconciliation   |

### 8.2 测试失败需要响应机制

测试失败本身没有价值，响应才有价值。

关键测试应定义：

* severity；
* owner；
* response time；
* exception rule；
* downstream impact；
* escalation path。

否则，数据质量平台会变成红灯仪表盘，而不是治理机制。

### 8.3 质量和发布应该绑定

关键数据产品的发布或变更，应该绑定质量检查。

例如：

* primary key test 失败不能发布；
* PII exposure test 失败不能发布；
* freshness 失败需要标记数据不可用；
* reconciliation 失败需要通知 consumer；
* schema contract break 需要 consumer sign-off。

---

## 9. Auditability：平台必须能解释发生了什么

审计不只是满足监管或合规要求。它也是复杂系统的可解释能力。

一个好的数据平台应该能回答：

* 数据从哪里来；
* 什么时候进入平台；
* 经过了哪些模型；
* 谁访问了它；
* 哪个任务生成了它；
* 哪个版本被消费；
* 哪些权限被授予；
* 哪些异常被处理；
* 哪些数据被发布到 serving store。

### 9.1 Audit 的几类证据

常见审计证据包括：

* ingestion log；
* load history；
* model run history；
* data quality results；
* access history；
* permission change history；
* serving publish log；
* error and DLQ records；
* cost attribution records；
* lineage metadata。

### 9.2 审计不应只靠平台日志

平台日志很重要，但业务级审计也需要模型 metadata。

例如：

* 某个字段是否 PII；
* 某个模型 owner 是谁；
* 某个指标定义是什么；
* 某个 serving payload 面向哪个 consumer；
* 某个模型为什么被 deprecate。

这些不是单靠 query history 能回答的。

---

## 10. Zero OLTP Write：不是禁止激活，而是明确边界

数据平台经常会产生有价值的派生结果。

例如：

* customer tags；
* risk signals；
* recommendations；
* operational status；
* financial estimates；
* eligibility flags；
* lifecycle states。

这些结果可能需要进入业务流程。这个过程通常被称为 reverse ETL、data activation 或 operational serving。

### 10.1 为什么不建议数据平台直接写业务 OLTP

数据平台直接写业务 OLTP 会带来几个问题：

* ownership 混乱：业务系统记录到底由谁负责；
* blast radius 增大：一个数据 job 错误可能影响生产交易系统；
* 审计范围扩大：数据平台变成业务写入方；
* 回滚复杂：数据模型错误变成业务状态错误；
* 权限风险：分析平台 credential 拥有业务写权限；
* 团队边界模糊：应用团队和数据团队责任混合。

因此，我倾向于使用这样的原则：

> 数据平台可以发布 operational projection，但不直接修改业务 OLTP 的 transaction source of truth。

### 10.2 Reverse ETL 有多种模式

Zero OLTP Write 不代表数据平台的结果不能进入业务流程。

可选模式包括：

| 模式                       | 说明                     | 适用场景                       |
| ------------------------ | ---------------------- | -------------------------- |
| Serving store            | 数据平台发布 projection，应用读取 | key-based operational read |
| Application-owned API    | 数据平台调用业务系统提供的受控 API    | 业务系统必须拥有写入规则               |
| Message queue            | 数据平台发布事件，业务系统订阅        | 异步激活                       |
| Reverse ETL tool         | 专门工具同步到 SaaS 或应用系统     | CRM / marketing activation |
| Manual approval workflow | 数据结果经人工确认后进入业务         | 高风险或财务场景                   |

使用哪种模式取决于业务风险、延迟要求、ownership 和审计要求。

### 10.3 Serving projection 是一种边界清晰的模式

在很多 near real-time read 场景中，serving projection 是一个低复杂度选择。

模式是：

```text
Gold model
  -> publish projection
  -> serving store
  -> application reads by key
```

它的好处是：

* 应用不直接查 Snowflake；
* 数据平台不写业务 OLTP；
* serving store 可以从 Gold 重建；
* 权限边界清楚；
* 发布过程可审计；
* 应用团队和数据团队职责清楚。

但它也不是万能模式。复杂事务写入、强一致状态更新、用户交互写入和毫秒级决策，不应该简单用 serving projection 解决。

---

## 11. Governance Metadata

治理需要 metadata 支撑。

没有 metadata，治理只能靠人工沟通和个人记忆。

### 11.1 建议记录的 metadata

重要数据对象应记录：

* domain；
* owner；
* layer；
* purpose；
* grain；
* PII classification；
* freshness expectation；
* materialization；
* quality checks；
* cost attribution；
* downstream consumers；
* retention；
* change policy；
* deprecation status。

这些 metadata 不一定一开始全部自动化，但应该成为设计目标。

### 11.2 Metadata 应该靠近代码和模型

如果 metadata 只存在于独立文档，很容易过期。

更好的方式是让 metadata 尽量靠近：

* model definition；
* schema file；
* catalog；
* data contract；
* CI checks；
* lineage system。

这样治理可以和开发流程绑定，而不是上线后人工补录。

---

## 12. Retention 与 Deletion

数据保留和删除也是治理的一部分。

不是所有数据都应该永久保留，也不是所有数据都可以随意删除。

### 12.1 按层考虑 retention

不同层的 retention 可以不同：

| 层                   | Retention 思路                       |
| ------------------- | ---------------------------------- |
| Landing             | 根据 replay 和 archive 要求保留           |
| Bronze              | 保留 raw analytical evidence，用于重建和审计 |
| Silver              | 保留业务镜像和必要历史                        |
| Gold                | 根据消费和指标需求保留                        |
| Serving             | 通常只保留当前 projection 或短期快照           |
| Temporary / sandbox | 应有短 TTL                            |

### 12.2 删除不只是 drop table

删除请求可能涉及：

* raw data；
* derived models；
* BI extracts；
* serving projections；
* caches；
* exports；
* logs；
* archive policies。

因此删除能力必须和 lineage、metadata、PII classification 结合。

### 12.3 Retention 也是成本治理

长期保留所有中间结果会增加 TCO。

需要区分：

* 哪些是 record of origin；
* 哪些可以重建；
* 哪些必须长期保留；
* 哪些只是临时计算结果；
* 哪些 serving projection 可以从 Gold 重新发布。

---

## 13. 治理与低复杂度的关系

治理常被误解为复杂度来源。

确实，过度流程化的治理会拖慢交付。但缺失治理会带来更大的复杂度。

没有治理时：

* 权限会扩散；
* PII 会扩散；
* 指标会漂移；
* owner 会消失；
* 质量失败无人处理；
* 成本无法解释；
* 下游依赖无法追踪；
* 临时数据产品无法下线。

好的治理不是增加无意义审批，而是把平台边界设计清楚，让团队可以更快、更安全地交付。

---

## 14. 核心 trade-off

| 设计选择                        | 好处                            | 代价                            |
| --------------------------- | ----------------------------- | ----------------------------- |
| 治理前置                        | 降低隐私、权限和审计债务                  | 前期设计成本更高                      |
| Ingest-time privacy control | 降低敏感数据扩散                      | 接入复杂度增加                       |
| 分层访问控制                      | raw、business、consumption 边界清楚 | 需要角色和权限治理                     |
| 最小权限                        | 降低误用和泄漏风险                     | 需要持续 review                   |
| Data product ownership      | 问题有人负责                        | 需要组织配合                        |
| Projection 控制敏感暴露           | 可按 consumer 暴露数据              | 过度使用会增加 lineage 复杂度           |
| Zero OLTP Write             | 降低业务系统 blast radius           | 需要 reverse ETL / serving 模式设计 |
| Serving projection          | 应用读取边界清楚                      | 需要发布、监控和一致性管理                 |
| Metadata-driven governance  | 治理可自动化、可审计                    | 需要元数据维护纪律                     |
| Retention policy            | 降低风险和成本                       | 需要 lineage 和分类支持              |

---

## 15. 常见反模式

| 反模式                      | 问题                 | 更好的做法                                 |
| ------------------------ | ------------------ | ------------------------------------- |
| 上线后再补治理                  | 权限、PII、审计债务扩大      | 设计阶段前置治理                              |
| Raw layer 默认开放给所有分析用户    | 敏感数据和源系统细节暴露       | 按层控制访问                                |
| 所有 Gold 都包多层 view        | lineage 和性能排查复杂    | 只对明确场景使用 projection                   |
| Service account 共享和长期高权限 | 风险半径大              | 按用途拆分、最小权限、定期 review                  |
| PII 进入平台后再处理             | 敏感数据扩散             | ingest-time handling 或隔离 staging      |
| 只有技术 owner，没有业务 owner    | 指标定义无人负责           | 定义 business owner 和 technical owner   |
| 测试失败无人响应                 | 数据质量治理失效           | 每个关键测试有 owner 和 severity              |
| 数据平台直接写业务 OLTP           | ownership 混乱，风险半径大 | 使用 serving、API、queue 或 reverse ETL 工具 |
| Serving store 变成事实源      | 无法重建，责任不清          | 明确 projection 和 rebuild path          |
| Metadata 只在文档里           | 容易过期               | 让 metadata 靠近模型和代码                    |
| 临时数据集永远保留                | 成本和风险增加            | 使用 TTL 和 deprecation policy           |

---

## 16. 成功标准

一套设计良好的 governance 和 privacy 架构，应满足：

* PII 和敏感数据在 ingestion 边界被识别；
* common landing 不接收未经治理的高敏数据；
* Bronze、Silver、Gold、Serving 的访问边界清楚；
* BI 主要访问 approved Gold models；
* sensitive projection 有明确用途和 owner；
* service account 权限按用途隔离；
* 关键数据产品有 owner、contract、freshness 和 quality checks；
* 数据质量失败有响应机制；
* 访问、发布、权限变化和 serving publish 可审计；
* 数据平台不直接写业务 OLTP；
* reverse ETL 或 operational activation 有明确模式；
* retention 和 deletion 有策略；
* governance metadata 靠近代码和模型；
* 治理机制降低长期复杂度，而不是制造不必要流程。

---

## 17. 小结

Governance 和 Privacy 是 lakehouse 架构中的基础能力，不是合规团队上线前才检查的清单。

如果治理后置，平台会积累权限债务、PII 暴露债务、语义债务、审计债务和成本债务。后续再补救时，往往需要更复杂的权限、更多的 view、更混乱的例外流程和更高的运营成本。

更合理的设计是：

* 在 ingestion 边界处理敏感数据；
* 按 Bronze / Silver / Gold / Serving 分层控制访问；
* 让 Gold 成为主要业务消费入口；
* 用 owner、contract 和 metadata 管理数据产品；
* 对 operational serving 明确 projection 边界；
* 避免数据平台直接写业务 OLTP；
* 将 auditability、quality、retention 和 FinOps 作为平台能力设计。

低复杂度不是没有治理。低复杂度是把治理边界设计清楚，让平台可以长期安全、稳定、可解释地运行。
