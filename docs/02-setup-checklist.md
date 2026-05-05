# Setup checklist — fresh workspace from scratch

This is the sequence to follow when creating a new AnythingLLM RAG workspace and preparing it for validation. Each step has an explicit verification so you do not move forward on an assumption.

## Prerequisites

Before starting:

- AnythingLLM is running and reachable on a known base URL (host plus port, e.g. `http://100.99.231.1:3001`).
- An API key is provisioned and tested. Per externalised-secrets discipline, the key lives in an environment variable, not in scripts or commit history.
- An Ollama (or other inference backend) instance is reachable from AnythingLLM with at least one model loaded.
- The source corpus is finalised as a Markdown file. RAG retrieval quality depends heavily on chunkable structure: clear headings, paragraphs that are self-contained, and named anchors for any fact you expect a query to retrieve. See `docs/04-failure-mode-triage.md` for the chunking failure mode this prevents.

## Step 1 — Identify the corpus

Decide what document or document set will be loaded. For a v1 validation, one document is enough. Note its word count and approximate token count; these inform later sanity checks against AnythingLLM's reported `wordCount` and the prompt token counts in chat responses.

Verification: read the document and write down three to five facts you expect the workspace to retrieve. These become the validation queries in `docs/03-validation-queries.md`.

## Step 2 — Create or locate the workspace

In the AnythingLLM web UI, create a new workspace with a descriptive name. Note the slug — it is the URL-safe identifier used in every API call and is generated from the name. Slugs containing trailing spaces or unusual characters are common gotchas; copy the slug from the workspace API response rather than reconstructing it from the name.

Verification:

```
curl -s -H "Authorization: Bearer <KEY>" \
  "<BASE>/api/v1/workspaces" \
  | python3 -m json.tool | grep -E '"slug"|"name"'
```

The new workspace appears with the expected slug.

## Step 3 — Upload and embed the document

Upload via the AnythingLLM web UI (drag and drop into the workspace) or via the REST API. Upload-via-UI is recommended for the first run because it makes chunking and embedding visible in the activity log; subsequent runs can be scripted against `/api/v1/document/upload` and `/api/v1/workspace/{slug}/update-embeddings`.

Verification:

```
curl -s -H "Authorization: Bearer <KEY>" \
  "<BASE>/api/v1/workspace/<SLUG>" \
  | python3 -m json.tool | grep -E '"filename"|"wordCount"|"docId"'
```

The document is listed with a `wordCount` close to the source file (AnythingLLM may strip frontmatter and special characters, so a small delta is normal). A `docId` is assigned. The `metadata.published` timestamp marks the embedding moment.

## Step 4 — Configure the chat model

A workspace with a document but no chat model can retrieve chunks but cannot generate an answer. Configure the workspace to use Ollama (or your chosen provider) and pick a model:

```
curl -s -X POST -H "Authorization: Bearer <KEY>" \
  -H "Content-Type: application/json" \
  -d '{"chatProvider":"ollama","chatModel":"<MODEL>","openAiTemp":0.2,"chatMode":"chat","queryRefusalResponse":"There is no relevant information in this workspace to answer your query."}' \
  "<BASE>/api/v1/workspace/<SLUG>/update"
```

Notes on each field:

- `openAiTemp: 0.2` — low temperature so the model paraphrases retrieved chunks rather than improvising. RAG is an extraction task, not a creative one.
- `chatMode: chat` — the workspace prompt that emphasises grounding in retrieved context. Note the appendix's "known quirks" entry: this is advisory, not enforced.
- `queryRefusalResponse` — explicit refusal string the model is *expected* to use when retrieval is thin. Same caveat: advisory.
- Leave `topN` and `similarityThreshold` at defaults for the first validation run. Tune only after a specific failure mode has been observed.

## Step 5 — Verify configuration

```
curl -s -H "Authorization: Bearer <KEY>" \
  "<BASE>/api/v1/workspace/<SLUG>" \
  | python3 -c "import json,sys;w=json.load(sys.stdin)['workspace'][0];print({k:w[k] for k in ['chatProvider','chatModel','chatMode','topN','similarityThreshold','openAiTemp']})"
```

All six fields reflect the values just set. If any field reads as `null`, the update did not take — re-issue the PUT.

## Step 6 — First-light query

Send one trivially-correct query to confirm the full Layer 1 path works before running the formal validation set:

```
python3 -c "import json,sys;print(json.dumps({'message':'Briefly summarise the document in one sentence.','mode':'chat'}))" \
  | curl -s -X POST -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
    --data-binary @- "<BASE>/api/v1/workspace/<SLUG>/chat"
```

The response should include a non-empty `textResponse`, a non-empty `sources` array, and a `metrics` block. If `sources` is empty on a summary query, retrieval is broken before validation has even started. If `textResponse` is the configured refusal string, the model is honouring grounding more strictly than expected — proceed but note the behaviour.

The workspace is now ready for the validation queries in `docs/03-validation-queries.md`.
