# TipTips

TipTips 是面向个人的原生 Windows 助理应用规划，统一待办、日历、AI 会话、自动化、应用连接和编码任务委派。它计划提供可安装客户端、静默后台、可选桌宠，以及能够部署在 Windows 或 macOS 上的局域网服务。

**当前状态：架构与产品设计阶段，仅有文档。尚未实现应用、安装包、接口或性能指标。**

## 产品方向

- 原生 Windows 界面，建议采用 C#、.NET 10 LTS 和 WPF；主界面与桌宠不使用浏览器容器。
- 待办、日历和提醒在没有大模型时仍可使用；AI 帮助收集、拆解、安排和执行任务。
- Hermes 通过独立 Adapter 接入，保留其助理、技能与工具能力；TipTips 拥有自己的业务数据与权限规则。
- Codex 和 Claude Code 作为编码执行器，任务有明确目录、权限、预算、运行状态和结果。
- 支持 OpenAI、Anthropic 和本地模型，并按真实能力选择协议和功能。
- 通过 Windows 通知、邮件、可配置推送渠道和桌宠与用户交互。
- 单机使用与局域网使用共享业务核心；同一个个人空间只有一个权威数据主机。
- 性能是架构约束：后台低占用、重模块按需加载、有界队列、按范围查询、虚拟化显示和分阶段性能验收。

## 阅读顺序

| 文档 | 解决的问题 |
| --- | --- |
| [产品范围与行为规则](docs/01-product-scope.md) | 为谁做、需要完成什么、版本范围如何划分 |
| [系统架构与技术选择](docs/02-system-architecture.md) | 为什么选原生 .NET，进程与部署如何拆分 |
| [完整模块规格](docs/03-module-specifications.md) | 各模块的责任、实现顺序、输入输出和验收 |
| [领域数据与接口契约](docs/04-domain-data-contracts.md) | 数据归属、状态机、事务、接口与同步规则 |
| [Hermes 与编码任务委派](docs/05-hermes-and-agent-delegation.md) | 如何实际接入 Hermes、Codex、Claude Code |
| [MCP 模型与配置中心](docs/06-mcp-models-and-settings.md) | 工具、模型、凭据、技能和配置如何管理 |
| [日历通知与桌宠](docs/07-calendar-notifications-and-pet.md) | 时间语义、邮件回复、推送、提醒与桌宠交互 |
| [原生界面与交互规格](docs/08-native-ui-specification.md) | 页面、导航、状态、快捷操作与可访问性 |
| [部署安全与运维](docs/09-deployment-security-and-operations.md) | 安装、后台运行、局域网、备份、升级和恢复 |
| [性能架构与验收预算](docs/10-performance-plan.md) | 优化策略、资源预算、测量口径与验收门槛 |
| [分阶段交付路线](docs/11-delivery-roadmap.md) | 阶段依赖、工作包、完成标准和停止点 |
| [sol6.1 max 执行交接](docs/12-sol61max-handoff.md) | 后续执行模型从哪里开始、如何逐步交付 |
| [依据与待验证事项](docs/13-evidence-and-decisions.md) | 已核实资料、参考版本、假设和能力验证清单 |
| [领域术语](CONTEXT.md) | 待办、日程、委派和运行等概念的统一含义 |
| [架构决定](docs/adr/0001-native-windows-client.md) | 原生客户端、权威核心、外部引擎的取舍 |

推荐先阅读 01、02、10、11，再按实现阶段阅读对应专题。执行入口是 12，但阅读交接文档不代表自动授权开始开发。

## 本阶段的决定

Windows 客户端优先；macOS 首先作为无界面服务主机。业务核心使用模块化单体，SQLite 作为单主机存储，MCP、模型和执行器通过窄 Interface 接入。Hermes 的 Python 运行时属于独立可选引擎，TipTips 自身的界面、业务核心、调度和安装流程不以 Python 脚本组织。

这些是供实施采用的设计基线。性能数字是待测目标；外部执行器和服务能力需要在具体版本、操作系统和账号下完成契约验证。

## 贡献与许可

项目所有者与需求提出者：[DeepSleep7433](https://github.com/DeepSleep7433)。

**Codex** 参与本阶段的产品拆解、参考资料核实、系统架构、性能方案和文档编写，详见 [贡献者说明](CONTRIBUTORS.md)。

本仓库采用 [MIT License](LICENSE)。第三方项目、模型、字体、图像和服务继续适用各自许可证与使用条款。

文档基线日期：2026 年 10 月 14 日。
