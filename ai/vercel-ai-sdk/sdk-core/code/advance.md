

---

# **COMPLETE GUIDE: `createProviderRegistry` & `customProvider` (0-100%)**

## **What Do These Do? (Plain English)**

Think of AI providers like different restaurants:
- **OpenAI** serves GPT models
- **Anthropic** serves Claude models  
- **Google** serves Gemini models
- etc.

### **`customProvider` = Create Your Own "Restaurant"**
Instead of using one restaurant directly, you create a custom restaurant that:
- Picks specific dishes (models) from different kitchens
- Gives those dishes new names (aliases)
- Pre-configures how each dish is prepared (settings)
- Has a backup kitchen if you order something they don't have (fallback)

### **`createProviderRegistry` = Create a "Food Directory"**
You manage multiple restaurants in one central place:
- Each restaurant has an ID (like `openai`, `anthropic`)
- You find dishes using a simple format: `restaurant:dish` (e.g., `openai:gpt-4`)
- One lookup system for all restaurants
- Customizable separator (can use `:` or `>` or anything else)

---

## **When to Use Each**

| Feature | Use `customProvider` | Use `createProviderRegistry` |
|---------|---------------------|------------------------------|
| Single provider customization | ✅ | ❌ |
| Multiple providers | ⚠️ Possible but manual | ✅ Best choice |
| Alias models | ✅ | ✅ |
| Pre-configure settings | ✅ | ✅ (via customProvider) |
| Centralized model access | ❌ | ✅ |
| Limit available models | ✅ | ✅ |

---

## **PRACTICAL EXAMPLES**

### **1. `customProvider` - Create Aliases**

```typescript
import { customProvider } from 'ai';
import { anthropic } from '@ai-sdk/anthropic';

// Instead of remembering "claude-3-5-sonnet-20240620"
// Just use "sonnet"
export const myAnthropic = customProvider({
  languageModels: {
    opus: anthropic('claude-3-opus-20240229'),
    sonnet: anthropic('claude-3-5-sonnet-20240620'),
    haiku: anthropic('claude-3-haiku-20240307'),
  },
  fallbackProvider: anthropic, // If model not found, try main anthropic
});

// Use it like:
const model = myAnthropic.languageModel('sonnet');
```

**Why this is useful:**
- Update model versions in ONE place
- Shorter, cleaner code
- Team consistency

---

### **2. `customProvider` - Pre-configure Settings**

```typescript
import { 
  customProvider, 
  wrapLanguageModel, 
  defaultSettingsMiddleware 
} from 'ai';
import { openai } from '@ai-sdk/openai';

export const myOpenAI = customProvider({
  languageModels: {
    // GPT-4 with HIGH reasoning effort built-in
    'gpt-4-reasoning': wrapLanguageModel({
      model: openai('gpt-4'),
      middleware: defaultSettingsMiddleware({
        settings: {
          providerOptions: {
            openai: {
              reasoningEffort: 'high', // Pre-configured!
            },
          },
        },
      }),
    }),
    // Different alias with FAST reasoning
    'gpt-4-fast': wrapLanguageModel({
      model: openai('gpt-4'),
      middleware: defaultSettingsMiddleware({
        settings: {
          providerOptions: {
            openai: {
              reasoningEffort: 'low',
            },
          },
        },
      }),
    }),
  },
  fallbackProvider: openai,
});

// Use it - settings already applied:
const model = myOpenAI.languageModel('gpt-4-reasoning');
```

**Why this is useful:**
- Don't repeat settings in every function call
- Ensure consistency across your app
- Easy A/B testing different configurations

---

### **3. `customProvider` - Limit Available Models**

```typescript
import { customProvider } from 'ai';
import { openai } from '@ai-sdk/openai';
import { anthropic } from '@ai-sdk/anthropic';

// Your app only uses these specific models
// Users can't accidentally request other models
export const approvedModels = customProvider({
  languageModels: {
    'fast': openai('gpt-4-mini'),
    'medium': anthropic('claude-3-5-sonnet-20240620'),
    'smart': openai('gpt-4'),
  },
  // NO fallback = only these 3 models work
});

// This works:
approvedModels.languageModel('fast');

// This throws error:
approvedModels.languageModel('gpt-3.5-turbo'); // ❌ Not in the list!
```

**Why this is useful:**
- Security: Only approved models
- Cost control: Prevent expensive model usage
- Simplicity: Users pick from curated options

---

### **4. `createProviderRegistry` - Multiple Providers**

```typescript
import { 
  createProviderRegistry, 
  customProvider 
} from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { openai } from '@ai-sdk/openai';
import { mistral } from '@ai-sdk/mistral';

// Create custom providers for each
const myAnthropic = customProvider({
  languageModels: {
    'opus': anthropic('claude-3-opus-20240229'),
    'sonnet': anthropic('claude-3-5-sonnet-20240620'),
  },
  fallbackProvider: anthropic,
});

const myOpenAI = customProvider({
  languageModels: {
    'gpt-4': openai('gpt-4'),
    'gpt-mini': openai('gpt-4-mini'),
  },
  fallbackProvider: openai,
});

// Create ONE registry with everything
export const registry = createProviderRegistry({
  anthropic: myAnthropic,
  openai: myOpenAI,
  mistral: mistral,
});

// Access via: provider:model
const claudeModel = registry.languageModel('anthropic:sonnet');
const gptModel = registry.languageModel('openai:gpt-4');
const mistralModel = registry.languageModel('mistral:large');
```

**Why this is useful:**
- One place to manage all AI models
- Easy to add/remove providers
- Simple string-based model selection

---

### **5. `createProviderRegistry` - Custom Separator**

```typescript
const registry = createProviderRegistry(
  {
    anthropic: customProvider({ /* ... */ }),
    openai: customProvider({ /* ... */ }),
  },
  { separator: ' > ' } // Use " > " instead of ":"
);

// Now use with custom separator:
const model = registry.languageModel('anthropic > sonnet');
// Instead of: anthropic:sonnet

// Other options:
// separator: '|' => 'anthropic|sonnet'
// separator: '-' => 'anthropic-sonnet'
// separator: '/' => 'anthropic/sonnet'
```

---

### **6. Real-World Complete Setup**

```typescript
import { 
  createProviderRegistry, 
  customProvider, 
  wrapLanguageModel,
  defaultSettingsMiddleware,
  generateText,
} from 'ai';
import { anthropic } from '@ai-sdk/anthropic';
import { openai } from '@ai-sdk/openai';
import { mistral } from '@ai-sdk/mistral';

// === Step 1: Create custom providers ===
const myAnthropic = customProvider({
  languageModels: {
    'cheap': anthropic('claude-3-haiku-20240307'),
    'balanced': anthropic('claude-3-5-sonnet-20240620'),
    'smart': anthropic('claude-3-opus-20240229'),
  },
  fallbackProvider: anthropic,
});

const myOpenAI = customProvider({
  languageModels: {
    'reasoning-on': wrapLanguageModel({
      model: openai('gpt-4'),
      middleware: defaultSettingsMiddleware({
        settings: {
          providerOptions: {
            openai: { reasoningEffort: 'high' },
          },
        },
      }),
    }),
    'reasoning-off': openai('gpt-4-mini'),
  },
  fallbackProvider: openai,
});

// === Step 2: Create registry ===
export const registry = createProviderRegistry({
  anthropic: myAnthropic,
  openai: myOpenAI,
  mistral: mistral,
});

// === Step 3: Use in your code ===
async function generateResponse(modelName: string, prompt: string) {
  const { text } = await generateText({
    model: registry.languageModel(modelName), // e.g., "anthropic:balanced"
    prompt,
  });
  return text;
}

// Usage:
generateResponse('anthropic:cheap', 'Write a haiku');
generateResponse('openai:reasoning-on', 'Solve this complex problem...');
generateResponse('mistral:large', 'Explain quantum computing');
```

---

## **With Vercel v6 & Gemini 3.0 Flash**

Here's how to set up with **Vercel AI SDK v6** + **Google Gemini 3.0 Flash**:

```typescript
// registry.ts
import { 
  createProviderRegistry, 
  customProvider 
} from 'ai';
import { google } from '@ai-sdk/google';

// Custom Google provider with Gemini 3.0 Flash
const myGoogle = customProvider({
  languageModels: {
    'flash': google('gemini-3.0-flash'),
    'pro': google('gemini-3.0-pro'),
    'vision': google('gemini-3.0-flash-vision'),
  },
  fallbackProvider: google,
});

export const registry = createProviderRegistry({
  google: myGoogle,
});
```

```typescript
// api/chat/route.ts (Next.js with Vercel)
import { generateText } from 'ai';
import { registry } from '@/lib/registry';

export async function POST(req: Request) {
  const { prompt } = await req.json();

  const { text } = await generateText({
    model: registry.languageModel('google:flash'),
    prompt,
  });

  return Response.json({ text });
}
```

---

## **Complete Flow: Step-by-Step**

### **Flow 1: Using `customProvider` Alone**
```
1. Create customProvider with models
2. Call customProvider.languageModel('modelId')
3. Get model back
4. Use in generateText, embed, etc.
```

### **Flow 2: Using `createProviderRegistry`**
```
1. Create multiple customProviders (optional)
2. Pass them to createProviderRegistry()
3. Call registry.languageModel('provider:modelId')
4. Registry splits 'provider:modelId' at ':'
5. Finds the provider
6. Calls provider.languageModel('modelId')
7. Returns the actual model
8. Use in generateText, embed, etc.
```

---

## **Key Points to Remember**

### **DO's ✅**
- Use `customProvider` to organize ONE provider's models
- Use `createProviderRegistry` to manage MULTIPLE providers
- Set `fallbackProvider` to catch unknown models gracefully
- Pre-configure settings to avoid repetition
- Create aliases for long model names

### **DON'Ts ❌**
- Don't forget the separator format: `provider:model` (default)
- Don't omit `fallbackProvider` if you want to support models outside your list
- Don't hardcode model IDs everywhere—use registry/custom provider instead
- Don't create nested registries (registry inside registry)

---

## **Error Handling**

```typescript
import { NoSuchModelError } from '@ai-sdk/provider';

try {
  const model = registry.languageModel('unknown:model');
} catch (error) {
  if (error instanceof NoSuchModelError) {
    console.error('Model not found:', error.message);
  }
}
```

---

## **Testing Your Setup**

```typescript
// test.ts
import { registry } from './registry';

// Test all models exist
async function testRegistry() {
  try {
    registry.languageModel('google:flash');
    registry.languageModel('google:pro');
    console.log('✅ All models registered correctly');
  } catch (error) {
    console.error('❌ Registry setup error:', error);
  }
}

testRegistry();
```

---

## **Summary Table**

| Concept | Purpose | When | How |
|---------|---------|------|-----|
| `customProvider` | Organize & configure ONE provider's models | Single provider, need aliases or pre-config | `customProvider({ languageModels: {...}, fallbackProvider })` |
| `createProviderRegistry` | Manage MULTIPLE providers centrally | Multiple providers, need unified access | `createProviderRegistry({ provider1, provider2 })` |
| Separator | Change the format dividing provider & model | Want `anthropic > sonnet` instead of `anthropic:sonnet` | `createProviderRegistry({...}, { separator: ' > ' })` |
| Fallback | Catch unknown models with a backup provider | Allow models outside your defined list | `fallbackProvider: openai` |
| Aliases | Give models shorter/meaningful names | Update versions easily, shorter code | `languageModels: { sonnet: anthropic(...) }` |
| Pre-config | Set default settings on models | Consistency, avoid repetition | `wrapLanguageModel` + `defaultSettingsMiddleware` |

---

**You now know 100% everything about `createProviderRegistry` and `customProvider`!** 🚀
