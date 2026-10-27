# Measuring Faithfulness in Chain-of-Thought Reasoning

[Paper](https://arxiv.org/abs/2307.13702) — Lanham et al. (Anthropic & Oxford), 2023

**Question:** does the model actually use its chain of thought (CoT) to get the answer, or is the CoT a post-hoc story?

**Answer:** it depends on the task and the model size. There is no single answer.

## Method: Change the CoT, See if the Answer Changes

| Intervention | What they did | If the answer changes → |
|:---|:---|:---|
| Early answering | Cut the CoT after each sentence, force an answer | The model uses the later reasoning |
| Adding mistakes | Insert one error into the CoT, let the model continue | The model uses the CoT content |
| Paraphrasing | Reword the CoT | Hidden information in the exact wording (encoded reasoning) |
| Filler tokens | Replace the CoT with "..." | The gain is only extra compute |

Models: 175B RLHF assistant (main), plus a size series (810M to 175B). Tasks: 8 multiple-choice benchmarks (ARC Easy/Challenge, AQuA, HellaSwag, LogiQA, MMLU, OpenBookQA, TruthfulQA) and a synthetic addition task.

## Findings

- **Task dependence:** on ARC and OpenBookQA, the CoT changed the answer in < 10% of cases (mostly post-hoc). On AQuA (math), > 60% (the CoT is used).
- **Early answering and adding mistakes agree:** same task ranking.
- **No encoded reasoning:** paraphrased CoT keeps the accuracy.
- **Not extra compute:** filler tokens do not help.
- **Inverse scaling:** larger models are *less* faithful. 13B was the most faithful on 6 of 8 tasks. On addition, post-hoc reasoning increases with model size and with easier problems.
- **Faithfulness ≠ accuracy gain:** the two are only weakly correlated.
- **Limit:** each test is a partial check. Without ground truth on the internal process, faithfulness cannot be fully proved.

## Relevance to My Thesis (Idea 1 — Trace Swap)

- **Same method:** "early answering" = cut the trace + force an answer. Cite it for the forced-answer step on exhausted traces.
- **Main risk:** the answerer can ignore the given trace. Then a good answer from a W4 trace says nothing about the W4 reasoning. → Add a **no-trace control** (empty thinking block) to measure how much the trace is used.
- **Size matters for the control:** larger models can use their CoT less. If 27B ignores the trace more than 9B, a "no difference" at 27B can have the wrong cause. → Run the no-trace control **per size**, not only at 9B.
- **Task matters:** DST is a state-tracking task. The trace use can be different from math. Measure it; do not assume it.
