# KaliCode vs Claude Code: Architectural Comparison

## Executive Summary

**KaliCode** is a **desktop GUI wrapper** around the Claude Code CLI, NOT a reimplementation. It runs Claude Code as a subprocess and provides a tabbed React UI for interaction. The restored-src folder contains the decompiled Claude Code source (v2.1.88) for reference, but KaliCode doesn't implement its own agent framework.

---

## 1. ARCHITECTURE PATTERNS

### Claude Code (Source Analysis)

**Type**: CLI-first terminal agent framework
**Entry Point**: Distributed via npm CLI package
**Runtime**: Node.js/Bun-based
**Paradigm**: Command-line tool with optional streaming output

### KaliCode (Your Implementation)

**Type**: Desktop GUI desktop wrapper
**Entry Point**: Electron app (main.js)
**Runtime**: Electron + Node.js subprocess + React UI
**Paradigm**: Desktop application spawning CLI processes

**Key Difference**:

- Claude Code = Agent engine
- KaliCode = **GUI frontend to agent engine** (not reimplementing the agent)

---

## 2. TOOL IMPLEMENTATION PATTERNS

### Claude Code Pattern

**File**: [restored-src/src/Tool.ts](restored-src/src/Tool.ts#L1-L100)

```typescript
export type ToolInputJSONSchema = {
  [x: string]: unknown
  type: 'object'
  properties?: {
    [x: string]: unknown
  }
}
```

**Tool Catalog**: [restored-src/src/tools/](restored-src/src/tools) contains 40+ specialized tools:

- `FileReadTool`, `FileWriteTool`, `FileEditTool` - File operations
- `BashTool`, `PowerShellTool` - Shell execution
- `REPLTool` - Code execution
- `MCPTool` - Model Context Protocol integration
- `AgentTool` - Nested agent invocation
- `SkillTool` - Custom skill execution
- `TaskTool` family - Task management
- `WebSearchTool`, `WebFetchTool` - Web operations

**Tool Definition Pattern**: Each tool in `/tools/{ToolName}/` has:

1. `manifest.ts` - Tool definition & metadata
2. `handler.ts` - Execution logic
3. `prompt.ts` - Claude system prompt injection
4. `utils/` - Helper functions

### KaliCode Implementation

**File**: [main.js](main.js#L70-L120)

```javascript
ipcMain.handle('run-cli', async (event, { 
  prompt, 
  model = 'openclaw',  // Local LLM via Ollama
  provider = 'ollama'   // or 'anthropic'
}) => {
  // Spawns subprocess:
  if (provider === 'ollama') {
    spawnChild('ollama', ['run', model, prompt])
  } else {
    spawnChild(process.execPath, [cliPath, '-p'], { writePromptToStdin: true })
  }
})
```

**KaliCode Tool Pattern**: **NONE** - You don't implement tools. You rely entirely on Claude Code's tools.

**Why**:

- Adding new tools would require modifying Claude Code source
- KaliCode's role is UI/UX, not tool extension
- Tools are executed by the CLI subprocess

---

## 3. AGENT ORCHESTRATION

### Claude Code - Multi-Agent Orchestration

**File**: [restored-src/src/QueryEngine.ts](restored-src/src/QueryEngine.ts#L1-L150)

Claude Code implements sophisticated multi-agent orchestration:

**Core Components**:

1. **QueryEngine** - Main loop for single agent
2. **KAIROS** (implied from code references) - Multi-turn planning
3. **ULTRAPLAN** (implied from analytics logging) - Strategic planning
4. **Agents Definition**: [restored-src/src/tools/AgentTool/loadAgentsDir.ts](restored-src/src/tools/AgentTool/loadAgentsDir.ts)

**Message Loop** (Simplified):

```text
User Query
    ↓
QueryEngine.query()
    ↓
Claude API call (stream)
    ↓
Parse tool_use blocks
    ↓
Execute tools (with permissions)
    ↓
Tool results → Message chain
    ↓
Repeat until stop_reason = "end_turn"
    ↓
Return final message
```

**Key Orchestration Features**:

- Session-based state management (`N8()` = session ID)
- Cost tracking per tool (`cost-tracker.ts`)
- Permission system enforcement (`permissions/filesystem.ts`)
- History compaction for long sessions
- Model switching mid-session

---

### KaliCode Orchestration

**File**: [renderer/App.jsx](renderer/App.jsx#L1-L150)

```jsx
const [tabs, setTabs] = useState([createTab(0)])
const [model, setModel] = useState('openclaw')
const [provider, setProvider] = useState('ollama')

// Single-tab example: spawn CLI, stream output
const sendMessage = async (prompt) => {
  const { code } = await window.electron.runCli({
    provider,
    model,
    prompt,
    env: { ANTHROPIC_API_KEY: anthropicKey }
  })
}
```

**KaliCode Orchestration**:

1. React state manages N tabs
2. Each tab = separate CLI invocation
3. No inter-agent communication
4. No session state sharing
5. Simple IPC message streaming

**Architectural Difference**:

| Aspect | Claude Code | KaliCode |
| --- | --- | --- |
| **Agent System** | Multi-agent with AgentTool | Single CLI instance per tab |
| **Session State** | Persistent session ID tracking | Independent invocations |
| **Planning** | KAIROS/ULTRAPLAN coordination | None (CLI decides) |
| **Cost Tracking** | Per-tool, per-session | Per-invocation only |
| **Continuations** | Automatic with history | Manual (copy/paste) |

## 4. MEMORY & STATE MANAGEMENT

### Claude Code - Memory System

**Files**: [restored-src/src/memdir/](restored-src/src/memdir)**Memory System**:

1. **User Memory** (`~/.claude/MEMORY.md`)
   - ~200 lines max, ~25KB max
   - Cached in system prompt
   - Persists across sessions

2. **Session Memory** (`~/.claude/sessions/{sessionId}/`)
   - Stores transcript, tool results, decisions
   - Used for session recovery

3. **Auto-Dream** (brain.ts implied)
   - Summarizes past sessions
   - Helps model understand long-term context

**Key Functions**:

```typescript
// memdir.ts
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200
export const MAX_ENTRYPOINT_BYTES = 25_000

export function truncateEntrypointContent(raw: string): EntrypointTruncation {
  // Ensures memory stays within limits
  // Line-truncates first, then byte-truncates
}

export function loadMemoryPrompt(): string {
  // Loads ~/.claude/MEMORY.md into system prompt
}
```

**State Helpers**:

```typescript
// bootstrap/state.ts (implied)
export function getSessionId(): string { return N8() }
export function getKairosActive(): boolean { }
export function getOriginalCwd(): string { }
```

### KaliCode State Management

**File**: [renderer/App.jsx](renderer/App.jsx)

```jsx
const [tabs, setTabs] = useState([createTab(0)])
const [anthropicKey, setAnthropicKey] = useState('')
const [ollamaKey, setOllamaKey] = useState('')
const [model, setModel] = useState('openclaw')

// Simple localStorage persistence
useEffect(() => {
  const storedKey = window.localStorage.getItem('KaliCodeAnthropicApiKey')
  if (storedKey) setAnthropicKey(storedKey)
})
```

**KaliCode State Architecture**:

1. **React component state** (in-memory only during session)
2. **localStorage** for API keys
3. **Tab-based isolation** (no cross-tab communication)
4. **No persistent session state** between app restarts
5. **No memory file** like Claude Code

**Architectural Difference**:

| Aspect | Claude Code | KaliCode |
| --- | --- | --- |
| **Memory System** | Structured MEMORY.md file | None |
| **Session Persistence** | Full session recovery | Lost on app close |
| **State Location** | Filesystem (~/.claude) | Browser localStorage |
| **Memory Limits** | Enforced (200 lines, 25KB) | None |
| **Cross-session Learning** | Via MEMORY.md | None |

---

## 5. PERMISSION SYSTEM

### Claude Code - Permission Model

**Files**: [restored-src/src/types/permissions.ts](restored-src/src/types/permissions.ts)

```typescript
export type PermissionMode = 'ask' | 'auto' | 'embedded'
export type PermissionResult = {
  granted: boolean
  tool: string
  reason?: string
  timestamp: Date
}
```

**Permission Tracking**:

- Deny tracking: `utils/permissions/denialTracking.ts`
- User prompts: `dialogLaunchers.tsx`
- Tool execution gates: Before tool runs, checks permission

**Tool-Specific Permissions**:

1. **FileEditTool** - Per-file confirmation
2. **BashTool** - Per-command review
3. **REPLTool** - Sandbox constraints
4. **TaskCreateTool** - Resource limits

### KaliCode Permission System

**Files**: [package/cli.js](package/cli.js) - **You DON'T implement this**

KaliCode inherits Claude Code's permission system via subprocess:

- When user runs CLI, Claude Code handles permissions
- KaliCode just displays output
- No independent permission gate

**Limitation**:

You can't intercept/modify permissions from the Electron layer. The CLI subprocess owns the permission decisions.

---

## 6. API INTEGRATION

### Claude Code - Direct API Integration

**Files**: [restored-src/src/bridge/bridgeApi.ts](restored-src/src/bridge/bridgeApi.ts#L1-L80)

**Direct Anthropic API Integration**:

```typescript
export function createBridgeApiClient(deps: BridgeApiDeps): BridgeApiClient {
  function getHeaders(accessToken: string): Record<string, string> {
    return {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
      'anthropic-version': '2023-06-01',
      'X-Trusted-Device-Token': getTrustedDeviceToken?.()
    }
  }
}
```

**API Polling**:

- Long-polling for work items
- OAuth2 token refresh
- Bearer token auth
- Trusted device token support

**Models Used**:

- Claude 3 Opus, Sonnet, Haiku
- Model deprecation tracking (in Qr8)

### KaliCode API Integration

**File**: [main.js](main.js#L70)

```javascript
// Two modes:

// 1. Local (Ollama):
spawnChild('ollama', ['run', model, prompt])

// 2. Anthropic (via Claude Code):
spawnChild(process.execPath, [
  cliPath, 
  '-p'
], { writePromptToStdin: true })
```

**Key Difference**:

- Claude Code talks to API directly
- KaliCode spawns CLI which does the API work
- KaliCode **cannot** use streaming Anthropic API directly (IPC overhead)

**Limitations of KaliCode's approach**:

1. No streaming token updates (waits for full completion)
2. No prompt caching via Anthropic API
3. No tool_use streaming (gets bulk result)
4. Added IPC latency layer

---

## 7. STRUCTURED COMPARISON TABLE

| Category | Claude Code | KaliCode | Status |
| --- | --- | --- | --- |
| **Agent Type** | Multi-agent framework | GUI wrapper | ✗ Different |
| **Tool System** | 40+ built-in tools | Uses Claude Code's tools | ✗ Different |
| **Orchestration** | QueryEngine + KAIROS | Tab-based invocation | ✗ Different |
| **Memory** | MEMORY.md + session state | localStorage only | ✗ Different |
| **Permissions** | Interactive gating | Inherited from CLI | ~ Partial |
| **API Integration** | Direct Anthropic + Bridge | Via CLI subprocess | ✗ Different |
| **Streaming** | Full streaming support | Limited (IPC-based) | ✗ Different |
| **Session Persistence** | Full recovery | None | ✗ Different |
| **Model Support** | Claude 3+ series | Ollama + Anthropic via CLI | ~ Limited |

---

## 8. KEY ARCHITECTURAL DECISIONS IN KALICODE

### What You DID Need to Implement

1. **Electron wrapper** - OS-level window management
2. **React UI** - Multi-tab chat interface
3. **IPC bridge** - Main ↔ Renderer communication
4. **CLI spawning** - Subprocess management
5. **Output streaming** - stdout/stderr capture

### What You DID NOT Need (Intentionally)

1. **Tool system** - Claude Code handles this
2. **Permission system** - Claude Code handles this
3. **Memory/state management** - Claude Code handles this
4. **API integration** - Claude Code handles this
5. **Agent orchestration** - Claude Code handles this

### Why This Design Works

- **Separation of concerns**: KaliCode = UI, Claude Code = Logic
- **Maintainability**: No need to keep agent code in sync
- **Simplicity**: Focus on user experience, not algorithmic complexity
- **Extensibility**: Support multiple providers (Ollama + Anthropic)

---

## 9. PATTERNS YOU SHADOW vs. PATTERNS YOU DIFF

### Patterns Shadowed (Same in both)

```text
✓ Tool use blocks: Parsing tool_use JSON
✓ Session concept: Each invocation = session
✓ Cost tracking: Display resource usage
```

### Patterns Different

```text
✗ Memory persistence: KaliCode uses localStorage only
✗ Tool execution: KaliCode delegates entirely to CLI
✗ Permission system: KaliCode can't customize
✗ Streaming model: KaliCode uses IPC, not direct streams
```

---

## 10. RECOMMENDATIONS

### If You Want to Add Features to **KaliCode**

**Do NOT**:

- Try to reimplement tools (they're in Claude Code)
- Create your own permission system (use CLI's)
- Build memory system (use Claude Code's ~/.claude/MEMORY.md)

**Do**:

- Enhance UI/UX (tabs, search, export, themes)
- Add Ollama model management
- Create KaliCode-specific settings (API keys, model presets)
- Build visualization for tool calls
- Add session history within Tab (read from ~/.claude/sessions/)

### If You Want to **Extend Claude Code**

**You'd need to modify**:

1. `restored-src/src/tools/` - Add new tools
2. `restored-src/src/memdir/` - Extend memory system
3. `restored-src/src/bridge/` - Add API support
4. Rebuild entire package with your changes

**Easier**: Fork Claude Code and add to it, OR use Claude Code's SDK APIs (if exposed).

---

## 11. FILE REFERENCE SUMMARY

### Claude Code Core Files (in restored-src/src/)

- [Tool.ts](restored-src/src/Tool.ts) - Tool type definitions
- [QueryEngine.ts](restored-src/src/QueryEngine.ts) - Main agent loop
- [tools/](restored-src/src/tools) - 40+ tool implementations
- [memdir/](restored-src/src/memdir) - Memory management
- [bridge/](restored-src/src/bridge) - API integration
- [services/api/claude.ts](restored-src/src/services/api/claude.ts) - Anthropic API calls
- [permissions/](restored-src/src/utils/permissions) - Permission system

### KaliCode Files

- [main.js](main.js) - Electron main process (CLI spawner)
- [renderer/App.jsx](renderer/App.jsx) - React UI + state
- [preload.js](preload.js) - IPC bridge
- [package/cli.js](package/cli.js) - Minified Claude Code CLI

---

## Conclusion

**KaliCode is structurally VERY DIFFERENT from Claude Code by design**. It's not an alternate architecture—it's a complementary architecture.

- **Claude Code** = Agent brain (decision-making, tool execution, planning)
- **KaliCode** = Agent interface (user interaction, visualization, multi-session management)

They work together: KaliCode UI → IPC → CLI spawn → Claude Code engine → Tool execution → Results back to UI.

To extend KaliCode, focus on the UI/UX layer. To extend the agent capabilities, you'd need to modify Claude Code source (in restored-src/src) and rebuild.
