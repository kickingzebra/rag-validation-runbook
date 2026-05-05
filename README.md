# RAG Validation Runbook

A runbook for validating that an AnythingLLM RAG workspace is fully operational end-to-end. Originated from a live validation run on 2026-05-04 / 2026-05-05 against workspace #10 on a self-hosted AnythingLLM instance, with the OpenClaw orchestrator and the Noor Telegram bot wired in via a custom MCP stdio server.

## What this runbook covers

A RAG workspace is "fully operational" only if it answers questions correctly across **four layers**, not just one:

1. **REST direct** — `curl` against the AnythingLLM `/api/v1/workspace/.../chat` endpoint returns grounded answers with source citations.
2. **MCP standalone** — the local MCP stdio server (e.g. `anythingllm-mcp/server.py`) can be invoked with line-delimited JSON-RPC and returns the same grounded answers, in isolation from the chat model that normally calls it.
3. **End-to-end via the assistant surface** — a question asked through the user-facing channel (here: Telegram → Noor → OpenClaw → Ollama → MCP → AnythingLLM) returns a grounded answer through the full chain.
4. **Data hygiene** — vector data is isolated per workspace (no cross-workspace leakage at retrieval time) and persists across container restarts.

A workspace that passes only Layer 1 has not been validated. A workspace that passes Layers 1, 2, and 3 but fails Layer 4 will silently corrupt under operational stress.

## How to use this runbook

- Read `docs/01-overview.md` first — it explains why each layer exists and when each test belongs in your sequence.
- For a fresh workspace, walk `docs/02-setup-checklist.md` end-to-end before any test.
- For validation, walk `docs/03-validation-queries.md` and run the five-query template against your corpus.
- When something fails, jump to `docs/04-failure-mode-triage.md` for the decision tree; that file is the single most useful page when debugging.
- Each layer has a dedicated file under `docs/05-layer-tests/` with the exact commands and expected outputs.
- `docs/06-pre-flight-checks.md` covers Ollama and AnythingLLM liveness before any layer test runs.
- The two appendix files (`appendix-api-snippets.md`, `appendix-known-quirks.md`) are reference material — keep them open while debugging.

## Scope and intent

This runbook is grounded in the experience of one specific stack: AnythingLLM (Docker, LanceDB, native embeddings) on a self-hosted Linux box, Ollama as the inference backend, an MCP stdio server bridging an orchestrator (OpenClaw), and a Telegram bot as the user-facing surface. Many findings generalise to other RAG stacks; some are platform-specific. Where a finding is platform-specific, it is flagged inline.

This is a v1 runbook. It captures what was tested in the originating session and nothing speculative. Future revisions will extend with additional probes (chunking inspection, embedding-model swap, similarity-threshold tuning) only when those probes have actually been run against a live system.

## Licence

All Rights Reserved. See `LICENSE`.
