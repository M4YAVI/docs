# AG-UI Messages: The Complete Guide

## What It Is (The 30-Second Version)

Messages are how humans and AI agents talk to each other. Think of it like a text conversation on your phone—each text is a "message" with a sender, content, and a unique ID.

AG-UI standardizes this conversation format so **any AI** (OpenAI, Anthropic, Google) can plug in without changing your app.

---

## The Core Structure

Every message has three things:

```
WHO sent it → role
WHAT they said → content  
WHICH message → id
```

```typescript
{
  id: "msg_001",      // unique identifier
  role: "user",       // who's talking
  content: "Hello"    // what they said
}
```

That's it. Everything else builds on this.

---

## The Six Roles (Who Can Talk)

| Role | Who | Purpose |
|------|-----|---------|
| `user` | You, the human | Ask questions, give input |
| `assistant` | The AI | Respond, use tools |
| `system` | The developer | Give AI instructions it must follow |
| `tool` | External services | Return results from actions |
| `developer` | Debug/internal | Development messages |
| `activity` | UI only | Show progress, never sent to AI |

---

## Each Message Type Explained

### 1. User Message
**What you say to the AI.**

```typescript
{
  id: "msg_1",
  role: "user",
  content: "What's 2+2?"
}
```

Can also send files/images:
```typescript
{
  id: "msg_2",
  role: "user",
  content: [
    { type: "text", text: "What's in this image?" },
    { type: "binary", mimeType: "image/png", url: "https://..." }
  ]
}
```

---

### 2. Assistant Message
**What the AI says back.**

Simple response:
```typescript
{
  id: "msg_3",
  role: "assistant",
  content: "The answer is 4."
}
```

With tool usage:
```typescript
{
  id: "msg_4",
  role: "assistant",
  content: "Let me calculate that...",
  toolCalls: [{
    id: "call_1",
    type: "function",
    function: {
      name: "calculator",
      arguments: '{"expression": "2+2"}'
    }
  }]
}
```

---

### 3. System Message
**Hidden instructions for the AI.**

```typescript
{
  id: "msg_0",
  role: "system",
  content: "You are a helpful math tutor. Always show your work."
}
```

The user never sees this. It shapes AI behavior.

---

### 4. Tool Message
**Results from external actions.**

```typescript
{
  id: "tool_result_1",
  role: "tool",
  content: "4",
  toolCallId: "call_1"  // links back to which call this answers
}
```

---

### 5. Activity Message
**Frontend-only UI elements.**

```typescript
{
  id: "activity_1",
  role: "activity",
  activityType: "SEARCH",
  content: { 
    status: "searching",
    query: "weather NYC" 
  }
}
```

**Key insight:** This NEVER goes to the AI. It's purely for showing the user "something is happening" (loading spinners, progress bars, checklists).

---

### 6. Developer Message
**Debugging/internal use.**

```typescript
{
  id: "dev_1",
  role: "developer",
  content: "Debug: context window at 80% capacity"
}
```

---

## How Messages Flow

### Option 1: Snapshot (All at Once)
Server sends entire conversation history:

```typescript
{
  type: "MESSAGES_SNAPSHOT",
  messages: [msg1, msg2, msg3, ...]  // everything
}
```

**Use when:** Starting up, reconnecting, major sync needed.

---

### Option 2: Streaming (Piece by Piece)
AI responses come word-by-word:

```
Step 1: TEXT_MESSAGE_START    → "New message coming"
Step 2: TEXT_MESSAGE_CONTENT  → "The" 
Step 3: TEXT_MESSAGE_CONTENT  → " answer"
Step 4: TEXT_MESSAGE_CONTENT  → " is 4."
Step 5: TEXT_MESSAGE_END      → "Done"
```

**Why streaming?** User sees text appear in real-time instead of waiting 10 seconds for complete response.

---

## Tool Calls: How AI Takes Actions

The conversation flow:

```
1. USER:      "What's the weather in NYC?"
                    ↓
2. ASSISTANT: "Let me check..." + toolCall to get_weather
                    ↓
3. TOOL:      Returns {"temp": 72, "condition": "sunny"}
                    ↓
4. ASSISTANT: "It's 72°F and sunny in NYC."
```

### Tool Call Structure:
```typescript
{
  id: "call_abc",
  type: "function",
  function: {
    name: "get_weather",
    arguments: '{"city": "NYC"}'
  }
}
```

### Tool Result Links Back:
```typescript
{
  role: "tool",
  toolCallId: "call_abc",  // ← THIS connects result to call
  content: '{"temp": 72}'
}
```

---

## Streaming Tool Calls

Same pattern as text—comes in pieces:

```
TOOL_CALL_START  → "Starting calculator call"
TOOL_CALL_ARGS   → '{"expr'
TOOL_CALL_ARGS   → 'ession":'
TOOL_CALL_ARGS   → '"2+2"}'
TOOL_CALL_END    → "Call complete"
```

User sees the tool being invoked in real-time.

---

## Why Vendor Neutral Matters

AG-UI message format:
```typescript
{ role: "user", content: "Hello" }
```

Works with **any** AI by converting:

| Provider | Their Format | AG-UI Handles It |
|----------|-------------|------------------|
| OpenAI | `{role, content}` | ✓ |
| Anthropic | `{role, content: [{type, text}]}` | ✓ |
| Google | `{parts: [{text}]}` | ✓ |

**You write once. AG-UI translates.**

---

## Complete Conversation Example

```typescript
[
  // System sets the rules
  { id: "1", role: "system", content: "You are a weather assistant." },
  
  // User asks
  { id: "2", role: "user", content: "Weather in NYC?" },
  
  // AI decides to use a tool
  { 
    id: "3", 
    role: "assistant", 
    content: "Checking...",
    toolCalls: [{
      id: "call_1",
      type: "function",
      function: { name: "get_weather", arguments: '{"city":"NYC"}' }
    }]
  },
  
  // Tool returns data
  { id: "4", role: "tool", toolCallId: "call_1", content: '{"temp":72}' },
  
  // AI uses that data to respond
  { id: "5", role: "assistant", content: "It's 72°F in NYC." }
]
```

---

## Summary

| Concept | What It Does |
|---------|--------------|
| **Message** | Single unit of conversation |
| **Role** | Who's speaking |
| **Snapshot** | Send all messages at once |
| **Streaming** | Send messages piece-by-piece |
| **Tool Call** | AI asks to run external function |
| **Tool Message** | Result comes back |
| **Activity** | UI-only, AI never sees it |
| **Vendor Neutral** | Same format works with any AI |

**The bottom line:** Messages are a standardized envelope for all human-AI communication. Doesn't matter who built the AI—they all speak this format.
