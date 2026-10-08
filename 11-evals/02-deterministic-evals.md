# Deterministic evals

> Module 11 · Lesson 2 · ~15 min

**Goals**
- Recognize which quality properties plain code can check
- Write pytest evals that run over a golden dataset
- Separate free, fast tests from tests that call a model
- Know where deterministic checks stop being useful

## Code is the cheapest grader

A deterministic eval checks an output with ordinary code. Same output in, same verdict out, every time. They're fast, free to run, and never argue with you, so use them for everything they can cover before you reach for a model judge.

More properties are checkable with code than you'd expect:

| Property | Check |
|---|---|
| Output format | Parses as JSON, validates against a Pydantic model |
| Required content | Contains certain keywords, numbers, or entities |
| Forbidden content | No "As an AI language model", no internal tool names, no leaked system prompt |
| Citations | Cites the expected source file, and every cited file actually exists |
| Length | Under N words or tokens |
| Behavior | Called the right tool, refused an out-of-scope question, stopped within K steps |
| Exact answers | Classification label matches, extracted date equals the reference |

What code can't judge well: whether a free-text answer is *correct* when it's phrased differently from the reference, whether it's helpful, and whether its tone is right. That's [lesson 3](03-model-and-human-evals.md).

## A pytest eval suite

Assume Course Buddy exposes something like `ask(question) -> Answer`, where `Answer` has `text` and `sources` (the lesson files it cited). That's the interface from the end of [Module 7's build lesson](../07-rag/04-build-rag-from-scratch.md#making-it-course-buddy). If you named things differently, adapt the import.

Add pytest and register a marker for tests that hit a real model:

```bash
uv add --dev pytest
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = ["llm: calls a real model API (slow, costs money)"]
```

Now the suite. One call per question, shared by every check through a cache, and one pytest case per golden example:

```python
# evals/test_deterministic.py
import json
from functools import cache
from pathlib import Path

import pytest

from course_buddy.rag import ask  # your M7 entry point

COURSE_ROOT = Path(__file__).parents[1] / "course"   # wherever your .md files live
GOLDEN = [json.loads(line) for line in Path("evals/golden.jsonl").read_text().splitlines() if line.strip()]
IDS = [case["id"] for case in GOLDEN]

@cache
def answer_for(question: str):
    return ask(question)          # one API call per question per test run

@pytest.mark.llm
@pytest.mark.parametrize("case", GOLDEN, ids=IDS)
def test_mentions_required_terms(case):
    text = answer_for(case["question"]).text.lower()
    missing = [term for term in case["must_include"] if term.lower() not in text]
    assert not missing, f"missing {missing} in: {text[:300]}"

@pytest.mark.llm
@pytest.mark.parametrize("case", GOLDEN, ids=IDS)
def test_cites_expected_source(case):
    sources = answer_for(case["question"]).sources
    if case["expected_source"] is None:            # out of scope: should cite nothing
        assert sources == []
    else:
        assert case["expected_source"] in sources, f"cited {sources}"

@pytest.mark.llm
@pytest.mark.parametrize("case", GOLDEN, ids=IDS)
def test_citations_exist(case):
    for src in answer_for(case["question"]).sources:
        assert (COURSE_ROOT / src).is_file(), f"hallucinated source: {src}"

@pytest.mark.llm
@pytest.mark.parametrize("case", GOLDEN, ids=IDS)
def test_no_boilerplate(case):
    text = answer_for(case["question"]).text
    assert "as an ai language model" not in text.lower()
    assert len(text.split()) < 400
```

Run it:

```bash
uv run pytest evals -m llm -v
```

Each golden example becomes a named test (`test_cites_expected_source[rag-001]`), so a failure points straight at the case. The `@cache` means four checks cost one API call per question, not four.

> [!NOTE]
> Keep pure functions (parsers, chunkers, citation extractors) under ordinary unit tests with no `llm` marker. Then `uv run pytest -m "not llm"` runs in seconds for free, and you save the paid suite for when it matters. You'll use exactly that split in CI in [lesson 5](05-regression-testing.md).

## Testing structured output

If you added a `--json` mode in [Module 2](../02-calling-models/04-structured-output.md), the schema is a free eval:

```python
from pydantic import BaseModel, Field, ValidationError

class BuddyReply(BaseModel):
    answer: str
    confidence: float = Field(ge=0, le=1)

@pytest.mark.llm
@pytest.mark.parametrize("case", GOLDEN, ids=IDS)
def test_json_mode_is_valid(case):
    raw = ask_json(case["question"])          # returns the raw JSON string
    try:
        BuddyReply.model_validate_json(raw)
    except ValidationError as e:
        pytest.fail(f"invalid JSON reply: {e}")
```

Even with constrained decoding, this catches mistakes in *your* code: a schema that drifted from the model class, or a code path that skips the structured call.

## Pass/fail per test vs pass rate per suite

Pytest is all-or-nothing per test. LLM output isn't. With 50 golden cases, a couple will fail on any given run because of sampling noise, and a suite that's always red stops being read.

Two ways to handle that:

- **Keep strict tests for hard rules.** "Never cite a file that doesn't exist" should be 100%. A failure there is a real bug.
- **Score soft properties as a rate.** For "mentions the required terms", compute a pass rate and compare it to a threshold or a baseline. Lesson 5 shows a small runner that does this.

<details><summary>Check yourself: a test asserts the answer contains "retrieval-augmented generation". The model wrote "retrieval augmented generation". What should you change?</summary>

The test, not the model. Your check is stricter than the property you care about. Normalize before comparing (lowercase, strip hyphens and punctuation), check for a shorter key term like "retriev", or accept a list of synonyms. If the property is really "explains RAG correctly", it needs a model judge instead.

</details>

> [!TIP]
> **Try it:** Add the four tests above to Course Buddy and run them against your golden set. Then break something on purpose: change your retriever to return the top 1 chunk instead of the top 5, and rerun. Which tests caught it? Which cases flipped? Revert when you're done.

<details><summary>Hint: what to expect</summary>

`test_cites_expected_source` usually catches retrieval changes first, especially for questions whose answer spans two lessons. `test_mentions_required_terms` may also drop because the model has less context to work from.

</details>

## Key takeaways
- Use code to grade everything code can grade: format, required and forbidden content, citations, length, tool calls.
- Parametrize pytest over the golden dataset so each case is a named, debuggable test.
- Cache model calls per run and mark paid tests so free tests can run on their own.
- Hard rules should pass 100%. Soft properties are better tracked as a rate against a threshold.

## Go deeper
- [How to parametrize fixtures and test functions](https://docs.pytest.org/en/stable/how-to/parametrize.html) — the pytest feature this lesson leans on.
- [Create strong empirical evaluations](https://platform.claude.com/docs/en/test-and-evaluate/develop-tests) — Anthropic's guide to eval design, with code-graded examples.

---
[← Why evals](01-why-evals.md) · [Module home](README.md) · [Model and human evals →](03-model-and-human-evals.md)
