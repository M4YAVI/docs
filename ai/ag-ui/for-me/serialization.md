# AG-UI Serialization: The Complete Guide

## What Is Serialization? (The 30-Second Version)

**Serialization = saving a conversation so you can restore it later.**

Every interaction between user and AI generates events. Serialization:
1. **Saves** those events to storage (database, file)
2. **Restores** them later (page refresh, reconnect)
3. **Compacts** them to save space
4. **Branches** them like git (time travel, alternative paths)

---

## The Three Core Concepts

| Concept | What It Does |
|---------|--------------|
| **Stream Serialization** | Convert events to JSON → save → load → convert back |
| **Event Compaction** | Shrink 100 events into 5 events (same meaning, less data) |
| **Run Lineage** | Track which run branched from which (like git history) |

---

## Why You Need This

| Problem | Serialization Solves It |
|---------|------------------------|
| User refreshes page | Restore entire conversation from storage |
| Connection drops | Resume exactly where you left off |
| User wants to "undo" | Branch from earlier point, try different path |
| Storage getting huge | Compact events to smaller representation |
| Debug what happened | Replay event stream step-by-step |

---

## How Events Become Storable

### Save Events
```typescript
// Your event stream (what happened during the session)
const events: BaseEvent[] = [
  { type: "RUN_STARTED", runId: "run1", ... },
  { type: "TEXT_MESSAGE_START", messageId: "msg1", ... },
  { type: "TEXT_MESSAGE_CONTENT", delta: "Hello", ... },
  { type: "TEXT_MESSAGE_END", ... },
  { type: "RUN_FINISHED", ... }
];

// Convert to JSON string
const serialized = JSON.stringify(events);

// Save to database/file/storage
await database.save(threadId, serialized);
```

### Restore Events
```typescript
// Load from storage
const serialized = await database.load(threadId);

// Convert back to events
const events = JSON.parse(serialized);

// Now replay these events to rebuild UI state
events.forEach(event => processEvent(event));
```

**That's serialization.** Events → JSON → storage → JSON → events.

---

## Event Compaction: The Space Saver

### The Problem

A simple "Hello world" message generates 4 events:

```typescript
[
  { type: "TEXT_MESSAGE_START", messageId: "msg1", role: "user" },
  { type: "TEXT_MESSAGE_CONTENT", messageId: "msg1", delta: "Hello " },
  { type: "TEXT_MESSAGE_CONTENT", messageId: "msg1", delta: "world" },
  { type: "TEXT_MESSAGE_END", messageId: "msg1" }
]
```

Multiply by thousands of messages = massive storage.

### The Solution: Compact

```typescript
const compacted = compactEvents(events);
```

**Before (4 events):**
```typescript
[
  { type: "TEXT_MESSAGE_START", messageId: "msg1", role: "user" },
  { type: "TEXT_MESSAGE_CONTENT", messageId: "msg1", delta: "Hello " },
  { type: "TEXT_MESSAGE_CONTENT", messageId: "msg1", delta: "world" },
  { type: "TEXT_MESSAGE_END", messageId: "msg1" }
]
```

**After (1 event):**
```typescript
[
  { 
    type: "MESSAGES_SNAPSHOT", 
    messages: [{ id: "msg1", role: "user", content: "Hello world" }]
  }
]
```

Same information. 75% less storage.

---

## Compaction Rules

| Event Type | Compaction Rule |
|------------|-----------------|
| `TEXT_MESSAGE_START/CONTENT/END` | → Single message in `MESSAGES_SNAPSHOT` |
| Multiple `TEXT_MESSAGE_CONTENT` | → Concatenate into one content string |
| `TOOL_CALL_START/ARGS/END` | → Single tool call record |
| Multiple `STATE_DELTA` | → Single final `STATE_SNAPSHOT` |
| Superseded snapshots | → Keep only the latest |

### Full Compaction Example

**Before (6 events):**
```json
[
  { "type": "TEXT_MESSAGE_START", "messageId": "msg1", "role": "user" },
  { "type": "TEXT_MESSAGE_CONTENT", "messageId": "msg1", "delta": "Hello " },
  { "type": "TEXT_MESSAGE_CONTENT", "messageId": "msg1", "delta": "world" },
  { "type": "TEXT_MESSAGE_END", "messageId": "msg1" },
  { "type": "STATE_DELTA", "delta": [{ "op": "add", "path": "/count", "value": 1 }] },
  { "type": "STATE_DELTA", "delta": [{ "op": "replace", "path": "/count", "value": 2 }] }
]
```

**After (2 events):**
```json
[
  {
    "type": "MESSAGES_SNAPSHOT",
    "messages": [{ "id": "msg1", "role": "user", "content": "Hello world" }]
  },
  {
    "type": "STATE_SNAPSHOT",
    "snapshot": { "count": 2 }
  }
]
```

---

## Run Lineage: Git for Conversations

### The Problem

User asks: "Tell me about Paris"
AI responds with Paris info.
User says: "Actually, I meant London"

**Option A:** Delete Paris response, start over (loses history)
**Option B:** Branch from before Paris, try London (keeps everything)

### The Solution: parentRunId

```typescript
// Original conversation about Paris
{
  type: "RUN_STARTED",
  threadId: "thread1",
  runId: "run1",
  input: { messages: [{ content: "Tell me about Paris" }] }
}
// ... Paris response events ...
{ type: "RUN_FINISHED", runId: "run1" }

// User wants to try London instead - BRANCH from run1
{
  type: "RUN_STARTED",
  threadId: "thread1",
  runId: "run2",
  parentRunId: "run1",  // ← THIS creates the branch
  input: { messages: [{ content: "Tell me about London instead" }] }
}
```

### Visual: The Branch Structure

```
run1 (Paris) ──────────────────────────────┐
     │                                      │
     ├── run2 (London) ← branches from run1 │
     │        │                             │
     │        └── run4 ← branches from run2 │
     │                                      │
     └── run3 (Tokyo) ← also branches run1  │
              │                             │
              └── run5 ← branches from run3 │
```

**Like git:**
- `run1` = main branch
- `run2`, `run3` = branches from main
- `run4`, `run5` = branches from branches

---

## The RunStarted Event (Updated)

```typescript
{
  type: "RUN_STARTED",
  threadId: "thread1",        // Which conversation
  runId: "run2",              // This run's unique ID
  parentRunId: "run1",        // Branch from this run (optional)
  input: {                    // Exact input for THIS run
    messages: [
      { id: "msg2", role: "user", content: "New question" }
    ]
  }
}
```

| Field | Purpose |
|-------|---------|
| `threadId` | Groups all runs in same conversation |
| `runId` | Unique identifier for this specific run |
| `parentRunId` | Which run this branches from (enables time travel) |
| `input` | Exactly what was sent to the agent for this run |

---

## Input Normalization

### The Problem

Run 2 needs messages from Run 1 as context. But those messages are already stored. Why duplicate?

### The Solution

Only store **new** messages in `input`:

```typescript
// Run 1: Full input
{
  type: "RUN_STARTED",
  runId: "run1",
  input: { 
    messages: [{ id: "msg1", role: "user", content: "Hello" }]
  }
}

// Run 2: Only NEW messages (msg1 already in history)
{
  type: "RUN_STARTED",
  runId: "run2",
  parentRunId: "run1",
  input: { 
    messages: [{ id: "msg2", role: "user", content: "How are you?" }]
    // msg1 NOT included - already exists in run1's history
  }
}
```

When restoring, walk the parent chain to reconstruct full context.

---

## Complete Workflow Example

### 1. Session Happens
```typescript
const events = [];

// User starts conversation
events.push({ type: "RUN_STARTED", runId: "run1", threadId: "t1", input: {...} });
events.push({ type: "TEXT_MESSAGE_START", messageId: "m1", role: "user" });
events.push({ type: "TEXT_MESSAGE_CONTENT", messageId: "m1", delta: "Hello" });
events.push({ type: "TEXT_MESSAGE_END", messageId: "m1" });
// ... AI response events ...
events.push({ type: "RUN_FINISHED", runId: "run1" });
```

### 2. Save to Storage
```typescript
const serialized = JSON.stringify(events);
await db.save("thread_t1", serialized);
```

### 3. User Closes Browser

### 4. User Returns, Restore Session
```typescript
const serialized = await db.load("thread_t1");
const events = JSON.parse(serialized);

// Option A: Replay all events
events.forEach(e => processEvent(e));

// Option B: Compact first, then restore
const compacted = compactEvents(events);
compacted.forEach(e => processEvent(e));
```

### 5. Continue Conversation
```typescript
// New run continues from where we left off
events.push({ 
  type: "RUN_STARTED", 
  runId: "run2", 
  parentRunId: "run1",  // continues from run1
  input: { messages: [newMessage] }
});
```

---

## Implementation Tips

| Tip | Why |
|-----|-----|
| **Append-only storage** | Never delete events; add new ones. Enables full history replay. |
| **Compress long histories** | Use gzip/brotli on serialized JSON |
| **Index by threadId + runId** | Fast lookup when restoring |
| **Compact before long-term storage** | Save space, keep meaning |
| **Keep raw events for debugging** | Compact copies, not originals |

---

## Storage Schema Example

```sql
CREATE TABLE event_streams (
  thread_id    VARCHAR(255),
  run_id       VARCHAR(255),
  parent_run   VARCHAR(255),
  events       JSONB,        -- Serialized event array
  created_at   TIMESTAMP,
  
  PRIMARY KEY (thread_id, run_id)
);

-- Fast lookups
CREATE INDEX idx_thread ON event_streams(thread_id);
CREATE INDEX idx_parent ON event_streams(parent_run);
```

---

## Restoring a Branch

To restore conversation at `run3`:

```typescript
function getFullHistory(runId) {
  const events = [];
  let currentRun = runId;
  
  // Walk up the parent chain
  while (currentRun) {
    const runData = await db.load(currentRun);
    events.unshift(...runData.events);  // prepend
    currentRun = runData.parentRunId;
  }
  
  return events;
}

// Get everything from start to run3
const fullHistory = await getFullHistory("run3");
```

---

## Summary

| Concept | What It Does |
|---------|--------------|
| **Serialization** | Convert events ↔ JSON for storage |
| **Compaction** | Shrink many events into fewer (same meaning) |
| **parentRunId** | Link runs together (enables branching) |
| **input** | Exact messages sent to agent for this run |
| **Normalized input** | Skip messages already in parent history |
| **Append-only** | Never delete; always add new events |
| **Branch restore** | Walk parent chain to rebuild full context |

**The bottom line:** Serialization lets you save entire conversations, restore them later, save storage space through compaction, and create alternative conversation branches—all from the same event stream. Think of it as "save game" for AI conversations, with the ability to load any checkpoint and try a different path.
