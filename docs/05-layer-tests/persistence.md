# Layer 4b — Persistence across container restart

Tests that workspace configuration and vector embeddings survive a restart of the AnythingLLM container. A failure here means the RAG state is in-memory only, and your workspace would silently empty out on the next node reboot, deploy, or crash.

## What this layer proves

- The workspace's chat-model configuration (provider, model, mode, `topN`, `similarityThreshold`, temperature, refusal string) persists to host disk.
- The document metadata (filename, doc ID, embedding timestamp, word count) persists.
- The vector embeddings themselves persist — a query after restart returns non-empty `sources` without any re-uploading.
- The container starts cleanly from disk, indexes the persisted vectors, and serves queries within a reasonable recovery window.

## What this layer does not prove

- That the embedding model itself is unchanged across versions. A platform upgrade that changes the embedding model would invalidate the persisted vectors and require re-embedding. This test catches simple persistence; it does not catch model-version drift.
- That the host disk is durable. If your AnythingLLM storage volume is on ephemeral disk (some container platforms do this by default), the test will pass within a single container lifetime but the data will not survive a host reboot.

## The probe

Three steps, performed in sequence.

### Step 1 — Capture pre-restart state

Run a representative Layer 1 query and record the full response:

```
python3 -c "import json,sys;print(json.dumps({'message':'<your validation question>','mode':'chat'}))" \
  | curl -s -X POST -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
    --data-binary @- "<BASE>/api/v1/workspace/<SLUG>/chat" \
  | tee /tmp/pre_restart.json
```

Note the `textResponse`, the `sources` array length and top-1 score, the document IDs visible in source metadata, and the workspace settings (`chatProvider`, `chatModel`, `topN`, etc.) via a `GET` on `/api/v1/workspace/<SLUG>`.

### Step 2 — Restart the container

On the AnythingLLM host:

```
docker restart anythingllm
```

The container exits and restarts. Expect a brief outage during which AnythingLLM returns connection-refused or 5xx. On typical hardware this window is well under a minute; on resource-constrained hosts it may take longer if the embedding model needs to reload.

Any in-flight RAG calls during the outage will fail. If your stack has user-facing channels actively querying the workspace, schedule the restart for a quiet window.

### Step 3 — Verify recovery and re-query

Poll for liveness:

```
for i in 1 2 3 4 5 6 7 8 9 10; do
  code=$(curl -s -o /dev/null -m 3 -w "%{http_code}" -H "Authorization: Bearer <KEY>" "<BASE>/api/v1/system")
  echo "attempt $i: HTTP $code  $(date +%H:%M:%S)"
  [ "$code" = "200" ] && break
  sleep 2
done
```

Once `/api/v1/system` returns 200, re-fetch the workspace state and re-run the same query as in Step 1:

```
curl -s -H "Authorization: Bearer <KEY>" "<BASE>/api/v1/workspace/<SLUG>"

python3 -c "import json,sys;print(json.dumps({'message':'<same question as Step 1>','mode':'chat'}))" \
  | curl -s -X POST -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
    --data-binary @- "<BASE>/api/v1/workspace/<SLUG>/chat" \
  | tee /tmp/post_restart.json
```

## What to read in the response

Compare `/tmp/pre_restart.json` and `/tmp/post_restart.json`. The expected outcome:

- **Workspace config identical.** Every field returned by the workspace `GET` matches the pre-restart values. Most importantly: `chatProvider`, `chatModel`, `chatMode`, `topN`, `similarityThreshold`, `openAiTemp`, and `queryRefusalResponse` all preserved.
- **Document metadata identical.** Same `docId`, same `wordCount`, same embedding `published` timestamp. The timestamp identical to before is the strongest signal that vectors were not re-embedded; AnythingLLM is using the on-disk vectors as they were before the restart.
- **Query answer functionally equivalent.** The `textResponse` may differ slightly word-for-word (the model is generative) but the substantive content matches. `sources` is non-empty with similar scores to pre-restart.

If `sources` is empty after restart on a query that returned `sources` before restart, the vectors did not persist. Possible causes: the storage volume is ephemeral, the embeddings index is rebuilt asynchronously and was not yet ready, or the platform upgraded the embedding model and invalidated old vectors. Investigate before relying on the workspace.

## Common observations

### Recovery time

Recovery is typically under a minute on a workstation-class host. If you observe much longer, the container may be re-importing or re-indexing on startup — a sign that some persistence is not optimal even if the test passes. Worth investigating, but not necessarily a failure.

### Workspace config preserved but document gone

Indicates AnythingLLM is persisting workspace settings (which live in a small SQLite-style store) but not the vector data (which lives in LanceDB on disk). Both should be on the same persistent volume. Check the container's volume mounts.

### Query answer changes substantively

If the post-restart answer references different facts than the pre-restart answer, the underlying retrieval may have changed. Inspect `sources` in both files: are they the same chunks? If not, either the embedding model changed (AnythingLLM upgrade) or `similarityThreshold` is near the edge of relevance for some chunks and they swap in or out across runs.

## Recording the result

Save both `/tmp/pre_restart.json` and `/tmp/post_restart.json`, plus the workspace `GET` responses. The diff between pre- and post-restart is the durable evidence that this layer passed at this point in time. Repeat after any platform upgrade.
