# Hallucination Notes — Missing Sources

Two notes without a paper reference. Find the paper, then move each to `Papers/` with the naming convention.

## Hallucinations Snowball

Probably Zhang et al. 2023, "How Language Model Hallucinations Can Snowball". Verify first.

Models keep defending a wrong answer instead of correcting it, because they optimize for conversational consistency over accuracy.

- Example: asked if 9677 is prime, ChatGPT said no and backed it with a false factorization (13 × 745) — but catches the same error when asked about it separately.

## Zero-Shot Generation from Knowledge Graphs

Paper unknown.

ChatGPT was given (subject, relation, object) triples — e.g. Ronaldo / Portugal / Soccer — and asked to generate natural sentences in English and Russian, across factual, counterfactual, and fictional data.

- English output was fine; Russian was noticeably worse.
- **Factual** input → more confident hallucinations that contradict the prompt.
- **Fictional** input → more likely to silently drop information.
- **Counterfactual** input → sentences rated less coherent by human readers.
