# DSP AI RAG2 - Advanced Features & Best Practices

> This section covers advanced features including security, reranking, MCP server integration, multi-configuration retrieval, and production best practices.

---

## Security & Authentication

### JWT Bearer Authentication

```json
{
  "security": {
    "enabled": true,
    "type": "jwt_bearer",
    "jwt_secret_key": "your-secret-key-at-least-32-chars",
    "jwt_algorithm": "HS256",
    "jwt_issuer": "your-issuer",
    "jwt_audience": "rag-service",
    "jwt_require_exp": true,
    "jwt_require_iat": true,
    "jwt_leeway": 0
  }
}
```

### Metadata-Based Filtering

JWT tokens can include metadata filters that automatically restrict document retrieval:

```json
{
  "sub": "user123",
  "iat": 1705320000,
  "exp": 1705321800,
  "metadata_filter": {
    "department": "engineering",
    "access_level": {"$in": ["public", "internal"]},
    "classification": {"$ne": "confidential"}
  }
}
```

### Using Authentication

```bash
# Query with JWT token
curl -X POST http://localhost:8000/api/v1/query \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are our security protocols?",
    "configuration_name": "secure_docs"
  }'
```

### Filter Operators

| Operator | Description   | Example                                           |
|----------|---------------|---------------------------------------------------|
| `$eq`    | Equals        | `{ "status": {"$eq": "published"} }`          |
| `$ne`    | Not equals    | `{ "type": {"$ne": "draft"} }`               |
| `$in`    | In array      | `{ "category": {"$in": ["tech", "docs"]} }` |
| `$nin`   | Not in array  | `{ "status": {"$nin": ["archived", "deleted"]} }` |
| `$gt`    | Greater than  | `{ "year": {"$gt": 2022} }`                    |
| `$gte`   | Greater equal | `{ "score": {"$gte": 0.8} }`                  |
| `$lt`    | Less than     | `{ "size": {"$lt": 1000000} }`                |
| `$lte`   | Less equal    | `{ "priority": {"$lte": 5} }`                 |
| `$and`   | Logical AND   | `{ "$and": [{"dept": "eng"}, {"level": 2}] }` |
| `$or`    | Logical OR    | `{ "$or": [{"type": "doc"}, {"type": "pdf"}] }` |

---

## Reranking

### What is Reranking?

Reranking is a post-processing step that reorders retrieved documents using a more sophisticated model to improve relevance. It is particularly useful when:

- Initial retrieval returns many documents
- Precision is more important than recall
- You need to filter out marginally relevant results

### Reranking Configuration

```json
{
  "reranking": {
    "enabled": true,
    "model": "cohere-rerank",
    "top_n": 10,
    "score_threshold": 0.5,
    "server_url": "http://localhost:9001"
  }
}
```

### Available Rerankers

| Model             | Provider | Description                 |
|-------------------|----------|-----------------------------|
| **cohere-rerank** | Cohere   | High-quality reranking API  |
| **cross-encoder** | Local    | BERT-based cross-encoder    |
| **custom**        | Custom   | Your own reranking model    |

---

## Multi-Configuration Retrieval

### Fusion Methods

When querying multiple configurations, results can be combined using different fusion strategies.

#### Reciprocal Rank Fusion (RRF)

- **How it works:** Combines rankings from multiple sources using reciprocal rank scores.
- **Formula:** `RRF_score = Σ(1 / (k + rank))`
- **Best for:** Combining results from heterogeneous sources.

```json
{
  "configuration_names": ["config1", "config2", "config3"],
  "fusion_method": "rrf",
  "rrf_k_constant": 60
}
```

#### Simple Fusion

- **How it works:** Averages similarity scores across configurations.
- **Best for:** Combining results from similar sources.

```json
{
  "configuration_names": ["config1", "config2"],
  "fusion_method": "simple"
}
```

### Multi-Configuration Example

```bash
curl -X POST http://localhost:8000/api/v1/retrieve \
  -H "Content-Type: application/json" \
  -d '{
    "query": "What are the best practices for microservices?",
    "configuration_names": [
      "engineering_docs",
      "architecture_guides",
      "best_practices_wiki"
    ],
    "fusion_method": "rrf",
    "rrf_k_constant": 60,
    "k": 15,
    "use_reranking": true,
    "filter_after_reranking": false
  }'
```

---

## MCP Server Integration

### What is MCP?

Model Context Protocol (MCP) is a standard for connecting AI systems with external tools and data sources. RAG configurations can expose MCP servers for integration with AI assistants like Claude or Cursor.

### MCP Server Configuration

```json
{
  "mcp_server": {
    "enabled": true,
    "startup_enabled": true,
    "name": "Company Knowledge Base MCP",
    "description": "Access company documentation via MCP",
    "protocols": ["http"],
    "http_host": "localhost",
    "http_port": 8080,
    "tools": [
      {
        "type": "retrieve",
        "enabled": true,
        "name": "search_docs",
        "description": "Search company documentation",
        "max_results": 10,
        "include_metadata": true
      }
    ],
    "inherit_security": true
  }
}
```

### Connecting AI Assistants

Example Claude Desktop configuration:

```json
{
  "mcpServers": {
    "company-knowledge": {
      "command": "http",
      "args": ["http://localhost:8080/company_kb/mcp"]
    }
  }
}
```

---

## Performance Optimization

### Chunking Strategies

| Strategy        | Chunk Size | Overlap | Use Case                         |
|-----------------|------------|---------|----------------------------------|
| **Small chunks**| 200–500    | 50–100  | Precise retrieval, Q&A           |
| **Medium chunks** | 500–1500 | 100–300 | Balanced, general purpose        |
| **Large chunks**| 1500–4000  | 200–500 | Context-heavy, summarization     |

### Embedding Optimization

> **Embedding best practices**
>
> - **Batch processing** – Process embeddings in batches (32–128).
> - **Model selection:**
>   - Fast: `all-MiniLM-L6-v2` (384 dim)
>   - Balanced: `all-mpnet-base-v2` (768 dim)
>   - High quality: `text-embedding-ada-002` (1536 dim)
> - **Caching** – Cache embeddings for frequently accessed documents.
> - **Normalization** – Normalize vectors for cosine similarity.

### Query Performance

```json
{
  "retrieval_k": 5,              // Limit initial retrieval
  "similarity_threshold": 0.7,    // Filter low-quality matches
  "reranking": {
    "enabled": true,
    "top_n": 10,                 // Rerank top 10 only
    "score_threshold": 0.5        // Final quality filter
  },
  "generation": {
    "max_tokens": 1024,           // Limit response size
    "temperature": 0.3            // Lower for consistency
  }
}
```

---

## Production Deployment

### Docker Deployment

```yaml
version: '3.8'

services:
  rag-service:
    build: .
    ports:
      - "8000:8000"
    environment:
      - GROQ_API_KEY=${GROQ_API_KEY}
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=INFO
    volumes:
      - ./storage:/app/storage
    depends_on:
      - redis
      - elasticsearch

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

volumes:
  redis_data:
  es_data:
```

### Monitoring & Observability

> **Key metrics to monitor**
>
> - **Response time** – P50, P95, P99 latencies
> - **Retrieval quality** – Average similarity scores
> - **Query expansion** – Success rate, expansion time
> - **Document processing** – Upload success/failure rates
> - **Vector store** – Index size, query performance
> - **LLM usage** – Token consumption, API errors
> - **Cache hit rate** – Embedding and result caching

### Backup & Recovery

```bash
#!/bin/bash
# Backup RAG configurations and vector stores

BACKUP_DIR="/backups/rag-$(date +%Y%m%d)"
mkdir -p $BACKUP_DIR

# Backup configurations
cp -r ./storage/configurations.json $BACKUP_DIR/

# Backup FAISS indices
cp -r ./storage/faiss_* $BACKUP_DIR/

# Backup Redis (if using)
redis-cli --rdb $BACKUP_DIR/redis.rdb

# Backup Elasticsearch (if using)
curl -X PUT "localhost:9200/_snapshot/backup/snapshot_1?wait_for_completion=true"

echo "Backup completed to $BACKUP_DIR"
```

---

## Best Practices

### Document Preparation

- **Clean text** – Remove headers, footers, page numbers.
- **Structured metadata** – Add department, date, category tags.
- **Consistent formatting** – Use standard document templates.
- **Version control** – Track document versions in metadata.
- **Regular updates** – Schedule periodic document refreshes.

### Configuration Management

- **Environment variables** – Use `${VAR}` for sensitive data.
- **Configuration versioning** – Track config changes.
- **Test configurations** – Maintain dev/staging/prod configs.
- **Documentation** – Document each configuration's purpose.
- **Validation** – Test configs before production deployment.

### Security Best Practices

- **Enable authentication** – Always use JWT in production.
- **Rotate secrets** – Maintain a regular key rotation schedule.
- **Audit logging** – Log all queries and access.
- **Rate limiting** – Implement per-user rate limits.
- **Data classification** – Use metadata filters for access control.

---

## Troubleshooting

### Poor Retrieval Quality

- **Problem:** Retrieved documents aren't relevant.
- **Solutions:**
  - Adjust chunk size and overlap.
  - Try different embedding models.
  - Enable query expansion.
  - Lower similarity threshold.
  - Enable reranking.

### Slow Query Performance

- **Problem:** Queries take too long.
- **Solutions:**
  - Reduce `retrieval_k` value.
  - Use a faster embedding model.
  - Enable result caching.
  - Optimize vector store indices.
  - Use BM25 for keyword search.

### Memory Issues

- **Problem:** Out-of-memory errors.
- **Solutions:**
  - Reduce embedding batch size.
  - Use smaller embedding models.
  - Implement document streaming.
  - Use external vector stores (Redis, Elasticsearch).
  - Limit concurrent requests.

---

## Related Documentation

- **RAG Service Documentation Part 1** – Overview and setup
- **RAG Service Documentation Part 2** – Query operations and API
- **Integration Guide** – Complete platform integration

> For additional support:
>
> - Check the API docs at `http://localhost:8000/docs`.
> - Review example scripts in the repository.
> - Contact the DSP AI Platform team.
