# 系统架构与技术选择

建议采用原生 Windows 客户端、跨平台核心主机、轻量用户会话代理和按需执行节点。业务 Module 先组成一个可部署核心，外部引擎与高风险执行在独立进程中运行。

## 1 技术基线

| 层次 | 建议技术 | 选择原因 |
| --- | --- | --- |
| Windows 主窗口 | C#、.NET 10 LTS、WPF、MVVM | 原生桌面窗口，成熟的数据绑定、输入、可访问性与窗口控制 |
| 原生桌宠 | 独立 WPF 透明窗口进程 | 与主窗口分离，可单独退出和节能 |
| 会话代理 | 小型 .NET Windows 进程，按需使用 Win32/通知 API | 托盘、全局快捷键、通知激活不依赖主窗口存活 |
| 核心主机 | .NET Generic Host、ASP.NET Core | 同一业务核心以用户后台、Windows Service 或 macOS daemon 方式承载 |
| 持久化 | SQLite、WAL、显式事务、版本迁移 | 个人与小规模 LAN 场景低部署成本，数据由一个核心拥有 |
| 查询 | Microsoft.Data.Sqlite；复杂读取可采用轻量映射 | SQL 和索引可审查；避免把 ORM 对象跟踪带入长会话热点 |
| 时间处理 | 成熟时区与重复规则库，实施时验证 Noda Time/Ical.Net 等候选 | 避免自写 DST 与完整 RRULE 解释器 |
| 进程间与 LAN 通信 | HTTPS JSON 请求与 SSE 事件；本机可用受 ACL 保护的命名管道 Adapter | 命令与事件契约一致，承载差异集中在通信层 |
| 扩展能力 | MCP 协议、HTTP Adapter、子进程协议 | 主程序不加载不可信第三方 DLL |
| 安装 | Windows 签名安装器，核心服务作为可选安装项；macOS 独立服务包 | 区分普通用户安装和需要系统权限的服务安装 |

.NET 10 是当前文档基线的 LTS 版本，实际 SDK 补丁在实施时锁定到支持中的版本。[Microsoft 支持策略](https://dotnet.microsoft.com/en-us/platform/support/policy)

WPF 的支持范围是 Windows；跨平台目标由无 UI 核心承担，不能把“服务支持 macOS”写成“WPF 界面支持 macOS”。[WPF 官方说明](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/overview/)

## 2 可选方案的取舍

| 方案 | 适用性 | 当前决定 |
| --- | --- | --- |
| WPF 与 .NET | 当前 Windows 界面、托盘、桌宠需求明确；核心可独立跨平台 | 采用 |
| WinUI 3 | Windows 视觉与平台集成良好；多窗口透明桌宠和部署需额外验证 | 保留为 UI 技术验证替代项，不同时维护两套主 UI |
| Avalonia | 适合同时要求 Windows/macOS/Linux 桌面 UI，非 Web UI | 若以后 Mac GUI 成为硬需求，再评估迁移成本 |
| Qt/C++ | 可原生跨平台、可精细控制资源；开发与内存安全成本更高 | 当前不为了未经测量的性能收益重写全部核心 |
| Electron/Tauri/WebView 业务 UI | 便于复用 Hermes 前端 | 不满足当前界面约束 |

进程少、启动快、资源有界，比简单选择“更底层语言”更能保证本产品日常体验。只在分析器证明瓶颈且收益明确时引入专用原生库。

## 3 进程职责

| 进程 | 拥有的责任 | 生命周期 | 不承担的工作 |
| --- | --- | --- | --- |
| TipTips.Desktop | 主窗口、列表、日历、会话、设置和审批界面 | 用户打开时运行；关闭后释放 | 正式数据写入、持久调度 |
| TipTips.SessionAgent | 托盘、快捷键、Toast、激活转发、在线设备状态 | 登录后可运行 | 模型推理、业务数据库直接写入 |
| TipTips.CoreHost | 业务 Module、数据库、调度、权限、通知意图、运行协调 | 本地用户后台或系统服务 | 直接展示桌宠或服务会话 UI |
| TipTips.Pet | 渲染桌宠、气泡与快捷动作 | 用户启用时启动 | 直接访问凭据、工具、业务数据库 |
| TipTips.ExecutionNode | 工作区授权、执行器与本机工具生命周期 | 启用本机执行能力后启动 | 决定业务待办是否完成 |
| Hermes/Codex/Claude/MCP 子进程 | 对应供应方或工具的执行 | 有任务时启动，依能力空闲退出 | 充当 TipTips 正式数据源 |

不把每个 Module 拆成常驻进程。执行节点在第一阶段可以是 CoreHost 内受控的宿主能力；跨机执行和独立凭据隔离需要出现时再部署为独立进程，保持同一 Interface。

Windows Service 运行在非交互会话，不能直接承担桌面通知和桌宠；由登录用户的 SessionAgent 处理。Windows 后台宿主使用官方 Worker/Service 集成路径。[Windows Service 指南](https://learn.microsoft.com/en-us/dotnet/core/extensions/windows-service)

## 4 三种拓扑

**本机个人模式**：Desktop、SessionAgent、可选 Pet → 本机 CoreHost → 本机 SQLite；引擎按需在本机或远端运行。

**局域网客户端模式**：Windows Desktop 与 SessionAgent → 经认证的 LAN CoreHost → 主机本地 SQLite；当前电脑的 ExecutionNode 主动连接主机，领取允许的工作。

**无界面主机模式**：Windows Service 或 macOS launchd → CoreHost → 调度、渠道和远程节点；没有登录桌面时仍可发送邮件和服务器推送，但不能在无人登录的会话里显示宠物。

CoreHost 的宿主模式可以替换，领域和业务应用层不引用 WPF、Win32 或 launchd。

## 5 数据权威与调度权威

每个个人空间只绑定一个权威主机；客户端只持有带版本的缓存和待同步意图。LAN 多台电脑不共同打开一个共享 SQLite 文件。SQLite WAL 要求访问者位于同一主机，不适合将数据库放到网络共享中。[SQLite WAL](https://www.sqlite.org/wal.html)

业务数据和 Outbox 事件在同一事务提交。提醒/自动化由权威主机领取租约后调度；本机 Toast 由渠道规则选出的设备呈现，避免每台设备都主动重复提醒。

主机离线不自动产生第二个调度主机。主机迁移是显式停写、导出、导入和重新绑定操作。远程空间不自动“降级”为本机正式数据库。

## 6 Module 与 Interface

Module 隐藏一个完整行为的复杂度，Interface 包含调用方式、权限、幂等性、错误和资源限制。只有确实变化的执行器、模型提供方、通知渠道、存储访问和平台集成位置才设置 Adapter。

业务 UI 调用任务 Interface 后，任务校验、版本冲突、提醒重排和事件提交都由对应 Module 负责。不能让 UI、邮件与 Hermes 分别复制同样规则。

建议后续项目布局如下；这些只是规划路径，本阶段不创建工程。

| 规划位置 | 内容 |
| --- | --- |
| src/TipTips.Domain | 领域对象、值对象、不依赖 UI 的规则 |
| src/TipTips.Application | 业务 Interface、用例、权限和事务协调 |
| src/TipTips.Infrastructure | SQLite、凭据、HTTP、文件、诊断等 Adapter |
| src/TipTips.CoreHost | 宿主、服务入口、请求和事件端点 |
| src/TipTips.Desktop | WPF 页面、ViewModel、原生控件 |
| src/TipTips.SessionAgent | 托盘、通知、快捷键与激活 |
| src/TipTips.Pet | 桌宠渲染、主题与交互 |
| src/TipTips.ExecutionNode | 节点配对、工作区执行与进程监督 |
| src/TipTips.Integrations | Hermes、Codex、Claude、MCP、模型和渠道 Adapter |
| tests | 按业务契约、平台集成与性能场景组织 |
| packaging | 后续安装与服务发布资源 |

不为每个 Module 立即建立独立程序集；先以目录和依赖规则约束，出现真实隔离需要再拆。

## 7 主要数据链路

| 场景 | 处理链路 | 持久化时机 |
| --- | --- | --- |
| 新建待办 | 输入 → 任务 Interface → 校验 → 任务与提醒 → Outbox → UI 更新 | 返回成功前提交 |
| 自然语言整理 | 原始输入 → AI 提案 → 授权裁决 → 原有业务 Interface | 输入先保存；提案与落地结果分别保存 |
| 委派编码 | 委派 → 预算/授权 → 节点租约 → 执行器 → 运行事件 → 产物 → 待验收 | 调度前保存委派；关键状态同步落盘 |
| 到时提醒 | 持久触发点 → 领取 → 通知意图 → 渠道队列 → 发送 → 回执 | 意图和发送尝试均可恢复 |
| 跨机修改 | 带基础版本的命令 → 权威主机 → 并发裁决 → 有序事件 → 缓存 | 以主机提交为成功依据 |

## 8 关键架构取舍

采用模块化单体降低进程通信、部署、监控和资源成本。采用持久 Outbox 与数据库队列保证可恢复性，不在首版引入消息中间件。附件与运行日志独立存储，数据库保存元数据、索引和结构化事件。

业务状态属于 TipTips；引擎会话状态仍由各引擎维护，通过映射关联。持久任务不依赖某个 SSE 连接存活；流断开进入重连/对账，不能直接判作失败。

安全与性能同时约束设计：权限范围在每次副作用前复查，预算在调用前预留，任务过载时排队或拒绝，不通过无限并发提高表面吞吐。

相关决定：[原生客户端](adr/0001-native-windows-client.md)、[单一权威核心](adr/0002-single-authority-core.md)、[Hermes 独立引擎](adr/0003-hermes-external-engine.md)。
