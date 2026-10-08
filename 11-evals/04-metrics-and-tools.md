# Metrics and tools

> Module 11 · Lesson 4 · ~20 min

**Goals**
- Pick metrics that match what your feature is supposed to do
- Measure retrieval separately from generation in a RAG app
- Run a DeepEval test with a G-Eval metric and a Claude judge
- Score faithfulness with RAGAS

## Choose metrics from the failure, not the menu

There are hundreds of named LLM metrics. Ignore most of them. Start from the question "what would a bad output from my app look like?" and pick a metric that catches it.

| Family | Examples | Use when | Watch out |
|---|---|---|---|
| Exact / overlap | exact match, F1, BLEU, ROUGE | Short answers, labels, extraction | Penalize correct paraphrases. Weak for open-ended text |
| Semantic similarity | Cosine similarity of embeddings to a reference | Rough "is this close to the reference?" | Similar ≠ correct. "Paris is the capital" and "Paris is not the capital" embed closely |
| Retrieval | hit rate, recall@k, MRR, context precision/recall | RAG: did we fetch the right chunks? | Needs labeled relevant docs per question |
| Groundedness | faithfulness, hallucination rate | RAG: does the answer stick to the retrieved context? | Usually needs an LLM judge |
| Answer quality | correctness vs reference, answer relevancy, rubric scores | Free-text answers | LLM judge, so calibrate it ([lesson 3](03-model-and-human-evals.md)) |
| Task success | tool called correctly, goal reached, steps taken | Agents | Define "success" per task |
| Operational | latency, tokens, cost per request | Always | Covered in [Module 12](../12-observability/02-cost-and-latency.md) |

> [!TIP]
> Report quality metrics next to cost and latency. "Correctness went from 84% to 88%" means something different when cost per answer also doubled.

## Split RAG into retrieval and generation

When a RAG answer is wrong, there are two very different causes: the right chunk was never retrieved, or it was retrieved and the model ignored or misread it. Measure each one on its own.

Retrieval metrics are deterministic once you know which file should be found, and your golden set already has `expected_source`:

```python
# evals/retrieval.py
from evals.helpers import GOLDEN
from course_buddy.rag import search     # returns Chunk objects with .source (see Module 7)

def retrieval_report(k: int = 5):
    cases = [c for c in GOLDEN if c["expected_source"]]
    hits, reciprocal_ranks = 0, []
    for c in cases:
        sources = [chunk.source for chunk in search(c["question"], k=k)]
        if c["expected_source"] in sources:
            hits += 1
            reciprocal_ranks.append(1 / (sources.index(c["expected_source"]) + 1))
        else:
            reciprocal_ranks.append(0)
    print(f"hit rate@{k}: {hits / len(cases):.0%}   MRR: {sum(reciprocal_ranks) / len(cases):.2f}")
```

**Hit rate@k** is the share of questions where the right file is anywhere in the top k. **MRR** (mean reciprocal rank) also rewards putting it near the top. Both are free to compute (embedding calls aside) and fast, so you can sweep chunk sizes or `k` in minutes.

Generation metrics need the retrieved context and an LLM judge. That's where the libraries help.

## DeepEval

[DeepEval](https://deepeval.com/docs/getting-started) is a pytest-style framework with built-in LLM-judged metrics. Its **G-Eval** metric turns a plain-language criterion into judge steps and a 0–1 score. It uses OpenAI by default, so point it at Claude explicitly:

```bash
uv add --dev deepeval
```

```python
# evals/test_deepeval.py
from deepeval import assert_test
from deepeval.metrics import FaithfulnessMetric, GEval
from deepeval.models import AnthropicModel
from deepeval.test_case import LLMTestCase, SingleTurnParams

from course_buddy.rag import ask

judge_model = AnthropicModel(model="claude-haiku-4-5-20251001")   # cheap judge (legacy model, see note)

correctness = GEval(
    name="Correctness",
    criteria="Determine whether the actual output is factually correct based on the expected output.",
    evaluation_params=[SingleTurnParams.INPUT, SingleTurnParams.ACTUAL_OUTPUT,
                       SingleTurnParams.EXPECTED_OUTPUT],
    model=judge_model,
    threshold=0.7,
)
faithfulness = FaithfulnessMetric(model=judge_model, threshold=0.8)

def test_rag_pipeline_answer():
    question = "What are the steps of a RAG pipeline?"
    answer = ask(question)
    case = LLMTestCase(
        input=question,
        actual_output=answer.text,
        expected_output="Chunk, embed, store, retrieve the most similar chunks, then generate.",
        retrieval_context=answer.contexts,   # the chunk texts your RAG step retrieved
    )
    assert_test(case, [correctness, faithfulness])
```

```bash
uv run deepeval test run evals/test_deepeval.py
```

`FaithfulnessMetric` extracts claims from the answer and checks each one against `retrieval_context`. A low score means the model added things the course doesn't say, even if they happen to be true. In practice you'd parametrize this over your golden set exactly like lesson 2.

> [!NOTE]
> Older DeepEval examples import `LLMTestCaseParams`. Current versions call it `SingleTurnParams` and keep the old name only as a deprecated alias. DeepEval needs a price table entry for the judge model, so use a model ID it knows or pass `cost_per_input_token` and `cost_per_output_token` to `AnthropicModel`. This lesson uses the legacy Haiku 4.5 for the DeepEval and RAGAS judges because eval libraries tend to lag behind new model releases. Their Anthropic adapters may not know `claude-haiku-5-5` yet, or may send sampling parameters that current Claude models reject. Once your library version supports it, switch to the current Haiku.

## RAGAS

[RAGAS](https://docs.ragas.io/en/stable/) focuses on RAG metrics: faithfulness, context precision and recall, answer relevancy, factual correctness. Current versions build an LLM with `llm_factory` and score one sample at a time:

```bash
uv add --dev ragas
```

```python
# evals/ragas_faithfulness.py
import asyncio

from anthropic import AsyncAnthropic
from ragas.llms import llm_factory
from ragas.metrics.collections import Faithfulness

from course_buddy.rag import ask

llm = llm_factory("claude-haiku-4-5-20251001", provider="anthropic", client=AsyncAnthropic())
scorer = Faithfulness(llm=llm)

async def main():
    question = "Why does chunk size matter?"
    answer = ask(question)
    result = await scorer.ascore(
        user_input=question,
        response=answer.text,
        retrieved_contexts=answer.contexts,
    )
    print(f"faithfulness: {result.value:.2f}")   # 1.0 = every claim supported by the context

asyncio.run(main())
```

RAGAS's API has changed a lot between versions (older tutorials use `evaluate()` with `SingleTurnSample` and `LangchainLLMWrapper`). Check the docs for the version you install.

### Library or hand-rolled?

| | Hand-rolled (lessons 2–3) | DeepEval / RAGAS |
|---|---|---|
| Setup | You write the judge prompt | Metrics ready to use |
| Transparency | You see and own every prompt | Prompts live inside the library |
| Fit | Exactly your rubric | Generic definitions, tunable |
| Best for | Product-specific criteria | Standard RAG metrics, quick baselines |

Many teams use both: a library for standard RAG metrics, plus a few custom judges for the criteria that make their product theirs.

<details><summary>Check yourself: hit rate@5 is 95%, but faithfulness is 0.6. Where's the problem?</summary>

In generation, not retrieval. The right file is almost always retrieved, but the answer makes claims the context doesn't support. Look at the prompt (does it tell the model to answer only from the context?), whether too many irrelevant chunks are diluting the good one, and a few failing answers by hand.

</details>

> [!TIP]
> **Try it:** Run `retrieval_report(k=3)` and `retrieval_report(k=8)` on your golden set. Then pick the three golden questions with the lowest DeepEval faithfulness scores and read the answer next to the retrieved chunks. Write one sentence per case on what went wrong.

## Key takeaways
- Pick metrics by asking what a bad output looks like for your feature.
- In RAG, measure retrieval (hit rate, MRR) and generation (faithfulness, correctness) separately.
- DeepEval gives pytest-style tests with judge metrics like G-Eval and Faithfulness, and it can use Claude as the judge.
- RAGAS specializes in RAG metrics. Its API moves fast, so pin versions.
- Always read quality metrics next to cost and latency.

## Go deeper
- [DeepEval: G-Eval](https://deepeval.com/docs/metrics-llm-evals) — how G-Eval turns criteria into evaluation steps.
- [DeepEval with Anthropic models](https://deepeval.com/integrations/models/anthropic) — configuring Claude as the judge.
- [RAGAS: Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) — how the score is calculated, with examples.
- [Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) — prompt-side fixes for low faithfulness.

---
[← Model and human evals](03-model-and-human-evals.md) · [Module home](README.md) · [Regression testing →](05-regression-testing.md)
