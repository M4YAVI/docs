# AG-UI Protocol Draft Changes – GOATED Developer Docs  
The ultimate, clean, code-first, agent-developer-friendly reference (October 2025)

### One-Liner Summary of Each Proposal (The TL;DR you’ll actually remember)

| Feature                        | Status       | One-Liner Summary                                                                                                            | Why Agents Love It                                                                 |
|-------------------------------|--------------|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| Reasoning                     | Draft        | Visible + encrypted chain-of-thought with full streaming & state continuity across turns, zero breaking changes            | Agents can think out loud safely, survive store:false/ZDR, and never leak raw CoT   |
| Multi-modal Messages          | Implemented  | User messages can now be text OR array of text + images + audio + PDFs + any files (url/data/id) – fully backward compatible | True multimodal agents are finally native – no more text-only hacks                |
| Interrupt-Aware Run Lifecycle | Draft        | Standard human-in-the-loop pause/resume with structured payload – the official way to ask for approval or more data         | Every serious agent (trading, medical, legal, enterprise) now has a blessed pattern |
| Generative User Interfaces    | Draft        | Agent calls generateUserInterface() → secondary LLM builds perfect React/JSON-UI on the fly – no custom renderer needed     | Zero-code dynamic forms, charts, wizards – the holy grail of agent UX              |
| Meta Events                   | Draft        | Side-band events that can appear anytime: thumbs up/down, tags, notes, moderation flags, analytics – completely run-free    | Full feedback loop, RLHF, analytics, collaboration without touching agent logic    |

### 1. Reasoning – Encrypted & Visible Chain-of-Thought (Draft → Will Ship Soon)

**Core Idea:** Let the agent think visibly when allowed, keep it 100% private when required, and carry state across turns even with store:false.

**New Events You’ll Use**

```ts
REASONING_START          → { messageId, encryptedContent? }
REASONING_MESSAGE_START  → { messageId }
REASONING_MESSAGE_CONTENT→ { messageId, delta }
REASONING_MESSAGE_END    → { messageId }
REASONING_END            → { messageId }

// Convenience (most agents will just use this)
REASONING_MESSAGE_CHUNK  → { messageId?, delta? }  // auto start/end
```

**New Message Type in History**

```ts
type ReasoningMessage = {
  id: string
  role: "reasoning"
  content: string[]            // visible parts (can be empty)
  encryptedContent?: string    // survives ZDR, carried over turns
}
```

**GOATED Pattern (use this)**

```ts
// Start hidden reasoning (e.g. private CoT)
REASONING_START { messageId: "r1", encryptedContent: "enc-abc..." }

// Show user you're thinking (optional)
REASONING_MESSAGE_CHUNK { delta: "Analyzing requirements..." }
REASONING_MESSAGE_CHUNK { delta: "Considering edge cases..." }
REASONING_MESSAGE_CHUNK { delta: "" }  // auto-closes

// End
REASONING_END { messageId: "r1" }
```

Old THINKING_TEXT_* events are dead – delete them from your code.

### 2. Multi-modal Messages – NOW IMPLEMENTED (Oct 16, 2025)

It’s live. Start using it today.

```ts
type UserMessage.content = string | InputContent[]

type InputContent =
  | { type: "text";  text: string }
  | { type: "binary"; mimeType: string; url?: string; data?: string; id?: string; filename?: string }
```

**Real Examples That Work Today**

```json
// Classic text (still works)
{ "content": "What's in this image?" }

// One image + text
{ "content": [
  { "type": "text", "text": "Explain this diagram" },
  { "type": "binary", "mimeType": "image/png", "data": "iVBORw0KGgo..." }
]}

// Multiple files
{ "content": [
  { "type": "text", "text": "Compare these two PDFs" },
  { "type": "binary", "mimeType": "application/pdf", "url": "https://a.com/report1.pdf" },
  { "type": "binary", "mimeType": "application/pdf", "url": "https://a.com/report2.pdf" }
]}
```

At least one of url, data, or id is required for binary.

### 3. Interrupt-Aware Run Lifecycle – The Official Human-in-the-Loop Standard (Draft)

This is how every serious agent will ask for permission from now on.

**RUN_FINISHED now supports outcome: "interrupt"**

```ts
{
  "type": "RUN_FINISHED",
  "outcome": "interrupt",
  "interrupt": {
    "id": "int-123",                     // optional but recommended
    "reason": "human_approval" | "upload_required" | "policy_hold",
    "payload": { /* anything you want to show in UI */ }
  }
}
```

**Next RunAgentInput resumes it**

```ts
{
  "threadId": "same-as-before",
  "resume": {
    "interruptId": "int-123",   // echo back if provided
    "payload": { "approved": true, "notes": "go ahead but cap spend at $500" }
  }
}
```

**Golden Use Cases**
- Send email → show preview → wait for approval
- Delete data → show exact SQL + rollback plan → wait
- Spend money → require 2FA + comment
- Need another file → pause and ask

### 4. Generative User Interfaces – Zero-Code Dynamic UI (Draft → Mind-Blowing)

**The single most powerful feature coming to agents.**

Agent just calls one tool:

```ts
generateUserInterface({
  description: "A beautiful form for collecting shipping address with validation",
  data: { firstName: "Ada", city: "London" },
  output: { /* JSON Schema of what you expect back */ }
})
```

Then CopilotKit (or your own generator) instantly returns perfect React + Zod + Tailwind form, or JSON-UI schema, or whatever you want.

No more writing custom tool renderers. Ever.

**You stay 100% in control** – swap the generator for your design system, add animations, force dark mode, output ShadCN, etc.

This is how agents will ship pixel-perfect UIs in 2026.

### 5. Meta Events – Side-band Superpower (Draft)

Events that can appear anytime, from anyone.

```ts
{
  "type": "META",
  "metaType": "thumbs_down",
  "payload": {
    "messageId": "msg-456",
    "reason": "hallucinated_source",
    "comment": "You made up the 2024 paper"
  }
}
```

Common metaTypes (you can invent your own):
- thumbs_up / thumbs_down
- tag
- note
- bookmark
- rating
- moderation
- analytics

Perfect for RLHF, analytics, collaboration, audit trails.

### Final Verdict – What You Should Do Right Now

| Priority | Action                                                                 |
|----------|------------------------------------------------------------------------|
| ★★★★★    | Upgrade to multi-modal messages TODAY (already implemented)           |
| ★★★★★    | Start designing your interrupt/approval flows – this pattern is final |
| ★★★★     | Prepare SDK for Reasoning events (coming next)                        |
| ★★★★     | Get ready for Generative UI – start thinking about your UI generator  |
| ★★★      | Add MetaEvent support for feedback buttons                             |

These five drafts together make AG-UI the most powerful, future-proof, production-ready agent protocol on earth.

You’re not just building agents anymore.  
You’re building the next generation of AI applications.

Welcome to the golden age.
