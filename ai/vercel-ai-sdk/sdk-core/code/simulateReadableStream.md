
---

# **simulateReadableStream() - Complete Guide (0 to 100%)**

## **What It Does (Plain English)**

`simulateReadableStream` creates a **fake stream** that slowly releases data to you piece-by-piece, with pauses in between. Think of it like a faucet that drips out water slowly instead of flowing all at once.

**Real-world analogy**: Imagine you have a list of 10 items to hand someone. Instead of giving them all 10 at once, you hand them item #1, wait 2 seconds, hand item #2, wait 2 seconds, and so on.

---

## **Core Concept**

```typescript
import { simulateReadableStream } from 'ai';

// Basic example - no delays
const stream = simulateReadableStream({
  chunks: ['Hello', ' ', 'World'],
});
```

This creates a stream that emits:
1. `'Hello'`
2. `' '`  
3. `'World'`

---

## **The Parameters Explained**

### **1. `chunks` (Required)**
- **What it is**: An array of values you want to emit
- **Type**: `T[]` (can be any data type)
- **Example**: `chunks: ['a', 'b', 'c']` or `chunks: [{id: 1}, {id: 2}]`

```typescript
// String chunks
simulateReadableStream({ chunks: ['Hello', ' ', 'World'] });

// Object chunks
simulateReadableStream({ 
  chunks: [
    { type: 'text-delta', text: 'Hi' },
    { type: 'text-delta', text: ' there' }
  ] 
});

// Number chunks
simulateReadableStream({ chunks: [1, 2, 3, 4, 5] });
```

---

### **2. `initialDelayInMs` (Optional)**
- **What it is**: How long (in milliseconds) to wait BEFORE emitting the first chunk
- **Default**: `0` (no delay)
- **Type**: `number | null`

```typescript
// Wait 1000ms (1 second) before first chunk
simulateReadableStream({
  chunks: ['Hello', ' ', 'World'],
  initialDelayInMs: 1000,
});

// No initial delay
simulateReadableStream({
  chunks: ['Hello', ' ', 'World'],
  initialDelayInMs: null,  // Skip initial delay
});

// 0ms delay (still goes through delay logic, but instantly)
simulateReadableStream({
  chunks: ['Hello', ' ', 'World'],
  initialDelayInMs: 0,  // Different from null - still async
});
```

---

### **3. `chunkDelayInMs` (Optional)**
- **What it is**: How long (in milliseconds) to wait BETWEEN each chunk after the first one
- **Default**: `0` (no delay)
- **Type**: `number | null`

```typescript
// Wait 500ms between each chunk
simulateReadableStream({
  chunks: ['Hello', ' ', 'World'],
  chunkDelayInMs: 500,
});
// Timeline:
// T=0ms:   'Hello'
// T=500ms: ' '
// T=1000ms: 'World'
```

---

## **Complete Parameter Breakdown**

```typescript
const stream = simulateReadableStream({
  chunks: ['a', 'b', 'c'],                    // ← What to emit
  initialDelayInMs: 1000,                      // ← Wait 1s before 'a'
  chunkDelayInMs: 500,                         // ← Wait 500ms before 'b' and 'c'
});
```

**Timeline example:**
- `T = 0ms`: Function is called
- `T = 1000ms`: First chunk `'a'` is emitted (waited for initialDelayInMs)
- `T = 1500ms`: Second chunk `'b'` is emitted (500ms after 'a')
- `T = 2000ms`: Third chunk `'c'` is emitted (500ms after 'b')
- `T = 2000ms+`: Stream closes

---

## **`null` vs `0` - Important Difference**

```typescript
// Option A: initialDelayInMs: 0
// Emits first chunk instantly BUT still waits async (goes through delay logic)
simulateReadableStream({
  chunks: ['a', 'b'],
  initialDelayInMs: 0,      // 0 milliseconds delay
});

// Option B: initialDelayInMs: null
// Emits first chunk WITHOUT any async delay (skips delay logic entirely)
simulateReadableStream({
  chunks: ['a', 'b'],
  initialDelayInMs: null,   // COMPLETELY skip delay
});
```

**In practice**: Use `null` when you want zero overhead, use `0` when you want async behavior.

---

## **Return Value**

Returns a `ReadableStream<T>` object that you can:
- **Consume with a reader**: `stream.getReader()`
- **Use with `for await...of`**: `for await (const chunk of stream) {}`
- **Pass to APIs expecting streams**

```typescript
const stream = simulateReadableStream({
  chunks: ['Hello', ' ', 'World'],
  chunkDelayInMs: 100,
});

// Method 1: Using reader
const reader = stream.getReader();
let done = false;
while (!done) {
  const { value, done: isDone } = await reader.read();
  console.log(value);
  done = isDone;
}

// Method 2: Using for-await loop
for await (const chunk of stream) {
  console.log(chunk);  // 'Hello', ' ', 'World' (with 100ms delays)
}
```

---

## **Where to Use It & When to Use It**

### **✅ USE FOR (Perfect Use Cases)**

1. **Testing Streaming Features**
   - Mock AI model responses that stream
   - Test how your app handles streaming data
   - Don't want to call real API

```typescript
import { streamText } from 'ai';
import { MockLanguageModelV3 } from 'ai/test';

const result = streamText({
  model: new MockLanguageModelV3({
    doStream: async () => ({
      stream: simulateReadableStream({
        chunks: [
          { type: 'text-start', id: 'text-1' },
          { type: 'text-delta', id: 'text-1', delta: 'Hello' },
          { type: 'text-delta', id: 'text-1', delta: ', world!' },
          { type: 'text-end', id: 'text-1' },
        ],
        chunkDelayInMs: 100,  // Simulate network delays
      }),
    }),
  }),
  prompt: 'Say hello!',
});
```

2. **Simulating Real-World Delays**
   - Network latency simulation
   - Show realistic loading states
   - Test timeout handling

```typescript
// Simulate slow network
simulateReadableStream({
  chunks: [...data],
  chunkDelayInMs: 500,  // 500ms per chunk (slow network)
});
```

3. **Development & Debugging**
   - Debug streaming UI without API calls
   - Prototype before API ready
   - Demo purposes

4. **Testing UI Message Streams (Next.js/Vercel)**

```typescript
// route.ts
import { simulateReadableStream } from 'ai';

export async function POST(req: Request) {
  return new Response(
    simulateReadableStream({
      initialDelayInMs: 1000,  // Delay before first chunk
      chunkDelayInMs: 300,     // Delay between chunks
      chunks: [
        `data: {"type":"start","messageId":"msg-123"}\n\n`,
        `data: {"type":"text-start","id":"text-1"}\n\n`,
        `data: {"type":"text-delta","id":"text-1","delta":"Hello"}\n\n`,
        `data: {"type":"text-delta","id":"text-1","delta":" world"}\n\n`,
        `data: {"type":"text-end","id":"text-1"}\n\n`,
      ],
    }),
  );
}
```

5. **Performance Testing**
   - Benchmark streaming handlers
   - Test backpressure handling
   - Memory leak detection

```typescript
// Benchmark stream processing
const stream = simulateReadableStream({
  chunks: Array(10000).fill('data chunk'),
  chunkDelayInMs: 1,  // Small delay to see real processing speed
});
```

---

## **Practical Examples**

### **Example 1: Basic Chat Response Testing**

```typescript
import { simulateReadableStream, streamText } from 'ai';
import { MockLanguageModelV3 } from 'ai/test';

// Test that your chat UI handles streaming correctly
test('should display streamed response', async () => {
  const result = streamText({
    model: new MockLanguageModelV3({
      doStream: async () => ({
        stream: simulateReadableStream({
          chunks: [
            { type: 'text-start', id: '1' },
            { type: 'text-delta', id: '1', delta: 'The answer is ' },
            { type: 'text-delta', id: '1', delta: '42' },
            { type: 'text-end', id: '1' },
            {
              type: 'finish',
              finishReason: 'stop',
              usage: { inputTokens: 5, outputTokens: 3 },
            },
          ],
          chunkDelayInMs: 50,  // Simulate typing speed
        }),
      }),
    }),
    prompt: 'What is the answer?',
  });

  let text = '';
  for await (const chunk of result.textStream) {
    text += chunk;
  }
  
  expect(text).toBe('The answer is 42');
});
```

---

### **Example 2: Next.js API Route with Simulated Delays**

```typescript
// app/api/chat/route.ts
import { simulateReadableStream } from 'ai';

export async function POST(req: Request) {
  return new Response(
    simulateReadableStream({
      initialDelayInMs: 500,    // Initial delay before response
      chunkDelayInMs: 200,      // Delay between words (simulate typing)
      chunks: [
        'Hello',
        ', ',
        'this',
        ' ',
        'is',
        ' ',
        'a',
        ' ',
        'simulated',
        ' ',
        'response.',
      ],
    }),
    {
      headers: { 'Content-Type': 'text/plain' },
    },
  );
}
```

---

### **Example 3: Testing Multiple Languages (With Gemini Mock)**

```typescript
import { simulateReadableStream, streamText } from 'ai';
import { MockLanguageModelV3 } from 'ai/test';

// Using with mock to test streaming behavior
const result = streamText({
  model: new MockLanguageModelV3({
    doStream: async () => ({
      stream: simulateReadableStream({
        chunks: [
          { type: 'text-start', id: 'text-1' },
          { type: 'text-delta', id: 'text-1', delta: '你好' },
          { type: 'text-delta', id: 'text-1', delta: '你好' },
          { type: 'text-delta', id: 'text-1', delta: '你好' },
          { type: 'text-end', id: 'text-1' },
          { type: 'finish', finishReason: 'stop' },
        ],
        chunkDelayInMs: 400,  // Simulate character-by-character output
      }),
    }),
  }),
  prompt: 'Say hello in Chinese!',
});

for await (const chunk of result.textStream) {
  console.log(chunk);
}
```

---

### **Example 4: Empty Stream / Edge Cases**

```typescript
// Empty stream (closes immediately)
const emptyStream = simulateReadableStream({
  chunks: [],
});

// Single chunk
const singleStream = simulateReadableStream({
  chunks: ['Only one value'],
});

// Large array of chunks
const bigStream = simulateReadableStream({
  chunks: Array(1000).fill('item'),
  chunkDelayInMs: 1,  // 1ms between each
});

// Complex objects
const objectStream = simulateReadableStream({
  chunks: [
    { id: 1, name: 'Alice', score: 95 },
    { id: 2, name: 'Bob', score: 87 },
    { id: 3, name: 'Carol', score: 92 },
  ],
  chunkDelayInMs: 100,
});
```

---

## **With Vercel v6 & Gemini 3.0 Flash (No Additional Config Needed)**

The `simulateReadableStream` function works seamlessly with:

- **Gemini 3.0 Flash**: Via `MockLanguageModelV3` for testing
- **Vercel SDK v6**: Already built into the `ai` package
- **No special configuration**: Just import and use

```typescript
// Works out of the box with Vercel v6
import { simulateReadableStream, streamText } from 'ai';
import { google } from '@ai-sdk/google';

// For REAL calls (not simulated):
const result = streamText({
  model: google('gemini-3.0-flash'),  // Real Gemini
  prompt: 'Hello!',
});

// For TESTING (simulated):
import { MockLanguageModelV3 } from 'ai/test';

const testResult = streamText({
  model: new MockLanguageModelV3({
    doStream: async () => ({
      stream: simulateReadableStream({
        chunks: [...],
        chunkDelayInMs: 100,
      }),
    }),
  }),
  prompt: 'Hello!',
});
```

---

## **Common Patterns & Best Practices**

| Use Case | Pattern |
|----------|---------|
| **Fast testing** | `chunkDelayInMs: null, initialDelayInMs: null` |
| **Realistic testing** | `initialDelayInMs: 500, chunkDelayInMs: 100` |
| **Typing effect UI** | `chunkDelayInMs: 50` (word-by-word) |
| **Network simulation** | `initialDelayInMs: 1000, chunkDelayInMs: 200` |
| **Performance testing** | `chunkDelayInMs: 1` (minimal delay) |
| **Demo/showcase** | `initialDelayInMs: 1000, chunkDelayInMs: 500` |

---

## **When NOT to Use It**

❌ **Don't use for**:
- Production code with real streams (use real API)
- Non-streaming operations
- When you have actual streaming data already
- For performance-critical paths (add overhead)

---

## **Summary**

**`simulateReadableStream` is your testing companion that:**
- Creates fake streams with chunks of data
- Adds realistic delays to simulate real-world behavior
- Lets you test streaming features without calling APIs
- Works perfectly with mock models for deterministic testing
- Enables debugging and prototyping before real implementation

**Use it whenever you need to test streaming functionality** in your AI-powered applications without external dependencies or API costs!
