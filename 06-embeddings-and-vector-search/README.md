# Module 6: Embeddings and Vector Search

LLMs read text, but search engines and databases need something they can compare quickly. Embeddings turn text into lists of numbers where "close together" means "similar in meaning". In this module you'll learn what those numbers are, how to choose a model that produces them, and how to store and search millions of them with a vector database. It's the foundation for RAG (next module), recommendations, deduplication, and a lot more.

**You'll be able to:**
- Explain what an embedding is and compute cosine similarity by hand
- Pick an embedding model (hosted or local) based on quality, cost, and privacy
- Use embeddings for search, classification, recommendations, and anomaly detection
- Store and query vectors in Chroma, and know when another vector database fits better
- Tune similarity search with the right distance metric, ANN index, and metadata filters

**Prerequisites:** [Module 1: How LLMs work](../01-how-llms-work/README.md) and [Module 2: Calling models](../02-calling-models/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [What embeddings are](01-what-embeddings-are.md) | ~15 min |
| 2 | [Embedding models](02-embedding-models.md) | ~15 min |
| 3 | [What embeddings are good for](03-what-embeddings-are-good-for.md) | ~15 min |
| 4 | [Vector databases](04-vector-databases.md) | ~15 min |
| 5 | [Indexing and similarity search](05-indexing-and-similarity-search.md) | ~20 min |
| ✓ | [Module review](review.md) | ~15 min |

## Capstone checkpoint
Course Buddy learns to *find* things. You'll split the course's Markdown files into chunks, embed them into a local Chroma collection, and add a `search` command that returns the most relevant passages with their file paths. Next module, those passages become the context for grounded answers. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../05-context-engineering/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-what-embeddings-are.md)
