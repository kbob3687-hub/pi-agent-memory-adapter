<p align="center"><b>简体中文</b> · <a href="./README.md">English</a></p>

# TencentDB Agent Memory — Pi 适配器

> 给 Pi 装上一个真正的记忆：跨会话召回、技能学习、扛得住崩溃和断网的可靠采集——
> 基于官方 TencentDB Agent Memory SDK 构建。

**Pi 是支持原生扩展的 agent，所以这个适配器能做到协议代理做不到的事：接入 Pi 的真实
生命周期、配对真实工具调用、交付"可靠 + 安全 + 真能复用"的记忆。**

---

## 为什么需要它

Pi 默认没有跨会话记忆。每次新开会话都忘记你是谁、做过什么、之前解决过什么问题——同样的
坑反复踩、同样的上下文反复讲。这个适配器让 Pi 通过 TencentDB Agent Memory 拥有长期记忆，
并且能从你的工作里**学习可复用的技能**。

## 和别的方案有什么不同

三个其它 Pi 集成给不了的东西：

1. **原生，不是代理。** Pi 支持扩展，所以适配器接入真实生命周期（`before_agent_start`、
   `agent_settled`）、按 id 配对真实的 `tool_call` / `tool_result` 事件、注册原生只读工具。
   不需要协议桥接，不需要教模型去 `curl`。

2. **可靠，不是尽力而为。** 采集的回合经过**跨进程 durable outbox**（lease 保护、离线补投、
   死信隔离）——记忆扛得住崩溃和断网。技能写入是 **at-most-once**：绝不重发、绝不污染服务端
   累计缓冲。

3. **安全是设计出来的。** 召回的记忆永远作为 **UNTRUSTED** 上下文注入——它是参考资料，
   不是指令，被污染的记忆无法劫持模型。密钥在落盘前**递归**脱敏。而且全程 **fail-open**：
   记忆挂了，Pi 照常回答，只是那一轮没有记忆。

## 你能得到什么

| 能力 | 实际发生什么 |
|---|---|
| **跨会话记忆** | 每次回答前，召回 L0 对话证据、L1 原子记忆、L2 场景笔记、L3 画像——有界且不可信 |
| **技能学习** | 工具密集的回合 → 五角色采集 → 服务端审查模型提炼 `SKILL.md` → 相关问题自动召回，也能同步成 Pi 原生 `/skills` |
| **可靠采集** | durable outbox：at-least-once、离线补投、死信隔离；一次失败不阻塞下一轮 |
| **分支隔离** | Pi `/tree` 分支获得独立记忆身份；`/fork` 天然隔离 |
| **一键 setup** | `/tdai-memory-setup` 验证身份 + L0–L3 访问 + 创建私有 Agent，复用团队可见 Agent 前会警告 |
| **只读工具** | `tdai_memory_search`、`tdai_conversation_search`、`tdai_skill_search`、`tdai_skill_read`——模型直接查记忆 |

## 为什么不用 MemoryProxy 或直接用 SDK？

官方 SDK 是正确的底座。但**agent 适配才是难的部分**——而官方 MemoryProxy 只覆盖走协议的 agent。

| | MemoryProxy | 裸 SDK | 本适配器 |
|---|---|---|---|
| 需要代理挡在 LLM 前面 | ✅ | — | ❌ |
| 接入 Pi 真实生命周期（`before_agent_start`） | ❌ | ❌ | ✅ |
| 按 id 配对真实 `tool_call`/`tool_result` | ⚠️ 从文本猜 | ❌ | ✅ |
| 原生只读工具 | ❌ | ❌ | ✅ |
| 跨进程 durable outbox | ❌ | ❌ | ✅ |
| 离线恢复 + 死信 | ❌ | ❌ | ✅ |
| 分支隔离（`/tree`） | ❌ | ❌ | ✅ |
| 同步成 Pi 原生技能（可编辑） | ❌ | ❌ | ✅ |
| 召回作为不可信上下文 | 部分 | ❌ | ✅ |
| `peerDependencies` 版本守卫 | — | — | ✅（`>=0.84.1 <0.85`） |

## 安全模型

- **不可信边界** —— 召回的记忆包在显式信任边界里；它是参考资料，绝不是指令。
- **递归脱敏** —— 工具参数和结果做结构化脱敏（敏感 key 整体遮蔽），不只是正则替换。
- **配置沙箱** —— 项目级配置只能调 `recall`；永远改不了 endpoint、身份和密钥。
- **强制 TLS** —— 证书校验不能关闭；远程端点必须 HTTPS。
- **格式 fail-closed** —— 如果 Pi 将来改消息结构，采集降级为不采集，绝不静默错解。

## 架构

```text
Pi 会话 ── 适配器（本仓库，原生扩展）── 官方 SDK ── MemoryCore
   │
   ├─ before_agent_start → 召回 L0-L3 + 相关技能 → 注入为 UNTRUSTED 上下文
   ├─ tool_call/result   → 按 id 配对、脱敏、字节有界
   ├─ agent_settled      → L0 走 durable outbox（at-least-once）· 技能 at-most-once
   └─ 命令 / 工具        → setup · status · sync-skills · 4 个只读搜索工具
```

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

在 Pi 里：运行 `/tdai-memory-setup` 走完向导即可开始使用。想体验技能学习，在
`~/.pi/agent/tdai-memory.json` 加一行 `"skills": { "enabled": true }`，做完一个
工具密集的任务后运行 `/tdai-memory-sync-skills`。

完整使用说明（配置、命令、端到端检查、卸载）见 [USAGE_CN.md](./USAGE_CN.md)。

## 前置条件

- Node.js `>= 22.19.0`
- Pi `0.84.1`（已验证；peer 范围 `>=0.84.1 <0.85`）
- 正在运行的 MemoryCore，已有 Team、Agent、User 和 User Key

## 开发与验证

```powershell
cd adapters\pi
npm run check            # typecheck + 119 单测
npm run verify:pi-load   # 离线 Pi 加载检查
npm run e2e:skill        # 完整技能闭环 E2E（真实 MemoryCore + 真实 Pi）
```

四套真实端到端套件覆盖 setup 向导、生命周期可靠性、L0–L3 召回注入和完整技能闭环——
全部对着真实 MemoryCore 和真实 Pi 验证，所以上面的承诺是**验证过**的，不是嘴上说的。

## 许可证

MIT。本仓库是独立适配器，与 TencentDB Agent Memory 项目无关联、也不代表其背书。
底层服务遵循其自身的许可证。

- TencentDB Agent Memory：<https://github.com/TencentCloud/TencentDB-Agent-Memory>
- CHANGELOG：<https://github.com/TencentCloud/TencentDB-Agent-Memory/blob/feat/server_team/CHANGELOG.md>
