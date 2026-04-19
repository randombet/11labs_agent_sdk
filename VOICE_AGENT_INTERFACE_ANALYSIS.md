# Voice Agent Integration Analysis

This document maps how voice input reaches the running agent in this repo, what interfaces are involved, and how to replace the voice layer without breaking the rest of the agent workflow.

Reference baseline: `CLAUDE.md` describes this as a pipeline architecture (ASR -> LLM -> TTS), and code confirms that the SDK transports audio/events while orchestration stays server-side.

## 1. Repo architecture relevant to voice flow

- `packages/client`: core runtime (`Conversation`, `VoiceConversation`, `TextConversation`, transport, audio I/O, event routing).
- `packages/react`: React integration (`ConversationProvider`, `useConversation`, control/status/input hooks).
- `packages/react-native`: platform-specific override of voice setup using `setSetupStrategy(...)`.
- `packages/types`: generated protocol contracts for incoming/outgoing socket events.
- `examples/agent-testbench`: practical wiring of `startSession(...)`, callbacks, and controls.

Core dependency direction:

- `@elevenlabs/types` -> `@elevenlabs/client` -> `@elevenlabs/react` / widget packages.

## 2. End-to-end voice input flow (web)

### 2.1 Session bootstrap

1. App calls `Conversation.startSession(options)`.
2. `packages/client/src/index.ts` routes to `VoiceConversation.startSession(...)` unless `textOnly` is true.
3. `VoiceConversation.startSession(...)`:
   - normalizes options via `BaseConversation.getFullOptions(...)`.
   - requests mic permission with preliminary `getUserMedia({ audio: true })`.
   - delegates transport + I/O setup to `setupStrategy(fullOptions)`.
4. Default strategy is `webSessionSetup` in `platform/VoiceSessionSetup.ts`.

### 2.2 Transport selection

`createConnection(...)` in `utils/ConnectionFactory.ts` chooses:

- `WebRTCConnection` by default for voice sessions.
- `WebSocketConnection` when `connectionType: "websocket"`, `signedUrl`, or `textOnly` inference applies.

### 2.3 Audio capture -> transport

There are two different paths:

#### A) WebSocket path (explicit PCM chunk forwarding)

- `MediaDeviceInput.create(...)` (`utils/input.ts`) creates AudioContext + worklet (`rawAudioProcessor`).
- Input worklet emits PCM chunks through `InputEventTarget` listener.
- `attachInputToConnection(...)` converts PCM bytes to base64 and sends:
  - `{ user_audio_chunk: "..." }`
- `connection.sendMessage(...)` forwards this event to orchestrator.

#### B) WebRTC path (track-based audio, not user_audio_chunk)

- `WebRTCConnection.create(...)` connects LiveKit room.
- Local mic track is published via LiveKit (`setMicrophoneEnabled(true)` / published track).
- `sendMessage(...)` in WebRTC connection ignores `user_audio_chunk` intentionally; audio flows through media track.

## 3. Where incoming agent events enter the workflow

Regardless of transport, incoming structured events converge in `BaseConversation`:

1. `BaseConnection.onMessage(...)` receives parsed incoming events.
2. `BaseConversation.onMessage(...)` switch-routes by `parsedEvent.type`.
3. Main workflow-relevant event handlers:
   - `agent_response` -> `onMessage({ source: "ai", ... })`
   - `user_transcript` -> `onMessage({ source: "user", ... })`
   - `client_tool_call` -> execute local tool, send `client_tool_result`
   - `agent_tool_request` / `agent_tool_response`
   - `mcp_tool_call` / `mcp_connection_status`
   - `interruption`, `vad_score`, `audio`, `error`, metadata events

This is the real "working agent workflow" boundary: once events are in `BaseConversation`, UI/hooks/tools follow a stable path independent of mic implementation.

## 3.1 Stateful vs stateless behavior

The workflow is **stateful per active conversation session**.

- Client and server events are exchanged over one active session connection and one `conversationId`.
- `BaseConversation` keeps in-memory session state (for example `status`, `mode`, `lastInterruptTimestamp`, `currentEventId`, feedback state).
- Incoming events are not treated as isolated requests; handlers can depend on prior events and update state for later ones.

How two consecutive events are handled:

1. They are sent on the same active connection/session.
2. They are routed one-by-one by `BaseConversation.onMessage(...)` based on event `type`.
3. Existing session state can affect handling of the second event.

Concrete examples:

- `interruption` updates `lastInterruptTimestamp`; later `audio` events are checked against that timestamp before playback callback/mode updates.
- `audio` updates `currentEventId`; later `sendFeedback(...)` uses that event id and prevents duplicate feedback for the same response.
- `mode`/`status` updates are incremental transitions, not stateless recomputation.

Ordering notes:

- Within one transport session, event delivery is intended to be in order for normal flow.
- `client_tool_call` handling is async; long-running tools can cause completion timing differences even if calls arrive consecutively. The protocol still links each result via `tool_call_id`.

## 4. Protocol/interface contract you must preserve

From `packages/types/generated/types/asyncapi-types.ts` and client event wrappers.

### 4.1 Outgoing events (client -> orchestrator)

Minimum important set:

- `UserAudio`: `{ user_audio_chunk: string }` (WebSocket audio path)
- `ConversationInitiation`: `{ type: "conversation_initiation_client_data", ... }`
- `UserMessage`: `{ type: "user_message", text }`
- `ContextualUpdate`: `{ type: "contextual_update", text }`
- `ClientToolResult`: `{ type: "client_tool_result", tool_call_id, result, is_error }`
- `McpToolApprovalResult`: `{ type: "mcp_tool_approval_result", ... }`
- `UserFeedback`, `UserActivity`, `MultimodalMessage`, `Pong`

### 4.2 Incoming events (orchestrator -> client)

Minimum important set:

- `conversation_initiation_metadata`
- `audio`
- `user_transcript`
- `agent_response`
- `interruption`
- `client_tool_call`
- `agent_tool_request` / `agent_tool_response`
- `mcp_tool_call` / `mcp_connection_status`
- `ping`, `error`

If your replacement keeps these event types/semantics, existing React hooks and tool workflow continue to work.

## 5. Runtime interfaces in code (adapter seams)

### 5.1 Transport seam: `BaseConnection`

Must provide:

- `conversationId`, `inputFormat`, `outputFormat`
- `sendMessage(message: OutgoingSocketEvent)`
- `close()`
- callbacks: `onMessage`, `onDisconnect`, `onModeChange`

This is the canonical boundary between network/voice transport and conversation workflow.

### 5.2 Input seam: `InputController`

Must provide:

- `setMuted`, `isMuted`, `setDevice`, `close`
- volume/frequency APIs for metering (`getVolume`, `getByteFrequencyData`)

### 5.3 Output seam: `OutputController`

Must provide:

- `setVolume`, `interrupt`, `setDevice`, `close`
- volume/frequency APIs

### 5.4 Platform override seam: `setSetupStrategy(...)`

`packages/client/src/internal.ts` exports `setSetupStrategy` and `webSessionSetup`.

React Native uses this exact seam in `packages/react-native/src/index.react-native.ts`:

- configure native audio session
- call `webSessionSetup(options)`
- wrap controllers/results (`attachNativeVolume(...)`)
- provide custom cleanup

This is the clean pattern to swap voice framework with minimal change to the agent workflow.

## 6. If you replace the voice framework, how to connect back

## Option A (recommended): keep ElevenLabs orchestrator protocol, replace platform voice plumbing

Use `setSetupStrategy(customStrategy)` and return `VoiceSessionSetupResult`:

- `connection: BaseConnection`
- `input: InputController`
- `output: OutputController`
- `playbackEventTarget` (or `null`)
- `detach()` cleanup

What this gives you:

- No changes needed in `BaseConversation`, React hooks, client tool handling, MCP approvals, feedback, mode/status callbacks.
- Existing controls (`sendUserMessage`, `sendContextualUpdate`, etc.) continue to work.

## Option B: external ASR/TTS framework, keep only text workflow

Start `Conversation` as text session (`textOnly: true` / websocket) and integrate your own voice stack outside SDK:

- external ASR -> call `sendUserMessage(text)` or `sendMultimodalMessage(...)`
- receive agent text via `onMessage`
- external TTS plays audio

Tradeoff:

- Fastest integration for non-Eleven voice runtime.
- Loses native audio-event coupling in SDK (`onAudio`, audio alignment, built-in interruption audio behavior).

## Option C: full custom transport bridge

Implement a new `BaseConnection` subclass that maps your framework protocol into Eleven-style incoming/outgoing events, then wire through setup strategy/factory.

Use when:

- your runtime cannot use LiveKit/WebSocket flows directly,
- but you still want existing SDK app-level abstractions.

## 7. Practical migration blueprint

1. Decide integration level:
   - A for minimal disruption.
   - B for text-only bridge.
   - C for deep protocol bridge.
2. Implement/wrap `InputController` and `OutputController` against your voice framework.
3. Ensure `sendMessage` and incoming event parsing satisfy `OutgoingSocketEvent` / `IncomingSocketEvent` expectations.
4. Register `setSetupStrategy(...)` in your platform entrypoint.
5. Validate with agent-testbench-style callbacks:
   - `onStatusChange`, `onModeChange`, `onMessage`, `onError`, tool call callbacks.
6. Confirm end-to-end:
   - mic audio reaches agent,
   - agent responses/tool calls arrive,
   - interruption/mode transitions behave correctly.

## 8. High-risk compatibility points

- Event shape drift: custom bridge must keep exact `type` + payload fields.
- Mode semantics: UI depends on `onModeChange` (`listening`/`speaking`).
- Interruption behavior: voice UX expects `interruption` to stop playback promptly.
- Tool loop reliability: `client_tool_call` must always produce `client_tool_result` (success/error).
- Device switching assumptions differ between WebRTC and WebSocket audio paths.

## 9. Key files to inspect when implementing

- `packages/client/src/index.ts`
- `packages/client/src/VoiceConversation.ts`
- `packages/client/src/BaseConversation.ts`
- `packages/client/src/platform/VoiceSessionSetup.ts`
- `packages/client/src/utils/ConnectionFactory.ts`
- `packages/client/src/utils/WebRTCConnection.ts`
- `packages/client/src/utils/WebSocketConnection.ts`
- `packages/client/src/utils/input.ts`
- `packages/client/src/utils/output.ts`
- `packages/client/src/utils/attachInputToConnection.ts`
- `packages/client/src/internal.ts`
- `packages/react/src/conversation/ConversationProvider.tsx`
- `packages/react/src/conversation/ConversationControls.tsx`
- `packages/react-native/src/index.react-native.ts`
- `packages/types/generated/types/asyncapi-types.ts`
- `CLAUDE.md`

## 10. Bottom line

The stable integration contract is event/protocol + `BaseConversation` callbacks, not the specific mic/audio implementation. The cleanest swap is to replace session setup and audio controllers via `setSetupStrategy(...)` while preserving `BaseConnection` event semantics.
