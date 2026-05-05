# Layer 3 — End-to-end via the assistant surface

Tests the full chain from the user-facing channel to the RAG workspace and back. In the originating stack this is Telegram → bot → orchestrator → inference model → MCP tool call → AnythingLLM → answer back through every hop. Layer 3 is the only layer that confirms the workspace is reachable as users will reach it.

## What this layer proves

- The user-facing channel (Telegram, web chat, voice, whatever it is in your stack) can route a message to the orchestrator.
- The orchestrator routes the message to the chat model.
- The chat model decides to invoke the RAG tool when appropriate, with the correct arguments and the correct workspace slug.
- The MCP server is reachable from the orchestrator's spawn path.
- The tool result returns through the orchestrator back into the chat model's context.
- The chat model produces a final answer that is grounded in the tool result.
- The final answer flows back through the channel to the user.

That is a lot of links, and any one of them can fail in ways the lower layers do not catch. Layer 3 is also the slowest and most coupled test — when it fails, Layers 1 and 2 are the fastest way to localise which link broke.

## What this layer does not prove

Counterintuitively: it does not prove RAG is working *correctly* end-to-end, only that the chain is *reachable*. A model that ignores the tool result and answers from training priors still produces a flowing answer through the chain — but the answer is not RAG-grounded. Read the answer carefully against the source document; do not assume a quick reply means RAG worked.

## The probe

Send a message that explicitly names the workspace, framed in a way the model is likely to honour. The originating run used:

> Use the `<workspace-slug>` workspace and answer: when am I allowed to use scp to deploy to GEEKOM?

Two design choices in this prompt matter:

- **Naming the workspace explicitly.** Some chat models will choose a default workspace if not told which one to use, and that default may not contain the document under test. Naming the slug avoids ambiguity.
- **Asking a question whose answer is verifiable against the source document.** Use one of the queries from `docs/03-validation-queries.md` that you have already run at Layer 1; the expected answer is then known and gradable.

Send the message through the user-facing channel. Capture the bot's reply.

## What to read in the response

Three things to grade against the source document:

- **Content.** Does the answer match the same passage you graded at Layer 1? If yes, the chain is operational. If the answer differs from Layer 1 — especially if it is more general or hedged — the model is not honouring the tool result.
- **Source citation, if any.** Does the response cite the source document or paraphrase its specifics? Some chat models include a citation in the user-facing reply; many do not, even when the tool result included one. Absence of citation in the user-facing reply is normal.
- **Latency.** The wall-clock time from message sent to reply received. Compare against Layer 2 latency to isolate the model's contribution. Long Layer 3 latency with healthy Layer 2 latency points to the model being slow, not the RAG plumbing.

## Common observations

### Workspace context bleed across turns

Once the model has called `rag_query` on a workspace in one turn, content from that workspace often surfaces in the *next* turn even when the user did not ask. The next turn's question may receive an answer that blends training-data knowledge with content the model retrieved earlier. This is conversational-context carryover, not a bug; the model is using its full context window.

The implication is operational, not a failure: when validating, ask each test question in a fresh chat thread to avoid the previous turn's tool result leaking into the current answer. When operating, this carryover is sometimes useful and sometimes a privacy concern, depending on workspace boundaries.

### Long round-trip latency

A user-facing reply that takes minutes is almost always the model's fault, not the RAG plumbing. Confirm by running Layer 2 standalone with the same question and comparing latencies. If Layer 2 returns in seconds and Layer 3 takes minutes, the model is the bottleneck — likely a combination of tool-call decision time, in-context-tool-result processing time, and final-answer generation time. Common remedies: smaller chat model for the orchestrator role, reduced context length, or a quantisation step.

### The model invents an answer instead of calling the tool

If the user-facing reply is a confident answer that does not match the source document, the chat model may have skipped the tool call entirely and answered from priors. Inspect the orchestrator's tool-call log if available. If the tool was never invoked, the model's system prompt may be too permissive (does not instruct the model to use RAG tools when the user references "the workspace" or "the rulebook"). Tightening the system prompt to mandate tool use when the user names a workspace is the usual fix.

### Bot is silent

A long silence from the bot can mean the channel is wedged, the orchestrator is queueing, the model is thinking, or the inference backend is hung. Run pre-flight checks (`docs/06-pre-flight-checks.md`) on the inference backend specifically; a wedged Ollama runner produces this exact symptom.

## Recording the result

Capture a screenshot or text export of the chat, with timestamps. Compare against Layer 1 result for the same question. Note the observed wall-clock latency and any retries or repeats. If the answer differs from Layer 1, note the divergence — that is your starting point for diagnosis.
