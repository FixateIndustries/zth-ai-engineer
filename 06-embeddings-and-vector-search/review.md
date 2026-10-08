# Module 6 review: Embeddings and Vector Search

Test yourself before moving on. Try to answer before opening each answer.

## Questions

### 1. What does an embedding model return for one input text?
a) The next most likely token
b) A fixed-length list of numbers
c) A list of keywords
d) A variable-length vector, one number per word

<details><summary>Answer</summary>

**b)** One fixed-length vector per input, regardless of how long the text is (up to the model's max input). See [What embeddings are](01-what-embeddings-are.md).

</details>

### 2. Two vectors are already normalized to length 1. What's the fastest way to compute their cosine similarity?

<details><summary>Answer</summary>

Their dot product. Cosine similarity is the dot product divided by the product of the lengths, and both lengths are 1. That's why OpenAI and Voyage vectors are often compared with `np.dot`. See [What embeddings are](01-what-embeddings-are.md).

</details>

### 3. What's wrong with this plan?
"We indexed our docs with `text-embedding-3-small` last year. To save money, we'll now embed user queries with a local `all-MiniLM-L6-v2` model and search the same index."

<details><summary>Answer</summary>

Vectors from different models live in unrelated spaces (they don't even have the same number of dimensions here: 1536 vs 384). Queries must be embedded with the same model as the documents. Switching models means re-embedding the whole corpus. See [Embedding models](02-embedding-models.md).

</details>

### 4. Your legal team says customer contracts may not be sent to any third-party API. Name two embedding options that still work.

<details><summary>Answer</summary>

Any locally run open-weight model: Sentence Transformers models like `all-MiniLM-L6-v2` or `BAAI/bge-small-en-v1.5`, open Jina weights, other Hugging Face models, or Voyage's open-weight `voyage-4-nano`. See [Embedding models](02-embedding-models.md).

</details>

### 5. Why do some embedding APIs ask for an `input_type` like `"query"` or `"document"`?

<details><summary>Answer</summary>

Short questions and the long passages that answer them are phrased differently. Retrieval-tuned models embed each side slightly differently so a query lands near its answer. Using the right type usually improves retrieval quality. See [Embedding models](02-embedding-models.md).

</details>

### 6. You have 10,000 support tickets and five categories, with three labeled examples per category. What's the cheapest reasonable way to auto-label them?

<details><summary>Answer</summary>

Embed everything, compute a centroid per category from the labeled examples, and assign each ticket to the nearest centroid. Flag tickets with low similarity to every centroid for a human or an LLM. See [What embeddings are good for](03-what-embeddings-are-good-for.md).

</details>

### 7. You already run Postgres and have 200,000 product descriptions to search. Which vector store would you try first, and why?

<details><summary>Answer</summary>

pgvector. It keeps vectors next to your relational data with SQL joins and transactions, and adds no new system to operate. 200K vectors is well within its comfort zone. Move to a dedicated vector DB only if scale or features demand it. See [Vector databases](04-vector-databases.md).

</details>

### 8. Chroma returns `distances: [[0.12, 0.40]]` for a collection created with `space: "cosine"`. Which result is more similar, and what are the cosine similarities?

<details><summary>Answer</summary>

The first. Smaller distance means closer. Similarities are 1 − 0.12 = 0.88 and 1 − 0.40 = 0.60. See [Indexing and similarity search](05-indexing-and-similarity-search.md).

</details>

### 9. True or false: an HNSW index always returns the exact top-k nearest vectors.

<details><summary>Answer</summary>

False. HNSW is an *approximate* nearest-neighbor index. It trades a little recall for big speed gains. Raising `ef_search` or `max_neighbors` improves recall at a cost. See [Indexing and similarity search](05-indexing-and-similarity-search.md).

</details>

### 10. Your multi-tenant SaaS stores every customer's docs in one collection. A customer reports seeing another company's data in an answer. What's the fix?

<details><summary>Answer</summary>

Store a `tenant_id` in each record's metadata and **always** filter queries with `where={"tenant_id": current_tenant}` (or use one collection per tenant). Access control belongs in retrieval, not in the prompt. See [Indexing and similarity search](05-indexing-and-similarity-search.md).

</details>

### 11. Users search your docs for `ERR_4417` and get irrelevant results, even though a page mentions that exact code. Why might pure vector search miss it?

<details><summary>Answer</summary>

Embeddings capture meaning and topic, and they handle rare exact identifiers (error codes, SKUs, function names) poorly. Add keyword search alongside vectors (hybrid search), or use `where_document={"$contains": ...}` for exact matches. See [What embeddings are](01-what-embeddings-are.md).

</details>

## Hands-on challenge

Build a "find similar lessons" feature. Using your `course` collection from lesson 5, write `similar(lesson_path, k=3)` that returns the three lessons *most similar to a given lesson*, excluding the lesson itself. Try it on `06-embeddings-and-vector-search/01-what-embeddings-are.md`. Does it suggest lessons you'd agree with?

<details><summary>Hint</summary>

Fetch all chunks for the lesson with `collection.get(where={"source": lesson_path}, include=["embeddings"])` and average them into one vector (this is the "user profile" idea from the recommendations section). Query with that vector, ask for more results than you need (say 30), drop chunks from the same file, and keep the first three distinct `source` values. Remember that `get` doesn't return embeddings unless you ask for them in `include`.

</details>

## Capstone checkpoint
- [ ] `course_buddy/index.py` chunks every course `.md` file and stores the chunks in a persistent local Chroma collection
- [ ] Each chunk has a deterministic id and `source` (relative file path) and `module` metadata
- [ ] The collection uses cosine distance and records which embedding model built it
- [ ] `course-buddy search "..."` prints the top matches with their source files
- [ ] You've written down five test queries and whether the right lesson came back in the top 3

---
[← Last lesson](05-indexing-and-similarity-search.md) · [Course home](../README.md) · [Next module →](../07-rag/README.md)
