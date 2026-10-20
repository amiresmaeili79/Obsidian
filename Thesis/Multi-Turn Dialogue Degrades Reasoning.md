# Multi-Turn Dialogue Degrades Reasoning

Two papers, same core finding: move a task from a single clean prompt into a multi-turn dialogue, and reasoning accuracy drops sharply — even for top models.

## LLMs Get Lost in Conversation

[Paper](https://arxiv.org/abs/2505.06120) — Laban, Hayashi, Zhou, Neville (Microsoft Research & Salesforce Research)

- Average **39% performance drop** in multi-turn, underspecified settings vs. single-turn.
- Not a reasoning problem: best-case aptitude drops only 16%. The real driver is **unreliability**, which more than doubles (+112%) — once a model takes a wrong turn, it doesn't recover.
- Failure modes: answering before requirements are clear, bloating old answers instead of rewriting (20–300% longer, worse output), forgetting constraints introduced mid-conversation, and verbosity that introduces unprompted assumptions.
- Doesn't fix it: more test-time reasoning (longer responses increase derailment risk), greedy decoding, or re-stating requirements every turn (recovers only 15–20% of the gap).
- Practical workaround: don't debug a derailed chat. Ask the model to consolidate all requirements into one prompt, then paste that into a fresh chat.

## BOULDER: Isolated vs. Dialogue Reasoning

[Paper](https://arxiv.org/abs/2603.20133) — Kartáč, Lango, Dušek (2026)

- Benchmark of 8 travel-reasoning tasks (arithmetic, temporal, spatial), each tested isolated, in dialogue, and in dialogue with a strict brevity constraint.
- All 8 tested LLMs lose accuracy moving from isolated to dialogue. Causes: multi-turn interaction disrupting the reasoning chain, "helpful assistant" role-play biasing toward conversational fluff, and tool-use context adding load even when tools aren't needed.
- Worst case: dialogue + brevity constraint — realistic conversational limits compound the damage.
- Takeaway: benchmarks that test reasoning in isolation overestimate real-world LLM robustness.
