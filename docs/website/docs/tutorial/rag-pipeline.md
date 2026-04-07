---
title: Build a RAG pipeline
description: How to load data into a vector database for retrieval-augmented generation (RAG) using dlt and LanceDB
keywords: [tutorial, rag, vector database, lancedb, embeddings, semantic search, ai, retrieval-augmented generation]
---

# Build a RAG pipeline

Retrieval-augmented generation (RAG) combines a large language model with your own data to produce grounded, accurate answers. The foundation of any RAG system is a **vector database** populated with your data and its embeddings.

In this tutorial, you use dlt to load data into [LanceDB](https://lancedb.github.io/lancedb/) — a lightweight, open-source vector database that runs locally. dlt handles schema management, embedding generation, and incremental updates so you can focus on your data, not your infrastructure.

## What you will learn

- Install and configure dlt with the LanceDB destination
- Load documents into a vector database with automatic embedding generation
- Query your data with semantic search
- Keep your vector database up to date with merge disposition and orphan removal

## Prerequisites

- Python 3.9 or higher
- A virtual environment set up (see the [installation guide](../reference/installation.md))

## 1. Install dlt with LanceDB support

Install dlt with the LanceDB extras and the `sentence-transformers` embedding provider. This combination runs entirely locally — no API keys or cloud accounts required.

```sh
pip install "dlt[lancedb]" "sentence-transformers>=2.0.0"
```

## 2. Configure the embedding model

Create a `.dlt` directory and a `secrets.toml` file to configure the embedding provider:

```sh
mkdir -p .dlt
```

Add the following to `.dlt/secrets.toml`:

```toml
[destination.lancedb]
embedding_model_provider = "sentence-transformers"
embedding_model = "all-MiniLM-L6-v2"
```

This tells dlt to use the `all-MiniLM-L6-v2` model from `sentence-transformers` to generate embeddings. The model downloads automatically on first run (about 80 MB).

:::tip
For production workloads, you can swap in a cloud-hosted embedding provider like OpenAI:

```toml
[destination.lancedb]
embedding_model_provider = "openai"
embedding_model = "text-embedding-3-small"

[destination.lancedb.credentials]
embedding_model_provider_api_key = "sk-..."
```

See the [LanceDB destination documentation](../dlt-ecosystem/destinations/lancedb.md) for the full list of supported providers.
:::

## 3. Load documents into LanceDB

Create a file called `rag_pipeline.py` with the following code:

```py
import dlt
from dlt.destinations.adapters import lancedb_adapter

# Sample knowledge base: product documentation snippets
documents = [
    {
        "doc_id": 1,
        "title": "Getting started with dlt",
        "content": "dlt is an open-source Python library for data loading. "
        "Install it with pip and create your first pipeline in minutes.",
        "category": "quickstart",
    },
    {
        "doc_id": 2,
        "title": "Incremental loading",
        "content": "dlt supports incremental loading to process only new or changed data. "
        "Use the incremental configuration to track state between pipeline runs.",
        "category": "feature",
    },
    {
        "doc_id": 3,
        "title": "Schema evolution",
        "content": "dlt automatically detects and evolves schemas as your data changes. "
        "New columns are added without manual migration.",
        "category": "feature",
    },
    {
        "doc_id": 4,
        "title": "Deploying to production",
        "content": "Deploy dlt pipelines with Airflow, GitHub Actions, or any orchestrator. "
        "dlt runs anywhere Python runs.",
        "category": "deployment",
    },
    {
        "doc_id": 5,
        "title": "REST API source",
        "content": "The REST API source loads data from any RESTful API. "
        "Configure endpoints, pagination, and authentication declaratively.",
        "category": "source",
    },
]

# Create a pipeline targeting LanceDB
pipeline = dlt.pipeline(
    pipeline_name="rag_pipeline",
    destination="lancedb",
    dataset_name="knowledge_base",
)

# Load documents with automatic embedding generation on the "content" field
info = pipeline.run(
    lancedb_adapter(
        documents,
        embed="content",
    ),
    table_name="docs",
    primary_key="doc_id",
)

print(info)
```

Run the pipeline:

```sh
python rag_pipeline.py
```

```text
Pipeline rag_pipeline completed in 2.35 seconds
1 load package(s) were loaded to destination lancedb and target dataset knowledge_base
```

The `lancedb_adapter` tells dlt which fields to generate embeddings for. In this case, the `content` field is embedded into a vector column that LanceDB uses for similarity search.

## 4. Query with semantic search

Now that your data is loaded, query it with natural language. Add the following to a new file called `search.py`:

```py
import lancedb

db = lancedb.connect(".lancedb")
table = db.open_table("knowledge_base___docs")

results = table.search("how do I handle changing data schemas").limit(3).to_pandas()
print(results[["title", "content", "_distance"]])
```

Run the search:

```sh
python search.py
```

```text
                          title                                            content  _distance
0            Schema evolution  dlt automatically detects and evolves schemas...   0.562341
1       Incremental loading  dlt supports incremental loading to process o...   1.104523
2  Getting started with dlt  dlt is an open-source Python library for data...   1.342187
```

The query "how do I handle changing data schemas" returns the **Schema evolution** document as the closest match — even though the query and document use different words. This is the power of semantic search: it matches on *meaning*, not keywords.

:::note
The table name in LanceDB follows the format `{dataset_name}___{table_name}` with three underscores as a separator. This is how dlt namespaces tables within a dataset.
:::

## 5. Keep your vector database up to date

In a real RAG system, your data changes over time. Use dlt's **merge** write disposition to update existing documents and add new ones without duplicates.

Create a file called `update_pipeline.py`:

```py
import dlt
from dlt.destinations.adapters import lancedb_adapter

# Updated and new documents
updated_documents = [
    {
        "doc_id": 1,
        "title": "Getting started with dlt",
        "content": "dlt is an open-source Python library for data loading. "
        "Install it with pip install dlt, create your first pipeline in minutes, "
        "and load data into 20+ destinations.",
        "category": "quickstart",
    },
    {
        "doc_id": 6,
        "title": "Data contracts",
        "content": "Schema contracts let you control how dlt handles schema changes. "
        "Set contracts to freeze, discard, or evolve to match your data governance needs.",
        "category": "feature",
    },
]

pipeline = dlt.pipeline(
    pipeline_name="rag_pipeline",
    destination="lancedb",
    dataset_name="knowledge_base",
)

info = pipeline.run(
    lancedb_adapter(
        updated_documents,
        embed="content",
    ),
    table_name="docs",
    primary_key="doc_id",
    write_disposition={"disposition": "merge", "strategy": "upsert"},
)

print(info)
```

Run the update:

```sh
python update_pipeline.py
```

This upserts the data: document 1 is updated with new content (and a fresh embedding), and document 6 is added. Existing documents 2–5 remain untouched.

:::tip
When your documents are split into chunks (common in RAG systems), use `merge_key` alongside `primary_key` to enable **orphan removal**. If a parent document is updated, old chunks that no longer exist are automatically deleted:

```py
info = pipeline.run(
    lancedb_adapter(
        chunked_documents,
        embed="chunk_text",
        merge_key="doc_id",
    ),
    table_name="doc_chunks",
    primary_key=["doc_id", "chunk_id"],
    write_disposition={"disposition": "merge", "strategy": "upsert"},
)
```
:::

## What's next?

You loaded documents into a vector database, ran semantic queries, and set up incremental updates — the three pillars of a RAG data pipeline.

To go further:

- **Use a real data source.** Combine this tutorial with the [REST API tutorial](rest-api.md) to load data from any API into your vector database.
- **Try other vector databases.** dlt also supports [Qdrant](../dlt-ecosystem/destinations/qdrant.md) and [Weaviate](../dlt-ecosystem/destinations/weaviate.md) as destinations.
- **Connect to an LLM.** Use the search results as context for a language model to build a complete RAG application.
- **Explore the data.** Learn to inspect pipeline results in the [pipeline tutorial](../build-a-pipeline-tutorial.md).
- **Go deeper.** The [advanced course](advanced-course.md) covers custom sources, destinations, and data contracts.
