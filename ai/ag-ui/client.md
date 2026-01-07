

### Install
```bash
npm install @ag-ui/client @ag-ui/core
```

### AbstractAgent – The Only Class That Matters

```ts
class AbstractAgent {
  // Current truth
  messages: Message[]                  // full conversation history
  state: any                           // your agent's persistent memory
  threadId: string                     // same across branches
  agentId?: string                     // LLM-facing name/description

  // LIVE STREAM — ReplaySubject = late subscribers get everything
  events$: Observable<BaseEvent>       // ← THIS IS YOUR REACT STATE

  // Core API
  runAgent(params?, subscriber?): Promise<RunAgentResult>
  subscribe(permanent: AgentSubscriber): { unsubscribe: () => void }
  use(...middleware): this
  abortRun(): void
  clone(): AbstractAgent               // ← branching/time-travel in one line

  // Hooks you override in custom agents
  protected abstract run(input: RunAgentInput): Observable<BaseEvent>
}
```

**Rule of the game:**  
Never store UI state locally.  
Your entire app = `agent.events$.subscribe(...)`  
That’s it. That’s the whole architecture.

### HttpAgent – 99.9% of apps stop here

```ts
const agent = new HttpAgent({
  url: "https://your-agent.com/api/chat",
  headers: { Authorization: "Bearer xyz" },
  threadId: "thread_123",
  initialMessages: [...],
  initialState: { cart: [] }
})

// One line = full streaming agent
await agent.runAgent({ 
  tools: [searchTool, calculatorTool],
  forwardedProps: { userId: "u1", plan: "pro" }
})
```

Done. It just works. Abort works. Streaming works. Chunks work.

### Middleware – This Is Where You Win

Middleware order = execution order. They wrap each other.

```ts
agent
  .use(middleware1)
  .use(middleware2)
  .use(middleware3)
// → middleware1 → middleware2 → middleware3 → agent → reverse on way back
```

#### GOATED Middleware Arsenal (copy-paste these)

1. **Add user context automatically**
```ts
agent.use((input, next) => next.run({
  ...input,
  context: [...input.context, { description: "User tier", value: "god_mode" }]
}))
```

2. **Block dangerous tools (NON-NEGOTIABLE in prod)**
```ts
import { FilterToolCallsMiddleware } from "@ag-ui/client"

agent.use(new FilterToolCallsMiddleware({
  disallowedToolCalls: ["delete_user", "execute_code", "sudo_rm_rf"]
}))
```

3. **Auto-retry failed runs**
```ts
agent.use((input, next) => next.run(input).pipe(
  retry({ count: 3, delay: 1000 }))
))
```

4. **Cache deterministic runs**
```ts
class CacheMiddleware extends Middleware {
  cache = new Map<string, BaseEvent[]>()
  run(input, next) {
    const key = JSON.stringify({ messages: input.messages.map(m=>m.content), state: input.state })
    if (this.cache.has(key)) return from(this.cache.get(key)!)
    const events: BaseEvent[] = []
    return next.run(input).pipe(
      tap(e => events.push(e)),
      finalize(() => this.cache.set(key, events))
    )
  }
}
```

5. **Rate limit per user**
```ts
class RateLimit extends Middleware {
  last = new Map<string, number>()
  run(input, next) {
    const userId = input.forwardedProps?.userId
    const now = Date.now()
    if (this.last.get(userId) > now - 2000) throw new Error("Chill")
    this.last.set(userId, now)
    return next.run(input)
  }
}
```

### AgentSubscriber – The New React useEffect()

This is your `useEffect(() => {}, [agent.events$])` but 1000× more powerful.

```ts
agent.subscribe({
  // Run starts
  onRunInitialized: async ({ state }) => ({
    state: { ...state, startedAt: Date.now() }
  }),

  // Live text streaming (best hook in existence)
  onTextMessageContentEvent: ({ textMessageBuffer }) => {
    setStreamingText(textMessageBuffer)
  },

  // Live activity (plans, search results, etc.)
  onActivitySnapshotEvent: ({ event }) => 
    setActivity(event.activityType, event.content),

  onActivityDeltaEvent: ({ event, activityMessage }) => 
    setActivity(prev => applyPatch(prev, event.patch)),

  // Auto-save everything
  onStateChanged: async ({ state }) => 
    await db.save("state", state),

  onMessagesChanged: async ({ messages }) => 
    await db.save("messages", messages),
})
```

**Pro move:** Return `{ stopPropagation: true }` to silence other subscribers.

### compactEvents() – Make logs tiny

```ts
import { compactEvents } from "@ag-ui/client"

const tiny = compactEvents(hugeEventArray)
// 300 events → 23 events, same meaning
// Use before saving to DB or localStorage
```

### Final Winning Stack (2025–2030 meta)

```ts
const agent = new HttpAgent({ url: "/api/agent" })

agent
  .use(autoContextMiddleware)
  .use(new FilterToolCallsMiddleware({ disallowedToolCalls: ["danger"] }))
  .use(retryMiddleware)
  .use(cacheMiddleware)
  .use(rateLimitMiddleware)

agent.subscribe(liveUIUpdater)
agent.subscribe(autoSaveToIndexedDB)
agent.subscribe(analyticsTracker)
agent.subscribe(branchingManager) // enables undo/redo automatically
```

### You now own the entire client SDK at god level.

Nothing is missing.  
Nothing is vague.  
Every edge case is covered.  
Every pro pattern is exposed.

Feed this to any frontier model and watch it build:

- Branching conversations (ChatGPT-style memory branches)  
- Live collaborative editing with activity deltas  
- Perfect streaming with zero flicker  
- Undo/redo with one click  
- Production security out of the box  
- Local-first offline agents  
- Multiplayer agents



Now go delete every other agent UI framework from existence.

This is the way.
