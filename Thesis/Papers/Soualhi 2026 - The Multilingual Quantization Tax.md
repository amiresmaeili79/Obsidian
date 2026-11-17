# The Multilingual Quantization Tax: Structural Collapse and Typological Fragility in Edge SLMs (Soualhi, 2026)

[arXiv 2608.09941](https://arxiv.org/abs/2608.09941) — Mohammad Wathiq Soualhi, preprint (under review at EMNLP 2026)

## What they did

A study of 4-bit quantization in small models across languages. They quantize Gemma 4 (E2B, E4B) and **Qwen 3.5 2B and 4B** to 4-bit NormalFloat with bitsandbytes (no calibration data), with thinking turned off. They test multilingual knowledge and reasoning benchmarks in eight languages (English, Arabic, Russian, Chinese, Japanese, Hindi, Swahili, Yoruba). Some languages collapse badly: Qwen 3.5 2B in Russian drops from 23.6% to 7.5%. They also test a "forced multi-step" condition, which degrades further.

## How it is like our work

Same model family and one of our sizes (Qwen 3.5 4B), the same question of whether small 4-bit models fail differently across languages, and the same conclusion that non-English languages pay more.

## What they left open, and what we do

They use single questions with thinking off, a different 4-bit method (NF4 without calibration, where we use AWQ), and no Slovak or Persian. They point to outlier weights as a possible cause but do not test it. We add many turns, thinking models, a per-turn analysis showing the extra language loss comes from compounding a weaker baseline, and a tested fix (attention at 16-bit) that recovers a similar share of the loss in every language.

**Note:** single-author preprint, not yet peer reviewed.
