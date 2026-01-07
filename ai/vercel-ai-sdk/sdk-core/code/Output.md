

---

# **OUTPUT - Complete 0-100% Guide**

## **What is Output? (Plain English)**

`Output` in the Vercel AI SDK is a **blueprint that tells the AI model what format you want the response in**. Think of it like giving the AI instructions before it speaks to you: "I want the answer as plain text," or "I want a JSON object with specific fields," or "I want a list of items."

**Simply put:** Output = **Data format specification** for AI responses.

---

## **Why Do You Need It?**

Without `Output`, the AI returns random text that you have to manually parse and validate. With `Output`, the AI SDK:
- ✅ **Automatically validates** the response matches your requirements
- ✅ **Converts the response** to the correct data type
- ✅ **Provides type safety** in TypeScript
- ✅ **Handles errors** when the AI doesn't follow the format

---

## **All Output Types (Complete Overview)**

### **1. `Output.text()` - Plain Text (Default)**

**What it does:** Generates regular text without any schema validation.

```typescript
import { generateText, Output } from 'ai';
import { openai } from '@ai-sdk/openai';

const { output } = await generateText({
  model: openai('gpt-4o-mini'),
  output: Output.text(), // ← Plain text
  prompt: 'Tell me a joke.',
});

console.log(output); // Returns: "Why did the chicken cross the road?..."
```

**When to use:**
- Writing essays, stories, or general content
- Conversations that don't need structured data
- Chat responses
- Content generation

**What you get:** `string`

---

### **2. `Output.object()` - Structured Data (Most Common)**

**What it does:** Forces the AI to return a validated object matching your schema.

```typescript
import { generateText, Output } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const { output } = await generateText({
  model: openai('gpt-4o-mini'),
  output: Output.object({
    schema: z.object({
      name: z.string(),
      age: z.number(),
      city: z.string(),
    }),
  }),
  prompt: 'Generate a user profile.',
});

console.log(output); 
// Returns: { name: "John", age: 30, city: "New York" }
```

**Advanced example with optional fields:**

```typescript
output: Output.object({
  schema: z.object({
    name: z.string(),
    age: z.number().nullable(), // Can be null
    labels: z.array(z.string()), // Array of strings
    isActive: z.boolean().optional(), // Optional field
  }),
  name: 'UserProfile', // Optional: helps AI understand what to generate
  description: 'A user profile with name, age, and labels', // Optional guidance
})
```

**When to use:**
- Extracting structured data from text
- Form data generation
- Database records
- API responses
- Classification tasks
- Data extraction from documents

**What you get:** `{ name: string, age: number, city: string }`

---

### **3. `Output.array()` - Lists of Items**

**What it does:** Generates an array of validated objects.

```typescript
import { generateText, Output } from 'ai';
import { openai } from '@ai-sdk/openai';
import { z } from 'zod';

const { output } = await generateText({
  model: openai('gpt-4o-mini'),
  output: Output.array({
    element: z.object({
      location: z.string(),
      temperature: z.number(),
      condition: z.string(),
    }),
  }),
  prompt: 'List the weather for San Francisco and Paris.',
});

console.log(output);
// Returns: [
//   { location: "San Francisco", temperature: 72, condition: "sunny" },
//   { location: "Paris", temperature: 65, condition: "rainy" }
// ]
```

**With streaming (get items as they're generated):**

```typescript
import { streamText, Output } from 'ai';
import { z } from 'zod';

const { elementStream } = streamText({
  model: openai('gpt-4o-mini'),
  output: Output.array({
    element: z.object({
      name: z.string(),
      class: z.string(),
      description: z.string(),
    }),
  }),
  prompt: 'Generate 3 heroes for a fantasy game.',
});

// Get items as soon as they're complete and validated
for await (const hero of elementStream) {
  console.log('Hero:', hero); // Each hero is fully validated
}
```

**When to use:**
- Generating lists (products, users, recipes)
- Batch data extraction
- Creating multiple items at once
- Real-time processing with `elementStream`

**What you get:** `Array<{ location: string, temperature: number, condition: string }>`

---

### **4. `Output.choice()` - Pick from Options**

**What it does:** AI must pick exactly one answer from your predefined list.

```typescript
import { generateText, Output } from 'ai';
import { openai } from '@ai-sdk/openai';

const { output } = await generateText({
  model: openai('gpt-4o-mini'),
  output: Output.choice({
    options: ['sunny', 'rainy', 'snowy', 'cloudy'],
  }),
  prompt: 'What is the weather today?',
});

console.log(output); // Returns: "sunny" (guaranteed to be one of the options)
```

**When to use:**
- Classification/categorization
- Sentiment analysis (positive, negative, neutral)
- Multiple choice questions
- Fixed enum values
- Quality assurance (no random answers)

**What you get:** `'sunny' | 'rainy' | 'snowy' | 'cloudy'`

---

### **5. `Output.json()` - Unstructured JSON**

**What it does:** AI returns any valid JSON without enforcing a specific structure.

```typescript
import { generateText, Output } from 'ai';
import { openai } from '@ai-sdk/openai';

const { output } = await generateText({
  model: openai('gpt-4o-mini'),
  output: Output.json(),
  prompt: 'Return weather data as JSON.',
});

console.log(output);
// Returns: ANY valid JSON - could be an object, array, string, number, etc.
// { temp: 72, city: "NYC" } or [1, 2, 3] or "hello" - no validation!
```

**When to use:**
- When you need flexibility in the structure
- Experimental/prototype code
- When structure varies
- For testing

**⚠️ WARNING:** No schema validation! Use `Output.object()` for type safety.

**What you get:** `any` (could be anything valid in JSON)

---

## **How Output Works Under the Hood**

```
┌─────────────────────┐
│  You provide Output │ (e.g., Output.object({ schema }))
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────┐
│ AI SDK sends the schema to the AI model │ (via JSON Schema format)
│ "Generate ONLY JSON matching this"      │
└──────────┬──────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────┐
│ AI model generates the response     │
│ (tries to match the schema)         │
└──────────┬──────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────┐
│ SDK Validation Step:                         │
│ 1. Parse the response (JSON parsing)        │
│ 2. Validate against schema (Zod/JSON-schema)│
│ 3. Return validated data or throw error     │
└──────────┬───────────────────────────────────┘
           │
           ▼
┌─────────────────────┐
│  Your typed result  │
│  (fully validated)  │
└─────────────────────┘
```

---

## **Real-World Examples with Vercel v6 + Gemini 3.0 Flash**

### **Example 1: Generate User Profiles**

```typescript
import { generateText, Output } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

const userSchema = z.object({
  name: z.string(),
  email: z.string().email(),
  age: z.number().min(18).max(120),
  role: z.enum(['admin', 'user', 'guest']),
  hobbies: z.array(z.string()),
});

const { output } = await generateText({
  model: google('gemini-3.0-flash'),
  output: Output.object({
    schema: userSchema,
    name: 'User',
    description: 'A user profile with contact and preferences',
  }),
  prompt: 'Generate a realistic user profile',
});

console.log(output.name); // TypeScript knows this is a string!
console.log(output.email); // TypeScript knows this is a string!
```

### **Example 2: Extract Data from Text**

```typescript
const { output } = await generateText({
  model: google('gemini-3.0-flash'),
  output: Output.object({
    schema: z.object({
      entities: z.array(z.object({
        name: z.string(),
        type: z.enum(['person', 'place', 'organization']),
      })),
      sentiment: z.enum(['positive', 'negative', 'neutral']),
    }),
  }),
  prompt: 'Extract entities and sentiment from: "Apple Inc. opened a new store in New York yesterday!"',
});

// output.entities is guaranteed to be an array of objects with name/type
// output.sentiment is guaranteed to be one of the three options
```

### **Example 3: Batch Process with Streaming**

```typescript
import { streamText, Output } from 'ai';
import { google } from '@ai-sdk/google';

const { elementStream } = streamText({
  model: google('gemini-3.0-flash'),
  output: Output.array({
    element: z.object({
      productId: z.string(),
      price: z.number(),
      inStock: z.boolean(),
    }),
  }),
  prompt: 'Generate 10 product listings with prices',
});

// Each product arrives as soon as it's complete and validated
for await (const product of elementStream) {
  console.log(`Product ${product.productId}: $${product.price}`);
  // Process item immediately
}
```

### **Example 4: Classification**

```typescript
const { output } = await generateText({
  model: google('gemini-3.0-flash'),
  output: Output.choice({
    options: ['spam', 'promotional', 'important', 'general'],
  }),
  prompt: 'Classify this email: "Dear customer, your order has shipped!"',
});

// output is GUARANTEED to be one of the 4 options
switch(output) {
  case 'spam': console.log('Filter email'); break;
  case 'promotional': console.log('Archive'); break;
  case 'important': console.log('Highlight'); break;
  case 'general': console.log('Normal inbox'); break;
}
```

---

## **Error Handling**

```typescript
import { generateText, Output, NoObjectGeneratedError } from 'ai';

try {
  const { output } = await generateText({
    model: google('gemini-3.0-flash'),
    output: Output.object({
      schema: z.object({
        name: z.string(),
        age: z.number(),
      }),
    }),
    prompt: 'Generate data',
  });
} catch (error) {
  if (NoObjectGeneratedError.isInstance(error)) {
    console.log('Failed to generate valid output');
    console.log('Error:', error.message);
    console.log('Original text:', error.text);
    console.log('Cause:', error.cause);
  }
}
```

---

## **When to Use Each Output Type**

| Output Type | Use Case | Example |
|---|---|---|
| **`text()`** | Plain text content | Stories, essays, chat |
| **`object()`** | Single structured item | User profile, product info |
| **`array()`** | Multiple structured items | List of products, weather for cities |
| **`choice()`** | Pick from fixed options | Classification, sentiment |
| **`json()`** | Flexible JSON | Experimental, prototyping |

---

## **Key Points to Remember**

✅ **Always use `Output.object()` or `Output.array()` for production** - gives you type safety and validation

✅ **Use schemas (Zod)** - makes your code more maintainable

✅ **Add `name` and `description`** to help the AI understand better

✅ **Error handling is important** - catch `NoObjectGeneratedError`

✅ **Streaming works great with `Output.array()`** - get items as they're generated

✅ **Never use `Output.json()` without validation** - defeats the purpose

---

## **Setup Instructions (Vercel v6 + Gemini 3.0 Flash)**

```bash
npm install ai @ai-sdk/google zod
```

```typescript
import { generateText, Output } from 'ai';
import { google } from '@ai-sdk/google';
import { z } from 'zod';

// Simple example
const { output } = await generateText({
  model: google('gemini-3.0-flash'), // ← Gemini 3.0 Flash model
  output: Output.object({
    schema: z.object({
      answer: z.string(),
    }),
  }),
  prompt: 'What is 2 + 2?',
});

console.log(output); // Fully typed and validated!
```

---

**This is the complete guide to Output from 0-100%.** You now understand what it does, why you need it, all the types, when to use each one, and how to implement it with Gemini 3.0 Flash!
