# Experiment Ideas

Ideas for the final experiments. Papers for each idea: [[Related Work]].

**Status:** no idea is accepted yet. Choose first, then plan.

---

## Common Method: Prefill

Ideas 1 and 2 use this method.

1. Build the same conversation again. The episode files have all the data.
2. Put the thinking text at the start of the model reply.
3. The model continues from that point.

Do not open a new chat. Do not ask the model about its thinking.

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

`init_episode_messages()` in `evaluation.py` builds the first part.

---

## Idea 1 — Swap the Thinker and the Answerer

**Main idea.** Most promising. Papers found: none for quantization.

**Question.** Where does W4 cause the damage? In the reasoning, or in the decision to stop?

**Models.** Qwen 9B (shows the effect). Qwen 27B (control: no W4 penalty). English, Slovak, Persian. Optional: Granite 8B and 30B.

**Steps.**

1. Take the think-on turns of W16_KV16 and W4_KV16.
2. Build each prompt with the prefill method.
3. Close each trace with `</think>`.
4. If the trace is exhausted, add a fixed sentence first. Example: "I have to give the answer now."
5. Let W16 answer from all traces. Then let W4 answer from all traces.
6. Score each answer with the scorer v2 code.

**Result table.**

| | Answer by W16 | Answer by W4 |
|:---|:---:|:---:|
| **W16 thinking** | A | B |
| **W4 thinking** | C | D |
| **W4 exhausted thinking** | E | F |

**How to read it.**

- C ≈ A: the W4 reasoning is good. The damage is in the decision to stop.
- C < A: the W4 reasoning is worse.
- B < A: W4 is worse at turning good reasoning into an answer.
- E, F: how many exhausted turns a forced answer can repair.

**Risks.**

- The answerer can ignore the trace. In Math_Off, thinking is not needed. Thinking off scores higher than thinking on.
- Larger models can use their trace less. A null result at 27B can have the wrong cause.
- W16 and W4 runs reach a turn with different earlier answers. Choose one history for both.
- An exhausted trace stops in the middle of a thought. This input is unusual. Compare E and F with W16 on the same turns.
- The original thinking used temperature 1.0. The new answers use 0. Cell A must be close to the original accuracy.
- Turns in one episode are not independent. Use a paired test. Resample episodes for the confidence interval.

**Claim to write.** "W4 traces lead to answers as good as W16 traces." Do not write "W4 reasons as well as W16."

**Pilot first.** Answer with the trace and with an empty thinking block. Use 9B and 27B. Use a few hundred turns. If both give the same answer, the trace is not used. Then E and F become the main result.

**Decisions for you.**

- Wording of the forcing sentence.
- Answer temperature.
- Which history to use.
- Which test to use.
- Which difference is meaningful.

**Cost.** Low. The model writes about 50 tokens per answer.

---

## Idea 2 — Measure the Stop Probability

**Question.** Why does W4 not stop?

**Background.** Before each token, the model gives a probability to all possible next tokens. You can read these values without text generation.

**Steps.**

1. Take a W16 trace. Cut it where W16 wrote `</think>`.
2. Build the prompt with the prefill method. Do not close the trace.
3. Ask W16 for the next-token probabilities. Ask W4 for the same text.
4. Record P(`</think>`). Record the top other tokens. Examples: "Wait", "Let me", "One final check".
5. Repeat at other points. Use the point after a correct draft. Use each paragraph break.

```python
out = llm.generate(prompt_with_partial_trace, SamplingParams(max_tokens=1, logprobs=20))
top = out[0].outputs[0].logprobs[0]   # p_stop = P("</think>"); if not in top 20: "< smallest value"
```

**Result.** A statement of this type: "At the W16 stop point, W16 gives `</think>` 0.9. W4 gives 0.3 and prefers Wait." (The values are an example, not data.)

**Thesis figure.** P(stop) along the trace, W16 against W4.

**Cost.** Very low. No text generation.

**Check first.** `</think>` must be one token in the tokenizer.

---

## Idea 3 — Calibrate AWQ With Reasoning Text

**Status.** Others did this before (see [[Related Work]]). Use it as a replication.

**Background.** AWQ uses about 256 example texts. It keeps the important weights precise. The current script uses `ultrachat_200k`. These texts have no thinking.

**Steps.**

1. Make about 200 new episodes with a new seed. Never use the 150 test episodes.
2. Run W16 9B with thinking on these episodes.
3. Make 256 calibration texts. The last assistant message has `<think>…</think>` and the answer.
4. Print one text. Check that the thinking is in it. The Qwen template removes thinking from earlier turns.
5. Quantize 9B. Name: `Qwen3.5-9B-AWQ-reasoning`.
6. Run the 150 test episodes. Think on and off. KV16 only.
7. Run the idea 2 probe on the new model.

**Control.** Make one W4 model calibrated on DST conversations without thinking. This shows if the reasoning text helps, or only text from the same domain.

**Result table.**

| | W16 | W4 (UltraChat) | W4 (reasoning) |
|:---|:---:|:---:|:---:|
| Success (9B, English, think on) | 79.3% | 62.7% | ? |
| Episodes exhausted | 4.7% | 20.0% | ? |
| P(stop) at the W16 stop point | ? | ? | ? |

**Cost.** Medium. New episodes, one new quantization, two new runs.

---

## Idea 4 — Language and Thinking Length

**Status.** Mostly answered. Language changes the level. The language × AWQ interaction is not significant.

**Optional work.**

- Label how the extra tokens are used. Examples: translation, cart tracking, second-guessing. Use `analytics/thinking_analysis/`.
- Test an English system prompt with a Slovak customer. This shows if the cause is the instructions or the customer text.
- Give the English translation with the Slovak message. If the extra thinking stops, the cause is translation.

---

## Idea 5 — Push the Model to Stop

**Status.** The general idea exists (see [[Related Work]]). This version is new.

**Question.** Can we repair W4 at generation time, without a new quantization?

**Steps.**

1. Add a logits processor in vLLM. It adds a bias b(n) to the `</think>` logit.
2. The bias grows with the number of thinking tokens n.
3. Tune b(n) on W4 so that its thinking length is close to W16. Use new episodes, not the test episodes.
4. Run the 150 test episodes with W4 and the bias.

**Baselines.**

- Budget forcing at 1024, 2048 and 4096 tokens. At the limit, add a closing sentence and `</think>`.
- The word penalty from Lotfi et al.

**Measure.** Success rate. Exhaustion rate. Tokens per successful episode.

**Result.** A targeted fix (the bias) against a blunt fix (the cut-off). Idea 2 shows if the bias is the correct fix.

---

## Thesis Storyline

**"Quantization breaks the stopping, not the thinking."**

| Step | Idea | Question |
|:---|:---|:---|
| Locate | 1 | Where is the damage? |
| Explain | 2 | Why does W4 not stop? |
| Fix | 3 or 5 | Can we repair it? |
| Generalize | Granite, 4 | Does it happen in other families and languages? |
