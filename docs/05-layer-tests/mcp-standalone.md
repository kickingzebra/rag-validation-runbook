# Layer 2 — MCP standalone

Tests the Model Context Protocol stdio server in isolation, without the orchestrator (e.g. OpenClaw) or the chat model that normally invokes it. This layer confirms the tool contract, surfaces the exact JSON-RPC schema, and gives a clean latency baseline for the MCP-mediated path.

## What this layer proves

- The MCP server starts cleanly given its required environment variables.
- The advertised tool schema matches what callers expect (argument names, types, required fields, response shape).
- The MCP server can complete a `tools/call` for `rag_query` against the target workspace and return a grounded answer plus source citations.
- The end-to-end MCP latency (subprocess spawn + import + RPC handshake + RAG round-trip) is bounded.

## What this layer does not prove

- That the orchestrator passes the right tool name, arguments, or workspace slug at runtime. The orchestrator namespaces tools (e.g. prepending the server name with a double underscore), so what callers send may differ from the bare MCP tool name.
- That the chat model decides to invoke the tool when it should. That is a model-behaviour question, not a transport question.

## Tool contract (from the originating run)

The server exposes three tools by their bare MCP names:

- `rag_query` — required arguments `workspace` (slug, string) and `question` (natural language, string). Returns `{"content": [{"type": "text", "text": "<answer>\n\n---\nSources:\n- <filename>"}]}`. On error, returns the same shape with `isError: true`.
- `list_workspaces` — no arguments. Returns a text listing of slugs, names, and document counts.
- `health` — no arguments. Returns a JSON-formatted reachability report.

When the orchestrator registers the server, it prepends the server name plus a double underscore to each tool. So the same tool that is `rag_query` to the MCP server is `<server-name>__rag_query` to the model. Knowing which side of that prefix a caller is on saves real time during debugging.

## The probe

The probe pipes line-delimited JSON-RPC into the MCP server's stdin and captures stdout. Three messages: `initialize`, `tools/list`, and one `tools/call` for `rag_query`.

Save this script to `/tmp/mcp_probe.py` on the host running the MCP server:

```python
#!/usr/bin/env python3
import json, subprocess, time

reqs = [
    {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"runbook","version":"0.1"}}},
    {"jsonrpc":"2.0","id":2,"method":"tools/list","params":{}},
    {"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"rag_query","arguments":{"workspace":"<SLUG>","question":"<QUESTION>"}}},
]
stdin_payload = "\n".join(json.dumps(r) for r in reqs) + "\n"
t0 = time.time()
p = subprocess.run(
    ["python3", "server.py"],
    input=stdin_payload, capture_output=True, text=True, timeout=180,
    cwd="<MCP_SERVER_DIR>",
)
elapsed = time.time() - t0
print(f"=== elapsed: {elapsed:.2f}s   exit: {p.returncode} ===")
print("--- STDOUT ---")
print(p.stdout)
if p.stderr.strip():
    print("--- STDERR ---")
    print(p.stderr)
```

Run with the same environment variables the orchestrator passes to the MCP server (typically `ANYTHINGLLM_BASE` and `ANYTHINGLLM_KEY`):

```
ANYTHINGLLM_BASE=http://127.0.0.1:3001 \
ANYTHINGLLM_KEY=<KEY> \
python3 /tmp/mcp_probe.py
```

Three responses come back on stdout, one per request.

## What to read in the response

For `id:1` (initialize):

```
{"jsonrpc":"2.0","id":1,"result":{"protocolVersion":"2024-11-05","capabilities":{"tools":{}},"serverInfo":{"name":"<SERVER_NAME>","version":"<X.Y.Z>"}}}
```

Confirms the server started and speaks the protocol version expected.

For `id:2` (tools/list):

The full tool array. Each tool has `name`, `description`, and `inputSchema`. Confirm `rag_query`'s required arguments are exactly `workspace` and `question`. Mismatched argument names — e.g. `query` instead of `question` — are a common source of silent tool-call failures from a model that picks the more obvious name.

For `id:3` (tools/call rag_query):

```
{"jsonrpc":"2.0","id":3,"result":{"content":[{"type":"text","text":"<answer>\n\n---\nSources:\n- <filename>"}]}}
```

The `text` field is the formatted answer plus a Sources section. Note: source citations here are filename-only by default, not chunk references. The model knows which document the answer came from but not which chunk; faithfulness checking in higher layers cannot rely on chunk-level provenance.

## Latency decomposition

Subtract Layer 2 latency from Layer 3 latency to isolate the model's contribution to a slow user-facing answer.

Example from one originating run:

- Layer 2 (MCP standalone, including the workspace's chat-model generation): ~16 seconds.
- Layer 3 (Telegram round-trip with a tool-calling model in the loop): ~6 minutes.
- Inferred model overhead (tool-call decision + final-answer generation): roughly 5 min 44 s.

This split is the most useful piece of information when a user complains about a slow chat experience. It tells you whether to optimise the RAG plumbing or the model side.

## Common observations

### Subprocess spawn cost is per-call

Many orchestrators spawn the MCP server as a fresh subprocess for every tool call rather than running it as a long-lived process. Python startup plus module import adds roughly 3–5 seconds of fixed overhead to every Layer 2 call on typical hardware. This is platform behaviour, not a bug. If sustained latency matters, the remedy is at the orchestrator layer, not the RAG layer.

### "Missing required argument" errors

If `tools/call` returns `{"isError": true, "content": [{"text": "Missing required argument: question"}]}`, the caller used the wrong argument name. Check `tools/list` output and pass the exact argument name the schema declares.

### Workspace not found

If `tools/call` returns an `isError` with a "workspace not found" message, the slug is wrong. Slugs are case-sensitive and must match the workspace exactly as returned by `list_workspaces` or by the AnythingLLM REST `/api/v1/workspaces` endpoint.

## Recording the result

Save the entire stdout of the probe. The `tools/list` schema is durable documentation of the contract at the moment of the test, useful if a later upstream upgrade changes argument names silently.
