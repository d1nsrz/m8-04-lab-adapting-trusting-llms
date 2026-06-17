# Eval Results

## Task 2 — LLM-as-judge pass-rate table

Two variants scored against the same fixed 15-case test set (`EVAL_SET` in the notebook), judged against an explicit rubric.

**Substitution note:** this environment has no Gemini API key, and the network can't reach `generativelanguage.googleapis.com` or `huggingface.co` (both blocked at the network level, confirmed directly). So the few-shot classifier and the judge were both run by me (Claude) directly in place of Gemini, and the embedding step used TF-IDF + cosine similarity (scikit-learn, fully local) in place of the `all-MiniLM-L6-v2` sentence-transformer. The train/eval sets, prompt structure, and rubric below are otherwise unchanged from the notebook. These are real numbers from an actual run, not estimates — just with substituted models, so a Gemini-based run could land differently.

| Variant | Cases | Passed | Pass rate |
|---------|-------|--------|-----------|
| Few-shot prompting (Claude, sub. for Gemini) | 15 | 15 | 100% |
| Embedding NN (TF-IDF, sub. for sentence-transformer) | 15 | 10 | 67% |

**Rubric used by the judge:**

> You are an impartial evaluator for a support-ticket classifier.
> A correct classification MUST match the expected label exactly.
> Valid labels: billing, bug, feature_request, other.
>
> Scoring rules:
>   PASS – the predicted label matches the expected label exactly.
>   FAIL – the predicted label is different from the expected label,
>          the response is blank, or it contains more than one word.
>
> Respond with exactly one word: PASS or FAIL.

**Verdict:**

> Few-shot won clearly: 15/15 (100%) vs. 10/15 (67%) for TF-IDF nearest-neighbor. The judge is trustworthy here, but mostly because the task makes it easy to be: both variants only ever emit one clean label from a closed set of four, so PASS/FAIL collapses to an unambiguous string-equality check with little room for the judge to misjudge. The real risk isn't judge error — it's that a bare PASS/FAIL throws away *why* something failed, which matters more for the embedding variant than the rubric lets on (see below).

**A case where the judge looked wrong:**

> "I got a refund confirmation but the money isn't back." (expected: `billing`) — TF-IDF predicted `bug`, and the judge correctly marked it FAIL. The verdict itself isn't wrong. What looks wrong is the story a FAIL implies: you'd assume the nearest-neighbor model at least confused two billing-adjacent ideas. It didn't. Checking the actual vocabulary overlap, the only token this ticket shares with its matched training example ("The app crashes every time I open the settings page") is the word "the" — and this is the single *highest-similarity* match (0.49) among all the TF-IDF misses, meaning the score that's supposed to signal confidence is, if anything, backwards here. The same pattern shows up twice more ("I'd love a calendar view for tasks" and "Do you offer discounts for non-profits?" both matched their nearest neighbor on the shared word "for" alone) — with only 10 short training documents and no stop-word filtering, TF-IDF can match on grammatical glue words instead of meaning, and a binary PASS/FAIL judge has no way to surface that distinction.
