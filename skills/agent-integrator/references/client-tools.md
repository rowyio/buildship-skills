---
name: "Client Tools Reference"
description: "Client-side tool registration for BuildShip agents — headless tools, widget tools, combo tools, pause/resume, and React useClientTool hook."
tags: [client-tools, registerClientTool, useClientTool, ToolRenderer, widget, headless, combo, pause, resume, zod, ClientToolConfig, ClientToolRenderProps]
---

# Client Tools Reference

Client tools let the agent invoke functionality on the client side. They support headless handlers, interactive widgets, and a combo mode. Available in both core and React APIs.

## Core API — registerClientTool

```typescript
import { BuildShipAgent, z } from "bs-agent/core";

const agent = new BuildShipAgent({ agentId: "..." });
```

### Fire-and-Forget Tool

Handler runs, result is discarded. The agent does not wait.

```typescript
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
```

### Blocking Tool (with handler)

Agent pauses until the handler's return value is sent back.

```typescript
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

### Pause & Resume (Manual)

For blocking tools without a handler, the agent pauses and you resume manually:

```typescript
agent.registerClientTool({
  name: "confirm_action",
  description: "Ask the user to confirm an action",
  parameters: z.object({
    action: z.string().describe("The action to confirm"),
  }),
  await: true,
  // No handler — you handle it manually
});

const session = await agent.execute("Delete my account", {
  onText: (t) => process.stdout.write(t),
  onPaused: (toolName, args) => {
    console.log(`Agent paused for: ${toolName}`, args);
  },
});

if (session.isPaused()) {
  const tool = session.getPausedTool();
  // ... show confirmation UI, then resume:
  await session.resume(
    { confirmed: true },
    { onText: (t) => process.stdout.write(t) },
  );
}
```

## React API — useClientTool

### Headless Tools

Register a tool that runs code without rendering any UI:

```tsx
import { useClientTool } from "bs-agent/react";
import { z } from "bs-agent/core";

function ChatPage() {
  // Fire-and-forget
  useClientTool("agent-id", {
    name: "show_notification",
    description: "Display a notification to the user",
    parameters: z.object({
      title: z.string(),
      message: z.string(),
    }),
    handler: (inputs) => {
      toast(inputs.title, inputs.message);
    },
  });

  // Blocking tool — handler result is sent back
  useClientTool("agent-id", {
    name: "get_location",
    description: "Get the user's current GPS location",
    parameters: z.object({}),
    await: true,
    handler: async () => {
      const pos = await getCurrentPosition();
      return { lat: pos.coords.latitude, lng: pos.coords.longitude };
    },
  });
}
```

### Widget Tools

Register a tool that renders interactive UI inline in the conversation:

```tsx
import { useClientTool, ToolRenderer } from "bs-agent/react";
import { z } from "bs-agent/core";

function ChatPage() {
  const { messages } = useAgent("agent-id", agentUrl);

  useClientTool("agent-id", {
    name: "feedback_form",
    description: "Collects user feedback",
    parameters: z.object({
      question: z.string().describe("The feedback question"),
    }),
    await: true, // Agent pauses until user submits
    render: ({ inputs, submit, status, result }) => (
      <div>
        <p>{inputs.question}</p>
        {status === "pending" ? (
          <button onClick={() => submit({ answer: "Great!" })}>Submit</button>
        ) : (
          <p>Submitted: {JSON.stringify(result)}</p>
        )}
      </div>
    ),
  });

  // Render messages with embedded widgets
  return (
    <div>
      {messages.map((msg) =>
        msg.parts?.map((part) => {
          if (part.type === "text") {
            return <p key={part.firstSequence}>{part.text}</p>;
          }
          if (part.type === "widget") {
            return <ToolRenderer key={part.callId} agentId="agent-id" part={part} />;
          }
          return null;
        }),
      )}
    </div>
  );
}
```

### Combo Tools (Handler + Render)

When both `handler` and `render` are provided, the tool acts as a widget that auto-executes:

1. The widget renders immediately with `status: "pending"`
2. The handler runs automatically in the background
3. When the handler resolves, the widget updates to `status: "submitted"` with the result
4. If the handler throws, the widget updates to `status: "error"` with the error message
5. The agent auto-resumes with the handler's return value

Ideal for async operations where you want to show progress and results inline (e.g., cloning a project, running a migration, processing data).

```tsx
useClientTool("agent-id", {
  name: "run_migration",
  description: "Runs a database migration",
  parameters: z.object({
    migrationName: z.string(),
  }),
  await: true,
  handler: async (inputs) => {
    const result = await runMigration(inputs.migrationName);
    return { rowsAffected: result.count, duration: result.ms };
  },
  render: ({ inputs, status, result, error }) => (
    <div>
      <strong>{inputs.migrationName}</strong>
      {status === "pending" && <Spinner />}
      {status === "submitted" && (
        <p>Done — {result.rowsAffected} rows in {result.duration}ms</p>
      )}
      {status === "error" && <p style={{ color: "red" }}>{error}</p>}
    </div>
  ),
});
```

> **Note:** With combo tools, the `submit` callback in render props is a no-op since the handler provides the result automatically. The widget is purely for display.

## Type Definitions

### ClientToolConfig

```typescript
interface ClientToolConfig {
  name: string;                          // Must match the tool name the agent knows
  description: string;                   // Description of what the tool does
  parameters: ZodSchema | Record<string, any>; // Zod schema or raw JSON Schema
  await?: boolean;                       // If true, agent pauses until result
  handler?: (inputs: any) => any | Promise<any>; // For headless tools or combo tools
  render?: (props: ClientToolRenderProps) => any; // For widget tools or combo tools
}
```

### ClientToolRenderProps

```typescript
interface ClientToolRenderProps<T = any> {
  inputs: T;                             // Parsed inputs from the agent
  submit: (result: any) => void;         // Submit a result (only for await: true tools)
  status: "pending" | "submitted" | "error"; // Widget status
  result?: any;                          // Persisted result after submission
  error?: string;                        // Error message (only when status is "error")
}
```

## Best Practices

- Use Zod schemas for type-safe parameter definitions — the SDK re-exports `z` from `bs-agent/core`.
- Use fire-and-forget tools (no `await`) for side effects like notifications or analytics that don't need to return data to the agent.
- Use blocking tools (`await: true`) when the agent needs the result to continue its response.
- Use manual pause/resume for user-facing confirmations where you need to show custom UI before proceeding.
- Use combo tools (handler + render) for async operations where you want inline progress display.
- Always unregister tools with `unregisterClientTool(name)` when they are no longer needed to prevent stale tool registrations.
