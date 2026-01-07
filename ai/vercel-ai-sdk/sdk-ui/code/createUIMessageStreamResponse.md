
# 🎯 createUIMessageStreamResponse() — Complete 0–100% Guide (Plain English)

## 🤔 What Is It? (Super Simple)

**`createUIMessageStreamResponse()` converts a UI message stream into an HTTP response that the frontend can read.**

```plaintext
Your Stream (ReadableStream)
        ↓
createUIMessageStreamResponse()
        ↓
HTTP Response (SSE + headers)
        ↓
Network (Server-Sent Events)
        ↓
useChat() on frontend
        ↓
UI updates in real-time ✨
````

**In one sentence:**
It packages your AI stream into a proper HTTP response that works with `useChat()`.

---

## 🎯 When Do You Need It?

### ✅ USE `createUIMessageStreamResponse()` When

* 🔄 Converting `createUIMessageStream()` → HTTP response
* 📤 Using non-Next.js backends (Express, Fastify, Hono)
* 🎨 Building custom streaming workflows
* 🛠️ Managing streams manually
* 📱 Supporting any HTTP framework

### ❌ DON’T Use It When

* Using `streamText().toUIMessageStreamResponse()` (already included)
* Generating plain text only → `toTextStreamResponse()`
* No streaming required

---

## 📊 Response Type Comparison

| Response                          | Use Case       | Format     | Frontend      |
| --------------------------------- | -------------- | ---------- | ------------- |
| `toUIMessageStreamResponse()`     | Standard chat  | SSE + JSON | `useChat()` ✅ |
| `createUIMessageStreamResponse()` | Custom streams | SSE + JSON | `useChat()` ✅ |
| `toTextStreamResponse()`          | Plain text     | Text       | Custom        |
| `toSseResponse()`                 | Low-level SSE  | SSE        | Custom        |

---

## 📍 Where to Use It

### ✅ Next.js

```ts
export async function POST() {
  const stream = createUIMessageStream({ ... });
  return createUIMessageStreamResponse({ stream });
}
```

### ✅ Express

```ts
app.post('/chat', (req, res) => {
  const stream = createUIMessageStream({ ... });
  const response = createUIMessageStreamResponse({ stream });

  res.status(response.status).set(response.headers);
  res.send(response.body);
});
```

### ✅ Fastify

```ts
reply.send(createUIMessageStreamResponse({ stream }));
```

### ✅ Hono

```ts
return createUIMessageStreamResponse({ stream });
```

---

## 🚀 Working Examples

### Option 1 — Next.js (Simplest)

```ts
const result = await streamText({ model, messages });
return result.toUIMessageStreamResponse();
```

---

### Option 2 — Custom Stream (Advanced)

```ts
const stream = createUIMessageStream({
  execute: async ({ writer }) => {
    writer.write({ type: 'start' });

    writer.write({
      type: 'data-metadata',
      data: { model: 'gemini-2.0-flash' },
    });

    const result = await streamText({ model, messages });
    writer.merge(result.toUIMessageStream({ sendStart: false }));

    writer.write({ type: 'finish' });
  },
});

return createUIMessageStreamResponse({ stream });
```

---

### Option 3 — Express Backend

```ts
const response = createUIMessageStreamResponse({ stream });
res.status(response.status).set(response.headers);
res.send(response.body);
```

---

## 🎛️ API Reference

```ts
createUIMessageStreamResponse({
  stream,              // REQUIRED
  status: 200,         // optional
  statusText: 'OK',    // optional
  headers: {},         // optional
  consumeSseStream,    // advanced
});
```

---

## 📦 Default Headers (Auto Added)

```ts
{
  'content-type': 'text/event-stream',
  'cache-control': 'no-cache',
  'connection': 'keep-alive',
  'x-vercel-ai-ui-message-stream': 'v1',
  'x-accel-buffering': 'no',
}
```

---

## 📋 What Frontend Receives

### SSE Payload

```plaintext
data: {"type":"start","messageId":"msg-123"}
data: {"type":"text-delta","delta":"Hello"}
data: {"type":"finish","finishReason":"stop"}
data: [DONE]
```

### Parsed by `useChat()`

```ts
{
  id: 'msg-123',
  role: 'assistant',
  parts: [{ type: 'text', text: 'Hello' }]
}
```

---

## 🔄 Flow Diagram

```plaintext
createUIMessageStream()
   ↓ ReadableStream
createUIMessageStreamResponse()
   ↓ HTTP Response (SSE)
Network
   ↓
useChat()
   ↓
Live UI updates
```

---

## 💡 Real-World Patterns

### 1️⃣ Add Metadata

```ts
writer.write({
  type: 'data-context',
  data: { userId, timestamp: Date.now() },
});
```

---

### 2️⃣ Multiple Data Chunks

```ts
writer.write({ type: 'data-search', data: results });
writer.merge(aiStream);
writer.write({ type: 'data-analytics', data: metrics });
```

---

### 3️⃣ Error Handling

```ts
writer.write({
  type: 'error',
  errorText: 'Generation failed',
});
```

---

## ⚠️ Common Mistakes

### ❌ Using Text Stream for Chat

```ts
return result.toTextStreamResponse(); // ❌
```

```ts
return result.toUIMessageStreamResponse(); // ✅
```

---

### ❌ Express: Sending JSON Instead of Body

```ts
res.json(createUIMessageStreamResponse({ stream })); // ❌
```

```ts
const r = createUIMessageStreamResponse({ stream });
res.status(r.status).set(r.headers).send(r.body); // ✅
```

---

## 🎯 When to Use Which

```ts
// Plain text
toTextStreamResponse();

// Simple chat (Next.js)
toUIMessageStreamResponse();

// Custom streaming
createUIMessageStreamResponse({ stream });

// Low-level SSE
toSseResponse();
```

---

## 🧪 Quick Test

```ts
const res = await fetch('/api/chat');
console.log(res.headers.get('content-type')); // text/event-stream
```

---

## 🚀 Quick Reference

| Task           | Code                              |
| -------------- | --------------------------------- |
| Simple chat    | `toUIMessageStreamResponse()`     |
| Custom stream  | `createUIMessageStreamResponse()` |
| Custom headers | `{ headers: {...} }`              |
| Express send   | `res.send(response.body)`         |

---

## 🎉 Final Summary

* **What:** Stream → HTTP response
* **Where:** Backend (any framework)
* **When:** Custom or non-Next.js streaming
* **Why:** Makes `useChat()` work
* **How:** `createUIMessageStreamResponse({ stream })`
* **Bonus:** Adds SSE headers automatically

🚀 You now fully understand **`createUIMessageStreamResponse()`**.
