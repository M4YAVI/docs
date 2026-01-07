# AG-UI Explained Simply

## What Is It?

**AG-UI is a shared language (protocol) that lets your app talk to AI agents.**

Think of it like this: Your app speaks English, an AI agent speaks French. AG-UI is the translator that lets them have a conversation.

---

## The Problem It Solves

```
Your App ←--???--→ AI Agent A
         ←--???--→ AI Agent B  
         ←--???--→ AI Agent C
```

Every AI agent works differently. Without a standard, you'd need custom code for each one.

**AG-UI gives you ONE way to talk to ALL of them.**

---

## How It Works: Events

Everything is an **event** — a small message with a type.

```
Agent sends: "Hey, I started running"         → RUN_STARTED
Agent sends: "Here's some text: Hello"        → TEXT_MESSAGE_CONTENT  
Agent sends: "I'm calling a tool now"         → TOOL_CALL_START
Agent sends: "I'm done"                       → RUN_FINISHED
```

Your app listens to this stream of events and reacts accordingly.

---

## The 16 Event Types (The Vocabulary)

| Category | Events | What They Do |
|----------|--------|--------------|
| **Lifecycle** | `RUN_STARTED`, `RUN_FINISHED`, `RUN_ERROR`, `STEP_STARTED`, `STEP_FINISHED` | Tell you when things begin/end |
| **Text** | `TEXT_MESSAGE_START`, `TEXT_MESSAGE_CONTENT`, `TEXT_MESSAGE_END` | Stream text to you piece by piece |
| **Tools** | `TOOL_CALL_START`, `TOOL_CALL_ARGS`, `TOOL_CALL_END` | Agent uses a tool (like searching) |
| **State** | `STATE_SNAPSHOT`, `STATE_DELTA`, `MESSAGES_SNAPSHOT` | Sync data efficiently |
| **Special** | `RAW`, `CUSTOM` | Anything else |

---

## The Architecture (Who Talks to Whom)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Your App  │ ←→  │ AG-UI Client│ ←→  │  AI Agent   │
│  (what user │     │ (translator)│     │  (does the  │
│    sees)    │     │             │     │   thinking) │
└─────────────┘     └─────────────┘     └─────────────┘
```

**AG-UI Client** = The piece that handles the translation. You use `HttpAgent` for HTTP connections.

---

## Two Key Superpowers

### 1. Transport Agnostic
Doesn't care HOW you send data:
- Server-Sent Events (SSE) ✓
- WebSockets ✓
- Webhooks ✓
- Whatever else ✓

### 2. Flexible Event Structure
Agents don't need to perfectly match AG-UI format. They just need to be "AG-UI-compatible." Existing systems adapt easily.

---

## State Management (Keeping Things in Sync)

Instead of sending ALL data every time:

| Event | What It Does |
|-------|--------------|
| `STATE_SNAPSHOT` | "Here's EVERYTHING right now" |
| `STATE_DELTA` | "Here's only WHAT CHANGED" (efficient) |
| `MESSAGES_SNAPSHOT` | "Here's the full conversation history" |

---

## Code Example (How You Actually Use It)

```javascript
// 1. Create a client
const agent = new HttpAgent({
  url: "https://my-agent.com/agent"
});

// 2. Run it and listen to events
agent.runAgent({ tools: [...] }).subscribe({
  next: (event) => {
    if (event.type === "TEXT_MESSAGE_CONTENT") {
      // Show text to user
    }
    if (event.type === "RUN_FINISHED") {
      // Agent is done
    }
  }
});
```

---

## The Core Contract

Every event has this shape:

```typescript
{
  type: "RUN_STARTED" | "TEXT_MESSAGE_CONTENT" | ...  // which event
  timestamp?: 1234567890                              // when (optional)
  rawEvent?: { ... }                                  // original data (optional)
}
```

---

## One-Sentence Summary

> **AG-UI is a standard set of 16 event types that let any frontend app communicate with any AI agent through a simple stream of typed messages.**
