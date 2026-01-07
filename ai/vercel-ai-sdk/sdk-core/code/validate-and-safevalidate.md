
# validateUIMessages & safeValidateUIMessages: Complete Guide (0-100%)

## What Do These Functions Do?

In plain English: These are "quality checkers" for your chat messages. They inspect UIMessages to make sure everything is correct before you use them.

Think of it like this:

```plaintext
You receive messages from the frontend/API
You want to make sure they have the right structure, types, and content
validateUIMessages and safeValidateUIMessages check if the messages follow all the rules
If something is wrong, they tell you exactly what the problem is
````

The main difference:

```plaintext
validateUIMessages - Throws an error if validation fails (crashes your program)
safeValidateUIMessages - Returns a result object instead (doesn't crash, lets you handle errors gracefully)
```

## What Gets Validated?

These functions check EVERYTHING about your messages:

### ✅ Basic Structure Checks

```plaintext
✓ Messages array is not empty
✓ Each message has: id, role (system|user|assistant), and at least one part
✓ Each part has a valid type (text, tool, reasoning, file, etc.)
```

### ✅ Message Parts Types

```plaintext
✓ Text parts: must have text content
✓ Reasoning parts: must have reasoning text
✓ File parts: must have mediaType and URL
✓ Tool parts: must have toolCallId, toolName, state
✓ Data parts: can have custom data with optional ID
✓ Source parts: must have sourceId and URL/document info
```

### ✅ Tool States

```plaintext
✓ input-streaming: tool is being called
✓ input-available: tool input ready
✓ output-available: tool executed successfully
✓ output-error: tool execution failed
✓ approval-requested: waiting for user approval
✓ approval-responded: user approved/denied
✓ output-denied: tool call was rejected
```

### ✅ Custom Validation (If Provided)

```plaintext
✓ Message metadata matches metadataSchema (Zod schema)
✓ Data parts match dataSchemas (one schema per data type)
✓ Tool inputs match tool's inputSchema
✓ Tool outputs match tool's outputSchema
```

## Comparison: Which One Should You Use?

| Aspect         | validateUIMessages     | safeValidateUIMessages                 |
| -------------- | ---------------------- | -------------------------------------- |
| Returns        | Valid messages array   | { success: true/false, data?, error? } |
| On Error       | Throws exception       | Returns error in result                |
| Use Case       | Backend route handlers | User input handling                    |
| Error Handling | Try/catch required     | Simple if check                        |
| Performance    | Identical              | Identical                              |

## What Errors Can Be Caught?

```plaintext
1. messages is null or undefined
   → "messages parameter must be provided"
2. messages array is empty
   → "Messages array must not be empty"
3. A message has no parts
   → "Message must contain at least one part"
4. Invalid message role
   → Must be: 'system', 'user', or 'assistant'
5. Invalid part type
   → Must be: 'text', 'reasoning', 'file', 'tool-*', 'data-*', etc.
6. Missing required fields (e.g., text in text part)
   → Detailed error about which field is missing
7. Tool input doesn't match schema
   → Detailed validation error from tool's inputSchema
8. Tool output doesn't match schema
   → Detailed validation error from tool's outputSchema
9. Metadata doesn't match metadataSchema
   → Detailed validation error for metadata fields
10. Data part doesn't match dataSchema
    → Detailed validation error for custom data
```

## How to Use: Basic Example

### Option 1: Using validateUIMessages (Throws on Error)

```typescript
import { validateUIMessages, UIMessage } from 'ai';

// Backend API route
export async function POST(req: Request) {
  const body = await req.json();
  
  try {
    // Validate messages from frontend
    const validMessages = await validateUIMessages({
      messages: body.messages,
    });
    
    console.log('✅ Messages are valid!', validMessages);
    // Now use validated messages safely
    
  } catch (error) {
    console.error('❌ Validation failed:', error.message);
    return Response.json(
      { error: error.message },
      { status: 400 }
    );
  }
}
```

### Option 2: Using safeValidateUIMessages (Doesn't Throw)

```typescript
import { safeValidateUIMessages, UIMessage } from 'ai';

export async function POST(req: Request) {
  const body = await req.json();
  
  // Validate messages - no try/catch needed
  const result = await safeValidateUIMessages({
    messages: body.messages,
  });
  
  if (!result.success) {
    // Error happened - handle it
    console.error('❌ Validation failed:', result.error.message);
    return Response.json(
      { error: result.error.message },
      { status: 400 }
    );
  }
  
  // Success - use validated messages
  const validMessages = result.data;
  console.log('✅ Messages are valid!', validMessages);
}
```

## Complete Example: Chat Backend with Gemini 3.0 Flash & Vercel v6

```typescript
// app/api/chat/route.ts
import {
  streamText,
  UIMessage,
  convertToModelMessages,
  validateUIMessages,
} from 'ai';
import { google } from '@ai-sdk/google';

export async function POST(req: Request) {
  const body = await req.json();

  try {
    // STEP 1: Validate incoming UIMessages
    const validatedMessages = await validateUIMessages({
      messages: body.messages,
    });

    // STEP 2: Convert UIMessages to ModelMessages for the AI
    const modelMessages = await convertToModelMessages(validatedMessages);

    // STEP 3: Call Gemini 3.0 Flash
    const result = streamText({
      model: google('gemini-3.0-flash-latest'),
      system: 'You are a helpful assistant.',
      messages: modelMessages,
    });

    // STEP 4: Return streamed response
    return result.toUIMessageStreamResponse();

  } catch (error: any) {
    console.error('Validation error:', error.message);
    return Response.json(
      { error: 'Invalid message format' },
      { status: 400 }
    );
  }
}
```

## Advanced: Validate with Custom Schemas

### Scenario: Message with Metadata

```typescript
import { validateUIMessages, UIMessage } from 'ai';
import { z } from 'zod';

// Define what metadata should look like
const metadataSchema = z.object({
  timestamp: z.number(),
  userId: z.string(),
  sessionId: z.string(),
});

type CustomMessage = UIMessage<{
  timestamp: number;
  userId: string;
  sessionId: string;
}>;

// Validate messages with metadata check
const messages = await validateUIMessages<CustomMessage>({
  messages: body.messages,
  metadataSchema, // ✅ Now validates metadata too
});
```

### Scenario: Messages with Custom Data Parts

```typescript
import { validateUIMessages, UIMessage } from 'ai';
import { z } from 'zod';

type CustomMessage = UIMessage<
  unknown,
  {
    weather: { temp: number; condition: string };
    location: { city: string; country: string };
  }
>;

// Validate messages with data part checks
const messages = await validateUIMessages<CustomMessage>({
  messages: body.messages,
  dataSchemas: {
    weather: z.object({
      temp: z.number(),
      condition: z.string(),
    }),
    location: z.object({
      city: z.string(),
      country: z.string(),
    }),
  },
});
```

### Scenario: Messages with Tool Calls

```typescript
import { validateUIMessages, UIMessage } from 'ai';
import { tool } from 'ai';
import { z } from 'zod';

// Define tools
const tools = {
  getWeather: tool({
    description: 'Get weather for a city',
    parameters: z.object({ city: z.string() }),
    outputSchema: z.object({
      temp: z.number(),
      condition: z.string(),
    }),
    execute: async ({ city }) => ({
      temp: 72,
      condition: 'Sunny',
    }),
  }),
  sendEmail: tool({
    description: 'Send an email',
    parameters: z.object({
      to: z.string().email(),
      subject: z.string(),
      body: z.string(),
    }),
    execute: async ({ to, subject, body }) => ({
      success: true,
      messageId: 'msg-123',
    }),
  }),
};

// Validate messages with tool call checks
const messages = await validateUIMessages({
  messages: body.messages,
  tools, // ✅ Validates all tool inputs/outputs
});
```

### Full Chat Application with All Validations

#### Backend: app/api/chat/route.ts

```typescript
import {
  streamText,
  UIMessage,
  convertToModelMessages,
  validateUIMessages,
} from 'ai';
import { google } from '@ai-sdk/google';
import { tool } from 'ai';
import { z } from 'zod';

// Define tools
const tools = {
  getWeather: tool({
    description: 'Get weather information for a city',
    parameters: z.object({
      city: z.string().describe('City name'),
    }),
    outputSchema: z.object({
      temp: z.number(),
      condition: z.string(),
      humidity: z.number(),
    }),
    execute: async ({ city }) => ({
      temp: 72,
      condition: 'Sunny',
      humidity: 65,
    }),
  }),
};

// Define message metadata schema
const metadataSchema = z.object({
  timestamp: z.number(),
  userId: z.string().optional(),
});

type CustomMessage = UIMessage<{
  timestamp: number;
  userId?: string;
}>;

export async function POST(req: Request) {
  try {
    const body = await req.json();

    // ✅ VALIDATE: Check message format, structure, and tool calls
    const validatedMessages = await validateUIMessages<CustomMessage>({
      messages: body.messages,
      metadataSchema,
      tools, // Validates tool input/output types
    });

    console.log('✅ Messages validated successfully');

    // ✅ CONVERT: Transform UIMessages to ModelMessages
    const modelMessages = await convertToModelMessages(validatedMessages, {
      tools,
    });

    // ✅ CALL: Stream response from Gemini 3.0 Flash
    const result = streamText({
      model: google('gemini-3.0-flash-latest'),
      system: `You are a helpful assistant with access to tools.
Current time: ${new Date().toISOString()}
You have access to tools like getWeather to help the user.`,
      messages: modelMessages,
      tools,
      toolChoice: 'auto',
    });

    // ✅ RETURN: Stream response as UIMessage format
    return result.toUIMessageStreamResponse();

  } catch (error: any) {
    console.error('❌ Error:', error.message);

    return Response.json(
      {
        error: error.message || 'Internal server error',
        code: error.code || 'UNKNOWN_ERROR',
      },
      { status: 400 }
    );
  }
}
```

#### Frontend: app/page.tsx

```typescript
'use client';

import { useChat } from '@ai-sdk/react';
import { UIMessage } from 'ai';
import { useState } from 'react';

export default function ChatPage() {
  const [input, setInput] = useState('');
  const { messages, sendMessage, isLoading } = useChat({
    api: '/api/chat',
  });

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!input.trim()) return;

    // Frontend automatically sends UIMessages
    // Backend will validate them
    sendMessage({ text: input });
    setInput('');
  };

  return (
    <div className="flex flex-col h-screen w-full max-w-2xl mx-auto p-4">
      {/* Messages Display */}
      <div className="flex-1 overflow-y-auto mb-4 space-y-4">
        {messages.map((message: UIMessage) => (
          <div
            key={message.id}
            className={`p-4 rounded-lg ${
              message.role === 'user'
                ? 'bg-blue-500 text-white ml-auto max-w-xs'
                : 'bg-gray-200 mr-auto max-w-xs'
            }`}
          >
            {message.parts.map((part, i) => (
              <div key={i}>
                {part.type === 'text' && (
                  <p>{part.text}</p>
                )}
                {part.type === 'tool-getWeather' && (
                  <div className="mt-2 p-2 bg-white rounded text-black">
                    <p className="font-bold">🌤️ Weather Check</p>
                    {part.state === 'input-available' && (
                      <p>Checking weather for: {part.input.city}</p>
                    )}
                    {part.state === 'output-available' && (
                      <div>
                        <p>🌡️ {part.output.temp}°F</p>
                        <p>💨 Humidity: {part.output.humidity}%</p>
                        <p>☁️ {part.output.condition}</p>
                      </div>
                    )}
                  </div>
                )}
              </div>
            ))}
          </div>
        ))}
      </div>

      {/* Input Form */}
      <form onSubmit={handleSubmit} className="flex gap-2 mt-4">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask about weather or anything else..."
          disabled={isLoading}
          className="flex-1 p-2 border rounded-lg focus:outline-none focus:ring-2 focus:ring-blue-500"
        />
        <button
          type="submit"
          disabled={isLoading}
          className="px-6 py-2 bg-blue-500 text-white rounded-lg hover:bg-blue-600 disabled:bg-gray-400"
        >
          {isLoading ? 'Sending...' : 'Send'}
        </button>
      </form>
    </div>
  );
}
```

## Error Handling Examples

### Example 1: Catching Invalid Tool Input

```typescript
const result = await validateUIMessages({
  messages: [
    {
      id: '1',
      role: 'assistant',
      parts: [
        {
          type: 'tool-getWeather',
          toolCallId: '123',
          state: 'output-available',
          input: { city: 123 }, // ❌ WRONG: should be string
          output: { temp: 72, condition: 'Sunny', humidity: 65 },
        },
      ],
    },
  ],
  tools: { getWeather }, // Provides schema validation
});

// Result: Error thrown!
// "Tool input 'city' must be a string, got number"
```

### Example 2: Using safeValidateUIMessages Instead

```typescript
const result = await safeValidateUIMessages({
  messages: [
    {
      id: '1',
      role: 'assistant',
      parts: [
        {
          type: 'tool-getWeather',
          toolCallId: '123',
          state: 'output-available',
          input: { city: 123 }, // ❌ WRONG
          output: { temp: 72, condition: 'Sunny', humidity: 65 },
        },
      ],
    },
  ],
  tools: { getWeather },
});

if (!result.success) {
  // Handle error gracefully
  console.log('Validation failed:', result.error.message);
  // Don't crash, return error response to user
} else {
  // Use result.data
}
```

### Example 3: Metadata Validation Error

```typescript
const metadataSchema = z.object({
  userId: z.string().uuid(),
  timestamp: z.number().positive(),
});

const result = await safeValidateUIMessages({
  messages: [
    {
      id: '1',
      role: 'user',
      metadata: {
        userId: 'not-a-uuid', // ❌ WRONG: not UUID format
        timestamp: -100, // ❌ WRONG: negative
      },
      parts: [{ type: 'text', text: 'Hello' }],
    },
  ],
  metadataSchema,
});

if (!result.success) {
  // result.error contains details about what's wrong
}
```

## When to Use Each Function

### ✅ Use validateUIMessages When:

```plaintext
Backend API route handler (can use try/catch)
You want errors to stop processing immediately
You're okay with throwing exceptions
Server-side validation in controlled environment
```

### ✅ Use safeValidateUIMessages When:

```plaintext
User-facing API handling
You need graceful error handling
Chaining multiple validations
Building robust error reporting
CLI tools and data processing scripts
```

## Validation Checklist

Always validate messages when:

```plaintext
✅ Receiving from frontend/client
✅ Receiving from API requests
✅ Loading from database/file
✅ Processing user-submitted data
✅ Working with tool calls
✅ Handling custom metadata
✅ Using custom data parts
```

Skip validation when:

```plaintext
❌ Creating messages in your own code (if 100% sure)
❌ Messages from useChat hook (already guaranteed to be UIMessage)
❌ Using TypeScript strictly (compile-time checks)
❌ In tight loops (validate once at entry point instead)
```

## Performance Notes

Validation is fast because:

```plaintext
Runs once per message batch (not per message)
Uses Zod schemas (very optimized)
Minimal string parsing
Short-circuits on first error
```

### Optimization tips

```typescript
// ✅ GOOD: Validate once at entry point
const validMessages = await validateUIMessages({ messages });
const modelMessages = await convertToModelMessages(validMessages);
// Use validMessages throughout function

// ❌ BAD: Don't validate repeatedly
for (const msg of messages) {
  await validateUIMessages({ messages: [msg] }); // Wasteful!
}
```

## Complete Summary Table

| Feature                | validateUIMessages    | safeValidateUIMessages              |
| ---------------------- | --------------------- | ----------------------------------- |
| Purpose                | Strict validation     | Graceful validation                 |
| Error Handling         | Throws exception      | Returns result object               |
| Return Type            | Promise<UIMessage[]>  | Promise<{ success, data?, error? }> |
| Use in Routes          | ✅ With try/catch      | ✅ Without try/catch                 |
| Validates Structure    | ✅ Yes                 | ✅ Yes                               |
| Validates Metadata     | ✅ If schema provided  | ✅ If schema provided                |
| Validates Tool Inputs  | ✅ If tools provided   | ✅ If tools provided                 |
| Validates Tool Outputs | ✅ If schemas provided | ✅ If schemas provided               |
| Validates Data Parts   | ✅ If schemas provided | ✅ If schemas provided               |
| Can be chained         | ❌ Error stops chain   | ✅ Can continue                      |


|

```

---

This is now fully **MDX-ready** with headings, code blocks, tables, and checklists. ✅  

If you want, I can also **add collapsible sections** for examples to make it much cleaner for MDX docs. That makes it super readable without scrolling endlessly.  

Do you want me to do that next?
```
