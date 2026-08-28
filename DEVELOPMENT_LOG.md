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

- 当前阶段：基础架构建设。
- 已完成：统一领域模型、确定性 Core Reducer、Session 状态重放、严格 JSON 协议 v1、Provider / Channel 独立端口、集中 Capability 路由与 Bridge 最小闭环。
- 下一目标：实现 Bridge 多端点注册与显式 Session–Channel 订阅/扇出。
- 尚未决定：持久化介质与恢复存储方案。数据库，特别是 SQLite，不是当前架构的既定依赖。

### 下一目标的验收边界

“Bridge 多端点注册与显式 Session–Channel 订阅/扇出”完成时应满足：

- Bridge 可注册多个异构 Provider 与 Channel Port，以强类型 ID 稳定查找 Descriptor，并拒绝重复注册。
- Session 明确关联所属 Provider；Channel 通过显式订阅与取消订阅决定接收哪些 Session 的 Event 与最新视图。
- Provider Event 只扇出到订阅目标；单个 Channel 交接失败不会阻止其他 Channel，并返回逐目标交付结果。
- Channel Action 根据来源 Channel、目标 Session 与所属 Provider 解析唯一链路，继续执行集中 Capability 与 Request 校验。
- 使用多个 Fake Provider 与 Test Channel 覆盖订阅、取消订阅、无串流、失败隔离及反向 Action 路由。

下一阶段仍不接入 Adapter 自动发现、真实 Provider、真实 Channel、网络 Transport、数据库或 Relay。

## 历史记录

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
