# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **recovered source code archive** of Claude Code CLI version 2.1.88, extracted from npm source maps. It is a TypeScript/React CLI application using the Ink terminal UI framework. The project is for **research and analysis purposes only** - it is not an official open-source release.

**Key Note**: This repository lacks build configuration (package.json, tsconfig.json, etc.). To make it runnable, you would need to:
1. Add `package.json` with dependencies
2. Set up the Bun bundler toolchain
3. Handle `bun:bundle` feature flag macros
4. Implement `MACRO.VERSION` and other build-time inlines

## High-Level Architecture

### Entry Points
- `src/entrypoints/cli.tsx` - Main CLI bootstrap with fast-path handling (--version, --dump-system-prompt)
- `src/entrypoints/init.ts` - Initialization logic for auth, configs, telemetry
- `src/entrypoints/mcp.ts` - MCP server entrypoint
- `src/main.tsx` - Full command parsing and REPL launch (800KB, heavy module)

### Command System (`src/commands/` and `src/commands.ts`)
Commands are registered in `commands.ts` with **feature flag gating**. Many commands are conditionally imported via `require()` based on `feature('FEATURE_NAME')` checks. Example:
```typescript
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null
```

Major commands: `login`, `mcp`, `review`, `commit`, `compact`, `config`, `memory`, `resume`, `agents`, `init`, `doctor`, `status`, etc.

### Tools System (`src/tools/` and `src/tools.ts`)
Core tools are defined as classes implementing the Tool interface. Key tools:
- **BashTool** - Shell command execution with sandbox support
- **FileReadTool/FileEditTool/FileWriteTool** - File operations
- **GlobTool/GrepTool** - File/search utilities (replaces `find`/`grep` CLI commands)
- **AgentTool** - Sub-agent spawning with built-in agents (explore, plan, general-purpose, verification)
- **MCPTool** - MCP server tool invocation
- **LSPTool** - Language Server Protocol integration
- **WebFetchTool/WebSearchTool** - HTTP/web operations
- **AskUserQuestionTool** - Interactive prompts
- **TaskCreateTool/TaskUpdateTool/TaskListTool** - Task tracking

### MCP Integration (`src/services/mcp/`)
Deep integration with Model Context Protocol:
- `config.ts` - MCP server configuration (stdio, SSE, HTTP, WebSocket transports)
- `client.ts` - Connection management and tool/resource discovery
- `MCPConnectionManager.tsx` - React hook for connection state
- Supports both local (stdio) and remote (SSE/HTTP/WebSocket) MCP servers

### Terminal UI (`src/ink/` and `src/components/`)
Custom fork/extension of Ink framework:
- `ink.tsx` - Main render loop
- `renderer.ts`, `render-to-screen.ts` - Output rendering
- `termio/` - ANSI/terminal I/O handling
- `components/` - React components (App, REPL screen, dialogs, etc.)

### Services (`src/services/`)
- `api/` - Claude API client, bootstrap, usage tracking
- `analytics/` - GrowthBook feature flags, event logging
- `compact/` - Conversation compaction/microcompact
- `oauth/` - OAuth authentication flow
- `lsp/` - LSP server management
- `SessionMemory/` - Session memory extraction

### State Management
- `src/bootstrap/state.ts` - Global state (CWD, model override, remote mode flags)
- `src/state/AppState.tsx` - React app state store
- `src/hooks/` - 100+ React hooks for UI state

## Key Technical Patterns

### Feature Flags (`bun:bundle`)
Build-time feature gating via `bun:bundle`. Dead code elimination removes unreachable code:
```typescript
import { feature } from 'bun:bundle'
if (feature('KAIROS')) {
  // KAIROS-specific code
}
```

Known flags: `DAEMON`, `KAIROS`, `VOICE_MODE`, `PROACTIVE`, `BRIDGE_MODE`, `COORDINATOR_MODE`, `WORKFLOW_SCRIPTS`, `CCR_REMOTE_SETUP`, etc.

### Conditional Imports
To avoid importing unnecessary modules, use dynamic requires:
```typescript
const module = feature('FEATURE')
  ? require('./path.js').default
  : null
```

### Settings/Configuration
- `src/utils/settings/` - Settings types, validation, managed paths
- Settings scopes: `local`, `user`, `project`, `dynamic`, `enterprise`, `claudeai`, `managed`
- MCP servers configured via JSON files at various scopes

### Security Patterns
- BashTool has extensive validation in `bashSecurity.ts`, `destructiveCommandWarning.ts`
- Permission handling via `src/hooks/toolPermission/`
- Sandbox runtime via `@anthropic-ai/sandbox-runtime`

## File Organization Summary

```
src/
├── entrypoints/      # CLI bootstrap and initialization
├── commands/         # 80+ CLI commands (login, mcp, review, etc.)
├── tools/            # Core tool implementations
├── services/         # API, MCP, analytics, oauth, lsp
├── components/       # React/Ink terminal UI components
├── hooks/            # React hooks for state management
├── ink/              # Custom Ink terminal rendering framework
├── utils/            # Auth, config, file operations, bash parsing
├── constants/        # Prompts, limits, product constants
├── state/            # App state management
├── skills/           # Bundled skills (simplify, loop, etc.)
├── plugins/          # Plugin system
├── coordinator/      # Coordinator mode (multi-agent)
├── tasks/            # Background task types
└── bootstrap/        # Global state initialization
```

## Notes for Analysis

- This code uses `MACRO.VERSION` which is inlined at build time
- The `process.env.USER_TYPE === 'ant'` checks indicate Anthropic-internal features
- Source maps were included accidentally in npm publish, enabling this recovery
- The codebase demonstrates sophisticated CLI architecture patterns for study