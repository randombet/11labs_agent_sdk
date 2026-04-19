# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is the ElevenLabs Agents SDK monorepo - JavaScript/TypeScript packages for building conversational AI agents. The SDK enables voice and text conversations via WebRTC/WebSocket connections using a **pipeline architecture** (ASR → LLM → TTS), not native audio-to-audio models like Gemini Live.

## Commands

```bash
# Install dependencies (uses pnpm workspaces)
pnpm install

# Build all packages
pnpm run build

# Run all tests
pnpm run test

# Run linting (eslint + prettier + type checking)
pnpm run lint

# Full CI check (types, lint, test, build)
pnpm run ci

# Start development mode (watch)
pnpm run dev

# Generate types from AsyncAPI schemas
pnpm run generate:types

# Create a new package
pnpm run create --name=my-new-package

# Create a changeset for releases
pnpm run changeset
```

### Package-specific commands

Run commands for a specific package using Turbo filters:

```bash
# Build specific package
pnpm turbo build --filter=@elevenlabs/client

# Test specific package
pnpm turbo test --filter=@elevenlabs/react

# Run single test file (package-dependent)
# client uses vitest:
cd packages/client && pnpm test src/BaseConversation.test.ts
# react/react-native use jest:
cd packages/react && pnpm test -- src/index.test.ts
```

## Architecture

### Package Dependency Graph

```
@elevenlabs/types (AsyncAPI schemas → generated TypeScript types)
       ↓
@elevenlabs/client (core conversation logic, WebRTC/WebSocket)
       ↓
┌──────┴──────┐
│             │
@elevenlabs/react    @elevenlabs/convai-widget-core
(useConversation     (Preact widget, uses client)
 React hook)                  ↓
                     @elevenlabs/convai-widget-embed
                     (bundled widget for embedding)

@elevenlabs/react-native (separate implementation using LiveKit RN)
       ↓
@elevenlabs/types (shared types only)
```

### Core Concepts

**Conversation Classes** (`packages/client/src/`):
- `BaseConversation` - Abstract base with event handling, message routing, feedback, and session lifecycle
- `VoiceConversation` - Extends base with audio I/O (Input/Output classes), WebRTC/WebSocket connections, wake lock
- `TextConversation` - Text-only mode without audio processing
- `Conversation.startSession()` - Factory that routes to Voice or Text based on `textOnly` option

**Connection Layer** (`packages/client/src/utils/`):
- `BaseConnection` - Interface for all connection types
- `WebRTCConnection` - LiveKit-based real-time audio streaming
- `WebSocketConnection` - Fallback/text-only connection
- `ConnectionFactory.createConnection()` - Selects connection type based on config

**React Integration** (`packages/react/src/`):
- `useConversation` hook wraps `Conversation` class with React state management
- Handles session lifecycle, controlled state (micMuted, volume), and callback routing
- Server location routing (us, eu-residency, in-residency, global)

**React Native** (`packages/react-native/src/`):
- Separate implementation using `@livekit/react-native` peer dependencies
- `ElevenLabsProvider` context with `useConversation` hook
- Platform-specific audio session handling

**Types Package** (`packages/types/`):
- AsyncAPI schemas define the agent communication protocol (`schemas/*.asyncapi.yaml`)
- Generated types in `generated/types/` (incoming/outgoing messages)
- Run `pnpm run generate:types` after schema changes

**Scribe** (`packages/client/src/scribe/`):
- Real-time transcription feature separate from conversation
- `Scribe` class with microphone or manual audio input
- Uses WebSocket connection with VAD (Voice Activity Detection)

### Voice Pipeline Architecture

This is a **pipeline-based** realtime system, NOT a native audio-to-audio model like Gemini Live API:

```
User Audio → [ASR] → Text → [LLM] → Text → [TTS] → Agent Audio
               ↓              ↓              ↓
          ElevenLabs    gemini-2.0-flash  eleven_turbo_v2
               ↓              ↓              ↓
        user_transcript  agent_response   audio event
```

**Pipeline stages** (configured in `conversation_config`):
- **ASR**: ElevenLabs speech-to-text (`asr.provider`, `asr.quality`)
- **LLM**: Text model like Gemini, GPT, Claude (`agent.prompt.llm`)
- **TTS**: ElevenLabs text-to-speech (`tts.model_id`, `tts.voice_id`)

The "realtime" aspect refers to streaming, VAD-based turn detection, and interruption handling - not native audio understanding.

### Tool System

**Client Tools** - Execute in browser/app, defined in `clientTools` config:
```typescript
// Server sends client_tool_call → SDK executes → sends client_tool_result
clientTools: {
  myTool: async (params) => { return "result"; }
}
```
Handling: `BaseConversation.handleClientToolCall()` at `packages/client/src/BaseConversation.ts:233`

**Agent Tools** - Execute server-side, SDK receives notifications only:
- `agent_tool_request` - Tool execution started
- `agent_tool_response` - Tool execution completed
- Types: `client`, `webhook`, `system`, `mcp`
- Special: `end_call` system tool terminates session

**MCP Tools** - Model Context Protocol integrations:
- States: `loading` → `awaiting_approval` → `success`/`failure`
- User approval via `sendMCPToolApprovalResult(toolCallId, isApproved)`
- Connection status tracked per integration

### Agent Configuration

**Session overrides** (`SessionConfig.overrides`):
- `agent.prompt.prompt` - System prompt override
- `agent.firstMessage` - Custom greeting
- `agent.language` - ASR/TTS language
- `tts.voiceId`, `tts.speed`, `tts.stability`

**Runtime parameters**:
- `dynamicVariables` - Key-value pairs for prompt templating
- `customLlmExtraBody` - Additional LLM provider parameters
- `userId` - User identification

**Agent transfers**: Handled server-side; SDK passes `dynamicVariables` which must include required variables for target agent (error: `missing_dynamic_variable_transfer`).

### Event Flow

1. Session starts → Connection established (WebRTC or WebSocket)
2. Incoming events parsed and routed by `BaseConversation.onMessage()`
3. Event types: `audio`, `agent_response`, `user_transcript`, `client_tool_call`, `interruption`, `ping`, etc.
4. Client tools execute locally and send results back via connection
5. Mode changes (listening/speaking) propagate through callbacks

### Build System

- **Turbo** orchestrates the monorepo build with dependency-aware caching
- **microbundle** bundles client/react/react-native packages (multiple formats: ESM, CJS, UMD)
- **Vite** builds widget packages (Preact-based)
- **tsc** compiles types package

### Testing

- `@elevenlabs/client` - Vitest with browser environment (Playwright)
- `@elevenlabs/react`, `@elevenlabs/react-native` - Jest with jsdom/react-native environments
- `@elevenlabs/convai-widget-core` - Vitest with browser environment
- `@elevenlabs/agents-cli` - Jest

### Release Process

Uses Changesets for versioning:
1. `pnpm run changeset` to create a changeset describing your change
2. Changesets action opens "Version Packages" PR automatically
3. Merge the PR to publish to npm
