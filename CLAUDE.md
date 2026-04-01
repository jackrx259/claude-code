# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

**Build** (produces `dist/cli.js`, ~22MB single ESM bundle):
```bash
bun run build.ts
```

**Run from source** (no build step needed):
```bash
bun src/entrypoints/cli.tsx
```

**Run compiled output**:
```bash
bun dist/cli.js
```

**Install dependencies**:
```bash
pnpm install
```

There are no test or lint scripts defined in `package.json`.

## Build System

- **Bundler**: Bun (not webpack/esbuild) — required because the build uses Bun's proprietary `bun:bundle` feature() API for compile-time feature flag tree-shaking
- **Package manager**: pnpm
- **Output**: Single file `dist/cli.js` with shebang, chmod 755
- **Build-time constants** injected via `MACRO.*`: `VERSION`, `BUILD_TIME`, `ISSUES_EXPLAINER`, etc.
- **Feature flags**: ~90+ boolean flags in `build.ts`. Most internal features are disabled in this external build. Enabled: `BUILTIN_EXPLORE_PLAN_AGENTS`, `TOKEN_BUDGET`, `MCP_SKILLS`. Disabled: `BRIDGE_MODE`, `KAIROS`, `DAEMON`, `VOICE_MODE`, `COORDINATOR_MODE`
- **Stubs**: Private/native packages are stubbed in `stubs/` (e.g., `color-diff-napi`, `modifiers-napi`, `@anthropic-ai/mcpb`)

## Architecture

### Entry & Initialization

```
src/entrypoints/cli.tsx       ← CLI bootstrap, fast-paths for --version, --daemon-worker, --bridge
    ↓
src/main.tsx                  ← Full REPL init (4,600+ lines): config, auth, MCP, analytics, tools, commands
    ↓
src/replLauncher.tsx          ← Launches interactive REPL
    ↓
src/components/REPL.tsx       ← Main interactive loop
```

### Core Systems

**Query Engine** (`src/QueryEngine.ts`):  
Orchestrates multi-turn Claude API calls. Manages message history, context windows, token budgets, tool execution/result injection, conversation compaction, and thinking mode.

**Tool System** (`src/Tool.ts` + `src/tools/`):  
49 tool implementations (BashTool, FileEditTool, FileReadTool, GlobTool, GrepTool, AgentTool, MCPTool, SkillTool, LSPTool, REPLTool, etc.). Each tool declares input/output Zod schemas and permission requirements.

**Task System** (`src/Task.ts`):  
Abstract task model supporting LocalBash, LocalAgent, RemoteAgent, InProcessTeammate, LocalWorkflow, MonitorMCP, Dream types. State machine: pending → running → completed/failed/killed. Output is persisted to disk.

**Commands** (`src/commands.ts` + `src/commands/`):  
105+ subdirectories, 207+ files implementing slash commands (`/commit`, `/review`, `/diff`, `/config`, `/mcp`, `/skills`, `/tasks`, etc.).

**Terminal UI**:  
- `src/ink/` — Custom React reconciler for terminal rendering (fork of Ink with deep modifications, 96 files)
- `src/components/` — 146 React components for the interactive UI
- `src/hooks/` — 87 React hooks for async ops, permissions, keybindings, state

**State** (`src/state/`, `src/context/`):  
React Context-based global AppState. Tracks messages, tool calls, tasks, sessions, settings, permissions.

**Services** (`src/services/`):  
39 subdirectories covering: Claude API, Files API, MCP server management, GrowthBook feature flags, analytics/telemetry, auth (OAuth, MDM, keychain), plugins, conversation compaction, remote sessions, voice input.

**Utilities** (`src/utils/`):  
330+ files. Key areas: shell execution, auth/OAuth (65KB), model info, git operations, settings/MDM, permissions, session history, file caching, cost tracking.

### Data Flow

```
User Input (REPL.tsx)
    → processUserInput() — parse, expand paths
    → QueryEngine.query() — orchestrate API call loop
        → getTools() — load available tools
        → Claude API — messages + tool definitions
        → Tool execution (Bash, File, Glob, Agent, MCP…)
        → Inject results → repeat until end_turn
    → Session persistence + analytics
    → Render in terminal
```

### Notable Patterns

- `src/main.tsx` is the single largest file and the central wiring point — most subsystem initialization lives here
- Feature flags in `build.ts` control dead-code elimination at build time; toggling them changes what ships in the bundle
- `vendor/` contains internal vendored code (not from npm)
- `patches/` contains a single pnpm patch for `commander@14.0.3` to allow multi-char short options
