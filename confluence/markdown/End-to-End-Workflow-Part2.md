# DSP AI Platform Workflow - Part 2: Production & Advanced Topics

> This section covers production deployment, monitoring, advanced use cases, and best practices for the DSP AI Platform workflow.

---

## Production Deployment

### Docker Compose Setup

```yaml
# docker-compose.production.yml
version: '3.8'

services:
  # Control Tower
  control-tower:
    image: dsp/control-tower:latest
    ports:
      - "8000:8000"
    environment:
      - ENVIRONMENT=production
      - SUPERUSER_SECRET=${CT_SUPERUSER_SECRET}
      - DATABASE_URL=postgresql://user:pass@postgres:5432/control_tower
    volumes:
      - ./manifests:/app/manifests
      - ./policies:/app/policies
    depends_on:
      - postgres
    restart: always

  # JWT Service
  jwt-service:
    image: dsp/jwt-service:latest
    ports:
      - "5000:5000"
    environment:
      - FLASK_ENV=production
      - SECRET_KEY=${JWT_SECRET_KEY}
      - JWE_ENCRYPTION_KEY=${JWE_KEY}
      - LDAP_SERVER=${LDAP_SERVER}
    volumes:
      - ./config/users.yaml:/app/config/users.yaml
      - ./config/api_keys:/app/config/api_keys
    restart: always

  # Front Door (FD2)
  front-door:
    image: dsp/front-door:latest
    ports:
      - "8080:8080"
    environment:
      - CONTROL_TOWER_URL=http://control-tower:8000
      - JWT_SERVICE_URL=http://jwt-service:5000
      - LOG_LEVEL=INFO
    depends_on:
      - control-tower
      - jwt-service
    restart: always

  # RAG Service
  rag-service:
    image: dsp/rag-service:latest
    ports:
      - "8001:8000"
    environment:
      - GROQ_API_KEY=${GROQ_API_KEY}
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REDIS_URL=redis://redis:6379
      - ES_URL=http://elasticsearch:9200
    volumes:
      - ./storage:/app/storage
    depends_on:
      - redis
      - elasticsearch
    restart: always

  # Supporting Services
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB=control_tower
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: always

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    restart: always

  elasticsearch:
    image: elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data
    restart: always

  # Monitoring
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: always

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
    depends_on:
      - prometheus
    restart: always

volumes:
  postgres_data:
  redis_data:
  es_data:
  prometheus_data:
  grafana_data:
```

### Environment Configuration

```bash
# .env.production

# Security
CT_SUPERUSER_SECRET=your-secure-superuser-secret
JWT_SECRET_KEY=your-jwt-secret-key-min-32-chars
JWE_KEY=your-jwe-encryption-key

# API Keys
GROQ_API_KEY=gsk_your_groq_api_key
OPENAI_API_KEY=sk-your-openai-api-key

# LDAP (optional)
LDAP_SERVER=ldap://ldap.company.com:389

# Monitoring
GRAFANA_PASSWORD=secure-grafana-password

# Database
DATABASE_URL=postgresql://user:pass@postgres:5432/control_tower

# Service URLs (internal Docker network)
CONTROL_TOWER_URL=http://control-tower:8000
JWT_SERVICE_URL=http://jwt-service:5000
RAG_SERVICE_URL=http://rag-service:8000
```

### Deployment Commands

```bash
# 1. Build images (if using custom builds)
docker-compose -f docker-compose.production.yml build

# 2. Start services
docker-compose -f docker-compose.production.yml up -d

# 3. Check service health
docker-compose -f docker-compose.production.yml ps

# 4. View logs
docker-compose -f docker-compose.production.yml logs -f

# 5. Scale services
docker-compose -f docker-compose.production.yml up -d --scale rag-service=3

# 6. Update services (zero-downtime)
docker-compose -f docker-compose.production.yml pull
docker-compose -f docker-compose.production.yml up -d --no-deps service-name
```

---

## Monitoring & Observability

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'control-tower'
    static_configs:
      - targets: ['control-tower:8000']
    metrics_path: /metrics

  - job_name: 'jwt-service'
    static_configs:
      - targets: ['jwt-service:5000']
    metrics_path: /metrics

  - job_name: 'front-door'
    static_configs:
      - targets: ['front-door:8080']
    metrics_path: /metrics

  - job_name: 'rag-service'
    static_configs:
      - targets: ['rag-service:8000']
    metrics_path: /metrics

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']

  - job_name: 'elasticsearch'
    static_configs:
      - targets: ['elasticsearch:9200']
    metrics_path: /_prometheus/metrics
```

### Key Metrics to Monitor

**Service metrics to track:**

- **Control Tower**
  - Manifest operations/sec
  - Policy evaluation time
  - Module deployment success rate
- **JWT Service**
  - Token generation rate
  - Authentication failures
  - Token validation latency
- **RAG Service**
  - Query response time
  - Retrieval accuracy / similarity scores
  - Embedding generation rate

Alert on, for example:

- P95 latency > 500ms
- Error rate > 1%
- Query time > 2s
- Retrieval score < 0.7
- Memory/CPU usage > 80%

---

## Advanced Use Cases

### Multi-Model Ensemble

```json
{
  "name": "ensemble-inference",
  "type": "inference_endpoint",
  "config": {
    "strategy": "weighted_ensemble",
    "models": [
      {"name": "gpt-4", "weight": 0.6},
      {"name": "claude-3", "weight": 0.4}
    ],
    "aggregation": "vote_and_score",
    "max_tokens": 1024,
    "temperature": 0.3
  }
}
```

Use cases:

- Higher answer robustness.
- Cross-model agreement scoring.

### RAG with Knowledge Graph (Neo4j)

**Cypher example:**

```cypher
// Find related concepts to user query
MATCH (q:Query {text: $query_text})
MATCH (q)-[:SIMILAR_TO*1..3]-(c:Concept)
MATCH (c)-[:RELATED_TO]-(d:Document)
RETURN d.content, d.metadata
ORDER BY d.relevance_score DESC
LIMIT 10
```

**Configuration snippet:**

```json
{
  "name": "knowledge-graph-rag",
  "type": "rag_config",
  "config": {
    "vector_store": {
      "type": "neo4j_knowledge_graph",
      "neo4j_uri": "neo4j://localhost:7687",
      "neo4j_user": "neo4j",
      "neo4j_password": "${NEO4J_PASSWORD}",
      "extraction_prompt": "Extract entities and relationships...",
      "kg_llm_config_name": "entity-extraction-llm"
    },
    "retrieval_strategy": "graph_traversal",
    "max_hops": 3
  }
}
```

### Streaming Pipeline

```python
# streaming_pipeline.py
import asyncio
from typing import AsyncGenerator
import aiohttp

class StreamingPipeline:
    def __init__(self):
        self.front_door_url = "http://localhost:8080"
        
    async def process_document_stream(
        self,
        document_stream: AsyncGenerator,
    ):
        """Process documents as they arrive."""
        async with aiohttp.ClientSession() as session:
            async for document in document_stream:
                # Step 1: Extract text
                text = await self.extract_text(document)
                
                # Step 2: Chunk document
                chunks = await self.chunk_document(text)
                
                # Step 3: Generate embeddings via Front Door
                embeddings = await self.generate_embeddings(session, chunks)
                
                # Step 4: Store in vector DB
                await self.store_vectors(embeddings)
                
                # Step 5: Trigger notifications
                await self.notify_completion(document.id)
                
                yield {
                    "document_id": document.id,
                    "status": "processed",
                    "chunks": len(chunks)
                }
    
    async def generate_embeddings(
        self,
        session: aiohttp.ClientSession,
        chunks: list,
    ):
        """Generate embeddings via Front Door."""
        async with session.post(
            f"{self.front_door_url}/api/embeddings",
            json={"texts": chunks},
        ) as response:
            return await response.json()
```

---

## Best Practices

### Prompt Engineering Guidelines

- **Be specific** – clear instructions reduce ambiguity.
- **Use examples** – few-shot prompts often perform better.
- **Structure outputs** – request JSON or structured formats.
- **Test extensively** – use promptfoo for regression tests.
- **Version prompts** – track changes in Git.
- **Monitor quality** – tie prompts to quality metrics.

### Manifest Management

- Use **templates** from the Manifestor CLI.
- Use **environment variables** for all secrets.
- Maintain clear **naming conventions** (`{project}-{module}-{type}`).
- Add **dependencies** explicitly.
- Tag manifests with **versions**.

### Security Considerations

Security checklist for production:

- Enable JWT authentication.
- Use JWE encryption for sensitive data.
- Rotate API keys regularly.
- Implement rate limiting.
- Use HTTPS/TLS for all endpoints.
- Audit log all API access.
- Implement RBAC with metadata filters.
- Run regular security scans.

---

## Troubleshooting Guide

Common issues and mitigations:

- **JWT token invalid (401)**
  - Check expiration, secret key, and algorithm.
- **Manifest not loading**
  - Validate JSON, ensure Control Tower is healthy, and check permissions.
- **RAG poor results**
  - Tune chunk size/overlap, try different embedding models, enable reranking and query expansion.
- **High latency**
  - Enable caching, reduce retrieval `k`, use faster models, or scale horizontally.

---

## Related Documentation

- **End-to-End Workflow Guide Part 1 & 1B** – Basic workflow and manifest generation.
- **Control Tower Documentation**
- **Front Door Documentation**
- **JWT Service Documentation**
- **RAG Service Documentation Part 1–3**

> After completing this workflow, you should set up dashboards in Grafana, configure alerting rules, implement CI/CD pipelines, schedule regular prompt testing, and document your specific use cases.
