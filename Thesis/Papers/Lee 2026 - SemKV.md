# SemKV: Semantic Mixed-Precision KV Cache Quantization (Lee et al., 2026)

[arXiv 2608.28911](https://arxiv.org/abs/2608.28911) — Daeha Lee et al., preprint (August 2026)

## What they did

They quantize only the **KV cache** (the stored attention keys and values), not the weights, and find a "quality cliff": below a certain bit width quality collapses, and above it the loss is small. Their method, SemKV, gives more bits to the tokens that matter most (mixed precision per token). They test Llama-3.1-8B-Instruct and Mistral-7B, including the MT-Eval multi-turn benchmark. In multi-turn dialogue they find a small but significant leftover loss even above the cliff, and they explain it by errors passing forward through the dialogue: each turn is built on earlier answers that were already slightly changed.

## How it is like our work

Their explanation for the multi-turn loss is the same compounding idea behind our results: a small change per turn becomes a large change over a conversation. They also support "protect what matters, quantize the rest", which is the logic of our attention-kept recipe.

## What they left open, and what we do

They quantize the KV cache, not the weights, test non-reasoning models in English, and report results per task type rather than per turn. We quantize the weights, use thinking models in three languages, and measure the error rate at every turn. Our own 4k grid also tested an 8-bit KV cache and found it nearly free (−0.8 points over 64 comparisons), so in our setting the damage comes from the weights, not the cache.
