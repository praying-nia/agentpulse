# AgentPulse 开发日志

本日志记录 AgentPulse 跨仓库开发里程碑、已经验证的成果、关键约束和下一阶段目标。它位于总控仓库中，用于统一追踪 `agentpulse-rs`、协议规范与各原生客户端的协作进度。

## 维护约定

- 开始新的实现工作前，先核对本日志的“当前状态”和目标仓库的实际状态。
- “当前状态”随项目推进更新；“历史记录”按时间倒序追加，不改写已经成立的历史事实。
- 只有代码、规范和相应检查均完成后，才能把里程碑标记为“已完成”。探索、计划或未验证的实现不计为完成。
- 每条完成记录必须包含完成内容、关键决策、验证结果、相关提交、遗留事项和唯一的下一目标。
- 提交与推送状态必须按记录时的 Git 事实填写，不能把工作区修改描述为已提交。

## 当前状态

最后更新：2026-08-29

- 当前阶段：基础运行时与首个真实 Provider 已经闭合，准备形成首条面向用户的本地只读 Channel 链路。
- 已完成：统一领域模型、确定性 Core Reducer、Session 状态重放、严格 JSON 协议 v1、Provider / Channel 独立端口、集中 Capability 路由、Bridge 多端点编排、Runtime Host，以及完整只读 Codex App Server Provider。
- 下一目标：实现 Native Channel 的本地只读 Transport 与 Session/Event 同步闭环。
- 尚未决定：持久化介质与恢复存储方案。数据库，特别是 SQLite，不是当前架构的既定依赖。

### 下一目标的验收边界

“Native Channel 的本地只读 Transport 与 Session/Event 同步闭环”完成时应满足：

- 实现首个具体 Native `ChannelPort` 与本地 `Transport`，让一个独立本地客户端能够发现当前 Session、订阅 Session，并持续接收标准 Session 视图与 Event。
- 定义完整的连接握手、协议版本、端点身份、初始同步边界、消息分帧、大小限制、断线清理与显式重连语义；不依赖数据库或 Relay。
- 继续复用现有 JSON Wire v1 领域消息；若握手或同步控制消息确有必要，先在权威协议中定义并以跨语言 Fixture 固定，不将运行时私有状态冒充领域 Event。
- 首版保持只读，不开放 Native 用户 Action；Channel 只声明实际完成的展示与实时同步 Capability。
- 使用独立 Fake Native Client 完成 Codex Fixture → Provider → RuntimeHost/Bridge → Native Transport 的进程内端到端测试，并补充真实本地 Socket 冒烟验证。

下一阶段不接入公网、Relay、数据库、持久化、Bot Channel 或 Provider 写回；完整性集中在首条本地只读产品链路。

## 历史记录

### 2026-08-29 — 完整只读 Codex App Server Provider

状态：已完成并通过验证；本里程碑完成时尚未创建独立提交。

完成内容：

- 将“每个阶段都交付边界明确、真实可用、测试完整的软件能力，而非演示性质的小实现”加入仓库维护原则，同时要求完整性服从显式范围并在选择下一目标前审视总体产品进度。
- 核实并采用官方 Codex App Server，新增 `agentpulse-provider-codex` workspace crate，成对实现只读 `ProviderPort` 与 `ProviderEventSource`，并提供配置、运行状态、计数器和共享 Unix URI 的公开构建接口。
- 固定支持 `codex-cli 0.150.1`，启动前精确检查版本；纳入由该版本生成的完整稳定 JSON Schema，按客户端请求/通知、服务端请求/通知、通用响应/错误及方法响应类型离线严格校验全部原始帧。
- Provider 在 Linux/macOS 创建私有 `0700` 运行目录，托管 `codex app-server --listen unix://...`，完成 Unix WebSocket Upgrade、`initialize/initialized`、全部显式 `thread/resume`、实时读取、停止、子进程回收和仅限自有目录的安全清理。
- 每个 Codex UUIDv7 Thread 稳定映射为同值 AgentPulse Session ID；将恢复快照、Thread/Turn 状态、非空 Agent Message、连接变化及 Completed/Interrupted/Failed 结果归一化为现有 Event，并在重启时重新对齐当前连接与执行状态而不重复创建 Session。
- 对 Schema 合法但领域不承载的消息和未配置 Thread 显式记为 `ValidatedUnmapped`；对服务端请求返回 JSON-RPC `-32601` 只读错误，不伪装审批、输入或命令能力，也不回放 `thread/resume` 内的历史 Turn。
- 保持 Bridge 提交语义：Channel 部分投递失败时 Event 与 Sequence 仍然提交并继续读取；协议、Socket 或进程故障会在入口仍有效时将全部已跟踪 Session 标为 `Disconnected`，保存终态错误并等待显式 RuntimeHost stop/start。
- 增加版本/路径保护、严格 Schema、请求关联、捕获实时流、完整状态/消息/结果链、多 Thread 部分恢复、Channel 部分投递、断线、重启和清理测试；增加独立 Unix WebSocket 测试及可选真实 Codex initialize 冒烟测试。
- 在权威协议中记录 Codex 拓扑、兼容矩阵、生命周期、字段映射、历史/只读边界与失败语义，并同步 Rust、总控 README 及 Provider 使用说明。

关键决策：

- 选择由 Provider 托管的共享 Unix App Server，而不是 `codex exec`、Hook 或 PTY/TUI 抓取；AgentPulse 与 `codex --remote unix://...` 使用同一 App Server，首版平台边界为 Linux/macOS。
- 完全严格兼容策略采用“精确 CLI 版本 + 随版本生成的完整 Schema”，不对未知版本或未知原始消息做猜测兼容；升级必须同时更新版本常量、Schema、Fixture 与测试。
- “完整原始流”表示每个 App Server 帧均被分类、Schema 校验和关联；不表示将所有 Codex 私有类型强行扩展到 AgentPulse 领域模型。只有既定 Session/State/Message/Outcome 语义产生标准 Event。
- 显式 Thread 列表是唯一发现边界；一个 Thread 对应一个 Session，Codex `sessionId` 线程树不合并 AgentPulse Session。
- Provider 声明的唯一扩展 Capability 是 `SESSION_STATE`；普通 Message 使用基础 Event 能力，Interaction Response 与 Agent Command 始终返回结构化只读错误。
- 不实施隐式进程自动重启，避免无提示中断共享 Remote Client；Provider Handle 负责暴露异步失败，调用方通过 RuntimeHost 显式恢复。
- stderr 诊断仅保留有界尾部，并且读取线程只能有界等待，不能让诊断捕获破坏 Provider 的停止超时。

验证结果：

- 全 workspace 的 17 个 Bridge、25 个 Core、9 个 Protocol 与 12 个 Codex Provider 常规测试全部通过，共 63 个单元/集成测试；Core 与 Protocol 各 1 个 doctest 通过。
- 另行在允许创建 Unix Socket 的环境中通过真实 Unix WebSocket 文本帧测试，并使用本机精确版本 `codex-cli 0.150.1` 通过受管 App Server `initialize/initialized` 冒烟测试；常规测试仍不要求安装 Codex 或访问网络。
- Rustfmt check、全 workspace Clippy `-D warnings`、Rustdoc `-D warnings`、锁定依赖的离线测试与 Release Build 全部通过。
- Schema SHA-256 为 `18ba0e2282f69f7b3a05ffdc8ab0801c1468f25d72de3b4a37f1c8be67432a1d`；Schema 与三份捕获 Fixture 均通过 JSON 语法及严格处理测试，三个仓库的 diff/whitespace 检查通过。
- 依赖树确认 Provider 直接依赖 Core、Bridge、离线 `jsonschema`、Serde JSON、Thiserror 与仅启用 Handshake 的 `tungstenite`；未引入 Tokio、TLS、HTTP Schema 获取、数据库或持久化依赖。

相关提交：

- 本次工作的已提交基线为总控 `e96f63e`、Rust `4ebb120` 与协议规范 `e63cde9`，均位于对应 `origin/master`。
- 本里程碑的实现、测试、Schema 与文档位于上述基线后的工作区，提交哈希留待后续提交时记录。

遗留事项：统一模型、协议、Bridge、RuntimeHost 与首个真实 Provider 已经闭合，但还没有任何真实 Channel 或 AgentPulse 客户端连接，因此尚未形成面向用户的产品级端到端链路。

下一目标：实现 Native Channel 的本地只读 Transport 与 Session/Event 同步闭环。

### 2026-08-29 — Runtime Host 与 Adapter 生命周期契约

状态：已完成并通过验证；本里程碑完成时尚未创建独立提交。

完成内容：

- 在 `agentpulse-bridge` 新增运行时中立的 `RuntimeHost`，成对注册并分别拥有 Bridge 内的 Provider/Channel Port 与 Host 内的 Provider Event/Channel Action Source，避免 Source 直接拥有 Bridge。
- 定义 `ProviderEventSource` 与 `ChannelActionSource` 生命周期契约，并为每个启动周期签发绑定强类型端点身份的可克隆受控句柄。
- Provider Event 与 Channel Action 句柄通过弱 Host 引用同步驱动现有 Bridge；Host 释放、端点停止、旧启动周期及身份不匹配均返回结构化入口错误。
- 使用标准库同步原语串行化不同线程的 Bridge 访问，并明确拒绝同线程 Port 回调同步重入，避免形成自引用所有权或同步死锁。
- 定义 Host 与逐 Adapter 的停止、运行、启动失败和停止失败状态，以及包含完整有序逐端点结果并保留 Adapter 原始错误链的生命周期报告。
- 启动按注册顺序尝试全部 Source；单点失败只撤销自身入口，不回滚其他端点或启动期间已经归约的 Session，但再次启动前必须先完成完整停止。
- 停止先撤销全部当前入口，再按注册逆序尝试 Source；停止失败不阻断其他端点，重复停止只重试失败目标，成功停止保留 Port、Session Aggregate 与订阅。
- Host 释放时反序尽力停止并先释放 Source、后释放 Bridge/Port；每次重启生成新句柄，旧句柄不会随新周期重新生效。
- 新增 4 个异构 Fake Adapter 端到端契约测试，覆盖事件泵、Action 入口、启动/停止幂等、失败隔离、错误链、同步重入、跨周期状态、入口撤销和资源释放顺序。
- 在权威规范中定义 Runtime Host 所有权、入口与生命周期语义，并同步 Rust、协议规范与总控 README。

关键决策：

- Runtime Host 采用 Port 与 Source 成对注册，不接收可任意挂载 Source 的预构建 Bridge，保证端点身份和生命周期始终一致。
- 生命周期只控制 Source 执行与 Source 到 Bridge 的入口；成功停止不会注销 Port、清空 Session/订阅、重放历史 Event 或自动重新同步 Channel。
- 句柄按启动周期隔离并弱引用 Host，既不会形成 Source–Host 所有权环，也不会让旧 Worker 在重启后意外恢复写入权限。
- 启动失败是非事务性的：失败前已由 Bridge 接收的合法 Event 保持生效；该 Source 随后进入启动失败状态并等待反序停止清理。
- 部分启动失败保留成功端点继续运行，不执行全局回滚；部分停止失败进入 `StopFailed`，完成失败目标清理前拒绝再次启动。
- 跨线程入口同步串行执行；同步重入显式失败。本阶段不增加队列、缓冲、自动重试、背压、异步运行时或第三方依赖。
- 未显式停止便释放 Host 时执行反序尽力清理，但只有显式 `stop` 能向调用方返回 Adapter 清理失败。

验证结果：

- 4 个 Runtime Host 契约测试、10 个 Bridge 多端点测试、3 个端口测试、25 个 Core 集成测试与 9 个 Protocol v1 集成测试全部通过，共 51 个集成测试；Core 与 Protocol 各 1 个 doctest 通过。
- Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 通过。
- Cargo dependency tree 确认 `agentpulse-bridge` 仍只有 `agentpulse-core` 一个直接依赖，未增加异步运行时、Serde、Transport 或持久化依赖。
- 六份权威 JSON Fixture 继续通过语法校验及 Rust 镜像目录逐字节一致性检查；三个仓库的 diff/whitespace 检查通过。

相关提交：

- 本次工作的已提交基线为总控 `a21b29d`、Rust `40e6b7a` 与协议规范 `5b1338c`，均位于对应 `origin/master`。
- 本里程碑的实现、测试与文档位于上述基线后的工作区，提交哈希留待后续提交时记录。

遗留事项：领域语义、状态归约、严格线协议、多端点 Bridge 与本地 Runtime 生命周期基础已经闭合，但仍没有任何真实 Provider/Channel、Transport、原生客户端连接或 Relay，因此尚未形成产品级端到端链路。

下一目标：实现最小只读 Codex Provider Adapter。

### 2026-08-29 — Bridge 多端点注册与显式 Session–Channel 订阅/扇出

状态：已完成并通过验证；本里程碑完成时尚未创建独立提交。

完成内容：

- 将单 Provider、单 Channel 的泛型 `Bridge<P, C>` 演进为唯一的非泛型多端点 `Bridge`，通过私有类型擦除注册并拥有多个具体类型及错误类型不同的 Provider/Channel Port。
- 注册时只获取一次 Descriptor 快照，以强类型 ID 提供稳定查询与有序遍历，并使用结构化错误拒绝 Provider 或 Channel 重复注册。
- Session 继续由 sequence 1 的 `SessionStarted` 创建并绑定唯一 Provider；Channel 只能显式订阅已存在 Session，不存在隐式全局订阅。
- 支持 `SESSION_VIEW` 的 Channel 在订阅生效前必须先接收当前 `AgentSession` 视图；初始交付失败不会留下活动订阅，重复订阅与取消订阅保持幂等。
- Provider Event 在状态修改前完成全部订阅目标的集中路由，Aggregate 只归约一次，再按 Channel ID 稳定扇出 Event 与可选 Session 视图。
- 单个 Channel 失败不会阻止其他目标；部分失败返回包含所有成功与失败目标的有序报告，并保留 Adapter 原始错误链和已经归约的状态。
- Channel Action 必须来自已注册且仍订阅目标 Session 的 Channel，再根据 Session 所属 Provider 执行 Request、关联与 Capability 校验并完成唯一反向交接。
- 将 Bridge 端到端测试扩展为 10 个多端点契约测试，并保留 3 个独立端口测试；Core 与 JSON Wire v1 未改变。
- 在权威规范中定义手工注册、订阅初始视图、稳定扇出、逐目标失败与反向 Action 语义，并同步 Rust 与总控 README。

关键决策：

- 基础阶段直接演进唯一 `Bridge` 公共模型，不保留旧泛型 Bridge 与新多端点 Bridge 两套长期并行的编排语义。
- 类型擦除仅存在于 Bridge 私有注册边界；公共接口继续使用强类型 ID、领域对象和结构化结果，不暴露具体 Adapter 或 `Any` 下转型。
- 订阅仅接受已存在 Session；支持 `SESSION_VIEW` 时先同步当前 Session 视图再提交订阅，保证后续增量 Event 不会先于可用的当前基线到达。
- 本阶段不补发历史 Event；不支持 Session 视图的 Channel 从订阅后的未来 Event 开始消费。
- 订阅同时是下行投递与反向 Action 的路由边界，取消订阅会立即撤销该 Channel 对 Session 的 Action 权限。
- Event 交付失败会跳过同一目标的 Session 视图，Session 视图失败则保留已经成功的 Event；两者均不阻断后续 Channel。
- 精确重复 Event 仍只返回 `AlreadyApplied`，不会重发到新订阅者，也不会重试此前失败目标。
- Bridge 继续保持同步、内存和运行时中立，仅依赖 Core；未增加 Transport、Serde、异步运行时或持久化依赖。

验证结果：

- 10 个 Bridge 多端点测试、3 个端口测试、25 个 Core 集成测试与 9 个 Protocol v1 集成测试全部通过，共 47 个集成测试；Core 与 Protocol 各 1 个 doctest 通过。
- Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 通过。
- Cargo dependency tree 确认 `agentpulse-bridge` 仍只有 `agentpulse-core` 一个直接依赖。
- 六份权威 JSON Fixture 继续通过语法校验及 Rust 镜像目录逐字节一致性检查；三个仓库的 diff/whitespace 检查通过。

相关提交：

- 本次工作的已提交基线为总控 `794d595`、Rust `515ba0c` 与协议规范 `94e0137`，均位于对应 `origin/master`。
- 本里程碑的实现、测试与文档位于上述基线后的工作区，提交哈希留待后续提交时记录。

遗留事项：当前基础已经覆盖领域语义、状态归约、严格线协议、端口、能力路由及多端点内存编排，但尚无负责启动、停止并驱动 Adapter 的 Runtime Host；真实 Provider/Channel、Transport、原生客户端和可选 Relay 也尚未形成产品级端到端闭环。

下一目标：定义并实现 Bridge Runtime Host 与 Adapter 生命周期契约。

### 2026-08-29 — Bridge 最小闭环

状态：已完成，当前总控、Rust 与协议规范三个仓库的相关工作区修改均尚未提交；submodule 指针尚未更新。

完成内容：

- 在 `agentpulse-bridge` 实现同步内存 `Bridge<P, C>`，每个实例注册一个 Provider Port 与一个 Channel Port，并使用 Descriptor 快照管理多个 Session Aggregate。
- Provider Event 先校验来源、Session 归属与集中 Capability Route，再创建或推进 Reducer；未知 Session 必须由 sequence 1 的 `SessionStarted` 开始。
- 已应用 Event 携带集中 `ChannelEventRoute` 交给 Channel；Session 状态类 Event 在 Channel 支持 `SESSION_VIEW` 时额外交付最新 Session 视图。
- Channel Interaction Response 根据当前待处理 Request 重新校验，Agent Command 根据当前 Session 重新校验，成功后交给目标 Provider。
- 增加 Provider Event Outcome、Channel/Provider Handoff 阶段及两类带 Adapter 原始错误源的结构化 Bridge Error。
- 使用 Fake Provider 与 Test Channel 增加 8 个端到端测试，覆盖 Event → Route → Reduce → Deliver 与 Action → Validate → Provider 完整闭环。
- 在权威规范中补充 Bridge 编排、顺序、状态提交及失败语义，并同步 Rust 与总控 README；Core 与 JSON Wire v1 保持不变。

关键决策：

- 最小 Bridge 采用单 Provider、单 Channel、多 Session 模型；多端点注册、订阅与扇出留到下一里程碑。
- Descriptor 与 Capability 在 Bridge 构造时形成固定快照，本阶段不支持动态变更或 Adapter 生命周期管理。
- Capability 与 Route 失败发生在状态修改前；Reducer 成功后即保留 Aggregate，即使后续 Channel 交接失败也不回滚。
- 精确重复的最新 Event 返回 `AlreadyApplied` 且不重复交付；本阶段不实现缓冲、重试、ACK 或失败恢复。
- 只有 `SessionStarted`、`StateChanged`、`ConnectionChanged` 与 `SessionEnded` 会触发可选 Session 视图交付，且 Channel 必须声明 `SESSION_VIEW`。
- Channel Action 成功交接不会直接修改 Aggregate 或合成 Event；Provider 继续作为 `InteractionResponded` 与 `CommandIssued` 标准化确认 Event 的来源。
- Bridge 继续采用同步、运行时中立的交接语义，仅依赖 Core，不引入 Serde、异步运行时、Transport 或持久化。

验证结果：

- 新增 8 个 Bridge 闭环测试与既有 3 个 Port Contract Tests、25 个 Core 集成测试、9 个 Protocol v1 集成测试全部通过，共 45 个集成测试；Core 与 Protocol 各 1 个 doctest 通过。
- Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 通过。
- Cargo dependency tree 确认 `agentpulse-bridge` 仍仅依赖 `agentpulse-core`，未新增第三方依赖。
- 六份权威 JSON Fixture 继续通过语法校验及 Rust 镜像目录逐字节一致性检查；三个仓库的 diff/whitespace 检查通过。

相关变更：

- 本次工作的已提交基线为总控 `58f3996`、Rust `74f77c0` 与协议规范 `792fe27`，均位于对应 `origin/master`。
- Rust 子模块：Bridge 编排、公开错误/结果类型、端到端闭环测试及 README，尚未提交。
- 协议规范子模块：Bridge 最小编排规范与 README，尚未提交。
- 总控仓库：README 与本开发日志，尚未提交；submodule 指针尚未更新。

遗留事项：尚未支持多个 Provider/Channel 的异构注册、显式 Session 订阅、扇出与逐 Channel 失败隔离。

下一目标：实现 Bridge 多端点注册与显式 Session–Channel 订阅/扇出。

### 2026-08-29 — Provider / Channel 端口与集中 Capability 路由

状态：已完成，当前总控、Rust 与协议规范三个仓库的相关工作区修改均尚未提交。

完成内容：

- 将 `agentpulse-rs/agentpulse-bridge` 建成 workspace crate，定义独立的 `ProviderPort`、`ProviderEventSink`、`ChannelPort` 与 `ChannelActionSink`。
- Provider 端口发布标准化 Agent Event 并接收已验证的 Interaction Response 与 Agent Command；Channel 端口消费带集中路由结果的 Event 或完整 Session 视图并提交用户 Action。
- 在 Core 实现无状态 `CapabilityRouter`、`InteractionRoute`、`ChannelEventRoute` 与结构化 `CapabilityRouteError`，集中校验端点身份、Session 归属、Interaction 语义和端到端 Capability。
- 补全所有 Agent Event 与 Interaction Response 到所需 Provider/Channel Capability 的映射；普通 Message 保持基础 Event，无额外 Provider Capability。
- 使用 Fake Provider、Test Channel 和记录型 Sink 增加端口 Contract Tests，并为 Approval、Choice、Text、SubmitPrompt、CancelSession、只读降级及错误关联增加 Capability Contract Tests。
- 在权威规范中新增端口与能力路由文档，并同步统一领域模型、Rust README 与总控 README；JSON Wire v1 与 Golden Fixtures 不变。

关键决策：

- 端口采用同步、运行时中立的消息交接语义；成功只表示接收或入队，不表示外部 I/O 完成，当前不引入 Tokio 或其他异步运行时。
- Provider 缺少 Request 发布能力属于非法 Event；Request 发布合法但 Provider 回写或 Channel 输入能力不足时统一降级为 `ReadOnly`。
- Channel 消费 Core 给出的路由结果，不自行组合能力；Bridge 在实际转发前仍必须重新校验每个 Response 与 Command。
- 端口只使用统一领域类型与实现本地错误，Provider 和 Channel 不互相依赖，也不包含具体 Adapter 行为。
- 路由结果属于 Core 内存语义而非 Wire v1 消息；本阶段不增加 Transport、Relay、网络、数据库或持久化要求。

验证结果：

- 3 个 Bridge Port Contract Tests、5 个 Capability Routing Contract Tests、既有 20 个 Core 集成测试及 9 个 Protocol v1 集成测试全部通过，Core 与 Protocol 各 1 个 doctest 通过。
- Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 通过。
- Cargo dependency tree 确认 `agentpulse-bridge` 仅依赖 `agentpulse-core`，Core 与 Bridge 均未引入 Serde、异步运行时或 Transport。
- 六份权威 JSON Fixture 继续通过语法校验及镜像逐字节一致性检查；三个仓库的 diff/whitespace 检查通过。

相关变更：

- Rust 子模块：Bridge crate、Core 路由策略、Capability 映射、Contract Tests、workspace/lockfile 及 README，尚未提交。
- 协议规范子模块：端口与能力路由规范、领域模型与 README，尚未提交。
- 总控仓库：README 与本开发日志，尚未提交；submodule 指针尚未更新。

遗留事项：尚未实现 Provider Event 到 Channel Delivery、Channel Action 到 Provider Handoff 的 Bridge 编排闭环。

下一目标：实现 Bridge 最小闭环。

### 2026-08-29 — JSON Protocol v1

状态：已完成，当前总控、Rust 与协议规范三个仓库的相关工作区修改均尚未提交。

完成内容：

- 将 `agentpulse-rs/agentpulse-protocol` 建成 workspace crate，提供版本常量、六类语义消息、JSON 编解码入口和结构化错误。
- 使用私有显式 Wire DTO 隔离线协议与 Core 内存结构；Serde 只进入 Protocol crate，Core 不派生或依赖 Serde。
- 实现 Provider/Channel Descriptor、AgentSession、AgentEvent、InteractionResponse 与 AgentCommand 的完整双向转换。
- 所有解码值重新经过 Core 构造器，保留 UUIDv7、非空文本、Kind、时间顺序、集合唯一性、Progress 与事件关联不变量。
- 定义严格 JSON v1 Envelope、Scalar、Capability 及全部 Tagged Payload，并增加六类顶层消息的跨语言 Golden Fixtures。
- 在规范仓库新增权威线协议文档；规范仓库与 Rust 测试镜像中的 Fixture 在总控布局下逐字节校验一致。

关键决策：

- Envelope 使用数值 `protocol_version: 1`；未知版本、字段、消息、枚举与 Capability 全部拒绝，协议变化必须升级版本。
- Sequence、Revision 与 Progress u64 使用规范十进制字符串，避免 JavaScript/Webhook 精度丢失。
- Timestamp 接受 RFC 3339 Offset，编码时统一归一化为 UTC `Z`；Optional 空值编码时省略，解码接受省略或 `null`。
- InteractionResponse 在缺少 Request 上下文时只校验自身结构；配对、过期与选择语义仍由 Core/Reducer 负责。
- v1 不包含握手、ACK、同步、分帧、大小限制、网络 I/O、数据库或异步运行时。

验证结果：

- 9 个 Protocol v1 集成测试通过，覆盖全部顶层消息与嵌套 Variant、严格拒绝策略、u64 边界、UTC 归一化、恶意输入和 Golden Fixtures。
- 既有 20 个 Core 集成测试继续通过，Core 与 Protocol 各 1 个 doctest 通过。
- Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 通过。
- 六份权威 JSON Fixture 语法校验、镜像一致性及三个仓库的 diff/whitespace 检查通过。
- Cargo dependency tree 确认 `agentpulse-core` 不依赖 Serde 或 Serde JSON。

相关变更：

- Rust 子模块：Protocol crate、Cargo workspace/lockfile、契约测试、Fixture 镜像及 README，尚未提交。
- 协议规范子模块：JSON v1 规范、权威 Fixtures、README/domain-model 与 CI 校验，尚未提交。
- 总控仓库：README 与本开发日志，尚未提交；submodule 指针尚未更新。

遗留事项：尚未定义 Provider / Channel 运行时端口、集中 Capability 路由判断或 Bridge 最小闭环。

下一目标：定义 Provider / Channel 独立端口与集中 Capability 路由判断。

### 2026-08-28 — 开发日志基础设施

状态：已完成；本条首次记录时总控仓库工作区尚未提交。

后续状态：相关变更已由总控提交 `9fdbb4db6b5cbb6a07baef0f60fbe7b7b6d3c509` 提交并同步至 `origin/master`。

完成内容：

- 建立总控仓库级开发日志，集中记录跨子模块里程碑、约束、验证结果和下一目标。
- 建立根级 Agent 维护规则，要求后续实现前读取日志、完成验证后同步更新日志。
- 更新总控 README 的项目状态，并提供开发日志入口。
- 回溯 Core Reducer 的实际提交、测试数量和工作区状态，修正此前“尚未提交”的过时描述。

关键决策：

- 日志放在总控仓库，而不是任一子模块，以覆盖跨仓库工作。
- 日志以中文为主，保留代码标识、命令和协议名称的原文。
- 以可验收里程碑为记录粒度，不记录日常探索和未完成尝试。

验证结果：

- README 中的开发日志链接及日志中引用的关键文件均存在。
- 总控仓库 Git diff 与 whitespace 检查通过。
- Rust workspace tests、Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 继续通过。

相关变更：

- 总控仓库：`README.md`、`DEVELOPMENT_LOG.md`、`AGENTS.md`，尚未提交。
- 所有 submodule 均未产生修改。

遗留事项：尚未开始实现版本化 Rust Protocol crate。

下一目标：实现版本化 Rust Protocol crate。

### 2026-08-28 — Core Reducer 基础建设

状态：已完成并已提交。

完成内容：

- 在 `agentpulse-core` 实现 `SessionAggregate`，支持从 `SessionStarted` 构建状态、逐事件确定性归约及完整事件流重放。
- 严格校验 Session 归属、Event Sequence 连续性、同 Sequence 冲突及旧事件；完整重试最后一个事件保持幂等。
- Plan、Progress 与 Session Revision 均按各自规则推进；失败事件不会留下部分状态更新。
- 配对并维护 Tool Call 与 Interaction 生命周期；Session 结束时映射最终执行状态并清理活跃 Tool 和待处理 Interaction。
- 保存最新 Session、Plan、Progress、Outcome、事件游标及可配置的近期事件窗口。窗口默认保留 256 条，可设为 0，并使用共享事件存储避免复制大型 Payload。
- 在规范仓库补充 Session 状态归约语义，明确 Reducer 不执行 I/O，也不规定重放来源。

关键决策：

- Reducer 是纯内存、同步且确定性的领域组件。
- Core 不引入 Serde、数据库、异步运行时或其他 I/O 依赖。
- 未解决但已过期的 Interaction 仍保留在 Aggregate 中；Channel 应停止展示可操作入口，Reducer 负责拒绝迟到响应。

验证结果：

- 10 个 Aggregate Reducer 集成测试通过。
- 10 个统一领域模型集成测试通过。
- 1 个 doctest 通过。
- Rustfmt、Clippy `-D warnings`、Rustdoc `-D warnings` 与 Release Build 通过。
- Git diff 与 whitespace 检查通过。
- 2026-08-28 回溯时再次通过 workspace tests、Rustfmt check 与 Clippy `-D warnings`。

相关提交：

- 总控仓库：`94e6e0c20eab28bae07ff9fbd2c674250ca70d70`
- Rust 子模块：`6b6d3d97503b4301175722e97ee5465f6149c664`
- 协议规范子模块：`a8951fb267dd4fd30ac53d9c0a22b22b72a7dd6f`
- 回溯时以上提交均位于对应仓库的 `origin/master`，工作区干净。

遗留事项：

- 尚无线协议版本、序列化 DTO 或 Golden Fixtures。
- 尚未选择持久化方式；该问题留到协议和 Bridge 最小闭环稳定后独立评估。

下一目标：实现版本化 Rust Protocol crate。
