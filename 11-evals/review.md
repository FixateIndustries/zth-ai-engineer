# Module 11 review: Evals

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. What are the four parts of an eval?

<details><summary>Answer</summary>

A dataset (inputs plus expectations), the task (your real pipeline running on each input), a grader (code, model, or human), and a score (aggregates plus the list of failures). See [Why evals](01-why-evals.md).

</details>

### 2. Which grader would you use for each property?
a) The reply parses as JSON matching a schema
b) The answer is correct, though worded differently from the reference
c) A medical answer is safe to show a patient

<details><summary>Answer</summary>

a) Code. b) An LLM judge with a rubric, calibrated against human labels. c) A human domain expert, at least for the labeled sample that calibrates any automated grader. Use the cheapest grader that can actually judge the property. See [Model and human evals](03-model-and-human-evals.md).

</details>

### 3. What's wrong with this judge prompt?

```text
Rate this answer from 1 to 10 for quality. Answer: {answer}
```

<details><summary>Answer</summary>

Lots of things. "Quality" isn't defined, so scores will be noisy. A 1–10 scale is hard to keep consistent. There's no reference or context to compare against, no request for reasoning before the score, and no structured output. The answer also isn't fenced off as data, so instructions inside it could steer the judge. Use a specific rubric, pass/fail per criterion, reasoning first, and `messages.parse` with a Pydantic verdict. See [Model and human evals](03-model-and-human-evals.md).

</details>

### 4. You compare two prompts pairwise with an LLM judge. Prompt B wins 70% of the time, but only when its output is shown second. What's going on, and what should you do?

<details><summary>Answer</summary>

Position bias. Run every comparison twice with the order swapped, and only count a win when the judge picks the same output both times. Inconsistent pairs are ties. See [Model and human evals](03-model-and-human-evals.md).

</details>

### 5. True or false: once your LLM judge agrees with humans 90% of the time, you can stop doing human evals.

<details><summary>Answer</summary>

False. Agreement can drift as your product, prompts, or the judge model change, and the 10% of disagreements might all be dangerous false passes. Keep relabeling a small sample periodically and read the disagreements. See [Model and human evals](03-model-and-human-evals.md).

</details>

### 6. Your RAG bot gives a wrong answer. Hit rate@5 for that question is 0. What do you work on first?

<details><summary>Answer</summary>

Retrieval. The right lesson never made it into the context, so the model couldn't have answered from it. Look at chunking, the embedding model, query phrasing, and `k` before touching the generation prompt. See [Metrics and tools](04-metrics-and-tools.md).

</details>

### 7. What does a faithfulness score measure, and why can a true statement still lower it?

<details><summary>Answer</summary>

It measures what share of the answer's claims are supported by the retrieved context. A true fact the model added from its own training data isn't supported by the context, so it counts against faithfulness. That's intended: in a grounded app, unsupported claims are where hallucinations hide. See [Metrics and tools](04-metrics-and-tools.md).

</details>

### 8. What's wrong with this test?

```python
@pytest.mark.parametrize("case", GOLDEN)
def test_answer(case):
    assert ask(case["question"]).text == case["reference"]
```

<details><summary>Answer</summary>

It demands an exact string match on free text, so it'll fail almost every time even when the answer is right. It also has no `llm` marker, so it would run (and spend money) with the free unit tests, and no `ids=`, so failures are hard to identify. Check required terms or sources with code, and judge correctness with a rubric. See [Deterministic evals](02-deterministic-evals.md).

</details>

### 9. Your pass rate drops from 88% to 84% on a 25-case golden set after a prompt change. Is it a regression?

<details><summary>Answer</summary>

Probably not on its own. That's a single case flipping (each case is 4 points on a 25-case set), which is well inside typical run-to-run noise. Check which cases flipped, rerun a few times, and compare against a tolerance you measured by running unchanged code repeatedly. If a hard rule failed, though, it's a regression no matter what. See [Regression testing](05-regression-testing.md).

</details>

### 10. Why does the CI workflow in this module use a `paths` filter, and what happens on PRs from forks?

<details><summary>Answer</summary>

The `paths` filter means the paid suite only runs when prompts, application code, or evals change, which saves money and time. GitHub doesn't expose repository secrets to workflows triggered by fork PRs, so the paid step can't call the API there and strangers can't spend your budget. See [Regression testing](05-regression-testing.md).

</details>

### 11. Name three ways to cut the cost of running evals.

<details><summary>Answer</summary>

Any three of: a smaller, calibrated judge model; a PR subset with a nightly full run; prompt caching for the shared rubric; the Batch API for large non-urgent runs; caching each answer so multiple checks share one call; `paths` filters so unrelated PRs skip the suite. See [Regression testing](05-regression-testing.md).

</details>

## Hands-on challenge

Add a **refusal eval** to Course Buddy. Write 8 golden cases that should be declined (off-topic requests, requests for things the course doesn't cover, one "ignore your instructions and…" attempt) and 8 near-miss cases that look off-topic but are in scope (e.g. "is Ollama free?"). Build a small judge with a `declined: bool` field, and report two numbers: how often it wrongly answers out-of-scope questions, and how often it wrongly refuses in-scope ones. Then tune your system prompt to improve one without hurting the other.

<details><summary>Hint</summary>

Refusals are partly deterministic. You can often catch them with code by checking for an empty `sources` list plus a phrase from your system prompt's decline wording. Use the judge only for the ambiguous ones. Track both error types separately. A prompt that refuses everything gets a perfect score on the first number and fails the second.

</details>

## Capstone checkpoint
- [ ] `evals/golden.jsonl` has at least 20 reviewed cases, including out-of-scope and multi-lesson questions
- [ ] Deterministic pytest checks for required terms, the expected source, and citations that exist
- [ ] An LLM judge with a written rubric and a structured `Verdict`, checked against your own labels on at least 10 cases
- [ ] Retrieval hit rate@k and MRR reported separately from answer quality
- [ ] `evals/baseline.json` committed, and `evals/compare.py` fails on regressions
- [ ] A GitHub Actions workflow runs free tests on every PR and the paid suite on relevant PRs and nightly

---
[← Last lesson](05-regression-testing.md) · [Course home](../README.md) · [Next module →](../12-observability/README.md)
