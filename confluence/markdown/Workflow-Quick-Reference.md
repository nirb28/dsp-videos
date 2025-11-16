# DSP AI Platform Workflow - Quick Reference Guide

> This quick reference provides a condensed overview of the complete DSP AI Platform workflow with ready-to-use commands and configurations.

---

## 🚀 Quick Start Workflow

```bash
# 1. Define your use case
echo "Use Case: Customer Support Bot with RAG"

# 2. Install promptfoo for testing
npm install -g promptfoo

# 3. Create and test prompts
cat > promptfoo.yaml << 'EOF'
prompts:
  - id: support_prompt
    raw: "Answer: {{query}} using Context: {{context}}"
providers:
  - id: groq:llama3-70b
tests:
  - vars:
      query: "Return policy?"
      context: "30-day returns allowed"
EOF
promptfoo eval

# 4. Generate manifest using CLI
cd dsp-ai-control-tower/examples/manifestor
python manifest_generator.py
# Select: 1 (Create new) → Add modules → Save

# 5. Start all services
docker-compose up -d

# 6. Upload manifest to Control Tower
curl -X POST http://localhost:8000/manifests \
  -H "X-Superuser-Secret: your-secret" \
  -d @manifests/your-manifest.json

# 7. Configure JWT authentication
curl -X POST http://localhost:5000/api/token \
  -d '{"username":"user","password":"pass"}'

# 8. Upload documents to RAG
curl -X POST http://localhost:8000/api/v1/documents \
  -F "file=@docs.pdf" \
  -F "configuration_name=your-config"

# 9. Test via Front Door
curl -X POST http://localhost:8080/api/chat \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query":"Your question here"}'

# 10. Monitor in Grafana
open http://localhost:3000
```

---

## 📋 Workflow Checklist

**Pre-deployment checklist**

| ✓  | Task                        | Command/Action                             |
|----|-----------------------------|--------------------------------------------|
| ☐  | Define use case requirements| Document objectives, inputs, outputs       |
| ☐  | Create and test prompts     | `promptfoo eval`                           |
| ☐  | Generate project manifest   | `python manifest_generator.py`             |
| ☐  | Configure environment vars  | `cp .env.example .env && vi .env`          |
| ☐  | Start Docker services       | `docker-compose up -d`                     |
| ☐  | Deploy manifest             | `curl -X POST /manifests`                  |
| ☐  | Configure authentication    | JWT service setup                          |
| ☐  | Upload knowledge base       | RAG document upload                        |
| ☐  | Test end-to-end flow        | Integration test script                    |
| ☐  | Set up monitoring           | Prometheus + Grafana                       |

---

## 🔧 Common Configurations

### Promptfoo Test Template

```yaml
# promptfoo-template.yaml
description: "{{YOUR_USE_CASE}}"

prompts:
  - id: v1
    file: prompts/{{prompt_v1.txt}}
  - id: v2
    file: prompts/{{prompt_v2.txt}}

providers:
  - id: groq:llama3-70b
    config:
      apiKey: ${GROQ_API_KEY}
      temperature: 0.3
  - id: openai:gpt-4
    config:
      apiKey: ${OPENAI_API_KEY}
      temperature: 0.3

tests:
  - description: "{{test_case_1}}"
    vars:
      input: "{{test_input}}"
    assert:
      - type: contains
        value: "{{expected_output}}"
      - type: llm-rubric
        value: "{{quality_criteria}}"

defaultTest:
  assert:
    - type: not-contains
      value: "error"
    - type: latency
      threshold: 2000
```

### Manifest Module Templates

```json
{
  "jwt_module": {
    "name": "{{project}}-jwt",
    "type": "jwt_config",
    "config": {
      "jwt_service_url": "${JWT_SERVICE_URL}",
      "consumer_key": "{{project}}-consumer",
      "rate_limit": 10,
      "token_expiration": 3600
    }
  },
  
  "rag_module": {
    "name": "{{project}}-rag",
    "type": "rag_config",
    "config": {
      "rag_service_url": "${RAG_SERVICE_URL}",
      "configuration_name": "{{project}}-docs",
      "vector_store": {"type": "faiss"},
      "embedding": {"model": "all-MiniLM-L6-v2"},
      "chunking": {
        "strategy": "recursive",
        "chunk_size": 1000,
        "chunk_overlap": 200
      }
    }
  },
  
  "llm_module": {
    "name": "{{project}}-llm",
    "type": "inference_endpoint",
    "config": {
      "model": "{{model_name}}",
      "endpoint": "${LLM_ENDPOINT}",
      "api_key": "${LLM_API_KEY}",
      "max_tokens": 1024,
      "temperature": 0.7
    }
  }
}
```

---

## 🎯 Use Case Examples

### Customer Support Bot

```bash
# 1. Test prompt
promptfoo eval --config support-bot.yaml

# 2. Generate manifest
python manifest_generator.py
# → JWT + RAG + Inference modules

# 3. Deploy
curl -X POST http://localhost:8000/manifests \
  -d @manifests/support-bot.json

# 4. Upload FAQs
curl -X POST http://localhost:8000/api/v1/documents \
  -F "file=@faqs.pdf"
```

### Code Translation Service

```bash
# 1. Test translation prompts
promptfoo eval --config sas2py.yaml

# 2. Generate manifest
python manifest_generator.py
# → JWT + Inference + Monitoring modules

# 3. Deploy
curl -X POST http://localhost:8000/manifests \
  -d @manifests/sas2py.json

# 4. Test translation
curl -X POST http://localhost:8080/api/translate \
  -d '{"code": "DATA work.out; SET work.in; RUN;"}'
```

### Document Analysis Pipeline

```bash
# 1. Test extraction prompts
promptfoo eval --config doc-analysis.yaml

# 2. Generate manifest
python manifest_generator.py
# → JWT + RAG + Multiple Inference + Data Pipeline

# 3. Deploy
curl -X POST http://localhost:8000/manifests \
  -d @manifests/doc-analysis.json

# 4. Process documents
curl -X POST http://localhost:8080/api/analyze \
  -F "file=@report.pdf" \
  -F "analysis_type=financial"
```

---

## 🛠️ Troubleshooting Commands

```bash
# Check service health
curl http://localhost:8000/health  # Control Tower
curl http://localhost:5000/health  # JWT Service
curl http://localhost:8080/health  # Front Door
curl http://localhost:8001/health  # RAG Service

# View logs
docker-compose logs -f control-tower
docker-compose logs -f jwt-service
docker-compose logs -f front-door
docker-compose logs -f rag-service

# Test JWT token
curl -X POST http://localhost:5000/api/token/verify \
  -H "Authorization: Bearer $TOKEN"

# List manifests
curl http://localhost:8000/manifests

# Check RAG configurations
curl http://localhost:8000/api/v1/configurations

# Test Front Door routing
curl -X GET http://localhost:8080/api/routes

# Monitor metrics
curl http://localhost:8000/metrics
curl http://localhost:9090/api/v1/query?query=up

# Restart service
docker-compose restart service-name

# Clear cache
docker-compose exec redis redis-cli FLUSHALL
```

---

## 📊 Performance Benchmarks

| Operation                    | Target Latency | Optimization Tips                        |
|-----------------------------|----------------|------------------------------------------|
| JWT Token Generation        | < 50ms         | Cache tokens, use faster algorithm       |
| RAG Query (no expansion)    | < 500ms        | Optimize chunk size, use FAISS           |
| RAG Query (with expansion)  | < 2s           | Limit expansion queries, cache results    |
| LLM Inference (small)       | < 1s           | Use smaller models, reduce `max_tokens`  |
| LLM Inference (large)       | < 5s           | Stream responses, use GPU inference      |
| Document Upload             | < 10s/MB       | Batch processing, async uploads          |

---

## 🔗 Quick Links

**Essential resources**

- **Services:**
  - Control Tower: `http://localhost:8000/docs`
  - JWT Service: `http://localhost:5000/docs`
  - Front Door: `http://localhost:8080/docs`
  - RAG Service: `http://localhost:8001/docs`
- **Monitoring:**
  - Prometheus: `http://localhost:9090`
  - Grafana: `http://localhost:3000`
- **Documentation:**
  - Full Workflow Guide (Part 1 & 2)
  - Individual service docs
  - API references

---

## 💡 Pro Tips

- Start simple – test with one module at a time.
- Version your prompts and manifests in Git.
- Use environment variables for all secrets.
- Monitor metrics from day one.
- Test prompts extensively before production.
- Always enable authentication in production.
- Document your specific configurations.
- Use Docker Compose for consistent deployments.
