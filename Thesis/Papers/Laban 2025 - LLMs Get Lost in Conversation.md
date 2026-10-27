# LLMs Get Lost in Multi-Turn Conversation (Laban 2025)

[Paper](https://arxiv.org/abs/2505.06120) — Laban, Hayashi, Zhou, Neville (Microsoft Research & Salesforce Research)

- Average **39% performance drop** in multi-turn, underspecified settings vs. single-turn.
- Not a reasoning problem: best-case aptitude drops only 16%. The real driver is **unreliability**, which more than doubles (+112%) — once a model takes a wrong turn, it doesn't recover.
- Failure modes: answering before requirements are clear, bloating old answers instead of rewriting (20–300% longer, worse output), forgetting constraints introduced mid-conversation, and verbosity that introduces unprompted assumptions.
- Doesn't fix it: more test-time reasoning (longer responses increase derailment risk), greedy decoding, or re-stating requirements every turn (recovers only 15–20% of the gap).
- Practical workaround: don't debug a derailed chat. Ask the model to consolidate all requirements into one prompt, then paste that into a fresh chat.

Related: [[Kartáč 2026 - BOULDER]] (same finding, different benchmark).
