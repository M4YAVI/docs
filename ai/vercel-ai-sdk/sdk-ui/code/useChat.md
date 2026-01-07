
# 🎯 useChat() - Complete 0-100% Guide (Plain English)

## 🤔 What is it? (Super Simple)

A React hook that manages your entire chat UI automatically.

Without `useChat()`:
```

├─ Manual state management (messages array)
├─ Handle input changes
├─ Send messages to API
├─ Parse streaming response
├─ Update UI in real-time
├─ Handle errors
└─ Manage loading states

```

With `useChat()`:
✅ One hook does everything!

In one sentence: It's a React hook that handles all the complex chat logic so you just focus on the UI.

## 📊 What useChat() Does For You

Automatically Handles:

```

✅ Message history management
✅ Input field state
✅ Streaming responses
✅ Real-time text updates
✅ Error handling
✅ Loading states
✅ Tool call processing
✅ Message regeneration
✅ Stream interruption
✅ Form submission

````

## 🎯 When Do You Need It?

✅ **USE `useChat()` FOR:**

- 💬 Chatbots (multi-turn conversations)
- 🤖 AI Assistants (Claude, GPT, Gemini)
- 🎯 Chat UIs (Discord-like interfaces)
- 📱 Messaging (SMS-like apps)
- 🔄 Conversational AI (back-and-forth dialogue)
- 🛠️ Tool-calling (AI calling functions)
- 📊 Agent applications (multi-step reasoning)

❌ **DON'T USE** if:

- ✂️ One-shot generation (use `useCompletion()`)
- 🖼️ Image generation only
- 📝 Form completion only
- 🔍 Search results only

## 📍 Installation

```bash
npm install ai @ai-sdk/react @ai-sdk/google
````

Set up environment:

```bash
export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"
```

## 🚀 Complete Working Example (Gemini 2.0 Flash, V6)

### Backend API Route

```ts
// app/api/chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { convertToModelMessages } from 'ai';

export const maxDuration = 30;

export async function POST(req: Request) {
  const { messages } = await req.json();

  // Convert UIMessages to ModelMessages
  const modelMessages = await convertToModelMessages(messages);

  // Create stream with Gemini
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: modelMessages,
    system: 'You are a helpful AI assistant.',
  });

  // Stream back to frontend
  return result.toUIMessageStreamResponse();
}
```

### Frontend Chat Component

```tsx
'use client';
import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function ChatApp() {
  const {
    messages,
    input,
    handleInputChange,
    handleSubmit,
    isLoading,
    error,
    append,
    reload,
    stop,
    setMessages,
  } = useChat({
    api: '/api/chat',
  });

  return (
    <div className="chat-container">
      {/* Messages Display */}
      <div className="messages">
        {messages.map((message) => (
          <div key={message.id} className={`message ${message.role}`}>
            {message.parts?.map((part, i) => {
              if (part.type === 'text') return <p key={i}>{part.text}</p>;
              if (part.type === 'tool-call') {
                return (
                  <div key={i} className="tool-call">
                    📞 Calling: {part.toolName}
                  </div>
                );
              }
              return null;
            })}
          </div>
        ))}

        {isLoading && <div className="loading">⏳ AI is thinking...</div>}
        {error && <div className="error">❌ {error.message}</div>}
      </div>

      {/* Input Form */}
      <form onSubmit={handleSubmit} className="input-form">
        <input
          type="text"
          value={input}
          onChange={handleInputChange}
          placeholder="Ask anything..."
          disabled={isLoading}
        />
        <button type="submit" disabled={isLoading}>
          {isLoading ? '⏳ Sending...' : '📤 Send'}
        </button>
        {isLoading && <button type="button" onClick={stop}>⏹️ Stop</button>}
      </form>

      {/* Action Buttons */}
      <div className="actions">
        <button onClick={() => reload()}>🔄 Retry</button>
        <button onClick={() => setMessages([])}>🗑️ Clear Chat</button>
      </div>
    </div>
  );
}
```

## 🎛️ Complete API Reference

### useChat() Hook

```ts
const {
  messages,
  input,
  error,
  isLoading,
  status,
  handleInputChange,
  handleSubmit,
  append,
  reload,
  stop,
  setMessages,
  setInput,
  clearError,
  regenerate,
  resumeStream,
  addToolOutput,
  addToolApprovalResponse,
} = useChat(options);
```

### useChat() Options

```ts
const chat = useChat({
  api: '/api/chat',                    // Backend endpoint
  id: 'chat-123',                      // Unique chat ID
  initialMessages: [],                 // Start with messages
  body: { userId: '123' },             // Extra data sent to backend
  credentials: 'same-origin',          
  headers: { 'X-Auth': 'token' },      
  onFinish: (message) => { console.log('Done!', message); },
  onError: (error) => { console.error('Error:', error); },
  experimental_throttle: 50,
  resume: false,
  fetch: customFetch,
});
```

### messages Array

```ts
messages = [
  {
    id: 'msg-1',
    role: 'user',
    parts: [{ type: 'text', text: 'Hello!' }]
  },
  {
    id: 'msg-2',
    role: 'assistant',
    parts: [{ type: 'text', text: 'Hi there!' }]
  }
]
```

### status States

```ts
status = 'ready'          // ✅ Ready for input
status = 'in_progress'    // ⏳ Streaming response
status = 'done'           // ✅ Finished
status = 'error'          // ❌ Error occurred
```

## 💡 Common Patterns

**Pattern 1: Basic Chat**

```tsx
export default function BasicChat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat();
  return (
    <div>
      {messages.map(m => <div key={m.id}>{m.parts?.[0]?.text}</div>)}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button>Send</button>
      </form>
    </div>
  );
}
```

**Pattern 2: Type Safety**
**Pattern 3: With Tools**
**Pattern 4: Shared State (Multi-Component)**
**Pattern 5: Programmatic Message Sending**

*(Patterns omitted for brevity here but can be included fully in MDX.)*

## 🔄 Message Flow

```
User Input
    ↓
handleInputChange() updates 'input' state
    ↓
handleSubmit() / append()
    ↓
POST to /api/chat with messages
    ↓
Backend creates stream
    ↓
useChat receives SSE chunks
    ↓
Real-time message update
    ↓
UI re-renders with new text
    ↓
Stream complete, status = 'done'
```

## 📱 Styling Example (Tailwind)

```tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function StyledChat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading, error } =
    useChat({ api: '/api/chat' });

  return (
    <div className="flex flex-col h-screen bg-gradient-to-b from-slate-900 to-slate-800">
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map(m => (
          <div key={m.id} className={`flex ${m.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`max-w-xs px-4 py-2 rounded-lg ${m.role === 'user' ? 'bg-blue-500 text-white' : 'bg-gray-700 text-gray-100'}`}>
              {m.parts?.[0]?.text}
            </div>
          </div>
        ))}
        {isLoading && <div className="flex justify-start"><div className="bg-gray-700 text-gray-100 px-4 py-2 rounded-lg">⏳ Thinking...</div></div>}
        {error && <div className="flex justify-center"><div className="bg-red-500 text-white px-4 py-2 rounded-lg">❌ {error.message}</div></div>}
      </div>
      <form onSubmit={handleSubmit} className="p-4 border-t border-gray-700">
        <div className="flex gap-2">
          <input type="text" value={input} onChange={handleInputChange} placeholder="Type message..." disabled={isLoading} className="flex-1 px-4 py-2 bg-gray-700 text-white rounded-lg outline-none"/>
          <button type="submit" disabled={isLoading} className="px-6 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:opacity-50">Send</button>
        </div>
      </form>
    </div>
  );
}
```

## ⚠️ Common Mistakes

1. Forgetting `'use client'`
2. Wrong API endpoint
3. Not handling loading state
4. Not handling errors
5. Calling `handleSubmit` without form

## 🎉 Summary

* **What:** React hook that manages entire chat UI
* **Where:** Frontend React components (`'use client'`)
* **When:** Building chat applications with AI
* **Why:** Handles streaming, state, errors automatically
* **How:** `const { messages, input, handleSubmit } = useChat()`
* **Install:** `npm install ai @ai-sdk/react @ai-sdk/google`
* **Backend:** Create `/api/chat` route that streams responses

## 🚀 Quick Checklist

```
✅ Add 'use client' to component
✅ Install: ai @ai-sdk/react @ai-sdk/google
✅ Create backend /api/chat route
✅ Use useChat() hook
✅ Render messages array
✅ Show input with handleSubmit
✅ Show loading state
✅ Show error handling
✅ Deploy!
```

You're now a `useChat()` expert! 🎊
This hook handles 95% of chat UI complexity. Just focus on:

* What to display (messages)
* How to style it (CSS)
* What interactions to add (buttons, etc.)

Everything else is automatic! 🚀



