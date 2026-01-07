
# 🎯 createUIMessageStream() — Complete 0–100% Guide (Plain English)

## 🤔 What Is It? (Super Simple)

**`createUIMessageStream()` lets you manually control what streams to the frontend.**

Instead of “AI does everything automatically,” *you* decide:

- when text starts
- what text chunks are sent
- when tools appear
- what custom data is streamed
- how multiple streams are merged

```plaintext
Regular Chat:
Frontend → Backend → AI → Stream → Frontend

createUIMessageStream():
Frontend → Backend (YOU control everything)
           ├─ Write text chunks
           ├─ Write tool calls
           ├─ Write custom data
           ├─ Merge AI streams
           └─ Stream to frontend in real-time
````

---

## 🎮 What It Gives You

* 🎮 **Manual streaming control**
* 📦 **Send custom data (JSON, metadata, UI state)**
* 🔀 **Merge multiple streams (AI + search + tools)**
* 🎨 **Inject data before / after AI response**
* 🛠️ **Works outside Next.js (Express, Hono, Fastify)**

---

## 🎯 When Do You Need It?

### ❌ DON’T Use If

* Simple chatbot
* Just generating text
* No custom UI data
* `useChat()` + `streamText()` is enough

---

### ✅ USE If You Need

* 🎨 Send data *before* AI responds
* 🔄 Merge multiple AI/tool streams
* 📦 Attach metadata or context
* 🎯 Multi-step workflows
* 🛠️ Non-Next.js backend

---

## 📋 What Can You Stream?

### 🧱 Chunk Types

```plaintext
TEXT
├─ text-start
├─ text-delta
└─ text-end

TOOLS
├─ tool-input-start
├─ tool-input-delta
├─ tool-input-available
├─ tool-output-available
├─ tool-output-error
└─ tool-approval-request

CUSTOM DATA
├─ data-*
└─ message-metadata

LIFECYCLE
├─ start
├─ finish
├─ error
└─ abort

REASONING
├─ reasoning-start
├─ reasoning-delta
└─ reasoning-end

SOURCES
├─ source-url
└─ source-document
```

---

## 📍 Where to Use It

### ✅ Backend (Next.js Route)

```ts
import {
  createUIMessageStream,
  createUIMessageStreamResponse,
} from 'ai';

export async function POST() {
  const stream = createUIMessageStream({
    execute: async ({ writer }) => {
      writer.write({ type: 'start' });
      writer.write({ type: 'text-delta', delta: 'Hello!' });
      writer.write({ type: 'finish' });
    },
  });

  return createUIMessageStreamResponse({ stream });
}
```

---

### ✅ Non-Next.js (Express / Hono)

```ts
app.post('/stream', (req, res) => {
  const stream = createUIMessageStream({
    execute: ({ writer }) => {
      writer.write({ type: 'start' });
      writer.write({
        type: 'text-delta',
        delta: 'Hello from Express!',
      });
      writer.write({ type: 'finish' });
    },
  });

  pipeUIMessageStreamToResponse({ stream, response: res });
});
```

---

## 🚀 Complete Working Example (Gemini 2.0 Flash)

### Backend

```ts
import {
  createUIMessageStream,
  createUIMessageStreamResponse,
  streamText,
} from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const stream = createUIMessageStream({
    execute: async ({ writer }) => {
      writer.write({
        type: 'start',
        messageId: `msg-${Date.now()}`,
      });

      writer.write({
        type: 'data-search-results',
        data: {
          query: prompt,
          sources: ['wikipedia.org', 'github.com'],
        },
      });

      const result = streamText({
        model: google('gemini-2.0-flash'),
        prompt,
      });

      writer.merge(
        result.toUIMessageStream({ sendStart: false })
      );

      writer.write({
        type: 'data-metadata',
        data: { model: 'gemini-2.0-flash' },
      });

      writer.write({ type: 'finish' });
    },
  });

  return createUIMessageStreamResponse({ stream });
}
```

---

### Frontend

```tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const { messages, input, handleSubmit, handleInputChange } =
    useChat({ api: '/api/chat' });

  return (
    <div>
      {messages.map(m => (
        <div key={m.id}>
          {m.parts?.map((p, i) => {
            if (p.type === 'text') return <p key={i}>{p.text}</p>;
            if (p.type === 'data-search-results')
              return <pre key={i}>{JSON.stringify(p.data, null, 2)}</pre>;
            return null;
          })}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
      </form>
    </div>
  );
}
```

---

## 🧠 API Reference

```ts
createUIMessageStream({
  execute,          // REQUIRED
  onError,          // optional
  originalMessages, // optional
  onFinish,         // optional
  generateId,       // optional
});
```

---

## ✍️ Writer Methods

### writer.write()

```ts
writer.write({
  type: 'text-delta',
  delta: 'Hello ',
});
```

```ts
writer.write({
  type: 'data-custom',
  data: { foo: 'bar' },
});
```

---

### writer.merge()

```ts
const aiStream = streamText({
  model: google('gemini-2.0-flash'),
  prompt: 'Hello',
}).toUIMessageStream();

writer.merge(aiStream);
```

---

## 💡 Real-World Patterns

### 1️⃣ AI + Search

```ts
writer.write({ type: 'data-search', data: results });
writer.merge(aiStream);
```

---

### 2️⃣ Multi-Step Workflow

```ts
writer.write({ type: 'data-step', data: { step: 1 } });
writer.merge(aiStream);
writer.write({ type: 'data-step', data: { step: 2 } });
```

---

### 3️⃣ UI State Streaming

```ts
writer.write({
  type: 'data-ui-state',
  data: { loading: true },
});
```

---

## 📝 Manual Text Streaming

```ts
const text = 'Hello world';

for (const char of text) {
  writer.write({ type: 'text-delta', delta: char });
  await delay(50);
}
```

---

## 🛠 Tool Calling Example

```ts
writer.write({
  type: 'tool-input-start',
  toolCallId: 'call-1',
  toolName: 'getWeather',
});
```

```ts
writer.write({
  type: 'tool-output-available',
  toolCallId: 'call-1',
  output: { temp: 72 },
});
```

---

## ⚠️ Common Mistakes

### ❌ Duplicate `start`

```ts
writer.merge(aiStream); // ❌ duplicates start
```

```ts
writer.merge(aiStream, { sendStart: false }); // ✅
```

---

### ❌ Finishing too early

```ts
writer.merge(stream);
writer.write({ type: 'finish' }); // ❌ too early
```

---

## 🎉 Summary (Memorize This)

* **What:** Manual streaming control
* **Where:** Backend only
* **When:** Complex workflows
* **Why:** Full real-time control
* **How:**

```ts
const stream = createUIMessageStream({
  execute: ({ writer }) => {
    writer.write(...);
    writer.merge(...);
  },
});
```

---

## 🚀 Quick Reference

| Task             | Code                    |
| ---------------- | ----------------------- |
| Send text        | `text-delta`            |
| Send custom data | `data-*`                |
| Merge AI         | `writer.merge()`        |
| Start            | `start`                 |
| Finish           | `finish`                |
| Error            | `error`                 |
| Tool start       | `tool-input-start`      |
| Tool output      | `tool-output-available` |

---

💥 **That’s it.**
You now fully understand **`createUIMessageStream()`** — the power tool for advanced streaming AI UIs 🚀
