# FORIA — AI Agent Project Reference for OpenCode

> This document is written for an AI assistant (you) to quickly understand the OpenCode codebase and assist developers effectively. Read this before answering any code question about this project.

---

## WHO YOU ARE HELPING

You are assisting a developer working on **OpenCode** — an open-source terminal AI coding assistant. The project is a Bun monorepo with 34 TypeScript packages. The entire codebase uses the **Effect v4** functional programming library. If you are not familiar with Effect, treat it as "typed async with dependency injection" — `Effect<A, E, R>` means "produces A, can fail with E, requires services R."

---

## CRITICAL FACTS TO REMEMBER

1. **Runtime is Bun, not Node.js.** Use `Bun.file()`, not `fs.readFile()`. Run with `bun`, not `node`.
2. **Default branch is `dev`**, not `main`.
3. **No `try/catch`** anywhere in core code — use Effect error channels.
4. **No `any`** types.
5. **No star imports** (`import * as X from "..."`) — use named namespace exports.
6. **Effect generators** bind services before calling methods: `const svc = yield* ServiceTag; yield* svc.method()`.
7. **Type-check from package dirs only**: `cd packages/core && bun typecheck` — never from root.
8. **Tests from package dirs only**: `cd packages/core && bun test` — never from root.
9. **After protocol/server changes**: run `cd packages/client && bun run generate`.
10. **After SDK source changes**: run `./packages/sdk/js/script/build.ts`.

---

## MONOREPO MAP

```
packages/
  opencode/        ← MAIN: CLI binary, server, TUI, ACP, MCP client, tools, agents, plugins, skills
  core/            ← Business logic: sessions, config, git, permission, snapshot, LSP, database, events
  llm/             ← LLM provider adapters (Anthropic, OpenAI, Google, Bedrock, Azure, etc.)
  tui/             ← Terminal UI framework (SolidJS + opentui)
  app/             ← Shared web/desktop UI (SolidJS + Vite)
  desktop/         ← Electron desktop app
  plugin/          ← Public plugin API (@opencode-ai/plugin) for external plugin authors
  schema/          ← Shared Effect schemas (agents, sessions, permissions, prompts)
  protocol/        ← Shared protocol types (client↔server communication)
  client/          ← HTTP + Effect client, generated from protocol
  sdk/             ← Public TypeScript SDK (+ openapi.json)
  sdk-next/        ← Next-gen SDK composing client+core+server
  server/          ← Server-side: auth, cors, routes, PTY env
  session-ui/      ← Shared SolidJS session UI components
  console/         ← Web console admin frontend
  docs/            ← Astro + Starlight documentation website
  enterprise/      ← Enterprise Cloudflare Worker + UI
  function/        ← Cloudflare Worker functions (GitHub App auth)
  slack/           ← Slack bot integration
  ui/              ← Shared SolidJS component library (published to npm)
  ...              ← 14 more support packages
```

**Dependency direction (strict — never violate):**
```
Schema → Core/Protocol → Server
Client depends on Schema/Protocol only
sdk-next composes all of them
```

---

## THE `packages/opencode` PACKAGE IN DETAIL

This is the main runnable package. Entry point: `src/index.ts` (yargs CLI).

### CLI Commands

| Command | File | Purpose |
|---|---|---|
| `run [prompt]` | `src/cli/cmd/run.ts` | Non-interactive or mini interactive prompt execution |
| `serve` | `src/cli/cmd/serve.ts` | Headless HTTP API server |
| `web` | `src/cli/cmd/web.ts` | Server + web interface |
| `acp` | `src/cli/cmd/acp.ts` | ACP server for IDE integration |
| `mcp` | `src/cli/cmd/mcp.ts` | MCP sub-commands (add/list/remove) |
| `agent` | `src/cli/cmd/agent.ts` | Agent management |
| `session` | `src/cli/cmd/session.ts` | Session management |
| `models`/`providers` | respective files | List available models/providers |
| `plug` | `src/cli/cmd/plug.ts` | Plugin management |
| `attach <url>` | `src/cli/cmd/attach.ts` | Attach TUI to remote running server |
| `debug` | `src/cli/cmd/debug/` | Debug utilities |
| `stats` | `src/cli/cmd/stats.ts` | Usage statistics |

### Source Sub-directories

| Path | Contents |
|---|---|
| `src/agent/` | `agent.ts` — Agent schema (Info, GeneratedAgent) + `Interface` service |
| `src/acp/` | Full ACP implementation: `agent.ts`, `service.ts`, `session.ts`, etc. |
| `src/mcp/` | MCP client: `index.ts` — stdio/SSE/StreamableHTTP transports, OAuth |
| `src/tool/` | Built-in tools: `shell.ts`, `read.ts`, `write.ts`, `edit.ts`, `glob.ts`, `grep.ts`, `webfetch.ts`, `websearch.ts`, `apply_patch.ts`, `task.ts`, `question.ts`, `todo.ts`, `skill.ts`, `lsp.ts`, `plan.ts`, `code-mode.ts` |
| `src/plugin/` | Plugin loader: built-in auth plugins + external plugin loading |
| `src/skill/` | Skill discovery: finds `SKILL.md` files in project/user dirs |
| `src/server/` | HTTP server: `server.ts` (Effect HttpServer + Node HTTP), `routes/` |
| `src/cli/cmd/run/` | `run` command internals: boot, lifecycle, queue, stdin, stream, trace, tool, turn-summary, permission/question/subagent footer components |

---

## AGENT SCHEMA

Defined in `packages/opencode/src/agent/agent.ts`:

```typescript
const Info = Schema.Struct({
  name: Schema.String,
  description: Schema.String,
  mode: Schema.Literal("subagent", "primary", "all"),
  permission: Schema.Array(PermissionRule),    // tool access rules
  model: Schema.optional(Schema.String),       // model override
  temperature: Schema.optional(Schema.Number),
  topP: Schema.optional(Schema.Number),
  color: Schema.optional(Schema.String),
  prompt: Schema.optional(Schema.String),      // system prompt override/append
  steps: Schema.optional(Schema.Number),       // max reasoning steps
})
```

**Agent discovery order (highest priority first):**
1. `.opencode/agent/<name>.md` — project-level
2. `~/.config/opencode/agent/<name>.md` — user-level
3. Built-in default agent

---

## CONFIG SCHEMA

Defined in `packages/core/src/config.ts`. The user-facing config file is `opencode.jsonc`.

Key top-level fields:
```typescript
{
  shell: string,                    // default shell
  model: string,                    // default model (e.g. "anthropic/claude-sonnet-5")
  default_agent: string,            // default primary agent name
  autoupdate: true | false | "notify",
  share: "manual" | "auto" | "disabled",
  enterprise: { url: string },
  username: string,
  permissions: PermissionRule[],    // ordered tool permission ruleset
  agents: Record<string, AgentInfo>,
  snapshots: boolean,
  watcher: WatcherConfig,
  formatter: FormatterConfig,
  lsp: LspConfig,
  attachments: AttachmentConfig,
  mcp: { servers: Record<string, McpServerConfig> },
  references: Record<string, ReferenceConfig>,
  provider: ProviderConfig,
  command: Record<string, CommandConfig>,
  compaction: CompactionConfig,
  experimental: ExperimentalFlags,
  plugin: PluginConfig,
}
```

**MCP server config types:**
```typescript
// Local (subprocess):
{ type: "local", command: string[], cwd?: string, environment?: Record<string,string>, disabled?: boolean, timeout?: number }

// Remote (HTTP/SSE):
{ type: "remote", url: string, headers?: Record<string,string>, oauth?: OAuth | false, disabled?: boolean, timeout?: number }
```

---

## THE V2 SESSION SYSTEM

Defined in `packages/core/src/session/` (complex — key concepts):

- **Prompt admission**: A message from the user enters a queue. The session runner processes one prompt at a time.
- **Session runner**: Owns the agent loop — calls LLM, dispatches tool calls, streams responses, emits `EventV2` events.
- **EventV2 replay**: Session history is stored as events; replaying them reconstructs session state.
- **Compaction**: When context gets too large, earlier messages are summarized and truncated.
- **Snapshots**: Before each tool invocation that writes files, a snapshot is taken (enabling undo/revert).
- **Subagents**: The `task` tool forks a new session with its own runner.

---

## LLM PROVIDERS

All in `packages/llm/src/providers/`. Each exports a provider object conforming to the Vercel AI SDK interface plus OpenCode-specific metadata.

| Module | Provider | Auth method |
|---|---|---|
| `anthropic.ts` | Anthropic | `ANTHROPIC_API_KEY` |
| `amazon-bedrock.ts` | AWS Bedrock | AWS credentials |
| `azure.ts` | Azure OpenAI | `AZURE_API_KEY` + endpoint |
| `cloudflare.ts` | Cloudflare AI Gateway + Workers AI | `CF_API_TOKEN` |
| `github-copilot.ts` | GitHub Copilot | GitHub OAuth |
| `google.ts` | Google Gemini | `GOOGLE_API_KEY` |
| `openai.ts` | OpenAI | `OPENAI_API_KEY` |
| `openai-compatible.ts` | Ollama/LM Studio/any OpenAI-compat | Custom base URL |
| `openrouter.ts` | OpenRouter | `OPENROUTER_API_KEY` |
| `xai.ts` | xAI Grok | `XAI_API_KEY` |

---

## TOOL SYSTEM

Tools are defined with `Tool.define(...)`. Each tool has:
- `id` — string identifier used in prompts
- `description` — shown to the LLM
- `parameters` — Effect Schema for input validation
- `execute` — `(params) => Effect<result, error, services>`

Permissions are checked before execution. The `permission` field in agent config is an ordered array of rules (`allow`/`deny`) matched against tool IDs.

---

## MCP CLIENT

`packages/opencode/src/mcp/index.ts`. Uses `@modelcontextprotocol/sdk` (with a large patch at `patches/@modelcontextprotocol/sdk@1.29.0.patch`).

**Transport selection** based on config:
- `type: "local"` → `StdioClientTransport` (spawns subprocess)
- `type: "remote"` with SSE URL → `SSEClientTransport`
- `type: "remote"` with Streamable HTTP → `StreamableHTTPClientTransport`

**Capabilities registered**: `roots` (filesystem roots for MCP servers to reference).

**OAuth flow**: Remote MCP servers can authenticate via `McpOAuthProvider` / `McpOAuthCallback`.

---

## ACP (AGENT CLIENT PROTOCOL)

`packages/opencode/src/acp/`. Implements `@agentclientprotocol/sdk`'s `Agent` interface.

Full surface implemented:
```
initialize, authenticate, newSession, loadSession, listSessions,
resumeSession, closeSession, forkSession, setSessionConfigOption,
setSessionMode, setSessionModel, prompt, cancel
```

Start: `opencode acp` — VS Code extension and other ACP clients connect to this.

---

## PLUGIN SYSTEM

`packages/opencode/src/plugin/index.ts`.

**Built-in auth plugins** (handle OAuth/token flows for providers):
`CodexAuth`, `CopilotAuth`, `Modal`, `Gitlab`, `Poe`, `CloudflareAIGateway`, `CloudflareWorkers`, `Azure`, `DigitalOcean`, `XAI`, `SnowflakeCortex`

**External plugins** are npm packages. Install: `opencode plug <package-name>`.

**Plugin API** (`@opencode-ai/plugin`): exports `tool`, `tui` — allows adding custom tools and TUI components.

---

## SKILL SYSTEM

`packages/opencode/src/skill/index.ts`.

Skills are `SKILL.md` files discovered in:
- `.opencode/skills/<name>/SKILL.md`
- `.agents/skills/<name>/SKILL.md`
- `skill/SKILL.md` or `skills/<name>/SKILL.md`

A skill injects structured knowledge/instructions into the agent's context when invoked with the `skill` tool. The built-in `customize-opencode` skill explains OpenCode's own config syntax.

---

## SERVER ROUTES

`packages/opencode/src/server/routes/instance/httpapi/`

Route groups: `config`, `control`, `control-plane`, `event`, `experimental`, `file`, `global`, `instance`, `mcp`, `metadata`, `permission`, `project`, `project-copy`, `provider`, `pty`, `query`, `question`, `session`, `sync`, `tui`, `workspace`

The server uses **Effect HttpApi** — routes are defined with typed request/response schemas. OpenAPI spec is auto-generated via `server.openapi()`.

---

## INFRASTRUCTURE

SST v4 app. Home: Cloudflare. Regions: AWS us-east-1.

| Infra module | File | Purpose |
|---|---|---|
| App | `infra/app.ts` | Main app resources |
| Console | `infra/console.ts` | Web console |
| Enterprise | `infra/enterprise.ts` | Enterprise sharing service |
| Lake | `infra/lake.ts` | Data lake / analytics ingestion |
| Monitoring | `infra/monitoring.ts` | Honeycomb observability |
| Secrets | `infra/secret.ts` | SSM/secret management |
| Stage | `infra/stage.ts` | Stage flags (deployAws, awsStage) |
| Stats | `infra/stats.ts` | Usage statistics service |

---

## HOW TO ANSWER CODE QUESTIONS

### "Where is X defined?"

| X | Where to look |
|---|---|
| CLI commands | `packages/opencode/src/cli/cmd/` |
| Agent schema/logic | `packages/opencode/src/agent/agent.ts` |
| Config schema | `packages/core/src/config.ts` |
| Tools | `packages/opencode/src/tool/` and `packages/core/src/tool/` |
| LLM providers | `packages/llm/src/providers/` |
| MCP client | `packages/opencode/src/mcp/index.ts` |
| ACP server | `packages/opencode/src/acp/` |
| Session logic | `packages/core/src/session/` |
| Plugin loader | `packages/opencode/src/plugin/index.ts` |
| Skill system | `packages/opencode/src/skill/index.ts` |
| HTTP server | `packages/opencode/src/server/server.ts` |
| API routes | `packages/opencode/src/server/routes/` |
| Event types | `packages/core/src/event.ts` |
| Git integration | `packages/core/src/git.ts` |
| Permissions | `packages/core/src/permission.ts` |
| Database schema | `packages/core/src/database/` |
| Shared schemas | `packages/schema/src/` |
| Protocol types | `packages/protocol/src/` |
| Public SDK | `packages/sdk/` |
| TUI components | `packages/tui/src/` |
| Web UI | `packages/app/src/` |
| Desktop (Electron) | `packages/desktop/` |

### "How do I add X?"

**New agent**: Create `.opencode/agent/<name>.md` with YAML frontmatter.

**New CLI command**: Add `packages/opencode/src/cli/cmd/<name>.ts`, register in `packages/opencode/src/index.ts`.

**New built-in tool**: Add `packages/opencode/src/tool/<name>.ts` with `Tool.define(...)`, register in tool index.

**New LLM provider**: Add `packages/llm/src/providers/<name>.ts`, export from providers index, register in config.

**New MCP server**: Add to `opencode.jsonc` under `mcp.servers` — no code changes needed.

**External plugin**: Create npm package using `@opencode-ai/plugin`, publish, install with `opencode plug`.

**New skill**: Create `SKILL.md` in `.opencode/skills/<name>/SKILL.md` — no code changes needed.

**New server route**: Add route file in `packages/opencode/src/server/routes/instance/httpapi/`, register in route group, then run `cd packages/client && bun run generate`.

### "How do I debug X?"

```bash
opencode --debug --print-logs [command]   # verbose debug output
opencode debug                            # debug utilities menu
opencode stats                            # usage statistics
opencode db                               # SQLite database inspection
opencode session list                     # list sessions
opencode attach http://localhost:4001     # attach TUI to running server
cd packages/<pkg> && bun typecheck        # TypeScript errors
cd packages/<pkg> && bun test             # run tests
```

---

## CODING STYLE (enforce when writing code)

```typescript
// ✅ Effect generators — bind services first
const result = yield* Effect.gen(function* () {
  const db = yield* Database
  const session = yield* db.getSession(id)
  return session
})

// ✅ Early returns, no else
if (!user) return yield* Effect.fail(new NotFoundError())
return yield* doWork(user)

// ✅ Bun file API
const content = await Bun.file("./path").text()

// ✅ Named namespace export
export * as MyModule from "./my-module.js"

// ✅ Drizzle snake_case columns
const users = sqliteTable("users", {
  created_at: integer("created_at"),
})

// ❌ No try/catch
// ❌ No any
// ❌ No star imports (import * as X)
// ❌ No else blocks (use early returns)
// ❌ No Node.js fs module
```

---

## COMMIT CONVENTION

Format: `type(scope): summary`

Valid types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`

Valid scopes: `core`, `opencode`, `tui`, `app`, `desktop`, `sdk`, `plugin`

Branch names: max 3 words, hyphen-separated, no type prefixes.

Example: `feat(core): add snapshot rollback command` on branch `snapshot-rollback-command`

---

## PATCHES

The repo maintains 16 patches in `patches/` applied during `bun install`. Notable ones:

| Patch | Why |
|---|---|
| `@modelcontextprotocol/sdk@1.29.0.patch` (33 KB) | Significant MCP SDK customization |
| `effect@4.0.0-beta.83.patch` | Effect v4 beta fixes |
| `@ai-sdk/google.patch`, `@ai-sdk/openai.patch`, etc. | AI SDK stream/response fixes |
| `solid-js@1.9.10.patch` | SolidJS fixes for TUI context |

If a dependency behaves unexpectedly, check if it has a patch applied.

---

## PROJECT CONVENTIONS

| Convention | Rule |
|---|---|
| Package manager | `bun` only — never `npm install` or `yarn` |
| Testing | Avoid mocks — test real implementations |
| Error handling | Effect error channels — no `try/catch` |
| Imports | Named namespace exports — no star imports |
| Variables | `const` > `let`; inline single-use values |
| Null checks | Early return on null/undefined — no `else` |
| Async patterns | Effect generators — no raw `async/await` in core |
| File I/O | `Bun.file()` — no Node `fs` |
| SQL | Drizzle ORM with snake_case column names |
| Config | Never hard-code — use `Config` service from Effect |

---

## QUICK REFERENCE: COMMON TASKS

```bash
# Start dev environment
bun run dev

# Run a prompt against the current project
opencode run "your prompt here"

# List agents
opencode agent list

# List MCP servers
opencode mcp list

# Install a plugin
opencode plug <package-name>

# View session history
opencode session list

# Attach to running server (remote or local)
opencode attach http://localhost:4001

# Run tests for a package
cd packages/<name> && bun test

# Type-check a package
cd packages/<name> && bun typecheck

# Generate client after protocol changes
cd packages/client && bun run generate

# Rebuild SDK after SDK source changes
./packages/sdk/js/script/build.ts

# Debug mode
opencode --debug --print-logs serve
```

---

*This document is a living AI reference. It summarizes the OpenCode repository as of 2026-08-19 for AI-assisted development. When in doubt, read the source — start with `packages/opencode/src/index.ts` and `packages/core/src/`.*
