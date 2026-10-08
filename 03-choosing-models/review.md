# Module 3 review: Choosing Models

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. You download `google/gemma-4-31B` (no `-it` suffix) and ask it a question. It replies with three more questions. Why?

<details><summary>Answer</summary>

It's a base model, trained only to continue text. It hasn't been instruction-tuned to answer. Use the `-it` (instruction-tuned) version. See [Closed vs open models](01-closed-vs-open-models.md#every-model-you-use-is-pre-trained).

</details>

### 2. Which of these are open-weight model families?
a) Llama  b) Claude  c) Qwen  d) Gemini  e) DeepSeek  f) Gemma

<details><summary>Answer</summary>

a, c, e and f. Claude and Gemini are closed, API-only models. (Gemma is Google's open-weight family, built from Gemini research.) See [The two camps](01-closed-vs-open-models.md#the-two-camps).

</details>

### 3. A teammate says "Llama is open source, so we can do anything with it." What's wrong with that?

<details><summary>Answer</summary>

Llama is open-weight under a custom community license, not an OSI open-source license. It has an acceptable-use policy, attribution requirements and a large-company user threshold. Always read the specific license. See [Open weights is not the same as open source](01-closed-vs-open-models.md#open-weights-is-not-the-same-as-open-source).

</details>

### 4. Roughly how much memory does an 8B-parameter model need at 4-bit quantization? At 16-bit?

<details><summary>Answer</summary>

About 5 GB at 4-bit (8 × 4 ÷ 8 = 4 GB, plus overhead) and about 16 GB at 16-bit, before the KV cache for context. See [Will it fit?](02-running-models-locally.md#will-it-fit-memory-math)

</details>

### 5. You have 24 GB of VRAM. Which is usually the better choice: a 14B model at 4-bit or a 7B model at 16-bit?

<details><summary>Answer</summary>

Usually the 14B at 4-bit. Both fit, and a bigger model with moderate quantization tends to beat a smaller one at full precision. Below 4-bit, quality drops faster. See [Quantization](02-running-models-locally.md#quantization-trading-precision-for-memory).

</details>

### 6. Your local RAG prototype on Ollama ignores most of the retrieved documents, and there's no error. What's the first thing to check?

<details><summary>Answer</summary>

The context length. Ollama defaults to a 4,096-token window and silently truncates longer prompts. Raise `OLLAMA_CONTEXT_LENGTH`, keeping memory in mind. See [Running models locally](02-running-models-locally.md#ollama-in-five-minutes).

</details>

### 7. What's the difference between `transformers`, Transformers.js and the `InferenceClient`?

<details><summary>Answer</summary>

`transformers` runs Hub models locally in Python. Transformers.js runs them in JavaScript, in the browser or Node, on the user's device. `InferenceClient` calls models hosted by Hugging Face's inference providers over the network. See [Hugging Face](03-hugging-face.md).

</details>

### 8. You need to flag spam in 2 million short messages a day. Why might a Hub text-classification model beat a frontier LLM?

<details><summary>Answer</summary>

A small specialist classifier is far cheaper and faster, runs on modest hardware, and gives deterministic results. At that volume, a per-token LLM bill adds up fast. Verify its accuracy on labeled examples first. See [Tasks](03-hugging-face.md#tasks-not-everything-needs-an-llm).

</details>

### 9. Estimate the daily cost: 5,000 requests, 1,000 input and 300 output tokens each, at $2 input and $10 output per million tokens.

<details><summary>Answer</summary>

Input: 5M tokens × $2 = $10. Output: 1.5M tokens × $10 = $15. Total about $25/day. See [Back-of-the-envelope cost](04-picking-the-right-model.md#back-of-the-envelope-cost).

</details>

### 10. Put these model-selection steps in order: test on real examples, apply hard constraints, define the quality bar, pick the cheapest that passes, shortlist across tiers.

<details><summary>Answer</summary>

Define the quality bar → apply hard constraints → shortlist across tiers → test on real examples → pick the cheapest that passes. Then repeat when models change. See [A five-step framework](04-picking-the-right-model.md#a-five-step-framework).

</details>

### 11. Scenario: legal says customer contracts may not leave your cloud account, but product wants frontier-level quality. What options do you put on the table?

<details><summary>Answer</summary>

Self-host the strongest open-weight model your infrastructure can serve (vLLM on GPUs in your account) and measure whether it clears the bar. Or check whether a closed model is available inside your cloud account under acceptable data terms (several vendors offer this through the major clouds). Test both on real contracts before deciding. See [Closed vs open models](01-closed-vs-open-models.md#the-trade-offs) and [Picking the right model](04-picking-the-right-model.md).

</details>

## Hands-on challenge

Run a mini bake-off for Course Buddy. Write 8 questions about Modules 1 and 2 of this course, along with what a correct answer must mention. Use the `compare.py` pattern from [lesson 4](04-picking-the-right-model.md) to run them through at least three models: one frontier, one mid-tier and one open model (through OpenRouter or local Ollama). Build a small table with correctness (out of 8), average latency, and estimated cost per 1,000 questions. Write two sentences on which model you'd ship and why.

<details><summary>Hint</summary>

Keep the expected facts short ("mentions that APIs are stateless", "says tokens are billed"), and check each answer by hand for now. Store the questions in a JSON file: you'll turn them into an automated eval set in [Module 11](../11-evals/README.md). For local models, `cost` is zero, but write down the hardware you used.

</details>

## Capstone checkpoint
- [ ] Course Buddy reads its provider and model from config (env vars or a config file), with no model IDs hard-coded in the chat logic
- [ ] `BUDDY_PROVIDER=anthropic` runs on Claude through the native SDK, with streaming
- [ ] `BUDDY_PROVIDER=openai` runs on an OpenAI model, and `BUDDY_PROVIDER=ollama` runs on a local model through the OpenAI-compatible API, both with streaming
- [ ] Conversation history is stored in a provider-neutral format, so you can switch providers between runs
- [ ] The `--json` mode still returns a validated `{answer, confidence}` on every provider (native structured output where supported, validate-and-retry elsewhere)

---
[← Last lesson](04-picking-the-right-model.md) · [Course home](../README.md) · [Next module →](../04-prompt-engineering/README.md)
