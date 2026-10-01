# GUIDE_STEPS.md — Kip Build Lab (verified, Step 1.1 → 5.4)

> **What this file is.** `FUTURE_PLANS.md` is the architecture doc — read it for *why* each layer (L0–L10) exists and how they fit together. This file is the **hand-hold lab exercise**: the actual code, checked against the real, installed `@opencode-ai/plugin`/`@opencode-ai/sdk` (v1.18.20) and OpenCode's own config schema — not the pseudocode `FUTURE_PLANS.md` Stage 2–5 snippets were written against.
>
> **Why a second file exists at all.** Going through Stage 2 as written produced code that doesn't compile and hooks that never fire — `session.message.after`, `session.compaction.ended`, `tool.before`, a `{tools, hooks}`-shaped default export, JSON-schema tool parameters — none of these exist in the installed package. They were pseudocode against an imagined API, written before anyone checked it against reality. This file replaces every code block in Stages 1–5 with something that actually runs, and explains *why* the correction was needed so you can spot the same class of mistake yourself later.
>
> **How to use it.** Each step has the same five parts:
> - **Why** — which memory layer (L0–L10) this step builds, one sentence.
> - **Code** — the real, working thing to type or paste.
> - **What this does** — a line-by-line-ish walkthrough, plain language.
> - **✅ Verify** — a command to run and the output that means it worked. This is the lab check — don't move to the next step until this passes.
> - **⚠️ Corrected vs FUTURE_PLANS.md** — only present where the original plan's code was wrong, with the reason.
>
> **Where the verified facts came from**, so you can re-check them yourself any time instead of trusting me blindly:
> - `kip-plugin/node_modules/@opencode-ai/plugin/dist/index.d.ts` and `tool.d.ts` — the real `Hooks`, `Plugin`, `tool()` types.
> - `kip-plugin/node_modules/@opencode-ai/sdk/dist/gen/{types,sdk}.gen.d.ts` — the real SDK client shapes.
> - `packages/opencode/src/plugin/{index,shared}.ts` — how OpenCode actually loads a plugin (entry-point resolution, blocking behavior).
> - `packages/core/src/tool/{write,bash}.ts` — the real built-in tools' argument field names.
> - `packages/core/src/v1/config/{agent,permission,config}.ts` — the real config/agent schema.
> - `packages/web/src/content/docs/{plugins,agents,skills}.mdx` — the official docs, which agree with the source and gave working examples for several patterns below.

---

## Quick-reference: what the plan got wrong

Skim this once before starting — it's the pattern to watch for, not just a list of fixes.

| FUTURE_PLANS.md said | Reality | Where it bites |
|---|---|---|
| `export default { tools: [], hooks: {} }` | Plugin is an `async function(input) => Hooks`; the returned object *is* the hooks, no wrapper | Step 2.1 |
| Hook `"session.message.after"` | Doesn't exist. Real: `event` hook, filter `event.type === "message.updated"` | Step 2.2 |
| Hook `"session.compaction.ended"` | Doesn't exist. Real: `event` hook, filter `event.type === "session.idle"` (or `"session.compacted"`) | Step 2.3 |
| Hook `"tool.before"`, `toolCall.parameters` | Real: `"tool.execute.before"`, input `{tool, sessionID, callID}`, output `{args}` (mutable) | Step 3.2 |
| `tool({ id, parameters: {type:"object",...} })` (JSON Schema) | Real: `tool({ description, args: <Zod raw shape via `tool.schema`>, execute(args, context) })` | Steps 3.1, 5.1 |
| Write tool arg `file_path` | Real built-in `write` tool's field is **`path`** | Step 3.2 |
| Agent frontmatter `permission:` as a YAML **list** (`- allow: write`) | Real: an **object** keyed by tool name (`edit`, `bash`, ...) → `"allow"/"deny"/"ask"`. There is no `write` key — file writes are governed by `edit` | Step 1.2 |
| `.opencode/agent/kip.md` with no `---` frontmatter fences | Frontmatter **requires** opening and closing `---` lines, or none of it is parsed at all | Step 1.2 |
| `SKILL.md` with no frontmatter | Requires `---\nname: <dir-name>\ndescription: ...\n---` or the skill isn't discoverable | Step 1.3 |
| `opencode.jsonc` → `"plugin": { "kip": "path" }` | Real: `"plugin"` is an **array** of strings (or `[name, options]` tuples), not a map | Steps 2.5, 5.4 |
| `"compaction": { "buffer": 20000, "keep": { "tokens": 8000 } }` | Real fields: `auto`, `prune`, `tail_turns`, `preserve_recent_tokens`, `reserved` | Step 5.4 |
| `"snapshots": true` | Real key is singular: `"snapshot"` | Step 5.4 |
| `opencode.sessions.create/get/prompt({sessionId, prompt: "text"})` | Real: `client.session.create({body:{title?}})`, `client.session.prompt({path:{id}, body:{parts:[{type:"text",text}]}})` — `parts` is a required array, not a bare string | Step 4.1 |

This shows up **everywhere** the plan writes plugin/hook/tool/agent code, because all of Stage 2–5's TypeScript was drafted before the installed package version was checked. The good news: it's one consistent class of error, and once you've internalized the corrected table above, the rest of this guide should read as "obviously right" rather than mysterious.

---

## Stage 1 — Identity & Config (no plugin API involved)

### Step 1.1 — Install / run OpenCode

**Why:** Nothing to do with memory layers yet — this just gets `opencode` runnable so every later step has something to test against.

**Code:**
```powershell
# Option A — global npm install (simple, but pin it; auto-update against a plan this specific is risky)
npm install -g opencode@latest
opencode --version

# Option B — run the local monorepo clone directly (what <KIP_ROOT> already is)
Set-Location E:\AgentMemoryProject\opencode
bun install
bun dev --help
```

**What this does:** Since `<KIP_ROOT>` *is* a clone of the OpenCode monorepo, `bun dev` runs the exact source you're looking at when you read `packages/opencode/src/...` — useful while you're cross-checking this guide's claims against real source. `npm install -g` gets you the published build instead. Pick one; don't mix — a globally-installed `opencode` and `bun dev` are two different versions reading the same `%USERPROFILE%\.local\share\opencode` data directory.

**✅ Verify:**
```powershell
opencode --version   # or: bun dev --version
```
Any version string, no error, means you're set.

---

### Step 1.2 — Kip Agent Definition (CORRECTED)

**Why:** L3 (identity/persona). This is the file that turns generic OpenCode into "Kip" — its model, permissions, and system prompt.

**⚠️ Corrected vs FUTURE_PLANS.md:** The version already sitting at `.opencode/agent/kip.md` in this repo is a direct copy of the plan's broken snippet: **no `---` frontmatter fences at all** (so `name`/`mode`/`model`/`permission` are never parsed — the whole file is probably being read as raw prompt text right now), and `permission` written as a YAML list with a `write` key that doesn't exist in the real schema (writes are governed by `edit`).

**Code** — replace the full contents of `.opencode/agent/kip.md`:
```markdown
---
name: kip
description: Kip — self-improving personal AI agent for cross-project coding, homelab operations, and standing personal context
mode: primary
model: anthropic/claude-sonnet-5
temperature: 0.3
permission:
  edit: allow
  bash: ask
  webfetch: allow
  skill: allow
  todowrite: allow
---

You are Kip. Read your identity from the kip-soul, kip-user, and kip-identity skills before responding.
Always begin sessions by loading SESSION-STATE.md to understand current project phase.
Follow the memory protocol defined in the kip-agents skill.
```

**What this does:** The `---`-fenced block is YAML frontmatter — OpenCode's markdown-agent loader (`packages/core/src/config/agent.ts`) parses everything between the fences as config and treats everything after the closing `---` as the system prompt. `permission` is a plain object: each key is a tool category (the real set is `read, edit, glob, grep, list, bash, task, external_directory, todowrite, question, webfetch, websearch, lsp, doom_loop, skill` — see `packages/core/src/v1/config/permission.ts`), each value is `"allow" | "deny" | "ask"`. I set `bash: ask` rather than `allow` — Kip gets to *propose* shell commands but you confirm them, which is a reasonable default until Shield (Step 3.2) exists to backstop it.

**✅ Verify:**
```powershell
opencode run --agent kip "Who are you, in one sentence?"
```
Expect a Kip-flavored answer, not a generic assistant response. If it still sounds generic, the frontmatter isn't being picked up — double check the `---` fences are exactly on their own lines with nothing else.

---

### Step 1.3 — Identity Skills (CORRECTED)

**Why:** L3 (identity) + L10 (procedural memory). Skills are how OpenCode injects structured knowledge into context on demand.

**⚠️ Corrected vs FUTURE_PLANS.md:** Same bug as Step 1.2 — `kip-soul/SKILL.md` (and presumably the other four) has **no frontmatter at all**. Per the official skill docs (`packages/web/src/content/docs/skills.mdx`): *"Each SKILL.md must start with YAML frontmatter"* with required `name` (must match the containing directory name) and `description`. Without it, `opencode skill list` won't show it and the agent has no way to know it exists.

**Code** — add this frontmatter block to the very top of each of the five files (`kip-soul`, `kip-user`, `kip-identity`, `kip-agents`, `kip-tools`), before the existing content:
```markdown
---
name: kip-soul
description: Kip's personality, values, and communication style — load before responding as Kip
---

<...existing content stays exactly as-is below this line...>
```
Repeat for the other four, changing only `name:` (must equal the folder name) and writing a one-line `description:` specific to that file's content, e.g. for `kip-agents`:
```markdown
---
name: kip-agents
description: Kip's behavioral rules and memory protocol — how Kip decides what to remember and how
---
```

**What this does:** `description` is what shows up when the agent (or `opencode skill list`) decides *which* skill to load for a given task — it's the one-line summary an LLM uses to pick relevant skills without reading every skill's full body up front. This is the exact mechanism you're seeing right now in this very conversation, in the `<system-reminder>` listing every available skill by name + one-liner.

**✅ Verify:**
```powershell
opencode run --agent kip "list your available skills"
# or, if the CLI exposes it directly:
opencode skill list
```
All five `kip-*` skills should appear with their descriptions. If one's missing, its frontmatter is malformed — check the `name:` matches the folder exactly and both `---` fences are present.

---

### Step 1.4 — Obsidian MCP Server

**Why:** L7 (Obsidian vault) — lets Kip read/write a human-curated knowledge base you already maintain outside Kip.

**Code** — merge into `.opencode\opencode.jsonc` (the file already has `provider`, `permission`, `references`, `tools` — leave those alone, only add/extend `mcp`):
```jsonc
{
  "mcp": {
    "obsidian": {
      "type": "local",
      "command": ["npx", "-y", "mcp-obsidian", "E:/Path/To/Your/Vault"],
      "timeout": 30000
    }
  }
}
```
This part of the plan was already right — the schema (`packages/core/src/v1/config/config.ts`) confirms `mcp` is a **flat** `Record<string, McpConfig>`, matching what's already in this repo's `opencode.jsonc`. Substitute your real vault path, or skip this step entirely if you don't use Obsidian — don't wire a vault you'll never write to.

**✅ Verify:**
```powershell
opencode mcp list
```
`obsidian` should show as connected. If you skipped it, that's fine — nothing downstream in this guide depends on L7.

---

### Step 1.5 — Heartbeat Scheduled Tasks

**Why:** L8 (session state + heartbeat) — gives Kip a proactive pulse instead of only reacting when you type to it.

This step is pure Windows Task Scheduler + PowerShell — it doesn't touch the OpenCode plugin API, so the original plan's code (`FUTURE_PLANS.md` §5 Step 1.5) is accurate as written. Use it directly: the `ritual.ps1` script and the three `Register-ScheduledTask` calls for morning/afternoon/evening.

**One thing worth calling out explicitly** since it's easy to miss: the `--session-id "kip-$(Get-Date -Format 'yyyyMMdd')-$Ritual"` pattern is a **CLI flag** (`opencode run --session-id`), which is separate from — and simpler than — the SDK's `client.session.create()` call you'll use in Step 4.1. The CLI lets you pick your own session ID string directly; the SDK auto-generates one and hands it back to you. Don't try to pass a custom ID through `client.session.create()` — its `body` only accepts `title` and `parentID`.

**✅ Verify:**
```powershell
Get-ScheduledTask -TaskPath '\Kip\' | Format-Table TaskName, State
Start-ScheduledTask -TaskName 'Kip\Ritual-morning'
Get-ScheduledTaskInfo -TaskName 'Kip\Ritual-morning'   # LastTaskResult should be 0
```
And `HEARTBEAT.md` should have a new entry after the test-fire.

---

### Step 1.6 — Memory Promotion Task

**Why:** L9 (promotion pipeline) — the daily job that reads today's L1 log and promotes the important bits into L2 (`MEMORY.md`).

Also pure PowerShell + a scheduled `opencode run` call — no plugin API involved, so `FUTURE_PLANS.md`'s `memory-promote.ps1` and its registration are accurate as written. One honest caveat carried over from the plan and worth taking seriously: asking an LLM to self-score `[ATOM]` entries in a free-text prompt is the least reliable part of this whole design — read `MEMORY.md`'s diff by hand for the first week rather than trusting it silently.

**✅ Verify:**
```powershell
Get-ScheduledTask -TaskName 'Kip\MemoryPromote'
Start-ScheduledTask -TaskName 'Kip\MemoryPromote'
Get-Content MEMORY.md   # should be ≤120 lines and reflect anything scored ≥2 today
```

---

### 🏁 Stage 1 checkpoint

- [X] `opencode run --agent kip "Who are you?"` sounds like Kip, not a generic assistant
- [X] `opencode skill list` (or the agent asked directly) shows all five `kip-*` skills
- [ ] `opencode mcp list` shows `obsidian` connected (or you deliberately skipped L7)
- [X] All four `Kip\*` scheduled tasks exist and test-fire clean (`LastTaskResult = 0`)

---

## Stage 2 — Memory Core (the plugin)

### Step 2.1 — Plugin package (already done — recap)

Already built and verified earlier in this session at `E:\AgentMemoryProject\opencode\kip-plugin\`:
```
kip-plugin/
├── package.json   → "main": "src/index.ts"  (required — see below)
├── bun.lock
├── tsconfig.json
└── src/
    └── index.ts
```

```ts
// src/index.ts
import type { Plugin } from "@opencode-ai/plugin"

export const KipPlugin: Plugin = async (_input) => {
  return {
    // hooks land here
  }
}

export default KipPlugin
```

**Why `"main": "src/index.ts"` matters:** OpenCode's plugin loader (`packages/opencode/src/plugin/shared.ts`, `resolvePackageEntrypoint`) reads `package.json`'s `main` field first; only if that's absent does it fall back to scanning for an `index.*` file **in the package root** — never inside `src/`. Without this field, a `src/`-layout plugin silently fails to load.

**✅ Verify (already done, re-run any time you touch `package.json`):**
```powershell
cd E:\AgentMemoryProject\opencode\kip-plugin
bunx tsc --noEmit -p tsconfig.json   # no output = clean
bun -e "import KipPlugin from './src/index.ts'; console.log(typeof KipPlugin)"   # → function
```

---

### Step 2.2 — L0: Session Transcript Writer (CORRECTED)

**Why:** L0 — the "security camera." Every message, raw, unfiltered, timestamped. You'll rarely read it directly, but it's there to reconstruct exactly what happened if the curated layers above it turn out to be wrong or incomplete.

**⚠️ Corrected vs FUTURE_PLANS.md:** `"session.message.after"` isn't a real hook. The real mechanism is the single catch-all `event` hook, filtering on `event.type === "message.updated"` (confirmed in `@opencode-ai/sdk`'s `Event` union — `EventMessageUpdated = { type: "message.updated", properties: { info: Message } }`).

**Code** — `src/hooks/transcript.ts`:
```ts
import { appendFile, mkdir } from "fs/promises"
import { dirname, join } from "path"
import type { Plugin } from "@opencode-ai/plugin"

// Forward slashes: Windows accepts them everywhere in Bun, and they dodge the
// "\t is a tab, not a folder" escape trap that bites every Windows path literal.
const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"
const TRANSCRIPT_DIR = join(KIP_ROOT, "memory", "transcripts")

export const transcriptHook: Awaited<ReturnType<Plugin>> = {
  event: async ({ event }) => {
    if (event.type !== "message.updated") return

    const date = new Date().toISOString().slice(0, 10) // YYYY-MM-DD
    const filePath = join(TRANSCRIPT_DIR, `${date}.jsonl`)
    await mkdir(dirname(filePath), { recursive: true })
    await appendFile(filePath, JSON.stringify(event.properties.info) + "\n", "utf8")
  },
}
```

**What this does:** `event` is the one hook OpenCode calls for *every* server-sent event across all sessions (`packages/opencode/src/plugin/index.ts` wires this up by listening to the whole event bus and calling `hook["event"]?.({event})` for each hook). We filter down to `"message.updated"`, which fires once per message as it's written/updated, and append the full `Message` object (`event.properties.info`) as one JSON line. I added `mkdir(..., {recursive:true})` — the plan's version assumed `memory/transcripts/` always exists, which it won't on a fresh clone.

**✅ Verify:**
```powershell
# Register this hook in src/index.ts first (see Step 3.4's full wiring — or just import
# and spread { ...transcriptHook } into the object your plugin function returns).
opencode run --agent kip "Say hello."
Get-ChildItem memory\transcripts\*.jsonl
Get-Content (Get-ChildItem memory\transcripts\*.jsonl | Select -Last 1) -Tail 1
```
Expect a JSON line containing your session's latest message.

---

### Step 2.3 — L1: Daily Memory Log Writer (CORRECTED)

**Why:** L1 — the "diary entry." At natural pause points, pull the notable bits out of the raw firehose and tag them `[ATOM]` so L9 can promote them into L2 later.

**⚠️ Corrected vs FUTURE_PLANS.md:** `"session.compaction.ended"` doesn't exist, and there is no hook that hands you a *finished* compaction summary — the real compaction hook (`"experimental.session.compacting"`) fires *before* compaction, letting you inject extra context into the prompt, not read the result. The closest real trigger for "a session just wrapped up" is the `session.idle` event. Also: `extractAtoms()` in the plan was a stub that always returned `[]`, and the template's `.replace()` chain only ever filled in 3 of its 9 placeholders — the rest would have been written to the file as the literal string `{objective}` forever. Both are fixed below with a minimal but real implementation.

**Code** — `src/hooks/daily-log.ts`:
```ts
import { appendFile, mkdir } from "fs/promises"
import { join } from "path"
import type { Plugin } from "@opencode-ai/plugin"

const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"
const MEMORY_DIR = join(KIP_ROOT, "memory")

type Atom = { type: "decision" | "lesson" | "preference" | "event" | "fact"; entity: string; detail: string }

// Deliberately simple heuristics, not an LLM call — see FUTURE_PLANS.md §13 Phase 4
// for why a structured-output pass eventually beats free-text pattern matching.
// This gets you *something real* now instead of a stub that always returns [].
const PATTERNS: Array<{ type: Atom["type"]; re: RegExp }> = [
  { type: "decision", re: /\b(decided to|will use|going with|chose)\b/i },
  { type: "lesson", re: /\b(learned that|found that|realized|turns out)\b/i },
  { type: "preference", re: /\b(prefer|better to|from now on|going forward)\b/i },
]

function extractAtoms(text: string): Atom[] {
  const atoms: Atom[] = []
  for (const line of text.split(/\n+/)) {
    for (const { type, re } of PATTERNS) {
      if (re.test(line)) {
        atoms.push({ type, entity: "session", detail: line.trim().slice(0, 200) })
        break
      }
    }
  }
  return atoms
}

function formatAtom(a: Atom): string {
  return `[ATOM] type=${a.type} | entity=${a.entity} | detail=${a.detail} | ref=session`
}

export const dailyLogHook: Awaited<ReturnType<Plugin>> = {
  event: async ({ event }, ctx) => {
    if (event.type !== "session.idle") return
    // ctx isn't actually passed to event hooks (see input shape below) — we need
    // the client from PluginInput instead. This hook has to be created as a
    // closure over `input` — see the factory pattern in src/index.ts (Step 3.4).
  },
}

// Real version: a factory that closes over PluginInput so it can call the SDK client.
export function makeDailyLogHook(client: import("@opencode-ai/plugin").PluginInput["client"]) {
  return {
    event: async ({ event }: { event: import("@opencode-ai/sdk").Event }) => {
      if (event.type !== "session.idle") return
      const sessionID = event.properties.sessionID

      const result = await client.session.messages({ path: { id: sessionID } })
      if (result.error || !result.data) return
      const text = result.data
        .flatMap((m) => m.parts)
        .filter((p): p is Extract<typeof p, { type: "text" }> => p.type === "text")
        .map((p) => p.text)
        .join("\n")

      const atoms = extractAtoms(text)
      if (atoms.length === 0) return // nothing worth logging this turn

      const date = new Date().toISOString().slice(0, 10)
      const time = new Date().toTimeString().slice(0, 5)
      const filePath = join(MEMORY_DIR, `${date}.md`)
      const entry = `\n## Session ${sessionID} — ${date} ${time}\n\n${atoms.map(formatAtom).join("\n")}\n`

      await mkdir(MEMORY_DIR, { recursive: true })
      await appendFile(filePath, entry, "utf8")
    },
  }
}
```

**What this does:** Since the daily-log hook needs to *call back into OpenCode* (fetch the session's messages) rather than just receive data, it can't be a static object like the transcript hook — it needs access to `client` from `PluginInput`, which only exists inside the plugin function's scope. So this is written as a **factory function** (`makeDailyLogHook(client)`) that your `src/index.ts` calls once, at plugin-init time, passing in the `client` it received. `session.idle` fires when the agent goes quiet after a turn — a reasonable proxy for "a chunk of work just finished." We fetch that session's messages, flatten out just the `text` parts, run the same lightweight keyword heuristics the plan sketched (but actually implemented), and append real `[ATOM]` lines — skipping the write entirely if nothing matched, so idle sessions with nothing notable don't pollute the log with empty headers.

I left the first `dailyLogHook` stub in place with a comment explaining *why* it's wrong, on purpose — it's the exact shape you'd naturally reach for coming from the plan's pseudocode, and it's worth seeing concretely why "the hook just needs `input.client`" isn't available on the per-event callback signature (`event` hooks only receive `{event}`, per the real `Hooks` type) and has to come from the closure instead.

**✅ Verify:**
```powershell
opencode run --agent kip "We decided to use PowerShell 7 for all scheduled tasks going forward."
# wait for the session to go idle, then:
Get-Content "memory\$(Get-Date -Format 'yyyy-MM-dd').md" -Tail 10
```
Expect a `[ATOM] type=decision | ...` or `type=preference | ...` line referencing PowerShell.

---

### Step 2.4 — L5: MCP Wrapper for ChromaDB + NetworkX

**Why:** L5 — lets Kip *search* old memory by meaning ("what do I know about X?"). This is a completely separate concern from Steps 2.2/2.3 — it doesn't write anything, it's a read-side query tool, wired in via MCP rather than the plugin hook system.

This step is Python + the MCP stdio JSON-RPC protocol, which doesn't depend on `@opencode-ai/plugin` at all, so `FUTURE_PLANS.md`'s `memory-mcp-server.py` skeleton is a reasonable starting point as written — the protocol boilerplate (`initialize`, `tools/list`, `tools/call` over stdin/stdout JSON lines) is genuinely just that simple, confirmed by MCP's own spec. The two things worth tightening when you actually implement `search_chromadb`/`lookup_graph`:
- The stray Windows backslash path literals in the plan's `subprocess.run([...])` calls (`"E:\AgentMemoryProject\..."`) are a real Python bug — `\A`, `\o` etc. are invalid escape sequences in a plain string. Use a raw string (`r"E:\AgentMemoryProject\..."`) or forward slashes, same rule as the TypeScript side.
- Don't shell out via `subprocess` from the MCP server to *another* Python script per call if you can avoid it — import `search.py`/`graph_lookup.py` as modules instead. It's faster and you get real Python exceptions instead of parsing subprocess stdout.

**✅ Verify:**
```powershell
opencode mcp list   # kip-memory should show connected once registered (Step 2.5/5.4)
```
Full functional verification (`memory_search` actually returning results) has to wait until you've indexed something — that's downstream of building `indexer.py`, which isn't part of Steps 1–5.4's core path.

---

### Step 2.5 — Register the Plugin (CORRECTED)

**⚠️ Corrected vs FUTURE_PLANS.md:** `"plugin": { "kip": "path" }` is wrong shape — the config schema (`packages/core/src/v1/config/config.ts`) defines `plugin` as `Schema.Array(ConfigPluginV1.Spec)`, an **array**, matching `@opencode-ai/plugin`'s own `Config` type (`plugin?: Array<string | [string, PluginOptions]>`) and the official docs' example (`"plugin": ["opencode-helicone-session", ...]`).

**Code** — add to `.opencode\opencode.jsonc`:
```jsonc
{
  "plugin": ["E:/AgentMemoryProject/opencode/kip-plugin"]
}
```

**✅ Verify:**
```powershell
opencode run --agent kip "Say hello."
Get-ChildItem memory\transcripts\*.jsonl   # Step 2.2's hook should have fired
```
If nothing appears, check `opencode`'s startup logs for a plugin load error — the loader publishes a `Session.Event.Error` with a message like `Failed to load plugin ...` (see `packages/opencode/src/plugin/index.ts`).

---

### 🏁 Stage 2 checkpoint

- [ ] `bunx tsc --noEmit` in `kip-plugin/` is clean
- [ ] A session produces a new line in `memory\transcripts\<date>.jsonl`
- [ ] A session with an obvious decision/preference produces a matching `[ATOM]` line in `memory\<date>.md`
- [ ] `opencode mcp list` shows `kip-memory` connected

---

## Stage 3 — Deterministic Enforcement

### Step 3.1 — Logician Tool (CORRECTED)

**Why:** L4 — binary verification before Kip is allowed to claim "done."

**⚠️ Corrected vs FUTURE_PLANS.md:** The plan's version uses JSON-Schema-style `parameters` and an `id` field. The real `tool()` helper (`@opencode-ai/plugin/dist/tool.d.ts`) takes `{ description, args: <Zod raw shape>, execute(args, context) }` — no `id` (the key you register it under in `Hooks.tool` *is* its name), and `args` values are Zod schemas built via the `tool.schema` namespace the package exposes specifically so plugin authors don't need their own `zod` dependency.

**Code** — `src/tools/logician.ts`:
```ts
import { tool } from "@opencode-ai/plugin"

export const LogicianTool = tool({
  description:
    "Binary verification before claiming a task complete. Returns verified=true only if the condition is provably met. Use before saying 'done'.",
  args: {
    check_type: tool.schema.enum(["file_exists", "file_contains", "cmd_succeeds", "dir_exists"]),
    target: tool.schema.string().describe("Path or command to verify"),
    expected: tool.schema.string().optional().describe("For file_contains: the string that must be present"),
  },
  execute: async ({ check_type, target, expected }) => {
    const args = ["pwsh", "-NoProfile", "-ExecutionPolicy", "Bypass",
      "-File", `${process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"}/kip-scripts/logician-verify.ps1`,
      "-CheckType", check_type, "-Target", target,
      ...(expected ? ["-Expected", expected] : [])]
    const result = Bun.spawnSync(args)
    const verified = result.exitCode === 0
    const output = new TextDecoder().decode(result.stdout).trim()
    return {
      output: JSON.stringify({ verified, check_type, target, detail: output || (verified ? "Check passed" : "Check failed") }),
    }
  },
})
```

**What this does:** `tool.schema` is just `zod` re-exported under a different name (`tool.schema.enum(...)` === `z.enum(...)`) — the package does this so you don't need `zod` as a separate dependency. `execute`'s return value matches the real `ToolResult` type: either a bare string, or `{ title?, output, metadata?, attachments? }` — I used the object form with `output` holding the JSON so it's easy for Kip to parse the result back out. `url_reachable` from the plan's `check_type` enum was dropped here since `logician-verify.ps1` (a Stage-4-adjacent script, not covered in this guide) would need to actually implement it — add it back once that script supports it.

**✅ Verify:**
```powershell
opencode run --agent kip "Verify file exists: E:\AgentMemoryProject\opencode\MEMORY.md"
```
Expect a response reflecting `{"verified":true,...}`.

---

### Step 3.2 — Shield Hook (CORRECTED)

**Why:** L4 — blocks writes outside Kip's approved directories, on the honest assumption it stops accidents, not a determined adversary.

**⚠️ Corrected vs FUTURE_PLANS.md:** Real hook name is `"tool.execute.before"`, not `"tool.before"`; its input/output shape is `(input: {tool, sessionID, callID}, output: {args})`, not `toolCall.parameters`. And the real built-in `write` tool's argument is named **`path`**, not `file_path` (confirmed in `packages/core/src/tool/write.ts`) — the plan's field name would have silently never matched anything, since `output.args.file_path` is always `undefined` on the real tool call.

**Code** — `src/hooks/shield.ts`:
```ts
import { resolve, sep } from "path"
import { tmpdir } from "os"
import { appendFile } from "fs/promises"
import { join } from "path"

const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"

// The repo root itself is deliberately NOT a safe zone — Kip shouldn't be able
// to rewrite OpenCode's own source tree by accident.
const SAFE_ZONES = [
  `${KIP_ROOT}/memory`,
  `${KIP_ROOT}/kip-plugin`,
  `${KIP_ROOT}/kip-scripts`,
  `${KIP_ROOT}/.opencode`,
  tmpdir(),
].map((z) => resolve(z).toLowerCase())

function isPathSafe(candidate: string): boolean {
  const target = resolve(candidate).toLowerCase()
  return SAFE_ZONES.some((zone) => target === zone || target.startsWith(zone + sep))
}

async function appendToLog(line: string) {
  await appendFile(join(KIP_ROOT, "KIP_FAILURE_AUDIT_LOG.md"), `${new Date().toISOString()}: ${line}\n`, "utf8")
}

export const shieldHook = {
  "tool.execute.before": async (
    input: { tool: string; sessionID: string; callID: string },
    output: { args: any },
  ) => {
    if (input.tool === "write" && output.args?.path) {
      if (!isPathSafe(output.args.path)) {
        await appendToLog(`Shield blocked write to '${output.args.path}'`)
        throw new Error(
          `Shield: write to '${output.args.path}' blocked. Safe zones: ${SAFE_ZONES.join(", ")}`,
        )
      }
    }

    if (input.tool === "bash" && output.args?.command) {
      const cmd = output.args.command as string
      // Smoke alarm, not a lock — a regex over shell text cannot reliably tell a
      // safe `rm` from a dangerous one. Logged for review, not blocked.
      const destructive = /\b(Remove-Item|rm|del|erase|rd|rmdir|Move-Item|mv|move|Set-Acl|icacls|takeown|format)\b/i
      if (destructive.test(cmd)) await appendToLog(`[SHIELD WARNING] Destructive command: ${cmd}`)
    }
  },
}
```

**What this does:** `tool.execute.before` fires immediately before any tool call runs, for every tool, built-in or custom. `output.args` is the actual argument object the model is about to send to the tool — and it's **mutable**: you could rewrite `output.args.path` instead of throwing, if you wanted to redirect rather than block. Throwing an `Error` inside the hook blocks the call — confirmed by OpenCode's own `.env`-protection example in the official plugin docs, which does exactly this (`if (...) throw new Error("Do not read .env files")`).

**✅ Verify:**
```powershell
opencode run --agent kip "Write a test file to C:\Windows\System32\test.txt"
```
Expect the write to fail with the Shield error message, and a new line in `KIP_FAILURE_AUDIT_LOG.md`.

---

### Step 3.3 — Guardian Scheduled Task

Pure PowerShell + Task Scheduler, no plugin API — `FUTURE_PLANS.md`'s `guardian.ps1` and its registration are accurate as written. Use them directly.

**✅ Verify:**
```powershell
schtasks /Query /TN "Kip\Guardian"
Get-Content memory\GUARDIAN_LOG.md -ErrorAction SilentlyContinue
```

---

### Step 3.4 — Wire everything into `src/index.ts` (CORRECTED)

**Why:** Ties Steps 2.2, 2.3, 3.1, 3.2 together into the one object OpenCode actually loads.

**Code** — final `src/index.ts`:
```ts
import type { Plugin } from "@opencode-ai/plugin"
import { transcriptHook } from "./hooks/transcript"
import { makeDailyLogHook } from "./hooks/daily-log"
import { shieldHook } from "./hooks/shield"
import { LogicianTool } from "./tools/logician"

export const KipPlugin: Plugin = async (input) => {
  const dailyLogHook = makeDailyLogHook(input.client)

  return {
    tool: {
      logician_verify: LogicianTool,
    },
    event: async (args) => {
      // Multiple hooks each want the `event` callback — call every one in turn.
      await transcriptHook.event?.(args)
      await dailyLogHook.event(args)
    },
    "tool.execute.before": shieldHook["tool.execute.before"],
  }
}

export default KipPlugin
```

**What this does:** OpenCode's plugin loader only ever looks for **one** `event` function per hooks object (see `Hooks` interface — each key is a single function, not an array). Since both the transcript writer and the daily-log writer need to react to events, `src/index.ts` is where you merge them manually into one `event` function that calls both in sequence, rather than trying to register `event` twice.

**✅ Verify (Stage 3 checkpoint):**
- [x] `opencode run --agent kip "Verify file exists: ...\MEMORY.md"` → `verified: true`
- [X] `opencode run --agent kip "Write to C:\Windows\System32\test.txt"` → blocked, logged
- [ ] `bunx tsc --noEmit` in `kip-plugin/` still clean after merging all four pieces
- [ ] A normal session still produces both a transcript line *and* (when relevant) an `[ATOM]` line — confirming the merged `event` handler didn't break either hook

---

## Stage 4 — WhatsApp Integration

> **Honesty note for this stage:** `whatsapp-web.js` (or Twilio/Cloud API, whichever you pick) isn't installed in this environment, so its usage below is illustrative, not independently compiled — verify method names against that library's own current docs when you get here. What **is** verified is the OpenCode SDK half of the bridge (`createOpencode`, `client.session.*`), which is the part `FUTURE_PLANS.md` got wrong.

### Step 4.1 — The Bridge (CORRECTED)

**⚠️ Corrected vs FUTURE_PLANS.md:** `opencode.sessions.get/create/prompt({sessionId, prompt: "text"})` doesn't exist. The real client (`@opencode-ai/sdk`'s `OpencodeClient`) nests session methods under **`.session`** (singular), takes a `{path, body, query}` options object per call (this is a generated [hey-api](https://heyapi.dev) client), and `prompt` requires a `parts` array, not a bare string. Calls return `{data, error}` by default rather than throwing.

**Code** — `whatsapp-bridge/src/index.ts`:
```ts
import { createOpencode } from "@opencode-ai/sdk"
import pkg from "whatsapp-web.js"
const { Client, LocalAuth } = pkg

const { client } = await createOpencode({ hostname: "localhost", port: 4001 })

const AUTHORIZED_NUMBER = process.env.AUTHORIZED_WHATSAPP_NUMBER!
let sessionID: string | undefined // resolved lazily below

const whatsapp = new Client({ authStrategy: new LocalAuth({ clientId: "kip" }), puppeteer: { args: ["--no-sandbox"] } })

whatsapp.on("message", async (msg) => {
  if (!msg.from.includes(AUTHORIZED_NUMBER) || msg.type !== "chat") return

  try {
    await msg.getChat().then((chat) => chat.sendStateTyping())

    if (!sessionID) {
      const created = await client.session.create({ body: { title: "Kip WhatsApp" } })
      if (created.error || !created.data) throw new Error("session.create failed")
      sessionID = created.data.id
    }

    const result = await client.session.prompt({
      path: { id: sessionID },
      body: { agent: "kip", parts: [{ type: "text", text: msg.body }] },
    })
    if (result.error) throw new Error(JSON.stringify(result.error))

    const text = result.data?.parts
      .filter((p): p is Extract<typeof p, { type: "text" }> => p.type === "text")
      .map((p) => p.text)
      .join("")
    await msg.reply(text || "(no response)")
  } catch (err) {
    console.error("Kip bridge error:", err)
    await msg.reply("Kip encountered an error. Check the logs.")
  }
})

whatsapp.initialize()
console.log("Kip WhatsApp bridge started. Scan QR code to authenticate.")
```

**What this does:** `createOpencode()` (not `createOpencodeClient()` — that's the lower-level variant for connecting to an *already-running* server) both spawns/connects an embedded server and hands back a ready-to-use `client`. Because `client.session.create()` doesn't accept a caller-chosen session ID, the bridge resolves one lazily on first message and holds it in memory for the process's lifetime, rather than trying to pass a fixed `WHATSAPP_SESSION_ID` in — that pattern from the plan doesn't map onto the real API. `session.prompt`'s response shape mirrors the request/message structure you already saw in Step 2.3's `session.messages()` — `parts` you filter down to `type === "text"`.

**✅ Verify:** covered under Step 4.2's checkpoint, once both processes are actually running.

### Step 4.2 — Run as a Background Task

Pure Windows Task Scheduler + secrets handling — no OpenCode API involved. `FUTURE_PLANS.md`'s registration script and its secrets warning (scheduled-task XML is world-readable; use `[Environment]::SetEnvironmentVariable(..., 'User')`, not a `.env` file in the repo) are accurate as written. Use them directly.

**✅ Verify (Stage 4 checkpoint):**
- [ ] `Get-ScheduledTask -TaskName 'Kip\OpenCodeServer'` is `Running`; `Invoke-WebRequest http://localhost:4001/health` → 200 (if your OpenCode build exposes `/health` — check `packages/opencode/src/server/routes` if it 404s)
- [ ] `Get-ScheduledTask -TaskName 'Kip\WhatsAppBridge'` is `Running`
- [ ] A WhatsApp message to the authorized number gets a Kip-persona reply within ~30s
- [ ] Rebooting brings both tasks back without manual steps

---

## Stage 5 — Polish & Hardening

### Step 5.1 — Memory Wiki Tool (CORRECTED)

**⚠️ Corrected vs FUTURE_PLANS.md:** Same JSON-Schema-vs-Zod mismatch as Step 3.1.

**Code** — add to `src/tools/wiki-synthesize.ts`:
```ts
import { tool } from "@opencode-ai/plugin"

export const WikiSynthesizeTool = tool({
  description: "Create or update a synthesis wiki page for a topic, with provenance tracking",
  args: {
    topic: tool.schema.string(),
    source_dates: tool.schema.array(tool.schema.string()).optional()
      .describe("YYYY-MM-DD dates of daily logs to synthesize from"),
  },
  execute: async ({ topic, source_dates }) => {
    const slug = topic.toLowerCase().replace(/\s+/g, "-")
    const wikiPath = `${process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"}/memory/wiki/${slug}.md`
    const header = `# ${topic}\n\n> Synthesized from: ${(source_dates ?? []).join(", ")}\n> Last updated: ${new Date().toISOString().slice(0, 10)}\n\n`
    await Bun.write(wikiPath, header)
    return { output: JSON.stringify({ path: wikiPath, status: "scaffold_created" }) }
  },
})
```
Register it in `src/index.ts`'s `tool` object alongside `logician_verify`. As the docs note: the tool then still needs the *model* to actually read sources and append the synthesis body — this tool only creates the provenance-headed scaffold, same intent as the plan.

**✅ Verify:**
```powershell
opencode run --agent kip "Synthesize a wiki page for: the Kip plugin hook design"
Test-Path memory\wiki\the-kip-plugin-hook-design.md
```

### Step 5.2 — Session-End Skill Detection Hook (CORRECTED)

**⚠️ Corrected vs FUTURE_PLANS.md:** `"session.end"` doesn't exist — same real trigger as Step 2.3, `session.idle`. It also referenced a bare `opencode` client that's never defined in that file's scope; it needs the same factory-closure pattern as `makeDailyLogHook`.

**Code** — `src/hooks/skill-detection.ts`:
```ts
import { appendFile, mkdir } from "fs/promises"
import { join } from "path"
import type { PluginInput } from "@opencode-ai/plugin"

const KIP_ROOT = process.env.KIP_ROOT ?? "E:/AgentMemoryProject/opencode"

export function makeSkillDetectionHook(client: PluginInput["client"]) {
  return {
    event: async ({ event }: { event: import("@opencode-ai/sdk").Event }) => {
      if (event.type !== "session.idle") return
      const sessionID = event.properties.sessionID

      const result = await client.session.prompt({
        path: { id: sessionID },
        body: {
          noReply: false,
          parts: [{
            type: "text",
            text:
              "Skill detection check (answer briefly): did you repeat a multi-step procedure, " +
              "use a non-obvious fix, or catch a mistake you've made before this session? " +
              "If yes: output SKILL_CANDIDATE: <name> | <1-sentence description>. If no: output SKIP.",
          }],
        },
      })
      if (result.error || !result.data) return
      const text = result.data.parts
        .filter((p): p is Extract<typeof p, { type: "text" }> => p.type === "text")
        .map((p) => p.text).join("")

      if (!text.includes("SKILL_CANDIDATE:")) return
      await mkdir(KIP_ROOT + "/memory", { recursive: true })
      await appendFile(
        join(KIP_ROOT, "memory", "skill-suggestions.md"),
        `${new Date().toISOString().slice(0, 10)} — ${text}\n`,
        "utf8",
      )
    },
  }
}
```

**What this does:** Same shape as the daily-log hook — a factory over `client` — but this one *prompts the model again* inside the hook (a second, cheap turn asking it to self-assess) rather than just reading past messages. Wire it into `src/index.ts`'s merged `event` function the same way as Step 3.4.

**⚠️ Worth knowing before you enable this:** chaining a `client.session.prompt()` call from *inside* an `event` hook that's listening for `session.idle` risks a feedback loop — that prompt call itself will eventually produce another `session.idle` event. Guard against it (e.g. tag skill-detection turns with a marker in the prompt and skip re-triggering on your own tagged turns) before running this unattended.

**✅ Verify:**
```powershell
Get-Content memory\skill-suggestions.md -ErrorAction SilentlyContinue
```
Grows over multiple sessions where you actually repeated a procedure.

### Step 5.3 — Health Monitoring

Pure PowerShell — no OpenCode API involved. `FUTURE_PLANS.md`'s guarded-restart logic (two-strikes-before-restart, process-alive check) is accurate as written. Use it directly.

**✅ Verify:**
```powershell
Stop-Process -Name opencode -ErrorAction SilentlyContinue
Start-ScheduledTask -TaskName 'Kip\Guardian'   # strike 1 — should NOT restart yet
Start-ScheduledTask -TaskName 'Kip\Guardian'   # strike 2 — should restart
Get-Process opencode   # exactly one process
```

### Step 5.4 — Final Configuration (CORRECTED)

**⚠️ Corrected vs FUTURE_PLANS.md:** `"plugin"` as a map (same bug as Step 2.5), `"compaction.buffer"`/`"compaction.keep.tokens"` (real fields are `auto, prune, tail_turns, preserve_recent_tokens, reserved` — see `packages/core/src/v1/config/config.ts`), and `"snapshots"` (real key is singular, `"snapshot"`).

**Code** — final `.opencode\opencode.jsonc` additions (merge; `provider`/`permission`/`references`/`tools` stay as-is):
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-sonnet-5",
  "default_agent": "kip",
  "snapshot": true,
  "compaction": {
    "auto": true,
    "reserved": 20000,
    "preserve_recent_tokens": 8000
  },
  "mcp": {
    "obsidian": { "type": "local", "command": ["npx", "-y", "mcp-obsidian", "E:/Path/To/Vault"], "timeout": 30000 },
    "kip-memory": { "type": "local", "command": ["python", "E:/AgentMemoryProject/opencode/kip-scripts/memory-mcp-server.py"], "timeout": 15000 }
  },
  "plugin": ["E:/AgentMemoryProject/opencode/kip-plugin"]
}
```

**✅ Verify (Stage 5 / full-system checkpoint):**
- [ ] `opencode run --agent kip "Synthesize a wiki page for: the Kip plugin hook design"` creates the file
- [ ] `memory\skill-suggestions.md` grows over time
- [ ] Guardian's two-strike restart logic behaves as tested in Step 5.3
- [ ] A full reboot brings the OpenCode server and WhatsApp bridge back with no manual steps
- [ ] Exactly one `opencode` process after all of the above (`Get-Process opencode`) — no restart pile-up

---

## What to do with this vs. FUTURE_PLANS.md going forward

Treat `FUTURE_PLANS.md` as the **architecture reference** (layers, rationale, the "why") and this file as the **buildable spec** (the "how", checked against real code). When they conflict on code, this file wins — it was checked against the actual installed package; the original was written against an imagined one. If you upgrade `@opencode-ai/plugin`/`@opencode-ai/sdk` later, re-run the `bunx tsc --noEmit` check after each stage — that's genuinely how you'd have caught this the first time.
