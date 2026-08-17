<p align="center"><b>English</b> · <a href="./README_CN.md">简体中文</a></p>

# TencentDB Agent Memory — Pi Adapter

> 一个本地 Pi 扩展，让 **Pi**（[pi.dev](https://pi.dev)）通过
> [TencentDB Agent Memory](https://github.com/TencentCloud/TencentDB-Agent-Memory)
> 获得跨会话的持久记忆与可沉淀的技能。

**Built on the official SDK. No hand-rolled HTTP, no private protocol.** This
repo contains **only the Pi adapter** — the glue that connects Pi to the
TencentDB Agent Memory service. It does **not** include the MemoryCore engine,
the SDK, or MemoryProxy source; those are used as dependencies.

---

## What it does

- **Cross-session memory** — before each Pi run, recalls bounded L0 conversation
  evidence, L1 atomic memories, relevant L2 scenario notes, and the L3 core
  profile, injected as explicitly **untrusted** context.
- **Reliable capture** — after a turn settles, persists the exchange to a
  durable cross-process outbox and delivers it to the service; offline periods
  are caught up later; fail-open everywhere (memory down never blocks Pi).
- **Skill learning** — tool-heavy turns are captured as five-role conversations;
  the server's review model mines them into reusable `SKILL.md` files, which are
  recalled automatically and can be **synced into Pi's native skills**.
- **Branch isolation** — Pi `/tree` branches get independent memory identities;
  `/fork` is naturally isolated.
- **Setup wizard + read-only tools** — `/tdai-memory-setup`, `/tdai-memory-status`,
  `/tdai-memory-sync-skills`, plus four read-only search tools.

## Scope of this repository

This is the **Pi adapter only**. Everything else comes from the official
ecosystem:

| Component                            | Where it lives                                    |
| ------------------------------------ | ------------------------------------------------- |
| Pi adapter (this repo)               | your Pi extension, ~3.1k lines of TypeScript      |
| Official SDK                         | `@tencentdb-agent-memory/memory-sdk-ts-v2` on npm |
| MemoryCore (memory + skill engine)   | official Docker image `agentmemory/memory-core`   |
| Memory Hub (management panel)        | official `agentmemory/memory-hub` image           |
| MemoryProxy (other-agent connectors) | official image `agentmemory/memory-proxy`         |

Upstream project: <https://github.com/TencentCloud/TencentDB-Agent-Memory>

## Architecture

```text
Pi session ── adapter (this repo, native Pi extension) ── official SDK ── MemoryCore
   │
   ├─ before_agent_start → recall L0-L3 + relevant skills, injected as UNTRUSTED context
   ├─ agent_settled      → capture the turn (L0 via a durable outbox; skills at-most-once)
   └─ commands / tools   → setup / status / sync-skills / 4 read-only search tools
```

## Requirements

- Node.js `>= 22.19.0`
- Pi `0.84.1` (verified; peer range `>=0.84.1 <0.85`)
- A running MemoryCore with an existing Team, Agent, User, and User Key
  (see the upstream README to start one)

## Quick start

```powershell
git clone https://github.com/kbob3687-hub/pi-agent-memory-adapter.git
cd pi-agent-memory-adapter
npm ci
npm run check
```

Start MemoryCore, then run Pi with the adapter loaded:

```powershell
pi -e <path-to-this-repo>
```

In Pi, run `/tdai-memory-setup`, finish the wizard, and start using it. To try
skill learning, add `"skills": { "enabled": true }` to `~/.pi/agent/tdai-memory.json`,
complete a tool-heavy task, then run `/tdai-memory-sync-skills`.

See [USAGE.md](./USAGE.md) for the full usage guide (configuration, commands,
end-to-end checks, uninstall).

## Development

```powershell
cd adapters\pi
npm run check            # typecheck + 119 unit tests
npm run verify:pi-load   # offline Pi load check
npm run e2e:skill        # full skill-loop E2E against a real MemoryCore (needs Docker + LLM keys)
```

End-to-end suites cover the setup wizard, lifecycle reliability, L0–L3 recall
injection, and the complete skill loop — all against a real MemoryCore and a
real Pi.

## License

MIT. This repository is an independent adapter; it is not affiliated with or
endorsed by the TencentDB Agent Memory project. The underlying service is
licensed by its own terms.

- TencentDB Agent Memory: <https://github.com/TencentCloud/TencentDB-Agent-Memory>
- Changelog: <https://github.com/TencentCloud/TencentDB-Agent-Memory/blob/feat/server_team/CHANGELOG.md>
