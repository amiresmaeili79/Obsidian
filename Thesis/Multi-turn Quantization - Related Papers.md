# Multi-turn Quantization — Related Papers and Where We Stand

Search date: 2026-11-17. Every paper below was checked on its arXiv page. Numbers marked "check before citing" in a paper's note came from an extracted summary of the full text and should be confirmed in the PDF.

## Our work in short

We study what 4-bit weight quantization (AWQ, W4A16, group size 128, calibrated on chat-formatted UltraChat) does to thinking models that must track the state of a conversation. Each episode is a 10-turn shopping-cart dialogue: the model reports the full cart after every turn, and the first wrong state ends the episode. We test Qwen 3.5 4B, 9B and 27B (DeltaNet layers mixed with attention) and dense Granite 4.2 3B, 8B and 30B, in English, Slovak and Persian, with an 8k thinking budget and 3 seeds × 150 episodes.

4-bit costs the large models almost nothing (−0.7 and −2.0 points of episode success) but costs the small and mid-size models a lot (Qwen 4B −14.3, Granite 8B −15.0, Granite 3B −22.0). 4-bit multiplies each turn's error rate by a factor that belongs to the model, and the factor measured in English predicts the losses in Slovak and Persian. So translated languages lose more points because their baseline makes more mistakes, not because 4-bit treats them worse. 4-bit also makes mid-size models think 17–31% longer per episode and multiplies runaway turns (thinking until the cap, always wrong) by about four. An ablation shows that keeping only the attention weights at 16-bit recovers 47–65% of the loss for 0.4–2.5 GB, 4–12× more per GB than protecting the MLP, and it works in every language.

## The papers

| Paper | Setting | What it adds for us |
|:---|:---|:---|
| [[Jang 2026 - Flat Score Amplified Failures]] | τ²-bench agents, AWQ, Qwen 3.5 9B/27B | **Closest.** A lenient score hides 4-bit damage; a strict error budget reveals it |
| [[Li 2024 - Evaluating Quantized LLMs]] | MT-Bench, 2 turns | First multi-turn look; names "several turns" as future work |
| [[Lee 2024 - QDPO Conversational Abilities of Quantized LLMs]] | MT-Bench, English + Korean | Fixes quantized chat with DPO; candidate for the post-training idea |
| [[Lee 2025 - Quantization Trade-offs Edge to Giant]] | MT-Bench, 1B–405B | Small models suffer most; harder tasks lose most |
| [[Dong 2025 - Can Compressed LLMs Truly Act]] | Agent benchmarks | Single steps survive 4-bit; long tasks lose 10–15% |
| [[Lian 2026 - Quantization Inflates Reasoning]] | Thinking models, multi-turn BFCL | 4-bit thinking models use more tokens |
| [[Lotfi 2026 - Quantized Reasoning Models Overthink]] | Reasoning models, single questions | Overthinking under quantization; word-level mechanism |
| [[Lee 2026 - SemKV]] | KV-cache quantization, MT-Eval | Errors compound through earlier answers |
| [[Marchisio 2024 - How Quantization Affects Multilingual LLMs]] | 23 languages, single prompts | Non-Latin scripts and math suffer most |
| [[Soualhi 2026 - The Multilingual Quantization Tax]] | Qwen 3.5 2B/4B, 8 languages | Some languages collapse in small 4-bit models |

Also related, with older notes: [[Liu 2025 - Quantization Hurts Reasoning]] (4-bit near-lossless in single-turn reasoning) and [[Laban 2025 - LLMs Get Lost in Conversation]] (multi-turn degradation without quantization).

## Where we stand

Multi-turn quantization has been studied before, so the thesis must not claim to be first. The earlier work, however, stops at two turns (MT-Bench), or reports one score per agent task, or quantizes the KV cache instead of the weights. None of it combines thinking models, turn-by-turn state tracking and several languages, and none looks inside the model for the cause.

Our broad results are **confirmations**, and should be cited as such: 4-bit hurts small models more (Lee 2025, Liu 2025), hurts non-English more (Marchisio 2024, Soualhi 2026), makes reasoning models think longer (Lian 2026, Lotfi 2026), and hurts multi-turn tasks more than single steps (Li 2024, Dong 2025, Jang 2026).

What is **ours**:

1. **Turn-level measurement over 10 turns.** Earlier papers stopped at two turns or reported one score per task. We show how a per-turn multiplier compounds into a large episode loss, and that the multiplier carries across languages.
2. **The combination**: thinking models, multi-turn state tracking, and Slovak and Persian.
3. **Runaway thinking as a cause of multi-turn failure**, not only a cost in tokens.
4. **The attention-vs-MLP ablation with a tested memory recipe.** No multi-turn study has done this.

Jang 2026 also supports one of our design choices: their strict error budget behaves like our "stop at the first wrong turn", and it is what makes the 4-bit damage visible.

**Framing for the thesis:** "extends multi-turn quantization studies to many turns, thinking models and three languages, and locates a cheap fix", not "the first study of multi-turn quantization".
