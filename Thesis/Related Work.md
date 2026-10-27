# Related Work — Final Experiments

Papers for each idea in [[Experiment Ideas]]. Search date: 2026-10-19. Most papers are arXiv preprints, and for most only the abstract was read. Before you cite a number, examine the paper. 

---

## 1. Most Important: Lotfi et al. (2026)

**"Quantized Reasoning Models Think They Need to Think Longer, but They Do Not"** — Meta/FAIR, May 2026. [arXiv 2606.00206](https://arxiv.org/abs/2606.00206)

- Models: R1-Distill Qwen 1.5B–14B, R1-Distill Llama 8B, QwQ-32B. Methods: GPTQ and AWQ (3-bit, 4-bit), FlatQuant.
- "Overthinking error" = wrong final answer **or** reasoning that never terminates. In up to 52% of quantized failures, a correct answer appears earlier in the thinking.
- Token-level KL between BF16 and quantized: the largest differences are at "Wait", "But", "Alternatively" (high-entropy positions, ρ = 0.92). Math tokens differ little.
- Fix: a fixed logit penalty on 50 "overthinking" words. CoT 12–23% shorter, accuracy same or better. Control: a penalty on random or low-KL tokens does not help.
- Largest effect at 3-bit; 4-bit is "typically less disruptive". The penalty also shortens BF16 → quantization **amplifies** an existing behavior.

**Not in their paper (our contribution):** P(`</think>`) analysis, trace swap W16 ↔ W4, multi-turn task, multilingual data, a significant penalty at **4-bit** under a 4096-token limit.

You must cite this paper. Use their word penalty as a second baseline in idea 5.

## 2. General Background

| Paper | Link | Relation |
|:---|:---|:---|
| Liu et al. 2025, "Quantization Hurts Reasoning?" (COLM 2025) → [[Liu 2025 - Quantization Hurts Reasoning]] | [2504.04823](https://arxiv.org/abs/2504.04823) | W4 near-lossless in single-turn (32k limit). Same direction as our results, much smaller effect. |
| "Quantization Inflates Reasoning" 2026 | [2606.25519](https://arxiv.org/abs/2606.25519) | Quantized models keep accuracy but use more CoT tokens. |
| Min et al. 2026, "Stop When Reasoning Converges" → [[Min 2026 - Stop When Reasoning Converges]] | [2605.17672](https://arxiv.org/abs/2605.17672) | 41–52% of reasoning tokens come after the final answer (full precision). Background only. |
| The Kaitchup (blog), "Qwen3.5 Quantization: Similar Accuracy, More Thinking" | [link](https://kaitchup.substack.com/p/qwen35-quantization-similar-accuracy) | Same model family: W4 fine with thinking off, unstable at 9B with thinking on, ~2× more truncation at 32k. Single-turn, no statistics. Must cite. |
| Marchisio et al. 2024 (EMNLP Findings) | [2407.03211](https://arxiv.org/abs/2407.03211) | Quantization hurts non-Latin scripts and math most. |
| Laban et al., "LLMs Get Lost in Multi-Turn Conversation" (ICLR 2026) → [[Laban 2025 - LLMs Get Lost in Conversation]] | [2505.06120](https://arxiv.org/abs/2505.06120) | Multi-turn degradation, no quantization. |

## 3. Per Idea

### Idea 1 — Trace Swap. Status: unique for quantization.

| Paper | Link | Relation |
|:---|:---|:---|
| Lanham et al. 2023 → [[Lanham 2023 - Measuring Faithfulness in CoT]] | [2307.13702](https://arxiv.org/abs/2307.13702) | Cut the CoT, force an answer: same method. Risk: the answerer can ignore the trace. |
| "Reasoning that Travels" 2026 | [2605.28913](https://arxiv.org/abs/2605.28913) | Trace of one model given to another; has a "force answer" mode. |
| DEER 2025 | [2504.15895](https://arxiv.org/abs/2504.15895) | "Answer inducing" step: justifies the forcing sentence. |

No paper found that crosses W16 and W4 traces.

### Idea 2 — Stop Probability. Status: supported; unique for quantization.

| Paper | Link | Relation |
|:---|:---|:---|
| ThinkBrake 2025 | [2510.00546](https://arxiv.org/abs/2510.00546) | Log-prob margin between top token and `</think>` as stop signal. Correct answers get overwritten. |
| "The Evolution of Thought" 2025 | [2508.17627](https://arxiv.org/abs/2508.17627) | `</think>` gets rank 1 only some steps after the answer is found. |
| "Do Thinking Tokens Help or Trap?" 2025 | [2506.23840](https://arxiv.org/abs/2506.23840) | "wait" gets ~0.88 probability at these points. |
| "Reasoning Models Know When They're Right" 2025 | [2504.05419](https://arxiv.org/abs/2504.05419) | Hidden-state probe; early stop, 24% fewer tokens. |

No paper found that compares P(`</think>`) for W16 and W4 on the same text.

### Idea 3 — Reasoning Calibration. Status: already done → replication.

| Paper | Link | Relation |
|:---|:---|:---|
| Liu et al. 2025 | [2504.04823](https://arxiv.org/abs/2504.04823) | GPTQ + self-generated reasoning: +9.8 vs WikiText2. No AWQ ablation found. |
| "Attend to Your Own Thoughts" 2026 | [2608.01078](https://arxiv.org/abs/2608.01078) | Self-generated CoT is the best calibration. |
| LAQuant 2026 | [2605.08755](https://arxiv.org/abs/2605.08755) | Reasoning-trace calibration + lookahead loss. |
| "Think Before You Prune" 2025 | [2511.18864](https://arxiv.org/abs/2511.18864) | Same result for pruning. |
| Williams & Aletras 2024 (ACL) | [2311.09755](https://arxiv.org/abs/2311.09755) | Calibration data has a large effect. |

New parts: AWQ 4-bit on a multi-turn task, the control without thinking, the effect on P(stop).

### Idea 4 — Language. Status: partly supported; the combination is unique.

| Paper | Link | Relation |
|:---|:---|:---|
| Yong et al. 2025 | [2505.05408](https://arxiv.org/abs/2505.05408) | Same "quote the user text, think in English" behavior. |
| Etxaniz et al. 2024 (NAACL), "self-translate" | [2308.01223](https://arxiv.org/abs/2308.01223) | Supports the optional test with the English translation. |
| PolyMath 2025 | [2504.18428](https://arxiv.org/abs/2504.18428) | Thinking length ≤ 1.45× across languages. Ours: sk 1.6×, fa 2.5×. Discuss. |
| "Hidden Language Consistency" 2026 | [2608.08447](https://arxiv.org/abs/2608.08447) | Quantization and thinking language; no thinking length. |

### Idea 5 — Push the Model to Stop. Status: the general idea is done; this version is unique.

| Paper | Link | Relation |
|:---|:---|:---|
| s1, Muennighoff et al. 2025 | [2501.19393](https://arxiv.org/abs/2501.19393) | Budget forcing: the baseline. |
| NoWait 2025 (EMNLP Findings) | [2506.08343](https://arxiv.org/abs/2506.08343) | "Wait"/"Hmm" logits to −∞; CoT 27–51% shorter. |
| AlphaOne 2025 | [2505.24863](https://arxiv.org/abs/2505.24863) | Replaces late "wait" with `</think>`. |
| Lotfi et al. 2026 | [2606.00206](https://arxiv.org/abs/2606.00206) | Word penalty in quantized models: the nearest method. |
| "Pareto Frontiers of Reasoning Termination" 2026 | [2610.13570](https://arxiv.org/abs/2610.13570) | Compares fixed budgets, early exit and routing. |

New part: a length-dependent bias on `</think>`, tuned to the W16 length distribution.

## 4. Summary

| Idea | Status |
|:---|:---|
| 1 Swap | Unique |
| 2 Stop probability | Unique for quantization |
| 3 Calibration | Replication |
| 4 Language | Combination unique; interaction not significant in our data |
| 5 Stop bias | Small step from Lotfi et al. |

**Watch:** set a Google Scholar alert for papers that cite Lotfi et al. The field moves fast.
