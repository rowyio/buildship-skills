---
name: "Core API Reference"
description: "BuildShipAgent class, AgentSession, stream callbacks, events, multimodal input, multi-turn, and abort for vanilla JS/TS usage."
tags: [core, BuildShipAgent, AgentSession, execute, session, streaming, SSE, callbacks, events, abort, multimodal, ContentPart, ToolType]
---

# Core API Reference (`bs-agent/core`)

The core module provides a class-based API that works in any JavaScript environment — Node.js, browser, Edge Runtime, etc.

```typescript
import { BuildShipAgent, z } from "bs-agent/core";
```

## BuildShipAgent

### Constructor

```typescript
const agent = new BuildShipAgent({
  agentId: "YOUR_AGENT_ID",
  accessKey: "YOUR_ACCESS_KEY", // optional
  baseUrl: "https://your-project.buildship.run",
});
```

### Methods

| Method | Description |
|---|---|
| `execute(input, callbacks, options?)` | Start a new conversation. `input` is `string \| ContentPart[]`. Returns `AgentSession`. |
| `session(sessionId)` | Continue an existing conversation. Returns `AgentSession`. |
| `registerClientTool(tool)` | Register a client-side tool. |
| `unregisterClientTool(name)` | Remove a registered tool. |

### Basic Usage

```typescript
const session = await agent.execute("Hello!", {
  onText: (text) => process.stdout.write(text),
  onComplete: (fullText) => console.log("\nDone:", fullText),
  onError: (err) => console.error(err),
});
```

## AgentSession

| Method | Description |
|---|---|
| `execute(input, callbacks, options?)` | Send a message (`string \| ContentPart[]`). |
| `resume(result, callbacks)` | Resume after a blocking tool pause. |
| `isPaused()` | Check if waiting for a tool result. |
| `getPausedTool()` | Get paused tool info (`{ callId, toolName, args }`). |
| `getSessionId()` | Get the session ID. |
| `abort()` | Cancel the current stream. |

## Multi-Turn Conversations

```typescript
// First message returns a session
const session = await agent.execute("What is 2 + 2?", {
  onText: (t) => process.stdout.write(t),
  onComplete: () => console.log(),
});

// Continue with the same session ID
const continued = agent.session(session.getSessionId());
await continued.execute("Now multiply that by 3", {
  onText: (t) => process.stdout.write(t),
  onComplete: () => console.log(),
});
```

## Multimodal Input

The `execute()` method accepts either a plain string or an array of content parts.

```typescript
type AgentInput = string | ContentPart[];

type ContentPart = TextPart | ImagePart | FilePart;

type TextPart = { type: "text"; text: string };

// mimeType is optional — defaults to "image/png" if omitted.
type ImagePart = { type: "image"; data: string; mimeType?: string };

// mimeType is required — files need an explicit MIME type.
type FilePart = {
  type: "file";
  data: string;
  mimeType: string;
  filename?: string;
};
```

### Data Field Formats

| Format | Example | Prefix |
|---|---|---|
| HTTP URL | `"https://example.com/photo.jpg"` | `http(s)://` |
| Data URL | `"data:image/png;base64,iVBOR..."` | `data:` |
| Raw base64 | `"iVBORw0KGgo..."` | (none) |

### Examples

```typescript
// Text-only (backward compatible)
await agent.execute("What is this?", callbacks);

// Image + text
await agent.execute(
  [
    { type: "text", text: "What's in this image?" },
    { type: "image", data: "https://example.com/photo.jpg", mimeType: "image/jpeg" },
  ],
  callbacks,
);

// File attachment
await agent.execute(
  [
    { type: "text", text: "Summarize this CSV" },
    {
      type: "file",
      data: "https://storage.example.com/report.csv",
      mimeType: "text/csv",
      filename: "report.csv",
    },
  ],
  callbacks,
);
```

## Stream Callbacks

```typescript
interface StreamCallbacks {
  /** Called for each text chunk from the agent. */
  onText?: (text: string) => void;
  /** Called for each reasoning chunk (models with chain-of-thought). */
  onReasoning?: (delta: string, index: number) => void;
  /** Called when control is handed off to a sub-agent. */
  onAgentHandoff?: (agentName: string) => void;
  /** Called when a tool execution starts. */
  onToolStart?: (toolName: string, toolType: ToolType) => void;
  /** Called when a tool execution completes. */
  onToolEnd?: (toolName: string, result?: any, error?: string) => void;
  /** Called when agent pauses for a blocking client tool. */
  onPaused?: (toolName: string, args: any) => void;
  /** Called when the stream completes successfully. */
  onComplete?: (fullText: string) => void;
  /** Called if an error occurs during streaming. */
  onError?: (error: Error) => void;
  /** Called for every raw SSE event. Useful for debug panels. */
  onEvent?: (event: StreamEvent) => void;
}
```

## Stream Events

All events share a `meta` object with `executionId` and `sequence`.

| Event Type | Description | Data |
|---|---|---|
| `text_delta` | Text chunk from the agent | `string` |
| `reasoning_delta` | Chain-of-thought reasoning chunk | `{ delta: string, index: number }` |
| `tool_call_start` | A tool execution started | `{ callId, toolName, toolType, inputs?, serverName?, paused? }` |
| `tool_call_end` | A tool execution completed | `{ callId, toolName, toolType, result?, error?, executionTime? }` |
| `agent_handoff` | Control transferred to a sub-agent | `{ agentName: string }` |

## Tool Types

```typescript
type ToolType = "flow" | "node" | "mcp" | "client" | "builtin" | "agent";
```

## Abort

```typescript
const session = agent.session(sessionId);

session.execute("Write a long essay...", {
  onText: (text) => {
    process.stdout.write(text);
    if (userCancelled) session.abort();
  },
});
```

## Best Practices

- Always provide an `onError` callback to handle stream failures gracefully.
- Use `session()` to continue conversations rather than creating new ones — this preserves context and history.
- For multimodal input, prefer HTTP URLs over base64 when possible to reduce payload size.
- Use `abort()` to cancel long-running streams and free resources.
- Register client tools before calling `execute()` so the agent knows they are available.
