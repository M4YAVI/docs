

### One-liner + pro tip for every single thing you will ever use

### Core Philosophy (read once, tattoo it)
Everything is a **stream of strongly-typed events**.  
You never get a full message at once → you get start/content/end chunks.  
This is intentional and makes streaming UIs buttery smooth.

### Install
```bash
npm install @ag-ui/core
```

### RunAgentInput – What you POST to start a run
```ts
{
  threadId: string          // conversation thread
  runId: string             // unique for this execution
  parentRunId?: string      // for branching/time-travel (GOATED feature)
  state: any                // your agent's memory / custom state
  messages: Message[]       // full history
  tools: Tool[]             // tools available this run
  context: Context[]        // extra facts injected
  forwardedProps: any       // whatever you want, gets passed through
}
```

### Message – The Union King
```ts
type Message = DeveloperMessage | SystemMessage | AssistantMessage | UserMessage | ToolMessage | ActivityMessage
```

### Role
```ts
"developer" | "system" | "assistant" | "user" | "tool" | "activity"
```

### DeveloperMessage / SystemMessage
Same shape, just different role. Use for hidden instructions.

### AssistantMessage – The star of the show
```ts
{
  role: "assistant"
  content?: string          // final text (optional if only tool calls)
  toolCalls?: ToolCall[]    // parallel tool calls = GOATED
}
```

### UserMessage – Multimodal god
```ts
content: string | InputContent[]   // string = text, array = images/files/text mixed
```

### InputContent – Multimodal pieces
```ts
{ type: "text";  text: string }
{ type: "binary"; mimeType: string; id?|url?|data?: string; filename?: string }
```

### ToolMessage – Tool response
```ts
{
  role: "tool"
  toolCallId: string        // MUST match the call
  content: string           // result
  error?: string            // if tool failed
}
```

### ActivityMessage – Structured live updates (pure sex for UI)
```ts
{
  role: "activity"
  activityType: "PLAN" | "SEARCH" | "BRAINSTORM" | custom
  content: Record<string, any>   // your structured data
}
```
Use this for live planning, search results, progress trees, etc. → replaces hacky markdown.

### ToolCall & FunctionCall
```ts
toolCalls: [{
  id: string
  type: "function"
  function: {
    name: string
    arguments: string   // JSON string, not object (OpenAI style)
  }
}]
```

### Tool definition
```ts
{
  name: string
  description: string
  parameters: JSONSchema   // exact OpenAI tools format
}
```

### Context – Inject facts without messages
```ts
{ description: "User's subscription tier", value: "pro" }
```

### State – Your agent's brain
```ts
type State = any   // usually a typed interface in real apps
```

### EventType – Memorize these 5 groups

**Lifecycle** (must handle)
- RUN_STARTED
- RUN_FINISHED
- RUN_ERROR
- STEP_STARTED
- STEP_FINISHED

**Text Streaming** (the classic)
- TEXT_MESSAGE_START → { messageId, role: "assistant" }
- TEXT_MESSAGE_CONTENT → { delta: "..." }
- TEXT_MESSAGE_END → { messageId }

**Tool Call Streaming** (parallel tool calls = 2025 meta)
- TOOL_CALL_START → { toolCallId, toolCallName }
- TOOL_CALL_ARGS → { delta: "{\"search\": \"xAI\"}" }
- TOOL_CALL_END → { toolCallId }
- TOOL_CALL_RESULT → { toolCallId, content, messageId }

**State Management** (reactive UIs go crazy)
- STATE_SNAPSHOT → full state
- STATE_DELTA → JSON Patch (tiny updates = performance god)

**Activity System** (the real GOATED feature nobody talks about)
- ACTIVITY_SNAPSHOT → full structured activity
- ACTIVITY_DELTA → JSON Patch updates in real time

**Bonus sugar**
- TEXT_MESSAGE_CHUNK → send one event, client auto-splits into start/content/end
- TOOL_CALL_CHUNK → same but for tools (use these in 99% of cases)

### Pro Tips That Separate Juniors from Gods

1. Always use TEXT_MESSAGE_CHUNK / TOOL_CALL_CHUNK in your backend → less code, perfect streaming
2. Use ActivityMessage + ACTIVITY_DELTA for anything structured (plans, search results, file analysis) → users will think you're from 2030
3. Parallel tool calls are free and encouraged → call 5 tools at once, show live results as they come
4. parentRunId + threadId = time travel / branching UIs basically for free
5. StateDelta + JSON Patch = you can have a live reactive canvas/state panel with zero effort
6. Never send full messages array in RunAgentInput if the agent already has history → waste of tokens

### Golden Rule
Your frontend should be 100% event-driven.  
Store nothing locally except what comes from events.  
This makes undo/redo, branching, multiplayer, debugging trivial.

You're now in the top 1% of agent UI engineers.

Go build the sickest agent frontend the world has ever seen.  
The protocol is perfect. Don't fight it — embrace it.

You are unstoppable.
