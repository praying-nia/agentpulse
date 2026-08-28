# AgentPulse 开发日志

本日志记录 AgentPulse 跨仓库开发里程碑、已经验证的成果、关键约束和下一阶段目标。它位于总控仓库中，用于统一追踪 `agentpulse-rs`、协议规范与各原生客户端的协作进度。

## 维护约定

- 开始新的实现工作前，先核对本日志的“当前状态”和目标仓库的实际状态。
- “当前状态”随项目推进更新；“历史记录”按时间倒序追加，不改写已经成立的历史事实。
- 只有代码、规范和相应检查均完成后，才能把里程碑标记为“已完成”。探索、计划或未验证的实现不计为完成。
- 每条完成记录必须包含完成内容、关键决策、验证结果、相关提交、遗留事项和唯一的下一目标。
- 提交与推送状态必须按记录时的 Git 事实填写，不能把工作区修改描述为已提交。

## 当前状态

最后更新：2026-08-28

- 当前阶段：基础架构建设。
- 已完成：统一领域模型、确定性 Core Reducer 与 Session 状态重放。
- 下一目标：实现版本化 Rust Protocol crate。
- 尚未决定：持久化介质与恢复存储方案。数据库，特别是 SQLite，不是当前架构的既定依赖。

### 下一目标的验收边界

“版本化 Rust Protocol crate”完成时应满足：

- 将 `agentpulse-rs/agentpulse-protocol` 建成真实 Rust crate，并加入 workspace。
- 首版使用版本化 JSON Envelope 和显式 DTO；Core 内部结构不直接充当线协议。
- Serde 依赖只进入 Protocol crate，`agentpulse-core` 继续保持无序列化、I/O 和异步运行时依赖。
- 提供 DTO 与 Core 类型的双向转换，并对未知版本及非法数据返回明确错误。
- 增加 JSON 往返测试、版本拒绝测试和跨实现可复用的 Golden Fixtures。
- 同步更新 `agentpulse-protocol` 规范仓库，正式记录协议 v1 与 JSON 线格式。

本阶段不实现数据库、网络 Transport、Provider、Channel、Bridge 或 Relay 运行时。

## 历史记录

### 2026-08-28 — 开发日志基础设施

状态：已完成，当前总控仓库工作区尚未提交。

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
