

---

# 🎯 COMPLETE GUIDE: AI Middleware Functions (0-100%)

## **Plain English Overview**

Middleware functions are like **interceptors** or **filters** that sit between your code and the AI model. They process requests going in and responses coming out, allowing you to customize how the AI behaves without modifying the core model itself.

Think of it like a security checkpoint: requests go through middleware → get modified → then reach the model. Responses come back → get modified by middleware → then reach your code.

---

## **1️⃣ extractReasoningMiddleware** 

### **What It Does (Simple Explanation)**

This middleware **extracts the AI's thinking process** from the response and separates it from the final answer.

When advanced AI models generate responses, they often include "reasoning" - the internal thought process wrapped in XML tags. This middleware:
1. Finds XML-tagged reasoning sections (e.g., `<think>...</think>`)
2. **Removes** them from the main text
3. **Stores** them separately as a `reasoningText` property

**Real-world analogy:** Imagine a student shows their work and answer on an exam. The middleware extracts the working/scratch work and puts it in a separate folder while keeping the final answer visible.

### **Parameters Explained**

```typescript
extractReasoningMiddleware({
  tagName: 'think',           // ← XML tag name to look for
  separator: '\n',             // ← What goes between multiple reasoning sections
  startWithReasoning: false    // ← If response ALWAYS starts with reasoning
})
```

| Parameter | Type | Required | What It Does |
|-----------|------|----------|------------|
| **tagName** | `string` | ✅ Yes | Name of the XML tag containing reasoning (e.g., `think`, `reasoning`, `analysis`) |
| **separator** | `string` | ❌ No | Divider between multiple reasoning sections. Default: `\n` (newline) |
| **startWithReasoning** | `boolean` | ❌ No | Set `true` if response always starts with reasoning (optimization). Default: `false` |

### **When to Use It**

✅ **Use when:**
- Working with reasoning models (Claude 3.5 with extended thinking, OpenAI o1, Mistral Deep Research)
- You want to display the AI's reasoning process separately from the answer
- You need to analyze or log the model's thought process
- Building educational or debugging tools where reasoning matters
- Using models that natively support thinking tags

❌ **Don't use when:**
- Working with basic models that don't output reasoning
- You don't care about the thinking process
- Using models that don't support XML-tagged reasoning

### **Real Example**

```typescript
import { generateText, wrapLanguageModel, extractReasoningMiddleware } from 'ai';
import { mistral } from '@ai-sdk/mistral';

async function solveComplexProblem() {
  const result = await generateText({
    model: wrapLanguageModel({
      model: mistral('magistral-medium-2506'),
      middleware: extractReasoningMiddleware({
        tagName: 'think',  // Mistral uses <think> tags
        separator: '\n\n',  // Double newline between sections
      }),
    }),
    prompt: 'If a train travels 60 mph for 2 hours, how far does it go?',
  });

  console.log('REASONING:');
  console.log(result.reasoningText);  // "Let me calculate... 60 * 2 = 120 miles"
  
  console.log('\nFINAL ANSWER:');
  console.log(result.text);           // "The train travels 120 miles."
}
```

### **What You Get Back**

```typescript
{
  text: "The train travels 120 miles.",
  reasoningText: "Let me calculate step by step...\n60 miles per hour × 2 hours = 120 miles",
  usage: { inputTokens: 50, outputTokens: 150 }
}
```

### **How It Works Internally**

1. **For streaming responses:** Uses a `TransformStream` to intercept chunks
2. **For non-streaming responses:** Uses regex to find and extract XML tags
3. **Handles edge cases:** Multiple reasoning sections, reasoning at different positions
4. **Maintains order:** Ensures reasoning and text stream in proper sequence

---

## **2️⃣ simulateStreamingMiddleware**

### **What It Does (Simple Explanation)**

This middleware **makes non-streaming models behave like streaming models**.

Some AI models can only return a complete response all at once (non-streaming). This middleware fakes streaming by breaking that complete response into chunks, so you can process it like a real stream.

**Real-world analogy:** Imagine downloading a movie file. If you can only download it whole, this middleware pretends to stream it by breaking it into pieces during playback.

### **Parameters Explained**

```typescript
simulateStreamingMiddleware()  // ← NO parameters needed!
```

This is the simplest middleware - it takes **zero configuration**.

### **When to Use It**

✅ **Use when:**
- Using non-streaming models but need streaming interface
- Want consistent code that works with both streaming/non-streaming models
- Building real-time UI that expects streaming chunks
- Need to process responses incrementally
- Using models that don't support streaming natively

❌ **Don't use when:**
- Model already supports native streaming
- Performance is critical (streaming is faster than simulating)
- You need actual real-time latency benefits
- Response is huge (entire response must load before chunking)

### **Real Example**

```typescript
import { streamText, wrapLanguageModel, simulateStreamingMiddleware } from 'ai';
import { openai } from '@ai-sdk/openai';

async function getStreamingResponse() {
  const result = streamText({
    model: wrapLanguageModel({
      model: openai('gpt-4o'),  // This model supports streaming
      middleware: simulateStreamingMiddleware(),  // But we simulate it anyway
    }),
    prompt: 'What cities are in the United States?',
  });

  // Process streaming chunks
  for await (const chunk of result.textStream) {
    console.log(chunk);  // Gets one word/phrase at a time (simulated)
    // In a React app, you could update UI with each chunk
  }
}
```

### **What It Does Behind the Scenes**

The middleware:
1. Calls the model's `doGenerate()` method (gets complete response)
2. Creates a `ReadableStream` 
3. Parses the complete response into proper chunk types:
   - `stream-start` (beginning signal)
   - `text-delta` (text chunks)
   - `reasoning-start/delta/end` (if reasoning exists)
   - `finish` (end signal with metadata)
4. Returns the stream as if it was native streaming

### **Practical Comparison**

```typescript
// WITHOUT simulateStreamingMiddleware (waits for everything)
const result = await streamText({
  model: openai('gpt-4o'),
  prompt: 'Hello',
});
// ⏳ Waits... waits... then everything arrives at once

// WITH simulateStreamingMiddleware (chunks arrive gradually)
const result = streamText({
  model: wrapLanguageModel({
    model: openai('gpt-4o'),
    middleware: simulateStreamingMiddleware(),
  }),
  prompt: 'Hello',
});
// ✅ Chunks arrive one by one
```

---

## **3️⃣ defaultSettingsMiddleware**

### **What It Does (Simple Explanation)**

This middleware **sets default parameters** that apply to every model call automatically.

Instead of passing the same settings every time you use a model, you set them once as defaults. When you call the model without specific settings, it uses these defaults. If you DO provide settings, yours override the defaults.

**Real-world analogy:** Like setting your printer's default to "black and white, double-sided." Every print uses those defaults unless you change them for that specific print job.

### **Parameters Explained**

```typescript
defaultSettingsMiddleware({
  settings: {
    temperature: 0.7,           // ← How creative (0=deterministic, 2=random)
    maxOutputTokens: 1000,      // ← Max response length
    topP: 0.9,                  // ← Token diversity
    topK: 50,                   // ← Token selection pool
    presencePenalty: 0,         // ← Penalize topic repetition
    frequencyPenalty: 0,        // ← Penalize word repetition
    stopSequences: ['\n---'],   // ← Stop generation at these
    seed: 42,                   // ← Make reproducible
    responseFormat: 'json',     // ← Force JSON output
    tools: [],                  // ← Available tools
    headers: {},                // ← Custom HTTP headers
    providerOptions: {}         // ← Provider-specific settings
  }
})
```

| Setting | Range | What It Does |
|---------|-------|------------|
| **temperature** | 0-2 | Higher = more creative/random. 0 = always same answer. 2 = wild randomness. |
| **maxOutputTokens** | 1-∞ | Maximum words/tokens in response. ~4 chars per token. |
| **topP** | 0-1 | Only consider top P% of likely tokens. More focused. |
| **topK** | 1-∞ | Only consider top K most likely tokens. More diverse if higher. |
| **presencePenalty** | -2 to 2 | Penalizes introducing new topics. Keeps focused. |
| **frequencyPenalty** | -2 to 2 | Penalizes repeating words. Better variety. |
| **stopSequences** | string[] | Stop generating when these strings appear. |
| **seed** | number | Same seed = same output (if model supports it). |
| **responseFormat** | string | Force specific format (e.g., 'json') |

### **When to Use It**

✅ **Use when:**
- Want consistent behavior across all model calls
- Need to enforce company-wide AI parameters
- Setting up a shared model for multiple features
- Want to change behavior globally without code changes
- Building reusable model configs

❌ **Don't use when:**
- Each call needs different settings
- You're already passing parameters manually
- Settings are dynamic/user-specific

### **Real Example 1: Consistent Tone**

```typescript
import { generateText, wrapLanguageModel, defaultSettingsMiddleware } from 'ai';
import { openai } from '@ai-sdk/openai';

// Create a model for professional customer service
const professionalModel = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: defaultSettingsMiddleware({
    settings: {
      temperature: 0.3,         // Professional = less creative
      maxOutputTokens: 500,     // Keep responses concise
      presencePenalty: 0.5,     // Avoid repeating topics
    },
  }),
});

// Use it - settings apply automatically
const response1 = await generateText({
  model: professionalModel,
  prompt: 'How do I reset my password?',
  // temperature: 0.3 applied automatically ✅
});

const response2 = await generateText({
  model: professionalModel,
  prompt: 'Tell me about your features',
  // temperature: 0.3 applied automatically ✅
});

// Override default if needed
const response3 = await generateText({
  model: professionalModel,
  prompt: 'Write something creative',
  temperature: 0.9,  // ← Overrides the 0.3 default
});
```

### **Real Example 2: Provider-Specific Settings**

```typescript
import { gateway } from '@ai-sdk/gateway';
import { defaultSettingsMiddleware } from 'ai';

// Set Claude-specific defaults through gateway
const modelWithDefaults = wrapLanguageModel({
  model: gateway('anthropic/claude-sonnet-4.5'),
  middleware: defaultSettingsMiddleware({
    settings: {
      temperature: 0.7,
      providerOptions: {
        anthropic: {
          thinkingBudget: 10000,  // Claude extended thinking
        },
      },
    },
  }),
});
```

### **Real Example 3: JSON-Only Model**

```typescript
// Ensure model always returns JSON
const jsonModel = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: defaultSettingsMiddleware({
    settings: {
      responseFormat: 'json',
      maxOutputTokens: 2000,
    },
  }),
});

const user = await generateText({
  model: jsonModel,
  prompt: 'Generate a user profile in JSON format',
  // Always gets JSON response ✅
});
```

### **How Settings Merge Works**

```typescript
// Middleware defaults:
const defaults = {
  temperature: 0.7,
  maxOutputTokens: 1000,
  topP: 0.9,
};

// Your call params:
const params = {
  temperature: 0.5,  // Override
  seed: 42,          // New setting
};

// Result: Defaults merge with your params
// Your params win on conflicts ✅
const final = {
  temperature: 0.5,     // ← Your value used
  maxOutputTokens: 1000, // ← Default used
  topP: 0.9,            // ← Default used
  seed: 42,             // ← Your value added
};
```

---

## **🚀 COMBINING ALL THREE MIDDLEWARE**

### **Advanced Example: Professional AI Assistant**

```typescript
import { 
  generateText, 
  wrapLanguageModel, 
  extractReasoningMiddleware,
  defaultSettingsMiddleware 
} from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

// Create a professional assistant with reasoning
const professionalAssistant = wrapLanguageModel({
  model: anthropic('claude-3-7-sonnet-20250219'),
  middleware: [
    // First: extract reasoning from response
    extractReasoningMiddleware({
      tagName: 'thinking',
      separator: '\n\n',
    }),
    // Second: apply default professional settings
    defaultSettingsMiddleware({
      settings: {
        temperature: 0.3,
        maxOutputTokens: 2000,
      },
    }),
  ],
});

// Usage
const response = await generateText({
  model: professionalAssistant,
  prompt: 'Analyze this business problem step by step',
});

console.log('What the AI thought:');
console.log(response.reasoningText);

console.log('\nFinal Analysis:');
console.log(response.text);
```

---

## **🔧 SETUP GUIDE: Complete Configuration**

### **Step 1: Install Dependencies**

```bash
npm install ai @ai-sdk/openai dotenv
```

### **Step 2: Set Environment Variables**

```bash
# .env file
OPENAI_API_KEY=sk-...
```

### **Step 3: Create Wrapped Models**

```typescript
import { openai } from '@ai-sdk/openai';
import { 
  wrapLanguageModel,
  extractReasoningMiddleware,
  simulateStreamingMiddleware,
  defaultSettingsMiddleware 
} from 'ai';

// Model 1: Reasoning extraction
export const reasoningModel = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: extractReasoningMiddleware({ tagName: 'thinking' }),
});

// Model 2: Simulate streaming
export const streamingModel = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: simulateStreamingMiddleware(),
});

// Model 3: Default settings
export const defaultModel = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: defaultSettingsMiddleware({
    settings: { temperature: 0.5 },
  }),
});

// Model 4: All combined
export const powerModel = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: [
    extractReasoningMiddleware({ tagName: 'thinking' }),
    defaultSettingsMiddleware({ settings: { temperature: 0.7 } }),
  ],
});
```

---

## **🎯 PRACTICAL USE CASES**

### **Use Case 1: Customer Support Bot**
```typescript
const supportBot = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: defaultSettingsMiddleware({
    settings: {
      temperature: 0.3,      // Professional tone
      maxOutputTokens: 300,  // Keep responses short
    },
  }),
});
```

### **Use Case 2: Content Generator**
```typescript
const contentGenerator = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: defaultSettingsMiddleware({
    settings: {
      temperature: 0.8,      // More creative
      maxOutputTokens: 2000, // Allow longer content
    },
  }),
});
```

### **Use Case 3: Research Assistant**
```typescript
const researchAssistant = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: [
    extractReasoningMiddleware({ 
      tagName: 'analysis',
      separator: '\n---\n' 
    }),
    defaultSettingsMiddleware({
      settings: {
        temperature: 0.5,
        maxOutputTokens: 4000,
      },
    }),
  ],
});
```

### **Use Case 4: Real-time Chat UI**
```typescript
const realtimeChat = wrapLanguageModel({
  model: openai('gpt-4o'),
  middleware: simulateStreamingMiddleware(),
});

// Use in React
const { fullStream } = streamText({
  model: realtimeChat,
  prompt: userMessage,
});

for await (const chunk of fullStream) {
  // Update UI in real-time with each chunk
  setMessages(prev => [...prev, chunk]);
}
```

---

## **⚡ PERFORMANCE NOTES**

| Middleware | Impact | When Slower |
|-----------|--------|-----------|
| **extractReasoningMiddleware** | Minimal | With very long reasoning sections |
| **simulateStreamingMiddleware** | Medium | Large responses (must load all before streaming) |
| **defaultSettingsMiddleware** | Minimal | None - just object merging |

**Performance tip:** Use native streaming models when possible instead of `simulateStreamingMiddleware`.

---

## **✅ KEY TAKEAWAYS**

1. **extractReasoningMiddleware** = Separates AI thinking from answers
2. **simulateStreamingMiddleware** = Fakes streaming on non-streaming models  
3. **defaultSettingsMiddleware** = Applies default parameters automatically
4. **Stack them** = Use multiple middleware on one model
5. **Use with wrapLanguageModel()** = The gateway to apply middleware

---

## **🔗 GEMINI 3.0 FLASH + VERCEL v6 SETUP**

### **Using Gemini 3.0 Flash with middleware:**

```typescript
import { google } from '@ai-sdk/google';
import { generateText, wrapLanguageModel, defaultSettingsMiddleware } from 'ai';

const geminiModel = wrapLanguageModel({
  model: google('gemini-2.0-flash'),
  middleware: defaultSettingsMiddleware({
    settings: {
      temperature: 0.7,
      maxOutputTokens: 1000,
    },
  }),
});

// In a Vercel Edge Function or API route
export default async function handler(req, res) {
  const result = await generateText({
    model: geminiModel,
    prompt: req.body.prompt,
  });
  
  res.status(200).json({ text: result.text });
}
```

---

This is the **complete 0-100% guide**. You now understand what each middleware does, when to use them, and how to implement them in production! 🚀
