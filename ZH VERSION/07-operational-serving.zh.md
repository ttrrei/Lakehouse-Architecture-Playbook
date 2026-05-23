# 07 · Operational Serving：不要把 Snowflake 当成应用 API

## 1. 文档目的

这篇文档讨论 lakehouse 架构中的 operational serving 设计，重点是 **为什么应用不应该直接查询 Snowflake 做高并发点查，为什么 serving store 应该被视为 projection 而不是 source of truth，以及数据平台如何以低复杂度方式支持 operational data activation**。

它不是一份 DynamoDB、DocumentDB、Redis、API Gateway 或 reverse ETL 工具的使用手册。这里关注的是更通用的系统设计问题：

> 当 Snowflake 或类似平台已经计算出有业务价值的派生结果时，如何把这些结果安全、稳定、可审计地提供给应用系统和业务流程，而不把分析平台变成 OLTP API，也不让数据平台直接写业务 OLTP？

我的基本观点是：

> Snowflake 适合作为分析型 source of truth，但不应该被设计成高并发应用点查后端。Operational serving 应该作为从 Gold 发布出来的可重建 projection，用于承接 key-based、低延迟、应用侧读取的场景。

这不是说所有 operational activation 都必须使用某个特定数据库。Document database、key-value store、search index、cache-backed service、application-owned API、message queue、reverse ETL 工具都可能合理。关键是要明确边界：

* Snowflake / Gold 负责派生结果和业务语义；
* Serving projection 负责应用读取形态；
* 业务 OLTP 系统继续拥有交易事实；
* 数据平台不直接污染业务 transaction source of truth。

---

## 2. 为什么需要 Operational Serving

在 lakehouse 架构中，Gold 层通常能产出很多有业务价值的数据：

* 客户状态；
* 用户标签；
* 风险信号；
* 推荐结果；
* 资格判断；
* 运营提示；
* 账户摘要；
* 财务预估；
* 产品使用状态；
* 生命周期阶段。

这些数据不只是给 BI 和分析师看。很多时候，应用系统、运营工具、客服系统、营销系统或风控系统也需要读取它们。

但这类读取不一定是分析型查询。

### 2.1 分析型消费

分析型消费通常包括：

* BI dashboards；
* management reporting；
* ad hoc SQL；
* historical analysis；
* metric exploration；
* reconciliation；
* finance / operations reporting。

这类查询通常是扫描、聚合、join、过滤、排序和多维分析。Snowflake 非常适合承载这类 workload。

### 2.2 Operational read

Operational read 更像应用点查。

典型问题是：

* 某个用户当前有什么标签？
* 某个账户是否符合某个规则？
* 某个业务对象当前状态是什么？
* 某个客户是否应该显示某个提示？
* 某个实体的最新派生结果是什么？

这类请求通常具有以下特征：

* key-based；
* 单条或小批量读取；
* 高并发；
* 低延迟；
* 响应形态稳定；
* 多由应用后端触发；
* 访问模式重复。

这种 workload 不适合直接打到 Snowflake Gold。

---

## 3. 为什么应用不应该直接查询 Snowflake

应用直接查询 Snowflake 看起来很简单：数据已经在 Gold 中，应用只要按 key 查一下即可。

但从系统设计角度看，这通常是一个危险边界。

### 3.1 Workload 模式不匹配

Snowflake 是分析型平台，擅长大规模扫描、聚合、join 和复杂 SQL。

应用点查则更关注：

* 毫秒到低百毫秒级响应；
* 高并发稳定性；
* predictable latency；
* fixed access pattern；
* key-value 或 bounded query；
* API-level availability。

这两类 workload 的优化目标不同。

把 operational point lookup 放进 Snowflake，容易让 analytical workload 和 application serving workload 互相影响。

### 3.2 成本和延迟不可预测

应用请求通常具有大量小查询、高频访问和用户行为驱动的特点。

如果这些请求直接打到 Snowflake，可能出现：

* warehouse 被大量小查询唤醒；
* query queue 增加；
* BI 和 transformation workload 被影响；
* result cache 命中不稳定；
* cost attribution 变复杂；
* latency 不符合应用 SLA。

这不一定是 Snowflake 的问题，而是 workload 放错了位置。

### 3.3 责任边界不清

如果应用直接依赖 Snowflake Gold，很多责任会变模糊：

* 应用 API SLA 由谁负责？
* Snowflake warehouse 可用性由谁保障？
* Gold schema 变化如何通知应用？
* 应用 retry 是否会放大查询压力？
* 数据模型 owner 是否要承担应用线上故障？
* 应用团队是否需要理解 Snowflake 权限和查询性能？

这些问题会让分析平台和应用平台的 ownership 混在一起。

### 3.4 安全和权限面扩大

应用直接查 Snowflake，通常意味着应用 service account 需要访问分析平台对象。

如果边界控制不好，可能带来：

* service account 权限过大；
* 应用访问了本不该访问的字段；
* BI 和 application 权限模型混用；
* 数据导出边界不清；
* 审计范围扩大；
* 凭证泄露时影响分析平台。

Operational serving projection 可以把应用读取面限制在更窄、更明确的数据结构上。

---

## 4. Operational Serving Projection 的基本模式

Operational serving projection 的核心模式是：

```text
Gold model
  -> incremental refresh / change detection
  -> publisher
  -> serving store
  -> application point lookup
```

这个模式中，Gold 仍然是分析型 source of truth。Serving store 只是为了应用读取而发布出来的 projection。

### 4.1 Projection 的含义

Projection 不是新的事实源。

它具有几个特点：

* 从 Gold 或 approved source model 生成；
* 可以被重新发布；
* 可以被清空后重建；
* 有明确 key；
* 有明确 payload schema；
* 有明确 freshness；
* 有明确 consumer；
* 有发布日志和错误处理。

如果 serving store 中的数据无法从上游重建，它就不再是 projection，而变成了新的事实源。这会显著增加治理和恢复复杂度。

### 4.2 典型数据流

一个典型 serving 流程可以是：

```text
Silver business mirror
  -> Gold serving source model
  -> detect changed keys
  -> publish changed payloads
  -> serving store
  -> application backend reads by key
```

这里的关键是 **changed keys**。

如果一个表只需要更新少量变化对象，不应该每次全量重写 serving store。全量刷新虽然简单，但在 near real-time 场景下可能带来不必要的 compute、write capacity、延迟和失败风险。

### 4.3 Serving store 的选择

Serving store 可以有多种形式：

| 类型                    | 适合场景                              | 注意点                      |
| --------------------- | --------------------------------- | ------------------------ |
| Key-value store       | 高并发按 key 点查                       | 不适合复杂过滤和 join            |
| Document database     | JSON payload、对象状态、灵活字段            | 需要治理 schema 演进           |
| Search index          | 搜索、过滤、文本查询                        | 不是事实源，需要重建能力             |
| Cache-backed service  | 低延迟读取、短 TTL                       | 需要 cache invalidation 策略 |
| Application-owned API | 业务系统控制访问和写入规则                     | 数据平台不应绕过业务系统 ownership   |
| Message queue         | 异步激活和事件通知                         | 消费方需要幂等和重试               |
| Reverse ETL tool      | SaaS / CRM / marketing activation | 要明确写入规则和审计边界             |

这里没有唯一正确答案。选择取决于 access pattern、SLA、数据结构、consistency、团队能力和治理要求。

---

## 5. Serving Contract

Operational serving 不能只靠口头约定。每一个 serving projection 都应该有 contract。

### 5.1 Contract 应包含什么

一个 serving contract 至少应该定义：

* business purpose；
* source Gold model；
* target serving store；
* owner；
* consumer team；
* primary key；
* optional sort key or secondary index；
* payload schema；
* field meaning；
* nullable rule；
* freshness target；
* availability expectation；
* compatibility rule；
* security classification；
* publish pattern；
* reconciliation requirement；
* failure handling；
* deprecation policy。

没有 contract 的 serving projection，很容易演变成一个隐形 API。

### 5.2 Key 和 grain 必须明确

Operational serving 最重要的是 key 和 grain。

必须明确：

* 一条记录代表什么？
* 是一个用户当前状态？
* 一个账户当前快照？
* 一个订单最新摘要？
* 一个业务实体的一组标签？
* 一个时间窗口内的聚合？

如果 grain 不清楚，应用很容易误用 projection。

### 5.3 Payload 应该面向消费场景

Serving payload 不应该简单复制 Gold 表。

它应该根据应用读取场景设计：

* 字段尽量少；
* 结构稳定；
* 避免应用再做复杂 join；
* 包含 last_updated timestamp；
* 包含 snapshot version；
* 包含必要状态和解释字段；
* 不包含不必要 PII。

Gold model 可以很丰富，但 serving payload 应该克制。

---

## 6. Ownership Boundary

Operational serving 成败很大程度取决于 ownership 是否清楚。

### 6.1 数据团队负责什么

数据团队通常负责：

* Gold source model；
* serving source model；
* change detection；
* publisher；
* publish audit log；
* publish error handling；
* data quality；
* freshness monitoring；
* reconciliation；
* serving contract 中的数据定义。

### 6.2 应用团队负责什么

应用团队通常负责：

* API layer；
* UI behavior；
* cache strategy；
* reader access pattern；
* application retry；
* fallback behavior；
* user-facing SLA；
* how the data is presented to users。

### 6.3 平台 / 安全团队负责什么

平台或安全团队通常负责：

* serving store provisioning；
* IAM / access boundary；
* encryption；
* network access；
* logging；
* infrastructure monitoring；
* backup / restore capability when required。

### 6.4 Business owner 负责什么

业务 owner 负责：

* 字段含义；
* 业务规则；
* 是否可展示给用户；
* freshness 是否足够；
* 异常时如何解释；
* 高风险字段是否需要人工确认。

如果这些 owner 不清楚，serving projection 会成为跨团队争议点。

---

## 7. Consistency 与 Freshness

Operational serving 通常不是强一致系统。

它更多是 eventual consistency：源系统变化后，经过数据平台处理，再发布到 serving store。

### 7.1 Freshness 应该端到端定义

不能只说 “publisher 每 5 分钟跑一次”。

应该定义：

```text
source change
  -> Bronze arrival
  -> Silver update
  -> Gold refresh
  -> projection publish
  -> application read
```

每一段都有可能产生延迟。

### 7.2 Last updated 和 snapshot version

Serving payload 通常应该包含：

* last_updated_at；
* source_event_time；
* snapshot_version；
* model_version；
* publish_time；
* optional freshness status。

这让应用和业务用户可以理解：当前读到的结果是什么时间点的结果。

### 7.3 Eventual consistency 需要业务接受

如果某个业务场景不能接受延迟或短暂不一致，就不应该简单使用 serving projection。

例如：

* 交易状态写入；
* 账户余额强一致更新；
* 实时授权；
* 毫秒级风控；
* 用户交互事务。

这些场景应该由业务 OLTP 或专门实时系统负责。

---

## 8. Reconciliation 与可解释性

如果 serving projection 会影响客户体验、财务判断、风险提示或运营动作，就需要 reconciliation。

### 8.1 哪些场景需要 reconciliation

通常包括：

* 财务相关数值；
* 客户可见金额；
* 风险标签；
* 资格判断；
* 营销分群；
* 自动化运营动作；
* 合规状态；
* 与业务系统权威状态有关的派生结果。

### 8.2 Reconciliation 可以比较什么

可以比较：

* Gold source model row count vs serving store item count；
* changed keys count vs published keys count；
* published payload hash；
* source timestamp vs serving timestamp；
* aggregate totals；
* authoritative business system result；
* consumer read result sample。

### 8.3 可解释性很重要

Serving projection 中的派生字段应该能解释：

* 这个值来自哪个模型；
* 什么时候计算；
* 使用哪个版本规则；
* 是否是估算值；
* 是否经过人工确认；
* 是否可以用于自动化决策。

如果应用展示的是估算值或建议值，必须避免让用户以为它是最终权威事实。

---

## 9. Error Handling 与 Recovery

Operational serving 的失败模式和普通 BI 不同。

BI 报表失败通常是用户看不到数据。Serving projection 失败可能影响应用功能、用户体验或业务流程。

### 9.1 常见失败模式

| 失败类型                  | 示例                       |
| --------------------- | ------------------------ |
| Publisher failure     | 发布任务失败、权限错误、网络错误         |
| Partial publish       | 一部分 key 成功，一部分失败         |
| Stale projection      | Gold 已更新，但 serving 未更新   |
| Schema mismatch       | 应用期待字段与 payload 不一致      |
| High write volume     | 突然大量变化导致写入压力             |
| Reader error          | 应用读取失败或权限错误              |
| Reconciliation breach | serving store 与 Gold 不一致 |
| Bad data publish      | 错误业务逻辑被发布到应用             |

### 9.2 发布必须可幂等

Publisher 应该尽量幂等。

同一条记录重复发布，不应该产生副作用。

常见方式包括：

* 按 primary key 覆盖；
* 使用 snapshot_version；
* 使用 payload hash；
* 使用 conditional update；
* 记录 publish log；
* 支持 failed keys retry。

### 9.3 Recovery 路径

Recovery 方式包括：

* retry failed keys；
* re-publish changed keys；
* full rebuild serving store；
* rollback to previous snapshot；
* disable consumer feature flag；
* fall back to last known good snapshot；
* mark projection stale。

如果 serving store 无法从 Gold 重建，recovery 会变得非常复杂。

---

## 10. Security 与 Privacy

Operational serving 经常比 BI 更接近用户体验和业务流程，因此安全边界必须清楚。

### 10.1 最小 payload

Serving payload 应只包含应用需要的字段。

不要因为 Gold 中有很多字段，就全部发布到 serving store。

尤其要避免：

* 不必要 PII；
* 原始敏感字段；
* 内部调试字段；
* 模型中间特征；
* 不应被应用展示的解释字段。

### 10.2 Reader access 和 writer access 分离

建议明确分离：

* publisher write access；
* application read access；
* admin / debug access；
* platform maintenance access。

应用团队通常不应该写 serving store。数据团队也不应该通过 serving store 绕过业务系统写入规则。

### 10.3 Derived data 也可能敏感

Serving store 中经常包含标签、评分、状态、风险、资格判断等 derived data。

这些字段可能不是原始 PII，但仍然可能敏感。它们可能影响用户体验、运营决策或合规判断。

因此 derived data 也需要 classification、owner 和 access policy。

---

## 11. Serving Store 的选择标准

选择 serving store 时，不应该只问“哪个数据库最快”。

应该看 access pattern。

### 11.1 关键问题

选择前应回答：

* 是单 key lookup 还是 bounded query？
* 是否需要 range query？
* 是否需要 search？
* 是否需要复杂 filter？
* 是否需要 join？
* 是否需要事务写入？
* QPS 大概是多少？
* latency target 是什么？
* payload 是固定 schema 还是灵活 JSON？
* 数据是否可以从 Gold 重建？
* 是否有 PII？
* consumer 是一个应用还是多个应用？
* 是否需要跨区域读取？

### 11.2 不同 serving 形态

| 形态                    | 优点                   | 限制                    |
| --------------------- | -------------------- | --------------------- |
| Key-value store       | 简单、高并发、低延迟           | 查询模式受限                |
| Document database     | 适合对象状态和 JSON payload | schema governance 更重要 |
| Relational serving DB | SQL 能力强，适合复杂查询       | 运维和连接管理更重             |
| Search index          | 搜索和过滤强               | 不应作为事实源               |
| Cache                 | 极低延迟                 | 失效和一致性复杂              |
| Application API       | 业务 ownership 清楚      | 数据平台依赖应用接口能力          |
| Reverse ETL tool      | 适合 SaaS activation   | 写入规则和审计要特别清楚          |

### 11.3 不要为了工具而服务

如果应用只是需要一个用户状态点查，不需要复杂 SQL。

如果业务需要复杂过滤和 join，也不应该强行用 key-value store。

如果结果需要写回业务事务状态，也不应该通过 serving projection 绕过业务系统。

Serving store 应该服务 access pattern，而不是相反。

---

## 12. 与 Reverse ETL 的关系

Operational serving 和 reverse ETL 相关，但不是完全相同。

### 12.1 Reverse ETL 的常见含义

Reverse ETL 通常指把数据仓库或 lakehouse 中的结果同步回业务工具或 SaaS 系统，例如 CRM、marketing platform、support system、sales tool。

这类模式常见于：

* 客户分群同步；
* 营销受众；
* 销售线索评分；
* 客服提示；
* 产品使用标签。

### 12.2 Operational serving 的不同点

Operational serving 更强调应用读取路径：

* 数据平台发布 projection；
* 应用通过 key lookup 读取；
* 数据通常不直接写入业务 OLTP；
* projection 可重建；
* latency 和 availability 更接近应用要求。

### 12.3 选择哪种模式

| 场景               | 更适合                      |
| ---------------- | ------------------------ |
| SaaS 工具需要用户分群    | Reverse ETL tool         |
| 应用后端需要按 key 读取状态 | Serving projection       |
| 业务系统必须更新交易状态     | Application-owned API    |
| 异步通知业务流程         | Message queue            |
| 高风险决策需要人工确认      | Manual approval workflow |

不要把所有 data activation 都理解成同一种模式。

---

## 13. Operational Serving 的成本

Serving projection 可以保护 Snowflake workload，但不是免费。

成本包括：

* Gold source model refresh；
* change detection；
* publisher compute；
* serving store storage；
* write capacity；
* read capacity；
* monitoring；
* reconciliation；
* retry；
* schema compatibility；
* incident response。

因此，不应该把所有 Gold 模型都发布成 serving projection。

只有当业务确实需要低延迟、key-based operational read 时，才应该引入 serving projection。

---

## 14. 核心 trade-off

| 设计选择                          | 好处                       | 代价                        |
| ----------------------------- | ------------------------ | ------------------------- |
| 应用不直接查询 Snowflake             | 保护分析 workload，降低应用延迟不确定性 | 需要额外 serving path         |
| 使用 serving projection         | 应用读取边界清楚，可从 Gold 重建      | 多一层存储、发布和监控               |
| 使用 key-value / document store | 低延迟点查，适合稳定 payload       | 不适合复杂 join 和 ad hoc 查询    |
| 使用 application-owned API      | 业务系统 ownership 清楚        | 数据平台依赖应用接口和写入规则           |
| 使用 reverse ETL 工具             | 适合 SaaS activation       | 写入边界和审计要明确                |
| Delta publish                 | 成本低，适合 near real-time    | 需要 change detection 和幂等处理 |
| Full rebuild                  | 简单、可恢复                   | 大数据量时成本和延迟高               |
| Eventual consistency          | 系统复杂度低                   | 不适合强一致事务场景                |
| Reconciliation                | 提升可信度                    | 增加监控和处理成本                 |
| 最小 payload                    | 降低隐私和耦合风险                | 应用可能需要额外版本迭代              |

---

## 15. 常见反模式

| 反模式                               | 问题                          | 更好的做法                                  |
| --------------------------------- | --------------------------- | -------------------------------------- |
| 应用直接查 Snowflake 做高并发点查            | 成本、延迟和 workload 边界不可控       | 使用 operational serving projection      |
| 把 serving store 当 source of truth | 无法重建，责任边界混乱                 | 保持 projection 可重建                      |
| 数据团队直接写业务 OLTP                    | blast radius 大，ownership 混乱 | 使用 serving、API、queue 或 reverse ETL 模式  |
| 所有 Gold 都发布到 serving store        | 成本和治理复杂度上升                  | 只发布明确 key-based use case               |
| Serving payload 过大                | 隐私风险和应用耦合增加                 | 最小必要字段                                 |
| 没有 serving contract               | schema、freshness、owner 不清   | 每个 projection 都定义 contract             |
| Publisher 不幂等                     | 重试可能产生副作用                   | 使用 key overwrite、version、hash 等机制      |
| 没有 reconciliation                 | 错误可能长期静默存在                  | 对关键业务字段做对账                             |
| 应用和数据团队 ownership 混乱              | 故障时责任不清                     | 明确 publisher、reader、business owner     |
| 用 key-value store 做复杂分析           | 查询模式不匹配                     | 复杂分析回到 Snowflake / analytical platform |

---

## 16. 成功标准

一套设计良好的 operational serving 架构，应满足：

* 应用点查不直接访问 Snowflake 分析模型；
* 每个 serving projection 都有明确 business purpose；
* Gold source model 是 projection 的上游来源；
* serving store 可以从 Gold 重建；
* key、grain 和 payload schema 明确；
* freshness 和 consistency expectation 明确；
* publisher 幂等；
* publish log 和 error log 可审计；
* reader access 和 writer access 分离；
* PII 和 derived sensitive data 被分类和控制；
* 关键业务值有 reconciliation；
* application owner、data owner、business owner 边界清楚；
* serving 成本被纳入 FinOps；
* projection 有 deprecation policy；
* 不适合 serving 的场景被明确排除。

---

## 17. 小结

Operational serving 的核心问题不是“用哪个数据库”，而是如何在分析平台和应用系统之间建立清晰边界。

Snowflake 或类似 lakehouse 平台非常适合计算和治理派生结果，但不应该被高并发应用点查直接访问，也不应该承担业务 OLTP 写入责任。

更稳健的模式是：

* Gold 产出稳定、可治理的业务结果；
* publisher 将必要结果发布为 serving projection；
* 应用按 key 读取 projection；
* serving store 可重建、可审计、可监控；
* 业务 OLTP 继续拥有交易事实；
* reverse ETL、API、message queue、serving store 根据场景选择。

这样既能让数据平台的价值进入业务流程，又不会让数据平台变成另一个隐形应用后端。
