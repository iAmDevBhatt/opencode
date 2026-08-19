# FUTURE.md — Patrick Memory System on OpenCode
## Implementation Roadmap for Humans and AI Agents

> **Context:** This document describes how to port the "Patrick" layered memory system (originally running on a Debian Beelink SER5 mini-PC in Timor-Leste, powering a WhatsApp-connected AI agent) onto the OpenCode platform. It is written for any developer or AI agent who picks this up with zero prior context.
>
> **Overall verdict:** Fully viable. OpenCode covers ~65% natively, ~25% via custom plugins/MCP wrappers, ~10% requires new builds. All gaps are fillable. Estimated total effort: 3–4 weeks for one developer working part-time.

---

## Table of Contents

1. [What We Are Building](#1-what-we-are-building)
2. [Patrick Memory Architecture — Quick Reference](#2-patrick-memory-architecture--quick-reference)
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

---

## 1. What We Are Building

A deployment of OpenCode as the runtime for **Patrick** — a persistent, layered-memory AI agent running on a Beelink SER5 mini-PC in Aileu, Timor-Leste (960m elevation, UTC+9). Patrick serves one human over WhatsApp across four domains: TOC (Theory of Change) research, creative writing, business operations, and health tracking.

**Patrick's design philosophy:**
> Memory is not about hoarding everything in the AI. It is about sparing humans from having to repeat themselves.

The layered system (L0–L10) provides progressive disclosure: the agent keeps high-level structure in context and drills down to raw evidence on demand. Every layer is human-readable Markdown or JSONL — no opaque vector piles, full traceability.

**What changes with this implementation:**
- Patrick's runtime moves from its current ad-hoc setup to OpenCode as the execution engine
- All existing scripts, Markdown files, and ChromaDB/NetworkX infra are preserved
- OpenCode's plugin system, MCP protocol, and skill system wire them together
- WhatsApp becomes the input channel via a thin Bun/Node wrapper calling the OpenCode HTTP API

---

## 2. Patrick Memory Architecture — Quick Reference

| Layer | Name | Format | Location | Size |
|---|---|---|---|---|
| L0 | Session transcripts | JSONL | `~/.local/share/opencode/transcripts/` | Unbounded (firehose) |
| L1 | Daily memory logs | Markdown + `[ATOM]` | `memory/YYYY-MM-DD.md` | ~123 files, growing |
| L2 | Curated long-term memory | Markdown | `MEMORY.md` | <120 lines enforced |
| L3 | Identity & persona | Markdown | `SOUL.md`, `USER.md`, `AGENTS.md`, `IDENTITY.md` | ~35KB total |
| L4 | Deterministic enforcement | Bash scripts | `/home/clawd/*.sh` | 3 scripts |
| L5 | Vector memory | ChromaDB + NetworkX | `/home/clawd/memory/` | ~200MB |
| L6 | Memory wiki | Markdown synthesis | `memory/wiki/` | Light |
| L7 | Obsidian vault | Markdown | `/home/Obsidian/` | Gigabytes |
| L8 | Session state + heartbeat | Markdown | `SESSION-STATE.md`, `HEARTBEAT.md` | Small |
| L9 | Memory promotion pipeline | Bash + cron | `memory-promote.sh` | 1 script |
| L10 | Procedural memory (skills) | Markdown | `.opencode/skills/`, `TOOLS.md` | Per-skill |

**[ATOM] format** (L1 structured extraction):
```
[ATOM] type=decision|event|lesson|preference|fact | entity=<subject> | detail=<what> | ref=<source>
```

---

## 3. Feasibility Analysis by Layer

### L0 — Session Transcripts

**OpenCode native state:** Partial. All session messages are stored in SQLite via Drizzle ORM (`packages/core/src/database/`). The `SessionStore` exposes `context()` and `runnerContext()`. Messages are sequenced and typed. The `opencode export` command exists but is not a live-append JSONL firehose.

**Gap:** Rows in SQLite, not raw JSONL files you can tail or grep in real time.

**Implementation:** Write a plugin hook on session message events that appends each message to `~/.local/share/opencode/transcripts/YYYY-MM-DD.jsonl`. Approximately 30 lines using the OpenCode plugin hook system.

---

### L1 — Daily Memory Logs

**OpenCode native state:** Not native. OpenCode has no "write a session summary after every session" primitive. However, the plugin system has hooks that fire on session events.

OpenCode's internal compaction system already does something structurally identical — `packages/core/src/session/compaction.ts` generates a `SUMMARY_TEMPLATE` with sections: Objective / Important Details / Work State (Completed/Active/Blocked) / Next Move / Relevant Files. This is stored as a session message internally.

**Implementation:** Plugin that:
1. Listens for session-end event (or idle + heartbeat detection)
2. Intercepts or re-runs the compaction summary LLM call
3. Reformats output into Patrick's L1 format (including `[ATOM]` extraction)
4. Appends to `memory/$(date +%Y-%m-%d).md`

The compaction template in `compaction.ts` is directly adaptable — Patrick's L1 format is a superset of the existing `SUMMARY_TEMPLATE`.

---

### L2 — MEMORY.md

**OpenCode native state:** Yes — this is exactly what OpenCode's own auto-memory system uses. The pattern: a `MEMORY.md` index + individual `memory/*.md` files, enforced at <200 lines. The agent reads it at session start.

**Gap:** Patrick enforces <120 lines (stricter than OpenCode's <200). Patrick's promotion logic scores atoms before promoting.

**Implementation:** Adapt the existing `MEMORY.md` format to Patrick's 120-line limit. The L9 promotion pipeline (a scheduled `opencode run` call) handles pruning.

---

### L3 — Identity & Persona Files

**OpenCode native state:** Fully supported via Skills + Config.

Direct equivalents in OpenCode:
- `AGENTS.md` at repo root → OpenCode reads as agent constitution
- `.opencode/opencode.jsonc` with `agents.<name>.prompt` → per-agent system prompt
- Skills (`SKILL.md` files) → inject structured knowledge into context
- The built-in `customize-opencode` skill demonstrates the exact pattern

**Implementation:** Map each identity file to a skill:

| Patrick file | OpenCode location |
|---|---|
| `SOUL.md` | `.opencode/skills/patrick-soul/SKILL.md` |
| `USER.md` | `.opencode/skills/patrick-user/SKILL.md` |
| `IDENTITY.md` | `.opencode/skills/patrick-identity/SKILL.md` |
| `AGENTS.md` | Agent prompt in `opencode.jsonc` + `.opencode/agent/patrick.md` |

Effort: Near-zero. Copy existing files into the skill directory structure.

---

### L4 — Deterministic Layer (Logician / Shield / Guardian)

**OpenCode native state:** Not native. OpenCode has no binary-verification, path-validation, or auto-healing primitives.

**Implementation per component:**

**Logician** (binary verification before claiming task complete):
- Implement as a custom tool in an OpenCode plugin
- Calls `logician-verify.sh` via `Bun.spawnSync`
- Agent can call `logician_verify({ check_type: "file_exists", target: "/path" })` and get `{ verified: true/false }`

**Shield** (path validation — blocks writes outside safe zones):
- Implement as a plugin hook on `tool.before`
- Intercepts every `write` and `bash` tool call
- Validates target path against allowlist: `["/home/clawd/vault", "/home/clawd/workspace", "/tmp"]`
- Returns a blocking error if path is outside safe zones

**Guardian** (auto-healing heartbeat):
- Stays as a systemd timer on the Beelink
- Calls `opencode run --agent patrick "run guardian health check"` at the 30-minute interval
- Guardian script checks: gateway connectivity, disk space, vector sync status, dispatch generation

---

### L5 — Vector Memory (ChromaDB + NetworkX)

**OpenCode native state:** Not present. Zero vector/embedding infrastructure in the entire OpenCode codebase. No ChromaDB, no sqlite-vec, no embedding calls.

**Implementation:** Wrap existing Python scripts as an MCP server. The MCP stdio protocol is ~100 lines of Python boilerplate. Add to `opencode.jsonc`:

```jsonc
{
  "mcp": {
    "servers": {
      "patrick-memory": {
        "type": "local",
        "command": ["python3", "/home/clawd/memory-mcp-server.py"]
      }
    }
  }
}
```

The `memory_search` natural language queries become MCP tool calls. OpenCode exposes them to Patrick natively. No changes to existing Python scripts beyond the MCP wrapper.

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
        "command": ["npx", "-y", "mcp-obsidian", "/home/Obsidian"]
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
- 3x daily heartbeat (8am/2pm/6pm UTC+9) → systemd timers on the Beelink invoking `opencode run --agent patrick "run morning/afternoon/evening ritual"`
- `HEARTBEAT.md` updated by the agent during the ritual run

---

### L9 — Memory Promotion Pipeline

**OpenCode native state:** Not native, but trivially implementable as a scheduled `opencode run` call.

**Implementation:**

```bash
# /etc/cron.d/patrick-promote
0 14 * * * clawd opencode run --agent patrick \
  "Run memory promotion ritual: read today's L1 log at memory/$(date +\%Y-\%m-\%d).md, \
   score each [ATOM] entry (decision=3, lesson=2, preference=2, event=1, fact=1), \
   promote items scoring >=2 to MEMORY.md, prune MEMORY.md if >180 lines keeping highest scores."
```

The agent already has all required tools (read, edit, write). No plugin needed.

---

### L10 — Procedural Memory (Skills)

**OpenCode native state:** Directly supported. OpenCode's skill system is the native equivalent of L10.

**Implementation:**
- Copy `TOOLS.md` content → `.opencode/skills/patrick-tools/SKILL.md`
- Each existing skill → `.opencode/skills/<name>/SKILL.md`
- Session-end checklist (skill detection) → plugin hook that asks at session end: "Was a procedure repeated 2+ times? Was a non-obvious fix used? If yes, suggest skill creation."

---

## 4. Master Summary Table

| Patrick Layer | What it is | OpenCode Native? | Implementation Path | Effort | Priority |
|---|---|---|---|---|---|
| L0 Session transcripts | Raw JSONL dialogue | Partial (SQLite) | Plugin hook → append to JSONL | Low | P2 |
| L1 Daily memory logs | Structured session notes + `[ATOM]` | No | Plugin hook + compaction intercept + file write | Medium | P1 |
| L2 MEMORY.md | Curated long-term memory <120 lines | Yes | Adapt format/line limit | Low | P1 |
| L3 SOUL/USER/IDENTITY | Identity & persona files | Yes (skills + agent prompt) | Copy files into `.opencode/skills/` dirs | Near-zero | P1 |
| L4 Logician/Shield/Guardian | Deterministic enforcement | No | Custom tools + plugin hook + systemd | Medium | P2 |
| L5 ChromaDB + NetworkX | Vector + graph memory | No | Wrap existing Python as MCP server | Medium | P2 |
| L6 Memory Wiki | Synthesis pages with provenance | No | Custom plugin tool `wiki_synthesize` | Medium | P3 |
| L7 Obsidian vault | Human-curated knowledge base | Yes (via MCP) | Add MCP server config (1 line) | Low | P1 |
| L8 SESSION-STATE + heartbeat | Session state + 3x daily cron | Partial | Plugin hook + systemd timers | Low | P1 |
| L9 Promotion pipeline | Scored daily→MEMORY.md promotion | No (trivial) | One crontab entry + agent prompt | Near-zero | P1 |
| L10 Skills + TOOLS.md | Procedural memory | Yes (native skills) | Copy files into skill dirs | Near-zero | P1 |

**P1 = Week 1 | P2 = Week 2 | P3 = Week 3**

---

## 5. Implementation Stages

---

### Stage 1 — Zero-Friction Wins (Days 1–3)

**Goal:** Get Patrick's identity and procedural knowledge into OpenCode with no new code. Patrick should be able to respond "as Patrick" with full persona and context from Day 1.

---

#### Step 1.1 — Install OpenCode on Beelink SER5

```bash
# On the Debian Beelink
curl -fsSL https://opencode.ai/install | bash
# Or via npm:
npm install -g opencode@latest

# Verify
opencode --version
```

---

#### Step 1.2 — Create Patrick Agent Definition

Create `.opencode/agent/patrick.md` in the project directory (or `~/.config/opencode/agent/patrick.md` for global):

```markdown
---
name: patrick
description: Patrick — personal AI agent for TOC research, creative writing, business ops, health tracking
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

You are Patrick. Read your identity from the patrick-soul, patrick-user, and patrick-identity skills before responding.
Always begin sessions by loading SESSION-STATE.md to understand current project phase.
Follow the memory protocol defined in patrick-agents skill.
```

---

#### Step 1.3 — Migrate Identity Files to Skills

```bash
mkdir -p .opencode/skills/patrick-soul
mkdir -p .opencode/skills/patrick-user
mkdir -p .opencode/skills/patrick-identity
mkdir -p .opencode/skills/patrick-agents
mkdir -p .opencode/skills/patrick-tools

# Copy existing files
cp /home/clawd/SOUL.md .opencode/skills/patrick-soul/SKILL.md
cp /home/clawd/USER.md .opencode/skills/patrick-user/SKILL.md
cp /home/clawd/IDENTITY.md .opencode/skills/patrick-identity/SKILL.md
cp /home/clawd/AGENTS.md .opencode/skills/patrick-agents/SKILL.md
cp /home/clawd/TOOLS.md .opencode/skills/patrick-tools/SKILL.md
```

Each file already contains the right content — OpenCode discovers `SKILL.md` files automatically. No reformatting needed.

---

#### Step 1.4 — Configure Obsidian MCP Server

Edit `~/.config/opencode/opencode.jsonc` (or project-level `.opencode/opencode.jsonc`):

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-5",
  "default_agent": "patrick",
  "mcp": {
    "servers": {
      "obsidian": {
        "type": "local",
        "command": ["npx", "-y", "mcp-obsidian", "/home/Obsidian"],
        "timeout": 30000
      }
    }
  }
}
```

Test: `opencode mcp list` — should show `obsidian` as connected.

---

#### Step 1.5 — Set Up Systemd Heartbeat Timers

Create three systemd timer units on the Beelink:

```ini
# /etc/systemd/system/patrick-morning.service
[Unit]
Description=Patrick Morning Ritual

[Service]
Type=oneshot
User=clawd
WorkingDirectory=/home/clawd/workspace
ExecStart=opencode run --agent patrick "Run morning ritual: re-read SOUL.md, check weather, review SESSION-STATE.md, check dispatch, verify context <120 lines. Log to HEARTBEAT.md."
```

```ini
# /etc/systemd/system/patrick-morning.timer
[Unit]
Description=Patrick Morning Ritual Timer

[Timer]
OnCalendar=*-*-* 08:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

Repeat for `patrick-afternoon.timer` (14:00) and `patrick-evening.timer` (18:00). Enable:

```bash
sudo systemctl enable --now patrick-morning.timer
sudo systemctl enable --now patrick-afternoon.timer
sudo systemctl enable --now patrick-evening.timer
```

---

#### Step 1.6 — Set Up Memory Promotion Cron

```bash
# Add to crontab: crontab -e
0 14 * * * cd /home/clawd/workspace && opencode run --agent patrick "Memory promotion ritual: Read today's L1 log at memory/$(date +\%Y-\%m-\%d).md. Score each [ATOM] entry: decision=3, lesson=2, preference=2, event=1, fact=1. Promote all items scoring 2 or above to MEMORY.md. If MEMORY.md exceeds 180 lines, prune to 120 lines keeping highest scores. Write updated MEMORY.md."
```

---

#### Stage 1 Acceptance Criteria

- [ ] `opencode run --agent patrick "Who are you?"` returns a Patrick-persona response referencing soul/identity files
- [ ] `opencode mcp list` shows `obsidian` connected
- [ ] `opencode skill list` shows all five patrick-* skills
- [ ] All three systemd timers are active: `systemctl list-timers | grep patrick`
- [ ] Promotion cron entry visible: `crontab -l | grep promote`

---

### Stage 2 — Memory Core (Days 4–10)

**Goal:** Implement L0 (JSONL transcripts), L1 (daily memory logs with `[ATOM]` extraction), and L5 (ChromaDB/NetworkX as MCP server).

---

#### Step 2.1 — Create the OpenCode Plugin Package

```bash
mkdir -p /home/clawd/patrick-plugin
cd /home/clawd/patrick-plugin
bun init -y
bun add @opencode-ai/plugin
```

Create `src/index.ts`:

```typescript
import { tool } from "@opencode-ai/plugin"

// All Patrick custom tools and hooks will be registered here
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

const TRANSCRIPT_DIR = "/home/clawd/memory/transcripts"

export const transcriptHook = {
  // Fires after every message is written
  "session.message.after": async (message: any) => {
    const date = new Date().toISOString().slice(0, 10)  // YYYY-MM-DD
    const filePath = join(TRANSCRIPT_DIR, `${date}.jsonl`)
    await appendFile(filePath, JSON.stringify(message) + "\n", "utf8")
  }
}
```

Register in plugin index. This gives Patrick a live-append JSONL firehose identical to L0.

---

#### Step 2.3 — L1: Daily Memory Log Writer

Add to `src/hooks/daily-log.ts`:

```typescript
import { appendFile, readFile } from "fs/promises"
import { join } from "path"

const MEMORY_DIR = "/home/clawd/memory"

// The SUMMARY_TEMPLATE adapted for Patrick's L1 format
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

Create `/home/clawd/memory-mcp-server.py`:

```python
#!/usr/bin/env python3
"""MCP server wrapping Patrick's ChromaDB + NetworkX memory."""

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
            "serverInfo": {"name": "patrick-memory", "version": "1.0.0"}
        }

    if method == "tools/list":
        return {
            "tools": [
                {
                    "name": "memory_search",
                    "description": "Search Patrick's long-term memory using natural language",
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
        ["python3", "/home/clawd/search.py", query, str(limit)],
        capture_output=True, text=True
    )
    return json.loads(result.stdout) if result.returncode == 0 else []


async def lookup_graph(entity: str, depth: int) -> dict:
    """Query NetworkX graph.json."""
    import subprocess
    result = subprocess.run(
        ["python3", "/home/clawd/graph_lookup.py", entity, str(depth)],
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
"patrick-memory": {
  "type": "local",
  "command": ["python3", "/home/clawd/memory-mcp-server.py"],
  "timeout": 15000
}
```

---

#### Step 2.5 — Install and Register the Plugin

```bash
cd /home/clawd/patrick-plugin
bun run build   # or: bun build src/index.ts --outdir dist

# Install into OpenCode
opencode plug /home/clawd/patrick-plugin
```

---

#### Stage 2 Acceptance Criteria

- [ ] After a session, `ls memory/*.jsonl` shows today's transcript file with content
- [ ] After a session, `ls memory/YYYY-MM-DD.md` shows today's daily log with `[ATOM]` entries
- [ ] `opencode run --agent patrick "What do you remember about last Tuesday?"` triggers a ChromaDB search
- [ ] `opencode run --agent patrick "Who is connected to the Aileu water project?"` triggers a graph lookup
- [ ] `opencode mcp list` shows `patrick-memory` as connected

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
      "bash", "/home/clawd/logician-verify.sh", check_type, target,
      ...(expected ? [expected] : [])
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

```typescript
// Safe zones — writes outside these are blocked
const SAFE_ZONES = [
  "/home/clawd/vault",
  "/home/clawd/workspace",
  "/home/clawd/memory",
  "/tmp",
  "/home/clawd/.opencode",
]

function isPathSafe(path: string): boolean {
  return SAFE_ZONES.some(zone => path.startsWith(zone))
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

    // Intercept bash tool — check for rm, mv targeting unsafe paths
    if (toolCall.id === "bash" && toolCall.parameters?.command) {
      const cmd = toolCall.parameters.command as string
      // Basic heuristic: flag destructive commands with absolute paths outside safe zones
      const unsafePattern = /\b(rm|mv|chmod|chown)\b.*\/(?!home\/clawd\/(vault|workspace|memory|\.opencode)|tmp)/
      if (unsafePattern.test(cmd)) {
        // Log the suspicious command but don't block — log for human review
        await appendToLog(`[SHIELD WARNING] Suspicious command: ${cmd}`)
      }
    }
  }
}
```

---

#### Step 3.3 — Guardian Systemd Service

Create `/etc/systemd/system/patrick-guardian.service`:

```ini
[Unit]
Description=Patrick Guardian Auto-Healer

[Service]
Type=oneshot
User=clawd
WorkingDirectory=/home/clawd/workspace
ExecStart=/home/clawd/guardian.sh
# guardian.sh checks: gateway, disk, vector sync, OpenCode server health
# If issues found, calls: opencode run --agent patrick "Guardian alert: {issue}"
```

```ini
[Unit]
Description=Patrick Guardian Timer

[Timer]
OnCalendar=*:0/30     # every 30 minutes
Persistent=true

[Install]
WantedBy=timers.target
```

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

- [ ] `opencode run --agent patrick "Verify file exists: /home/clawd/MEMORY.md"` → `{ verified: true }`
- [ ] `opencode run --agent patrick "Write a test file to /etc/passwd"` → blocked by Shield with error message
- [ ] Guardian timer fires every 30 minutes: `systemctl list-timers | grep guardian`
- [ ] `PATRICK_FAILURE_AUDIT_LOG.md` is written to when Shield fires

---

### Stage 4 — WhatsApp Integration (Days 18–25)

**Goal:** Connect Patrick to WhatsApp so the human can send messages and receive replies without a terminal.

---

#### Step 4.1 — Create the WhatsApp Bridge

This is the solution to the "one real limitation" — OpenCode is stateless between invocations. A thin always-on process watches for incoming WhatsApp messages and calls the OpenCode HTTP API.

Create `/home/clawd/whatsapp-bridge/src/index.ts`:

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
  baseUrl: "http://localhost:4001",  // OpenCode server on Beelink
})

// Persistent session ID for the WhatsApp thread
const WHATSAPP_SESSION_ID = "patrick-whatsapp-main"

const whatsapp = new Client({
  authStrategy: new LocalAuth({ clientId: "patrick" }),
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
        agent: "patrick",
        id: WHATSAPP_SESSION_ID
      })
    }

    // Send the message to Patrick
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
    console.error("Patrick bridge error:", err)
    await msg.reply("Patrick encountered an error. Check the logs.")
  }
})

whatsapp.initialize()
console.log("Patrick WhatsApp bridge started. Scan QR code to authenticate.")
```

---

#### Step 4.2 — Run the Bridge as a Systemd Service

```ini
# /etc/systemd/system/patrick-whatsapp.service
[Unit]
Description=Patrick WhatsApp Bridge
After=network-online.target opencode.service
Wants=network-online.target

[Service]
Type=simple
User=clawd
WorkingDirectory=/home/clawd/whatsapp-bridge
ExecStart=bun run src/index.ts
Restart=always
RestartSec=10
Environment=AUTHORIZED_WHATSAPP_NUMBER=+670XXXXXXXX
Environment=ANTHROPIC_API_KEY=sk-...

[Install]
WantedBy=multi-user.target
```

---

#### Step 4.3 — Run OpenCode Server as a Systemd Service

```ini
# /etc/systemd/system/opencode.service
[Unit]
Description=OpenCode Server (Patrick Runtime)
After=network.target

[Service]
Type=simple
User=clawd
WorkingDirectory=/home/clawd/workspace
ExecStart=opencode serve --port 4001
Restart=always
RestartSec=5
Environment=ANTHROPIC_API_KEY=sk-...

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now opencode.service
sudo systemctl enable --now patrick-whatsapp.service
```

---

#### Stage 4 Acceptance Criteria

- [ ] `systemctl status opencode.service` → active (running)
- [ ] `systemctl status patrick-whatsapp.service` → active (running)
- [ ] Sending "Hello" to the authorized WhatsApp number gets a Patrick-persona response within 30 seconds
- [ ] Sending "What files are in my Obsidian vault?" triggers an Obsidian MCP call and returns results
- [ ] Sending "Search memory for: water project budget" triggers a ChromaDB search

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
    const wikiPath = `/home/clawd/memory/wiki/${slug}.md`
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
      await appendFile("/home/clawd/memory/skill-suggestions.md",
        `${new Date().toISOString().slice(0,10)} — ${check}\n`)
    }
  }
}
```

---

#### Step 5.3 — Health Monitoring

Add a simple health check endpoint check to Guardian:

```bash
# In guardian.sh — add OpenCode server health check
OPENCODE_STATUS=$(curl -sf http://localhost:4001/health || echo "DOWN")
if [ "$OPENCODE_STATUS" = "DOWN" ]; then
  systemctl restart opencode.service
  echo "$(date): OpenCode restarted by Guardian" >> /home/clawd/memory/GUARDIAN_LOG.md
fi
```

---

#### Step 5.4 — Final Configuration Consolidation

Final `~/.config/opencode/opencode.jsonc`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-5",
  "default_agent": "patrick",
  "snapshots": true,
  "compaction": {
    "auto": true,
    "buffer": 20000,
    "keep": { "tokens": 8000 }
  },
  "mcp": {
    "servers": {
      "obsidian": {
        "type": "local",
        "command": ["npx", "-y", "mcp-obsidian", "/home/Obsidian"],
        "timeout": 30000
      },
      "patrick-memory": {
        "type": "local",
        "command": ["python3", "/home/clawd/memory-mcp-server.py"],
        "timeout": 15000
      }
    }
  },
  "plugin": {
    "patrick": "/home/clawd/patrick-plugin"
  }
}
```

---

#### Stage 5 Acceptance Criteria

- [ ] `opencode run --agent patrick "Synthesize a wiki page for: Aileu water project"` creates `memory/wiki/aileu-water-project.md`
- [ ] Skill suggestions file grows over time: `cat memory/skill-suggestions.md`
- [ ] Guardian auto-restarts OpenCode if it goes down: kill it and verify restart within 60s
- [ ] Full system survives a Beelink reboot: all services come back, WhatsApp bridge reconnects

---

## 6. Architecture Diagram

```
                    ┌─────────────────────────────────┐
                    │   Human (WhatsApp, Timor-Leste)  │
                    └──────────────┬──────────────────┘
                                   │ WhatsApp message
                    ┌──────────────▼──────────────────┐
                    │   patrick-whatsapp bridge        │
                    │   (Bun process, systemd)         │
                    └──────────────┬──────────────────┘
                                   │ HTTP POST /session/{id}/prompt
                    ┌──────────────▼──────────────────┐
                    │   OpenCode Server (:4001)        │
                    │   opencode serve (systemd)       │
                    │                                  │
                    │  ┌──────────────────────────┐   │
                    │  │  Patrick Agent           │   │
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
                    │  │  Patrick Plugin          │   │
                    │  │  - logician_verify       │   │
                    │  │  - wiki_synthesize       │   │
                    │  │  - Shield hook           │   │
                    │  │  - L0 transcript hook    │   │
                    │  │  - L1 daily log hook     │   │
                    │  └──────────────────────────┘   │
                    └──────┬────────────┬─────────────┘
                           │            │ MCP protocol
              ┌────────────▼──┐  ┌──────▼──────────────┐
              │ Obsidian MCP  │  │ Patrick Memory MCP   │
              │ /home/Obsidian│  │ ChromaDB + NetworkX  │
              └───────────────┘  └──────────────────────┘

Systemd timers (separate processes, call opencode run):
  08:00 → morning ritual
  14:00 → memory promotion (L9)
  18:00 → evening ritual
  */30m → guardian health check
```

---

## 7. File & Directory Layout After Implementation

```
/home/clawd/
├── workspace/                     # Primary working directory
│   └── .opencode/
│       ├── opencode.jsonc         # Master config
│       ├── agent/
│       │   └── patrick.md         # Patrick agent definition
│       └── skills/
│           ├── patrick-soul/SKILL.md
│           ├── patrick-user/SKILL.md
│           ├── patrick-identity/SKILL.md
│           ├── patrick-agents/SKILL.md
│           └── patrick-tools/SKILL.md
├── memory/
│   ├── 2026-08-19.md              # L1 daily logs (existing ~123 files)
│   ├── transcripts/
│   │   └── 2026-08-19.jsonl       # L0 raw transcripts
│   ├── wiki/                      # L6 synthesis pages
│   ├── refs/                      # L4 symbolic offload refs
│   └── skill-suggestions.md       # L10 auto-detected skill candidates
├── MEMORY.md                      # L2 curated long-term (<120 lines)
├── SESSION-STATE.md               # L8 current project phase
├── HEARTBEAT.md                   # L8 daily ritual log
├── PATRICK_FAILURE_AUDIT_LOG.md   # L4 Shield + Logician failures
├── SOUL.md                        # L3 (source — mirrored to skill)
├── USER.md                        # L3 (source — mirrored to skill)
├── IDENTITY.md                    # L3 (source — mirrored to skill)
├── AGENTS.md                      # L3 (source — mirrored to skill)
├── TOOLS.md                       # L10 (source — mirrored to skill)
│
├── patrick-plugin/                # OpenCode plugin package
│   ├── package.json
│   └── src/
│       ├── index.ts
│       ├── tools/
│       │   ├── logician.ts
│       │   └── wiki-synthesize.ts
│       └── hooks/
│           ├── transcript.ts
│           ├── daily-log.ts
│           ├── shield.ts
│           └── skill-detection.ts
│
├── whatsapp-bridge/               # WhatsApp → OpenCode bridge
│   ├── package.json
│   └── src/index.ts
│
├── memory-mcp-server.py           # L5: MCP wrapper for ChromaDB + NetworkX
├── logician-verify.sh             # L4: Binary verification
├── guardian.sh                    # L4: Auto-healing heartbeat
├── memory-promote.sh              # L9: Memory promotion pipeline
├── graph.json                     # L5: NetworkX knowledge graph (19MB)
├── search.py                      # L5: ChromaDB search
├── indexer.py                     # L5: ChromaDB indexer
└── .local/share/opencode/         # OpenCode SQLite storage
    └── sessions/                  # Session database
```

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
name: patrick
prompt: |
  Load your identity skills before every response:
  1. Load skill: patrick-soul
  2. Load skill: patrick-user
  Always check SESSION-STATE.md at the start of a new conversation.
  Always use logician_verify before claiming a task complete.
---
```

### Pattern 3: Scheduled OpenCode Run

```bash
# Pattern for all heartbeat/promotion crons:
opencode run \
  --agent patrick \
  --session-id "patrick-$(date +%Y%m%d)-ritual" \
  "Your ritual prompt here. Be specific about what files to read/write."
```

Using a date-based `--session-id` prevents each cron from accumulating into one infinite session while still keeping daily history grouped.

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

OpenCode is **stateless between invocations** at the agent-context level. There is no background process that "stays alive as Patrick." Each `opencode run` or HTTP prompt call starts with a fresh context load (though it incorporates prior session history via compaction).

**This means:** Patrick does not "notice" things on its own. It only acts when:
1. A human sends a WhatsApp message (→ whatsapp-bridge calls the API)
2. A systemd timer fires (→ cron calls `opencode run`)
3. A human opens a terminal session

**This is fine for the stated deployment.** Patrick is not a daemon — it is a responsive assistant. The WhatsApp bridge gives the appearance of always-on presence. The heartbeat timers give proactive behavior.

**If you ever need genuinely always-on presence** (Patrick monitoring a file, watching a sensor, or reacting to an external event stream without human input):

Option A — Use the OpenCode HTTP API in a long-running Bun process:
```typescript
// long-running-monitor.ts
const opencode = await createOpencode({ baseUrl: "http://localhost:4001" })
// Watch a file, API, or sensor
const watcher = fs.watch("/home/clawd/inbox/")
for await (const event of watcher) {
  await opencode.sessions.prompt({
    sessionId: "patrick-monitor",
    prompt: `New file detected: ${event.filename}. Process it.`
  })
}
```

Option B — Use the OpenCode event stream (SSE) to react to server-side events:
```typescript
const events = await opencode.events.stream({ sessionId: "patrick-monitor" })
for await (const event of events) {
  if (event.type === "tool.result" && event.tool === "my-sensor-tool") {
    // React to tool results in real time
  }
}
```

---

## 10. Acceptance Criteria

### Full System (all stages complete)

- [ ] Sending a WhatsApp message gets a Patrick-persona response within 30 seconds
- [ ] Patrick responds to "Who are you?" with content from SOUL.md
- [ ] Patrick responds to "What are we working on?" from SESSION-STATE.md context
- [ ] Patrick can search Obsidian vault: "Do I have notes on X?"
- [ ] Patrick can search ChromaDB: "What do you remember about Y?"
- [ ] Patrick writes daily log to `memory/YYYY-MM-DD.md` after each session with `[ATOM]` entries
- [ ] MEMORY.md never exceeds 120 lines (enforced by L9 cron)
- [ ] Writing to `/etc/passwd` is blocked by Shield with an error
- [ ] `logician_verify` returns accurate results for file existence checks
- [ ] Guardian auto-restarts OpenCode if it crashes
- [ ] All systemd services survive a Beelink reboot
- [ ] BP/health readings tracked in daily log when mentioned in conversation
- [ ] HANDOVER sections written in daily log (enabling model switches without context loss)

---

## 11. Open Questions

These require human decisions before or during implementation:

1. **WhatsApp API approach:** `whatsapp-web.js` (free, fragile, needs Chromium) vs Twilio WhatsApp API (paid, reliable) vs Meta WhatsApp Business Cloud API (free tier, requires business verification). Which is acceptable for this deployment?

2. **Embeddings provider:** The current Patrick system uses OpenAI for embeddings (noted as `⚠️ not fully local`). For a fully local option, consider `nomic-embed-text` via Ollama on the Beelink. Is the OpenAI API cost acceptable, or should embeddings be fully local?

3. **Session continuity via WhatsApp:** Should all WhatsApp messages go into one persistent session (`patrick-whatsapp-main`) for maximum context continuity, or should each day be a fresh session (with compaction providing the bridge)? One session accumulates indefinitely; daily sessions require good compaction summaries.

4. **Skill file sync:** SOUL.md etc. currently exist as primary files in `/home/clawd/`. After migration, the canonical versions should be in `.opencode/skills/`. Do we keep the originals as symlinks, or do we accept two copies?

5. **OpenCode version pinning:** The Beelink needs a specific OpenCode version pinned to avoid breaking changes on auto-update. Should `autoupdate` be set to `"notify"` (conservative) or `false` (fully manual)?

6. **Network resilience:** Aileu has intermittent connectivity. What should Patrick do if the Anthropic API is unreachable? Options: (a) queue messages and respond when connectivity returns, (b) fall back to a locally-running Ollama model, (c) respond with a connectivity error immediately.

---

*Document created: 2026-08-19. Authors: Debajyoti Bhattacharjee + Claude Sonnet 4.6.*
*This document is the canonical implementation plan. Update it as stages complete or decisions are made.*
