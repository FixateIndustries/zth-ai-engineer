# Module 11: Evals

Every change you make to an LLM feature (a prompt tweak, a new model, a different chunk size) can make some answers better and others quietly worse. Evals are how you find out which, before your users do. In this module you'll build a golden dataset, grade outputs with plain code, with a model acting as judge, and with humans, then wire the whole thing into CI so regressions block a merge the way a failing unit test would.

**You'll be able to:**
- Explain why "it looked fine when I tried it" isn't a test, and what to measure instead
- Write deterministic evals with pytest: string checks, schema checks, citation checks
- Build an LLM-as-judge with a rubric and a structured verdict, and check it against human labels
- Choose metrics for a task, and use DeepEval and RAGAS for common RAG metrics
- Run evals in GitHub Actions and catch regressions against a stored baseline

**Prerequisites:** [Module 2: Calling models](../02-calling-models/README.md) (structured output especially), [Module 4: Prompt engineering](../04-prompt-engineering/README.md), and [Module 7: RAG](../07-rag/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [Why evals](01-why-evals.md) | ~10 min |
| 2 | [Deterministic evals](02-deterministic-evals.md) | ~15 min |
| 3 | [Model and human evals](03-model-and-human-evals.md) | ~20 min |
| 4 | [Metrics and tools](04-metrics-and-tools.md) | ~20 min |
| 5 | [Regression testing](05-regression-testing.md) | ~15 min |
| ✓ | [Module review](review.md) | ~15 min |

## Capstone checkpoint
Course Buddy gets a test suite. You'll write a golden Q&A set about this course in `evals/golden.jsonl`, grade answers with deterministic checks (does it cite the right lesson file?) and an LLM judge (is the answer correct and grounded?), and run the suite in GitHub Actions on every pull request. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../10-multimodal/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-why-evals.md)
