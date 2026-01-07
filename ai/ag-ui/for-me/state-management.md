# AG-UI State Management: The Complete Guide

## What Is State? (The 30-Second Version)

**State = the current situation of your application.**

Think of a chess game:
- Where each piece is located
- Whose turn it is
- What moves happened before

That's state. AG-UI lets the **AI and the UI share this state** in real-time—both can read it, both can change it.

---

## Why Share State?

Without shared state:
```
AI works → finishes → shows result
(User waits blindly)
```

With shared state:
```
AI works → User sees progress → User can intervene → AI adapts
(True collaboration)
```

**The AI knows what the user is doing. The user knows what the AI is thinking.**

---

## Two Ways to Sync State

| Method | What It Sends | When to Use |
|--------|--------------|-------------|
| **Snapshot** | Everything | Starting up, reconnecting, major changes |
| **Delta** | Only what changed | Frequent small updates |

---

## Method 1: State Snapshot

**Send the entire state at once.**

```typescript
{
  type: "STATE_SNAPSHOT",
  snapshot: {
    user: { name: "John", plan: "pro" },
    document: { title: "Report", pages: 12 },
    aiStatus: "thinking"
  }
}
```

**When frontend receives this:** Throw away old state. Replace with this.

**Use snapshots when:**
- App first loads
- Connection was lost and restored
- Something major changed (easier to send everything than calculate differences)

---

## Method 2: State Delta

**Send only what changed.**

```typescript
{
  type: "STATE_DELTA",
  delta: [
    { op: "replace", path: "/aiStatus", value: "done" },
    { op: "replace", path: "/document/pages", value: 13 }
  ]
}
```

This says: "Change `aiStatus` to 'done' and `document.pages` to 13. Leave everything else alone."

**Why deltas?**
- Faster (less data to send)
- More efficient for large states with small changes
- Better for real-time streaming

---

## JSON Patch: The Delta Language

AG-UI uses **JSON Patch (RFC 6902)**—a standard way to describe changes.

### The 6 Operations

| Operation | What It Does | Example |
|-----------|-------------|---------|
| `add` | Insert new value | Add a new field |
| `remove` | Delete value | Remove a field |
| `replace` | Swap value | Change existing field |
| `move` | Relocate value | Move from one place to another |
| `copy` | Duplicate value | Copy to new location |
| `test` | Verify value | Check before changing |

---

### Practical Examples

**Starting state:**
```json
{
  "user": { "name": "John" },
  "tasks": ["task1", "task2"],
  "status": "active"
}
```

---

**ADD** - Insert new value:
```json
{ "op": "add", "path": "/user/email", "value": "john@example.com" }
```
Result: `user` now has `{ name: "John", email: "john@example.com" }`

---

**REPLACE** - Change existing value:
```json
{ "op": "replace", "path": "/status", "value": "paused" }
```
Result: `status` is now `"paused"`

---

**REMOVE** - Delete value:
```json
{ "op": "remove", "path": "/tasks/0" }
```
Result: `tasks` is now `["task2"]` (first item removed)

---

**MOVE** - Relocate value:
```json
{ "op": "move", "from": "/tasks/0", "path": "/completedTasks/0" }
```
Result: First task moved from `tasks` array to `completedTasks` array

---

## The Path System

Paths use **JSON Pointer (RFC 6901)**:

```
/user/name       → state.user.name
/tasks/0         → state.tasks[0]  (first item)
/tasks/-         → append to end of tasks array
/deep/nested/val → state.deep.nested.val
```

---

## How It Works in Code

Frontend receives delta and applies it:

```typescript
case "STATE_DELTA": {
  // Get the patch operations
  const { delta } = event;
  
  // Apply all changes to current state
  const result = applyPatch(currentState, delta);
  
  // Update to new state
  currentState = result.newDocument;
  
  // Re-render UI with new state
  updateUI(currentState);
}
```

**Key behaviors:**
- Changes apply in order (first patch, then second, etc.)
- All-or-nothing (if one fails, none apply)
- Original state isn't mutated until all patches succeed

---

## Real-Time Collaboration Flow

```
┌─────────────┐                    ┌─────────────┐
│   FRONTEND  │◄──── SHARED ──────►│    AGENT    │
│             │       STATE        │             │
└─────────────┘                    └─────────────┘
      │                                   │
      │  User types in form               │
      ├──────────────────────────────────►│
      │  Delta: /userInput = "..."        │
      │                                   │
      │                                   │  AI processes
      │                                   │
      │◄──────────────────────────────────┤
      │  Delta: /aiSuggestion = "..."     │
      │                                   │
      │  User sees suggestion             │
      │  User approves                    │
      ├──────────────────────────────────►│
      │  Delta: /approved = true          │
      │                                   │
```

Both sides read and write to the same state object.

---

## Human-in-the-Loop Example

**Scenario:** AI drafts an email, user approves before sending.

### Step 1: AI proposes action
```typescript
// AI emits delta
{
  op: "add",
  path: "/proposal",
  value: {
    action: "send_email",
    to: "client@example.com",
    subject: "Project Update",
    body: "Hi, here's the latest...",
    status: "pending_approval"
  }
}
```

### Step 2: Frontend shows proposal
```jsx
function ProposalCard({ proposal }) {
  return (
    <div>
      <h3>AI wants to send email:</h3>
      <p>To: {proposal.to}</p>
      <p>Subject: {proposal.subject}</p>
      <p>{proposal.body}</p>
      <button onClick={approve}>Approve</button>
      <button onClick={reject}>Reject</button>
    </div>
  );
}
```

### Step 3: User approves
```typescript
// Frontend emits delta
{
  op: "replace",
  path: "/proposal/status",
  value: "approved"
}
```

### Step 4: AI sees approval, executes
```typescript
// AI sees status changed to "approved"
// Proceeds with sending email
// Emits completion delta
{
  op: "replace",
  path: "/proposal/status",
  value: "completed"
}
```

---

## CopilotKit Integration

Real-world implementation pattern:

**Frontend (React):**
```typescript
const { state, setState } = useCoAgent({
  name: "research-agent",
  initialState: {
    topic: "",
    findings: [],
    status: "idle"
  }
});

// Read AI state
<p>Status: {state.status}</p>
<ul>{state.findings.map(f => <li>{f}</li>)}</ul>

// Write to AI state
<input onChange={e => setState({ ...state, topic: e.target.value })} />
```

**Backend (LangGraph):**
```python
async def research_node(state, config):
    # Do research...
    new_findings = await do_research(state["topic"])
    
    # Update state - frontend sees this immediately
    await copilotkit_emit_state(config, {
        "findings": new_findings,
        "status": "complete"
    })
    
    return state
```

---

## Error Handling

What if a patch fails?

```typescript
try {
  state = applyPatch(state, delta).newDocument;
} catch (error) {
  // Patch failed - state might be inconsistent
  // Request fresh snapshot to resync
  console.warn("Patch failed, requesting snapshot");
  requestStateSnapshot();
}
```

**Common failures:**
- Path doesn't exist (`/user/email` when `user` is null)
- Wrong type (trying to add to array index on object)
- Test operation fails

---

## Snapshot vs Delta Decision Tree

```
State change happened
        │
        ▼
Is this the first sync?
        │
   YES──┴──NO
    │      │
    ▼      ▼
SNAPSHOT  How much changed?
              │
        LITTLE┴──LOTS
          │       │
          ▼       ▼
        DELTA   SNAPSHOT
```

---

## Best Practices

| Do | Don't |
|----|-------|
| Use deltas for small frequent updates | Send snapshots every time |
| Structure state for easy patching | Deeply nest everything |
| Handle patch failures gracefully | Ignore errors |
| Keep sensitive data out of shared state | Put passwords in state |
| Request snapshot if desync suspected | Assume state is always correct |

---

## State Structure Tips

**Good** (easy to patch):
```json
{
  "user": { "name": "John", "email": "..." },
  "tasks": { 
    "task_1": { "title": "..." },
    "task_2": { "title": "..." }
  },
  "ui": { "currentTab": "home" }
}
```

**Bad** (hard to patch efficiently):
```json
{
  "data": {
    "deeply": {
      "nested": {
        "value": "hard to reference"
      }
    }
  }
}
```

---

## Summary

| Concept | What It Does |
|---------|--------------|
| **Shared State** | Single source of truth both AI and UI can access |
| **Snapshot** | Complete state replacement |
| **Delta** | Incremental state change |
| **JSON Patch** | Standard format for describing changes |
| **Path** | JSON Pointer to locate values (`/user/name`) |
| **Operations** | add, remove, replace, move, copy, test |
| **Human-in-the-loop** | User sees AI state, can intervene |

**The bottom line:** State management lets AI and humans work on the same "document" simultaneously. Snapshots give you the full picture. Deltas give you efficient updates. Together, they enable real-time collaboration between human and machine.
