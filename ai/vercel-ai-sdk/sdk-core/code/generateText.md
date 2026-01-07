🎯 streamText() - Complete 0-100% Guide (Plain English)

🤔 What is it? (Super Simple)

A server-side function that generates AI text and streams it back chunk-by-chunk in real-time.
plaintext

generateText():
Backend → AI → Wait for complete response → Return all at once
streamText():
Backend → AI → Stream chunks → Frontend sees live typing ✨
         chunk 1 ("Hello")
         chunk 2 (" there")
         chunk 3 (" friend")
         (done!)

In one sentence: It's a function that generates AI text and sends it to the frontend one piece at a time so users see typing in real-time.

📊 streamText() vs generateText() vs useChat()
Feature	generateText()	streamText()	useChat()
Where	Backend only	Backend only	Frontend hook
Streaming	❌ No	✅ Yes	✅ Yes (auto)
Wait for complete	✅ Yes	❌ No	❌ No
Use Case	One-shot	Chat/API	Chat UI
Returns	Result object	Stream object	Hook state
Speed	Slower (waits)	⚡ Fastest	⚡ Fastest
Real-time	❌ No	✅ Yes	✅ Yes
Complexity	Simple	Medium	Simple
Messages	Prompt only	Support messages	Auto managed

🎯 When Do You Need It?

✅ USE streamText() FOR:


    💬 Chat applications (real-time typing)

    🤖 AI assistants (live response display)

    📝 Text generation with streaming (articles, emails)

    🎯 API endpoints that need streaming

    ⚡ Better UX (users see progress)

    🔄 Multi-turn conversations (with messages)

    🛠️ Tool calling (with streaming)

    📱 Real-time updates needed

❌ DON'T USE if:


    🔧 One-shot generation (use generateText())

    💾 Don't need streaming (use generateText())

    🎨 Frontend UI only (use useChat() with /api/chat)

📍 Installation
bash

npm install ai @ai-sdk/google

Set up environment:
bash

export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"

🚀 Complete Working Example (Gemini 2.0 Flash, V6)

Example 1: Basic Streaming API
ts

// app/api/stream/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
    system: 'You are a helpful AI assistant.',
    maxTokens: 1000,
    temperature: 0.7,
  });

  // Convert stream to HTTP response
  return result.toTextStreamResponse();
}

Example 2: Streaming to Frontend
tsx

// app/page.tsx
'use client';
import { useState } from 'react';

export default function StreamingDemo() {
  const [input, setInput] = useState('');
  const [output, setOutput] = useState('');
  const [isLoading, setIsLoading] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setIsLoading(true);
    setOutput('');

    try {
      const response = await fetch('/api/stream', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ prompt: input }),
      });

      // Read streaming response
      const reader = response.body?.getReader();
      const decoder = new TextDecoder();

      while (true) {
        const { done, value } = await reader!.read();
        if (done) break;

        // Decode and append each chunk
        const text = decoder.decode(value);
        setOutput((prev) => prev + text);
      }
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="max-w-2xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">✨ Streaming AI</h1>

      <form onSubmit={handleSubmit} className="space-y-4">
        <textarea
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask anything..."
          className="w-full px-4 py-2 border rounded-lg h-24"
          disabled={isLoading}
        />
        <button
          type="submit"
          disabled={isLoading}
          className="w-full px-4 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:opacity-50"
        >
          {isLoading ? '⏳ Streaming...' : '📤 Send'}
        </button>
      </form>

      {output && (
        <div className="mt-6 p-4 bg-gray-50 rounded-lg">
          <h2 className="font-bold mb-2">Response:</h2>
          <p className="whitespace-pre-wrap">{output}</p>
        </div>
      )}
    </div>
  );
}

Example 3: Chat API with Messages
ts

// app/api/chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { convertToModelMessages } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  // Convert UI messages to model format
  const modelMessages = await convertToModelMessages(messages);

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: modelMessages,  // ← Multiple turns
    system: 'You are a helpful assistant.',
  });

  // Stream response
  return result.toUIMessageStreamResponse();
}

Example 4: With Tools/Functions
ts

// app/api/stream-with-tools/route.ts
import { streamText, tool } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

const weatherTool = tool({
  description: 'Get weather for a city',
  inputSchema: z.object({
    city: z.string().describe('City name'),
  }),
  execute: async ({ city }) => {
    return {
      city,
      temperature: 72,
      condition: 'sunny',
    };
  },
});

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    tools: {
      weather: weatherTool,
    },
    toolChoice: 'auto',
    prompt,
    system: 'You can use tools to get information.',
  });

  return result.toTextStreamResponse();
}

Example 5: Stream with onFinish Callback
ts

// app/api/stream-save/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { db } from '@/lib/db';

export async function POST(req: Request) {
  const { prompt, userId } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
    async onFinish({ text, finishReason, usage }) {
      // Save complete response to database
      await db.insert('ai_responses', {
        userId,
        prompt,
        response: text,
        finishReason,
        tokens: usage,
        createdAt: new Date(),
      });
    },
  });

  return result.toTextStreamResponse();
}

Example 6: Using with useChat()
ts

// app/api/chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { convertToModelMessages } from 'ai';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: await convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}

// app/page.tsx
'use client';
import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit, isLoading } =
    useChat({ api: '/api/chat' });

  return (
    <div className="chat">
      {messages.map((m) => (
        <div key={m.id}>{m.parts?.[0]?.text}</div>
      ))}
      <form onSubmit={handleSubmit}>
        <input value={input} onChange={handleInputChange} />
        <button disabled={isLoading}>Send</button>
      </form>
    </div>
  );
}

Example 7: Custom Stream Processing
ts

// app/api/custom-stream/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
    experimental_telemetry: {
      isEnabled: true,
    },
    onChunk({ chunk }) {
      // Process each chunk
      if (chunk.type === 'text-delta') {
        console.log('Chunk:', chunk.delta);
      }
    },
    onStepFinish({ text, finishReason }) {
      console.log('Step complete. Reason:', finishReason);
    },
  });

  return result.toTextStreamResponse();
}

Example 8: Error Handling
ts

// app/api/stream-safe/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  try {
    const { prompt } = await req.json();

    if (!prompt) {
      return new Response('Prompt required', { status: 400 });
    }

    const result = await streamText({
      model: google('gemini-2.0-flash'),
      prompt,
      maxRetries: 2,
      onError: (error) => {
        console.error('Stream error:', error);
      },
    });

    return result.toTextStreamResponse();
  } catch (error) {
    console.error('API error:', error);
    return new Response('Internal error', { status: 500 });
  }
}

🎛️ Complete API Reference

streamText() Function
ts

const result = await streamText({
  // REQUIRED
  model: google('gemini-2.0-flash'),  // AI model

  // Prompt or Messages (choose one or both)
  prompt: 'Write a story',             // Single prompt
  messages: [{                         // Conversation
    role: 'user',
    content: 'Write a story'
  }],

  // Optional: Output Structure
  output: Output.object({              // Structured output
    schema: z.object({
      title: z.string(),
      content: z.string(),
    })
  }),

  // Optional: System Settings
  system: 'You are helpful',           // System prompt
  maxTokens: 1000,                     // Max response length
  temperature: 0.7,                    // Creativity (0-2)
  topP: 0.9,                          // Nucleus sampling
  topK: 40,                           // Top-K sampling
  frequencyPenalty: 0,                // Repeat penalty
  presencePenalty: 0,                 // New topic penalty
  stopSequences: ['END'],             // Stop generation

  // Optional: Tools
  tools: {
    weather: tool({ ... }),
  },
  toolChoice: 'auto',                 // 'auto', 'required', or tool name

  // Optional: Callbacks
  onChunk({ chunk }) {                // Each chunk received
    console.log(chunk);
  },
  onStepFinish({ text, usage }) {     // Step complete
    console.log('Step done:', text);
  },
  onFinish({ text, usage }) {         // Stream complete
    console.log('Done:', text);
  },
  onError(error) {                    // Error occurred
    console.error(error);
  },

  // Optional: Advanced
  maxRetries: 2,                      // Retry failed calls
  abortSignal: controller.signal,     // Cancel request
  headers: { 'X-Custom': 'value' },   // Custom headers
});

Return Value
ts

const result = await streamText({...});

// Use .toTextStreamResponse() for plain text
return result.toTextStreamResponse();

// Or .toUIMessageStreamResponse() for chat UI
return result.toUIMessageStreamResponse();

// Access stream directly
const stream = result.toAIStream();

💡 Real-World Examples

Example 1: Live Blog Writing
ts

// app/api/write-blog/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { topic, wordCount = 1000 } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: `Write a comprehensive blog post about ${topic}. Target: ${wordCount} words.`,
    system: `You are a professional blog writer. Write engaging, well-structured posts.
             Use markdown formatting. Include headings and sections.`,
    maxTokens: wordCount + 500,
  });

  return result.toTextStreamResponse();
}

Example 2: Streaming Chat with Context
ts

// app/api/context-chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { convertToModelMessages } from 'ai';

export async function POST(req: Request) {
  const { messages, context } = await req.json();

  const systemPrompt = `You are a helpful AI assistant.
  
Context about the user:
${context}

Use this context to provide personalized responses.`;

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    system: systemPrompt,
    messages: await convertToModelMessages(messages),
  });

  return result.toUIMessageStreamResponse();
}

Example 3: Code Generation Stream
ts

// app/api/generate-code/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { description, language = 'typescript' } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: `Generate ${language} code for: ${description}`,
    system: `You are an expert ${language} developer.
             Generate clean, well-documented code.
             Include comments and error handling.
             Use best practices.`,
    maxTokens: 2000,
  });

  return result.toTextStreamResponse();
}

Example 4: Document Processing
ts

// app/api/analyze-document/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { document } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: `Analyze this document and provide:
1. Summary
2. Key points
3. Action items
4. Sentiment

Document:
${document}`,
    system: 'You are a document analyst. Provide structured analysis.',
  });

  return result.toTextStreamResponse();
}

Example 5: Real-time Translation
ts

// app/api/translate-stream/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { text, targetLanguage } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: text,
    system: `Translate to ${targetLanguage}.
             Maintain tone and style.
             Return only the translation.`,
    maxTokens: text.length * 1.5,
  });

  return result.toTextStreamResponse();
}

🔄 Complete Flow
plaintext

Frontend Component
    ↓
User clicks "Send"
    ↓
fetch('/api/chat', { method: 'POST', body })
    ↓
Backend: streamText()
    ↓
Connect to Gemini AI
    ↓
Start streaming response ✨
    chunk 1 → chunk 2 → chunk 3 → ...
    ↓
Frontend receives chunks
    ↓
.toTextStreamResponse() sends as SSE
    ↓
Frontend reads stream
    ↓
Display in real-time ⏱️
    ↓
Stream completes

📊 Output Modes

Plain Text Stream
ts

const result = await streamText({ ... });
return result.toTextStreamResponse();

// Sends: "Hello world"

UI Message Stream (for useChat)
ts

const result = await streamText({ ... });
return result.toUIMessageStreamResponse();

// Sends structured chunks for chat UI

AI Stream (low-level)
ts

const result = await streamText({ ... });
const stream = result.toAIStream();
// Use with custom consumers

⚠️ Common Mistakes

❌ Mistake 1: Not Converting Messages
ts

// ❌ WRONG - UI messages won't work directly
const result = await streamText({
  messages: uiMessages,  // ❌ Wrong format!
});

// ✅ CORRECT - Convert to model format
const result = await streamText({
  messages: await convertToModelMessages(uiMessages),
});

❌ Mistake 2: Forgetting Response Format
ts

// ❌ WRONG - Returns stream object, not HTTP response
export async function POST() {
  const result = await streamText({ ... });
  return result;  // ❌ Not a valid HTTP response
}

// ✅ CORRECT - Convert to HTTP response
export async function POST() {
  const result = await streamText({ ... });
  return result.toTextStreamResponse();
}

❌ Mistake 3: Not Handling Errors
ts

// ❌ WRONG - Errors not caught
export async function POST() {
  const result = await streamText({ ... });
  return result.toTextStreamResponse();
}

// ✅ CORRECT - Handle errors
export async function POST() {
  try {
    const result = await streamText({ ... });
    return result.toTextStreamResponse();
  } catch (error) {
    return new Response('Error', { status: 500 });
  }
}

❌ Mistake 4: Wrong Model
ts

// ❌ WRONG - Old model
await streamText({
  model: openai('gpt-3.5-turbo'),
});

// ✅ CORRECT - Use Gemini 2.0 Flash
await streamText({
  model: google('gemini-2.0-flash'),
});

❌ Mistake 5: Not Setting Token Limits
ts

// ❌ WRONG - Unlimited tokens
const result = await streamText({
  prompt: 'Write a book',
  // No maxTokens!
});

// ✅ CORRECT - Set limit
const result = await streamText({
  prompt: 'Write a book',
  maxTokens: 5000,
});

🎯 When to Use Each

Use streamText() When:
ts

// ✅ Need real-time streaming
const result = await streamText({ ... });
return result.toTextStreamResponse();

// ✅ Building chat
const result = await streamText({
  messages: await convertToModelMessages(messages),
});
return result.toUIMessageStreamResponse();

// ✅ Want better UX (users see progress)

Use generateText() When:
ts

// ✅ Don't need streaming
const { text } = await generateText({ ... });

// ✅ Need complete response immediately
// ✅ Batch processing
// ✅ Saving to database

Use useChat() When:
ts

// ✅ Building chat UI
const { messages } = useChat({
  api: '/api/chat',  // Backend uses streamText
});

🎉 Summary (Memorize This)


    What: Server function that streams AI text in real-time

    Where: Backend API routes only

    When: Need live typing effect, chat, streaming responses

    Why: Better UX, users see progress, faster perceived speed

    How: const result = await streamText({ model, prompt })

    Returns: Stream object → convert with .toTextStreamResponse()

    vs generateText: Stream chunks vs wait for complete

    vs useChat: Manual streaming vs automatic UI management

🚀 Quick Checklist


    ✅ Use on server only

    ✅ Set maxTokens limit

    ✅ Handle errors

    ✅ Use .toTextStreamResponse() or .toUIMessageStreamResponse()

    ✅ Convert messages with convertToModelMessages()

    ✅ Use google('gemini-2.0-flash')

    ✅ Add optional callbacks if needed

    ✅ Test streaming in browser DevTools

💻 Copy-Paste Template
ts

// app/api/chat/route.ts
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { convertToModelMessages } from 'ai';

export async function POST(req: Request) {
  try {
    const { messages } = await req.json();

    const result = await streamText({
      model: google('gemini-2.0-flash'),
      messages: await convertToModelMessages(messages),
      system: 'You are a helpful assistant.',
      maxTokens: 2000,
      temperature: 0.7,
    });

    return result.toUIMessageStreamResponse();
  } catch (error) {
    return new Response('Error streaming', { status: 500 });
  }
}

// Frontend: Use with useChat()
'use client';
import { useChat } from '@ai-sdk/react';

export default function Chat() {
  const { messages, input, handleInputChange, handleSubmit } = useChat({
    api: '/api/chat',
  });

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

You're now a streamText() expert! 🎊

Use it for:


    ✅ Real-time chat applications

    ✅ Live AI assistants

    ✅ Streaming text generation

    ✅ Better user experience

    ✅ Multi-turn conversations

Perfect for creating ChatGPT-like experiences! 🚀 ::in mdx
