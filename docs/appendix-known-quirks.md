# Appendix — Known quirks

Behaviour observed in the originating validation run that is not obvious from the documentation and is easy to misdiagnose. Keep this page open while debugging.

## `queryRefusalResponse` is advisory, not enforced

The workspace's `queryRefusalResponse` field is named in a way that suggests it is the literal string AnythingLLM returns when retrieval is empty. It is not. The field is passed into the model's system prompt as guidance; the model decides whether to use it.

Observed behaviour: a query whose retrieval returned `sources: []` produced a confident fabricated answer rather than the configured refusal string. A second query with a similar shape produced a natural-language refusal in the model's own words. Both behaviours are normal model behaviour — neither is a bug in AnythingLLM.

Implication: you cannot rely on `queryRefusalResponse` as a hard guarantee. To enforce strict grounding, use `chatMode: query` (stricter system prompt), use a stronger chat model, or layer your own refusal-detection at the calling layer.

## Source citations are filename-only at the MCP layer

The MCP `rag_query` tool's response formats sources as filenames only:

```
Sources:
- development-rules.md
```

There is no chunk identifier, no offset, no page reference. The chat model receiving this output knows *which document* the answer came from but not *which passage*. Faithfulness checking at the MCP-mediated path is therefore by-document, not by-passage.

The REST `/chat` endpoint returns richer source data including chunk text and scores, so faithfulness checking is possible there. If chunk-level provenance matters in your stack, prefer the REST path or extend the MCP server to surface chunk identifiers.

## Orchestrator spawns MCP server per call

In stacks that wrap MCP servers as on-demand stdio subprocesses (e.g. OpenClaw), the server is spawned fresh on every tool call. Python startup plus module import adds 3–5 seconds of fixed overhead per call on typical hardware.

This is platform behaviour, not an MCP-spec requirement. Long-lived MCP servers exist; the choice between on-demand and long-lived is at the orchestrator layer. If sustained tool-call latency matters, the optimisation is in the orchestrator's MCP integration, not in the RAG plumbing.

## Tool name prefixing across the boundary

The MCP server advertises tools by their bare names (`rag_query`, `list_workspaces`, `health`). When the orchestrator registers the server, it prepends the server name plus a double underscore, producing `<server-name>__rag_query` etc. for the chat model.

This means the same tool has two names depending on where you are looking:

- Inside the MCP server's source and `tools/list` response: `rag_query`.
- In the chat model's tool-call JSON and the orchestrator's registration table: `<server-name>__rag_query`.

Knowing which side of the prefix a caller is on saves real time when debugging "tool not found" errors.

## `topN` filter is post-similarity, not pre-similarity

A common assumption: `topN: 4` means "return the four chunks above the similarity threshold." In practice, `topN` is the maximum number of chunks returned, and `similarityThreshold` is a separate filter. Chunks below the threshold are excluded; chunks above the threshold are ordered by score and the top `topN` are returned.

Implication: bumping `topN` can surface the right chunk if it was just below the cutoff at the original `topN`, but it cannot rescue a chunk that is below `similarityThreshold`. If queries fail with `sources: []` consistently, lowering `similarityThreshold` is the next lever, not raising `topN`.

## Conversation context carryover at the chat-bot layer

When the user-facing channel is a chat bot with a multi-turn thread, content retrieved in one turn often reappears in the next turn's answer even when the user did not ask. This is the chat model using its full context window — earlier tool results are still in context for the next turn.

For validation: ask each test question in a fresh thread. For operation: be aware that workspace content can leak across turns in a way that is sometimes useful (continuity) and sometimes a privacy concern (when boundaries between workspaces matter for the user).

## Prompt-token count is high even with empty sources

Even when `sources` is empty, the chat response's `metrics.prompt_tokens` is typically in the 2,500–3,000 range for a 5,000-word document. This is the workspace's `openAiPrompt` plus the chat-history scaffolding plus formatting overhead. It is not the document being silently included.

Implication: the floor cost of every RAG query is non-trivial in tokens, regardless of retrieval quality. For high-volume use, the workspace prompt is worth optimising; for one-off validation, ignore.

## Embedded documents are referenced by both `id` and `docId`

Workspace responses include each document with two identifiers: an `id` (integer, internal AnythingLLM document index) and a `docId` (UUID, in metadata). The `id` is unstable across re-uploads; the `docId` is the durable reference. When recording a Layer 4b persistence baseline, capture the `docId`, not the `id`.

## AnythingLLM version drift can change behaviours quietly

AnythingLLM ships frequently. A subset of behaviours (default chunk size, embedding model, exact `chatMode` system prompt wording) can change between minor versions. After any upgrade, re-run at minimum Layer 1 and Layer 4a, and inspect the chunking of one document to confirm the chunk-size default has not shifted under you. The runbook tests are the durable signal; the platform is the variable.
