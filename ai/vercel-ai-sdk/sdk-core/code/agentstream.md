

---

# **Complete Guide: `createAgentUIStream`, `createAgentUIStreamResponse`, `pipeAgentUIStreamToResponse`**

## **Plain English Explanation**

Think of these three functions as different ways to **run an AI agent** (which can use tools, search, execute code, etc.) and **stream the results back to users in real-time**.

### **What is an Agent?**
An Agent is an AI that can:
- Think step-by-step
- Use tools (like weather API, calculator, web search)
- Loop: call AI → get tool calls → execute tools → repeat → finally respond

### **The Three Functions - Quick Overview**

| Function | Purpose | Returns | Use Case |
|----------|---------|---------|----------|
| **`createAgentUIStream`** | Runs agent, gives you a stream you control | Async iterator (loop through chunks) | Custom runtime, dashboards, custom integrations |
| **`createAgentUIStreamResponse`** | Runs agent, returns HTTP Response ready to send | `Promise<Response>` | Next.js, serverless, edge functions |
| **`pipeAgentUIStreamToResponse`** | Runs agent, pipes directly to Node.js response | `Promise<void>` | Express, Hono, Node.js servers |

---

## **1. `createAgentUIStream` - The Raw Stream**

### **What it does:**
Executes your agent and gives you a stream of UI message chunks that you can iterate over and handle however you want.

### **When to use:**
- Building custom streaming solutions
- Creating dashboards that need real-time updates
- Integrating with non-standard frameworks
- You need full control over the stream

### **Syntax:**
```typescript
const stream = await createAgentUIStream({
  agent: myAgent,              // Your AI agent
  uiMessages: messages,        // Array of user/assistant messages
  abortSignal: controller.signal, // Optional: to cancel streaming
  options: callOptions,        // Optional: agent-specific settings
  sendStart: true,             // Optional: emit a start event
  sendSources: true,          // Optional: include sources in output
  includeUsage: true,         // Optional: include token usage
});

for await (const chunk of stream) {
  // Each chunk is a UI message piece
  console.log(chunk);
}
```

### **Example with Gemini 3.0 Flash (Next.js):**
```typescript
'use server';  // Server action in Next.js

import { google } from '@ai-sdk/google';
import { ToolLoopAgent, createAgentUIStream } from 'ai';
import { tool } from '@ai-sdk/provider-utils';
import { z } from 'zod';

// Define a tool the agent can use
const weatherTool = tool({
  description: 'Get weather for a city',
  parameters: z.object({
    city: z.string(),
  }),
  execute: async ({ city }) => {
    // Your weather API call here
    return { temperature: 72, condition: 'sunny' };
  },
});

// Create the agent with Gemini 3.0 Flash
const agent = new ToolLoopAgent({
  model: google('gemini-2.0-flash'),  // Latest Gemini model
  instructions: 'You are a helpful weather assistant.',
  tools: { weather: weatherTool },
});

// Stream the agent output
export async function streamWeatherAgent(messages: unknown[]) {
  const stream = await createAgentUIStream({
    agent,
    uiMessages: messages,
  });

  for await (const chunk of stream) {
    // Process each chunk - could send to client via SSE, WebSocket, etc.
    console.log(chunk);
    yield chunk;  // In an async generator
  }
}
```

### **What you get back:**
```typescript
// Each chunk looks like:
{
  type: 'text',
  text: 'The weather in San Francisco...'
}
// or
{
  type: 'tool-call',
  toolName: 'weather',
  toolCallId: 'call-123',
  args: { city: 'San Francisco' }
}
// or
{
  type: 'tool-result',
  toolCallId: 'call-123',
  result: { temperature: 72, condition: 'sunny' }
}
```

---

## **2. `createAgentUIStreamResponse` - HTTP Response (For Web/Serverless)**

### **What it does:**
Runs your agent AND wraps the stream in an HTTP `Response` object ready to send back to clients. Perfect for modern web frameworks.

### **When to use:**
- **Next.js** (App Router - `export async function POST`)
- **Vercel Edge Functions**
- **Serverless functions** (AWS Lambda, etc.)
- **Any HTTP framework** that returns `Response`
- **Vercel v6** (as you requested)

### **Syntax:**
```typescript
export async function POST(request: Request) {
  const { messages } = await request.json();

  return createAgentUIStreamResponse({
    agent: myAgent,
    uiMessages: messages,
    headers: { 'Custom-Header': 'value' },  // Optional
    status: 200,                             // Optional
    consumeSseStream: false,                // Optional: use Server-Sent Events
  });
}
```

### **Complete Example with Gemini 3.0 Flash (Next.js with Vercel v6):**

**File: `app/api/chat/route.ts`**
```typescript
import { google } from '@ai-sdk/google';
import { ToolLoopAgent, createAgentUIStreamResponse } from 'ai';
import { tool } from '@ai-sdk/provider-utils';
import { z } from 'zod';

// 1. Define tools the agent can use
const weatherTool = tool({
  description: 'Get current weather',
  parameters: z.object({
    city: z.string().describe('City name'),
    country: z.string().optional(),
  }),
  execute: async ({ city, country }) => {
    // In real app, call OpenWeatherMap, WeatherAPI, etc.
    const weather = {
      city,
      temperature: 72,
      condition: 'Sunny',
      humidity: 65,
    };
    return weather;
  },
});

const searchTool = tool({
  description: 'Search the web',
  parameters: z.object({
    query: z.string(),
  }),
  execute: async ({ query }) => {
    // In real app, call web search API
    return {
      results: [`Result about: ${query}`],
    };
  },
});

// 2. Create agent with Gemini 2.0 Flash (latest model)
const chatAgent = new ToolLoopAgent({
  model: google('gemini-2.0-flash'),  // Newest Gemini model
  instructions: `You are a helpful AI assistant. You can:
    - Check weather for any city
    - Search the web for information
    - Answer questions conversationally
    
    Always be accurate and cite your sources.`,
  tools: {
    weather: weatherTool,
    search: searchTool,
  },
  // Optional: configure agent behavior
  temperature: 0.7,
  maxTokens: 1024,
});

// 3. API Route - Handles POST from frontend
export async function POST(request: Request) {
  try {
    const { messages } = await request.json();

    // Validate messages exist
    if (!messages || !Array.isArray(messages)) {
      return new Response('Invalid messages format', { status: 400 });
    }

    // Create and return the streaming response
    return createAgentUIStreamResponse({
      agent: chatAgent,
      uiMessages: messages,
      headers: {
        'Content-Type': 'text/event-stream',
        'Cache-Control': 'no-cache',
      },
      status: 200,
      includeUsage: true,     // Include token counts in response
      sendSources: true,      // Include source information
    });
  } catch (error) {
    console.error('Chat error:', error);
    return new Response('Internal server error', { status: 500 });
  }
}
```

**File: `app/page.tsx`** (Frontend)
```typescript
'use client';

import { useChat } from 'ai/react';

export default function ChatPage() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: '/api/chat',  // Calls our POST endpoint
    });

  return (
    <div>
      <div className="chat-messages">
        {messages.map((msg) => (
          <div key={msg.id} className="message">
            <strong>{msg.role}:</strong> {msg.content}
          </div>
        ))}
      </div>

      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask about weather..."
        />
        <button type="submit" disabled={isLoading}>
          Send
        </button>
      </form>
    </div>
  );
}
```

### **How it works:**
1. ✅ Frontend sends POST request with messages
2. ✅ Agent processes messages, uses tools
3. ✅ Wraps stream in HTTP Response
4. ✅ Frontend receives streamed chunks in real-time
5. ✅ `useChat` hook updates UI incrementally

---

## **3. `pipeAgentUIStreamToResponse` - Node.js Servers**

### **What it does:**
Runs your agent and **pipes the stream directly** to a Node.js `ServerResponse` object. No return value - it writes directly to the response.

### **When to use:**
- **Express.js** servers
- **Hono.js** (with Node.js adapter)
- **Custom Node.js HTTP servers**
- **Any Node.js framework** using native `ServerResponse`
- **NOT for Vercel/serverless** (use `createAgentUIStreamResponse` instead)

### **Syntax:**
```typescript
app.post('/chat', async (req, res) => {
  await pipeAgentUIStreamToResponse({
    response: res,  // Express/Hono response object
    agent: myAgent,
    uiMessages: req.body.messages,
    abortSignal: req.signal,  // Optional: cancel on disconnect
  });
  // No return needed - function writes directly to res
});
```

### **Example with Express.js and Gemini 3.0 Flash:**

**File: `server.ts`**
```typescript
import express, { Request, Response } from 'express';
import { google } from '@ai-sdk/google';
import { ToolLoopAgent, pipeAgentUIStreamToResponse } from 'ai';
import { tool } from '@ai-sdk/provider-utils';
import { z } from 'zod';
import 'dotenv/config';

const app = express();
app.use(express.json());

// 1. Define tools
const calculatorTool = tool({
  description: 'Perform math calculations',
  parameters: z.object({
    operation: z.enum(['add', 'subtract', 'multiply', 'divide']),
    a: z.number(),
    b: z.number(),
  }),
  execute: async ({ operation, a, b }) => {
    const results: Record<string, number> = {
      add: a + b,
      subtract: a - b,
      multiply: a * b,
      divide: a / b,
    };
    return { result: results[operation] };
  },
});

// 2. Create agent
const agent = new ToolLoopAgent({
  model: google('gemini-2.0-flash'),
  instructions: 'You are a math tutor. Help users with calculations.',
  tools: { calculator: calculatorTool },
});

// 3. Express route
app.post('/api/chat', async (req: Request, res: Response) => {
  try {
    const { messages } = req.body;

    // Set headers for streaming
    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');

    // Pipe agent stream directly to response
    await pipeAgentUIStreamToResponse({
      response: res,
      agent,
      uiMessages: messages,
      abortSignal: req.signal,  // Abort on client disconnect
      status: 200,
    });
  } catch (error) {
    console.error('Error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.listen(8080, () => {
  console.log('Server running on http://localhost:8080');
});
```

### **Using with Hono.js (Node adapter):**

**File: `hono-server.ts`**
```typescript
import { serve } from '@hono/node-server';
import { Hono } from 'hono';
import { google } from '@ai-sdk/google';
import { ToolLoopAgent, pipeAgentUIStreamToResponse } from 'ai';
import { tool } from '@ai-sdk/provider-utils';
import { z } from 'zod';

const app = new Hono();

const agent = new ToolLoopAgent({
  model: google('gemini-2.0-flash'),
  instructions: 'Helpful assistant',
  tools: { /* ... tools ... */ },
});

app.post('/chat', async (c) => {
  const { messages } = await c.req.json();

  // Get Node.js response from Hono context
  const res = c.env?.outgoing as any;

  await pipeAgentUIStreamToResponse({
    response: res,
    agent,
    uiMessages: messages,
  });

  return c.body(res);
});

serve({ fetch: app.fetch, port: 8080 });
```

---

## **Key Concepts Explained**

### **What are "UI Messages"?**
Input messages the user sends. They contain conversation history:
```typescript
[
  {
    role: 'user',
    content: 'What is the weather?'
  },
  {
    role: 'assistant',
    content: 'I will check for you...'
  },
  {
    role: 'user',
    content: 'In San Francisco please'
  }
]
```

### **What are "UI Message Chunks"?**
The streaming output from the agent - pieces of the response:
```typescript
// Chunk 1: Start of response
{ type: 'start' }

// Chunk 2: Agent thinks and calls a tool
{ type: 'tool-call', toolName: 'weather', args: { city: 'SF' } }

// Chunk 3: Tool result comes back
{ type: 'tool-result', result: { temperature: 72 } }

// Chunk 4: Final text response
{ type: 'text', text: 'The weather in SF is 72°F' }

// Chunk 5: Done
{ type: 'finish' }
```

### **What is an AbortSignal?**
A way to cancel streaming early (e.g., user closes browser):
```typescript
const controller = new AbortController();

// Later, cancel:
controller.abort();  // Stops agent immediately
```

---

## **Comparison Table: Which Function to Use?**

| Scenario | Function | Why |
|----------|----------|-----|
| Next.js API Route | `createAgentUIStreamResponse` ✅ | Returns `Response` object automatically |
| Vercel Edge Function | `createAgentUIStreamResponse` ✅ | Perfect for edge runtime |
| Express.js | `pipeAgentUIStreamToResponse` ✅ | Works with Node.js ServerResponse |
| Hono.js (Node) | `pipeAgentUIStreamToResponse` ✅ | Works with Node.js ServerResponse |
| Custom Streaming | `createAgentUIStream` ✅ | Full control, async iteration |
| WebSocket | `createAgentUIStream` ✅ | Iterate and send to socket |
| Browser-only | ❌ None | Must use server-side function |

---

## **Complete Working Example: Full Stack Chat App**

### **Setup:**
```bash
npm install ai @ai-sdk/google
# Set environment variable
export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"
```

### **Backend (Next.js 15 + Vercel v6):**

**`app/api/chat/route.ts`**
```typescript
import { google } from '@ai-sdk/google';
import {
  ToolLoopAgent,
  createAgentUIStreamResponse,
} from 'ai';
import { tool } from '@ai-sdk/provider-utils';
import { z } from 'zod';

const webSearchTool = tool({
  description: 'Search the web',
  parameters: z.object({
    query: z.string(),
  }),
  execute: async ({ query }) => {
    // Real implementation would call actual search API
    return {
      results: [
        { title: 'Result 1', snippet: `Info about ${query}` },
      ],
    };
  },
});

const agent = new ToolLoopAgent({
  model: google('gemini-2.0-flash'),
  instructions: 'You are helpful. Use web search when needed.',
  tools: { webSearch: webSearchTool },
});

export async function POST(request: Request) {
  const { messages } = await request.json();

  return createAgentUIStreamResponse({
    agent,
    uiMessages: messages,
    sendSources: true,
    includeUsage: true,
  });
}
```

**`app/page.tsx`**
```typescript
'use client';

import { useChat } from 'ai/react';
import { useState } from 'react';

export default function Home() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({
      api: '/api/chat',
      streamProtocol: 'text/event-stream',
    });

  return (
    <div className="max-w-2xl mx-auto p-4">
      <h1 className="text-2xl font-bold mb-4">AI Chat with Gemini 3.0 Flash</h1>

      <div className="space-y-4 mb-4 h-96 overflow-auto border rounded p-4">
        {messages.map((msg) => (
          <div key={msg.id} className={`p-2 rounded ${
            msg.role === 'user' ? 'bg-blue-100' : 'bg-gray-100'
          }`}>
            <strong className="capitalize">{msg.role}:</strong>
            <p>{msg.content}</p>
          </div>
        ))}
        {isLoading && <p className="text-gray-500">Thinking...</p>}
      </div>

      <form onSubmit={handleSubmit} className="flex gap-2">
        <input
          value={input}
          onChange={handleInputChange}
          placeholder="Ask anything..."
          className="flex-1 border rounded px-3 py-2"
        />
        <button
          type="submit"
          disabled={isLoading}
          className="bg-blue-500 text-white px-4 py-2 rounded disabled:opacity-50"
        >
          {isLoading ? 'Sending...' : 'Send'}
        </button>
      </form>
    </div>
  );
}
```

---

## **Optional Parameters Deep Dive**

### **`abortSignal`**
```typescript
const controller = new AbortController();

const stream = await createAgentUIStream({
  agent,
  uiMessages,
  abortSignal: controller.signal,
});

// In another part of code:
controller.abort();  // Stops streaming
```

### **`sendStart` / `sendSources` / `includeUsage`**
```typescript
await createAgentUIStreamResponse({
  agent,
  uiMessages,
  sendStart: true,        // Send { type: 'start' } event
  sendSources: true,      // Include source info from web search
  includeUsage: true,     // Include token counts: { inputTokens, outputTokens }
});
```

### **`headers` / `status`**
```typescript
return createAgentUIStreamResponse({
  agent,
  uiMessages,
  headers: {
    'X-Custom-Header': 'value',
    'X-Request-ID': generateRequestId(),
  },
  status: 200,
  statusText: 'OK',
});
```

---

## **Error Handling**

```typescript
export async function POST(request: Request) {
  try {
    const { messages } = await request.json();

    if (!messages || !Array.isArray(messages)) {
      return new Response('Invalid messages', { status: 400 });
    }

    return createAgentUIStreamResponse({
      agent,
      uiMessages: messages,
    });
  } catch (error) {
    console.error('Chat error:', error);

    if (error instanceof Error && error.message.includes('abort')) {
      return new Response('Request cancelled', { status: 408 });
    }

    return new Response('Internal server error', { status: 500 });
  }
}
```

---

## **Performance Tips**

1. **Use `abortSignal`** - Cancel requests when users disconnect
2. **Enable `includeUsage`** - Track token costs
3. **Set `temperature` on agent** - Lower = faster, deterministic
4. **Use tool validation** - Catch errors early with Zod schemas
5. **Stream early** - Don't batch - send chunks as they arrive

---

## **Summary**

| Function | Use Case | Best For |
|----------|----------|----------|
| **`createAgentUIStream`** | Control the stream yourself | Custom integrations |
| **`createAgentUIStreamResponse`** | Return HTTP Response | Next.js, Vercel, Serverless |
| **`pipeAgentUIStreamToResponse`** | Pipe to Node.js response | Express, Hono, Node servers |

**All three work seamlessly with Gemini 2.0 Flash and Vercel v6!**
