# Quantization Hurts Reasoning? (Liu et al., 2025)

[Paper](https://arxiv.org/abs/2504.04823) — Liu et al. (Huawei Noah's Ark & Tsinghua), COLM 2025

First large benchmark of quantized reasoning models. Main message: **4-bit weight-only quantization is near-lossless.** This is the baseline claim my thesis argues does not transfer to multi-turn dialogue.

## Setup (§3.1)

- Models: R1-Distill Qwen 1.5B–32B, R1-Distill Llama 8B/70B, QwQ-32B, Qwen3-8B (Table 5; thinking mode not stated).
- Methods: weight-only (AWQ, GPTQ, group size 128), KV cache, weight+activation.
- Benchmarks: AIME, MATH-500, GSM8K, GPQA-Diamond, LiveCodeBench. **Single-turn only.**
- Generation: **32,768-token limit**, temperature 0.6, top-p 0.95, **3 seeds**.
- Truncation at 32k: **never reported**.

## Key Numbers — AWQ W4G128, Average Accuracy

| Model | BF16 | AWQ W4 | Drop |
|:---|---:|---:|---:|
| R1-Distill-Qwen-7B | 63.8 | 61.7 | −2.1 |
| R1-Distill-Llama-8B | 59.8 | 56.0 | −3.9 |
| R1-Distill-Qwen-14B | 71.5 | 70.4 | −1.1 |
| R1-Distill-Qwen-32B | 74.8 | 74.4 | −0.4 |
| QwQ-32B | 78.7 | 77.9 | −0.8 |

- 7B drop is largest on the hardest task: AIME −3.9; GSM8K −0.5.

## Other Findings

- **Output length (§4.1, Fig. 3):** W4 output length "closely aligns" with BF16. Only 3-bit and W4A4KV4 produce longer outputs. Plot only: no numbers, no distribution tails.
- **Difficulty (§3.4):** harder tasks lose more.
- **Size:** larger models are more robust.
- **Calibration (§4.2, App. C.1):** reasoning data instead of WikiText2 gives +9.8 for **GPTQ**. AWQ was calibrated on Pile; no AWQ calibration ablation found.
- **Test-time scaling (§4.4):** quantized models gain less from longer thinking.
- **Qualitative (App. D):** quantized models "follow a similar reasoning pattern" as BF16. No analysis of loops.
- **Their limitation:** the mechanisms are "incompletely understood"; causal analysis left for future work.

## Relation to My Results

Same direction, much larger effect:

| | Liu et al. | My results |
|:---|:---|:---|
| W4 hurts small models more | Yes (−2 to −4 at 7–8B, −0.4 at 32B) | Yes (−17 pp at 9B, ~0 at 27B) |
| Large models robust | Yes | Yes (27B) |
| Harder items lose more | Yes (AIME) | Yes (ambiguous operations exhaust most) |
| W4 near-lossless | Yes (single-turn, 32k) | **No** at 9B with thinking; yes with thinking off |

**Why the effect is larger in my setting:**

1. **4096 vs 32k token limit:** a small increase in length becomes exhaustion.
2. **Episode stops at the first error:** per-turn exhaustion 0.5% → 2.3% compounds over ~9 turns (~96% vs ~81% of episodes without exhaustion).
3. **Averages hide tails:** they report mean length; I count turns that cross a hard limit.

**Thesis framing:** not "opposing" — "4-bit quantization, near-lossless in single-turn benchmarks (Liu et al.), causes a large penalty for small models in multi-turn dialogue with a realistic token limit."

## Risk and Answer

- **Expected examiner question:** "Is the effect only an artifact of the short 4096 limit?"
- **Argument:** 4096 tokens per turn is realistic for interactive dialogue.
- **Optional check (~121 turns, cheap):** regenerate the 9B W4 exhausted turns with a 16k limit. Finish + correct → W4 "needs more tokens". Never stops → W4 "does not stop". Both results support the thesis. Fits idea 2 in [[Experiment Ideas]].

## Still to Verify

- Table 5: Qwen3-8B W4 numbers, and whether thinking was on.
