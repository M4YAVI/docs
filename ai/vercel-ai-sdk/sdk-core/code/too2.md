
---

## **hasToolCall() - Complete Guide (0-100%)**

### **WHAT DOES IT DO? (Plain English)**

`hasToolCall()` is a **stop condition function** for the AI SDK that tells your AI agent to **STOP executing when a specific tool gets called**. 

Think of it like this: You're running an AI agent that can use multiple tools (like searching the web, calculating math, or providing answers). By default, the agent keeps running tool after tool. `hasToolCall()` says: "**Hey, stop the whole loop once this particular tool is called.**"

---

### **WHERE IS IT DEFINED?**

**File:** `packages/ai/src/generate-text/stop-condition.ts`

```typescript
// packages/ai/src/generate-text/stop-condition.ts (Lines 12-17)
export function hasToolCall(toolName: string): StopCondition<any> {
  return ({ steps }) =>
    steps[steps.length - 1]?.toolCalls?.some(
      toolCall => toolCall.toolName === toolName,
    ) ?? false;
}
```

**It's exported from:** `packages/ai/src/generate-text/index.ts`

```typescript
export { hasToolCall, stepCountIs, type StopCondition } from './stop-condition';
```

---

### **HOW DOES IT WORK INTERNALLY?**

```typescript
export function hasToolCall(toolName: string): StopCondition<any> {
  return ({ steps }) =>                          // Takes all the steps executed so far
    steps[steps.length - 1]?.toolCalls?.some(   // Looks at the LAST step's tool calls
      toolCall => toolCall.toolName === toolName,  // Checks if any match the name you provided
    ) ?? false;                                 // Returns true if found, false otherwise
}
```

**What it does step-by-step:**
1. Takes an array of `steps` (each step = one AI generation cycle)
2. Gets the **last step** in the array: `steps[steps.length - 1]`
3. Looks at all `toolCalls` in that last step
4. Checks if ANY of those tool calls have the name you specified
5. Returns `true` if found (stops the loop) or `false` if not found (keeps going)

---

### **PARAMETER EXPLANATION**

```typescript
hasToolCall(toolName: string)
```

| Parameter | Type | Required | What it means |
|-----------|------|----------|---------------|
| `toolName` | `string` | ✅ YES | The exact name of the tool that should trigger the stop condition |

**Example:**
```typescript
hasToolCall('finalAnswer')     // Stop when 'finalAnswer' tool is called
hasToolCall('search')          // Stop when 'search' tool is called
hasToolCall('weather')         // Stop when 'weather' tool is called
```

---

### **RETURN VALUE**

Returns a **`StopCondition`** function (not a boolean).

```typescript
export type StopCondition<TOOLS extends ToolSet> = (options: {
  steps: Array<StepResult<TOOLS>>;
}) => PromiseLike<boolean> | boolean;
```

This means it returns a function that:
- **Takes:** An object with all the `steps` executed so far
- **Returns:** Either `true` (stop now) or `false` (keep going)
- Can be **async or sync**

---

### **WHEN TO USE IT?**

#### **1. Agent Pattern - Final Answer Tool**
Stop the agent loop once it provides a final answer:

```typescript
import { generateText, hasToolCall } from 'ai';
import { google } from '@ai-sdk/google';

const result = await generateText({
  model: google('gemini-3.0-flash'),
  tools: {
    search: searchTool,
    calculate: calculateTool,
    finalAnswer: {
      description: 'Provide the final answer to the user',
      parameters: z.object({
        answer: z.string(),
      }),
      execute: async ({ answer }) => answer,
    },
  },
  // STOP when finalAnswer tool is called
  stopWhen: hasToolCall('finalAnswer'),
});
```

#### **2. Multi-Step Workflow**
Stop at specific checkpoints:

```typescript
// Example: Stop after the AI has gathered enough information
const result = await generateText({
  model: google('gemini-3.0-flash'),
  tools: {
    search: searchTool,
    analyzeData: analyzeDataTool,
    submitReport: submitReportTool,  // Stop here
  },
  stopWhen: hasToolCall('submitReport'),
});
```

#### **3. Combining Multiple Stop Conditions**
Stop when ANY condition is met:

```typescript
import { generateText, hasToolCall, stepCountIs } from 'ai';

const result = await generateText({
  model: google('gemini-3.0-flash'),
  tools: {
    weather: weatherTool,
    search: searchTool,
    finalAnswer: finalAnswerTool,
  },
  // Stop if ANY of these conditions are true:
  stopWhen: [
    hasToolCall('weather'),        // Stop if weather tool called
    hasToolCall('finalAnswer'),    // Stop if finalAnswer called
    stepCountIs(5),                // Stop after 5 steps max
  ],
});
```

#### **4. Stream Text (Real-time Output)**
Same thing but with `streamText`:

```typescript
import { streamText, hasToolCall } from 'ai';

const result = streamText({
  model: google('gemini-3.0-flash'),
  tools: {
    search: searchTool,
    answer: answerTool,
  },
  stopWhen: hasToolCall('answer'),
});

// Process the stream...
for await (const chunk of result.textStream) {
  console.log(chunk);
}
```

---

### **KEY CONCEPTS & IMPORTANT DETAILS**

#### **1. It Only Checks the LAST Step**
```typescript
steps[steps.length - 1]?.toolCalls  // ← Last step ONLY
```
It doesn't check if the tool was called 5 steps ago—only in the most recent step.

#### **2. It Checks Tool NAME, Not Execution**
The tool must actually be **CALLED** (be in the `toolCalls` array), not just **EXECUTED**. If a tool is called but has no `execute` function, it still counts as a call.

#### **3. Multiple Tools Can Be Called in One Step**
```typescript
// If multiple tools are called in the last step:
const step = {
  toolCalls: [
    { toolName: 'search', ... },
    { toolName: 'finalAnswer', ... },
  ]
};

// hasToolCall('search') → true
// hasToolCall('finalAnswer') → true
// hasToolCall('other') → false
```

#### **4. It Returns a Function, Not a Boolean**
```typescript
// ❌ WRONG - hasToolCall returns a function
const stopped = hasToolCall('tool'); // This is a function, not true/false

// ✅ CORRECT - Pass it to stopWhen
stopWhen: hasToolCall('tool')  // Pass the function itself
```

#### **5. Used with stopWhen Parameter**
Available in:
- `generateText({ stopWhen: ... })`
- `streamText({ stopWhen: ... })`
- `ToolLoopAgent({ stopWhen: ... })`

---

### **REAL-WORLD EXAMPLE - RESEARCH AGENT**

```typescript
import { generateText, hasToolCall, stepCountIs } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

// Define tools
const tools = {
  search: tool({
    description: 'Search the web',
    parameters: z.object({ query: z.string() }),
    execute: async ({ query }) => {
      // Search implementation
      return 'Search results...';
    },
  }),
  
  fetchArticle: tool({
    description: 'Fetch full article',
    parameters: z.object({ url: z.string() }),
    execute: async ({ url }) => {
      // Fetch implementation
      return 'Article content...';
    },
  }),
  
  submitResearch: tool({
    description: 'Submit final research',
    parameters: z.object({ report: z.string() }),
    execute: async ({ report }) => {
      // Save to database
      return 'Research saved';
    },
  }),
};

// Run agent with stop condition
const result = await generateText({
  model: google('gemini-3.0-flash'),
  tools,
  prompt: 'Research the latest AI trends and submit your findings',
  
  // Stop when the research is submitted
  stopWhen: [
    hasToolCall('submitResearch'),  // Primary: Stop when research submitted
    stepCountIs(10),                // Safety: Don't loop more than 10 times
  ],
});

console.log(result.finishReason);  // 'tool-call' (stopped by hasToolCall)
```

---

### **COMPARISON WITH OTHER STOP CONDITIONS**

| Condition | Purpose | Example |
|-----------|---------|---------|
| `hasToolCall('tool')` | Stop when specific tool is called | `hasToolCall('answer')` |
| `stepCountIs(10)` | Stop after N steps | `stepCountIs(10)` |
| Custom function | Stop on custom logic | `({ steps }) => steps.length > 5 && someCondition` |

```typescript
// All three types combined
stopWhen: [
  hasToolCall('finalAnswer'),           // Stop when finalAnswer is called
  stepCountIs(20),                      // Stop after 20 steps max
  ({ steps }) => {                      // Custom: Stop if costs exceed $1
    const totalCost = steps.reduce(
      (acc, step) => acc + (step.usage.cost ?? 0), 0
    );
    return totalCost > 1.0;
  },
]
```

---

### **WITH GEMINI 3.0 FLASH & VERCEL AI SDK v6**

```typescript
// ✅ For Gemini 3.0 Flash with Vercel AI SDK v6+
import { generateText, hasToolCall } from 'ai';
import { google } from '@ai-sdk/google';

const result = await generateText({
  model: google('gemini-3.0-flash'),  // Latest Gemini model
  tools: {
    getWeather: tool({
      description: 'Get weather',
      parameters: z.object({ location: z.string() }),
      execute: async ({ location }) => `Weather for ${location}`,
    }),
    finalAnswer: tool({
      description: 'Provide answer',
      parameters: z.object({ answer: z.string() }),
      execute: async ({ answer }) => answer,
    }),
  },
  prompt: 'What is the weather in New York?',
  stopWhen: hasToolCall('finalAnswer'),  // Stop when agent is done
});

// Result will have:
// - result.text: The final answer
// - result.finishReason: 'tool-call' (stopped by hasToolCall)
// - result.steps: All execution steps
```

---

### **EDGE CASES & GOTCHAS**

#### **1. Tool Not in Tools Object**
```typescript
// ❌ This will never trigger (tool doesn't exist)
stopWhen: hasToolCall('nonExistentTool')
```

#### **2. Tool Called But Execution Fails**
```typescript
// ✅ Still counts as a call (even if execute fails)
stopWhen: hasToolCall('tool')  // Triggers even if tool.execute() throws
```

#### **3. Combining with Array vs Single**
```typescript
// ✅ Single condition
stopWhen: hasToolCall('answer')

// ✅ Multiple conditions (stops when ANY is true)
stopWhen: [
  hasToolCall('answer'),
  stepCountIs(10),
]
```

#### **4. Case Sensitivity**
```typescript
hasToolCall('finalAnswer')  // Case-sensitive!
hasToolCall('FinalAnswer')  // Different tool name
```

---

### **COMPLETE WORKING EXAMPLE**

```typescript
import { generateText, hasToolCall, stepCountIs } from 'ai';
import { google } from '@ai-sdk/google';
import { tool } from 'ai';
import { z } from 'zod';

async function runResearchAgent() {
  const result = await generateText({
    model: google('gemini-3.0-flash'),
    
    // Tools available to the agent
    tools: {
      searchWeb: tool({
        description: 'Search the web for information',
        parameters: z.object({
          query: z.string().describe('Search query'),
        }),
        execute: async ({ query }) => {
          // Simulate search
          return `Results for: ${query}`;
        },
      }),
      
      analyzeData: tool({
        description: 'Analyze data from search results',
        parameters: z.object({
          data: z.string(),
        }),
        execute: async ({ data }) => {
          return `Analysis of: ${data}`;
        },
      }),
      
      provideFinalAnswer: tool({
        description: 'Provide final answer',
        parameters: z.object({
          answer: z.string(),
        }),
        execute: async ({ answer }) => {
          return answer;
        },
      }),
    },
    
    // Initial prompt
    prompt: 'Research and answer: What are the benefits of AI?',
    
    // STOP CONDITIONS
    stopWhen: [
      hasToolCall('provideFinalAnswer'),  // Primary: Stop when done
      stepCountIs(5),                     // Safety: Max 5 steps
    ],
  });

  console.log('Finish Reason:', result.finishReason);  // 'tool-call'
  console.log('Final Text:', result.text);
  console.log('Steps Taken:', result.steps.length);
  
  return result;
}

runResearchAgent();
```

---

### **SUMMARY - QUICK REFERENCE**

| Aspect | Details |
|--------|---------|
| **What** | Stops AI agent loop when specific tool is called |
| **Where** | `packages/ai/src/generate-text/stop-condition.ts` |
| **Import** | `import { hasToolCall } from 'ai'` |
| **Parameter** | Tool name (string) |
| **Returns** | StopCondition function |
| **Use With** | `generateText()`, `streamText()`, `ToolLoopAgent` |
| **Best For** | Agent patterns, multi-step workflows, checkpoints |
| **Combine With** | `stepCountIs()`, custom conditions |
| **Gemini 3.0** | ✅ Works perfectly with `google('gemini-3.0-flash')` |

---

This is the complete 0-100% guide on `hasToolCall()`!
