**User:**

generateText:: what does it do , explain everything about in just pain english  and where to use it and when to use it. give complete guide on it 0 to 100% thoroughly without leaving anything behind so i can be perfect  and also use gemini 3.0 flash modal as ai, vercel v6 only

---

**Assistant:**

I'll help you understand `generateText`. Let me search for it in the repository first.

Now let me read the main documentation for generateText:

Now let me search for information about Gemini and Vercel v6:

Now let me look at the package.json to understand the Vercel AI SDK version:

Let me find the actual AI SDK core source code:

Perfect! Now let me read the actual generateText source code:

Let me get more details about generateText parameters:

Now let me check a complete example with Gemini 3 Flash:

Perfect! Now I have all the information. Let me create a comprehensive guide for you:

---

# 🚀 Complete Guide to `generateText` - From 0 to 100%

## **WHAT IS `generateText`?**

`generateText` is a **core function** from the Vercel AI SDK (version 6.0.3) that generates text responses from Large Language Models (LLMs) **without streaming**. It's a **synchronous, promise-based** function that:

- Takes a prompt as input
- Sends it to an AI model
- Waits for the **complete response**
- Returns the full text at once

Think of it like sending a message and waiting for the entire reply before showing it to the user.

---

## **PLAIN ENGLISH EXPLANATION**

**What happens internally:**

1. You call `generateText()` with your question/prompt
2. The AI SDK packages your message with the model settings
3. It sends everything to the AI provider (Google, OpenAI, etc.)
4. The model thinks and generates the complete answer
5. You get back the full response object with text, token usage, finish reason, and more
6. Your code continues executing with the complete result

**Key principle:** Unlike `streamText` (which sends chunks as they're generated), `generateText` waits for the ENTIRE response before returning.

---

## **BASIC SYNTAX**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google'; // For Gemini

const { text } = await generateText({
  model: google('gemini-3-flash-preview'),
  prompt: 'Why is the sky blue?',
});

console.log(text);
```

---

## **USING GEMINI 3.0 FLASH (What You Asked For)**

### **Setup**

```bash
# Install required packages
pnpm install ai @ai-sdk/google
```

### **Basic Example with Gemini 3 Flash**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

const result = await generateText({
  model: google('gemini-3-flash-preview'),
  prompt: 'Explain quantum computing in simple terms',
});

console.log('Generated Text:', result.text);
console.log('Tokens Used:', result.usage);
console.log('Finish Reason:', result.finishReason);
```

### **With System Prompt (Instructions)**

```typescript
const result = await generateText({
  model: google('gemini-3-flash-preview'),
  system: 'You are a professional technical writer. Be clear, concise, and accurate.',
  prompt: 'What is React.js?',
});
```

### **With Enhanced Reasoning (Thinking Mode)**

Gemini 3 models can use enhanced reasoning:

```typescript
import { google, GoogleGenerativeAIProviderOptions } from '@ai-sdk/google';
import { generateText } from 'ai';

const result = await generateText({
  model: google('gemini-3-flash-preview'),
  prompt: 'What is the sum of the first 10 prime numbers?',
  providerOptions: {
    google: {
      thinkingConfig: {
        includeThoughts: true,
        thinkingLevel: 'low', // Options: 'minimal', 'low', 'medium', 'high' for Flash
      },
    } satisfies GoogleGenerativeAIProviderOptions,
  },
});

console.log('Result:', result.text);
console.log('Reasoning:', result.reasoning); // Model's thinking process
console.log('Reasoning Text:', result.reasoningText);
```

---

## **VERCEL AI SDK V6 SPECIFIC USAGE**

The current version is **6.0.3**. Key changes:

### **V6-Specific Features:**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

const result = await generateText({
  model: google('gemini-3-flash-preview'),
  
  // Standard inputs
  prompt: 'Your question here',
  system: 'Optional system instructions',
  
  // Model control
  temperature: 0.7, // 0-1: lower = more deterministic
  maxOutputTokens: 1000, // Max tokens to generate
  topP: 0.9, // Nucleus sampling
  topK: 40, // Top-K sampling
  
  // Tool calling (for multi-step workflows)
  tools: {
    getWeather: {
      description: 'Get weather for a location',
      inputSchema: z.object({
        location: z.string(),
      }),
      execute: async ({ location }) => ({ temp: 72 }),
    },
  },
  
  // Multi-step control
  stopWhen: stepCountIs(1), // How many steps to execute
  
  // Callbacks
  onFinish({ text, usage, finishReason }) {
    console.log('Generation complete');
    console.log('Total tokens:', usage.totalTokens);
  },
});

console.log(result.text);
```

---

## **COMPLETE RESULT OBJECT**

Here's EVERYTHING you get back:

```typescript
const result = await generateText({
  model: google('gemini-3-flash-preview'),
  prompt: 'Hello',
});

// The result object contains:
{
  // Main outputs
  text: string;                          // The generated text
  content: ContentPart[];                // Raw content parts
  reasoning: ReasoningOutput[];          // Thinking process (if enabled)
  reasoningText: string | undefined;     // Text version of reasoning
  
  // Files
  files: GeneratedFile[];                // Any files generated
  
  // Tool calling
  toolCalls: TypedToolCall[];            // Tools the model called
  toolResults: TypedToolResult[];        // Results of tool calls
  
  // Status info
  finishReason: FinishReason;            // 'stop' | 'length' | 'tool-calls' | etc
  rawFinishReason: string | undefined;   // Provider's raw reason
  
  // Token usage
  usage: {
    completionTokens: number;            // Tokens generated
    promptTokens: number;                // Tokens in prompt
    totalTokens: number;                 // Sum of above
  };
  totalUsage: LanguageModelUsage;        // Total across all steps
  
  // Metadata
  warnings: CallWarning[] | undefined;
  providerMetadata: ProviderMetadata;
  
  // Response details
  response: {
    headers: Record<string, string>;
    body?: unknown;
    messages: ResponseMessage[];         // Full conversation
  };
  
  // Multi-step details
  steps: StepResult[];                   // Info about each step
  sources: Source[];                     // References used
  
  // Structured output
  output: InferCompleteOutput;           // If using output schema
}
```

---

## **WHEN TO USE `generateText`**

### ✅ **PERFECT FOR:**

1. **Simple Q&A / Completions**
   ```typescript
   const { text } = await generateText({
     model: google('gemini-3-flash-preview'),
     prompt: 'What is machine learning?',
   });
   ```

2. **Content Generation** (email, blog, etc.)
   ```typescript
   const { text } = await generateText({
     model: google('gemini-3-flash-preview'),
     system: 'You are a professional copywriter',
     prompt: 'Write a product description for sneakers',
   });
   ```

3. **Data Summarization**
   ```typescript
   const { text } = await generateText({
     model: google('gemini-3-flash-preview'),
     prompt: `Summarize this article in 3 sentences: ${article}`,
   });
   ```

4. **Agents with Tool Calling**
   ```typescript
   const result = await generateText({
     model: google('gemini-3-flash-preview'),
     prompt: 'What is the weather in San Francisco?',
     tools: { /* your tools */ },
   });
   ```

5. **Server-side Processing**
   - API routes, backend jobs, one-time processing
   - No need for real-time streaming

### ❌ **NOT FOR:**

1. **Chat Interfaces** - Use `streamText()` instead (better UX with real-time updates)
2. **Long-running Tasks** - User has to wait for full response
3. **Real-time Interaction** - Use streaming for immediate feedback

---

## **NEXT.JS API ROUTE EXAMPLE (V6)**

```typescript
// app/api/generate/route.ts
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const { prompt, system } = await req.json();

  const { text, usage } = await generateText({
    model: google('gemini-3-flash-preview'),
    system: system || 'You are a helpful assistant',
    prompt,
  });

  return Response.json({
    text,
    tokensUsed: usage.totalTokens,
  });
}
```

---

## **REACT SERVER COMPONENT EXAMPLE**

```typescript
// app/actions.ts
'use server';

import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

export async function getAnswer(question: string) {
  const { text, finishReason, usage } = await generateText({
    model: google('gemini-3-flash-preview'),
    prompt: question,
  });

  return { text, finishReason, usage };
}
```

```typescript
// app/page.tsx
'use client';

import { useState } from 'react';
import { getAnswer } from './actions';

export default function Home() {
  const [result, setResult] = useState('');

  return (
    <div>
      <button
        onClick={async () => {
          const { text } = await getAnswer('Why is the sky blue?');
          setResult(text);
        }}
      >
        Ask
      </button>
      <div>{result}</div>
    </div>
  );
}
```

---

## **NODE.JS EXAMPLE**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

async function main() {
  const result = await generateText({
    model: google('gemini-3-flash-preview'),
    prompt: 'Generate a funny joke about programming',
    maxOutputTokens: 200,
  });

  console.log('Generated Joke:');
  console.log(result.text);
  console.log('\nTokens used:', result.usage.totalTokens);
  console.log('Finish reason:', result.finishReason);
}

main();
```

---

## **ADVANCED: MULTI-STEP GENERATION WITH TOOLS**

```typescript
import { generateText, tool, stepCountIs } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

const result = await generateText({
  model: google('gemini-3-flash-preview'),
  prompt: 'Get the weather in NYC and LA, then tell me which is warmer',
  
  tools: {
    getWeather: tool({
      description: 'Get weather for a location',
      inputSchema: z.object({
        location: z.string().describe('City name'),
      }),
      execute: async ({ location }) => ({
        temperature: Math.floor(Math.random() * 80) + 50,
        location,
      }),
    }),
  },
  
  // Allow multiple tool calls
  stopWhen: stepCountIs(5),
});

console.log('Analysis:', result.text);
console.log('Tool calls made:', result.toolCalls.length);
console.log('Tool results:', result.toolResults);
```

---

## **ERROR HANDLING**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

try {
  const { text } = await generateText({
    model: google('gemini-3-flash-preview'),
    prompt: 'Hello',
    maxRetries: 0, // Disable automatic retries
  });
} catch (error) {
  if (error instanceof Error) {
    console.error('Generation failed:', error.message);
    // Handle specific error types
  }
}
```

---

## **GEMINI 3 FLASH vs GEMINI 3 PRO**

| Feature | Gemini 3 Flash | Gemini 3 Pro |
|---------|---|---|
| **Speed** | ⚡ Very fast | Fast |
| **Cost** | 💰 Cheaper | More expensive |
| **Reasoning** | Good | Excellent |
| **Thinking Levels** | 4 levels (minimal, low, medium, high) | 2 levels (low, high) |
| **Use Case** | Quick queries, chat | Complex reasoning, analysis |
| **Model String** | `gemini-3-flash-preview` | `gemini-3-pro-preview` |

---

## **COMPLETE WORKING EXAMPLE**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

async function analyzeCode() {
  const codeSnippet = `
    function fibonacci(n) {
      if (n <= 1) return n;
      return fibonacci(n - 1) + fibonacci(n - 2);
    }
  `;

  const result = await generateText({
    model: google('gemini-3-flash-preview'),
    system: 'You are an expert code reviewer. Be concise and helpful.',
    prompt: `Review this code and suggest improvements:\n\n${codeSnippet}`,
    temperature: 0.5, // Deterministic
    maxOutputTokens: 500,
  });

  console.log('=== CODE REVIEW ===');
  console.log(result.text);
  console.log('\n=== STATS ===');
  console.log(`Prompt tokens: ${result.usage.promptTokens}`);
  console.log(`Completion tokens: ${result.usage.completionTokens}`);
  console.log(`Total tokens: ${result.usage.totalTokens}`);
  console.log(`Finish reason: ${result.finishReason}`);

  // Save to log
  console.log('\nFull metadata available:', {
    warnings: result.warnings,
    providerMetadata: result.providerMetadata,
  });
}

analyzeCode();
```

---

## **KEY DIFFERENCES: `generateText` vs `streamText`**

| Feature | `generateText` | `streamText` |
|---------|---|---|
| **Return Type** | Complete text | AsyncIterable of chunks |
| **User Experience** | Wait for full answer | Real-time updates |
| **Best For** | Backend, servers | Chat, UI streaming |
| **Usage** | `await result.text` | `for await (const chunk of result.textStream)` |
| **Error Handling** | Throws on error | Errors in stream |
| **API Route** | Return JSON | Return stream response |

---

## **INSTALLATION CHECKLIST**

```bash
# 1. Install main SDK
pnpm install ai

# 2. Install Google provider for Gemini
pnpm install @ai-sdk/google

# 3. Set API key
export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"

# 4. Ready to use!
```

---

## **QUICK START TEMPLATE**

```typescript
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

async function main() {
  const result = await generateText({
    // Model selection
    model: google('gemini-3-flash-preview'),
    
    // Input (choose one)
    prompt: 'Your question here',
    // OR
    // messages: [...], // For chat-style input
    
    // Optional system instructions
    system: 'Optional: Add system prompt here',
    
    // Optional controls
    temperature: 0.7,
    maxOutputTokens: 1000,
  });

  // Access the results
  console.log('Response:', result.text);
  console.log('Tokens:', result.usage.totalTokens);
  console.log('Finish Reason:', result.finishReason);
}

main();
```

---

## **SUMMARY**

- **`generateText`** = Wait-for-complete-response AI function
- **Use it for**: Background tasks, summaries, content generation, agents
- **Don't use for**: Chat UIs (use `streamText` instead)
- **V6.0.3**: Latest Vercel AI SDK version
- **Gemini 3 Flash**: Fast, cost-effective, perfect for most tasks
- **Result object**: Contains text, tokens, metadata, tool calls, and more

You're now a `generateText` expert! 🎉
