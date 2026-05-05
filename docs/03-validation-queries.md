# Validation queries — the five-query template

A validation set is more useful than ad-hoc questions because each query targets a specific failure mode. Five queries are enough to cover the failure surface of a typical doc-shaped corpus without overweighting any single mode. Adapt the wording to your corpus, but keep the *shape* of each query the same.

## The five-query taxonomy

### Query 1 — Direct lookup baseline

Ask about a fact that is named in a clear heading or table row in the source document, such that the answer should retrieve cleanly. Phrase the question close to the source phrasing.

What this tests: the most basic case of retrieval plus generation. If Query 1 fails, retrieval itself is broken — do not bother with the next four. The expected output is a verbatim or near-verbatim paraphrase of the named fact, with `sources` listing the chunk that contains it.

### Query 2 — Carve-out / exception lookup

Ask about a specific exception, carve-out, or edge case that is buried inside a longer rule or paragraph rather than named in a heading. The answer should be precise (the exact carve-out wording) rather than a general summary of the surrounding rule.

What this tests: whether retrieval surfaces the *specific* chunk containing the exception, rather than a *related* chunk that mentions the parent topic. A common failure here is the model paraphrasing the parent rule instead of citing the carve-out.

### Query 3 — Cross-rule synthesis

Ask a question whose answer requires combining two distinct sections of the document. For a rulebook corpus, this is "rule A applied under condition B from rule C." For a technical manual, it is "feature X behaviour when used with subsystem Y."

What this tests: whether retrieval surfaces both relevant chunks simultaneously and whether the chat model can combine them coherently. A weak chat model may receive both chunks but still invent a categorisation. This is the query most sensitive to the size of the chat model — a 3B model often fabricates a structure that does not exist in the source.

### Query 4 — Disambiguation between adjacent rules

Pick two rules or sections that sound similar on a surface read but have distinct meanings. Ask the workspace to explain the difference. The expected answer differentiates them in the source's own terms, not by general industry definitions.

What this tests: whether retrieval pulls the right two chunks rather than conflating them, and whether the model leans on retrieved context rather than its training-data prior on similar-sounding terms. A correct answer that quotes the source's specific phrasing is the signal that retrieval and generation are both working.

### Query 5 — Out-of-corpus negative test

Ask about something that genuinely does not appear in the corpus. Two variants are useful:
- A token that is never present in any document loaded into the workspace.
- A topic adjacent to the corpus but not covered by it (e.g. a rule that *might* apply but is not actually written).

What this tests: the refusal path. With a strict workspace prompt and a configured `queryRefusalResponse`, the model should refuse cleanly. In practice, many models will fabricate from training-data priors and stitch in unrelated retrieved chunks. Query 5 is the most informative query in the set because it exposes failures that the first four cannot.

## Grading method

For every query, evaluate three things separately. Confounding them is a common diagnostic mistake.

### Retrieval quality

Open the source document. For each chunk in the `sources` array, ask: does this chunk contain the answer the query was asking about? A single relevant chunk in the top-N is enough; if `sources` is non-empty but no chunk is actually about the question, retrieval has failed even if generation produces a plausible answer.

For Query 5, retrieval quality is judged inversely: empty or low-relevance `sources` is the *correct* outcome.

### Generation faithfulness

Compare `textResponse` against the relevant chunks in `sources`. Three categories of result:

- **Faithful paraphrase** — the answer restates the source content in different words but preserves all specifics. This is the goal.
- **Faithful quote** — the answer quotes source phrasing directly. Acceptable but indicates low generation effort; usually fine.
- **Fabrication** — the answer contains specifics (numbers, names, exceptions, lists) that are not in any retrieved chunk and not in the source document. This is the failure mode that erodes user trust quietly.

For Query 5, the generation goal is either the configured refusal string verbatim, or a clear "I do not have information about this" framing. A confident answer to Query 5 is a fabrication.

### Refusal behaviour

Did the configured `queryRefusalResponse` fire when retrieval was thin? If `sources` is empty and the model still produced a confident answer, the refusal path is not enforced — the answer is invented from training-data priors. This is a known platform behaviour rather than a bug; the refusal string is advisory and the model decides whether to use it. See `docs/appendix-known-quirks.md`.

## Recording results

For each query, log: the question, `textResponse`, the top-N `sources` with scores, and a one-line grade against each of retrieval / generation / refusal. The combined record makes the failure-mode triage in `docs/04-failure-mode-triage.md` actionable. Keep the raw JSON responses; chunk-text wording matters during follow-up.
