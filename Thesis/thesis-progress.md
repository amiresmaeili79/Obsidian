# Motivation

AI improves every day, and models keep getting bigger. At the same time, another side of AI is growing quietly: small language models. These models fit on consumer devices, such as a desktop, a laptop, or even a phone.

Small local models are popular with people who care about privacy, especially when they handle personal or sensitive data. Local models also work offline, without internet. More and more devices are expected to come with a language model built in, to make them more helpful.

The main limit here is resources. Most people do not own a GPU cluster that can run a large model. Many people also do not need the power of the big providers, such as OpenAI, Claude or Gemini. For small daily jobs, such as a quick assistant, a simple routine workflow or a text formatter, a small model is enough. So the real question is: how large does a model need to be for a given task? And what can we do if that model does not fit on the device?

This is where quantisation helps. Quantisation makes a model smaller, so it fits on smaller devices. There are many ways to quantise a model, and each has pros and cons. In this research, we study how quantisation affects small language models: what we lose, and what we keep, when we shrink them.

Many recent papers measure the loss from quantisation, and they give good benchmarks. However, we noticed a gap. Most of these works test single-turn tasks: the model gets the whole problem at once and answers once. They report that careful quantisation loses very little, especially when the model is a bit larger.

Real assistants often work differently. They run many steps, one after another, and each step needs the previous one to be correct. If one step is wrong, every later step is built on a wrong result. In a single-turn task, the model sees everything at the same time. In a multi-turn task, it gets the details step by step and has to remember them. We suspected that quantised models may not stay "almost lossless" when they get the task step by step.

To test this, we chose a state tracking task. The model keeps a state and updates it at every turn. A single mistake ruins all later turns.

We want to answer these questions:
- How do different quantisation methods (different ways of saving memory) affect the model's accuracy?
- Does it matter if the model's thinking (chain of thought) is on or off?
- Does the effect depend on the task, or is it the same everywhere?
- If there is a loss, does it grow over the turns or stay flat?
- Is the effect the same in different languages?
- Does model size change the sensitivity?
- Can the effect come from a specific model architecture?

# Key Terms

Short definitions of terms used in the rest of this note.

- **Quantisation**: storing the model's numbers with fewer bits. 16 bits is the normal (native) precision here. Fewer bits means less memory, but possibly lower accuracy.
- **Weights**: the learned numbers of the model. They decide how the model processes its input. Quantising them from 16 to 4 bits makes them about four times smaller.
- **KV cache**: the model's working memory of the conversation so far. The model keeps it so it does not have to re-read the whole conversation for every new word. It grows with every turn.
- **Thinking mode (chain of thought, CoT)**: the model first writes its reasoning inside `<think> … </think>`, and then gives the answer. The models we use can switch this on or off.
- **Episode**: one full conversation. **Turn**: one user request and the model's answer to it.

# Dataset

## Why Not SCONE

We started with SCONE (Sequential CONtext-dependent Execution dataset). It is a simple dataset with three themes: alchemy, scene and tangrams. Each episode has 5 turns. An episode starts with an initial state, and each turn asks, in natural language, to change that state.

Below is an example from the alchemy theme. Each number is a beaker, and each letter is one unit of a colour, written as its first letter (for example, `y` for yellow).

Initial state:
```
1:yy 2:ooo 3:pppp 4:o 5:gggg 6:rrr 7:ppp
```
Command:
```
throw out one unit of last beaker
```
Expected result:
```
1:yy 2:ooo 3:pppp 4:o 5:gggg 6:rrr 7:pp
```

Our first tests with this dataset looked good. But during manual review we found two problems:
1. Some episodes had logic errors: the ground truth did not match the command.
2. Many commands were ambiguous. In the example below, in turn 3, it is not clear what "it" refers to.

```
"domain": "alchemy",
"initial_state": "1:ggg 2:_ 3:_ 4:gggg 5:_ 6:oooo 7:oo",
"turns": [
  {"turn_number": 1, "instruction": "leave one unit in the first flask",
   "ground_truth_state": "1:g 2:_ 3:_ 4:gggg 5:_ 6:oooo 7:oo"},
  {"turn_number": 2, "instruction": "then, pour what's left into the last one",
   "ground_truth_state": "1:_ 2:_ 3:_ 4:gggg 5:_ 6:oooo 7:oog"},
  {"turn_number": 3, "instruction": "then, empty it out",
   "ground_truth_state": "1:_ 2:_ 3:_ 4:gggg 5:_ 6:oooo 7:_"},
```

## Our Own Dataset

Because of these problems, we built our own dataset. A small proof of concept, made with some help from AI, worked well, so we continued with it.

The dataset simulates a shopping assistant for IT equipment. In each turn, the user asks the assistant to change a shopping cart. The generator is flexible: you can change the actions per turn, how often each action appears, the product names, and more.

The generator has two parts:
1. **State engine** (deterministic): picks the action for each turn, keeps the cart state, and keeps everything in order.
2. **AI generation**: a language model turns each action into a natural sentence.

### State Engine

The engine is controlled by a JSON config file. The main settings are:
- the number of episodes,
- the number of turns per episode,
- a random seed, so the same config always gives the same dataset,
- whether arithmetic is on. With arithmetic on, the model must also report the total price of the cart (see Task).

```json
{
	"generation_settings": {
		"episode_count": 1,
		"global_seed": 42,
		"max_turns": 10,
		"arithmetic_enabled": false
	}
}
```

**Product names.** You can choose any naming pattern. We used made-up names, so the models could not have seen them in their training data. A pattern is a string with variables in it, and you give a list of values for each variable. One variable sets the price tier, and each tier has a price range from which a random price is drawn. We used the pattern `"[brand]-[model]-[variant]"`, and the `variant` sets the price.

The engine works in three steps:
1. **Build the catalogue.**
   1. Create every product name that the pattern and the values allow.
   2. Apply the collision threshold. It sets what share of the products use the same brand (the first part of the name). This creates many names that look alike, so the model has to remember exactly which product is which. We added it to put pressure on the model's memory (the KV cache).
   3. The result is a catalogue (all products available in the episode) and an active set (the products that the turns will use).
2. **Sample the episodes.**
   1. You set the number of episodes, the number of turns, and how often each kind of action appears. Actions are grouped into three tiers. The mix is controlled so each episode gets a fair share of each tier, and no episode is too easy or full of the same hard action.
      - **Tier 1 (T1)**: simple actions, such as adding or removing a product, with a simple request.
      - **Tier 2 (T2)**: anything about shipping or discounts. The model has to understand what these mean.
      - **Tier 3 (T3)**: actions that refer back to the previous action, not only to the current cart. For example, "cancel what I just added". The model has to look at the previous request to know what to undo.
      - **Full query**: at fixed turns, the user asks to see the whole order so far.
   2. For each turn, an action is chosen by simple rules (for example, the first action must be an add). Each turn also gets a style, which the AI generation step uses for the sentence: direct, hesitant, implicit, comparative or approximate.
3. **Save the result** to a JSON file.

### AI Generation

We used `Gemma4-31B-it` to write the sentences. The process was different for each language.

English is the main language. For English, the model writes each sentence from the state engine's output:
1. A fixed base prompt holds the general rules for every sentence.
2. A short instruction on how to phrase the action is added.
3. The chosen style is added.
4. Shipping and discount turns get extra rules. A discount must mention its percentage and a short code name (for example, SAVE18). A shipping method must use its exact English name.

We also tested two other languages: Persian and Slovak. For these, we used the same model, but we translated the English dataset instead of generating new sentences. There were two reasons:
1. The episodes stay the same in every language, but the sentences still have the natural features of each language.
2. Gemma was weak at generating sentences directly in these languages. Its sentences read like English.

Because all languages share the same episodes, we can compare them episode by episode.

### What the Dataset Looks Like

The previous sections covered how the dataset is made. This section describes what it looks like.

The dataset has 150 episodes, with 10 turns each.
- The first turn is always an add, to start the cart.
- Each catalogue has 6 to 12 products, chosen at random. The active set has 3 to 7 of them.
- The collision threshold is 0.6, so at least 3 products share a brand and can be confused. We tested the effect of this setting and it was not clear, so we kept it as a design choice.
- Each episode has two full queries: one late in the episode (turn 7) and one at the end (turn 10). In early tests, a full query earlier in the episode did not change accuracy, so we dropped it.

The tiers are mixed as follows:
- **T1: 60%.** All changed products are named in the request, so the model does not need to resolve any reference. We kept this share high because it matches how a simple, step-by-step agent works: take the state, apply a change, return the result (each step depends only on the current state).
- **T2: 20%.** Adding or removing a discount code, or changing the shipping.
- **T3: 20%.** The model has to look back at the previous action to see what was added or increased, so it can undo it. We expected these actions to show the effect of quantisation more than the others.

### Known Flaws

18 of the 150 episodes have a flawed turn (19 turns in total). The same episodes are flawed in Slovak and Persian, because they are translations.
- **Foreign product (11 turns):** the AI generation step named a product that is not in the catalogue, for example "XPS-15-Laptop".
- **No change (9 turns):** the turn leaves the order as it is, for example "change X to 3" when X is already 3. The expected answer still lists the line.
- Episode 45 has both flaws in the same turn.

Dropping these 18 episodes raises the success rate by about 2 points per run (range −2.1 to +6.1), but the effects stay the same. Pooled change in pp, 95% interval:

| Effect | All 150 episodes | Without the 18 |
|:---|---:|---:|
| 4-bit weights, small models, thinking on | −13.7 [−16.4, −11.1] | −13.3 [−16.1, −10.4] |
| 4-bit weights, small models, thinking off | −3.3 [−5.7, −1.1] | −3.5 [−5.9, −1.1] |
| 4-bit weights, large models | −0.3 [−1.5, +0.9] | −0.1 [−1.3, +1.1] |
| 4-bit weights, large models, arithmetic, thinking on | −2.0 [−4.3, +0.4] | −2.7 [−5.1, −0.3] |
| 8-bit KV cache, all models | −1.0 [−1.7, −0.2] | −0.9 [−1.7, −0.1] |
| More episodes ending on the token limit, small models | +13.7 [+10.8, +16.8] | +13.1 [+9.8, +16.3] |

- The only change: with arithmetic, the 4-bit loss of the large models gets slightly larger, and its interval no longer includes zero.
- So the flawed episodes are kept, and each analysis is also checked without them. A rerun would only replace them with new random runs.
- Source: `results/analysis/flawed_episodes.md`, made by `analytics/dst_report/flawed_episodes.py`.

# Task

This section explains what the model has to do to pass.

## What the Model Sees

First, the model gets a system prompt, the catalogue, and an acknowledgement ("Got it"). The model does not generate the acknowledgement. We add it to the history so every model starts from exactly the same point:

```txt
[USER] Here are the available products in the inventory:
NEXON-X200-MAX $839.00\nBLAKE-R500-ULTRA $1913.00\nBLAKE-K300-ULTRA $1609.00\nBLAKE-R500-PLUS $393.00\nDRAVEN-R500-PLUS $375.00\nCYPHER-R500-LITE $146.00
[Assistant] Got it
```

Then, at each turn, the model gets the whole conversation so far plus the new request. We remove the reasoning (CoT) of earlier turns from the history, to keep the history clean.

The system prompt is in the same language as the dataset (English, Persian or Slovak). To keep things as similar as possible, product names, shipping methods and discount codes stay in English in every language, and the model is asked to keep them in English too.

## What the Model Must Answer

The model must answer with only the lines that changed in this turn (the change since the previous turn). On a full query, it must list the whole order, including shipping and discount.

There are two variants of the task:
- **Arithmetic off (Math_Off)**: only the changed lines.
- **Arithmetic on (Math_On)**: the changed lines, plus the total price of the cart after the discount. This uses a modified system prompt.

An example first request:
```txt
Add 1 unit of DRAVEN-R500-PLUS to the order.
```

Expected answer with arithmetic off:
```txt
DRAVEN-R500-PLUS: 1
```

With arithmetic on:
```txt
DRAVEN-R500-PLUS: 1\ntotal: $375.00
```

An example full query:
```txt
Can I get a quick look at everything in my order, including the shipping and any discounts?
```

Arithmetic off:
```txt
DRAVEN-R500-PLUS: 1\nBLAKE-K300-ULTRA: 10\nNEXON-X200-MAX: 5\nshipping: standard\ndiscount: none
```

Arithmetic on:
```txt
DRAVEN-R500-PLUS: 1\nBLAKE-K300-ULTRA: 10\nNEXON-X200-MAX: 5\nshipping: standard\ndiscount: none\ntotal: $20660.00
```

## How a Turn Is Scored

- **One failure ends the episode.** An agent builds each step on its previous state. After a wrong turn, the next turns cannot be scored fairly, so we stop (abort) the episode at its first failed turn.
- **Extra correct lines are allowed, but flagged.** Some models also print lines that did not change, with correct values. We still count these turns as correct, so the episode can continue, but we flag them as **redundant**. This is a problem of following the instructions or the format, not of tracking the state.
- **Token limit.** Each turn can generate at most 4096 tokens. A shopping assistant should be fast, so we did not want very long answers. This limit matters mostly when thinking is on: if the model thinks for more than 4096 tokens, the turn is cut off, counted as failed, and the episode is aborted.

# Experiment Design

## Models

We chose five models: three from Qwen3.5 and two from Granite-4.2.
- Qwen3.5: 4B, 9B, 27B
- Granite-4.2: 8B, 30B

## Quantisation

We used two kinds of quantisation:
1. **Weights to 4 bits (AWQ, asymmetric).** The weights handle the logic at each step, so we expected reasoning to get worse when they are quantised.
2. **KV cache to 8 bits (fp8).** In a multi-turn conversation, the KV cache grows at every turn. With thinking on, it can grow a lot, because the reasoning of the current turn is also stored. This matters because the model may be asked to change something it added many turns earlier. Quantising the cache could make that history "blurry", especially when product names look alike.

We quantised all five models with 4-bit asymmetric AWQ. We left `lm_head` and `embed_tokens` (the output and input layers) untouched, because quantising them causes a large accuracy loss. For calibration, we used 256 samples from UltraChat (multi-turn conversations). We chose it because it is a common dataset for quantisation, it is multi-turn like our task, and it is still different from our task.

vLLM handles KV cache quantisation natively.

## Thinking Mode

These models have a hybrid thinking mode, so thinking can be turned off. We used both modes, to see whether quantisation hurts one of them more, or both in the same way.

## The 2×2 Grid

For each model and thinking mode, we ran four configurations:
- **W16_KV16**: native, nothing quantised
- **W16_KV8**: only the KV cache quantised
- **W4_KV16**: only the weights quantised
- **W4_KV8**: both quantised

## What Was Run Where

- **Math_On** was run only on the two largest models (Qwen3.5-27B and Granite-4.2-30B). A pilot on 9B gave poor results, so we dropped it.
- **Persian and Slovak** were not run on Qwen3.5-4B, because it already did poorly in English.
- **All cells use the same 150 episodes** (translated for the other languages). This makes episode-by-episode comparison easy.

## Sampling Parameters

We used the same values for all models, so they can be compared directly. Some values match what the model makers recommend and some do not.

When thinking is **off**:
- **Temperature 0.1.** In our task, the answers are very short and should be deterministic. A low temperature makes the model almost always pick the most likely token.
- **top_p 0.95.** Keep only the most likely tokens that together make up 95% of the probability.
- **top_k 20.** Keep only the 20 most likely tokens.
- At temperature 0.1, top_p and top_k change almost nothing. We kept them only to match thinking mode.

When thinking is **on**:
- **Temperature 1.0.** It is also the value Qwen and Granite recommend. A high temperature is important in thinking mode, because it lets the model explore different paths. With a low temperature the model becomes greedy and can get stuck in a loop.
- **top_p 0.95, top_k 20.** At temperature 1.0, some nonsense tokens can still be picked. One bad token can ruin the reasoning, so we filter them out to keep the reasoning clean.
- **Repetition penalty 1.05.** It scales down the scores of tokens that were already used, to reduce repetition. It counts tokens from both the prompt and the output, so it should be kept mild.
- **Presence penalty 1.5.** It does the same job in a different way: it subtracts a fixed amount from the scores of tokens already used.
- Together, the two penalties help the model avoid repeating itself and getting stuck in a loop.

# Results

In this section I log whatever I see. Later we can compare the observations and decide how strong each one is.

Each observation has a check under it:
- ✅ supported by the data
- ⚠️ partly supported, or too few cases to be sure
- ❌ not supported

Bullets marked ➕ were added after review: patterns that were missing from my first pass.

Terms used below:
- **Success rate**: share of episodes that pass all 10 turns.
- **Exhausted**: the model hit the 4096-token limit while thinking, so it gave no answer.
- **Unclosed**: the model ended its turn without closing `</think>`.
- **Quantity / Shipping / Total / Hallucination / Format errors**: the kind of mistake that ended an episode. Hallucination means naming a product that does not exist, or that is not in the cart. Format means the answer is not in the `key: value` form.
- **Redundant**: correct turns that also repeated unchanged lines (see Task).
- **Significant**: the paired test over the same episodes says the difference is unlikely to be chance, after correcting for testing many comparisons.

## English

### Qwen3.5

**4B, Math_Off**
- With thinking off, all four configurations have similar accuracy. The accuracy is not high, because it is a small model. So quantisation does not seem to affect it, at least in this task.
	- ✅ All three quantised configurations are within a few points of native, and none of the differences is close to significant.
- With thinking on, quantisation lowers accuracy. The drop from KV8 is mostly visible with 16-bit weights. In general, thinking hurts 4B, and it hurts the quantised model most.
	- ✅ The 4-bit drop is large and significant at both KV settings. Thinking makes the 4-bit model significantly worse than without thinking, while for the native model thinking on and off are a tie.
	- ⚠️ The KV8 drop is indeed larger with 16-bit weights, but neither KV8 drop is significant. Over the whole study, KV8 costs only about 1 point.
- With thinking off, the 4-bit model follows the instruction better: it prints fewer lines that are not part of the change.
	- ✅ The native model repeats unchanged lines on most of its correct turns; the 4-bit model on only a small share. The confidence intervals are far apart. The 4-bit answers are also shorter.
- With thinking on, the redundancy is almost gone, but the quantised model is still not as good as the native one. **Maybe thinking improves instruction following.**
	- ✅ Redundancy falls to almost zero with thinking. This holds for every model that had high redundancy without thinking (Qwen 4B, Granite 8B, Granite 30B). With thinking on, W4_KV8 repeats more than native, with intervals that do not overlap. Granite 8B shows the same pattern. The amounts are tiny.
- With thinking on, the quantised model more often keeps thinking until it is cut off (almost twice as often). Sometimes the model also seems unable to start its thinking correctly.
	- ✅ With 4-bit weights, clearly more episodes end on the token limit (roughly 1.7 times as many). But the typical turn is not longer: the average tokens per turn stay the same. So the quantised model does not think longer in general; more of its turns run away until the limit.
	- ⚠️ The "Unclosed" turns are not a failure to start thinking. In these turns the model writes its reasoning, and often the answer too, but never closes the thinking block, so the answer is lost inside the reasoning. These are only a few episodes per configuration.
- With thinking on, quantity errors are halved. Reasoning helps the model keep the correct quantity for each product.
	- ⚠️ Quantity errors do go down with thinking, but per answered turn by about a third, not half. The share of episodes ending on a quantity error falls more, but partly because with thinking on many episodes have already ended on the token limit, before they could make a quantity error.
- The first full query (turn 7) causes more failures than the second one (turn 10). Probably the second query can use the answer to the first one, which makes it easier.
	- ⚠️ The pattern is real with thinking off: the turn-7 query fails several times more often than the turn-10 query (Granite 8B shows the same). With thinking on it is mixed. The reason is not proven. Another explanation is selection: only episodes that already passed the turn-7 query reach turn 10. To separate the two, check whether turn-10 answers copy the turn-7 list.
- ➕ With thinking on, 4B writes the longest reasoning of all the models, about twice as many tokens per turn as 9B or 27B. Quantisation does not change the typical length.
	- ✅ Mean and median tokens per turn are about the same in all four configurations.
- ➕ With thinking off, the shipping errors are the same in every configuration, so quantisation does not cause them. Most of them are the model writing "express" when the user asked for "priority". The example in the system prompt shows `shipping: express`, and the prompt never lists the valid shipping tiers, so the model seems to copy the example.
	- ✅ Without thinking, about a third of 4B's "priority" requests get "express". With thinking this becomes rare. The larger models do it less, but 9B and 27B also do it sometimes.

**9B, Math_Off**
- With thinking off, 9B behaves like 4B. But because the model is stronger, quantisation has a small visible effect. KV8 does not move the accuracy and looks like a free lunch. With thinking on, accuracy is again lower than with thinking off, which again suggests thinking is not useful for every task. With thinking on, the KV8 result may be noise, but the 16-bit vs 4-bit gap is visible at both KV settings.
	- ✅ With thinking off, 4-bit weights cost a few points, significant at both KV settings. KV8 changes nothing.
	- ✅ Thinking never helps 9B: all four configurations are lower with thinking on. For one 4-bit configuration the gap is significant.
	- ✅ With thinking on, the KV8 differences are small and not significant, and the 4-bit drop is large and significant at both KV settings. Most of the 4-bit drop comes from episodes ending on the token limit.
- Hallucination is much lower than in 4B. This shows that when the model gets bigger, it absorbs the quantisation effect. Thinking causes some hallucination (not much).
	- ❌ Hallucinations are rare in both models (a handful of episodes per configuration), and 4B and 9B have about the same rate per turn. Thinking on and off differ by one or two episodes. This is too few for any conclusion. Also, hallucination counts cannot show that size absorbs quantisation; the success rates of 27B show that.
- As in 4B, quantity errors are the most common, and quantisation increases them. KV8 does not matter for this or any other error type. Thinking balances the quantity errors of the quantised versions, while the native one stays the same.
	- ✅ With thinking off, 4-bit weights raise the quantity error rate per turn by about half, and KV8 changes nothing. With thinking on, the 4-bit versions have the same quantity error rate as native, and native is the same with or without thinking.
- Redundancy increases with quantisation (here KV8 also seems to matter), but the effect is not strong.
	- ⚠️ The order fits (native lowest, both quantised highest), but the amounts are small and the intervals overlap. Note it is the opposite of 4B, where 4-bit weights reduced redundancy.
- ➕ With thinking on, the 4-bit model thinks clearly longer than native: about half again as many tokens per turn. About four times as many episodes end on the token limit. Unlike 4B, here all turns get longer, not only a few runaway turns.
	- ✅ Both the mean and the median tokens per turn rise by about half with 4-bit weights. KV8 changes neither.

**27B, Math_Off**
- The effect of quantisation is not visible. When the model gets bigger, it seems to absorb the accuracy loss. Again, thinking lowers the accuracy.
	- ✅ All quantised configurations are within 2 points of native, with or without thinking. Thinking on is lower than off in all four configurations. Each gap alone is not significant, but pooled over the large models (27B and Granite 30B, all languages) the drop from thinking is clear.
- Thinking fixes many problems the model had without thinking, such as shipping errors and redundancy.
	- ⚠️ The direction is right: shipping errors and redundancy mostly disappear with thinking. But both were already rare without thinking (a few episodes), so the effect is small.
- The reason 27B with thinking falls behind 27B without thinking seems to be token exhaustion.
	- ✅ The share of episodes ending on the token limit is about the same size as the gap between thinking on and off. The other error types are about the same in both modes.
- ➕ Quantisation does not change how long 27B thinks.
	- ✅ Tokens per turn are about the same in all four configurations.

**27B, Math_On**
- Without thinking, the model is very bad (about 30% success).
	- ✅
- With thinking, it gets its strength back. Thinking helps a lot with arithmetic.
	- ✅ Thinking raises success by more than 50 points in all four configurations, all significant.
- Thinking also solves many other problems, but most failures without thinking are exactly the total price.
	- ✅ Without thinking, a wrong total ends almost 9 out of 10 failed episodes. With thinking, a wrong total almost never happens.
- What hurts the thinking model is token exhaustion.
	- ⚠️ Exhaustion is one of two main causes. Wrong quantities end about as many episodes. Arithmetic itself is no longer the problem.
- ➕ With arithmetic on, the model thinks about half again as long as without it. Quantisation does not change this.
	- ✅ Tokens per turn are clearly higher than on Math_Off, and the same in all four configurations.

### Granite-4.2

**8B, Math_Off**
- Without thinking, the model is really bad, so I don't compare quantisation there: the model has hit the floor. (I am not looking at its other errors either: all error types appear, and redundancy is very high.) With thinking, the effect of quantisation is clear: the 4-bit versions are lower than native. Again, KV8 does not matter much.
	- ✅ With thinking off, all configurations are low, and their order makes no sense (one 4-bit configuration is far below the other). With thinking on, both 4-bit configurations are significantly below native, and the KV8 differences are not significant.
- With thinking on, the quantised versions run out of tokens more often.
	- ✅ More episodes end on the token limit with 4-bit weights, the most with both quantised. Unlike Qwen 4B, the average turn also gets longer as more parts are quantised.
- KV8 seems to increase quantity errors, and so does 4-bit.
	- ⚠️ The quantity error rate per turn rises step by step from native to KV8 to 4-bit to both. The direction is consistent, but the counts are small and not tested for significance.
- The same applies to redundancy.
	- ✅ With thinking on, redundancy rises from native to both quantised, and the intervals of these two do not overlap. The amounts are tiny.

**30B, Math_Off**
- KV8 drops success by about 4 points in both weight settings. This might be interesting to check, because Granite has a different architecture that relies more on the KV cache.
	- ⚠️ In English, KV8 is lower in all four comparisons (both weights, both modes), and pooled over these four the drop is small but real. But over all three languages, Granite 30B's KV8 cost is about 1 point and not significant. Granite as a whole is only slightly more sensitive to KV8 than Qwen, within noise. The architecture explanation (every Granite layer keeps a KV cache, while Qwen3.5 has one in only a quarter of its layers) is plausible but not tested. Check the model configs before relying on it.
- Thinking lowers success a bit more when the model is quantised.
	- ⚠️ The gap between thinking off and on is larger with 4-bit weights than with 16-bit, but none of these gaps is significant.
- "Quantised models think more" does not really apply here. But format errors with thinking seem to increase with KV8 (this should really be checked).
	- ⚠️ The quantised versions do think a little longer (about 10–15% more tokens per turn), but this does not lead to more exhaustion. So the first part is mostly right.
	- ⚠️ KV8 adds only a few format errors, so the link to KV8 is weak. The bigger finding is what these errors are. Almost all are answers like "(no output – the quantity was already 3)", and they come in two kinds. About half are on dataset turns that really change nothing (for example "change X to 3" when X is already 3). There the model is right, and the dataset is at fault: it expects the unchanged line to be repeated. Most of the rest are real mistakes: the user asks to add 3 of a product the cart already holds 3 of, and the model reads it as "set to 3". Both kinds appear much more with thinking on than off.
- Hallucination goes down with thinking.
	- ⚠️ The direction is right, but these are only 0 to 4 episodes per configuration.

**30B, Math_On**
- Similar to Qwen: thinking is necessary for arithmetic. Without it, most errors are the total price.
	- ✅ Without thinking, success is about 10%, and a wrong total ends most episodes. Thinking raises success by about 50 to 60 points, significant in all four configurations.
- Quantisation lowers the accuracy, but the pattern looks more like noise (better check with other seeds).
	- ✅ Already checked: native vs W4_KV16 was rerun with two more seeds. The first run overstated the drop, but a smaller drop of about 6 points is real over the three runs. Single runs of the same configuration varied by up to about 9 points.
- Interestingly, 4-bit seems to match better with KV8: W4_KV8 beats W4_KV16 on every error type. Format errors improved.
	- ❌ W4_KV8 has fewer exhausted and format errors, but more quantity and shipping errors, so it is not better on every type. Also, the W4_KV16 run used here was a low one: in the two extra seed runs, W4_KV16 scored higher, close to W4_KV8. The gap is within normal run-to-run noise.

### Across Models

- ➕ **Does the loss grow over the turns?** For the small models with thinking on (Qwen 4B and 9B, Granite 8B), 4-bit weights add only a few failures in the first half of the conversation and more in the second half. A large part of the late gap is at the last turn, the final full query. For the large models (Qwen 27B, Granite 30B), the gap stays flat, near zero.
	- ✅ Failure rate at each turn, among episodes still alive, for native vs 4-bit weights: the gap is bigger in turns 6–10 than in turns 1–5 for all three small models. It is flat for both large ones.
- ➕ **Which actions does 4-bit hurt most?** For Qwen 4B and 9B with thinking on, the extra errors from 4-bit weights are largest on full-order queries and on cancel-last-add (the only T3 action), and small on simple adds. For Granite 8B they are largest on increase-quantity and cancel-last-add. So, as we expected, T3 shows the effect more than other actions. The full-order queries show it too.
	- ⚠️ Each action has only about 60 to 250 turns per configuration, so the numbers per action are noisy. Only some of them are significant on their own.
- ➕ **How much do single runs vary?** Rerunning the same thinking-on configuration with a different seed changed success by up to about 9 points. So a difference of a few points in one thinking-on cell is not meaningful on its own. Claims should rest on effects that repeat across configurations, or on pooled tests.
	- ✅ Seed repeats of Granite 30B (English, Math_On) and Qwen 9B (Slovak).

## Farsi

### Qwen3.5

**9B, Math_Off**
- With thinking off, quantisation has no effect: all four configurations land at about the same success rate.
	- None of the quantised configurations is more than a point or two from native, and none is significant.
- With thinking on, 4-bit weights clearly lower success, while KV8 does not. The drop comes almost entirely from running out of tokens.
	- The 4-bit drop is significant at both KV settings. With 4-bit weights, about twice as many episodes end on the token limit. The other error types stay the same or go down.
- Thinking does not help 9B. It hurts the 4-bit model most.
	- With native weights, thinking on is a bit lower than off (not significant). With 4-bit weights, thinking on is much lower, significant at both KV settings.
- 9B thinks very long in Persian. Even at native precision, about 1 in 6 episodes ends on the token limit. 4-bit weights make the reasoning a little longer still.
	- Mean tokens per turn are about twice the English value. 4-bit weights add about a sixth.
- Without thinking, many episodes end on a missing product line, not only on a wrong quantity. The model leaves items out of the full-order list, and it often fails to undo the last add. The Persian cancel requests are often vague (like "forget that last part"), and the model does not link them to the last added product.
	- Product errors end about 1 in 10 episodes. Most are on cancel-last-add and full-order queries.
- As in English, quantity errors are the most common error type, in every configuration.

**27B, Math_Off**
- Quantisation has no visible effect, with or without thinking.
	- All quantised configurations are within a few points of native, and none is significant.
- Thinking lowers success in all four configurations. Again, the reason is running out of tokens.
	- The gap is significant for one configuration after correction. The share of episodes ending on the token limit is about the size of the gap.
- 27B thinks longer in Persian than in English, but quantisation does not change how long it thinks.

**27B, Math_On** (only thinking on was run)
- With thinking, arithmetic works: a wrong total almost never ends an episode.
- Quantisation lowers success by a few points, but none of the drops is significant. The KV8 configurations have a few more episodes ending on the token limit. This looks like noise.
- With arithmetic on, the model thinks about half again as long as on Math_Off.

### Granite-4.2

**8B, Math_Off** (only thinking on was run)
- Granite 8B is very weak in Persian: only about a third of the episodes succeed at native precision.
- 4-bit weights roughly halve the success rate. KV8 does not matter much.
	- The 4-bit drop is large and significant at both KV settings. The KV8 differences are small and not significant.
- There are two main ways to fail: wrong quantities and running out of tokens. Quantisation only increases the second one.
	- Wrong quantities end about a third of the episodes in every configuration. Episodes ending on the token limit about double with 4-bit weights.
- The 4-bit model thinks much longer.
	- Tokens per turn rise by about a third with 4-bit weights.
- The extra failures from 4-bit weights pile up at the end of the conversation, mostly at the final full query.
	- In the last turn, the 4-bit model fails about three times as often as native. Full-order queries get the largest extra error from 4-bit weights of all actions.

**30B, Math_Off**
- Quantisation costs only a few points, with or without thinking. None of the drops is significant. The 30B absorbs quantisation here too.
- Thinking does not help 30B: success with thinking on and off is about the same.
	- One configuration is lower with thinking on. None of the gaps is significant after correction.
- Wrong quantities are the main error, especially without thinking. Thinking reduces them, but adds episodes that end on the token limit.
	- Without thinking, wrong quantities end almost a third of the episodes. With thinking, fewer, while about 1 in 7 episodes ends on the token limit.
- With thinking on, a few answers are strange: a clarifying question in Persian, a reply in Russian, a cut-off line, or "(no output)". These are scored as format or parse errors.
- 4-bit weights make the reasoning slightly longer (about a tenth).

**30B, Math_On** (only thinking on was run)
- Only about half of the episodes succeed. The failures are spread over three causes: running out of tokens, wrong quantities, and wrong totals. Unlike Qwen 27B, Granite still makes arithmetic mistakes when thinking.
- W4_KV16 is clearly lower than native (significant). The other two quantised configurations are lower too, but not significantly, so the pattern is not clean. Single thinking-on runs can vary by up to about 9 points, so this should be checked with more seeds before relying on it.

### Across Models

- **Loss over turns:** as in English, the extra failures from 4-bit weights grow late in the conversation for the small models (Granite 8B strongly, Qwen 9B too), mostly at the final full query. For the large models the gap stays flat.
- **Which actions:** for both small models, full-order queries get the largest extra error from 4-bit weights.
- **Shipping:** the "express" instead of "priority" mistake almost never happens in Persian.
- **Dataset caveat:** about 6 change-quantity turns use an "instead of …" phrasing that can point to the wrong product (found in native-speaker review). The impact is under one episode per configuration.

## Slovak

### Qwen3.5

**9B, Math_Off**
- With thinking off, quantisation has no effect. All four configurations are within a few points of each other, and the KV8 one is even slightly higher than native.
	- None of the differences is significant.
- With thinking on, success falls step by step: native, then KV8, then 4-bit, then both. Only "both quantised" is significant on its own. But the 4-bit loss is real: two extra seed runs show it in every run.
	- Over three runs, 4-bit weights cost about 9 points, with a confidence interval that does not include zero.
- The cause is again running out of tokens. But the typical reasoning does not get longer: more turns run away to the limit.
	- Episodes ending on the token limit about double with 4-bit weights, while the mean tokens per turn stay the same. Format errors also go up a little with 4-bit.
- Thinking hurts 9B: success is lower with thinking on in all four configurations.
	- The gap is significant in three of the four.
- Without thinking, quantisation makes the model repeat more unchanged lines.
	- Redundancy goes from about 1% of correct turns at native to about 7% with both quantised. The intervals barely touch. This is the opposite of English 4B, where 4-bit weights reduced redundancy.

**27B, Math_Off**
- Quantisation has no effect, with or without thinking.
	- All configurations are within a few points of native, none significant.
- Thinking clearly hurts 27B in Slovak, more than in English. The cause is running out of tokens.
	- Thinking on is lower in all four configurations, significant in three. About 1 in 10 episodes ends on the token limit.
- Quantisation does not change how long it thinks.

**27B, Math_On** (only thinking on was run)
- With thinking, arithmetic works: a wrong total rarely ends an episode.
- Quantisation has no effect. The 4-bit configurations are even slightly higher (not significant).

### Granite-4.2

**8B, Math_Off** (only thinking on was run)
- About two thirds of the episodes succeed at native precision.
- 4-bit weights cost about 20 points (significant). KV8 alone costs about 8 points, which is borderline: it is not significant after correction.
- 4-bit weights hurt in two ways: about three times as many episodes end on the token limit, and the reasoning gets about a third longer. Wrong quantities also go up a little with any quantisation.
- The extra failures from 4-bit weights pile up at the end, mostly at the final full query.
	- In the last turn, the 4-bit model fails several times as often as native. Full-order queries get the largest extra error from 4-bit weights.
- Redundancy is small but rises with quantisation.
	- Native is lowest and "both quantised" is highest, with intervals that do not overlap.

**30B, Math_Off**
- With thinking on, quantisation has no effect: all four configurations land at the same success rate. Without thinking, the quantised configurations are a few points lower, but not significantly.
- Thinking does not help: success is lower with thinking on in all four configurations.
	- None of the gaps is significant after correction.
- Without thinking, shipping errors are unusually common. The model writes "express" when the user asks for "priority", probably copying the `shipping: express` example in the system prompt.
	- About a third of the "priority" requests get "express" without thinking. With thinking this is rare.
- With thinking, about 1 in 6 episodes ends on the token limit, even at native precision. There are also some empty answers (scored as parse errors). About half of them are on the dataset turns that ask for no change (the known dataset flaw). The rest are empty answers on real turns.
- Without thinking, 4-bit weights make the model repeat more unchanged lines.
	- Redundancy rises from about 2% of correct turns at native to 7–9% with 4-bit weights. The intervals do not overlap.

**30B, Math_On** (only thinking on was run)
- Only about half of the episodes succeed, and quantisation has no effect.
	- All four configurations are within a few points, none significant.
- The failures are mostly running out of tokens and wrong totals. Even with thinking, Granite makes arithmetic mistakes.
- 4-bit weights make the reasoning about a fifth longer, but this does not lead to more episodes ending on the token limit.

### Across Models

- **Loss over turns:** for Granite 8B, the extra failures from 4-bit weights grow late, mostly at the final full query. For Qwen 9B the growth is small. For the large models the gap stays flat.
- **Which actions:** for Granite 8B, full-order queries get the largest extra error from 4-bit weights, with smaller extra errors spread over several other actions. For Qwen 9B, the extra errors are spread out and none is significant on its own.
- **Redundancy:** in Slovak, 4-bit weights make models repeat *more* unchanged lines (Qwen 9B and Granite 30B without thinking, Granite 8B with thinking). In English 4B it was the other way round.
