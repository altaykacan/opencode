# Custom Tools and Skills in This Repo

This guide explains how to define your own:

- custom tools (callable functions the agent can execute)
- skills (reusable workflows/instructions loaded on demand)

It covers both:

- user-level extension (`.opencode/` in your project or `~/.config/opencode/`)
- core extension (adding built-in tools in `packages/opencode/src/tool`)

## 1) Define Custom Tools (No Core Code Changes)

OpenCode auto-loads tool files from config directories using this pattern:

- `{tool,tools}/*.{ts,js}`

In practice, use one of these:

- project-local: `.opencode/tool/` or `.opencode/tools/`
- global: `~/.config/opencode/tool/` or `~/.config/opencode/tools/`
- optional custom config dir from `OPENCODE_CONFIG_DIR` (same `tool/` or `tools/` layout)

You can see real examples in this repo:

- `.opencode/tool/github-pr-search.ts`
- `.opencode/tool/github-triage.ts`

### Minimal template

```ts
// .opencode/tool/hello.ts
import { tool } from "@opencode-ai/plugin"

export default tool({
  description: "Return a greeting",
  args: {
    name: tool.schema.string().describe("Name to greet"),
  },
  async execute(args, ctx) {
    // ctx.directory: current session directory
    // ctx.worktree: git worktree root
    // ctx.ask(...) and ctx.metadata(...) are available for permission/UI metadata flows
    return `hello ${args.name}`
  },
})
```

Tool name is based on filename/export:

- default export in `hello.ts` -> `hello`
- named export `foo` in `math.ts` -> `math_foo`

### Multi-tool file

```ts
// .opencode/tools/math.ts
import { tool } from "@opencode-ai/plugin"

export const add = tool({
  description: "Add numbers",
  args: {
    a: tool.schema.number(),
    b: tool.schema.number(),
  },
  async execute(args) {
    return String(args.a + args.b)
  },
})

export const sub = tool({
  description: "Subtract numbers",
  args: {
    a: tool.schema.number(),
    b: tool.schema.number(),
  },
  async execute(args) {
    return String(args.a - args.b)
  },
})
```

This creates tools `math_add` and `math_sub`.

Note: plugin-defined custom tools return a string output.

### Dependencies for local tools

If your tool imports packages, add a `package.json` in the same config directory (for example `.opencode/package.json`). OpenCode installs dependencies there on startup.

```json
{
  "dependencies": {
    "zod": "^4.0.0"
  }
}
```

### Tool permissions

Control access in `opencode.json` using tool IDs:

```json
{
  "permission": {
    "hello": "allow",
    "math_*": "ask"
  }
}
```

## 2) Define Skills (Workflows)

Skills are markdown workflows loaded via the built-in `skill` tool.

Create folders like:

- `.opencode/skills/<name>/SKILL.md` (or `.opencode/skill/<name>/SKILL.md`)
- `~/.config/opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md` and `.agents/skills/<name>/SKILL.md` are also discovered
- `OPENCODE_CONFIG_DIR` can also provide `skill/` or `skills/`

### Minimal SKILL.md

```md
---
name: pr-review
description: Review a PR for regressions and missing tests
---

## When to use

Use this when the user asks for a code review.

## Workflow

1. Identify changed files.
2. Prioritize correctness and behavioral regressions.
3. Report findings with severity and file references.
```

The runtime requires at least `name` and `description` frontmatter to load a skill.

### Skill permissions

```json
{
  "permission": {
    "skill": {
      "*": "allow",
      "internal-*": "deny",
      "experimental-*": "ask"
    }
  }
}
```

### Extra skill sources from config

You can add more skill directories or hosted skill packs:

```json
{
  "skills": {
    "paths": ["./team-skills", "~/company/skills"],
    "urls": ["https://example.com/.well-known/skills/"]
  }
}
```

Hosted URL format needs an `index.json` like:

```json
{
  "skills": [
    {
      "name": "agents-sdk",
      "description": "Cloudflare Agents SDK",
      "files": ["SKILL.md", "references/callable.md"]
    }
  ]
}
```

## 3) How To Verify

- List discovered skills:
  - `opencode skill`
- Start OpenCode in this repo and ask it to use your new tool or load your skill by name.
- If a tool/skill does not appear, check:
  - path and filename (`SKILL.md` must be uppercase)
  - frontmatter (`name`, `description`)
  - permission rules (`deny` hides/blocks)

## 4) Extend Core Built-In Tools (Repo Contributor Path)

If you want a first-class built-in tool in OpenCode itself:

1. Add a tool module under `packages/opencode/src/tool/<name>.ts` using `Tool.define(...)`.
2. Add its description prompt file `packages/opencode/src/tool/<name>.txt` if needed.
3. Register it in `packages/opencode/src/tool/registry.ts`.
4. Add tests under `packages/opencode/test/tool/<name>.test.ts`.
5. Run tests from package dir (not repo root), for example:
   - `cd packages/opencode && bun test test/tool/<name>.test.ts`

Useful references:

- Tool runtime types: `packages/opencode/src/tool/tool.ts`
- Custom tool loading: `packages/opencode/src/tool/registry.ts`
- Skill discovery/loading: `packages/opencode/src/skill/skill.ts`
