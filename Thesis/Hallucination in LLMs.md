# Hallucination in LLMs

## Hallucinations Snowball

Models keep defending a wrong answer instead of correcting it, because they optimize for conversational consistency over accuracy.

- Example: asked if 9677 is prime, ChatGPT said no and backed it with a false factorization (13 × 745) — but catches the same error when asked about it separately.

## Zero-Shot Generation from Knowledge Graphs

ChatGPT was given (subject, relation, object) triples — e.g. Ronaldo / Portugal / Soccer — and asked to generate natural sentences in English and Russian, across factual, counterfactual, and fictional data.

- English output was fine; Russian was noticeably worse.
- **Factual** input → more confident hallucinations that contradict the prompt.
- **Fictional** input → more likely to silently drop information.
- **Counterfactual** input → sentences rated less coherent by human readers.

## Is Hallucination Control Even Possible?

[Paper](https://arxiv.org/pdf/2506.06382)

Formal impossibility result: no LLM can be simultaneously truthful, information-conserving, knowledge-revealing, and knowledge-optimal on non-trivial queries. Hallucination isn't a fixable bug — it's a structural limit of the inference process itself.

## Anti-Hallucination Prompting Has a Cost

[Paper](https://arxiv.org/pdf/2601.02023)

Telling a model not to hallucinate helps some models a lot and barely moves others — and the fix has a "safety tax."

- ChatGPT-5-mini: faithfulness 74.1% → 89.8% with anti-hallucination (AH) instructions.
- Gemini-2.5-flash: barely moves (86.5% → 87.0%). Claude-4.5-haiku: slightly worse (83.2% → 82.4%).
- Where AH instructions help faithfulness, they tend to hurt literal extraction and logical inference — worse at large context sizes.
- Longer context alone doesn't help either: performance drops when relevant evidence is diluted or spread out.
