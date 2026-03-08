# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Requirements:** Bun 1.3+

```bash
# Install dependencies
bun install

# Run OpenCode TUI (in packages/opencode dir by default)
bun dev

# Run against a specific directory
bun dev <directory>

# Run against the repo root itself
bun dev .

# Start headless API server (port 4096)
bun dev serve

# Start web UI (requires server running separately)
bun run --cwd packages/app dev

# Run tests (must be run from package directory, NOT repo root)
bun test --cwd packages/opencode
bun test --cwd packages/opencode --timeout 30000

# Run a single test file
bun test packages/opencode/test/session/session.test.ts

# Typecheck
bun turbo typecheck

# Regenerate SDK after API/server changes
./script/generate.ts

# Build standalone executable
./packages/opencode/script/build.ts --single
# Output: ./packages/opencode/dist/opencode-<platform>/bin/opencode

# Desktop app (requires Rust toolchain)
bun run --cwd packages/desktop tauri dev
```

**Important:** `bun test` at the repo root is guarded and will exit with an error. Always run tests from `packages/opencode` or another specific package directory.

## Architecture

OpenCode is an open-source AI coding agent with a **client/server architecture**:

- **`packages/opencode`** — Core server + CLI. The server exposes an HTTP/SSE API (Hono framework, port 4096 by default). The CLI also includes the TUI (SolidJS + opentui). Entry: `src/index.ts`.
- **`packages/app`** — Shared web UI components in SolidJS/Tailwind. Used by both the web interface and desktop app.
- **`packages/desktop`** — Native desktop app built with Tauri (wraps `packages/app`).
- **`packages/sdk/js`** — Auto-generated TypeScript SDK from the OpenAPI spec. Regenerate with `./script/generate.ts`.
- **`packages/plugin`** — `@opencode-ai/plugin` package for extending OpenCode.
- **`packages/ui`** — Shared UI component library.
- **`packages/util`** — Shared utilities.

### Core Server (`packages/opencode/src/`)

Key subsystems:

| Directory | Purpose |
|-----------|---------|
| `server/` | Hono HTTP server; routes in `server/routes/` |
| `session/` | Chat session management, LLM streaming, compaction |
| `agent/` | Agent definitions (build, plan, general), prompt assembly |
| `tool/` | All agent tools (bash, edit, read, write, grep, glob, lsp, mcp, etc.) |
| `provider/` | AI provider adapters (wraps Vercel AI SDK) |
| `lsp/` | Language Server Protocol integration |
| `mcp/` | Model Context Protocol server support |
| `config/` | Configuration loading (`opencode.jsonc`) |
| `project/` | Project/instance lifecycle, worktrees |
| `storage/` | SQLite via Drizzle ORM; JSON migration from legacy format |
| `bus/` | Internal event bus for server-side pub/sub |
| `skill/` | Slash command skills |
| `permission/` | Tool permission system |
| `control-plane/` | Multi-workspace/worktree coordination |

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

## Style Guide (MANDATORY for agent-written code)

From `AGENTS.md`:

- **Single-word variable names** by default. Multi-word names (camelCase) only when a single word would be ambiguous. Good: `pid`, `cfg`, `err`, `opts`, `dir`. Bad: `inputPID`, `connectTimeout`.
- Inline values used only once rather than introducing a variable.
- Avoid `else` — prefer early returns.
- Prefer `.catch(...)` over `try`/`catch`.
- Prefer `const` over `let`.
- Avoid unnecessary destructuring — use dot notation.
- Avoid `any` type.
- Use Bun APIs: `Bun.file()`, etc.
- Use functional array methods (`flatMap`, `filter`, `map`) over `for` loops.
- Drizzle schema: use `snake_case` field names.

## PR Conventions

PR titles follow conventional commits: `feat:`, `fix:`, `docs:`, `chore:`, `refactor:`, `test:`. Optional scope: `feat(app):`, `fix(desktop):`.

All PRs must reference an existing issue (`Fixes #123`).

## Default Branch

The default branch is `dev`. Use `dev` or `origin/dev` (not `main`) for diffs and PRs.
