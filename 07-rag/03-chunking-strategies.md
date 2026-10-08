# Chunking strategies

> Module 7 · Lesson 3 · ~15 min

**Goals**
- Explain why chunk size and boundaries make or break retrieval
- Compare fixed-size, recursive, structure-aware, semantic, and contextual chunking
- Write a heading-aware Markdown chunker for the course

## Why chunking matters so much

Retrieval returns chunks, not documents. So a chunk is both the unit you *search* and the unit you *show the model*. That creates a tension:

- **Too big:** one vector has to represent many ideas, so it matches none of them sharply. Worse, embedding models have an input limit (`all-MiniLM-L6-v2` reads 256 tokens) and silently ignore the rest.
- **Too small:** the chunk matches precisely but lacks context. "Set it to `cosine` before creating it." Set *what*?

A good chunk is about one idea and makes sense on its own. Every strategy below is a different way to approximate that.

## The strategies

### Fixed-size with overlap
Cut every N characters (or tokens), with some overlap so a sentence on a boundary appears in both neighbors.

```python
def fixed_chunks(text: str, size: int = 800, overlap: int = 100) -> list[str]:
    step = size - overlap
    return [text[i : i + size] for i in range(0, len(text), step)]
```

Dead simple and predictable. But it cuts through sentences, code blocks, and tables without mercy. Fine as a baseline, rarely the best.

### Recursive splitting
Try to split on the biggest natural boundary first (blank lines between paragraphs), and only fall back to smaller ones (lines, sentences, words) when a piece is still too big. This is LangChain's `RecursiveCharacterTextSplitter`, and it's the most common default in practice because it respects paragraphs with little effort.

### Structure-aware
Use the document's own structure: Markdown headings, HTML sections, code functions and classes, PDF layout. Each section becomes a chunk (split further if it's too long). For well-structured docs like this course, this is usually the best simple option, because authors already grouped related ideas under headings.

### Semantic
Embed each sentence and start a new chunk where the meaning shifts (when similarity between neighboring sentences drops). Nice in theory for unstructured text like transcripts. More compute, and in practice often no better than recursive splitting. Try it only if an eval says boundaries are your problem.

### Contextual
Prepend context to each chunk *before embedding it*, so it stands on its own. The cheap version is a breadcrumb: `file > section heading`. The richer version, which Anthropic calls **contextual retrieval**, asks an LLM to write a sentence or two situating the chunk within the whole document ("This section of the Q2 report covers EMEA revenue..."). Anthropic reported this substantially reduced retrieval failures, especially combined with BM25 and reranking. With prompt caching, the per-chunk LLM cost is small.

### Small-to-big (parent-child)
Search over small, precise chunks, but send the model the larger parent section they came from. You get sharp matching *and* full context. Easy to add once you have section metadata.

<details><summary>Check yourself: you're indexing a codebase. Which strategy would you start with?</summary>

Structure-aware: split by function and class (using a language-aware splitter or a parser), with the file path and class name as context. Fixed-size cuts would split functions in half, and a half-function rarely answers anything.

</details>

## How big?

There's no universal answer, but these defaults work for most text:

- **200–500 tokens** per chunk (roughly 800–2,000 characters of English), staying under your embedding model's input limit.
- **Little or no overlap.** Overlap (10–20%) helps fixed-size splitting, which cuts mid-sentence. With paragraph- or heading-aware splitting you often need none; Chroma's tests found 200–400 token recursive chunks with zero overlap did consistently well.
- **Keep metadata:** source path, section heading, position, and anything you'll filter on.
- **Drop noise:** navigation, footers, boilerplate, and link lists match lots of queries without answering any.

Then tune against real questions. In Chroma's chunking study, the strategies tested spread over about 6 points of recall, and results varied by corpus and embedding model, so there's no universal winner.

## A chunker for this course

The Module 6 chunker split only on `##` and ignored length. Here's a better one, tailored to these Markdown files. It:

1. Splits at `#`, `##`, and `###` headings, but not inside code blocks (where `#` is a comment).
2. Skips "Go deeper" link lists and the navigation footer.
3. Packs whole paragraphs into chunks of up to ~800 characters.
4. Prefixes every chunk with a breadcrumb (`file > heading`), a cheap form of contextual chunking that also feeds citations.

```python
import re

SKIP = {"Go deeper"}  # link lists add noise to retrieval


def split_sections(text: str) -> list[tuple[str, str]]:
    """Split Markdown into (heading, body) pairs at #, ##, ### headings, ignoring code blocks."""
    sections, heading, lines, in_code = [], "", [], False
    for line in text.splitlines():
        if line.startswith("```"):
            in_code = not in_code
        if not in_code and re.match(r"#{1,3} ", line):
            sections.append((heading, "\n".join(lines)))
            heading, lines = line.lstrip("#").strip(), []
            continue  # the heading goes in the breadcrumb, not the body
        lines.append(line)
    sections.append((heading, "\n".join(lines)))
    return [(h, body) for h, body in sections if body.strip() and h not in SKIP]


def chunk_markdown(text: str, source: str, max_chars: int = 800) -> list[str]:
    """Section-aware chunks of ~max_chars, each prefixed with a breadcrumb for context."""
    text = text.rsplit("\n---\n", 1)[0]  # drop this course's nav footer
    chunks = []
    for heading, body in split_sections(text):
        crumb = f"{source} > {heading}" if heading else source
        buf = ""
        for para in body.split("\n\n"):  # pack whole paragraphs, never split mid-paragraph
            if buf and len(buf) + len(para) > max_chars:
                chunks.append(f"{crumb}\n\n{buf.strip()}")
                buf = ""
            buf += para + "\n\n"
        if buf.strip():
            chunks.append(f"{crumb}\n\n{buf.strip()}")
    return chunks
```

A chunk from it looks like this:

```text
07-rag/01-why-rag.md > What RAG gets you

- **Fresh and private knowledge.** Update the index and answers change immediately. ...
```

Trade-offs, stated honestly: a single paragraph longer than `max_chars` (a big code block or table) becomes one oversized chunk, and the embedding model will only read its first 256 tokens. On this course that's under 10% of chunks, and the full text is still stored and sent to the LLM, so it's an acceptable start. If an eval shows misses there, switch to a model with a longer input (e.g. `BAAI/bge-small-en-v1.5` reads 512 tokens) or split long blocks further.

> [!TIP]
> **Try it:** Run both chunkers (the Module 6 one and this one) over the course and compare: number of chunks, median length, and how many exceed 256 tokens. Count tokens with the embedding model's own tokenizer: `len(model.tokenizer(chunk)["input_ids"])`. Then rebuild your index with the new chunker and re-run the five test queries you saved in Module 6. Did more of them find the right lesson in the top 3?
>
> <details><summary>What to expect</summary>
>
> The new chunker produces more, smaller chunks, and far fewer are truncated by the embedding model. "Go deeper" lists stop showing up in results, and specific questions (like "what is prompt caching") should land on the right section more often.
>
> </details>

## Key takeaways
- A chunk is both the unit of search and the unit of context, so aim for "one idea that stands on its own".
- Recursive splitting is a solid default; structure-aware splitting wins for well-structured docs and code.
- Prepending context (breadcrumbs or LLM-written summaries) makes chunks retrievable on their own.
- Stay under your embedding model's input limit, drop boilerplate, keep metadata, and tune with real questions.

## Go deeper
- [Anthropic: Introducing Contextual Retrieval](https://www.anthropic.com/news/contextual-retrieval) — LLM-written chunk context, with measured results
- [Chroma research: Evaluating chunking strategies](https://www.trychroma.com/research/evaluating-chunking) — how much chunking choices change retrieval, measured
- [Pinecone: Chunking strategies](https://www.pinecone.io/learn/chunking-strategies/) — overview of the main approaches and their trade-offs
- [LangChain text splitters](https://docs.langchain.com/oss/python/integrations/splitters) — ready-made recursive, Markdown, and code-aware splitters

---
[← The RAG pipeline](02-the-rag-pipeline.md) · [Module home](README.md) · [Build RAG from scratch →](04-build-rag-from-scratch.md)
