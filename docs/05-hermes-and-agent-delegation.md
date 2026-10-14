# Hermes 与编码任务委派

TipTips 分别管理助理交互、业务状态和执行器运行。Hermes 是可接入的助理引擎；Codex 与 Claude Code 是具有工作区操作能力的执行器；OpenAI/Anthropic API 是模型调用渠道。三类能力不能互相冒充。

## 1 Hermes 参考基线

本次参考安装包声明版本为 0.21.0-cn.18，核心版本 0.21.0，来源仓库 Eynzof/Hermes-CN-Core，提交 b9f82a21fd9268bd499051e23b614ecb06d5b888。依据是安装包的公开生成说明、运行时 manifest、静态前端资源，以及该提交下的 API Server 源码。

参考桌面仓库使用 Tauri、TypeScript 与 Rust；本地前端包含 React 构建资源，运行包包含 Python 动态库。TipTips 借鉴功能组织，独立实现 WPF 交互。[Hermes CN Desktop](https://github.com/Eynzof/Hermes-CN-Desktop)

参考前端中可见会话、模型、Profile、Cron、MCP、Skills、Channels、Pairing、Webhooks、Analytics、System 等页面。可以迁移其“连接集中配置、档案隔离、运行可观测”的思想，但页面存在不代表 TipTips 已具备对应能力。

## 2 已找到的 Hermes 接口

| 能力 | 参考版本入口 | TipTips 用途 |
| --- | --- | --- |
| 能力描述 | GET /v1/capabilities | 连接时确认可用功能和执行位置 |
| 模型目录 | GET /v1/models | 展示引擎可用模型/别名 |
| 会话资源 | /api/sessions 及消息、fork 相关操作 | 建立和保存外部会话映射 |
| 异步运行 | POST /v1/runs | 取得稳定 run_id，适合长任务 |
| 运行查询 | GET /v1/runs/{run_id} | 重连、重启与超时后的对账 |
| 运行事件 | GET /v1/runs/{run_id}/events | 接收结构化 SSE 进度 |
| 审批与引导 | 运行的 approval、steer 入口 | 转交用户决定与补充要求 |
| 停止 | 运行的 stop 入口 | 发起受控取消 |
| 兼容模型 API | /v1/chat/completions、/v1/responses | 受限聊天兼容场景 |

上述接口来自固定提交的源码，尚未在当前安装实例上进行运行验证。该版本的能力描述明确表示工具在 Hermes API Server 主机执行，且 admin_config_rw、jobs_admin、memory_write_api 为 false。聊天/运行接口不能被当成通用管理接口。[固定版本 API Server](https://github.com/Eynzof/Hermes-CN-Core/blob/b9f82a21fd9268bd499051e23b614ecb06d5b888/gateway/platforms/api_server.py)

## 3 Hermes 接入策略

**优先级一：连接独立 Hermes 服务。** 用户填写地址、Profile 与凭据引用；Adapter 先探测能力，再进行一个无副作用会话验证。认证失败、能力缺失和版本不兼容分别展示。

**优先级二：连接本机独立运行包。** 由用户配置受支持的 Hermes 可执行入口，使用独立数据目录和 Profile；进行版本检测、健康探测和进程监督。基础 TipTips 安装不强制包含 Python 运行时。

**优先级三：正式可选引擎包。** 只有在许可证、签名、依赖、更新和停止行为通过验证后才考虑随可选组件分发 Hermes。不能把现有安装目录整体复制到项目或安装包。

如果要求客户端机器完全不含 Python，使用远程 Hermes；原生 TipTips 核心仍独立运行。把 Hermes 全部重写为 C# 不属于该项目初始目标。

## 4 运行与管理分离

Hermes 的 Dashboard 管理入口在参考前端中包含配置、MCP、渠道和模型操作；这些不自动属于稳定公开 API。

首个 Hermes 版本实现连接、能力、会话和运行。TipTips 自有的模型/MCP/渠道配置通过自己的配置中心管理。修改 Hermes 配置仅在用户启用“管理此 Hermes 实例”并验证该版本管理契约后开放；每次写入限定 Profile、展示变更并做备份。

对于用户已有实例，默认不改原 Profile、不关闭既有渠道、不接管 Cron。新增自动化默认在 TipTips 调度，同一规则不能在 Hermes Cron 与 TipTips 同时运行。以后导入 Hermes Cron 时明确选择保留原运行或迁移并停用原规则。

## 5 Hermes Adapter 的完整闭环

1. 连接测试读取版本与能力，确认 Bearer 认证以及实际执行主机。
2. 建立 TipTips 对话和 Hermes 会话映射，限定 Profile 与上下文范围。
3. 先保存 Run 和 operation_id，再提交引擎任务；使用引擎声明支持的幂等能力。
4. 归一文本、工具进度、审批和终态；保存外部 run_id 与事件游标。
5. UI 关闭后核心继续跟踪。连接中断先查询运行状态，不能自动新建一次相同任务。
6. 审批绑定运行和操作摘要。用户作出决定时验证请求仍有效，再转给引擎。
7. 取消先记为 cancel_requested；确认执行器停止后收口。无法确认时保留 unknown。
8. 最终结果与产物进入 TipTips 历史；业务完成与否由业务规则决定。

不要假设 Hermes 的事件重放、进程重启恢复和每种操作幂等性都可用。能力不满足时明确降级；需要可靠长期运行的场景不能使用仅依赖一次 HTTP 流的简化实现。

## 6 防止双重编排

直连模型模式由 TipTips 控制工具循环；Hermes 模式由 Hermes 控制其内部循环。TipTips 只发起目标、提供授权的业务工具、接收事件和结果，不在每次 token 到达时再启动另一个规划器。

TipTips 通过自身 MCP 服务或等效受控工具入口向 Hermes 暴露“读取今日任务、提出日程、创建已授权待办、申请委派”等能力。公开的是业务命令，不能提供任意数据库写入。

Hermes 的技能自动沉淀只进入引擎草稿/候选区；TipTips 的正式技能与记忆必须保留来源与启用决定。两者的长期记忆同步不能假定存在，第一版只做必要上下文投递和引用映射。

## 7 统一委派规格

| 字段 | 含义 |
| --- | --- |
| 目标与验收条件 | 用户想得到什么，如何判断完成 |
| 项目与上下文引用 | 仅包含当前授权资料，固定引用版本 |
| runner 与 model_profile | Codex、Claude Code 或其他 Adapter；不混同模型 API |
| node_id 与 workspace_binding_id | 明确在哪台机器、哪个根目录执行 |
| 允许行为 | 只读、限定写入、运行验证等；额外副作用另行授权 |
| 预算 | 墙钟时间、最大调用/步骤、令牌或金额估算、输出和磁盘大小 |
| 网络与凭据 | 允许的目标、可用的秘密引用、不能扩散的配置 |
| 结果要求 | 摘要、变更文件、验证结果、产物引用与未解决项 |
| 重叠与重试 | 工作区互斥、重试次数、结果未知的处理 |

每次 Run 固定这些字段快照，后续修改委派不应静默改变正在执行的尝试。

## 8 Codex Adapter

正式交互方案采用 Codex app-server 的 stdio 协议：初始化、建立/恢复会话、启动 turn、接收事件和处理服务端审批。协议 schema 与安装版本绑定，Adapter 要保存版本和能力。避免把实验性的远程传输直接作为 LAN 服务。[Codex App Server](https://developers.openai.com/codex/app-server)

有限无人值守任务可采用非交互执行和 JSONL 输出作为较小实现入口，但其交互审批、续接与取消能力不能假定等同于 app-server。[Codex 非交互模式](https://developers.openai.com/codex/noninteractive)

实施时从本机受支持 CLI 发现版本与模型目录；文档中的“sol6.1 max”是后续开发执行环境的指定名称，不硬编码为产品 API 模型标识。

TipTips 管理自己创建的外部会话映射，不读写 Codex Desktop 的私有数据库，不承诺自动控制当前已打开的 Codex 聊天。登录走受支持认证流程，不复制另一应用的认证文件。

## 9 Claude Code Adapter

基础入口采用官方程序化运行模式，消费结构化结果与 stream-json。它适合受限的一次性任务；不能仅凭标准输出就宣称具备完整的双向审批会话。[Claude Code 程序化运行](https://code.claude.com/docs/en/headless)

需要持续输入、审批回调、会话续接时，评估官方 Claude Agent SDK，通过可选 Node 宿主与 .NET 通信；这是执行器 Adapter，不承担业务 UI。采用的 SDK、运行时和许可必须在 S5 固定并验收。[Claude Agent SDK](https://code.claude.com/docs/en/agent-sdk/overview)

首期不得通过全局跳过权限选项来消除集成难度。无法将审批安全接回 TipTips 时，该 Adapter 限制在预授权能力内，超出部分进入 needs_attention。

## 10 执行隔离与恢复

只读工作可以共享目录；写任务默认按工作区串行。Git 工作区可为每个写任务建立独立 worktree；非 Git 工作区用受控副本或明确目录授权。合并、发布、推送和删除属于独立动作，不由“执行成功”自动触发。

进程监督需要资源上限、输出上限、超时和进程树退出。Windows Job Object 适合限制与回收进程，不等同于文件/网络沙箱；真正的权限隔离还需要低权限身份、ACL、受支持的执行器沙箱或可用的隔离环境。

模型或工具已经执行外部动作后，断线重试不能保证撤销。遇到未知结果先对账，保存执行日志和已有产物；没有可靠检查方法时向用户呈现明确的恢复选择。

## 11 集成验收

每种引擎至少验证：无副作用启动、稳定会话映射、流式输出、拒绝审批、取消、错误退出、断流重连、引擎/核心重启、预算达到上限、输出过大以及凭据失效。

Hermes 另外验证工具实际落在哪台主机、管理能力是否存在。Codex/Claude 另外验证工作区隔离、修改清单和验证结果。界面显示“连接成功”不能替代这些契约验收。
