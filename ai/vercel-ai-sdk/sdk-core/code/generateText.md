
```mdx
🎯 streamText() - Complete 0-100% Guide (Plain English)

🤔 What is it? (Super Simple)

A server-side function that generates AI text and streams it back chunk-by-chunk in real-time.
plaintext

generateText():
Backend → AI → Wait for complete response → Return all at once
streamText():
Backend → AI → Stream chunks → Frontend sees live typing ✨
         chunk 1 ("Hello")
         chunk 2 (" there")
         chunk 3 (" friend")
         (done!)

In one sentence: It's a function that generates AI text and sends it to the frontend one piece at a time so users see typing in real-time.

📊 streamText() vs generateText() vs useChat()
Feature	generateText()	streamText()	useChat()
Where	Backend only	Backend only	Frontend hook
Streaming	❌ No	✅ Yes	✅ Yes (auto)
Wait for complete	✅ Yes	❌ No	❌ No
Use Case	One-shot	Chat/API	Chat UI
Returns	Result object	Stream object	Hook state
Speed	Slower (waits)	⚡ Fastest	⚡ Fastest
Real-time	❌ No	✅ Yes	✅ Yes
Complexity	Simple	Medium	Simple
Messages	Prompt only	Support messages	Auto managed

🎯 When Do You Need It?

✅ USE streamText() FOR:


    💬 Chat applications (real-time typing)

    🤖 AI assistants (live response display)

    📝 Text generation with streaming (articles, emails)

    🎯 API endpoints that need streaming

    ⚡ Better UX (users see progress)

    🔄 Multi-turn conversations (with messages)

    🛠️ Tool calling (with streaming)

    📱 Real-time updates needed

❌ DON'T USE if:


    🔧 One-shot generation (use generateText())

    💾 Don't need streaming (use generateText())

    🎨 Frontend UI only (use useChat() with /api/chat)

📍 Installation
bash

npm install ai @ai-sdk/google

Set up environment:
bash

export GOOGLE_GENERATIVE_AI_API_KEY="your-key-here"

🚀 Complete Working Example (Gemini 2.0
```
