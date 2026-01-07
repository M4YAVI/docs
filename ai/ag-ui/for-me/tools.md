# AG-UI Tools: The Complete Guide

## What Are Tools? (The 30-Second Version)

**Tools = things the AI can ask to happen.**

AI can think and write. But it can't:
- Check your calendar
- Send an email
- Ask you to approve something
- Navigate to a different page

Tools let AI request these actions. The frontend executes them.

```
AI: "I need to check the weather" → calls weather tool → gets result → uses it
```

---

## The Core Idea

```
┌─────────────────┐         ┌─────────────────┐
│       AI        │         │    FRONTEND     │
│   (reasoning)   │────────►│   (doing)       │
└─────────────────┘  tool   └─────────────────┘
                     call
```

**AI decides WHAT to do. Frontend decides HOW to do it.**

---

## Tool Structure

Every tool has three parts:

```typescript
{
  name: "sendEmail",           // What it's called
  description: "Send an email to someone",  // What it does
  parameters: {                // What inputs it needs
    type: "object",
    properties: {
      to: { type: "string", description: "Recipient email" },
      subject: { type: "string", description: "Email subject" },
      body: { type: "string", description: "Email content" }
    },
    required: ["to", "subject", "body"]
  }
}
```

| Part | Purpose |
|------|---------|
| `name` | Unique identifier AI uses to call it |
| `description` | Tells AI when/why to use this tool |
| `parameters` | JSON Schema defining expected inputs |

---

## Frontend Defines Tools (Key Insight)

**Tools are defined in YOUR app, not the AI.**

```typescript
// You define what tools exist
const tools = [
  {
    name: "confirmAction",
    description: "Ask user to approve something",
    parameters: { ... }
  },
  {
    name: "fetchUserData",
    description: "Get user information",
    parameters: { ... }
  }
];

// You pass them to the agent
agent.runAgent({
  tools: tools,  // ← AI now knows these tools exist
  messages: [...]
});
```

### Why This Matters

| Benefit | Explanation |
|---------|-------------|
| **Security** | You control what AI can do |
| **Dynamic** | Add/remove tools based on user permissions |
| **Separation** | AI reasons, frontend executes |
| **Context-aware** | Different tools for different situations |

---

## Tool Call Lifecycle

When AI wants to use a tool, three events happen:

### Step 1: TOOL_CALL_START
```typescript
{
  type: "TOOL_CALL_START",
  toolCallId: "call_123",     // Unique ID for this call
  toolCallName: "sendEmail",  // Which tool
  parentMessageId: "msg_456"  // Optional: which message triggered this
}
```

### Step 2: TOOL_CALL_ARGS (Streaming)
Arguments come in pieces:
```typescript
{ type: "TOOL_CALL_ARGS", toolCallId: "call_123", delta: '{"to":"bob@' }
{ type: "TOOL_CALL_ARGS", toolCallId: "call_123", delta: 'example.com",' }
{ type: "TOOL_CALL_ARGS", toolCallId: "call_123", delta: '"subject":"Hi"}' }
```

Frontend accumulates: `{"to":"bob@example.com","subject":"Hi"}`

### Step 3: TOOL_CALL_END
```typescript
{
  type: "TOOL_CALL_END",
  toolCallId: "call_123"
}
```

Now frontend has complete arguments. Execute the tool.

---

## Visual Flow

```
AI                          Frontend                      User
 │                              │                           │
 ├─ TOOL_CALL_START ───────────►│                           │
 │  (sendEmail)                 │                           │
 │                              │                           │
 ├─ TOOL_CALL_ARGS ────────────►│                           │
 ├─ TOOL_CALL_ARGS ────────────►│                           │
 ├─ TOOL_CALL_ARGS ────────────►│                           │
 │                              │                           │
 ├─ TOOL_CALL_END ─────────────►│                           │
 │                              │                           │
 │                              ├── Execute sendEmail() ────┤
 │                              │                           │
 │                              ├── Show "Email Sent!" ────►│
 │                              │                           │
 │◄──── TOOL MESSAGE ───────────┤                           │
 │      (result: "success")     │                           │
 │                              │                           │
 ├─ Continue response ─────────►│                           │
```

---

## Tool Results

After execution, send result back as a **tool message**:

```typescript
{
  id: "result_789",
  role: "tool",
  content: "Email sent successfully",  // Result as string
  toolCallId: "call_123"               // Links to original call
}
```

AI now knows the outcome and can respond accordingly.

---

## Human-in-the-Loop: The Power Pattern

This is where tools shine.

### The Problem
AI wants to do something important. Should it just do it? No. Ask the human.

### The Solution
Define a tool that asks for confirmation:

```typescript
{
  name: "confirmAction",
  description: "Ask user to approve before proceeding",
  parameters: {
    type: "object",
    properties: {
      action: { 
        type: "string", 
        description: "What action needs approval" 
      },
      importance: { 
        type: "string", 
        enum: ["low", "medium", "high", "critical"] 
      }
    },
    required: ["action"]
  }
}
```

### The Flow

```
1. AI: "I should delete old files"
          │
          ▼
2. AI calls confirmAction tool
   args: { action: "Delete 500 old files", importance: "high" }
          │
          ▼
3. Frontend shows dialog to user:
   ┌─────────────────────────────────┐
   │  ⚠️ AI wants to:                │
   │  "Delete 500 old files"         │
   │                                 │
   │  [Cancel]  [Approve]            │
   └─────────────────────────────────┘
          │
          ▼
4. User clicks Approve
          │
          ▼
5. Frontend sends tool result:
   { role: "tool", content: "approved", toolCallId: "..." }
          │
          ▼
6. AI receives approval, proceeds with deletion
```

**AI proposes. Human disposes.**

---

## Implementing Tools (React Example)

Using CopilotKit:

```typescript
import { useCopilotAction } from "@copilotkit/react-core";

useCopilotAction({
  name: "confirmAction",
  description: "Ask user to confirm an action",
  parameters: {
    type: "object",
    properties: {
      action: { type: "string" }
    },
    required: ["action"]
  },
  handler: async ({ action }) => {
    // Show confirmation dialog
    const userConfirmed = await showConfirmDialog(action);
    
    // Return result to AI
    return userConfirmed ? "approved" : "rejected";
  }
});
```

The `handler` function:
1. Receives parsed arguments
2. Does the actual work (show dialog, call API, etc.)
3. Returns result to AI

---

## Common Tool Types

### 1. User Confirmation
**Ask human to approve something**
```typescript
{
  name: "confirmAction",
  description: "Ask user to approve before proceeding",
  parameters: {
    properties: {
      action: { type: "string" },
      importance: { type: "string", enum: ["low", "medium", "high", "critical"] }
    }
  }
}
```

---

### 2. Data Retrieval
**Fetch information from your systems**
```typescript
{
  name: "fetchUserData",
  description: "Get information about a user",
  parameters: {
    properties: {
      userId: { type: "string" },
      fields: { type: "array", items: { type: "string" } }
    }
  }
}
```

---

### 3. UI Navigation
**Control the interface**
```typescript
{
  name: "navigateTo",
  description: "Go to a different page",
  parameters: {
    properties: {
      destination: { type: "string" },
      params: { type: "object" }
    }
  }
}
```

---

### 4. External Actions
**Do things in the real world**
```typescript
{
  name: "sendEmail",
  description: "Send an email",
  parameters: {
    properties: {
      to: { type: "string" },
      subject: { type: "string" },
      body: { type: "string" }
    }
  }
}
```

---

### 5. Content Generation
**Create media or documents**
```typescript
{
  name: "generateImage",
  description: "Create an image from description",
  parameters: {
    properties: {
      prompt: { type: "string" },
      style: { type: "string" },
      dimensions: {
        type: "object",
        properties: {
          width: { type: "number" },
          height: { type: "number" }
        }
      }
    }
  }
}
```

---

## Complete Example: Delete Files Workflow

### 1. Define the Tool
```typescript
const tools = [{
  name: "confirmDeletion",
  description: "Ask user to confirm file deletion",
  parameters: {
    type: "object",
    properties: {
      files: { 
        type: "array", 
        items: { type: "string" },
        description: "List of files to delete"
      },
      reason: {
        type: "string",
        description: "Why these files should be deleted"
      }
    },
    required: ["files", "reason"]
  }
}];
```

### 2. Handle the Tool Call
```typescript
function handleToolCall(toolName, args) {
  if (toolName === "confirmDeletion") {
    return showDeleteConfirmation(args.files, args.reason);
  }
}

async function showDeleteConfirmation(files, reason) {
  // Show modal with file list
  const modal = document.createElement('div');
  modal.innerHTML = `
    <h3>Delete ${files.length} files?</h3>
    <p>Reason: ${reason}</p>
    <ul>${files.map(f => `<li>${f}</li>`).join('')}</ul>
    <button id="confirm">Delete</button>
    <button id="cancel">Cancel</button>
  `;
  
  // Wait for user choice
  return new Promise(resolve => {
    modal.querySelector('#confirm').onclick = () => resolve("confirmed");
    modal.querySelector('#cancel').onclick = () => resolve("cancelled");
  });
}
```

### 3. Process Events
```typescript
let currentToolCall = { id: null, name: null, args: "" };

function processEvent(event) {
  switch (event.type) {
    case "TOOL_CALL_START":
      currentToolCall = { 
        id: event.toolCallId, 
        name: event.toolCallName, 
        args: "" 
      };
      break;
      
    case "TOOL_CALL_ARGS":
      currentToolCall.args += event.delta;
      break;
      
    case "TOOL_CALL_END":
      // Parse complete arguments
      const args = JSON.parse(currentToolCall.args);
      
      // Execute tool
      const result = await handleToolCall(currentToolCall.name, args);
      
      // Send result back
      sendToolResult(currentToolCall.id, result);
      break;
  }
}

function sendToolResult(toolCallId, result) {
  agent.sendMessage({
    role: "tool",
    content: result,
    toolCallId: toolCallId
  });
}
```

---

## Best Practices

| Practice | Why |
|----------|-----|
| **Clear names** | `confirmDeletion` not `tool1` |
| **Detailed descriptions** | AI uses this to decide when to call |
| **Precise parameters** | Include descriptions, enums, constraints |
| **Only require essentials** | Optional params = flexibility |
| **Handle errors** | Tools can fail; plan for it |
| **Good UX** | Confirmation dialogs should explain context |

---

## Parameter Schema Tips

**Good schema:**
```typescript
{
  type: "object",
  properties: {
    importance: {
      type: "string",
      enum: ["low", "medium", "high"],  // Constrain options
      description: "How urgent is this action"  // Explain purpose
    },
    deadline: {
      type: "string",
      format: "date-time",  // Specify format
      description: "When this must be done by"
    }
  },
  required: ["importance"]  // Only truly required fields
}
```

**Bad schema:**
```typescript
{
  type: "object",
  properties: {
    x: { type: "string" },  // What is x?
    y: { type: "string" }   // No descriptions
  },
  required: ["x", "y"]  // Both required when maybe they shouldn't be
}
```

---

## Summary

| Concept | What It Does |
|---------|--------------|
| **Tool** | Function AI can request to be executed |
| **name** | Identifier AI uses to call the tool |
| **description** | Tells AI when to use it |
| **parameters** | JSON Schema defining required inputs |
| **TOOL_CALL_START** | AI begins calling a tool |
| **TOOL_CALL_ARGS** | Arguments stream in pieces |
| **TOOL_CALL_END** | Call complete, ready to execute |
| **Tool Message** | Result sent back to AI |
| **Human-in-the-loop** | Tools that ask humans for input |

**The bottom line:** Tools let AI do things beyond text generation. Frontend defines what's possible. AI decides when to use them. Results flow back. The killer pattern: tools that ask humans before doing important things—AI proposes, human approves.
