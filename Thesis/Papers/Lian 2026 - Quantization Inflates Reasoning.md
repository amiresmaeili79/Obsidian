# Quantization Inflates Reasoning: Token Inflation as a Hidden Cost of Low-Bit Reasoning Models (Lian et al., 2026)

[arXiv 2606.25519](https://arxiv.org/abs/2606.25519) — Xinyu Lian et al., preprint (June 2026)

## What they did

They show that quantized reasoning models often keep their accuracy but spend more reasoning tokens, so the memory saving is partly paid back in compute. They test weight-only quantization (GPTQ, AWQ, HQQ, ParoQuant, RTN) at 4 and 3 bits with group size 128, on thinking models: Qwen3-4B-Thinking-2507, Phi-4-Reasoning-Plus (14B) and Qwen3-30B-A3B-Thinking. One test is multi-turn: on the BFCLv4 multi-turn tool-calling benchmark, Qwen3-4B's reasoning grows from 6,556 to 7,425 tokens per step at 4-bit (+13%). They try several fixes: a repetition penalty, prompting for conciseness, better calibration data and quantization-aware training. Repetition penalties shorten the reasoning but cost accuracy.

## How it is like our work

It confirms our token finding with a similar small thinking model and a multi-turn benchmark: 4-bit thinking models think longer. Our 8k runs show the same for mid-size models (17–31% more thinking tokens per episode for Qwen 9B, Granite 8B and Granite 3B), and nothing for the large ones.

## What they left open, and what we do

They treat longer thinking as a cost in tokens, not as a cause of failure. We show the link to failure: a turn that runs into the cap is always wrong, and 4-bit makes such runaway turns about four times more common in mid-size models. They do not break results down by turn, test only English, and do not ask which part of the model causes the extra length. We show that no single half owns it: in Granite 3B, 4-bit attention drives the runaways, while in Granite 8B the MLP looked responsible in English only.
