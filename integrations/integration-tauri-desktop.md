# Amplifier Desktop (Tauri) Integration

## Overview

Amplifier Desktop is a Tauri-based native desktop application providing a full-featured AI coding assistant. It uses a three-tier architecture with WebSocket streaming for real-time communication between the React frontend and Python sidecar.

**Value Proposition:**
- Native desktop experience with system tray, notifications, keyboard shortcuts
- Offline-capable with local providers (Ollama)
- Real-time streaming UI with thinking blocks and tool execution visualization
- Cross-platform: macOS, Windows, Linux

---

## Quick Links

| Resource | URL |
|----------|-----|
| **Download** | [Latest Release (v1.1.0)](https://github.com/michaeljabbour/amplifier-desktop/releases/latest) |
| **Getting Started** | [Installation & First Steps](https://github.com/michaeljabbour/amplifier-desktop/blob/main/docs/GETTING_STARTED.md) |
| **User Guide** | [Complete Feature Reference](https://github.com/michaeljabbour/amplifier-desktop/blob/main/docs/USER_GUIDE.md) |
| **Developer Guide** | [Build from Source](https://github.com/michaeljabbour/amplifier-desktop/blob/main/docs/DEVELOPERS.md) |
| **GitHub** | [michaeljabbour/amplifier-desktop](https://github.com/michaeljabbour/amplifier-desktop) |

### Download Links

| Platform | Download |
|----------|----------|
| macOS (Apple Silicon) | [Amplifier_1.1.0_aarch64.dmg](https://github.com/michaeljabbour/amplifier-desktop/releases/latest/download/Amplifier_1.1.0_aarch64.dmg) |
| macOS (Intel) | [Amplifier_1.1.0_x64.dmg](https://github.com/michaeljabbour/amplifier-desktop/releases/latest/download/Amplifier_1.1.0_x64.dmg) |
| Windows (x64) | [Amplifier_1.1.0_x64-setup.exe](https://github.com/michaeljabbour/amplifier-desktop/releases/latest/download/Amplifier_1.1.0_x64-setup.exe) |
| Windows (ARM64) | [Amplifier_1.1.0_arm64-setup.exe](https://github.com/michaeljabbour/amplifier-desktop/releases/latest/download/Amplifier_1.1.0_arm64-setup.exe) |
| Linux (Debian/Ubuntu) | [amplifier_1.1.0_amd64.deb](https://github.com/michaeljabbour/amplifier-desktop/releases/latest/download/amplifier_1.1.0_amd64.deb) |

---

## Architecture

### Three-Tier Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           TAURI SHELL (Rust)                            │
│  - Window management, system tray, native menus                         │
│  - Sidecar process management (spawn, health check, restart)            │
│  - SQLite database (conversations, messages, settings)                  │
│  - IPC bridge between frontend and native APIs                          │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
┌───────────────────────────────┐   ┌───────────────────────────────────┐
│     REACT FRONTEND (TS)       │   │      PYTHON SIDECAR (FastAPI)     │
│                               │   │                                   │
│  - Zustand state management   │◄─►│  - AmplifierSession orchestration │
│  - WebSocket client           │WS │  - Provider loading (local/cached)│
│  - Markdown rendering         │   │  - Tool execution                 │
│  - Thinking/tool UI           │   │  - Hook event emission            │
│  - Settings/profiles UI       │   │  - Memory extraction              │
└───────────────────────────────┘   └───────────────────────────────────┘
```

### Component Locations

| Layer | Directory | Key Files |
|-------|-----------|-----------|
| Tauri Shell | `src-tauri/` | `lib.rs`, `main.rs`, `Cargo.toml` |
| React Frontend | `src/` | `App.tsx`, `components/`, `hooks/`, `lib/` |
| Python Sidecar | `sidecar/` | `server.py`, `local_modules.py`, `config.py` |

---

## Streaming Architecture

### Data Flow

```
User Input → WebSocket → Sidecar → AmplifierSession → Provider
                                                          │
                                                          ▼
UI Update ← WebSocket ← Hooks ←──────────────────── Streaming Events
```

### WebSocket Message Protocol

**Client → Server (Request Types)**

| Type | Purpose | Payload |
|------|---------|---------|
| `prompt` | Send user message | `{content, images?, history?, conversationId}` |
| `cancel` | Stop execution | `{conversationId}` |
| `clear` | Reset session | `{}` |
| `settings` | Sync settings | `{}` (triggers backend reload) |

**Server → Client (Response Types)**

| Type | Purpose | Payload |
|------|---------|---------|
| `delta` | Incremental text content | `{content: string, conversationId}` |
| `thinking` | Extended thinking content | `{content: string, conversationId}` |
| `tool_start` | Tool execution begins | `{content: toolName, metadata: {input, call_id}}` |
| `tool_end` | Tool execution complete | `{content: toolName, metadata: {output, success}}` |
| `done` | Response complete | `{content: response, metadata: {tokens, cost}}` |
| `error` | Error occurred | `{content: errorMessage}` |
| `cancelled` | Execution stopped | `{content: "Execution stopped by user"}` |

### Hook-to-WebSocket Mapping

The sidecar registers hooks on `AmplifierSession.coordinator.hooks` that convert provider events to WebSocket messages:

```python
# server.py - Hook Registration
hooks.register("content_block:delta", on_content_delta)   → type="delta"
hooks.register("thinking:delta", on_thinking_delta)       → type="thinking"
hooks.register("tool:pre", on_tool_pre)                   → type="tool_start"
hooks.register("tool:post", on_tool_post)                 → type="tool_end"
```

### Streaming vs Non-Streaming Providers

| Provider Source | Streaming Support | Event Emission |
|-----------------|-------------------|----------------|
| Local (`sidecar/providers/`) | Yes | Emits `content_block:delta` during API streaming |
| Cached (`~/.amplifier/module-cache/`) | No | Returns complete response |

**Non-Streaming Fallback:**

When using cached providers (Amplifier mode), the sidecar extracts content from the complete response and sends synthetic delta messages:

```python
# server.py - Fallback for non-streaming providers
if response and not self._streaming_happened:
    if response_thinking:
        await self._send(websocket, StreamMessage(type="thinking", content=response_thinking))
    if response_text:
        await self._send(websocket, StreamMessage(type="delta", content=response_text))
```

---

## Frontend State Management

### Zustand Store (`src/lib/store.ts`)

```typescript
interface ChatState {
  conversations: Conversation[];
  currentConversationId: string | null;

  // Actions for streaming
  addMessage: (msg: Message) => void;
  appendToMessage: (id: string, delta: string) => void;
  appendThinking: (id: string, delta: string) => void;
  addToolCall: (msgId: string, toolCall: ToolCall) => void;
  updateToolCall: (msgId: string, callId: string, update: Partial<ToolCall>) => void;
  updateMessage: (id: string, update: Partial<Message>) => void;
}

interface Message {
  id: string;
  role: 'user' | 'assistant';
  content: string;           // Accumulated from 'delta' messages
  thinking?: string;         // Accumulated from 'thinking' messages
  toolCalls?: ToolCall[];    // From tool_start/tool_end
  isStreaming: boolean;      // True while receiving
  contentParts?: ContentBlock[]; // Structured blocks from 'done'
}
```

### Message Accumulation Flow

```
1. User submits prompt
   └─► addMessage({id: streamingId, role: 'assistant', isStreaming: true, content: ''})

2. Delta arrives
   └─► appendToMessage(streamingId, delta.content)
   └─► UI re-renders with accumulated content

3. Thinking arrives
   └─► appendThinking(streamingId, thinking.content)
   └─► ThinkingBlock expands

4. Tool starts
   └─► addToolCall(streamingId, {name, input, status: 'running'})
   └─► ToolCallDisplay shows spinner

5. Tool ends
   └─► updateToolCall(streamingId, callId, {output, status: 'complete'})
   └─► ToolCallDisplay shows result

6. Done arrives
   └─► updateMessage(streamingId, {isStreaming: false, contentParts})
   └─► Final render with all content
```

---

## UI Components

### Component Hierarchy

```
ChatContainer
├── MessageList
│   └── ChatMessage (for each message)
│       ├── ThinkingBlock (collapsible)
│       ├── ToolCallDisplay[] (for each tool call)
│       │   ├── Tool name + input preview
│       │   ├── Spinner (while running)
│       │   └── Output (when complete)
│       └── MarkdownRenderer (main content)
│           ├── Code blocks with syntax highlighting
│           ├── Tables, lists, headings
│           └── Inline code, links
├── ChatInput
│   ├── Textarea with Ctrl+Enter submit
│   ├── Image attachment button
│   └── Send/Cancel button
└── StatusBar
    └── Token count, cost estimate, model info
```

### Key Component Files

| Component | File | Purpose |
|-----------|------|---------|
| ChatMessage | `src/components/ChatMessage.tsx` | Message container, role-based styling |
| ThinkingBlock | `src/components/ThinkingBlock.tsx` | Collapsible thinking content |
| ToolCallDisplay | `src/components/ToolCallDisplay.tsx` | Tool execution visualization |
| MarkdownRenderer | `src/components/MarkdownRenderer.tsx` | react-markdown with plugins |
| ChatInput | `src/components/ChatInput.tsx` | User input with image support |

---

## Provider Configuration

### Execution Modes

| Mode | Provider Source | Use Case |
|------|-----------------|----------|
| `local` | `sidecar/providers/` | Development, offline, custom providers |
| `amplifier` | `~/.amplifier/module-cache/` | Production, versioned modules |
| `hybrid` | Cache preferred, local fallback | Recommended default |

### Extended Thinking

Extended thinking (Claude's internal reasoning) requires:

1. **Model support**: Opus 4.5, Sonnet 4.5 with thinking capability
2. **Provider configuration**: `extended_thinking=True` kwarg
3. **UI handling**: ThinkingBlock component renders collapsed by default

```python
# local_modules.py - Enable thinking for capable models
if reasoning_cap:
    provider_kwargs["extended_thinking"] = True
    provider_kwargs["thinking_budget_tokens"] = 8192
return await provider.complete(request, **provider_kwargs)
```

---

## Session Management

### Session Pool (`server.py`)

Sessions are cached per conversation with config-based invalidation:

```python
class SessionPool:
    sessions: dict[str, AmplifierSession]

    async def get_or_create_session(self, conversation_id, websocket, provider, model):
        # Hash current config
        config_hash = hash(provider + model + settings_json)

        existing = self.sessions.get(conversation_id)
        if existing and existing._config_hash == config_hash:
            # Reuse session, update websocket reference
            self._websocket = websocket
            return existing

        # Create new session
        session = AmplifierSession(config, loader=LocalLoader(...))
        await session.initialize()
        self.sessions[conversation_id] = session
        return session
```

### Context Restoration

When switching conversations, message history is restored to the session:

```python
async def restore_context(self, session, history):
    for msg in history:
        session.coordinator.context.add_message(
            role=msg.role,
            content=msg.content,
            tool_calls=msg.tool_calls
        )
```

---

## Settings & Profiles

### Settings Hierarchy

```
1. Profile settings (from active profile YAML)
2. Global settings (~/.amplifier/settings.json)
3. Environment variables
4. Defaults (hardcoded)
```

### Profile Format

```yaml
# ~/.amplifier/profiles/dev.md
---
name: dev
version: "1.0"
description: Development profile

providers:
  - module: provider-anthropic
    config:
      model: claude-opus-4-5-20251101

tools:
  - module: tool-filesystem
  - module: tool-bash
  - module: tool-grep

session:
  max_iterations: 100
  temperature: 0.7
---

# Profile documentation in markdown body
```

---

## Build & Distribution

### PyInstaller Bundling

The Python sidecar is bundled as a native binary:

```bash
# scripts/bundle-sidecar.cjs
pyinstaller sidecar.spec --distpath src-tauri/binaries/
```

### Sidecar Naming Convention

Tauri requires platform-specific naming:
- macOS ARM: `amplifier-sidecar-aarch64-apple-darwin`
- macOS Intel: `amplifier-sidecar-x86_64-apple-darwin`
- Windows: `amplifier-sidecar-x86_64-pc-windows-msvc.exe`
- Linux: `amplifier-sidecar-x86_64-unknown-linux-gnu`

---

## Testing

### Frontend Tests (Vitest)

```bash
npm run test:frontend
# Covers: store actions, WebSocket handling, component rendering
```

### Backend Tests (pytest)

```bash
cd sidecar && uv run pytest tests/ -v
# Covers: API endpoints, session management, tool execution
```

### Integration Testing

```bash
npm run dev:all  # Start full stack
# Manual testing: send prompts, verify streaming, check tool calls
```

---

## Decisions & Trade-offs

### Streaming (Resolved 2025-12-10)

**Decision:** Keep the non-streaming fallback approach for cached providers.

**Rationale:**
- Cached providers use `messages.create()` API (non-streaming) vs local's `messages.stream()`
- Current workarounds emit synthetic delta messages after response completes
- Content appears BEFORE "done" message, maintaining responsive UX
- Thinking signatures and tool calls work correctly
- No upstream changes required to amplifier-core

**Trade-off:** Text appears all at once (not character-by-character) in Amplifier mode. This is acceptable because:
1. Response still appears before completion message
2. Consistent architecture across all cached providers
3. Future improvement possible via upstream contribution

---

## Comparison: Desktop vs CLI

Key differences between `amplifier-desktop` and `amplifier-app-cli`:

| Aspect | Desktop | CLI |
|--------|---------|-----|
| **Architecture** | Three-tier (React + Tauri + Python) | Single Python process |
| **Session** | Custom LocalOrchestrator | Direct AmplifierSession |
| **Module Loading** | LocalLoader (local/cached modes) | Module resolver (git+, collections) |
| **Providers** | Bundled + module-cache | Entry points from packages |
| **Config** | 2-scope JSON | 3-scope YAML with profiles |
| **Agent Delegation** | Not implemented | Full sub-session support |
| **Memory** | SQLite with auto-extraction | Not built-in |
| **MCP** | Full client | Not built-in |

### Desktop-Exclusive Features
- React GUI with real-time streaming
- MCP tool integration
- Memory system with auto-extraction
- Voice input/output
- Artifacts panel

### CLI-Exclusive Features
- Profile/collection system
- Agent delegation (sub-sessions)
- Plan mode (read-only)
- @mentions for context loading
- Module-based extensibility

### Convergence Opportunities
1. Add profile system to Desktop
2. Add agent delegation to Desktop
3. Add MCP to CLI
4. Add memory system to CLI

---

## Open Questions

1. **Session persistence**: Should sessions survive app restart? Currently they're recreated.

2. **Multi-window support**: How to handle multiple chat windows with separate sessions?

3. **Offline mode**: Full offline support requires local embeddings for memory search.
