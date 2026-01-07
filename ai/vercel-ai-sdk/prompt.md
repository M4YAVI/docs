Without AI SDK: Learn OpenAI's way + Google's way + Anthropic's way
With AI SDK: Learn one way → works with everyone

# prompt

# Prompts: How to Talk to AI Models

## The Big Picture
A prompt is your instruction to an AI. Like asking someone a question: clear instructions = better answers.

---

## Three Ways to Prompt

### 1. **Text Prompts** (The Simple Way)
**What**: Just a string of text  
**When to use**: Simple, one-off requests

```ts
prompt: 'Invent a new holiday and describe its traditions.'
```

**With variables** (using template literals):
```ts
prompt: `Trip to ${destination} for ${lengthOfStay} days. Suggest activities.`
```

---

### 2. **System Prompts** (The Instruction Manual)
**What**: Background instructions that guide the AI's behavior for ALL responses  
**When to use**: Setting personality, rules, or context that applies to everything

```ts
system: 'You help plan travel. Always respond with a list of activities.',
prompt: 'Trip to Paris for 3 days.'
```

**Key point**: System prompt = permanent background instructions. User prompt = the actual question.

---

### 3. **Message Prompts** (The Conversation)
**What**: Array of back-and-forth messages (like a chat history)  
**When to use**: Chat interfaces, complex multi-turn conversations

```ts
messages: [
  { role: 'user', content: 'Hi!' },
  { role: 'assistant', content: 'Hello, how can I help?' },
  { role: 'user', content: 'Where to eat in Berlin?' }
]
```

---

## Message Types & Their Parts

### **User Messages** (What you send)

#### Text Part:
```ts
{ role: 'user', content: 'Where to eat in Berlin?' }
// OR with explicit type:
{ role: 'user', content: [{ type: 'text', text: 'Where to eat?' }] }
```

#### Image Part (multi-modal):
```ts
{ 
  role: 'user', 
  content: [
    { type: 'text', text: 'Describe this image.' },
    { type: 'image', image: Buffer }  // or URL, base64 string, etc.
  ]
}
```

#### File Part (PDFs, audio, etc.):
```ts
{ 
  role: 'user', 
  content: [
    { type: 'text', text: 'Summarize this PDF.' },
    { 
      type: 'file', 
      mediaType: 'application/pdf',
      data: fs.readFileSync('./doc.pdf')
    }
  ]
}
```

---

### **Assistant Messages** (What AI sends back)

#### Regular response:
```ts
{ role: 'assistant', content: 'Here are the best restaurants...' }
```

#### Tool call (AI asking to use a function):
```ts
{ 
  role: 'assistant', 
  content: [{
    type: 'tool-call',
    toolCallId: '12345',
    toolName: 'get-nutrition-data',
    input: { cheese: 'Roquefort' }
  }]
}
```

---

### **Tool Messages** (Function results you send back)

```ts
{ 
  role: 'tool', 
  content: [{
    type: 'tool-result',
    toolCallId: '12345',  // Must match the assistant's tool-call ID
    toolName: 'get-nutrition-data',
    output: { calories: 369, fat: 31, protein: 22 }
  }]
}
```

**The flow**:
1. User asks → "How many calories in this cheese?"
2. Assistant calls tool → `get-nutrition-data(cheese: 'Roquefort')`
3. You run the function → Get result
4. You send tool message back → `{ calories: 369 }`
5. Assistant uses that data → Gives final answer

---

### **System Messages** (Alternative to `system` property)

```ts
{ role: 'system', content: 'You are a travel planner.' }
```

Same as using `system:` property, but inside the messages array.

---

## Provider Options (Advanced Customization)

Some providers (OpenAI, Anthropic, etc.) have special features. You can enable them at 3 levels:

### 1. **Function Level** (applies to whole request):
```ts
providerOptions: {
  openai: { reasoningEffort: 'low' }
}
```

### 2. **Message Level** (applies to one message):
```ts
{
  role: 'system',
  content: 'Cached system message',
  providerOptions: {
    anthropic: { cacheControl: { type: 'ephemeral' } }
  }
}
```

### 3. **Message Part Level** (applies to one image/text within a message):
```ts
{
  type: 'image',
  image: 'url-to-image',
  providerOptions: {
    openai: { imageDetail: 'low' }  // Use low-res version to save tokens
  }
}
```

---

## Key Mental Model

```
Text Prompt = Single question
System Prompt = Rulebook that applies to everything
Message Prompt = Full conversation with context

User role = You talking
Assistant role = AI talking  
Tool role = Function results
System role = Background instructions
```

**Parts within messages** let you mix text, images, files, and tool calls in one message.
