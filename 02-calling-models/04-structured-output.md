# Structured output

> Module 2 · Lesson 4 · ~15 min

**Goals**
- Explain the difference between asking for JSON, JSON mode, and schema-constrained output
- Get Pydantic objects back from Claude, OpenAI and Gemini
- Write a portable fallback that validates and retries on any model
- Design schemas that models fill in well

## Why "just ask for JSON" isn't enough

Your code can't do much with a friendly paragraph. To route a ticket, fill a form or render a card, you need data with a known shape. The obvious move is to put "respond in JSON" in the prompt, and it works... most of the time. Then one day the model adds "Sure! Here's the JSON:" before the brace, or forgets a field, or wraps it in a Markdown code fence, and your `json.loads` throws in production.

There are three levels of control:

| Level | How | Guarantee |
|---|---|---|
| Ask nicely | Describe the format in the prompt | None. Usually valid, sometimes not |
| JSON mode | A flag that forces syntactically valid JSON | Valid JSON, but any shape |
| Structured output | You send a JSON Schema, and the provider constrains generation to match it | Valid JSON that matches your schema |

Structured output works by **constrained decoding**: at each step, the server only allows tokens that keep the output valid against your schema. The model literally can't produce a missing brace or a misspelled key.

## One schema, written in Pydantic

Define the shape once as a Pydantic model. It gives you the JSON Schema to send, and a validator for what comes back. Here's Course Buddy's answer format:

```python
from typing import Literal
from pydantic import BaseModel, Field

class BuddyAnswer(BaseModel):
    answer: str = Field(description="The answer, in at most three sentences")
    confidence: Literal["low", "medium", "high"] = Field(
        description="How well the answer is supported by what you know"
    )
```

## Claude

`messages.parse()` takes the Pydantic class, sends its schema, and validates the result:

```python
import anthropic

client = anthropic.Anthropic()
response = client.messages.parse(
    model="claude-sonnet-5-5",
    max_tokens=16000,
    messages=[{"role": "user", "content": "What does a context window limit?"}],
    output_format=BuddyAnswer,
)
result = response.parsed_output  # a BuddyAnswer instance
print(result.confidence, "-", result.answer)
```

Without Pydantic, pass a raw schema with `output_config={"format": {"type": "json_schema", "schema": {...}}}` on `messages.create()` and `json.loads` the text block yourself.

## OpenAI

```python
from openai import OpenAI

client = OpenAI()
response = client.responses.parse(
    model="gpt-6.1-sol",
    input="What does a context window limit?",
    text_format=BuddyAnswer,
)
result = response.output_parsed
```

The raw form is `text={"format": {"type": "json_schema", "name": "buddy_answer", "schema": {...}, "strict": True}}`. Note that OpenAI's strict mode requires every property to be listed in `required` and `additionalProperties: false`.

## Gemini

```python
from google import genai

client = genai.Client()
response = client.models.generate_content(
    model="gemini-3.8-flash",
    contents="What does a context window limit?",
    config={"response_mime_type": "application/json", "response_schema": BuddyAnswer},
)
result = response.parsed  # a BuddyAnswer instance
```

## The portable fallback: validate and retry

Some models and compatible servers don't support schemas, or support them partly (you saw silent ignoring in [lesson 2](02-openai-compatible-apis.md)). For those, combine a clear prompt with validation and one retry that shows the model its mistake:

```python
import json
from pydantic import ValidationError

SCHEMA = json.dumps(BuddyAnswer.model_json_schema())

def ask_structured(chat, question: str, retries: int = 1) -> BuddyAnswer:
    """chat(messages) -> str is any function that calls a model and returns text."""
    messages = [{"role": "user", "content":
                 f"{question}\n\nReply with only a JSON object matching this schema:\n{SCHEMA}"}]
    for attempt in range(retries + 1):
        text = chat(messages)
        try:
            cleaned = text.strip().removeprefix("```json").removesuffix("```").strip()
            return BuddyAnswer.model_validate_json(cleaned)
        except ValidationError as err:
            if attempt == retries:
                raise
            messages += [
                {"role": "assistant", "content": text},
                {"role": "user", "content": f"That didn't validate:\n{err}\nReply with corrected JSON only."},
            ]
```

Even with native structured output, **keep validating**. It costs microseconds and catches the cases constrained decoding can't.

## What can still go wrong

- **Truncation.** If the model hits `max_tokens`, the JSON is cut off. Check the stop reason before you trust the result.
- **Refusals.** A refused request returns a refusal, not your schema. Claude signals this with `stop_reason == "refusal"`.
- **Unsupported schema features.** Providers support a subset of JSON Schema. Claude, for example, doesn't enforce `minimum`/`maximum` or string lengths, and doesn't support recursive schemas. The Python SDK's `parse()` moves such constraints into field descriptions and checks them locally, so validation still catches violations.
- **Valid but wrong.** A schema guarantees shape, not truth. `confidence: "high"` on a wrong answer is perfectly valid JSON. Measuring correctness is what [evals](../11-evals/README.md) are for.
- **Streaming.** You can't validate a half-finished object. Validate the final message.

## Designing schemas models fill well

- **Keep it flat and small.** Every field is something the model has to get right.
- **Use enums (`Literal`) for categories** instead of free strings. They're easier to branch on and harder to get wrong. Compare them case-insensitively to be safe.
- **Write field descriptions.** They're part of the prompt. "At most three sentences" in a description works like it would in instructions.
- **Put reasoning before conclusions** if you want both. Fields are generated in order, so a `reasoning` field before `answer` lets the model think first.
- **Prefer coarse confidence labels over decimals.** A model's self-reported `0.87` sounds precise but isn't calibrated. Three buckets are more honest.

> [!TIP]
> **Try it**
> Add a third field to `BuddyAnswer`: `related_topics: list[str]`. Ask a question with `messages.parse()` and print the result. Then ask the same question through the `ask_structured` fallback against a local Ollama model (use the OpenAI client from lesson 2 inside `chat`). Did the local model need the retry?
>
> <details><summary>Hint</summary>
>
> A minimal `chat` for Ollama: `lambda msgs: client.chat.completions.create(model="llama3.2", messages=msgs).choices[0].message.content`. Print `attempt` inside the loop to see whether a retry happened. Small models fail validation more often, which is exactly why the fallback exists.
>
> </details>

<details><summary>Check yourself: you switched from JSON mode to structured output, but a downstream crash still happens on rare responses. What two things should you check first?</summary>

The stop reason (a `max_tokens` cut-off or a refusal won't match your schema) and whether your schema uses features the provider doesn't enforce. Validating with Pydantic on every response turns both into clear errors at the boundary.

</details>

## Key takeaways
- Prompting for JSON is a request, JSON mode guarantees syntax, and structured output guarantees your schema.
- Define the schema once in Pydantic and use it both to constrain generation and to validate the result.
- Claude uses `messages.parse(output_format=...)`, OpenAI uses `responses.parse(text_format=...)`, and Gemini takes `response_schema` in its config.
- For models without schema support, validate and retry with the error message.
- A valid object can still be wrong, truncated or a refusal. Check the stop reason and keep validating.

## Go deeper
- [Claude structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs): supported schema features and SDK helpers
- [OpenAI structured outputs](https://developers.openai.com/api/docs/guides/structured-outputs): strict mode rules and refusals
- [Gemini structured output](https://ai.google.dev/gemini-api/docs/structured-output): schemas with the Gen AI SDK
- [Pydantic models](https://docs.pydantic.dev/latest/concepts/models/): validation, `Field` and JSON Schema generation

---
[← Streaming responses](03-streaming-responses.md) · [Module home](README.md) · [Function calling →](05-function-calling.md)
