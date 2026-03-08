# OpenCode Local Build + VS Code Debug Guide (Beginner TypeScript)

This guide is for this repository at:

- `/home/altay/src/opencode`

It focuses on the fastest path to run, build, and debug without needing deep TypeScript knowledge.

## 1. What this repo is

This is a **Bun + TypeScript monorepo**.

- Main CLI/server package: `packages/opencode`
- Web UI package: `packages/app`
- Desktop package: `packages/desktop`

Important: TypeScript is mostly run directly by Bun in dev (you usually do not run `tsc` manually to start the app).

## 2. Prerequisites

Install these first:

1. **Git**
2. **Bun 1.3+** (repo uses Bun 1.3.10)
3. **VS Code**
4. VS Code extension: **Bun for Visual Studio Code** (`oven.bun-vscode`)

Check Bun:

```bash
bun --version
```

## 3. First-time setup

From repo root:

```bash
cd /home/altay/src/opencode
bun install
```

This installs all workspace dependencies.

## 4. First run (core CLI in dev)

From repo root:

```bash
bun dev
```

Useful variants:

```bash
bun dev --help
bun dev .
bun dev serve
bun dev web
```

- `bun dev` runs the local OpenCode CLI entrypoint.
- `bun dev serve` runs headless API server (default port `4096`).
- `bun dev web` runs server + web interface.

## 5. Build commands (step by step)

This repo has multiple build targets.

### A) Build the core CLI package

```bash
bun run --cwd packages/opencode build
```

### B) Build a standalone local binary ("localcode")

```bash
./packages/opencode/script/build.ts --single
```

Binary output is under:

- `packages/opencode/dist/opencode-<platform>/bin/opencode`

### C) Build web app package

```bash
bun run --cwd packages/app build
```

### D) Build desktop app (Tauri)

```bash
bun run --cwd packages/desktop tauri build
```

Needs Rust/Tauri prerequisites.

## 6. Type checking and tests

### Type check all workspaces

```bash
bun typecheck
```

### Tests

Do **not** run tests from repo root (root test script is intentionally blocked).

Run tests from package directories, for example:

```bash
bun run --cwd packages/opencode test
bun run --cwd packages/app test
```

## 7. VS Code setup for debugging (recommended path)

Bun debugging is most reliable with **manual run + attach**.

### Step 1: Create VS Code config files

From repo root:

```bash
cp .vscode/launch.example.json .vscode/launch.json
cp .vscode/settings.example.json .vscode/settings.json
```

`launch.json` should contain an attach config similar to:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "bun",
      "request": "attach",
      "name": "opencode (attach)",
      "url": "ws://localhost:6499/"
    }
  ]
}
```

### Step 2: Start OpenCode in inspect mode (terminal 1)

From repo root:

```bash
bun run --inspect=ws://localhost:6499/ dev spawn .
```

Why `spawn`: in this repo, server breakpoints are often more reliable with `spawn` than plain `bun dev`.

### Step 3: Add breakpoints

In VS Code, open TypeScript files (for example in `packages/opencode/src/...`) and click the gutter to add breakpoints.

### Step 4: Attach debugger

1. Open Run and Debug panel
2. Choose `opencode (attach)`
3. Press `F5`

Now interact with the running CLI to hit breakpoints.

## 8. If breakpoints do not hit

Use one of these fallback flows.

### Fallback A: Debug server only

Terminal 1:

```bash
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096
```

Terminal 2:

```bash
opencode attach http://localhost:4096
```

Then attach from VS Code with `opencode (attach)`.

### Fallback B: Debug TUI process directly

```bash
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode --conditions=browser ./src/index.ts
```

Then attach in VS Code.

### Optional quality-of-life

Set Bun inspect option once per shell session:

```bash
export BUN_OPTIONS=--inspect=ws://localhost:6499/
```

Then run your normal `bun dev ...` commands.

## 9. Common beginner workflow (copy/paste)

```bash
cd /home/altay/src/opencode
bun install
cp .vscode/launch.example.json .vscode/launch.json
cp .vscode/settings.example.json .vscode/settings.json
bun run --inspect=ws://localhost:6499/ dev spawn .
```

Then in VS Code: set breakpoint -> Run and Debug -> `opencode (attach)` -> `F5`.

## 10. When to regenerate SDK files

If you change API/server contract code and need updated JS SDK artifacts:

```bash
./packages/sdk/js/script/build.ts
```

Also check if repo-level generation is needed:

```bash
./script/generate.ts
```

## 11. Quick TypeScript mental model for this repo

- You write `.ts` files.
- Bun runs/transpiles them in dev.
- `typecheck` validates types across packages.
- `build` commands create production artifacts where relevant.
- Debugging is done by attaching VS Code to Bun inspector.
