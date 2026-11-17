# Improving Conversational Abilities of Quantized LLMs via Direct Preference Alignment (Lee et al., 2024)

[arXiv 2407.03051](https://arxiv.org/abs/2407.03051) — Janghwan Lee et al., ACL 2024 (main)

## What they did

They observe that 4-bit models can score close to full precision on standard benchmarks yet still drift in conversation: an answer leaves the full-precision wording early and turns repetitive. To fix this they propose **QDPO** (Quantization-aware Direct Preference Optimization). For each prompt, the full-precision model's answer is the "preferred" one and the quantized model's answer the "rejected" one, and the quantized model is trained with DPO to prefer the full-precision behaviour. They test 7B models (Mi:dm-7B, Vicuna-v1.5) at W4A16 against RTN, AWQ and distillation, on MT-Bench (two turns) in English and Korean. QDPO brings the quantized models' chat quality almost back to the original (for Mi:dm, a judge score of 4.03 against 4.09 for full precision and 3.55 for RTN).

## How it is like our work

It is the closest paper to the post-training idea for later: make a small quantized model think and answer like its 16-bit version. It also tests a non-English language, as we do.

## What they left open, and what we do

They measure chat quality in two turns with a judge, not exact state tracking over many turns, and they use non-reasoning models. They do not ask which part of the model causes the drift. Our results suggest where QDPO-style training could aim in thinking models: runaway thinking, which is always wrong in our data, and the turn-by-turn compounding of errors. Our attention-kept models would also make a stronger starting point than plain 4-bit.

**Possibly the "DTO" my supervisor mentioned:** check with them.

**Check before citing:** the score numbers above came from an extracted summary of the paper.
