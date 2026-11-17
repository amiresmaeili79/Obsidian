# Evaluating Quantized Large Language Models (Li et al., 2024)

[arXiv 2402.18158](https://arxiv.org/abs/2402.18158) — Shiyao Li et al. (Tsinghua), ICML 2024

## What they did

A broad benchmark of quantization across 11 model families from 125M to 180B parameters (OPT, LLaMA2, Falcon, Mistral, Gemma, Mamba and others). They quantize weights (2 to 8 bits), activations and the KV cache with RTN, AWQ and SmoothQuant, and test many abilities. For dialogue they use MT-Bench, which has **two turns** per conversation, scored by a GPT-4 judge on a 1–10 scale, with LLaMA2-chat models. Their main dialogue conclusion is that "the effects of quantization do not show significant differences for two-turn dialogues in most LLMs". At 3-bit weights or a 3-bit KV cache, most models start to repeat sentences.

## How it is like our work

It is the earliest systematic look at quantization in a multi-turn setting, and it uses AWQ, our method. Their turn-by-turn table already hints at our pattern: for LLaMA2-13B at 4-bit, the first-turn score stays the same while the second-turn score drops (Table 6).

## What they left open, and what we do

They name the gap themselves: what happens "after several turns of dialogue" is left as future work. We run 10 turns, score the full dialogue state after every turn, and show that a per-turn loss too small to notice in two turns compounds into a 14–22 point loss in episode success for small and mid-size models. They also used non-reasoning chat models, English only, and a judge score, while we use thinking models, three languages and an exact, rule-based check of the state.

**Check before citing:** the Table 6 per-turn numbers came from an extracted summary of the paper, not from reading the table directly.
