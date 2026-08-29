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

The native Android, iOS, and HarmonyOS applications remain the full AgentPulse experience, including session browsing, plans, progress, real-time events, approvals, input, LAN connections, and public-network connections through the optional Relay. Bot Channels provide a lightweight, degraded experience constrained by each third-party platform and may communicate without the AgentPulse Relay.

Android、iOS 与 HarmonyOS 原生应用仍然提供完整的 AgentPulse 体验，包括 Session 浏览、Plan、进度、实时事件、审批、输入、LAN 直连，以及通过可选 Relay 进行公网连接。Bot Channel 是受第三方平台能力限制的轻量兼容通道，并且可以不经过 AgentPulse Relay。

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

The Rust workspace, channel-neutral domain model, deterministic Core Reducer, strict JSON protocol v1, independent Provider/Channel ports, centralized capability routing, multi-endpoint Bridge orchestration, and runtime-neutral Adapter lifecycle hosting are implemented. The next target is a minimal read-only Codex Provider; Relay, transport, and concrete Channel runtime implementations remain planned.

Rust workspace、与 Channel 无关的统一领域模型、确定性 Core Reducer、严格 JSON 协议 v1、Provider/Channel 独立端口、集中 Capability 路由、Bridge 多端点编排与运行时中立的 Adapter 生命周期托管已经完成。下一目标是最小只读 Codex Provider；Relay、Transport 及具体 Channel 运行时实现仍处于规划阶段。

Verified milestones, current constraints, and the next target are maintained in the [Development Log / 开发日志](DEVELOPMENT_LOG.md).

已经验证的里程碑、当前约束和下一目标统一维护在[开发日志](DEVELOPMENT_LOG.md)中。
