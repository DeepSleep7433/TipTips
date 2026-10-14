# 完整模块规格

模块清单覆盖完整产品，不代表一次性全部实施。每个 Module 要通过有限的 Interface 提供完整行为，UI、AI、邮件和桌宠复用相同规则。这里的路径与 Interface 名称是后续实施约定，尚未创建代码。

## 1 模块总览

| 编号 | Module | 主要数据 | 首次交付阶段 |
| --- | --- | --- | --- |
| M01 | 原生桌面与用户会话 | 窗口状态、设备偏好 | S1 |
| M02 | 收件箱待办与项目 | InboxItem、Task、Project | S1 |
| M03 | 日历与时间块 | Calendar、Event、Occurrence | S2 |
| M04 | 持久调度与自动化 | Schedule、Trigger、Job | S2；AI 自动化在 S4 |
| M05 | 通知与投递 | NotificationIntent、Delivery | S2 |
| M06 | 对话与助理档案 | Conversation、Message、Profile | S3 |
| M07 | 行动提案与执行协调 | Proposal、Action、授权快照 | S3 |
| M08 | Hermes 接入 | EngineConnection、SessionMapping | S4 |
| M09 | 编码任务委派 | Delegation、Run、RunEvent | S5 |
| M10 | MCP 连接与工具 | McpServer、ToolDescriptor | S4 |
| M11 | 模型提供方 | Provider、ModelProfile、Usage | S3 |
| M12 | 技能与扩展目录 | Skill、ExtensionManifest | S4 |
| M13 | 记忆检索与回顾 | MemoryItem、Source、Index | S4 |
| M14 | 配置与凭据 | SettingRevision、SecretReference | S1 基础；S3 扩充 |
| M15 | 邮件与双向渠道 | ChannelConnection、InboundMessage | S6 |
| M16 | 桌宠 | PetProfile、PetPreference | S7 |
| M17 | 局域网设备与节点 | Device、Node、Capability | S8 |
| M18 | 数据持久化与同步 | Operation、Outbox、ChangeCursor | S1；远程同步在 S8 |
| M19 | 授权审批与审计 | Grant、Approval、AuditEvent | S1 基础；S4 扩充 |
| M20 | 安装更新与恢复 | Version、Migration、Backup | S1 基础；S9 完善 |
| M21 | 诊断与性能 | Health、Metric、DiagnosticBundle | S1 持续建设 |
| M22 | 工作区与产物 | WorkspaceBinding、Artifact | S5 |

阶段定义见 [交付路线](11-delivery-roadmap.md)。

## 2 M01 原生桌面与用户会话

**行为规则**：首次启动引导选择本机或已有主机；窗口关闭后后台是否继续由明确设置决定。托盘、主窗口和桌宠的退出互不混淆。快捷键冲突时给出替代，不强行占用。

**Interface 与数据链路**：DesktopActivation 负责打开目标页面、转发通知动作和单实例激活；SessionPresence 只报告在线、锁屏、空闲等必要状态。主窗口通过业务 Interface 获取数据。

**实现位置**：Desktop、SessionAgent 和平台 Adapter。单实例标识按个人空间与用户会话作用域区分，避免不同账号互相激活。

**性能与验收**：主窗口关闭后不保留整个视觉树；接收提醒不能启动完整会话页。验证重启、通知激活、多显示器、快捷键冲突、后台退出语义和键盘操作。

## 3 M02 收件箱待办与项目

**行为规则**：支持快速记录、标题描述、优先级、标签、项目、子任务、依赖、截止日期、预计耗时、附件、排序、归档、软删除和恢复。依赖不能形成环；完成父任务时显示未完成子项，不默默完成它们。

**Interface 与数据链路**：TaskCommands 处理创建、更新、完成、恢复、移动及批量操作；TaskQueries 提供今天、即将到期、项目、筛选和搜索的分页结果。输入 → 校验 → 对象版本更新 → 提醒重排 → Outbox。

**实现位置**：Domain/Tasks、Application/Tasks、Infrastructure/Persistence。项目模板和固定过滤视图在基础闭环稳定后加入。

**性能与验收**：查询使用状态、日期、项目索引；批量编辑限制批量大小。验证无日期任务、撤销完成、依赖环、重复请求、并发冲突和一万条任务的筛选。

## 4 M03 日历与时间块

**行为规则**：内置日历支持日、周、月、议程和任务时间块。截止日期仅以标记显示，不自动占满一天。支持全天、跨天、重复实例例外、时区和外部只读事件。

**Interface 与数据链路**：CalendarCommands 处理安排与改期；CalendarRangeQuery 按明确起止范围返回实例。规则与例外 → 有限范围展开 → 冲突计算 → UI；外部变更经过同步映射进入业务层。

**实现位置**：Domain/Calendar、Application/Calendar、Integrations/Calendars。时间库和重复规则库在 S0 验证，禁止手写完整日历协议。

**性能与验收**：只展开当前范围和少量预取，避免无限系列物化。验证夏令时、31 日重复、跨天、修改单次、外部取消与双向更新。

## 5 M04 持久调度与自动化

**行为规则**：提醒、周期回顾和用户自动化都由正式数据库记录；支持启停、下次触发预览、历史、最大运行时间、重叠策略和错过触发策略。定时等待不调用 LLM。

**Interface 与数据链路**：ScheduleCommands 管理规则；Scheduler 领取到期 Job 并产生业务动作或 Run。领取租约、执行状态、重试时点都可恢复。

**实现位置**：Application/Scheduling、Infrastructure/Jobs。先实现最近触发计时器与数据库领取；不把系统内存计时器作为唯一事实来源。

**性能与验收**：最近到期索引、批量领取、有界并发；待机无高频全表扫描。验证核心崩溃、休眠恢复、时钟调整、租约过期和重复触发。

## 6 M05 通知与投递

**行为规则**：一个业务事实形成一个通知意图；渠道按优先级、勿扰、合并窗口和设备策略投递。支持延后、已处理、打开对象；已读不等于完成任务。

**Interface 与数据链路**：NotificationPolicy 选择渠道；DeliveryAdapter 发送并返回实际可用的回执。意图 → 投递记录 → 渠道 → 回执/未知结果 → 对账。

**实现位置**：Application/Notifications、SessionAgent/Notifications、Integrations/Delivery。模板按语言与隐私级别生成。

**性能与验收**：重复投递键、失败退避、摘要合并；不为每个 token 推送。验证同一提醒多设备、勿扰、限流、渠道超时与动作重复点击。

## 7 M06 对话与助理档案

**行为规则**：支持会话分组、搜索、关联项目、消息引用、附件、模型选择、停止生成和会话归档。助理档案选择角色与能力；模型切换不静默丢弃上下文。

**Interface 与数据链路**：ConversationCommands 保存用户消息；AssistantSession 选择直连模型或 Hermes 并返回统一事件。运行与对话分开，单个会话初期只允许一个改变会话上下文的活跃运行。

**实现位置**：Application/Conversations、Desktop/Assistant。历史消息分页，正文按需读取；Markdown 用原生渲染且禁用可执行 HTML。

**性能与验收**：可见消息虚拟化、分块存储、摘要缓存；验证长历史、途中停止、断流、重新打开会话与含大附件输入。

## 8 M07 行动提案与执行协调

**行为规则**：自然语言动作先生成结构化提案，包含对象、变化、依据与权限。明确授权内的低风险动作可直接执行；存在歧义或超出范围时再询问。撤回支持程度必须按动作说明。

**Interface 与数据链路**：ActionPlanner 生成提案；PolicyDecision 校验；ActionExecutor 调用已有业务 Interface。提案保存基础版本，真正执行前再次检查对象和权限。

**实现位置**：Application/AssistantActions。待办、日程、渠道发送和委派都提供有限业务动作，禁止通用“写数据库”工具。

**性能与验收**：简单命令优先确定性解析；模型只获取必要上下文。验证提案过期、用户先行修改、批量部分失败、取消与重放。

## 9 M08 Hermes 接入

**行为规则**：支持远程 Hermes 和用户配置的本机独立 Hermes；显示连接版本、能力、宿主位置与真实状态。TipTips 不接管用户现有 Hermes 数据目录。

**Interface 与数据链路**：HermesAdapter 负责能力探测、会话映射、启动运行、事件归一、审批转发和停止；配置管理作为另一个可选管理能力。

**实现位置**：Integrations/Hermes。优先实际存在的能力与运行 API，必要时再评估 ACP；不能靠模拟键盘读取终端画面。

**性能与验收**：懒连接、共享 HTTP 连接、运行事件持久化、断线对账。验证拒绝授权、缺能力、引擎重启及工具实际执行主机。

## 10 M09 编码任务委派

**行为规则**：Codex 与 Claude Code 分别作为 Adapter。委派必须包含目标、项目、工作区、节点、读写范围、截止、预算和产物要求。成功执行后进入待验收。

**Interface 与数据链路**：DelegationCommands 创建目标；ExecutionCoordinator 选择节点；RunnerAdapter 启动、续接、取消和收集事件。进程 ID 不能作为唯一恢复标识。

**实现位置**：Application/Delegations、ExecutionNode/Runners、Integrations/Codex、Integrations/ClaudeCode。

**性能与验收**：默认一个写工作区只有一个活跃写入者；并发分析和不同工作区有全局预算。验证运行崩溃、审批、孤儿进程、超时、停止、结果与修改清单。

## 11 M10 MCP 连接与工具

**行为规则**：支持本地 stdio 和远程 Streamable HTTP。工具发现、授权与执行是三个阶段；描述中的只读标记不能独自决定授权。导入配置先预览差异。

**Interface 与数据链路**：ToolCatalog 返回有来源与版本的能力；ToolInvocation 处理输入校验、授权、调用、大小限制和结果归一。

**实现位置**：Integrations/Mcp、Application/ToolPolicy。按机器作用域管理可执行文件、参数、环境白名单和凭据引用。

**性能与验收**：工具描述缓存、按需进程、每连接并发限制和有界输出。验证工具列表变化、恶意超大输出、超时、进程退出和 OAuth 过期。

## 12 M11 模型提供方

**行为规则**：区分模型 API 与 Agent 执行器；提供方、协议、模型标识和能力分别配置。直连模式控制工具循环；Hermes 模式由 Hermes 控制其内部循环。

**Interface 与数据链路**：ModelAdapter 输出统一文本、工具请求、用量与结束原因。ModelRouter 按用户选择、隐私、能力和预算决定可用目标，无法满足时显式返回。

**实现位置**：Integrations/Models、Application/ModelRouting。至少验证 OpenAI、Anthropic 与一个本地端点。

**性能与验收**：长连接、上下文预算、取消、令牌限额；不默认并行请求多个模型“选最好”。验证不支持工具的模型、429、流中断和用量缺失。

## 13 M12 技能与扩展目录

**行为规则**：技能是带来源的指令与资源；MCP 是工具协议；原生功能扩展是另一类对象。安装、启用、版本固定、升级差异和卸载分别操作。

**Interface 与数据链路**：ExtensionCatalog 管理目录；SkillResolver 只把当前运行授权的资源加入上下文。自动生成的技能先进入草稿区。

**实现位置**：Application/Extensions、Integrations/SkillPackages。外部执行部分通过独立进程，而非在 CoreHost 加载第三方程序集。

**性能与验收**：延迟读取技能正文、按关键词/目标筛选，限制数量与上下文大小。验证路径越界、缺依赖、版本变更和禁用后不再生效。

## 14 M13 记忆检索与回顾

**行为规则**：记忆有来源、形成时间、确认状态、适用范围、过期规则；用户能纠正、删除和禁止某类记忆。会话总结与正式记忆分开。

**Interface 与数据链路**：MemoryQueries 检索；MemoryCommands 受控写入；ReviewGenerator 从任务和日历的实际数据形成回顾。未完成任务不能因叙述被改为完成。

**实现位置**：Application/Memory、Infrastructure/Search。全文检索先行，向量嵌入后置并可关闭。

**性能与验收**：增量索引、有限候选集、低优先级嵌入任务；中文检索必须实测分词和召回。验证删除传播、错误记忆修订、权限变化和索引重建。

## 15 M14 配置与凭据

**行为规则**：配置有默认、空间、设备、Profile 和运行快照作用域；凭据只保存引用。修改展示影响范围、校验结果和是否需重启。

**Interface 与数据链路**：SettingsCommands 校验并形成版本；SecretStore 只向获准 Adapter 提供必要值；启用前可测试连接。

**实现位置**：Application/Settings、Infrastructure/Secrets、Desktop/Settings。服务账号与登录用户分别拥有凭据上下文。

**性能与验收**：配置按版本缓存，更新精确失效；不每次工具调用解析全量配置。验证脱敏导出、撤销凭据、服务重启和错误配置回滚。

## 16 M15 邮件与双向渠道

**行为规则**：先绑定账号和可信发送者，入站消息先去重保存。回复可以延后、完成或继续已关联会话；敏感操作必须通过可信认证确认。

**Interface 与数据链路**：InboundChannelAdapter → 入站归一 → 身份/关联判定 → 提案或业务命令；OutboundChannelAdapter 只接受已授权投递。

**实现位置**：Integrations/Mail、Integrations/Push、Application/ChannelActions。邮箱推送能力与轮询能力分开检测。

**性能与验收**：增量游标、后台长轮询/推送、指数退避；限制附件和正文大小。验证重投、伪造 From、自动回复循环、退信和撤销授权。

## 17 M16 桌宠

**行为规则**：桌宠提供快速输入、今日任务、下一提醒与运行状态；用户可拖动、隐藏、静音、关闭和更换外观。它不自行授予执行权限。

**Interface 与数据链路**：PetStateSubscription 接收精简事件；PetAction 转发到主系统；私密详情由主窗口展示。

**实现位置**：Pet、SessionAgent/PetControl。首期只支持有来源的静态与序列帧资源，后续再考虑 Live2D 等渲染引擎。

**性能与验收**：静止无刷新、空闲低帧率、活动限帧、锁屏停止；透明窗口和多屏 DPI 在原生环境验收。

## 18 M17 局域网设备与节点

**行为规则**：设备配对需短期一次性挑战与主机指纹；连接可撤销。设备访问数据权限和节点执行权限分开。

**Interface 与数据链路**：DevicePairing 建立信任；NodeRegistration 上报能力；NodeLease 分发绑定到具体节点的运行。节点主动出站连接主机，避免要求每台客户端开入站端口。

**实现位置**：CoreHost/Devices、ExecutionNode/Connectivity、Desktop/HostConnection。

**性能与验收**：心跳合并与退避、能力缓存、事件续传；测试离线、撤销、证书变化、重复领取和旧租约回报。

## 19 M18 数据持久化与同步

**行为规则**：核心是唯一正式写入者；每个更新携带基础版本与操作标识。软删除保留墓碑供同步，备份与恢复维持对象身份。

**Interface 与数据链路**：业务事务包含实体与 Outbox；SyncFeed 提供有序变化；客户端缓存只从确认结果更新正式版本。

**实现位置**：Infrastructure/Persistence、Application/Sync。迁移版本与应用版本分开，恢复到独立位置验证后再切换。

**性能与验收**：短事务、按键分页、有限缓存、附件分离。验证断电恢复、磁盘满、损坏备份、并发更新和大历史查询。

## 20 M19 授权审批与审计

**行为规则**：授权包含主体、空间、节点、对象范围、动作、有效期和预算。审批绑定不可变操作摘要，参数或目标变化后旧审批失效。

**Interface 与数据链路**：PolicyDecision 先裁决；ApprovalCommands 等待可信用户决定；AuditWriter 记录关键副作用及来源，避免保存凭据正文。

**实现位置**：Application/Policy、Application/Approvals、Infrastructure/Audit。关键权限不能只在 UI 按钮上判断。

**性能与验收**：权限可按版本短缓存，撤销能立即使缓存失效。验证跨空间访问、过期审批、重复确认和提示注入内容无法扩大权限。

## 21 M20 安装更新与恢复

**行为规则**：安装分普通用户与服务模式；更新先校验签名，再排空运行并备份。卸载默认保留用户数据，删除数据需用户明确选择。

**Interface 与数据链路**：UpdatePlan 描述组件版本与依赖；MigrationRunner 互斥执行迁移；BackupService 提供可恢复快照。

**实现位置**：packaging、Infrastructure/Updates、CoreHost/Maintenance。

**性能与验收**：增量或组件更新在可信验证后采用；备份限速且避开交互高峰。验证中断升级、降级限制、清洁系统安装和恢复演练。

## 22 M21 诊断与性能

**行为规则**：诊断页显示核心、节点、引擎、渠道和存储健康，以及用户可操作的故障原因。每次运行关联统一追踪标识。

**Interface 与数据链路**：HealthSnapshot 提供轻量读数；DiagnosticExport 输出脱敏日志与版本；Metrics 收集延迟、队列、资源与错误计数。

**实现位置**：Infrastructure/Diagnostics、Desktop/Diagnostics。OpenTelemetry 可作为实施候选，默认不向外部发送遥测。

**性能与验收**：不采集高基数消息正文标签，日志轮转限额；验证长时间运行无增长失控、日志导出脱敏及指标可用于定位瓶颈。

## 23 M22 工作区与产物

**行为规则**：项目通过节点绑定目录；路径必须规范化并检查重解析点、软链接和挂载边界。产物记录来源、内容摘要、实际位置与访问范围。

**Interface 与数据链路**：WorkspaceGrant 允许执行；ArtifactRegistry 接收结果元数据；ArtifactAccess 提供授权下载/本机打开。文件缺失时保留历史但显示不可用。

**实现位置**：ExecutionNode/Workspaces、Application/Artifacts。需要隔离写入时使用专用目录或 Git worktree；非 Git 项目不能假设存在版本回滚。

**性能与验收**：流式传输、大文件不进入 SSE 或数据库正文；变更摘要限制大小。验证两个执行器竞争目录、越界路径、大产物取消与产物来源追踪。
