# Module 13: Safety and Ethics

An LLM follows instructions, and it can't reliably tell your instructions apart from instructions hidden in a web page, an email, or a retrieved document. Add access to private data and tools that can send things out, and you have a security problem traditional apps don't have. In this module you'll learn the threat model for LLM apps, defend against direct and indirect prompt injection, add input and output guardrails, think about bias, fairness, and privacy, and attack your own app before someone else does.

**You'll be able to:**
- Describe the main risks of LLM apps: prompt injection, data leakage, harmful output, bias, and privacy
- Explain the "lethal trifecta" and use it to judge whether an agent design is dangerous
- Defend against indirect prompt injection in RAG and agent pipelines, and say honestly what each defense won't stop
- Add moderation, input/output constraints, and end-user IDs to a production app
- Red-team your app by hand and with tools like promptfoo and garak, then turn findings into regression tests

**Prerequisites:** [Module 7: RAG](../07-rag/README.md), [Module 8: Agents](../08-agents/README.md), and [Module 11: Evals](../11-evals/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [The threat model](01-the-threat-model.md) | ~15 min |
| 2 | [Prompt injection](02-prompt-injection.md) | ~20 min |
| 3 | [Guardrails](03-guardrails.md) | ~20 min |
| 4 | [Red teaming](04-red-teaming.md) | ~15 min |
| ✓ | [Module review](review.md) | ~15 min |

## Capstone checkpoint
Course Buddy learns to distrust its own inputs. You'll plant a prompt injection inside a course file, make sure Course Buddy treats retrieved text as data rather than instructions, add input and output guardrails (moderation, scope and length checks, citation validation), send a pseudonymous user ID with each request, and add your attacks to the eval suite. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../12-observability/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-the-threat-model.md)
