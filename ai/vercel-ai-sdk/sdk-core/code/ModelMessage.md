

---

# **Complete Guide to ModelMessage in Vercel AI SDK (0-100%)**

## **What is ModelMessage?**

`ModelMessage` is a **TypeScript type** in the Vercel AI SDK that represents a conversation message you send to an AI model (like Google Gemini 3 Flash). Think of it as a standardized format for structuring messages in a chat conversation. The AI SDK uses ModelMessage to ensure your messages are properly formatted before sending them to the model.

---

## **Plain English Explanation**

Imagine you're building a chatbot. Users type messages, the AI responds, and you need to keep track of the conversation. **ModelMessage is the blueprint** that says: *"Every message in this conversation must have this structure."*

It's like saying: "When you talk to the AI, you need to tell it:
1. **Who's speaking?** (Are you a user, the assistant, or the system?)
2. **What are they saying?** (Text, images, files, tool results, etc.)
3. **Any special instructions?** (Provider-specific settings)"

---

## **The 4 Types of ModelMessage**

### **1. SystemModelMessage (The Boss/Rulebook)**

**What it does:** Sets the rules for how the AI should behave. It's like giving instructions before asking a question.

**Plain English:** "Here are the rules the AI must follow for this entire conversation."

**Example:**
```typescript
import { ModelMessage } from 'ai';

const systemMessage: ModelMessage = {
  role: 'system',
  content: 'You are a helpful assistant. Be concise and friendly.',
};
```

**Real-world analogy:** Like telling a waiter: "I want you to be extra polite and suggest vegetarian options."

---

### **2. UserModelMessage (The Customer/User)**

**What it does:** Represents what the user is asking or saying. This is the user's input.

**Plain English:** "This is what the human user is saying/asking."

**Simple Text Example:**
```typescript
const userMessage: ModelMessage = {
  role: 'user',
  content: 'What is the weather today?',
};
```

**With Multiple Content Types (Text + Image):**
```typescript
const userMessageWithImage: ModelMessage = {
  role: 'user',
  content: [
    {
      type: 'text',
      text: 'Describe this image in detail.'
    },
    {
      type: 'image',
      image: 'https://example.com/photo.jpg' // Can be URL or base64
    }
  ],
};
```

**With File (like PDF):**
```typescript
import fs from 'fs';

const userMessageWithFile: ModelMessage = {
  role: 'user',
  content: [
    {
      type: 'text',
      text: 'Summarize this document.'
    },
    {
      type: 'file',
      mediaType: 'application/pdf',
      data: fs.readFileSync('./document.pdf'), // Binary data
      filename: 'document.pdf' // Optional
    }
  ],
};
```

**Real-world analogy:** Like a customer at a restaurant saying "I want this dish, and here's a photo of how I want it prepared."

---

### **3. AssistantModelMessage (The AI Response)**

**What it does:** Represents what the AI has said previously. Used to show the AI's past responses in a conversation.

**Plain English:** "This is what the AI said before."

**Simple Text Response:**
```typescript
const assistantMessage: ModelMessage = {
  role: 'assistant',
  content: 'The weather today is sunny with a high of 75°F.',
};
```

**With Tool Calls (AI calling functions):**
```typescript
const assistantWithToolCall: ModelMessage = {
  role: 'assistant',
  content: [
    {
      type: 'text',
      text: 'Let me check the weather for you.'
    },
    {
      type: 'tool-call',
      toolCallId: 'call_123',
      toolName: 'get_weather',
      input: { location: 'San Francisco' }
    }
  ],
};
```

**Real-world analogy:** Like a waiter repeating their previous suggestion: "I recommended the pasta, and I'll now check if it's available."

---

### **4. ToolModelMessage (The Tool Result)**

**What it does:** Sends back the result of a tool call. When the AI calls a function, you send the result back using this message type.

**Plain English:** "Here's the result from calling that tool/function the AI requested."

**Example:**
```typescript
const toolResultMessage: ModelMessage = {
  role: 'tool',
  content: [
    {
      type: 'tool-result',
      toolCallId: 'call_123', // Must match the tool-call ID
      content: JSON.stringify({ 
        location: 'San Francisco', 
        temperature: 75, 
        condition: 'Sunny' 
      })
    }
  ],
};
```

**Real-world analogy:** The restaurant's kitchen saying back to the waiter: "We checked—the pasta is available. Here are the details."

---

## **What Content Can Go Inside ModelMessage?**

Each message type can contain different "content parts":

| Content Type | Use Case | Example |
|---|---|---|
| `text` | Plain text content | `{ type: 'text', text: 'Hello!' }` |
| `image` | Send images to the AI | `{ type: 'image', image: 'https://...' }` |
| `file` | Send PDFs, audio, documents | `{ type: 'file', mediaType: 'application/pdf', data: Buffer }` |
| `reasoning` (assistant only) | AI's internal thinking | `{ type: 'reasoning', text: 'Thinking about...' }` |
| `tool-call` (assistant only) | AI requesting a function call | `{ type: 'tool-call', toolName: 'search', ... }` |
| `tool-result` (tool role only) | Result from a function | `{ type: 'tool-result', toolCallId: '123', content: '...' }` |

---

## **Provider Options (Advanced)**

You can add special instructions for specific AI providers (like Google Gemini, OpenAI, Anthropic, etc.):

```typescript
const messageWithProviderOptions: ModelMessage = {
  role: 'system',
  content: 'You are helpful.',
  providerOptions: {
    anthropic: {
      // Anthropic-specific setting (caching)
      cacheControl: { type: 'ephemeral' }
    },
    openai: {
      // OpenAI-specific setting (reasoning effort)
      reasoningEffort: 'high'
    }
  },
};
```

---

## **How to Use ModelMessage - Real World Example**

### **Setting Up Your Chat with Gemini 3 Flash**

**Step 1: Install the AI SDK**
```bash
npm install ai @ai-sdk/google
```

**Step 2: Create Your Message Array**
```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';
import type { ModelMessage } from 'ai';

// Build your conversation
const messages: ModelMessage[] = [
  // System instruction
  {
    role: 'system',
    content: 'You are a helpful travel assistant. Give concise recommendations.',
  },
  // User asks a question
  {
    role: 'user',
    content: 'What should I do in Paris?',
  },
  // AI previously responded
  {
    role: 'assistant',
    content: 'Here are top things to do in Paris:\n1. Eiffel Tower\n2. Louvre Museum\n3. Notre-Dame',
  },
  // User follows up
  {
    role: 'user',
    content: 'Tell me more about the Louvre.',
  },
];

// Send to Gemini 3 Flash
const result = await generateText({
  model: google('gemini-2.0-flash'),
  messages: messages,
  system: 'You are a helpful assistant.',
});

console.log(result.text);
```

---

## **Real-World Chatbot Example with Vercel v6**

### **Backend (Next.js API Route)**

```typescript
// app/api/chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { convertToModelMessages } from 'ai';
import type { UIMessage } from 'ai';

export async function POST(req: Request) {
  const { messages }: { messages: UIMessage[] } = await req.json();

  // Convert UI messages to ModelMessages
  const modelMessages = await convertToModelMessages(messages);

  // Send to Gemini 3 Flash with streaming
  const result = streamText({
    model: google('gemini-2.0-flash'),
    system: 'You are a helpful AI assistant.',
    messages: modelMessages, // This is where ModelMessage is used!
  });

  return result.toUIMessageStreamResponse();
}
```

### **Frontend (React Component)**

```typescript
// app/page.tsx
'use client';

import { useChat } from '@ai-sdk/react';
import { useState } from 'react';

export default function ChatPage() {
  const [input, setInput] = useState('');
  const { messages, sendMessage, status } = useChat();

  return (
    <div className="flex flex-col w-full max-w-2xl mx-auto p-4">
      <h1 className="text-3xl font-bold mb-4">Chat with Gemini 3 Flash</h1>

      {/* Display conversation */}
      <div className="flex-1 overflow-y-auto mb-4 border rounded p-4">
        {messages.map((message) => (
          <div key={message.id} className="mb-4">
            <strong>{message.role === 'user' ? 'You: ' : 'AI: '}</strong>
            {message.parts.map((part, i) => (
              part.type === 'text' ? (
                <div key={i}>{part.text}</div>
              ) : null
            ))}
          </div>
        ))}
      </div>

      {/* Input form */}
      <form
        onSubmit={(e) => {
          e.preventDefault();
          if (input.trim()) {
            sendMessage({ text: input });
            setInput('');
          }
        }}
      >
        <div className="flex gap-2">
          <input
            value={input}
            onChange={(e) => setInput(e.target.value)}
            placeholder="Type your message..."
            disabled={status !== 'ready'}
            className="flex-1 border rounded px-3 py-2"
          />
          <button
            type="submit"
            disabled={status !== 'ready'}
            className="bg-blue-500 text-white px-4 py-2 rounded"
          >
            Send
          </button>
        </div>
      </form>
    </div>
  );
}
```

---

## **When to Use Each Message Type**

| Situation | Message Type | Example |
|---|---|---|
| Set AI behavior rules | `SystemModelMessage` | "You're a Python expert" |
| User types something | `UserModelMessage` | User asks "How do I use React?" |
| AI responded before | `AssistantModelMessage` | Show previous AI responses |
| AI called a tool/function | `ToolModelMessage` | AI called `get_weather()`, return result |
| Continuing a conversation | Array of all 4 types | Multi-turn chat conversation |

---

## **Advanced Use Cases**

### **1. Multi-turn Chat with Tool Calling**

```typescript
import { generateText, tool, stepCountIs } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';
import type { ModelMessage } from 'ai';

const messages: ModelMessage[] = [
  {
    role: 'user',
    content: 'What is the weather in SF?',
  },
  // After first call, AI generates tool-call
  {
    role: 'assistant',
    content: [
      {
        type: 'tool-call',
        toolCallId: 'call_1',
        toolName: 'get_weather',
        input: { location: 'San Francisco' }
      }
    ]
  },
  // Return tool result
  {
    role: 'tool',
    content: [
      {
        type: 'tool-result',
        toolCallId: 'call_1',
        content: '{"temp": 72, "condition": "sunny"}'
      }
    ]
  }
];

const result = await generateText({
  model: google('gemini-2.0-flash'),
  messages: messages,
  tools: {
    get_weather: tool({
      description: 'Get weather for a location',
      inputSchema: z.object({
        location: z.string(),
      }),
      execute: async ({ location }) => ({
        temp: 72,
        condition: 'sunny'
      })
    })
  }
});

console.log(result.text);
```

### **2. Multi-Modal Message (Image + Text)**

```typescript
const multiModalMessage: ModelMessage = {
  role: 'user',
  content: [
    {
      type: 'text',
      text: 'Analyze this chart and explain the trend.',
    },
    {
      type: 'image',
      image: 'https://example.com/chart.png',
      mediaType: 'image/png',
    }
  ]
};
```

### **3. With Caching (Anthropic Provider)**

```typescript
const cachedMessage: ModelMessage = {
  role: 'system',
  content: 'Long context that will be cached...',
  providerOptions: {
    anthropic: {
      cacheControl: { type: 'ephemeral' }
    }
  }
};
```

---

## **Key Points to Remember**

✅ **ModelMessage is a TypeScript type** - It's how you structure messages for AI models  
✅ **4 main types** - System, User, Assistant, Tool  
✅ **Content can be complex** - Text, images, files, tool calls  
✅ **Vercel AI SDK converts messages** - From UI format to ModelMessage automatically  
✅ **Provider options available** - For Gemini, OpenAI, Anthropic, etc.  
✅ **Order matters** - Messages are processed in sequence  
✅ **Tool calls need results** - If AI calls a tool, you must return a ToolModelMessage  

---

## **Complete Working Example (Copy & Paste Ready)**

```typescript
// Full working example with Gemini 3 Flash

import { generateText } from 'ai';
import { google } from '@ai-sdk/google';
import type { ModelMessage } from 'ai';

async function chatWithGemini() {
  // Create a conversation
  const messages: ModelMessage[] = [
    {
      role: 'system',
      content: 'You are a helpful JavaScript expert who explains concepts clearly.',
    },
    {
      role: 'user',
      content: 'Explain async/await in simple terms.',
    },
  ];

  // Send to Gemini 3 Flash
  const { text } = await generateText({
    model: google('gemini-2.0-flash'),
    messages: messages,
  });

  console.log('AI Response:', text);
}

chatWithGemini().catch(console.error);
```

---

## **Summary**

**ModelMessage** is the fundamental building block for conversations with AI models in Vercel AI SDK. It's a structured format that ensures every message (whether from a user, system, assistant, or tool) is properly formatted and understood by the AI provider. By mastering ModelMessage, you can build sophisticated chatbots, multi-turn conversations, tool-calling systems, and multi-modal applications with Gemini, GPT, Claude, and other models.
