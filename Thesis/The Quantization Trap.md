# The Quantization Trap

**The myth:** 4-bit/8-bit quantization cuts memory and energy nearly for free.
**The reality:** in multi-hop reasoning, aggressive compression can collapse accuracy while *increasing* latency and energy — a "sustainability inversion."

## The 3D Sustainability Index

- **Trust:** % of reasoning accuracy preserved.
- **Economic:** tokens/sec vs. VRAM footprint.
- **Environmental:** Joules per query, including hardware power spikes.

## Why It Happens

- **Casting overhead:** GPUs lack native sub-8-bit math, so weights must keep unpacking to 16-bit at every token step.
- **Amortization failure:** sequential reasoning forces batch size ≈ 1, so this unpacking tax can't be shared — it compounds linearly with reasoning depth.

## Key Findings

- **Energy inversion (A100):** FP16 uses 235 J/query; 4-bit spikes to 503 J; 8-bit reaches 693 J. 8-bit is worst because it pays the casting tax without saving enough bandwidth.
- **Size paradox (<1B models):** matrix math finishes almost instantly, so unpacking overhead dominates — a 400% energy penalty plus accuracy collapse.
- **Multi-GPU reversal**, governed by model scale:
  - Single GPU (32B): bandwidth-bound — 4-bit savings offset the casting cost, and the trap dissolves.
  - Multi-GPU cluster (72B): space is abundant, so quantization's bandwidth advantage disappears — 4× slower, 4× more energy.
- **Stochastic regularization:** at ≥14B, 4-bit accuracy sometimes *exceeds* FP16 — quantization noise can shake the model out of deterministic failure loops.

## Trap Taxonomy

- **Full trap (0.6B–7B):** accuracy and efficiency both collapse — FP16 wins outright.
- **Efficiency-only trap (14B):** accuracy holds, but un-amortized casting still hurts speed and power.
- **Conditional dissolution (32B):** quantization works on a single GPU, only for short contexts with larger batches.
- **Memory-only victory (72B+):** VRAM drops 60%, but speed/energy suffer on multi-GPU — worth it only when memory is the hard constraint.
