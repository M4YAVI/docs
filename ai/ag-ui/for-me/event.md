
# AG-UI Events: The Complete Guide

## What Are Events?

**Events are messages.**  
Every time the AI agent does something, it sends an event to your app.

- Agent: “I’m starting” → Event  
- Agent: “Here’s some text” → Event  
- Agent: “I’m using a tool” → Event  
- Agent: “I’m done” → Event  

Your app listens to these events and updates the screen accordingly.

---

## Every Event Has This Shape

```ts
{
  type: "TEXT_MESSAGE_CONTENT", // What kind of event
  timestamp: 1234567890,        // When (optional)
  rawEvent: { ... }             // Original data if transformed (optional)
}
````

---

## The 6 Event Categories

| Category         | Purpose                                |
| ---------------- | -------------------------------------- |
| Lifecycle        | Run started, finished, or failed       |
| Text Message     | Streaming text word-by-word            |
| Tool Call        | Agent using tools (search, calculator) |
| State Management | Efficient data synchronization         |
| Activity         | Progress updates between messages      |
| Special          | Custom or external events              |

---

## Category 1: Lifecycle Events

**Purpose:** Know when things start and end.

### The Flow

```
RUN_STARTED
    ├── STEP_STARTED (optional)
    │   └── STEP_FINISHED
    ├── STEP_STARTED (optional)
    │   └── STEP_FINISHED
    └── RUN_FINISHED  ──or──  RUN_ERROR
```

### The Events

| Event        | What It Means                | Key Properties  |
| ------------ | ---------------------------- | --------------- |
| RunStarted   | Agent begins work            | threadId, runId |
| RunFinished  | Agent completed successfully | runId, result?  |
| RunError     | Something broke              | message, code   |
| StepStarted  | Sub-task begins              | stepName        |
| StepFinished | Sub-task ends                | stepName        |

**Rule:** Every run **must** have `RunStarted` and either `RunFinished` or `RunError`.

---

## Category 2: Text Message Events

**Purpose:** Stream text as it’s generated (ChatGPT-style).

### Pattern: Start → Content → End

```
TextMessageStart
TextMessageContent
TextMessageContent
TextMessageContent
TextMessageEnd
```

### Example

```
TextMessageStart    (messageId: "msg_1")
TextMessageContent  "Hello"
TextMessageContent  " world"
TextMessageContent  "!"
TextMessageEnd
```

Your app concatenates: `Hello` + ` world` + `!` → **Hello world!**

### The Events

| Event              | Purpose          | Key Properties   |
| ------------------ | ---------------- | ---------------- |
| TextMessageStart   | Message begins   | messageId, role  |
| TextMessageContent | Text chunk       | messageId, delta |
| TextMessageEnd     | Message complete | messageId        |

### Shortcut: `TEXT_MESSAGE_CHUNK`

```ts
{ type: "TEXT_MESSAGE_CHUNK", messageId: "msg_1", delta: "Hello" }
{ type: "TEXT_MESSAGE_CHUNK", messageId: "msg_1", delta: " world!" }
```

AG-UI automatically expands to **Start → Content → End**.

---

## Category 3: Tool Call Events

**Purpose:** Show when the agent uses tools.

### Pattern

```
ToolCallStart
ToolCallArgs
ToolCallArgs
ToolCallEnd
ToolCallResult
```

### The Events

| Event          | Purpose                | Key Properties           |
| -------------- | ---------------------- | ------------------------ |
| ToolCallStart  | Tool invocation begins | toolCallId, toolCallName |
| ToolCallArgs   | Argument chunk (JSON)  | toolCallId, delta        |
| ToolCallEnd    | Arguments complete     | toolCallId               |
| ToolCallResult | Tool output            | toolCallId, content      |

### Shortcut: `TOOL_CALL_CHUNK`

```ts
{
  type: "TOOL_CALL_CHUNK",
  toolCallId: "tc_1",
  toolCallName: "search",
  delta: '{"query":"weather"}'
}
```

Automatically expands to **Start → Args → End**.

---

## Category 4: State Management Events

**Purpose:** Keep your app’s data in sync efficiently.

### Pattern: Snapshot → Delta

```
StateSnapshot
StateDelta
StateDelta
StateSnapshot (optional resync)
```

### The Events

| Event            | Purpose                 | Key Properties     |
| ---------------- | ----------------------- | ------------------ |
| StateSnapshot    | Full state              | snapshot           |
| StateDelta       | Incremental update      | delta (JSON Patch) |
| MessagesSnapshot | Full conversation state | messages[]         |

### Why This Pattern?

* Snapshot: ~100 KB
* Delta: ~50 bytes

Send full state once, then tiny updates.

### JSON Patch Example

```js
// State: { user: { name: "Bob", age: 30 } }

{
  delta: [
    { op: "replace", path: "/user/name", value: "Alice" }
  ]
}

// Result:
// { user: { name: "Alice", age: 30 } }
```

---

## Category 5: Activity Events

**Purpose:** Show progress updates *between* messages
(e.g. “Searching…”, “Analyzing…”).

### The Events

| Event            | Purpose             | Key Properties                   |
| ---------------- | ------------------- | -------------------------------- |
| ActivitySnapshot | Full activity state | messageId, activityType, content |
| ActivityDelta    | Incremental update  | messageId, activityType, patch   |

### Example

```
ActivitySnapshot
{ status: "searching", query: "weather" }

ActivityDelta
{ status: "found 10 results" }
```

---

## Category 6: Special Events

### Raw Event

**Purpose:** Pass through events from external systems.

```ts
{
  type: "RAW",
  event: { /* external payload */ },
  source: "external-system"
}
```

### Custom Event

**Purpose:** Your own application-specific events.

```ts
{
  type: "CUSTOM",
  name: "user_preference_changed",
  value: { theme: "dark" }
}
```

---

## Draft / Experimental Events

### Reasoning Events

Expose the model’s reasoning stream.

```
ReasoningStart
ReasoningMessageStart
ReasoningMessageContent
ReasoningMessageEnd
ReasoningEnd
```

### Meta Events

Side-band annotations (feedback, reactions).

```ts
{
  type: "META",
  metaType: "thumbs_up",
  payload: { messageId: "msg_1" }
}
```

---

## The Two Core Patterns

### Pattern 1: Start → Content → End (Streaming)

Used for:

* Text Messages
* Tool Calls
* Reasoning

```
START → CONTENT → CONTENT → END
```

**Why:** Real-time UI updates as data is generated.

---

### Pattern 2: Snapshot → Delta (State Sync)

Used for:

* State
* Activities

```
SNAPSHOT → DELTA → DELTA → (optional) SNAPSHOT
```

**Why:** Send full state once, then only changes. Saves bandwidth.

---

## Visual Summary

```
LIFECYCLE                    TEXT                         TOOLS
─────────────────           ─────────────────           ─────────────────
RunStarted                  TextMessageStart            ToolCallStart
  StepStarted               TextMessageContent          ToolCallArgs
  StepFinished              TextMessageEnd              ToolCallEnd
RunFinished                 TextMessageChunk            ToolCallResult
RunError                                              ToolCallChunk

STATE                        ACTIVITY                    SPECIAL
─────────────────           ─────────────────           ─────────────────
StateSnapshot               ActivitySnapshot            Raw
StateDelta                  ActivityDelta               Custom
MessagesSnapshot
```

---

## Implementation Rules

* Process events **in order**
* Match by **ID** (`messageId`, `toolCallId`)
* Concatenate **deltas**
* Apply **JSON Patch operations sequentially**
* Handle errors gracefully — `RunError` is terminal

---

## One-Line Summary

**AG-UI events are typed messages that follow two core patterns—`Start → Content → End` for streaming and `Snapshot → Delta` for state sync—enabling real-time, efficient communication between AI agents and your app.**

```
```
