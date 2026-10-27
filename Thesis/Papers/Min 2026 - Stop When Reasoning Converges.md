# Stop When Reasoning Converges (PUMA)

[Paper](https://arxiv.org/abs/2605.17672) — Min et al., May 2026 (under review). Mainly a method paper (early exit). We only need its motivation result.

## What We Cite

- **41–52% of reasoning tokens come after the model has reached its final answer.** 5 models, 5 benchmarks, full precision, single-turn math/science.
- Measure: the first step where the trial answer equals the **final** answer. All tokens after that step count as redundant.

## How to Use It

- Use as background only: "continuing to think after the answer is common, even without quantization".
- It does **not** show "had the *right* answer, then doubted it" — the final answer can be wrong. For that, cite ThinkBrake (2510.00546) and Lotfi et al. (2606.00206).
- Do not compare 41–52% with our 48% (old scorer; see to-do 7 in [[Final Experiments]]): we count exhausted traces with a *correct* draft. Different measures.

## Citation Chain

1. **Min et al.** — thinking after the answer is common (full precision).
2. **ThinkBrake** — sometimes the extra thinking overwrites a correct answer.
3. **Lotfi et al.** — quantization makes this worse.
4. **This thesis** — the same failure in multi-turn dialogue, where it causes exhaustion.
