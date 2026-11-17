# Flat Score, Amplified Failures: How the Error Budget Masks Damage in Quantized LLM Agents (Jang et al., 2026)

[arXiv 2607.27275](https://arxiv.org/abs/2607.27275) — Jiwon Jang et al., preprint (July 2026)

**The closest paper to our work.**

## What they did

They test quantized agents on τ²-bench, where an agent helps a simulated customer (retail and telecom) through many turns with stateful tools: 114 tasks, 4 trials each. Models include Gemma-4 (31B, 26B-A4B), Qwen-3.6 (27B, 35B-A3B) and **Qwen-3.5 27B and 9B**, at 16, 8 and 4 bits, with **AWQ** for 4-bit, weights only. Their main point is in the title: the final task score stays flat (Gemma-4-31B telecom: 66.67 vs 65.35, not significant), while failures inside the episode double (tool-name hallucinations 19.5% → 38.3% of calls). The benchmark lets an agent make up to 10 errors before ending an episode, and that tolerance hides the damage. With a budget of only 2 errors, the 4-bit gap grows from 1.3 to 16.7 points. Their fix is a repair loop that lets the agent retry a failed call up to three times.

## How it is like our work

Same quantization method, the same Qwen 3.5 family at two of our sizes, a multi-turn task with state, and the same core message: a single final score underestimates what 4-bit does across many turns. Their error-budget result supports our design choice directly. Our episodes end at the first wrong turn, which is their strictest budget, and that is exactly why our gaps are large and clear.

## What they left open, and what we do

They study large agents (9B and up) in English with tool calls, and they do not look at thinking length or at which part of the model causes the damage. We cover the small models where 4-bit hurts most (Qwen 4B, Granite 3B and 8B), score the full dialogue state at every turn, add Slovak and Persian, show that 4-bit makes mid-size models think longer and loop, and find a cheap fix inside the model: attention at 16-bit. Our Qwen 9B and 27B results match theirs, with small final-score gaps (−2.1 and −0.7 points).

**Check before citing:** they have per-turn results in Figure 6; we only know this from an extracted summary.
