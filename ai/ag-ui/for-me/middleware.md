# AG-UI Middleware: The Complete Guide

## What Is Middleware?

**Middleware is code that sits BETWEEN the agent and your app, intercepting every event.**

```
Your App ← Middleware ← Middleware ← Middleware ← Agent
              ↑            ↑            ↑
           (logs)      (filters)    (adds auth)
```

Every event the agent emits passes through ALL middleware before reaching your app.

---

## Why Use Middleware?

Without modifying the agent, you can:

| Use Case | What It Does |
|----------|--------------|
| **Logging** | Record every event |
| **Filtering** | Block certain events/tools |
| **Auth** | Add API keys to requests |
| **Rate Limiting** | Slow down requests |
| **Transform** | Modify events on the fly |
| **Metrics** | Track performance |

---

## The Mental Model

Think of middleware like airport security checkpoints:

```
Passenger (Event) → Gate 1 → Gate 2 → Gate 3 → Airplane (Your App)
                    (scan)   (check)  (stamp)
```

Each gate can:
- **Inspect** the passenger
- **Modify** their ticket
- **Block** them entirely
- **Let them through** unchanged

---

## Two Ways to Write Middleware

### 1. Function-Based (Simple)

For quick, stateless transformations:

```typescript
const loggingMiddleware: MiddlewareFunction = (input, next) => {
  console.log("Request started")
  
  return next.run(input).pipe(
    tap(event => console.log("Event:", event.type))
  )
}

agent.use(loggingMiddleware)
```

**Pattern:** `(input, next) => Observable<Event>`

- `input` = what's going into the agent
- `next` = the next middleware (or the agent itself)
- Return = the event stream, possibly modified

---

### 2. Class-Based (Complex)

For stateful logic or configuration:

```typescript
class MetricsMiddleware extends Middleware {
  private eventCount = 0  // State!

  constructor(private metricsService: MetricsService) {
    super()
  }

  run(input: RunAgentInput, next: AbstractAgent): Observable<BaseEvent> {
    const startTime = Date.now()

    return next.run(input).pipe(
      tap(event => {
        this.eventCount++
        this.metricsService.recordEvent(event.type)
      }),
      finalize(() => {
        this.metricsService.recordDuration(Date.now() - startTime)
      })
    )
  }
}

agent.use(new MetricsMiddleware(myMetricsService))
```

---

## The Chain: How Events Flow

```typescript
agent.use(middleware1, middleware2, middleware3)
```

**Request flows IN:**
```
Input → middleware1 → middleware2 → middleware3 → Agent
```

**Events flow OUT:**
```
Agent → middleware3 → middleware2 → middleware1 → Your App
```

Each middleware wraps the next:

```
middleware1(
  middleware2(
    middleware3(
      agent.run()
    )
  )
)
```

---

## Built-in Middleware

### FilterToolCallsMiddleware

Block or allow specific tools:

```typescript
// ONLY allow these tools
agent.use(new FilterToolCallsMiddleware({
  allowedToolCalls: ["search", "calculate"]
}))

// Block these tools
agent.use(new FilterToolCallsMiddleware({
  disallowedToolCalls: ["delete", "sendEmail"]
}))
```

Agent tries to call `delete`? Blocked. Never reaches your app.

---

## Common Patterns

### 1. Logging

```typescript
const loggingMiddleware: MiddlewareFunction = (input, next) => {
  console.log("Request:", input.messages)
  
  return next.run(input).pipe(
    tap(event => console.log("Event:", event.type)),
    catchError(error => {
      console.error("Error:", error)
      throw error
    })
  )
}
```

### 2. Authentication

```typescript
class AuthMiddleware extends Middleware {
  constructor(private apiKey: string) {
    super()
  }

  run(input, next) {
    // Inject auth into the request
    const authInput = {
      ...input,
      context: [...input.context, { type: "auth", apiKey: this.apiKey }]
    }
    return next.run(authInput)
  }
}
```

### 3. Rate Limiting

```typescript
class RateLimitMiddleware extends Middleware {
  private lastCall = 0

  constructor(private minInterval: number) {
    super()
  }

  run(input, next) {
    const timeSinceLast = Date.now() - this.lastCall
    
    if (timeSinceLast < this.minInterval) {
      // Wait before calling
      const delay = this.minInterval - timeSinceLast
      return timer(delay).pipe(
        switchMap(() => {
          this.lastCall = Date.now()
          return next.run(input)
        })
      )
    }
    
    this.lastCall = Date.now()
    return next.run(input)
  }
}
```

### 4. Transform Events

```typescript
const prefixMiddleware: MiddlewareFunction = (input, next) => {
  return next.run(input).pipe(
    map(event => {
      if (event.type === EventType.TEXT_MESSAGE_CONTENT) {
        return { ...event, delta: `[AI]: ${event.delta}` }
      }
      return event
    })
  )
}
```

**Before:** `"Hello"`
**After:** `"[AI]: Hello"`

### 5. Conditional Logic

```typescript
const debugMiddleware: MiddlewareFunction = (input, next) => {
  const isDebugMode = input.context.some(c => c.type === "debug")
  
  if (isDebugMode) {
    return next.run(input).pipe(
      tap(event => console.debug("[DEBUG]", event))
    )
  }
  
  return next.run(input)  // No debugging, pass through
}
```

### 6. Throttle (Slow Down Events)

```typescript
const throttleMiddleware: MiddlewareFunction = (input, next) => {
  return next.run(input).pipe(
    throttleTime(50)  // Max 1 event per 50ms
  )
}
```

---

## Combining Multiple Middleware

```typescript
agent.use(
  loggingMiddleware,     // 1st: Log the request
  authMiddleware,        // 2nd: Add authentication
  rateLimitMiddleware,   // 3rd: Enforce rate limits
  filterMiddleware       // 4th: Filter tool calls
)
```

**Execution:**

```
Request  → logging → auth → rateLimit → filter → Agent
                                                    ↓
Your App ← logging ← auth ← rateLimit ← filter ← Events
```

---

## Visual: What Middleware Can Do

```
                    MIDDLEWARE
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │ MODIFY  │    │ FILTER  │    │ OBSERVE │
   │ INPUT   │    │ EVENTS  │    │ (LOG)   │
   └─────────┘    └─────────┘    └─────────┘
        │               │               │
        ▼               ▼               ▼
   Add auth        Block tools      Record
   Transform       Rate limit       metrics
   Inject data     Drop events      Debug
```

---

## Middleware vs No Middleware

**Without middleware:**
```typescript
agent.runAgent().subscribe((event) => {
  // You handle EVERYTHING here
  // Logging? Here.
  // Filtering? Here.
  // Auth? Somewhere else...
  // Gets messy.
})
```

**With middleware:**
```typescript
agent.use(loggingMiddleware, authMiddleware, filterMiddleware)

agent.runAgent().subscribe((event) => {
  // Clean! Only handle what matters.
  // Logging, auth, filtering already done.
})
```

---

## Best Practices

| Rule | Why |
|------|-----|
| One job per middleware | Easier to test, reuse, debug |
| Handle errors | Use `catchError` to prevent crashes |
| Don't block | Use async/RxJS operators |
| Document side effects | If it modifies state, say so |
| Test independently | Unit test each middleware alone |

---

## Quick Reference

```typescript
// Function middleware template
const myMiddleware: MiddlewareFunction = (input, next) => {
  // Before agent runs
  const modifiedInput = { ...input, /* changes */ }
  
  return next.run(modifiedInput).pipe(
    // Process each event
    map(event => event),           // Transform
    filter(event => true),         // Filter
    tap(event => console.log()),   // Side effects
    catchError(err => throwError(err))  // Error handling
  )
}

// Class middleware template
class MyMiddleware extends Middleware {
  private state = {}  // Can have state
  
  constructor(private config: Config) {
    super()
  }
  
  run(input, next) {
    return next.run(input).pipe(/* ... */)
  }
}

// Using middleware
agent.use(middleware1, middleware2, middleware3)
```

---

## One-Sentence Summary

> **Middleware is a chain of interceptors that wrap your agent — each one can inspect, modify, filter, or log events without touching the agent's code.**
