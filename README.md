<p align="center"><b>English</b> · <a href="./README_CN.md">简体中文</a></p>

# TencentDB Agent Memory — Pi Adapter

> Give Pi a real memory: cross-session recall, skill learning, and capture that
> survives crashes and outages — built on the official TencentDB Agent Memory SDK.

**Pi is a native-extension agent, so this adapter does what a protocol proxy
can't: it hooks Pi's real lifecycle, pairs real tool calls, and delivers
memory that is reliable, safe, and actually reusable.**

---

## Why this adapter exists

Pi has no memory across sessions. Every new session forgets who you are, what
you've built, and what it already solved — so the same pitfalls get re-hit and
the same context gets re-explained. This adapter gives Pi a long-term memory
through TencentDB Agent Memory, and lets it **learn reusable skills from your
work**.

## What makes it different

Three things no other Pi integration gives you:

1. **Native, not proxied.** Pi supports extensions, so the adapter hooks the
   real lifecycle (`before_agent_start`, `agent_settled`), pairs real
   `tool_call` / `tool_result` events by id, and registers native read-only
   tools. No protocol bridging, no teaching the model to `curl`.

2. **Reliable, not best-effort.** Captured turns flow through a **cross-process
   durable outbox** (lease-protected, offline catch-up, dead-letter quarantine)
   — memory survives crashes and outages. Skill writes are **at-most-once**:
   never re-appended, never polluting the server's cumulative buffer.

3. **Safe by construction.** Recalled memory is always injected as
   **UNTRUSTED** context — it is reference data, never instructions, so a
   poisoned memory can't hijack the model. Secrets are redacted **recursively**
   before persistence. And it **fails open**: if memory is down, Pi answers
   normally, it just has no memory that turn.

## What you get

| Capability | What actually happens |
|---|---|
| **Cross-session memory** | before each answer, recalls L0 conversation evidence, L1 atomic memories, L2 scenario notes, and the L3 profile — bounded and untrusted |
| **Skill learning** | tool-heavy turns → five-role capture → the server's review model mines `SKILL.md` → auto-recalled on related questions, and syncable into Pi's native `/skills` |
| **Reliable capture** | durable outbox: at-least-once, offline catch-up, dead-letter quarantine; one failure never blocks the next turn |
| **Branch isolation** | Pi `/tree` branches get independent memory identities; `/fork` is naturally isolated |
| **One-command setup** | `/tdai-memory-setup` verifies identity + L0–L3 access, creates a private agent, and warns before reusing a team-visible one |
| **Read-only tools** | `tdai_memory_search`, `tdai_conversation_search`, `tdai_skill_search`, `tdai_skill_read` — the model queries memory directly |

## Why not just MemoryProxy or the raw SDK?

The official SDK is the right foundation. But **agent adaptation is the hard
part** — and the official MemoryProxy only covers protocol-based agents.

| | MemoryProxy | Raw SDK | This adapter |
|---|---|---|---|
| Requires a proxy in front of the LLM | ✅ | — | ❌ |
| Hooks Pi's real lifecycle (`before_agent_start`) | ❌ | ❌ | ✅ |
| Real `tool_call`/`tool_result` pairing by id | ⚠️ inferred from text | ❌ | ✅ |
| Native read-only tools | ❌ | ❌ | ✅ |
| Durable cross-process outbox | ❌ | ❌ | ✅ |
| Offline recovery + dead-letter | ❌ | ❌ | ✅ |
| Branch isolation (`/tree`) | ❌ | ❌ | ✅ |
| Synced Pi-native skills (editable) | ❌ | ❌ | ✅ |
| Recall as untrusted context | partial | ❌ | ✅ |
| `peerDependencies` version guard | — | — | ✅ (`>=0.84.1 <0.85`) |

## Security model

- **Untrusted boundary** — recalled memory is wrapped in an explicit trust
  boundary; it is reference data, never instructions.
- **Recursive redaction** — tool arguments and results are redacted
  structurally (sensitive keys blanked wholesale), not just regex-scrubbed.
- **Config sandbox** — a project-level config can only tune `recall`; it can
  never change the endpoint, identity, or keys.
- **TLS enforced** — certificate verification cannot be disabled; remote
  endpoints require HTTPS.
- **Fail-closed on format** — if Pi ever changes its message shape, capture
  degrades to nothing rather than silently mis-parsing.

## Architecture

```text
Pi session ── adapter (this repo, native extension) ── official SDK ── MemoryCore
   │
   ├─ before_agent_start → recall L0-L3 + relevant skills → injected as UNTRUSTED context
   ├─ tool_call/result   → paired by id, redacted, byte-bounded
   ├─ agent_settled      → L0 via durable outbox (at-least-once) · skills at-most-once
   └─ commands / tools   → setup · status · sync-skills · 4 read-only search tools
```

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

In Pi: run `/tdai-memory-setup`, finish the wizard, and start using it. To see
skill learning, add `"skills": { "enabled": true }` to
`~/.pi/agent/tdai-memory.json`, complete one tool-heavy task, then run
`/tdai-memory-sync-skills`.

See [USAGE.md](./USAGE.md) for the full usage guide (configuration, commands,
end-to-end checks, uninstall).

## Requirements

- Node.js `>= 22.19.0`
- Pi `0.84.1` (verified; peer range `>=0.84.1 <0.85`)
- A running MemoryCore with an existing Team, Agent, User, and User Key

## Development & verification

```powershell
cd adapters\pi
npm run check            # typecheck + 119 unit tests
npm run verify:pi-load   # offline Pi load check
npm run e2e:skill        # full skill-loop E2E (real MemoryCore + real Pi)
```

Four real end-to-end suites exercise the setup wizard, lifecycle reliability,
L0–L3 recall injection, and the complete skill loop — all against a real
MemoryCore and a real Pi, so the guarantees above are verified, not claimed.

## License

MIT. This repository is an independent adapter; it is not affiliated with or
endorsed by the TencentDB Agent Memory project. The underlying service is
licensed by its own terms.

- TencentDB Agent Memory: <https://github.com/TencentCloud/TencentDB-Agent-Memory>
- Changelog: <https://github.com/TencentCloud/TencentDB-Agent-Memory/blob/feat/server_team/CHANGELOG.md>
