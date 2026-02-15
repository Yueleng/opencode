# opencode agent guidelines

## Build/Test Commands

- **Install**: `bun install`
- **Run**: `bun run --conditions=browser ./src/index.ts`
- **Typecheck**: `bun run typecheck` (npm run typecheck)
- **Test**: `bun test` (runs all tests)
- **Single test**: `bun test test/tool/tool.test.ts` (specific test file)

## Code Style

- **Runtime**: Bun with TypeScript ESM modules
- **Imports**: Use relative imports for local modules, named imports preferred
- **Types**: Zod schemas for validation, TypeScript interfaces for structure
- **Naming**: camelCase for variables/functions, PascalCase for classes/namespaces
- **Error handling**: Use Result patterns, avoid throwing exceptions in tools
- **File structure**: Namespace-based organization (e.g., `Tool.define()`, `Session.create()`)

## Architecture

- **Tools**: Implement `Tool.Info` interface with `execute()` method
- **Context**: Pass `sessionID` in tool context, use `App.provide()` for DI
- **Validation**: All inputs validated with Zod schemas
- **Logging**: Use `Log.create({ service: "name" })` pattern
- **Storage**: Use `Storage` namespace for persistence
- **API Client**: The TypeScript TUI (built with SolidJS + OpenTUI) communicates with the OpenCode server using `@opencode-ai/sdk`. When adding/modifying server endpoints in `packages/opencode/src/server/server.ts`, run `./script/generate.ts` to regenerate the SDK and related files.

# OpenCode Agent Architecture

## Core Design

**CLI Entry Point** (`src/index.ts`)

- yargs-based CLI with commands: run, generate, auth, agent, serve, models, etc.
- Initializes logging, sets environment flags
- Error handling with NamedError patterns

**HTTP Server** (`src/server/server.ts`)

- Hono-based REST API with OpenAPI spec
- SSE event streaming for real-time updates
- Routes: /auth, /project, /session, /pty, /config, /provider, /question, /permission
- CORS with mDNS discovery for local network access
- Instance-based DI pattern (`Instance.provide`)

## Agent System (`src/agent/agent.ts`)

**Built-in Agents:**

- `build` - Default, executes tools with permissions
- `plan` - Planning mode, disallows edit tools
- `general` - Multi-step parallel task execution
- `explore` - Fast codebase exploration/search
- `compaction`, `title`, `summary` - Hidden internal agents

**Agent Schema:**

```ts
{
  name, description, mode: "subagent"|"primary"|"all",
  permission: PermissionNext.Ruleset,
  model?, prompt, temperature, topP, steps, options
}
```

## Tool System (`src/tool/`)

**Tool Interface** (`src/tool/tool.ts`):

```ts
Tool.define(id, {
  parameters: z.ZodType,
  description: string,
  execute: (args, ctx) =>
    Promise<{
      title
      output
      metadata
      attachments?
    }>,
})
```

**Built-in Tools:** bash, read, edit, write, glob, grep, webfetch, websearch, task, todo, skill, apply_patch, lsp

**Context includes:** sessionID, messageID, agent, abort, messages, `ask()` for permissions

## Permission System (`src/permission/`)

Two approaches coexist:

- **Legacy** (`index.ts`): Pending/approved state, wildcard matching
- **Next** (`next.ts`): Ruleset-based with actions (allow/deny/ask)

Rules can target tools (bash, read), operations (external_directory), or features (question, plan_enter)

## Session Management (`src/session/`)

**Session Info:** id, slug, projectID, directory, parentID, title, time, permission, revert info

**Message V2:** User/Assistant with parts (text, reasoning, tool_call, file)

**Operations:** create, fork, update, remove, messages list, diff, share/unshare

## LLM Integration (`src/session/llm.ts`)

- Uses Vercel AI SDK (`streamText`)
- Supports multiple providers via @ai-sdk packages
- System prompt injection (agent, provider, custom)
- Plugin hooks for transforming params/headers
- Reasoning extraction middleware

## Skills (`src/skill/`)

Scans directories for `SKILL.md`:

- `.claude/skills/**/SKILL.md` (Claude-compatible)
- `.opencode/skill/**/SKILL.md`
- Globally from `~/.claude`

## Key Patterns

1. **Namespace-based organization** (e.g., `Tool.define`, `Session.create`)
2. **Instance state** - Per-project DI container with `Instance.state()`
3. **Event bus** - Bus.publish/Bus.subscribe for decoupled communication
4. **Zod schemas** - Type-safe validation everywhere
5. **Lazy initialization** - Deferred loading for parsers, providers
6. **Truncation** - Output size limits for LLM context management
7. **Plugin system** - Extensible via trigger hooks

---
