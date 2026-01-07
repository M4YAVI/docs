
---

# **`wrapLanguageModel()` - Complete Guide (0-100%)**

## **Plain English Explanation**

`wrapLanguageModel()` is a function that lets you **wrap/enhance** a language model with **middleware** - think of it like adding a filter or processor that intercepts and modifies how your AI model works. It allows you to add features like logging, caching, RAG (Retrieval Augmented Generation), reasoning extraction, and more **without modifying the original model code**.

---

## **What Does It Do? (In Detail)**

### **Core Purpose**
It takes three things:
1. **A language model** (like OpenAI's GPT-4)
2. **One or more middleware functions** (custom logic)
3. **Optional overrides** (custom model ID or provider ID)

...and returns a **new wrapped model** that behaves like the original but with your middleware applied.

### **The Three Middleware Hooks**

When you write middleware, you can intercept model calls at 3 points:

#### **1. `transformParams`** (BEFORE the call)
Modifies the input parameters before they reach the model.
```typescript
transformParams: async ({ params, type, model }) => {
  // type is either 'generate' or 'stream'
  // Modify params here - add system prompts, inject context, etc.
  return modifiedParams;
}
```
**Use case**: RAG (add context to prompts), modify temperature dynamically, add system instructions

#### **2. `wrapGenerate`** (Around non-streaming calls)
Intercepts and wraps the generate operation.
```typescript
wrapGenerate: async ({ doGenerate, params, model }) => {
  // Log, cache, or monitor the call
  const result = await doGenerate();
  // Process the result
  return result;
}
```
**Use case**: Caching results, logging API calls, adding monitoring

#### **3. `wrapStream`** (Around streaming calls)
Intercepts and wraps the streaming operation.
```typescript
wrapStream: async ({ doStream, params, model }) => {
  const { stream, ...rest } = await doStream();
  // Transform the stream
  return {
    stream: stream.pipeThrough(customTransform),
    ...rest,
  };
}
```
**Use case**: Logging streamed text in real-time, extracting reasoning, transforming tokens

---

## **How to Use It: Step-by-Step**

### **Step 1: Import the Function**
```typescript
import { wrapLanguageModel } from 'ai';
import { google } from '@ai-sdk/google'; // Using Gemini 3.0 Flash
```

### **Step 2: Create Your Middleware**
```typescript
import { LanguageModelV3Middleware } from '@ai-sdk/provider';

const myMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3', // Required!
  
  transformParams: async ({ params, type }) => {
    console.log(`${type} called with:`, params);
    return params;
  },
  
  wrapGenerate: async ({ doGenerate, params }) => {
    console.log('Starting generate...');
    const result = await doGenerate();
    console.log('Generate finished');
    return result;
  },
};
```

### **Step 3: Wrap the Model**
```typescript
const wrappedModel = wrapLanguageModel({
  model: google('gemini-2.0-flash'), // Gemini 3.0 Flash
  middleware: myMiddleware,
  // Optional:
  modelId: 'custom-id', // Override model ID
  providerId: 'custom-provider', // Override provider
});
```

### **Step 4: Use the Wrapped Model Everywhere**
```typescript
import { generateText, streamText } from 'ai';

// Use in generateText
const result = await generateText({
  model: wrappedModel,
  prompt: 'What is AI?',
});

// Use in streamText
const stream = await streamText({
  model: wrappedModel,
  prompt: 'Explain quantum computing',
});

for await (const chunk of stream.textStream) {
  process.stdout.write(chunk);
}
```

---

## **Real-World Examples**

### **Example 1: Logging Middleware**
```typescript
import { LanguageModelV3Middleware } from '@ai-sdk/provider';

const loggingMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3',
  
  wrapGenerate: async ({ doGenerate, params }) => {
    console.log('🔵 API Call Starting');
    console.log('Prompt:', params.prompt);
    console.log('Tools:', params.tools?.length ?? 0);
    
    const start = Date.now();
    const result = await doGenerate();
    const duration = Date.now() - start;
    
    console.log('✅ API Call Finished');
    console.log('Duration:', duration, 'ms');
    console.log('Tokens:', result.usage);
    
    return result;
  },
  
  wrapStream: async ({ doStream, params }) => {
    console.log('🔵 Streaming Started');
    
    const { stream, ...rest } = await doStream();
    let tokenCount = 0;
    
    const transformStream = new TransformStream({
      transform(chunk, controller) {
        if (chunk.type === 'text-delta') {
          tokenCount++;
          console.log('Token:', chunk.delta);
        }
        controller.enqueue(chunk);
      },
      flush() {
        console.log('✅ Stream Finished. Total tokens:', tokenCount);
      },
    });
    
    return {
      stream: stream.pipeThrough(transformStream),
      ...rest,
    };
  },
};

// Usage
const model = wrapLanguageModel({
  model: google('gemini-2.0-flash'),
  middleware: loggingMiddleware,
});
```

### **Example 2: RAG (Retrieval Augmented Generation) Middleware**
```typescript
const ragMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3',
  
  transformParams: async ({ params }) => {
    // Extract the last user message
    const lastMessage = params.prompt[params.prompt.length - 1];
    
    if (lastMessage?.role === 'user') {
      // Simulate fetching relevant documents
      const relevantDocs = await searchDatabase(lastMessage.content);
      
      // Inject the context
      const context = relevantDocs
        .map(doc => `Document: ${doc.title}\n${doc.content}`)
        .join('\n\n');
      
      // Modify the last message
      const modifiedPrompt = [...params.prompt];
      modifiedPrompt[modifiedPrompt.length - 1] = {
        role: 'user',
        content: `Use this context to answer:\n${context}\n\nQuestion: ${lastMessage.content}`,
      };
      
      return { ...params, prompt: modifiedPrompt };
    }
    
    return params;
  },
};

async function searchDatabase(query: string) {
  // Simulate database search
  return [
    { title: 'Doc 1', content: 'Relevant content...' },
    { title: 'Doc 2', content: 'More context...' },
  ];
}
```

### **Example 3: Caching Middleware**
```typescript
const cacheMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3',
  
  wrapGenerate: async ({ doGenerate, params }) => {
    // Create a cache key from parameters
    const cacheKey = JSON.stringify({
      prompt: params.prompt,
      temperature: params.temperature,
      topP: params.topP,
    });
    
    // Check if result is cached
    if (cache.has(cacheKey)) {
      console.log('✅ Returning cached result');
      return cache.get(cacheKey);
    }
    
    // Call the model if not cached
    console.log('🔄 Calling model...');
    const result = await doGenerate();
    
    // Cache the result
    cache.set(cacheKey, result);
    
    return result;
  },
};

const cache = new Map();
```

### **Example 4: Using Built-in Middleware (Default Settings)**
```typescript
import { defaultSettingsMiddleware } from 'ai';

const model = wrapLanguageModel({
  model: google('gemini-2.0-flash'),
  middleware: defaultSettingsMiddleware({
    settings: {
      temperature: 0.7, // Always use this temperature
      maxOutputTokens: 2000,
      topK: 40,
      topP: 0.95,
    },
  }),
});

// Now every call will use these defaults
const result = await generateText({
  model,
  prompt: 'Hello',
  // temperature will be 0.7 unless explicitly overridden
});
```

---

## **When to Use `wrapLanguageModel`?**

### ✅ **USE IT WHEN YOU NEED:**

1. **Logging & Monitoring**
   - Track API usage, costs, response times
   - Debug model behavior

2. **Caching**
   - Speed up repeated queries
   - Reduce costs

3. **RAG (Context Injection)**
   - Add documents to prompts
   - Provide real-time information

4. **Input/Output Transformation**
   - Modify prompts before sending
   - Process responses after receiving

5. **Rate Limiting & Retries**
   - Control request rate
   - Implement exponential backoff

6. **Safety & Guardrails**
   - Filter inappropriate content
   - Validate outputs

7. **Reasoning Extraction** (for models like DeepSeek R1)
   - Extract `<think>` tags as reasoning
   ```typescript
   import { extractReasoningMiddleware } from 'ai';
   
   const model = wrapLanguageModel({
     model: google('gemini-2.0-flash'),
     middleware: extractReasoningMiddleware({ tagName: 'think' }),
   });
   ```

---

## **Multiple Middlewares (Chaining)**

You can stack multiple middlewares - they execute in order:

```typescript
const model = wrapLanguageModel({
  model: google('gemini-2.0-flash'),
  middleware: [
    firstMiddleware,    // Applied first (transforms input)
    secondMiddleware,   // Applied second
    thirdMiddleware,    // Applied last (closest to model)
  ],
});

// Execution flow: firstMiddleware → secondMiddleware → thirdMiddleware → model
```

---

## **Using with Vercel AI SDK v6**

Vercel AI SDK v6 fully supports `wrapLanguageModel`. Here's a complete example:

### **Installation**
```bash
npm install ai @ai-sdk/google
```

### **Complete App Example**
```typescript
import { generateText, streamText, wrapLanguageModel } from 'ai';
import { google } from '@ai-sdk/google';
import { LanguageModelV3Middleware } from '@ai-sdk/provider';

// 1. Create middleware
const loggingMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3',
  
  wrapGenerate: async ({ doGenerate, params }) => {
    console.time('api-call');
    const result = await doGenerate();
    console.timeEnd('api-call');
    return result;
  },
};

// 2. Wrap the model
const model = wrapLanguageModel({
  model: google('gemini-2.0-flash'),
  middleware: loggingMiddleware,
});

// 3. Use it
async function main() {
  // For generateText
  const text = await generateText({
    model,
    prompt: 'What is the capital of France?',
  });
  console.log(text.text);
  
  // For streamText
  const stream = streamText({
    model,
    prompt: 'Tell me about Python programming',
  });
  
  for await (const chunk of stream.textStream) {
    process.stdout.write(chunk);
  }
}

main();
```

### **In a Next.js Server Action (Vercel)**
```typescript
'use server';

import { generateText, wrapLanguageModel } from 'ai';
import { google } from '@ai-sdk/google';
import { LanguageModelV3Middleware } from '@ai-sdk/provider';

const cachingMiddleware: LanguageModelV3Middleware = {
  specificationVersion: 'v3',
  wrapGenerate: async ({ doGenerate, params }) => {
    // Implement caching logic
    const cacheKey = JSON.stringify(params.prompt);
    // ... cache implementation
    return doGenerate();
  },
};

const model = wrapLanguageModel({
  model: google('gemini-2.0-flash'),
  middleware: cachingMiddleware,
});

export async function askAI(prompt: string) {
  return generateText({
    model,
    prompt,
  });
}
```

---

## **Key Differences: When to Use What**

| Feature | `wrapLanguageModel` | `generateText`/`streamText` options |
|---------|-------------------|----------------------------------|
| **Global behavior** | ✅ Yes (applies to all calls) | ❌ Per-call only |
| **Reusable** | ✅ Yes | ❌ Per-call |
| **Complex logic** | ✅ Yes | ❌ Limited |
| **Middleware stacking** | ✅ Yes | ❌ No |
| **Access to raw streams** | ✅ Yes | ⚠️ Limited |

---

## **Middleware Properties Reference**

```typescript
interface LanguageModelV3Middleware {
  specificationVersion: 'v3'; // Always required
  
  // Transform parameters before sending to model
  transformParams?: async (options: {
    params: LanguageModelV3CallOptions;
    type: 'generate' | 'stream';
    model: LanguageModelV3;
  }) => Promise<LanguageModelV3CallOptions>;
  
  // Wrap generate (non-streaming) calls
  wrapGenerate?: async (options: {
    doGenerate: () => Promise<LanguageModelV3GenerateResult>;
    doStream: () => Promise<LanguageModelV3StreamResult>;
    params: LanguageModelV3CallOptions;
    model: LanguageModelV3;
  }) => Promise<LanguageModelV3GenerateResult>;
  
  // Wrap stream calls
  wrapStream?: async (options: {
    doGenerate: () => Promise<LanguageModelV3GenerateResult>;
    doStream: () => Promise<LanguageModelV3StreamResult>;
    params: LanguageModelV3CallOptions;
    model: LanguageModelV3;
  }) => Promise<LanguageModelV3StreamResult>;
  
  // Override provider name
  overrideProvider?: (options: { model: LanguageModelV3 }) => string;
  
  // Override model ID
  overrideModelId?: (options: { model: LanguageModelV3 }) => string;
  
  // Override supported URLs
  overrideSupportedUrls?: (options: { model: LanguageModelV3 }) => Record<string, RegExp[]>;
}
```

---

## **Best Practices**

1. **Always include `specificationVersion: 'v3'`** in middleware
2. **Keep middleware focused** - one concern per middleware
3. **Use array form for multiple middlewares** - cleaner and composable
4. **Handle errors gracefully** - don't let middleware crash the app
5. **Cache appropriately** - don't cache everything (dynamic content)
6. **Log strategically** - avoid logging sensitive data
7. **Test middleware independently** - before wrapping production models

---

## **Summary**

- **What**: A wrapper function that enhances language models with custom logic
- **How**: Takes a model + middleware(s) and returns an enhanced model
- **When**: Use for logging, caching, RAG, transformation, safety checks
- **Vercel v6**: Fully supported, works with `generateText` and `streamText`
- **Gemini 3.0 Flash**: Works perfectly with `google('gemini-2.0-flash')`
- **Power**: Reusable, composable, and keeps code clean and maintainable
