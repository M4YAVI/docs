

---

# **Complete Guide to `tool` and `dynamicTool` in Vercel AI SDK v6** 

## **Plain English Overview**

Think of **`tool`** and **`dynamicTool`** as functions that let you define what abilities an AI model can use. Just like a person has a toolbox with specific tools, you give an AI model specific "tools" it can call to complete tasks.

- **`tool`** = For tools where you know exactly what inputs/outputs the function will have (type-safe, fully typed)
- **`dynamicTool`** = For tools where inputs/outputs are determined at runtime, not compile time (flexible, runtime validation)

---

## **1. What is `tool()`?**

### **Definition in Plain English**
`tool()` is a TypeScript helper function that:
1. Defines what an AI model can do (a tool description)
2. Specifies what inputs the tool needs (using Zod schema)
3. Provides the code that actually does the work (execute function)
4. Gives TypeScript perfect type inference so you don't make mistakes

### **Key Characteristics**
- ✅ **Type-safe** - TypeScript knows exactly what types your inputs/outputs are
- ✅ **Compile-time validation** - Errors caught before runtime
- ✅ **Full IDE support** - Autocomplete works perfectly
- ✅ **Automatic validation** - The schema is used to validate AI model outputs

### **How It Works**
```typescript
import { tool } from 'ai';
import { z } from 'zod';

const weatherTool = tool({
  // Description tells the AI when to use this tool
  description: 'Get the weather in a location',
  
  // Schema defines what inputs the tool accepts
  // This is BOTH for the AI to understand AND for TypeScript to know the types
  inputSchema: z.object({
    location: z.string().describe('The city name'),
    unit: z.enum(['C', 'F']).describe('Temperature unit'),
  }),
  
  // Execute function - the actual code that runs
  // TypeScript KNOWS that location is a string and unit is 'C' or 'F'
  execute: async ({ location, unit }) => {
    // location: string ← automatically inferred!
    // unit: 'C' | 'F' ← automatically inferred!
    return {
      location,
      temperature: 72,
      unit,
    };
  },
});
```

### **When to Use `tool()`**
✅ **Use when:**
- You know the input/output types at development time
- You want maximum type safety and IDE support
- You're building static tools (weather, calculator, file operations)
- You need the model to understand exactly what structure to provide

❌ **Don't use when:**
- The tool structure is loaded from a database at runtime
- The tool inputs/outputs vary completely based on user input
- You need extreme flexibility (use `dynamicTool` instead)

---

## **2. What is `dynamicTool()`?**

### **Definition in Plain English**
`dynamicTool()` is for when you don't know the exact structure of inputs/outputs at build time. It accepts and returns `unknown` types, meaning:
- The AI can call it with any structure
- You validate the input at runtime (when the tool actually runs)
- TypeScript doesn't restrict you

### **Key Characteristics**
- ✅ **Runtime flexibility** - Works with any input structure
- ✅ **Perfect for MCP (Model Context Protocol) tools** - When tools come from external sources
- ✅ **User-defined tools** - When users provide tool definitions
- ⚠️ **Less type-safe** - You lose compile-time type checking

### **How It Works**
```typescript
import { dynamicTool } from 'ai';
import { z } from 'zod'

const customTool = dynamicTool({
  description: 'Execute any user-defined function',
  
  // Can use z.unknown() or z.any() for total flexibility
  inputSchema: z.object({}),
  
  // Input is typed as 'unknown' - you must validate it yourself
  execute: async (input) => {
    // input is type 'unknown' - we need to validate/cast it
    const { action, parameters } = input as any;
    
    // Your validation logic here
    if (!action || typeof action !== 'string') {
      throw new Error('Invalid action');
    }
    
    // Perform dynamic logic
    return {
      result: `Executed ${action}`,
    };
  },
});
```

### **When to Use `dynamicTool()`**
✅ **Use when:**
- Tools are loaded from MCP (Model Context Protocol) servers
- Tools are fetched from a database at runtime
- User input determines tool structure
- You're building a plugin system
- You need maximum flexibility

❌ **Don't use when:**
- You know the tool structure at development time (use `tool()` instead)
- You want maximum type safety

---

## **3. Complete Comparison Table**

| Feature | `tool()` | `dynamicTool()` |
|---------|---------|-----------------|
| **Type Safety** | ✅ Full | ⚠️ Partial |
| **Compile-time Errors** | ✅ Yes | ❌ No |
| **IDE Autocomplete** | ✅ Full | ⚠️ Limited |
| **Input Type** | Known (`string`, `number`, etc.) | Unknown (`unknown`) |
| **Output Type** | Known | Unknown |
| **Runtime Validation** | Automatic | Manual |
| **Use Case** | Static tools | Dynamic/plugin tools |
| **Complexity** | Simple | More complex |
| **Performance** | Identical | Identical |

---

## **4. Complete Practical Examples**

### **Example 1: Basic `tool()` - Weather API**
```typescript
import { tool } from 'ai';
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Define a static weather tool
const weatherTool = tool({
  description: 'Get weather for a city',
  inputSchema: z.object({
    location: z.string().describe('City name'),
  }),
  execute: async ({ location }) => {
    // In real app, call actual weather API
    return {
      location,
      temperature: 72,
      condition: 'Sunny',
    };
  },
});

// Use it in an AI call
const result = await generateText({
  model: openai('gpt-4o'),
  tools: {
    weather: weatherTool,
  },
  prompt: 'What is the weather in Paris?',
});

// TypeScript knows tool structure perfectly!
```

### **Example 2: Multiple Static Tools**
```typescript
import { tool } from 'ai';
import { generateText } from 'ai';
import { z } from 'zod';

const tools = {
  // Tool 1: Get weather
  getWeather: tool({
    description: 'Get weather',
    inputSchema: z.object({ city: z.string() }),
    execute: async ({ city }) => ({
      city,
      temp: 72,
    }),
  }),

  // Tool 2: Search web
  searchWeb: tool({
    description: 'Search the web',
    inputSchema: z.object({ query: z.string() }),
    execute: async ({ query }) => ({
      query,
      results: ['Result 1', 'Result 2'],
    }),
  }),

  // Tool 3: Calculate
  calculate: tool({
    description: 'Do math',
    inputSchema: z.object({ 
      expression: z.string() 
    }),
    execute: async ({ expression }) => ({
      result: eval(expression),
    }),
  }),
};

// Full type safety on all tools!
```

### **Example 3: `dynamicTool()` - MCP Tools**
```typescript
import { dynamicTool, generateText } from 'ai';
import { z } from 'zod';

// Function that loads tools from MCP server at runtime
function loadMCPTools() {
  const tools = {};
  
  // Simulating loading from external MCP server
  const mcpToolDefinitions = [
    { name: 'runQuery', schema: {} },
    { name: 'fetchData', schema: {} },
  ];

  for (const toolDef of mcpToolDefinitions) {
    tools[toolDef.name] = dynamicTool({
      description: `Tool: ${toolDef.name}`,
      inputSchema: z.unknown(), // Unknown structure
      execute: async (input) => {
        // Validate at runtime
        console.log('Executing', toolDef.name, 'with:', input);
        return { success: true };
      },
    });
  }

  return tools;
}

const result = await generateText({
  model: openai('gpt-4o'),
  tools: loadMCPTools(),
  prompt: 'Execute a query',
});
```

### **Example 4: Mixing Static and Dynamic Tools**
```typescript
import { tool, dynamicTool, generateText } from 'ai';
import { z } from 'zod';

const result = await generateText({
  model: openai('gpt-4o'),
  tools: {
    // Static tool - fully typed
    weather: tool({
      description: 'Get weather',
      inputSchema: z.object({ city: z.string() }),
      execute: async ({ city }) => ({
        city,
        temp: 72,
      }),
    }),

    // Dynamic tool - flexible
    custom: dynamicTool({
      description: 'Custom user tool',
      inputSchema: z.unknown(),
      execute: async (input) => {
        console.log('Dynamic input:', input);
        return { result: 'done' };
      },
    }),
  },
  onStepFinish: ({ toolCalls }) => {
    for (const toolCall of toolCalls) {
      // Check if it's a dynamic tool
      if (toolCall.dynamic) {
        console.log('Dynamic tool call:', toolCall.toolName);
        // input/output are 'unknown' here
        continue;
      }

      // Static tools have full type safety
      switch (toolCall.toolName) {
        case 'weather':
          console.log(toolCall.input.city); // string ✅
          break;
      }
    }
  },
  prompt: 'Help me',
});
```

---

## **5. Schema Definition (Zod)**

### **Basic Schemas**
```typescript
import { z } from 'zod';
import { tool } from 'ai';

// Simple string input
tool({
  description: 'Echo text',
  inputSchema: z.object({
    text: z.string(),
  }),
  execute: async ({ text }) => ({ echo: text }),
});

// Multiple field types
tool({
  description: 'Complex tool',
  inputSchema: z.object({
    name: z.string().describe('Person name'),
    age: z.number().describe('Age in years'),
    email: z.string().email(),
    active: z.boolean(),
  }),
  execute: async (input) => ({ success: true }),
});

// Optional fields
tool({
  description: 'Optional params',
  inputSchema: z.object({
    required: z.string(),
    optional: z.string().optional(),
  }),
  execute: async (input) => ({ success: true }),
});

// Enums
tool({
  description: 'Unit converter',
  inputSchema: z.object({
    value: z.number(),
    unit: z.enum(['C', 'F', 'K']),
  }),
  execute: async (input) => ({ converted: 0 }),
});

// Arrays
tool({
  description: 'Sum numbers',
  inputSchema: z.object({
    numbers: z.array(z.number()),
  }),
  execute: async ({ numbers }) => ({
    sum: numbers.reduce((a, b) => a + b, 0),
  }),
});

// Nested objects
tool({
  description: 'Create user',
  inputSchema: z.object({
    name: z.string(),
    address: z.object({
      street: z.string(),
      city: z.string(),
      zip: z.string(),
    }),
  }),
  execute: async (input) => ({ userId: '123' }),
});
```

---

## **6. Advanced Features**

### **6.1 Strict Mode (Validation)**
```typescript
tool({
  description: 'Strict tool',
  inputSchema: z.object({
    email: z.string().email(),
  }),
  strict: true, // LLM will ONLY generate valid emails
  execute: async ({ email }) => ({ valid: true }),
});
```

### **6.2 Input Examples**
```typescript
tool({
  description: 'Weather tool',
  inputSchema: z.object({
    location: z.string(),
  }),
  // Help the AI understand expected format
  inputExamples: [
    { input: { location: 'New York' } },
    { input: { location: 'London' } },
    { input: { location: 'Tokyo' } },
  ],
  execute: async ({ location }) => ({ temp: 72 }),
});
```

### **6.3 Tool Approval (Security)**
```typescript
tool({
  description: 'Delete file',
  inputSchema: z.object({
    filePath: z.string(),
  }),
  needsApproval: true, // Requires user approval before execution
  execute: async ({ filePath }) => {
    // User must approve before this runs!
    return { deleted: true };
  },
});
```

### **6.4 Output Schema**
```typescript
tool({
  description: 'Search',
  inputSchema: z.object({
    query: z.string(),
  }),
  outputSchema: z.object({
    results: z.array(z.string()),
    count: z.number(),
  }),
  execute: async ({ query }) => ({
    results: ['Result 1'],
    count: 1,
  }),
});
```

### **6.5 Result Transformation**
```typescript
tool({
  description: 'Tool with custom output',
  inputSchema: z.object({ query: z.string() }),
  execute: async ({ query }) => ({
    data: 'raw data',
  }),
  toModelOutput: ({ input, output, toolCallId }) => {
    // Transform before sending to model
    return {
      type: 'text' as const,
      text: `Found: ${output.data}`,
    };
  },
});
```

---

## **7. How to Use in Next.js + Vercel AI SDK v6**

### **Server-Side (API Route)**
```typescript
// app/api/chat/route.ts
import { streamText, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages,
    tools: {
      getWeather: tool({
        description: 'Get weather for a city',
        inputSchema: z.object({
          city: z.string(),
        }),
        execute: async ({ city }) => ({
          city,
          temperature: 72,
          condition: 'Sunny',
        }),
      }),
    },
    system: 'You are a helpful assistant with weather access.',
  });

  return result.toTextStreamResponse();
}
```

### **Client-Side (React)**
```typescript
// app/page.tsx
'use client';

import { useChat } from '@ai-sdk/react';

export default function Page() {
  const { messages, input, setInput, handleSubmit } = useChat({
    api: '/api/chat',
  });

  return (
    <div>
      {messages.map(msg => (
        <div key={msg.id}>
          <strong>{msg.role}:</strong> {msg.content}
        </div>
      ))}

      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={e => setInput(e.target.value)}
          placeholder="Ask about weather..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

---

## **8. With Gemini 3.0 Flash Model**

```typescript
import { streamText, tool } from 'ai';
import { google } from '@ai-sdk/google'; // Note: Install @ai-sdk/google
import { z } from 'zod';

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    // Use Gemini 3.0 Flash (or newer available model)
    model: google('gemini-2.0-flash'), // Latest available
    messages,
    tools: {
      getWeather: tool({
        description: 'Get weather',
        inputSchema: z.object({
          location: z.string(),
        }),
        execute: async ({ location }) => ({
          location,
          temp: 72,
          condition: 'Sunny',
        }),
      }),
    },
  });

  return result.toTextStreamResponse();
}
```

---

## **9. Tool Execution Flow (Step-by-Step)**

```
1. User Message
   ↓
2. AI Model Analyzes Tools
   (reads descriptions and schemas)
   ↓
3. Model Decides to Use Tool
   ↓
4. Model Generates Tool Input
   (must match inputSchema)
   ↓
5. SDK Validates Input
   ↓
6. execute() Function Runs
   (your custom code)
   ↓
7. Tool Result Returned
   ↓
8. AI Processes Result
   ↓
9. AI Generates Final Response
```

---

## **10. Complete Real-World Example**

```typescript
// app/api/agent/route.ts
import { streamText, tool, stepCountIs } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Real tool definitions
const tools = {
  // Search tool
  search: tool({
    description: 'Search the web for information',
    inputSchema: z.object({
      query: z.string().describe('Search query'),
    }),
    execute: async ({ query }) => {
      // Call real search API
      return {
        results: [
          { title: 'Result 1', url: 'https://example.com' },
          { title: 'Result 2', url: 'https://example.com' },
        ],
      };
    },
  }),

  // Weather tool
  getWeather: tool({
    description: 'Get current weather',
    inputSchema: z.object({
      city: z.string().describe('City name'),
      units: z.enum(['C', 'F']).optional(),
    }),
    execute: async ({ city, units = 'C' }) => {
      // Call weather API
      return {
        city,
        temperature: 72,
        units,
        condition: 'Partly Cloudy',
      };
    },
  }),

  // Calculator tool
  calculate: tool({
    description: 'Perform mathematical calculations',
    inputSchema: z.object({
      expression: z.string().describe('Math expression'),
    }),
    execute: async ({ expression }) => {
      try {
        const result = eval(expression); // Use math-parser in production!
        return { result, expression };
      } catch (error) {
        return { error: 'Invalid expression', expression };
      }
    },
  }),
};

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: openai('gpt-4o'),
    messages,
    tools,
    system: `You are a helpful assistant. You have access to:
- Web search
- Weather information
- Calculator

Use these tools to help answer user questions.`,
    // Stop after 5 tool calls
    stopWhen: stepCountIs(5),
  });

  return result.toTextStreamResponse();
}
```

---

## **11. Common Mistakes & How to Avoid Them**

| Mistake | Problem | Fix |
|---------|---------|-----|
| Missing `describe()` | AI doesn't understand input | Add `.describe()` to schema fields |
| Wrong Zod type | Type mismatch errors | Match Zod schema to actual execute types |
| `execute` not async | Runtime error | Always make execute async |
| No error handling | App crashes | Wrap execute in try-catch |
| Complex nested schemas | AI confused | Keep schemas simple and clear |
| Forgetting tool name | Tool not callable | Name tools in the tools object |

---

## **12. When to Use What - Decision Tree**

```
Do you know the tool structure at development time?
│
├─→ YES
│   └─→ Use `tool()`
│       └─→ Full TypeScript support
│       └─→ Maximum type safety
│       └─→ Example: weatherTool, calculator
│
└─→ NO / DYNAMIC
    └─→ Use `dynamicTool()`
        └─→ Loaded from database
        └─→ Loaded from MCP server
        └─→ User-defined at runtime
        └─→ Example: plugin system
```

---

## **Summary: Everything You Need to Know**

| Aspect | Answer |
|--------|--------|
| **What is `tool()`?** | TypeScript helper for defining AI tools with known types |
| **What is `dynamicTool()`?** | TypeScript helper for AI tools with runtime-determined types |
| **When use `tool()`?** | Always, when you know the structure (static tools) |
| **When use `dynamicTool()`?** | When structure is dynamic (MCP, plugins, databases) |
| **Are they required?** | No, but recommended for type safety and validation |
| **Performance difference?** | None - identical at runtime |
| **Can I mix them?** | Yes! Use both static and dynamic in same AI call |
| **With Gemini 3.0 Flash?** | Yes, works identically with any provider |
| **Vercel v6 only?** | Yes, designed for AI SDK v6.0+ |

---

This guide covers **everything** from basic concepts to advanced production patterns. You now have complete knowledge of `tool` and `dynamicTool` to build professional AI applications! 🚀
