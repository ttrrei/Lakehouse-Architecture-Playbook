# 04 · Data Modeling：Medallion 不是分层命名，而是语义解耦

## 1. 文档目的

这篇文档讨论 lakehouse 架构中的数据建模方式，重点是 **Bronze / Silver / Gold 到底应该承担什么职责，以及为什么 Medallion 架构的核心价值不是分层命名，而是解耦 OLTP 结构和业务语义**。

它不是一份维度建模教程，也不是某个行业的数据模型模板。这里关注的是更通用的系统设计问题：

> 当数据从源系统进入分析平台之后，如何从“源系统存储结构”逐步演进成“业务可理解、可治理、可消费的数据产品”？

我的基本观点是：

> Medallion 架构的价值，不是把表分成 Bronze / Silver / Gold 三个名字，而是把两类不同的数据操作解耦：第一类是把 OLTP 系统的存储结构转换成稳定的业务镜像，第二类是从业务镜像中提取指标、报表、数据产品和 operational projection。

这种解耦带来几个重要结果：

* OLTP 系统可以为了应用需求继续变化；
* 数据平台不需要让所有下游直接依赖源系统结构；
* 业务逻辑可以在 Silver / Gold 中逐步升级；
* 数据质量、权限、成本和 lineage 更容易管理；
* near real-time 场景可以按层定义 freshness，而不是混在一条不可解释的脚本里。

---

## 2. 为什么需要数据建模层

很多数据平台的问题不是没有数据，而是数据没有形成稳定语义。

如果数据从源系统接入后直接被 BI、报表、分析师和应用消费，平台会很快遇到以下问题。

### 2.1 源系统结构不等于业务语义

OLTP 表通常服务于应用写入、事务一致性和查询性能。它们的结构可能体现的是系统实现方式，而不是业务用户理解世界的方式。

例如，一个业务对象可能拆在多个应用表中；一个源表可能混合多个业务概念；字段名可能来自历史实现；主键可能只在某个系统内有意义；状态变化可能隐藏在更新记录中。

如果 BI 和指标直接依赖这些结构，源系统的每一次变化都会传导到业务消费层。

### 2.2 业务逻辑会散落

没有清晰建模层时，业务逻辑很容易散落到：

* BI 工具；
* 临时 SQL；
* Python 脚本；
* 应用代码；
* 导出任务；
* 人工 Excel；
* 数据同步 job。

结果是同一个指标有多个版本，同一个实体有多个定义，同一个报表问题需要追查多个系统。

### 2.3 数据质量难以定位

如果没有层次边界，数据质量问题很难定位。

当报表数字错误时，团队很难判断：

* 是源系统数据错误；
* 是 ingestion 丢失；
* 是 Bronze 解析错误；
* 是 Silver 去重逻辑错误；
* 是 Gold 指标定义错误；
* 是 BI 展示层计算错误。

分层建模的价值之一，就是让问题定位变成逐层排查，而不是在整条链路里猜测。

### 2.4 治理和成本无法落到具体对象

没有稳定模型，就很难定义：

* owner；
* freshness SLA；
* quality test；
* PII classification；
* retention；
* consumer contract；
* cost attribution；
* downstream dependency。

治理和 FinOps 需要落在具体数据产品上，而不是只落在一堆源表和报表上。

---

## 3. Medallion 的核心理解

很多人把 Medallion 理解成三层数据表：Bronze、Silver、Gold。

我认为这个理解还不够。更准确地说，Medallion 是一种 **语义演进和责任分离模式**。

```text
Source storage structure
  -> Bronze: raw evidence
  -> Silver: business mirror
  -> Gold: business consumption
```

### 3.1 第一类操作：从 OLTP 结构到业务镜像

源系统中的表结构通常不是稳定业务语义。

第一类操作是把这些表结构转换成相对稳定的业务镜像。这一层通常发生在 Bronze 到 Silver。

它包括：

* 清洗格式；
* 统一类型；
* 去重；
* 处理 update / delete；
* 解析源系统状态；
* 对齐多个源系统；
* 建立业务 key；
* 合并同一实体的多个来源；
* 处理历史变化；
* 屏蔽源系统特有字段。

这一步的目标不是做最终指标，而是让下游可以面对业务对象，而不是面对 OLTP 表结构。

### 3.2 第二类操作：从业务镜像到有效信息

第二类操作是从业务镜像中提取有效信息。这通常发生在 Silver 到 Gold。

它包括：

* 指标定义；
* fact / dimension 建模；
* aggregation；
* business mart；
* semantic-ready model；
* operational serving source model；
* reporting model；
* data product。

这一步的目标是支持业务消费，而不是继续清理源系统细节。

### 3.3 解耦之后的好处

解耦后，平台具备几个优势：

* OLTP schema 可以变化，但 Silver 尽量保持业务镜像稳定；
* 业务指标可以升级，但不必重新解释所有源系统细节；
* 下游消费可以基于 Gold，而不是直接读源表；
* 数据质量可以按层测试；
* lineage 更容易解释；
* 成本可以按模型和 use case 归因。

---

## 4. Bronze：Raw Evidence Layer

Bronze 是数据进入分析平台后的原始证据层。

它的职责是保留源系统输入和技术上下文，为后续处理提供可重建基础。

### 4.1 Bronze 应该保留什么

Bronze 通常应该保留：

* 源系统标识；
* 原始业务字段；
* source event time；
* ingestion time；
* source operation，例如 insert / update / delete；
* source offset 或 file reference；
* pipeline identifier；
* raw payload；
* schema version；
* record hash 或 fingerprint。

这些字段的目的不是为了业务用户直接查询，而是为了让后续层能够理解数据从哪里来、什么时候来、如何变化、是否可重放。

### 4.2 Bronze 不应该做什么

Bronze 不应该承担：

* 核心业务指标；
* 面向 BI 的语义模型；
* 复杂跨源 join；
* 客户分群、风险评分、财务口径等业务逻辑；
* operational serving；
* 最终权限投影。

如果 Bronze 开始直接服务大量业务报表，说明平台缺少稳定的 Silver / Gold。

### 4.3 Bronze 的设计原则

Bronze 的设计应遵循：

1. **保真优先**：尽量保留源系统信息。
2. **技术元数据完整**：支持 replay、debug、lineage。
3. **schema drift 友好**：不要过早丢弃未知字段。
4. **append-friendly**：尤其是 CDC 和 event 数据。
5. **不面向业务消费**：避免把 raw layer 变成事实上的 reporting layer。

### 4.4 Bronze 与 Landing 的关系

Landing 是源数据进入平台的 handoff 和 evidence layer，Bronze 是进入分析平台后的 raw query layer。

多数企业级场景下，两者物理分离有助于 replay、audit、governance 和 cost boundary。但在小规模、低风险或特定 table-format 架构中，两者也可以合并实现。

关键不是物理上必须拆两套系统，而是逻辑职责必须清楚。

---

## 5. Silver：Business Mirror Layer

Silver 是 Medallion 架构中最容易被低估的一层。

很多团队把 Silver 简单理解成“清洗层”。但我认为 Silver 更准确的定位是：**业务镜像层**。

### 5.1 Silver 的核心职责

Silver 负责把源系统结构转成稳定业务对象。

它通常处理：

* 去重；
* 类型标准化；
* null 和 default 处理；
* 枚举值标准化；
* 多源 schema 对齐；
* source key 到 business key 的映射；
* entity resolution；
* update / delete 语义；
* late-arriving data；
* SCD 或历史状态；
* 基础业务规则；
* 基础质量检查。

Silver 的目标是让下游能够使用业务概念，而不是源系统概念。

### 5.2 Business Mirror 的含义

Business mirror 不是最终报表，也不是完整业务指标。

它是对业务实体和业务过程的稳定映射。

例如，一个通用企业平台可能会形成这样的 Silver 对象：

* customer；
* account；
* product；
* order；
* transaction；
* subscription；
* payment；
* event；
* case；
* user activity。

这些对象不是某个 OLTP 表的简单复制，而是经过标准化和语义整理后的业务镜像。

### 5.3 Silver 应该保持相对稳定

Silver 的价值在于屏蔽源系统变化。

当源系统新增字段、拆表、合表、调整字段名或升级 API 时，Silver 应尽量保持对下游的稳定性。

这不意味着 Silver 永远不变，而是它的变化应该通过数据契约、版本管理和兼容性策略进行。

### 5.4 Silver 不应该过早承载最终指标

Silver 可以包含基础业务规则，但不应该过早承载大量最终指标。

例如：

* 标准化交易金额可以在 Silver；
* 最终收入指标应在 Gold；
* 标准化客户状态可以在 Silver；
* 客户生命周期报表指标应在 Gold。

Silver 的目标是形成干净、稳定、可复用的业务对象，而不是变成另一个 Gold。

---

## 6. Gold：Business Consumption Layer

Gold 是业务消费层。

它的目标是把 Silver 中的业务镜像转成可以被 BI、分析、管理层、业务系统或数据产品直接使用的模型。

### 6.1 Gold 的常见对象

Gold 通常包括：

* fact tables；
* dimension tables；
* aggregate marts；
* business metrics tables；
* reporting models；
* semantic-ready models；
* operational serving source models；
* data product tables。

Gold 的消费者包括：

* BI dashboards；
* business analysts；
* finance / operations / product teams；
* executive reporting；
* downstream applications；
* data product consumers。

### 6.2 Gold 的核心职责

Gold 负责：

* 统一指标口径；
* 提供业务可读模型；
* 支持 BI 查询性能；
* 定义消费契约；
* 管理 freshness；
* 暴露 approved data products；
* 为 operational serving 提供稳定来源。

如果一个指标对业务重要，它应该尽量在 Gold 层定义，而不是在多个 BI 报表中重复实现。

### 6.3 Gold 不等于一个大宽表

很多平台会把 Gold 理解成“把所有字段拼到一张宽表里”。这通常会导致：

* 表过宽；
* 逻辑耦合；
* 指标重复；
* 刷新成本高；
* 下游依赖难管理；
* schema change 风险大。

更好的方式是根据消费模式设计 Gold：

* 稳定分析用 fact / dimension；
* 高频报表用 aggregate mart；
* 自助分析用 semantic-ready model；
* 应用点查用 serving source model；
* 特定数据产品用明确 contract 的 data product table。

---

## 7. Fact、Dimension 与 Wide Table 的取舍

Gold 层常见争议是：应该做 Kimball 风格的 fact / dimension，还是直接做宽表。

### 7.1 Fact / Dimension 的价值

Fact / dimension 建模适合：

* 多维分析；
* 指标稳定；
* BI 复用；
* 多报表共享；
* 维度缓慢变化；
* 历史分析；
* 复杂 slicing and dicing。

它的好处是结构清晰、语义稳定、复用性好。

代价是建模成本较高，对团队的数据建模能力有要求。

### 7.2 Wide Table 的价值

宽表适合：

* 特定报表；
* 特定数据产品；
* 高性能读取；
* 下游消费简单；
* serving source model；
* 低维度、低变化场景。

它的好处是使用简单、查询直接。

代价是容易耦合过多逻辑，长期维护成本可能更高。

### 7.3 建议判断方式

可以用消费模式判断：

| 场景           | 更适合                             |
| ------------ | ------------------------------- |
| 多个报表共享同一业务过程 | fact / dimension                |
| 高频固定报表       | aggregate mart 或宽表              |
| 自助分析         | semantic-ready fact / dimension |
| 应用点查         | serving-oriented wide model     |
| 一次性分析        | 临时 model 或 view                 |
| 复杂历史维度       | dimension with SCD              |

不要因为“现代 lakehouse”就放弃维度建模，也不要因为“宽表简单”就把所有逻辑压进一张表。

---

## 8. SCD 与历史变化

不是所有数据都需要 SCD，但需要历史语义的维度必须认真处理。

### 8.1 什么时候需要 SCD

SCD 适合：

* 客户属性历史；
* 账户状态历史；
* 产品属性变化；
* 组织结构变化；
* 合同或订阅状态；
* 需要按历史当时状态还原分析的问题。

如果业务问题需要回答“当时是什么状态”，就需要保留历史。

### 8.2 什么时候不应该用 SCD

不建议在以下场景滥用 SCD：

* 高容量事件表；
* append-only fact；
* 不需要历史语义的临时属性；
* 可从事件重建的状态；
* 低价值字段变化。

SCD 有成本，包括存储、merge、测试、查询复杂度和理解成本。

### 8.3 常见字段

典型 SCD2 维度通常需要：

* surrogate key；
* natural key；
* valid_from；
* valid_to；
* is_current；
* record_hash；
* source_system；
* updated_at。

但具体实现不应该机械套模板，而应根据业务需要决定。

---

## 9. 物化策略：不是所有模型都需要 Near Real-Time

物化策略是数据建模和 FinOps 的交汇点。

同一个逻辑模型，可以选择：

* view；
* table；
* incremental table；
* Dynamic Table；
* materialized view；
* stream-driven table；
* serving projection source。

选择不应只看技术偏好，而应看 freshness、查询模式、成本和复杂度。

### 9.1 常见物化方式

| 物化方式                 | 适合场景                 | 风险                         |
| -------------------- | -------------------- | -------------------------- |
| View                 | 轻量逻辑复用、低频查询          | 高频查询可能慢，成本不可控              |
| Table                | 稳定报表、每日或每小时刷新        | 数据不是实时，需要调度                |
| Incremental table    | 大数据量、增量处理            | 需要 merge / dedupe 逻辑       |
| Dynamic Table        | 明确 near real-time 需求 | 持续刷新可能增加成本                 |
| Materialized view    | 特定查询加速               | 适用范围有限，维护成本需评估             |
| Stream + Task        | 需要显式增量逻辑             | 复杂度高于简单 batch              |
| Serving source model | 应用点查来源               | 需要 serving contract 和一致性管理 |

### 9.2 Dynamic Table 不是默认答案

Dynamic Table 对 near real-time 模型很有价值，但不应该成为所有 Gold 模型的默认物化方式。

如果一个模型只服务每日报表，或者业务只按小时查看，持续刷新可能只是增加成本，而不增加业务价值。

判断一个模型是否需要 near real-time，应问：

* 消费者真的需要分钟级 freshness 吗？
* 数据变化频率是否足够高？
* 查询频率是否足够高？
* 延迟降低是否影响业务决策？
* 成本是否可接受？
* 是否有更简单的 batch 方案？

### 9.3 Freshness 应按层定义

Near real-time 不是全链路所有层都同样实时。

可以按层定义：

* Bronze：源变化进入 raw layer 的延迟；
* Silver：业务镜像更新延迟；
* Gold：业务指标刷新延迟；
* Serving：应用可读取延迟。

只有这样，团队才能知道 freshness 瓶颈在哪里。

---

## 10. Data Contract：模型不是表，而是承诺

数据模型不只是表结构。一个成熟模型应该是一个 data contract。

### 10.1 Contract 应该包含什么

一个 Silver 或 Gold 模型至少应该明确：

* 模型用途；
* owner；
* 业务定义；
* grain；
* primary key 或 uniqueness；
* freshness expectation；
* quality checks；
* PII classification；
* retention；
* materialization；
* cost attribution；
* downstream consumers；
* change policy。

没有这些信息，模型只是一个表，不是一个可靠数据产品。

### 10.2 Grain 是最重要的字段之一

很多数据模型问题来自 grain 不清楚。

例如：

* 一行代表一个事件？
* 一个订单？
* 一个客户一天？
* 一个账户当前状态？
* 一个聚合结果？

如果 grain 不清楚，指标就很容易重复计算或错误 join。

### 10.3 Change policy 很重要

数据模型一定会变化。

因此需要定义：

* 哪些变化是 backward compatible；
* 哪些变化需要新版本；
* 哪些消费者需要通知；
* 字段删除如何处理；
* 指标口径变化如何记录；
* historical backfill 是否需要执行。

没有 change policy，Gold 层会随着业务变化逐渐失去可信度。

---

## 11. Data Quality：逐层测试，而不是最后补救

数据质量不应该只在 Gold 层检查。

不同层应该有不同测试重点。

### 11.1 Bronze 测试

Bronze 重点测试：

* 文件或 batch 是否加载；
* 技术字段是否存在；
* source offset 是否连续；
* operation type 是否有效；
* raw payload 是否可解析；
* 是否有明显格式错误。

### 11.2 Silver 测试

Silver 重点测试：

* primary key 唯一性；
* business key 可解析；
* 去重是否正确；
* referential integrity；
* entity state 是否合理；
* SCD 是否连续；
* 核心字段 not null；
* accepted values。

### 11.3 Gold 测试

Gold 重点测试：

* 指标口径；
* 汇总一致性；
* 与来源模型的 reconciliation；
* freshness；
* completeness；
* consumer-facing constraints；
* PII exposure；
* performance expectation。

### 11.4 Quality failure 应该有 owner

测试失败只是开始。每个关键测试都应该有：

* owner；
* severity；
* expected response；
* exception process；
* downstream impact。

否则数据质量会变成一堆无人处理的红灯。

---

## 12. Modeling for Operational Serving

Gold 不只服务 BI，也可以作为 operational serving projection 的来源。

但这类模型需要单独思考。

### 12.1 Serving source model 的特点

Serving source model 通常需要：

* key-based grain；
* 明确 primary key；
* 当前状态或最新快照；
* compact payload；
* freshness target；
* last_updated timestamp；
* snapshot version；
* change detection；
* reconciliation rule。

它不是普通 BI mart，也不是事务表。

### 12.2 Serving projection 不应成为事实源

Operational serving store 中的数据应该可以从 Snowflake Gold 重建。

这意味着：

* Gold 仍然是 analytical source of truth；
* serving store 是 projection；
* 应用读取 projection；
* 业务交易状态仍然由 OLTP 系统拥有；
* 数据平台不直接写业务 OLTP。

### 12.3 Serving 模型需要 contract

如果一个 Gold 模型要发布到 serving store，它应该有明确 contract：

* 目标 consumer；
* key；
* payload schema；
* freshness；
* compatibility rule；
* reconciliation；
* failure handling；
* owner。

否则 serving projection 很容易演变成新的隐形 API。

---

## 13. 命名和组织原则

命名不是最重要的事情，但一致命名可以降低长期复杂度。

### 13.1 建议命名思想

命名应体现：

* layer；
* domain；
* entity；
* grain；
* purpose。

例如可以使用类似模式：

```text
bronze_<source>__<entity>
silver_<domain>__<entity>
fact_<business_process>
dim_<entity>
gold_<business_output>
serving_<target>
```

这只是示例，不是强制标准。

关键是团队能通过名称理解模型所在层级和用途。

### 13.2 不要把实现细节泄漏到业务模型

Gold 模型命名不应过度依赖源系统名、connector 名或临时项目名。

业务消费层应该表达业务概念，而不是表达源系统实现。

### 13.3 Domain ownership

模型最好按业务 domain 管理，例如：

* customer；
* product；
* finance；
* operations；
* risk；
* marketing；
* platform。

Domain ownership 有助于 owner、质量、成本和变更管理。

---

## 14. 核心 trade-off

| 设计选择                      | 好处                       | 代价                         |
| ------------------------- | ------------------------ | -------------------------- |
| 使用 Bronze / Silver / Gold | 职责清晰，lineage 清楚          | 层数增加，需要建模纪律                |
| Bronze 保留 raw payload     | 支持 schema drift 和 replay | 存储和查询需要治理                  |
| Silver 构建业务镜像             | 解耦 OLTP 和业务语义            | 需要理解源系统和业务概念               |
| Gold 统一业务消费               | 指标一致，BI 更稳定              | 需要 owner、contract 和质量测试    |
| 使用 fact / dimension       | 复用性和分析能力强                | 建模成本较高                     |
| 使用宽表                      | 消费简单，性能直接                | 长期可能耦合过重                   |
| 使用 Dynamic Table          | 支持 near real-time        | 持续刷新可能增加成本                 |
| 使用 batch table            | 简单、成本可控                  | freshness 较低               |
| Gold 作为 serving 来源        | 复用分析平台结果                 | 需要 serving contract 和一致性管理 |
| 严格 data contract          | 稳定、可治理                   | 增加开发流程                     |

---

## 15. 常见反模式

| 反模式                                | 问题           | 更好的做法                                       |
| ---------------------------------- | ------------ | ------------------------------------------- |
| BI 直接读取 Bronze                     | 业务语义不稳定      | 通过 Silver / Gold 提供消费模型                     |
| Silver 只是字段重命名                     | 没有形成业务镜像     | 在 Silver 中处理实体、key、状态和质量                    |
| Gold 变成所有字段大宽表                     | 耦合过重，维护困难    | 按消费模式设计 fact、dimension、mart 或 serving model |
| 每个报表单独实现指标                         | 指标口径漂移       | 在 Gold 统一核心指标                               |
| 所有模型都 near real-time               | 成本高，收益不明确    | 根据 freshness 和业务价值选择物化方式                    |
| 所有历史字段都做 SCD                       | 存储和查询复杂      | 只对需要历史语义的维度做 SCD                            |
| 模型没有 grain                         | join 和聚合容易错误 | 每个模型明确 grain                                |
| 模型没有 owner                         | 出问题无人负责      | 每个数据产品定义 owner                              |
| Contract 只在文档里，不在流程里               | 变更不可控        | 把 contract 纳入 review 和 CI                   |
| Serving projection 没有版本和 freshness | 应用消费不可控      | 明确定义 serving contract                       |

---

## 16. 成功标准

一套设计良好的数据建模体系，应满足：

* Bronze、Silver、Gold 职责清晰；
* 下游消费不直接依赖 OLTP schema；
* Silver 能形成稳定业务镜像；
* Gold 承载核心指标和消费模型；
* 每个重要模型有明确 grain；
* 关键模型有 owner、freshness、quality 和 change policy；
* BI 指标主要来自 Gold，而不是分散在报表工具中；
* SCD 只用于需要历史语义的维度；
* near real-time 模型有明确业务理由；
* 物化策略与 freshness、成本和查询模式匹配；
* serving source model 有 contract；
* 数据质量测试按层设计；
* 源系统变化不会直接破坏业务消费。

---

## 17. 小结

Data modeling 是 lakehouse 架构中连接技术和业务的关键层。

如果没有建模层，平台只是把数据从源系统搬到了 Snowflake；如果建模层设计不好，复杂度会从源系统转移到 BI、脚本和应用代码中。

Medallion 的价值在于语义解耦：

* Bronze 保留 raw evidence；
* Silver 把 OLTP 存储结构转换成稳定业务镜像；
* Gold 从业务镜像中提取指标、报表、数据产品和 serving source model。

这样，源系统可以变化，业务逻辑可以升级，下游消费可以稳定，治理和 TCO 也更容易管理。

真正成熟的数据模型，不只是能被查询，而是能被理解、被测试、被治理、被迁移、被下线，也能被业务长期信任。
