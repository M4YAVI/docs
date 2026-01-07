
# 🎯 pipeUIMessageStreamToResponse() — Complete 0–100% Guide (Plain English)

## 🤔 What Is It? (Super Simple)

**`pipeUIMessageStreamToResponse()` pipes a UI message stream directly into an HTTP response.**  
No intermediate `Response` object. No extra wrapping.

```plaintext
Regular Way:
Stream → createUIMessageStreamResponse({ stream })
      → Response object → Send to client

Fast Way (pipeUIMessageStreamToResponse):
Stream → Directly written to HTTP response
      ✨ Fewer steps, more direct
````

**In one sentence:**
It’s a shortcut that writes a UI message stream straight into a Node.js/Express response.

---

## 📊 Comparison: Three Ways to Send Streams

| Method                            | Code                                                 | Best For          | Overhead     |
| --------------------------------- | ---------------------------------------------------- | ----------------- | ------------ |
| `toUIMessageStreamResponse()`     | `return streamText(...).toUIMessageStreamResponse()` | Next.js           | Low ✅        |
| `createUIMessageStreamResponse()` | `createUIMessageStreamResponse({ stream })`          | Custom streams    | Medium       |
| `pipeUIMessageStreamToResponse()` | `result.pipeUIMessageStreamToResponse(res)`          | Express / Node.js | **Lowest ✅** |

---

## 🎯 When Do You Need It?

### ✅ USE `pipeUIMessageStreamToResponse()` When

* 🚀 You’re using **Express.js**
* 🔌 You’re using **raw Node.js HTTP**
* ⚡ You want **maximum performance**
* 📤 You already have a `ServerResponse`
* 🎯 You don’t need to return a `Response` object

### ❌ DON’T Use It When

* Using **Next.js App Router**
* You need a returned `Response`
* You’re building a **custom UI message stream**
* You’re on the Fetch API (no `ServerResponse`)

---

## 🔄 Flow Comparison

### Regular (Return a Response)

```plaintext
streamText()
  ↓
toUIMessageStreamResponse()
  ↓ Response
return Response
```

### Fast (Pipe Directly)

```plaintext
streamText()
  ↓
pipeUIMessageStreamToResponse(res)
  ↓
Response already sent (void)
```

---

## 📍 Where to Use It

### ✅ Express.js

```ts
app.post('/chat', async (req, res) => {
  const result = await streamText(...);
  result.pipeUIMessageStreamToResponse(res);
});
```

### ✅ Raw Node.js

```ts
createServer((req, res) => {
  const result = await streamText(...);
  result.pipeUIMessageStreamToResponse(res);
});
```

### ❌ Next.js (Don’t Do This)

```ts
// ❌ No ServerResponse in Next.js
result.pipeUIMessageStreamToResponse(res);
```

```ts
// ✅ Correct for Next.js
return result.toUIMessageStreamResponse();
```

---

## 🚀 Complete Working Example (Express + Gemini)

### Simple Streaming (Most Common)

```ts
app.post('/api/chat', async (req, res) => {
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: req.body.messages,
  });

  result.pipeUIMessageStreamToResponse(res);
});
```

---

### Custom Stream + Pipe

```ts
const stream = createUIMessageStream({
  execute: async ({ writer }) => {
    writer.write({ type: 'start' });

    writer.write({
      type: 'data-info',
      data: { model: 'gemini-2.0-flash', ts: Date.now() },
    });

    const result = await streamText({ model, prompt });
    writer.merge(result.toUIMessageStream({ sendStart: false }));

    writer.write({ type: 'finish' });
  },
});

pipeUIMessageStreamToResponse({ response: res, stream });
```

---

## 🎛️ API Reference

### Method on `streamText()` result

```ts
result.pipeUIMessageStreamToResponse(res);

result.pipeUIMessageStreamToResponse(res, {
  status: 200,
  headers: {
    'X-Custom': 'value',
  },
});
```

### Function for Custom Streams

```ts
pipeUIMessageStreamToResponse({
  response: res,        // REQUIRED (ServerResponse)
  stream,               // REQUIRED (UI message stream)
  status: 200,
  headers: {},
  consumeSseStream,     // advanced
});
```

---

## 💡 Real-World Examples

### 1️⃣ Express Chat Server

```ts
app.post('/chat', async (req, res) => {
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: req.body.messages,
  });

  result.pipeUIMessageStreamToResponse(res);
});
```

---

### 2️⃣ With Validation + Errors

```ts
app.post('/chat', async (req, res) => {
  if (!req.body.messages) {
    res.status(400).json({ error: 'Messages required' });
    return;
  }

  const result = await streamText({ model, messages });
  result.pipeUIMessageStreamToResponse(res);
});
```

---

## ⚠️ Common Mistakes

### ❌ Returning the Pipe Call

```ts
return result.pipeUIMessageStreamToResponse(res); // ❌ returns void
```

```ts
result.pipeUIMessageStreamToResponse(res); // ✅ correct
```

---

### ❌ Writing After Piping

```ts
result.pipeUIMessageStreamToResponse(res);
res.send('done'); // ❌ too late
```

---

### ❌ Not Awaiting `streamText()`

```ts
const result = streamText(...); // ❌ Promise
result.pipeUIMessageStreamToResponse(res);
```

```ts
const result = await streamText(...); // ✅
```

---

## 🔄 What Happens Internally

```plaintext
UIMessageStream
  ↓ JSON → SSE transform
  ↓ TextEncoderStream
  ↓ res.write() loop
  ↓ res.end()
```

* Sets headers
* Handles backpressure
* Streams chunks as they arrive
* Closes response automatically

---

## 📊 Performance Notes

**Why it’s faster:**

* No `Response` object creation
* No extra buffering
* Direct write to socket

Typical gain: **~5–10% faster streaming** under load.

---

## 🎯 When to Use Each

```ts
// Next.js
return result.toUIMessageStreamResponse();

// Express + streamText()
result.pipeUIMessageStreamToResponse(res);

// Express + custom stream
pipeUIMessageStreamToResponse({ response: res, stream });

// Need a Response object
createUIMessageStreamResponse({ stream });
```

---

## 🧪 Testing

```ts
const res = await fetch('/api/chat');
console.log(res.headers.get('content-type')); // text/event-stream
```

---

## 🎉 Final Summary

* **What:** Pipes a UI message stream directly to HTTP response
* **Where:** Express / Node.js
* **When:** You want the fastest streaming path
* **Why:** No intermediate objects
* **How:** `result.pipeUIMessageStreamToResponse(res)`
* **Returns:** `void`
* **Best For:** Express chat servers

---

## 🚀 Quick Reference

| Task                  | Code                                                       |
| --------------------- | ---------------------------------------------------------- |
| Simple Express stream | `result.pipeUIMessageStreamToResponse(res)`                |
| Custom stream         | `pipeUIMessageStreamToResponse({ response: res, stream })` |
| Custom headers        | `pipeUIMessageStreamToResponse(res, { headers })`          |
| Next.js               | `toUIMessageStreamResponse()`                              |

---

🎊 **You’re now an expert in `pipeUIMessageStreamToResponse()`**
Fast, direct, and perfect for Express-based AI streaming 🚀
