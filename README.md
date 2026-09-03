# AgentPulse

> Stay connected to your coding agents.

AgentPulse is a self-hosted, multi-channel interaction system for AI coding agents. It connects to CLI agents through Providers, normalizes their runtime events and interactions, and delivers them through Channels such as the native AgentPulse apps, Feishu Bot, QQ Bot, and Webhook.

AgentPulse 是面向 AI Coding Agent 的自托管多渠道交互系统。它通过 Provider 连接本地 CLI Agent，统一处理运行状态、Plan/Todo、工具调用、等待输入、权限请求、完成和失败等事件，再通过 Native App、飞书 Bot、QQ Bot、Webhook 等 Channel 将信息交付给用户。

When both the Provider and Channel support a capability, users can also perform remote interactions such as approving or rejecting an operation, answering a question, submitting short text, or sending a simple command.

当 Provider 与 Channel 同时支持相应能力时，用户还可以远程批准或拒绝操作、回答问题、提交简短文本或发送简单指令。

## Architecture / 架构

```text
Codex / Claude Code / OpenCode / DeepSeek Harness
                         ↓
                     Provider
                         ↓
                 AgentPulse Core
                         ↓
                      Channel
             ┌───────────┼───────────┐
             ↓           ↓           ↓
          Native      Feishu/QQ    Webhook
     Android/iOS/HarmonyOS
```

- A **Provider** determines how AgentPulse receives information from an AI agent and, where supported, writes responses back to it.
- A **Channel** determines how AgentPulse presents information to a user and receives that user's response.
- Provider and Channel implementations are independent and communicate only through the shared domain and protocol models.

- **Provider** 负责 AgentPulse 如何从 AI Agent 获取信息，并在能力允许时向 Agent 回写响应。
- **Channel** 负责 AgentPulse 如何向用户展示信息并接收用户响应。
- Provider 与 Channel 彼此独立，仅通过统一的领域模型和协议模型协作。

The native Android, iOS, and HarmonyOS applications are intended to become the full AgentPulse experience, including session browsing, plans, progress, real-time events, approvals, input, LAN connections, and public-network connections through the optional Relay. Native Transport v3 now retains the current Host run in memory and incrementally repairs client gaps after disconnects. Bot Channels provide a lightweight, degraded experience constrained by each third-party platform and may communicate without the AgentPulse Relay.

Android、iOS 与 HarmonyOS 原生应用的目标是提供完整 AgentPulse 体验，包括 Session 浏览、Plan、进度、实时事件、审批、输入、LAN 直连，以及通过可选 Relay 进行公网连接；Native Transport v3 会在内存中保留当前 Host 运行的完整事件，并在断连后增量修复客户端缺口。Bot Channel 是受第三方平台能力限制的轻量兼容通道，并且可以不经过 AgentPulse Relay。

## Repositories / 仓库组成

| Repository | Responsibility / 职责 |
| --- | --- |
| `agentpulse-rs` | Shared Rust core, bridge, optional relay, transports, Providers, and Channels / Rust 核心、桥接、可选 Relay、传输层、Provider 与 Channel |
| `agentpulse-android` | Native Android companion application / Android 原生移动客户端 |
| `agentpulse-ios` | Native iOS companion application / iOS 原生移动客户端 |
| `agentpulse-harmony` | Native HarmonyOS companion application / HarmonyOS 原生移动客户端 |
| `agentpulse-protocol` | Canonical, channel-neutral cross-platform protocol specification / 与 Channel 无关的跨平台协议规范 |

This repository is the umbrella repository for the AgentPulse project. Each component remains an independent Git repository and is included here as a submodule.

本仓库是 AgentPulse 的总控仓库。各组件保留独立的 Git 历史，并通过 submodule 聚合到这里。

Bot Channels are implemented inside `agentpulse-rs`; they do not require separate `agentpulse-feishu`, `agentpulse-qq`, or similar repositories.

Bot Channel 直接在 `agentpulse-rs` 内实现，不新增 `agentpulse-feishu`、`agentpulse-qq` 等独立仓库。

## Getting the source / 获取源码

```bash
git clone --recurse-submodules <agentpulse-repository-url>
```

For an existing clone:

```bash
git submodule update --init --recursive
```

## Status / 状态

AgentPulse now provides an Android observation, approval, atomic Plan-form, and common remote-command path from the managed App Server through RuntimeHost/Bridge and authenticated Native TLS over either private LAN or the public Relay. First trust is established only by scanning a short-lived Host-terminal QR code: its route is published through Relay, still requires Host-terminal approval, and needs no USB, ADB, Bluetooth, or shared LAN. Native Transport v3 retains the complete current Host-run history in memory and incrementally repairs per-Session gaps after ordinary disconnects; Android keeps the matching process-memory cache and resets it when the Host run changes. Remote prompts use bounded process-memory FIFO queues, and `/resume` hydrates message history through paginated App Server reads. iOS and HarmonyOS remain scaffolds; cross-process persistence remains outside the current boundary.

AgentPulse 现已形成从受管 Codex App Server 经 RuntimeHost/Bridge、认证 Native TLS，并通过私有 LAN 或公网 Relay 到 Android 原生 App 的观察、审批、原子 Plan 表单与常用远程指令链路。首次信任只允许扫描 Host 终端生成的短时二维码：临时路由经 Relay 发布，仍需 Host 终端确认，并且不依赖 USB、ADB、蓝牙或共享局域网。Native Transport v3 在内存中保留当前 Host 运行周期的完整历史，普通断连后按 Session 增量补齐；Android 保留对应的进程内缓存，Host 运行周期改变时明确重置。远程 Prompt 使用有界进程内 FIFO，`/resume` 通过 App Server 分页恢复消息历史。iOS 与 HarmonyOS 仍为 Scaffold；跨进程持久化不在当前边界内。

Verified milestones, current constraints, and the next target are maintained in the [Development Log / 开发日志](DEVELOPMENT_LOG.md).

已经验证的里程碑、当前约束和下一目标统一维护在[开发日志](DEVELOPMENT_LOG.md)中。
