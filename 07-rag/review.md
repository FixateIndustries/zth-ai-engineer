# Module 7 review: Retrieval-Augmented Generation (RAG)

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. In one sentence, what does RAG do?

<details><summary>Answer</summary>

At question time it retrieves the passages from your data most relevant to the question, adds them to the prompt, and has the model generate an answer grounded in (and citing) those passages. See [Why RAG](01-why-rag.md).

</details>

### 2. Your team wants the model to "know" your 5,000-page product documentation, which changes weekly. Which approach fits best?
a) Fine-tune the model on the docs every week
b) RAG over the docs
c) Paste all 5,000 pages into every prompt
d) Rely on the model's training data

<details><summary>Answer</summary>

**b)** RAG. The corpus is too big for every prompt, changes too often for fine-tuning, and needs citations. Fine-tuning is for behavior and style, not for injecting facts. See [Why RAG](01-why-rag.md).

</details>

### 3. Name the two pipelines in a RAG system and when each runs.

<details><summary>Answer</summary>

**Indexing** (load, chunk, embed, store) runs offline: on deploy, on a schedule, or when documents change. **Querying** (embed question, retrieve, optionally rerank, build prompt, generate) runs on every request. See [The RAG pipeline](02-the-rag-pipeline.md).

</details>

### 4. Your RAG bot cites the wrong document. What do you check first?

<details><summary>Answer</summary>

The retrieval log for that question. Was the correct passage retrieved at all? If not, it's a retrieval problem (chunking, embedding, k, filters). If it was retrieved but the model cited something else, it's a generation problem (prompt, ordering, citation format). Always look at retrieved chunks before changing the prompt. See [The RAG pipeline](02-the-rag-pipeline.md#debug-retrieval-first).

</details>

### 5. Users search for the error code `ERR_4417` and the bot retrieves unrelated pages. What upgrade most directly helps?

<details><summary>Answer</summary>

Hybrid search: add keyword (BM25) retrieval alongside vector search and merge the results. Embeddings handle rare exact identifiers poorly; keyword search nails them. See [The RAG pipeline](02-the-rag-pipeline.md#common-upgrades).

</details>

### 6. What's wrong with this chunking setup for a Markdown docs site?
```python
chunks = [text[i:i + 4000] for i in range(0, len(text), 4000)]
vectors = SentenceTransformer("all-MiniLM-L6-v2").encode(chunks)
```

<details><summary>Answer</summary>

Two problems. The fixed 4,000-character cuts ignore headings, paragraphs, and code blocks and have no overlap, so ideas get split. And `all-MiniLM-L6-v2` only reads the first 256 tokens (roughly 1,000 characters), so most of each chunk is never embedded. Use smaller, structure-aware chunks within the model's limit. See [Chunking strategies](03-chunking-strategies.md).

</details>

### 7. A chunk reads: "Set it to `cosine` before you create it." Why does it retrieve poorly, and what's a cheap fix?

<details><summary>Answer</summary>

On its own it doesn't say what "it" is, so its embedding doesn't match questions about Chroma distance metrics. Prepend context before embedding: at minimum a breadcrumb like `06-embeddings.../05-indexing... > Distance metrics`, or an LLM-written situating sentence (contextual retrieval). See [Chunking strategies](03-chunking-strategies.md#contextual).

</details>

### 8. Why does the from-scratch script tell the model it may answer "I couldn't find that in the course"?

<details><summary>Answer</summary>

Without explicit permission to say it doesn't know, the model tends to fill gaps from its own training data, producing confident answers (often with plausible-looking citations) that the sources don't support. See [Build RAG from scratch](04-build-rag-from-scratch.md).

</details>

### 9. The script checks citations with a regex after generation. What does that check catch, and what doesn't it catch?

<details><summary>Answer</summary>

It catches citations to files that weren't retrieved (invented paths, or claims cited from memory). It does **not** catch a citation to a retrieved file that doesn't actually support the claim. That needs a faithfulness check, like an LLM judge or the eval metrics in [Module 11](../11-evals/README.md). Claude's search result blocks make citations structured, but you still need to check support. See [Build RAG from scratch](04-build-rag-from-scratch.md).

</details>

### 10. When does RAG *not* make sense?

<details><summary>Answer</summary>

When the whole corpus fits comfortably in the context window and is reused often (just send it, with prompt caching), when the goal is changing behavior or style rather than knowledge (prompting or fine-tuning), or when the answer needs reasoning over *all* documents at once, like "summarize every ticket this month", which retrieval of top-k chunks can't do. See [Why RAG](01-why-rag.md).

</details>

### 11. Match the tool to the situation: LangChain, LlamaIndex, Haystack, RAGFlow.
- (i) Non-engineers need to upload scanned PDFs and manage a knowledge base through a UI.
- (ii) You want an explicit, serializable pipeline where every component connection is visible and testable.
- (iii) You want the shortest path from a folder of files to a working query engine.
- (iv) You need integrations with many vector stores and model providers in a general LLM app.

<details><summary>Answer</summary>

(i) RAGFlow, (ii) Haystack, (iii) LlamaIndex, (iv) LangChain. All of them can do most of these things; these are each one's sweet spot. See [RAG frameworks](05-rag-frameworks.md).

</details>

## Hands-on challenge

Add **reranking** to your from-scratch `rag.py` and measure whether it helps.

1. Retrieve 20 candidates instead of 6.
2. Rescore them with a cross-encoder, `cross-encoder/ms-marco-MiniLM-L6-v2` from Sentence Transformers, and keep the top 6.
3. Write 10 questions where you know which lesson file should be cited. Record for each, with and without reranking, whether the expected file is among the final 6 chunks.

<details><summary>Hint</summary>

```python
from sentence_transformers import CrossEncoder

reranker = CrossEncoder("cross-encoder/ms-marco-MiniLM-L6-v2")

def rerank(question: str, chunks: list[dict], keep: int = 6) -> list[dict]:
    scores = reranker.predict([(question, c["text"]) for c in chunks])
    ranked = sorted(zip(scores, chunks), key=lambda pair: pair[0], reverse=True)
    return [c for _, c in ranked[:keep]]
```

Cross-encoder scores aren't cosine similarities, so apply your `MIN_SCORE` filter *before* reranking (on the retrieval scores), not after. Your 10 questions with expected files are a first golden set; keep them for [Module 11](../11-evals/README.md).

</details>

## Capstone checkpoint
- [ ] Course Buddy re-indexes the course with a structure-aware chunker that skips link lists and adds file/heading breadcrumbs
- [ ] Every answer comes from retrieved chunks, with citations to lesson files like `[07-rag/01-why-rag.md]`
- [ ] Off-topic questions get "I couldn't find that in the course" (via a similarity threshold and/or the prompt)
- [ ] Citations are checked against the retrieved files, and the retrieved files are logged or shown
- [ ] Retrieved text is treated as data, not instructions, in the system prompt
- [ ] You have 10+ questions with expected source files saved for Module 11

---
[← Last lesson](05-rag-frameworks.md) · [Course home](../README.md) · [Next module →](../08-agents/README.md)
