# 08 · Migration Playbook：不要 Big Bang，也不要永久双运行

## 1. 文档目的

这篇文档讨论如何从 legacy 数据平台迁移到新的 lakehouse 架构，重点是 **如何降低迁移风险，如何建立 dual-run、parity、cutover、shadow 和 decommission 机制，以及如何避免新旧平台长期并存造成永久复杂度**。

它不是一份项目计划模板，也不会定义具体月份、预算、团队编制或内部审批流程。这里关注的是更通用的系统设计和迁移治理问题：

> 当一个组织决定建设新的 Snowflake-centric 或其他 lakehouse 架构时，如何让迁移真正降低复杂度，而不是在旧平台旁边再增加一个新平台？

我的基本观点是：

> 数据平台迁移的核心目标不是“新平台上线”，而是“旧复杂度被安全移除”。如果没有明确的 cutover 和 decommission 机制，新平台上线之后，组织很可能只是获得了第二套平台。

因此，成熟的数据平台迁移不应该追求 Big Bang，也不应该接受无限期 dual-run。更稳健的方式是：

* 先建立清晰的 legacy inventory；
* 按业务价值和风险排序；
* 对关键链路进行 dual-run；
* 用 parity 和业务签核建立信任；
* 有计划地 cutover；
* 保留有限 shadow period；
* 最终 decommission 旧链路。

---

## 2. 为什么 Big Bang 风险高

Big Bang 迁移看起来干净：一次性切换所有 pipeline、模型、报表和消费者。

但在数据平台场景中，Big Bang 通常风险很高。

### 2.1 数据平台依赖复杂

数据平台不是一个单一应用。它通常连接：

* 源系统；
* ingestion pipeline；
* transformation jobs；
* BI dashboards；
* ad hoc users；
* data exports；
* downstream applications；
* finance reports；
* operational workflows；
* compliance or audit processes。

很多依赖并不总是有文档。某个旧表、旧报表或旧导出可能仍然支撑关键业务流程。

如果一次性切换，隐藏依赖会在 cutover 后暴露。

### 2.2 数据口径需要信任建立

新平台即使技术上正确，也需要业务信任。

业务方通常不会因为“新架构更现代”就立即相信新报表。他们需要看到：

* 新旧结果是否一致；
* 差异是否可以解释；
* 新指标口径是否被确认；
* 数据 freshness 是否满足需求；
* 关键业务周期是否经过验证；
* 异常时是否有 rollback。

这些都需要 dual-run 和 parity，而不是一次性切换。

### 2.3 回滚路径复杂

数据平台 cutover 不只是切换一个 endpoint。

它可能涉及：

* BI dataset；
* scheduled reports；
* downstream exports；
* application integration；
* user bookmarks；
* metric definitions；
* permission model；
* historical backfill。

如果没有分阶段迁移，回滚会变得非常困难。

### 2.4 Big Bang 容易掩盖真实成本

一次性迁移往往低估：

* parity testing；
* business validation；
* user communication；
* dashboard rebuild；
* edge cases；
* historical mismatch；
* legacy shutdown；
* training and support。

真正困难的部分不是新平台建设，而是让消费者安全迁移，并且让旧平台退出。

---

## 3. 迁移的核心原则

### 3.1 业务优先，而不是技术优先

迁移优先级不应该只由技术团队决定。

更合理的排序应综合：

* 业务价值；
* 使用频率；
* 风险等级；
* 当前痛点；
* 成本影响；
* 数据质量问题；
* 平台复杂度；
* 下游依赖数量。

技术上容易迁移的对象，不一定最应该先迁。

### 3.2 不追求一次性完成

数据平台迁移应采用渐进式方式。

建议把迁移看成多个小 cutover，而不是一个大 cutover。

每一组 pipeline、模型、报表或 consumer 都应该经过：

```text
Discover
  -> Rebuild
  -> Dual-run
  -> Validate
  -> Cutover
  -> Shadow
  -> Decommission
```

### 3.3 Dual-run 必须有退出条件

Dual-run 是为了建立信任和降低风险，不是为了永久保留两套系统。

每个 dual-run 对象都应该有明确退出条件：

* parity 达标；
* 差异已解释；
* consumer 已签核；
* rollback path 明确；
* shadow period 结束；
* legacy dependency 清零；
* decommission 已执行。

没有退出条件的 dual-run，会变成永久复杂度。

### 3.4 迁移必须同时管理 TCO

迁移期间成本通常会上升，因为新旧平台同时运行。

这不一定是坏事，但必须被标记和解释。

Migration / backfill / dual-run workload 应该和 steady-state workload 区分，否则管理层会误判新平台成本。

更重要的是，只有旧链路被下线，TCO 才可能下降。

---

## 4. Legacy Inventory：先知道自己有什么

迁移第一步不是建新模型，而是建立 legacy inventory。

没有 inventory，就无法判断迁移范围、优先级、风险和退出路径。

### 4.1 Inventory 应包含什么

建议至少记录：

* pipeline name；
* source system；
* target table / file / report；
* schedule；
* owner；
* consumer；
* business purpose；
* data freshness；
* downstream dependencies；
* cost or runtime；
* failure history；
* data sensitivity；
* migration status；
* decommission candidate；
* known issues。

### 4.2 不要只盘点 pipeline

很多团队只盘点 ETL job，但真正的依赖可能在其他地方。

还需要盘点：

* BI dashboards；
* semantic models；
* scheduled exports；
* spreadsheets；
* API consumers；
* manually consumed tables；
* ad hoc analyst workflows；
* reconciliation processes；
* operational reports；
* compliance or audit extracts。

如果只迁移 pipeline，不迁移消费关系，cutover 会失败。

### 4.3 给 legacy 对象分类

可以把 legacy 对象分为几类：

| 类型                   | 处理方式                   |
| -------------------- | ---------------------- |
| High-value active    | 优先迁移                   |
| High-risk / critical | 谨慎迁移，强 parity          |
| Low-use but required | 保留到明确替代路径              |
| Unknown owner        | 进入 owner discovery     |
| No recent usage      | decommission candidate |
| Duplicated logic     | 合并或替换                  |
| Temporary workaround | 迁移时消除                  |
| Compliance-related   | 单独审查                   |

分类的目标不是整理文档，而是为迁移决策服务。

---

## 5. Migration Prioritization：如何决定先迁什么

迁移顺序应基于价值、风险和复杂度。

### 5.1 优先迁移的对象

通常适合优先迁移：

* 使用频率高；
* 当前质量问题明显；
* 维护成本高；
* legacy 成本高；
* 下游业务价值高；
* 可以证明新平台价值；
* 数据源和逻辑相对清楚；
* consumer 愿意参与验证。

### 5.2 不适合最早迁移的对象

不适合第一批迁移：

* owner 不清；
* 业务定义有争议；
* 下游依赖过多但不透明；
* 合规要求特殊；
* 历史口径复杂；
* 缺少可验证样本；
* 当前系统虽然旧但稳定且低成本。

这些对象可能仍然需要迁移，但不适合作为验证新平台的第一批。

### 5.3 使用 value / risk / effort 矩阵

可以用三维判断：

| 维度     | 问题                    |
| ------ | --------------------- |
| Value  | 迁移后是否能明显提升业务价值、质量或成本？ |
| Risk   | 迁移失败是否影响关键业务、财务或合规？   |
| Effort | 源数据、逻辑、依赖和验证是否复杂？     |

第一批最好选择高价值、中低风险、工作量可控的对象。

---

## 6. Rebuild：在新架构中重建，而不是机械搬迁

迁移不是把旧 pipeline 原样搬到新平台。

如果只是把旧逻辑复制到 Snowflake，旧复杂度会被带到新平台。

### 6.1 先判断旧逻辑是否仍然合理

重建前要问：

* 这个数据产品是否仍然被使用？
* 指标定义是否仍然正确？
* 源系统是否仍然是权威来源？
* 旧 pipeline 是否包含历史 workaround？
* 是否有重复模型可以合并？
* 是否可以用新的 Medallion 分层重构？
* 是否需要 near real-time，还是 batch 足够？

### 6.2 按新架构分层重建

新平台应尽量按标准链路重建：

```text
Source
  -> Landing
  -> Bronze
  -> Silver business mirror
  -> Gold consumption model
  -> BI / serving / export
```

不要把旧平台的所有中间表一比一复制。

迁移是清理语义、简化模型和减少技术债的机会。

### 6.3 记录口径差异

重建过程中，很可能发现新旧逻辑不完全一致。

差异可能来自：

* 旧逻辑 bug；
* 新逻辑修正；
* 源系统变化；
* 时间区间处理不同；
* null/default 处理不同；
* 去重规则不同；
* late data 处理不同；
* 业务定义升级。

这些差异必须记录和解释，而不是简单追求数字完全一致。

---

## 7. Dual-Run：迁移信任的核心机制

Dual-run 指新旧链路在一段时间内同时运行。

它的目的不是长期保留两套系统，而是建立信任、发现差异、验证稳定性。

### 7.1 Dual-run 应验证什么

应验证：

* row count；
* key coverage；
* metric totals；
* dimensions；
* freshness；
* late data handling；
* null / default handling；
* duplicate handling；
* downstream BI behavior；
* performance；
* cost；
* access control。

### 7.2 Dual-run 时间不应无限延长

Dual-run 应该有明确窗口和退出标准。

可以按业务周期定义，例如：

* 一个完整日报周期；
* 一个周报周期；
* 一个财务月结周期；
* 一个业务高峰周期；
* 一个特定 campaign 或运营周期。

关键不是固定天数，而是覆盖足够的业务场景。

### 7.3 Dual-run 应标记成本

Dual-run 期间成本上升是正常现象。

但这些成本应该被标记为 migration / validation cost，而不是误认为新平台 steady-state cost。

否则管理层可能会误判新架构经济性。

---

## 8. Parity Check：一致性不是简单数字相等

Parity check 是 dual-run 的核心。

但 parity 不应该被理解为所有数字完全一致。

### 8.1 Parity 的层次

可以分层做 parity：

| 层级                | 检查内容             |
| ----------------- | ---------------- |
| Source coverage   | 新旧是否覆盖同一源数据范围    |
| Row count         | 记录数量是否一致或差异可解释   |
| Key coverage      | 主键集合是否一致         |
| Metric parity     | 核心指标是否一致         |
| Distribution      | 分布、分组、top N 是否一致 |
| Freshness         | 新旧延迟是否符合预期       |
| Business scenario | 关键业务用例是否一致       |
| Consumer behavior | 报表和下游应用表现是否一致    |

### 8.2 差异需要分类

差异不一定代表新平台错误。

差异可以分为：

| 差异类型                 | 含义       | 处理                            |
| -------------------- | -------- | ----------------------------- |
| Expected difference  | 新旧口径已知不同 | 记录并获得业务确认                     |
| Legacy bug fixed     | 新平台修复旧问题 | 记录并解释                         |
| New platform bug     | 新逻辑错误    | 修复后重新验证                       |
| Timing difference    | 刷新时间不同   | 按 freshness window 比较         |
| Source coverage gap  | 数据范围不一致  | 修复 ingestion 或 source mapping |
| Definition ambiguity | 业务定义不清   | 业务 owner 决策                   |

### 8.3 Parity 应自动化，但需要业务解释

技术可以自动比较 row count、hash、metric totals、key coverage。

但业务差异需要业务 owner 解释。

尤其是指标定义变化时，不应该由工程团队单独决定差异是否可接受。

---

## 9. Cutover：切换不是一个按钮

Cutover 是将消费者从旧路径切到新路径。

它不是简单“改一个连接字符串”。

### 9.1 Cutover 前置条件

建议 cutover 前确认：

* 新模型已稳定运行；
* parity 达到约定标准；
* 差异已解释；
* consumer 已验证；
* 权限已配置；
* BI 或应用已更新；
* rollback path 明确；
* monitoring 已上线；
* owner 已确认；
* decommission plan 已准备。

### 9.2 Cutover 类型

Cutover 可以分几类：

| 类型                    | 说明                                   |
| --------------------- | ------------------------------------ |
| BI cutover            | dashboard 或 semantic model 改读新 Gold  |
| Export cutover        | downstream file 或 table export 改用新模型 |
| API / serving cutover | 应用改读 serving projection              |
| User workflow cutover | 分析师或业务用户改用新数据产品                      |
| Pipeline cutover      | 下游 pipeline 改用新表或新 contract          |

不同 cutover 需要不同验证方式。

### 9.3 Cutover 需要沟通

即使技术上无缝切换，也需要沟通：

* 切换范围；
* 切换时间；
* 预期变化；
* 已知差异；
* 回滚方式；
* 联系人；
* 旧路径下线时间。

没有沟通的 cutover 会降低业务信任。

---

## 10. Shadow Period：有限保留，不是无限双运行

Cutover 后，可以保留一段 shadow period。

Shadow period 的目的，是在消费者已经使用新路径的情况下，旧路径暂时保留用于对比和回滚。

### 10.1 Shadow period 应做什么

应关注：

* 新路径稳定性；
* 业务反馈；
* 数据差异；
* performance；
* cost；
* hidden dependencies；
* rollback readiness。

### 10.2 Shadow period 必须有结束条件

Shadow 不应该变成永久 dual-run。

结束条件可以包括：

* 无重大数据问题；
* consumer 无异议；
* hidden dependency 清理；
* rollback 不再需要；
* decommission 已批准；
* 旧路径访问量降为零。

### 10.3 Shadow 期间禁止新增依赖

Shadow 期间旧路径应该被冻结。

不应允许新的报表、导出、应用或用户依赖旧路径。

否则旧平台会重新获得生命，decommission 会失败。

---

## 11. Decommission：迁移真正完成的标志

Decommission 是迁移最容易被忽略的部分。

但从 TCO 和复杂度角度看，它是最重要的一步。

### 11.1 Decommission 应包含什么

下线旧链路可能包括：

* 停止旧 pipeline；
* 删除旧调度；
* 移除旧 BI 数据源；
* 归档旧表或文件；
* 删除过期权限；
* 清理 service account；
* 停止旧监控；
* 更新文档；
* 通知消费者；
* 标记旧对象 deprecated；
* 释放基础设施成本。

### 11.2 下线前要确认依赖

下线前必须确认：

* 没有活跃消费者；
* 没有 scheduled report；
* 没有 downstream export；
* 没有应用读取；
* 没有审计或合规保留要求；
* 数据已按策略归档；
* rollback 窗口已结束；
* owner 已批准。

### 11.3 没有 decommission，就没有复杂度下降

新平台上线只是迁移的一半。

旧平台下线，才是复杂度下降的开始。

如果旧平台继续运行，团队仍然需要维护旧调度、旧告警、旧权限、旧模型、旧报表和旧知识。结果是 TCO 上升，而不是下降。

---

## 12. Migration Governance

迁移需要治理，但治理不应变成低效审批。

治理的目标是让迁移状态透明、风险可控、责任清楚。

### 12.1 每个迁移对象应有状态

建议状态包括：

```text
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

也可以根据团队需要调整，但必须能表达对象当前处于迁移生命周期哪一阶段。

### 12.2 每个迁移对象应有 owner

至少需要：

* technical owner；
* business owner；
* consumer owner；
* platform owner when relevant。

没有 owner 的对象，不应该进入生产 cutover。

### 12.3 迁移决策应可追踪

重要迁移决策应记录：

* 为什么迁移；
* 迁移范围；
* 新旧差异；
* parity 结果；
* cutover 结论；
* rollback plan；
* decommission 日期；
* 已知风险；
* 业务签核。

这不是为了形式化，而是为了防止几个月后没人记得为什么这样切换。

---

## 13. Migration 与 FinOps

迁移和 FinOps 必须结合。

### 13.1 迁移期间成本上升是正常的

Dual-run、backfill、parity、shadow 都会增加成本。

这些成本应该被识别为 migration cost，而不是 steady-state cost。

### 13.2 成本应该按阶段解释

可以分为：

| 阶段           | 成本特征         |
| ------------ | ------------ |
| Build        | 新平台开发和测试成本上升 |
| Backfill     | 历史数据重建成本上升   |
| Dual-run     | 新旧平台同时运行     |
| Cutover      | 支持和验证成本上升    |
| Shadow       | 旧平台短期保留      |
| Decommission | 成本开始下降       |
| Steady-state | 新平台稳定运行成本    |

### 13.3 Decommission 才能兑现 TCO 收益

如果迁移只建设新平台，不关闭旧平台，TCO 很难下降。

FinOps review 应持续追踪：

* 已迁移多少对象；
* 已下线多少旧对象；
* 仍在 dual-run 的对象；
* shadow 超期对象；
* 无 owner legacy 对象；
* 旧平台残余成本。

---

## 14. Data Quality 与 Reconciliation

迁移中的数据质量不只是测试新模型是否通过。

它更关注新旧平台是否在业务上可替代。

### 14.1 Reconciliation 维度

迁移验证可以包括：

* row count；
* key coverage；
* metric totals；
* aggregate by dimension；
* historical trend；
* null rate；
* duplicate rate；
* freshness；
* business scenario sample；
* consumer output comparison。

### 14.2 历史回填要单独验证

历史 backfill 容易出现：

* 源数据缺失；
* 历史 schema 变化；
* 旧业务规则不同；
* 时间区间错位；
* 迟到数据处理不同；
* timezone 差异；
* historical correction 未包含。

这些问题不能用当前数据验证替代。

### 14.3 不要盲目追求 100% 相等

如果旧平台有 bug，新平台不应该复制 bug 只是为了 parity。

更合理的做法是：

* 找出差异；
* 分类差异；
* 解释差异；
* 由 business owner 确认是否接受；
* 记录新口径；
* 必要时回填历史。

---

## 15. Communication 与 Change Management

数据平台迁移不是纯技术活动。

如果用户不知道什么时候切换、指标为什么变化、旧报表什么时候下线，就会产生不信任。

### 15.1 需要沟通什么

至少沟通：

* 迁移对象；
* 业务影响；
* 新模型位置；
* 指标变化；
* 已知差异；
* cutover 时间；
* shadow period；
* 旧路径下线时间；
* 支持渠道；
* rollback 条件。

### 15.2 面向不同人群的沟通

不同人关注不同问题：

| 角色                    | 关注点                |
| --------------------- | ------------------ |
| Business owner        | 指标是否可信，口径是否变化      |
| Analyst               | 新表在哪里，怎么使用         |
| BI owner              | dashboard 是否需要调整   |
| Application owner     | API / serving 是否稳定 |
| Finance / Operations  | 报表周期是否受影响          |
| Platform team         | 权限、监控、运行成本         |
| Governance / Security | 敏感数据和访问边界          |

### 15.3 培训和支持

如果新平台改变了使用方式，需要提供：

* quick start guide；
* model catalog；
* example queries；
* known differences；
* office hours；
* migration FAQ；
* owner contacts。

迁移成功不是用户被迫使用新平台，而是用户理解为什么新平台更可信。

---

## 16. 风险与控制

### 16.1 常见风险

| 风险                   | 说明                      | 控制方式                                  |
| -------------------- | ----------------------- | ------------------------------------- |
| Hidden dependency    | 旧对象仍被使用但没人知道            | usage tracking、consumer discovery     |
| Metric mismatch      | 新旧指标不一致                 | parity、business sign-off              |
| Cutover failure      | 新路径切换后不可用               | rollback plan、shadow period           |
| Cost spike           | dual-run 和 backfill 成本高 | migration workload tagging            |
| Scope creep          | 迁移过程中不断加需求              | change control、phase boundary         |
| Owner missing        | 无人决策口径或切换               | owner assignment                      |
| Governance gap       | 新平台权限过宽                 | access review、layered access          |
| Legacy never retired | 旧平台长期保留                 | decommission gate                     |
| User distrust        | 用户不信任新结果                | communication、reconciliation evidence |

### 16.2 风险控制原则

建议原则：

* 没有 owner，不 cutover；
* 没有 parity，不 cutover；
* 没有 rollback，不 cutover；
* 没有 shadow end date，不进入 shadow；
* 没有 decommission plan，不算迁移完成；
* 没有 business explanation，不强行接受指标差异。

---

## 17. 核心 trade-off

| 设计选择              | 好处            | 代价                     |
| ----------------- | ------------- | ---------------------- |
| 渐进式迁移             | 风险低，便于验证      | 总周期更长                  |
| Dual-run          | 建立信任，可回滚      | 短期成本上升                 |
| Parity check      | 差异可解释         | 需要测试和业务参与              |
| 业务优先排序            | 迁移价值更高        | 需要跨团队协调                |
| 按新架构重建            | 消除旧复杂度        | 不是简单复制，工作量更高           |
| Shadow period     | 降低 cutover 风险 | 若无结束条件，会变成永久双运行        |
| Decommission gate | 真正降低 TCO      | 需要依赖清理和组织决策            |
| 严格 owner 机制       | 责任清楚          | 需要组织配合                 |
| 迁移成本单独归因          | 防止误判新平台成本     | 需要 tagging 和 FinOps 纪律 |

---

## 18. 常见反模式

| 反模式                 | 问题                    | 更好的做法                            |
| ------------------- | --------------------- | -------------------------------- |
| Big Bang 迁移         | 隐藏依赖和口径差异集中爆发         | 分阶段迁移和 cutover                   |
| 只建新平台，不下线旧平台        | TCO 和复杂度上升            | 每个对象都有 decommission path         |
| 旧逻辑原样复制到新平台         | 旧技术债被带入新架构            | 按 Medallion 和新数据契约重建             |
| 没有 inventory 就开始迁移  | 范围和风险不清               | 先建立 legacy inventory             |
| Dual-run 没有结束条件     | 变成永久双运行               | 定义退出标准和 shadow end date          |
| Parity 只看总数         | 掩盖 key coverage 和业务差异 | 多层次 parity 和业务场景验证               |
| 新旧差异不记录             | 后续无法解释                | 分类、记录、签核差异                       |
| Cutover 没有 rollback | 切换风险过高                | 每次 cutover 都有 rollback path      |
| 迁移成本不单独标记           | 误判 steady-state 成本    | 标记 migration / backfill workload |
| Shadow 期间允许新增旧依赖    | 旧平台重新增长               | 冻结旧路径新增消费                        |
| 没有用户沟通              | 业务不信任新结果              | 提前说明变化、时间和支持方式                   |

---

## 19. 成功标准

一套设计良好的数据平台迁移，应满足：

* legacy inventory 覆盖主要 pipeline、报表、导出和 consumer；
* 每个迁移对象有 owner、状态和优先级；
* 高价值对象优先迁移；
* 新平台按标准架构重建，而不是机械复制旧逻辑；
* dual-run 有明确目标和退出条件；
* parity check 覆盖 row count、key、metric、freshness 和业务场景；
* 差异被分类、解释和签核；
* cutover 前有 rollback path；
* shadow period 有结束时间；
* 旧路径被冻结新增依赖；
* decommission 真实执行；
* migration / backfill / dual-run 成本被单独归因；
* 用户理解新旧差异和切换时间；
* 旧复杂度被移除，而不是隐藏在新平台旁边。

---

## 20. 小结

数据平台迁移不是把旧系统复制到新系统，也不是让新平台上线就算完成。

真正的迁移完成，是业务消费者安全切换到新平台，关键数据经过验证，旧链路被下线，权限和调度被清理，成本和复杂度真正下降。

一个稳健的迁移 playbook 应该遵循：

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

Big Bang 风险太高，永久 dual-run 也不可接受。

迁移的本质是通过可验证的小步切换，把 legacy 复杂度逐步收敛到新的 lakehouse 架构中，并最终移除旧系统负担。

如果新平台上线后旧平台仍然长期运行，那么迁移还没有完成，只是复杂度换了一个更现代的外壳。
