# Module 7: Retrieval-Augmented Generation (RAG)

LLMs know a lot, but not your docs, your tickets, or anything that happened after their training cutoff. RAG fixes that by searching your own data at question time and handing the relevant pieces to the model, so it answers from sources instead of memory, and can cite them. It's the most common pattern in production AI apps. In this module you'll build one from scratch, learn where it breaks, and see when a framework is worth it.

**You'll be able to:**
- Explain what RAG is, where it fits, and how it compares to fine-tuning and long context
- Draw the full pipeline (chunk, embed, store, retrieve, generate) and name what fails at each step
- Choose a chunking strategy for your documents
- Build a working RAG app with citations using only the Anthropic SDK, Sentence Transformers, and Chroma
- Decide when LangChain, LlamaIndex, Haystack, or RAGFlow will save you time

**Prerequisites:** [Module 4: Prompt engineering](../04-prompt-engineering/README.md), [Module 5: Context engineering](../05-context-engineering/README.md), and [Module 6: Embeddings and vector search](../06-embeddings-and-vector-search/README.md)

| # | Lesson | Time |
|---|--------|------|
| 1 | [Why RAG](01-why-rag.md) | ~10 min |
| 2 | [The RAG pipeline](02-the-rag-pipeline.md) | ~15 min |
| 3 | [Chunking strategies](03-chunking-strategies.md) | ~15 min |
| 4 | [Build RAG from scratch](04-build-rag-from-scratch.md) | ~25 min |
| 5 | [RAG frameworks](05-rag-frameworks.md) | ~15 min |
| ✓ | [Module review](review.md) | ~15 min |

## Capstone checkpoint
Course Buddy stops answering from memory. It retrieves the most relevant course passages for each question, answers only from them, and cites the lesson files it used, like `07-rag/03-chunking-strategies.md`. If the course doesn't cover something, it says so. See the [capstone overview](../capstone/README.md).

---
[← Previous module](../06-embeddings-and-vector-search/README.md) · [Course home](../README.md) · [Start lesson 1 →](01-why-rag.md)
