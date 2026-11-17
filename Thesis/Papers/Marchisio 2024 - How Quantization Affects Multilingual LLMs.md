# How Does Quantization Affect Multilingual LLMs? (Marchisio et al., 2024)

[arXiv 2407.03211](https://arxiv.org/abs/2407.03211) — Kelly Marchisio et al. (Cohere), Findings of EMNLP 2024

## What they did

The first focused study of quantization across languages. They quantize Command R+ (103B), Command R (35B) and Aya 23 (35B, 8B) at 8 and 4 bits with several methods (GPTQ, SmoothQuant, bitsandbytes), and test about 23 languages, with 10 in focus. Three findings stand out. Non-Latin-script languages lose more than Latin-script ones (−1.9% vs −0.7% for the 103B model). Automatic metrics hide much of the damage, which human raters see (Japanese: 1.7% drop by metrics, 16.0% by humans). Math reasoning is hit hardest (−13.1% on MGSM for the 35B model at 4-bit).

## How it is like our work

It is the standard reference for "4-bit hurts non-English languages more", which we confirm: Granite 8B loses 8.0 points in English, 15.3 in Slovak and 21.6 in Persian. Persian uses a non-Latin script, which fits their script finding.

## What they left open, and what we do

They test single prompts, so they cannot tell whether a language loses more because 4-bit damages each step more, or because its baseline already makes more mistakes that then add up. We can separate the two. The per-turn error multiplier is about the same in English, Slovak and Persian for most models, and the multiplier measured in English predicts the Slovak and Persian losses. So the extra loss comes from compounding a weaker baseline, not from 4-bit treating the language worse. Neither Slovak nor Persian is in their language set.
