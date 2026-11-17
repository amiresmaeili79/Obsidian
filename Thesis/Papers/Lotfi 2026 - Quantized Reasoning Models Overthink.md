# Quantized Reasoning Models Think They Need to Think Longer, but They Do Not (Lotfi et al., 2026)

[arXiv 2606.00206](https://arxiv.org/abs/2606.00206) — Sanae Lotfi et al. (Meta FAIR), preprint (May 2026)

## What they did

They study why quantized reasoning models "overthink". They quantize DeepSeek-R1-Distill models (Qwen 1.5B, 7B, 14B; Llama 8B) and QwQ-32B with GPTQ and AWQ at 3 and 4 bits, and test on single-question benchmarks (MATH-500, GSM8K, AIME, LiveCodeBench, GPQA). The effect is strongest at 3-bit: on MATH-500, R1-Distill-Qwen-1.5B drops from 85.6% to 47.0% while its reasoning grows from 5.2K to 23.4K tokens, and overthinking errors rise from 19 to 139. Comparing token probabilities, they find the biggest changes at hesitation words such as "Wait", "But" and "Alternatively". Their fix is a fixed penalty on 50 such words, which shortens reasoning by 12–23%.

## How it is like our work

It is the main reference for "4-bit thinking models think longer, and that causes errors", which is part of our finding. Their words-level mechanism is a natural partner for our model-level ablation.

## What they left open, and what we do

They test single questions, mostly at 3-bit, and note that 4-bit is "typically less disruptive". We show that at 4-bit, in a 10-turn conversation, the extra thinking does cause real failures: runaway turns become about four times more common in mid-size models, and each one is wrong. They test only English and do not ask which part of the network produces the overthinking. We test three languages and find that the answer depends on the model (attention in Granite 3B).
