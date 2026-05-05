# Layer 4a — Workspace isolation

Tests that vector data is partitioned per workspace, so a query against workspace A cannot retrieve chunks from workspace B. A failure here is a serious infrastructure-level bug and means RAG answers may be silently contaminated with content from other workspaces.

## What this layer proves

- The vector database (LanceDB by default) enforces per-workspace partitioning at retrieval time.
- A query against workspace A returns only chunks from documents loaded into workspace A.
- Documents in other workspaces are invisible to this workspace's queries even when the question would match content in those other workspaces.

## What this layer does not prove

- That the model honours the empty-retrieval case correctly. The model may still fabricate from training-data priors when retrieval returns nothing — that is a separate failure mode (refusal-path failure, see `docs/04-failure-mode-triage.md`).
- Anything about the actual content of other workspaces. This test is about boundary enforcement, not about what is in the other side.

## The probe

Pick a workspace other than the one under test and identify a token or topic that appears in *that* workspace but not in the workspace being tested. The token must be specific enough that a substring search of the test corpus confirms it is absent.

Concrete example from the originating run: the workspace under test was `continuous-delivery-guidelines-rule-book-for-development`, loaded with `development-rules.md`. The `core-memory` workspace contains files named `BOOTSTRAP`, `SOUL`, `HEARTBEAT`, `IDENTITY`, etc. The token `BOOTSTRAP` (capitalised) appears in `core-memory` and does not appear in `development-rules.md`. So a query referencing `BOOTSTRAP` is the right shape for the probe.

Send the probe against the workspace under test:

```
python3 -c "import json,sys;print(json.dumps({'message':'What is the purpose of the BOOTSTRAP persona file?','mode':'chat'}))" \
  | curl -s -X POST -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" \
    --data-binary @- "<BASE>/api/v1/workspace/<SLUG_UNDER_TEST>/chat"
```

Run two probes, not one. The model's response varies: a single probe might happen to produce a refusal even on a leaky workspace, or a confident hallucination on a healthy one. Two probes with different tokens give a clearer signal.

## What to read in the response

The critical field is `sources`. The expected outcome is `sources: []` — an empty array. That confirms the vector database returned no chunks for the cross-workspace token, which is what isolation means.

If `sources` is non-empty *and* contains a chunk whose text references content from the other workspace (the file named in the probe), isolation has failed. Capture the full response, restart the AnythingLLM container, and re-run. If the leak persists, escalate to the platform layer — this is not a tuning issue.

If `sources` is non-empty but the chunks all come from the workspace under test (just weakly-relevant ones, not chunks that actually match the probe token), isolation is fine — the vector search returned its top-N from the local partition, none of which match well. This is normal behaviour with a low `similarityThreshold`.

The `textResponse` is secondary. The model may or may not honour the empty-retrieval case correctly:

- If `textResponse` is the configured `queryRefusalResponse` verbatim — clean refusal, ideal.
- If `textResponse` is a natural-language refusal in the model's own words — acceptable. Note the wording but do not flag as a failure.
- If `textResponse` is a confident answer with specifics about the probed token — this is a refusal-path failure, not an isolation failure. The vectors are isolated correctly; the model is fabricating from training priors. See `docs/04-failure-mode-triage.md`.

## Common observations

### `sources: []` plus a fabricated answer

The most common outcome on a healthy isolation pipeline. Vectors are partitioned correctly, but the configured refusal string is advisory rather than enforced — the model decides whether to refuse or fabricate. This is not an isolation failure and not a bug; it is platform behaviour. Treat the fabrication as a separate finding to report against the chat model, not against AnythingLLM.

### `sources: []` plus a clean natural-language refusal

The ideal outcome. Vectors are isolated, retrieval correctly returned nothing, and the model honoured the empty-retrieval case. If your stack consistently produces this on isolation probes, the model is well-suited for the strict-grounding role.

### Non-zero `sources` but with chunks from the local workspace

Vectors are isolated; the top-N just happens to contain weakly-relevant local chunks because `similarityThreshold` is low. Either accept it (and rely on the model to ignore weak chunks during generation) or raise `similarityThreshold` to filter aggressively at retrieval time.

### Non-zero `sources` with chunks from another workspace

Isolation has failed. This is rare and indicates either a serious AnythingLLM bug or a mid-flight migration error. Restart the container, re-run; if the leak persists, raise upstream and consider the workspace untrusted until the platform issue is resolved.

## Recording the result

Capture the full JSON response from each of the two probes. Note the workspace slugs used (under-test and source-of-token), the tokens probed, and whether `sources` was empty. The combined record is what later validates that this layer continues to pass after platform upgrades.
