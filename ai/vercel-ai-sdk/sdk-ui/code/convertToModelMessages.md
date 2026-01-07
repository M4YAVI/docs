# 🎯 convertToModelMessages() — Complete 0–100% Guide (Plain English)

## 🤔 What Is It? (Super Simple)

A **translator between two message formats**:

- **UIMessage** ← what `useChat()` uses (frontend)
- **ModelMessage** ← what AI models (Gemini, GPT, etc.) need (backend)

```plaintext
Frontend (useChat)         Backend (AI Model)
   UIMessage      ──convert──>      ModelMessage
  (chat format)                   (AI format)
````

---

## ❌ The Problem Without It

```plaintext
useChat gives:
{ id, role, parts: [{ type: 'text', text: "Hi" }] }

Gemini expects:
{ role: 'user', content: [{ type: 'text', text: "Hi" }] }

↑ Different structure → CRASH 💥
```

---

## ✅ With convertToModelMessages()

```plaintext
useChat gives:
{ id, role, parts: [{ type: 'text', text: "Hi" }] }

↓ convertToModelMessages()

Gemini gets:
{ role: 'user', content: [{ type: 'text', text: "Hi" }] }

✅ Perfect match!
```

---

## 📍 Where & When to Use It (ALWAYS Backend)

### ✅ USE HERE (Backend API Route)

```ts
// /app/api/chat/route.ts
import { convertToModelMessages, streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { messages } = await req.json(); // From useChat()

  // 🔁 CONVERT
  const modelMessages = await convertToModelMessages(messages);

  // 🤖 SEND TO AI
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: modelMessages,
  });

  return result.toUIMessageStreamResponse();
}
```

---

### ❌ NEVER HERE (Frontend)

```tsx
// ❌ DON'T DO THIS
import { useChat } from '@ai-sdk/react';

function Chat() {
  const { messages } = useChat();

  const modelMsgs = convertToModelMessages(messages); // ❌ Server-only
}
```

---

## 🔄 What It Converts (Message Types)

### 1️⃣ System Messages (Instructions)

```plaintext
BEFORE (UIMessage):
{
  role: 'system',
  parts: [{ type: 'text', text: 'You are helpful.' }]
}

AFTER (ModelMessage):
{
  role: 'system',
  content: 'You are helpful.'
}
```

---

### 2️⃣ User Messages (Text Input)

```plaintext
BEFORE:
{
  role: 'user',
  parts: [{ type: 'text', text: 'What is AI?' }]
}

AFTER:
{
  role: 'user',
  content: [{ type: 'text', text: 'What is AI?' }]
}
```

---

### 3️⃣ Assistant Messages (Text + Tools)

```plaintext
BEFORE:
{
  role: 'assistant',
  parts: [
    { type: 'text', text: 'AI is...' },
    {
      type: 'tool-call',
      toolCallId: 'call_123',
      toolName: 'search',
      input: { query: 'AI' }
    }
  ]
}

AFTER:
{
  role: 'assistant',
  content: [
    { type: 'text', text: 'AI is...' },
    {
      type: 'tool-call',
      toolCallId: 'call_123',
      toolName: 'search',
      input: { query: 'AI' }
    }
  ]
}
```

---

### 4️⃣ File / Image Messages

```plaintext
BEFORE:
{
  role: 'user',
  parts: [
    {
      type: 'file',
      mediaType: 'image/png',
      url: 'https://example.com/image.png'
    }
  ]
}

AFTER:
{
  role: 'user',
  content: [
    {
      type: 'file',
      mediaType: 'image/png',
      data: 'https://example.com/image.png'
    }
  ]
}
```

---

## 🛠️ Complete Working Example (Gemini Flash)

### Step 1: Frontend (`useChat`)

```tsx
// app/page.tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading
  } = useChat({ api: '/api/chat' });

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.parts?.[0]?.text}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button disabled={isLoading}>Send</button>
      </form>
    </div>
  );
}
```

---

### Step 2: Backend (Convert + Stream)

```ts
// app/api/chat/route.ts
import { convertToModelMessages, streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const modelMessages = await convertToModelMessages(messages);

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    system: 'You are a helpful assistant.',
    messages: modelMessages,
  });

  return result.toUIMessageStreamResponse();
}
```

---

## 📦 Installation

```bash
npm install ai @ai-sdk/google @ai-sdk/react
```

---

## 🔑 Gemini API Key

```bash
# https://aistudio.google.com/app/apikeys
export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"
```

---

## ▶️ Run

```bash
npm run dev
# Open http://localhost:3000
```

---

## 🎛️ Advanced Options

```ts
const modelMessages = await convertToModelMessages(messages, {
  tools: {
    weather: tool({
      execute: async ({ city }) => ({ temp: 72 })
    })
  },
  ignoreIncompleteToolCalls: true,
  convertDataPart: (part) => ({
    type: 'text',
    text: JSON.stringify(part.data)
  })
});
```

---

## 🔍 Under the Hood (Simplified)

```ts
export async function convertToModelMessages(messages) {
  return messages.map(msg => ({
    role: msg.role,
    content: msg.parts.map(part => {
      if (part.type === 'text') return { type: 'text', text: part.text };
      if (part.type === 'file') {
        return { type: 'file', mediaType: part.mediaType, data: part.url };
      }
      if (part.type === 'tool-call') {
        return {
          type: 'tool-call',
          toolName: part.toolName,
          toolCallId: part.toolCallId,
          input: part.input
        };
      }
    })
  }));
}
```

---

## ⚠️ Common Mistakes

### ❌ Passing UIMessage directly

```ts
messages: messages // ❌
```

✅ Fix:

```ts
messages: await convertToModelMessages(messages)
```

---

### ❌ Using it on the frontend

```tsx
convertToModelMessages(messages); // ❌
```

✅ Fix: Backend only.

---

### ❌ Forgetting `await`

```ts
const msgs = convertToModelMessages(messages); // ❌ Promise
```

✅ Fix:

```ts
const msgs = await convertToModelMessages(messages);
```

---

## 📊 Full Flow

```plaintext
useChat() → UIMessage[]
   ↓ POST /api/chat
convertToModelMessages()
   ↓
streamText(gemini)
   ↓
toUIMessageStreamResponse()
   ↓
useChat() (live streaming)
```

---

## 📋 Quick Reference

| Item                   | Where              | When              |
| ---------------------- | ------------------ | ----------------- |
| convertToModelMessages | Backend API        | Every request     |
| Input                  | useChat            | Frontend          |
| Output                 | streamText         | Backend           |
| Async                  | Yes                | Always `await`    |
| Handles                | Text, files, tools | All message types |

---

## 🎉 Summary (Memorize This)

* **What:** UIMessage → ModelMessage
* **Where:** Backend API route only
* **When:** Before sending messages to AI
* **Why:** Frontend & models speak different formats
* **How:** `await convertToModelMessages(messages)`

