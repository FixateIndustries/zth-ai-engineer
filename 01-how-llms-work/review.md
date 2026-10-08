# Module 1 review: How LLMs Work

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Roughly how many tokens is a 2,000-character English paragraph?
a) About 50
b) About 500
c) About 2,000
d) About 8,000

<details><summary>Answer</summary>

**b.** The rule of thumb for English prose is about 4 characters per token. Code, numbers, and non-English text usually take more. See [From text to tokens](01-text-to-tokens.md).

</details>

### 2. A teammate estimates Claude costs using `tiktoken`. What's wrong with that?

<details><summary>Answer</summary>

`tiktoken` is OpenAI's tokenizer. Claude uses a different tokenizer, so the counts (and the cost estimate) will be off, typically too low. Use `client.messages.count_tokens` with the exact model you'll call. See [Counting tokens for Claude](01-text-to-tokens.md#counting-tokens-for-claude).

</details>

### 3. Why might a model struggle to count the letter "r" in a long word?

<details><summary>Answer</summary>

It sees tokens (multi-character chunks), not individual letters, so character-level details aren't directly visible to it. For exact character or arithmetic work, have the model call code. See [Why you should care](01-text-to-tokens.md#why-you-should-care).

</details>

### 4. Put these in order: supervised fine-tuning, pretraining, preference tuning (RLHF).

<details><summary>Answer</summary>

Pretraining, then supervised fine-tuning, then preference tuning. Pretraining builds knowledge, SFT teaches instruction following, and preference tuning shapes the model toward helpful, honest, safe responses. See [How a model gets trained](02-training-vs-inference.md#how-a-model-gets-trained).

</details>

### 5. Why do output tokens typically cost more, and take longer, than input tokens?

<details><summary>Answer</summary>

The prompt is processed in one parallel pass, but output is generated one token at a time, each step depending on the last. That sequential decoding is the expensive, slow part. See [Inference](02-training-vs-inference.md#inference-generating-one-token-at-a-time).

</details>

### 6. Your chatbot answers well on turn 1 but costs climb steadily over a 40-turn conversation. Why?

<details><summary>Answer</summary>

The model is stateless, so every turn resends the whole history. Input tokens (and cost) grow with each turn. Summarizing old turns and prompt caching help, and both are covered in [Module 5](../05-context-engineering/README.md). See [Chat history is just more context](03-context-windows.md#chat-history-is-just-more-context).

</details>

### 7. What's wrong with this code?

```python
response = client.messages.create(model="claude-sonnet-5-5", max_tokens=300, messages=msgs)
summary = response.content[0].text
save_to_database(summary)
```

<details><summary>Answer</summary>

It never checks `stop_reason`. If the answer hit `max_tokens` (or the context window), a truncated summary gets saved as if it were complete. It also assumes the first content block is text, but responses can include other block types, such as thinking blocks. Check `response.stop_reason` and filter for `block.type == "text"`. See [What happens at the limit](03-context-windows.md#what-happens-at-the-limit).

</details>

### 8. Your input alone is larger than the model's context window. What happens?
a) The model reads as much as fits and ignores the rest
b) The API rejects the request with an error
c) The response comes back with `stop_reason: "max_tokens"`
d) The model automatically summarizes the overflow

<details><summary>Answer</summary>

**b.** The Claude API returns a 400 `invalid_request_error` ("prompt is too long"). Some frameworks silently truncate instead, which is worse because you don't find out. See [What happens at the limit](03-context-windows.md#what-happens-at-the-limit).

</details>

### 9. A model has a 1M-token window, so you send your entire 800K-token knowledge base with every question. Name two problems.

<details><summary>Answer</summary>

Any two of: high cost per request, high latency, and worse accuracy because models use long contexts imperfectly (facts in the middle get missed, and irrelevant text distracts). Retrieving only relevant chunks usually works better ([Module 7](../07-rag/README.md)). See [Fitting isn't the same as working well](03-context-windows.md#fitting-isnt-the-same-as-working-well).

</details>

### 10. Next-token probabilities are mat 0.54, floor 0.24, sofa 0.13, roof 0.07, moon 0.02. With `top_p=0.75`, which tokens can be chosen?

<details><summary>Answer</summary>

Only "mat" and "floor". Their cumulative probability (0.78) is the first to reach 0.75. See [A worked example](04-sampling-knobs.md#a-worked-example).

</details>

### 11. You raise temperature from 0.5 to 1.5. What happens to the distribution?

<details><summary>Answer</summary>

It flattens. The top token loses probability and unlikely tokens gain it, so output becomes more varied and more prone to odd choices. See [A worked example](04-sampling-knobs.md#a-worked-example).

</details>

### 12. You set `temperature=0` on `claude-sonnet-5-5` to make a classifier deterministic, and the request fails with a 400. What now?

<details><summary>Answer</summary>

Current frontier Claude models don't accept non-default sampling parameters, so remove `temperature`. Get consistency another way: a tighter prompt, a fixed set of allowed labels via structured output ([Module 2](../02-calling-models/04-structured-output.md)), and evals to measure it. Temperature 0 never guaranteed identical outputs anyway. See [What you can actually set today](04-sampling-knobs.md#what-you-can-actually-set-today).

</details>

## Hands-on challenge

Build `tokens.py` for Course Buddy: it walks this course's folder, counts the tokens in every `.md` file with `client.messages.count_tokens`, and prints

1. the ten largest files with their token counts,
2. the total for the whole course, and
3. the estimated input cost of sending the entire course in one prompt, for two models of your choice (look up current prices on the [pricing page](https://platform.claude.com/docs/en/about-claude/pricing)).

Then answer: does the whole course fit in the context window? Even if it does, would you send it all with every question?

<details><summary>Hint</summary>

Use `pathlib.Path(course_dir).rglob("*.md")` to find files. Count each file by sending its text as a single user message:

```python
n = client.messages.count_tokens(
    model="claude-sonnet-5-5",
    messages=[{"role": "user", "content": path.read_text()}],
).input_tokens
```

Cost is `total_tokens / 1_000_000 * price_per_million_input_tokens`. Token counting is free but rate-limited, so a few hundred files is fine. If you want to go further, store the counts in a JSON file so later modules can reuse them.

</details>

## Capstone checkpoint
- [ ] Course Buddy has a command that counts tokens across all the course's `.md` files using the Claude token counting endpoint
- [ ] It prints a per-file breakdown and a total
- [ ] It estimates the input cost of stuffing the whole course into one prompt
- [ ] You've written down (in the project README or a comment) why you won't send the whole course on every question

See the [capstone overview](../capstone/README.md) for the full milestone list.

---
[← Last lesson](04-sampling-knobs.md) · [Course home](../README.md) · [Next module →](../02-calling-models/README.md)
