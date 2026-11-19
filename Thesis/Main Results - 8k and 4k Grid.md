---
tags: [thesis, quantization, main-results]
date: 2026-11-19
status: done, written up
---

# What does 4-bit cost each model? The 8k results and the first 4k grid

> [!summary] TL;DR
> **Big models barely notice 4-bit.** Qwen 27B loses 0.7 points of episode success and Granite 30B 2.0.
> **Small and mid-size models still lose a lot,** 14 to 22 points, even with a bigger thinking budget.
> The first 4k grid **overstated** the loss for Qwen 9B (−11.3 at 4k, −2.1 at 8k), because 4-bit thinks longer and ran out of budget.
> The 8-bit KV cache is nearly free (−0.8 points over 64 comparisons).
>
> Full tables: Hugging Face `analysis/1_main_results_8k.md` and `analysis/4_full_grid_4k_paired_tests.md`. This note explains **why** we ran each step.

**Read time:** Sections 1–2 take 2 minutes. The rest is detail, and the folded boxes hide the extra.

Related: [[How Models Pay - Error Multiplier]] (why the loss happens), [[Attention vs MLP - Ablation Test]] (the fix), [[Multi-turn Quantization - Related Papers]].

---

## 1. Where it stands (8k budget, 3 seeds)

Episode success is the share of episodes where all ~10 turns are right. **Gap** is 4-bit minus 16-bit, in percentage points. **Gap @4k** simulates the old 4,096-token cap on the same runs.

| Model | 16-bit | 4-bit | Gap @8k | Gap @4k (simulated) |
|:---|---:|---:|---:|---:|
| Qwen 3.5 4B | 72.4% | 58.1% | **−14.3** | −19.1 |
| Qwen 3.5 9B | 80.8% | 78.7% | −2.1 | −11.3 |
| Qwen 3.5 27B | 89.4% | 88.7% | −0.7 | +0.7 |
| Granite 4.2 3B | 28.0% | 6.0% | **−22.0** | −16.4 |
| Granite 4.2 8B | 63.5% | 48.5% | **−15.0** | −17.5 |
| Granite 4.2 30B | 81.1% | 79.1% | −2.0 | −1.1 |

**Two groups.** Qwen 27B and Granite 30B absorb 4-bit. Qwen 4B, Granite 3B and Granite 8B lose 14 to 22 points. Qwen 9B sits in between: with a 4k cap it looked bad, but at 8k it nearly recovers.

**Translated languages usually lose more.** Granite 8B loses 8.0 points in English, 15.3 in Slovak and 21.6 in Persian. The large gaps repeat in every seed, so they are not noise.

> [!tip] Simple version
> The first experiment gave every model a short time limit to think. The 4-bit models think longer, so they got cut off more, and the loss looked bigger than it was. We then doubled the time limit. For some models (Qwen 9B) the loss mostly went away, and for others (Granite 3B) it did not.

---

## 2. The question and the plan in plain words

**The question.** How much episode success does 4-bit (AWQ W4A16) cost each model, and does it depend on language, thinking, and the KV cache?

**The plan, in two steps.**
1. **First grid (4k cap).** Run every combination of settings once. This gave a broad map, but at a 4,096-token cap.
2. **Rerun the core at 8k.** Thinking on, math off, 16-bit vs 4-bit weights, an 8,192-token cap, 3 seeds. This is the main result.

### Why we ran it this way

| Decision | Why |
|:---|:---|
| Rerun at 8k with 3 seeds | At 4k, 4-bit models often ran out of budget, so part of the loss was just the cap. One seed also leaves a single cell moving up to about 7 points |
| Simulate a 4,096 cap on the 8k runs | To see how much the old cap mattered. A success counts only if every turn used fewer than 4,096 tokens |
| Check the simulation against the real 4k grid | It agrees to within a few points, about the size of run-to-run noise, so the simulation is fair |
| Pool over languages, except Granite 3B Persian | Its 16-bit model succeeds in only 7.3% of episodes, so there is almost nothing left to lose |
| Keep thinking off, math on and the 8-bit KV cache only in the 4k grid | They were not rerun at 8k, so they exist only there, with one run each |

---

## 3. The 8k results

**Big models absorb 4-bit, small ones do not.** Even with the larger budget, Qwen 4B loses 14.3 points, Granite 8B 15.0 and Granite 3B 22.0 (30.1 without Persian).

**4-bit pushes mid-size models into the cap more often.** For Granite 8B in Persian, the share of episodes with a turn at 8,192 tokens rises from 4.9% to 20.4%.

**Qwen 9B was mostly a budget problem.** The gap shrinks from −11.3 at 4k to −2.1 at 8k. Granite 3B is the opposite: its gap grows with the cap (−23.7 at 4k, −30.1 at 8k), because more room means more looping.

| Gap by model | at 4k | at 8k | Reading |
|:---|---:|---:|:---|
| Qwen 4B | −19.1 | −14.3 | still large: plain wrong answers |
| Qwen 9B | −11.3 | −2.1 | budget effect |
| Granite 3B (English, Slovak) | −23.7 | −30.1 | worse with more budget: loops |
| Granite 8B | −17.5 | −15.0 | still large |

> [!note]- Per language, Granite 8B (click to open)
> Gap at 8k, with 95% intervals: English −8.0 [−12.2, −4.0], Slovak −15.3 [−21.6, −9.3], Persian −21.6 [−26.9, −16.0]. Per seed, the gaps are always in the same direction: Persian −21.3, −21.3 and −22.0.
>
> Why non-English loses more points is in [[How Models Pay - Error Multiplier]].

---

## 4. The first 4k grid: what it found

This was the first full experiment: every combination run once (seed 0), a 4,096-token cap, the same 150 episodes in each language. 128 cells, covering Qwen 3.5 4B (English only), 9B, 27B and Granite 4.2 8B, 30B, with math off and on, thinking off and on, and 16-bit vs 4-bit weights crossed with a 16-bit vs 8-bit KV cache.

Average effect over every matching pair of cells:

| Effect | Cell pairs | Mean change (pp) |
|:---|---:|---:|
| 8-bit KV cache (all cells) | 64 | **−0.8** |
| 4-bit weights, thinking on, small/mid models | 14 | **−14.0** |
| 4-bit weights, thinking off, small/mid models | 10 | −3.4 |
| 4-bit weights, large models, math off | 24 | −0.1 |
| 4-bit weights, large models, math on, thinking on | 12 | −2.6 |
| Thinking off → on, large models, math off | 24 | −5.2 |
| Thinking off → on, math on | 8 | **+56.9** |

**The 8-bit KV cache is nearly free.** Over all 64 comparisons it costs 0.8 points, and none is significant. Every real gap comes from the weights.

**Thinking matters in both directions.** It is essential for arithmetic (+57 points), but on plain state tracking it costs the large models about 5 points. With thinking off, 4-bit costs the small models only about 3 points, against 14 with thinking on. The large models lose nothing on plain state tracking, but about 3 points on arithmetic.

> [!note]- How the grid compared conditions (click to open)
> Every condition sees the same episodes, so each comparison is paired by episode and tested with the exact McNemar test, with a Holm correction for multiple tests. "Significant" means a Holm-adjusted p below 0.05. General claims rest on pooled effects across matching cells.
>
> **A data fix.** 18 of the 150 episodes had a flawed turn (a product not in the shop, or a turn that changes nothing). They were fixed and rerun, and the grid uses the fixed episodes. A check on the 132 clean episodes only moved the pooled effects by less than one point.

---

## 5. Next steps (not decided yet)

- **Understand why.** The next note, [[How Models Pay - Error Multiplier]], splits the loss into loops and wrong answers and asks whether it works like a multiplier.
- **Fix it.** [[Attention vs MLP - Ablation Test]] and [[DPO Pilot - Termination Test]] are the two fixes.
- **Other settings at 8k.** Thinking off, math on and the 8-bit KV cache were not rerun at 8k, so they stay one-seed, 4k results.

---

## 6. Caveats

- **The 4k thinking-on gaps are not final.** At a 4,096 cap, part of the loss is running out of budget. The 8k reruns replace them.
- **The 4k grid has one seed.** A single cell can move by up to about 7 points between runs.
- **The language × 4-bit interaction is not significant** in the 4k grid. Persian and Slovak lose more points, but with 150 episodes and one seed it is within noise.
- **Granite 3B Persian means little.** Its 16-bit model succeeds in only 7.3% of episodes.
