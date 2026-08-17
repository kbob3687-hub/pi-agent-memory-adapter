<p align="center"><b>简体中文</b> · <a href="./README.md">English</a></p>

# TencentDB Agent Memory — Pi 适配器

> 一个本地 Pi 扩展，让 **Pi**（[pi.dev](https://pi.dev)）通过
> [TencentDB Agent Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory)
> 获得跨会话的持久记忆与可沉淀的技能。

**基于官方 SDK，不手写 HTTP、不走私有协议。** 本仓库只包含 **Pi 适配器**——
连接 Pi 与 TencentDB Agent Memory 服务的胶水。它**不包含** MemoryCore 引擎、
SDK、MemoryProxy 的源码；那些是作为依赖使用的。

---

## 它能做什么

- **跨会话记忆** —— 每次 Pi 回答前，自动召回有界的 L0 对话证据、L1 原子记忆、
  相关 L2 场景笔记和 L3 核心画像，并作为显式**不可信**上下文注入。
- **可靠采集** —— 回合落定后，把对话持久化到跨进程的 durable outbox 并投递到
  服务；离线期间的记录稍后自动补投；全程 fail-open（记忆服务挂了绝不阻塞 Pi）。
- **技能学习** —— 工具密集的回合按五角色会话采集，服务端审查模型把它们提炼成
  可复用的 `SKILL.md`，既会自动召回，也能**同步成 Pi 的原生技能**。
- **分支隔离** —— Pi `/tree` 分支获得独立的记忆身份；`/fork` 天然隔离。
- **setup 向导 + 只读工具** —— `/tdai-memory-setup`、`/tdai-memory-status`、
  `/tdai-memory-sync-skills`，外加四个只读搜索工具。

## 本仓库的范围

这是 **Pi 适配器本身**。其余组件全部来自官方生态：

| 组件                             | 在哪里                                              |
| -------------------------------- | --------------------------------------------------- |
| Pi 适配器（本仓库）              | 你的 Pi 扩展，约 3.1k 行 TypeScript                 |
| 官方 SDK                         | npm 上的 `@tencentdb-agent-memory/memory-sdk-ts-v2` |
| MemoryCore（记忆 + 技能引擎）    | 官方 Docker 镜像 `agentmemory/memory-core`          |
| Memory Hub（管理面板）           | 官方镜像 `agentmemory/memory-hub`                   |
| MemoryProxy（其它 agent 连接器） | 官方镜像 `agentmemory/memory-proxy`                 |

上游项目：<https://github.com/TencentCloud/TencentDB-Agent-Memory>

## 架构

```text
Pi 会话 ── 适配器（本仓库，Pi 原生扩展）── 官方 SDK ── MemoryCore
   │
   ├─ before_agent_start → 召回 L0-L3 + 相关技能，注入为「不可信」上下文
   ├─ agent_settled      → 采集回合（L0 走 durable outbox；技能 at-most-once）
   └─ 命令 / 工具        → setup / status / sync-skills / 4 个只读搜索工具
```

## 前置条件

- Node.js `>= 22.19.0`
- Pi `0.84.1`（已验证；peer 范围 `>=0.84.1 <0.85`）
- 正在运行的 MemoryCore，已有 Team、Agent、User 和 User Key
  （如何启动见上游 README）

## 快速开始

```powershell
git clone https://github.com/kbob3687-hub/pi-agent-memory-adapter.git
cd pi-agent-memory-adapter
npm ci
npm run check
```

启动 MemoryCore 后，用适配器加载 Pi：

```powershell
pi -e <本仓库路径>
```

在 Pi 里运行 `/tdai-memory-setup` 走完向导即可开始使用。想体验技能学习，在
`~/.pi/agent/tdai-memory.json` 加一行 `"skills": { "enabled": true }`，做完一个
工具密集的任务后运行 `/tdai-memory-sync-skills`。

完整使用说明（配置、命令、端到端检查、卸载）见 [USAGE_CN.md](./USAGE_CN.md)。

## 开发

```powershell
cd adapters\pi
npm run check            # typecheck + 119 单测
npm run verify:pi-load   # 离线 Pi 加载检查
npm run e2e:skill        # 对真实 MemoryCore 跑完整技能闭环 E2E（需 Docker + LLM 密钥）
```

端到端套件覆盖 setup 向导、生命周期可靠性、L0–L3 召回注入和完整技能闭环——全部
对着真实 MemoryCore 和真实 Pi 验证。

## 许可证

MIT。本仓库是独立适配器，与 TencentDB Agent Memory 项目无关联、也不代表其背书。
底层服务遵循其自身的许可证。

- TencentDB Agent Memory：<https://github.com/TencentCloud/TencentDB-Agent-Memory>
- CHANGELOG：<https://github.com/TencentCloud/TencentDB-Agent-Memory/blob/feat/server_team/CHANGELOG.md>
