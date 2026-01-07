

---

# **COMPLETE GUIDE: Agent Interface & ToolLoopAgent (0-100%)**

## **Plain English Explanation**

### **What is an Agent?**

An **Agent** is like an intelligent assistant that can think, make decisions, and take actions. Think of it like hiring an employee who:
- Reads instructions (prompt/messages)
- Thinks about what to do
- Can use tools (like calling functions) to complete tasks
- Keeps learning from results and adjusts their actions
- Continues until they finish the task

### **What is ToolLoopAgent?**

**ToolLoopAgent** is the actual implementation of an Agent that works by:
1. Calling an AI model (like GPT-4)
2. The model thinks and decides what tool to use (if any)
3. Executes that tool
4. Feeds the result back to the model
5. **Repeats this loop** until the task is done

**The loop stops when:**
- The model says "I'm done" (no more tool calls needed)
- A tool doesn't have an execute function (can't execute)
- A tool needs your approval before running
- Maximum 20 steps by default (safety limit)

---

## **Core Components Explained**

### **1. Agent Interface (agent.ts)**

```typescript
interface Agent<CALL_OPTIONS, TOOLS, OUTPUT> {
  readonly version: 'agent-v1';
  readonly id: string | undefined;
  readonly tools: TOOLS;
  
  generate(options: AgentCallParameters<CALL_OPTIONS>): Promise<GenerateTextResult>;
  stream(options: AgentStreamParameters<CALL_OPTIONS, TOOLS>): Promise<StreamTextResult>;
}
```

**In plain English:**
- **version**: Tracks which version of Agent interface you're using (for future updates)
- **id**: Optional name for your agent
- **tools**: The functions/tools your agent can use
- **generate()**: Get the full response at once (non-streaming)
- **stream()**: Get the response word-by-word as it's generated (streaming)

### **2. ToolLoopAgent Class**

This is the **actual implementation** that does the work. It:
- Takes settings (model, instructions, tools)
- Calls `generateText()` or `streamText()` internally
- Manages the tool loop automatically
- Handles errors and retries

---

## **ToolLoopAgentSettings (Configuration)**

This is how you **set up your agent**. Here are all the options:

```typescript
{
  // ✅ REQUIRED
  model: LanguageModel,              // GPT-4, Claude, etc.
  
  // ⚙️ CONFIGURATION
  id?: string,                        // Optional agent name
  instructions?: string,              // System prompt/instructions
  tools?: TOOLS,                      // Available tools
  toolChoice?: 'auto' | 'required',  // Whether model should use tools
  
  // 🎛️ MODEL PARAMETERS
  maxOutputTokens?: number,           // Max response length
  temperature?: number,               // Creativity (0-1)
  topP?: number,                      // Diversity
  topK?: number,                      // Top K options
  presencePenalty?: number,           // Penalize repetition
  frequencyPenalty?: number,          // Penalize frequent words
  stopSequences?: string[],           // Stop at these strings
  seed?: number,                      // Reproducible results
  
  // 🔄 AGENT BEHAVIOR
  stopWhen?: StopCondition,           // When to stop the loop (default: 20 steps)
  activeTools?: string[],             // Limit which tools are available
  output?: Output,                    // Structured output schema
  prepareStep?: Function,             // Customize each step
  onStepFinish?: Callback,            // Called after each step
  onFinish?: Callback,                // Called when completely done
  
  // 🔧 ADVANCED
  experimental_telemetry?: Settings,  // Track performance
  experimental_repairToolCall?: Fn,   // Fix broken tool calls
  experimental_context?: unknown,     // Pass data to tools
  experimental_download?: Function,   // Custom file downloader
  callOptionsSchema?: Schema,         // Validate call options
  providerOptions?: Options,          // Provider-specific settings
  prepareCall?: Function,             // Transform call parameters
}
```

---

## **How to Use ToolLoopAgent (Step-by-Step)**

### **Basic Example (No Tools)**

```typescript
import { ToolLoopAgent } from 'ai';
import { openai } from '@ai-sdk/openai';

// 1. Create the agent
const agent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  instructions: 'You are a helpful assistant.',
});

// 2. Use it to generate text
const result = await agent.generate({
  prompt: 'Invent a new holiday and describe its traditions.',
});

console.log(result.content); // The response text
```

---

### **Example with Tools (Agent Can Do Things)**

```typescript
import { ToolLoopAgent, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const agent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  instructions: 'You are a weather assistant.',
  
  // 🔧 Define your tools
  tools: {
    weather: tool({
      description: 'Get the weather in a location',
      inputSchema: z.object({
        location: z.string(),
      }),
      execute: async ({ location }) => {
        // This code runs when the AI decides to use this tool
        return {
          location,
          temperature: 72,
          condition: 'Sunny',
        };
      },
    }),
  },
});

// Use it
const result = await agent.generate({
  prompt: 'What is the weather in Tokyo?',
});

// The agent will:
// 1. Read your prompt
// 2. Decide it needs to call the weather tool
// 3. Call it with location="Tokyo"
// 4. Get back the weather
// 5. Return "The weather in Tokyo is 72°F and sunny"
console.log(result.content);
```

---

### **Streaming Example (Get Response Word-by-Word)**

```typescript
const agent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  instructions: 'You are a helpful assistant.',
});

// 📺 Stream the response
const result = await agent.stream({
  prompt: 'Tell me a story.',
});

// Print text as it arrives
for await (const textPart of result.textStream) {
  process.stdout.write(textPart); // Outputs: "Once upon a..." etc
}
```

---

### **Advanced: Listen to Each Step**

```typescript
const agent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  instructions: 'You are a helpful assistant.',
  
  // Called after EACH step (each LLM call)
  onStepFinish: async (stepResult) => {
    console.log('Step:', stepResult.stepIndex);
    console.log('Reason:', stepResult.finishReason); // 'tool-calls' or 'stop'
  },
  
  // Called when EVERYTHING is done
  onFinish: async (event) => {
    console.log('Total steps:', event.steps.length);
    console.log('Total tokens used:', event.totalUsage.totalTokens);
  },
});

await agent.generate({ prompt: 'Do something' });
```

---

### **Advanced: Customize Calls with prepareCall**

```typescript
const agent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  callOptionsSchema: z.object({
    topic: z.string(),
    style: z.enum(['formal', 'casual']),
  }),
  
  // Transform the call based on options
  prepareCall: ({ options, ...rest }) => {
    const style = options?.style === 'formal' ? 'professional and formal' : 'friendly and casual';
    
    return {
      ...rest,
      instructions: `You are an expert in ${options?.topic}. Be ${style}.`,
    };
  },
});

// Now you can pass custom options
await agent.generate({
  prompt: 'Tell me about AI',
  options: { topic: 'Artificial Intelligence', style: 'formal' },
});
```

---

### **Advanced: Structured Output**

```typescript
import { Output } from 'ai';

const agent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  
  // Force the AI to return structured data
  output: Output.object({
    schema: z.object({
      recipe: z.object({
        name: z.string(),
        ingredients: z.array(z.object({
          name: z.string(),
          amount: z.string(),
        })),
        steps: z.array(z.string()),
      }),
    }),
  }),
});

const result = await agent.generate({
  prompt: 'Generate a lasagna recipe',
});

console.log(result.output); // Structured object, not just text!
```

---

## **AgentCallParameters (What You Pass to generate/stream)**

```typescript
{
  // Choose ONE of these:
  prompt: string | ModelMessage[], // Simple string OR array of messages
  // OR
  messages: ModelMessage[],         // Full conversation history
  
  // Optional:
  abortSignal?: AbortSignal,        // Cancel the request
  options?: CALL_OPTIONS,           // Custom options (if defined in schema)
}
```

**Example:**
```typescript
// Method 1: Simple prompt
await agent.generate({ prompt: 'Hello!' });

// Method 2: Conversation history
await agent.generate({
  messages: [
    { role: 'user', content: 'What is AI?' },
    { role: 'assistant', content: 'AI is...' },
    { role: 'user', content: 'Tell me more' },
  ]
});

// Method 3: With options
await agent.generate({
  prompt: 'Do something',
  options: { topic: 'Science' },
  abortSignal: controller.signal, // Can cancel anytime
});
```

---

## **Callbacks Explained**

### **onStepFinish Callback**

Called **after each LLM call** (each step of the loop):

```typescript
onStepFinish: async (stepResult) => {
  // Information about this specific step:
  stepResult.stepIndex;        // Step number (0, 1, 2...)
  stepResult.finishReason;     // Why it stopped: 'tool-calls', 'stop', 'end-turn'
  stepResult.text;             // Text generated in this step
  stepResult.toolCalls;        // Tools the AI wanted to call
  stepResult.toolResults;      // Results from tools
  stepResult.usage;            // Tokens used in this step
  stepResult.request;          // Raw request sent to model
}
```

**Use this when:** You want to monitor progress, log each step, or update UI in real-time.

### **onFinish Callback**

Called **once when everything is complete**:

```typescript
onFinish: async (event) => {
  // All steps combined:
  event.text;                  // Full final text
  event.steps;                 // Array of ALL StepResults
  event.totalUsage;            // Sum of ALL tokens used
  event.finishReason;          // Overall finish reason
  event.toolCalls;             // All tool calls made
  event.toolResults;           // All tool results
  event.experimental_context;  // Context passed to tools
}
```

**Use this when:** You want final stats, logging, or cleanup.

---

## **Understanding the Tool Loop**

Here's what happens internally, step-by-step:

```
START: User asks "What's the weather in Tokyo?"
  ↓
STEP 1: Send prompt + tools to GPT-4
  ↓
GPT-4 responds: "I need to call the weather tool with location=Tokyo"
  ↓
Execute: weather({ location: "Tokyo" })
  → Returns: { temperature: 72, condition: "Sunny" }
  ↓
STEP 2: Send previous request + tool result back to GPT-4
  ↓
GPT-4 responds: "Based on the data, the weather in Tokyo is 72°F and sunny"
  → Finish reason: "stop" (no more tools needed)
  ↓
END: Return final response to user
```

---

## **When to Use ToolLoopAgent vs generateText**

### **Use ToolLoopAgent when:**
- ✅ You want an **agentic AI** that makes decisions
- ✅ You need **multi-step reasoning** (use tools, get results, think again)
- ✅ Tasks require **tool loop execution** (like API calls, calculations)
- ✅ You want built-in **callbacks** for each step

### **Use generateText when:**
- ✅ You just want a **simple response** (no tools)
- ✅ You need **one-shot generation** (no loop)
- ✅ Simple Q&A, summarization, translation, etc.

---

## **Complete Real-World Example**

```typescript
import { ToolLoopAgent, tool } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

// Define tools the agent can use
const tools = {
  // Tool 1: Search the web
  webSearch: tool({
    description: 'Search the internet for information',
    inputSchema: z.object({
      query: z.string(),
    }),
    execute: async ({ query }) => {
      // Real implementation would call a search API
      return {
        results: [`Result 1 about ${query}`, `Result 2 about ${query}`],
      };
    },
  }),

  // Tool 2: Calculate math
  calculate: tool({
    description: 'Perform mathematical calculations',
    inputSchema: z.object({
      expression: z.string(),
    }),
    execute: async ({ expression }) => {
      return { result: eval(expression) }; // Very unsafe example!
    },
  }),
};

// Create the agent
const researchAgent = new ToolLoopAgent({
  model: openai('gpt-4o'),
  instructions: `You are a research assistant. 
    When asked a question you don't know, use webSearch.
    When asked math questions, use calculate.
    Combine findings into a clear answer.`,
  tools,

  // Monitor progress
  onStepFinish: ({ stepIndex, finishReason }) => {
    console.log(`Step ${stepIndex}: ${finishReason}`);
  },

  // Final report
  onFinish: ({ totalUsage, steps }) => {
    console.log(`Completed in ${steps.length} steps`);
    console.log(`Used ${totalUsage.totalTokens} tokens`);
  },
});

// Use the agent
async function askAgent(question: string) {
  const result = await researchAgent.generate({
    prompt: question,
  });
  return result.content;
}

// Examples
console.log(await askAgent('What is the capital of France?'));
// Agent: Uses webSearch → Returns "Paris"

console.log(await askAgent('What is 2 + 2?'));
// Agent: Uses calculate → Returns "4"

console.log(await askAgent('What is the weather and the population of Tokyo?'));
// Agent: Uses webSearch twice → Combines results → Returns both answers
```

---

## **With Gemini 3.0 Flash & Vercel v6**

### **Installation**

```bash
npm install ai @ai-sdk/google
```

### **Using Gemini with ToolLoopAgent**

```typescript
import { ToolLoopAgent, tool } from 'ai';
import { google } from '@ai-sdk/google'; // Gemini 3.0 Flash
import { z } from 'zod';

const agent = new ToolLoopAgent({
  // Use Gemini 3.0 Flash
  model: google('gemini-3.0-flash'),
  instructions: 'You are a helpful AI assistant.',
  tools: {
    weather: tool({
      description: 'Get weather',
      inputSchema: z.object({ location: z.string() }),
      execute: async ({ location }) => ({
        temperature: 72,
        location,
      }),
    }),
  },
});

// Use exactly the same way
const result = await agent.generate({
  prompt: 'What is the weather?',
});
```

### **Key Advantages with Gemini 3.0 Flash:**
- **Fast**: Lower latency than larger models
- **Cheap**: Costs much less per request
- **Smart enough**: Good for most agent tasks
- **Perfect for**: Tool calling, decision making

### **With Vercel v6**

```typescript
import { ToolLoopAgent } from 'ai';

// Just use the string reference (requires Vercel AI Gateway setup)
const agent = new ToolLoopAgent({
  model: 'google/gemini-3.0-flash', // Via Vercel AI Gateway
  instructions: 'You are helpful.',
});

// Works exactly the same!
```

---

## **Common Patterns**

### **Pattern 1: Validation Tool Loop**

```typescript
const validationAgent = new ToolLoopAgent({
  model: google('gemini-3.0-flash'),
  tools: {
    validateEmail: tool({
      description: 'Validate email format',
      inputSchema: z.object({ email: z.string() }),
      execute: async ({ email }) => ({
        valid: email.includes('@'),
        email,
      }),
    }),
  },
});
```

### **Pattern 2: Multi-Step Task**

```typescript
onFinish: async (event) => {
  // Save to database
  await db.logs.create({
    steps: event.steps.length,
    tokens: event.totalUsage.totalTokens,
    timestamp: new Date(),
  });
}
```

### **Pattern 3: Error Handling**

```typescript
try {
  const result = await agent.generate({
    prompt: userInput,
    abortSignal: AbortSignal.timeout(30000), // 30 second timeout
  });
} catch (error) {
  console.error('Agent failed:', error);
}
```

---

## **Summary**

| Concept | Explanation |
|---------|------------|
| **Agent Interface** | Contract defining what an agent must have |
| **ToolLoopAgent** | Implementation that loops: ask model → execute tools → repeat |
| **Settings** | Configuration: model, instructions, tools, callbacks |
| **generate()** | Get full response at once |
| **stream()** | Get response word-by-word |
| **Tools** | Functions the AI can call (search, calculate, etc) |
| **onStepFinish** | Callback after each LLM call |
| **onFinish** | Callback when completely done |
| **stopWhen** | Condition to end the loop |
| **Gemini 3.0 Flash** | Perfect for agents: fast & cheap |
| **Vercel v6** | Use string references for models |

---

## **You Now Know Everything! 🎉**

You can now:
- ✅ Understand what agents are
- ✅ Create a ToolLoopAgent
- ✅ Define and use tools
- ✅ Stream responses
- ✅ Handle callbacks
- ✅ Use Gemini 3.0 Flash
- ✅ Deploy with Vercel v6
- ✅ Build production agents

This is the complete guide from 0-100%!
