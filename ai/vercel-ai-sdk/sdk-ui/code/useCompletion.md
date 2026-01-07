
# 🎯 useCompletion() — Complete 0–100% Guide (Plain English)

## 🤔 What Is It? (Super Simple)

A **React hook to generate text from a single prompt** — no chat history, no memory.

```plaintext
useChat()        →  "Hi! How are you?"  →  "I'm good!"
                    (Multi-turn conversation)

useCompletion()  →  "Complete this: The sky is..." → "blue."
                    (One-shot completion)
````

### Key Difference

* **useChat()** → Multi-turn conversation (memory + history)
* **useCompletion()** → Single prompt → single response (stateless)

---

## 🎯 When to Use It?

### ✅ USE `useCompletion()` FOR

* Auto-complete
  `"The capital of France is..." → "Paris"`

* Text generation
  `"Write a poem about the ocean"`

* Code completion
  `"function add(a, b) {" → "return a + b; }"`

* Form suggestions
  `"Subject: Payment for..." → "Invoice #123"`

* Email drafting
  `"Write a professional email about..."`

* Blog/article generation
  `"Write a 500-word article about..."`

---

### ❌ USE `useChat()` FOR

* Chatbots
* Q&A with memory
* Back-and-forth dialogue
* Tool calling / agents

---

## 📊 useChat() vs useCompletion()

| Feature    | useChat()          | useCompletion()   |
| ---------- | ------------------ | ----------------- |
| Messages   | Full message array | Single prompt     |
| Memory     | ✅ Yes              | ❌ No              |
| History    | ✅ Yes              | ❌ None            |
| Tools      | ✅ Yes              | ❌ No              |
| Use case   | Chatbots           | Text generation   |
| Backend    | `/api/chat`        | `/api/completion` |
| Complexity | Medium             | ✅ Simple          |

---

## 📍 Where to Use It?

### ✅ Frontend (Client Component)

```tsx
'use client';
import { useCompletion } from '@ai-sdk/react';

export default function Generator() {
  const {
    completion,
    complete,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
  } = useCompletion();

  return (...);
}
```

---

### ✅ Backend (Required)

```ts
// app/api/completion/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
  });

  return result.toTextStreamResponse();
}
```

> ⚠️ This returns **plain text streaming**, not chat messages.

---

## 🚀 Installation & Setup

```bash
npm install ai @ai-sdk/google @ai-sdk/react
```

### Get Gemini API Key

```bash
# https://aistudio.google.com/app/apikeys
export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"
```

---

## 🛠️ Complete Working Example (Gemini 2.0 Flash)

### ✅ Frontend

```tsx
// app/page.tsx
'use client';
import { useCompletion } from '@ai-sdk/react';

export default function TextGenerator() {
  const {
    completion,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    stop,
    error,
  } = useCompletion({
    api: '/api/completion',
  });

  return (
    <div>
      <h1>✨ Text Generator</h1>

      <form onSubmit={handleSubmit}>
        <textarea
          value={input}
          onChange={handleInputChange}
          placeholder="Write a poem about..."
          disabled={isLoading}
        />
        <button disabled={isLoading}>
          {isLoading ? 'Generating...' : 'Generate'}
        </button>
        {isLoading && <button onClick={stop}>Stop</button>}
      </form>

      {error && <p>Error: {error.message}</p>}
      {completion && <pre>{completion}</pre>}
    </div>
  );
}
```

---

### ✅ Backend

```ts
// app/api/completion/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export const maxDuration = 30;

export async function POST(req: Request) {
  const { prompt } = await req.json();

  if (!prompt) {
    return new Response('Prompt required', { status: 400 });
  }

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
    temperature: 0.7,
    maxTokens: 1000,
  });

  return result.toTextStreamResponse();
}
```

---

## 📁 Project Structure

```plaintext
app/
├── page.tsx
└── api/
    └── completion/
        └── route.ts
.env.local
```

---

## ▶️ Run

```bash
npm run dev
# Open http://localhost:3000
```

---

## 📋 Return Values Explained

```ts
const {
  completion,          // Generated text
  input,               // Prompt input
  isLoading,           // Is generating?
  error,               // Error (if any)

  complete,            // (prompt) => Promise
  handleInputChange,   // Input handler
  handleSubmit,        // Submit handler
  setCompletion,       // Manually set output
  setInput,            // Manually set input
  stop,                // Stop generation
} = useCompletion();
```

---

## 🎛️ Options

```ts
useCompletion({
  api: '/api/completion',
  initialInput: 'Write about...',
  initialCompletion: '',
  id: 'generator-1',
  headers: { Authorization: 'Bearer token' },
  body: { userId: '123' },
  credentials: 'same-origin',
  streamProtocol: 'data',
  experimental_throttle: 50,
  onFinish: (prompt, completion) => {
    console.log('Done!', completion);
  },
  onError: (err) => {
    console.error(err);
  },
});
```

---

## 💡 Practical Examples

### 1️⃣ Auto-Complete Field

```tsx
const { completion, complete } = useCompletion();

complete(`Complete this professionally: "${value}"`);
```

---

### 2️⃣ Code Generator

```tsx
await complete('Complete this function: function sum(a, b) {');
```

---

### 3️⃣ Blog Generator

```tsx
await complete('Write a 500-word article about AI in healthcare');
```

---

## 🔄 How It Works

```plaintext
useCompletion()
  ↓ POST /api/completion
{ prompt }
  ↓
streamText()
  ↓
toTextStreamResponse()
  ↓
completion updates live in UI
```

---

## ⚠️ Common Mistakes

### ❌ Calling backend APIs in frontend

```tsx
streamText(...) // ❌ Server only
```

✅ Use `useCompletion()` instead.

---

### ❌ Forgetting error handling

```tsx
const { error } = useCompletion();
```

Always handle `error`.

---

## 🎯 useCompletion() vs useChat()

| Aspect   | useCompletion     | useChat      |
| -------- | ----------------- | ------------ |
| Purpose  | One-shot text     | Conversation |
| Input    | Prompt string     | Messages     |
| Memory   | ❌ None            | ✅ Yes        |
| Backend  | `/api/completion` | `/api/chat`  |
| Best for | Generation        | Chatbots     |

---

## 🎉 Summary (Memorize This)

* **What:** One-shot text generation hook
* **Where:** Frontend hook + backend route
* **When:** No conversation needed
* **Why:** Simpler than `useChat()`
* **How:**

  ```ts
  const { complete, completion } = useCompletion();
  await complete('Your prompt');
  ```

🚀 **That’s it!** You now fully understand `useCompletion()` 🎊
