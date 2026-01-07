
# 🎯 InferUITool - Complete 0-100% Guide (Plain English)

## 🤔 What is it? (Super Simple)

`InferUITool` is a **TypeScript type** that extracts input/output types from a **single tool**.

```ts
// One Tool:
const weatherTool = tool({
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ temp: z.number() })
})

// ❌ Without InferUITool:
type MyTool = any;  // No idea what inputs/outputs are

// ✅ With InferUITool:
type MyTool = InferUITool<typeof weatherTool>;
// Now TypeScript knows: input has 'city', output has 'temp'!
````

**In one sentence:** It extracts the input and output types from a single tool so you can use them for **type-safe handling**.

---

## 📊 Comparison: InferUITool vs InferUITools

| Feature    | InferUITool                   | InferUITools                           |
| ---------- | ----------------------------- | -------------------------------------- |
| Input      | Single tool                   | Multiple tools (object)                |
| Output     | `{ input: ..., output: ... }` | `{ toolName: { input, output }, ... }` |
| Use Case   | One tool handling             | All tools in app                       |
| Complexity | Simple                        | Complex                                |
| Common     | Less common                   | More common                            |

---

## 🎯 When Do You Need It?

### ✅ USE InferUITool WHEN:

* Handling one specific tool (not all tools)
* Type-safe tool processing for a single tool
* Custom UI components for one tool
* Tool-specific logic (weather handler, search handler, etc.)
* Building tool handlers independently
* Processing single tool results type-safely
* Testing one tool in isolation

### ❌ DON'T USE IF:

* You need all tools together (use `InferUITools` instead)
* No type safety needed
* Just using tools in backend

---

## 📍 Where to Use It?

### ✅ USE FOR INDIVIDUAL TOOL HANDLING

```ts
import { InferUITool } from 'ai';

const weatherTool = tool({
  inputSchema: z.object({ city: z.string() }),
  outputSchema: z.object({ temp: z.number() }),
  execute: async ({ city }) => ({ temp: 72 })
});

// ✅ Get types for this specific tool
type WeatherToolType = InferUITool<typeof weatherTool>;
// Result: { input: { city: string }, output: { temp: number } }

// Now use it for type-safe handling
function handleWeatherTool(input: WeatherToolType['input']) {
  console.log(input.city); // ✅ Works
  // input.query does NOT exist ❌
}
```

---

## 🚀 Complete Working Example (Gemini 2.0 Flash, V6)

### Example 1: Individual Tool Types

```ts
// app/api/chat/tools.ts
import { tool } from 'ai';
import { z } from 'zod';

export const weatherTool = tool({
  description: 'Get weather for a city',
  inputSchema: z.object({
    city: z.string().describe('City name'),
    unit: z.enum(['C', 'F']).default('F'),
  }),
  outputSchema: z.object({
    city: z.string(),
    temperature: z.number(),
    unit: z.string(),
    condition: z.string(),
  }),
  execute: async ({ city, unit }) => ({
    city,
    temperature: 72,
    unit,
    condition: 'sunny',
  }),
});
```

```ts
export const searchTool = tool({
  description: 'Search the web',
  inputSchema: z.object({
    query: z.string().describe('Search query'),
    limit: z.number().default(5),
  }),
  outputSchema: z.object({
    query: z.string(),
    results: z.array(z.object({
      title: z.string(),
      url: z.string(),
      snippet: z.string(),
    })),
  }),
  execute: async ({ query, limit }) => ({
    query,
    results: [
      { title: 'Result 1', url: 'https://example.com/1', snippet: 'First result' },
      { title: 'Result 2', url: 'https://example.com/2', snippet: 'Second result' },
    ],
  }),
});
```

```ts
export const calculatorTool = tool({
  description: 'Calculate math expressions',
  inputSchema: z.object({ expression: z.string() }),
  outputSchema: z.object({ expression: z.string(), result: z.number() }),
  execute: async ({ expression }) => ({
    expression,
    result: Function('"use strict"; return (' + expression + ')')(),
  }),
});

export const tools = {
  weather: weatherTool,
  search: searchTool,
  calculator: calculatorTool,
} as const;
```

---

### Example 2: Type-Safe Tool Handlers

```ts
import { InferUITool } from 'ai';
import { weatherTool, searchTool, calculatorTool } from './tools';

type WeatherToolInput = InferUITool<typeof weatherTool>['input'];
type WeatherToolOutput = InferUITool<typeof weatherTool>['output'];

export async function handleWeatherTool(input: WeatherToolInput): Promise<WeatherToolOutput> {
  return {
    city: input.city,
    temperature: 72,
    unit: input.unit,
    condition: 'sunny',
  };
}
```

---

### Example 3: Type-Safe Tool Router

```ts
export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = await streamText({
    model: google('gemini-2.0-flash'),
    messages,
    tools,
    async onToolCall({ toolName, input }) {
      let output: any;

      if (toolName === 'weather') {
        output = await handleWeatherTool(input as InferUITool<typeof weatherTool>['input']);
      } else if (toolName === 'search') {
        output = await handleSearchTool(input as InferUITool<typeof searchTool>['input']);
      } else if (toolName === 'calculator') {
        output = await handleCalculatorTool(input as InferUITool<typeof calculatorTool>['input']);
      }

      return output;
    },
  });

  return result.toUIMessageStreamResponse();
}
```

---

### Example 4: UI Component for Weather Tool

```tsx
import { InferUITool } from 'ai';
import { weatherTool } from '@/api/chat/tools';

type WeatherInput = InferUITool<typeof weatherTool>['input'];
type WeatherOutput = InferUITool<typeof weatherTool>['output'];

interface Props {
  input: WeatherInput;
  output?: WeatherOutput;
  loading?: boolean;
}

export function WeatherToolRenderer({ input, output, loading }: Props) {
  return (
    <div>
      <h3>🌤️ Weather Tool</h3>
      <p>City: {input.city}</p>
      <p>Unit: {input.unit}</p>
      {loading && <p>Loading...</p>}
      {output && (
        <div>
          <p>Temperature: {output.temperature}°{output.unit}</p>
          <p>Condition: {output.condition}</p>
        </div>
      )}
    </div>
  );
}
```

---

### Example 5: Frontend Tool Display

```tsx
import { useChat } from '@ai-sdk/react';
import { InferUITool } from 'ai';
import { weatherTool } from '@/api/chat/tools';
import { WeatherToolRenderer } from '@/components/WeatherToolRenderer';

export default function Chat() {
  const { messages } = useChat();
  type WeatherType = InferUITool<typeof weatherTool>;

  return (
    <div>
      {messages.map(m =>
        m.parts?.map((part, i) =>
          part.type === 'tool-call' && part.toolName === 'weather' ? (
            <WeatherToolRenderer
              key={i}
              input={part.input as WeatherType['input']}
              output={part.output as WeatherType['output'] | undefined}
            />
          ) : null
        )
      )}
    </div>
  );
}
```

---

## 💡 Quick Reference

| Task                | Code                                                             |
| ------------------- | ---------------------------------------------------------------- |
| Extract full type   | `type T = InferUITool<typeof myTool>;`                           |
| Extract input only  | `type Input = InferUITool<typeof myTool>['input'];`              |
| Extract output only | `type Output = InferUITool<typeof myTool>['output'];`            |
| Use in function     | `function handle(input: InferUITool<typeof myTool>['input']) {}` |
| Use in component    | `type Props = { input: InferUITool<typeof myTool>['input']; };`  |
| Multiple tools      | Use `InferUITools` instead                                       |

---

You are now an **InferUITool expert**! 🎉

Use it for:

* ✅ Type-safe single tool handling
* ✅ Tool-specific UI components
* ✅ Individual tool handlers
* ✅ Compile-time error catching
* ✅ Full IDE autocomplete

All with just:

```ts
InferUITool<typeof myTool>['input'] 
InferUITool<typeof myTool>['output']
```

