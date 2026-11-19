---
tags: [thesis, quantization, ablation, attention, mlp]
date: 2026-11-19
status: done in 3 models, written up
---

# Attention vs MLP: which half of the model breaks under 4-bit?

> [!summary] TL;DR
> Keeping **only the attention weights at 16-bit** recovers about **half to two-thirds** of what 4-bit loses, in all three models tested.
> It is also the cheap fix: it adds only 0.4 to 2.5 GB, while keeping the MLP at 16-bit costs about four times as much memory.
> It works in every language, and helps most where 4-bit hurts most.
>
> Full tables: Hugging Face `analysis/3_attention_mlp_ablation.md`. This note explains **why** we ran each step.

**Read time:** Sections 1–2 take 2 minutes. The rest is detail, and the folded boxes hide the extra.

Related: [[Multi-turn Quantization - Related Papers]], [[DPO Pilot - Termination Test]] (the other fix, a different kind).

---

## 1. Where it stands

Episode success is the share of episodes where every turn is right. **Loss recovered** is how much of the 4-bit loss a version wins back. **Extra memory** is the cost over plain 4-bit.

| Model | 16-bit | Plain 4-bit | Attention kept 16 | MLP kept 16 |
|:---|---:|---:|---:|---:|
| Granite 8B (3 languages) | 63.5% | 48.5% | **58.3%** | 58.2% |
| Granite 3B (en, sk) | 38.3% | 8.2% | **25.7%** | 13.9% |
| Qwen 3.5 4B (3 languages) | 72.4% | 58.1% | **64.7%** | not run |

| Model | Loss recovered: attention | Loss recovered: MLP | Extra memory: attention | Extra memory: MLP | Attention better per GB |
|:---|---:|---:|---:|---:|---:|
| Granite 8B | **65%** | 65% | +2.5 GB | +9.3 GB | 3.8× |
| Granite 3B | **58%** | 19% | +0.9 GB | +3.7 GB | 12× |
| Qwen 3.5 4B | **47%** | not run | +0.43 GB | (+3.6 GB) | ≥ 3.9× (a bound) |

**Attention kept is the best-value fix.** In 8B it recovers the same as keeping the MLP for a quarter of the memory. In 3B it recovers three times as much.

> [!tip] Simple version of "loss recovered"
> A student normally scores 63.5. After a bad night's sleep (4-bit) they score 48.5, a loss of 15 points. One fix brings them back to 58.3, which is 9.8 of the 15 points, so **65% of the loss recovered**.
> 0% means the fix did nothing, and 100% means full recovery. It lets us compare fixes across models that start from different places.

---

## 2. The question and the plan in plain words

**The question.** A transformer layer has two halves. **Attention** looks back at the earlier conversation, and the **MLP** processes what attention found. Plain 4-bit shrinks both. Which half does the damage?

**The plan.** Build versions of the model where one half stays 16-bit and only the other is 4-bit, with the same AWQ recipe and one extra entry in its ignore list. Run them on exactly the same episodes, seeds and settings as the main 8k runs.

| Version | Stays 16-bit | Is 4-bit |
|:---|:---|:---|
| 16-bit | everything | nothing |
| All 4-bit | embeddings, output layer | attention + MLP |
| **Attention kept** | attention + embeddings | MLP |
| **MLP kept** | MLP + embeddings | attention |

> [!tip] Simple version
> Same students, same exam. We protect only one half of their brain from the bad night's sleep, and see which protection helps more.

### Why we ran it this way

| Decision | Why |
|:---|:---|
| Start with Granite 8B | That is where the damage grew during the conversation, and where the "same multiplier in every language" idea failed |
| Test the hypothesis it came from | Qwen keeps its 24 history-reading layers at 16-bit and shows no growth. Granite quantises all its attention and does. So we asked whether attention is the cause |
| Replicate on Granite 3B and Qwen 4B | One model is not a result. 3B fails by looping, Qwen 4B by plain wrong answers, so they cover other failure types |
| Leave out Granite 3B Persian | Its 16-bit model succeeds in only 7% of episodes, so there is almost nothing left to lose |
| Run only the attention version for Qwen | The question there was the recipe. A simple bound (Section 4) shows attention wins per GB whatever the MLP version would give |

---

## 3. Attention kept recovers most of the loss, in every model

On all three models, protecting attention wins back 47% to 65% of the loss. Granite 8B is the cleanest case.

| Granite 8B | Size | Success | Gap vs 16-bit |
|:---|---:|---:|---:|
| 16-bit | 17.6 GB | 63.5% | – |
| All 4-bit | 5.8 GB | 48.5% | −15.0 |
| **Attention kept** | 8.3 GB | 58.3% | −5.2 |
| MLP kept | 15.1 GB | 58.2% | −5.3 |

**The two fixes are equal in accuracy but not in price.** Attention is about 19% of the weights and the MLP about 72%, so keeping the MLP puts the model back at 86% of its 16-bit size, which removes most of the reason to quantize.

**It works in every language.** It recovers a similar share (41–68%) in each, so the points regained are largest where the loss is largest.

| Attention kept | English | Slovak | Persian |
|:---|---:|---:|---:|
| Granite 8B | +4.6 | +10.4 | +14.3 |
| Qwen 4B | +5.8 | +5.8 | +8.4 |

For a thesis about non-English dialogue, the recipe does not favour English. It helps most exactly where quantization does the most harm.

> [!note]- Why "per GB" and the Qwen bound (click to open)
> Per GB means the share of the loss recovered for each extra gigabyte kept in 16-bit. For Granite 8B, attention gives 65% over 2.5 GB, about 26% per GB. The MLP gives 65% over 9.3 GB, about 7% per GB. That is the 3.8×.
>
> For Qwen 4B we did not run the MLP version. But an MLP version can recover at most 100% of the loss. Spread over its 3.6 GB, that is at most 28% per GB, against 108% per GB for attention. So the conclusion holds whatever that run would show.
>
> Why the ratio is so high in Qwen: only 8 of its 32 layers have full attention, and the other 24 are already 16-bit. So "attention kept" adds very little memory.

---

## 4. The two halves fail in different ways, and not the same in every model

We split the extra damage into **runaway turns** (thinking until the cap, always wrong) and **plain wrong answers**.

| Model | What 4-bit mainly breaks | What the data shows |
|:---|:---|:---|
| Granite 3B | Runaway turns | Runaways go from 2.3% (16-bit) to 10.1% (4-bit). With only attention in 4-bit they are 8.3%, with only the MLP in 4-bit 4.7%. So here attention drives them |
| Qwen 4B | Plain wrong answers | Runaways are just 0.26%. Protecting attention lowers the plain error from 4.80% to 4.06% (16-bit: 3.03%) |
| Granite 8B | A mix | The 8B hint that "4-bit MLPs drive runaways" reverses in 3B |

**So no half owns thinking length.** Which half drives long thinking depends on the model. What is shared is that protecting attention helps in all of them.

**4-bit also shifts the whole thinking distribution.** In 8B the median rises 26%, and attention kept removes about half of the extra tokens (52%), while the MLP removes 75%. In 3B attention kept removes 69%, and the MLP only 9%.

---

## 5. What holds only in Granite 8B

Two things looked important at first and then did not replicate. We keep them as 8B observations, and took them off the slides.

| Observation | In 8B | In the other models |
|:---|:---|:---|
| **Super-additive damage:** both halves together lose more than the sum of each | −5.2 and −5.3 add to −10.5, but the observed loss is −15.0 | 3B is the opposite (−30.1 vs −37.1), but it is near the floor, so it cannot be estimated cleanly |
| **Damage grows with depth** | ×2.19 later in the conversation | 3B and Qwen 4B show no significant growth |

> [!note]- The depth-growth test in simple words (click to open)
> **What "growth" measures.** The error multiplier is how many times more errors 4-bit makes than 16-bit, per turn. Growth is the multiplier at turns 7 and later divided by turns 1–3. For 8B with everything in 4-bit it is 2.04 ÷ 0.93 = 2.19: the relative damage is more than twice as big deep into the conversation.
>
> **The logic.** Say attention caused it. Then the version with attention kept at 16-bit should show no growth (×1.0). It still shows ×1.43. Say the MLP caused it. Then the version with the MLP kept should show ×1.0, and it still shows ×1.32. So neither half alone explains it. Each half left in 4-bit produces some growth, and 1.43 × 1.32 ≈ 1.9, close to 2.19, so the halves multiply.
>
> **Caveat.** With runaway turns removed, the attention-kept growth is no longer significant (1.18). And later turns also have longer prompts, so depth and prompt length cannot be separated in this data.

> [!note]- The hypothesis this tested (click to open)
> Mekala et al. (EMNLP 2025) suggest, without testing it, that rounding errors in the positional encoding (RoPE) may build up over long inputs. They studied older non-thinking models on single-prompt long-context tasks, so it is a different setting.
>
> In our test the growth did not belong to attention alone, so the idea was not supported for multi-turn depth in Granite 8B.

---

## 6. Next steps (not decided yet)

- **Combine with DPO.** On 3B, attention kept lifts success from 8.2% to 25.7%, and DPO lifts 11% to 27%. They fix different things, so they might add up. The cleanest comparison on 8B is plain 4-bit, DPO, attention kept, and both together. See [[DPO Pilot - Termination Test]].
- **Share the models.** The five attention-kept and MLP-kept models are public on Hugging Face. A short model card on each repo would say which half stays 16-bit and how it was made.
- **No MLP version for Qwen 4B.** The bound shows the per-GB claim holds anyway, so running it would only add a second data point for the super-additivity question.

---

## 7. Caveats

- **One quantization method.** AWQ, 4-bit, group size 128, three models.
- **Qwen 4B has no MLP-kept version.** Its attention arm also protects the 24 DeltaNet layers, which the main Qwen recipe already did, so it differs from the main recipe only in the 8 full-attention layers.
- **Granite 3B excludes Persian,** and its success-based interaction is unreliable near the floor.
- **Percentile differences have no confidence intervals yet,** and depth and prompt length are confounded.
