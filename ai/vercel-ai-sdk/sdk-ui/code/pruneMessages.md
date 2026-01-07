
# 🎯 pruneMessages() — Complete 0–100% Guide (Plain English)

## 🤔 What Is It? (Super Simple)

**`pruneMessages()` cleans up chat history before sending it to the AI**  
→ fewer tokens, lower cost, faster responses, no internal junk.

```plaintext
BEFORE (Full chat — expensive & slow):
├─ User: "Hi, how are you?"
├─ Assistant: [Internal reasoning: "Let me think..."]
├─ Assistant: "I'm good! Let me check the weather"
├─ Assistant: [Tool call: GET /weather]
├─ Assistant: [Tool result: sunny 72°F]
├─ Assistant: "The weather is sunny!"
└─ User: "Great, thanks!"
   ↑ 10 messages sent to AI

AFTER (Pruned):
├─ User: "Hi, how are you?"
├─ Assistant: "I'm good! Let me check the weather"
├─ Assistant: "The weather is sunny!"
└─ User: "Great, thanks!"
   ↑ 4 messages → Faster & cheaper!
````

---

## ❌ Problems It Solves

* ❌ **Too much context** → higher token cost
* ❌ **Internal reasoning leaks** → confusing & unsafe
* ❌ **Old tool calls** → irrelevant clutter
* ❌ **Token waste** → $$$ burned

---

## 💡 Why You Need It

### 💰 The Money Problem

```plaintext
Full chat (100 msgs)   = $1.00 / request
Pruned chat (20 msgs) = $0.20 / request

1000 requests/day:
Without pruning: $1000/day
With pruning:    $200/day
Savings:        $800/day 💸
```

---

### 🚀 The Speed Problem

```plaintext
Full chat:
- DB load: 200ms
- Send to AI: 500ms
- Processing: 1000ms
TOTAL: 1700ms ❌

Pruned chat:
- DB load: 50ms
- Send to AI: 100ms
- Processing: 900ms
TOTAL: 1050ms ✅ (~40% faster)
```

---

## 📍 Where & When to Use It

### ✅ USE IT IN THE BACKEND (Before sending to AI)

```ts
// app/api/chat/route.ts
import { pruneMessages, streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  let { messages } = await req.json();

  messages = pruneMessages({
    messages,
    reasoning: 'all',
    toolCalls: 'before-last-message',
    emptyMessages: 'remove',
  });

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages,
  });

  return result.toTextStreamResponse();
}
```

---

### ❌ NEVER prune on the frontend

```tsx
// ❌ WRONG
const pruned = pruneMessages(messages);
// Users should see full chat history!
```

---

## 🎛️ What Can It Remove?

---

### 1️⃣ Reasoning (AI Internal Thoughts)

**Problem**

```plaintext
Assistant:
- "I need to check the weather..."
- "Let me search..."
- "The weather is sunny"
```

**Solution**

```ts
pruneMessages({
  messages,
  reasoning: 'all',
});
```

**Result**

```plaintext
"The weather is sunny"
```

**Options**

* `'none'` (default)
* `'all'`
* `'before-last-message'`

---

### 2️⃣ Tool Calls (Function / API Calls)

**Problem**

```plaintext
Old weather searches polluting history
```

**Solution**

```ts
pruneMessages({
  messages,
  toolCalls: 'before-last-message',
});
```

**Options**

* `'none'`
* `'all'`
* `'before-last-message'`
* `'before-last-5-messages'`
* Custom tool filters:

```ts
toolCalls: [{ type: 'all', tools: ['search'] }]
```

---

### 3️⃣ Empty Messages

**Problem**

```plaintext
Messages become empty after pruning
```

**Solution**

```ts
pruneMessages({
  messages,
  emptyMessages: 'remove',
});
```

**Options**

* `'remove'` ✅ (default)
* `'keep'` ❌

---

## 🚀 Complete Working Example (Gemini Flash)

```ts
// app/api/chat/route.ts
import {
  convertToModelMessages,
  pruneMessages,
  streamText,
} from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  let { messages } = await req.json();

  let modelMessages = await convertToModelMessages(messages);

  modelMessages = pruneMessages({
    messages: modelMessages,
    reasoning: 'all',
    toolCalls: 'before-last-2-messages',
    emptyMessages: 'remove',
  });

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: modelMessages,
  });

  return result.toTextStreamResponse();
}
```

---

## 🔄 How It Works

```plaintext
Frontend (useChat)
│ Full history (with reasoning & tools)
↓ POST /api/chat
Backend
│ pruneMessages()
│ ├─ removes reasoning
│ ├─ removes old tools
│ └─ removes empty messages
↓
Gemini (clean context)
↓
Frontend (new message only)
```

---

## 📋 All Options Reference

```ts
pruneMessages({
  messages,

  reasoning: 'none' | 'all' | 'before-last-message',

  toolCalls:
    | 'none'
    | 'all'
    | 'before-last-message'
    | 'before-last-3-messages'
    | [{ type: 'all', tools: ['weather'] }],

  emptyMessages: 'remove' | 'keep',
});
```

---

## 🎯 Real-World Scenarios

### Long Conversations

```ts
pruneMessages({
  reasoning: 'all',
  toolCalls: 'before-last-message',
});
```

### Budget-Conscious App

```ts
pruneMessages({
  reasoning: 'all',
  toolCalls: 'all',
});
```

### Deep Reasoning Models

```ts
pruneMessages({
  reasoning: 'before-last-message',
});
```

---

## 💡 Pro Tips

✅ Keep **recent tools**
❌ Remove **old reasoning**
❌ Never prune **user messages**
✅ Prune **only before sending to AI**

---

## 🔕 When NOT to Prune

* Saving full history to DB
* Debugging tools
* Auditing model behavior
* Showing users raw reasoning

---

## 📊 Token Savings Example

```plaintext
Before: 1000 tokens → $0.01
After:  400 tokens  → $0.004
Savings/request: $0.006
10k requests/year = $60 saved 🎉
```

---

## 🎉 Summary (Memorize This)

* **What:** Cleans chat history
* **Where:** Backend only
* **When:** Before sending to AI
* **Why:** Save money + speed
* **How:**

```ts
pruneMessages({ messages, reasoning, toolCalls, emptyMessages });
```

---

## 🚀 Quick Reference

| Scenario     | reasoning           | toolCalls              | emptyMessages |
| ------------ | ------------------- | ---------------------- | ------------- |
| Long chat    | all                 | before-last-message    | remove        |
| Budget tight | all                 | all                    | remove        |
| Keep context | before-last-message | before-last-3-messages | remove        |
| Safe default | none                | none                   | remove        |

---

💥 **That’s it.**
You now fully understand **`pruneMessages()`** — use it to **cut costs, boost speed, and keep AI inputs clean** 🚀
