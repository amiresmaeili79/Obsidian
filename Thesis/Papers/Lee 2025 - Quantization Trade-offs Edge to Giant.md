# Exploring the Trade-Offs: Quantization Methods, Task Difficulty, and Model Size (Lee et al., 2025)

[arXiv 2409.11055](https://arxiv.org/abs/2409.11055) — Jemin Lee et al., IJCAI 2025

## What they did

A large evaluation of instruction-tuned models from 1B to 405B parameters (Llama 3.x, Vicuna, Gemma) on 13 datasets. They compare GPTQ, AWQ, SmoothQuant and FP8 at 4 and 8 bits. For conversation they use MT-Bench (80 questions, GPT-4 judge), which shows "significant performance declines in Coding and STEM tasks" under quantization. They find that small models suffer far more than 70B-scale models, and that quantized models can become overly verbose on STEM answers.

## How it is like our work

It confirms two of our background results across a much wider range of sizes: 4-bit hurts small models much more than large ones, and harder, reasoning-heavy tasks lose most. It also includes AWQ.

## What they left open, and what we do

MT-Bench gives one judged score for a short conversation, so they cannot say where in a conversation the damage appears or whether it builds up. All their tests are in English, and none uses a thinking model. We measure the error rate at every turn of a 10-turn dialogue, in three languages, with thinking models, and we explain the loss through a per-turn multiplier and runaway thinking.

**Check before citing:** a search summary claimed this paper reports a larger drop in the second turn than the first. We could not find that in the text, so do not cite it without checking.
