# js

To install dependencies:

```bash
bun install
```

To run:

```bash
bun run index.ts
```

This project was created using `bun init` in bun v1.2.12. [Bun](https://bun.sh) is a fast all-in-one JavaScript runtime.

---

# 🤖 AI Agent System Architecture

The AI agent system in OpenCode is a sophisticated architecture that orchestrates LLM interactions, tool execution, permission management, and session handling.

## 1. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         USER INPUT                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        SESSION                                   │
│  • Manages conversation state                                    │
│  • Stores messages & parts                                       │
│  • Tracks costs & tokens                                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         AGENT                                    │
│  • build, plan, general, explore, etc.                          │
│  • Permission rules                                              │
│  • Custom prompts                                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       PROCESSOR                                  │
│  • Orchestrates the LLM loop                                    │
│  • Handles streaming                                             │
│  • Executes tool calls                                           │
│  • Manages retries & errors                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────┬────────────────┬───────────────────────────────┐
│     LLM        │    TOOLS       │         PROVIDER              │
│  (Vercel AI)   │  (bash, edit,  │  (Claude, OpenAI, Gemini...)  │
│                │   read, etc.)  │                               │
└────────────────┴────────────────┴───────────────────────────────┘
```

## 2. Agents (`src/agent/agent.ts`)

Agents are **predefined personas/profiles** that control how OpenCode behaves. Each agent has:

| Property      | Description                                      |
| ------------- | ------------------------------------------------ |
| `name`        | Identifier (e.g., `build`, `plan`)               |
| `mode`        | `primary` (user-facing) or `subagent` (internal) |
| `permission`  | Rules for what tools can be used                 |
| `prompt`      | Custom system prompt (optional)                  |
| `temperature` | LLM temperature setting                          |
| `model`       | Specific model override (optional)               |

### Built-in Agents

```typescript
// From agent.ts
{
  build: {
    name: "build",
    description: "The default agent. Executes tools based on configured permissions.",
    mode: "primary",
    // Full access to tools
  },

  plan: {
    name: "plan",
    description: "Plan mode. Disallows all edit tools.",
    mode: "primary",
    // Read-only - denies edit/write tools
  },

  general: {
    name: "general",
    description: "General-purpose agent for researching complex questions...",
    mode: "subagent",  // Used internally via @general
  },

  explore: {
    name: "explore",
    description: "Fast agent specialized for exploring codebases...",
    mode: "subagent",
    // Only allows: grep, glob, list, bash, read, websearch
  }
}
```

## 3. Tool System (`src/tool/`)

Tools are the **actions** agents can take. Each tool is defined using `Tool.define()`:

```typescript
// Pattern from tool.ts
export const BashTool = Tool.define("bash", async () => ({
  description: "Execute shell commands...",
  parameters: z.object({
    command: z.string(),
    timeout: z.number().optional(),
    workdir: z.string().optional(),
    description: z.string(),
  }),
  async execute(params, ctx) {
    // Execute the command
    // Returns { title, metadata, output }
  },
}))
```

### Built-in Tools

| Tool                 | Purpose                  |
| -------------------- | ------------------------ |
| `bash`               | Execute shell commands   |
| `read`               | Read file contents       |
| `edit`               | Edit existing files      |
| `write`              | Create new files         |
| `grep`               | Search code with ripgrep |
| `glob`               | Find files by pattern    |
| `task`               | Launch subagents         |
| `webfetch`           | Fetch web content        |
| `websearch`          | Search the web           |
| `codesearch`         | Semantic code search     |
| `todowrite/todoread` | Task tracking            |
| `question`           | Ask user questions       |
| `skill`              | Read skill files         |

### Tool Context

Each tool receives a context object:

```typescript
type Context = {
  sessionID: string
  messageID: string
  agent: string
  abort: AbortSignal
  messages: MessageV2.WithParts[]
  metadata(input: { title?: string; metadata? }): void // Stream metadata
  ask(permission): Promise<void> // Request permission
}
```

## 4. Session Processor (`src/session/processor.ts`)

The processor is the **main loop** that orchestrates agent execution:

```typescript
// Simplified flow
while (true) {
  // 1. Stream from LLM
  const stream = await LLM.stream(input)

  for await (const value of stream.fullStream) {
    switch (value.type) {
      case "text-delta":
        // Stream text to user
        break

      case "tool-call":
        // Execute tool, check permissions
        break

      case "tool-result":
        // Store result, continue loop
        break

      case "finish-step":
        // Track usage, check if compaction needed
        break
    }
  }

  // 2. Check if done or need to continue
  if (needsCompaction) return "compact"
  if (blocked) return "stop"
  return "continue"
}
```

### Key Features

- **Doom Loop Detection**: Detects when the agent is stuck calling the same tool repeatedly
- **Automatic Retries**: Handles rate limits and transient errors
- **Snapshot Tracking**: Captures file state before/after each step
- **Cost Tracking**: Calculates token usage and costs in real-time

## 5. LLM Integration (`src/session/llm.ts`)

Uses the **Vercel AI SDK** to communicate with various providers:

```typescript
return streamText({
  model: wrapLanguageModel({
    model: language,
    middleware: [
      extractReasoningMiddleware({ tagName: "think" }), // For reasoning models
    ],
  }),
  tools,
  messages: [...systemMessages, ...userMessages],
  temperature,
  topP,
  maxOutputTokens,
  // ...
})
```

### Provider-Specific Prompts

Different prompts for different models (from `src/session/system.ts`):

```typescript
export function provider(model: Provider.Model) {
  if (model.api.id.includes("gpt-5")) return [PROMPT_CODEX]
  if (model.api.id.includes("gemini-")) return [PROMPT_GEMINI]
  if (model.api.id.includes("claude")) return [PROMPT_ANTHROPIC]
  return [PROMPT_ANTHROPIC_WITHOUT_TODO]
}
```

## 6. Permission System (`src/permission/`)

Controls what tools can do:

```typescript
// Permission rules
const defaults = PermissionNext.fromConfig({
  "*": "allow", // Default allow
  doom_loop: "ask", // Ask on repeat actions
  question: "deny", // Deny question tool by default
  read: {
    "*": "allow",
    "*.env": "ask", // Ask before reading .env files
  },
})
```

### Permission Flow

1. Tool tries to execute
2. Permission check against agent's ruleset
3. If `ask` → prompts user for approval
4. If `deny` → throws `RejectedError`
5. If `allow` → proceeds

## 7. System Prompts (`src/session/prompt/`)

The main prompt teaches the agent:

1. **Identity**: "You are OpenCode, the best coding agent on the planet."
2. **Tone**: Concise, no emojis, professional
3. **Task Management**: Use TodoWrite frequently
4. **Tool Usage Policy**:
   - Use Task tool for complex exploration
   - Prefer specialized tools over bash
   - Parallelize independent tool calls

## 8. Complete Request Flow

```
1. User sends message
   ↓
2. Session.create() or Session.get()
   ↓
3. Agent.get() → loads agent config
   ↓
4. SessionProcessor.create() → initializes processor
   ↓
5. LLM.stream() → sends to provider
   ↓
6. For each stream event:
   - text-delta → update UI
   - tool-call → check permissions → execute → store result
   - finish-step → track costs, snapshot
   ↓
7. Loop continues until:
   - No more tool calls (finish)
   - Error occurs (stop)
   - Compaction needed (compact)
   - User aborts (stop)
```

## 9. Key Takeaways

| Aspect                 | Implementation                                   |
| ---------------------- | ------------------------------------------------ |
| **Multi-Agent**        | Agents can spawn subagents via `Task` tool       |
| **Streaming**          | Real-time output using Vercel AI SDK             |
| **Permission Control** | Granular per-tool, per-file permissions          |
| **Provider Agnostic**  | Unified interface for Claude/OpenAI/Gemini/local |
| **Extensible**         | Custom tools via plugins or config directories   |
| **Stateful**           | Sessions persist messages and costs              |
