# Progress Log

*date: 10.05.2026*

## Papers

I read the following papers.

### How Language Model Hallucinations Can Snowball

What they did:

- Created 3 datasets of yes / no questions and asked AI. if answered wrong -> Explain why?
    - Is 9677 a prime number?
    - ChatGPT: No. 13 x 745 (which is wrong and asking separately founds its error)

Key takeaways:

- The reason AI models hallucinations snowball, is because they want to keep the chat consistent!

### Using Large Language Models for Zero-Shot Natural Language Generation from Knowledge Graphs

What they did:

1.  Gave ChatGPT, a real dataset of triples ( "Ronaldo / Portugal / Soccer" ) and asked to generate a human-like sentence in English and Russian
2.  Created 3 type of datasets (factual, counterfactual, fictional) and asked them to generate sentences

Key Takeaways:

- In standard test, English was okay, but Russian was really bad
- When the AI was given **Factual** data, it was more likely to confidently output statements that contradicted the prompt (hallucinations).
- When the AI was given **Fictional** data, it was more likely to just accidentally leave important information out of the sentence entirely.
- When the AI was given **Counterfactual** data, human readers rated the resulting sentences as noticeably less coherent and harder to follow

* * *

## A Few Papers that Are Against This

### On the Fundamental Impossibility of Hallucination Control in Large Language Models

[Paper](https://arxiv.org/pdf/2506.06382)

> **Theorem (Impossibility Theorem). No LLM can simultaneously achieve truthfulness, information conservation, knowledge revelation, and knowledge-constrained optimality when responding to non-trivial queries requiring knowledge integration.** **Hallucination is a manifestation of fundamental limitation of the inference process itself: there is no inference mechanism that can simultaneously satisfy four essential properties required for perfect hallucination control**.

### **Not All Needles Are Found**

**[Paper](https://arxiv.org/pdf/2601.02023)**

> ****Anti‑hallucination (AH) instructions can make some models overly conservative, sharply reducing accuracy in literal extraction and logical inference.** **We find that longer contexts alone do not guarantee better performance and can be detrimental when relevant evidence is diluted or widely dispersed.****

### What the paper actually reports (concrete examples)

- **Significant improvement (example):** *ChatGPT‑5‑mini* shows a large aggregate faithfulness jump under AH prompting: **74.1% → 89.8%** (Standard → AH) in the reported table.
    
- **Small or negative changes (examples):** *Gemini‑2.5‑flash* moves only slightly (**86.5% → 87.0%**), and *Claude‑4.5‑haiku* slightly decreases (**83.2% → 82.4%**). *Deepseek‑v3.2‑chat* shows mixed results depending on metric and capacity.
    
- **Trade‑off:** Where AH helped faithfulness, it often **reduced literal extraction and logical inference**, especially at large context sizes (the “safety tax”).