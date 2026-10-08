# Module 0 review: Orientation

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. Which of these is the best description of an AI engineer's core work?
a) Training large language models from scratch on GPU clusters
b) Building products and workflows on top of pre-trained models
c) Labeling data for machine learning teams
d) Researching new neural network architectures

<details><summary>Answer</summary>

**b.** AI engineers start from an existing model and build the system around it: prompts, context, retrieval, tools, evals, monitoring. See [What AI engineers do](01-what-ai-engineers-do.md).

</details>

### 2. Name two things an ML engineer typically owns that an AI engineer typically doesn't.

<details><summary>Answer</summary>

Any two of: training data pipelines, model architecture choices, training runs and hyperparameter tuning, serving a custom-trained model. The roles overlap, but ML engineers center on producing the model and AI engineers on applying it. See [AI engineer vs ML engineer](01-what-ai-engineers-do.md#ai-engineer-vs-ml-engineer).

</details>

### 3. True or false: every API call you make to Claude is "training" the model.

<details><summary>Answer</summary>

**False.** API calls are *inference*: running an already-trained model. Your requests don't update the model's weights. See [The vocabulary](02-the-vocabulary.md#inference).

</details>

### 4. In one sentence, what is an embedding?

<details><summary>Answer</summary>

A vector of numbers that represents the meaning of a piece of text, so that texts with similar meaning have nearby vectors. See [Embeddings](02-the-vocabulary.md#embeddings).

</details>

### 5. Your company's product docs change weekly, and you want a bot that answers questions from them with citations. Which approach do you reach for first?
a) Fine-tune a model on the docs every week
b) RAG: retrieve relevant docs and put them in the prompt
c) Train a new LLM
d) Increase the temperature

<details><summary>Answer</summary>

**b.** RAG keeps answers grounded in the current docs and makes citations easy. Fine-tuning is slow to update and doesn't reliably recall facts. See [RAG](02-the-vocabulary.md#rag-retrieval-augmented-generation).

</details>

### 6. What separates an agent from a single LLM call?

<details><summary>Answer</summary>

An agent runs in a loop: the model chooses actions, calls tools, observes the results, and decides what to do next until the task is done. A single call is one request and one response. See [AI agents](02-the-vocabulary.md#ai-agents).

</details>

### 7. What's wrong with this code?

```python
import anthropic
client = anthropic.Anthropic(api_key="sk-ant-api03-Xk9...")
```

<details><summary>Answer</summary>

The API key is hard-coded. It will end up in git history and possibly a public repo. Put it in `.env` (listed in `.gitignore`), load it with `load_dotenv()`, and let `anthropic.Anthropic()` read `ANTHROPIC_API_KEY` from the environment. See [Set up your workbench](03-set-up-your-workbench.md#3-get-an-api-key-and-store-it-safely).

</details>

### 8. You run `uv run main.py` and the answer is cut off mid-sentence. `stop_reason` is `max_tokens`. What happened, and what do you change?

<details><summary>Answer</summary>

The model hit the output limit you set with `max_tokens` before it finished. Raise `max_tokens`, or ask for a shorter answer. Real code should check `stop_reason` rather than assume the response is complete. See [Hello, model](03-set-up-your-workbench.md#4-hello-model).

</details>

### 9. Which command adds a dependency to a uv project?
a) `pip install anthropic`
b) `uv add anthropic`
c) `uv install anthropic`
d) `python -m uv anthropic`

<details><summary>Answer</summary>

**b.** `uv add` installs the package into the project's environment and records it in `pyproject.toml` and `uv.lock`. Plain `pip install` would bypass the project's lockfile. See [Set up your workbench](03-set-up-your-workbench.md#2-create-the-course-buddy-project).

</details>

### 10. Where does the `usage` field in a response help you as an engineer?

<details><summary>Answer</summary>

It reports input and output tokens for the request, which is what you're billed on and what counts against limits. You'll log it for cost tracking in [Module 12](../12-observability/README.md). See [Hello, model](03-set-up-your-workbench.md#4-hello-model).

</details>

## Hands-on challenge

Turn `main.py` into a tiny command-line tool: `uv run main.py "your question here"` should send the question to Claude and print the answer, then print one line like `[in: 18 tok, out: 42 tok, stop: end_turn]`. If no question is given, print a usage message and exit with a non-zero status.

<details><summary>Hint</summary>

Use `sys.argv` (or `argparse`) to read the question. Join `sys.argv[1:]` with spaces so the question doesn't need quotes. Use `sys.exit(1)` for the missing-argument case. Everything you need from the response is in `response.content`, `response.usage`, and `response.stop_reason`.

</details>

## Capstone checkpoint
- [ ] `course-buddy` exists as a uv project with `anthropic` and `python-dotenv` installed
- [ ] `ANTHROPIC_API_KEY` lives in `.env`, and `.env` is in `.gitignore`
- [ ] `uv run main.py` makes a hello-world call to Claude and prints the answer and token usage

See the [capstone overview](../capstone/README.md) for the full milestone list.

---
[← Last lesson](03-set-up-your-workbench.md) · [Course home](../README.md) · [Next module →](../01-how-llms-work/README.md)
