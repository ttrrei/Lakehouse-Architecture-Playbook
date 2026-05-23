# 03 · Ingestion 与 Landing：统一接入不是为了多一层，而是为了降低长期复杂度

## 1. 文档目的

这篇文档讨论 lakehouse 架构中的数据接入层，重点是 **Landing 层为什么存在、它和 Bronze 有什么区别、CDC 与 near real-time 的关系是什么，以及如何设计一个可回放、可治理、可扩展的数据接入模式**。

它不是具体 connector、SQL、Terraform 或云服务配置手册。这里的目标是建立一套公开、通用、可复用的系统设计原则。

我的基本观点是：

> Ingestion 层的目标不是尽快把数据塞进 Snowflake，而是以可治理、可回放、可审计、可扩展的方式，把源系统的数据稳定交接给分析平台。

统一 Landing 的价值也不在于“多存一份数据”，而在于建立一个清晰的系统边界：

* 源系统负责交付数据；
* Landing 负责接收、记录和保留交接证据；
* Bronze 负责把数据变成分析平台内可查询的 raw layer；
* Silver / Gold 负责业务语义和消费模型。

---

## 2. Ingestion 层要解决的问题

数据接入看起来只是“把数据从 A 搬到 B”，但在企业平台中，它实际上要解决很多长期问题。

### 2.1 多源接入的一致性

企业数据源通常包括：

* operational databases；
* batch files；
* SaaS exports；
* third-party feeds；
* application events；
* APIs；
* manually supplied files；
* legacy system extracts。

如果每个源都单独设计一套 ingestion pattern，平台很快会出现大量特殊路径。每个路径都有自己的调度、命名、错误处理、补数方式、监控方式和 owner。

统一接入模式的意义，是让不同源系统进入平台后尽量遵循同一套交接规则。

### 2.2 Replay 与 backfill

接入层必须回答：如果下游加载失败、模型逻辑错误、数据质量规则误判、或者需要重新计算历史数据，平台能否重新播放原始输入？

如果没有稳定的 landing evidence，很多 backfill 会依赖源系统重新导出或人工补数。这会增加源系统负担，也让平台恢复路径不可靠。

### 2.3 Audit 与数据证据

企业数据平台经常需要回答：

* 源系统在某个时间点交付了什么？
* 哪些文件或变化记录进入了平台？
* 哪些成功加载？
* 哪些失败或被隔离？
* 哪些数据被后续处理为业务模型？

这些问题不能只靠最终 Gold 模型回答。它们需要 ingestion 层保留证据。

### 2.4 Governance 与隐私边界

并不是所有源数据都应该直接进入公共分析层。

某些数据可能包含 PII、secret、敏感业务字段、受限区域数据或需要特殊授权的数据。Ingestion 层必须支持在数据进入 common landing 或 Bronze 之前进行隔离、脱敏、tokenization 或拒绝。

如果治理逻辑后置到 Silver / Gold，敏感数据可能已经扩大了暴露范围。

### 2.5 Near real-time 的真实性

很多团队会把 CDC、streaming、near real-time 混在一起讨论。

但 ingestion 层必须把这个问题拆开：

* CDC 是否捕获了变化？
* 变化是否进入 Landing？
* 变化是否加载到 Bronze？
* Silver / Gold 是否刷新？
* BI 或 serving projection 是否可读？
* 业务是否真的消费到了新结果？

只有整条链路都满足目标 freshness，业务才真正获得 near real-time 数据。

---

## 3. Landing 和 Bronze：职责必须拆开，物理实现可以合并

在这套 reference architecture 中，Landing 和 Bronze 被建模为两个不同职责：

* **Landing** 是源数据进入平台的交接与证据层；
* **Bronze** 是数据进入分析平台后的 raw query layer。

多数企业级场景下，两者物理分离会带来更清晰的 replay、audit、governance 和 cost boundary；但在小规模、低风险或特定 table-format 架构中，两者也可以合并实现。

### 3.1 Landing 是 handoff boundary

Landing 更像是源系统和数据平台之间的交接点。

它回答的问题是：

* 源系统交付了什么？
* 数据是什么时候到达的？
* 原始输入是否还能重放？
* 加载失败后是否可以恢复？
* 哪些数据可以进入公共分析路径？
* 哪些数据需要隔离或处理？

Landing 的关键词是：

```text
handoff / replay / audit / archive / decoupling / source evidence
```

### 3.2 Bronze 是 raw query layer

Bronze 是进入 Snowflake 或其他分析平台后的 raw layer。

它回答的问题是：

* 数据如何被 SQL 查询？
* 如何支持后续 Silver 增量处理？
* 如何保留 source metadata？
* 如何记录 ingestion timestamp、source offset、operation type、file reference？
* 如何让下游模型不直接扫描对象存储文件？

Bronze 的关键词是：

```text
queryable raw layer / source-aligned tables / technical metadata / incremental processing
```

### 3.3 为什么职责拆开有价值

职责拆开的价值在于排障和治理边界更清楚。

如果数据没有出现在 Gold 中，团队可以逐层判断：

1. 源系统是否产生数据？
2. 数据是否到达 Landing？
3. 数据是否加载到 Bronze？
4. Silver 是否处理成功？
5. Gold 是否刷新？
6. 消费层是否读取到最新结果？

如果 Landing 和 Bronze 的职责混在一起，很多问题会变成模糊的“数据没到”。

### 3.4 什么时候可以合并实现

Landing 和 Bronze 不一定永远需要物理拆成两套系统。

可以合并或弱化 Landing 的情况包括：

* 小规模 PoC；
* 低价值、可丢弃数据；
* 源系统或 connector 已经提供可靠历史重放；
* 使用 Iceberg / Delta / external table 等 table-format 架构，object storage 本身就是 raw table 存储；
* 对端到端延迟极度敏感，且 replay / audit 要求较低。

但即使物理实现合并，逻辑职责仍然应该清楚：什么是 source handoff evidence，什么是 analytical raw query layer。

---

## 4. 统一 Landing 的设计原则

### 4.1 统一不是所有源都用同一个 connector

统一 Landing 并不意味着所有源系统必须使用完全相同的接入工具。

数据库可能使用 CDC；第三方系统可能推送文件；SaaS 可能通过 API 拉取；应用事件可能以 micro-batch 形式落地。

统一的重点是：进入平台后，数据应尽量遵循一致的交接模式。

这包括：

* 标准路径或命名；
* 明确 source identifier；
* 明确 event time 和 ingestion time；
* 明确 batch 或 change boundary；
* 明确 schema version；
* 明确 file 或 message identity；
* 明确 replay 和 DLQ 规则；
* 明确 PII 和敏感数据处理边界。

### 4.2 Landing 应该尽量保持业务逻辑中立

Landing 层不应该承载复杂业务逻辑。

它可以做：

* 格式标准化；
* 基本结构校验；
* 文件完整性检查；
* PII 识别或隔离；
* 重复文件识别；
* metadata 补充；
* 拒绝明显不合规数据。

它不应该做：

* 核心业务指标计算；
* 客户分群；
* 财务口径；
* 风险规则；
* 大量业务 join；
* 面向 BI 的建模。

这些业务逻辑应该进入 Silver / Gold，而不是藏在接入脚本里。

### 4.3 Landing 应该支持幂等

数据接入必须考虑重复交付。

源系统可能重发文件，CDC connector 可能重放一段 offset，API 拉取可能重复获取同一批记录，人工补数也可能覆盖已有数据。

因此 Landing 需要有某种形式的 identity 或 fingerprint，用于判断：

* 这是新文件还是重复文件？
* 这是同一 source offset 的重复交付吗？
* 这是修正后的再交付吗？
* 是否应该跳过、覆盖、隔离或重新加载？

没有幂等设计，backfill 和 replay 会变得危险。

### 4.4 Landing 应该支持分区和生命周期

Landing 不是无限期热存储。

它通常需要生命周期策略：

* 最近数据保留在热访问层；
* 历史数据转入低成本层；
* 需要长期保留的数据进入 archive；
* 临时或敏感 staging 按更短周期清理。

这也是 Landing 和 Bronze 分离的一个价值：

* Landing 可以按存储成本和保留策略管理；
* Bronze 可以按查询性能和建模需求管理。

---

## 5. 常见接入模式

### 5.1 Batch file ingestion

Batch file ingestion 适合：

* 每日或每小时文件；
* 第三方 feed；
* SaaS export；
* 财务或运营对账文件；
* 低频业务数据。

典型链路：

```text
Source export
  -> Landing
  -> validation
  -> Bronze load
  -> Silver / Gold refresh
```

优点：

* 简单；
* 易审计；
* 易重放；
* 成本可控。

代价：

* freshness 受批次频率影响；
* 迟到文件需要处理；
* 文件 schema drift 需要治理。

### 5.2 CDC-based ingestion

CDC 适合：

* operational database changes；
* 增量加载；
* 需要捕获 insert / update / delete；
* 两次批处理之间的变化追踪；
* 分钟级 freshness。

典型链路：

```text
OLTP change log
  -> CDC capture
  -> Landing change files / batches
  -> Bronze raw change table
  -> Silver merge / business mirror
```

关键点：

* CDC 捕获变化，不等于实时消费；
* 需要 source offset 或 log position；
* 需要处理 delete；
* 需要处理 out-of-order；
* 需要处理 schema evolution；
* 需要定义 replay window。

### 5.3 API ingestion

API ingestion 适合：

* SaaS 系统；
* 第三方平台；
* 外部业务接口；
* 没有数据库级访问的系统。

典型链路：

```text
API pull / webhook
  -> normalized payload
  -> Landing
  -> Bronze
```

需要注意：

* rate limit；
* pagination；
* retry；
* partial failure；
* API schema version；
* token / credential rotation；
* source-side historical replay 是否可靠。

### 5.4 Event micro-batch ingestion

Event micro-batch 适合：

* 应用事件；
* 用户行为；
* 状态变化；
* 轻量 near real-time 分析。

它介于 batch 和 streaming 之间。

典型链路：

```text
Application events
  -> micro-batch buffer
  -> Landing
  -> Bronze
  -> incremental processing
```

这种模式比真正 streaming 简单，但比每日 batch 更新更及时。

### 5.5 Streaming ingestion

Streaming ingestion 适合真正需要 event-by-event processing 的场景。

例如：

* 毫秒级 SLA；
* 实时风控；
* 高频事件流；
* complex event processing；
* 在线推荐；
* 交易级实时决策。

如果业务确实需要这些能力，streaming platform 应该被作为一等架构组件设计。

但如果需求只是分钟级 freshness，streaming-first 往往会引入过度复杂度。

---

## 6. CDC 不等于 Real Time

CDC 是 ingestion 设计里最容易被误解的概念之一。

CDC 的价值是捕获源系统变化，让平台知道两次加载之间发生了什么。它非常适合增量加载、对账、补偿、更新业务镜像和降低 full reload 成本。

但 CDC 只解决了链路的第一段。

从源系统变化到业务消费，仍然需要经过：

```text
Change capture
  -> Landing
  -> Bronze load
  -> Silver processing
  -> Gold refresh
  -> BI refresh or serving publish
  -> downstream consumption
```

所以，判断 near real-time 不能只看是否有 CDC，而要看端到端 freshness。

### 6.1 什么时候只需要 CDC，不需要 streaming

很多场景只是需要知道 batch loading 间隔内发生了什么：

* 哪些订单更新了；
* 哪些客户状态变化了；
* 哪些账户需要重新计算；
* 哪些聚合需要刷新；
* 哪些数据需要补偿处理。

这类需求需要 CDC，但不一定需要完整 streaming stack。

如果业务可以接受分钟级刷新，那么 CDC + micro-batch + Snowflake-native incremental processing 可能已经足够。

### 6.2 什么时候 CDC 不够

CDC 不够的场景包括：

* 业务需要毫秒级响应；
* 每个事件都需要立即触发决策；
* 需要复杂窗口计算；
* 需要跨事件状态机；
* 需要高吞吐 event processing；
* 下游应用依赖事件流而不是数据表刷新。

这些场景应该进入 streaming / event-driven architecture，而不是只加强 CDC。

---

## 7. Schema Drift 与数据契约

Ingestion 层必须默认假设 schema 会变化。

源系统可能新增字段、删除字段、改变类型、改变枚举值、调整主键、改变文件格式或升级 API version。

### 7.1 Schema drift 的基本策略

常见策略包括：

| 变更类型           | 建议处理                                  |
| -------------- | ------------------------------------- |
| 新增 nullable 字段 | 接受到 raw payload，后续决定是否提升到 Silver      |
| 新增重要业务字段       | 通过 schema review 后进入 Silver / Gold    |
| 删除字段           | 触发兼容性检查和下游影响分析                        |
| 类型变窄           | 谨慎处理，可能进入 DLQ                         |
| 类型放宽           | 可接受，但需要记录 schema version              |
| 主键变化           | 需要 migration plan 和 backfill plan     |
| 枚举值变化          | 需要 accepted values 或 business rule 更新 |

### 7.2 为什么不要在 Bronze 过度结构化

Bronze 需要保留足够的原始信息，以便后续处理 schema drift。

如果 Bronze 过早强制一个过窄 schema，一旦源系统新增或调整字段，数据可能丢失或加载失败。

更好的做法是：

* Bronze 保留 raw payload；
* 关键技术字段结构化；
* Silver 再决定哪些字段成为稳定业务字段。

### 7.3 数据契约应该逐层增强

数据契约不应该只存在于 Gold。

不同层有不同契约：

* Landing：文件格式、source、batch identity、schema version；
* Bronze：技术元数据、raw payload、source offset；
* Silver：业务 key、去重规则、实体定义；
* Gold：指标口径、freshness、quality、owner、consumer contract。

契约逐层增强，平台才不会把所有复杂度都推给最后一层。

---

## 8. DLQ 和错误处理

Ingestion 层必须有明确的错误处理机制。

不是所有坏数据都应该阻塞整条 pipeline，也不是所有错误都应该静默忽略。

### 8.1 常见错误类型

| 错误类型              | 示例                             |
| ----------------- | ------------------------------ |
| 格式错误              | 文件无法解析、JSON malformed、CSV 列数错误 |
| schema drift      | 字段类型变化、缺少必要字段、未知结构             |
| duplicate input   | 文件重复、offset 重放、API 重试重复        |
| PII violation     | 明文敏感字段出现在不应出现的位置               |
| referential issue | 上游 key 缺失、父子记录顺序异常             |
| load failure      | Snowflake load 失败、权限或格式问题      |
| late arrival      | 超过预期窗口的数据迟到                    |

### 8.2 DLQ 的作用

DLQ 的目的不是把问题藏起来，而是隔离问题并保留诊断信息。

DLQ 需要回答：

* 哪条记录或哪个文件失败；
* 失败原因是什么；
* 属于哪个 source、batch、schema version；
* 是否可以重试；
* 是否需要人工处理；
* 是否涉及 PII 或安全事件。

### 8.3 错误处理原则

建议原则：

* 可隔离的坏记录不要阻塞整个源；
* PII 或安全问题应 fail closed；
* schema drift 应触发 review，而不是静默丢字段；
* duplicate input 应通过幂等机制处理；
* DLQ 应被监控，而不是长期堆积；
* 修复后应有 replay 或 reprocess 路径。

---

## 9. Replay、Backfill 与 Reprocessing

Replay 是 Landing 层最重要的价值之一。

没有 replay 能力，数据平台会过度依赖源系统重新导出、人工补数或手工修改下游表。

### 9.1 Replay 的常见场景

* Snowflake load 失败；
* Bronze schema mapping 错误；
* Silver 逻辑修复；
* Gold 指标口径升级；
* 历史数据重新计算；
* late-arriving data；
* duplicate 修复；
* PII 处理规则升级。

### 9.2 Replay 的层级

不同问题需要不同层级的 replay：

| 问题位置        | Replay 来源                       |
| ----------- | ------------------------------- |
| Bronze 加载失败 | 从 Landing 重载                    |
| Silver 逻辑错误 | 从 Bronze 重建 Silver              |
| Gold 指标错误   | 从 Silver 或 Gold source model 重建 |
| 源数据错误       | 需要源系统重新交付或修正 feed               |
| PII 处理规则变化  | 可能需要从安全 staging 或历史受控源重新处理      |

### 9.3 Backfill 不应该是特殊事故流程

Backfill 是数据平台的正常能力，不应该每次都靠人工临时脚本。

一个成熟 ingestion pattern 应该让 backfill 成为受控流程：

* 指定 source；
* 指定时间范围；
* 指定 schema version；
* 指定目标层；
* 记录执行结果；
* 验证数据质量；
* 避免重复写入。

---

## 10. PII 和 Sensitive Data Handling

Ingestion 层是隐私控制的第一道边界。

如果敏感数据已经进入 common landing、Bronze、BI 或通用分析 schema，再想控制暴露范围会困难得多。

### 10.1 Common landing 不应该接收未经治理的敏感数据

建议原则：

> Common landing 应该接收已经被允许进入分析平台的数据。未经处理的敏感数据应进入隔离 staging，经过处理后再进入 common landing 或 Bronze。

常见处理包括：

* hash；
* tokenize；
* redact；
* mask；
* drop；
* isolate；
* reject。

### 10.2 Fail closed

如果 ingestion 层检测到不应出现的敏感字段，应优先 fail closed。

也就是说：

* 不加载到公共路径；
* 不进入 Bronze；
* 写入受控错误区；
* 触发安全或数据治理 review；
* 修复后再重新处理。

### 10.3 隐私规则应该可回放

如果 hash / tokenization 规则升级，平台需要考虑历史数据如何重新处理。

这要求 ingestion metadata 中保留：

* privacy rule version；
* schema version；
* source batch；
* processing timestamp；
* reprocessing path。

---

## 11. Freshness、Completeness 与可观测性

Ingestion 的成功不只是“job 成功”。

一个 ingestion pipeline 至少要观察以下维度。

### 11.1 Freshness

需要知道：

* 源事件时间到当前时间的 lag；
* 文件到达时间；
* Bronze 加载时间；
* Silver / Gold 刷新时间；
* serving projection 更新时间。

Freshness 应该端到端衡量，而不是只看 connector 是否运行。

### 11.2 Completeness

需要知道：

* 预期 batch 是否都到了；
* row count 是否异常；
* 文件大小是否异常；
* CDC offset 是否连续；
* 是否有重复或缺口；
* 是否有 source 长时间无数据。

### 11.3 Correctness

需要知道：

* schema 是否符合预期；
* 类型是否正确；
* 必填字段是否为空；
* enum 是否在允许范围；
* business key 是否可解析；
* PII 是否出现在不应出现的位置。

### 11.4 Cost and operational signals

还需要观察：

* 小文件数量；
* load cost；
* failed file count；
* DLQ growth；
* replay frequency；
* source-specific latency；
* ingestion compute consumption。

这些指标不仅用于排障，也用于 FinOps 和 TCO 管理。

---

## 12. Ingestion Pattern 的核心 trade-off

| 设计选择                  | 好处                           | 代价                                     |
| --------------------- | ---------------------------- | -------------------------------------- |
| 使用统一 Landing          | 解耦源系统和分析平台，支持 replay 和 audit | 增加对象存储、metadata 和生命周期管理                |
| Landing / Bronze 职责拆开 | 边界清晰，便于治理和排障                 | 小规模场景可能显得偏重                            |
| 使用 CDC                | 捕获 batch 间变化，支持增量处理          | 不等于端到端 real time，需要 offset 和 schema 管理 |
| 使用 micro-batch        | 降低 streaming 复杂度，支持分钟级刷新     | 不适合毫秒级事件处理                             |
| Bronze 保留 raw payload | 支持 schema drift 和后续字段提升      | 查询和存储需要治理                              |
| Ingestion 层不放业务逻辑     | 职责清晰，便于维护                    | 部分业务处理需要推迟到 Silver / Gold              |
| PII 在进入公共路径前处理        | 降低隐私风险                       | 增加接入复杂度和重处理要求                          |
| DLQ 隔离坏数据             | 避免局部坏数据阻塞整条链路                | 需要监控和处理流程                              |
| 支持 replay / backfill  | 可恢复、可重建                      | 需要幂等、metadata 和执行纪律                    |

---

## 13. 常见反模式

| 反模式                             | 问题                          | 更好的做法                                |
| ------------------------------- | --------------------------- | ------------------------------------ |
| 每个源系统定制一套 ingestion 架构          | 长期维护和排障复杂                   | 使用统一 Landing 和接入契约                   |
| 直接把源数据写入 Gold                   | 缺少 raw evidence 和 replay 边界 | 先进入 Landing / Bronze                 |
| 在 ingestion adapter 中写大量业务逻辑    | 难测试、难复用、难治理                 | 业务逻辑放入 Silver / Gold                 |
| 有 CDC 就默认建设完整 streaming stack   | 把变化捕获误解为实时消费                | 根据端到端 freshness SLA 设计               |
| Landing 无限期热存储                  | 成本不可控                       | 使用 lifecycle 和 archive 策略            |
| Bronze 过度结构化                    | schema drift 时容易丢数据或阻塞      | 保留 raw payload，逐层增强契约                |
| 明文敏感数据进入 common landing         | 扩大隐私风险面                     | 使用隔离 staging 和 ingest-time handling  |
| DLQ 只写不管                        | 问题被隐藏，数据债务累积                | 监控 DLQ 并设计处理流程                       |
| Backfill 依赖临时脚本                 | 不可审计、不可重复                   | 建立受控 replay / backfill 机制            |
| 只看 connector 成功，不看端到端 freshness | 误判数据可用性                     | 监控从 source event 到 consumption 的完整链路 |

---

## 14. 成功标准

一套设计良好的 ingestion 和 landing 架构，应该满足以下标准：

* 新数据源可以复用标准接入模式；
* Landing 和 Bronze 的职责清楚；
* 多数企业级数据源具有 replay 路径；
* Bronze 保留足够技术元数据；
* CDC 链路能解释 source offset 和变化语义；
* near real-time freshness 是端到端定义的；
* PII 和敏感数据在进入公共路径前被处理或隔离；
* schema drift 有处理策略；
* DLQ 有 owner、监控和处理流程；
* backfill 不是一次性临时事故流程；
* 成本和小文件问题被纳入监控；
* ingestion 层不隐藏核心业务逻辑；
* 下游 Silver / Gold 可以基于 Bronze 稳定重建。

---

## 15. 小结

Ingestion 和 Landing 是 lakehouse 架构中最容易被低估的一层。

如果只把它理解成“把数据加载到 Snowflake”，平台很容易在后期付出代价：源系统耦合、无法回放、审计不清、PII 边界模糊、schema drift 失控、CDC 被误解为实时消费、backfill 依赖人工脚本。

一个更稳健的设计是：

* Landing 作为源数据进入平台的 handoff 和 evidence layer；
* Bronze 作为分析平台内的 raw query layer；
* CDC 用于 change awareness 和 incremental capture，而不是自动代表 real time；
* PII 和敏感数据在进入公共路径前处理；
* ingestion 层保持业务逻辑中立；
* replay、DLQ、schema drift 和 freshness 从一开始就是平台能力。

这套设计会比直接接入多一些前期结构，但它换来的是更低的长期复杂度、更清晰的治理边界和更可靠的迁移与恢复能力。
