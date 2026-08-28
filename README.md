# AgentPulse

> Stay connected to your coding agents.

AgentPulse is a self-hosted native mobile companion system for AI coding agents. It connects to CLI agents through hooks, plugins, or protocol integrations, normalizes their runtime events, and synchronizes task state to mobile clients.

AgentPulse 是面向 AI Coding Agent 的自托管原生移动伴侣系统。它通过 Hook、Plugin 或协议接口连接本地 CLI Agent，统一处理运行状态、Plan/Todo、工具调用、等待输入、权限请求、完成和失败等事件，并将任务状态同步到移动端。

When an adapter and the underlying agent support it, users can also perform lightweight remote interactions such as approving or rejecting an operation, answering a question, or submitting short text.

当 Adapter 与对应 Agent 支持时，用户还可以在移动端批准或拒绝操作、回答问题，或提交简短文本。

## Repositories / 仓库组成

| Repository | Responsibility / 职责 |
| --- | --- |
| `agentpulse-rs` | Shared Rust core, bridge, relay, transports, and CLI agent adapters / Rust 核心、桥接、同步服务、传输层与 Agent 适配器 |
| `agentpulse-android` | Native Android companion application / Android 原生移动客户端 |
| `agentpulse-ios` | Native iOS companion application / iOS 原生移动客户端 |
| `agentpulse-harmony` | Native HarmonyOS companion application / HarmonyOS 原生移动客户端 |
| `agentpulse-protocol` | Canonical cross-platform protocol specification / 跨平台协议规范 |

This repository is the umbrella repository for the AgentPulse project. Each component remains an independent Git repository and is included here as a submodule.

本仓库是 AgentPulse 的总控仓库。各组件保留独立的 Git 历史，并通过 submodule 聚合到这里。

## Getting the source / 获取源码

```bash
git clone --recurse-submodules <agentpulse-repository-url>
```

For an existing clone:

```bash
git submodule update --init --recursive
```

## Status / 状态

AgentPulse is currently in its initial architecture and repository-bootstrap phase.

AgentPulse 目前处于架构设计与仓库初始化阶段。
