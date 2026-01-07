

##The official standard for how AI agents talk to UIs in 2025+

Just like HTTP is the standard for web pages,
AG-UI is now the standard for AI chat interfaces.

# The two packages = the full stack

@ag-ui/core → the language (events, types, rules)
@ag-ui/client → the universal chat UI brain (works with any backend)


| Feature you love in ChatGPT            | AG-UI name                              | One-liner                                  |
|--------------------------------------|------------------------------------------|---------------------------------------------|
| Streaming text                        | TEXT_MESSAGE_CHUNK                       | Just works perfectly                        |
| Live tool use                         | TOOL_CALL_CHUNK                          | Parallel tools = built-in                   |
| “Assistant is thinking…”              | ActivityMessage + ACTIVITY_DELTA         | The real magic                              |
| Upload images/files                   | BinaryInputContent                       | Native multimodal                           |
| Branching / memory versions           | parentRunId + clone()                    | ChatGPT’s new feature, but free             |
| Undo / redo                           | Just replay old run events               | Free                                        |
| Multiplayer / live sync               | State delta + activity delta             | Free                                        |
| Perfect loading states                | StepStarted / StepFinished               | Built-in                                    |


AG-UI is the HTTP of AI agents.

Once you use it, you can never go back to the old way.

You are not early.
You are right on time.

And now you get it.

Welcome to the future.
It’s already here.
