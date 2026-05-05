# Pre-flight checks

Before any layer test, confirm the underlying infrastructure is healthy. Many apparent RAG failures are actually upstream problems — a wedged inference runner, an unreachable AnythingLLM, a misconfigured environment. Pre-flight catches these in seconds and prevents them from contaminating the diagnostic.

## Inference backend (Ollama)

### Liveness

```
curl -s -m 5 -w "\nHTTP=%{http_code} time=%{time_total}s\n" <OLLAMA_HOST>/api/version
curl -s -m 5 -w "\nHTTP=%{http_code} time=%{time_total}s\n" <OLLAMA_HOST>/api/ps
```

Expected: both return HTTP 200 in under 200 ms. `/api/version` reports the Ollama version (note this — version drift on the inference backend has been a real source of regressions). `/api/ps` reports currently-loaded models with their VRAM footprint.

### Smoke test the inference path

```
curl -s -m 60 <OLLAMA_HOST>/api/generate \
  -d '{"model":"<SMALL_MODEL>","prompt":"Reply with just: OK","stream":false,"options":{"num_predict":10}}'
```

Use a small, fast model (e.g. a 3B-parameter model) for the smoke test. Expected: a JSON response with `"response": "OK"` (or close) within a few seconds. The `total_duration` and `load_duration` fields indicate model-load time vs inference time.

### Recognising the wedged-runner symptom

A specific Ollama failure mode looks like this:

- `/api/version` returns 200 immediately.
- `/api/ps` reports a model "loaded" with a long `expires_at`.
- `/api/generate` either hangs indefinitely or returns: `{"error":"llama runner process has terminated: %!w(<nil>)"}`.

This means the inference subprocess has died but Ollama's process tracker still reports the model as loaded. The state is desynced. This pattern has been linked to AMD ROCm with high context sizes on some Ollama versions; see upstream issue trackers for the latest status.

The recovery:

```
sudo systemctl restart ollama
```

This drops the daemon and respawns the runner cleanly. Side effects: ~5–15 second outage where Ollama returns 503. Any active chat session mid-tool-call fails that call. The previously-loaded model needs to reload on next use.

If the wedge recurs after restart, the underlying bug is upstream — investigate version pinning or context-size tuning rather than restarting in a loop.

## RAG platform (AnythingLLM)

### Liveness

```
curl -s -m 5 -o /dev/null -w "HTTP=%{http_code} time=%{time_total}s\n" \
  -H "Authorization: Bearer <KEY>" \
  <ANYTHINGLLM_BASE>/api/v1/system
```

Expected: HTTP 200 in under 500 ms. A 401 means the API key is wrong; a 403 may indicate a permissions issue; 5xx or no response means AnythingLLM is down or restarting.

### Configuration sanity

```
curl -s -H "Authorization: Bearer <KEY>" "<ANYTHINGLLM_BASE>/api/v1/system" \
  | python3 -m json.tool | grep -E '"VectorDB"|"EmbeddingEngine"|"EmbeddingModelPref"|"LLMProvider"'
```

Expected: `VectorDB` is set (default `lancedb`), `EmbeddingEngine` is set (default `native`), `EmbeddingModelPref` is the embedding model name, and `LLMProvider` is the configured chat-model provider. A `null` or empty value here means the platform is misconfigured at the global level — workspaces inherit these defaults.

### List workspaces

```
curl -s -H "Authorization: Bearer <KEY>" "<ANYTHINGLLM_BASE>/api/v1/workspaces" \
  | python3 -m json.tool | grep -E '"slug"|"name"'
```

Expected: the workspace under test appears in the list with the correct slug. If a workspace is missing, either it was never created, was deleted, or the API key lacks visibility (multi-user mode permissioning).

## MCP server

### Configuration

The MCP server typically requires environment variables (`ANYTHINGLLM_BASE`, `ANYTHINGLLM_KEY`). If those are unset or wrong, the server starts but fails on first tool call with an auth or unreachable error.

Confirm the orchestrator passes the correct environment to the spawned MCP server. Check the orchestrator's MCP registration config (e.g. for OpenClaw: `openclaw mcp list` and inspect the env block).

### Path

The MCP server binary or script must be readable and executable by the orchestrator's spawn user. Permissions issues here surface as immediate "command not found" or "permission denied" errors in orchestrator logs, distinct from RAG-layer failures.

## Chat-bot front end

### Bot reachability

If the user-facing channel is a Telegram bot or similar, confirm:

- The bot is online (e.g. for Telegram: send a `/start` and see a reply).
- The orchestrator process is running on the host that owns the bot.
- The model the orchestrator routes to is loaded in the inference backend.

A bot that does not reply at all, or replies after a long silence, is most often the sign of a wedged inference runner — see the Ollama section above.

## Putting it together

The pre-flight is a 30-second sequence: Ollama version, Ollama ps, Ollama generate-on-small-model, AnythingLLM system, AnythingLLM workspaces. If all five return clean, the infrastructure is healthy and you can proceed to layer tests. If any fail, fix the failing component first; do not start RAG diagnostics on a broken platform.
