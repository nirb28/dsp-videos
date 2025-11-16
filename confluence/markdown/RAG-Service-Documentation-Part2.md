# DSP AI RAG2 - Query Operations & API

> This section covers query operations, OpenAI-compatible API, query expansion, and vector store configurations.

---

## Query Operations

### Basic Query

```bash
curl -X POST http://localhost:8000/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What is our company policy on remote work?",
    "configuration_name": "company_knowledge_base",
    "k": 5,
    "include_metadata": true
  }'
```

### Query with Expansion

```bash
curl -X POST http://localhost:8000/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "machine learning applications",
    "configuration_name": "tech_docs",
    "query_expansion": {
      "enabled": true,
      "strategy": "multi_query",
      "llm_config_name": "groq-llama3",
      "num_queries": 4,
      "include_metadata": true
    }
  }'
```

### Multi-Configuration Query

```bash
curl -X POST http://localhost:8000/api/v1/retrieve \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are best practices for API design?",
    "configuration_names": ["tech_docs", "engineering_wiki", "standards"],
    "fusion_method": "rrf",
    "k": 10,
    "use_reranking": true
  }'
```

### Query with Metadata Filtering

```bash
curl -X POST http://localhost:8000/api/v1/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "security protocols",
    "configuration_name": "company_docs",
    "filter": {
      "$and": [
        {"department": "engineering"},
        {"year": {"$gte": 2023}},
        {"classification": {"$ne": "confidential"}}
      ]
    }
  }'
```

---

## OpenAI-Compatible API

### Chat Completions Endpoint

> **Drop-in replacement**
>
> The RAG service provides an OpenAI-compatible API that works with existing OpenAI client libraries. Simply point your client to the RAG service and use configuration names as model names.

```python
from openai import OpenAI

# Point to RAG service instead of OpenAI
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-needed"  # Set if using JWT auth
)

# Use configuration name as model
response = client.chat.completions.create(
    model="company_knowledge_base",  # Your RAG configuration
    messages=[
        {"role": "user", "content": "What is our vacation policy?"}
    ]
)

print(response.choices[0].message.content)
```

### Streaming Responses

```python
# Enable streaming for real-time responses
stream = client.chat.completions.create(
    model="company_knowledge_base",
    messages=[{"role": "user", "content": "Explain our product features"}],
    stream=True
)

for chunk in stream:
    if chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="")
```

### List Available Models

```bash
# List all available configurations (models)
curl http://localhost:8000/v1/models

# Response
{
  "object": "list",
  "data": [
    {
      "id": "company_knowledge_base",
      "object": "model",
      "created": 1705320000,
      "owned_by": "organization"
    },
    {
      "id": "tech_docs",
      "object": "model",
      "created": 1705320000,
      "owned_by": "organization"
    }
  ]
}
```

---

## Query Expansion

### What is Query Expansion?

Query expansion automatically generates multiple variations or related queries to improve retrieval coverage. This helps find relevant documents even when the exact terms don't match.

### Expansion Strategies

| Strategy       | Description                                   | Use Case                               |
|----------------|-----------------------------------------------|----------------------------------------|
| **Fusion**     | Generates semantically similar variations     | Finding documents with different terms |
| **Multi-Query**| Generates related queries exploring aspects   | Comprehensive topic exploration        |

### Configuring LLM for Expansion

```bash
curl -X POST http://localhost:8000/api/v1/llm-configs \
  -H "Content-Type: application/json" \
  -d '{
    "name": "groq-llama3-expansion",
    "provider": "groq",
    "model": "llama3-70b-8192",
    "endpoint": "https://api.groq.com/openai/v1",
    "api_key": "${GROQ_API_KEY}",
    "temperature": 0.8,
    "max_tokens": 512
  }'
```

### Expansion Example

**Query:** `"machine learning applications"`

- **Fusion strategy** might generate:
  - "practical uses of machine learning"
  - "ML implementation examples"
  - "artificial intelligence applications in industry"

- **Multi-query strategy** might generate:
  - "What are the main applications of machine learning?"
  - "How is ML used in healthcare and finance?"
  - "What are emerging machine learning use cases?"

### Expansion Metadata

When `include_metadata: true` is set, the response includes detailed expansion information:

```json
{
  "query_expansion_metadata": {
    "original_query": "machine learning applications",
    "strategy": "multi_query",
    "llm_config_name": "groq-llama3-expansion",
    "llm_provider": "groq",
    "requested_num_queries": 4,
    "actual_num_queries": 4,
    "processing_time_seconds": 1.23,
    "expansion_successful": true,
    "expanded_queries": [
      "What are the main applications of machine learning?",
      "How is ML used in healthcare and finance?",
      "What are emerging machine learning use cases?",
      "Real-world machine learning implementations"
    ],
    "query_results_summary": [
      {
        "query": "machine learning applications",
        "is_original": true,
        "results_count": 5,
        "top_similarity_score": 0.95
      },
      {
        "query": "What are the main applications of machine learning?",
        "is_original": false,
        "results_count": 4,
        "top_similarity_score": 0.92
      }
    ],
    "total_unique_results": 12
  }
}
```

---

## Vector Stores

### FAISS (Facebook AI Similarity Search)

**Best for:** High-performance, in-memory similarity search.

Example configuration:

```json
{
  "vector_store": {
    "type": "faiss",
    "index_path": "./storage/faiss_index",
    "dimension": 384,
    "normalize_similarity_scores": true
  }
}
```

**Features:**

- Fast similarity search
- GPU acceleration support
- Multiple index types
- Persistent storage

### Redis Vector Store

**Best for:** Distributed, scalable vector search.

```json
{
  "vector_store": {
    "type": "redis",
    "redis_host": "localhost",
    "redis_port": 6379,
    "redis_username": null,
    "redis_password": "your_password",
    "redis_index_name": "documents"
  }
}
```

**Features:**

- Distributed storage
- Real-time updates
- Metadata filtering
- High availability

### Elasticsearch

**Best for:** Hybrid search (vector + keyword).

```json
{
  "vector_store": {
    "type": "elasticsearch",
    "es_url": "http://localhost:9200",
    "es_index_name": "rag_documents",
    "es_user": "elastic",
    "es_password": "password",
    "es_search_type": "hybrid",
    "es_fulltext_field": "content",
    "es_semantic_field": "text"
  }
}
```

**Search types:**

- `fulltext` – BM25 keyword search
- `vector` – pure vector similarity
- `semantic` – ELSER semantic search
- `hybrid` – combined vector + keyword
- `query_dsl` – custom Elasticsearch query

### Neo4j Knowledge Graph

**Best for:** Semantic relationships and graph traversal.

```json
{
  "vector_store": {
    "type": "neo4j_knowledge_graph",
    "neo4j_uri": "neo4j://localhost:7687",
    "neo4j_user": "neo4j",
    "neo4j_password": "password",
    "neo4j_database": "neo4j",
    "kg_llm_config_name": "graph-extraction-llm"
  }
}
```

**Features:**

- Entity extraction
- Relationship mapping
- Graph queries (Cypher)
- Semantic search
- Multi-hop traversal

### BM25 (Best Matching 25)

**Best for:** Keyword-based search without embeddings.

```json
{
  "vector_store": {
    "type": "bm25"
  }
}
```

**Features:**

- No embeddings required
- Fast keyword matching
- TF-IDF-style scoring
- Language-agnostic
- Low resource usage

---

## API Reference

### Core Endpoints

| Method | Endpoint                 | Description              |
|--------|--------------------------|--------------------------|
| `POST` | `/api/v1/query`          | Query with generation    |
| `POST` | `/api/v1/retrieve`       | Retrieve documents only  |
| `POST` | `/api/v1/documents`      | Upload documents         |
| `GET`  | `/api/v1/documents`      | List documents           |
| `DELETE` | `/api/v1/documents`    | Delete all documents     |

### Configuration Endpoints

| Method | Endpoint                                 | Description             |
|--------|------------------------------------------|-------------------------|
| `GET`  | `/api/v1/configurations`                 | List configurations     |
| `POST` | `/api/v1/configurations`                 | Create configuration    |
| `GET`  | `/api/v1/configurations/{name}`          | Get configuration       |
| `DELETE` | `/api/v1/configurations/{name}`        | Delete configuration    |
| `POST` | `/api/v1/configurations/duplicate`       | Duplicate configuration |

### OpenAI-Compatible Endpoints

| Method | Endpoint               | Description                        |
|--------|------------------------|------------------------------------|
| `POST` | `/v1/chat/completions` | Chat completions (OpenAI format)   |
| `GET`  | `/v1/models`           | List available models              |

---

## Related Documentation

- **RAG Service Documentation Part 1** – Overview and setup
- **RAG Service Documentation Part 3** – Advanced features and best practices
