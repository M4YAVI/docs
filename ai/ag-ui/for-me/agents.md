AG-UI Agents: The Simple Truth
What Problem Does This Solve?
Imagine you have 10 different AI services (Claude, GPT-4, custom systems). Each one speaks a different "language" for communication. Your app would need 10 different ways to talk to them.
AG-UI says: "Everyone speaks ONE language." Now your app learns one way to communicate, and it works with everything.
What Is an Agent?
An agent is a middleman between your app and an AI service.
Your App  →  Agent  →  AI Service (Claude, GPT-4, etc.)

The agent does four jobs:

1. Remembers the conversation (message history)
2. Keeps track of data (state)
3. Sends responses piece-by-piece (streaming)
4. Can use tools (like calling functions)

How Communication Works: Events
Instead of waiting for one big response, the agent sends small messages (events) as things happen:
RUN_STARTED         → "I'm starting"
TEXT_MESSAGE_START  → "Here comes a message"
TEXT_MESSAGE_CONTENT → "Hel"
TEXT_MESSAGE_CONTENT → "lo "
TEXT_MESSAGE_CONTENT → "world"
TEXT_MESSAGE_END    → "Message done"
RUN_FINISHED        → "All done"

This is why you see AI typing character-by-character. Each chunk is an event.
The Two Agent Types You'll Use
1. HttpAgent — Connects to a remote AI service over the internet:
javascriptDownloadCopy codeconst agent = new HttpAgent({
  url: "https://ai-service.com/agent"
})
2. Custom Agent — You build it yourself by extending the base class:
javascriptDownloadCopy codeclass MyAgent extends AbstractAgent {
  run(input) {
    // Your logic here
  }
}
Tools: How Agents Do Things
Tools are functions the agent can call. The important part: your app defines the tools, not the agent.
Example flow:

1. Your app tells the agent: "You can use a tool called confirmAction"
2. Agent decides it needs confirmation from the user
3. Agent sends events: TOOL_CALL_START → TOOL_CALL_ARGS → TOOL_CALL_END
4. Your app runs the tool, gets the result
5. Your app sends the result back to the agent
6. Agent continues with that information

This creates human-in-the-loop workflows. The AI proposes, the human approves, the AI continues.
State: Memory That Persists
javascriptDownloadCopy codeagent.state = {
  userName: "John",
  preferences: { theme: "dark" }
}
State survives across messages. Updated two ways:

* STATE_DELTA — "Change this one thing"
* STATE_SNAPSHOT — "Here's the complete new state"

Messages: The Conversation History
javascriptDownloadCopy codeagent.messages = [
  { id: "1", role: "user", content: "Hello" },
  { id: "2", role: "assistant", content: "Hi there" }
]
The agent always knows what was said before.
Running an Agent (Putting It Together)
javascriptDownloadCopy codeagent.runAgent({
  runId: "run_123",
  tools: [myTool],      // What the agent can do
  context: []           // Extra info
}).subscribe({
  next: (event) => {
    // Handle each event as it arrives
  },
  complete: () => {
    // Done
  }
})
The .subscribe() pattern means you're listening to a stream of events, not waiting for one response.
The Core Insight
AG-UI is a contract. It says:

"All agents will send these specific event types, manage state this way, handle tools this way, and track messages this way."

Because everyone follows the same contract, you can:

* Swap AI providers without rewriting your app
* Build UIs that work with any agent
* Have multiple agents hand off work to each other
* Mix human decisions into AI workflows seamlessly


One sentence: AG-UI is a universal translator between your app and any AI, using a stream of small standardized messages instead of one big response.
