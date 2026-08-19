# OpenCode — Application Understanding Guide

> Generated from live repository analysis. Last updated: 2026-08-19.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture & Components](#2-architecture--components)
3. [How to Make Changes](#3-how-to-make-changes)
4. [Debugging Guide](#4-debugging-guide)
5. [Deployment & Testing](#5-deployment--testing)
6. [Practical Examples](#6-practical-examples)
7. [Next Steps](#7-next-steps)
8. [Topics to Learn](#8-topics-to-learn)
9. [Missing Features](#9-missing-features)

---

## 1. Project Overview

### What is OpenCode?

OpenCode is an **open-source, terminal-native AI coding assistant** — think "Claude Code / Cursor for the terminal, fully self-hostable." It runs as a local process that:

- Spawns a local HTTP server (default `localhost:4001`)
- Serves a TUI (terminal UI), a web UI, and an Electron desktop app against the same server
- Exposes an OpenAPI-compliant REST + WebSocket API consumed by IDE plugins (VS Code extension, ACP-compatible clients)
- Connects to any LLM provider (Anthropic, OpenAI, Google, Bedrock, Azure, xAI, Cloudflare, OpenRouter, Ollama, etc.)
- Integrates with MCP (Model Context Protocol) servers for tool expansion
- Implements ACP (Agent Client Protocol) for IDE/tool host integration

### Core Purpose

| Dimension | Description |
|---|---|
| **User-facing** | A fully keyboard-driven AI pair programmer that works in any terminal |
| **Developer-facing** | An extensible platform: add agents, tools, skills, plugins, and MCP servers |
| **Enterprise-facing** | Self-hostable + optional SaaS sharing/console via Cloudflare + AWS infra |

### Design Philosophy

1. **Effect-first**: The entire codebase uses [Effect](https://effect.website/) v4 (beta) for dependency injection, error handling, streaming, and concurrency — no `async/await` in core logic.
2. **Single server, many surfaces**: One `opencode serve` process powers TUI, web, desktop, and IDE clients simultaneously.
3. **Bun runtime**: Everything runs on Bun (not Node.js) for fast startup and native TS execution.
4. **Composition over monolith**: 34 scoped packages in a Bun monorepo, each with a clear responsibility boundary.
5. **Protocol-driven**: Client↔Server communication is defined in `packages/protocol` and `packages/schema`; code is generated from those definitions.
6. **Skill/plugin extensibility**: Users add `SKILL.md` files or npm plugins without touching core code.

---

## 2. Architecture & Components

### 2.1 Monorepo Layout

```
opencode/
├── packages/                  # All sub-packages (34 total)
│   ├── opencode/              # Main CLI + server + TUI (primary entrypoint)
│   ├── core/                  # Business logic: sessions, agents, tools, config, git, LSP
│   ├── llm/                   # LLM provider adapters + routing
│   ├── tui/                   # Terminal UI (SolidJS + opentui)
│   ├── app/                   # Shared web/desktop UI (SolidJS + Vite)
│   ├── desktop/               # Electron wrapper
│   ├── plugin/                # Public plugin API
│   ├── schema/                # Shared Effect schemas
│   ├── protocol/              # Shared protocol types (client↔server)
│   ├── client/                # HTTP + Effect client (generated)
│   ├── sdk/                   # Public TypeScript SDK
│   ├── sdk-next/              # Next-gen SDK composing client+core+server
│   ├── server/                # Server-side auth, CORS, routes, PTY
│   ├── session-ui/            # SolidJS session UI components
│   ├── console/               # Web console frontend
│   ├── docs/                  # Astro + Starlight docs site
│   ├── enterprise/            # Enterprise Cloudflare Worker + UI
│   ├── function/              # Cloudflare Worker functions (GitHub App auth)
│   └── ...                    # 20+ more support packages
├── infra/                     # SST v4 infrastructure (Cloudflare + AWS)
├── sdks/vscode/               # VS Code extension SDK
├── specs/                     # Architecture and design specs
├── patches/                   # Bun patches for 16 third-party packages
├── script/                    # Release, changelog, version, signing scripts
├── nix/                       # Nix packages for opencode + desktop
├── .opencode/                 # Project-level OpenCode config (agents, commands, skills)
├── sst.config.ts              # SST v4 infra config
├── package.json               # Bun monorepo root
├── bunfig.toml                # Bun install settings
├── AGENTS.md                  # AI agent coding rules
└── CONTRIBUTING.md            # Contributor guide
```

### 2.2 Key Configuration Files

**`bunfig.toml`** — Controls Bun's package manager behavior:
```toml
[install]
exact = true                        # Pin exact versions (no ^ ranges)
minimumReleaseAge = 259200          # 3-day release age gate (security)
# excludes fast-moving internal packages from the age gate
```

**`sst.config.ts`** — Cloud infrastructure (SST v4):
```typescript
// App: "opencode", home: Cloudflare
// Providers: AWS us-east-1, Stripe, PlanetScale, Honeycomb
// Infra modules: app, console, lake, stats, enterprise, monitoring
```

**`package.json`** (root) — Bun workspaces:
```json
{
  "packageManager": "bun@1.3.14",
  "workspaces": ["packages/*", "packages/console/*", "packages/stats/*", "packages/sdk/js", "packages/slack"],
  "scripts": {
    "dev": "turbo dev",
    "dev:desktop": "turbo dev --filter=@opencode-ai/desktop",
    "dev:web": "turbo dev --filter=@opencode-ai/web",
    "test": "turbo test",
    "typecheck": "turbo typecheck"
  }
}
```

**`AGENTS.md`** — Rules for AI coding agents:
- Default branch is `dev` (not `main`)
- Branch names: max 3 words, hyphen-separated, no `feat/`/`fix/` prefixes
- Commit style: conventional commits (`feat(core): ...`)
- Valid scopes: `core`, `opencode`, `tui`, `app`, `desktop`, `sdk`, `plugin`
- No `try/catch`, no `any`, no star imports, use Effect generators

**`.opencode/opencode.jsonc`** — Project-level OpenCode self-configuration:
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "references": {
    "effect": { "repository": "github.com/Effect-TS/effect-smol" },
    "opencode-local": { "path": "~/.local/share/opencode" }
  },
  "tools": { "github-triage": false, "github-pr-search": false }
}
```

### 2.3 The `packages/opencode` Package — Main Entrypoint

This is the runnable package (`bun run packages/opencode/src/index.ts`).

**CLI commands registered in `src/index.ts`:**

| Command | Purpose |
|---|---|
| `run [prompt]` | Send a prompt non-interactively, or launch mini interactive mode |
| `serve` | Start headless HTTP API server |
| `web` | Start server + open web interface |
| `acp` | Start ACP (Agent Client Protocol) server for IDE integration |
| `mcp` | MCP sub-commands (add, list, remove) |
| `agent` | Agent management (list, create, edit) |
| `session` | Session management (list, delete, export) |
| `models` / `providers` | List available models/providers |
| `generate` | Trigger code generation |
| `export` / `import` | Session data portability |
| `attach <url>` | Attach TUI to a running remote server |
| `github` / `pr` | GitHub integration |
| `plug` | Plugin management (add, remove, list) |
| `db` | Database operations (migrate, inspect) |
| `debug` | Debug utilities |
| `stats` | Usage statistics |
| `upgrade` / `uninstall` | Self-management |

### 2.4 Core Package — Business Logic

`packages/core/src/` contains:

| Module | Role |
|---|---|
| `session.ts` / `session/` | V2 session: prompt admission, execution, runner, store, compaction, revert |
| `agent.ts` | Agent registry, `AgentV2` definition, default agent |
| `config.ts` | Full config schema + loader |
| `tool/` | Built-in tool implementations (bash, edit, glob, grep, read, write, ...) |
| `event.ts` | Event bus — 25+ event types |
| `git.ts` | Git integration (37 KB) |
| `permission.ts` | Tool permission system (12 KB) |
| `integration.ts` | Third-party integrations (22 KB) |
| `snapshot.ts` | File snapshot/undo system |
| `lsp/` | Language Server Protocol integration |
| `database/` | SQLite via Drizzle ORM |
| `filesystem/` | File watching |

### 2.5 Agents

An agent is defined by `packages/opencode/src/agent/agent.ts`:

```typescript
const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.String,
  mode: Schema.Literal("subagent", "primary", "all"),  // where agent can be used
  permission: Schema.Array(PermissionRule),             // tool access rules
  model: Schema.optional(Schema.String),               // model override
  temperature: Schema.optional(Schema.Number),
  topP: Schema.optional(Schema.Number),
  color: Schema.optional(Schema.String),
  prompt: Schema.optional(Schema.String),              // system prompt override
  steps: Schema.optional(Schema.Number),               // max reasoning steps
})
```

**Agent sources (in priority order):**
1. `.opencode/agent/*.md` — project-level agent definitions
2. `~/.config/opencode/agent/*.md` — user-level agents
3. Built-in default agent

**Built-in custom agents** (in `.opencode/agent/`):
- `triage.md` — GitHub issue triage agent
- `duplicate-pr.md` — Duplicate PR detection agent

**Generate an agent dynamically:**
```bash
opencode agent generate "An agent that reviews TypeScript for Effect anti-patterns"
```

### 2.6 Built-in Tools

All tools live in `packages/opencode/src/tool/`:

| Tool | ID | Description |
|---|---|---|
| `shell.ts` | `bash` | Execute shell commands (bash/pwsh/cmd) |
| `read.ts` | `read` | Read files/directories with offset/limit |
| `write.ts` | `write` | Write file content |
| `edit.ts` | `edit` | Edit files (diff-based, exact string replacement) |
| `glob.ts` | `glob` | Glob file pattern matching |
| `grep.ts` | `grep` | Ripgrep-powered content search |
| `webfetch.ts` | `webfetch` | HTTP fetch with content extraction |
| `websearch.ts` | `websearch` | Web search |
| `apply_patch.ts` | `apply_patch` | Apply unified diff patches |
| `task.ts` | `task` | Spawn subagent sessions (foreground/background) |
| `question.ts` | `question` | Ask the user a question |
| `todo.ts` | `todowrite` | Write to todo list |
| `skill.ts` | `skill` | Load/invoke skills |
| `lsp.ts` | `lsp` | LSP diagnostics |
| `plan.ts` | — | Enter/exit plan mode |
| `code-mode.ts` | — | Confined code execution (Acorn AST) |

### 2.7 MCP Integration

`packages/opencode/src/mcp/index.ts` — full MCP client:

**Supported transports:**
- `StdioClientTransport` — launch a local subprocess (most common)
- `SSEClientTransport` — connect to a remote SSE endpoint
- `StreamableHTTPClientTransport` — connect to a Streamable HTTP endpoint

**Config example** (in `opencode.jsonc`):
```jsonc
{
  "mcp": {
    "servers": {
      "my-local-mcp": {
        "type": "local",
        "command": ["node", "/path/to/mcp-server.js"],
        "environment": { "API_KEY": "..." }
      },
      "my-remote-mcp": {
        "type": "remote",
        "url": "https://my-mcp.example.com/mcp",
        "headers": { "Authorization": "Bearer ..." }
      }
    }
  }
}
```

**OAuth support:** Remote MCP servers can use OAuth via `McpOAuthProvider`.

### 2.8 ACP (Agent Client Protocol)

`packages/opencode/src/acp/agent.ts` implements the full ACP `Agent` interface:

```typescript
// Full ACP surface implemented:
initialize, authenticate, newSession, loadSession, listSessions,
resumeSession, closeSession, forkSession, setSessionConfigOption,
setSessionMode, setSessionModel, prompt, cancel
```

Start ACP server:
```bash
opencode acp
```

This enables VS Code (and other ACP-compatible clients) to connect directly.

### 2.9 LLM Providers

`packages/llm/src/providers/`:

| Provider | Notes |
|---|---|
| `anthropic.ts` | Claude models via Anthropic API |
| `amazon-bedrock.ts` | Claude + other models via AWS Bedrock |
| `azure.ts` | OpenAI models via Azure OpenAI Service |
| `cloudflare.ts` | Cloudflare AI Gateway + Workers AI |
| `github-copilot.ts` | Models via GitHub Copilot auth |
| `google.ts` | Gemini via Google AI |
| `openai.ts` | GPT models via OpenAI API |
| `openai-compatible.ts` | Ollama, LM Studio, any OpenAI-compatible endpoint |
| `openrouter.ts` | 200+ models via OpenRouter |
| `xai.ts` | Grok models via xAI |

### 2.10 Plugin System

`packages/opencode/src/plugin/index.ts`:

**Built-in auth plugins:**
- `CodexAuth` (OpenAI), `CopilotAuth` (GitHub), `Modal`, `Gitlab`, `Poe`, `CloudflareAIGateway`, `CloudflareWorkers`, `Azure`, `DigitalOcean`, `XAI`, `SnowflakeCortex`

**External plugins** are npm packages exporting a plugin using `@opencode-ai/plugin`:
```typescript
// packages/plugin/src/index.ts exports:
import { tool, tui } from "@opencode-ai/plugin"
```

**Install a plugin:**
```bash
opencode plug @my-org/my-opencode-plugin
```

### 2.11 Skill System

Skills are Markdown files (`SKILL.md`) discovered in:
- `.opencode/skills/<name>/SKILL.md`
- `.agents/skills/<name>/SKILL.md`
- `skill/SKILL.md` or `skills/<name>/SKILL.md` in the project root

They inject structured knowledge into agent context. Example (`.opencode/skills/effect/SKILL.md`):
```markdown
# Effect Framework
Use generators with Effect.gen(...) instead of .pipe() for sequential operations.
...
```

### 2.12 Server Routes

`packages/opencode/src/server/routes/instance/httpapi/` — route groups:

`config`, `control`, `control-plane`, `event`, `experimental`, `file`, `global`, `instance`, `mcp`, `metadata`, `permission`, `project`, `project-copy`, `provider`, `pty`, `query`, `question`, `session`, `sync`, `tui`, `workspace`

---

## 3. How to Make Changes

### 3.1 Prerequisites

```bash
# Install Bun 1.3.14+
curl -fsSL https://bun.sh/install | bash

# Clone and install deps
git clone https://github.com/anomalyco/opencode
cd opencode
bun install

# Build
bun run build  # or: turbo build
```

### 3.2 Modifying an Existing Agent

Agent definitions live as Markdown files with a YAML frontmatter-like header:

```bash
# Project-level agents
.opencode/agent/<name>.md

# User-level agents
~/.config/opencode/agent/<name>.md
```

**Example agent file:**
```markdown
---
name: my-reviewer
description: Reviews PRs for security issues
mode: primary
model: claude-opus-4-8
temperature: 0.2
permission:
  - allow: read
  - allow: bash
    description: git commands only
    patterns: ["git *"]
---

You are a security-focused code reviewer. Always check for:
- SQL injection
- XSS vulnerabilities
- Secrets in code
```

Then reference it: `opencode --agent my-reviewer`

### 3.3 Modifying Core Agent Logic

Core agent source: `packages/opencode/src/agent/agent.ts`

After editing, the agent schema is in `packages/schema/src/` — update schemas there first if you're adding new fields, then update `agent.ts`.

**Dependency direction:**
```
Schema → Core/Protocol → Server
Client depends on Schema/Protocol only
sdk-next composes all
```

### 3.4 Adding a New CLI Command

1. Create `packages/opencode/src/cli/cmd/mycommand.ts`:
```typescript
import { Cmd } from "./cmd.js"
export const MyCommand = Cmd.make("mycommand", {
  describe: "What it does",
  handler: (args) => Effect.gen(function*() {
    // implementation
  })
})
```

2. Register in `packages/opencode/src/index.ts`:
```typescript
import { MyCommand } from "./cli/cmd/mycommand.js"
yargs.command(MyCommand)
```

### 3.5 Adding a New Built-in Tool

1. Create `packages/opencode/src/tool/mytool.ts`:
```typescript
import { Tool } from "../tool.js"
export const MyTool = Tool.define({
  id: "mytool",
  description: "...",
  parameters: Schema.Struct({ ... }),
  execute: (params) => Effect.gen(function*() {
    // implementation — use Effect, not try/catch
  })
})
```

2. Register in `packages/opencode/src/tool/index.ts`.

### 3.6 Adding a New MCP Server

No code changes needed — add to config:
```jsonc
// .opencode/opencode.jsonc or ~/.config/opencode/opencode.jsonc
{
  "mcp": {
    "servers": {
      "my-new-server": {
        "type": "local",
        "command": ["npx", "-y", "@my-org/my-mcp-server"],
        "environment": { "TOKEN": "..." }
      }
    }
  }
}
```

Or via CLI: `opencode mcp add`

### 3.7 Adding a New LLM Provider

1. Create `packages/llm/src/providers/myprovider.ts` following the pattern in `anthropic.ts`.
2. Export from `packages/llm/src/providers/index.ts`.
3. Register in the provider registry in `packages/core/src/config.ts`.

### 3.8 Writing a Plugin (External npm Package)

```typescript
// my-opencode-plugin/src/index.ts
import { tool } from "@opencode-ai/plugin"

export default {
  tools: [
    tool({
      id: "my-custom-tool",
      description: "Does something useful",
      parameters: { /* JSON Schema */ },
      execute: async (params) => {
        // implementation
        return { result: "..." }
      }
    })
  ]
}
```

Publish to npm, then: `opencode plug my-opencode-plugin`

### 3.9 Writing a Skill

Create `.opencode/skills/my-skill/SKILL.md`:
```markdown
# My Skill Name
Brief description of when to use this skill.

## Usage
Detailed instructions, examples, code patterns...
```

The agent will discover and use it automatically.

### 3.10 After Making Schema/Protocol Changes

```bash
# If you changed packages/protocol or packages/server HttpApi:
cd packages/client && bun run generate

# If you changed packages/sdk source:
./packages/sdk/js/script/build.ts

# Re-run codegen across the monorepo:
bun run generate
```

### 3.11 Best Practices (from AGENTS.md + CONTRIBUTING.md)

- No `try/catch` — use Effect's error channels
- No `any` type — be explicit
- No star imports (`import * as X`) — use named namespace exports
- Prefer `const` over `let`; ternary over reassignment
- Early returns instead of `else` blocks
- Use `Bun.file()` instead of Node's `fs`
- Bind Effect services before calling methods in generators
- Inline single-use values — reduce variable count
- Drizzle ORM: use snake_case for SQLite column names
- Test from the package directory, not monorepo root
- Type-check with `bun typecheck` from package dir only

---

## 4. Debugging Guide

### 4.1 Debug Mode

```bash
# Run with verbose debug output
opencode --debug --print-logs

# Or set env variable
OPENCODE_DEBUG=1 opencode run "fix the bug"

# Debug a specific subsystem
opencode --debug --print-logs serve
```

### 4.2 Built-in Debug Commands

```bash
opencode debug          # Enter debug utilities menu
opencode stats          # Show usage statistics
opencode db             # Database operations (migrate, inspect SQLite)
```

### 4.3 Common Issues

**Dependency errors on install:**
```bash
# Force clean install
rm bun.lock
bun install --force

# If a patch fails to apply:
bun patch --commit packages/patches/<package>.patch
```

**Stream errors (LLM responses cut off):**
- Check that `packages/patches/@ai-sdk/*.patch` is applied (run `bun install`)
- Verify provider API key is valid
- Try a different model: `opencode --model claude-sonnet-5 run "test"`

**Agent misrouting (wrong agent responds):**
- Check `default_agent` in config: `opencode config get default_agent`
- List available agents: `opencode agent list`
- Explicitly specify: `opencode --agent <name>`

**MCP server not connecting:**
```bash
# List configured MCP servers
opencode mcp list

# Check MCP server logs
opencode --debug --print-logs
# Look for lines with "MCP" in the debug output

# Test a specific MCP server manually
node /path/to/mcp-server.js  # should start without errors
```

**Permission denied on tool use:**
```bash
# Check permission config
opencode config get permissions

# Temporarily allow all (dev only)
opencode config set permissions '[]'
```

### 4.4 Tracing Agent Behavior

```bash
# Run with trace output — shows each tool call and its result
opencode run --trace "my prompt"

# Attach to a running server and see live events
opencode attach http://localhost:4001

# View session history
opencode session list
opencode session show <session-id>
```

### 4.5 MCP/ACP Log Tracing

```bash
# MCP debug: set environment variable before starting
MCP_LOG_LEVEL=debug opencode --debug --print-logs serve

# ACP debug:
opencode acp --debug
```

### 4.6 Database Inspection

```bash
# OpenCode stores state in SQLite at ~/.local/share/opencode/
opencode db              # interactive database menu
ls ~/.local/share/opencode/  # raw SQLite files
```

### 4.7 TypeScript Type Errors

```bash
# Always type-check from the specific package
cd packages/core && bun typecheck
cd packages/opencode && bun typecheck

# Never run typecheck from monorepo root — it will fail
```

---

## 5. Deployment & Testing

### 5.1 Installation Methods

**npm (recommended):**
```bash
npm install -g opencode@latest
```

**Bun:**
```bash
bun install -g opencode@latest
```

**Homebrew (macOS/Linux):**
```bash
brew install anomalyco/tap/opencode
```

**Scoop (Windows):**
```bash
scoop bucket add anomalyco https://github.com/anomalyco/scoop
scoop install opencode
```

**Pacman (Arch Linux):**
```bash
yay -S opencode-bin
```

**Nix (NixOS/nix):**
```bash
# From flake
nix profile install github:anomalyco/opencode#opencode

# Or add to flake.nix
inputs.opencode.url = "github:anomalyco/opencode";
# Then use: inputs.opencode.packages.${system}.opencode
```

**Shell installer:**
```bash
curl -fsSL https://opencode.ai/install | bash
```

### 5.2 Running Locally (Development)

```bash
# Start the full dev stack (TUI + server)
bun run dev

# Start just the desktop app
bun run dev:desktop

# Start just the web UI
bun run dev:web

# Start the console (admin UI)
bun run dev:console

# Start headless server only
opencode serve

# Start server + web interface
opencode web

# Start ACP server for IDE integration
opencode acp
```

### 5.3 Testing Agents

```bash
# Run a prompt non-interactively
opencode run "Summarize the changes in the last 5 commits"

# Run with a specific agent
opencode run --agent triage "Fix this issue: #1234"

# Run with a specific model
opencode run --model anthropic/claude-opus-4-8 "Refactor this function"

# Attach TUI to a remote running server
opencode attach http://remote-server:4001
```

### 5.4 Containerized Deployment

```bash
# Build the base container
docker build -t opencode-base packages/containers/base/

# Build the bun+node container
docker build \
  --build-arg NODE_VERSION=24.4.0 \
  --build-arg BUN_VERSION=1.3.14 \
  -t opencode-runtime packages/containers/bun-node/

# Run opencode inside container
docker run -it \
  -e ANTHROPIC_API_KEY=sk-... \
  -v $(pwd):/workspace \
  opencode-runtime \
  opencode run "fix the bug"
```

### 5.5 Nix Deployment

```bash
# Build the opencode package
nix build .#opencode

# Build with desktop app
nix build .#opencode-desktop

# Enter dev shell
nix develop
```

### 5.6 Validating Changes

```bash
# Run tests from a specific package
cd packages/core && bun test
cd packages/opencode && bun test

# Run all tests via turbo
bun run test

# Type-check a package
cd packages/core && bun typecheck

# Run the linter
bunx oxlint .
```

### 5.7 Plan Before Production

```bash
# Use plan mode to review before executing
opencode run --plan "Refactor the auth module"
# Agent shows plan → you approve → agent executes
```

---

## 6. Practical Examples

### 6.1 Example: Edit an Agent → Debug → Test → Deploy

```bash
# 1. Create or edit an agent
cat > .opencode/agent/my-agent.md << 'EOF'
---
name: my-agent
description: Explains TypeScript errors in plain English
mode: primary
model: claude-sonnet-5
temperature: 0.3
---
You explain TypeScript type errors in simple, beginner-friendly language.
EOF

# 2. Run with debug mode to see what's happening
opencode --debug --print-logs run --agent my-agent "Explain this error: Type 'string' is not assignable to 'number'"

# 3. Check it works non-interactively
opencode run --agent my-agent "Explain: Property 'foo' does not exist on type 'Bar'"

# 4. Test in interactive mode
opencode  # opens TUI — switch agent with /agent my-agent

# 5. For team sharing: commit the .opencode/agent/ directory
git add .opencode/agent/my-agent.md
git commit -m "feat(core): add TypeScript error explainer agent"
```

### 6.2 Example: Adding a Plugin

```bash
# 1. Install the plugin
opencode plug @my-org/opencode-jira-plugin

# 2. Verify it's loaded
opencode plug list

# 3. Use a tool the plugin provides
opencode run "Create a JIRA ticket for the bug I just fixed"

# 4. To remove
opencode plug remove @my-org/opencode-jira-plugin
```

### 6.3 Example: Configuring an MCP Server

```bash
# 1. Add an MCP server via CLI
opencode mcp add

# Or manually edit config
cat >> ~/.config/opencode/opencode.jsonc << 'EOF'
{
  "mcp": {
    "servers": {
      "filesystem": {
        "type": "local",
        "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"]
      }
    }
  }
}
EOF

# 2. Restart opencode and verify
opencode mcp list

# 3. Test the MCP tools
opencode run "List all Python files in my projects folder"
```

### 6.4 Example: Using Plan Mode Safely

```bash
# Run with planning before execution
opencode run --plan "Delete all console.log statements from the codebase"

# Agent will show:
# Plan:
#   1. Find all files with console.log (grep)
#   2. List them for review
#   3. Edit each file to remove statements
# Proceed? [y/N]
```

---

## 7. Next Steps

### 7.1 Extending OpenCode

**New Agent Types:**
- Create specialized agents for your stack (e.g., a Rust safety reviewer, a SQL query optimizer)
- Build multi-step agents using the `task` tool to spawn subagents
- Combine with MCP servers for domain-specific data access

**New Integrations:**
- Add a new LLM provider in `packages/llm/src/providers/`
- Add a Slack bot using `packages/slack/` as a reference
- Build a GitHub Action using `.github/workflows/opencode.yml` as a model

**New Plugins:**
- The `@opencode-ai/plugin` API allows adding custom tools, TUI components, and workspace adapters
- Look at `packages/plugin/src/` for the full public API

**MCP Ecosystem:**
- Any MCP-compatible server works with OpenCode — Postgres, GitHub, filesystem, web search, etc.
- See [modelcontextprotocol.io](https://modelcontextprotocol.io) for the server registry

### 7.2 Community

- **Discord:** Join via the link in the README or website
- **X.com:** Follow `@anomalyco` for updates
- **GitHub Discussions:** [github.com/anomalyco/opencode/discussions](https://github.com/anomalyco/opencode/discussions)
- **Issues:** [github.com/anomalyco/opencode/issues](https://github.com/anomalyco/opencode/issues)

### 7.3 Contributing

1. Fork and clone the repo
2. Create a branch: max 3 words, hyphen-separated (e.g., `fix-mcp-timeout`)
3. Make changes following AGENTS.md style guide
4. Test: `cd packages/<pkg> && bun test`
5. Type-check: `cd packages/<pkg> && bun typecheck`
6. Commit: `feat(core): add new feature`
7. Open PR against `dev` branch (not `main`)

---

## 8. Topics to Learn

To contribute effectively to OpenCode, a developer should understand:

### Core Language & Runtime
- **TypeScript** (advanced: generics, conditional types, mapped types, template literals)
- **Bun** runtime (APIs, package manager, test runner, bundler)
- **JavaScript ESModules** (dynamic imports, tree shaking)

### Functional Programming / Effect
- **Effect v4** — this is the most critical library in the codebase
  - `Effect.gen` generators for sequential async code
  - `Layer` / dependency injection
  - `Schema` for runtime validation
  - `Stream` for streaming data
  - `Fiber` for concurrency
  - Error channels (`Effect<A, E, R>`)
  - `Context` / `Tag` for service definitions
- **Functional programming patterns** (composition, pipelines, algebraic data types)

### UI Frameworks
- **SolidJS** — used for TUI, web UI, and desktop UI
- **opentui** — custom terminal rendering layer built on SolidJS
- **Vite** — build tool for web/desktop
- **Electron** — desktop app wrapper
- **Astro + Starlight** — docs site

### Infrastructure & DevOps
- **SST v4** — cloud infrastructure as code
- **Cloudflare Workers** — edge functions
- **AWS** (basic: Lambda, SSM, us-east-1 deployment)
- **GitHub Actions** — CI/CD (25 workflow files)
- **Nix/NixOS** — reproducible builds
- **Docker** — containerization

### Protocols & Standards
- **MCP (Model Context Protocol)** — tool/resource extension protocol
- **ACP (Agent Client Protocol)** — IDE integration protocol
- **OpenAPI / REST** — server API surface
- **WebSockets** — event streaming
- **LSP (Language Server Protocol)** — editor integrations

### Data & Storage
- **SQLite** — local state storage
- **Drizzle ORM** — Effect-native ORM
- **Effect-drizzle-sqlite** / **Effect-sqlite-node** — custom Effect bindings

### AI / LLM
- **Vercel AI SDK (v6)** — LLM call abstraction
- **Anthropic API** — primary LLM provider
- **Model Context Protocol** — tool definitions
- **Prompt engineering** — writing effective agent prompts
- **Context window management** — compaction strategies

### Tooling
- **Turborepo** — monorepo task orchestration
- **Oxlint** — Rust-based linter
- **Prettier** — code formatter
- **Conventional Commits** — commit message standard
- **Husky** — git hooks

---

## 9. Missing Features

Based on analysis of the codebase, specs, and architecture, these features appear to be absent or incomplete:

### Observability & Monitoring
- No built-in token usage dashboard (tracked externally via Honeycomb)
- No per-session cost tracking visible to the user
- No built-in session replay beyond raw export

### Agent Capabilities
- No native multi-agent coordination (beyond spawning subagents via `task` tool)
- No persistent agent memory across sessions (only in-session context)
- No agent-to-agent communication channels
- No visual agent flow builder/editor

### MCP & Integration
- No MCP server marketplace / discovery UI inside the tool
- No built-in MCP server hosting (only client)
- No native database connectors (must use MCP)
- No native HTTP API call tool (requires MCP or webfetch workaround)

### UI/UX
- No mobile client (iOS/Android)
- No collaborative/shared sessions in real time (share link is read-only)
- No built-in diff viewer for large changesets
- No session search/semantic memory

### Security & Enterprise
- No SSO/SAML integration in open-source tier
- No audit log export (only internal Honeycomb tracing)
- No secrets management beyond environment variables
- No fine-grained RBAC for enterprise teams

### Developer Experience
- No hot-reload for agent definitions in production mode
- No built-in profiler for tool execution times
- No automated snapshot testing for TUI components
- Python SDK exists but is not in the main monorepo (`publish-python-sdk.yml` workflow suggests it's external)
- No official Java/Go/Rust SDK

### Deployment
- No one-click cloud deployment (e.g., Railway, Render, Fly.io templates)
- No Kubernetes Helm chart
- No official ARM64 Docker image

---

*This document was generated by analyzing the live repository structure, source files, and configuration. For the authoritative developer guide, see `CONTRIBUTING.md` and `AGENTS.md` in the repository root.*
