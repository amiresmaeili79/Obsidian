---
tags: [thesis, dpo, granite-3b]
date: 2026-11-19
status: round 2 done, scaling plan proposed
---

# DPO pilot: did it fix 4-bit, or just teach the task?

> [!summary] TL;DR
> A small DPO adapter fixes the main way 4-bit breaks Granite 3B: **never finishing its thinking**.
> Adapter B beats every budget cut on 1,500 test turns (**89.7%** vs 83.5% and 87.3%) at a lower cost, and the effect carries over to math.
> One side effect is left: the **leak** (4.1% of turns).
>
> Full tables: Hugging Face `analysis/6_dpo_pilot_termination.md`. This note explains **why** we ran each step.

**Read time:** Sections 1–2 take 2 minutes. The rest is detail, and the folded boxes hide the extra.

Related: [[Lee 2024 - QDPO Conversational Abilities of Quantized LLMs]] (16-bit chosen, 4-bit rejected, for chat), [[Lotfi 2026 - Quantized Reasoning Models Overthink]] and [[Lian 2026 - Quantization Inflates Reasoning]] (4-bit thinks longer).

---

## 1. Where it stands (1,500 cart turns)

Every model answers the same turns. **Correct** is the share answered right. **Tokens** is the mean spent per turn. **Leaks** are answers where the model closes its thinking and then keeps reasoning.

| Model | Correct | Tokens / turn | Leaks |
|:---|---:|---:|---:|
| 16-bit | 92.0% | 1,645 | 0.1% |
| Plain 4-bit | 79.4% | 3,398 | 0.5% |
| Plain 4-bit, cut at 2,048 tokens | 83.5% | 1,653 | – |
| Plain 4-bit, cut at 4,096 tokens | 87.3% | 2,522 | – |
| Pilot adapter | 87.3% | 1,651 | 4.7% |
| Adapter A (matched cut) | 73.9% | 1,202 | 19.2% |
| **Adapter B (4k cut)** | **89.7%** | 1,485 | 4.1% |

**B is the best version.** It beats the 2,048 cut by 6.2 points and the 4,096 cut by 2.4 points, with 41% fewer tokens than the 4,096 cut.

**The leak is the last gap.** Scored by their valid lines only, B's leaked answers would reach 92.7%, about 16-bit's 92.0%. That is a ceiling, not a promise.

---

## 2. The question and the plan in plain words

**The pilot.** Plain 4-bit Granite 3B loses most of its episode success: 50.4% at 16-bit, 11.1% at 4-bit. A small adapter trained with DPO lifted that to 27.1%.

**The doubt.** We trained on the same task we test on. So did DPO repair what 4-bit broke, or just learn shopping carts?

**The plan.** Instead of asking "did accuracy go up?", ask **what 4-bit broke**, then test whether DPO fixes that better than the trivial fix, a budget cut.

> [!tip] Simple version
> We give all models the exact same questions. We see who gets stuck, how many tokens they spend, and who is right. Then we ask: would cutting the 4-bit model off early have done as well as training it?

### Steps we dropped, and why

| Idea | Decision | Why |
|:---|:---|:---|
| Train on a different dataset | Dropped | Results could differ for reasons unrelated to quantization |
| Test Slovak and Persian | Dropped | The model thinks in English even in those dialogues, so it would mostly retest English |
| Repeat on Granite 8B | Dropped for now | It shows the method generalises, not whether it is about quantization |
| Same training on 16-bit (control) | Dropped | 16-bit may be capped near 55%, so no gain would not prove anything |

### Steps we kept

| Test | Why |
|:---|:---|
| Same prompts for every model, plus a budget-cut baseline | A cut is the trivial way to stop long thinking. If DPO only taught "stop earlier", a cut would do as well |
| MATH-500, never trained on | If DPO were only cart calibration, nothing would change there |

---

## 3. What 4-bit broke

On the same 1,261 round-1 prompts, 4-bit loses mainly in **finishing**, not in reasoning.

| | 16-bit | Plain 4-bit |
|:---|---:|---:|
| Correct, all turns | 94.3% | 81.0% |
| Turns that never finish thinking | 1.7% | 14.1% |
| Correct on turns 4-bit finished | 95.9% | 94.3% |
| Median thinking tokens on those turns | 928 | 1,837 |

When 4-bit finishes, it is almost as accurate as 16-bit. It just thinks twice as long and gets stuck on one turn in seven. That is the target.

> [!note]- How the prompts were built (click to open)
> Each prompt is one turn with a **fixed, correct history** before it. Whatever a model answered earlier never enters a later prompt, so all models see identical questions.
>
> **Round 1:** the history is 16-bit's own correct answers, up to its first wrong turn, which gives 1,261 prompts from the 150 test episodes.
> **Round 2:** the history is written from the dataset's true cart state, which gives all 1,500 turns, including the late, hard ones.
>
> Cutting a generation at B tokens is the same as sampling with a B-token budget, so every budget comes from the same generations.

---

## 4. Round 1: DPO rescues stuck turns, but leaks

The 1,261 prompts split by what plain 4-bit does: **stuck** turns (178) never finish, **healthy** turns (1,083) finish on their own.

| Share correct | Stuck (178) | Healthy (1,083) | All |
|:---|---:|---:|---:|
| 16-bit | 84.3% | 95.9% | 94.3% |
| Plain 4-bit | 0% | 94.3% | 81.0% |
| 4-bit cut at 2,048 | 60.1% | 91.3% | 86.9% |
| 4-bit cut at 4,096 | 64.6% | 94.1% | 89.9% |
| 4-bit + DPO (pilot) | **80.9%** | 88.1% | 87.1% |

**On stuck turns, DPO clearly wins** (p < 0.001). A cut makes the model answer from a looping, garbled trace. DPO mostly avoids the loop.

**On healthy turns, DPO loses.** It writes the closing tag too early and keeps reasoning in the answer, which the scorer marks wrong. That is the **leak**: 75 of its 79 leaks are on these turns.

**Overall the two cancel.** Against the 2,048 cut (same tokens: 1,615 vs 1,598), DPO wins 37 stuck turns and loses 35 healthy ones: +2 out of 1,261, a tie (p = 0.95).

> [!tip] Why "better" and "a tie" are both true
> DPO is like a medicine that cures sick patients much better than the alternative, but gives healthy patients a side effect. Counted over everyone, the same number end up well. The medicine works, but the side effect hides it in the total.
>
> "Better than the cap" is what DPO **can do**. "A tie" is the score of **this version**, which has the leak.

> [!note]- Why compare with the 2k cut, not 8k? (click to open)
> Leaving the model to run to 8k is just plain 4-bit, and it does worse than any cap from 2k up. More room does not help a loop, and at 8k nothing forces an answer.
>
> The 2k cut is not the best cut. It is the **fair** one, because it costs the same tokens as DPO. The 4k cut scores higher but spends about 50% more tokens.
>
> | Cut at | 256 | 512 | 1,024 | 2,048 | 4,096 | 8,192 (no forced close) |
> |:---|---:|---:|---:|---:|---:|---:|
> | Correct | 72.0% | 76.9% | 81.4% | 86.9% | 89.9% | 81.0% |
> | Mean tokens | 295 | 530 | 962 | 1,615 | 2,415 | 3,186 |
>
> A forced close at exactly 8k was never tested. It is an easy extra run and the one baseline a skeptic could ask for.

---

## 5. Math: transfer, not a win over a cut

On MATH-500, which the adapter never saw, DPO takes 4-bit from 76.4% to 83.4%, level with 16-bit.

| MATH-500 | Correct | Never finish |
|:---|---:|---:|
| 16-bit | 82.2% | 80 |
| Plain 4-bit | 76.4% | 109 |
| 4-bit + DPO (pilot) | 83.4% | 70 |
| 4-bit cut at 4,096 | 84.4% | – |

**What it shows:** the change is in how the model ends its thinking, in general. It is not cart calibration. Nothing got worse on problems 4-bit already finished (97.2% vs 97.7%).

**What it does not show:** a win over a cut. The 4,096 cut does just as well (p = 0.54). The stuck math problems are hard, not loops: 16-bit solves only 24% of the 109 where 4-bit gets stuck, against 82% of all 500. With so little headroom, math can show transfer and no harm, but not the loop rescue. That case rests on the carts.

---

## 6. Round 2: removing the "close early" shortcut

**The idea.** In 232 of 266 pilot pairs, the rejected trace never closes its thinking, while every chosen one does. The easiest rule to learn is "close early", which would cause leaks. So the rejected side now **closes too**. We cut each stuck 4-bit trace, add the closing tag, let the 4-bit model write its own answer, and score it.

**Two designs, one adapter each.**

| | Adapter A: matched cut | Adapter B: 4k cut |
|:---|:---|:---|
| Cut the rejected trace | at the same length as its chosen trace | after 4,096 tokens, so the loop stays in |
| Chosen vs rejected differ by | reasoning quality only | the loop and the extra length |
| Pairs kept | 120 (rejected answer still wrong) | 265 (all) |
| Result | worse: 73.9% | **best: 89.7%** |

> [!tip] Simple version
> A says "same length, but this reasoning is worse". B says "this one went round in circles before answering". We did not know which contrast the model needed, so we tested both. The 4k cut was your idea of letting the model see its own loop.

### The side finding: 63%

Cut a stuck 4-bit training trace at 16-bit's thinking length and force an answer: **63% of the answers are right** (145 of 231). The 4-bit model often already has the answer. It just does not stop.

### Why A failed

A's rejected side was ordinary 4-bit reasoning, with no loop in it. Pushing that down taught the model to distrust its own normal reasoning. It now thinks less than 16-bit (median 752 vs 1,090 tokens) and leaks on 19.2% of turns.

**Lesson:** the rejected side must contain the bad behaviour itself, the loop.

### The test got bigger

The history is now written from the dataset's true cart state, so all **1,500 turns** are covered, not only those before 16-bit's first mistake. The gold replies match 16-bit's own correct answers exactly in 90.5% of turns. This is why the pilot, a tie in round 1, now beats the 2,048 cut (87.3% vs 83.5%, p = 0.001): the new turns are late, hard ones where 4-bit gets stuck most.

### The leak is not mainly about "never closes"

B's rejected traces all close, yet B still leaks on 4.1% of turns, almost as often as the pilot (4.7%). So that shortcut was not the main cause, and the leak needs another fix.

> [!note]- The numbers behind the pair choices (click to open)
> **Pairs.** A keeps 120 pairs, those whose rejected answer is still wrong (86 cut stuck traces plus 34 that closed wrong on their own). If the rejected answer were right, a pair would push down a good trace. The 120 split by episode into 103 train and 17 validation, across 85 episodes. B keeps all 265 (216 train, 49 validation, 134 episodes).
>
> **Imitation weight.** The loss has two parts: DPO ("prefer chosen over rejected") and an imitation term ("also make chosen likely", which is plain fine-tuning). It stays at **0.2**. Raising it to 1.0 might help the leak, but it would turn the adapter into mostly task fine-tuning, the very confound this test checks for. So every adapter differs from the pilot only in its pairs.
>
> **LoRA settings.** Rank 16, alpha 32, dropout 0.05, on all projections; lr 5e-5, 2 epochs. These are common small-adapter defaults and were **not tuned** (related work uses rank 32).
>
> **A first attempt that failed.** We first cut at the loop's start. The loop detector fired on ordinary repetition, so cuts landed at about 495 words while chosen thinking runs about 800. That would have brought back a length difference, so we dropped it.

---

## 7. Plan for scaling (proposed, not started)

We are proposing a fix, not comparing DPO variants, so we freeze **adapter B** and scale it to Granite 3B and 8B on the real dataset.

**Order:**
1. **Run the 8k forced-close baseline.** Only the stuck turns need new answers, so it is a short job.
2. **One bounded leak round.** Use B's own leaked answers as rejected examples. Then freeze whichever recipe is better on validation, and stop polishing.
3. **Train on a larger pool of training episodes.** Use 2–3 training seeds on 3B, so we can report variance.
4. **Granite 8B.** Expect a smaller gain: its damage is a mix of loops (+8 points) and wrong answers (+7), and B targets loops.
5. **Evaluate once.** Full ten-turn episodes (the thesis metric) plus the 1,500-turn test. Slovak and Persian are evaluation only.

**One rule:** decide the recipe on training-side data only. We picked B partly by looking at the 1,500-turn test, so that test is no longer clean for design choices.

**A thesis angle.** On 3B, keeping attention at 16-bit recovers 8.2% to 25.7%, and DPO lifts 11% to 27%. They fix different things, so they might add up. The cleanest comparison on 8B is plain 4-bit, DPO, attention kept, and both combined.

---

## 8. Caveats

- **The leak is not fixed.** B leaks on 4.1% of turns, and the 92.7% figure is a ceiling.
- **One model, English only.** One training run per adapter and one sample per prompt.
- **No full episodes for B yet.** Only the pilot was run on full episodes (27.1% vs 50.4% for 16-bit).
- **Shorter thinking does not explain the gain.** Per-prompt thinking length tracks 16-bit's no better with DPO than without it (Spearman 0.60 vs 0.58).
