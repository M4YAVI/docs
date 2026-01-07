

---

# UIMessage: Complete Guide (0-100%)

## **What is UIMessage?**

In plain English: **UIMessage is a standardized message format specifically designed for AI chat applications running in user interfaces (UIs).** It's like a special envelope that holds conversation data in a way that's easy for your front-end to display and for your back-end to process.

Think of it this way:
- When you send a message to an AI in a chat app, the message needs to travel from your browser to the server and back
- **UIMessage** is the standard format that carries this data so both the client (browser) and server can understand it perfectly

---

## **Key Differences: UIMessage vs ModelMessage**

| Aspect | UIMessage | ModelMessage |
|--------|-----------|--------------|
| **Purpose** | Used in UI/Frontend | Used by AI models |
| **Metadata** | ✅ Includes timestamps, IDs, metadata | ❌ No metadata |
| **Complexity** | Contains UI-specific parts (text, tools, reasoning, files) | Simplified format for the model |
| **Usage** | Rendering chat UI, storing message history | Passing to AI models like GPT, Claude |
| **Conversion** | Uses `convertToModelMessages()` to convert | Already in the right format |

---

## **UIMessage Structure**

```typescript
interface UIMessage {
  id: string;                    // Unique identifier for the message
  role: 'system' | 'user' | 'assistant';  // Who sent the message
  metadata?: unknown;            // Custom metadata (timestamps, etc.)
  parts: UIMessagePart[];        // Array of content parts
}
```

### **Parts of a UIMessage**

A UIMessage can contain multiple **parts**. Each part represents different types of content:

```typescript
// 1. TEXT PART - Regular message text
{
  type: 'text',
  text: 'Hello, how can I help?',
  state?: 'streaming' | 'done'
}

// 2. TOOL PART - Tool/function calls with states
{
  type: 'tool-weatherCheck',  // Type matches tool name
  toolCallId: 'call-123',
  state: 'output-available',  // or 'input-streaming', 'input-available', 'output-error'
  input: { city: 'New York' },
  output: { temp: 72, condition: 'Sunny' }
}

// 3. REASONING PART - AI thinking process
{
  type: 'reasoning',
  text: 'I need to check the weather...'
}

// 4. FILE PART - Images, PDFs, etc.
{
  type: 'file',
  mediaType: 'image/png',
  url: 'https://example.com/image.png'
}

// 5. DATA PART - Custom data structures
{
  type: 'data-weather',
  data: { temperature: 72, humidity: 65 }
}
```

---

## **When to Use UIMessage**

### ✅ **Use UIMessage When:**

1. **Building chat interfaces** - Storing and displaying chat history in React/Next.js
2. **Streaming responses** - Sending real-time AI responses to the frontend
3. **Tool interactions** - Rendering tool calls and their results
4. **Client-server communication** - Passing messages between frontend and API
5. **Using `useChat` hook** - All messages from `useChat` are UIMessages

### ❌ **Don't Use UIMessage When:**

1. **Calling AI models directly** - Use `ModelMessage` instead
2. **Backend-to-model communication** - Convert to `ModelMessage` first
3. **Storing in databases** - UIMessages are for temporary UI state

---

## **How to Convert UIMessage to ModelMessage**

**Why convert?** AI models don't need UI metadata. They only need the essential content.

```typescript
import { convertToModelMessages, UIMessage } from 'ai';

// Frontend sends these UIMessages:
const uiMessages: UIMessage[] = [
  {
    id: 'msg-1',
    role: 'user',
    parts: [{ type: 'text', text: 'What is the weather?' }]
  },
  {
    id: 'msg-2',
    role: 'assistant',
    parts: [{ type: 'text', text: 'Let me check...' }]
  }
];

// Backend converts them:
const modelMessages = await convertToModelMessages(uiMessages);

// Now use with AI model:
const result = await streamText({
  model: openai('gpt-4'),
  messages: modelMessages  // ✅ Ready for the model
});
```

---

## **Complete Example: Chat Application with Gemini 3.0 Flash & Vercel v6**

### **1. Backend Route Handler** (`app/api/chat/route.ts`)

```typescript
import { streamText, UIMessage, convertToModelMessages } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  // 1. Extract UIMessages from request
  const { messages }: { messages: UIMessage[] } = await req.json();

  // 2. Convert UIMessages to ModelMessages (required for the model)
  const modelMessages = await convertToModelMessages(messages);

  // 3. Call the AI model with Gemini 3.0 Flash
  const result = streamText({
    model: google('gemini-3.0-flash-latest'),  // Gemini 3.0 Flash
    system: 'You are a helpful assistant.',
    messages: modelMessages  // ✅ Now in the right format
  });

  // 4. Return streamed response as UIMessage format
  return result.toUIMessageStreamResponse();
}
```

**Installation:**
```bash
pnpm add ai @ai-sdk/google @ai-sdk/react
```

### **2. Frontend Component** (`app/page.tsx`)

```typescript
'use client';

import { useChat } from '@ai-sdk/react';
import { UIMessage } from 'ai';

export default function ChatPage() {
  // useChat returns UIMessages automatically
  const { messages, sendMessage, input, setInput } = useChat({
    api: '/api/chat'
  });

  return (
    <div className="flex flex-col w-full max-w-2xl mx-auto p-4">
      {/* Display chat messages */}
      <div className="flex-1 overflow-y-auto mb-4 space-y-4">
        {messages.map((message: UIMessage) => (
          <div
            key={message.id}
            className={`p-3 rounded ${
              message.role === 'user'
                ? 'bg-blue-500 text-white ml-auto'
                : 'bg-gray-200'
            } max-w-xs`}
          >
            {/* Render message parts */}
            {message.parts.map((part, i) => (
              <div key={i}>
                {part.type === 'text' && part.text}
                {part.type === 'reasoning' && (
                  <div className="text-sm italic">💭 {part.text}</div>
                )}
              </div>
            ))}
          </div>
        ))}
      </div>

      {/* Input form */}
      <form
        onSubmit={(e) => {
          e.preventDefault();
          sendMessage({ text: input });
          setInput('');
        }}
        className="flex gap-2"
      >
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Type a message..."
          className="flex-1 p-2 border rounded"
        />
        <button
          type="submit"
          className="px-4 py-2 bg-blue-600 text-white rounded"
        >
          Send
        </button>
      </form>
    </div>
  );
}
```

---

## **Advanced: Using UIMessage with Tools**

```typescript
import { streamText, UIMessage, convertToModelMessages } from 'ai';
import { google } from '@ai-sdk/google';
import { tool } from 'ai';
import { z } from 'zod';

// Define tools
const tools = {
  getWeather: tool({
    description: 'Get weather for a city',
    parameters: z.object({ city: z.string() }),
    execute: async ({ city }) => {
      return { temp: 72, condition: 'Sunny', city };
    }
  })
};

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  // Convert UIMessages to ModelMessages
  const modelMessages = await convertToModelMessages(messages, {
    tools  // Pass tools so they're properly handled
  });

  const result = streamText({
    model: google('gemini-3.0-flash-latest'),
    messages: modelMessages,
    tools,  // ✅ Register tools
    toolChoice: 'auto'
  });

  return result.toUIMessageStreamResponse();
}
```

### **Frontend with Tool Rendering:**

```typescript
'use client';

import { useChat } from '@ai-sdk/react';
import { UIMessage } from 'ai';

export default function ChatWithTools() {
  const { messages, sendMessage, input, setInput } = useChat();

  return (
    <div className="space-y-4">
      {messages.map((message: UIMessage) => (
        <div key={message.id} className="p-4 border rounded">
          <strong>{message.role}:</strong>
          {message.parts.map((part, i) => {
            // Text content
            if (part.type === 'text') {
              return <p key={i}>{part.text}</p>;
            }

            // Tool calls (e.g., getWeather)
            if (part.type === 'tool-getWeather') {
              return (
                <div key={i} className="mt-2 p-2 bg-blue-50 rounded">
                  <p>🔍 Checking weather for: {part.input.city}</p>
                  {part.state === 'output-available' && (
                    <p className="font-bold">
                      🌤️ {part.output.temp}°F - {part.output.condition}
                    </p>
                  )}
                </div>
              );
            }

            return null;
          })}
        </div>
      ))}

      <form
        onSubmit={(e) => {
          e.preventDefault();
          sendMessage({ text: input });
          setInput('');
        }}
      >
        <input value={input} onChange={(e) => setInput(e.target.value)} />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

---

## **Key Takeaways**

| Point | Explanation |
|-------|-------------|
| **What** | UIMessage is a message format for chat UIs with UI-specific metadata |
| **Where** | Used in React components, `useChat` hook, and client-server communication |
| **When** | Use for UI rendering; convert to ModelMessage before calling AI models |
| **How** | Use `convertToModelMessages()` to transform for backend use |
| **Why** | Keeps UI data separate from model input, making code cleaner and more efficient |

---

## **Checklist for Using UIMessage in Your App**

- ✅ Frontend uses `useChat()` hook → returns `UIMessage[]`
- ✅ Frontend sends `UIMessage[]` to backend API
- ✅ Backend receives `UIMessage[]`
- ✅ Backend converts: `await convertToModelMessages(messages)`
- ✅ Backend passes converted messages to `streamText()` or `generateText()`
- ✅ Backend returns `result.toUIMessageStreamResponse()`
- ✅ Frontend receives and displays messages using `message.parts` array
- ✅ All message types (text, tools, reasoning, files) render correctly

---

That's everything you need to know about UIMessage! It's simply a **message wrapper designed for chat UIs** that carries conversation data with metadata, and must be converted before passing to AI models.
