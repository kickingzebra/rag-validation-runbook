# Failure-mode triage — decision tree

When a validation query returns a wrong, fabricated, or evasive answer, the question is which stage of the pipeline failed. This decision tree localises the fault using the data already in the chat response (`sources`, `textResponse`, `metrics`) plus one or two follow-up probes. Use this page when something fails; do not skip to remediation without first identifying the stage.

## The five primary failure modes

The pipeline fails in five distinct ways. Each has a different signature and a different remedy.

1. **Retrieval failure** — the relevant chunk exists in the document but does not appear in `sources`.
2. **Chunking failure** — the relevant fact exists in the document but no single chunk contains it as its own retrievable unit; the fact is buried inside a larger chunk dominated by other content.
3. **Generation grounding failure** — the relevant chunk *is* in `sources` but `textResponse` ignores it and answers from training-data priors instead.
4. **Refusal-path failure** — `sources` is empty (or all below similarity threshold) and the model still produces a confident answer instead of using the configured `queryRefusalResponse`.
5. **Cross-workspace leak** — `sources` contains chunks from a workspace other than the one being queried. (Rare; if seen, it is a serious infrastructure failure, not a tuning problem.)

## The decision tree

Start at Step 1 and follow the path the answer takes you down.

### Step 1 — Is the relevant chunk in `sources`?

Open the source document. Find the passage that contains the answer to the query. Now look at every chunk in the `sources` array. Does any chunk contain that passage?

- **Yes** — go to Step 4.
- **No** — go to Step 2.

### Step 2 — Bump `topN` and re-run

Increase `topN` from its current value (typically 4) to 8 or 12 and re-run the same query. Did the relevant chunk appear in `sources` this time?

- **Yes** — this was a **retrieval failure** at the original `topN`. The right chunk exists and is reachable; it just was not in the top-N at the original cutoff. Remedy: keep the higher `topN`, or raise `similarityThreshold` so weak chunks stop crowding the top-N. See `docs/appendix-api-snippets.md` for the workspace `update` payload.
- **No** — go to Step 3.

### Step 3 — Inspect chunking

Open AnythingLLM's workspace settings or the document detail view and examine how the source document was split into chunks. The relevant fact may exist inside a larger chunk whose embedding is dominated by other content, leaving no chunk addressable for that specific fact.

This is a **chunking failure**. No `topN` setting can fix it because the unit of retrieval is the chunk itself. Remedies, in order of effort:

- Restructure the source document so the buried fact has its own subheading or stands out structurally — often the cheapest fix and the only fix that does not require re-embedding.
- Reduce chunk size (e.g. 1000 → 500 characters) and increase overlap. This produces more, smaller chunks; the buried fact is more likely to land in a chunk centred on it.
- Re-upload after restructuring. AnythingLLM will re-chunk and re-embed.

The signature of a chunking failure: the fact exists in the source, queries about it fail at any `topN`, and inspecting chunks shows the fact is embedded inside a chunk dominated by adjacent content.

### Step 4 — Compare `textResponse` against the relevant chunk

The right chunk is in `sources`. Now compare `textResponse` against that chunk's content.

- The answer faithfully paraphrases the chunk → no failure; the pipeline worked.
- The answer ignores the chunk and produces a generic answer that could have been written without the document → **generation grounding failure**.
- The answer mixes content from the relevant chunk with invented specifics not in the chunk → **partial generation failure**, often confused with retrieval failure.

For both generation failures, no `topN` change will help. The retrieved context is fine; the model is choosing to lean on its training priors instead. Remedies:

- Switch to a stronger chat model. Small models (3B parameter range) frequently fabricate on general topics where they have a strong training prior; larger models are more likely to use in-context grounding faithfully. This is the most common single fix.
- Lower `openAiTemp` further (e.g. 0.2 → 0.1 → 0.0). Reduces creative drift.
- Use `chatMode: query` instead of `chat` if your AnythingLLM version exposes it; the `query` system prompt is stricter about grounding.
- Tighten the workspace `openAiPrompt` to explicitly forbid answers that go beyond retrieved context.

### Step 5 — Was `sources` empty?

If the original query returned `sources: []`, you are dealing with either a refusal-path failure or a correct refusal. Check `textResponse`:

- It returns the configured `queryRefusalResponse` verbatim → correct refusal, no failure.
- It returns a natural-language refusal in the model's own words ("there is no mention of X in the provided context") → acceptable; the model is being honest even though it did not use the configured string. Note this in your records.
- It returns a confident answer with specifics → **refusal-path failure**. The model is fabricating from training priors despite empty retrieval.

The remedy is the same as for generation grounding failure: stronger model, lower temperature, stricter system prompt. There is no AnythingLLM setting that *forces* the configured refusal string when retrieval is empty.

## Cross-workspace leak (rare)

If `sources` contains a chunk whose `title` or content does not match the workspace under test, the LanceDB partition isolation has failed. This is rare and typically indicates an AnythingLLM bug or a mid-flight migration error. Capture the exact response, restart the AnythingLLM container, and re-run; if the leak persists, it is a platform-level bug worth opening upstream.

## What this tree does not cover

- **Whole-platform outage.** If AnythingLLM itself is unreachable or returning 5xx, run pre-flight checks first (`docs/06-pre-flight-checks.md`) and only resume here when the platform is back.
- **Inference-backend failure.** A wedged Ollama runner can produce hangs and weird timeouts that look like RAG failures but are not. The pre-flight page covers the symptoms and the recovery procedure.
- **Latency anomalies.** A correct answer that takes too long is not a correctness failure. Use the latency-decomposition method in `docs/05-layer-tests/mcp-standalone.md` to split model time from RAG time.
