# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Requirements:** Bun 1.3+

```bash
# Install dependencies
bun install

# Run OpenCode TUI (defaults to packages/opencode dir)
bun dev

# Run against a specific directory
bun dev <directory>

# Run against the repo root itself
bun dev .

# Start headless API server (port 4096 by default)
bun dev serve
bun dev serve --port 8080

# Start server + open web interface
bun dev web

# Start web UI dev server (requires server running separately)
bun run --cwd packages/app dev
# Web UI available at http://localhost:5173

# Run tests (must be run from package directory, NOT repo root)
bun test --cwd packages/opencode
bun test --cwd packages/opencode --timeout 30000

# Run a single test file
bun test packages/opencode/test/session/session.test.ts

# Typecheck all workspaces
bun turbo typecheck

# Regenerate JS SDK after API/server changes
./script/generate.ts
# OR: ./packages/sdk/js/script/build.ts

# Build standalone executable
./packages/opencode/script/build.ts --single
# Output: ./packages/opencode/dist/opencode-<platform>/bin/opencode

# Desktop app (requires Rust toolchain + Tauri prerequisites)
bun run --cwd packages/desktop tauri dev
bun run --cwd packages/desktop tauri build
```

**Important:** `bun test` at the repo root is guarded and will exit with an error. Always run tests from `packages/opencode` or another specific package directory.

### Debugging

The most reliable debug method is **manual run + attach**:

```bash
# Terminal 1: start with Bun inspector (spawn recommended for server breakpoints)
bun run --inspect=ws://localhost:6499/ dev spawn .

# VS Code: set up config files once
cp .vscode/launch.example.json .vscode/launch.json
cp .vscode/settings.example.json .vscode/settings.json
# Then: Run and Debug → "opencode (attach)" → F5

# Fallback: debug server and TUI separately
# Terminal 1 (server):
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096
# Terminal 2 (TUI):
opencode attach http://localhost:4096

# Optional: set once per shell to avoid repeating --inspect flag
export BUN_OPTIONS=--inspect=ws://localhost:6499/
```

## Architecture

OpenCode is an open-source AI coding agent with a **client/server architecture**:

### Packages

| Package | Purpose |
|---------|---------|
| `packages/opencode` | Core server + CLI + TUI. HTTP/SSE API via Hono (port 4096). TUI in SolidJS + opentui. Entry: `src/index.ts` |
| `packages/app` | Shared web UI (SolidJS + Tailwind). Used by web interface and desktop app |
| `packages/desktop` | Native desktop app (Tauri wrapping `packages/app`) |
| `packages/sdk/js` | Auto-generated TypeScript SDK from OpenAPI spec |
| `packages/plugin` | `@opencode-ai/plugin` — framework for custom tools and extensions |
| `packages/ui` | Shared UI component library |
| `packages/util` | Shared utilities (`@opencode-ai/util`) |
| `packages/script` | Build/generation scripts |
| `packages/console` | Console utilities |
| `packages/containers` | Container/infrastructure support |
| `packages/docs` | Documentation site |
| `packages/enterprise` | Enterprise features |
| `packages/identity` | Identity/auth service |
| `packages/extensions` | Extension system |
| `packages/slack` | Slack integration |
| `packages/web` | Web application |

### Core Server (`packages/opencode/src/`)

**Core Infrastructure:**

| Directory | Purpose |
|-----------|---------|
| `server/` | Hono HTTP server; routes in `server/routes/` |
| `session/` | Chat session management, LLM streaming, compaction (192KB across 14 files) |
| `provider/` | AI provider adapters (wraps Vercel AI SDK); 20+ providers |
| `config/` | Config loading with precedence tiers (`opencode.jsonc`) |
| `storage/` | SQLite via Drizzle ORM; JSON migration from legacy format |
| `bus/` | Internal event bus for server-side pub/sub |

**Agent & Tools:**

| Directory | Purpose |
|-----------|---------|
| `agent/` | Agent definitions: `build` (default), `plan` (experimental), `general` |
| `tool/` | 25+ agent tools: bash, edit, read, write, grep, glob, lsp, mcp, webfetch, websearch, etc. |
| `skill/` | Skill discovery and loading system |
| `permission/` | Tool access control (allow/deny/ask + wildcards) |

**Protocol Support:**

| Directory | Purpose |
|-----------|---------|
| `mcp/` | Model Context Protocol client (30KB); OAuth support |
| `lsp/` | Language Server Protocol integration (63KB server) |
| `acp/` | Agent Client Protocol support (59KB agent) |

**CLI & Project:**

| Directory | Purpose |
|-----------|---------|
| `cli/` | CLI commands: run, serve, web, auth, agent, github, mcp, stats, session, etc. |
| `project/` | Instance lifecycle, worktrees, bootstrap, VCS |
| `control-plane/` | Multi-workspace/worktree coordination |
| `worktree/` | Git worktree management |
| `snapshot/` | Session snapshots |

**Other Subsystems:**

| Directory | Purpose |
|-----------|---------|
| `auth/` | Authentication handling |
| `file/` | File monitoring, watcher, ripgrep integration |
| `format/` | Code formatters |
| `ide/` | IDE integration |
| `util/` | 27 utility modules (log, filesystem, glob, process, git, hash, lock, etc.) |
| `flag/` | Feature flags |
| `id/` | Identifier generation (ULID) |
| `pty/` | PTY/shell utilities |
| `scheduler/` | Task scheduler |
| `share/` | Sharing utilities |

### Server Routes (`packages/opencode/src/server/routes/`)

`config`, `experimental`, `file`, `global`, `mcp`, `permission`, `project`, `provider`, `pty`, `question`, `session`, `tui`

### Tools (`packages/opencode/src/tool/`)

`apply_patch`, `bash`, `batch`, `codesearch`, `edit`, `external-directory`, `glob`, `grep`, `invalid`, `ls`, `lsp`, `multiedit`, `plan`, `question`, `read`, `registry`, `skill`, `task`, `todo`, `tool`, `truncation`, `webfetch`, `websearch`, `write`

### Supported AI Providers

Anthropic, OpenAI, Azure, Google, Vertex, Amazon Bedrock, Mistral, Groq, Cohere, DeepInfra, Cerebras, Together AI, Perplexity, XAI, OpenRouter, GitLab, and any OpenAI-compatible endpoint.

### Configuration Precedence (highest to lowest)

1. Managed config (enterprise)
2. Inline config (`OPENCODE_CONFIG_CONTENT` env var)
3. Project config (`opencode.json{,c}`, `.opencode/` directories)
4. Custom config dir (`OPENCODE_CONFIG` env var)
5. Global config (`~/.config/opencode/opencode.json{,c}`)
6. Remote org defaults (`.well-known/opencode`)

### Data Flow

1. User interacts via TUI or Web UI → sends requests to Hono server
2. Server manages sessions in SQLite (Drizzle ORM)
3. Agent orchestrates tool calls using the Vercel AI SDK
4. Events stream back to clients via SSE

### SDK Regeneration

When you modify `packages/opencode/src/server/server.ts` or any route, run:
```bash
./script/generate.ts
```
This regenerates `packages/sdk/js/` and `packages/sdk/openapi.json`.

## Key Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| `hono` | 4.10.7 | Web framework |
| `ai` | 5.0.124 | Vercel AI SDK |
| `zod` | 4.1.8 | Schema validation |
| `drizzle-orm` | 1.0.0-beta.16 | ORM for SQLite |
| `solid-js` | 1.9.10 | UI framework |
| `@modelcontextprotocol/sdk` | 1.25.2 | MCP support |
| `@agentclientprotocol/sdk` | 0.14.1 | ACP support |
| `@opentui/core` | latest | Terminal UI |
| `remeda` | 2.26.0 | Functional utilities |
| `ulid` | 3.0.1 | Unique ID generation |
| `web-tree-sitter` | 0.25.10 | Code parsing |

## Custom Tools & Skills

### Custom Tools (no core changes needed)

Place tool files in any of these locations (auto-loaded on startup):
- `.opencode/tool/*.{ts,js}` or `.opencode/tools/*.{ts,js}` (project-local)
- `~/.config/opencode/tool/` or `~/.config/opencode/tools/` (global)

```ts
// .opencode/tool/hello.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Return a greeting",
  args: { name: tool.schema.string().describe("Name to greet") },
  async execute(args, ctx) {
    return `hello ${args.name}`
  },
})
```

- Default export in `hello.ts` → tool name `hello`
- Named export `foo` in `math.ts` → tool name `math_foo`
- Add a `package.json` beside tool files if they need npm dependencies

### Skills

Place skill markdown files in:
- `.opencode/skills/<name>/SKILL.md` (or `.opencode/skill/`)
- `~/.config/opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md` or `.agents/skills/<name>/SKILL.md`

Minimum required frontmatter: `name` and `description`.

### Adding Core Built-In Tools

1. Add `packages/opencode/src/tool/<name>.ts` using `Tool.define(...)`
2. Add description prompt `packages/opencode/src/tool/<name>.txt` if needed
3. Register in `packages/opencode/src/tool/registry.ts`
4. Add tests under `packages/opencode/test/tool/<name>.test.ts`

### Tool Permissions (in `opencode.jsonc`)

```json
{
  "permission": {
    "hello": "allow",
    "math_*": "ask",
    "skill": { "*": "allow", "internal-*": "deny" }
  }
}
```

## Testing

- Tests live in `packages/opencode/test/` with subdirectories: `agent/`, `acp/`, `auth/`, `cli/`, `config/`, `control-plane/`, `file/`, `format/`, `provider/`, `session/`, `tool/`, `util/`, `worktree/`
- **Avoid mocks** — test actual implementation, do not duplicate logic
- Tests **cannot** run from repo root; always use `--cwd packages/opencode` or `cd` into the package

## Style Guide (MANDATORY for agent-written code)

From `AGENTS.md`:

### Naming (strictly enforced)

- **Single-word variable names** by default. Multi-word (camelCase) only when a single word is genuinely ambiguous.
- Good: `pid`, `cfg`, `err`, `opts`, `dir`, `root`, `child`, `state`, `timeout`
- Bad: `inputPID`, `existingClient`, `connectTimeout`, `workerPath`
- Inline values used only once instead of introducing a variable

```ts
// Good
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// Bad
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

### Other Rules

- **No destructuring** — use dot notation to preserve context (`obj.a` not `const { a } = obj`)
- **No `else`** — prefer early returns
- **No `try`/`catch`** — prefer `.catch(...)` chains
- **`const` over `let`** — use ternaries or early returns instead of reassignment
- **No `any` type** — prefer precise types; rely on type inference
- **Bun APIs** — use `Bun.file()`, etc. over Node.js equivalents
- **Functional methods** — use `flatMap`, `filter`, `map` over `for` loops (with type guards on `filter`)
- **Keep functions small** — avoid breaking out logic unless it's reusable
- **Drizzle schema**: use `snake_case` field names (avoids string column name redefinition)

```ts
// Good schema
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})
```

## PR Conventions

- PR titles follow conventional commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`
- Optional scope: `feat(app):`, `fix(desktop):`, `chore(opencode):`
- All PRs **must** reference an existing issue (`Fixes #123`)
- UI/core product features require design review with the core team before implementation
- Keep PRs small and focused; include screenshots/videos for UI changes
- No AI-generated walls of text in PR descriptions

## Default Branch

The default branch is `dev`. Use `dev` or `origin/dev` (not `main`) for diffs and PRs. Local `main` ref may not exist.

## Agent Behavior Notes

- ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE
- Prefer automation: execute requested actions without confirmation unless blocked by missing info, safety, or irreversibility
- To regenerate the JavaScript SDK: `./packages/sdk/js/script/build.ts`
