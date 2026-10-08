# Hugging Face

> Module 3 · Lesson 3 · ~15 min

**Goals**
- Find your way around the Hugging Face Hub: models, datasets, Spaces and model cards
- Use Hugging Face Tasks to find the right kind of model for a problem
- Run a model in Python with `transformers`, and in the browser with Transformers.js
- Call hosted models for non-chat tasks with the `InferenceClient`

## The GitHub of models

[Hugging Face](https://huggingface.co) is where open models live. Nearly every open-weight release (Llama, Qwen, Gemma, DeepSeek, thousands of fine-tunes and small task models) is published on its **Hub**. Think of it as GitHub for machine learning:

- **Models**: each one is a Git repo holding weights, config, tokenizer and a **model card** (the README). The card tells you what the model is for, how it was trained, its license and its limits. Read it before you use the model.
- **Datasets**: training and evaluation data, browsable in the page.
- **Spaces**: small hosted demo apps. Great for trying a model before downloading anything.

Each model has an ID of the form `org/name`, like `Qwen/Qwen3-8B`. That one string works in `transformers`, in Inference Providers, and (for GGUF repos) in Ollama and LM Studio.

## Tasks: not everything needs an LLM

Hugging Face organizes models by **task**: text classification, summarization, translation, speech recognition, image classification, feature extraction (embeddings) and many more. The [Tasks page](https://huggingface.co/tasks) explains each one and lists good models for it.

This matters because a lot of problems don't need a general LLM. A 67-million-parameter sentiment classifier is thousands of times cheaper and faster than a frontier model, runs on a CPU, and gives the same answer every time. Reach for a specialist model when the task is narrow and well defined, and for an LLM when it needs open-ended language or reasoning.

You can also search programmatically:

```python
from huggingface_hub import list_models

for m in list_models(pipeline_tag="summarization", sort="downloads", limit=5):
    print(m.id, m.downloads)
```

## Running a model with `transformers`

The `transformers` library downloads a model from the Hub and runs it on your machine. Its `pipeline` function wraps a whole task in one call:

```python
# uv add transformers torch
from transformers import pipeline

classifier = pipeline("sentiment-analysis",
                      model="distilbert/distilbert-base-uncased-finetuned-sst-2-english")
print(classifier("This course finally made function calling click for me."))
# [{'label': 'POSITIVE', 'score': 0.99...}]

chat = pipeline("text-generation", model="Qwen/Qwen2.5-0.5B-Instruct")
messages = [{"role": "user", "content": "Explain quantization in one sentence."}]
out = chat(messages, max_new_tokens=80)
print(out[0]["generated_text"][-1]["content"])
```

The first run downloads the weights into a local cache (`~/.cache/huggingface`). After that it works offline. Both models are small enough to run on a CPU. For bigger ones, the quick tour linked below shows how to put the model on a GPU.

When should you use `transformers` instead of Ollama? Use Ollama for chatting with LLMs through an API. Use `transformers` when you need non-chat tasks, models Ollama doesn't package, access to the model's internals, or fine-tuning.

## In the browser with Transformers.js

[Transformers.js](https://huggingface.co/docs/transformers.js/index) runs many of the same models directly in JavaScript, in the browser or Node, with no server at all:

```javascript
// npm install @huggingface/transformers
import { pipeline } from "@huggingface/transformers";

const classify = await pipeline(
  "sentiment-analysis",
  "Xenova/distilbert-base-uncased-finetuned-sst-2-english",
  { dtype: "q8" }  // a quantized build, smaller to download
);
console.log(await classify("Running models in the browser is wild."));
```

Pass `device: "webgpu"` to use the GPU where the browser supports it. The data never leaves the user's device and you pay nothing for inference, which suits small models: classification, embeddings for search, speech recognition, or a tiny LLM. The costs are download size (users fetch the weights once) and the speed of their hardware.

## Hosted inference with the `InferenceClient`

In [Module 2](../02-calling-models/02-openai-compatible-apis.md) you used Inference Providers for chat. The same `InferenceClient` covers other tasks too, with a method per task:

```python
from huggingface_hub import InferenceClient

client = InferenceClient()  # reads HF_TOKEN
result = client.text_classification(
    "The checkout page keeps timing out.",
    model="distilbert/distilbert-base-uncased-finetuned-sst-2-english",
)
print(result[0].label, round(result[0].score, 3))
```

There are methods for `summarization`, `translation`, `feature_extraction` (embeddings, used in [Module 6](../06-embeddings-and-vector-search/README.md)), `automatic_speech_recognition`, `text_to_image` and more. A model page's "Inference Providers" panel shows whether it's hosted and by whom.

> [!TIP]
> **Try it**
> 1. Go to [huggingface.co/tasks](https://huggingface.co/tasks), pick "Zero-Shot Classification", and open one of the suggested models.
> 2. On its page, try the inference widget with a sentence and your own labels (for example "billing, bug report, feature request").
> 3. Run the same model locally with `pipeline("zero-shot-classification", model="<that id>")`, calling it with `candidate_labels=[...]`.
>
> <details><summary>Hint</summary>
>
> `facebook/bart-large-mnli` is a classic choice. Call it as `clf("The app crashes on login", candidate_labels=["billing", "bug report", "feature request"])`. The result lists the labels sorted by score. You've built a ticket router with no training and no LLM.
>
> </details>

<details><summary>Check yourself: you need to tag 5 million product reviews as positive, negative or neutral by tomorrow, on a small budget. LLM API or Hub model?</summary>

Start with a specialist text-classification model from the Hub, run locally or on a single GPU. It's fast, cheap and deterministic at that volume. Check its accuracy on a few hundred hand-labeled reviews first, and fall back to an LLM only for cases it's unsure about.

</details>

## Key takeaways
- The Hugging Face Hub hosts almost every open model, with model cards that tell you purpose, license and limits.
- Tasks help you pick the right kind of model. Many problems are cheaper and better solved by a small specialist.
- `transformers` runs Hub models locally in Python. Transformers.js runs them in the browser.
- `InferenceClient` gives you hosted inference for chat and many other tasks with one token.

## Go deeper
- [Hugging Face Hub documentation](https://huggingface.co/docs/hub/index): repos, model cards and gated models
- [Tasks](https://huggingface.co/tasks): every task explained, with recommended models
- [Transformers quick tour](https://huggingface.co/docs/transformers/quicktour): pipelines, AutoModel and generation
- [Run inference on servers](https://huggingface.co/docs/huggingface_hub/guides/inference): the `InferenceClient` guide

---
[← Running models locally](02-running-models-locally.md) · [Module home](README.md) · [Picking the right model →](04-picking-the-right-model.md)
