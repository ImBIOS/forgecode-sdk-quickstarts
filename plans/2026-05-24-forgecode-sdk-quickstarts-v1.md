# ForgeCode SDK Quickstarts — TypeScript Port Plan

**Date:** 2026-05-24
**Goal:** Re-implement all [anthropics/claude-quickstarts](https://github.com/anthropics/claude-quickstarts) projects as TypeScript applications using the [ForgeCode SDK](https://github.com/ImBIOS/forgecode-sdk) (`@imbios/forgecode-sdk`) as the AI provider layer.

**Modality policy:**

- All models are treated as **text-to-text by default** — no image input capability is assumed.
- Any feature requiring image understanding (e.g. screenshot analysis, chart reading, computer/browser use) is routed through an **MCP server**.
- The default image-to-text MCP is **MiniMax MCP** (`minimax-coding-plan-mcp`). This is **not built-in** — it must be manually configured via `.mcp.json` and requires `uvx` + a valid `MINIMAX_API_KEY`. Users may substitute any compatible MCP server.

---

## 1. Modality Policy & MCP Strategy

### 1.1 Text-to-Text Default

Every quickstart in this repo is designed to work with any text-only LLM. No quickstart hard-codes the assumption that the underlying model can receive images. This ensures:

- Portability across models (local, cloud, open-weight)
- Predictable cost — no accidental image token billing
- Compatibility with the ForgeCode agent's default text-focused workflow

### 1.2 Image-to-Text via MiniMax MCP

When a quickstart genuinely requires image understanding (screenshot reading, chart analysis, visual browser content), it delegates that capability to an **MCP server** rather than passing images directly to the main model.

**Default image MCP: MiniMax MCP**

MiniMax MCP (`minimax-coding-plan-mcp`) is the project-default for image-to-text operations. It is **not bundled or auto-discovered** — it must be explicitly configured in both `.mcp.json` (for forge CLI pickup) and `shared/mcp-config.ts` (for programmatic SDK use).

```
Capability          Provider (default)         Requires
─────────────────   ─────────────────────────  ────────────────────────────────────
Text generation     forge agent (SDK)          forge CLI binary
Image understanding MiniMax MCP (manual setup) uvx, MINIMAX_API_HOST, MINIMAX_API_KEY
Browser control     Playwright MCP             pnpx, @playwright/mcp
Code execution      forge agent (built-in)     —
```

### 1.3 MCP Configuration — Two Layers

MCP servers must be wired up in **two places**:

#### Layer 1 — `.mcp.json` (forge CLI config file)

The `forge` CLI auto-reads `.mcp.json` from the project root. This is the same format used in the local forge installation at `/home/imbios/forge/.mcp.json`. A committed `.mcp.json` at the repo root makes MCP servers available to any forge CLI invocation:

```json
{
  "mcpServers": {
    "MiniMax": {
      "command": "uvx",
      "args": ["minimax-coding-plan-mcp"],
      "env": {
        "MINIMAX_API_HOST": "https://api.minimax.io",
        "MINIMAX_API_KEY": "<your key here>"
      },
      "disable": false
    }
  }
}
```

> The `.mcp.json` in the repo root **must not commit real API keys** — use `.env` substitution or a secrets manager. Commit a `.mcp.json.example` with placeholder values instead.

#### Layer 2 — `shared/mcp-config.ts` (programmatic SDK use)

When calling `query()` programmatically, MCP servers are passed via the `mcpServers` option. The shared config reads keys from environment variables:

```typescript
// shared/mcp-config.ts
export const minimaxMcpServer = {
  name: "MiniMax",
  command: "uvx",
  args: ["minimax-coding-plan-mcp"],
  env: {
    MINIMAX_API_HOST: process.env.MINIMAX_API_HOST ?? "https://api.minimax.io",
    MINIMAX_API_KEY: process.env.MINIMAX_API_KEY ?? "",
  },
};

// Usage in any quickstart that needs vision:
import { query } from "@imbios/forgecode-sdk";
import { minimaxMcpServer } from "../shared/mcp-config";

for await (const msg of query(prompt, {
  mcpServers: [minimaxMcpServer],  // loaded before the run
})) { ... }
```

> Users may replace `minimaxMcpServer` with any MCP-compatible vision provider by updating `shared/mcp-config.ts` and the corresponding `.mcp.json`.

---

## 2. Source Analysis

### 2.1 claude-quickstarts Projects

| Directory | Description | Original Stack |
|---|---|---|
| `customer-support-agent` | AI-assisted customer support system with knowledge base access | Python / Node.js |
| `financial-data-analyst` | Interactive financial data analysis via chat with data visualization | Python |
| `computer-use-demo` | Docker-based desktop control using Claude's computer_use tools | Python + Docker |
| `computer-use-best-practices` | Native macOS computer-use agent with best-practice patterns | Python (macOS only) |
| `browser-use-demo` | Playwright-backed browser automation with Claude controlling DOM | Python + Playwright |
| `autonomous-coding` | Two-agent pattern (initializer + coder) building apps incrementally via git | Python (Claude Agent SDK) |
| `agents` | General-purpose agent patterns and orchestration examples | Python |

### 2.2 ForgeCode SDK (`@imbios/forgecode-sdk`)

**Package:** `@imbios/forgecode-sdk`
**Install:** `pnpm add @imbios/forgecode-sdk`
**Requirements:** Bun >= 1.0.0, `forge` CLI binary installed

**Core API:**

```typescript
import { query } from "@imbios/forgecode-sdk";

for await (const message of query(prompt, options?)) {
  switch (message.type) {
    case "system":    // session_id and metadata
    case "assistant": // streamed response tokens (message.content)
    case "tool_use":  // agent invoking a tool
    case "result":    // final result (message.result)
    case "error":     // error (message.error)
  }
}
```

**`QueryOptions` interface:**

| Option | Type | Description |
|---|---|---|
| `agent` | `string` | Agent name (default: `"forge"`) |
| `model` | `string` | Model to use |
| `maxTurns` | `number` | Maximum conversation turns |
| `conversationId` | `string` | Continue an existing conversation (session) |
| `systemPrompt` | `string` | Custom system prompt |
| `env` | `Record<string, string>` | Environment variables |
| `outputFormat` | `OutputFormat` | Structured output configuration (Zod schema) |

**Binary resolution order:**

1. `FORGE_PATH` environment variable
2. `config.forgePath` (global config)
3. `~/.local/bin/forge`
4. `forge` on system PATH

**Key architectural difference vs raw Claude API:**
The ForgeCode SDK wraps the `forge` CLI binary, which is a full AI coding agent. Instead of calling `anthropic.messages.create()` and managing tool loops manually, you issue prompts to `query()` and the agent handles multi-turn tool execution internally. This means:

- No manual message array management
- No manual tool dispatch loops
- Tool calls are visible as `tool_use` events but executed by the agent internally
- Session continuity via `conversationId`

---

## 3. Project Structure

```
forgecode-sdk-quickstarts/
├── plans/
│   └── 2026-05-24-forgecode-sdk-quickstarts-v1.md
├── customer-support-agent/
│   ├── package.json
│   ├── tsconfig.json
│   ├── src/
│   │   ├── index.ts           # CLI entry point
│   │   ├── agent.ts           # ForgeCode query wrapper with system prompt
│   │   ├── knowledge-base.ts  # In-memory / file-based KB loader
│   │   └── types.ts           # Shared types
│   └── data/
│       └── knowledge-base.json
├── financial-data-analyst/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts           # Express server + SSE streaming
│       ├── agent.ts           # ForgeCode query with CSV/JSON data context
│       ├── data-loader.ts     # CSV/JSON dataset loader
│       └── public/
│           └── index.html     # Chart.js frontend
├── browser-use-demo/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts           # CLI entry point
│       ├── agent.ts           # ForgeCode query with browser system prompt
│       └── browser-tools.ts   # Playwright browser tool stubs for MCP
├── autonomous-coding/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts           # Orchestration entry point
│       ├── initializer.ts     # First-pass agent: scaffold and feature list
│       ├── coder.ts           # Second-pass agent: implement features via forge
│       └── git-helpers.ts     # Git commit helpers per feature
├── agents/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts           # Demo runner
│       ├── multi-agent.ts     # Orchestrator → subagent pattern
│       ├── session-chain.ts   # Chained sessions via conversationId
│       └── tool-monitor.ts    # Real-time tool_use event capture
├── computer-use-demo/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts           # CLI entry point
│       ├── agent.ts           # ForgeCode query with screenshot loop
│       └── screen-capture.ts  # OS screenshot → MiniMax MCP describe → text action
├── computer-use-best-practices/
│   ├── package.json
│   ├── tsconfig.json
│   └── src/
│       ├── index.ts           # CLI entry point
│       ├── agent.ts           # Best-practice patterns: caching, batching, pruning
│       └── screen-capture.ts  # Screenshot capture + MiniMax MCP describe
└── shared/
    ├── stream-helpers.ts      # Shared streaming/output utilities
    ├── forge-client.ts        # Configured query wrapper with defaults
    └── mcp-config.ts          # MCP server definitions (MiniMax default + swappable)
```

---

## 4. Per-Quickstart Implementation Plan

### 4.1 `customer-support-agent`

**What it does:** An AI customer support chat interface. The agent answers questions using a knowledge base (product docs, FAQs). Demonstrates knowledge retrieval + conversational memory.

**TypeScript adaptation with ForgeCode SDK:**

- Embed the knowledge base as context in the `systemPrompt` option
- Use `conversationId` to maintain multi-turn chat sessions
- Stream `assistant` messages to the terminal or a simple HTTP interface
- Expose a readline CLI or a lightweight Express chat endpoint

**Key files:**

| File | Purpose |
|---|---|
| `src/knowledge-base.ts` | Load a JSON/markdown knowledge base from disk |
| `src/agent.ts` | `query()` call with system prompt containing KB context + `conversationId` for sessions |
| `src/index.ts` | Readline REPL loop or Express `/chat` endpoint with SSE streaming |

**Modality:** Text-to-text only. No images involved — knowledge base is plain text/JSON.

**Dependencies:** `@imbios/forgecode-sdk`, `express`, `readline` (built-in)
**Dependencies:** `@imbios/forgecode-sdk` (no extra HTTP or readline deps — `Bun.serve()` and `Bun.stdin` are built-in)

**Implementation steps:**
1. Create `package.json` from the quickstart template (pnpm + oxlint/oxfmt scripts)
2. Load knowledge base from `data/knowledge-base.json` via `Bun.file().json()`
3. Build `systemPrompt` that injects KB content
4. In `agent.ts`, wrap `query()` to track `conversationId` for session continuity
5. Implement readline REPL in `index.ts` using `Bun.stdin` line iteration
6. Add optional HTTP chat endpoint via `Bun.serve()` with streaming `Response` (no express)
---

### 4.2 `financial-data-analyst`

**What it does:** Chat interface for financial data analysis. User uploads/references a CSV dataset; Claude analyzes it and generates chart recommendations or Python/SQL queries. Includes an interactive visualization.

**TypeScript adaptation with ForgeCode SDK:**

**TypeScript adaptation with ForgeCode SDK:**
- Load CSV/JSON financial data and inject as context in the prompt using `Bun.file()`
- Use ForgeCode's structured output (`outputFormat` + Zod) to request JSON chart specs
- Render charts using Chart.js in a `Bun.serve()`-served HTML frontend (HTML imports)
- Stream analysis text via native `ReadableStream` / `Response` streaming

**Key files:**

| File | Purpose |
|---|---|
| `src/data-loader.ts` | Parse CSV/JSON datasets using `Bun.file()` + built-in text splitting (no papaparse) |
| `src/agent.ts` | `query()` with data context injected in prompt, Zod schema for chart spec output |
| `src/index.ts` | `Bun.serve()` server: `POST /analyze` streams agent response, `GET /` serves HTML UI |
| `src/public/index.html` | Chart.js frontend consuming the streamed response |

**Modality:** Text-to-text only. Raw CSV/JSON data is stringified and injected as text context. Chart rendering is client-side (Chart.js) from a JSON spec; no image is ever sent to the model.

**Dependencies:** `@imbios/forgecode-sdk`, `zod` (no express, no papaparse — Bun built-ins cover both)

**Implementation steps:**
1. Implement CSV loader in `data-loader.ts` using `Bun.file().text()` + manual line splitting
2. Define Zod schema for chart spec output (`{ type, labels, datasets, title }`)
3. In `agent.ts`, call `query()` with data context and `outputFormat` for structured chart JSON
4. Build server in `index.ts` with `Bun.serve()`: `POST /analyze` returns a streaming `Response`, `GET /` serves `index.html` via HTML import
5. Build Chart.js `index.html` + `frontend.ts` (Bun bundles automatically via HTML imports)
---

### 4.3 `browser-use-demo`

**What it does:** Gives Claude the ability to navigate websites, interact with DOM, extract content, and fill forms via Playwright.

**TypeScript adaptation with ForgeCode SDK:**

- ForgeCode agent natively supports MCP servers; use an MCP-based Playwright server to expose browser tools
- Alternatively, implement browser operations as part of a prompted workflow where the forge agent issues Playwright commands through a custom tool
- Provide a system prompt instructing the agent to perform web tasks
- Monitor `tool_use` events to log browser actions in real-time

**Key files:**

| File | Purpose |
|---|---|
| `src/browser-tools.ts` | Launch Playwright + expose navigate/click/type/extract functions |
| `src/agent.ts` | `query()` with browser task system prompt; log `tool_use` events |
| `src/index.ts` | CLI: accept a task string, run the agent, print result |

**Modality:** Text-to-text for navigation, clicking, and content extraction. If the task requires reading visual page content (e.g. CAPTCHA, canvas-rendered text, images), the agent calls the **MiniMax MCP** tool to describe the screenshot; the description is returned as text and fed back into the text-only agent loop.

**MCP servers used:**

- Playwright MCP (browser control tools)
- MiniMax MCP (screenshot-to-text, only when visual reading is required)

**Key files:**

| File | Purpose |
|---|---|
| `src/browser-tools.ts` | Launch Playwright + expose navigate/click/type/extract functions |
| `src/agent.ts` | `query()` with browser task system prompt + `mcpServers: [playwrightMcp, minimaxMcpServer]` |
| `src/index.ts` | CLI: accept a task string, run the agent, print result |

**Dependencies:** `@imbios/forgecode-sdk`, `playwright` (pnpm add playwright; `pnpx @playwright/mcp` invoked at runtime — not installed as a dep)

**Implementation steps:**

1. Install Playwright + browser binaries
2. In `browser-tools.ts`, create helper functions for core browser operations
3. Write system prompt that instructs the agent: use text extraction first; only call the MiniMax MCP `describe_image` tool when text extraction is insufficient
4. In `agent.ts`, load both `playwrightMcp` and `minimaxMcpServer` via `mcpServers` option; capture `tool_use` events to log what the agent is doing
5. CLI entry point accepts a task description and prints a final summary

---

### 4.4 `autonomous-coding`

**What it does:** Two-agent pattern. Agent 1 (initializer) scaffolds a project and writes a feature list. Agent 2 (coder) works through the feature list incrementally, persisting progress via git commits.

**TypeScript adaptation with ForgeCode SDK:**
This is the most natural fit for ForgeCode SDK since `forge` is specifically a coding agent.

- **Initializer agent:** Uses `query()` with a prompt to scaffold the project and generate `FEATURES.md`
- **Coding agent:** Loops over features in `FEATURES.md`, calling `query()` per feature with `conversationId` for continuity; commits after each feature
- Git helpers wrap `git add/commit` calls

**Key files:**

| File | Purpose |
|---|---|
| `src/initializer.ts` | `query()` call to scaffold project + emit `FEATURES.md` |
| `src/coder.ts` | Read `FEATURES.md`, iterate features, call `query()` per feature, git commit |
| `src/git-helpers.ts` | `commitFeature(name)` wraps `git add . && git commit -m "feat: <name>"` |
| `src/index.ts` | CLI: `--init` flag for initializer, default runs coder loop |

**Modality:** Text-to-text only. The coding agent reads and writes source files as text; no image input at any step.

**Dependencies:** `@imbios/forgecode-sdk`, `simple-git`

**Implementation steps:**

1. Implement `initializer.ts`: send a structured prompt to create project scaffold + write feature list to `FEATURES.md`
2. Implement `coder.ts`: parse `FEATURES.md`, loop features, call `query()` with feature description, commit on completion
3. Use `conversationId` from the first turn to maintain context across feature iterations
4. Implement `git-helpers.ts` using `simple-git` for clean git operations
5. CLI entry point with `--init` / `--build` flags

---

### 4.5 `agents`

**What it does:** General agent orchestration patterns — multi-agent pipelines, tool monitoring, session chaining.

**TypeScript adaptation with ForgeCode SDK:**
Showcase the ForgeCode SDK's streaming and session features directly:

| Demo | Description |
|---|---|
| `multi-agent.ts` | Orchestrator agent fans out to N specialized subagents via parallel `query()` calls |
| `session-chain.ts` | Multi-turn conversation using `conversationId` — show session persistence |
| `tool-monitor.ts` | Monitor all `tool_use` events across an agentic run and log tool invocations |

**Modality:** Text-to-text only. All agent patterns operate on text prompts and text results.

**Dependencies:** `@imbios/forgecode-sdk`

**Implementation steps:**

1. `multi-agent.ts`: orchestrator sends decomposed subtasks to parallel `query()` calls, merges results
2. `session-chain.ts`: demonstrates `conversationId` passing across turns for stateful conversations
3. `tool-monitor.ts`: runs a complex prompt and logs every `tool_use` event with timing
4. `index.ts` demo runner that lets user choose which pattern to run

---

### 4.6 `computer-use-demo`

**What it does:** Desktop automation — take a screenshot of the screen, understand what is visible, issue mouse/keyboard actions, repeat.

**TypeScript adaptation with ForgeCode SDK + MiniMax MCP:**

Instead of sending raw screenshot bytes directly to the model (which would require a vision-capable model), this quickstart uses a two-step loop:

1. **Capture** — take a screenshot using `screenshot-desktop` or `@nut-tree/nut-js`
2. **Describe** — call the **MiniMax MCP** `describe_image` tool; it returns a text description of what is on screen
3. **Act** — the text-only forge agent reads the description and decides the next action (click, type, scroll) which is executed via `@nut-tree/nut-js`

This keeps the main agent loop fully text-to-text while still supporting visual understanding.

**Key files:**

| File | Purpose |
|---|---|
| `src/screen-capture.ts` | Capture screenshot to a temp file; call MiniMax MCP to get text description |
| `src/agent.ts` | `query()` loop: inject screen description as text context, parse action from result |
| `src/index.ts` | CLI entry: run loop until task complete or max turns reached |

**MCP servers used:** MiniMax MCP (image-to-text)

**Dependencies:** `@imbios/forgecode-sdk`, `screenshot-desktop`, `@nut-tree/nut-js`

**Implementation steps:**

1. Implement `screen-capture.ts`: capture screenshot → save to temp PNG → pass path to MiniMax MCP `describe_image` → return description string
2. In `agent.ts`, build the action loop: describe screen → `query()` with description → parse action (e.g. `{ action: "click", x: 100, y: 200 }`) from structured output → execute → repeat
3. Use Zod schema for structured action output from the agent
4. CLI entry with `--task` flag and `--max-turns` safety cap

---

### 4.7 `computer-use-best-practices`

**What it does:** Same desktop automation as above, but demonstrates best-practice patterns: image pruning (don't keep all screenshots in context), prompt caching, batched actions, trajectory recording.

**TypeScript adaptation with ForgeCode SDK + MiniMax MCP:**

Same MiniMax MCP screenshot-to-text bridge as `computer-use-demo`, plus:

- **Image pruning:** only the last N screen descriptions are kept in the session context (passed via the system prompt, not as images)
- **Prompt caching:** `conversationId` reuse avoids re-describing unchanged parts of the screen
- **Batched actions:** agent emits a JSON array of actions per turn instead of one at a time
- **Trajectory recording:** every (description, action) pair is appended to `trajectory.jsonl` for debugging and replay

**Key files:**

| File | Purpose |
|---|---|
| `src/screen-capture.ts` | Screenshot + MiniMax MCP describe (shared with `computer-use-demo`) |
| `src/trajectory.ts` | Append `{ turn, description, actions, timestamp }` to `trajectory.jsonl` |
| `src/agent.ts` | Loop with sliding window of last N descriptions; batched action Zod schema |
| `src/index.ts` | CLI with `--task`, `--max-turns`, `--window-size`, `--replay` flags |

**MCP servers used:** MiniMax MCP (image-to-text)

**Dependencies:** `@imbios/forgecode-sdk`, `screenshot-desktop`, `@nut-tree/nut-js`, `zod`

**Implementation steps:**

1. Reuse `screen-capture.ts` from `computer-use-demo`
2. Implement `trajectory.ts` for JSONL recording
3. In `agent.ts`, maintain a sliding window array of last N text descriptions; include only these in the prompt context
4. Define a batched action Zod schema: `{ actions: Array<{ type, x?, y?, text?, key? }> }`
5. Implement `--replay` mode that reads `trajectory.jsonl` and replays actions without calling the agent

---

## 5. Shared Infrastructure

### 5.1 `shared/forge-client.ts`

A configured wrapper around `query()` with project-wide defaults:

```typescript
import { query, type QueryOptions } from "@imbios/forgecode-sdk";

export const defaultOptions: Partial<QueryOptions> = {
  agent: "forge",
  model: "claude-opus-4-5",
};

export async function* forgeQuery(prompt: string, opts: Partial<QueryOptions> = {}) {
  yield* query(prompt, { ...defaultOptions, ...opts });
}
```

### 5.2 `shared/mcp-config.ts`

Centralized MCP server definitions. Any quickstart that needs vision imports from here.

**Prerequisites (manual setup required):**

- `uvx` must be installed (`pip install uv` or `brew install uv`)
- `MINIMAX_API_KEY` and `MINIMAX_API_HOST` must be set in `.env`
- MiniMax MCP is **not auto-installed** — `uvx` fetches `minimax-coding-plan-mcp` on first run

```typescript
// No dotenv import — Bun automatically loads .env files

/**
 * MiniMax MCP — default image-to-text provider.
 * Package: minimax-coding-plan-mcp (fetched via uvx on first use)
 * Manual setup required: uvx + MINIMAX_API_KEY env var.
 */
export const minimaxMcpServer = {
  name: "MiniMax",
  command: "uvx",
  args: ["minimax-coding-plan-mcp"],
  env: {
    MINIMAX_API_HOST: process.env.MINIMAX_API_HOST ?? "https://api.minimax.io",
    MINIMAX_API_KEY: process.env.MINIMAX_API_KEY ?? "",
  },
};

/** Playwright MCP — browser control tools */
export const playwrightMcpServer = {
  name: "playwright",
  command: "pnpx",
  args: ["@playwright/mcp"],
};

/**
 * Returns the MiniMax MCP config.
 * Throws if required env vars are not set, so callers fail fast
 * rather than silently hitting the MCP with an empty key.
 */
export function requireImageMcp() {
  if (!process.env.MINIMAX_API_KEY) {
    throw new Error(
      "MINIMAX_API_KEY is required for image-to-text features.\n"
      + "Set MINIMAX_API_KEY (and optionally MINIMAX_API_HOST) in .env\n"
      + "and ensure uvx is installed (pip install uv)."
    );
  }
  return minimaxMcpServer;
}
```

**Companion `.mcp.json.example`** (commit this; users copy to `.mcp.json` and fill in their key):

```json
{
  "mcpServers": {
    "MiniMax": {
      "command": "uvx",
      "args": ["minimax-coding-plan-mcp"],
      "env": {
        "MINIMAX_API_HOST": "https://api.minimax.io",
        "MINIMAX_API_KEY": "YOUR_MINIMAX_API_KEY_HERE"
      },
      "disable": false
    }
  }
}
```

> **Swapping the image MCP:** Any MCP server that exposes image description tools is compatible. Update `shared/mcp-config.ts` and `.mcp.json` to point to your server.

### 5.3 `shared/stream-helpers.ts`

Utilities for collecting streamed output and printing to terminal:

```typescript
// Collect all assistant tokens into a single string
export async function collectText(gen: AsyncGenerator): Promise<string>

// Print tokens to stdout as they arrive
export async function printStream(gen: AsyncGenerator): Promise<string>

// Convert agent stream to a native ReadableStream for Bun.serve() Response
// Usage: return new Response(agentToStream(gen), { headers: { "Content-Type": "text/plain" } })
export function agentToStream(gen: AsyncGenerator): ReadableStream<Uint8Array>
```

> No SSE helper needed — `Bun.serve()` accepts a `ReadableStream` directly as a `Response` body.

### 5.4 Root `package.json` (pnpm workspaces)

```json
{
  "name": "forgecode-sdk-quickstarts",
  "private": true,
  "scripts": {
    "lint": "pnpm -r run lint",
    "fmt": "pnpm -r run fmt",
    "fmt:check": "pnpm -r run fmt:check"
  },
  "devEngines": {
    "packageManager": {
      "name": "pnpm",
      "version": "^11.1.2",
      "onFail": "download"
    }
  }
}
```

Workspace members are declared in `pnpm-workspace.yaml`:

```yaml
# pnpm-workspace.yaml
packages:
  - "shared"
  - "customer-support-agent"
  - "financial-data-analyst"
  - "browser-use-demo"
  - "autonomous-coding"
  - "agents"
  - "computer-use-demo"
  - "computer-use-best-practices"
```

### 5.5 Per-package `package.json` template

Every quickstart package follows the template established in `quickstart-template/package.json`:

```json
{
  "name": "<quickstart-name>",
  "version": "1.0.0",
  "private": true,
  "type": "module",
  "main": "src/index.ts",
  "module": "src/index.ts",
  "scripts": {
    "start": "bun run src/index.ts",
    "typecheck": "tsc --noEmit",
    "lint": "oxlint",
    "lint:fix": "oxlint --fix",
    "fmt": "oxfmt",
    "fmt:check": "oxfmt --check"
  },
  "dependencies": {
    "@imbios/forgecode-sdk": "latest"
  },
  "devDependencies": {
    "@types/bun": "latest",
    "oxfmt": "^0.51.0",
    "oxlint": "^1.66.0",
    "typescript": "^5"
  },
  "devEngines": {
    "packageManager": {
      "name": "pnpm",
      "version": "^11.1.2",
      "onFail": "download"
    }
  }
}
```

### 5.6 `tsconfig.json` (per-package, matches template)

```json
{
  "compilerOptions": {
    "lib": ["ESNext"],
    "target": "ESNext",
    "module": "Preserve",
    "moduleDetection": "force",
    "jsx": "react-jsx",
    "allowJs": true,
    "types": ["bun"],

    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "verbatimModuleSyntax": true,
    "noEmit": true,

    "strict": true,
    "skipLibCheck": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,

    "noUnusedLocals": false,
    "noUnusedParameters": false,
    "noPropertyAccessFromIndexSignature": false
  }
}
```

Key differences from default configs:
- `"module": "Preserve"` — Bun's recommended module setting (not `"ESNext"`)
- `"moduleDetection": "force"` — treats every file as a module
- `"types": ["bun"]` — Bun global types; no `@types/node` needed
- `"verbatimModuleSyntax": true` — enforces `import type` for type-only imports
- `"allowImportingTsExtensions": true` — allows `import "./foo.ts"` directly

---

## 6. Implementation Priority Order

| Priority | Quickstart | Modality | Rationale |
|---|---|---|---|
| 1 | `agents` | Text only | Core SDK patterns; foundational for all others |
| 2 | `customer-support-agent` | Text only | Simplest full-stack demo; no external vision deps |
| 3 | `autonomous-coding` | Text only | Best natural fit for ForgeCode (coding agent) |
| 4 | `financial-data-analyst` | Text only | Adds structured output + `Bun.serve()` + Chart.js frontend |
| 5 | `browser-use-demo` | Text + MiniMax MCP (visual fallback) | Playwright MCP + MiniMax MCP for screenshot reading |
| 6 | `computer-use-demo` | Text + MiniMax MCP | MiniMax MCP bridges screenshot → text description |
| 7 | `computer-use-best-practices` | Text + MiniMax MCP | Same bridge + pruning/caching/batching patterns |

---

## 7. Environment Setup

### 7.1 Prerequisites

```bash
# 1. Install Bun (runtime)
curl -fsSL https://bun.sh/install | bash

# 2. Install pnpm (package manager)
curl -fsSL https://get.pnpm.io/install.sh | sh
# or: npm install -g pnpm

# 3. Install forge CLI
curl -fsSL https://forgecode.dev/cli | sh

# 4. Verify
bun --version
pnpm --version
forge --version
```

### 7.2 Manual MiniMax MCP Setup

MiniMax MCP is **not built in** to the ForgeCode SDK or forge CLI. Before running any quickstart that uses vision, complete these steps once:

```bash
# Step 1: install uv (provides the uvx command)
pip install uv
# or: brew install uv  (macOS)
# or: curl -LsSf https://astral.sh/uv/install.sh | sh

# Step 2: verify uvx can reach the package (dry run)
uvx minimax-coding-plan-mcp --help

# Step 3: copy and fill in the MCP config for forge CLI
cp .mcp.json.example .mcp.json
# then edit .mcp.json and set MINIMAX_API_KEY
```

### 7.3 Environment Variables

```bash
# .env (root — inherited by all workspaces)

# Required for all quickstarts
FORGE_PATH=~/.local/bin/forge          # path to forge binary (if not on PATH)

# Required only for quickstarts that use MiniMax MCP (browser-use, computer-use-*)
MINIMAX_API_KEY=your_minimax_api_key
MINIMAX_API_HOST=https://api.minimax.io  # default; override if using a different region

# Optional — swap the image MCP provider
# MCP_IMAGE_PROVIDER=custom
```

### 7.4 Per-project setup pattern

Text-only quickstarts (priorities 1–4):
Text-only quickstarts (priorities 1–4):
```bash
pnpm install
cp .env.example .env    # only FORGE_PATH needed
bun run src/index.ts
```

Vision quickstarts (priorities 5–7, require MiniMax MCP):
```bash
pnpm install
cp .env.example .env    # set FORGE_PATH + MINIMAX_API_KEY + MINIMAX_API_HOST
cp .mcp.json.example .mcp.json  # fill in MINIMAX_API_KEY
bun run src/index.ts
```

Quickstarts that require MiniMax MCP call `requireImageMcp()` at startup and throw a descriptive error if `MINIMAX_API_KEY` is missing, rather than failing silently mid-run.

> **Package manager vs runtime split:** `pnpm` manages dependencies (`pnpm install`, `pnpm add`). `bun` runs the code (`bun run src/index.ts`, `bun test`). Never use `bun install` or `npm install`.
---

## 8. Key Compatibility Notes

| Concern | Notes |
|---|---|
| **No raw `anthropic` client** | The ForgeCode SDK does NOT expose `client.messages.create()`. All calls go through `query()`. |
| **Text-to-text default** | No quickstart passes image bytes to the main model. Vision is always delegated to MiniMax MCP (or a user-provided substitute). |
| **MiniMax MCP is not built-in** | Must be manually set up: install `uvx`, copy `.mcp.json.example` → `.mcp.json`, set `MINIMAX_API_KEY`. |
| **MiniMax MCP package** | `minimax-coding-plan-mcp`, fetched by `uvx` at runtime. Not `@minimax/mcp` or any npm package. |
| **MiniMax MCP env vars** | `MINIMAX_API_KEY` and `MINIMAX_API_HOST` (default: `https://api.minimax.io`). No `MINIMAX_GROUP_ID`. |
| **MCP servers are optional** | Quickstarts 1–4 (`agents`, `customer-support-agent`, `autonomous-coding`, `financial-data-analyst`) require no MCP servers — they are pure text-to-text. |
| **Tool loops are internal** | ForgeCode agent handles tool invocations internally. You observe `tool_use` events but don't dispatch tool results manually. |
| **Session ID ≠ Anthropic session** | `conversationId` is ForgeCode's session identifier, not an Anthropic session ID. |
| **Binary dependency** | The `forge` CLI must be installed on the host machine. This is a hard requirement for all quickstarts. |
| **PNPM for packages, Bun for runtime** | `pnpm install` / `pnpm add` for dependency management. `bun run` / `bun test` for execution. Never mix — `bun install` is not used. |
| **No express** | Use `Bun.serve()` for HTTP servers. It supports routes, streaming responses, WebSockets, and HTML imports natively. |
| **No dotenv** | Bun automatically loads `.env` files. Never `import { config } from "dotenv"`. |
| **No papaparse / node:fs readFile** | Use `Bun.file().text()` or `Bun.file().json()` for file I/O. Use `bun:sqlite` for SQLite. |
| **oxlint + oxfmt** | All packages use `oxlint` for linting and `oxfmt` for formatting. No ESLint or Prettier. Run `pnpm run lint` and `pnpm run fmt`. |
| **Bun runtime** | The TypeScript SDK is optimized for Bun. Node.js compatibility is not guaranteed by the SDK. |
| **Streaming** | `assistant` messages are individual tokens; accumulate them for full text. |
| **Structured output** | Use `outputFormat` + Zod schemas instead of Claude's JSON mode / `tool_use` schema workaround. |
| **MCP swap** | Update `minimaxMcpServer` in `shared/mcp-config.ts` and the matching `.mcp.json` entry to point to any other vision-capable MCP server. |

---

## 9. Open Questions (resolve before implementation starts)

| # | Question | Why it matters |
|---|---|---|
| Q1 | Does `QueryOptions` include `mcpServers`? | Used in 3 quickstarts; need fallback plan if absent |
| Q2 | Does `QueryOptions` include `outputFormat` + Zod integration? | Used in `financial-data-analyst` and `computer-use-*` |
| Q3 | Does `@nut-tree/nut-js` run under Bun? | Native addon package; may need Node.js compatibility shim |
| Q4 | Does `screenshot-desktop` run under Bun? | Same concern — native binary wrapper |
| Q5 | Does `simple-git` run under Bun? | Uses `child_process` — likely fine, but confirm |
| Q6 | What model name to use as default? | `claude-opus-4-5` is in the plan; `claude-sonnet-4-5` may be better for quickstarts (cost + speed) |
| Q7 | Should `shared/` have a root `tsconfig.json` with project references? | Needed for `tsc --noEmit` to work correctly across the monorepo |

---

## 10. Growth Strategy — 1K Stars in a Week

### 10.1 Why this repo can go viral

This repo has structural advantages that map directly to GitHub trending mechanics:

- **First-mover:** No TypeScript-native quickstarts repo exists for ForgeCode SDK yet
- **Coattail effect:** Anthropic's `claude-quickstarts` already has thousands of stars; this is a direct TypeScript port of a known, popular resource
- **Stack alignment:** Bun + pnpm + TypeScript + oxlint is the exact stack the current TypeScript community gravitates toward in 2026
- **MCP angle:** MCP is a hot topic — any repo prominently featuring MCP integration gets organic search traffic

### 10.2 Pre-launch checklist (technical)

These must be done before any public announcement:

- [ ] Every quickstart runs with a single command (`bun run src/index.ts`)
- [ ] Root-level `README.md` with a demo GIF or asciinema recording for each quickstart
- [ ] `AGENTS.md` at repo root (forge agents read this; it signals the repo is forge-native)
- [ ] `.devcontainer/devcontainer.json` — one-click GitHub Codespaces launch (zero local setup)
- [ ] GitHub Actions CI: `pnpm install && pnpm run typecheck && pnpm run lint`
- [ ] Issue + PR templates in `.github/`
- [ ] `CONTRIBUTING.md` — makes it easy for others to add new quickstarts
- [ ] Each quickstart has a demo screenshot or GIF in its own `README.md`
- [ ] `awesome-claude` submission prepared (PR draft ready)

### 10.3 Launch sequence

**Day 0 (prep):**
- Record a 60-second screen recording showing all 7 quickstarts running
- Prepare a thread for each platform — write, don't improvise

**Day 1 (launch):**
1. Post to **Hacker News** as `Show HN: TypeScript quickstarts for ForgeCode SDK — Bun-native, MCP-first` — submit between 09:00–11:00 US Eastern for maximum visibility
2. Post a **Twitter/X thread** tagging `@anthropic`, `@imbIOS` (ForgeCode), `@bunjavascript`, `@pnpmjs` — include the asciinema demo
3. Post to **r/LocalLLaMA**, **r/typescript**, **r/MachineLearning** — different angles per subreddit

**Day 2–3 (amplification):**
4. Submit to **awesome-anthropic** and **awesome-claude** lists via PR
5. Write a `dev.to` article: _"Building AI agents with TypeScript, Bun, and ForgeCode SDK"_ — link back to the repo
6. Post in **Anthropic Discord** `#community-projects` channel and relevant TypeScript Discord servers

**Day 4–7 (maintenance window):**
7. Respond to every issue within 24h — GitHub's trending algorithm favors active repos
8. Merge community PRs quickly (typo fixes, new quickstarts) — activity signals compound
9. Pin the repo to your GitHub profile

### 10.4 Content hooks by platform

| Platform | Angle |
|---|---|
| Hacker News | "No Python required" + Bun runtime performance numbers |
| Twitter/X | Demo GIF + "7 AI agent patterns, TypeScript-native, runs with one command" |
| Reddit r/LocalLLaMA | MCP-first design — swap any model, swap any image MCP |
| Reddit r/typescript | Bun + oxlint + oxfmt stack; `verbatimModuleSyntax`; modern TypeScript setup |
| Dev.to / Medium | Tutorial-style walkthrough of `autonomous-coding` (most impressive demo) |
| Discord (Anthropic) | "Port of claude-quickstarts in TypeScript using ForgeCode SDK" |

### 10.5 Repo surface-area optimisations

- **Topics/tags on GitHub:** `typescript`, `bun`, `anthropic`, `claude`, `ai-agents`, `mcp`, `forgecode`, `llm`, `computer-use`, `playwright`
- **Homepage URL:** link to the ForgeCode SDK docs or forgecode.dev
- **Social preview image:** a clean screenshot showing multiple quickstarts running with the ForgeCode + Anthropic logos
- **Star prompt in README:** _"If this saved you time, a ⭐ helps others find it"_ — one line, not aggressive
- **`used-by` section in README:** once a few projects adopt it, list them — social proof accelerates stars

---

## 11. Verification Criteria

Each quickstart is considered complete when:

- [ ] TypeScript compiles without errors (`bun run typecheck` or `tsc --noEmit`)
- [ ] No lint errors (`pnpm run lint`)
- [ ] Code is formatted (`pnpm run fmt:check`)
- [ ] The application runs with `bun run src/index.ts`
- [ ] Agent responses stream in real-time to terminal or HTTP client
- [ ] Session continuity works (follow-up questions are coherent)
- [ ] Error cases are handled (invalid input, forge binary not found, abort)
- [ ] **No image bytes are sent to the main model** — verified by inspecting `tool_use` events (all vision calls go through MiniMax MCP tool)
- [ ] Running without `MINIMAX_API_KEY` on text-only quickstarts works fine; running without it on vision quickstarts calls `requireImageMcp()` which throws a clear, actionable error before any agent call is made
- [ ] `.mcp.json.example` and `.env.example` are committed; `.mcp.json` and `.env` (with real keys) are in `.gitignore`
- [ ] A `README.md` inside the quickstart directory explains setup + usage, and clearly marks whether `MINIMAX_API_KEY` is required
