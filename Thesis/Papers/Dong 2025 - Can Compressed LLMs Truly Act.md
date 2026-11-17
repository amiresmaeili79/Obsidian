# Can Compressed LLMs Truly Act? (Dong et al., 2025)

[arXiv 2505.19433](https://arxiv.org/abs/2505.19433) — Peijie Dong et al., ICML 2025 (poster)

## What they did

A benchmark of how compression affects agent abilities. They compress 15 models, including Qwen2.5 from 1.5B to 32B and the DeepSeek-R1-Distill reasoning models, with quantization (GPTQ, AWQ, SmoothQuant) and pruning (SparseGPT, Wanda). They test workflow generation, tool use and real-world agent tasks, including multi-turn and multi-round benchmarks (WorfBench, AgentBoard). Their main finding is that "4-bit quantization preserves workflow generation and tool use (1%-3% drop) but degrades real-world application accuracy by 10%-15%".

## How it is like our work

It shows the same split we see: short, well-defined steps survive 4-bit, but longer interactive tasks that chain many steps lose much more. It also includes distilled reasoning models.

## What they left open, and what we do

They report one score per benchmark, with no breakdown by turn, so they cannot show how the damage builds up during an interaction. They do not measure thinking length or loops, test only English, and do not look inside the model. We follow the error turn by turn, link part of the loss to runaway thinking, test three languages, and locate the cause in the attention weights.
