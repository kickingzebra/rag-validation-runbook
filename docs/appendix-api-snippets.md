# Appendix — API snippets

Reference snippets for the AnythingLLM REST endpoints used across this runbook. Each snippet uses placeholders (`<KEY>`, `<BASE>`, `<SLUG>`, `<MODEL>`); substitute values from your stack. All snippets assume `python3` is available for JSON building and inspection.

## Authentication

Every AnythingLLM REST call requires a Bearer token in the `Authorization` header. The key is provisioned in the AnythingLLM admin UI and lives in an environment variable, not in scripts.

```
export ANYTHINGLLM_KEY="<KEY>"
export ANYTHINGLLM_BASE="<BASE>"   # e.g. http://127.0.0.1:3001
```

Subsequent snippets use those variables.

## System and workspace listing

```
# System status
curl -s -H "Authorization: Bearer $ANYTHINGLLM_KEY" \
  "$ANYTHINGLLM_BASE/api/v1/system" | python3 -m json.tool

# All workspaces
curl -s -H "Authorization: Bearer $ANYTHINGLLM_KEY" \
  "$ANYTHINGLLM_BASE/api/v1/workspaces" | python3 -m json.tool

# One workspace by slug (includes documents and threads)
curl -s -H "Authorization: Bearer $ANYTHINGLLM_KEY" \
  "$ANYTHINGLLM_BASE/api/v1/workspace/<SLUG>" | python3 -m json.tool
```

## Update workspace settings

The workspace `update` endpoint accepts a partial JSON body. Only the fields you include are changed; other fields are preserved.

Common fields:

- `chatProvider` — string. Examples: `"ollama"`, `"openai"`, `"anthropic"`.
- `chatModel` — string. The model name as your provider expects it, e.g. `"qwen3.5:27b"` or `"llama3.2:3b"` for Ollama.
- `chatMode` — `"chat"` or `"query"`. `chat` uses a permissive system prompt that emphasises grounding; `query` is stricter.
- `openAiTemp` — float, 0.0 to 1.0. Lower is more deterministic.
- `topN` — integer. Number of chunks the vector search returns to the chat model. Default 4; 8 is a reasonable next step if Layer 1 retrieval misses chunks that exist in the document.
- `similarityThreshold` — float, 0.0 to 1.0. Minimum cosine similarity for a chunk to be included in `sources`. Default 0.25.
- `queryRefusalResponse` — string. The refusal text the model is *expected* to use when retrieval is thin. See `docs/appendix-known-quirks.md` for why this is advisory rather than enforced.
- `openAiPrompt` — string. The workspace's system prompt. Tightening this is the strongest available lever for controlling generation behaviour.

```
curl -s -X POST \
  -H "Authorization: Bearer $ANYTHINGLLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"chatProvider":"ollama","chatModel":"<MODEL>","chatMode":"chat","openAiTemp":0.2,"topN":8,"queryRefusalResponse":"There is no relevant information in this workspace to answer your query."}' \
  "$ANYTHINGLLM_BASE/api/v1/workspace/<SLUG>/update" \
  | python3 -m json.tool
```

The response includes the full updated workspace object. Confirm the changed fields reflect the new values.

## Chat (RAG query)

Build the JSON payload via Python to avoid shell-quoting issues with apostrophes in user questions.

```
python3 -c "import json,sys;print(json.dumps({'message':'<question>','mode':'chat'}))" \
  | curl -s -X POST -H "Authorization: Bearer $ANYTHINGLLM_KEY" -H "Content-Type: application/json" \
    --data-binary @- "$ANYTHINGLLM_BASE/api/v1/workspace/<SLUG>/chat" \
  | python3 -m json.tool
```

The response includes:

- `textResponse` — the model's answer.
- `sources` — array of retrieved chunks. Each has `title`, `score`, and `text` (the chunk content).
- `metrics` — token counts, duration, model name. Useful for latency analysis.
- `chatId` — server-side thread identifier; subsequent messages in the same thread can pass this to maintain context.

## Document upload (REST path)

Upload-via-UI is preferred for first-time validation because the chunking and embedding are visible in the activity log. For scripted re-uploads:

```
curl -s -X POST -H "Authorization: Bearer $ANYTHINGLLM_KEY" \
  -F "file=@<path-to-file>" \
  "$ANYTHINGLLM_BASE/api/v1/document/upload"

curl -s -X POST -H "Authorization: Bearer $ANYTHINGLLM_KEY" \
  -H "Content-Type: application/json" \
  -d '{"adds":["<doc-path-from-upload-response>"]}' \
  "$ANYTHINGLLM_BASE/api/v1/workspace/<SLUG>/update-embeddings"
```

The two-step pattern: upload populates a "custom-documents" location, then `update-embeddings` attaches the document to the workspace and triggers chunking plus embedding. Either step failing cleanly is recoverable; a successful upload followed by a failed embed leaves a document in custom-documents that can be re-attached.

## Useful inspection one-liners

Pretty-print only the fields that matter most when grading a RAG response:

```
cat /tmp/response.json | python3 -c "
import json, sys
d = json.load(sys.stdin)
print('textResponse:', d.get('textResponse', '')[:1500])
print('sources count:', len(d.get('sources', [])))
for i, s in enumerate(d.get('sources', [])[:8]):
    print(f'  [{i+1}] score={s.get(\"score\"):.3f} title={s.get(\"title\")}')
m = d.get('metrics', {})
if m:
    print(f'tokens: total={m.get(\"total_tokens\")} duration={m.get(\"duration\")}s')
"
```

This pattern recurs across layer tests; capture it once in a shell helper if you find yourself running it more than a few times.
