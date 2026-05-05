# Overview — the four-layer model

A RAG workspace is fully operational only when it answers questions correctly through every layer of the stack a real user touches. Testing one layer in isolation proves that layer works; it does not prove the workspace is reachable from where users actually ask questions. This runbook treats validation as a four-layer concern, with each layer testable independently and contributing different diagnostic signal.

## The four layers

### Layer 1 — REST direct

The AnythingLLM HTTP API at `/api/v1/workspace/{slug}/chat` is the foundation of every other layer. A direct REST query confirms that embedding, vector storage, retrieval, and generation all work in their simplest possible form. If Layer 1 fails, every other layer will fail; if Layer 1 passes, higher layers can still fail for reasons unrelated to RAG.

This layer is the right place to grade retrieval quality (`sources` array), generation faithfulness (does `textResponse` quote or paraphrase the source, or invent), and refusal behaviour when retrieval comes up empty.

### Layer 2 — MCP standalone

If your stack uses a Model Context Protocol server to bridge an orchestrator to AnythingLLM, that MCP layer adds a process boundary, a tool-naming convention, and a JSON-RPC schema. Layer 2 tests the MCP server in isolation by piping line-delimited JSON-RPC into its stdio harness, with no model in the loop.

Layer 2 confirms two things you cannot infer from Layer 1 alone: the tool contract (argument names, response shape, error format) is what callers expect; and the MCP wrapping latency is bounded. Subtracting Layer 2 latency from Layer 3 latency tells you whether a slow user-facing answer is the model's fault or the RAG plumbing's fault.

### Layer 3 — End-to-end via the assistant surface

The actual user-facing surface in this stack is a Telegram bot (Noor) that talks to an orchestrator (OpenClaw) that runs an Ollama-served model that decides to call MCP tools. Layer 3 sends a real message and observes the answer, grading both the content and the round-trip latency.

Layer 3 is the only layer that confirms the workspace is reachable as users will reach it. It is also the slowest and most coupled — many failure modes can produce a wrong answer here, and Layers 1 and 2 are the way to localise which one.

### Layer 4 — Data hygiene

Two distinct probes:

- **4a — Workspace isolation.** A query against workspace A must not retrieve chunks from workspace B. Test by asking workspace A a question whose answer only exists in workspace B, then confirming `sources` is empty. If chunks from B leak in, vector partitioning is broken at the storage layer.
- **4b — Persistence across restart.** Stop and restart the AnythingLLM container, then re-run a Layer 1 query. The same answer with non-empty sources confirms vectors persist to host disk; an empty `sources` array or a refusal confirms the embedding store is in-memory-only and your RAG state would not survive a node reboot.

Layer 4 protects against silent corruption modes that Layers 1–3 will not catch on a healthy day. If 4a or 4b fail, the workspace is one operational event away from giving wrong answers without any obvious symptom.

## Sequencing recommendation

When validating a fresh workspace, run layers in this order: pre-flight checks, Layer 1, Layer 4a, Layer 2, Layer 4b, Layer 3. The rationale: pre-flight rules out infrastructure noise; Layer 1 establishes a baseline; 4a is cheap and catches a particularly bad failure mode early; Layer 2 confirms the orchestrator-side contract; 4b requires a service interruption and is best done before the user-facing test rather than after; Layer 3 is the slowest and confirms operational reachability last.

When debugging a regression, run only the layers necessary to localise the fault. The triage decision tree in `docs/04-failure-mode-triage.md` walks that path.
