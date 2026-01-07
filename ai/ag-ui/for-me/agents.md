# AG-UI Agents: The Complete Guide

## What Is an Agent?

**An agent is a class that talks to an AI and streams back events.**

```
Your App → Agent → AI Service (GPT-4, Claude, etc.)
                ← Stream of Events
```

The agent is the middleman. It takes your request, sends it to the AI, and gives you back a stream of events.

---

## The Core Contract

Every agent must do ONE thing: implement the `run` method.

```typescript
class MyAgent extends AbstractAgent {
  run(input: RunAgentInput): RunAgent {
    // Send events back
  }
}
```

**Input:** What to do (messages, tools, context)
**Output:** A stream of events (RUN_STARTED, TEXT_MESSAGE_CONTENT, RUN_FINISHED, etc.)

---

## The 5 Core Components of an Agent

```
┌─────────────────────────────────────────────────────────┐
│                        AGENT                            │
├─────────────────────────────────────────────────────────┤
│  1. Configuration  │ agentId, threadId, description     │
│  2. Messages       │ Conversation history                │
│  3. State          │ Persistent data across interactions │
│  4. Events         │ What it sends back to your app      │
│  5. Tools          │ Functions the agent can call        │
└─────────────────────────────────────────────────────────┘
```

---

## Agent Types

### 1. AbstractAgent (Base Class)
The blueprint. All agents extend this.

```typescript
import { AbstractAgent } from "@ag-ui/client"

class MyAgent extends AbstractAgent {
  run(input) { /* your logic */ }
}
```

### 2. HttpAgent (Ready to Use)
Connects to a remote AI endpoint via HTTP. No custom code needed.

```typescript
const agent = new HttpAgent({
  url: "https://your-ai-service.com/agent",
  headers: { Authorization: "Bearer your-key" }
})
```

### 3. Custom Agents
Build your own for special needs.

---

## Building an Agent: The Minimal Example

```typescript
class SimpleAgent extends AbstractAgent {
  run(input: RunAgentInput): RunAgent {
    const { threadId, runId } = input
    
    return () => new Observable((observer) => {
      // 1. Say "I'm starting"
      observer.next({ type: EventType.RUN_STARTED, threadId, runId })
      
      // 2. Say "Here's my message"
      const messageId = "msg_1"
      observer.next({ type: EventType.TEXT_MESSAGE_START, messageId, role: "assistant" })
      observer.next({ type: EventType.TEXT_MESSAGE_CONTENT, messageId, delta: "Hello!" })
      observer.next({ type: EventType.TEXT_MESSAGE_END, messageId })
      
      // 3. Say "I'm done"
      observer.next({ type: EventType.RUN_FINISHED, threadId, runId })
      
      // 4. Close the stream
      observer.complete()
    })
  }
}
```

**That's it.** Emit events in order. Start → Content → Finish.

---

## Agent Capabilities

### 1. Streaming Responses

The agent doesn't wait until the AI is done. It streams character-by-character:

```
User sees: "H" → "He" → "Hel" → "Hell" → "Hello" → "Hello!"
```

Not:
```
User waits 3 seconds... then sees "Hello!"
```

---

### 2. Tools (Functions the Agent Can Call)

**Key insight:** Tools are defined by YOUR APP and passed TO the agent.

```typescript
// Your app defines the tool
const searchTool = {
  name: "search",
  description: "Search the web",
  parameters: {
    type: "object",
    properties: {
      query: { type: "string" }
    }
  }
}

// You pass it to the agent
agent.runAgent({
  tools: [searchTool]  // Agent now knows it can search
})
```

**The Flow:**

```
Agent: "I need to search" → TOOL_CALL_START
Agent: '{"query": "weather"}' → TOOL_CALL_ARGS
Agent: "Done specifying" → TOOL_CALL_END

Your App: *executes search* → Returns result

Agent: "Based on the search..." → TEXT_MESSAGE_CONTENT
```

**Why this matters:** Human-in-the-loop. The agent asks, your app (or user) decides, result goes back to agent.

---

### 3. State Management

Agents remember things across conversations:

```typescript
// State persists
agent.state = { 
  userName: "Alice",
  preferredLanguage: "English",
  conversationTopic: "weather"
}

// Access it anytime
console.log(agent.state.userName)  // "Alice"

// Updated via events during a run
agent.runAgent().subscribe((event) => {
  if (event.type === EventType.STATE_DELTA) {
    // State just changed
    console.log(agent.state)
  }
})
```

---

### 4. Message History

The agent keeps track of the entire conversation:

```typescript
agent.messages = [
  { id: "1", role: "user", content: "What's the weather?" },
  { id: "2", role: "assistant", content: "It's sunny!" },
  { id: "3", role: "user", content: "Thanks!" }
]

// Add new messages
agent.messages.push({
  id: "4",
  role: "user", 
  content: "What about tomorrow?"
})
```

The AI uses this history for context.

---

### 5. Multi-Agent Handoff

Agents can pass work to other agents:

```
General Agent: "This is a coding question..."
    ↓ handoff
Coding Agent: "Here's the Python code..."
    ↓ handoff  
General Agent: "The coding agent wrote this for you..."
```

Context and state transfer with the handoff.

---

### 6. Human-in-the-Loop

The agent can pause and ask for human input:

```
Agent: "Should I delete this file?" → TOOL_CALL (confirmAction)
Human: *clicks "Yes"*
Agent: "Okay, deleting..." → continues
```

AI proposes, human approves, AI executes.

---

## Using an Agent (Full Example)

```typescript
// 1. Create the agent
const agent = new HttpAgent({
  url: "https://your-ai-service.com/agent"
})

// 2. Set initial messages (optional)
agent.messages = [
  { id: "1", role: "user", content: "Hello!" }
]

// 3. Run it
agent.runAgent({
  runId: "run_123",
  tools: [searchTool],
  context: [{ name: "user_info", value: "Premium subscriber" }]
}).subscribe({
  next: (event) => {
    switch (event.type) {
      case EventType.RUN_STARTED:
        console.log("Agent started")
        break
      case EventType.TEXT_MESSAGE_CONTENT:
        console.log("Text:", event.delta)
        break
      case EventType.TOOL_CALL_START:
        console.log("Tool called:", event.toolCallName)
        break
      case EventType.RUN_FINISHED:
        console.log("Agent done")
        break
    }
  },
  error: (err) => console.error(err),
  complete: () => console.log("Stream closed")
})
```

---

## Agent Configuration Options

```typescript
const agent = new HttpAgent({
  // Identity
  agentId: "assistant-v1",
  description: "A helpful coding assistant",
  
  // Conversation
  threadId: "thread-abc-123",
  
  // Initial state
  initialMessages: [
    { id: "sys", role: "system", content: "You are helpful." }
  ],
  initialState: {
    language: "TypeScript",
    verbosity: "concise"
  },
  
  // Connection
  url: "https://api.example.com/agent",
  headers: { Authorization: "Bearer xxx" }
})
```

---

## Cloning Agents

Need a copy with the same state?

```typescript
const agentCopy = agent.clone()

// agentCopy has same messages, state, config
// But runs independently
```

---

## The Big Picture

```
┌─────────────────────────────────────────────────────────────────┐
│                         YOUR APP                                │
│                                                                 │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐        │
│   │   UI Layer  │    │   Agent     │    │  AI Service │        │
│   │             │←──→│  (AG-UI)    │←──→│  (GPT, etc) │        │
│   │  - Shows    │    │             │    │             │        │
│   │    text     │    │  - Streams  │    │  - Thinks   │        │
│   │  - Handles  │    │    events   │    │  - Generates│        │
│   │    tools    │    │  - Manages  │    │    text     │        │
│   │             │    │    state    │    │             │        │
│   └─────────────┘    └─────────────┘    └─────────────┘        │
│                                                                 │
│   Events flow:  RUN_STARTED → TEXT → TOOL_CALL → TEXT → DONE   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Summary Table

| Component | What It Is | What It Does |
|-----------|------------|--------------|
| `AbstractAgent` | Base class | Blueprint for all agents |
| `HttpAgent` | Built-in agent | Connects to HTTP endpoints |
| `run()` | Required method | Returns event stream |
| `messages` | Array | Conversation history |
| `state` | Object | Persistent data |
| `tools` | Array | Functions agent can call |
| `runAgent()` | Method | Starts the agent, returns Observable |

---

## One-Sentence Summary

> **An AG-UI Agent is a class with a `run()` method that takes input (messages, tools, context), talks to an AI, and streams back typed events — giving your app a universal way to connect to any AI service.**
