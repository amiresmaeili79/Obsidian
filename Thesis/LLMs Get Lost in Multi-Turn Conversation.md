# Paper Summary: LLMs Get Lost in Multi-Turn Conversation

**Authors:** Philippe Laban, Hiroaki Hayashi, Yingbo Zhou, Jennifer Neville (Microsoft Research & Salesforce Research)

[Paper Link](https://arxiv.org/abs/2505.06120)

## 📌 Core Finding ("Lost in Conversation")

While top Large Language Models (LLMs) excel at single-turn, fully-specified instructions, their performance drops by an average of **39%** in multi-turn conversations where user requirements are gradually revealed (underspecified settings).

> **The Central Thesis:** When LLMs take a wrong turn in a conversation, they get lost and do not recover.

## 📊 Aptitude vs. Reliability

The paper decomposes multi-turn performance degradation into two distinct metrics:

1.  **Aptitude (**$A$ **):** The model's best-case potential (90th percentile performance). This drops by a non-significant **16%** in multi-turn settings.
    
2.  **Unreliability (**$U$**):** The gap between the best and worst simulated runs. This skyrockets by **112% (more than doubling)** in multi-turn conversations.
    

- **Conclusion:** Multi-turn degradation is **not** caused by a lack of core reasoning capability (aptitude); it is driven entirely by extreme, cascading **unreliability** over the course of multiple conversational turns.

## 🔍 Root Causes of Conversational Failure

Through analysis of 200,000+ simulated conversations, the authors identified four behavioral failures:

- **Premature Answer Attempts:** LLMs routinely try to generate final code, SQL, or solutions in early turns when the prompt is still highly underspecified, rather than asking clarifying questions.
    
- **Answer Bloat:** LLMs overly rely on their own previous (incorrect) answer attempts. As new requirements are revealed, they patch their old responses instead of rewriting them, resulting in final solutions that are **20% to 300% longer** (and qualitatively worse) than single-turn counterparts.
    
- **Loss-in-Middle-Turns:** Just like the "lost in the middle" phenomenon for long-context windows, multi-turn LLMs pay heavy attention to details introduced in the first and last turns, while completely forgetting constraints introduced in the middle turns.
    
- **Over-Verbosity:** Longer responses from the assistant tend to contain unprompted assumptions or hypotheses, which ultimately derail the model's focus in subsequent turns.
    

## 🚫 Remedies That Fail

Common intuition-based solutions do not mitigate this multi-turn degradation:

- **Test-Time Compute / Reasoning Tokens (e.g., o3, DeepSeek-R1):** Reasoning models suffer from the same degradation because their thinking tokens lead to lengthier, more verbose responses that increase the risk of conversational derailment.
    
- **Greedy Decoding (Temperature** $T=0$**):** Reducing generation temperature to zero fails to fix unreliability in multi-turn settings. Sub-token hardware differences compound over sequential turns, triggering massive cascading deviations.
    
- **Agentic Orchestration (e.g., RECAP, SNOWBALL prompts):** Having the framework repeat past requirements at every turn (`SNOWBALL`) only recovers **15% to 20%** of the performance gap.
    

## 💡 Practical Workarounds for Users

Because LLMs cannot reliably self-correct once a conversation drifts, users should employ the following strategies when interacting with conversational AI:

1.  **If it fails, start fresh:** Do not try to debug or correct a derailed model in an active chat window.
    
2.  **Consolidate and reset:** Ask the LLM, *"Please consolidate everything we have discussed and all of my requirements so far into a single prompt."* Copy that summary, open a **completely new chat thread**, and paste it to get a single-turn, highly reliable response.