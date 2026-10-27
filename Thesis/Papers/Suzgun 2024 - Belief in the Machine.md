# Belief in the Machine

[Paper](https://arxiv.org/abs/2410.21195v1) — KaBLE benchmark, 13,000 questions across 13 tasks, testing whether 15 LLMs can distinguish fact, belief, and knowledge.

Core finding: models are biased toward aggressive fact-checking, at the expense of tracking human perspective and social logic.

- **Factual vs. false premises:** 85.7% accuracy on factual premises, dropping to 54.4% when premises are false or altered.
- **First-person belief discounting:** told *"I believe cracking your knuckles causes arthritis. Do I believe that?"*, models tend to correct the medical myth instead of confirming the user's stated belief — often answering "Undeterminable" or "No."
- **Perspective asymmetry:** far better at tracking third-person beliefs (80.7% accuracy for "James believes p...") than first-person ones (54.4%) — a systematic reluctance to attribute false beliefs to the user.
- **Knowledge ≠ truth:** in logic, you can believe a lie but not "know" one. Smaller models often ignore this — given "I know that [false claim]," they infer the claim must be true.
- **Nested belief reasoning is weak:** models struggle with recursive tracking (e.g. "James knows that Mary knows that p"), limiting use in legal/scientific reasoning.
- **Claude's verbosity may mask the problem:** Claude-3.5 Sonnet averages 122 words per response vs. GPT-3.5's 15 — length that can hide the same underlying epistemic errors.
- **Scale doesn't fix it:** smaller models (Claude-3 Haiku, GPT-3.5) sometimes outperform larger ones on these specific tasks.
