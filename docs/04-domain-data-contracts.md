# 领域数据与接口契约

业务数据由 TipTips 权威核心管理。下列实体、状态和接口是实现规格，不是已经存在的数据库或接口。领域词汇以 [CONTEXT](../CONTEXT.md) 为准。

## 1 共同数据规则

所有业务对象使用稳定、不暴露本机路径的 ID，并保存 space_id、创建/更新时间和 revision。用户可创建的离线对象使用可在客户端生成的随机 ID；服务端验证唯一性。标识格式实施时统一，不依赖 ID 推断真实时间顺序。

绝对时点保存 UTC，同时保留事件原始时区；全天日期保存日期范围，不能先转 UTC 再取日期。跨系统时区转换明确处理 Windows 与 IANA 标识。所有者、设备、连接、Profile 和执行节点有各自 ID，禁止共用一个“账号 ID”替代。

修改以对象 revision 做乐观并发控制。重试以 operation_id 去重；同标识不同内容返回冲突。业务删除默认软删除，保留删除版本和同步墓碑。正式清除按保留策略执行，同时清理派生索引与附件引用。

## 2 逻辑数据模型

| 逻辑集合 | 关键字段与关系 | 主要约束 |
| --- | --- | --- |
| personal_spaces | owner_id、authority_id、authority_epoch、display_name | 一个空间只有一个正式主机 |
| devices | space_id、identity、revoked_at、last_seen | 设备访问与执行权限分开 |
| projects | title、status、description、sort_key | 不把项目 ID 当目录 |
| inbox_items | original_text、source、source_message_id、processed_links | 原文与生成对象关联 |
| tasks | project_id、parent_id、title、status、priority、due_kind、due_date/due_at、estimate | 日期型截止与时点型截止互斥 |
| task_dependencies | predecessor、successor、relation | 禁止自依赖和环 |
| tags 与 task_tags | space_id、normalized_name、task_id | 空间内名称规则一致 |
| task_series | rule、timezone、completion_policy、next_due | 规则与具体任务分离 |
| calendars | title、provider_connection_id、read_only、color | 外部权限向业务层传递 |
| calendar_events | calendar_id、start/end、all_day、timezone、task_id | 结束晚于开始；全天结束日期为排他边界 |
| recurrence_series | dtstart、rule、timezone、revision | 只按有限范围展开 |
| recurrence_exceptions | series_id、original_occurrence_key、replacement/cancelled | 改期后仍能定位原始实例 |
| reminders | target_type/id、relative_or_absolute、channel_policy、snooze_until | 业务对象变化时重算 |
| schedules | trigger_kind、trigger_spec、timezone、misfire_policy、enabled | 计算逻辑具有确定性 |
| scheduled_jobs | schedule_id、occurrence_key、due_at、state、lease_owner/expiry | schedule 与 occurrence 唯一 |
| conversations 与 messages | profile_id、project_id、role、content_ref、ordinal、run_id | 顺序由服务端序号定义 |
| assistant_profiles | persona、model_profile、allowed_skills、policy | 人设不含凭据正文 |
| proposals 与 proposed_actions | base_revisions、action_type、arguments、authorization_ref | 执行前复核版本 |
| delegations | goal、acceptance_criteria、workspace_binding、runner、budget | 目标与运行分开 |
| runs | delegation/conversation_id、attempt、state、external_session/run_id、node_id | 每次尝试独立标识 |
| run_events | run_id、sequence、kind、payload_ref、created_at | run 内 sequence 唯一 |
| approvals | run_id、operation_digest、expires_at、decision、decider | 过期与内容变化不能复用 |
| grants | principal、capability、scope、expires_at、policy_revision | 每次副作用前验证 |
| artifacts | run_id、kind、location_ref、digest、size、retention | 位置按节点解释 |
| workspace_bindings | project_id、node_id、root_ref、write_policy | 同一项目可有多个明确绑定 |
| connections | provider_type、endpoint、scope、secret_ref、status | 模型与应用连接分类型 |
| settings_revisions | scope、revision、validated_values、applied_at | 校验、保存、生效可区分 |
| model_profiles 与 usage_ledger | provider、model_id、capabilities、reserved/actual_usage | 预算预留与实际结算分开 |
| mcp_servers 与 tool_catalog | transport、node_id、manifest_version、schema_hash | 工具列表变化可发现 |
| memory_items | fact、source、confirmation、scope、expires_at | 可撤回，可追溯 |
| notification_intents | reason、target、dedup_key、policy_snapshot | 同一事实稳定去重 |
| deliveries | intent_id、channel、recipient_ref、attempt、provider_id、state | 结果未知区别于失败 |
| inbound_messages | channel、provider_message_id、sender_binding、correlation | 渠道与消息 ID 唯一 |
| operations 与 outbox | operation_id、request_digest、response_ref；event_sequence | 业务事务内一起保存 |
| sync_cursors 与 tombstones | device_id、ack_sequence；entity_id、deleted_revision | 清理前考虑离线设备水位 |
| audit_events | actor、operation、object、decision、trace_id | 不含 secret 和完整私人上下文 |

表名是逻辑设计，实施可以合理合并，但不能损失状态区分、唯一约束和归属规则。索引以查询和领取语句为依据，详见 [性能计划](10-performance-plan.md)。

## 3 状态机

| 对象 | 状态 | 合法转换与限制 |
| --- | --- | --- |
| 待办 | open、in_progress、blocked、done、cancelled | done 可显式重新打开；归档和删除为独立属性 |
| 提案 | draft、ready、executing、applied、partially_applied、rejected、stale | 基础对象变化后变 stale；部分执行保存逐项结果 |
| 委派 | draft、queued、active、needs_attention、awaiting_review、accepted、cancelled | 一个委派可产生多个 Run；accepted 由验收动作决定 |
| 运行 | queued、starting、running、waiting_approval、waiting_input、cancel_requested、succeeded、failed、cancelled、interrupted、unknown | 终态不复活；重试创建新 attempt；unknown 先对账 |
| 审批 | pending、approved、denied、expired、superseded | 同一审批只能决定一次；批准仍要检查权限是否已撤销 |
| 投递 | queued、sending、accepted、delivered、read、failed、unknown、suppressed | 只有提供方有相应回执才能提升至 delivered/read |
| 节点任务 | pending、leased、executing、reporting、closed、lost | 租约过期后不能直接认定工具未执行 |

运行 succeeded 表示执行器报告并确认一次运行成功；委派 accepted 表示用户或预定义验收规则接受结果；待办 done 表示业务目标完成。三者不自动互相代替。

## 4 调度与一致性

业务创建提醒时，同事务保存目标、提醒、下一触发点与 Outbox。调度器以 due_at 索引定位，条件更新取得租约，生成有稳定 occurrence_key 的通知意图或执行请求。租约携带 authority_epoch 与单调递增的 fencing token，旧领取者不能提交新的正式结果。

数据提交与外部调用不是同一事务。外部动作使用提供方支持的幂等键；无此能力时，先保存发送尝试，超时后进入 unknown，查询对方状态或等待人工判断，不能直接重试发送邮件或再次提交编码任务。

目标是“至少一次调度，加上业务去重与副作用对账”。不得宣称对所有外部平台实现严格 exactly-once。输出日志、网络连接或进程退出不能替代持久成功记录。

长期操作采用明确取消信号；取消请求也持久化。CoreHost 重启后先对账运行、节点租约和投递，再决定补偿。没有可靠执行状态时标记 interrupted/unknown，并保留已经产生的产物。

## 5 时间与重复

| 情况 | 规定 |
| --- | --- |
| “周五前” | 解析为日期型截止，展示具体日期；不默认为某个分钟级时点 |
| 每月 31 日 | 默认跳过无该日期月份；“每月最后一天”是另一条明确规则 |
| DST 不存在时刻 | 普通重复事件按采用的标准语义处理；提醒可选择跳过或顺延并在预览中显示，不能静默混用 |
| DST 重复时刻 | 保存所选偏移或明确策略，默认只生成一个实例 |
| 跨时区旅行 | 事件按事件时区保存，界面按显示时区转换；“当地每天九点”需要独立浮动语义 |
| 全天事件 | 保存原始日期及排他结束日期，展示不因设备时区变化而漂移 |
| 只修改本次 | 保存以原始 occurrence_key 定位的例外 |
| 修改本次及以后 | 分割系列，保留旧系列历史和新系列起点 |
| 重复待办 | 区分固定日历节奏与完成后间隔；一条规则只选择一种生成策略 |

时间处理依赖注入可控时钟，避免用真实等待测试重复与错过触发。

## 6 业务接口目录

计划中的 HTTP 前缀为 /api/v1。这里列出行为，实际路由设计在对应阶段生成并版本化，不在文档阶段生成 OpenAPI 或客户端代码。

| 资源 | 操作 | 必须体现的契约 |
| --- | --- | --- |
| tasks、projects、inbox | 创建、分页查询、更新、完成、恢复 | operation_id、revision、空间权限 |
| calendar/range | 有限时间范围实例查询 | 时区、范围上限、日历筛选 |
| events/series/occurrences | 创建、改期、单次例外、系列分割 | 外部只读保护与版本冲突 |
| reminders、automations | 保存、预览、启停、手动触发 | 重叠策略、错过策略、预算 |
| conversations/messages | 保存消息、分页读取、关联上下文 | 消息序号、附件引用、来源 |
| proposals | 预览、执行、拒绝 | 基础版本、逐项授权与结果 |
| delegations/runs | 提交、查询、取消、创建重试尝试 | 202 与稳定运行 ID；不能超时后盲重建 |
| approvals | 读取、决定 | 决策人、内容摘要、时效、单次消费 |
| integrations | 能力探测、连接测试、配置应用 | 凭据引用，读写能力分离 |
| devices/nodes | 配对、能力、撤销、领取、回报 | 节点身份、租约、fencing token |
| artifacts | 元数据、授权读取、删除申请 | 大小、摘要、节点位置与保留策略 |
| sync/changes | 按游标增量拉取 | 权限过滤、墓碑、游标过期重新快照 |
| events | 订阅变更与运行通知 | SSE 序号、断点、重放范围、心跳 |
| health、diagnostics | 轻量健康、详细诊断 | 公共存活信息与受限诊断分开 |

命令统一返回操作 ID、对象 ID、新 revision 或运行 ID，以及可理解的错误。错误分为未认证、无权限、版本冲突、输入无效、能力不支持、过载、依赖不可用和外部结果未知。UI 应据此显示具体动作，不能全部简化成“请求失败”。

## 7 事件契约

事件至少包括 event_id、space_id、sequence、entity_id/revision、event_type、occurred_at、operation_id、trace_id；运行事件另带 run_id 和 run_sequence。正文只保留订阅者所需的最少数据，不在事件流广播密钥或整个配置。

客户端只按服务端 sequence 前进；重复事件忽略，发现间隙则补拉。SSE 只负责传输已存在状态，终态以持久记录为准。队列满时可合并 UI 进度，但审批、状态变更和结果事件不能丢弃。

初期保留近期可重放事件，过期游标返回明确的重新快照要求。日志保留期、事件保留期与审计保留期分别配置。

## 8 离线与同步范围

首个 LAN 版本允许缓存读取、离线快速捕获和新待办草稿，禁止离线审批、外部发送和触发自动化。离线更新已存在对象可后续开放，但必须提交基础 revision，冲突进入用户可理解的比较界面。

简单独立字段可以按明确规则合并；截止日期、完成状态、日程时间和审批不做静默“最后写入胜出”。设备重新上线先拉取变化，再提交待同步命令；确认后清除本地待同步标识。

同一空间从本机迁往 LAN 主机时停止原调度、备份、校验导入、提升 authority_epoch、重新绑定设备，最后启用新调度。没有自动多主或数据库文件同步。

迁移还需给旧主机持久写入 retired 标记并禁用旧服务。权威代次只约束参与校验的节点，不能单独阻止旧主机直接向外部平台发消息；必要时撤销旧主机专用的渠道/执行凭据。恢复历史备份默认进入待激活状态，不立即启动外部副作用。

## 9 保留与恢复

附件以内容摘要或稳定对象 ID 存放，数据库保存引用计数和归属；删除经过宽限期，避免备份恢复后引用缺失。会话原文、AI 使用记录、审计与诊断日志分别设保留策略。

备份采用 SQLite 支持的备份方式，不能只拷贝仍在写入的主数据库文件而忽略 WAL。[SQLite Backup API](https://www.sqlite.org/backup.html)

恢复必须验证数据库完整性、迁移版本、附件摘要、空间 ID 和权威身份；凭据采用独立恢复流程，不能把某台机器的加密内容直接当成跨机可用密钥。
