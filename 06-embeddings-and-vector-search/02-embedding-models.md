# Embedding models

> Module 6 · Lesson 2 · ~15 min

**Goals**
- Know the main hosted and open-weight embedding model families
- Call a hosted embeddings API (OpenAI) and a local model (Sentence Transformers) side by side
- Choose a model using dimensions, max input, cost, language, and privacy

## Two ways to get embeddings

Just like chat models, embedding models come in two flavors:

- **Hosted APIs.** Send text, get vectors back. Nothing to run, usually excellent quality, priced per token (and cheap: embedding a whole book costs cents).
- **Open-weight models.** Download from Hugging Face, run on your own CPU or GPU. Free per call, works offline, and your data never leaves the machine.

> [!NOTE]
> **Anthropic doesn't offer an embeddings model.** If you're building on Claude, Anthropic's docs point to [Voyage AI](https://platform.claude.com/docs/en/build-with-claude/embeddings) (now part of MongoDB) as a recommended provider. Any provider works with any chat model, though. Embeddings and generation are independent choices.

## The landscape

| Provider | Example models | Runs | Notes |
|---|---|---|---|
| OpenAI | `text-embedding-3-small`, `text-embedding-3-large` | API | Cheap, solid default. 1536 / 3072 dims, shortenable with `dimensions`. |
| Google Gemini | `gemini-embedding-001`, `gemini-embedding-2` | API | v2 is multimodal (text, images, audio, video in one space). |
| Cohere | `embed-v4.0` | API | Strong multilingual; requires an `input_type` (`search_document` / `search_query`). |
| Voyage AI | `voyage-4`, `voyage-4-lite`, `voyage-code-4` | API | Anthropic's suggested provider; domain models for code, law, finance. |
| Sentence Transformers | `all-MiniLM-L6-v2`, `BAAI/bge-small-en-v1.5` | Local | The standard Python library for open models. Tiny models run fine on CPU. |
| Jina AI | `jina-embeddings-v3`, `jina-embeddings-v4` | API or local | Long inputs, multilingual; check the license before commercial use. |
| Hugging Face Hub | thousands (e5, gte, nomic, Qwen3-Embedding, ...) | Local | Compare them on the MTEB leaderboard. |

Model names change fast. Treat this table as a map of the territory, and check each provider's docs for the current lineup.

## Same task, two models

Here's one embedding with a hosted model and one with a local model.

```bash
uv add openai sentence-transformers
```

```python
from openai import OpenAI
from sentence_transformers import SentenceTransformer

text = "Chroma is an open-source vector database."

# Hosted: OpenAI (reads OPENAI_API_KEY from the environment)
client = OpenAI()
resp = client.embeddings.create(model="text-embedding-3-small", input=[text])
openai_vec = resp.data[0].embedding
print(len(openai_vec), resp.usage.total_tokens)   # 1536 <tokens billed>

# Local: Sentence Transformers (no key, runs on your CPU)
local = SentenceTransformer("all-MiniLM-L6-v2")
local_vec = local.encode(text)
print(local_vec.shape)                            # (384,)
```

Pass a list to `input` / `encode` to embed many texts in one call. Batching is far faster than looping one string at a time.

For reference, the Voyage version (from Anthropic's docs) looks like this:

```python
import voyageai

vo = voyageai.Client()  # reads VOYAGE_API_KEY
docs = vo.embed(["Chroma is an open-source vector database."],
                model="voyage-4", input_type="document").embeddings
query = vo.embed(["what is chroma?"], model="voyage-4", input_type="query").embeddings[0]
```

Notice `input_type`. Many retrieval-tuned models embed **queries** and **documents** slightly differently, because a short question and a long passage that answers it aren't phrased alike. Cohere and Voyage take an `input_type` parameter, Gemini v1 a `task_type`, and Sentence Transformers has `encode_query()` / `encode_document()` for models that support it. When a model offers this, use it.

## How to choose

Work through these in order:

1. **Can the data leave your machine?** If not, you're choosing among open models. Done.
2. **Max input length.** `all-MiniLM-L6-v2` truncates at 256 tokens; OpenAI's models take ~8K; Voyage takes 32K. This bounds your chunk size.
3. **Quality on *your* data.** Leaderboards (MTEB) are a starting point, not an answer. Embed 50 real queries, check whether the right documents come back, and compare 2–3 models. You'll formalize this in [Module 11](../11-evals/README.md).
4. **Dimensions.** More dimensions can capture more nuance but cost more storage and slower search. 1536 floats is about 6 KB per vector; a million chunks is 6 GB before index overhead. Models that support shortening (OpenAI's `dimensions`, Voyage's `output_dimension`, Gemini's `output_dimensionality`) let you trade a little quality for a lot of space.
5. **Language and domain.** Multilingual content or code? Pick a model trained for it.
6. **Cost.** Usually the least important. `text-embedding-3-small` costs $0.02 per million tokens at the time of writing, so embedding this entire course costs well under a cent.

> [!WARNING]
> **You can't mix models.** Vectors from different models (or even different versions of one model) live in unrelated spaces. If you switch models, you must re-embed every document. Store the model name alongside your index so you never query it with the wrong one.

<details><summary>Check yourself: your company forbids sending customer tickets to third-party APIs. Which rows in the table are still on the menu?</summary>

The local ones: Sentence Transformers models, open Jina weights, and other Hugging Face models (or self-hosting any open-weight model behind your own endpoint). Voyage also publishes an open-weight `voyage-4-nano` you can run yourself.

</details>

> [!TIP]
> **Try it:** Embed the same three sentences from lesson 1 with a second local model, `BAAI/bge-small-en-v1.5`. Print both similarity matrices. Are the *rankings* the same even though the raw numbers differ? If you have an OpenAI key, add `text-embedding-3-small` as a third column (compute similarity with `np.dot`, since OpenAI vectors are normalized).
>
> <details><summary>Hint</summary>
>
> ```python
> for name in ["all-MiniLM-L6-v2", "BAAI/bge-small-en-v1.5"]:
>     m = SentenceTransformer(name)
>     v = m.encode(sentences)
>     print(name, m.similarity(v, v))
> ```
>
> Expect different absolute scores (bge models tend to score everything higher) but the same ordering. That's why thresholds don't transfer between models but rankings mostly do.
>
> </details>

## Key takeaways
- Hosted embedding APIs (OpenAI, Gemini, Cohere, Voyage) trade a small per-token cost for zero ops; open models (Sentence Transformers, Jina, Hugging Face) run locally and privately.
- Anthropic has no embeddings API; its docs recommend Voyage AI, but any provider pairs with Claude.
- Use query/document input types when a model supports them.
- Choose by privacy, max input length, quality on your own data, dimensions, then cost.
- Switching models means re-embedding everything.

## Go deeper
- [MTEB leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — benchmark scores for hundreds of embedding models
- [OpenAI embeddings guide](https://developers.openai.com/api/docs/guides/embeddings) — models, the `dimensions` parameter, pricing
- [Gemini API embeddings](https://ai.google.dev/gemini-api/docs/embeddings) — Gemini embedding models and task types
- [Anthropic: Embeddings with Voyage AI](https://platform.claude.com/docs/en/build-with-claude/embeddings) — Voyage models, `input_type`, quantization
- [Sentence Transformers pretrained models](https://sbert.net/docs/sentence_transformer/pretrained_models.html) — which open model to start with

---
[← What embeddings are](01-what-embeddings-are.md) · [Module home](README.md) · [What embeddings are good for →](03-what-embeddings-are-good-for.md)
