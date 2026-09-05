# AgentPulse 开发日志

本日志记录 AgentPulse 跨仓库开发里程碑、已经验证的成果、关键约束和下一阶段目标。它位于总控仓库中，用于统一追踪 `agentpulse-rs`、协议规范与各原生客户端的协作进度。

## 维护约定

- 开始新的实现工作前，先核对本日志的“当前状态”和目标仓库的实际状态。
- “当前状态”随项目推进更新；“历史记录”按时间倒序追加，不改写已经成立的历史事实。
- 只有代码、规范和相应检查均完成后，才能把里程碑标记为“已完成”。探索、计划或未验证的实现不计为完成。
- 每条完成记录必须包含完成内容、关键决策、验证结果、相关提交、遗留事项和唯一的下一目标。
- 提交与推送状态必须按记录时的 Git 事实填写，不能把工作区修改描述为已提交。

## 当前状态

最后更新：2026-09-05

- 当前阶段：Android 链路已升级为 Domain JSON v2 / Native Transport v3，支持观察、审批、原生 Codex Plan 协作模式选择/文本表单、带 Host 确认的消息输入和常用 Slash Command。Host 继续只在本次进程内保留完整 Event 历史，Android 继续只在本次进程内按 Cursor 增量补齐；展示层保持会话、待处理项与 Event 最新在上。
- 已完成：`item/tool/requestUserInput` 原子表单、Other/自由文本与敏感输入、类型化远程指令、默认 FIFO Prompt Queue 与显式 Steer、`/model`、`/resume`、`/clear`、`/plan`、`/compact`、`/review`、`/rename`、`/fork`、`/status`、`/permissions`、`/stop` 和 Queue 控制。`/resume` 列表当前工作目录优先且组内最新优先，恢复后按 `thread/items/list` 升序分页补齐完整消息。
- Model 选择安全性：`/model <id> [effort]` 先用实时 `model/list` 校验模型和该模型声明的推理强度，非法输入只产生明确拒绝 Event，不再写入 Turn 默认值、导致后续 `turn/start` 失败并暂停消息队列；Android 同时拒绝多余参数并规范化 effort 大小写。
- 真机进度：Android 15 已通过公网 Relay 收到真实 Codex `request_user_input` 单选 A/B/C，手机选择 C 后结果返回原桌面 TUI；`/model` 和普通消息已到达 Host/Codex 并回显结果，提交框只在收到相关 `command_result` 后清空。真机复测发现旧 `/plan` 仅伪造提示词、实际 Turn 仍为 Default；该实现已修复并完成自动化与真实 App Server 握手验证，等待换用新 Host 后复验原生 Plan 多字段、Other 与敏感字段组合。
- 退出稳定性：Host 会主动中断 Relay 与 Codex Proxy 活动连接，受管 App Server 使用独立进程组；手机在线、桌面 TUI 在线且 Turn 持续输出时实测 5.35 秒退出，无 systemd 超时或遗留 Runtime，并可立即重启。
- 明确决策：Prompt Queue 每 Session 最多 32 项、单项 64 KiB、全局 1 MiB，仅存在 Provider 进程内；`stop` 保留并暂停 Queue，`turn/start` 拒绝也保留队首并暂停。Session/Event、Queue、待处理交互与 Thread 历史都不使用数据库；敏感表单答案写入 Codex 后不保留。Native v3 不兼容旧 Native 端点，但既有设备凭据继续有效。
- 唯一下一目标：在 Android 15 公网 Relay 链路完成 Prompt Queue 断连恢复验收。

### 下一目标的验收边界

- 真机通过公网 Relay 收到至少一个包含选项、Other/文本与敏感字段的 Plan 表单；必须一次性提交全部字段，敏感答案不得出现在后续状态或历史中。
- `/model`、`/resume`、`/clear`、`/plan` 与普通 Prompt 均实际到达受管 Codex App Server；`/resume` 分页历史无重复，当前工作目录优先。
- 活动 Turn 中默认发送进入 FIFO，显式 `/steer` 才改变当前 Turn；断网与 `stop` 后 Queue 不丢失，重连/显式恢复后按原顺序发送。
- 同一 Android/Host 进程中跨越 128 条 Event 的断连缺口按 Cursor 分页补齐且不产生历史通知风暴；Host 重启后新 `host_run_id` 清空旧历史，全程不引入数据库。

## 历史记录

### 2026-09-05 — `/model` 非法输入不再锁死后续消息

状态：根因、Host 防护、Android 参数解析、自动化回归、Release Host、USB 安装和 Android 15 公网 Relay 真机验证均已完成。本次 Rust、Android、协议说明与总控日志修改均尚未提交、未推送。

完成内容：

- 根因是旧 Host 直接把任意 model/effort 保存为 Session Turn 默认值，直到下一条 Prompt 的 `turn/start` 才被 Codex 拒绝；失败路径会按可靠队列策略保留队首并暂停，因此用户看到的是 `/model` 后手机再也发不出消息。
- Model 选择现改为先请求实时 `model/list`，同时核对 model ID 与该模型的 `supportedReasoningEfforts`；只有完整匹配才原子更新默认值，非法 model/effort 返回可见的 `Model selection rejected` Event，既有默认值和 Prompt Queue 均不改变。
- `/model` 目录输出附带每个模型支持的 effort，Android Parser 只接受一个 model 加至多一个 effort，并把 effort 规范化为小写；多余参数在手机端直接判为无效命令。

关键决策：

- 不在 Android 硬编码会随 Codex 版本变化的模型或 effort 清单；Host 对每次选择使用 App Server 的实时目录作为权威来源。Native `command_result` 仍只确认命令已交付 Provider，最终选择或拒绝结果继续通过有序 Domain Event 呈现。

验证结果：

- Codex Provider 测试 37 项通过、2 项按设计 ignored；Clippy `-D warnings`、Rustfmt、Release Host 构建与 `git diff --check` 通过。Android `testDebugUnitTest`、`lintDebug`、`assembleDebug` 通过。
- Debug APK 已通过 USB 覆盖安装到 Android 15 设备 `10CD5Q1FAS0007Y`，Release Host 已安装并以 `codex-rinia` 启动。手机经既有公网 Relay 连接真实会话。
- 真机先提交 `/model invalid-model impossible`，手机收到 `Model selection rejected: unknown model` 且输入框立即恢复；随后普通消息得到真实 Codex 回复 `AP_MODEL_RECOVERY_OK`。再提交合法 `/model gpt-5.6-sol medium`，收到 `Model set`，后续普通消息再次得到真实回复 `AP_VALID_MODEL_OK`。两条路径结束后输入框均可继续发送。

相关提交：

- 当前基线为总控 `58666b7`、Rust `1be8a7a`、Android `e700f15`；本次修改位于工作区，未创建提交、未推送，也未更新总控 Submodule 指针。

遗留事项：本轮只关闭 `/model` 输入污染队列的问题；Plan 多字段、Queue 断连恢复和跨 128 Event 增量补齐仍按既有验收边界继续。

下一目标：在当前 Android 15 公网 Relay 链路完成 Prompt Queue 断连恢复验收。

### 2026-09-04 — 原生 Codex Plan 协作模式

状态：错误的提示词模拟 Plan 已替换为 Codex App Server 原生 `turn/start.collaborationMode`，协议、实现、回归测试和 Release Host 构建均已完成；未自动启动或切换 Host，Android 真机原生 Plan 表单仍待下一轮复验。本次 Rust、生成 Schema 与总控日志修改均尚未提交。

完成内容：

- App Server 初始化声明 `capabilities.experimentalApi=true`，每次 `turn/start` 根据 `/plan` 状态发送 `plan` 或 `default` 的顶层 `collaborationMode`；`developer_instructions:null` 使用 Codex 内建模式指令，不再把 `Work in Plan mode...` 冒充成用户消息。
- Provider 从 `thread/start`、`thread/resume`、`thread/started` 和 `thread/settings/updated` 跟踪当前模型与推理强度；手机未先执行 `/model` 时也能为 Collaboration Mode 填入当前 Thread 的真实模型，显式选择仍优先。
- 删除无效的 `thread/start.config.collaboration_mode`。历史补齐继续过滤旧版本已经写入 Rollout 的伪 Plan 提示，避免把遗留实现文本重新显示到手机。
- 使用本机 Codex CLI `0.153.0` 重新生成 experimental App Server Schema，将 `0.153.0` 加入明确验证版本；更高合法版本继续沿用 best-effort 策略。

关键决策：

- Plan/Default 是逐 Turn 的 App Server 协作模式，不是 AgentPulse 自定义 Prompt，也不绑定或持久化到全局配置。`/plan` 只改变当前 Host 进程内该 Session 的后续 Turn 默认值。
- Collaboration Mode 的模型优先采用手机显式选择，其次采用 App Server 报告的当前 Thread 设置；不硬编码模型名称。关闭 Plan 后显式发送 `default`，避免 App Server 的粘性设置让后续 Turn 继续停留在 Plan。
- 本轮不自动启动或切换 Host，避免擅自改变用户的运行状态；Release 二进制已构建，用户下次启动 Host 时即可加载本次修复。

验证结果：

- 新增集成回归验证 `turn/start` 只包含原始用户文本、携带 `collaborationMode.mode=plan`、继承 Thread 模型且不再包含旧 `config`；初始化回归验证 experimental API opt-in。
- Codex Provider 36 项常规测试全部通过，2 项专用测试保持 ignored；Rust Workspace `--all-targets --test-threads=1` 全部通过，Clippy `-D warnings` 与 Rustfmt Check 通过。
- 本机 Codex CLI `0.153.0` 的真实 App Server 初始化握手通过；`cargo build --release -p agentpulse-host` 通过。未消耗真实模型 Turn 做自动化测试，也未将尚未执行的手机复验描述为通过。

相关提交：

- 当前基线仍为总控 `d22fed3`、Rust `311fed1`、Android `3845d7b`；本次修复连同此前工作区修改均未创建提交、未推送，也未更新总控 Submodule 指针。

遗留事项：需用新 Release 二进制启动 Host，并由手机确认 Rollout 的 `collaboration_mode_kind` 为 `plan`，随后完成多字段、Other/敏感输入、Queue 断连与 128 Event 验收。

下一目标：换用新 Release Host，在 Android 15 公网 Relay 上确认真实 Plan Turn 与多字段表单，并继续完成 Queue 断连恢复和跨 128 Event 增量补齐验收。

### 2026-09-04 — Native 指令确认、并发解锁与连续对话

状态：手机指令确认、Native/Codex 并发死锁修复和已结束会话的连续对话已完成并验证；完整 Native v3 多字段、Queue 断连和 128 Event 验收尚未完成。本次 Rust、Android 与总控日志修改均尚未提交。

完成内容：

- Android 为每次 `submit_command` 跟踪 Request/Command/Session 关联状态；输入在等待 Host 时保留并锁定，只在收到匹配 `command_result` 后清空，协议错误或连接提前结束则保留原文并显示失败原因，未确认请求不自动重发。
- Native Worker 显式缩短 Delivery Mutex 的持有范围，修复断连/发送失败路径在同一线程再次获取该锁造成的自锁；真实 Socket 回归覆盖有待发送 Event 时的突发断连、恢复 Listening 和再次握手。
- Codex Worker 在取出命令、进行中 Session 列表和 Prompt Session 列表后立即释放 Control Mutex，消除 Provider 发布 Event 与 Native 提交下一条命令之间的 Bridge/Control AB-BA 死锁。
- Codex 的 `Completed`、`Failed` 与 `Cancelled` 只表示上一轮 Turn 的结果，不再被当作 Thread 永久不可输入；同一 Session 可继续发起后续 `turn/start`。

关键决策：

- Native `command_result` 只确认 Host 已完成校验并将命令交给 Provider，不伪装成 Codex 已完成；Codex 的用户消息、状态和最终回复仍由后续有序 Domain Event 表达。
- 发送失败保留文本供用户检查或手动重试；断连时不隐式重放不确定是否已被 Host 接收的命令，避免重复 Prompt。
- Codex Thread 是可多轮继续使用的会话；单个 Turn 的完成、失败或取消不是 Thread 的终止条件。所有提交状态和 Prompt Queue 仍只存在于当前进程，不新增数据库或跨启动恢复。

验证结果：

- Android 15 公网 Relay 真机确认 `/model` 返回可用模型列表，普通消息到达 Codex TUI 并将回复同步回手机；等待确认状态不再无限卡住。USB 仅用于安装、界面操作和观察，业务连接仍使用原二维码凭据与公网 Relay。
- 新增 Android JVM 测试覆盖匹配/不匹配确认、可恢复协议错误和断连失败；`testDebugUnitTest`、`lintDebug` 与 `assembleDebug` 通过。
- 新增 Codex 集成回归在状态 Event 投递时验证 Control Mutex 已释放，并验证 `Completed` Session 的后续 Prompt 会产生新的 `turn/start`；Rust Workspace 全目标测试、Clippy `-D warnings` 和 Release Host 构建通过。
- Native 突发断连真实 Loopback 测试单独通过；修复后的 Host 在测试 TUI 退出后 0.38 秒完成停止并可立即重启，最终 Host 保持运行且手机重新连接。

相关提交：

- 当前基线为总控 `d22fed3`、Rust `311fed1`、Android `3845d7b`；本次 Rust、Android 与总控日志修改位于工作区，未创建提交、未推送，也未更新总控 Submodule 指针。

遗留事项：尚未完成 Plan 多字段/Other/敏感输入、活动 Turn FIFO、Queue 断连恢复和跨 128 Event Cursor 补齐的同轮真机验收；诊断期间强制退出后改名保留的 Orphan Runtime 目录尚未删除。

下一目标：继续在同一 Android 15 公网 Relay 链路上完成多字段表单、Queue 断连恢复与跨 128 Event 增量补齐验收。

### 2026-09-04 — 真机单选交互与可靠退出

状态：真实 Android 15 公网 Relay 单选交互与活动 Turn 退出验收已完成；完整 Native v3 多字段、指令、Queue 和 128 Event 验收尚未完成。本次 Rust 与总控日志修改均尚未提交。

完成内容：

- 使用受管 Codex App Server 与桌面 Proxy 发起真实 `item/tool/requestUserInput` A/B/C 单选，Android 真机收到交互，手机选择 C 后响应准确返回发起请求的桌面 TUI。
- Relay Host Connector 新增一生命周期连接取消器，停止时主动关闭活动 TCP/TLS Socket；Relay 线程等待设置 2 秒硬上限，覆盖 DNS、连接建立和状态切换无法即时中断的边界。
- Codex Client Proxy 跟踪每条 Route 的上下游 Unix Socket，停止时先关闭全部活动连接再 Join，覆盖桌面 WebSocket 保持在线时的退出路径。
- 受管 Codex App Server 在独立 Unix 进程组中启动；先向全组发送 SIGTERM，超时后向全组发送 SIGKILL，避免 Node 启动器退出后 Codex/MCP 子进程遗留在 systemd Unit 中。
- 强制退出留下且确认无人占用的 Runtime 目录均通过改名保留，没有删除用户数据；最终候选版本能自行删除活动 Runtime 并立即重新启动。

关键决策：

- Host 停止不等待不可中断的公网 DNS/连接操作超过自身退出预算；Relay Worker 超时后仅 Detach，进程完成其余安全清理并退出。
- App Server 及其子进程属于同一次临时 Host 生命周期，停止时整组终止；不保存或恢复本次 `ap` 选择、Thread 映射或正在执行的 Turn。
- USB/ADB 仅用于启动、观察和操作测试 App；配对凭据与业务连接仍完全来自二维码和公网 Relay，没有注入调试连接参数。

验证结果：

- Rust `cargo test --workspace` 在允许 Unix Socket 的环境通过；新增覆盖 Relay 阻塞读取消、Relay Join 上限、活动桌面 Proxy 中断和 App Server 完整进程组超时终止。
- `cargo clippy --workspace --all-targets -- -D warnings`、`cargo build --release -p agentpulse-host` 与 `git diff --check` 通过。
- 真机隔离结果为仅手机连接 0.32 秒退出、仅桌面 TUI 连接 0.25 秒退出；最终在 Android 公网 Relay 已连接、桌面 TUI 在线且 Turn 持续输出时 5.35 秒退出。最终 systemd 日志没有 `stop-sigterm timed out` 或代为强杀残留进程，活动 Runtime 目录不存在，随后 `ap host rinia` 启动成功并恢复手机连接。

相关提交：

- 当前基线为总控 `a843ed1`、Rust `4fd64b9`；本次 Rust 修改与总控日志位于工作区，未创建提交、未推送，也未更新总控 Submodule 指针。

遗留事项：尚未完成多字段 Plan 表单、Other/敏感输入、常用远程指令、Prompt Queue 断连恢复和跨 128 Event 增量补齐的同轮真机验收；此前改名保留的 Orphan Runtime 目录尚未删除。

下一目标：继续在同一 Android 15 公网 Relay 链路上完成多字段表单、常用指令、Queue 断连恢复与跨 128 Event 增量补齐验收。

### 2026-09-04 — Plan 原子表单与常用远程指令

状态：Domain JSON v2、Native Transport v3、Rust/Android 实现、跨语言 Fixture 与自动化验证已完成；真实 Android 公网 Relay 端到端验收尚未执行。本次总控、Rust、Android 与协议规范修改均尚未提交。

完成内容：

- Core 新增消息角色、原子 Form Request/Response、字段与答案强类型，以及覆盖常用控制面的 Agent Command；Bridge 继续集中校验完整 Provider/Channel Capability Route。
- Codex Provider 将 `item/tool/requestUserInput` 映射为多字段表单，保留选项、Other/自由文本、阻塞和敏感属性；回写后只保留“已响应”状态，不保留敏感答案。
- Codex Provider 新增有界进程内 FIFO Prompt Queue、显式 Steer、停止/恢复/清空 Queue，以及 model、resume/new、plan、compact、review、rename、fork、status 和 permission profile 调用；所有 App Server 请求及响应继续通过生成 Schema 严格校验。
- `/resume` 列表按当前 CWD 优先、组内更新时间倒序展示，并在首次恢复后使用 `thread/items/list` 升序分页补齐用户/助手消息；重复恢复已跟踪 Thread 不重复历史。
- Native Transport v3 新增 `submit_command`/`command_result`，Domain JSON v2 新增 Form、消息角色和类型化指令；Android 新增可输入 Composer、Slash 建议、Plan 开关选择、表单 Card、Other/密码输入及原子提交。

关键决策：

- 只实现高频 Slash Command，不镜像全部 Codex App Server API；普通文本默认 Queue，只有显式 `/steer` 修改活动 Turn。
- Queue 上限为每 Session 32 项、单项 64 KiB、Provider 合计 1 MiB；`stop` 与 `turn/start` 拒绝均保留并暂停 Queue，避免断连或暂时错误导致丢失/重试风暴。
- Session/Event、Queue、交互和恢复历史均限定在当前进程内，不新增数据库或磁盘持久化。敏感表单答案只存在于一次 outbound payload，成功写入后立即丢弃。
- 严格协议发生不兼容变更，因此 Domain 与 Native 分别升级至 v2/v3；配对协议仍为 v1，广告的新版本号与既有 CA/Token 可继续使用。

验证结果：

- Rust `cargo check --workspace --all-targets` 通过；`cargo test --workspace --all-targets -- --test-threads=1` 通过（常规测试 110 项，6 项需专用网络/真实 Codex 环境的测试保持 ignored）。新增覆盖全部指令 Payload、Queue 边界、原子表单、敏感答案生命周期、严格 App Server 控制请求与 Native v3 Fixture。
- Android `./gradlew test lintDebug assembleDebug --console=plain` 通过，新增覆盖指令 Codec/Parser、Form 原子校验与敏感答案不进入 Reducer State；Debug APK 可构建。
- Rustfmt、Clippy `-D warnings`、各仓库 `git diff --check` 与权威/Rust Native Fixture 镜像一致性通过。本里程碑未执行真机或真实受管 Codex 指令 E2E，不将其描述为已验证。

相关提交：

- 修改位于当前各仓库基线后的工作区；未创建提交、未推送，也未更新总控 Submodule 指针。

遗留事项：Android 15 公网 Relay 下的表单、指令、Queue 断连恢复、跨 128 Event 补齐与 Host 重启 Reset 仍需一次完整真机验收；iOS/HarmonyOS 尚未实现 Native v3。

下一目标：在 Android 15 真机公网 Relay 上完成 Plan 表单、常用指令、Queue 断连恢复与跨 128 Event 增量补齐的 Native v3 端到端验收。

### 2026-09-03 — Native Transport v2 当前 Host 运行历史与增量恢复

状态：实现、协议、跨语言 Fixture、自动化与真实 Loopback WebSocket 断线恢复测试已完成；未执行 Android 真机公网多页验收。本次总控、Rust、Android 与协议规范修改均尚未提交。

完成内容：

- Core 新增显式完整内存 Event 保留配置和按 Cursor 连续后缀查询；Host 生产路径使用该配置，默认有界 Reducer 配置仍可用于其他调用方。
- Bridge 新增最多 128 条的 Session Sync Page，每条保留 Event 重新经过集中 Capability Route；只有最终页成功交付当前 Session/Pending Interaction Baseline 后才注册 Live Subscription。
- Native Transport 直接升级为 v2 端点与 Subprotocol。Client Hello 携带 Host Run 和逐 Session Cursor，Server Hello 明确恢复是否接受；分页结果包含 Event Count、Reset 与 Catching-up 状态，最终 Baseline 与后续 Live Event 保持有序。
- Android 客户端按 Host 保留进程内 Native State，重连 Hello 上报已连续应用的 Cursor；Reducer 按页暂存、校验计数与 Sequence、原子提交，移除旧的 256 Event 截断。Host Run 不同时立即清除旧 Session 缓存。
- ConnectionService 在重试期间保留已有 UI 状态，禁止 Catch-up 历史通知，进入 Live 时只对仍 Pending 的 Interactive Approval 各通知一次。Native/配对 Golden Fixture、README 与权威协议同步 v2 语义。

关键决策：

- `host_run_id` 由 Host 进程中的 Native Channel 创建，WebSocket、Relay 与 RuntimeHost Source 重启不改变它；新 Host 进程创建新 ID。
- Host 与 Android 都只保留本次运行/进程的完整历史，不使用 Session/Event 数据库或磁盘存储。断连是可恢复的运输事件，进程死亡则是明确的历史边界。
- Native v2 不在同一端点兼容 v1 APK；已配对的 CA 与设备 Token 无需重新签发，Pairing 广告的 Native Transport Version 同步改为 2。
- “最新在上”仅属于 Presentation；传输、缓存和 Reducer 必须按 Sequence 升序验证和应用。

验证结果：

- Rustfmt Check、Workspace Clippy `-D warnings`、Workspace `--all-targets` 105 个常规测试通过；Native 的 2 个 ignored 真实 Loopback WebSocket 测试单独通过，覆盖严格 v2 握手、订阅、断连、同 Run Cursor 增量恢复和错误路径。
- Bridge 分页测试覆盖非最终页不进入 Live；Core 保留测试和 Android Reducer 测试均覆盖超过原 256 条窗口，Android 另覆盖同/异 Host Run 与多页原子提交。Rust/权威规范 Native v2 与 Pairing Fixture 逐字节一致。
- Android `./gradlew test lintDebug assembleDebug compileDebugAndroidTestKotlin` 通过，Debug APK 与 Instrumentation 测试源码可构建；本里程碑未在真机安装或运行 Instrumentation，不将其记为已验证。各工作区 Diff/Whitespace Check 通过。

相关提交：

- 当前已提交基线为总控 `ef82119`、Rust `87de57a`、Android `800d6b9`、协议规范 `2d2022e`。本里程碑修改位于这些提交后的工作区；未创建提交、未推送，也未更新总控 Submodule 指针。

遗留事项：Android 真机公网 Relay 的跨 128 Event 多页恢复和 Host 重启 Reset 尚未验收；Native v1 APK 必须升级才能连接 v2。跨进程历史与 Session/Event 数据库明确不在当前产品边界。

下一目标：在 Android 15 真机公网 Relay 链路上完成跨 128 Event 断网增量恢复、通知抑制与 Host 新 Run 重置验收。

### 2026-09-03 — Android 视觉、导航与最新优先信息架构重构

状态：实现及 JVM 测试、Android 测试源码编译、Lint 与 Debug APK 构建已完成；连接真机存在，但用户在设备端拒绝测试 APK 安装，因此本次 Instrumentation 实际运行 0 项，不记为通过。本次 Android 与总控日志修改尚未提交。

完成内容：

- 将单文件 Compose 界面拆为 Activity 边界、应用页面层和主题层；手机采用连接、会话、设置三栏底部导航，会话详情为二级页，840dp 以上使用 Navigation Rail 和会话列表/详情双栏。一级页切换和手机会话列表/详情进入返回均使用带方向的短距离滑动与淡入淡出，实时 Event 更新通过稳定页面 Key 避免重复触发整页动画。
- 连接首页重组扫码、Host 状态、LAN/Relay 选择、连接操作和最近会话；会话页新增标题/工作区搜索及全部、运行、等待、结束状态筛选；设置页集中主题、Relay、通知入口、版本和带确认的忘记设备操作。
- 会话继续按 `updatedAt` 倒序；事件只在展示层按 `sequence` 倒序，待审批按请求时间倒序固定在事件之前。Reducer 的正序追加、连续序号检查、256 条窗口与通知增量检测均未改变。
- 新增独立的非敏感 UI Preferences DataStore，支持跟随系统/浅色/深色、Android 12+ 动态色、五套品牌预设及六位 HEX 自定义主色；自定义色生成浅深 Compose ColorScheme，成功、警告、错误语义色保持稳定。
- 参考稿中的消息输入框作为可点击只读占位，点击明确说明当前版本只支持查看与审批，不产生协议请求；现有审批选项、确认与提交状态完整保留。中英文资源和主题切换时的系统栏图标可读性同步完成。

关键决策：

- 本阶段只改变 Android 展示与本地偏好，不修改 Domain/Native 协议、ConnectionService Action、Host、Relay 或审批回写边界。
- Reducer 保留协议权威顺序；“最新在上”是可独立测试的 Presentation 规则，避免破坏事件连续性和重要事件通知。
- UI 偏好不属于 Host 凭据，不进入 Android Keystore 加密 Payload；动态色在 Android 12 以下确定性回退为 AgentPulse 靛蓝。
- 未引入整套 View Material Components 依赖；预设/自定义 seed 由轻量、确定性的 Compose 色调生成器转换为浅深 ColorScheme，保持离线构建可复现。

验证结果：

- `./gradlew test lintDebug assembleDebug` 通过；App JVM 测试 7 项全部通过，其中新增 3 项覆盖事件倒序且不修改输入、会话最新优先/筛选搜索和 HEX 校验。Protocol 测试随同 `test` 通过。
- `:app:compileDebugAndroidTestKotlin` 通过，新增启动与底部导航设置页测试可编译；`git diff --check` 在 Android 与总控仓库通过。Debug APK 位于 `app/build/outputs/apk/debug/app-debug.apk`。
- `connectedDebugAndroidTest` 在 vivo V2282A Android 15 安装阶段被设备端以 `INSTALL_FAILED_ABORTED: User rejected permissions` 拒绝，测试框架报告 0 tests；未把该外部条件失败描述为测试成功，也未绕过设备确认。

相关提交：

- 当前已提交基线为总控 `4e749a6`、Android `5e7cd7f`。本里程碑代码与日志位于这些提交后的工作区；未创建提交、未推送，也未更新总控仓库中的 Android Submodule 指针。

遗留事项：新视觉尚未在真机允许安装后执行 Instrumentation 和人工截图对照；自由文本发送仍按设计不可用，界面只提供带原因说明的占位。协议、Host 与 Relay 均无本次修改。

下一目标：实现 Codex Provider → Bridge/Native Transport → Android 的远程批准/拒绝闭环，并在同一 QR 配对真机公网链路上验证能力门禁、一次性响应、超时和错误路径。

### 2026-09-03 — `ap` 无状态 Thread 发现、Proxy 与逐次工作目录

状态：实现、文档、自动化、真实 Codex 进程参数与 Android 15 真机连接验证已完成；本次 Rust、Android 与总控日志修改尚未提交。当前 Host 以 `codex-nona` profile 正常运行。

完成内容：

- `ap` 启动 Host 时启用新的 `serve --discover-threads` 模式；Codex Provider 可以从零个保存的 Thread 启动，并根据同一受管 App Server 广播的 `thread/started` 动态创建 Session。RuntimeHost 停止后映射立即消失，快捷器不再依赖、读取、修改或保存全局 `agentpulse threads` allowlist。
- `ap` 比对调用 Shell 与活动 Host 进程的大小写 `HTTP_PROXY`、`HTTPS_PROXY`、`ALL_PROXY`、`NO_PROXY` 环境；不一致时有序重启 Host，并由 `systemd-run` 安全继承当前值。受管 App Server 与随后打开的 Codex TUI 因此使用所选 profile 的原环境和当前 Proxy。
- 每次 `ap [profile]` 打开前台 Codex 时都会默认追加 `-C "$PWD"`，所以复用早先从 `$HOME` 或其他工程启动的后台 Host 不再污染本次工作目录。用户显式提供 `-C`、紧凑 `-C<path>`、`--cd <path>` 或 `--cd=<path>` 时保持原参数并优先采用用户目录。
- Android Pairing Client 的 WebSocket/TLS 清理移入 `NonCancellable + Dispatchers.IO`，且清理异常只记录而不覆盖已经收到的 `Succeeded`，消除了成功后主线程关闭 Socket 导致的无消息“配对失败”。
- `ap --help` 与 Host README 同步说明逐次目录、显式覆盖、Proxy 继承和无状态 Thread 生命周期；Provider 与总 README 同步临时 Thread 发现边界。

关键决策：

- `nona`/`rinia` 只选择对应 Codex executable/profile；它们不是 AgentPulse 全局配置或 Thread 集合。Host 停止后 Session 映射不保留，只有 Host 机器身份、已配对设备凭据与 Relay 配置继续作为机器级状态存在。
- 工作目录属于每一次前台 Codex 调用，不属于长驻 Host。依据 Codex CLI 的正式 `-C/--cd` 语义，快捷器给未指定目录的调用补上调用时的 `$PWD`，而不是为切换目录重启 Host。
- Proxy 属于受管 App Server 的进程环境，无法在已运行进程中补写；只有检测到差异时才重启 Host，目录变化则完全不触发服务重启。

验证结果：

- Rust 通过 Rustfmt、Workspace Clippy `-D warnings`、95 个常规测试、6 个串行 ignored 真 Socket/真实 Codex 测试、Rustdoc `-D warnings` 与 Release `--workspace --all-targets --all-features --locked` Build；动态发现专门测试验证未保存 Thread ID 时可由 `thread/started` 建立 Session。
- 两个连接同一临时真实 Codex App Server 的客户端验证：一个客户端启动 Thread 时，观察客户端收到同一 `thread/started`；`ap host nona` 也能在无保存 Thread 依赖下启动 Provider。Android Debug APK 通过 `lintDebug testDebugUnitTest assembleDebug` 并覆盖安装，用户随后确认 QR 配对后的手机连接恢复正常。
- 真实运行中的 `codex-nona` Host 保持复用；从 `/home/nona/connection/agentpulse` 与 `/home/nona/mcg` 分别执行 `ap nona --version`，系统调用跟踪确认最终 Codex 参数分别只含对应目录的 `-C`，两次均返回 `codex-cli 0.152.1`。另一次 `--cd=/tmp` 跟踪确认没有注入调用目录且显式值原样到达 Codex。
- `bash -n scripts/ap`、Rust/Android/总控 `git diff --check` 通过；真实 Host 与子 App Server 的 Proxy、profile、工作目录和运行状态也通过进程环境、状态及命令参数核对。

相关提交：

- 当前已提交基线为总控 `b72e8e5`、Rust `19a6500`、Android `8755a09`、协议 `f9bca8f`。本里程碑修改位于这些提交后的工作区；未创建提交、未推送，也未更新总控仓库中的 Submodule 指针。

遗留事项：单个 Host 同时只运行一个 Codex profile，切换 profile 会使手机和已有 Remote TUI 短暂断开后重连；停止 Host 会按设计丢弃临时 Session 映射。稳定签名 Android Release 与 Actions Artifact 非 USB 安装仍待 App 功能完成后验收。

下一目标：实现 Codex Provider → Bridge/Native Transport → Android 的远程批准/拒绝闭环，并在同一 QR 配对真机公网链路上验证能力门禁、一次性响应、超时和错误路径。

### 2026-09-03 — `ap` 单命令 Host、Codex 与二维码工作流

状态：实现、文档、自动化与本机 systemd/Codex/二维码生命周期验证已完成；本次 Rust 与总控日志修改尚未提交。Host 当前由 `ap` 以 `codex-rinia` profile 保持运行。

完成内容：

- 新增 Linux 快捷器 `scripts/ap` 并通过 `~/.local/bin/ap` 暴露：`ap`、`ap nona`、`ap rinia` 会启动或复用匹配 profile 的 Host，再连接 Codex TUI；额外 Codex 参数原样转发。`ap status`、`ap stop`、`ap logs` 和仅启动服务的 `ap host [profile]` 覆盖日常运维。
- 冷启动或 profile 切换后，`ap` 会在当前终端自动运行一次 QR-only Pairing；扫码完成后继续进入 Codex。用户可用 `Ctrl+C` 取消这次二维码，systemd Host 不受影响；Host 已运行时不会重复强制二维码，`ap qrcode` 可随时创建新的两分钟一次性二维码。
- Host 运行状态新增 PID 和可选 Codex executable，快捷器据此只复用完全相同的 profile；旧 Host 状态缺少 executable 时仍可反序列化，并会执行一次明确的有序切换。`codex`、`codex-nona`、`codex-rinia` 原命令没有被覆盖。
- Host 改由 `systemd-run --user` transient service 托管，使用 `Type=exec`、`KillMode=mixed`、30 秒停止上限和 Journal 日志。Workspace 为 `ctrlc` 启用 `termination` feature，使 systemd 的 SIGTERM/SIGHUP 进入现有 stop 路径，而不是直接杀死主进程并遗留 Codex socket。
- 本机安装两个符号链接：`~/.local/bin/agentpulse` 指向当前 Workspace Release 二进制，`~/.local/bin/ap` 指向版本化脚本；`.bashrc` 已经把 `~/.local/bin` 放在 PATH 首位，因此无需修改或重新加载 shell 配置。

关键决策：

- Codex `--remote` 只是把终端 UI 连接到 App Server；认证、历史和 Thread 由 App Server profile 决定。因此快捷器必须让 Host 与 TUI 使用同一个 executable wrapper，不能把 `nona`/`rinia` 随意连到默认 profile。默认 profile 缺少当前配置 Thread 时的真实启动失败验证了这一边界。
- Host 留在后台由 systemd user manager 保证，不依赖 `nohup` 或调用终端的进程组。profile 切换会有意停止单一 Host、等待 PID 与 transient unit 完全释放，再启动新 profile，避免两个 App Server 争用同一私有 socket。
- 自动二维码只发生在本次调用实际启动 Host 时；已配对用户的每次 Codex 启动不会被多余 QR 阻塞。独立 `ap qrcode` 仍严格调用现有 `agentpulse pair`，没有新增手工 Token、URI、ADB 或蓝牙路径。

验证结果：

- `ap host rinia` 在独立命令结束后仍由 systemd 保持 Host；`ap status` 返回 Provider `running`、Relay `waiting_or_tunneling`、PID 与 `/usr/local/bin/codex-rinia`。`ap rinia --version` 在已运行时直接复用并返回 `codex-cli 0.152.1`。
- 完整冷启动 `ap rinia --version` 实际启动 Host 并自动输出一次终端 QR；取消 Pairing 后 `ap status` 仍显示同一 Host 在线。`ap qrcode` 也独立输出真实两分钟 QR，取消后不影响稳定 Host。
- 启用 SIGTERM 处理后的 `ap stop` 使 systemd unit 变为 `not-found/inactive/dead`，私有 Codex runtime directory 与 socket 均不存在；修复前真实复现的 systemd SIGKILL 子进程与孤立 socket 不再出现。
- Bash 语法、Rustfmt、Workspace Clippy `-D warnings`、95 个常规测试、6 个串行 ignored 真 Socket/真实 Codex 测试、Rustdoc `-D warnings`、Release `--workspace --all-targets --all-features --locked` Build 及 Rust/总控 `git diff --check` 通过。一次首次并行 Workspace 测试中的既有私有目录测试失配，目标测试、Provider 全集及完整 Workspace 立即独立复跑均通过，未把首次失败记为成功。

相关提交：

- 当前已提交基线为总控 `78fb284`、Rust `0736dab`、Android `8755a09`；快捷脚本、Host 状态/SIGTERM 支持、README 与本日志位于这些提交后的未提交工作区。未创建提交、未推送，也未更新总控仓库的 Rust Submodule 指针；`~/.local/bin` 两个符号链接属于本机安装状态而非 Git 文件。

遗留事项：单个 Host 同时只运行一个 Codex profile，切换 profile 会使手机和已有 Remote TUI 短暂断开后重连；不同 profile 必须配置实际存在于对应 `CODEX_HOME` 的 Thread。稳定签名 Android Release 与 Actions Artifact 安装仍待 App 功能完成后验收。

下一目标：实现 Codex Provider → Bridge/Native Transport → Android 的远程批准/拒绝闭环，并在同一 QR 配对真机公网链路上验证能力门禁、一次性响应、超时和错误路径。

### 2026-09-03 — QR-only 真机配对闭环与终态可靠交付

状态：实现、自动化与 Android 15 真机公网验证已完成；本次 Rust/Android 修复尚未提交。开发期使用本地 Debug APK，GitHub Actions APK Artifact 验收推迟到 App 功能完成后。

完成内容：

- 修复 Pairing Server 发送 `succeeded` 或终态错误后立即返回、Host 随即停止一次性 Relay Connector，导致终态 WebSocket Frame 尚未抵达 Android 就被截断的问题。服务端现在发送 Close 后最多等待两秒完成客户端关闭握手，再允许临时 route 清理；成功、拒绝、容量、非法请求和非法凭据路径使用同一终止语义。
- 新增真实 TLS/WebSocket 回归测试，证明客户端收到 `succeeded` 和服务端 Close 之前 Pairing Session 不会提前返回；Android 实测从原来的 `Pending → EOFException` 变为 `Pending → Succeeded → Close 1000`。
- 修复常驻 Host 收到 stop 后可能永久卡在 Relay Connector join 的问题：Relay 等待循环在阻塞读取前后均检查退出标志，持续 Ping 不再无限刷新读取超时并掩盖退出。修复后二次 `agentpulse stop` 在约两秒内完整退出，运行目录正常释放，无需强制终止或人工清理。
- Android 为配对与 Relay 外层握手增加不包含 Token/URI 的生命周期日志；配对失败标题不再继续显示“正在配对”，空异常消息使用明确兜底文案。USB 仅用于覆盖安装本地 Debug APK、启动应用和读取日志，实际 Bootstrap 与后续业务链路仍只经过应用内相机 QR 和公网 Relay。

关键决策：

- `succeeded` 应用 Frame 的写入成功不等于公网链路已可靠交付；Pairing route 的生命周期必须覆盖 WebSocket Close 握手，短超时只负责给异常客户端提供有界退出。
- Host 的 stop 标志必须能在 Relay 心跳活跃时生效；检查点放在每次控制 Frame 读取前后，既不改变正常 Ping/Pong，也保证下一次心跳或既有读取超时能够唤醒退出。
- 开发期真机允许 USB 调试以提高诊断效率，但 USB 不是产品配对方式，也不得通过 ADB reverse/forward、参数注入或业务转发绕过二维码。Actions Artifact 属于 App 完成后的发布验收，不作为当前开发迭代的前置条件。

验证结果：

- Android 本地 Debug APK 通过 `lintDebug testDebugUnitTest assembleDebug` 并覆盖安装到 vivo V2282A（Android 15）；真机经相机扫描终端 QR 和 `ap.nonamenona.top:19191` 收到 `Pending`、`Succeeded`、Close 1000，随后自动保存凭据并建立正式 Relay tunnel。
- 真机恢复真实 Codex Session `重构SummonGssProjectileTask上下文`、Cursor 1，界面显示“空闲 · 已连接”、工作区 `/home/nona/mcg` 与 Provider `Codex`。新 route 在 Host 刷新前出现的短暂 `authentication_failed` 可见并自动重试，刷新后恢复。
- 修复后的 Host 完成真实 stop/start：旧进程约两秒退出，新 Host 正常启动；手机先得到有界 `host_unavailable`，约十八秒后自动重新建立 tunnel，同一 Session 再次显示“空闲 · 已连接”。
- Rust 通过 Rustfmt、Workspace Clippy `-D warnings`、95 个常规测试、6 个串行 ignored 真 Socket/真实 Codex 测试、Rustdoc `-D warnings` 与 Release `--workspace --all-targets --all-features --locked` 构建；Rust、Android 与总控仓库 `git diff --check` 通过。

相关提交：

- 当前已提交基线为总控 `4bbe0f5`、Rust `6a71657`、Android `4465dab`；Rust 配对/Relay 退出修复、Android 诊断/UI 修复及本日志位于这些提交后的未提交工作区。未创建提交、未推送，也未更新总控仓库的 Rust/Android Submodule 指针。

遗留事项：当前手机因调试过程中重复完成配对而在 Host 留有多条同设备凭据，可在确认保留的当前凭据后显式撤销旧设备；稳定签名 Release 与 Actions Artifact 真机安装待 App 功能完成后验收。任意命令、结构化输入、离线恢复、Bot Channel、iOS/HarmonyOS 和数据库仍未实现。

下一目标：实现 Codex Provider → Bridge/Native Transport → Android 的远程批准/拒绝闭环，并在同一 QR 配对真机公网链路上验证能力门禁、一次性响应、超时和错误路径。

### 2026-09-03 — Codex 新版本尽力启动兼容门禁

状态：实现、文档与发布级自动化验证已完成；本次兼容修改尚未提交。QR-only 真机物理扫码仍是唯一下一目标。

完成内容：

- Codex Provider 首选已验证版本提升到 `0.152.1`，精确已验证集合为 `0.150.1`、`0.152.0`、`0.152.1`；公共版本常量、Provider Descriptor、错误文案与中英文 README 同步更新。
- 版本探测改为严格解析 `codex-cli <SemVer>`：精确集合直接启动；SemVer 优先级高于当前稳定基线 `0.152.1` 的版本在 stderr 输出兼容性警告后尽力启动；未验证且不高于基线的版本、同核心版本的 Build Metadata 变体与 Pre-release、非法输出继续在创建运行目录前拒绝。
- 前向启动不放宽协议：Provider 继续使用完整 JSON Schema 严格校验 App Server 消息；未知或结构不兼容的 Frame 仍进入可见失败，不静默忽略，也不把未验证新版本写入“明确支持”集合。
- 本机 `codex-cli 0.152.1` 重新生成的官方聚合 Schema 与仓库内 `0.152.0` 文件逐字节一致，因此保留原 Schema 文件和 SHA-256，仅记录两版本共同验证事实。
- 前一 QR-only 阶段的修改现已提交并推送：总控 `56eabdf`、Rust `14de45f`、协议 `f9bca8f`、Android `4465dab`。Android Actions Run `33493119530` 的常规检查与 Emulator Smoke Test 成功；Rust Run `33493124819` 的检查、同 Artifact 生产自动部署及公网 Probe 成功。

关键决策：

- “支持”仍表示 Schema、Fixtures 与真实 App Server 握手均已验证；更高版本只是默认获准尝试运行，不获得兼容保证。若上游协议发生真实破坏，严格解析会让 Provider 明确失败，由后续适配处理。
- 新旧判断遵循 SemVer 优先级而不是字符串比较。`0.153.0-beta.1` 因核心版本高于 `0.152.1` 可尝试启动；`0.152.1+unverified` 与 `0.152.1-beta.1` 都不高于稳定基线且不在精确集合，因此拒绝。
- 官方文档说明生成 Schema 与运行它的 Codex 版本精确对应；只有本地重新生成并确认字节一致后，才把 `0.152.1` 提升为已验证版本。对未来版本则保留启动兼容性但不提前宣称 Schema 兼容。

验证结果：

- `codex app-server generate-json-schema` 在本机 `codex-cli 0.152.1` 成功；新旧聚合文件 SHA-256 均为 `d8faa38d5f00aa7ddfe635a2d374ee5f871ffd217d4d175c72fbe7f009f4f669`，`diff --brief` 无差异。
- 新增版本分类测试覆盖三个精确已验证版本、`0.152.2`、`0.153.0-beta.1`、`1.0.0` 及旧版本、Build Metadata、Pre-release、非法前缀和非法 SemVer 拒绝；既有“版本不匹配时不创建运行目录”回归继续通过。
- 本机 `codex-cli 0.152.1` 的真实受管 App Server 完成 `initialize`/`initialized` 握手。Rust Workspace 通过 Rustfmt、Clippy `-D warnings`、95 个常规测试、5 个串行 ignored 真 Socket/真实 Codex 测试、Rustdoc `-D warnings` 与 Release `--workspace --all-targets --all-features --locked` 构建。
- Rust 仓库与总控仓库 `git diff --check` 在最终修改后通过；协议和 Android 仓库没有本里程碑工作区修改。

相关提交：

- 已推送基线为总控 `56eabdf`、Rust `14de45f`、协议 `f9bca8f`、Android `4465dab`，均位于各自 `master`。
- 本里程碑的 Rust 兼容代码、依赖锁定与 README 位于 `14de45f` 后的未提交工作区；本日志位于总控 `56eabdf` 后的未提交工作区。未创建提交、未推送，也未更新 Rust Submodule 指针。

遗留事项：更高 Codex 版本发生不兼容时仍需按实际错误重新生成 Schema 并适配；这符合“默认运行、出错后再兼容”的既定策略。Android CI Artifact 仍需以非 USB 方式干净安装，并完成唯一尚未验证的物理二维码闭环；稳定签名 Release 在该验收之后再规划。

下一目标：不使用 USB，在 Android 15 真机干净安装 CI APK，通过蜂窝公网扫描 Host 终端 QR，完成首次批准、自动 Relay 连接和 Session/Event 恢复验收。

### 2026-09-01 — QR-only 公网首次配对实现与生产路由发布

状态：实现边界已完成并通过自动化、最终生产二进制及 Host 公网临时 route 验证；新 APK 尚未在真机完成实际扫码，作为唯一下一目标保留；本里程碑完成时尚未创建独立提交。

完成内容：

- Pairing v1 Bundle 新增必填、严格规范化的 `relay_endpoint`；首次配对的 Listener 强制只绑定 Loopback，临时 Relay route 从 QR 内 256-bit Bootstrap Token 经 `SHA-256` 存储根及既有 endpoint-bound、domain-separated Relay v1 KDF 派生。Relay 看不到原始 Bootstrap Token 或内层 Pairing 内容。
- Host `pair` 现在要求已运行 Host 和已配置 Relay，先发布临时 route 并等待 Relay 返回 `host_waiting`，最多等待 15 秒；只有公网入口可用后才显示两分钟终端 QR。手工 URI 输出、BLE Advertiser、BlueZ/D-Bus 依赖及其 Linux 构建依赖全部删除，本机批准、五次尝试上限和单次凭据签发保持不变。
- Relay waiting 状态从单一 Slot 改为最多四个互不重叠的认证注册，同一 Host 的稳定设备 route 和临时 QR Pairing route 可以并存；匹配 Client 只消费对应注册，RAII Guard 只清理自身 connection ID，重复 route 继续 fail closed 为 `host_busy`。
- Android 删除全部 Bluetooth 权限、BLE Feature、Nearby Pairing Controller、Deep Link Intent Filter 和手工 Pairing URI 入口；相机成为必需硬件和唯一首次配对 UI。扫码后外层使用平台信任 TLS 连接 QR 指定 Relay，内层继续钉扎临时 Pairing Leaf；成功结果保存规范化 Relay endpoint、选择 Relay 并立即自动连接。
- 协议规范、Rust/Android Fixtures、跨语言 Relay 派生向量及 README 同步 QR-only 约束。通用示例、占位符和测试统一使用 `relay.example.com:2333`/`0.0.0.0:2333`；实际生产地址继续为 `ap.nonamenona.top:19191`。Android Emulator CI 从 API 36 调整为 API 35，避开此前启动后缺失 Package Service 的系统镜像故障。
- Relay `check-config` 输出实际 Bind Address，原子部署脚本从已验证配置解析健康检查端口，不再硬编码生产端口；CI 和 tag Release Workflow 均不再安装已删除 BLE 所需的 `libdbus-1-dev`/`pkg-config`。

关键决策：

- QR 是首次信任与参数绑定的唯一入口，不是 USB/蓝牙失败后的回退。二维码同时绑定 Host 临时证书、Bootstrap Token、有效期和 Relay endpoint；用户无需也不能通过 Deep Link 或手工 URI 建立首次配对。
- 临时 Pairing WSS 不暴露 LAN Socket；公网 Relay 只认证并转发不透明字节，Host 终端批准仍是凭据签发的最终门槛。配对成功后的 LAN route 只作为既有凭据下的显式后续选择，不改变首次配对必须扫码的约束。
- 示例端口固定使用 `2333`，避免把生产部署细节伪装成产品默认值；`19191` 仅保留在生产事实、配置和 Probe 记录中。
- 真机未连接 ADB 时不以 Emulator 或旧 APK 冒充物理扫码验收；代码、协议和公网 route 发布可形成一个已验证实现边界，最终用户链路单独以非 USB 新 APK 扫码作为下一目标。

验证结果：

- Rust 通过 Rustfmt、Workspace Clippy `-D warnings`、Rustdoc `-D warnings`、`--workspace --all-targets --all-features --locked` 常规测试和 Release Build；所有需真实 Loopback/Unix Socket、认证 LAN 及本机 Codex 的 ignored 测试串行通过。一次并行 Cargo 构建期间 Codex 临时运行目录测试失配，独立与串行全量复跑均通过。
- Android 通过 `lintDebug testDebugUnitTest assembleDebug`；新增测试验证扫码成功必定创建已选择 Relay 的 Profile，并拒绝配对期间 Host 身份变化。最终 APK Manifest 确认 Camera 为必需，未包含 Bluetooth 权限、BLE Feature、BROWSABLE Pairing Activity 或 Pairing Deep Link。
- 最终 `agentpulse pair` 在生产 Relay 已有稳定 Native registration 时成功发布第二条临时 registration，收到公网就绪后只输出 QR；中断 Pairing 后临时 registration 清理，稳定 Host 状态继续为 `waiting_or_tunneling`。
- 本地与生产 `/opt/agentpulse-relay/current/agentpulse-relay` 的 SHA-1 均为 `a90f3f5bacd22c47587421d57b9bee0ecd931daf`，部署脚本 SHA-256 均为 `a4390d93138945c4c659d72f03278cace0b01bf089a5be943d5b0d930c1137e4`；systemd 为 `active`，公网 TLS/Relay-v1 Probe 通过。
- Rust/Android GitHub Workflow YAML、部署 Shell Syntax、权威 JSON Fixtures 和四个仓库 `git diff --check` 通过；Cargo/Workflow 不再包含 D-Bus、libdbus 或 bluer 依赖。Android Instrumentation 未执行，因为没有连接真机或 Emulator；未把这一项记为通过。

相关提交：

- 本阶段记录时的已提交基线为总控 `c3a9aa4`、Rust `7478605`、协议规范 `a5122c4` 与 Android `9312b33`。
- 本里程碑全部修改仍位于上述基线后的工作区；未创建提交、未推送，也未更新总控仓库中的 submodule 指针。生产 `a90f3f5...` 是未提交 release 二进制的内容 revision，不是 Git Commit。

遗留事项：新 APK 尚未在 Android 15 真机完成非 USB 扫码，因此实际相机识别、Host 批准、凭据落盘、自动 Relay 连接和 Session/Event 恢复仍需一次物理验收。GitHub `production` Environment 的 Secrets/Variables 及第一次真实 `master` 自动部署仍需用户在网页端配置并观察；生产证书仍选择手工轮换。iOS/HarmonyOS、Bot Channel、Provider 写回和恢复存储未实现，SQLite 仍不是既定依赖。

下一目标：不使用 USB，在 Android 15 真机安装本次新 APK，通过蜂窝公网扫描 Host 终端 QR，完成首次批准、自动 Relay 连接和 Session/Event 恢复验收。

### 2026-09-01 — 可选认证 Relay 只读公网产品闭环

状态：已完成并通过自动化、生产部署及 Android 15 真机公网验证；GitHub `production` Environment 的私密值仍需用户在网页端手工配置，本里程碑完成时尚未创建独立提交。

完成内容：

- 权威协议新增 Relay v1：严格端点语法、4-byte 大端长度前缀、严格 JSON Envelope、Host/Client Challenge Proof、域隔离 HMAC KDF、一次性 Host Enrollment、错误码、心跳、内存 route、64 KiB 双向 buffer、慢端与 idle timeout，并提供九份跨语言 Fixtures 与稳定向量。
- Rust 新增完整 `agentpulse-relay` crate 与 `init/serve/check-config/probe` CLI。公网 Relay 终止平台信任的外层 TLS，只保存经认证 Host 注册的内存 route，并在匹配后泵送不透明的内层 Host-CA TLS；限制一个 Host waiting slot、16 个设备 route、32 个外层连接和有界线程/文件/内存资源。
- Host CLI 新增 `relay configure/status/disable`，Enrollment Token 只从 stdin 读取并以 `0600` 保存；Host 从既有 Native Bearer Token 的 SHA-256 存储根派生每设备 route，不把原 Token 交给 Relay。配对/撤销变化会刷新 route，Loopback Native 仅在 Relay 已配置时允许，LAN 与 Relay 故障域保持独立。
- Android Vault schema 从 v1 原位迁移到 v2，Host Profile 新增可选 Relay endpoint 与显式 LAN/Relay 选择；新增平台信任外层 TLS、SNI/hostname 验证、严格 Relay 控制握手和 SocketFactory Tunnel，随后仍通过 Host CA 校验、Bearer 身份绑定和原 Native v1 Reducer。没有自动 fallback、离线 Session/Event、审批、输入或 Command。
- 修复真机 route 刷新暴露的服务端 waiting-slot 泄漏：Host 在心跳前发现 route 变化会提前断开，旧实现中的 `?` 绕过末尾清理并永久返回 `HostBusy`；改用匹配 connection ID 的 RAII Guard，在所有正常、错误和提前返回路径释放且不会清除后继 slot，并增加回归测试。
- 生产部署新增 hardened systemd Unit、一次性 Bootstrap、专用 `agentpulse-deploy` 用户/Ed25519 Key、严格 40 位 revision、配置/证书预检、原子 release symlink、服务 Probe 与失败回滚。GitHub Workflow 只在 `master` Push 且完整 Rust Job 成功后下载同 SHA Artifact、部署并执行公网 Probe。
- 阿里云 Ubuntu 24.04 LTS 已在安全组开放的 TCP 19191 上部署；证书与私钥归 `root:agentpulse-relay`、配置私密，Nginx 未参与，证书手工轮换。生产当前运行修复后二进制内容 revision `0bb67216ba8a04a844dd2e78a7e3e972efb9ef89`。
- Codex Provider 根据本机真实 `codex-cli 0.152.0` 重新生成完整官方 App Server Schema，Preferred Version 更新为 `0.152.0`；兼容策略使用精确集合 `0.150.1`/`0.152.0`，旧捕获 Fixture 继续通过新 Schema，未知版本仍 fail closed。

关键决策：

- Relay 外层必须使用公网 CA TLS，内层继续使用配对得到的 Host CA TLS；Relay 可以看到连接元数据和字节数，但不能看到 Native Bearer Token、Host Certificate Identity 或 Session/Event 明文。
- route ID、Client Auth Key 和 Proof 全部由现有设备随机 Token 通过 endpoint-bound、domain-separated SHA-256/HMAC 派生；Host Enrollment Token 与设备 Token 不复用，Relay 配置只存 Enrollment Token 的 SHA-256 proof key。
- LAN 与 Relay 都由用户显式选择，客户端只重试当前路径，不做安全语义不透明的自动切换。Android 16 本地网络权限仅在 LAN 路径申请。
- Relay 不引入数据库：进程重启后 Host 必须重新注册 route；撤销最终仍由内层 Native Authorizer 强制，Relay route 快照在有界心跳窗口内刷新。
- Codex 兼容不采用宽松 SemVer 范围。只有生成 Schema、Fixture 和真实 initialize handshake 均验证过的精确 CLI 版本进入 Allowlist；上游再改协议时重新生成并评审。
- GitHub 不持有服务器 TLS 私钥或 Relay Enrollment Token，只持有专用部署 SSH Key；`known_hosts` 必须预先验证，部署 Artifact 必须来自前置 Rust Job。

验证结果：

- Rust 最终通过 Rustfmt、Workspace Clippy `-D warnings`、Rustdoc `-D warnings`、Release `--workspace --all-targets --all-features --locked` Build、91 个常规测试与 2 个 Doctest；额外 5 个 ignored 真 Socket 测试全部通过，其中包含认证 LAN 撤销、2 个 Native Socket、捕获 Codex → Native 以及本机 `codex-cli 0.152.0` 真实 initialize。
- Android 最终通过 `lintDebug testDebugUnitTest connectedDebugAndroidTest assembleDebug`，92 个 Gradle Action 成功；V2282A 真机完成 4 个 Instrumentation Test，其中 1 个既有环境条件测试按设计跳过。Relay JVM 向量与 Rust/权威 Fixture 的 route/proof 字节一致。
- 真机 `10CD5Q1FAS0007Y`（vivo V2282A、Android 15/API 35）在没有 Native ADB reverse 的情况下，通过蜂窝公网与 Host 建立两条不同公网来源的 Relay 外层连接；内层 Native 显示 `connected`，Session `重构SummonGssProjectileTask上下文` 恢复为 Cursor 1、`空闲 · 已连接`。
- 新配对设备在旧 route 快照窗口内先被正确拒绝为不泄漏细节的 `authentication_failed`；Host 刷新、修复后 Relay 重启、旧设备撤销触发的 waiting route 重注册均自动恢复。RAII 回归中服务端没有再次出现 `HostBusy`，随后当前设备重新连接成功。
- 关闭蜂窝数据后默认路由明确不可达、Android 进入可见重试；恢复后无需重启 Host/Relay，手机从 `122.96.32.252` 切到新蜂窝出口 `122.192.14.149` 并自动恢复。手机未加入任何已保存 Wi-Fi，因此未把单独 Wi-Fi 切换宣称为已验证。
- Relay 原子部署会重启生产服务并自动恢复 Host/Android；本地 Host 也完成优雅 stop/start，35 秒内 Provider、Loopback Native、Host Connector、两条公网 Relay 连接和 Android Session 全部恢复。
- 生产服务 `active` 且 `enabled`，以 `agentpulse-relay` 用户运行、无 systemd Restart；`check-config`、公网 `probe`、证书 SAN/Key 匹配和剩余 90 天检查通过。Shell Syntax、GitHub Actions YAML、Relay JSON Fixtures 及四个仓库 `git diff --check` 全部通过。

相关提交：

- 本阶段记录时的已提交基线为总控 `42982d4`、Rust `fd92130`、协议规范 `0a802dc` 与 Android `3d14c40`。
- 本里程碑的 Rust、协议、Android、README、Workflow、部署文件与本日志修改仍位于上述基线后的工作区；未创建里程碑提交、未推送，也未更新总控仓库中的 submodule 指针。生产的 `0bb672...` 是未提交 release 二进制的内容 revision，不是 Git Commit。

遗留事项：GitHub `production` Environment 尚未手工填入两个 Secret 与三个 Variable，因此 Workflow 代码和相同受限部署流程虽已实现并以本机生产式部署通过，尚未由 GitHub Runner 执行第一次真实 `master` 自动部署。生产证书在 2026-11-30 到期且当前选择手工轮换。iOS/HarmonyOS、Bot Channel、Provider 写回和恢复存储仍未实现；SQLite 仍不是既定依赖。

下一目标：配置并验收 GitHub `production` Environment，使第一次真实 `master` push 在完整 Rust CI 后自动部署同 SHA Relay Artifact 并通过公网 Probe。

### 2026-08-30 — Android 原生客户端与认证 LAN 只读产品闭环

状态：已完成并通过自动化及 Android 真机验证；本里程碑完成时尚未创建独立提交。

完成内容：

- Rust 新增 `agentpulse-pairing` 与 `agentpulse-host`，并将 `agentpulse-transport`、`agentpulse-channel-native` 扩展为私有/链路本地地址上的 TLS WebSocket、Bearer 设备认证、HTTP Client ID 与 Native Hello 身份绑定、在线撤销及有界消息/超时/关闭语义；原有 Loopback 模式保持独立。
- Pairing v1 已覆盖 Host CA/Leaf 证书、叶证书指纹钉扎、两分钟单次配对、五次失败上限、本机批准、每设备随机 Token 的哈希存储、最多 16 台设备、撤销、全量凭据轮换、Linux BlueZ 安全 GATT 与 QR 回退。
- Host CLI 覆盖初始化、Codex Thread 管理、服务启动、Codex Remote 启动、配对、设备查询/撤销、凭据轮换、状态和优雅停止，并发布 `_agentpulse._tcp.local.` mDNS 服务；认证 Native WSS 默认使用稳定端口 `49320`，仍允许 `--port` 显式覆盖。
- Android 工程已形成可构建的 App + 纯 JVM Protocol 双模块：严格 Domain/Native/Pairing v1 编解码、UUIDv7、Discovery → Subscribe → Baseline → Live Reducer、256 条 Event 有界状态、CA/指纹 TLS 校验、Android Keystore AES-GCM 凭据、前台连接服务、显式重连与退避、mDNS、BLE Companion Device、CameraX/ML Kit QR、手机/平板 Compose UI、中英文、明暗主题、通知和完整 Release 配置。
- Codex Provider 运行目录改用 Provider UUIDv7 全部 74 个随机位派生的稳定 19 位十六进制键，正常桌面 Host 路径不再超过 Unix Socket 上限，真正过长的根路径仍显式拒绝；Host 顶层错误同时输出完整 source chain。
- Android TLS Client 增加 10 秒 WebSocket Ping，使 Host 停止或半关闭连接能在有界时间内进入可见重试；每次重试重新读取 Vault 最新凭据，mDNS 解析结果仅作为当前连接候选，避免旧重试协程把已撤销 Token 覆盖到刚完成配对的新凭据上。
- Android Compose 启动测试改用 Compose Test API；Vault 仪器测试增加可选的真实 Host 地址/端口断言。Lint 继续对源码 Warning 执行 `warningsAsErrors`，但排除依赖与 AGP “有新版本”这两类依赖网络和时间的非确定性提示，依赖升级改为显式评审。
- 权威协议新增 `pairing-v1.md` 与五份 Pairing Fixtures，Native Transport 规范同步增加认证 LAN、请求头身份绑定和撤销语义；Rust Fixture 镜像测试已经通过。
- CI 已覆盖 Rust 和 Android 的格式/检查/测试/Lint/Debug 构建、API 36 Emulator smoke，以及带外部 Keystore Secrets 的 Release Lint/APK/AAB 构建。

关键决策：

- Native LAN 端口必须跨 Host 重启稳定；`49320` 是产品默认值，配对结果保存该端口，因此同一 LAN 地址上的恢复不依赖热点环境中不可靠的 mDNS。
- Android 与 Host 两端都保留独立心跳和 timeout：服务端清理慢/无响应客户端，客户端 Ping 负责识别 FIN 未完整传播的半关闭 Host 连接并启动退避重连。
- 重连始终以 Vault 中最新 CA/Token 为权威，发现出的地址只影响本次尝试；只有 Pairing Client 能替换持久凭据，网络发现不能携带陈旧 Token 回写整个 Profile。
- 真实 Codex E2E 使用专用 legacy-history Thread `01a05178-cec1-7cb0-920a-9454c0c4eb83`，不污染既有用户 Thread。当前 App Server 可以恢复 legacy Thread；按[官方 App Server 文档](https://developers.openai.com/codex/app-server)，paginated history 的完整读取与 resume 仍会 fail closed，因此不宣称支持该模式。
- Android Debug APK 与 AndroidTest 使用 Android Studio 现有 Debug Keystore 覆盖安装，证书 SHA-256 保持 `8edb9a9adae80ab16358cf043e91c097109a81495e4adfd2fa29bd168590b886`；未卸载 App、未清除用户数据。
- 本阶段严格只读，不新增审批、输入、Command、Relay、数据库、离线历史或 Provider 写回能力。

验证结果：

- Rust 最终通过 Rustfmt、全 Workspace Check、Clippy `-D warnings`、Rustdoc `-D warnings`、Release `--all-targets --all-features` Build、84 个常规测试与 2 个 Doctest；其中包含新增的 Host 稳定默认端口/覆盖回归和 Codex Socket 短路径/真正长路径回归。
- Rust 隔离集成通过 2 个真实 Loopback Native Socket 测试、1 个认证 TLS/LAN 错误凭据/身份绑定/在线撤销测试、1 个捕获 Codex Fixture → Native Socket 测试，以及本机精确 `codex-cli 0.150.1` App Server 初始化测试。
- Android 最终通过 4 个 Protocol JVM Tests、Debug Lint、Debug APK、AndroidTest APK、Release Lint、R8/资源收缩、Unsigned Release APK 与 AAB 构建；因并行 Lint 内存峰值出现过一次退出码 `137`，改用 `--no-daemon --max-workers=1` 后相同门禁完整通过。
- 真机 `10CD5Q1FAS0007Y`（vivo V2282A、Android 15/API 35）通过 Compose `AppLaunchTest`、Vault 重建/磁盘无 Token 与 Host Name 明文测试，以及保存端点 `192.168.77.213:49320` 断言；最终 App、Native 与 Provider 均恢复为健康连接。
- 真机首次配对、本机批准、加密凭据跨进程恢复、只读标记及 Baseline Cursor 通过；专用 Codex Thread 的真实 turn 精确返回 `AgentPulse live event verified.`，Android 会话从 Cursor 1 推进到 5，并显示 `running → message → idle → completed`、Provider `Codex` 与正确工作区。
- Host stop/start 测试中，Android 在约 12 秒内显示未收到 Pong 并进入重试，Host 在同一 `49320` 端口恢复后约 8 秒自动连接；App force-stop/cold-start 后凭据与会话同样恢复。
- 在线撤销使活动设备断开，下一次 Android 连接明确显示 `401 Unauthorized`；重新配对、新 Token 持久化和冷启动恢复通过。独立 TLS Pairing v1 测试端点声明 Native v2 时，Android 明确显示 `Host identity or protocol changed during pairing` 且未覆盖真实 Host。
- 真机慢客户端测试通过：对已连接 App 进程执行可恢复的 `SIGSTOP` 后，Host 在有界 idle timeout 内从 `connected` 清理到 `listening`；`SIGCONT` 后约 3 秒自动恢复为 `connected`，完成态 Cursor 5 未丢失。Rust 的 256 Frame 有界队列测试同时验证溢出会记录 abort reason 并显式失败，不会静默丢帧。
- 22 份权威 Domain/Native/Pairing JSON Fixtures 与 Rust 镜像逐字节一致，全部源码 JSON 语法检查、四个相关仓库的 `git diff --check` 均通过。

相关提交：

- 本阶段开始时的已提交基线为总控 `530a7da`、Rust `85695ad`、协议规范 `aaaab4d` 与 Android `97bd065`。
- 本里程碑的 Rust、协议、Android、README 与本日志修改仍位于上述基线后的工作区；未创建里程碑提交、未推送，也未更新总控仓库中的 submodule 指针。

遗留事项：iOS 与 HarmonyOS 仍为 Scaffold，尚无 Relay、公网连接、Bot Channel 实现或 Provider 写回。手机热点环境没有把 Host mDNS 解析到 Android；已保存的稳定地址与 `49320` 能覆盖同一热点内的 Host 重启，但网络地址变化仍依赖可工作的 mDNS 或重新配对。Codex paginated-history Thread 的 resume 仍受上游 App Server 限制。持久化介质与恢复存储方案继续未决定，SQLite 仍不是既定依赖。

下一目标：实现可选、认证的 Relay 只读公网链路，并在真实外部网络与 Android 设备上完成 Session/Event、身份失败、网络切换、重连、背压和 Host/Relay 重启恢复闭环。

### 2026-08-29 — Native Channel 本地只读 Transport 与同步闭环

状态：已完成并通过验证；本里程碑完成时尚未创建独立提交。

完成内容：

- 扩展 Bridge 与 RuntimeHost 的 Channel 入口：提供有序 Provider/Session Discovery Snapshot、每个 Session 的精确最新 Event Cursor、显式 Subscribe/Unsubscribe，并在 Subscribe Result 中返回 Baseline Sequence。
- 增加 `Persistent` 与 `SourceGeneration` 两种订阅生命周期；RuntimeHost 在停止或释放 Connection-oriented Channel Source 前先清除该 Channel 的 Generation-scoped Subscription，避免断线 Source 继续保留不可见投递关系。
- 新增 `agentpulse-transport`，实现只允许 Loopback Bind 的同步 WebSocket Server；严格校验固定 Path 与 Subprotocol，限制完整消息大小，拒绝 Binary Application Message，并提供有界 Handshake/I/O Poll、Ping、Close 与可停止 Listener 行为。
- 新增完整 `agentpulse-channel-native` Port/Source/Handle Factory：默认绑定 `127.0.0.1:0`，只允许一个活动客户端，公开实际监听地址、健康状态、有界计数器、活动 Client ID 与最后错误。
- Native Channel 精确声明 `NOTIFICATION | SESSION_VIEW | REALTIME_SYNC`，不声明或接受任何 Action；Interaction Event 只能消费 Bridge 给出的 `ObserveOnly` 或 `InteractionReadOnly` Route。
- 定义严格 Native Transport v1：完成 Client/Server Hello、Discovery Batch、Subscribe/Unsubscribe、Subscription Cursor、Domain Delivery Context、稳定 Error Code 与显式重连状态机；全部 Domain Payload 继续嵌套未经改写的 JSON Wire v1 Envelope。
- 实现 Cursor-safe Subscription Ordering：先建立 Bridge 订阅并捕获当前 Baseline Sequence，再严格发送 Subscription Result、Baseline Session 与竞态期间缓冲的 Live Event；不重放历史 Event，也不允许 Live Event 越过 Baseline。
- 对输出采用 256 Frame 默认有界 Queue，对输入输出采用 1 MiB 默认完整消息限制；慢客户端、队列溢出、协议错误、Handshake/Idle Timeout 与网络故障都会执行有界关闭和 Subscription 清理，不进行静默丢帧或隐式重试。
- 新增 11 份权威 Native v1 跨语言 Golden Fixtures 及 Rust 镜像，覆盖全部四种 Client Message 与当前 Server Message Family，并验证严格解码、语义无漂移重编码和总控布局下逐字节一致。
- 使用独立 Tungstenite Client 覆盖真实 TCP/WebSocket Hello → Discovery → Subscribe → Baseline → Live Event/Session → Disconnect Cleanup → Reconnect 流程，并覆盖错误 Path、第二 Client Busy 与不兼容协议 Hello。
- 将捕获的真实 Codex App Server Fixture 接入完整 Provider → RuntimeHost/Bridge → Native Channel → 独立 WebSocket Client 链路，验证 sequence 2–6 实时 Event 与最终完成消息均越过实际 Socket 到达客户端。
- 新增 Native Transport 权威双语规范、Transport/Native crate 使用说明，并同步协议、Rust 与总控 README 的当前能力和边界。

关键决策：

- 首版 Transport 选择 Loopback WebSocket，固定 Path `/agentpulse/native/v1` 与 Subprotocol `agentpulse.native.v1`；默认使用 Ephemeral Port，既不开放 LAN，也不以无认证 Listener 冒充远程连接能力。
- Native 控制协议与 JSON Wire v1 领域协议独立版本化；控制元数据位于外层，Provider Descriptor、Channel Descriptor、Session 与 Event 保持既有严格领域 Envelope，不复制第二套领域 Schema。
- 一个 Native Channel 实例只服务一个显式完成 Hello 的 Client。重连必须新建 WebSocket 并重新执行 Hello、Discovery 与 Subscribe，不持久化或猜测恢复旧订阅。
- Discovery 只给出稳定快照与 Cursor，不隐式订阅；Subscribe 只接受最近 Discovery 中的 Session，并以 Baseline Cursor 作为客户端替换当前 Session View 的权威边界。
- Channel 全程只读，不提供占位 Action Message，也不声明 Approval/Input/Remote Command Capability；完整性体现在发现、同步、实时流、清理、背压和失败语义全部闭合。
- Worker 使用标准库线程和同步 RuntimeHost Handle，不引入异步运行时；Transport 只依赖现有无 TLS Tungstenite Handshake 能力。
- 本阶段不引入数据库、持久化、历史回放、ACK、自动重试、TLS、认证、Relay、LAN、公网、Native UI 或 Provider 写回。

验证结果：

- 全 Workspace 73 个常规单元/集成测试与 2 个 Doctest 全部通过；其中包含 18 个 Bridge/RuntimeHost、7 个 Native Channel、25 个 Core、9 个 Protocol、12 个 Codex Provider 与 2 个 Transport 测试。
- 在允许创建 Loopback Socket 的环境中，2 个独立 Native Client 真实 Socket 测试全部通过；另行通过 1 个捕获 Codex Fixture 到 Native WebSocket 的完整端到端测试。
- Rustfmt Check、全 Workspace `cargo check --all-targets`、Clippy `-D warnings`、Rustdoc `-D warnings`、锁定依赖的离线测试与 Release Build 全部通过。
- 11 份权威 Native Fixture 与 11 份 Rust 镜像均通过 JSON 语法校验；Fixture 测试同时验证语义 Round-trip 与总控布局下逐字节一致。
- 依赖树确认 Native Channel 直接复用 Core、Bridge、Protocol、Transport、Serde JSON、Thiserror、UUID 与仅启用 Handshake 的 Tungstenite；Transport 只直接依赖 Thiserror 与 Tungstenite，未引入 Tokio、TLS、数据库或持久化依赖。
- 总控、Rust 与协议规范三个仓库的 Diff/Whitespace Check 全部通过。

相关提交：

- 本次工作的已提交基线为总控 `4774824`、Rust `bb38276` 与协议规范 `d4e6442`，均位于对应 `origin/master`。
- 本里程碑的实现、测试、Fixtures 与文档位于上述基线后的工作区，提交哈希留待后续提交时记录。

遗留事项：Rust 侧本地只读服务链路已经完整，但 Android、iOS 与 HarmonyOS 仓库仍只有 Scaffold；Loopback Listener 也不能直接供手机使用。因此当前尚无真实原生 UI，也没有认证 LAN、Relay 或公网连接。

下一目标：实现 Android 原生客户端与认证 LAN 直连的完整只读 Session/Event 展示闭环。

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
