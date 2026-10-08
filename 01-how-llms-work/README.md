# Module 1: How LLMs Work

You don't need to know how to build an LLM to build with one, but you do need a working mental model of what happens between your prompt and its answer. This module gives you that model: text becomes tokens, a trained network predicts one token at a time, everything it can see fits in a fixed-size window, and a few sampling knobs decide which token comes next. Almost every cost, limit, and weird behavior you'll hit later traces back to one of these four ideas.

**You'll be able to:**
- Explain what tokens are and count them for a given text and model
- Describe how pretraining, fine-tuning, and RLHF produce a chat model, and what happens at inference time
- Reason about context windows: what fills them, what happens at the limit, and why "fits" isn't the same as "works well"
- Predict how temperature, top-k, top-p, and repetition penalties change a model's output

**Prerequisites:** [Module 0: Orientation](../00-orientation/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [From text to tokens](01-text-to-tokens.md) | ~15 min |
| 2 | [Training vs inference](02-training-vs-inference.md) | ~15 min |
| 3 | [Context windows](03-context-windows.md) | ~15 min |
| 4 | [Sampling knobs](04-sampling-knobs.md) | ~15 min |
| ✓ | [Module review](review.md) | ~10 min |

## Capstone checkpoint
Course Buddy gets its first real feature: count the tokens in every Markdown file of this course and estimate what it would cost to stuff them all into a single prompt. That number will shape your design decisions for the rest of the course. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../00-orientation/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-text-to-tokens.md)
