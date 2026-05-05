# Layer 1 — REST direct

Tests the AnythingLLM HTTP API in isolation, without any orchestrator, MCP server, or chat-bot front-end in the loop. This is the foundation; every higher layer assumes Layer 1 works.

## What this layer proves

- The embedding model accepted the document and produced vectors.
- The vector database (LanceDB by default) holds those vectors and returns them on similarity search.
- The configured chat model is reachable from AnythingLLM and produces a response.
- The workspace's prompt configuration shapes the answer as expected.

## What this layer does not prove

- That any orchestrator or MCP server can reach the workspace (covered by Layer 2).
- That the user-facing channel (chat bot, web UI integration) can reach the workspace (covered by Layer 3).
- That vector data is properly isolated or persistent (covered by Layer 4).

## The probe

Send the validation queries from `docs/03-validation-queries.md` against the workspace's `/chat` endpoint. The minimum viable test is one query:

```
python3 -c "import json,sys;print(json.dumps({'message':'<your question>','mode':'chat'}))" \
  | curl -s -X POST -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
    --data-binary @- "<BASE>/api/v1/workspace/<SLUG>/chat"
```

Note: feed the JSON via `--data-binary @-` rather than embedding it inline. Inline JSON triggers shell-quoting issues with apostrophes in the question. The Python helper builds the payload safely.

## What to read in the response

A correct Layer 1 response looks like:

```
{
  "id": "<uuid>",
  "type": "chat",
  "textResponse": "<the model's answer>",
  "sources": [
    {"title": "<filename>", "score": <0..1>, "text": "<chunk text with metadata>"},
    ...
  ],
  "close": true,
  "chatId": <int>,
  "metrics": {"prompt_tokens": ..., "completion_tokens": ..., "total_tokens": ..., "outputTps": ..., "duration": ..., "model": "<name>", "provider": "<provider>"}
}
```

Three fields matter for grading:

- `textResponse` — the answer to grade against the source document.
- `sources` — the retrieved chunks. Each has a `score` (cosine similarity, higher is more relevant), a `title` (the filename), and a `text` field with the chunk content. Inspect the `text` to confirm whether retrieval found the right passage.
- `metrics.duration` — generation time in seconds. Useful as a baseline for Layer 2 latency comparison.

The `sources` array is the single most useful field in the entire RAG pipeline for diagnosis. If a query gives a wrong answer, the question is almost always "did `sources` contain the right chunk." Open the source document and check.

## Common observations

### `sources` empty or near-empty

Either the query genuinely has no relevant content in the corpus (legitimate negative test outcome), or `topN` and `similarityThreshold` are filtering too aggressively. Try bumping `topN` to 8. If the right chunk surfaces at the higher `topN`, the original cutoff was too tight. If the right chunk never surfaces at any `topN`, see the chunking failure mode in `docs/04-failure-mode-triage.md`.

### `textResponse` correct but `sources` does not contain the obvious chunk

The model may have answered correctly from a different chunk that happens to mention the same fact, or from training-data priors that align with the source. Do not assume a correct answer means retrieval worked — always grade `sources` independently.

### Long prompt-token counts even with small documents

The workspace's `openAiPrompt` and the chat-history scaffolding contribute substantial fixed overhead per request. For an empty `sources` array against a 5,000-word document, prompt-token counts in the range of 2,500–3,000 are typical. This is not a bug; it is the cost of the workspace's system prompt.

## Recording the result

Save the raw JSON response. The chunk text in `sources` is what you compare against the source document to grade retrieval, and the wording often matters when investigating partial failures. Capturing the full response now is cheaper than re-running the query later.
