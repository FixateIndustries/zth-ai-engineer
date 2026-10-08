# RAG frameworks

> Module 7 · Lesson 5 · ~15 min

**Goals**
- See the same RAG app in LangChain, LlamaIndex, and Haystack
- Know what RAGFlow is and how it differs from the code libraries
- Decide when a framework saves you time and when plain SDK code is better

## What a framework gives you

In the last lesson you wrote loading, chunking, embedding, storage, retrieval, and prompting yourself. Frameworks package those as interchangeable components:

- **Loaders** for PDFs, web pages, Notion, Google Drive, databases, and hundreds of other sources
- **Splitters** (recursive, Markdown, code-aware, semantic)
- **Integrations** with dozens of embedding models, vector stores, and LLM providers behind one interface
- **Ready-made patterns:** hybrid search, reranking, query rewriting, parent-child retrieval, agents
- **Tracing hooks** that plug into observability tools ([Module 12](../12-observability/README.md))

The cost: another abstraction layer to learn and debug, APIs that change between versions, and defaults (prompts, chunk sizes) you didn't choose.

Each example below indexes this course and answers a question with Claude. Compare them to [your from-scratch version](04-build-rag-from-scratch.md).

## LangChain

The biggest ecosystem. LangChain is a general LLM application framework, so RAG is one use among many (its sister project LangGraph handles agents and stateful workflows).

```bash
uv add "langchain[anthropic]" langchain-chroma langchain-huggingface langchain-text-splitters
```

```python
from pathlib import Path

from langchain.chat_models import init_chat_model
from langchain_chroma import Chroma
from langchain_core.documents import Document
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

COURSE_DIR = Path("../zth-ai-engineer")

# Load + chunk
docs = [
    Document(page_content=p.read_text(encoding="utf-8"),
             metadata={"source": p.relative_to(COURSE_DIR).as_posix()})
    for p in COURSE_DIR.rglob("*.md")
]
chunks = RecursiveCharacterTextSplitter(chunk_size=800, chunk_overlap=100).split_documents(docs)

# Embed + store
store = Chroma(
    collection_name="course_lc",
    embedding_function=HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2"),
    persist_directory="./chroma_lc",
)
store.add_documents(chunks)

# Retrieve + generate
question = "When should I use RAG instead of fine-tuning?"
hits = store.similarity_search(question, k=5)
context = "\n\n".join(f"[{d.metadata['source']}]\n{d.page_content}" for d in hits)
llm = init_chat_model("anthropic:claude-sonnet-5-5")
reply = llm.invoke(
    "Answer only from these sources and cite the file in brackets.\n\n"
    f"{context}\n\nQuestion: {question}"
)
print(reply.text)
```

Notice how close this is to your own script: the framework mostly gives you swappable parts. Changing `Chroma` to `PGVector` or `HuggingFaceEmbeddings` to `OpenAIEmbeddings` is a one-line change. LangChain's current docs also show an agentic variant, where the model gets a retrieval tool and decides when to search.

## LlamaIndex

Built specifically around connecting LLMs to your data. Its defaults get you from folder to answers in very few lines, and it goes deep on indexing and retrieval strategies.

```bash
uv add llama-index-core llama-index-llms-anthropic llama-index-embeddings-huggingface
```

```python
from llama_index.core import Settings, SimpleDirectoryReader, VectorStoreIndex
from llama_index.embeddings.huggingface import HuggingFaceEmbedding
from llama_index.llms.anthropic import Anthropic

Settings.llm = Anthropic(model="claude-sonnet-5-5", max_tokens=4096)
Settings.embed_model = HuggingFaceEmbedding(model_name="sentence-transformers/all-MiniLM-L6-v2")

docs = SimpleDirectoryReader("../zth-ai-engineer", recursive=True, required_exts=[".md"]).load_data()
index = VectorStoreIndex.from_documents(docs)  # chunks + embeds, in memory by default

response = index.as_query_engine(similarity_top_k=5).query(
    "When should I use RAG instead of fine-tuning?"
)
print(response)
for node in response.source_nodes:
    print(f"{node.score:.2f}  {node.metadata['file_path']}")
```

Five lines of real work. The trade-off is that chunking, the prompt template, and how sources are formatted are all defaults you'd need to dig into to change. (Set `max_tokens` explicitly: the integration's default is small.) For persistence you'd plug in a vector store such as Chroma via `llama-index-vector-stores-chroma`.

## Haystack

From deepset. You build explicit **pipelines** of typed components wired together, which makes data flow easy to see, test, and serialize. It's popular for production search systems. This example uses BM25 keyword retrieval, a nice contrast with vector search:

```bash
uv add haystack-ai anthropic-haystack
```

```python
from pathlib import Path

from haystack import Document, Pipeline
from haystack.components.builders import ChatPromptBuilder
from haystack.components.preprocessors import DocumentSplitter
from haystack.components.retrievers.in_memory import InMemoryBM25Retriever
from haystack.dataclasses import ChatMessage
from haystack.document_stores.in_memory import InMemoryDocumentStore
from haystack_integrations.components.generators.anthropic import AnthropicChatGenerator

# Index: load, split, write (a BM25 keyword index, no embeddings needed)
store = InMemoryDocumentStore()
docs = [Document(content=p.read_text(encoding="utf-8"), meta={"source": str(p)})
        for p in Path("../zth-ai-engineer").rglob("*.md")]
splitter = DocumentSplitter(split_by="word", split_length=200, split_overlap=20)
store.write_documents(splitter.run(documents=docs)["documents"])

# Query pipeline: retriever -> prompt builder -> Claude
template = [
    ChatMessage.from_system("Answer only from the documents. Cite the source path in brackets."),
    ChatMessage.from_user(
        "{% for doc in documents %}[{{ doc.meta.source }}]\n{{ doc.content }}\n\n{% endfor %}"
        "Question: {{ question }}"
    ),
]
rag = Pipeline()
rag.add_component("retriever", InMemoryBM25Retriever(document_store=store, top_k=5))
rag.add_component("prompt", ChatPromptBuilder(template=template, required_variables="*"))
rag.add_component("llm", AnthropicChatGenerator(model="claude-sonnet-5-5",
                                                generation_kwargs={"max_tokens": 4096}))
rag.connect("retriever", "prompt.documents")
rag.connect("prompt", "llm")

question = "When should I use RAG instead of fine-tuning?"
result = rag.run({"retriever": {"query": question}, "prompt": {"question": question}})
print(result["llm"]["replies"][0].text)
```

More verbose, but every connection is explicit. Swapping in an embedding retriever, or adding a reranker between `retriever` and `prompt`, is adding one component and one `connect`.

## RAGFlow

RAGFlow (from InfiniFlow, Apache 2.0) is a different kind of tool: not a library but a **RAG application you deploy**, with a web UI, an HTTP API, and a Python SDK (`ragflow-sdk`). You run it with Docker Compose (it wants a few CPU cores and roughly 16 GB of RAM), upload documents, and get chat assistants and APIs on top.

Its focus is the hardest part of real-world RAG: messy documents. It does layout-aware parsing of PDFs, scans (OCR), tables, slides, and spreadsheets, offers chunking templates per document type, lets you inspect and fix chunks in the UI, and returns answers with traceable citations. It also includes agent workflows.

Consider it when non-engineers need to manage knowledge bases, or your documents are complex PDFs and you'd rather not build parsing yourself. Skip it if you need RAG as a small piece inside your own codebase.

## Which should you use?

| | From scratch | LangChain | LlamaIndex | Haystack | RAGFlow |
|---|---|---|---|---|---|
| What it is | Your code + SDKs | General LLM app framework | Data/RAG framework | Pipeline framework | Deployable RAG app |
| Lines to first answer | ~100 | ~25 | ~10 | ~30 | 0 (UI) |
| Control | Total | High | Medium | High | Low |
| Strength | Clarity, no lock-in | Integrations, ecosystem | Indexing and retrieval depth | Explicit, testable pipelines | Document parsing, UI |
| Watch out for | You build every upgrade | Abstraction churn | Hidden defaults | Verbosity | Running another service |

A pragmatic path:

1. **Start from scratch** (as you did) while you're learning and while the app is simple. You'll understand every failure.
2. **Pull in pieces, not whole frameworks.** Using a framework's PDF loader or text splitter inside your own pipeline is perfectly fine.
3. **Adopt a framework** when you need many data sources, several vector stores, or advanced retrieval you'd otherwise write yourself, and when your team is willing to track its releases.
4. **Use RAGFlow** (or a managed RAG service) when the product is "a searchable knowledge base" more than "a feature in our app".

Whatever you pick, the fundamentals don't change: chunk well, retrieve and log, ground the prompt, verify citations, and measure.

> [!TIP]
> **Try it:** Pick one framework and run its example against the course. Ask the same three questions you used in the previous lesson and compare: which files were retrieved, and how good are the answers? Then find where that framework sets its chunk size and its answer prompt.
>
> <details><summary>Where to look</summary>
>
> LangChain: you set both explicitly above (`RecursiveCharacterTextSplitter` and the `invoke` string). LlamaIndex: chunking is `Settings.node_parser` / `Settings.chunk_size`, and the prompt is the query engine's `text_qa_template` (see `query_engine.get_prompts()`). Haystack: `DocumentSplitter` and `template`, both explicit in the pipeline.
>
> </details>

## Key takeaways
- Frameworks bundle loaders, splitters, integrations, and advanced retrieval patterns behind swappable interfaces.
- LangChain has the broadest ecosystem, LlamaIndex the fastest path from data to answers, Haystack the most explicit pipelines.
- RAGFlow is a deployable RAG application with strong document parsing, not a library.
- Learn RAG without a framework, then adopt one (or just pieces of one) when it clearly saves work.

## Go deeper
- [LangChain: RAG tutorial](https://docs.langchain.com/oss/python/langchain/rag) — the official LangChain RAG walkthrough, including the agentic variant
- [LlamaIndex starter example](https://developers.llamaindex.ai/python/framework/getting_started/starter_example/) — the minimal LlamaIndex RAG app
- [Haystack: Get started](https://docs.haystack.deepset.ai/docs/get-started) — first Haystack RAG pipeline
- [RAGFlow on GitHub](https://github.com/infiniflow/ragflow) — features, architecture, and Docker setup

---
[← Build RAG from scratch](04-build-rag-from-scratch.md) · [Module home](README.md) · [Module review →](review.md)
