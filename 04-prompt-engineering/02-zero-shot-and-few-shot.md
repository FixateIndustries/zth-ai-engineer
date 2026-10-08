# Zero-shot and few-shot prompting

> Module 4 · Lesson 2 · ~12 min

**Goals**
- Tell zero-shot and few-shot prompting apart, and know when each is the right tool
- Write few-shot examples that teach format and judgment without causing copycat outputs
- Put examples in a prompt in two ways: inline in tags, or as example conversation turns

## Telling vs showing

**Zero-shot** means you describe the task and give no examples. The model relies on what it learned in training. It works well for tasks the model has seen a million times: summarize, translate, extract names.

**Few-shot** means you include a handful of worked examples (input and ideal output) before the real input. You're showing, not just telling. This is how you teach things that are hard to put into words: a house style, an unusual label set, where the line falls between two categories.

A good rule: **start zero-shot, add examples when instructions alone stop working.** Examples cost tokens on every request, so they should earn their place.

## Before and after: classifying feedback

Say you're tagging app-store reviews for a product team. Zero-shot first:

```text
Classify this app review as positive, negative, or mixed.

Review: "Love the new dark mode. Still crashes every time I open settings though."
```

Typical results across runs: `Mixed`, `mixed`, `This review is mixed because...`. The label is reasonable, but the format drifts, and your team actually wanted something richer: the *feature* involved and whether it's a bug report. That's hard to define in prose. So show it:

```text
Classify app reviews. For each review, output one line:
sentiment | feature | is_bug

<examples>
<example>
Review: "Sync is finally fast. Great update."
Output: positive | sync | no
</example>
<example>
Review: "App logs me out every morning. Annoying."
Output: negative | login | yes
</example>
<example>
Review: "Nice widgets, but I wish I could change the font size."
Output: mixed | widgets | no
</example>
</examples>

Review: "Love the new dark mode. Still crashes every time I open settings though."
Output:
```

Now the model returns `mixed | settings | yes` in the exact format, and it has learned two judgment calls you never wrote down: a feature request is not a bug, and when a review mentions two features, the one with the problem is what matters.

## What makes good examples

- **Cover the boundaries.** Include the cases where a careless reader would get it wrong (the "wish I could" review above is a feature request, not a bug). Easy examples teach little.
- **Vary them.** If every example is about login, the model may over-predict `login`. Vary length, topic, and wording.
- **Balance the labels.** Three positive examples and no negative ones nudge outputs toward positive.
- **Keep them consistent.** Inconsistent examples teach inconsistency. Every example must follow your own rules exactly.
- **Three to five is usually enough.** Add more only if you can see a specific failure they fix.

> [!WARNING]
> Models copy examples closely. If all your examples are one short sentence, outputs will be short too, even when the input needs more. If examples repeat a phrase ("Great question!"), expect to see it everywhere. Make sure the things you *don't* want copied vary across examples.

## Two ways to include examples

**1. Inline, in tags** (like above). Simple, and easy to keep in a single system prompt.

**2. As fake conversation turns.** You write previous user and assistant messages that never happened, so the model sees itself answering in the right format:

```python
import anthropic

client = anthropic.Anthropic()

few_shot = [
    {"role": "user", "content": 'Review: "Sync is finally fast. Great update."'},
    {"role": "assistant", "content": "positive | sync | no"},
    {"role": "user", "content": 'Review: "App logs me out every morning. Annoying."'},
    {"role": "assistant", "content": "negative | login | yes"},
]

def classify(review: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-5-5",
        max_tokens=100,
        system="Classify app reviews as: sentiment | feature | is_bug. Output one line only.",
        messages=few_shot + [{"role": "user", "content": f'Review: "{review}"'}],
    )
    return "".join(b.text for b in response.content if b.type == "text").strip()

print(classify("Love the new dark mode. Still crashes every time I open settings though."))
```

The conversation-turn style is handy in chat apps, where it doubles as a demonstration of tone. The tag style is easier to read and edit. Either works; pick one and stay consistent.

**Check yourself:** Your few-shot classifier keeps labeling everything `negative`. You look at the prompt and see four negative examples and one positive. What do you try first?

<details><summary>Answer</summary>

Balance the examples (add positive and mixed ones). Lopsided examples bias the model toward the majority label.

</details>

## When few-shot isn't the answer

- **The task needs knowledge the model doesn't have.** Examples teach format and judgment, not facts. Provide the facts as context (or use [RAG](../07-rag/01-why-rag.md)).
- **You need guaranteed structure.** Examples make a format likely; [structured output](../02-calling-models/04-structured-output.md) makes it certain.
- **You have hundreds of good examples.** At that point, fine-tuning or retrieving the most similar examples per request (dynamic few-shot) can beat stuffing them all in.

> [!TIP]
> **Try it.** Pick a classification task from your own work (tag commit messages as `feat`, `fix`, `docs`, `chore`). Write it zero-shot and run it on 10 real inputs. Note which ones it gets wrong. Then add three examples that target those exact mistakes and rerun.
>
> <details><summary>Hint</summary>
>
> Keep a small list of the 10 inputs and expected labels in a Python list so you can rerun with one command. You'll turn this into a proper habit in [Prompts as code](04-prompts-as-code.md).
>
> </details>

## Key takeaways
- Zero-shot tells; few-shot shows. Start zero-shot and add examples when instructions fall short.
- Examples are best at teaching format, style, and judgment calls that are hard to describe.
- Good example sets cover tricky boundaries, vary in content, and are balanced and consistent.
- Models copy examples closely, including the parts you didn't mean to teach.
- Examples don't add knowledge, and they don't guarantee structure.

## Go deeper
- [Language Models are Few-Shot Learners (GPT-3 paper)](https://arxiv.org/abs/2005.14165) — the paper that showed in-context examples could replace task-specific training
- [Claude prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) — includes guidance on using examples effectively
- [Prompt Engineering Guide](https://www.promptingguide.ai/) — see the zero-shot and few-shot technique pages

---
[← Anatomy of a prompt](01-anatomy-of-a-prompt.md) · [Module home](README.md) · [Reasoning patterns: chain of thought and ReAct →](03-reasoning-patterns.md)
