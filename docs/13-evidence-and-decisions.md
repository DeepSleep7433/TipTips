# 依据与待验证事项

文档基线日期为 2026 年 10 月 14 日。资料用于确定设计可行路径，运行能力最终以锁定版本的契约和目标设备验收为准。

## 1 证据级别

| 级别 | 含义 | 本阶段情况 |
| --- | --- | --- |
| 需求 | 用户明确提出的目标和约束 | 原生 Windows、后台、Hermes、MCP、委派、日历、消息、桌宠、LAN、多模型、性能、MIT 公开项目 |
| 静态核实 | 阅读文件、官方文档或源码 | 已进行 |
| 设计决定 | 基于目标与证据提出的方案 | 已形成，未实现 |
| 运行验证 | 在具体实例执行接口并检查结果 | 应用集成未进行 |
| 性能实测 | 固定环境下测量实际应用 | 未进行 |
| 平台验收 | 安装、原生 UI、服务和桌宠实际操作 | 未进行 |

Git 仓库发布是文档交付行为，不代表软件已经完成或发布安装包。

## 2 Hermes 本地参考

只读检查的对象是已有安装包中不含个人业务内容的生成说明、manifest、静态前端和运行包文件目录；没有读取用户的聊天历史、业务数据库、凭据或环境密钥。

| 相对位置 | 核实事实 |
| --- | --- |
| dashboard/README.generated.txt | 前端构建来源提交 b9f82a21fd9268bd499051e23b614ecb06d5b888 |
| bundled-runtime/README.generated.txt | 核心仓库 Eynzof/Hermes-CN-Core，运行时 0.21.0-cn.18 |
| bundled-runtime/stable-win32-x64.json | win32/x64 运行包与同一来源提交 |
| dashboard/web_dist/assets | 会话、模型、Profile、MCP、Skills、Cron、渠道、配对、Webhooks、诊断等构建资源 |
| 运行包文件目录 | 独立运行时可执行文件、Python 动态库、相关依赖许可 |
| 运行包 Hermes LICENSE | 上游 Hermes 核心包含 MIT 许可；不能据此推断所有资源和依赖均为同一许可 |

这里只公开必要的版本与相对文件名，不公开用户本机路径、数据、账号配置或运行包副本。

## 3 官方与上游资料

| 编号 | 资料 | 支持的结论 |
| --- | --- | --- |
| E01 | [.NET 支持策略](https://dotnet.microsoft.com/en-us/platform/support/policy) | 选择 .NET 10 LTS 作为当前技术基线 |
| E02 | [WPF 概览](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/overview/) | Windows 原生桌面能力及平台范围 |
| E03 | [WPF 控件性能](https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/optimizing-performance-controls) | UI 虚拟化、数据虚拟化与容器回收的区别 |
| E04 | [Windows Service](https://learn.microsoft.com/en-us/dotnet/core/extensions/windows-service) | .NET 后台宿主的服务运行路径 |
| E05 | [Windows 通知](https://learn.microsoft.com/en-us/windows/apps/develop/notifications/) | WPF 通知入口和 WNS 前提需要分别处理 |
| E06 | [Apple launchd](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html) | 用户 Agent 与系统 daemon 生命周期不同；归档文档需在目标 OS 复核部署细节 |
| E07 | [SQLite WAL](https://www.sqlite.org/wal.html) | WAL、并发、检查点与网络文件系统限制 |
| E08 | [SQLite Backup](https://www.sqlite.org/backup.html) | 一致备份应使用受支持机制 |
| E09 | [Hermes CN Desktop](https://github.com/Eynzof/Hermes-CN-Desktop) | 参考桌面项目技术方向 |
| E10 | [Hermes CN Core](https://github.com/Eynzof/Hermes-CN-Core) | 社区核心项目和上游关系 |
| E11 | [Hermes 固定提交 API Server](https://github.com/Eynzof/Hermes-CN-Core/blob/b9f82a21fd9268bd499051e23b614ecb06d5b888/gateway/platforms/api_server.py) | 能力查询、运行、审批、停止和服务端执行语义 |
| E12 | [Hermes 上游](https://github.com/NousResearch/hermes-agent) | 助理引擎与生态参考 |
| E13 | [Codex App Server](https://developers.openai.com/codex/app-server) | 程序化会话与审批入口 |
| E14 | [Codex 非交互模式](https://developers.openai.com/codex/noninteractive) | 有限无人值守任务和结构化事件 |
| E15 | [Claude Code 程序化运行](https://code.claude.com/docs/en/headless) | 非交互输入、结构化输出和流 |
| E16 | [Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview) | 较完整的程序化 Agent 集成路径 |
| E17 | [MCP 传输](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) | stdio、Streamable HTTP 与协议生命周期 |
| E18 | [MCP 授权](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization) | 远程授权发现和资源访问规则 |
| E19 | [Ollama 兼容性](https://docs.ollama.com/api/openai-compatibility) | 本地兼容接口存在功能范围 |
| E20 | [Google Calendar 同步](https://developers.google.com/workspace/calendar/api/guides/sync) | 增量游标与失效处理 |
| E21 | [Microsoft Graph 日历增量](https://learn.microsoft.com/en-us/graph/delta-query-events) | 日历范围变化的增量同步 |
| E22 | [MIT License](https://opensource.org/license/mit) | 仓库采用的许可文本 |

表中资料是所列结论的依据，不是 TipTips 所有设计细节的外部证明。预算、队列策略、进程划分、同步范围和默认行为属于本项目设计。

## 4 已采用的设计基线

| 决定 | 原因 | 可改变的条件 |
| --- | --- | --- |
| WPF Windows 客户端与 .NET 核心 | 原生 Windows 优先，服务可分离跨平台 | 完整 Mac UI 成为硬需求或原生验证发现阻断 |
| 模块化单体 | 控制资源与部署复杂度 | 测量证实某模块需要独立扩缩或隔离 |
| 单空间单权威主机 | 避免多主同步和重复调度 | 明确需要离线多主并愿意承担冲突协议成本 |
| SQLite 主机本地存储 | 个人使用部署轻，负载可控 | 写入吞吐、多人规模或可用性要求超出实测范围 |
| Hermes 外部引擎 | 满足原生核心约束并保留真实能力 | 上游提供更合适嵌入契约，经过版本验证 |
| 本地基础功能不依赖 AI | 保持可靠、快速和离线可用 | 不应改变 |
| 桌宠独立可选 | 可关闭、节能、隔离 UI 资源 | 可优化宿主组织，但关闭必须释放资源 |

## 5 需在实施阶段回答的问题

| 优先级 | 问题 | 当前默认 | 验证/决定阶段 |
| --- | --- | --- | --- |
| 高 | 当前 Hermes 安装实例是否启用运行 API | 不自动启动/改配置，先测试已授权实例 | S0/S4 |
| 高 | Hermes 断连后运行是否继续、事件能否重放 | 不假定，采用查询与 unknown 状态 | S0/S4 |
| 高 | Claude 完整审批是否需 SDK 宿主 | CLI 做受限模式，必要时 Node 宿主 | S0/S5 |
| 高 | .NET 通知和透明窗口能否满足目标 | WPF + 官方通知路径 | S0 |
| 高 | 服务身份如何使用执行器认证 | 专用身份或用户执行节点 | S0/S8 |
| 高 | Windows/macOS 支持版本与安装许可 | 实施时锁定正式支持矩阵 | S0/S9 |
| 中 | 首个外部日历/邮箱/推送提供方 | 依据所有者实际账号选择 | S6 |
| 中 | 桌宠风格和资源 | 先有来源的静态/序列帧 | S7 |
| 中 | 离线编辑范围 | 缓存阅读和新记录，敏感操作在线 | S8 |
| 中 | 中文检索实现 | 全文优先，实测分词/n-gram | S4 |
| 中 | 真实硬件资源预算 | 按性能计划基线先测 | 各阶段 |

这些问题不妨碍当前文档完成。需要用户偏好或账号时，在相应实现阶段提出具体问题，不将所有未来选项变成当前的阻塞。

## 6 文档变更规则

需求变更先更新范围，架构取舍更新 ADR，接口变更更新契约，性能目标调整附测量依据。每阶段实际状态与规划分开维护。

本基线由 Codex 在项目所有者指导下编写；此后的实现、测试、打包、部署和外部集成结果应按真实完成情况补充。
