# Test Think and No Think on Qwen3.5-9B on 150
Key Findings
Phase 1 — No-think

W16_KV8 = W16_KV16 at 150 episodes (83.3% = 83.3%). KV8 quantization is a free lunch for W16: +78% cache capacity, zero measurable accuracy cost.
W4 carries a real accuracy penalty (~8pp episode success rate vs W16). W4_KV16 > W4_KV8 (75.3% vs 73.3%) — expected direction confirmed.
Format errors near-zero in no-think (0.1-0.2%) across all conditions. Quantization does not cause format compliance failures in no-think mode.
Query turn accuracy is excellent without scratchpad (97-100%). Full-state reconstruction does not require chain-of-thought at 9B.
Hallucination is 100% catalog-sourced. Never fabricated. Manifests as catalog-bleed continuation drift, especially under W4.
cancel_last_add is a model capability ceiling (~7% W4, 1-2% W16), not a quantization artifact.

Phase 2 — Think-on

Thinking mode is net harmful for strict structured output DST. All conditions degrade vs no-think. W4_KV16 loses 12.6pp, W16_KV16 loses 4pp.
Format error is the dominant thinking-mode failure mechanism. W4 thinking: 2.7% vs W16 thinking: 0.7%.
W4_KV8 > W4_KV16 in thinking mode (67.3% vs 62.7%). Mechanism confirmed: KV8 precision reduction acts as cognitive regularization — prevents precise over-retrieval of edge-case constraints that W4_KV16 misapplies.
W16_KV8 is more hurt than W16_KV16 by thinking (−7.3pp vs −4pp) — KV8 slightly amplifies thinking instability even at W16.
20x latency cost for thinking with net negative accuracy outcome on this task.

Cross-cutting

W4+KV8 combination in no-think crosses an instruction-following stability threshold. Manifests as spontaneous thinking token generation despite suppression. Neither W4_KV16 nor W16_KV8 alone triggers this.
Thinking mode is not a universal mitigation for quantization degradation. It amplifies W4 costs rather than recovering them on structured output tasks.

---
## T11: KV Cache Optimization Profiling Qwen
*Extracted from vLLM engine logs. Shows the maximum number of tokens that can fit in the GPU KV Cache based on the data type used, and the corresponding accuracy trade-off.*

| Config    | KV Cache Dtype | Max KV Cache Size (Tokens) | Overall Accuracy | Accuracy Gap vs Base-BF16 |
|:----------|:---------------|---------------------------:|-----------------:|--------------------------:|
| Base-BF16 | bfloat16 (16b) | 1,402,850                  | 61.3%            | 0.0%                      |
| Base-FP8  | fp8 (8b)       | 2,497,266 (+78%)           | 61.7%            | +0.4%                     |
| AWQ-BF16  | bfloat16 (16b) | 1,615,129                  | 59.3%            | -2.0%                     |
| AWQ-FP8   | fp8 (8b)       | 2,882,721 (+78%)           | 59.7%            | -1.6%                     |


## T11: KV Cache Optimization Profiling Ministral
*Extracted from vLLM engine logs. Shows the maximum number of tokens that can fit in the GPU KV Cache based on the data type used, and the corresponding accuracy trade-off.*

| Config    | KV Cache Dtype | Max KV Cache Size (Tokens) | Overall Accuracy | Accuracy Drop vs Base |
|:----------|:---------------|---------------------------:|-----------------:|----------------------:|
| Base-BF16 | bfloat16 (16b) | 384,896                    | 61.6%            | 0.0%                  |
| Base-FP8  | fp8 (8b)       | 755,120 (+96%)             | 61.5%            | -0.1%                 |
| AWQ-BF16  | bfloat16 (16b) | 468,864                    | 57.8%            | -3.8%                 |
| AWQ-FP8   | fp8 (8b)       | 933,168 (+99%)             | 55.7%            | -5.9%                 |