---
name: "React API Reference"
description: "React hooks, context providers, components, sessions, messages & parts, and text modifiers for building chat UIs with BuildShip agents."
tags: [react, useAgent, useAgentContext, AgentContextProvider, ToolRenderer, sessions, messages, MessagePart, textDeltaModifier, fullTextModifier, handleSend]
---

# React API Reference (`bs-agent/react`)

The React module provides hooks, context providers, and components for building chat UIs with full session management, client tool support, and debug panels.

```typescript
import { AgentContextProvider, useAgent, useAgentContext, useClientTool, ToolRenderer } from "bs-agent/react";
import { z } from "bs-agent/core";
```

## Setup

Wrap your app (or the chat area) with `AgentContextProvider`:

```tsx
import { AgentContextProvider } from "bs-agent/react";

function App() {
  return (
    <AgentContextProvider>
      <ChatPage />
    </AgentContextProvider>
  );
}
```

## useAgent Hook

The main hook for interacting with an agent. Manages messages, streaming, and sessions.

```tsx
import { useAgent } from "bs-agent/react";

function ChatPage() {
  const {
    messages,         // Message[] — full conversation history
    inProgress,       // boolean — true while streaming
    handleSend,       // (input, options?) => Promise — send a message
    resumeTool,       // (callId, result) => Promise — resume a paused tool
    abort,            // () => void — cancel the current stream
    sessionId,        // string — current session ID
    sessions,         // Session[] — all sessions for this agent
    switchSession,    // (sessionId?) => void — switch to a session (or create new)
    deleteSession,    // (sessionId) => void — delete a session
    addOptimisticMessage, // (input) => void — add a user message immediately
  } = useAgent(
    "agent-id",
    "https://your-project.buildship.run/executeAgent/AGENT_ID",
    "access-key",
  );

  return (
    <div>
      {messages.map((msg, i) => (
        <div key={i} className={msg.role}>{msg.content}</div>
      ))}
      <button onClick={() => handleSend("Hello!")} disabled={inProgress}>Send</button>
    </div>
  );
}
```

### handleSend Options

```typescript
handleSend(input: AgentInput, options?: {
  context?: Record<string, unknown>;  // Additional context passed to the agent
  skipUserMessage?: boolean;          // Don't add a user message to the conversation
});
```

`AgentInput` is `string | ContentPart[]` — see [Core API Reference](core-api.md) for multimodal input types.

## useAgentContext Hook

An alternative to `useAgent` for multi-agent setups. Initializes agents declaratively and shares state through context.

```tsx
import { useAgentContext } from "bs-agent/react";

function ChatPage() {
  const agent = useAgentContext(
    "agent-id",
    "https://your-project.buildship.run/executeAgent/AGENT_ID",
    "access-key",
  );

  // Same return shape as useAgent
  const { messages, handleSend, inProgress, sessions, ... } = agent;
}
```

## Text Modifiers

`useAgent` accepts optional modifier functions that transform streamed text before it is stored in messages.

### textDeltaModifier

Runs on each incoming text chunk before it is appended. Receives the raw delta, the full text accumulated so far, and event metadata.

```typescript
const agent = useAgent(myAgent, {
  textDeltaModifier: (delta, fullText, meta) => {
    // Example: strip <think>...</think> tags from each chunk
    return delta.replace(/<\/?think>/g, "");
  },
});
```

### fullTextModifier

Runs on the accumulated full text after every delta is appended. The return value becomes the displayed text, while the unmodified accumulation is preserved internally as `_rawText`.

```typescript
const agent = useAgent(myAgent, {
  fullTextModifier: (fullText, meta) => {
    // Example: render LaTeX-style math as Unicode
    return convertLatex(fullText);
  },
});
```

### Chaining Order

When both modifiers are provided they chain in order:

```
raw delta -> textDeltaModifier(delta) -> accumulated text -> fullTextModifier(accumulated) -> display
```

`textDeltaModifier` acts as a per-chunk preprocessor; `fullTextModifier` acts as a post-accumulation formatter on top of the already-modified text.

> **Note:** If `textDeltaModifier` strips or transforms content, `fullTextModifier` will only see the already-modified accumulation — the original unmodified stream text is not preserved.

## Messages & Parts

Messages can contain rich, interleaved content via `parts`:

```typescript
type Message = {
  role: "user" | "agent";
  content: string;                              // Full text content
  parts?: MessagePart[];                        // Rich content (text, widgets, tool calls, reasoning, etc.)
  executionId?: string;                         // Execution ID for this turn
  attachments?: Array<ImagePart | FilePart>;    // Multimodal user message attachments
};

type MessagePart =
  | { type: "text"; text: string; firstSequence: number; lastSequence: number }
  | {
      type: "widget";
      toolName: string;
      callId: string;
      inputs: any;
      paused?: boolean;
      status?: "pending" | "submitted" | "error";
      result?: any;
      error?: string;
    }
  | {
      type: "tool_call";
      toolName: string;
      callId: string;
      toolType: ToolType;
      status: "progress" | "complete" | "error";
      inputs?: unknown;
      output?: unknown;
      error?: string;
      serverName?: string;   // MCP server name
    }
  | { type: "reasoning"; reasoning: string; index?: number }
  | { type: "handoff"; agentName: string }
  | { type: "run_error"; message: string; code?: string };
```

> **Tip:** When rendering messages, iterate over `msg.parts` instead of `msg.content` to get text, widgets, tool calls, reasoning, handoffs, and errors interleaved in chronological order.

## Sessions

Sessions are automatically persisted to local storage and synced across tabs.

```tsx
const { sessions, switchSession, deleteSession, sessionId } = useAgent(...);

// List all sessions
sessions.map((s) => (
  <button key={s.id} onClick={() => switchSession(s.id)}>
    {s.name} ({s.messages.length} messages)
  </button>
));

// Create a new session
switchSession(); // No argument = new session

// Delete a session
deleteSession(sessionId);
```

### Session Type

```typescript
type Session = {
  id: string;
  createdAt: number;
  updatedAt: number;
  messages: Message[];
  name?: string;
};
```

## Inline Debug Info

Tool calls, reasoning, agent handoffs, and errors are all embedded directly in the agent message's `parts` array — no separate debug state. Filter by type to render them:

```typescript
const { messages } = useAgent(...);

const debugParts = message.parts?.filter(
  (p) =>
    p.type === "tool_call" ||
    p.type === "reasoning" ||
    p.type === "handoff" ||
    p.type === "run_error",
);
```

## Hooks Reference

| Hook | Description |
|---|---|
| `useAgent(agentId, agentUrl, accessKey?)` | Main hook — messages, streaming, sessions |
| `useAgentContext(agentId, agentUrl, key?)` | Context-based alternative for multi-agent setups |
| `useClientTool(agentId, config)` | Register a client tool (headless or widget) |

## Components Reference

| Component | Description |
|---|---|
| `<AgentContextProvider>` | Provides shared agent state (sessions) |
| `<ToolRenderer agentId={id} part={part} />` | Renders a widget tool from a message part |

## Utilities Reference

| Export | Description |
|---|---|
| `tryParseJSON(value)` | Safely parse a JSON string, returns parsed object or original string |
| `updateAgentMessageParts(msg, event)` | Append/merge parts into an agent message |

## Best Practices

- Always wrap your chat UI with `<AgentContextProvider>` to enable session management and multi-agent support.
- Use `msg.parts` for rendering instead of `msg.content` to get the full rich message experience including widgets, tool calls, and reasoning.
- Use `addOptimisticMessage` for immediate UI feedback before the server responds.
- Use `useAgentContext` instead of `useAgent` when building multi-agent UIs that need shared session state.
- Apply `textDeltaModifier` for per-chunk transformations (e.g., stripping tags) and `fullTextModifier` for formatting the accumulated output.
