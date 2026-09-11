[Paper](https://arxiv.org/abs/2603.20133)
This paper introduces BOULDER, a benchmark designed to test how well large language models perform reasoning when the same task is presented in two formats:

1.  Isolated — a clean, standalone reasoning question
2.  Dialogue — the same question embedded inside a multi‑turn task‑oriented conversation
3.  Dialogue‑concise — the dialogue version but with strict constraints (e.g., max two sentences)

The benchmark covers eight travel‑related tasks requiring arithmetic, temporal, and spatial reasoning (e.g., ticket prices, train times, opening hours, distances, directional relations, shortest paths). Each example is dynamically generated and automatically verifiable.

Across eight modern LLMs, the authors find a large and consistent performance drop when moving from isolated tasks to dialogue settings. Even very strong models lose accuracy. The main causes are:

- Multi‑turn interaction, which disrupts reasoning chains
- Role conditioning, where acting as a “helpful travel assistant” biases the model toward conversational fluff instead of precise reasoning
- Tool‑use context, which adds cognitive load even when tools don’t solve the reasoning task

The strict dialogue‑concise setting performs worst, showing that real‑world constraints (brevity, politeness, formatting) further degrade reasoning. The authors conclude that current benchmarks overestimate LLM robustness and that reasoning must be evaluated in realistic interactive scenarios, not just isolated prompts.