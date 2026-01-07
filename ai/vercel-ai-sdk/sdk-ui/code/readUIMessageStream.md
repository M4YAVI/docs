
# 🎯 `readUIMessageStream()` - Complete 0-100% Guide (Plain English)

## 🤔 What is it? (Super Simple)

Convert a stream of small chunks into complete UI message objects, one at a time.

**Stream Chunks (raw from server):**
```plaintext
├─ { type: 'text-delta', delta: 'Hello' }
├─ { type: 'text-delta', delta: ' world' }
└─ { type: 'finish' }
````

**↓ readUIMessageStream()**

**Complete Messages (usable):**

```plaintext
├─ { role: 'assistant', parts: [{text: 'Hello'}] }
├─ { role: 'assistant', parts: [{text: 'Hello world'}] }
└─ { role: 'assistant', parts: [{text: 'Hello world'}], DONE! }
```

*In one sentence:* It reads raw stream chunks and assembles them into complete `UIMessage` objects you can use.

---

## 🎯 When Do You Need It?

✅ **USE `readUIMessageStream()` FOR:**

* 📖 Reading streams in backend (server-side processing)
* 🔄 Processing streaming data after it leaves the API
* 🎯 Building custom processing pipelines (receive → transform → send)
* 💾 Saving streaming messages to database while streaming
* 📊 Analytics/logging on streaming responses
* 🤖 Building tools/agents that consume streams
* 🔌 Non-frontend backends that need to read streams

❌ **DON'T USE if:**

* 🎨 Frontend UI (use `useChat()` — it does this automatically)
* ⚡ Simple chat (streams are auto-parsed by `useChat()`)
* 🌐 Just sending to frontend (no processing needed)

---

## 📍 Where to Use It?

✅ **USE IN BACKEND (Server-side only)**

```ts
// ✅ Backend processing pipeline
import { readUIMessageStream, streamText } from 'ai';

const result = await streamText({ ... });
const stream = result.toUIMessageStream();

// Read and process each message as it arrives
for await (const uiMessage of readUIMessageStream({ stream })) {
  // Process complete message
  await saveToDatabase(uiMessage);
  await sendAnalytics(uiMessage);
}
```

❌ **DON'T USE in Frontend**

```tsx
// ❌ WRONG - useChat() already does this!
const { messages } = useChat();  // Already parsed & assembled!
// Don't need readUIMessageStream here
```

---

## 📋 What Happens Inside

**Input (Raw Chunks from Network):**

```plaintext
{ type: 'start', messageId: 'msg-123' }
{ type: 'text-start', id: 'text-1' }
{ type: 'text-delta', id: 'text-1', delta: 'Hello' }
{ type: 'text-delta', id: 'text-1', delta: ' ' }
{ type: 'text-delta', id: 'text-1', delta: 'world' }
{ type: 'text-end', id: 'text-1' }
{ type: 'finish', finishReason: 'stop' }
```

**Processing**

```plaintext
readUIMessageStream() processes:
├─ Group chunks by what they represent
├─ Assemble complete parts
├─ Build full UIMessage
└─ Emit complete message at each important step
```

**Output (Complete Messages)**

```plaintext
UIMessage 1: { role: 'assistant', parts: [] }
             (Just started, no content yet)
UIMessage 2: { role: 'assistant', parts: [{ type: 'text', text: '', state: 'streaming' }] }
             (Text starting, empty so far)
UIMessage 3: { role: 'assistant', parts: [{ type: 'text', text: 'Hello', state: 'streaming' }] }
             (First chunk: "Hello")
UIMessage 4: { role: 'assistant', parts: [{ type: 'text', text: 'Hello ', state: 'streaming' }] }
             (Second chunk: " ")
UIMessage 5: { role: 'assistant', parts: [{ type: 'text', text: 'Hello world', state: 'streaming' }] }
             (Third chunk: "world")
UIMessage 6: { role: 'assistant', parts: [{ type: 'text', text: 'Hello world', state: 'done' }] }
             (Completed!)
```

---

## 🚀 Complete Working Example (Gemini 2.0 Flash, V6)

### Example 1: Read Stream & Log Messages

```ts
import { readUIMessageStream, streamText } from 'ai';
import { google } from '@ai-sdk/google';

async function streamAndLog() {
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: 'Tell me a story',
  });

  const stream = result.toUIMessageStream();

  for await (const uiMessage of readUIMessageStream({ stream })) {
    console.log('Message update:');
    console.log(`- Parts: ${uiMessage.parts.length}`);
    console.log(`- Role: ${uiMessage.role}`);
    
    if (uiMessage.parts[0]?.type === 'text') {
      console.log(`- Text so far: "${uiMessage.parts[0].text}"`);
      console.log(`- State: ${uiMessage.parts[0].state}`);
    }
  }
  
  console.log('Stream complete!');
}

streamAndLog();
```

### Example 2: Save to Database While Streaming

```ts
import { readUIMessageStream, streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { db } from './database';

async function streamAndSave(userId: string, conversationId: string) {
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: 'Write code example',
  });

  const stream = result.toUIMessageStream();

  for await (const uiMessage of readUIMessageStream({ stream })) {
    await db.insertStreamUpdate({
      userId,
      conversationId,
      messageId: uiMessage.id,
      content: uiMessage,
      updatedAt: new Date(),
    });

    console.log(`Saved message update to DB`);
  }

  console.log('Streaming complete, message saved!');
}
```

### Example 3: Process Stream in Backend Pipeline

```ts
import { readUIMessageStream, streamText } from 'ai';
import { google } from '@ai-sdk/google';

async function processStreamingResponse(prompt: string) {
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
  });

  const stream = result.toUIMessageStream();
  const finalMessage = await readAndProcess(stream);

  return finalMessage;
}

async function readAndProcess(stream: ReadableStream) {
  let finalMessage: any = null;

  for await (const uiMessage of readUIMessageStream({ stream })) {
    console.log(`Received chunk, now have ${uiMessage.parts.length} parts`);
    const textPart = uiMessage.parts.find((p: any) => p.type === 'text');
    if (textPart?.type === 'text') {
      console.log(`Current text: "${textPart.text}"`);
    }
    finalMessage = uiMessage;
  }

  return finalMessage;
}
```

### Example 4: Handle Tool Calls While Reading

```ts
import { readUIMessageStream, streamText, tool } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

async function streamWithTools() {
  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt: 'Get weather and translate',
    tools: {
      getWeather: tool({
        description: 'Get weather',
        inputSchema: z.object({ city: z.string() }),
        execute: async ({ city }) => ({ temp: 72, city }),
      }),
      translate: tool({
        description: 'Translate text',
        inputSchema: z.object({ text: z.string(), lang: z.string() }),
        execute: async ({ text, lang }) => ({ translated: `${text} in ${lang}` }),
      }),
    },
  });

  const stream = result.toUIMessageStream();

  for await (const uiMessage of readUIMessageStream({ stream })) {
    for (const part of uiMessage.parts) {
      if (part.type === 'tool-call') {
        console.log(`📞 Tool called: ${part.toolName}`);
        console.log(`   Input: ${JSON.stringify(part.input)}`);
      }

      if (part.type === 'tool-result') {
        console.log(`✅ Tool result for ${part.toolName}:`);
        console.log(`   ${JSON.stringify(part.output)}`);
      }

      if (part.type === 'text') {
        console.log(`💬 Text: ${part.text}`);
      }
    }
  }
}
```

---

## 🎛️ API Reference

**Options for `readUIMessageStream()`**

```ts
const uiMessageStream = readUIMessageStream({
  stream: ReadableStream<UIMessageChunk>,  // REQUIRED
  message: {                                // OPTIONAL: resume
    id: 'msg-123',
    role: 'assistant',
    parts: []
  },
  terminateOnError: false,                  // OPTIONAL: stop on error
  onError: (error) => {                     // OPTIONAL: error handler
    console.error('Stream error:', error);
  },
});
```

**Return Type:** `AsyncIterableStream<UIMessage>`

```ts
for await (const message of uiMessageStream) {
  console.log(message.id);
  console.log(message.role);
  console.log(message.parts);
}
```

---

## ⚠️ Common Mistakes

1. **Using in frontend**
2. **Not awaiting the loop**
3. **Trying to index the result**
4. **Not handling errors**
5. **Wrong stream source**

*(Refer to your examples above; already fully detailed.)*

---

## 📋 UIMessage Structure Reference

```ts
{
  id: 'msg-123',
  role: 'assistant',
  metadata: undefined,
  parts: [
    { type: 'text', text: 'Hello world', state: 'done', providerMetadata: undefined },
    { type: 'tool-call', toolCallId: 'call-1', toolName: 'search', input: { query: 'AI' }, state: 'input-available' },
    { type: 'tool-result', toolName: 'search', output: { results: [...] } }
  ]
}
```

---

## 🎉 Summary

* **What:** Read stream chunks → assemble `UIMessage` objects
* **Where:** Backend, server-side only
* **When:** Need to process streaming data in backend
* **Why:** Read data as it arrives, not as final result
* **How:** `for await (const msg of readUIMessageStream({ stream }))`
* **Returns:** `AsyncIterableStream<UIMessage>`
* **Use Case:** Analytics, logging, database saves, validation, transformation

---

## 🚀 Quick Reference

| Task             | Code                                                       |
| ---------------- | ---------------------------------------------------------- |
| Read stream      | `for await (const msg of readUIMessageStream({ stream }))` |
| Get text         | `msg.parts.find(p => p.type === 'text')?.text`             |
| Get tool calls   | `msg.parts.filter(p => p.type === 'tool-call')`            |
| Convert to array | `await readUIMessageStream({ stream }).toArray()`          |
| Handle errors    | `readUIMessageStream({ stream, onError: (e) => {...} })`   |
| Resume stream    | `readUIMessageStream({ stream, message: lastMsg })`        |

---

## 💻 Complete Copy-Paste Example (Gemini 2.0 Flash)

```ts
// backend/stream-processor.ts
import { readUIMessageStream, streamText } from 'ai';
import { google } from '@ai-sdk/google';
import { db } from './database';

export async function processAndSaveStream(
  userId: string,
  prompt: string
): Promise<string> {
  console.log(`[${userId}] Starting stream...`);
  const startTime = Date.now();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    prompt,
    system: 'You are a helpful assistant.',
  });

  let finalText = '';
  let messageCount = 0;

  try {
    for await (const uiMessage of readUIMessageStream({
      stream: result.toUIMessageStream(),
      onError: (error) => console.error(`[${userId}] Stream error:`, error),
    })) {
      messageCount++;

      const textPart = uiMessage.parts.find((p: any) => p.type === 'text');
      if (textPart?.type === 'text') {
        finalText = textPart.text;

        if (messageCount % 5 === 0) {
          console.log(`[${userId}] Progress: ${textPart.text.length} chars`);
        }

        await db.saveStreamUpdate({
          userId,
          messageId: uiMessage.id,
          content: textPart.text,
          updatedAt: new Date(),
        });
      }

      const toolCalls = uiMessage.parts.filter((p: any) => p.type === 'tool-call');
      if (toolCalls.length > 0) {
        console.log(`[${userId}] Tool calls detected: ${toolCalls.length}`);
      }
    }
  } catch (error) {
    console.error(`[${userId}] Failed to process stream:`, error);
    throw error;
  }

  const duration = Date.now() - startTime;
  console.log(`[${userId}] Done! ${finalText.length} chars in ${duration}ms`);

  return finalText;
}

// Usage
const response = await processAndSaveStream(
  'user-123',
  'Write a short story about AI'
);
console.log('Final response:', response);
```

You're now a `readUIMessageStream()` expert! 🎊

Use it to read streams in your backend, process them, and build advanced features like:

* Real-time analytics
* Database persistence
* Stream validation
* Message transformation
* Custom pipelines

All while the frontend seamlessly displays the results with `useChat()`! 🚀

```


