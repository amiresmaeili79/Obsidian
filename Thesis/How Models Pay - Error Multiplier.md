---
tags: [thesis, quantization, multiplier, loops]
date: 2026-11-19
status: done, written up
---

# How does 4-bit hurt? Each model pays in its own way, and a multiplier describes the rest

> [!summary] TL;DR
> 4-bit **multiplies each turn's error rate** by a fixed, model-specific factor **k**, and k is similar across English, Slovak and Persian for four of five models (Granite 8B is the exception).
> Non-English loses more points mostly because its 16-bit baseline already errs more, and the same k **compounds over ten turns**.
> Models also **pay in different ways**: Granite 3B loops until the cap, Qwen 4B gives wrong answers, Qwen 9B thinks longer, the big models absorb it.
>
> Full tables: Hugging Face `analysis/2_how_4bit_fails_8k.md`. This note explains **why** we ran each step.

**Read time:** Sections 1–2 take 2 minutes. The rest is detail, and the folded boxes hide the extra.

Related: [[Main Results - 8k and 4k Grid]] (how much it costs), [[Attention vs MLP - Ablation Test]] (the fix), [[Multi-turn Quantization - Related Papers]].

---

## 1. Where it stands

**How each model pays.** *Gap* is the 4-bit minus 16-bit episode success, in points. *Extra loops* and *extra wrong answers* are the extra failures (in points of episodes) at 8k, split by what the failing turn did.

| Model | How it pays | Gap @4k | Gap @8k | Extra loops | Extra wrong answers |
|:---|:---|---:|---:|---:|---:|
| Qwen 3.5 4B | **Wrong answers** | −19.1 | −14.3 | +1.3 | +13.0 |
| Qwen 3.5 9B | **Thinks longer** | −11.3 | −2.1 | +0.8 | +1.3 |
| Qwen 3.5 27B | Absorbs | +0.7 | −0.7 | −0.1 | +0.8 |
| Granite 4.2 3B | **Loops** | −23.7 | −30.1 | +40.4 | −10.3 |
| Granite 4.2 8B | **Mixed** | −17.5 | −15.0 | +8.2 | +6.7 |
| Granite 4.2 30B | Absorbs | −1.1 | −2.0 | +0.6 | +1.4 |

**The multiplier k** is the 4-bit per-turn error rate divided by the 16-bit one. 1.00 means no extra mistakes.

| Model | English | Slovak | Persian |
|:---|---:|---:|---:|
| Qwen 4B | 1.69 | 1.57 | 1.62 |
| Granite 8B | 1.41 | 1.48 | 1.48 |
| Qwen 9B | 1.15 | 1.06 | 1.17 |
| Qwen 27B | 1.05 | 1.02 | 1.15 |
| Granite 30B | 1.18 | 1.08 | 1.10 |

Granite 3B (2.37, 1.71, 1.33) is dominated by loops, and Persian is at the floor, so one factor does not describe it.

> [!tip] Simple version of the multiplier
> k = 1.7 means 4-bit makes **1.7 times as many mistakes per turn** as 16-bit. It is the same factor in every language, so a language where 16-bit already makes more mistakes ends up with a bigger loss in points.

---

## 2. The question and the plan in plain words

**The question.** [[Main Results - 8k and 4k Grid]] showed how much 4-bit costs. Here we ask *how*: is it more wrong answers, longer thinking, or loops? And does the cost follow a simple rule across languages?

**The plan.**
1. Measure the **error rate per turn**, and the multiplier k = 4-bit rate ÷ 16-bit rate.
2. **Test the rule:** fit k on English only, then predict the Slovak and Persian losses.
3. Split failed turns into **loops** (thinking until the cap, always wrong) and **plain wrong answers**.

### Why we ran it this way

| Decision | Why |
|:---|:---|
| Use per-turn error, not episode success | An episode stops at its first wrong turn, so success mixes the per-turn error with the number of turns |
| Fit k on English only, then predict | `success ≈ (1 − per-turn error)^turns` is just an identity on the same data, so it is not a result. Predicting other languages from English **is** a test |
| Split failures into loops and wrong answers | The remedy differs. A loop is a termination problem, a wrong answer is a reasoning problem |
| Classify with fixed rules written before the data | To avoid fitting the labels to the results (see below) |

> [!tip] Simple version of "compounding"
> Qwen 4B in English makes a mistake on 2.1% of turns in 16-bit and 3.5% in 4-bit. Over ten turns, 0.979^10 ≈ 81% of episodes survive in 16-bit, and 0.965^10 ≈ 70% in 4-bit. That is a drop of about 12 points from a small per-turn change. A weaker language starts lower, so the same factor takes more points off it.

> [!note]- How the labels are decided (click to open)
> - **Absorbs:** the gap's interval includes zero at both 4k and 8k.
> - **Thinks longer:** a significant loss at 4k, and at 8k either the interval includes zero or the loss is at most a third of the 4k loss, and the token cost ratio is above 1.
> - Otherwise, if the 8k loss is significant, the extra failures are split into loops and wrong answers. **Loops** if loops are at least two-thirds of them, **Fails** (wrong answers) if wrong answers are at least two-thirds, else **Mixed**.
>
> Floor cells (16-bit success below 15%, here only Granite 3B Persian) are left out of the pooled rows.

---

## 3. Each model pays in its own way

The labels describe what the extra failures are made of.

| Model | In plain words |
|:---|:---|
| Qwen 27B, Granite 30B | **Absorb it.** No significant loss at either cap |
| Qwen 9B | **Thinks longer.** It loses at 4k, because it runs out of budget, and nearly recovers at 8k. Its tokens go up 1.14× |
| Qwen 4B | **Wrong answers.** Almost no loops (+1.3 points), but +13 points of plain mistakes |
| Granite 3B | **Loops.** +40 points of loop failures, using 1.87× the tokens on turns both get right |
| Granite 8B | **Mixed.** Both kinds of failure, about equally |

**Granite 3B in numbers.** 66% of its English 4-bit episodes end on a turn that ran to the 8,192 cap, against 16% in 16-bit. That is why more budget does not help: the gap grows with the cap.

| Granite 3B gap at cap | 2k | 3k | 4k | 6k | 8k |
|:---|---:|---:|---:|---:|---:|
| Gap (pp) | −4.6 | −14.9 | −23.7 | −27.1 | −30.1 |

---

## 4. The multiplier carries over between languages, for four models

The test: fit k on English, then predict the Slovak and Persian episode gaps from each language's 16-bit error rate.

| Model | Slovak: predicted / measured | Persian: predicted / measured |
|:---|---:|---:|
| Qwen 4B | −14.8 / −14.0 | −17.4 / −17.3 |
| Qwen 9B | −3.1 / −1.3 | −2.6 / −3.1 |
| Qwen 27B | −0.6 / −0.2 | −0.4 / −1.3 |
| Granite 30B | −2.8 / −1.1 | −3.9 / −2.7 |
| **Granite 8B** | **−10.5 / −15.3** | **−13.1 / −21.6** |

**It holds for every Qwen model and Granite 30B.** For Granite 8B the prediction is too optimistic by 5 to 8 points, and we have not isolated why.

**So sensitivity is a property of the model, not the language.** Non-English loses more points because its 16-bit baseline already makes more mistakes, and the same k compounds over the turns.

> [!note]- How this relates to Mekala et al. (click to open)
> Mekala et al. (EMNLP 2025) found 4-bit hurting non-English input more, up to "five times more than in English". They tested older non-thinking models (Llama-3.1, Qwen-2.5) on single-prompt long-context retrieval, in 26 languages. The 5× is a loose statement, and the paper gives no formula for it.
>
> Our setting is different: multi-turn, thinking models, per-turn error. Here the per-turn factor is similar across languages and the extra loss comes from compounding. The two results are not in conflict.

---

## 5. What holds only in Granite 8B, or did not hold

| Observation | What we found |
|:---|:---|
| **Damage grows with depth** in Granite 8B | Multiplier 0.93 in turns 1–3, 2.04 from turn 7. It shows in each language and rises with prompt length. Qwen 9B and 27B stay flat |
| **4-bit hurts turns that recall the history more** | Not supported: recall turns 1.49 vs local turns 1.42. The growth tracks how far into the conversation a turn is, not the turn type |
| **Granite 3B beats 8B with thinking off** | 44% vs 33% in English 16-bit. It is a formatting problem of 8B without thinking (21 episodes lost to format errors, 3B none), not a scoring change |

Depth growth is an 8B observation, so it is not on the slides. The test of its cause is in [[Attention vs MLP - Ablation Test]].

> [!note]- Survival bias, and the test of the cause (click to open)
> **Survival bias.** Late turns only include episodes that survived, which should make 4-bit look better late, not worse. So the growth is not an artefact.
>
> **The hypothesis.** Qwen kept its 24 DeltaNet (history-reading) layers in 16-bit and shows no growth, while Granite quantised all its attention and does. So the growth might belong to attention.
>
> **The result.** Not supported as stated. The growth stayed when only one half was 4-bit (×1.43 and ×1.32, against ×2.19 with both), so it does not belong to attention alone.

---

## 6. Next steps (not decided yet)

- **Explain the Granite 8B miss.** Why does the language prediction fail only there? One guess is the late-conversation growth, but it is not isolated.
- **The fixes.** [[Attention vs MLP - Ablation Test]] and [[DPO Pilot - Termination Test]] target the two kinds of damage (plain errors, and termination).

---

## 7. Caveats

- **`success ≈ (1 − per-turn error)^turns` is not a result.** It is an identity on the same data. The result is that k carries over between languages.
- **The prediction test is out of sample across languages only** (same models, seeds and episodes).
- **Depth growth is not general.** It appears clearly only in Granite 8B.
- **Granite 3B does not fit the single-multiplier picture,** and its Persian runs are at the floor.
- **The labels are conventions** fixed before looking at the data, not measured facts.
