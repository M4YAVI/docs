

---

# **Complete Guide: `createMCPClient` & `Experimental_StdioMCPTransport` (0-100%)**

## **Plain English Explanation**

### **What is MCP?**
MCP (Model Context Protocol) is like a **universal translator** that lets AI models (like OpenAI, Gemini, etc.) talk to and use tools from remote servers in a standardized way. Think of it as a bridge between your AI and external services/tools.

### **What Does `createMCPClient` Do?**
`createMCPClient` is a **factory function** that creates a client object to connect to an MCP server. It's like building a bridge that allows your AI application to:
- 🔧 Access **tools** (functions the server provides)
- 📚 Read **resources** (data/files from the server)
- 💬 Retrieve **prompts** (pre-built AI instructions)
- 🎯 Handle **elicitation** (server asking for user input during execution)

**In plain terms**: It's your gateway to remote tools and data that your AI can use.

### **What Does `Experimental_StdioMCPTransport` Do?**
`Experimental_StdioMCPTransport` is a **communication mechanism** (transport layer) that connects your client to an MCP server using **stdin/stdout** (standard input/output streams). 

Think of it like two programs talking through pipes:
- Your app writes messages to the server's stdin
- The server writes responses to stdout
- Your app reads those responses

---

## **When & Where to Use Each**

### **`createMCPClient`**
**WHEN to use:**
- You need to connect to an MCP server
- You want AI models to access external tools
- You're building an AI application with tool use

**WHERE to use:**
- ✅ Backend/Server (Node.js)
- ✅ API routes (Next.js)
- ✅ Command-line tools
- ❌ Browser/Frontend (not supported)

### **`Experimental_StdioMCPTransport`**
**WHEN to use:**
- You have a local MCP server running
- You're in development/testing
- You want the simplest local setup

**WHERE NOT to use:**
- ❌ **Production deployments** (can't be deployed to serverless)
- ❌ **Remote servers** (use HTTP/SSE instead)
- ❌ **Browser environments**

---

## **Complete Technical Breakdown**

### **1. `createMCPClient` - Full API**

```typescript
import { createMCPClient } from '@ai-sdk/mcp';

// Signature
const client = await createMCPClient({
  transport: MCPTransportConfig | MCPTransport,
  name?: string,                          // e.g., "my-ai-client"
  version?: string,                       // e.g., "1.0.0"
  onUncaughtError?: (error: unknown) => void,
  capabilities?: ClientCapabilities,     // e.g., { elicitation: {} }
});
```

**Returns an MCPClient with these methods:**

| Method | Purpose | Returns |
|--------|---------|---------|
| `tools(options?)` | Get all tools from server with optional schema definitions | `Promise<McpToolSet>` |
| `listResources(options?)` | List all available resources | `Promise<ListResourcesResult>` |
| `readResource({uri})` | Read a specific resource by URI | `Promise<ReadResourceResult>` |
| `listResourceTemplates()` | List dynamic resource URI patterns | `Promise<ListResourceTemplatesResult>` |
| `experimental_listPrompts()` | List available prompts | `Promise<ListPromptsResult>` |
| `experimental_getPrompt({name, arguments?})` | Get prompt messages | `Promise<GetPromptResult>` |
| `onElicitationRequest(schema, handler)` | Handle server's input requests | `void` |
| `close()` | Disconnect and cleanup | `Promise<void>` |

---

### **2. `Experimental_StdioMCPTransport` - Full API**

```typescript
import { Experimental_StdioMCPTransport } from '@ai-sdk/mcp/mcp-stdio';

const transport = new Experimental_StdioMCPTransport({
  command: 'node',                    // Command to run the MCP server
  args?: ['script.js'],               // Arguments for the command
  env?: { KEY: 'value' },             // Environment variables
  stderr?: 'pipe' | Stream | number,  // Where to pipe stderr
  cwd?: '/path/to/dir',               // Working directory
});

// Then pass to createMCPClient
const client = await createMCPClient({ transport });
```

**What it does internally:**
1. Spawns a child process running your MCP server
2. Creates pipes for stdin/stdout communication
3. Sends/receives JSON-RPC messages line-by-line
4. Each message ends with a newline (`\n`)

---

## **Step-by-Step Implementation Guide**

### **Step 0: Setup**

```bash
npm install @ai-sdk/mcp @ai-sdk/openai ai dotenv
# For Gemini support:
npm install @ai-sdk/google
# For Vercel AI SDK v6 with TypeScript:
npm install --save-exact ai@6
```

---

### **Step 1: Create Your MCP Server**

Create `mcp-server.js`:

```javascript
// Example MCP server that provides a simple tool
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';

const server = new Server({
  name: 'example-server',
  version: '1.0.0',
});

// Define a tool
server.setRequestHandler('tools/list', async () => {
  return {
    tools: [
      {
        name: 'get_weather',
        description: 'Get weather for a city',
        inputSchema: {
          type: 'object',
          properties: {
            city: { type: 'string', description: 'City name' },
          },
          required: ['city'],
        },
      },
    ],
  };
});

// Handle tool calls
server.setRequestHandler('tools/call', async (request) => {
  if (request.params.name === 'get_weather') {
    const city = request.params.arguments.city;
    return {
      content: [
        {
          type: 'text',
          text: `Weather in ${city}: Sunny, 72°F`,
        },
      ],
    };
  }
  throw new Error('Unknown tool');
});

// Start the server
const transport = new StdioServerTransport();
await server.connect(transport);
```

---

### **Step 2: Basic Client with Stdio Transport (Local)**

Create `client.ts`:

```typescript
import { createMCPClient } from '@ai-sdk/mcp';
import { Experimental_StdioMCPTransport } from '@ai-sdk/mcp/mcp-stdio';
import { generateText } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

async function main() {
  let client;

  try {
    // Step 1: Create stdio transport (spawns your MCP server)
    const transport = new Experimental_StdioMCPTransport({
      command: 'node',
      args: ['mcp-server.js'],
      env: {
        // Pass environment variables if needed
        NODE_ENV: 'production',
      },
    });

    // Step 2: Create MCP client connected to the transport
    client = await createMCPClient({
      transport,
      name: 'my-ai-app',
      version: '1.0.0',
    });

    // Step 3: Get tools from the server
    const tools = await client.tools({
      schemas: {
        get_weather: {
          inputSchema: z.object({
            city: z.string().describe('The city name'),
          }),
        },
      },
    });

    // Step 4: Use with OpenAI's GPT-4o
    const { text } = await generateText({
      model: openai('gpt-4o'),
      tools,
      prompt: 'What is the weather in New York?',
    });

    console.log('Response:', text);
  } finally {
    // Always cleanup!
    await client?.close();
  }
}

main().catch(console.error);
```

---

### **Step 3: Using with Google Gemini (v6)**

```typescript
import { createMCPClient } from '@ai-sdk/mcp';
import { Experimental_StdioMCPTransport } from '@ai-sdk/mcp/mcp-stdio';
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

async function main() {
  let client;

  try {
    const transport = new Experimental_StdioMCPTransport({
      command: 'node',
      args: ['mcp-server.js'],
    });

    client = await createMCPClient({
      transport,
      name: 'gemini-mcp-app',
    });

    const tools = await client.tools();

    // Use with Gemini 2.0 Flash (fastest model)
    const { text } = await generateText({
      model: google('gemini-2.0-flash'),  // Latest & fastest
      tools,
      prompt: 'Tell me the weather in London',
      system: 'You are a helpful weather assistant',
    });

    console.log(text);
  } finally {
    await client?.close();
  }
}

main().catch(console.error);
```

---

### **Step 4: Production Setup - HTTP Transport (NOT Stdio)**

```typescript
import { createMCPClient } from '@ai-sdk/mcp';
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

async function main() {
  let client;

  try {
    // ✅ For production: use HTTP transport
    client = await createMCPClient({
      transport: {
        type: 'http',  // Can be deployed!
        url: 'https://your-mcp-server.com/mcp',
        headers: {
          Authorization: `Bearer ${process.env.MCP_API_KEY}`,
        },
      },
      name: 'production-app',
    });

    const tools = await client.tools();

    const { text } = await generateText({
      model: google('gemini-2.0-flash'),
      tools,
      prompt: 'Get me some data',
    });

    console.log(text);
  } finally {
    await client?.close();
  }
}

main().catch(console.error);
```

---

### **Step 5: Advanced - Handling Elicitation (Server Asking for User Input)**

```typescript
import { createMCPClient } from '@ai-sdk/mcp';
import { Experimental_StdioMCPTransport } from '@ai-sdk/mcp/mcp-stdio';
import { generateText, stepCountIs } from 'ai';
import { google } from '@ai-sdk/google';
import { ElicitationRequestSchema } from '@ai-sdk/mcp';

async function main() {
  let client;

  try {
    const transport = new Experimental_StdioMCPTransport({
      command: 'node',
      args: ['mcp-server.js'],
    });

    client = await createMCPClient({
      transport,
      // Tell server we support elicitation
      capabilities: {
        elicitation: {},
      },
    });

    // Register handler for when server needs user input
    client.onElicitationRequest(ElicitationRequestSchema, async (request) => {
      console.log('Server is asking:', request.params.message);
      
      // In real app, get input from user
      const userInput = await getUserInput(request.params.requestedSchema);
      
      return {
        action: 'accept',
        content: userInput,
      };
    });

    const tools = await client.tools();

    const { text } = await generateText({
      model: google('gemini-2.0-flash'),
      tools,
      stopWhen: stepCountIs(10),  // Max 10 steps
      prompt: 'Help me register for a service',
    });

    console.log(text);
  } finally {
    await client?.close();
  }
}
```

---

### **Step 6: Using in Next.js Route Handler (Vercel)**

Create `app/api/mcp-chat/route.ts`:

```typescript
import { createMCPClient } from '@ai-sdk/mcp';
import { streamText } from 'ai';
import { google } from '@ai-sdk/google';

export const maxDuration = 60;  // For Vercel Pro

export async function POST(req: Request) {
  let client;

  try {
    const { message } = await req.json();

    // For Vercel deployments: use HTTP transport, not stdio
    client = await createMCPClient({
      transport: {
        type: 'http',
        url: process.env.MCP_SERVER_URL!,
        headers: {
          Authorization: `Bearer ${process.env.MCP_API_KEY}`,
        },
      },
    });

    const tools = await client.tools();

    const stream = await streamText({
      model: google('gemini-2.0-flash'),
      tools,
      messages: [{ role: 'user', content: message }],
      system: 'You are a helpful AI assistant with access to MCP tools',
    });

    return stream.toDataStreamResponse();
  } finally {
    await client?.close();
  }
}
```

---

### **Step 7: Full Example with Schema Validation**

```typescript
import { createMCPClient } from '@ai-sdk/mcp';
import { Experimental_StdioMCPTransport } from '@ai-sdk/mcp/mcp-stdio';
import { generateText, stepCountIs } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

async function main() {
  let client;

  try {
    const transport = new Experimental_StdioMCPTransport({
      command: 'node',
      args: ['mcp-server.js'],
      cwd: process.cwd(),
      env: {
        ...process.env,
        DEBUG: '1',
      },
    });

    client = await createMCPClient({
      transport,
      name: 'advanced-app',
      onUncaughtError: (error) => {
        console.error('Uncaught error in MCP client:', error);
      },
    });

    // Schema definition for type safety
    const tools = await client.tools({
      schemas: {
        get_weather: {
          inputSchema: z.object({
            city: z.string().describe('City name'),
            units: z.enum(['celsius', 'fahrenheit']).optional(),
          }),
        },
        get_forecast: {
          inputSchema: z.object({
            city: z.string(),
            days: z.number().min(1).max(7),
          }),
        },
        // Tool with no inputs
        get_current_location: {
          inputSchema: z.object({}),
        },
      },
    });

    const { text, toolResults } = await generateText({
      model: google('gemini-2.0-flash'),
      tools,
      stopWhen: stepCountIs(5),
      maxSteps: 5,
      prompt: 'Get me the 5-day forecast for Paris in Celsius',
      system:
        'You are a weather assistant. Use available tools to provide accurate information.',
    });

    console.log('=== Final Response ===');
    console.log(text);
    console.log('\n=== Tool Calls Made ===');
    console.log(JSON.stringify(toolResults, null, 2));
  } finally {
    await client?.close();
  }
}

main().catch(console.error);
```

---

## **Key Differences: When to Use Each Transport**

| Feature | Stdio | HTTP | SSE |
|---------|-------|------|-----|
| **Local Development** | ✅ Best | ❌ Needs server | ❌ Needs server |
| **Production** | ❌ No | ✅ Yes | ✅ Yes |
| **Vercel/Serverless** | ❌ No | ✅ Yes | ✅ Yes |
| **Setup Complexity** | 🟢 Simple | 🟡 Medium | 🟡 Medium |
| **Real-time Streaming** | ✅ Yes | ✅ Yes | ✅ Yes |
| **Custom Transports** | ✅ Can create | ✅ Yes | ✅ Yes |

---

## **Complete Architecture Diagram**

```
┌─────────────────────────────────────────────────────────────┐
│                      Your AI App                             │
│           (Node.js, Next.js, CLI, etc.)                      │
└────────────────┬────────────────────────────────────────────┘
                 │
                 │ createMCPClient()
                 ▼
┌─────────────────────────────────────────────────────────────┐
│              MCP Client (MCPClient)                           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ Tools        │ Resources    │ Prompts   │Elicitation│    │
│  └─────────────────────────────────────────────────────┘    │
└────────────────┬────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
  Stdio        HTTP          SSE
Transport   Transport    Transport
    │            │            │
    ├─DEVELOPMENT        PRODUCTION─────────┐
    │                                        │
    ▼                                        ▼
Local MCP Server              Remote MCP Server
(node mcp-server.js)          (https://api.example.com/mcp)
```

---

## **Error Handling Best Practices**

```typescript
try {
  client = await createMCPClient({
    transport,
    onUncaughtError: (error) => {
      console.error('MCP Client Error:', error);
      // Send to error tracking (Sentry, etc.)
    },
  });

  const tools = await client.tools();
  // Use tools...
} catch (error) {
  if (error instanceof MCPClientError) {
    console.error('MCP-specific error:', error.message);
  } else if (error instanceof TypeError) {
    console.error('Connection error:', error.message);
  }
} finally {
  // CRITICAL: Always close
  if (client) {
    await client.close();
  }
}
```

---

## **Performance Tips**

1. **Reuse clients** - Don't create new clients per request in production
2. **Close properly** - Always await `client.close()`
3. **Use schemas** - Define schemas for faster type inference
4. **Limit tools** - Only load tools you actually use
5. **Set timeouts** - Use request options to prevent hanging

---

## **Common Mistakes**

❌ **WRONG:**
```typescript
// Creating new transport each time
for (let i = 0; i < 100; i++) {
  const transport = new Experimental_StdioMCPTransport(...);
  const client = await createMCPClient({ transport });
  // Doesn't close properly!
}
```

✅ **RIGHT:**
```typescript
const client = await createMCPClient({...});
try {
  for (let i = 0; i < 100; i++) {
    const tools = await client.tools();
    // Use tools...
  }
} finally {
  await client.close();  // Once!
}
```

---

## **Summary Table**

| Concept | What It Does | When To Use |
|---------|-------------|-----------|
| **createMCPClient** | Creates connection to MCP server | Always first step |
| **Experimental_StdioMCPTransport** | Pipe-based local communication | Local dev only |
| **HTTP Transport** | HTTP requests to remote server | Production |
| **SSE Transport** | Server-Sent Events streaming | Alternative HTTP |
| **tools()** | Get executable functions | For AI tool use |
| **listResources()** | Get available data | To provide context |
| **readResource()** | Fetch resource data | When you need data |
| **onElicitationRequest()** | Handle server input requests | Interactive tools |

---

This covers everything from 0-100%! 🎯
