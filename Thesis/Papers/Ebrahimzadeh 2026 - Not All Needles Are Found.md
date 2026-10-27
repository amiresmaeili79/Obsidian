# Not All Needles Are Found: Anti-Hallucination Prompts in Long Context (Ebrahimzadeh 2026)

[Paper](https://arxiv.org/abs/2601.02023) — Ebrahimzadeh, 2026

Telling a model not to hallucinate helps some models a lot and barely moves others — and the fix has a "safety tax."

- ChatGPT-5-mini: faithfulness 74.1% → 89.8% with anti-hallucination (AH) instructions.
- Gemini-2.5-flash: barely moves (86.5% → 87.0%). Claude-4.5-haiku: slightly worse (83.2% → 82.4%).
- Where AH instructions help faithfulness, they tend to hurt literal extraction and logical inference — worse at large context sizes.
- Longer context alone doesn't help either: performance drops when relevant evidence is diluted or spread out.
