
# 🎯 InferUITools - Complete 0-100% Guide (Plain English)

## 🤔 What is it? (Super Simple)

TypeScript type that extracts tool input/output types from your tools, so messages know about them.

```ts
// Your Tools:
const tools = {
  weather: tool({ 
    inputSchema: z.object({ city: z.string() }),
    outputSchema: z.object({ temp: z.number() })
  })
}

// ❌ Without InferUITools:
type MyMessage = UIMessage;  // No idea about tools!

// ✅ With InferUITools:
type MyMessage = UIMessage<never, UIDataTypes, InferUITools<typeof tools>>;
// Now TypeScript knows: weather tool has city input & temp output!
````

In one sentence: It's a TypeScript helper that lets your messages know the exact structure of your tools so you get autocomplete and type safety.

---

## 🎯 When Do You Need It?

### ✅ USE InferUITools WHEN:

* 🎯 Type-safe tool handling (autocomplete for tool calls)
* 🛡️ Catch errors at compile-time (not runtime)
* 🎨 Frontend rendering (know tool input/output types)
* 💾 Type-safe messages (message types match tools)
* 📱 Building custom UI for tool calls
* 🔄 Multiple tools (keep track of all types)
* 🧪 Type checking (validate tool integration)

### ❌ DON'T USE if:

* No tools in your app
* Don't care about TypeScript types
* Just using `useChat()` without custom handling

---

## 📍 Where to Use It?

### ✅ USE IN MESSAGE TYPES (Type definitions)

```ts
import { UIMessage, InferUITools } from 'ai';

const tools = {
  weather: tool({ ... }),
  search: tool({ ... }),
};

// ✅ Tell TypeScript about your tools
type ChatMessage = UIMessage<never, UIDataTypes, InferUITools<typeof tools>>;

// Now use it everywhere
const messages: ChatMessage[] = [];

// TypeScript knows about tool calls!
messages[0]?.parts?.filter(p => p.type === 'tool-call')
  .forEach(tool => {
    // Autocomplete for toolName: "weather" | "search" ✅
    tool.toolName;
  });
```

---

## 🚀 Complete Working Example (Gemini 2.0 Flash, V6)

### Backend Setup

```ts
// app/api/chat/types.ts
import { UIMessage, InferUITools, UIDataTypes } from 'ai';
import { tools } from './tools';  // Your tool definitions

// ✅ Extract tool types and create message type
export type ChatMessage = UIMessage<
  never,                           // metadata type (not used)
  UIDataTypes,                     // data parts type
  InferUITools<typeof tools>       // ← Magic! Infers all tool types
>;
```

```ts
// app/api/chat/tools.ts
import { tool } from 'ai';
import { z } from 'zod';
import { google } from '@ai-sdk/google';

export const tools = {
  weather: tool({
    description: 'Get weather for a city',
    inputSchema: z.object({
      city: z.string().describe('City name'),
      unit: z.enum(['C', 'F']).optional(),
    }),
    execute: async ({ city, unit = 'F' }) => ({
      city,
      temperature: 72,
      unit,
      condition: 'sunny',
    }),
  }),

  search: tool({
    description: 'Search the web',
    inputSchema: z.object({
      query: z.string().describe('Search query'),
      limit: z.number().default(5),
    }),
    execute: async ({ query, limit }) => ({
      query,
      results: [
        { title: 'Result 1', url: 'https://example.com/1' },
        { title: 'Result 2', url: 'https://example.com/2' },
      ],
    }),
  }),

  calculator: tool({
    description: 'Calculate math expressions',
    inputSchema: z.object({
      expression: z.string().describe('Math expression'),
    }),
    execute: async ({ expression }) => {
      try {
        const result = Function('"use strict"; return (' + expression + ')')();
        return { result, expression };
      } catch (e) {
        return { error: 'Invalid expression' };
      }
    },
  }),
} as const;
```

```ts
// app/api/chat/route.ts
import { streamText, convertToModelMessages } from 'ai';
import { google } from '@ai-sdk/google';
import { tools } from './tools';
import { ChatMessage } from './types';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const typedMessages: ChatMessage[] = messages;

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages: await convertToModelMessages(typedMessages),
    tools,
  });

  return result.toUIMessageStreamResponse();
}
```

---

### Frontend Type-Safe Usage

```tsx
// app/page.tsx
'use client';
import { useChat } from '@ai-sdk/react';
import { ChatMessage } from './api/chat/types';

export default function Chat() {
  const { messages } = useChat<ChatMessage>();

  return (
    <div className="chat">
      {messages.map(m => (
        <div key={m.id}>
          {m.parts?.map((part, i) => {
            if (part.type === 'tool-call') {
              return (
                <div key={i} className="tool-call">
                  <strong>Calling {part.toolName}</strong>
                  <pre>{JSON.stringify(part.input, null, 2)}</pre>
                </div>
              );
            }

            if (part.type === 'tool-result') {
              return (
                <div key={i} className="tool-result">
                  Result: {JSON.stringify(part.output)}
                </div>
              );
            }

            if (part.type === 'text') {
              return <p key={i}>{part.text}</p>;
            }
          })}
        </div>
      ))}
    </div>
  );
}
```

---

## 🔬 How InferUITools Works (Deep Dive)

```ts
// Step 1: Raw Tools
const tools = {
  weather: tool({
    inputSchema: z.object({ city: z.string() }),
    outputSchema: z.object({ temp: z.number() }),
    execute: async ({ city }) => ({ temp: 72 })
  })
}

// Step 2: InferUITools Extracts Types
// InferUITools<typeof tools> produces:
{
  weather: {
    input: { city: string };
    output: { temp: number };
  }
}

// Step 3: UIMessage Gets Tool Types
type ChatMessage = UIMessage<
  never,
  UIDataTypes,
  {
    weather: { input: { city: string }, output: { temp: number } }
  }
>
```

Result: Full type safety!

---

## 📋 TypeScript Benefits

* ✅ Autocomplete for Tool Names
* ✅ Type-Safe Tool Input & Output
* ✅ IDE Suggestions
* ✅ Less runtime bugs

```ts
const toolCall: ToolUIPart = {
  toolName: 'weather',  // ✅ autocomplete: "weather" | "search" | "calculator"
};
```

---

## 🎯 Common Usage Patterns

### Pattern 1: Simple Tool Type Export

```ts
export const tools = { weather, search } as const;
export type ToolTypes = InferUITools<typeof tools>;
export type Message = UIMessage<never, UIDataTypes, ToolTypes>;
```

### Pattern 2: Nested Tools

```ts
const tools = {
  category1: { weather: tool({ ... }) },
  category2: { search: tool({ ... }) },
} as const;
type ToolTypes = InferUITools<typeof tools>;
```

### Pattern 3: Dynamic Tool Registration

```ts
const createTools = () => ({ weather: tool({ ... }), search: tool({ ... }) });
const myTools = createTools();
type ToolTypes = InferUITools<typeof myTools>;
export type ChatMessage = UIMessage<never, UIDataTypes, ToolTypes>;
```

---

## ⚠️ Common Mistakes

1. Forgetting `as const`
2. Wrong type parameter order
3. Not using extracted types
4. Importing `InferUITools` incorrectly
5. Using before tools are defined

---

## 🎨 Rendering Tools Safely

```tsx
function ToolCallRenderer({ toolCall }: { toolCall: ToolUIPart<MessageTools> }) {
  if (toolCall.toolName === 'weather') {
    return <div>Weather for {toolCall.input.city}</div>;
  }
  if (toolCall.toolName === 'search') {
    return <div>Search: {toolCall.input.query}</div>;
  }
  return <div>Unknown tool</div>;
}
```

---

## 📊 Before & After Comparison

❌ Before (No InferUITools)

```ts
const messages: any[] = [];
messages[0].parts.forEach((part: any) => {
  console.log(part.input.city); // Any type
});
```

✅ After (With InferUITools)

```ts
type ChatMessage = UIMessage<never, UIDataTypes, InferUITools<typeof tools>>;
const messages: ChatMessage[] = [];
messages[0].parts.forEach(part => {
  console.log(part.input.city); // ✅ string, type-safe
});
```

---

## 🎉 Summary

* **What:** Extract tool input/output types
* **Where:** Type definitions
* **When:** Building type-safe chat
* **Why:** Autocomplete, type safety, catch errors early
* **How:** `InferUITools<typeof tools>`
* **Must:** Use `as const` on tools
* **Use in:** `UIMessage<never, UIDataTypes, InferUITools<typeof tools>>`

---

## 🚀 Quick Reference

| Task             | Code                                                                |
| ---------------- | ------------------------------------------------------------------- |
| Define tools     | `const tools = { weather: tool(...), search: tool(...) } as const;` |
| Extract types    | `type Tools = InferUITools<typeof tools>;`                          |
| Message type     | `type ChatMessage = UIMessage<never, UIDataTypes, Tools>;`          |
| Use in useChat   | `useChat<ChatMessage>()`                                            |
| Render tool call | `if (part.toolName === 'weather') { ... }`                          |
| Access input     | `part.input`                                                        |
| Access output    | `part.output`                                                       |

---

## 💻 Complete Copy-Paste Template (Gemini 2.0 Flash)

```ts
// tools.ts
export const tools = { ... } as const;

// types.ts
export type ToolTypes = InferUITools<typeof tools>;
export type ChatMessage = UIMessage<never, UIDataTypes, ToolTypes>;

// route.ts
export async function POST(req: Request) { ... }

// page.tsx
export default function Chat() { ... }
```

You're now an **InferUITools expert!** 🎊

Use it to get:
✅ Full TypeScript autocomplete
✅ Compile-time error checking
✅ Type-safe tool handling
✅ Better IDE support
✅ Fewer runtime bugs

All with just one line: `InferUITools<typeof tools>`! 🚀

