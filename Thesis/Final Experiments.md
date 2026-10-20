# Final Experiments — To-Do and Ideas

Date: 2026-10-17. Repo: `Dialogue-State-Tracking`, branch `feat/persian-language`. Related: [[Key Findings]], [[The Quantization Trap]].

---

## 1. Current Status

### 1.1 Facts From the Data

- **Generation limit:** 4096 generated tokens. The 512-token budget is a hint only.
- **"Exhausted":** the model does not close `</think>` in 4096 tokens. The turn has no answer.
- **Main cause of failure:** exhaustion causes 55% of the failed think-on episodes.
- **Significant W4 penalty:** only in en 4B, en 9B and fa 9B, with thinking on. Test: paired McNemar, Holm correction.
- **27B:** W4 has no penalty. Pooled over 8 cells, W4 is slightly better (p = .002, one seed only).
- **KV8:** no significant effect in 23 of 23 comparisons.
- **Thinking:** it decreases Math_Off success at 27B (−6 to −13 pp). It is necessary for Math_On (+53 pp).
- **Cost:** W4 9B uses 12.8k tokens per successful episode. W16 9B uses 7.4k (+73%).
- **Triggers:** ambiguous operations cause exhaustion. `cancel_last_add`: 5.6% per turn. `remove_item`: 0.2% per turn.
- **Thinking language:** the model thinks in English for all input languages.

### 1.2 W4 Increases Exhaustion, Not Normal Errors

Think on, W16_KV16 → W4_KV16. Rates are per turn.

| Cell | Success | Exhaustion | Other errors | Success without exhaustion (estimate) |
|:---|:---|:---|:---|:---|
| en 4B | 66.7 → 49.3% | 2.1 → 4.0% | 1.8 → 2.4% | 83.0 vs 77.9% |
| en 9B | 79.3 → 62.7% | **0.5 → 2.3%** | 1.8 → 2.1% | 83.8 vs 81.3% |
| fa 9B | 60.0 → 36.7% | **2.8 → 7.6%** | 2.1 → 1.3% | 80.7 vs 88.0% |
| en 27B | 84.7 → 82.7% | 0.8 → 0.8% | 0.9 → 1.1% | 91.7 vs 89.6% |

Note: the last column is optimistic. It assumes that an exhausted turn has the average error rate.

### 1.3 The Model Finds the Answer, Then Continues

For the 1245 exhausted turns, a script graded the answer drafts inside the thinking text:

- 86% of the traces contain an answer draft.
- 48% of the traces contain a correct draft.
- 27% of the traces end with a correct draft.
- The first correct draft is at approximately token 2400 (median).

**Example (en, 9B, episode 52, turn 7):** the user asks for the full order.

- **W16:** thinks for 354 tokens. The answer is correct.
- **W4:** writes the correct cart at approximately token 2200. It writes "*Calculation holds.*" Then it writes "*One final check: The prompt…*" and stops at 4096 tokens without an answer.

**Conclusion:** do not only decrease the budget. 44% of the successful turns use more than 1024 tokens.

---

## 2. To-Do

### 2.1 Fix the Farsi Dataset — HIGH PRIORITY

**Problem:** the dataset generator has two steps.

1. The code selects the operation and the correct answer. Example: "undo the last add" → `NEXON-N80-ULTRA: 0`.
2. An LLM writes the customer sentence for that operation.

The Farsi prompt tells the LLM: "use the exact product names, e.g. BLAKE-X100-LITE" (`paraphraser_fa.py:103`). The LLM copies this example name into the sentences. The Slovak prompt had the same example until commit `2488b28`.

**Result:** approximately 11% of the turns name the wrong product.

> Customer (sk, episode 4, turn 2): "Forget that last addition, **BLAKE-X100-LITE**."
> Answer key: `NEXON-N80-ULTRA: 0`
> Model: `BLAKE-X100-LITE: 0` → scored **wrong**. The episode stops.

The model does what the customer says, but the sentence and the answer key do not agree.

| | fa | sk (native) | en, sk_translated |
|:---|:---:|:---:|:---:|
| Episodes with ≥ 1 bad turn | 94 / 150 | 107 / 150 | 3 / 150 |
| Bad turns, part of all failures | 52% | 63% | — |

Bad turns fail in 10–27% of cases. Clean turns fail in 0–5% of cases.

- **Valid:** W16 vs W4 in the same language. All quantization settings get the same bad turns.
- **Not valid:** absolute fa and sk numbers, and all comparisons between languages.

**Procedure:**

1. ✅ Make `translator_fa.py`. Use `translator_sk.py` as the template.
2. ✅ Add `"fa"` to `LANGUAGE_FUNCS` in `apply_translation.py`.
3. ✅ Add a validator check (`identifier_problems` in `apply_translation.py`). Product names, discount codes and shipping tiers must stay the same as in the English source. Test: 0 false alarms on `sk_translated`; it finds 10.8% of fa turns and 11.9% of sk turns.
4. ✅ (2026-10-19) Translate the English episodes to Farsi on the cluster. Result: 1500/1500 turns valid. A recheck pass corrected 179 turns ("shipping" as cost, reversed redistribute direction, English filler words). See `TRANSLATION_INFO.txt`.
   Method: `sbatch translate_local.sh --language fa` (vLLM, `gemma-4-31b-it`, same prompt as before). The Gemini API failed in 50–90% of calls, so do not use it. Output: `annotated_episodes_fa_translated`, with `TRANSLATION_INFO.txt` for provenance.
5. Run all fa cells again: `python submit.py -c configs/qwen_all_fa_translated.json` (20 runs, output `results/fa_translated/raw`).
6. ✅ Mark the native fa and sk results as contaminated (warnings in the result READMEs). Use `sk_translated` for Slovak.

Note: translation is a valid method (MGSM and XNLI use it). It gives the same conversations and answer keys in all languages. Limitation: "translationese" — the text can be less natural than native text. Let a native speaker examine approximately 30 sentences.

### 2.2 Statistics

- Use paired McNemar tests with Holm correction. All conditions use the same 150 episodes.

### 2.3 Report

- Update the error legend in `generate_dst_report.py`. The legend says "JSON". The output format is `key: value`.
- In the thesis, write that the limit is 4096 generated tokens.

### 2.4 Keep the Analysis Scripts

- The scripts `build.py` and `rescue.py` are in a temporary folder:
  `/private/tmp/claude-502/-Users-amir-Projects-Dialogue-State-Tracking/60aa6733-8ae5-477b-a4bc-16d9f3021f0c/scratchpad/analysis/`
- Move them to `analytics/` before the system deletes them.

### 2.5 Examine [[Key Findings]]

- Two claims do not agree with the paired tests: "W4_KV8 beats W4_KV16 (regularizer)" and "KV8 amplifies thinking instability". The KV8 differences are not significant.

### 2.6 Optional: Compare With Lotfi et al.

- Lotfi et al. ([2606.00206](https://arxiv.org/abs/2606.00206)) define an "overthinking error" as a wrong final answer **or** reasoning that never terminates. They report that up to 52% of quantized failures have a correct answer earlier in the thinking.
- Our 48% (section 1.3) counts only exhausted turns. The two numbers are not comparable.
- To compare: run the grader on the closed-but-wrong turns too. Report one number per cell, split into exhausted and closed-but-wrong.
- Not needed for the swap experiment. Do it only if the thesis compares the two numbers directly. Priority: low.

---

## 3. Ideas for the Final Experiment

### 3.0 Common Method: Prefill

Ideas 1 and 2 use this method. Do not open a new chat, and do not ask the model about the thinking.

1. Build the same conversation again. The saved episode files have all the data: system prompt, catalog, earlier user messages and earlier answers (`instruction`, `response_tokens`).
2. Put the thinking text at the start of the model reply.
3. The model "thinks" that it wrote the text. It continues from that point.

```
<|im_start|>system     You are an order management assistant ...     ← same as the run
<|im_start|>user       Here are the available products ...           ← same as the run
<|im_start|>assistant  Got it
<|im_start|>user       (turns 1–6 + the model answers)                ← from the episode file
<|im_start|>user       Um, could I maybe just see the full order?     ← turn 7
<|im_start|>assistant
<think>
  [ THINKING TEXT ]                                                    ← prefill
</think>
                                                                       ← model writes only the answer
```

Note: `init_episode_messages()` in `evaluation.py` builds the first part.

---

### 3.1 Idea 1 — Swap the Thinker and the Answerer

**Goal:** find where W4 causes the damage. Is it in the reasoning, or in the decision to stop?

**Decision (2026-10-19):** use 9B and 27B, all three languages (en, sk_translated, fa_translated). 9B has the W4 penalty. 27B is the control: it has no W4 penalty, so the swap must show almost no difference. If 27B shows a large difference, the method has a problem. 4B is not used (English only, weak also at W16).

**Available data** (Math_Off, think on, KV16; closed = trace has `</think>`):

| Cell | Success W16 → W4 | Closed traces W16 / W4 | Exhausted traces W16 / W4 |
|:---|:---|:---|:---|
| en 9B | 79.3 → 62.7% | 1368 / 1250 | 7 / 30 |
| sk 9B | 69.3 → 61.3% | 1274 / 1205 | 19 / 34 |
| fa 9B | 65.3 → 50.7% | 1249 / 1143 | 25 / 57 |
| en 27B | 84.7 → 82.7% | 1401 / 1388 | 11 / 11 |
| sk 27B | 78.7 → 82.0% | 1353 / 1369 | 14 / 12 |
| fa 27B | 80.7 → 81.3% | 1365 / 1368 | 12 / 13 |

- Cells E/F are small (9B: 121 W4 exhausted traces in total). Pool the languages for the main result. Give the per-language values as secondary results.
- An episode stops at the first failure. Thus W16 and W4 complete different turns. Compare A and C only on turns that both runs completed.

**Procedure:**

1. Use all think-on turns of 9B and 27B, en/sk/fa (W16_KV16 and W4_KV16).
2. Build the prompt with the prefill method.
3. Close each trace with `</think>`. If the trace is exhausted, add one fixed sentence first: "*I have to give the answer now.*"
4. Load W16. Let it answer from all traces (its own traces and the W4 traces).
5. Load W4. Do step 4 again.
6. Score each answer with `_parse_output()`.

```python
messages = rebuild_history(episode, turn)
prompt = tok.apply_chat_template(messages, tokenize=False,
                                 add_generation_prompt=True, enable_thinking=True)  # ends with "<think>\n"
prompt += thinking_text + "\n</think>\n\n"
answer = llm.generate(prompt, SamplingParams(temperature=0, max_tokens=200))
correct = _parse_output(..., answer, turn_data=ground_truth_turn, ...)["correct"]
```

**Measure:** answer accuracy per cell.

| | Answer by W16 | Answer by W4 |
|:---|:---:|:---:|
| **W16 thinking** | A | B |
| **W4 thinking** | C | D |
| **W4 exhausted thinking** | E | F |

**Result:**

- **C ≈ A:** the W4 reasoning is good. The damage is in the decision to stop.
- **C < A:** the W4 reasoning is worse.
- **B < A:** W4 is worse at changing good reasoning into an answer.
- **E, F:** the number of exhausted turns that a forced answer can repair.

**Cost:** low. The model writes only answers of approximately 50 tokens.

---

### 3.2 Idea 2 — Measure the Stop Probability

**Goal:** find why W4 does not stop.

**Background:** before each token, the model gives a probability to all possible next tokens. You can read these values without text generation.

**Procedure:**

1. Use a W16 trace. Cut it at the point where W16 wrote `</think>`. Example: "*…Let me output all relevant lines for the full order view.*" + newline.
2. Build the prompt with the prefill method. Do **not** close the trace.
3. Ask W16 for the next-token probabilities. Ask W4 for the same text.
4. Record P(`</think>`) and the top alternatives ("Wait", "Let me", "One final check").
5. Do steps 1–4 again at other points:
   - Immediately after a correct draft. Example: after "*Calculation holds.*" in the W4 trace.
   - At each paragraph break. This gives a curve along the trace.

```python
out = llm.generate(prompt_with_partial_trace, SamplingParams(max_tokens=1, logprobs=20))
top = out[0].outputs[0].logprobs[0]   # 20 most probable next tokens
# p_stop = probability of "</think>" (if not in top 20: "< smallest value")
```

**Measure:** P(stop) for W16 and W4 on the same text, and the preferred alternative token.

**Result:** a statement of this type: "At the W16 stop point, W16 gives `</think>` 0.9. W4 gives 0.3 and prefers 'Wait'." (These values are an example, not data.) A plot of P(stop) along the trace is a good thesis figure.

**Cost:** very low. No text generation.

Note: make sure that `</think>` is one token in the Qwen3.5 tokenizer.

---

### 3.3 Idea 3 — Calibrate AWQ With Reasoning Text

**Background:** AWQ uses approximately 256 example texts to find the important weights. It keeps these weights precise. The current script uses `ultrachat_200k` (`quantize_qwen.py:55`). These texts contain no thinking. Thus the weights that control "stop thinking" are not in the calibration.

**Procedure:**

1. Make approximately 200 new episodes with a different seed.
   CAUTION: do not use the 150 test episodes for calibration.
2. Run W16 9B with thinking on these episodes (`run_experiment.py`, no change).
3. Make 256 calibration texts. Each text is one conversation. The last message is one assistant reply with `<think>…</think>` and the answer.
   CAUTION: the Qwen template removes the thinking from earlier assistant turns. Thus put the thinking in the last message. Print one text and make sure that it contains the thinking.
4. Change `quantize_qwen.py` so that it can load a local file. The format is the same `messages` list as UltraChat.
5. Quantize 9B (and 4B). Name: `Qwen3.5-9B-AWQ-reasoning`.
6. Run the same 150 test episodes: think on and off, KV16 only (2 runs per model).
7. Run the idea 2 probe on the new model.

**Measure:**

| | W16 | W4 (UltraChat) | W4 (reasoning) |
|:---|:---:|:---:|:---:|
| Success rate | 79.3% | 62.7% | ? |
| Exhaustion (episode) | 4.7% | 20.0% | ? |
| Median thinking tokens | 456 | 732 | ? |
| P(stop) at W16 stop point | ? | ? | ? |

**Control (recommended):** also make one W4 model calibrated on DST conversations **without** thinking. This shows if the reasoning text helps, or only text from the same domain.

**Result:**

- **Success:** a fix at the source, and a rule: "quantize reasoning models with reasoning data".
- **No success:** a negative result. Ideas 1 and 2 still explain the cause.

---

### 3.4 Idea 4 — Language Causes More Thinking Too

**Goal:** show that a second stress (language) causes the same failure as quantization.

**Data now (en vs sk_translated, same conversations, same turns, 9B, think on):**

| | English | Slovak (translated) |
|:---|:---:|:---:|
| Median thinking tokens (W16) | 462 | 791 (1.6×) |
| Exhaustion per turn (W16) | 0.51% | 1.47% (approx. 3×) |
| Traces that quote or translate the user text | 34% | 80% |

- The model translates the request in its English thinking. This uses part of the budget.
- At 27B the effect is smaller (1.2×).
- The two effects do not add: W4 in Slovak is not worse than W4 in English (669 vs 729 tokens). This is a research question.

**Procedure:**

1. Use the same 150 conversations in English, Slovak (translated) and Farsi (translated, from to-do 2.1).
2. Run the fa cells: 9B and 27B, W16 and W4, think on and off (approximately 8 runs). The en and sk runs are complete.
3. Compare each turn with the English turn (offline):
   - thinking length,
   - exhaustion rate,
   - the use of the extra tokens: translation, cart tracking, format check, second-guessing. Use the LLM labeler in `analytics/thinking_analysis/`.
4. Make a language × precision table. Do the two effects add?

**Optional tests:**

- English system prompt + Slovak customer. This shows if the cause is the instructions or the customer text.
- Give the English translation together with the Slovak message. If the extra thinking stops, the cause is translation.

**Result:**

- A wider thesis claim: the same failure (exhaustion from overthinking) occurs with two different stresses.
- A practical message: small quantized models for non-English users use more tokens and fail more.
- The Farsi data becomes clean.

---

### 3.5 Idea 5 — Push the Model to Stop

**Goal:** a fix at generation time, without a new quantization.

**Procedure:**

1. Add a logits processor in vLLM. It adds a bias b(n) to the `</think>` logit. The bias increases with the number of thinking tokens n.
2. Tune b(n) on W4 so that the W4 thinking-length distribution is similar to the W16 distribution. Use the new episodes from idea 3, not the test episodes.
3. Run the 150 test episodes with W4 + bias.
4. Compare with the baseline (below).

**Baseline — hard cut-off (budget forcing):** when the trace reaches B tokens without `</think>`, add a closing sentence and `</think>`. Then generate the answer. Use B = 1024, 2048, 4096.

**Measure:** success rate, exhaustion rate, tokens per successful episode.

**Result:** a targeted fix (bias) against a blunt fix (cut-off). Idea 2 tells you if the bias is the correct fix.

---

## 4. Order of Work

1. **To-do 2.1** (Farsi fix). Start now. The runs can continue in the background.
2. **Ideas 1 and 2.** They use the same code. They use the saved results. The GPU cost is low.
3. **Idea 3 or idea 5.** Select with the result of idea 2.
4. **Idea 4.** It needs the fixed Farsi data.
5. **Optional:** 3 seeds on the key cells. Thinking uses temperature 1.0, so each result now is one sample.

## 5. Thesis Storyline

**"Quantization breaks the stopping, not the thinking."**

| Step | Idea | Question |
|:---|:---|:---|
| Locate | 1 | Where is the damage? |
| Explain | 2 | Why does W4 not stop? |
| Fix | 3 or 5 | Can we repair it? |
| Generalize | 4 | Does language cause the same failure? |

**Literature check:** "Quantization Hurts Reasoning?" (Liu et al., 2025) reports longer outputs from quantized reasoning models. Lotfi et al. (2026) is very close to this storyline (see 6.1). Make sure that the contribution is clear: the mechanism, the multi-turn task, and the fix.

---

## 6. Related Work

Date of search: 2026-10-19. Most papers are arXiv preprints. For most papers, only the abstract was read. Before you cite a number, examine the paper.

### 6.1 Most Important: Lotfi et al. (2026)

**"Quantized Reasoning Models Think They Need to Think Longer, but They Do Not"** — Meta/FAIR, May 2026. [arXiv 2606.00206](https://arxiv.org/abs/2606.00206)

- Models: R1-Distill Qwen 1.5B–14B, R1-Distill Llama 8B, QwQ-32B. Methods: GPTQ and AWQ (3-bit, 4-bit), FlatQuant.
- In up to 52% of quantized failures, the model finds the correct answer during thinking, but does not give it as the final answer.
- Token-level KL between BF16 and quantized: the largest differences are at "Wait", "But", "Alternatively". The math tokens have small differences.
- Fix: a fixed logit penalty on 50 "overthinking" words. CoT length decreases by 12–23%. Accuracy stays the same or increases.
- The largest effect is at 3-bit. They write that 4-bit is "typically less disruptive".

**They did not do these items. These items are our contribution:**

- P(`</think>`) analysis (idea 2).
- Trace swap between W16 and W4 (idea 1).
- Multi-turn task, multilingual data (idea 4).
- A significant penalty at **4-bit** (9B, 4096-token limit).

You must cite this paper. Use their word penalty as a second baseline in idea 5.

### 6.2 General Background

| Paper | Link | Relation |
|:---|:---|:---|
| Liu et al. 2025, "Quantization Hurts Reasoning?" (COLM 2025) | [2504.04823](https://arxiv.org/abs/2504.04823) | Longer outputs at low bits (W3, W4A4KV4). |
| "Quantization Inflates Reasoning" 2026 | [2606.25519](https://arxiv.org/abs/2606.25519) | Quantized models keep accuracy but use more CoT tokens. |
| "Stop When Reasoning Converges" 2026 | [2605.17672](https://arxiv.org/abs/2605.17672) | 41–52% of reasoning tokens come after the model has its final answer. |

### 6.3 Per Idea

**Idea 1 — Swap the thinker and the answerer. Status: unique for quantization.**

| Paper | Link | Relation |
|:---|:---|:---|
| Lanham et al. 2023, "Measuring Faithfulness in CoT" | [2307.13702](https://arxiv.org/abs/2307.13702) | Cut the CoT, force an early answer. Same method. |
| "Reasoning that Travels" 2026 | [2605.28913](https://arxiv.org/abs/2605.28913) | Give the trace of one model to a different model. Has a "force answer" mode. |

No paper found that crosses W16 and W4 traces.

**Idea 2 — Measure the stop probability. Status: supported; unique for quantization.**

| Paper | Link | Relation |
|:---|:---|:---|
| ThinkBrake 2025 | [2510.00546](https://arxiv.org/abs/2510.00546) | Log-probability margin between the top token and `</think>` as a stop signal. |
| "The Evolution of Thought" 2025 | [2508.17627](https://arxiv.org/abs/2508.17627) | `</think>` gets rank 1 only some steps after the answer is found. |
| "Do Thinking Tokens Help or Trap?" 2025 | [2506.23840](https://arxiv.org/abs/2506.23840) | The model gives "wait" approx. 0.88 probability at these points. |
| DEER 2025 (ICLR 2026) | [2504.15895](https://arxiv.org/abs/2504.15895) | Stop when the trial answer has high confidence. |
| "Reasoning Models Know When They're Right" 2025 | [2504.05419](https://arxiv.org/abs/2504.05419) | Probe on hidden states; stop early, 24% fewer tokens. |

No paper found that compares P(`</think>`) for W16 and W4 on the same text.

**Idea 3 — Calibrate AWQ with reasoning text. Status: already done. Use as a replication.**

| Paper | Link | Relation |
|:---|:---|:---|
| Liu et al. 2025 | [2504.04823](https://arxiv.org/abs/2504.04823) | GPTQ with self-generated reasoning traces: +9.8% compared to WikiText2. Other methods are less sensitive. |
| "Attend to Your Own Thoughts" 2026 | [2608.01078](https://arxiv.org/abs/2608.01078) | Self-generated CoT is the best calibration (MATH-500 58.4 vs 24.2 without traces). |
| LAQuant 2026 | [2605.08755](https://arxiv.org/abs/2605.08755) | Calibration with reasoning traces + lookahead loss. |
| "Think Before You Prune" 2025 | [2511.18864](https://arxiv.org/abs/2511.18864) | Same result for pruning. |
| Williams & Aletras 2024 (ACL) | [2311.09755](https://arxiv.org/abs/2311.09755) | General: calibration data has a large effect. |

New parts: AWQ 4-bit on a multi-turn task, the control without thinking, the effect on P(stop).

**Idea 4 — Language causes more thinking. Status: partly supported; the combination is unique.**

| Paper | Link | Relation |
|:---|:---|:---|
| Yong et al. 2025, "Crosslingual Reasoning through Test-Time Scaling" | [2505.05408](https://arxiv.org/abs/2505.05408) | Same "quote the user text, think in English" behavior. |
| Marchisio et al. 2024 (EMNLP Findings) | [2407.03211](https://arxiv.org/abs/2407.03211) | Quantization causes the most damage to non-Latin scripts and to math. Supports fa × W4. |
| Etxaniz et al. 2024 (NAACL), "self-translate" | [2308.01223](https://arxiv.org/abs/2308.01223) | Supports the optional test with the English translation. |
| PolyMath 2025 | [2504.18428](https://arxiv.org/abs/2504.18428) | Thinking length changes ≤ 1.45× across languages. Our values are larger (sk 1.6×, fa 2.5×). Discuss. |
| "Hidden Language Consistency" 2026 | [2608.08447](https://arxiv.org/abs/2608.08447) | Quantization and the thinking language. Does not measure thinking length. |

No paper found that measures thinking length per language for quantized reasoning models.

**Idea 5 — Push the model to stop. Status: the general idea is done; this version is unique.**

| Paper | Link | Relation |
|:---|:---|:---|
| s1, Muennighoff et al. 2025 | [2501.19393](https://arxiv.org/abs/2501.19393) | Budget forcing. This is the baseline. |
| NoWait 2025 (EMNLP Findings) | [2506.08343](https://arxiv.org/abs/2506.08343) | "Wait"/"Hmm" logits to −∞. CoT 27–51% shorter. |
| AlphaOne 2025 | [2505.24863](https://arxiv.org/abs/2505.24863) | Replaces late "wait" with `</think>`. |
| Lotfi et al. 2026 | [2606.00206](https://arxiv.org/abs/2606.00206) | Penalty on overthinking words in quantized models. The nearest method. |
| "Pareto Frontiers of Reasoning Termination" 2026 | [2610.13570](https://arxiv.org/abs/2610.13570) | Compares fixed budgets, early exit and routing. |

New part: a bias on `</think>` that increases with length, tuned to the W16 length distribution. Compare it with the Lotfi word penalty.

### 6.4 Summary

| Idea | Status |
|:---|:---|
| 1 Swap | Unique |
| 2 Stop probability | Unique for quantization |
| 3 Calibration | Replication |
| 4 Language | Combination is unique |
| 5 Stop bias | Small step from Lotfi et al. |
