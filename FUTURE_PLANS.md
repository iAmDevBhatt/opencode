# FUTURE_PLANS.md — Kip Memory System on OpenCode
## Implementation Roadmap for Humans and AI Agents

> **Context:** This document describes how to build the "Kip" layered memory system — a persistent, self-improving personal AI agent — on top of the OpenCode platform. It is written for any developer or AI agent who picks this up with zero prior context.
>
> **Overall verdict:** Fully viable. OpenCode covers ~65% natively, ~25% via custom plugins/MCP wrappers, ~10% requires new builds. All gaps are fillable. Estimated total effort: 3–4 weeks for one developer working part-time.

---

## 0. Environment & Path Conventions

> **Added 2026-08-20 (host correction).** Earlier revisions of this document assumed a Debian Beelink SER5 mini-PC in Aileu, Timor-Leste, with all files under `/home/clawd/`, systemd timers, and bash scripts. **None of that was ever real for this build.** That premise was inherited from the `Patrick` deployment this plan was originally adapted from (see the older `FUTURE.md` in this directory, which still carries it). Every path, scheduler, and shell reference below has been rewritten for the actual environment.

**The actual environment:**

| Thing | Value |
|---|---|
| Host | `djdesktop` — a Windows desktop |
| Timezone | Asia/Kolkata (UTC+5:30) |
| Project root (`<KIP_ROOT>`) | `E:\AgentMemoryProject\opencode` |
| What that directory *is* | A working clone of the **OpenCode monorepo itself** (`packages/`, `sdks/`, `specs/`, `nix/`, …) |
| Shell | PowerShell 7+ (`pwsh`) |
| Scheduler | Windows Task Scheduler (no systemd, no cron) |
| Runtime | Bun (plugin + bridge), Python 3 (`python` on PATH, or `py -3`) |
| OpenCode data dir | `%USERPROFILE%\.local\share\opencode` (confirmed by this repo's own `.opencode/opencode.jsonc` → `references.opencode-local`) |

**Path conventions used throughout this document:**

- `<KIP_ROOT>` = `E:\AgentMemoryProject\opencode`. Windows paths are written with backslashes in prose, shell, and config; **forward slashes inside TypeScript/JSON string literals**, to avoid escape-sequence bugs.
- Kip's runtime files (`memory/`, `MEMORY.md`, `SESSION-STATE.md`, …) live **inside this repo**, per the decision to keep everything in one directory. `memory/` already exists here with `refs/`, `transcripts/`, and `wiki/` scaffolded.
- Kip's PowerShell scripts live in `<KIP_ROOT>\kip-scripts\` — deliberately *not* the repo's existing `script/` directory, which belongs to OpenCode upstream.
- Config lives in the **project-level** `<KIP_ROOT>\.opencode\opencode.jsonc`, not the global `%USERPROFILE%\.config\opencode\opencode.jsonc`. It is version-controlled next to this plan and avoids any ambiguity about where OpenCode looks on Windows.

### ⚠️ Consequences of living inside the OpenCode clone

Keeping Kip's files in this repo is convenient but has three sharp edges. Handle them before Stage 1, not after:

1. **`AGENTS.md` at the repo root is OpenCode's own contributor guide**, tracked upstream. Kip's persona `AGENTS.md` must **not** overwrite it. Kip's copy stays at `<KIP_ROOT>\kip-persona\AGENTS.md` and is installed to `.opencode\skills\kip-agents\SKILL.md`.
2. **Git noise.** `memory/`, `MEMORY.md`, `SESSION-STATE.md`, `HEARTBEAT.md`, `kip-plugin/`, `whatsapp-bridge/`, and `KIP_FAILURE_AUDIT_LOG.md` are personal runtime state, not upstream code. Add them to `.git/info/exclude` (local-only, never committed) rather than `.gitignore` (which *is* tracked upstream and would show up in any PR you open per §14).
3. **Rebasing upstream** (`git pull` from `anomalyco/opencode`) will churn this tree constantly. If that becomes painful, revisit the "sibling directory" layout — this decision is reversible, and only §0 and §7 encode it.

### Missing companion files

This document cites several companion docs. Status as of 2026-08-20:

| Cited as | Actual status |
|---|---|
| `AI_UNDERSTANDING_OPENCODE.md` | **Renamed** — the real file is `APPUNDERSTANDING.md` at the repo root ("OpenCode — Application Understanding Guide"). All citations below have been corrected. Its section numbering matches (§2 Architecture, §3 How to Make Changes, §9 Missing Features). |
| `ai-memory-overview.md` | **Does not exist yet.** Cited by §12 and §13. Write it (or drop the citations) before starting Phase J1. |
| `OpenViking-overview.md` | **Does not exist yet.** Cited by §13. Same. |
| `kip-persona/` | **Does not exist yet.** Step 1.3 assumes starter drafts here. Create the directory and its five files before running Step 1.3's copy commands. |

---

## Table of Contents

0. [Environment & Path Conventions](#0-environment--path-conventions)
1. [What We Are Building](#1-what-we-are-building)
2. [Kip Memory Architecture — Quick Reference](#2-kip-memory-architecture--quick-reference)
3. [Feasibility Analysis by Layer](#3-feasibility-analysis-by-layer)
4. [Master Summary Table](#4-master-summary-table)
5. [Implementation Stages](#5-implementation-stages)
   - [Stage 1 — Zero-Friction Wins (Days 1–3)](#stage-1--zero-friction-wins-days-13)
   - [Stage 2 — Memory Core (Days 4–10)](#stage-2--memory-core-days-410)
   - [Stage 3 — Deterministic Enforcement (Days 11–17)](#stage-3--deterministic-enforcement-days-1117)
   - [Stage 4 — WhatsApp Integration (Days 18–25)](#stage-4--whatsapp-integration-days-1825)
   - [Stage 5 — Polish & Hardening (Days 26–30)](#stage-5--polish--hardening-days-2630)
6. [Architecture Diagram](#6-architecture-diagram)
7. [File & Directory Layout After Implementation](#7-file--directory-layout-after-implementation)
8. [Key Code Patterns](#8-key-code-patterns)
9. [The One Real Limitation and How to Solve It](#9-the-one-real-limitation-and-how-to-solve-it)
10. [Acceptance Criteria](#10-acceptance-criteria)
11. [Open Questions](#11-open-questions)
12. [Adopt ai-memory + Build the Kip Layers (current plan)](#12-adopt-ai-memory--build-the-kip-layers-current-plan)
13. [Future Path — Forking ai-memory for Full Customization](#13-future-path--forking-ai-memory-for-full-customization)
14. [Parallel Track — Contributing to OpenCode](#14-parallel-track--contributing-to-opencode)
15. [Verifying Progress with the OpenCode CLI (and, later, a Local LLM)](#15-verifying-progress-with-the-opencode-cli-and-later-a-local-llm)

---

## 1. What We Are Building

A deployment of OpenCode as the runtime for **Kip** — a persistent, layered-memory AI agent running on `djdesktop` (Windows, Asia/Kolkata) out of `E:\AgentMemoryProject\opencode`.

Kip serves one human across three domains:

1. **Cross-project coding** — makes changes in your repos through OpenCode, remembering per-project decisions without cross-contaminating them.
2. **Homelab operations** — controls your infrastructure, and writes down what it learns about it.
3. **Standing personal context** — profiles how you work and carries that across every project and every session.

> **2026-08-20 correction:** earlier revisions described Kip as a WhatsApp bot serving four domains (TOC research, creative writing, business ops, health tracking) for a user in Timor-Leste. That was inherited from the `Patrick` deployment this plan was adapted from and is **not** what is being built. WhatsApp remains a planned *input channel* (§5 Stage 4 / §12 Phase J5), not the defining feature. Where §5's acceptance criteria still reference that old scenario, they have been rewritten below.

**Kip's design philosophy:**
> Memory is not about hoarding everything in the AI. It is about sparing humans from having to repeat themselves.

The layered system (L0–L10) provides progressive disclosure: the agent keeps high-level structure in context and drills down to raw evidence on demand. Every layer is human-readable Markdown or JSONL — no opaque vector piles, full traceability.

**What this implementation establishes:**
- OpenCode is the execution engine; Kip is an agent definition plus a plugin plus memory files on top of it
- All Kip state is human-readable Markdown/JSONL under `<KIP_ROOT>`, with any index (SQLite/ChromaDB) treated as derived and rebuildable
- OpenCode's plugin system, MCP protocol, and skill system wire the layers together
- WhatsApp becomes an additional input channel via a thin Bun wrapper calling the OpenCode HTTP API

> **Note on "existing infrastructure":** earlier revisions said "all existing scripts, Markdown files, and ChromaDB/NetworkX infra are preserved." There is no prior Kip deployment on this machine — `memory/` here contains empty `refs/`, `transcripts/`, and `wiki/` scaffolding and nothing else. Every script, persona file, and index referenced below is **to be created**, not migrated. Treat any step that says "your existing X" as "the X you will write in this step."

---

## 2. Kip Memory Architecture — Quick Reference

| Layer | Name | Format | Location | Size |
|---|---|---|---|---|
| L0 | Session transcripts | JSONL | `%USERPROFILE%\.local\share\opencode\transcripts\\` | Unbounded (firehose) |
| L1 | Daily memory logs | Markdown + `[ATOM]` | `memory\YYYY-MM-DD.md` | 0 files today; grows daily |
| L2 | Curated long-term memory | Markdown | `MEMORY.md` | <120 lines enforced |
| L3 | Identity & persona | Markdown | `kip-persona\{SOUL,USER,IDENTITY,AGENTS}.md` | To be written |
| L4 | Deterministic enforcement | PowerShell scripts | `E:\AgentMemoryProject\opencode\kip-scripts\*.ps1` | 3 scripts |
| L5 | Vector memory | ChromaDB + NetworkX | `E:\AgentMemoryProject\opencode\memory\index\` | Empty until indexed |
| L6 | Memory wiki | Markdown synthesis | `memory\wiki\` (exists, empty) | Light |
| L7 | Obsidian vault | Markdown | `<YOUR_OBSIDIAN_VAULT>` — **path not yet decided** | Gigabytes |
| L8 | Session state + heartbeat | Markdown | `SESSION-STATE.md`, `HEARTBEAT.md` | Small |
| L9 | Memory promotion pipeline | PowerShell + Task Scheduler | `kip-scripts\memory-promote.ps1` | 1 script |
| L10 | Procedural memory (skills) | Markdown | `.opencode\skills\`, `kip-persona\TOOLS.md` | Per-skill |

**[ATOM] format** (L1 structured extraction):
```
[ATOM] type=decision|event|lesson|preference|fact | entity=<subject> | detail=<what> | ref=<source>
```

---

## 3. Feasibility Analysis by Layer

### L0 — Session Transcripts

**OpenCode native state:** Partial. All session messages are stored in SQLite via Drizzle ORM (`packages/core/src/database/`). The `SessionStore` exposes `context()` and `runnerContext()`. Messages are sequenced and typed. The `opencode export` command exists but is not a live-append JSONL firehose.

**Gap:** Rows in SQLite, not raw JSONL files you can tail or grep in real time.

**Implementation:** Write a plugin hook on session message events that appends each message to `%USERPROFILE%\.local\share\opencode\transcripts\\YYYY-MM-DD.jsonl`. Approximately 30 lines using the OpenCode plugin hook system.

---

### L1 — Daily Memory Logs

**OpenCode native state:** Not native. OpenCode has no "write a session summary after every session" primitive. However, the plugin system has hooks that fire on session events.

OpenCode's internal compaction system already does something structurally identical — `packages/core/src/session/compaction.ts` generates a `SUMMARY_TEMPLATE` with sections: Objective / Important Details / Work State (Completed/Active/Blocked) / Next Move / Relevant Files. This is stored as a session message internally.

**Implementation:** Plugin that:
1. Listens for session-end event (or idle + heartbeat detection)
2. Intercepts or re-runs the compaction summary LLM call
3. Reformats output into Kip's L1 format (including `[ATOM]` extraction)
4. Appends to `memory/$(date +%Y-%m-%d).md`

The compaction template in `compaction.ts` is directly adaptable — Kip's L1 format is a superset of the existing `SUMMARY_TEMPLATE`.

---

### L2 — MEMORY.md

**OpenCode native state:** Yes — this is exactly what OpenCode's own auto-memory system uses. The pattern: a `MEMORY.md` index + individual `memory/*.md` files, enforced at <200 lines. The agent reads it at session start.

**Gap:** Kip enforces <120 lines (stricter than OpenCode's <200). Kip's promotion logic scores atoms before promoting.

**Implementation:** Adapt the existing `MEMORY.md` format to Kip's 120-line limit. The L9 promotion pipeline (a scheduled `opencode run` call) handles pruning.

---

### L3 — Identity & Persona Files

**OpenCode native state:** Fully supported via Skills + Config.

Direct equivalents in OpenCode:
- `AGENTS.md` at repo root → OpenCode reads as agent constitution
- `.opencode/opencode.jsonc` with `agents.<name>.prompt` → per-agent system prompt
- Skills (`SKILL.md` files) → inject structured knowledge into context
- The built-in `customize-opencode` skill demonstrates the exact pattern

**Implementation:** Map each identity file to a skill:

| Kip file | OpenCode location |
|---|---|
| `SOUL.md` | `.opencode/skills/kip-soul/SKILL.md` |
| `USER.md` | `.opencode/skills/kip-user/SKILL.md` |
| `IDENTITY.md` | `.opencode/skills/kip-identity/SKILL.md` |
| `AGENTS.md` | Agent prompt in `opencode.jsonc` + `.opencode/agent/kip.md` |

Effort: Near-zero. Copy existing files into the skill directory structure.

---

### L4 — Deterministic Layer (Logician / Shield / Guardian)

**OpenCode native state:** Not native. OpenCode has no binary-verification, path-validation, or auto-healing primitives.

**Implementation per component:**

**Logician** (binary verification before claiming task complete):
- Implement as a custom tool in an OpenCode plugin
- Calls `logician-verify.ps1` via `Bun.spawnSync(["powershell", "-NoProfile", "-File", ...])`
- Agent can call `logician_verify({ check_type: "file_exists", target: "/path" })` and get `{ verified: true/false }`

**Shield** (path validation — blocks writes outside safe zones):
- Implement as a plugin hook on `tool.before`
- Intercepts every `write` and `bash` tool call
- Validates target path against allowlist: `["E:/AgentMemoryProject/opencode/memory", "E:/AgentMemoryProject/opencode/vault", "E:/AgentMemoryProject/opencode/.opencode", os.tmpdir()]`
- **Windows caveat:** path comparison must be case-insensitive and normalize `/` vs `\` before matching, or the allowlist is trivially bypassable (see Step 3.2)
- Returns a blocking error if path is outside safe zones

**Guardian** (auto-healing heartbeat):
- Stays as a Windows Task Scheduler task on djdesktop
- Calls `opencode run --agent kip "run guardian health check"` at the 30-minute interval
- Guardian script checks: gateway connectivity, disk space, vector sync status, dispatch generation

---

### L5 — Vector Memory (ChromaDB + NetworkX)

**OpenCode native state:** Not present. Zero vector/embedding infrastructure in the entire OpenCode codebase. No ChromaDB, no sqlite-vec, no embedding calls.

**Implementation:** Wrap existing Python scripts as an MCP server. The MCP stdio protocol is ~100 lines of Python boilerplate. Add to `opencode.jsonc`:

```jsonc
{
  "mcp": {
    "servers": {
      "kip-memory": {
        "type": "local",
        "command": ["python", "E:/AgentMemoryProject/opencode/kip-scripts/memory-mcp-server.py"]
      }
    }
  }
}
```

The `memory_search` natural language queries become MCP tool calls. OpenCode exposes them to Kip natively. No changes to existing Python scripts beyond the MCP wrapper.

---

### L6 — Memory Wiki

**OpenCode native state:** No native equivalent. No `memory-wiki` plugin exists in the repo.

**Implementation:** Custom plugin tool `wiki_synthesize`:
1. Takes a topic string
2. Reads relevant daily notes (via `read` tool)
3. Calls LLM to write a synthesis page with provenance links
4. Writes to `memory/wiki/<topic-slug>.md`

Optional: add contradiction detection by comparing new synthesis against existing wiki pages.

---

### L7 — Obsidian Vault

**OpenCode native state:** Fully supported via MCP. A mature MCP server exists for Obsidian.

**Implementation:** Add one entry to `opencode.jsonc`. Preserve the existing discipline: agent reads for context, writes only on explicit instruction.

```jsonc
{
  "mcp": {
    "servers": {
      "obsidian": {
        "type": "local",
        "command": ["npx", "-y", "mcp-obsidian", "<YOUR_OBSIDIAN_VAULT>"]
      }
    }
  }
}
```

---

### L8 — Session State & Heartbeat

**OpenCode native state:** Partial.

- OpenCode emits a `server.heartbeat` SSE event every 10 seconds (in `packages/opencode/src/server/routes/instance/httpapi/handlers/global.ts`) — but this is a keep-alive ping for connected clients, not a task scheduler.
- No native cron scheduler exists in OpenCode.

**Implementation:**
- `SESSION-STATE.md` → written by a plugin hook at session end via the `write` tool
- 3x daily heartbeat (8am/2pm/6pm local, Asia/Kolkata) → Task Scheduler tasks on djdesktop invoking `opencode run --agent kip "run morning/afternoon/evening ritual"`
- `HEARTBEAT.md` updated by the agent during the ritual run

---

### L9 — Memory Promotion Pipeline

**OpenCode native state:** Not native, but trivially implementable as a scheduled `opencode run` call (Windows Task Scheduler here, not cron).

**Implementation:**

```powershell
# Registered as the scheduled task Kip\MemoryPromote — see §5 Step 1.6 for the full script
$today = Get-Date -Format 'yyyy-MM-dd'
opencode run --agent kip @"
Run memory promotion ritual: read today's L1 log at memory\$today.md,
score each [ATOM] entry (decision=3, lesson=2, preference=2, event=1, fact=1),
promote items scoring >=2 to MEMORY.md, prune MEMORY.md if >180 lines keeping highest scores.
"@
```

The agent already has all required tools (read, edit, write). No plugin needed.

---

### L10 — Procedural Memory (Skills)

**OpenCode native state:** Directly supported. OpenCode's skill system is the native equivalent of L10.

**Implementation:**
- Copy `TOOLS.md` content → `.opencode/skills/kip-tools/SKILL.md`
- Each existing skill → `.opencode/skills/<name>/SKILL.md`
- Session-end checklist (skill detection) → plugin hook that asks at session end: "Was a procedure repeated 2+ times? Was a non-obvious fix used? If yes, suggest skill creation."

---

## 4. Master Summary Table

| Kip Layer | What it is | OpenCode Native? | Implementation Path | Effort | Priority |
|---|---|---|---|---|---|
| L0 Session transcripts | Raw JSONL dialogue | Partial (SQLite) | Plugin hook → append to JSONL | Low | P2 |
| L1 Daily memory logs | Structured session notes + `[ATOM]` | No | Plugin hook + compaction intercept + file write | Medium | P1 |
| L2 MEMORY.md | Curated long-term memory <120 lines | Yes | Adapt format/line limit | Low | P1 |
| L3 SOUL/USER/IDENTITY | Identity & persona files | Yes (skills + agent prompt) | Copy files into `.opencode/skills/` dirs | Near-zero | P1 |
| L4 Logician/Shield/Guardian | Deterministic enforcement | No | Custom tools + plugin hook + Task Scheduler | Medium | P2 |
| L5 ChromaDB + NetworkX | Vector + graph memory | No | Wrap existing Python as MCP server | Medium | P2 |
| L6 Memory Wiki | Synthesis pages with provenance | No | Custom plugin tool `wiki_synthesize` | Medium | P3 |
| L7 Obsidian vault | Human-curated knowledge base | Yes (via MCP) | Add MCP server config (1 line) | Low | P1 |
| L8 SESSION-STATE + heartbeat | Session state + 3x daily schedule | Partial | Plugin hook + Task Scheduler tasks | Low | P1 |
| L9 Promotion pipeline | Scored daily→MEMORY.md promotion | No (trivial) | One scheduled task + agent prompt | Near-zero | P1 |
| L10 Skills + TOOLS.md | Procedural memory | Yes (native skills) | Copy files into skill dirs | Near-zero | P1 |

**P1 = Week 1 | P2 = Week 2 | P3 = Week 3**

---

## 5. Implementation Stages

---

### Stage 1 — Zero-Friction Wins (Days 1–3)

**Goal:** Get Kip's identity and procedural knowledge into OpenCode with no new code.
 Kip should be able to respond "as Kip" with full persona and context from Day 1.

---

#### Step 1.1 — Install OpenCode on djdesktop

The `curl | bash` installer from earlier revisions does not apply on Windows. Use npm (or run from the local clone).

```powershell
# On djdesktop, in PowerShell
npm install -g opencode@latest
opencode --version
```

Because `<KIP_ROOT>` **is** a clone of the OpenCode monorepo, you have a second option that is often better while you're also contributing upstream (§14) — run the local build instead of the published package:

```powershell
Set-Location E:\AgentMemoryProject\opencode
bun install
bun dev --help          # local equivalent of the `opencode` command
bun dev serve           # headless API server
```

Per this repo's `CONTRIBUTING.md`, `bun dev` is the development equivalent of `opencode`. Pick one and be consistent — mixing a globally-installed `opencode` with `bun dev` from this clone means two different versions reading the same config and data directory.

**Decide now (this is Open Question #5 in §11):** if you use the global npm install, pin it. OpenCode auto-update against a plan this specific is a recipe for silent breakage.

---

#### Step 1.2 — Create Kip Agent Definition

> **2026-08-20 correction**: renamed from the original "Kip" example 
(TOC research/creative writing/business ops/health tracking — a different scenario's domains)
 to match what's actually being built here: a self-improving coding + homelab + personal-context agent. 
 Skill names below now match `kip-persona/` from Step 1.3.

Create `.opencode/agent/kip.md` in the project directory (or `~/.config/opencode/agent/kip.md` for global):

```markdown
---
name: kip
description: Kip — self-improving personal AI agent for cross-project coding, homelab operations, and standing personal context
mode: primary
model: anthropic/claude-sonnet-5
temperature: 0.3
permission:
  - allow: read
  - allow: write
    description: Only within approved paths
  - allow: bash
    description: Safe shell operations only
  - allow: skill
  - allow: todowrite
---

You are Kip. Read your identity from the kip-soul, kip-user, and kip-identity skills before responding.
Always begin sessions by loading SESSION-STATE.md to understand current project phase.
Follow the memory protocol defined in the kip-agents skill.
```

---

#### Step 1.3 — Migrate Identity Files to Skills

> **2026-08-20 correction**: the original version of this step assumed `SOUL.md`/`USER.md`/`IDENTITY.md`/`AGENTS.md`/`TOOLS.md` already existed under `/home/clawd/` from a prior Kip deployment. They don't — that path was never real for this build.
>
> **Status check (verify before running the commands below):** `kip-persona\` **does not exist in this repo yet.** Create it and draft its five files first. The suggested starting point is OpenViking's `bot/workspace/{SOUL,USER,TOOLS}.md` (its "vikingbot" agent template), plus two newly-drafted files (`IDENTITY.md`, `AGENTS.md`) with no existing analog. Treat them as structured drafts with placeholders — your name, timezone (Asia/Kolkata), the actual repos you work in, the actual homelab services you run, your coding preferences — not finished identity. **Read and personalize every file before copying it anywhere.**
>
> ⚠️ **Do not put Kip's `AGENTS.md` at the repo root.** `<KIP_ROOT>\AGENTS.md` is OpenCode's own contributor guide, tracked upstream. Kip's version lives only at `kip-persona\AGENTS.md` and is installed into `.opencode\skills\kip-agents\SKILL.md`.

```powershell
Set-Location E:\AgentMemoryProject\opencode

foreach ($s in 'kip-soul','kip-user','kip-identity','kip-agents','kip-tools') {
  New-Item -ItemType Directory -Force -Path ".opencode\skills\$s" | Out-Null
}

# Copy your personalized versions (edit kip-persona\*.md first!)
Copy-Item kip-persona\SOUL.md     .opencode\skills\kip-soul\SKILL.md
Copy-Item kip-persona\USER.md     .opencode\skills\kip-user\SKILL.md
Copy-Item kip-persona\IDENTITY.md .opencode\skills\kip-identity\SKILL.md
Copy-Item kip-persona\AGENTS.md   .opencode\skills\kip-agents\SKILL.md
Copy-Item kip-persona\TOOLS.md    .opencode\skills\kip-tools\SKILL.md
```

Note that `.opencode\skills\` here is the **project-scoped** skill directory — per `APPUNDERSTANDING.md` §2.11, OpenCode discovers `SKILL.md` under `.opencode/skills/<name>/`, `.agents/skills/<name>/`, and `skills/<name>/` in the project root. Project-scoped means these skills only load when OpenCode runs *in this repo*. That's fine for Stage 1, and it's precisely the limitation Phase J2 fixes by moving identity into `_global` scope.

This is still the §5-era "skills as identity files" mechanism from the original Stages 1–5 plan. Once you reach **§12 Phase J2**, the canonical home for these files shifts: `SOUL.md`/`USER.md`/`IDENTITY.md` move into `ai-memory`'s `_global` scope via `memory_write_page` (tagged `canonical`+`pinned`) instead of living only as skill files, so they follow you across every project automatically rather than needing to be present in each repo's `.opencode/skills/`. `AGENTS.md` and `TOOLS.md` stay reasonable as skill files either way, since they're behavioral rules rather than personal facts.

Each file already contains the right content — OpenCode discovers `SKILL.md` files automatically. No reformatting needed.

---

#### Step 1.4 — Configure Obsidian MCP Server

Edit the **project-level** `E:\AgentMemoryProject\opencode\.opencode\opencode.jsonc`.

> ⚠️ **This file already exists and is non-empty.** It currently defines an `ollama` provider (with `deepseek-r1:32b`, `qwen3-coder:30b`, `mixtral`, and others against `http://localhost:11434/v1`), a `references` block, and a `tools` block. **Merge into it — do not replace it.** The Ollama provider in particular is already what §15 wants for local verification.
>
> Also note `"mcp": {}` is already present and empty; fill it in rather than adding a second key.

Merged result (the `provider`, `references`, and `tools` blocks it already has are elided here for brevity — leave them in place):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-5",
  "default_agent": "kip",

  // ... existing "provider", "permission", "references", "tools" blocks stay as they are ...

  "mcp": {
    "obsidian": {
      "type": "local",
      "command": ["npx", "-y", "mcp-obsidian", "<YOUR_OBSIDIAN_VAULT>"],
      "timeout": 30000
    }
  }
}
```

Two things to verify against the installed version before trusting this block, since the original was written for a different setup:

1. **The `mcp` shape.** Earlier revisions nested servers under `"mcp": { "servers": { ... } }`. This repo's own config has a flat `"mcp": {}`. Check `packages/opencode/src/config/` or run `opencode mcp list` after editing — if the server doesn't appear, the nesting is wrong.
2. **The Obsidian vault path.** `<YOUR_OBSIDIAN_VAULT>` is a placeholder — the original `/home/Obsidian` was from the Timor-Leste deployment. Decide where your vault actually lives on Windows (e.g. `E:\Obsidian\MainVault`) and substitute it. If you don't use Obsidian, **drop L7 entirely** rather than wiring a vault you'll never write to; it's marked P1 in §4 only because it was near-free in the original plan.

Test: `opencode mcp list` — should show `obsidian` as connected.

---

#### Step 1.5 — Set Up Heartbeat Scheduled Tasks

There is no systemd here. The equivalent is **Windows Task Scheduler**, driven from PowerShell. Create one ritual script plus three tasks.

First, the ritual script — `E:\AgentMemoryProject\opencode\kip-scripts\ritual.ps1`:

```powershell
param(
  [Parameter(Mandatory)]
  [ValidateSet('morning','afternoon','evening')]
  [string]$Ritual
)

$ErrorActionPreference = 'Stop'
Set-Location 'E:\AgentMemoryProject\opencode'

$prompts = @{
  morning   = 'Run morning ritual: re-read the kip-soul skill, review SESSION-STATE.md, check for a pending handoff, verify MEMORY.md is under 120 lines. Log what you found to HEARTBEAT.md.'
  afternoon = 'Run afternoon ritual: review what changed since morning, update SESSION-STATE.md with current phase and next move. Log to HEARTBEAT.md.'
  evening   = 'Run evening ritual: summarize the day from memory/<today>.md, confirm the daily log has [ATOM] entries, write tomorrow''s next move into SESSION-STATE.md. Log to HEARTBEAT.md.'
}

# Date-based session id keeps daily history grouped without one infinite session (see §8 Pattern 3)
$sessionId = "kip-$(Get-Date -Format 'yyyyMMdd')-$Ritual"

opencode run --agent kip --session-id $sessionId $prompts[$Ritual]
```

Then register the three tasks (run this PowerShell **as Administrator** once):

```powershell
$pwsh   = (Get-Command pwsh).Source          # or powershell.exe on Windows PowerShell 5.1
$script = 'E:\AgentMemoryProject\opencode\kip-scripts\ritual.ps1'

$rituals = @{ morning = '08:00'; afternoon = '14:00'; evening = '18:00' }

foreach ($r in $rituals.GetEnumerator()) {
  $action  = New-ScheduledTaskAction -Execute $pwsh `
               -Argument "-NoProfile -ExecutionPolicy Bypass -File `"$script`" -Ritual $($r.Key)"
  $trigger = New-ScheduledTaskTrigger -Daily -At $r.Value
  # StartWhenAvailable is the analogue of systemd's Persistent=true —
  # it runs a missed task once the machine is back up, which matters on a desktop that sleeps.
  $settings = New-ScheduledTaskSettingsSet -StartWhenAvailable -WakeToRun:$false `
                -DontStopIfGoingOnBatteries -AllowStartIfOnBatteries

  Register-ScheduledTask -TaskName "Kip\Ritual-$($r.Key)" `
    -Action $action -Trigger $trigger -Settings $settings `
    -Description "Kip $($r.Key) ritual" -Force
}
```

> **Desktop-vs-server caveat, new to this environment.** The original design assumed an always-on mini-PC. A desktop sleeps, hibernates, and gets shut down. `-StartWhenAvailable` catches up *one* missed run, not every missed run, and a machine that's off at 08:00 and 14:00 will only fire the 14:00 catch-up. If ritual reliability turns out to matter, either enable `-WakeToRun` or accept that rituals are best-effort and make each one idempotent. Don't let Kip's memory correctness depend on a ritual having fired.

Verify:

```powershell
Get-ScheduledTask -TaskPath '\Kip\' | Format-Table TaskName, State
Start-ScheduledTask -TaskName 'Kip\Ritual-morning'   # test-fire without waiting
Get-ScheduledTaskInfo -TaskName 'Kip\Ritual-morning' # LastTaskResult should be 0
```

---

#### Step 1.6 — Set Up the Memory Promotion Task

The cron entry becomes a fourth scheduled task. Script — `E:\AgentMemoryProject\opencode\kip-scripts\memory-promote.ps1`:

```powershell
$ErrorActionPreference = 'Stop'
Set-Location 'E:\AgentMemoryProject\opencode'

$today = Get-Date -Format 'yyyy-MM-dd'
$log   = "memory\$today.md"

if (-not (Test-Path $log)) {
  Write-Host "No L1 log for $today — nothing to promote."
  exit 0
}

$prompt = @"
Memory promotion ritual: Read today's L1 log at $log.
Score each [ATOM] entry: decision=3, lesson=2, preference=2, event=1, fact=1.
Promote all items scoring 2 or above to MEMORY.md.
If MEMORY.md exceeds 180 lines, prune to 120 lines keeping highest scores.
Write updated MEMORY.md.
"@

opencode run --agent kip --session-id "kip-$(Get-Date -Format 'yyyyMMdd')-promote" $prompt
```

Register it:

```powershell
$action  = New-ScheduledTaskAction -Execute (Get-Command pwsh).Source `
             -Argument '-NoProfile -ExecutionPolicy Bypass -File "E:\AgentMemoryProject\opencode\kip-scripts\memory-promote.ps1"'
$trigger = New-ScheduledTaskTrigger -Daily -At '14:00'
Register-ScheduledTask -TaskName 'Kip\MemoryPromote' -Action $action -Trigger $trigger `
  -Settings (New-ScheduledTaskSettingsSet -StartWhenAvailable) `
  -Description 'Kip L9 memory promotion' -Force
```

> **§13 Phase 4 supersedes this step's approach.** Asking an LLM to self-score atoms in a free-text chat turn is the least reliable part of the whole L9 design. Once `llmCompile.ts` exists, this task should call a CLI entrypoint with a structured-output contract instead of a prose prompt. Until then, this works — just don't trust it silently; read `MEMORY.md`'s diff for the first week.

---

#### Stage 1 Acceptance Criteria

- [ ] `opencode run --agent kip "Who are you?"` returns a Kip-persona response referencing soul/identity files
- [ ] `opencode mcp list` shows `obsidian` connected
- [ ] `opencode skill list` shows all five kip-* skills
- [ ] All four scheduled tasks are registered: `Get-ScheduledTask -TaskPath '\Kip\'` lists `Ritual-morning`, `Ritual-afternoon`, `Ritual-evening`, `MemoryPromote`
- [ ] A test-fired ritual exits clean: `Start-ScheduledTask -TaskName 'Kip\Ritual-morning'` then `Get-ScheduledTaskInfo` shows `LastTaskResult = 0`
- [ ] `HEARTBEAT.md` exists at `<KIP_ROOT>` and has an entry from that test fire

---

### Stage 2 — Memory Core (Days 4–10)

**Goal:** Implement L0 (JSONL transcripts), L1 (daily memory logs with `[ATOM]` extraction), and L5 (ChromaDB/NetworkX as MCP server).

---

#### Step 2.1 — Create the OpenCode Plugin Package

```powershell
New-Item -ItemType Directory -Force -Path 'E:\AgentMemoryProject\opencode\kip-plugin'
Set-Location 'E:\AgentMemoryProject\opencode\kip-plugin'
bun init -y
bun add @opencode-ai/plugin
```

> ⚠️ **Bun workspace collision.** `<KIP_ROOT>` is itself a Bun monorepo with a root `package.json` and `bun.lock`. Creating `kip-plugin/` *inside* it means Bun may resolve it as a workspace member and hoist its dependencies into the root `node_modules`, which will dirty the OpenCode clone and can break `bun install` upstream. Check the root `package.json`'s `workspaces` globs before running the above. If `kip-plugin` would be captured, either add it to the root `.git/info/exclude` **and** an explicit workspace exclusion, or keep the plugin in a sibling directory (`E:\AgentMemoryProject\kip-plugin`) and reference it by absolute path from `opencode.jsonc`. The sibling option is the safer default — the plugin doesn't need to live in the repo for OpenCode to load it.

Create `src/index.ts`:

```typescript
import { tool } from "@opencode-ai/plugin"

// All Kip custom tools and hooks will be registered here
export default {
  tools: [],
  hooks: {}
}
```

---

#### Step 2.2 — L0: Session Transcript JSONL Writer

Add to `src/hooks/transcript.ts`:

```typescript
import { appendFile } from "fs/promises"
import { join } from "path"

// Forward slashes: Windows accepts them everywhere in Node/Bun, and they avoid
// the "\t is a tab, not a folder" escape trap that bites every Windows path literal.
const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"
const TRANSCRIPT_DIR = join(KIP_ROOT, "memory", "transcripts")

export const transcriptHook = {
  // Fires after every message is written
  "session.message.after": async (message: any) => {
    const date = new Date().toISOString().slice(0, 10)  // YYYY-MM-DD
    const filePath = join(TRANSCRIPT_DIR, `${date}.jsonl`)
    await appendFile(filePath, JSON.stringify(message) + "\n", "utf8")
  }
}
```

Register in plugin index. This gives Kip a live-append JSONL firehose identical to L0.

---

#### Step 2.3 — L1: Daily Memory Log Writer

Add to `src/hooks/daily-log.ts`:

```typescript
import { appendFile, readFile } from "fs/promises"
import { join } from "path"

const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"
const MEMORY_DIR = join(KIP_ROOT, "memory")

// The SUMMARY_TEMPLATE adapted for Kip's L1 format
const L1_TEMPLATE = `
## Session Summary — {DATE} {TIME}

### Objective
{objective}

### Decisions Made
{decisions}

### Work State
**Completed:** {completed}
**Active:** {active}
**Blocked:** {blocked}

### Files Modified
{files}

### Next Move
{next_move}

### Health / Context Notes
{health_notes}

### HANDOVER (for model switches)
{handover}

### L1 Atoms
{atoms}
`

export const dailyLogHook = {
  "session.compaction.ended": async (compactionSummary: any, ctx: any) => {
    const date = new Date().toISOString().slice(0, 10)
    const time = new Date().toTimeString().slice(0, 5)
    const filePath = join(MEMORY_DIR, `${date}.md`)

    // Transform compaction summary into L1 format
    // The compaction summary already has the right sections
    // We add [ATOM] extraction on top
    const atoms = extractAtoms(compactionSummary)

    const entry = L1_TEMPLATE
      .replace("{DATE}", date)
      .replace("{TIME}", time)
      .replace("{atoms}", atoms.map(formatAtom).join("\n"))
      // ... map other fields from compactionSummary

    await appendFile(filePath, entry, "utf8")
  }
}

function extractAtoms(summary: any): Atom[] {
  // Parse decisions, lessons, preferences from the summary text
  // Return structured [ATOM] entries
  const atoms: Atom[] = []
  // decision detection: look for "decided", "will use", "chosen"
  // lesson detection: look for "learned", "found that", "realized"
  // preference detection: look for "prefer", "better to", "going forward"
  return atoms
}

function formatAtom(atom: Atom): string {
  return `[ATOM] type=${atom.type} | entity=${atom.entity} | detail=${atom.detail} | ref=${atom.ref}`
}
```

---

#### Step 2.4 — L5: MCP Wrapper for ChromaDB + NetworkX

Create `E:\AgentMemoryProject\opencode\kip-scripts\memory-mcp-server.py`:

```python
#!/usr/bin/env python3
"""MCP server wrapping Kip's ChromaDB + NetworkX memory."""

import asyncio
import json
import sys
from typing import Any

# MCP stdio protocol boilerplate
async def handle_request(request: dict) -> dict:
    method = request.get("method")
    params = request.get("params", {})

    if method == "initialize":
        return {
            "protocolVersion": "2024-11-05",
            "capabilities": {"tools": {}},
            "serverInfo": {"name": "kip-memory", "version": "1.0.0"}
        }

    if method == "tools/list":
        return {
            "tools": [
                {
                    "name": "memory_search",
                    "description": "Search Kip's long-term memory using natural language",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "query": {"type": "string", "description": "Natural language search query"},
                            "limit": {"type": "integer", "default": 5}
                        },
                        "required": ["query"]
                    }
                },
                {
                    "name": "memory_graph_lookup",
                    "description": "Look up entity relationships in the knowledge graph",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "entity": {"type": "string"},
                            "depth": {"type": "integer", "default": 2}
                        },
                        "required": ["entity"]
                    }
                }
            ]
        }

    if method == "tools/call":
        tool_name = params.get("name")
        tool_args = params.get("arguments", {})

        if tool_name == "memory_search":
            results = await search_chromadb(tool_args["query"], tool_args.get("limit", 5))
            return {"content": [{"type": "text", "text": json.dumps(results)}]}

        if tool_name == "memory_graph_lookup":
            results = await lookup_graph(tool_args["entity"], tool_args.get("depth", 2))
            return {"content": [{"type": "text", "text": json.dumps(results)}]}

    return {"error": {"code": -32601, "message": f"Method not found: {method}"}}


async def search_chromadb(query: str, limit: int) -> list:
    """Call existing search.py / auto_retrieve.py."""
    import subprocess
    result = subprocess.run(
        ["python", "E:\AgentMemoryProject\opencode\kip-scripts\search.py", query, str(limit)],
        capture_output=True, text=True
    )
    return json.loads(result.stdout) if result.returncode == 0 else []


async def lookup_graph(entity: str, depth: int) -> dict:
    """Query NetworkX graph.json."""
    import subprocess
    result = subprocess.run(
        ["python", "E:\AgentMemoryProject\opencode\kip-scripts\graph_lookup.py", entity, str(depth)],
        capture_output=True, text=True
    )
    return json.loads(result.stdout) if result.returncode == 0 else {}


async def main():
    """MCP stdio loop."""
    while True:
        line = await asyncio.get_event_loop().run_in_executor(None, sys.stdin.readline)
        if not line:
            break
        try:
            request = json.loads(line.strip())
            response = await handle_request(request)
            response["id"] = request.get("id")
            response["jsonrpc"] = "2.0"
            print(json.dumps(response), flush=True)
        except Exception as e:
            error_response = {
                "jsonrpc": "2.0",
                "id": request.get("id") if "request" in dir() else None,
                "error": {"code": -32603, "message": str(e)}
            }
            print(json.dumps(error_response), flush=True)


if __name__ == "__main__":
    asyncio.run(main())
```

Add to `opencode.jsonc`:
```jsonc
"kip-memory": {
  "type": "local",
  "command": ["python", "E:/AgentMemoryProject/opencode/kip-scripts/memory-mcp-server.py"],
  "timeout": 15000
}
```

---

#### Step 2.5 — Install and Register the Plugin

```powershell
Set-Location 'E:\AgentMemoryProject\opencode\kip-plugin'
bun run build   # or: bun build src/index.ts --outdir dist

# Install into OpenCode
opencode plug 'E:\AgentMemoryProject\opencode\kip-plugin'
```

Per `APPUNDERSTANDING.md` §2.10, `opencode plug` expects an npm package name in normal use; verify that a local absolute path is accepted by the installed version. If it isn't, register the plugin through the `"plugin"` key in `opencode.jsonc` (as Step 5.4 does) instead.

---

#### Stage 2 Acceptance Criteria

- [ ] After a session, `Get-ChildItem memory\transcripts\*.jsonl` shows today's transcript file with content
- [ ] After a session, `Get-ChildItem memory\*.md` shows today's daily log with `[ATOM]` entries
- [ ] `opencode run --agent kip "What do you remember about last Tuesday?"` triggers a ChromaDB search
- [ ] `opencode run --agent kip "What is connected to the kip-plugin work?"` triggers a graph lookup
- [ ] `opencode mcp list` shows `kip-memory` as connected

---

### Stage 3 — Deterministic Enforcement (Days 11–17)

**Goal:** Implement L4 (Logician, Shield, Guardian) as OpenCode plugin tools and hooks.

---

#### Step 3.1 — Logician Tool

Add to `src/tools/logician.ts` in the plugin:

```typescript
import { tool } from "@opencode-ai/plugin"

export const LogicianTool = tool({
  id: "logician_verify",
  description: "Binary verification before claiming a task complete. Returns verified=true only if the condition is provably met. Use before saying 'done'.",
  parameters: {
    type: "object",
    properties: {
      check_type: {
        type: "string",
        enum: ["file_exists", "file_contains", "cmd_succeeds", "url_reachable", "dir_exists"],
        description: "Type of check to perform"
      },
      target: {
        type: "string",
        description: "Path, command, or URL to verify"
      },
      expected: {
        type: "string",
        description: "For file_contains: the string that must be present"
      }
    },
    required: ["check_type", "target"]
  },
  execute: async ({ check_type, target, expected }) => {
    const result = Bun.spawnSync([
      "pwsh", "-NoProfile", "-ExecutionPolicy", "Bypass",
      "-File", `${process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"}/kip-scripts/logician-verify.ps1`,
      "-CheckType", check_type,
      "-Target", target,
      ...(expected ? ["-Expected", expected] : []),
    ])
    const verified = result.exitCode === 0
    const output = new TextDecoder().decode(result.stdout).trim()
    return {
      verified,
      check_type,
      target,
      detail: output || (verified ? "Check passed" : "Check failed"),
    }
  }
})
```

---

#### Step 3.2 — Shield Hook (Path Validation)

Add to `src/hooks/shield.ts`:

> **Rewritten for Windows.** The original `path.startsWith(zone)` check was already weak on Linux; on Windows it is **broken**, for three separate reasons. Do not port the old version:
>
> 1. **Case-insensitivity.** `e:\agentmemoryproject\...` and `E:\AgentMemoryProject\...` are the same file. A case-sensitive `startsWith` blocks nothing.
> 2. **Separator mixing.** `E:/AgentMemoryProject/opencode/memory` and `E:\AgentMemoryProject\opencode\memory` are the same directory and compare unequal.
> 3. **Prefix-matching is not containment.** `startsWith("E:\\...\\opencode")` also accepts `E:\AgentMemoryProject\opencode-evil\`. And neither version resolves `..`, so `E:\...\opencode\memory\..\..\Windows\System32` sails straight through.

Add to `src/hooks/shield.ts`:

```typescript
import { resolve, sep } from "path"
import { tmpdir } from "os"

const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"

// Safe zones — writes outside these are blocked.
// Note the repo root itself is NOT a safe zone: Kip should not be able to
// rewrite OpenCode's own source tree by accident.
const SAFE_ZONES = [
  `${KIP_ROOT}/memory`,
  `${KIP_ROOT}/vault`,
  `${KIP_ROOT}/.opencode`,
  `${KIP_ROOT}/kip-persona`,
  tmpdir(),
].map(z => resolve(z).toLowerCase())

function isPathSafe(candidate: string): boolean {
  // resolve() normalizes separators, collapses "..", and makes the path absolute.
  const target = resolve(candidate).toLowerCase()
  return SAFE_ZONES.some(zone =>
    // Exact match, or genuinely inside the zone — the trailing separator is what
    // stops "opencode" from matching "opencode-evil".
    target === zone || target.startsWith(zone + sep)
  )
}

export const shieldHook = {
  "tool.before": async (toolCall: { id: string; parameters: any }) => {
    // Intercept write tool
    if (toolCall.id === "write" && toolCall.parameters?.file_path) {
      if (!isPathSafe(toolCall.parameters.file_path)) {
        throw new Error(
          `Shield: write to '${toolCall.parameters.file_path}' blocked. ` +
          `Safe zones: ${SAFE_ZONES.join(", ")}`
        )
      }
    }

    // Intercept bash tool — on Windows this runs PowerShell/cmd, not sh,
    // so the destructive-verb list is different from the original Linux one.
    if (toolCall.id === "bash" && toolCall.parameters?.command) {
      const cmd = toolCall.parameters.command as string
      const destructive =
        /\b(Remove-Item|rm|del|erase|rd|rmdir|Move-Item|mv|move|Set-Acl|icacls|takeown|format)\b/i
      if (destructive.test(cmd)) {
        // Log for human review — deliberately does NOT block, because a regex over
        // shell text cannot reliably tell a safe `rm` from a dangerous one.
        await appendToLog(`[SHIELD WARNING] Destructive command: ${cmd}`)
      }
    }
  }
}
```

> **Be honest about what this buys you.** The `write`-tool check is a real boundary. The `bash`-tool check is a smoke alarm, not a lock: any command string can evade a regex (variables, encoded commands, `&`-chaining, a script that shells out). §13 Phase 8 closes the structural half of this gap by routing *every* write through one atomic writer that Shield gates, so there is no second code path to bypass. Until then, the honest security model is: **Shield stops accidents, not an adversary.** Don't grant Kip credentials on that assumption.

---

#### Step 3.3 — Guardian Scheduled Task

Create `E:\AgentMemoryProject\opencode\kip-scripts\guardian.ps1` — checks OpenCode server health, disk, and vector sync; escalates to Kip only when something is actually wrong:

```powershell
$ErrorActionPreference = 'Continue'
Set-Location 'E:\AgentMemoryProject\opencode'

$issues = @()

# 1. OpenCode server reachable?
try   { Invoke-WebRequest -Uri 'http://localhost:4001/health' -TimeoutSec 5 -UseBasicParsing | Out-Null }
catch { $issues += 'OpenCode server not responding on :4001' }

# 2. Disk headroom on the drive Kip writes to
$drive = Get-PSDrive -Name 'E'
if (($drive.Free / 1GB) -lt 5) { $issues += "Low disk: $([math]::Round($drive.Free/1GB,1)) GB free on E:" }

# 3. Today's daily log exists by evening
if ((Get-Date).Hour -ge 20) {
  $log = "memory\$(Get-Date -Format 'yyyy-MM-dd').md"
  if (-not (Test-Path $log)) { $issues += "No L1 daily log for today ($log)" }
}

if ($issues.Count -eq 0) { exit 0 }

"$(Get-Date -Format s): $($issues -join '; ')" |
  Add-Content -Path 'memory\GUARDIAN_LOG.md'

opencode run --agent kip "Guardian alert: $($issues -join '; '). Diagnose and, if safe, fix. Log what you did to memory\GUARDIAN_LOG.md."
```

Register it on a 30-minute repeat:

```powershell
$action  = New-ScheduledTaskAction -Execute (Get-Command pwsh).Source `
             -Argument '-NoProfile -ExecutionPolicy Bypass -File "E:\AgentMemoryProject\opencode\kip-scripts\guardian.ps1"'
# Task Scheduler has no direct "*:0/30" — a daily trigger with a repetition interval is the equivalent
$trigger = New-ScheduledTaskTrigger -Once -At (Get-Date).Date `
             -RepetitionInterval (New-TimeSpan -Minutes 30) `
             -RepetitionDuration ([TimeSpan]::MaxValue)
Register-ScheduledTask -TaskName 'Kip\Guardian' -Action $action -Trigger $trigger `
  -Settings (New-ScheduledTaskSettingsSet -StartWhenAvailable) `
  -Description 'Kip Guardian auto-healer' -Force
```

> **The original Guardian auto-restarted OpenCode. This one doesn't.** Restarting a service you own is fine; on a desktop, a background task that silently relaunches processes every 30 minutes is a good way to end up with four OpenCode servers fighting over one SQLite file. Step 5.3 adds a guarded restart — read the caveat there before enabling it.

---

#### Step 3.4 — Update Plugin Registration

```typescript
// src/index.ts — final plugin export
import { LogicianTool } from "./tools/logician.js"
import { transcriptHook } from "./hooks/transcript.js"
import { dailyLogHook } from "./hooks/daily-log.js"
import { shieldHook } from "./hooks/shield.js"

export default {
  tools: [LogicianTool],
  hooks: {
    ...transcriptHook,
    ...dailyLogHook,
    ...shieldHook,
  }
}
```

---

#### Stage 3 Acceptance Criteria

- [ ] `opencode run --agent kip "Verify file exists: E:\AgentMemoryProject\opencode\MEMORY.md"` → `{ verified: true }`
- [ ] `opencode run --agent kip "Write a test file to C:\Windows\System32\test.txt"` → blocked by Shield with an error message
- [ ] Guardian task fires every 30 minutes: `schtasks /Query /TN "Kip\Guardian"`
- [ ] `KIP_FAILURE_AUDIT_LOG.md` is written to when Shield fires

---

### Stage 4 — WhatsApp Integration (Days 18–25)

**Goal:** Connect Kip to WhatsApp so the human can send messages and receive replies without a terminal.

---

#### Step 4.1 — Create the WhatsApp Bridge

This is the solution to the "one real limitation" — OpenCode is stateless between invocations. A thin always-on process watches for incoming WhatsApp messages and calls the OpenCode HTTP API.

Create `E:\AgentMemoryProject\opencode\whatsapp-bridge/src/index.ts`:

```typescript
import { createOpencode } from "@opencode-ai/sdk"

// Option A: Use whatsapp-web.js (free, requires WhatsApp account)
// Option B: Use Twilio WhatsApp API (paid, more reliable)
// Option C: Use WhatsApp Business Cloud API (free tier available)

// Example using whatsapp-web.js:
import pkg from "whatsapp-web.js"
const { Client, LocalAuth } = pkg

// Initialize OpenCode SDK
const opencode = await createOpencode({
  baseUrl: "http://localhost:4001",  // OpenCode server on djdesktop
})

// Persistent session ID for the WhatsApp thread
const WHATSAPP_SESSION_ID = "kip-whatsapp-main"

const whatsapp = new Client({
  authStrategy: new LocalAuth({ clientId: "kip" }),
  puppeteer: { args: ["--no-sandbox"] }
})

// Handle incoming messages
whatsapp.on("message", async (msg) => {
  // Only respond to the authorized human's number
  const AUTHORIZED_NUMBER = process.env.AUTHORIZED_WHATSAPP_NUMBER
  if (!msg.from.includes(AUTHORIZED_NUMBER)) return
  if (msg.type !== "chat") return

  try {
    // Send typing indicator
    await msg.getChat().then(chat => chat.sendStateTyping())

    // Create or resume session
    let session = await opencode.sessions.get(WHATSAPP_SESSION_ID).catch(() => null)
    if (!session) {
      session = await opencode.sessions.create({
        agent: "kip",
        id: WHATSAPP_SESSION_ID
      })
    }

    // Send the message to Kip
    const response = await opencode.sessions.prompt({
      sessionId: WHATSAPP_SESSION_ID,
      prompt: msg.body,
    })

    // Collect streamed response
    let fullResponse = ""
    for await (const chunk of response) {
      if (chunk.type === "text") fullResponse += chunk.text
    }

    // Send reply back to WhatsApp
    await msg.reply(fullResponse)

  } catch (err) {
    console.error("Kip bridge error:", err)
    await msg.reply("Kip encountered an error. Check the logs.")
  }
})

whatsapp.initialize()
console.log("Kip WhatsApp bridge started. Scan QR code to authenticate.")
```

---

#### Step 4.2 — Run the Bridge as a Background Task

Windows has no `Restart=always`. There are three ways to get a long-running, auto-restarting process, in increasing order of robustness:

| Approach | Restart on crash | Runs without login | Effort |
|---|---|---|---|
| Task Scheduler, trigger *At startup* + *Restart on failure* | Yes (task-level) | Yes (run as SYSTEM or stored creds) | Low |
| [NSSM](https://nssm.cc/) — wraps any exe as a real Windows service | Yes (service-level) | Yes | Low, needs a download |
| `sc.exe create` with a service wrapper | Yes | Yes | Higher |

Task Scheduler is enough to start. Register both processes:

```powershell
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable `
              -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1) `
              -ExecutionTimeLimit ([TimeSpan]::Zero)   # zero = no time limit
$trigger  = New-ScheduledTaskTrigger -AtStartup

# --- OpenCode server ---
$ocAction = New-ScheduledTaskAction -Execute (Get-Command opencode).Source `
              -Argument 'serve --port 4001' `
              -WorkingDirectory 'E:\AgentMemoryProject\opencode'
Register-ScheduledTask -TaskName 'Kip\OpenCodeServer' -Action $ocAction `
  -Trigger $trigger -Settings $settings -Description 'OpenCode server (Kip runtime)' -Force

# --- WhatsApp bridge ---
$waAction = New-ScheduledTaskAction -Execute (Get-Command bun).Source `
              -Argument 'run src/index.ts' `
              -WorkingDirectory 'E:\AgentMemoryProject\opencode\whatsapp-bridge'
Register-ScheduledTask -TaskName 'Kip\WhatsAppBridge' -Action $waAction `
  -Trigger $trigger -Settings $settings -Description 'Kip WhatsApp bridge' -Force
```

**Ordering.** systemd's `After=opencode.service` has no Task Scheduler equivalent. Both tasks fire at startup with no guaranteed order, so the bridge must tolerate the server not being up yet — retry the `createOpencode({ baseUrl })` call with backoff instead of crashing on first connect. (Task-level restart will eventually paper over this, but a bridge that crash-loops for a minute at every boot will also lose its WhatsApp session.)

**Secrets — read this before pasting keys.** The original put `ANTHROPIC_API_KEY=sk-...` and the phone number directly in the unit file. Do not do the equivalent here: scheduled-task definitions are stored as world-readable XML under `C:\Windows\System32\Tasks\`, and — because `<KIP_ROOT>` is a git repo you may open PRs from (§14) — a `.env` in the working directory is one `git add -A` away from being published. Instead:

```powershell
# User-scoped, not in the repo, not in the task XML
[Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY', 'sk-...', 'User')
[Environment]::SetEnvironmentVariable('AUTHORIZED_WHATSAPP_NUMBER', '+91XXXXXXXXXX', 'User')
```

Tasks running as your user inherit these. If you run a task as SYSTEM instead, it will **not** see user-scoped variables — use `'Machine'` scope, and accept that any process on the box can then read the key. Either way, add `.env` and `*.key` to `.git/info/exclude` now rather than after the first accidental commit.

---

#### Stage 4 Acceptance Criteria

- [ ] `Get-ScheduledTask -TaskName 'Kip\OpenCodeServer'` is `Running`, and `Invoke-WebRequest http://localhost:4001/health` returns 200
- [ ] `Get-ScheduledTask -TaskName 'Kip\WhatsAppBridge'` is `Running`
- [ ] Sending "Hello" to the authorized WhatsApp number gets a Kip-persona response within 30 seconds
- [ ] Sending "What files are in my Obsidian vault?" triggers an Obsidian MCP call and returns results (skip if you dropped L7 in Step 1.4)
- [ ] Sending "What did we decide about the plugin hook names?" triggers a memory search and returns something from a real prior session
- [ ] Rebooting djdesktop brings both tasks back up without manual intervention

---

### Stage 5 — Polish & Hardening (Days 26–30)

**Goal:** Reliability, observability, and the remaining optional layers (L6 Memory Wiki, L10 skill detection).

---

#### Step 5.1 — Memory Wiki Tool (L6)

Add `wiki_synthesize` tool to the plugin:

```typescript
export const WikiSynthesizeTool = tool({
  id: "wiki_synthesize",
  description: "Create or update a synthesis wiki page for a topic, with provenance tracking",
  parameters: {
    type: "object",
    properties: {
      topic: { type: "string" },
      source_dates: {
        type: "array",
        items: { type: "string" },
        description: "YYYY-MM-DD dates of daily logs to synthesize from"
      }
    },
    required: ["topic"]
  },
  execute: async ({ topic, source_dates }) => {
    const slug = topic.toLowerCase().replace(/\s+/g, "-")
    const wikiPath = `${process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"}/memory/wiki/${slug}.md`
    // The agent will read sources and write the synthesis —
    // this tool just creates the file scaffold with provenance header
    const header = `# ${topic}\n\n> Synthesized from: ${(source_dates || []).join(", ")}\n> Last updated: ${new Date().toISOString().slice(0,10)}\n\n`
    await Bun.write(wikiPath, header)
    return { path: wikiPath, status: "scaffold_created" }
  }
})
```

---

#### Step 5.2 — Session-End Skill Detection Hook

Add to `src/hooks/skill-detection.ts`:

```typescript
export const skillDetectionHook = {
  "session.end": async (session: any) => {
    // Ask the agent to self-assess for repeatable procedures
    // This fires a lightweight prompt after every session
    const check = await opencode.sessions.prompt({
      sessionId: session.id,
      prompt: `Skill detection check (answer briefly):
1. Did you perform the same multi-step procedure more than once this session?
2. Did you use a non-obvious fix or workaround?
3. Did you catch yourself about to make a mistake you've made before?
If yes to any: output SKILL_CANDIDATE: <procedure name> | <1-sentence description>
If no: output SKIP`
    })
    // If SKILL_CANDIDATE detected, append to a suggestions file
    if (check.includes("SKILL_CANDIDATE:")) {
      await appendFile(`${process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"}/memory/skill-suggestions.md`,
        `${new Date().toISOString().slice(0,10)} — ${check}\n`)
    }
  }
}
```

---

#### Step 5.3 — Health Monitoring

Add a simple health check endpoint check to Guardian:

```powershell
# In guardian.ps1 — guarded restart of the OpenCode server.
# Only fires if the server has been unreachable on TWO consecutive checks (~30 min apart)
# and no opencode process is already alive. Both guards matter: a single failed health
# probe during a long request would otherwise restart a perfectly healthy server, and
# blind restarts stack up multiple servers on one SQLite file.

$stateFile = 'E:\AgentMemoryProject\opencode\memory\.guardian-state'
$healthy = $false
try   { Invoke-WebRequest -Uri 'http://localhost:4001/health' -TimeoutSec 5 -UseBasicParsing | Out-Null; $healthy = $true }
catch { $healthy = $false }

if ($healthy) { Remove-Item $stateFile -ErrorAction SilentlyContinue; return }

$strikes = if (Test-Path $stateFile) { [int](Get-Content $stateFile) } else { 0 }
$strikes++
Set-Content -Path $stateFile -Value $strikes

if ($strikes -lt 2)                   { return }   # one bad probe is not an outage
if (Get-Process opencode -ErrorAction SilentlyContinue) { return }   # it's alive, just slow

Start-ScheduledTask -TaskName 'Kip\OpenCodeServer'
"$(Get-Date -Format s): OpenCode restarted by Guardian after $strikes failed probes" |
  Add-Content -Path 'E:\AgentMemoryProject\opencode\memory\GUARDIAN_LOG.md'
Remove-Item $stateFile -ErrorAction SilentlyContinue
```

---

#### Step 5.4 — Final Configuration Consolidation

Final `E:\AgentMemoryProject\opencode\.opencode\opencode.jsonc` — again, **merge**, don't replace: the `provider` (Ollama), `permission`, `references`, and `tools` blocks already in this file stay.

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-5",
  "default_agent": "kip",
  "snapshots": true,
  "compaction": {
    "auto": true,
    "buffer": 20000,
    "keep": { "tokens": 8000 }
  },
  "mcp": {
    "obsidian": {
      "type": "local",
      "command": ["npx", "-y", "mcp-obsidian", "<YOUR_OBSIDIAN_VAULT>"],
      "timeout": 30000
    },
    "kip-memory": {
      "type": "local",
      "command": ["python", "E:/AgentMemoryProject/opencode/kip-scripts/memory-mcp-server.py"],
      "timeout": 15000
    }
  },
  "plugin": {
    "kip": "E:/AgentMemoryProject/opencode/kip-plugin"
  }

  // ... plus the existing "provider" (ollama), "permission", "references", "tools" blocks ...
}
```

> **Forward slashes in JSON are deliberate.** `"E:\AgentMemoryProject\..."` in JSON requires doubled backslashes (`E:\\AgentMemoryProject`), and a single backslash before a letter is either an invalid escape or, worse, a valid-but-wrong one. Forward slashes work fine on Windows and remove the whole class of bug.

---

#### Stage 5 Acceptance Criteria

- [ ] `opencode run --agent kip "Synthesize a wiki page for: the Kip plugin hook design"` creates `memory\wiki\kip-plugin-hook-design.md`
- [ ] Skill suggestions file grows over time: `Get-Content memory\skill-suggestions.md`
- [ ] Guardian restarts OpenCode after a real outage: `Stop-Process -Name opencode`, then confirm restart on the *second* Guardian run — and confirm a single slow request does **not** trigger a restart
- [ ] Full system survives a reboot: both tasks come back, WhatsApp bridge reconnects
- [ ] Exactly one `opencode` process is running afterwards (`Get-Process opencode`) — no restart pile-up

---

## 6. Architecture Diagram

```
                    ┌─────────────────────────────────┐
                    │   Human (WhatsApp / terminal)    │
                    └──────────────┬──────────────────┘
                                   │ WhatsApp message
                    ┌──────────────▼──────────────────┐
                    │   kip-whatsapp bridge        │
                    │   (Bun process, Task Sched.)     │
                    └──────────────┬──────────────────┘
                                   │ HTTP POST /session/{id}/prompt
                    ┌──────────────▼──────────────────┐
                    │   OpenCode Server (:4001)        │
                    │   opencode serve (Task Sched.)   │
                    │                                  │
                    │  ┌──────────────────────────┐   │
                    │  │  Kip Agent           │   │
                    │  │  - SOUL/USER/IDENTITY    │   │
                    │  │    (skills)              │   │
                    │  │  - Session compaction    │   │
                    │  │  - MEMORY.md context     │   │
                    │  └──────────┬───────────────┘   │
                    │             │ tool calls         │
                    │  ┌──────────▼───────────────┐   │
                    │  │  Built-in Tools          │   │
                    │  │  read/write/edit/bash    │   │
                    │  │  glob/grep/webfetch      │   │
                    │  └──────────────────────────┘   │
                    │  ┌──────────────────────────┐   │
                    │  │  Kip Plugin          │   │
                    │  │  - logician_verify       │   │
                    │  │  - wiki_synthesize       │   │
                    │  │  - Shield hook           │   │
                    │  │  - L0 transcript hook    │   │
                    │  │  - L1 daily log hook     │   │
                    │  └──────────────────────────┘   │
                    └──────┬────────────┬─────────────┘
                           │            │ MCP protocol
              ┌────────────▼──┐  ┌──────▼──────────────┐
              │ Obsidian MCP  │  │  Kip Memory MCP      │
              │ (vault TBD)   │  │  ChromaDB + NetworkX │
              └───────────────┘  └──────────────────────┘

Windows Task Scheduler (separate processes, each calls `opencode run`):
  08:00 IST → Kip\Ritual-morning
  14:00 IST → Kip\Ritual-afternoon + Kip\MemoryPromote (L9)
  18:00 IST → Kip\Ritual-evening
  every 30m → Kip\Guardian health check
  at startup → Kip\OpenCodeServer, Kip\WhatsAppBridge
```

---

## 7. File & Directory Layout After Implementation

Kip's files interleave with the OpenCode monorepo's own. Entries marked **[oc]** are upstream OpenCode files that already exist — **do not modify or overwrite them**. Everything else is Kip's, and belongs in `.git/info/exclude`.

```
E:\AgentMemoryProject\opencode\           # <KIP_ROOT> — also the OpenCode monorepo clone
│
├── packages\                             # [oc] OpenCode source — untouched
├── sdks\  specs\  nix\  infra\  script\  # [oc] untouched
├── AGENTS.md                             # [oc] OpenCode's contributor guide — NOT Kip's persona
├── CONTRIBUTING.md  README.md            # [oc]
├── APPUNDERSTANDING.md                   # architecture reference (cited throughout this doc)
├── FORIA.md  CONTEXT.md                  # further OpenCode references
├── FUTURE.md                             # superseded earlier draft of this plan ("Patrick")
├── FUTURE_PLANS.md                       # ← this document
│
├── .opencode\
│   ├── opencode.jsonc                    # [oc, MODIFIED] merge Kip's model/mcp/plugin keys in
│   ├── agent\
│   │   └── kip.md                        # Kip agent definition
│   └── skills\
│       ├── kip-soul\SKILL.md             # installed copies (Step 1.3)
│       ├── kip-user\SKILL.md
│       ├── kip-identity\SKILL.md
│       ├── kip-agents\SKILL.md
│       └── kip-tools\SKILL.md
│
├── kip-persona\                          # L3 SOURCE of truth for identity files
│   ├── SOUL.md                           #   (does not exist yet — Step 1.3)
│   ├── USER.md
│   ├── IDENTITY.md
│   ├── AGENTS.md                         #   Kip's, NOT the root [oc] one
│   └── TOOLS.md                          # L10
│
├── memory\                               # exists today: refs\, transcripts\, wiki\ (all empty)
│   ├── 2026-08-20.md                     # L1 daily logs — one per day
│   ├── transcripts\
│   │   └── 2026-08-20.jsonl              # L0 raw transcripts
│   ├── wiki\                             # L6 synthesis pages
│   ├── refs\                             # L4 symbolic offload refs
│   ├── index\                            # L5 ChromaDB store + graph.json
│   ├── skill-suggestions.md              # L10 auto-detected skill candidates
│   ├── GUARDIAN_LOG.md                   # L4 Guardian actions
│   └── .guardian-state                   # Step 5.3 strike counter
│
├── MEMORY.md                             # L2 curated long-term (<120 lines)
├── SESSION-STATE.md                      # L8 current project phase
├── HEARTBEAT.md                          # L8 ritual log
├── KIP_FAILURE_AUDIT_LOG.md              # L4 Shield + Logician failures
│
├── kip-scripts\                          # NOT script\ — that's [oc]
│   ├── ritual.ps1                        # Step 1.5
│   ├── memory-promote.ps1                # L9
│   ├── guardian.ps1                      # L4
│   ├── logician-verify.ps1               # L4
│   ├── memory-mcp-server.py              # L5 MCP wrapper
│   ├── search.py                         # L5 ChromaDB search
│   ├── graph_lookup.py                   # L5 NetworkX lookup
│   └── indexer.py                        # L5 ChromaDB indexer
│
├── kip-plugin\                           # OpenCode plugin package
│   ├── package.json
│   └── src\
│       ├── index.ts
│       ├── tools\
│       │   ├── logician.ts
│       │   └── wiki-synthesize.ts
│       └── hooks\
│           ├── transcript.ts
│           ├── daily-log.ts
│           ├── shield.ts
│           └── skill-detection.ts
│
└── whatsapp-bridge\                      # WhatsApp → OpenCode bridge
    ├── package.json
    └── src\index.ts

%USERPROFILE%\.local\share\opencode\      # OpenCode's own data dir (outside the repo)
└── sessions\                             # session SQLite database
```

Suggested `.git\info\exclude` (local-only; deliberately **not** `.gitignore`, which is tracked upstream):

```gitignore
/kip-persona/
/kip-scripts/
/kip-plugin/
/whatsapp-bridge/
/memory/
/MEMORY.md
/SESSION-STATE.md
/HEARTBEAT.md
/KIP_FAILURE_AUDIT_LOG.md
.env
```

> Note `.opencode/opencode.jsonc` is **not** excluded — it is a tracked upstream file that you are modifying. Keep your Kip additions in a small, reviewable diff, and be ready to stash them before opening any PR from this clone (§14).

---

## 8. Key Code Patterns

### Pattern 1: Effect-style Tool in Plugin

OpenCode core uses Effect v4 everywhere, but the **public plugin API** (`@opencode-ai/plugin`) uses plain async/Promise — you do not need to learn Effect to write a plugin.

```typescript
// Plugin tools use plain async — NO Effect needed
import { tool } from "@opencode-ai/plugin"

export const MyTool = tool({
  id: "my_tool",
  description: "...",
  parameters: { type: "object", properties: { input: { type: "string" } }, required: ["input"] },
  execute: async ({ input }) => {
    // Plain async/await here
    return { result: await doSomething(input) }
  }
})
```

### Pattern 2: Agent Prompt with Skill Loading

```markdown
---
name: kip
prompt: |
  Load your identity skills before every response:
  1. Load skill: kip-soul
  2. Load skill: kip-user
  Always check SESSION-STATE.md at the start of a new conversation.
  Always use logician_verify before claiming a task complete.
---
```

### Pattern 3: Scheduled OpenCode Run

```powershell
# Pattern for all heartbeat/promotion scheduled tasks:
opencode run `
  --agent kip `
  --session-id "kip-$(Get-Date -Format 'yyyyMMdd')-ritual" `
  "Your ritual prompt here. Be specific about what files to read/write."
```

Using a date-based `--session-id` prevents each scheduled run from accumulating into one infinite session while still keeping daily history grouped.

### Pattern 3b: Calling PowerShell from a Plugin Tool

Windows has no `/bin/bash`, and `Bun.spawnSync(["bash", ...])` will fail. Shell out like this instead:

```typescript
const result = Bun.spawnSync([
  "pwsh", "-NoProfile", "-ExecutionPolicy", "Bypass",
  "-File", "E:/AgentMemoryProject/opencode/kip-scripts/thing.ps1",
  "-SomeParam", value,
])
const ok  = result.exitCode === 0
const out = new TextDecoder().decode(result.stdout).trim()
```

Two gotchas that will cost you an hour each if you skip them:

- **`pwsh` vs `powershell`.** `pwsh` is PowerShell 7+ (what these scripts assume); `powershell.exe` is the bundled Windows PowerShell 5.1, which lacks some of the syntax used here. If `pwsh` isn't on PATH, install PowerShell 7 rather than downgrading the scripts.
- **`-ExecutionPolicy Bypass` is required** for unsigned local scripts under the default machine policy — otherwise every call fails with a policy error that looks nothing like a path problem.

### Pattern 4: MCP Server stdio Protocol

```python
# Minimal MCP server pattern (Python):
# Every request comes in as a JSON line on stdin
# Every response goes out as a JSON line on stdout
# That's it — the full protocol in 2 lines
request = json.loads(sys.stdin.readline())
print(json.dumps({"jsonrpc":"2.0","id":request["id"],"result": handle(request)}))
```

---

## 9. The One Real Limitation and How to Solve It

OpenCode is **stateless between invocations** at the agent-context level. There is no background process that "stays alive as Kip." Each `opencode run` or HTTP prompt call starts with a fresh context load (though it incorporates prior session history via compaction).

**This means:** Kip does not "notice" things on its own. It only acts when:
1. A human sends a WhatsApp message (→ whatsapp-bridge calls the API)
2. A Task Scheduler task fires (→ the task calls `opencode run`)
3. A human opens a terminal session

On a desktop rather than an always-on mini-PC, there is a **second** statelessness to plan around: the machine itself sleeps. A ritual that "runs at 08:00" runs at 08:00 *if djdesktop is awake*. Design every ritual to be idempotent and to reconcile from files rather than assume it ran yesterday.

**This is fine for the stated deployment.** Kip is not a daemon — it is a responsive assistant. The WhatsApp bridge gives the appearance of always-on presence. The heartbeat timers give proactive behavior.

**If you ever need genuinely always-on presence** (Kip monitoring a file, watching a sensor, or reacting to an external event stream without human input):

Option A — Use the OpenCode HTTP API in a long-running Bun process:
```typescript
// long-running-monitor.ts
const opencode = await createOpencode({ baseUrl: "http://localhost:4001" })
// Watch a file, API, or sensor
const watcher = fs.watch("E:/AgentMemoryProject/opencode/inbox")
for await (const event of watcher) {
  await opencode.sessions.prompt({
    sessionId: "kip-monitor",
    prompt: `New file detected: ${event.filename}. Process it.`
  })
}
```

Option B — Use the OpenCode event stream (SSE) to react to server-side events:
```typescript
const events = await opencode.events.stream({ sessionId: "kip-monitor" })
for await (const event of events) {
  if (event.type === "tool.result" && event.tool === "my-sensor-tool") {
    // React to tool results in real time
  }
}
```

---

## 10. Acceptance Criteria

### Full System (all stages complete)

- [ ] Sending a WhatsApp message gets a Kip-persona response within 30 seconds
- [ ] Kip responds to "Who are you?" with content from SOUL.md
- [ ] Kip responds to "What are we working on?" from SESSION-STATE.md context
- [ ] Kip can search the Obsidian vault: "Do I have notes on X?" (skip if L7 was dropped)
- [ ] Kip can search ChromaDB: "What do you remember about Y?"
- [ ] Kip writes daily log to `memory/YYYY-MM-DD.md` after each session with `[ATOM]` entries
- [ ] MEMORY.md never exceeds 120 lines (enforced by the `Kip\MemoryPromote` task)
- [ ] Writing to `C:\Windows\System32\drivers\etc\hosts` is blocked by Shield with an error
- [ ] So is `E:\AgentMemoryProject\opencode\memory\..\..\evil.txt` (path traversal) and `e:\agentmemoryproject\opencode-evil\x.txt` (case + prefix)
- [ ] `logician_verify` returns accurate results for file existence checks on Windows paths (both `E:\...` and `E:/...` forms)
- [ ] Guardian auto-restarts OpenCode if it crashes
- [ ] All scheduled tasks survive a reboot of djdesktop
- [ ] HANDOVER sections written in daily log (enabling model switches without context loss)

---

## 11. Open Questions

These require human decisions before or during implementation:

1. **WhatsApp API approach:** `whatsapp-web.js` (free, fragile, needs Chromium) vs Twilio WhatsApp API (paid, reliable) vs Meta WhatsApp Business Cloud API (free tier, requires business verification). Which is acceptable for this deployment?

2. **Embeddings provider:** The current Kip system uses OpenAI for embeddings (noted as `⚠️ not fully local`). For a fully local option, consider `nomic-embed-text` via Ollama on djdesktop. Is the OpenAI API cost acceptable, or should embeddings be fully local?

3. **Session continuity via WhatsApp:** Should all WhatsApp messages go into one persistent session (`kip-whatsapp-main`) for maximum context continuity, or should each day be a fresh session (with compaction providing the bridge)? One session accumulates indefinitely; daily sessions require good compaction summaries.

4. **Skill file sync:** SOUL.md etc. currently exist as primary files in `E:\AgentMemoryProject\opencode\`. After migration, the canonical versions should be in `.opencode/skills/`. Do we keep the originals as symlinks, or do we accept two copies?

5. **OpenCode version pinning:** The djdesktop needs a specific OpenCode version pinned to avoid breaking changes on auto-update. Should `autoupdate` be set to `"notify"` (conservative) or `false` (fully manual)?

6. **Network resilience / offline fallback:** What should Kip do if the Anthropic API is unreachable? Options: (a) queue messages and respond when connectivity returns, (b) fall back to a locally-running Ollama model, (c) respond with a connectivity error immediately. **Note this is closer to decided than the other questions:** this repo's `.opencode/opencode.jsonc` already defines an Ollama provider against `http://localhost:11434/v1` with several local models (`deepseek-r1:32b`, `qwen3-coder:30b`, `mixtral`, others). Option (b) is largely wired already — what's missing is the failover policy, not the provider.

7. **Ritual reliability on a desktop:** rituals depend on djdesktop being awake at 08:00/14:00/18:00. Accept best-effort (idempotent rituals, catch-up runs), or enable `-WakeToRun` so the machine wakes for them? See the caveat in §5 Step 1.5.

8. **Repo cohabitation:** Kip's runtime files currently live inside the OpenCode monorepo clone (§0). Is that still right once you start pulling upstream regularly and opening PRs (§14), or should Kip move to a sibling directory? Revisit after the first painful rebase.

---

---

## 12. Adopt ai-memory + Build the Kip Layers (current plan)

> Added 2026-08-20. **The real target isn't "Kip, a WhatsApp bot for four domains" — it's Kip**: a self-improving personal agent that profiles you and learns your working style, uses OpenCode to make code changes across your projects, runs your homelab, and talks to you over WhatsApp. Everything in §1–§11 above still applies (persona files, WhatsApp bridge design, scheduled rituals) — this section replaces the memory-*engine* design in what is now §13 with a build strategy fit for that fuller scope: **adopt `ai-memory` unmodified as the memory substrate, and build the Kip-specific layers on top of it.**
>
> **Decision made 2026-08-20**: build this way *specifically to learn `ai-memory`'s internals by running them at full intensity* — reading its source as you configure and extend it — with an explicit plan to graduate to §13 (forking and customizing it directly) once comfortable. Each phase below therefore pairs a "what to build" deliverable with a "what to read first" learning task.

### 12.1 Why `ai-memory` fits Kip almost exactly

| Kip requirement | What `ai-memory` already has | Where (see [`ai-memory-overview.md`](ai-memory-overview.md) for detail) |
|---|---|---|
| "Learns my way of working," follows me everywhere | The reserved `_global` scope — union'd into every project query automatically, no per-project setup | §3, §4 of the overview |
| "Do code changes across my projects" without cross-contamination | Per-project isolation by a typed `(workspace_id, project_id)` tuple resolved from `cwd`, plus an explicit `.ai-memory.toml` marker for monorepos/worktrees | §1 "Multi-agent / multi-machine support model" |
| "Self-improving" | The auto-improvement background loop: reviews completed sessions, proposes wiki edits, stages them in an audited pending-writes table, configurable auto-approve, with a pinned-page-immutability safety rail | §1 "Consolidation," `docs/auto-improvement-loop.md` |
| Cross-agent continuity (you'll likely drive Kip from OpenCode *and* occasionally Claude Code/Codex directly) | `ai-memory run` managed workstreams + `memory_handoff_*` — already supports OpenCode alongside 11 other harnesses | §1 "Workstream / session-resume feature" |
| Talks over WhatsApp | Not native — this is genuinely new work (§12.3, Phase J5) | — |
| Runs your homelab | Not native — this is genuinely new work (§12.3, Phase J6) | — |
| A named persona ("Kip," not a generic assistant) | Not native as a *concept*, but trivially expressed as `_global`-scope identity pages with `canonical`/`pinned` tags | §12.3, Phase J2 |

The takeaway: the hard, easy-to-get-wrong 80% (durable storage, retrieval quality, self-improvement safety rails, multi-project scoping) is already built and battle-tested. What's actually novel to Kip is the persona, the WhatsApp mouth, and the homelab hands — a much smaller, much safer scope to build from scratch than a whole memory engine.

### 12.2 Phase plan — adopt & extend

Same "fits in ~100k tokens per AI session" sizing as before, but most early phases are **configuration and reading, not new code** — that's intentional; it's how "learn by running it" works.

---

#### Phase J1 — Stand up `ai-memory`, wired to OpenCode, on one real repo

- **Goal**: Get `ai-memory` installed, configured, and talking to OpenCode on a single project before touching anything Kip-specific.
- **Read first**: `docs/ARCHITECTURE.md` in full (steady-state data flow diagram, the config.toml reference) — this is genuinely worth reading end to end once, not skimmed; it's the map you'll use for every later phase.
- **Do**:
  ```powershell
  ai-memory init
  ai-memory install-mcp --client opencode
  ai-memory install-hooks --agent opencode
  ai-memory serve   # or register as a Task Scheduler task / Windows service from the start
  ```
  **Check Windows support first.** `ai-memory` is a Rust workspace; confirm it ships a Windows build (or that it builds from source with your toolchain) before planning six phases around it. If it's Linux/macOS-only, the realistic options are WSL2 or a rethink of §12 — find that out in an hour, not in Phase J3.
  (Verify exact flags against the installed version's `--help` and the current README Support Matrix row for OpenCode — "Remote MCP config + generated TypeScript plugin" — since CLI surfaces drift between releases.)
- **Deliverables**: a working local `ai-memory` server; a normal OpenCode coding session in one repo that visibly produces a session summary page on exit; a `config.toml` you've hand-edited at least once (e.g. to point at your chosen LLM provider) and understand every key you changed.
- **Acceptance**:
  - [ ] `ai-memory mcp list`-equivalent (or OpenCode's `mcp list`) shows `ai-memory` connected.
  - [ ] After a real coding session, `memory_recent` (via OpenCode) or `ai-memory show` returns that session's summary.
  - [ ] You can explain, in your own words, what each top-level `config.toml` section does.

---

#### Phase J2 — Kip persona in `_global` scope

- **Goal**: Give the agent a name and a consistent identity across every project, using `ai-memory`'s intended mechanism instead of a bespoke one.
- **Read first**: the `_global` scope contract in `docs/ARCHITECTURE.md`/`docs/design-decisions.md`, and the pinned-page-immutability behavior (search the store crate's tests for how `approve` refuses to touch pinned pages) — you want to understand *why* pinning matters before you rely on it.
- **Do**: write Kip's identity/persona/user-preference pages via `memory_write_page` with `scope: "global"`, tagged `canonical` and `pinned` so the self-improvement loop (Phase J4) can never silently rewrite who Kip is.
- **Deliverables**: a small set of global pages (identity, standing preferences, coding style) committed to the wiki.
- **Acceptance**:
  - [ ] Asking "who are you?" in *any* repo returns Kip's persona with zero per-repo config.
  - [ ] A `memory_query` from an unrelated project shows the global pages as `global_scope_hits`.
  - [ ] Attempting to auto-improve-rewrite a pinned identity page is refused (verify this deliberately — don't just assume it).

---

#### Phase J3 — Cross-repo validation

- **Goal**: Prove the actual claim "codes across my projects without cross-contamination" — this is the phase that validates the whole strategy, not just a feature.
- **Read first**: nothing new — apply what J1/J2 taught you about scope resolution.
- **Do**: run real Kip-assisted sessions in two unrelated repos. Record a project-specific decision in repo A; set a coding-style preference in `_global`.
- **Deliverables**: a short personal note (can literally be a memory page) on how scope resolution behaved in practice.
- **Acceptance**:
  - [ ] The repo-A decision does **not** appear when querying from repo B.
  - [ ] The global coding-style preference **does** appear in both.
  - [ ] You can name the exact rule (`cwd` → nearest `.ai-memory.toml` → repo root) that decided project identity in each case.

---

#### Phase J4 — Self-improvement, turned on deliberately

- **Goal**: Enable the auto-improve loop with eyes open, not as a default you never looked at.
- **Read first**: `docs/auto-improvement-loop.md` and `docs/auto-improve-eval-gates.md` in full.
- **Do**: start with `[auto_improve] require_approval = true` (safer while learning). Review a batch of real proposals through `memory_auto_improve`/`ai-memory pending-writes`. Once you trust the pattern of what it proposes, decide deliberately whether to flip to auto-approve, optionally scoped with an eval gate on `_rules/`/`procedures/`.
- **Deliverables**: a documented `[auto_improve]` config decision (and why), reviewed in this doc or as a memory page itself.
- **Acceptance**:
  - [ ] At least one real proposal reviewed end-to-end, approved or rejected, with your own reasoning written down.
  - [ ] You can explain what the pending-writes audit trail would show a year from now if something went wrong.

---

#### Phase J5 — WhatsApp bridge

- **Goal**: Same shape as the original Stage 4 (§5) but pointed at the real OpenCode+`ai-memory` stack instead of a placeholder script.
- **Do**: reuse §5 Stage 4's bridge design (`@opencode-ai/sdk`, persistent session id, always-on background task) — no `memory-mcp-server.py` needed anymore, since `ai-memory`'s own MCP server is already the memory backend.
- **Acceptance**: identical to §5 Stage 4's acceptance criteria — a WhatsApp message gets a Kip-persona reply informed by real memory within 30 seconds.

---

#### Phase J6 — Homelab skill

- **Goal**: Give Kip hands, scoped to your homelab, and teach it to remember what it learns about your infrastructure.
- **Do**: build (or adopt) an MCP server exposing your actual homelab control surface (SSH commands, container/VM APIs, whatever you already run). Language choice here is genuinely open — Rust if you want the practice, TypeScript if you want speed, doesn't need to match the memory engine's language since it talks to `ai-memory` only via MCP/HTTP like everything else. Wire a convention: any non-obvious fix Kip performs on the homelab gets written back as a `gotchas/`/`procedures/`-kind page via `memory_write_page`.
- **Acceptance**:
  - [ ] A real homelab action (e.g., restarting a service) works end-to-end through Kip.
  - [ ] The first time a fix requires a non-obvious workaround, it leaves a durable `gotchas/` page behind — and the *second* time you hit the same problem, Kip recalls it via `memory_query` instead of re-deriving the fix.

---

### 12.3 Note on scope

This is intentionally light on new systems programming. Phases J1–J4 are mostly configuration, reading, and validation — that's the point of choosing "adopt, learn, then fork later" over building from scratch. The only genuinely new code is J5 (WhatsApp bridge — small, already designed in §5) and J6 (homelab MCP server — new, but a bounded integration, not a storage engine).

---

## 13. Future Path — Forking ai-memory for Full Customization

> This section is the plan that was originally written as the primary path (2026-08-20, before the Kip scope was clarified) — a from-scratch reimplementation of `ai-memory`'s and `OpenViking`'s best ideas as new Rust crates. **It is now deferred**: per the decision in §12, run `ai-memory` unmodified first, learn its internals by using them, and come back here once you're comfortable enough with that codebase to fork and diverge it deliberately (Option 2 from the original build-strategy discussion). Keep this section as-is; it will still be directionally correct when you're ready — the ideas it borrows from `OpenViking` in particular (L0/L1/L2 tiered `memory://` addressing, observable retrieval trajectories, the session Create→Interact→Commit lifecycle with a diff-audit log) are things `ai-memory` itself doesn't have, and are the most likely reasons you'd eventually want to fork rather than stay on stock `ai-memory` forever.

### 13.0 Strategic note before you start

The Stages 1–5 plan above (§5) is still the fastest path to *something working*. This section is a different thing: a **redesign of the memory engine's internals** — Kip's L0–L10 layers, reimplemented with the retrieval quality, durability, and observability discipline that `ai-memory` and `OpenViking` have already spent real engineering effort on. Use it when Stage 2's "MCP-wraps-your-existing-Python-scripts" approach starts showing its limits (bad recall, no versioning, no way to tell *why* the agent surfaced a memory, MEMORY.md pruning that loses good context).

Two honest alternatives worth weighing before committing to a multi-week custom build:

- **Just run `ai-memory` directly.** Its README already lists OpenCode as "Supported: Remote MCP config + generated TypeScript plugin." If Kip's persona/skills/deterministic layer/WhatsApp bridge can sit *on top of* an unmodified `ai-memory` server, you get 49 migrations' worth of hardened SQLite+FTS5+RRF+consolidation for free, in exchange for giving up some control over the exact `[ATOM]` schema and the 120-line `MEMORY.md` convention. Worth a 1-day spike before committing to Phase 1 below.
- **Build custom, but steal the *design*, not the code.** This is what the phase plan below does. Rationale: Kip has requirements neither project fully covers out of the box (WhatsApp bridge, Logician/Shield/Guardian deterministic layer, a single-user persona, scheduled rituals) and both source projects are large multi-language codebases (Rust workspace / Python+Rust+3 SDKs) that are heavier than one WhatsApp agent needs. The phases below re-implement the *proven parts* of both designs natively in the TypeScript/Bun stack Kip's OpenCode plugin already uses — no new runtime, no second language to maintain.

If you (or an AI agent) pick "build custom," the rest of this section is the roadmap.

### 13.1 What we're borrowing, and from where

| Idea | Source | Why it matters for Kip | Kip layer it upgrades |
|---|---|---|---|
| Markdown-in-git is the source of truth; a local index is *derived*, never authoritative | `ai-memory` (`docs/design-decisions.md` §3) | Kip already does this instinctively (Markdown everywhere, "no opaque vector piles" is literally OpenViking's pitch too) — this formalizes it as an invariant so future code can't accidentally make SQLite/ChromaDB the source of truth | L1, L2, L6, L7 |
| Single-writer actor + atomic tmp+rename+fsync writes | `ai-memory` (`ai-memory-store/src/writer.rs`, `ai-memory-wiki/src/atomic.rs`) | Kip's memory files are written by ad hoc plugin hooks + scheduled scripts with no concurrency control — a WhatsApp message and a ritual task firing at the same moment could corrupt `MEMORY.md` | L1, L2, L8 |
| `Sanitized<T>` as the single typed boundary for untrusted text | `ai-memory` (`ai-memory-core/src/sanitize.rs`) | Every byte that reaches memory today started as a WhatsApp message or LLM output — a single sanitize-on-ingest chokepoint (size caps, redaction) is cheap insurance | L0, L1 |
| Hybrid retrieval: FTS5 + entity-match + graph-neighbor + optional vector, fused by RRF, then authority-aware re-ranking (never a hard filter) | `ai-memory` (`ai-memory-store/src/reader.rs::hybrid_search_inner`) | Kip's L5 (ChromaDB) is pure vector search with no lexical fallback and no way to say "trust `_rules/`-equivalent pages more than a random session note" | L5, L6 |
| Memory tiers with an explicit decay function (working / episodic / semantic / procedural) | `ai-memory` (`docs/ARCHITECTURE.md` tier table) | Gives Kip's L0→L2 promotion pipeline (currently ad hoc `[ATOM]` scoring + a 120-line hard cap) a principled formula instead of a magic number | L0–L2, L9 |
| Consolidation as an explicit "compile" step, versioned by supersession, never a destructive overwrite | `ai-memory` (`ai-memory-consolidate/src/consolidator.rs`) | `MEMORY.md` pruning today just deletes low-score lines; supersession keeps them recoverable via `git log` | L2, L9 |
| A narrow, "every tool has to earn its slot" MCP surface | `ai-memory` (18 tools total, `docs/design-decisions.md` §10) | A concrete naming/scoping convention for the `memory_*` tools Kip's plugin will expose, instead of inventing ad hoc tool names per stage | L4, L5, L10 |
| Cross-agent/cross-session **handoff** as a first-class object | `ai-memory` (`memory_handoff_begin/accept/cancel`) | Formalizes the "HANDOVER (for model switches)" section already sketched in Stage 2's `L1_TEMPLATE` | L1, L8 |
| `viking://` — a single URI scheme addressing memories, resources, and skills as one browsable tree | OpenViking (`docs/en/concepts/04-viking-uri.md`) | Gives Kip's plugin tools (`memory_search`, `memory_graph_lookup`, Obsidian MCP, ChromaDB MCP) one consistent addressing scheme instead of three different query shapes | L5, L6, L7, L10 |
| L0 (abstract) / L1 (overview) / L2 (details) sidecars generated **per directory**, loaded only as deep as needed | OpenViking (`docs/en/concepts/03-context-layers.md`) | This *is* Kip's stated design philosophy ("progressive disclosure... drills down to raw evidence on demand") — OpenViking has a concrete, working implementation of exactly that idea to copy | L0–L2, L6 |
| Hierarchical, directory-recursive retrieval that preserves an **observable trajectory** | OpenViking (`HierarchicalRetriever`, `DebugService`) | Directly useful for Guardian/Logician-style deterministic debugging: "why did the agent recall this?" becomes answerable instead of a vector-similarity black box | L4, L5 |
| Session lifecycle Create → Interact → Commit, with an async Phase 2 that writes a `memory_diff.json` audit log of every add/update/delete | OpenViking (`docs/en/concepts/08-session.md`) | A concrete, better-specified version of Stage 2.3's `session.compaction.ended` hook — the diff-log idea alone gives Kip a transparency trail for what got silently learned each session | L1, L9 |
| Richer memory-type taxonomy (`preferences`, `entities`, `events`, `identity`, `cases`, `experiences` vs. Kip's single `[ATOM]` type) | OpenViking (`docs/en/concepts/02-context-types.md`) | Optional refinement — only adopt if the single-`[ATOM]` schema starts feeling too coarse in practice | L1 |

### 13.2 Target architecture after this phase plan

One additional always-on local process, `kip-memory-server` (Bun/TypeScript), replaces the split "plugin hooks + separate Python MCP script" from Stage 2. It:

- owns a single SQLite file (`E:\AgentMemoryProject\opencode\memory\memory.db`, via `bun:sqlite`, which ships with FTS5 compiled in — no extra native dependency) as the **derived index only**;
- owns the existing Markdown tree (`memory/`, `MEMORY.md`, `wiki/`) as the **source of truth**, written exclusively through one atomic-write function;
- exposes a `memory://` URI scheme (`memory://memories/...`, `memory://resources/...`, `memory://skills/...`, mirroring `viking://`'s scope split) so every tool addresses memory the same way;
- exposes a narrow MCP tool surface to OpenCode's plugin (see per-phase tool lists below) instead of the ad hoc `memory_search`/`memory_graph_lookup` pair from Stage 2.4;
- is the *only* process that opens the SQLite file for writing (single-writer discipline).

```
OpenCode (Kip agent)
   │  MCP (stdio)
   ▼
kip-memory-server (Bun/TS, long-running, Task Scheduler)
   ├─ memory://  URI resolver  ──┐
   ├─ hybrid retrieval (FTS5+entity+graph+vector RRF + authority rank)
   ├─ consolidation/compile pipeline (LLM, optional — degrades to rule-based)
   ├─ tiers + decay + forget-sweep
   ├─ session lifecycle (create/interact/commit) + memory_diff.json audit
   └─ single-writer SQLite actor + atomic markdown writer
         │
         ▼
   memory/*.md, MEMORY.md, wiki/*.md  (git repo — source of truth)
   memory.db  (SQLite — FTS5 + entities + vectors, derived, rebuildable)
```

### 13.3 Phase plan

Each phase below is scoped so a single AI coding session (~100k tokens of context) can read the relevant slice of this doc plus the two overview files, implement it, write tests, and stop at a clean acceptance boundary — without needing to hold the whole system in context at once. **Do not start a phase without re-reading its own section here plus the specific `ai-memory`/`OpenViking` file paths it cites** (those are in the two overview docs, not reproduced in full here, to keep each phase's context load small).

Each phase entry has: **Goal**, **Borrows from**, **Build on top of**, **Deliverables**, **Out of scope this phase**, **Acceptance criteria**.

---

#### Phase 1 — Storage foundations (single-writer SQLite + atomic markdown writes)

- **Goal**: Replace ad hoc `fs.writeFile`/scheduled-script writes with one durable storage core: a `bun:sqlite` database opened by exactly one process, and one atomic-write function every markdown mutation must go through.
- **Borrows from**: `ai-memory-store/src/writer.rs` (single-writer actor pattern), `ai-memory-wiki/src/atomic.rs` (tmp+rename+fsync), `ai-memory-core/src/sanitize.rs` (typed sanitize boundary).
- **Build on top of**: nothing yet — this is the new foundation layer, separate from (not replacing) the existing `kip-plugin` from Stage 2.
- **Deliverables**:
  - `kip-memory-server/` package scaffold (`bun init`, `package.json`, `src/index.ts` as a long-running process, not a plugin).
  - `src/store/db.ts` — opens `memory.db`, WAL mode, creates schema (`observations`, `pages`, `sessions` tables) via a tiny sequential-migration runner (`src/store/migrations/001_init.sql`, ...).
  - `src/store/writer.ts` — a single in-process queue/actor that serializes all writes (even though Bun's SQLite is synchronous, this makes the invariant explicit and future-proofs against a multi-process split later).
  - `src/wiki/atomic.ts` — `writeAtomic(path, content)`: write to `<path>.tmp`, `fsync`, `rename`.
  - `src/sanitize.ts` — `Sanitized<T>` wrapper type + one `sanitizeObservation()` function (size caps: 16 KiB for prompts/summaries, 2 KiB for tool excerpts, matching `ai-memory`'s documented limits) — the only function allowed to construct a "trusted" observation.
  - A `README.md` in the new package stating the invariant in one sentence: *"No code outside `writer.ts` opens `memory.db` for writing; no code outside `atomic.ts` writes a `.md` file."*
- **Out of scope this phase**: no MCP tools yet, no retrieval, no consolidation. This phase produces a library with unit tests, not a runnable agent-facing feature.
- **Acceptance criteria**:
  - [ ] `bun test` covers: concurrent writes to the same page don't corrupt it (spawn N concurrent `writeAtomic` calls, assert final content is one full write, never a torn write).
  - [ ] Killing the process mid-write (simulate by throwing after tmp-file write, before rename) never leaves a truncated `.md` file in place — only a stray `.tmp`.
  - [ ] `sanitizeObservation()` truncates oversized input and rejects `null`/non-string bodies with a typed error, verified by tests.

---

#### Phase 2 — Capture pipeline v2 (hooks → sanitize → spool)

- **Goal**: Route OpenCode plugin lifecycle events through Phase 1's storage core instead of directly appending JSONL (upgrades Stage 2.2's `transcriptHook`).
- **Borrows from**: `ai-memory-hooks/src/router.rs` (event→ObservationKind mapping), `ai-memory-hooks/src/capture_policy.rs` (nearest-marker path-exclusion policy — adapt as a simple `.kipignore`-style config if useful for Kip's four domains).
- **Build on top of**: Phase 1's `db.ts`, `writer.ts`, `sanitize.ts`.
- **Deliverables**:
  - `src/capture/observationKind.ts` — a typed enum mirroring `ai-memory`'s `ObservationKind` (`session-start`, `user-prompt`, `pre-tool-use`, `post-tool-use`, `notification`, `stop`, `session-end`), adapted to whatever OpenCode plugin hook names actually exist (verify against `packages/plugin/src/index.ts` in the OpenCode source, per `APPUNDERSTANDING.md` §2.10 *Plugin System* and §2.11 *Skill System* — note the original cited §2.6/§2.7, which are Built-in Tools and MCP Integration, not the plugin surface; don't assume 1:1 parity with Claude Code's hook names).
  - `src/capture/router.ts` — the plugin hook handler that classifies each event, sanitizes it, and enqueues a write via `writer.ts`.
  - A thin `kip-plugin` hook (`src/hooks/transcript.ts` from Stage 2.2) now calls `router.ts`'s HTTP or IPC endpoint instead of writing JSONL directly — decide IPC shape now (local HTTP on a fixed port is simplest and matches `ai-memory`'s own `/hook` design; a Unix socket is an alternative if you want to avoid a port).
- **Out of scope this phase**: no daily-log synthesis yet (that's Phase 4). This phase only gets raw observations durably and safely into `memory.db`'s `observations` table — the L0 transcript layer.
- **Acceptance criteria**:
  - [ ] After a real OpenCode session, `SELECT * FROM observations` shows sanitized, size-capped rows for every hook that fired.
  - [ ] An oversized user prompt (>16 KiB) is truncated, not rejected outright — session should never fail because memory couldn't keep up.
  - [ ] The plugin hook call has a hard client-side timeout (≤200ms, matching `ai-memory`'s documented invariant) so a slow/stuck memory server never blocks the agent's response.

---

#### Phase 3 — `memory://` addressing & tiered reads (L0/L1/L2 sidecars)

- **Goal**: Give every memory/resource/skill directory a `memory://` URI and generate cheap L0 (abstract)/L1 (overview) sidecars so the agent can browse before it reads.
- **Borrows from**: OpenViking `docs/en/concepts/04-viking-uri.md` (URI scheme, scope split) and `docs/en/concepts/03-context-layers.md` (L0/L1/L2 sidecar generation, bottom-up aggregation, `freshness`/sampling to avoid re-summarizing unchanged trees — read the "known open issue" callout in [`OpenViking-overview.md`](OpenViking-overview.md) §3 before implementing the bubbling logic, and implement the fix *from the start* rather than the naive version OpenViking itself is still patching).
- **Build on top of**: Phase 1's atomic writer (sidecars are just `.md` files written the same way).
- **Deliverables**:
  - `src/addressing/uri.ts` — `memory://memories/...`, `memory://resources/...`, `memory://skills/...` parse/resolve functions mapping to real filesystem paths under Kip's existing `memory/`, `vault/`, `.opencode/skills/` directories.
  - `src/sidecar/generate.ts` — on a debounced timer (not per-write, to avoid the write-amplification bug OpenViking's own docs flag), generates/refreshes `.abstract.md` (≤256 chars, one sentence) and `.overview.md` (≤2000 chars — deliberately smaller than OpenViking's 4000, since Kip's whole `MEMORY.md` budget is 120 lines) per directory, bottom-up.
  - `src/tools/memory_ls.ts`, `memory_tree.ts`, `memory_read.ts` — MCP tools: `ls`/`tree` return names + L0 abstracts (cheap); `read` takes a `tier: "abstract" | "overview" | "detail"` parameter.
- **Out of scope this phase**: no search/ranking yet — this phase is pure browsing + tiered reading, no relevance scoring.
- **Acceptance criteria**:
  - [ ] `memory_tree("memory://memories/")` returns a directory tree with each node's L0 abstract inline, costing a small fraction of the tokens a full `memory_read` of every file would.
  - [ ] Editing one file inside a large directory does **not** trigger a full-tree sidecar regeneration — only that file's ancestors, and only if their own summary would actually change (coalesced, not per-write).
  - [ ] `memory_read(uri, tier="detail")` returns byte-identical content to reading the underlying file directly.

---

#### Phase 4 — Consolidation / compile pipeline (daily log → atoms → durable pages)

- **Goal**: Formalize Stage 2.3's `dailyLogHook` into a real compile step: raw session → `[ATOM]`-tagged daily log (rule-based, no LLM required) → optional LLM consolidation into durable pages, versioned by supersession instead of overwrite.
- **Borrows from**: `ai-memory-consolidate/src/consolidator.rs` (batch consolidation, JSON-schema-constrained LLM call), `ai-memory-wiki`'s supersession model (`supersedes` chain + `is_latest` flag — adapt as simple frontmatter fields, no need for a full SQL chain), OpenViking's `memory_diff.json` audit log (`docs/en/concepts/08-session.md`) for transparency into what changed each session.
- **Build on top of**: Phase 1 (atomic writes, sanitize), Phase 2 (observations to summarize).
- **Deliverables**:
  - `src/consolidate/synthesize.ts` — rule-based, no-LLM: turns the session's observations into the `L1_TEMPLATE` daily-log entry (this works even with zero LLM configured — matches `ai-memory`'s "zero-LLM default path" invariant, and is a real resilience win for offline/degraded-network operation, per Open Question #6 in §11).
  - `src/consolidate/extractAtoms.ts` — replaces Stage 2.3's stub `extractAtoms()` with real pattern-based extraction (decision/lesson/preference/event/fact heuristics), producing `[ATOM]` lines.
  - `src/consolidate/llmCompile.ts` — **optional**, only runs if an LLM is configured: rewrites the day's atoms into `MEMORY.md` promotions and/or new `wiki/*.md` pages, using a JSON-schema-constrained structured output call (no free-text parsing of LLM responses — matches `ai-memory`'s invariant #7).
  - `src/consolidate/supersede.ts` — any page rewrite adds `supersedes: <old-version-id>` frontmatter and writes the new version atomically instead of overwriting; old versions stay on disk under `memory/_history/` or equivalent (or just rely on `git log` — decide based on how much you want queryable in SQLite vs. left to git).
  - `src/consolidate/diffLog.ts` — writes `memory/_diffs/<date>-<session>.json` per commit: every add/update/delete with before/after snippets (OpenViking's audit-trail idea).
  - Update the L9 promotion task prompt (§5 Step 1.6 in this doc) to call `llmCompile.ts` via a CLI entrypoint instead of a raw `opencode run` free-text prompt — more reliable than asking the LLM to self-score atoms in an unstructured chat turn.
- **Out of scope this phase**: no retrieval yet — pages exist and are versioned, but nothing ranks or searches them (Phase 5).
- **Acceptance criteria**:
  - [ ] With **no LLM configured**, a full session still produces a valid daily log with `[ATOM]` entries — memory capture never depends on LLM availability.
  - [ ] Promoting an atom to `MEMORY.md` never deletes the previous version outright — `git log -p MEMORY.md` shows the supersession.
  - [ ] `memory/_diffs/` contains one JSON file per consolidation run, and its `adds`/`updates`/`deletes` counts match what actually changed on disk.

---

#### Phase 5 — Hybrid retrieval (FTS5 + entity + graph + vector, RRF-fused, authority-ranked)

- **Goal**: Replace the single ChromaDB vector call from Stage 2.4 with `ai-memory`'s multi-stream fused retrieval — this is the single highest-leverage upgrade in this whole plan for actual recall quality.
- **Borrows from**: `ai-memory-store/src/reader.rs::hybrid_search_inner` (RRF fusion, `k=60`), `ai-memory-store/src/fts_query.rs` (FTS5 query sanitization — OR-join natural language, avoid the AND-default recall trap), the `entities:` frontmatter + entity-match stream design, and the bounded authority-multiplier re-ranking (page kind / tier / pinned / tags — clamped, never a hard filter).
- **Build on top of**: Phase 1 (SQLite + FTS5 virtual table), Phase 4 (pages to index).
- **Deliverables**:
  - `src/store/migrations/002_fts.sql` — `pages_fts` FTS5 virtual table + triggers to keep it synced with the `pages` table.
  - `src/retrieve/ftsQuery.ts` — sanitizes/rewrites free-text queries the way `ai-memory`'s `prepare_fts5_query` does (OR-join bare multi-word queries).
  - `src/retrieve/entities.ts` — during consolidation (Phase 4), extract up to ~10 canonical nouns per page into an `entities` column/table; implement exact/prefix matching for the entity stream.
  - `src/retrieve/vector.ts` — keep the existing ChromaDB call as the vector stream (don't rebuild a vector DB from scratch — `ai-memory` itself defers this; reuse what Stage 2.4 already wired), OR swap to Kip's own brute-force cosine over locally-stored embeddings if you want to drop the ChromaDB dependency entirely — either is defensible, pick one and document the choice.
  - `src/retrieve/rrf.ts` — fuses the (FTS5, entity, vector) streams via `1/(k + rank)`, `k=60`.
  - `src/retrieve/authority.ts` — bounded multiplier from page `kind` (treat `_rules/`, `SOUL.md`-equivalent identity pages as `rule`-tier; daily logs as `session`-tier) and `pinned`/`canonical` tags, clamped to roughly `[0.55, 1.5]` like `ai-memory`'s.
  - `src/tools/memory_query.ts` — the MCP tool replacing Stage 2.4's `memory_search`, with an `explain: boolean` parameter that returns per-stream RRF contributions (useful for debugging recall problems later).
- **Out of scope this phase**: no reranker LLM pass yet (defer — `ai-memory` treats it as optional and fail-open; add only if plain RRF+authority isn't good enough in practice).
- **Acceptance criteria**:
  - [ ] A natural-language multi-word query (e.g. "homelab container restart decisions") returns relevant hits even when no single page contains all three words verbatim (proves the OR-join + entity + graph streams are contributing, not just literal FTS AND-match).
  - [ ] A `_rules/`-equivalent identity page and a random old session page that both match a query lexically equally well — the identity page ranks higher (proves authority weighting works).
  - [ ] `explain=true` output lets you answer "why did this come back" for at least 3 manually-inspected queries.

---

#### Phase 6 — Tiers, decay & forget-sweep

- **Goal**: Replace the flat "keep everything / cap `MEMORY.md` at 120 lines" model with `ai-memory`'s four-tier decay policy, mapped onto Kip's existing L0–L10 layers.
- **Borrows from**: `ai-memory`'s tier table (Working/Episodic/Semantic/Procedural, `docs/ARCHITECTURE.md`), its tombstone-based eviction (never a hard delete without a recovery path) instead of a silent prune.
- **Build on top of**: Phase 4 (pages have a tier), Phase 5 (retrieval already reads tier, via authority.ts).
- **Deliverables**:
  - Tier mapping decision, written into this doc as a small table once decided: e.g. L0 transcripts → Working (session-only), L1 daily logs → Episodic (30d hot / 180d cold), L2 `MEMORY.md` + L3 identity + L10 skills → Semantic (indefinite, supersede-only), procedures/gotchas → Procedural (frequency-decay).
  - `src/decay/score.ts` — implements the decay formula (exponential for episodic, frequency-based for procedural).
  - `src/decay/sweep.ts` — the forget-sweep: soft-deletes (tombstones) instead of `rm`, run on a schedule (extends the existing `Kip\MemoryPromote` task).
  - `memory_forget_sweep` MCP tool (manual trigger + dry-run mode, matching `ai-memory`'s tool).
- **Out of scope this phase**: no UI for reviewing tombstoned pages — `git log`/a tombstone table is enough for v1.
- **Acceptance criteria**:
  - [ ] A 90-day-old episodic page that hasn't been accessed scores lower than a 90-day-old page that was read last week (recency/reinforcement working).
  - [ ] Running `memory_forget_sweep --dry-run` reports what *would* be evicted without touching disk.
  - [ ] A tombstoned page's content is still recoverable (`git show` or an explicit "restore" path) for at least 30 days after eviction.

---

#### Phase 7 — Session lifecycle & handoff (Create → Interact → Commit + cross-agent handoff)

- **Goal**: Formalize session boundaries and give Kip a real handoff mechanism for model/session switches, instead of a free-text `HANDOVER` section the next session may or may not read carefully.
- **Borrows from**: OpenViking's Create → Interact → Commit lifecycle with a sync Phase 1 (fast) + async Phase 2 (background extraction) split (`docs/en/concepts/08-session.md`); `ai-memory`'s `memory_handoff_begin/accept/cancel` as a structured object instead of prose.
- **Build on top of**: Phase 2 (capture), Phase 4 (consolidation), Phase 6 (tiers).
- **Deliverables**:
  - `src/session/lifecycle.ts` — wraps session-end: Phase 1 (sync) immediately archives the raw transcript and returns; Phase 2 (background, non-blocking) runs consolidation (Phase 4) + decay scoring (Phase 6) + writes the diff log (already built in Phase 4).
  - `src/tools/memory_handoff_begin.ts` / `memory_handoff_accept.ts` — structured handoff object (what was being worked on, open questions, next move) written at session end, read at the *next* session's start instead of relying on the agent to notice a `HANDOVER` heading in prose.
  - Wire this into the existing scheduled ritual scripts (§5 Step 1.5/1.6) so morning/afternoon/evening rituals check for a pending handoff explicitly.
- **Out of scope this phase**: no multi-user/peer concept — Kip is single-user, skip OpenViking's `peers/` entirely.
- **Acceptance criteria**:
  - [ ] A WhatsApp message right after a session ends gets a fast reply (Phase 1 sync path) without waiting for background consolidation to finish.
  - [ ] Starting a new session after a mid-task interruption surfaces the pending handoff automatically, not just as inline prose the agent might skim past.
  - [ ] Killing the memory server mid-Phase-2 (background extraction) leaves no partial/corrupt page — Phase 1's guarantees from `atomic.ts` hold here too.

---

#### Phase 8 — Deterministic layer integration & observability

- **Goal**: Wire Stage 3's Logician/Shield/Guardian into the new storage/retrieval core's invariants, and add OpenViking-style observable retrieval trajectories so "why did the agent recall/act on this" is always answerable.
- **Borrows from**: OpenViking's `DebugService`/`ObserverService` + `query_plan`/`query_results` trajectory capture; `ai-memory`'s "live-process check before destructive ops" invariant (apply this inside Shield, not just at the CLI level).
- **Build on top of**: everything above — this phase is integration, not new subsystems.
- **Deliverables**:
  - Update Shield (Stage 3.2) to route all path-safety checks through Phase 1's `atomic.ts` (a write that fails Shield's check should never even reach the atomic-writer, closing a small gap in the original Stage 3 design where Shield and the write path were separate).
  - `src/retrieve/trajectory.ts` — `memory_query` (Phase 5) records which directories/streams contributed each hit; a new `memory_explain_last_query` (or reuse the existing `explain` flag more richly) surfaces it.
  - Logician (Stage 3.1) gains a `memory_verify` variant that can check "was this claim actually written to memory" (query the store directly) in addition to filesystem checks.
  - Curator-style maintenance report (`ai-memory-consolidate/src/curator.rs` pattern): a scheduled, rule-based (no-LLM) report of stale sidecars, orphaned tombstones, and pages with no incoming links — written to `memory/_lint/report.md`.
- **Acceptance criteria**:
  - [ ] Every `memory_query` call's trajectory is inspectable after the fact.
  - [ ] Shield blocking a write is provably the *same* code path the atomic writer would have used — no way to bypass Shield by calling a different internal function.
  - [ ] The curator report runs weekly via a `Kip\Curator` scheduled task and flags at least one real issue in a memory tree that's had a few weeks of use (or reports clean, which is also a valid result).

---

#### Phase 9 — WhatsApp bridge integration & hardening

- **Goal**: Point Stage 4's WhatsApp bridge at the new `kip-memory-server`'s MCP surface (Phases 1–8) instead of the ad hoc Python MCP script from the original Stage 2.4, and do a final pass matching Stage 5's polish goals against the new architecture.
- **Borrows from**: nothing new — this phase is convergence, wiring Stage 4/5's already-written code (§5) onto the rebuilt engine.
- **Build on top of**: all previous phases.
- **Deliverables**:
  - Update `opencode.jsonc`'s `mcp.servers` entry to point at `kip-memory-server` instead of `memory-mcp-server.py`.
  - Re-run Stage 4/5's acceptance criteria (§5, Stages 4 and 5) against the new engine end-to-end.
  - A migration note: what happens to the *existing* `memory/`, `MEMORY.md`, ChromaDB data accumulated under the old Stage-1/2 setup — write a one-time import script (`src/migrate/importLegacy.ts`) rather than starting Kip's memory from zero.
- **Acceptance criteria**: all of §10's "Full System" acceptance criteria, re-verified, plus:
  - [ ] Pre-migration memory content (existing `memory/*.md`, `MEMORY.md`) is present and queryable after `importLegacy.ts` runs — nothing gets silently dropped.
  - [ ] Full system survives a reboot with the new `kip-memory-server` as its own `Kip\MemoryServer` scheduled task (at-startup trigger, restart-on-failure). Since Task Scheduler cannot express `After=`, the server must tolerate starting before or after `Kip\OpenCodeServer` — see the ordering note in §5 Step 4.2.

---

### 13.4 Sequencing notes

- Phases 1→5 are strictly sequential (each depends on the previous). Phases 6, 7, and 8 can be reordered or parallelized across two AI sessions once Phase 5 is done, since they touch mostly-disjoint files (decay vs. session-lifecycle vs. deterministic-layer). Phase 9 must be last.
- If token budget or time is tight, **Phases 1, 2, 4, 5 are the minimum viable upgrade** — durable storage, capture, compile-step consolidation, and real hybrid retrieval. Phases 3 (tiered `memory://` reads), 6 (decay), 7 (formal handoff), 8 (observability) are genuine quality-of-life upgrades but Kip functions without them (falling back to Stage 2/3's simpler originals).
- Each phase's "Acceptance criteria" checklist is also the right unit for a CHANGELOG-style commit message and a natural stopping point to hand off to a *different* AI session or a human reviewer — mirroring `ai-memory`'s own "work milestone-by-milestone, don't start the next until every 'Done when' bullet passes" discipline.

---

---

## 14. Parallel Track — Contributing to OpenCode

> Added 2026-08-20. Kip runs *on* OpenCode — building it means spending a lot of time inside OpenCode's plugin API, MCP integration, and agent config surface. That's a natural position from which to contribute back, not a separate project competing for time. Source: `APPUNDERSTANDING.md` in this directory (full architecture/contribution reference) — this section is the actionable subset, oriented around what you're likely to hit *while* building Kip.

### 14.1 Why this pairs naturally with the Kip build

Every piece of friction you hit wiring `ai-memory` and the homelab skill into OpenCode (Phases J1–J6, §12) is potential contribution material: a docs gap you had to work around, a plugin-API rough edge, a missing example for "how to wire an external memory MCP server into OpenCode." You don't need to go looking for a separate project to contribute to — write down friction as you hit it, and periodically triage that list against the process below.

### 14.2 How contribution works (from `APPUNDERSTANDING.md` §3, §7.3)

You already have the clone — `<KIP_ROOT>` **is** `anomalyco/opencode`. So the setup step is just:

```powershell
Set-Location E:\AgentMemoryProject\opencode
bun install
```

⚠️ **Before opening any PR from this clone**, confirm Kip's files are excluded (§7) and that your `.opencode/opencode.jsonc` changes aren't accidentally staged. The cleanest habit is a dedicated worktree for contribution branches, so upstream work never sees Kip's runtime state at all:

```powershell
git worktree add ..\opencode-pr dev
```

- Branch off **`dev`** (not `main`), branch name ≤3 words, hyphen-separated, no `feat/`/`fix/` prefix (e.g. `mcp-timeout-fix`).
- Conventional commit PR titles: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`. Per the repo's `CONTRIBUTING.md` the package **scope is optional** — `docs: update contributing guidelines` is a valid title. (Earlier revisions of this section said scope was required and listed seven scopes; the file itself only exemplifies `app`, `desktop`, `opencode`.)
- Style rules from `AGENTS.md` / `CONTRIBUTING.md`: no `any`, no star imports, early returns over `else`, `Bun.file()` not Node's `fs`. On error handling, `CONTRIBUTING.md`'s wording is "prefer `.catch(...)` instead of `try`/`catch` **when possible**" — softer than the absolute ban earlier revisions of this doc claimed. Read both files yourself before your first PR rather than trusting this summary.
- **`CONTRIBUTING.md` explicitly rejects long AI-generated PR descriptions.** Write short ones in your own words. Given how much of this plan is AI-assisted, that's the easiest way to get a first PR ignored.
- **Test and type-check from the specific package directory, never the monorepo root**: `cd packages/<pkg> && bun test && bun typecheck`.
- Open the PR against `dev`.

### 14.3 Concrete entry points, prioritized by what you'll hit building Kip

1. **Document the "external memory MCP server" integration pattern.** OpenCode's own docs don't currently cover "here's how to wire in a persistent, cross-session memory backend" as a first-class pattern — you'll have worked this out in full by the end of Phase J1/J2. Writing it up (a docs page, or a reference example under `packages/plugin/` or the docs site) is a genuinely useful contribution that didn't exist before, sourced directly from real friction, not a guess.
2. **MCP ecosystem gaps** flagged in `APPUNDERSTANDING.md` §9: *"No MCP server marketplace/discovery UI inside the tool," "No built-in MCP server hosting (only client)."* You'll be running at least two MCP servers (`ai-memory`, your homelab skill) day-to-day — you're well positioned to notice and report (or fix) friction here.
3. **Plugin API ergonomics** — as you build the homelab skill (Phase J6) against `@opencode-ai/plugin` (`packages/plugin/src/index.ts`), any rough edge you hit is a concrete, well-reproduced bug report or small PR, not a vague complaint.
4. **"No persistent agent memory across sessions"** and **"no session search/semantic memory"** — both listed as gaps in §9 of the understanding doc. You will, by construction, have solved this for yourself. Resist the urge to try to upstream the whole `ai-memory` integration as a core feature (that's a large, opinionated architectural change); a smaller, welcome contribution is the documented pattern from item 1 above, or a minimal reference plugin.
5. **Smaller good-first-PRs**, useful for learning the review/CI loop before attempting anything larger: a docs correction, a new LLM provider adapter under `packages/llm/src/providers/` (if you end up using a provider not yet listed), a small bug fix in an existing tool under `packages/opencode/src/tool/`.

### 14.4 Suggested sequencing

Don't open your first OpenCode PR as something large. Start with one small, low-risk contribution (a docs fix, a small bug) specifically to learn their CI/review loop (`turbo dev`, per-package `bun test`/`bun typecheck`, conventional-commit scope rules) — the same "learn by doing, on real infrastructure" approach as §12's Kip plan. Once that lands, the memory-integration docs contribution (§14.3 item 1) is the natural first substantial one, since by then you'll have lived it end to end.

---

---

## 15. Verifying Progress with the OpenCode CLI (and, later, a Local LLM)

> Added 2026-08-20, revised same day. **Start simple**: this was the actual reason to have OpenCode in the first place — a CLI to test how the agent behaves, on demand, no separate harness needed. `opencode run "<prompt>"` (or `opencode run --agent kip "<prompt>"` once the agent's named) *is* the verification tool for almost every phase in this document, and every acceptance-criteria checklist above already assumes you're using it this way. Read the reply yourself. That covers the large majority of "did this phase work" questions.
>
> The rest of this section (§15.2 onward) is an **optional automation layer** for later — once manually reading CLI output after every change starts feeling repetitive, or once you want a written record instead of relying on memory. It is not a replacement for §15.1, and it's a separate mechanism from `ai-memory`'s own `[auto_improve.eval]` gate (which is intentionally deterministic and forbids calling an LLM — see `docs/auto-improve-eval-gates.md` — because it guards what gets auto-written to *Kip's memory*, a different job from judging whether *you* finished a build phase).

### 15.1 Start here: `opencode run` as the test CLI

Concrete examples, one per phase from §12, using the CLI directly — no new tooling required, works from day one:

```powershell
# After Phase J1 (ai-memory wired to OpenCode)
opencode run "What do you remember about our last session?"

# After Phase J2 (Kip persona in _global scope)
opencode run "Who are you?"
opencode run "What's my preferred coding style?"

# After Phase J3 (cross-repo isolation) — run from two different repos, compare answers
Set-Location E:\projects\repo-a; opencode run "What did we decide about the API design here?"
Set-Location E:\projects\repo-b; opencode run "What did we decide about the API design here?"   # must NOT know repo-a's answer

# After Phase J4 (self-improvement, review-gated)
opencode run "List the pending memory auto-improve proposals and what they'd change"
# or, direct: ai-memory pending-writes

# After Phase J5 (WhatsApp bridge) — sanity-check the underlying session before trusting the phone
opencode run --session-id kip-whatsapp-main "test message, ignore"

# After Phase J6 (homelab skill)
opencode run "Restart the Plex container and tell me what you did and why"
```

This is the same pattern the checklists already use throughout this doc (e.g. §5 Stage 1's `opencode run --agent kip "Who are you?"`) — §15.1 just names it explicitly as *the* default verification method, not an example buried in a checklist.

### 15.2 When manual reading gets tedious: automate the judgment call (optional, later)

Once you're running dozens of these a day and re-reading replies by eye stops scaling, the pattern below adds a written, repeatable check on top of the same `opencode run` output — it doesn't replace §15.1, it just stops making you the only judge every single time.

### 15.3 Split every acceptance criterion into two kinds first

Not everything belongs in front of an LLM judge — most of what's actually wrong with "just ask an LLM to check it" is using it where a script would be both cheaper and more reliable.

| Kind | Example from this doc | How to verify |
|---|---|---|
| **Objective / scriptable** | "`bun test` passes," "`memory_query` returns the page," "`Get-ScheduledTask` reports Running," "a tombstoned page is still recoverable via `git show`" | A real command, exit code, or file check — this is exactly what the `logician_verify` tool (§5 Stage 3.1, extended in §12 Phase J's design) is for. **Never** route these through an LLM judge; it's slower, costs tokens, and is strictly less reliable than the real check. |
| **Qualitative / fuzzy** | "the reply sounds like Kip's persona," "the retrieval `explain` output's reasoning actually makes sense for this query," "the daily log's `[ATOM]` extraction captured the *right* decision, not just *a* decision" | This is where a local LLM judge earns its keep — these genuinely can't be scripted with a regex or exit code. |

The local Ollama judge below is for the second column only.

### 15.4 Set up Ollama as the judge

`ai-memory` already treats Ollama as a first-class provider — it's just a keyless OpenAI-compatible endpoint (`ai-memory-llm/src/openai_compat.rs`, per [`ai-memory-overview.md`](ai-memory-overview.md) §6). Smoke-test connectivity with the CLI's own built-in tool before writing anything custom:

> **You already have most of this.** This repo's `.opencode/opencode.jsonc` defines an Ollama provider at `http://localhost:11434/v1` with `deepseek-r1:32b`, `qwen3-coder:30b`, `mixtral:latest`, `mistral-small3.1:latest`, `gemma3:4b`, and others. Check what's actually pulled (`ollama list`) before downloading a new judge model — one of these may already serve.

```powershell
ollama list                       # what's already available locally
ollama pull qwen2.5:7b-instruct   # only if you want a dedicated small judge

ai-memory llm-test `
  --provider openai-compat `
  --base-url http://localhost:11434/v1 `
  --model qwen2.5:7b-instruct `
  --prompt "Reply with exactly: ok"
```

If that returns `ok`, the provider path works end to end — the same command shape (`--provider openai-compat --base-url <ollama-url>`) is what you'd also use if you decide to run `ai-memory`'s *production* consolidation/embedding against local Ollama instead of a cloud provider (relevant to Open Question #6 in §11 — offline fallback).

**Model choice for a judge** — size it to whatever djdesktop actually has spare while OpenCode and the bridge are also running: the judge doesn't need to be big — most of these checks are closer to "does this paragraph match this rubric" than open-ended reasoning.

| Tier | Model | Notes |
|---|---|---|
| Tight (≤8GB RAM free) | `qwen2.5:3b-instruct` | Fast, good enough for pass/fail-with-reason on short rubrics |
| Default | `qwen2.5:7b-instruct` or `llama3.1:8b-instruct` | Good balance of judgment quality and speed on CPU-only inference |
| If RAM allows (≥32GB) | `qwen2.5:14b-instruct` | Meaningfully better reasoning on the fuzzier criteria, slower |

### 15.5 The verifier script

A small tool, `verify-phase`, that turns a phase's checklist into a judged report. Keep it language-agnostic in design (build it in whatever you're already writing that phase's code in — Bash for early phases, or fold it into the Rust/TS work once you're past Phase J1):

1. **Extract** the target phase's `- [ ]` checklist lines directly from this document (grep the phase's heading block — the checklists are the literal source of truth, so the verifier never drifts from what this doc says "done" means).
2. **Gather evidence** — whatever's relevant: recent `memory_query`/`memory_explain` output, the last N captured observations, a pasted command output, a session transcript excerpt. For objective criteria, this evidence *is* the verdict (§15.1) — skip the LLM entirely for those and just record pass/fail.
3. **Judge the qualitative criteria** with a structured prompt (mirroring `ai-memory`'s own eval-gate request/response JSON-contract style, for consistency with a pattern you'll already know from §12 Phase J4):

   **Request** (stdin to the judge call):
   ```json
   {
     "phase": "J2",
     "criterion": "Asking 'who are you?' in any repo returns Kip's persona with zero per-repo config",
     "evidence": "Transcript excerpt: [agent's actual reply text]"
   }
   ```

   **Response** (required from the judge):
   ```json
   { "passed": true, "confidence": 0.9, "reason": "Reply references the persona traits from the SOUL page verbatim; no generic-assistant fallback language present." }
   ```

4. **Write the verdict back** — once `ai-memory` is running (post Phase J1), literally `memory_write_page` the verification report under a `verification/` prefix instead of a throwaway file. This closes a nice loop: Kip ends up with a durable memory of its own build history, and a future `memory_query` can answer "when did Phase J4 get verified, and what was uncertain about it?"

### 15.6 Two-model discipline — don't let the judge grade its own homework

If your *production* LLM (consolidation, chat replies) is also Ollama-hosted, use a **different model** as the judge, or at minimum a judge call with zero shared conversation history. The risk isn't malice, it's correlated blind spots: the same model is disproportionately likely to rate its own mistake as fine. A cheap, concrete rule: production model is whatever you configured in `ai-memory`'s `[embedding]`/chat provider config (cloud or local, your call); the judge is *always* a separate local Ollama call, specifically because it needs to stay cheap and fast enough to run after every single phase without a second thought about API cost.

### 15.7 A note on trusting the judge itself

Treat judge verdicts as a second opinion, not a ground truth — especially early on, while you're still calibrating what "confidence: 0.9" from a 7B model is actually worth in practice. For the first few phases, read the judge's `reason` field yourself every time, even on a `passed: true`, until you've built a feel for when it's being lazy (rubber-stamping plausible-looking output) versus genuinely checking. This is the same "learn by running it" discipline as §12 generally — you're calibrating a tool, not outsourcing judgment to it on day one.

---

*Document created: 2026-08-19. Authors: Debajyoti Bhattacharjee + Claude Sonnet 4.6.*
*Section 12 (adopt-ai-memory Kip plan) and Section 13 (deferred fork-and-customize plan) added/restructured 2026-08-20, synthesizing ideas from `ai-memory` (akitaonrails/ai-memory) and `OpenViking` (volcengine/OpenViking) — see the companion overview files, which **do not exist in this directory yet** (§0). Section 14 (OpenCode contribution track) and Section 15 (CLI/local-LLM phase verification) added the same day.*
*Section 0 added 2026-08-20: the document's host, paths, scheduler, and shell were rewritten from the inherited Debian/`/home/clawd/` premise to the real environment — `E:\AgentMemoryProject\opencode` on `djdesktop` (Windows, Asia/Kolkata), Windows Task Scheduler, PowerShell. §1's scope and §5's acceptance criteria were corrected to match §12's stated target. The superseded Timor-Leste framing survives in `FUTURE.md`.*
*This document is the canonical implementation plan. Update it as stages complete or decisions are made.*
