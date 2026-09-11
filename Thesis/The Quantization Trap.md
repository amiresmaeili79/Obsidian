# 📌 Core Concept

- **The Myth:** LLM quantization (4-bit/8-bit) seamlessly cuts memory and energy with near-lossless accuracy.
- **The Reality:** In **multi-hop reasoning**, aggressive compression triggers a "Sustainability Inversion"—collapsing accuracy, slowing speeds, and spiking energy usage.

* * *

# 📊 The 3D Sustainability Index

Evaluates performance across a balanced 3D manifold:

1.  **Trust ($T_{SI}$):** Percentage of reasoning accuracy preserved.
2.  **Economic ($E_{SI}$):** Trade-off between processing speed (TPS) and VRAM footprint.
3.  **Environmental ($S_{SI}$):** Total Joules used per query, capturing transient hardware power spikes.

* * *

# ⚙️ Architectural Drivers

- **Casting Overhead:** Modern GPUs lack native sub-8-bit arithmetic; weights must constantly unpack ("cast") back to 16-bit registers at every token step.
- **Amortization Failure:** Sequential reasoning forces a batch size of 1 ($B \approx 1$). The model cannot share the unpacking tax, compounding energy costs linearly with depth.

* * *

# 📈 Key Paradoxes & Findings

- **Energy Inversion (A100):** Uncompressed FP16 uses **235J/query**, 4-bit spikes to **503J**, and 8-bit explodes to **693J**. *Why 8-bit is worst:* It pays the casting tax without saving enough bandwidth.
- **The Size Paradox (<1B Models):** Matrix math finishes instantly, so unpacking overhead dominates—causing a **400% energy penalty** and accuracy collapse.
- **The Multi-GPU Reversal:** Governed by Critical Model Scale ($N^*$):
    - *Single GPU (32B):* Bandwidth-bound. 4-bit savings offset casting costs, dissolving the trap.
    - *Multi-GPU Cluster (72B):* Space is abundant. Quantization's bandwidth advantage evaporates, making it **4× slower and 4× more energy-intensive**.
- **Stochastic Regularization:** At scales $\ge 14\text{B}$, 4-bit accuracy sometimes *exceeds* FP16. The quantization noise ($\xi_p$) acts as a regularizer, shaking the model out of deterministic failure loops.

* * *

# 📝 Trap Taxonomy

- **Full Trap (0.6B–7B):** Accuracy collapses and efficiency tanks; FP16 wins entirely.
- **Efficiency-Only Trap (14B):** Accuracy holds, but un-amortized casting causes speed and power penalties.
- **Conditional Dissolution (32B):** Quantization succeeds on a single GPU, but *only* for short contexts with larger batch sizes.
- **Memory-Only Victory (72B+):** VRAM drops by 60%, but triggers a speed/energy trap on multi-GPU setups. Used strictly out of hardware necessity.