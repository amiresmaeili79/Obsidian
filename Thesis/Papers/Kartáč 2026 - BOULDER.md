# BOULDER: Isolated vs. Dialogue Reasoning (Kartáč 2026)

[Paper](https://arxiv.org/abs/2603.20133) — Kartáč, Lango, Dušek (2026)

- Benchmark of 8 travel-reasoning tasks (arithmetic, temporal, spatial), each tested isolated, in dialogue, and in dialogue with a strict brevity constraint.
- All 8 tested LLMs lose accuracy moving from isolated to dialogue. Causes: multi-turn interaction disrupting the reasoning chain, "helpful assistant" role-play biasing toward conversational fluff, and tool-use context adding load even when tools aren't needed.
- Worst case: dialogue + brevity constraint — realistic conversational limits compound the damage.
- Takeaway: benchmarks that test reasoning in isolation overestimate real-world LLM robustness.

Related: [[Laban 2025 - LLMs Get Lost in Conversation]] (same finding, different benchmark).
