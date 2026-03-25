---
name: agent-integrator
description: helps implement Buildship agent into frontend apps using the bs-agent npm package
license: MIT
metadata:
  author: buildship
  repository: "https://github.com/buildship/agent-skills"
  version: "1.0.0"
  keywords: "ai, agent, skill, buildship, sdk, streaming, react, chat"
---

# BuildShip Agent Integration

A type-safe TypeScript SDK for integrating BuildShip agents into any JavaScript/TypeScript project with streaming support.

**Key features:**

- Streaming-first — real-time text, reasoning, tool calls & handoffs via SSE
- Client tools — headless handlers, interactive widgets, pause/resume
- Multimodal input — send text, images & files in a single message
- React bindings — hooks & context for chat UIs with session management
- Multi-turn — session-based conversations with persistent history
- Abort — cancel any streaming request mid-flight
- Inline debug info — tool calls, reasoning, handoffs & errors as message parts
- Zero extra deps — native fetch + ReadableStream, only zod as a dependency

## Install

```bash
npm install bs-agent
```

The package exposes two entry points:

```typescript
import { ... } from "bs-agent/core";   // Vanilla JS/TS — works anywhere
import { ... } from "bs-agent/react";   // React hooks, context & components
```

## Quick Start — Vanilla JS/TS

```typescript
import { BuildShipAgent, z } from "bs-agent/core";

const agent = new BuildShipAgent({
  agentId: "YOUR_AGENT_ID",
  accessKey: "YOUR_ACCESS_KEY", // optional
  baseUrl: "https://your-project.buildship.run",
});

// Simple one-shot
const session = await agent.execute("Hello!", {
  onText: (text) => process.stdout.write(text),
  onComplete: (fullText) => console.log("\nDone:", fullText),
  onError: (err) => console.error(err),
});
```

## Quick Start — React

```tsx
import { AgentContextProvider, useAgent } from "bs-agent/react";

function App() {
  return (
    <AgentContextProvider>
      <ChatPage />
    </AgentContextProvider>
  );
}

function ChatPage() {
  const {
    messages,
    inProgress,
    handleSend,
    abort,
    sessionId,
    sessions,
    switchSession,
  } = useAgent(
    "agent-id",
    "https://your-project.buildship.run/executeAgent/AGENT_ID",
    "access-key",
  );

  return (
    <div>
      {messages.map((msg, i) => (
        <div key={i} className={msg.role}>
          {msg.content}
        </div>
      ))}
      <button onClick={() => handleSend("Hello!")} disabled={inProgress}>
        Send
      </button>
    </div>
  );
}
```

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

The `execute()` method accepts either a plain string or an array of content parts for sending text alongside images and files:

```typescript
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
    { type: "file", data: "https://storage.example.com/report.csv", mimeType: "text/csv", filename: "report.csv" },
  ],
  callbacks,
);
```

Data field accepts: HTTP URLs (`https://...`), Data URLs (`data:image/png;base64,...`), or raw base64 strings.

## Client Tools

Register tools the agent can invoke on the client side. See [Client Tools Reference](references/client-tools.md) for full details.

```typescript
import { BuildShipAgent, z } from "bs-agent/core";

const agent = new BuildShipAgent({ agentId: "..." });

// Fire-and-forget tool
agent.registerClientTool({
  name: "show_notification",
  description: "Display a notification to the user",
  parameters: z.object({
    title: z.string().describe("Notification title"),
    message: z.string().describe("Notification body"),
  }),
  handler: (args) => {
    showNotification(args.title, args.message);
  },
});

// Blocking tool — agent pauses until result is returned
agent.registerClientTool({
  name: "get_location",
  description: "Get the user's current location",
  parameters: z.object({}),
  await: true,
  handler: async () => {
    const pos = await navigator.geolocation.getCurrentPosition();
    return { lat: pos.coords.latitude, lng: pos.coords.longitude };
  },
});
```

## Reference Documentation

- [Core API Reference](references/core-api.md) — `BuildShipAgent`, `AgentSession`, stream callbacks, events, multimodal input types
- [React API Reference](references/react-api.md) — `useAgent`, `useAgentContext`, `AgentContextProvider`, sessions, messages & parts, text modifiers
- [Client Tools Reference](references/client-tools.md) — headless tools, widget tools, combo tools, pause/resume, `useClientTool`, `ToolRenderer`

## License

MIT
