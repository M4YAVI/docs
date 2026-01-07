
---

# **Complete Guide: `embed`, `embedMany`, `rerank`, `cosineSimilarity` (0-100%)**

## **Table of Contents**
1. What Each Function Does (Plain English)
2. How to Use Each One
3. When to Use Each One
4. Complete Code Examples with Google Gemini 3.0 Flash + Vercel v6
5. Real-world Use Cases
6. Performance Tips

---

## **1. WHAT EACH FUNCTION DOES**

### **`embed()` - Convert Single Text to Vector**
**In Plain English:**
Think of `embed()` as converting a piece of text into a list of numbers (called a "vector" or "embedding"). These numbers represent the meaning of that text in a mathematical space. For example:
- Input: `"sunny day at the beach"`
- Output: `[0.123, -0.456, 0.789, ..., 0.234]` (array of ~1536 numbers for OpenAI's model)

**Why?** These numbers are useful because:
- Similar text gets similar numbers
- You can compare texts by comparing their number arrays
- Machine learning models can work with these numbers

**Real-world analogy:** Like converting a person's face into measurements (height, weight, eye color) so you can compare if two people look alike.

---

### **`embedMany()` - Convert Multiple Texts to Vectors (Batch)**
**In Plain English:**
Same as `embed()`, but you give it 10, 100, or 1000 texts at once instead of just one. It's faster and cheaper than calling `embed()` multiple times.

**Input:**
```typescript
['sunny day at the beach', 'rainy afternoon in the city', 'snowy night in the mountains']
```

**Output:**
```typescript
[
  [0.123, -0.456, 0.789, ...],  // "sunny day..." as numbers
  [0.234, -0.567, 0.890, ...],  // "rainy afternoon..." as numbers  
  [0.345, -0.678, 0.901, ...]   // "snowy night..." as numbers
]
```

**Why batch?** Embedding 1000 items with `embedMany()` costs ~1/10 the price of calling `embed()` 1000 times.

---

### **`cosineSimilarity()` - Compare Two Vectors (0-100% Match)**
**In Plain English:**
Takes two number arrays (embeddings) and tells you how similar they are, on a scale from -1 to 1:
- **1.0** = identical meaning
- **0.5** = moderately similar
- **0.0** = completely different
- **-1.0** = opposite meaning

**Formula (what it does mathematically):**
```
similarity = (a·b) / (||a|| × ||b||)
```
Where `a` and `b` are vectors, `·` is dot product, and `||...||` is magnitude.

**Simple example:**
```typescript
const vec1 = [0.8, 0.2, 0.5];  // embedding for "dog"
const vec2 = [0.7, 0.3, 0.4];  // embedding for "puppy"
cosineSimilarity(vec1, vec2);   // Returns ~0.95 (very similar!)

const vec3 = [0.1, 0.9, -0.5]; // embedding for "car"
cosineSimilarity(vec1, vec3);   // Returns ~0.15 (very different!)
```

**Use it for:**
- Finding similar documents/products
- Searching a database
- Detecting duplicate content
- Recommendation systems

---

### **`rerank()` - Smart Re-Sort Search Results**
**In Plain English:**
Imagine you search for "best pizza in NYC" and get 100 results. `rerank()` takes those 100 results and re-sorts them by **actual relevance** using an AI model trained specifically for relevance ranking.

**Difference from cosineSimilarity:**
- `cosineSimilarity` just does math (fast but basic)
- `rerank()` uses an AI model (slower but much smarter about relevance)

**Example:**
```typescript
const documents = [
  "The weather is sunny today",
  "It is raining heavily outside",
  "Today's weather forecast shows rain"
];

const query = "talk about rain";

// rerank() understands that document[2] and document[1] 
// are more relevant than document[0]
```

**Input:**
- A list of documents (strings or JSON objects)
- A search query
- How many top results you want (`topN`)

**Output:**
- Documents **re-sorted by relevance**
- Each with a relevance score
- Original index (so you know where each came from)

---

## **2. HOW TO USE EACH ONE**

### **Step 1: Install & Setup**

```bash
npm install ai @ai-sdk/google
# or with Vercel's recommended setup
npm install ai
```

### **Step 2: Setup Environment**
```bash
# .env.local
GOOGLE_GENERATIVE_AI_API_KEY=your_key_here
```

---

### **`embed()` - Single Embedding Example**

```typescript
import { embed } from 'ai';
import { google } from '@ai-sdk/google';

async function getEmbedding(text: string) {
  const { embedding, usage } = await embed({
    model: google.embedding('text-embedding-004'), // Gemini 3.0 Flash uses this
    value: text, // Single text input
  });

  console.log('Embedding (first 5 values):', embedding.slice(0, 5));
  console.log('Tokens used:', usage.tokens);
  
  return embedding;
}

// Usage
await getEmbedding('What is artificial intelligence?');
```

---

### **`embedMany()` - Batch Embedding Example**

```typescript
import { embedMany } from 'ai';
import { google } from '@ai-sdk/google';

async function batchEmbed(texts: string[]) {
  const { embeddings, usage } = await embedMany({
    model: google.embedding('text-embedding-004'),
    values: texts, // Array of texts
    maxParallelCalls: 2, // Limit concurrent requests to avoid rate limits
  });

  console.log(`Embedded ${embeddings.length} texts`);
  console.log(`Total tokens used: ${usage.tokens}`);
  
  return embeddings;
}

// Usage
const texts = [
  'Machine learning is powerful',
  'AI can solve many problems',
  'Neural networks are inspired by brains'
];

await batchEmbed(texts);
```

---

### **`cosineSimilarity()` - Compare Embeddings**

```typescript
import { cosineSimilarity, embedMany } from 'ai';
import { google } from '@ai-sdk/google';

async function findSimilarity() {
  // Step 1: Get embeddings for multiple texts
  const { embeddings } = await embedMany({
    model: google.embedding('text-embedding-004'),
    values: [
      'I love eating pizza',
      'I enjoy eating pasta',
      'I hate vegetables',
    ],
  });

  // Step 2: Compare each pair
  console.log('Pizza vs Pasta:', 
    cosineSimilarity(embeddings[0], embeddings[1])); // ~0.92
  
  console.log('Pizza vs Hate Veggies:', 
    cosineSimilarity(embeddings[0], embeddings[2])); // ~0.15
}

await findSimilarity();
```

---

### **`rerank()` - Re-Sort Search Results**

```typescript
import { rerank } from 'ai';
import { cohere } from '@ai-sdk/cohere';

async function searchAndRank(query: string, documents: string[]) {
  const { ranking, rerankedDocuments } = await rerank({
    model: cohere.reranking('rerank-v3.5'), // Cohere for reranking
    documents: documents,
    query: query,
    topN: 3, // Return top 3 most relevant
  });

  // ranking = [
  //   { originalIndex: 1, score: 0.95, document: "..." },
  //   { originalIndex: 2, score: 0.87, document: "..." },
  //   { originalIndex: 0, score: 0.42, document: "..." },
  // ]

  console.log('Top result:', rerankedDocuments[0]);
  console.log('Top 3 scores:', ranking.map(r => r.score));

  return rerankedDocuments;
}

// Usage
const docs = [
  'The weather today is sunny',
  'It is raining outside now',
  'Snow is falling in the mountains'
];

await searchAndRank('tell me about rain', docs);
// Returns: ['It is raining outside now', 'Snow is falling...', '...']
```

---

## **3. WHEN TO USE EACH ONE**

| Function | When to Use | Example Use Case |
|----------|------------|------------------|
| `embed()` | You need embedding for a **single** piece of text | Embedding a user's search query to find related documents |
| `embedMany()` | You need embeddings for **multiple** texts at once | Loading 1000 product descriptions into a vector database |
| `cosineSimilarity()` | Comparing two embeddings for similarity | Finding products similar to what user viewed |
| `rerank()` | You have search results but need better relevance sorting | Improving search accuracy after vector similarity search |

---

## **4. COMPLETE WORKING EXAMPLES (Gemini 3.0 Flash + Vercel v6)**

### **Example 1: RAG (Retrieval-Augmented Generation) Chat Bot**

```typescript
// lib/rag-chatbot.ts
import { embed, embedMany, cosineSimilarity } from 'ai';
import { generateText } from 'ai';
import { google } from '@ai-sdk/google';

// Fake "knowledge base" (in real app, this is a database)
const knowledgeBase = [
  { id: 1, text: 'Cats are domestic animals that are often kept as pets' },
  { id: 2, text: 'Dogs are loyal companions known for their friendliness' },
  { id: 3, text: 'Birds can fly and have feathers' },
  { id: 4, text: 'Fish live in water and have scales' },
];

export async function ragChatbot(userQuestion: string) {
  // Step 1: Embed the user's question
  const { embedding: questionEmbedding } = await embed({
    model: google.embedding('text-embedding-004'),
    value: userQuestion,
  });

  // Step 2: Embed all knowledge base documents
  const { embeddings: docEmbeddings } = await embedMany({
    model: google.embedding('text-embedding-004'),
    values: knowledgeBase.map(doc => doc.text),
  });

  // Step 3: Find most similar documents using cosineSimilarity
  const similarities = docEmbeddings.map((docEmbed, idx) => ({
    id: knowledgeBase[idx].id,
    text: knowledgeBase[idx].text,
    similarity: cosineSimilarity(questionEmbedding, docEmbed),
  }));

  // Sort by similarity and get top 2
  const topMatches = similarities
    .sort((a, b) => b.similarity - a.similarity)
    .slice(0, 2);

  const context = topMatches.map(m => m.text).join('\n');

  // Step 4: Use Gemini to generate answer based on context
  const { text } = await generateText({
    model: google('gemini-2.0-flash'), // Gemini 2.0 Flash
    system: `You are a helpful assistant. Use the following context to answer the user's question:\n${context}`,
    prompt: userQuestion,
  });

  return {
    answer: text,
    sources: topMatches.map(m => ({ id: m.id, similarity: m.similarity })),
  };
}

// Usage in API route
export async function POST(req: Request) {
  const { question } = await req.json();
  const result = await ragChatbot(question);
  return Response.json(result);
}
```

**Call it from client:**
```typescript
const response = await fetch('/api/rag', {
  method: 'POST',
  body: JSON.stringify({ question: 'Tell me about cats' })
});
const { answer, sources } = await response.json();
console.log(answer);
```

---

### **Example 2: Semantic Search Engine**

```typescript
// lib/semantic-search.ts
import { embedMany, cosineSimilarity } from 'ai';
import { rerank } from 'ai';
import { google } from '@ai-sdk/google';
import { cohere } from '@ai-sdk/cohere';

const DATABASE = [
  'Apple is a fruit',
  'Apple Inc. is a technology company',
  'Orange is a citrus fruit',
  'Microsoft is a software company',
  'Banana is yellow and sweet',
];

export async function semanticSearch(query: string, useReranking = true) {
  // Step 1: Vector similarity search (fast, basic)
  const { embedding: queryEmbedding } = await embed({
    model: google.embedding('text-embedding-004'),
    value: query,
  });

  const { embeddings } = await embedMany({
    model: google.embedding('text-embedding-004'),
    values: DATABASE,
  });

  const vectorResults = DATABASE.map((doc, idx) => ({
    text: doc,
    score: cosineSimilarity(queryEmbedding, embeddings[idx]),
  }))
    .sort((a, b) => b.score - a.score)
    .slice(0, 5); // Top 5

  if (!useReranking) return vectorResults;

  // Step 2: Re-rank results (smarter, slower)
  const { ranking } = await rerank({
    model: cohere.reranking('rerank-v3.5'),
    documents: vectorResults.map(r => r.text),
    query: query,
    topN: 3,
  });

  return ranking.map(r => ({
    text: r.document,
    score: r.score,
  }));
}

// Usage
const results = await semanticSearch('apple fruit', true);
// With reranking: Returns more relevant results for "fruit apple"
```

---

### **Example 3: Duplicate Content Detection**

```typescript
// lib/duplicate-detector.ts
import { embedMany, cosineSimilarity } from 'ai';
import { google } from '@ai-sdk/google';

export async function findDuplicates(
  contents: string[],
  threshold = 0.95
) {
  // Embed all contents
  const { embeddings } = await embedMany({
    model: google.embedding('text-embedding-004'),
    values: contents,
  });

  // Find pairs with high similarity
  const duplicates: Array<{
    index1: number;
    index2: number;
    similarity: number;
  }> = [];

  for (let i = 0; i < embeddings.length; i++) {
    for (let j = i + 1; j < embeddings.length; j++) {
      const similarity = cosineSimilarity(embeddings[i], embeddings[j]);
      if (similarity > threshold) {
        duplicates.push({
          index1: i,
          index2: j,
          similarity,
        });
      }
    }
  }

  return duplicates;
}

// Usage
const dupes = await findDuplicates([
  'Hello world',
  'Hello world', // Exact duplicate
  'Hi there'     // Not a duplicate
]);
// Returns: [{ index1: 0, index2: 1, similarity: 0.99 }]
```

---

### **Example 4: Product Recommendation System**

```typescript
// lib/recommendations.ts
import { embed, embedMany, cosineSimilarity } from 'ai';
import { google } from '@ai-sdk/google';

const PRODUCTS = [
  { id: 1, name: 'iPhone 15', desc: 'Smartphone with great camera' },
  { id: 2, name: 'Samsung Galaxy', desc: 'Android phone with large screen' },
  { id: 3, name: 'Laptop Dell', desc: 'Powerful computer for work' },
  { id: 4, name: 'iPad Pro', desc: 'Tablet for creative professionals' },
  { id: 5, name: 'Sony Camera', desc: 'Professional mirrorless camera' },
];

export async function recommendProducts(
  userViewedProduct: string,
  count = 3
) {
  // Step 1: Embed the viewed product
  const { embedding: viewedEmbed } = await embed({
    model: google.embedding('text-embedding-004'),
    value: userViewedProduct,
  });

  // Step 2: Embed all products
  const { embeddings } = await embedMany({
    model: google.embedding('text-embedding-004'),
    values: PRODUCTS.map(p => p.desc),
  });

  // Step 3: Find most similar
  const recommendations = PRODUCTS.map((prod, idx) => ({
    ...prod,
    similarity: cosineSimilarity(viewedEmbed, embeddings[idx]),
  }))
    .sort((a, b) => b.similarity - a.similarity)
    .slice(0, count);

  return recommendations;
}

// Usage
const recs = await recommendProducts(
  'I like smartphones with excellent cameras'
);
// Returns: iPhone 15, Samsung Galaxy, iPad Pro (sorted by relevance)
```

---

## **5. REAL-WORLD WORKFLOW (Complete App)**

### **Setup (one-time):**

```bash
npm install ai @ai-sdk/google @ai-sdk/cohere dotenv
```

```env
GOOGLE_GENERATIVE_AI_API_KEY=your_key
COHERE_API_KEY=your_key
```

### **API Route (pages/api/search.ts):**

```typescript
import { embed, embedMany, cosineSimilarity } from 'ai';
import { rerank } from 'ai';
import { google } from '@ai-sdk/google';
import { cohere } from '@ai-sdk/cohere';

export default async function handler(req, res) {
  if (req.method !== 'POST') return res.status(405).end();

  const { query, documents } = req.body;

  try {
    // 1. Vector search
    const { embedding: q } = await embed({
      model: google.embedding('text-embedding-004'),
      value: query,
    });

    const { embeddings } = await embedMany({
      model: google.embedding('text-embedding-004'),
      values: documents,
    });

    const vectorScores = documents.map((doc, i) => ({
      doc,
      score: cosineSimilarity(q, embeddings[i]),
    }))
      .sort((a, b) => b.score - a.score)
      .slice(0, 5);

    // 2. Rerank for better results
    const { ranking } = await rerank({
      model: cohere.reranking('rerank-v3.5'),
      documents: vectorScores.map(v => v.doc),
      query,
      topN: 3,
    });

    res.json({ results: ranking });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
}
```

### **Frontend:**

```tsx
'use client';
import { useState } from 'react';

export default function SearchPage() {
  const [results, setResults] = useState([]);
  const [query, setQuery] = useState('');

  const handleSearch = async () => {
    const response = await fetch('/api/search', {
      method: 'POST',
      body: JSON.stringify({
        query,
        documents: [
          'The sky is blue',
          'Cats are animals',
          'Dogs are loyal',
          'Water is wet'
        ]
      })
    });

    const data = await response.json();
    setResults(data.results);
  };

  return (
    <div>
      <input 
        value={query} 
        onChange={(e) => setQuery(e.target.value)} 
        placeholder="Search..."
      />
      <button onClick={handleSearch}>Search</button>

      {results.map((r, i) => (
        <div key={i}>
          <p>{r.document}</p>
          <small>Relevance: {(r.score * 100).toFixed(1)}%</small>
        </div>
      ))}
    </div>
  );
}
```

---

## **6. PERFORMANCE TIPS**

| Tip | Why | Example |
|-----|-----|---------|
| Use `embedMany()` not `embed()` in loops | 100x cheaper | ✅ `embedMany([...1000 items])` ❌ `for(item) embed(item)` |
| Cache embeddings | Don't re-compute | Store in database, reuse |
| Use `maxParallelCalls` to limit load | Avoid rate limits | `maxParallelCalls: 2` |
| Vector search then rerank | Fast + accurate | First 5 with cosine, then top 3 with rerank |
| Set `maxRetries: 0` for low-stakes | Faster response | `maxRetries: 0` for UI, `2` for critical |

---

## **7. TROUBLESHOOTING**

| Error | Solution |
|-------|----------|
| `API rate limit exceeded` | Use `maxParallelCalls: 1`, add delays |
| `Vectors have different lengths` | Make sure both embeddings from same model |
| `Poor search results` | Use `rerank()` for better relevance |
| `Embedding too expensive` | Use smaller model or batch with `embedMany()` |
| `No results` | Lower similarity threshold from 0.8 to 0.5 |

---

## **8. MODEL RECOMMENDATIONS**

### **For Embeddings (with Gemini):**
```typescript
google.embedding('text-embedding-004')  // Best quality
```

### **For Text Generation (with Gemini):**
```typescript
google('gemini-2.0-flash')  // Fast + cheap
google('gemini-1.5-pro')    // More powerful
```

### **For Reranking (with Cohere):**
```typescript
cohere.reranking('rerank-v3.5')  // Best overall
cohere.reranking('rerank-v2')    // Cheaper
```

---

**Now you have a complete 0-100% understanding of all 4 functions!** 🚀
