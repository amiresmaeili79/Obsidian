# Key Findings — Qwen3.5-9B DST, Quantization × Thinking Mode

150 episodes, dialogue state tracking task.

## Phase 1 — No-Think

- **KV8 is free:** W16_KV8 = W16_KV16 (83.3% both). +78% cache capacity, zero accuracy cost.
- **W4 has a real cost:** ~8pp lower episode success than W16. W4_KV16 > W4_KV8 (75.3% vs 73.3%), as expected.
- **Format errors near-zero** (0.1–0.2%) across all conditions — quantization doesn't break format compliance here.
- **No chain-of-thought needed:** query-turn accuracy is 97–100% without a scratchpad.
- **Hallucinations are 100% catalog-sourced**, never fabricated — shows up as catalog-bleed continuation drift, worse under W4.
- `cancel_last_add` failures (~7% W4, 1–2% W16) are a model capability ceiling, not a quantization artifact.

## Phase 2 — Think-On

- Thinking is net harmful for structured-output DST: every condition degrades vs. no-think (W4_KV16 −12.6pp, W16_KV16 −4pp).
- Format errors become the dominant failure mode (W4 thinking 2.7% vs W16 thinking 0.7%).
- W4_KV8 beats W4_KV16 in thinking mode (67.3% vs 62.7%) — KV8's precision loss seems to act as a regularizer, preventing over-retrieval of edge-case constraints that W4_KV16 misapplies.
- W16_KV8 is hurt more by thinking than W16_KV16 (−7.3pp vs −4pp) — KV8 slightly amplifies thinking instability even at full weight precision.
- 20x latency cost for a net negative accuracy outcome.

## Cross-Cutting

- W4+KV8 in no-think crosses an instruction-following stability threshold: spontaneous thinking-token generation despite suppression. Neither W4_KV16 nor W16_KV8 alone triggers this.
- Thinking mode is not a universal fix for quantization loss — it amplifies W4's cost instead of recovering it, at least on structured output tasks.

## Appendix — KV Cache Profiling (from vLLM logs)

**Qwen**

| Config    | KV Cache Dtype | Max KV Cache (Tokens) | Accuracy | Gap vs Base-BF16 |
|:----------|:---------------|-----------------------:|---------:|------------------:|
| Base-BF16 | bf16           | 1,402,850               | 61.3%    | 0.0%               |
| Base-FP8  | fp8            | 2,497,266 (+78%)        | 61.7%    | +0.4%              |
| AWQ-BF16  | bf16           | 1,615,129               | 59.3%    | −2.0%              |
| AWQ-FP8   | fp8            | 2,882,721 (+78%)        | 59.7%    | −1.6%              |

**Ministral**

| Config    | KV Cache Dtype | Max KV Cache (Tokens) | Accuracy | Gap vs Base |
|:----------|:---------------|-----------------------:|---------:|-------------:|
| Base-BF16 | bf16           | 384,896                 | 61.6%    | 0.0%          |
| Base-FP8  | fp8            | 755,120 (+96%)          | 61.5%    | −0.1%         |
| AWQ-BF16  | bf16           | 468,864                 | 57.8%    | −3.8%         |
| AWQ-FP8   | fp8            | 933,168 (+99%)          | 55.7%    | −5.9%         |
