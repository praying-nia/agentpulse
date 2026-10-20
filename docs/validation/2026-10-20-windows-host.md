# Windows Host 本机验收（2026-10-20）

## 范围与环境

- Windows x86_64；隔离安装的 Rust 1.99.1，目标 `x86_64-pc-windows-gnu`。
- 使用已有 MinGW GCC 8.1.0；缩短工作区构建目录以避开该旧工具链的长路径限制。
- 本机 Codex CLI `0.163.0-alpha.10.2`；按现有策略以 best-effort 模式启动，不将它加入完整协议已验证版本列表。
- 总仓库基线 `31dbcfc`，Rust 基线 `bf0a13e`，Rust 签名修复提交 `a736c78`。
- 只验证本地 Host / Provider、私有存储、管理 IPC 和配对协议；没有启动模型 Turn，没有手机或公网端到端验收。

## 修复与检查

1. Windows Codex 解析器补齐 npm 将官方可选平台包放在 `node_modules/@openai/codex-win32-*` 同级目录的布局。保留原有嵌套和旧版 vendor 布局，仍只启动原生 `.exe`。
2. 新增回归覆盖同级平台包的 `bin` / `codex` 目录、`.cmd` / `.ps1` 入口、中文和空格路径、缺失二进制及自定义包装器拒绝。
3. Host 清理状态文件时移除单元素循环，通过新版 Clippy 的严格检查；错误处理行为不变。
4. 平台 CI 增加配对 socket 测试步骤，覆盖此前被默认忽略的五项测试。远端 CI 结果尚未确认，不能以本机 GNU 验收代替 MSVC CI 结果。

## 验证结果

在 Rust 仓库中执行；本机另使用 `--offline` 和工作区内的 Cargo 缓存：

```text
cargo test -p agentpulse-platform -p agentpulse-provider-codex -p agentpulse-pairing -p agentpulse-host --locked -- --test-threads=1
cargo test -p agentpulse-pairing --locked -- --include-ignored --test-threads=1
cargo test -p agentpulse-provider-codex installed_ --locked -- --ignored --test-threads=1
cargo test -p agentpulse-host --test local_cli --locked -- --include-ignored --test-threads=1
cargo fmt --all --check
cargo clippy -p agentpulse-platform -p agentpulse-provider-codex -p agentpulse-pairing -p agentpulse-host --all-targets --no-deps --locked -- -D warnings
```

以上最终检查通过。首轮常规回归 72 项；配对全量 13 项，其中新增执行 5 项；真实 Codex 2 项；Host CLI 全量 3 项，其中新增执行真实 Host 1 项。去重后共 **80 项测试通过**。四个相关包的 `cargo doc --no-deps --locked` 配合 `RUSTDOCFLAGS=-D warnings` 也通过。

- 实际检查文件/目录/命名管道 ACL、原子替换与共享冲突、消息限长、超时、管道复用、Job Object 根进程及后代清理。
- 真实 Codex 完成初始化、代理 `model/list` 往返、异常客户端隔离和两次启停。
- 最终 Host CLI 三项测试共 14.40 秒，通过 `serve → status → stop → restart`。另一个独立 LAN 诊断实例也能响应管理命令并正常停止。
- 首次真实 Host 验收曾在 30 秒就绪期限内超时；独立实例、加入阶段诊断后的复跑、移除诊断后的最终串行复跑均通过。没有确定根因，不将其记录为已修复缺陷；若再现，应继续定位并保留启动阶段证据。
- 旧 MinGW 在部分测试链接时报告 `corrupt .drectve` 警告；测试实际通过，严格 Clippy 通过。未通过屏蔽警告更改项目配置。

## Git 状态与遗留事项

Rust 修复已签名提交并推送至 `fix/windows-host-validation`。首次未签名推送曾被 GitHub 的 verified-signature 规则拒绝；用户登记现有 SSH 签名公钥后，补签名推送成功，没有绕过规则。签名配置只写入本次两个仓库。总仓库在本次提交中同步 Rust 指针、使用手册和开发日志。工作区内临时工具链、依赖缓存和构建产物已清理。

本机验收不覆盖 Windows ARM64、MSVC CI、其他用户账户访问隔离、手机扫码、真实模型 Turn 或公网链路。原有控制连接退出后回复未回传的问题仍待处理。

唯一下一目标：在 Windows Host 上复现并定位控制连接退出后，手机发起消息的模型回复未回传的问题。
