# DSP AI RAG2 (Retrieval-Augmented Generation) – Part 1

> A comprehensive RAG platform built with FastAPI, offering configurable chunking strategies, vector stores, embedding models, and generation models. Features OpenAI-compatible API, query expansion, multi-vector retrieval, and enterprise security.

---

## Overview

### What is DSP AI RAG2?

DSP AI RAG2 is an enterprise-grade RAG platform that combines document retrieval with Large Language Models (LLMs) to provide accurate, context-aware responses over your organization’s documents.

### Key Features

- **OpenAI-Compatible API** – Drop-in replacement for OpenAI chat completions
- **Multiple Vector Stores** – FAISS, Redis, Elasticsearch, Neo4j, BM25
- **Query Expansion** – Automatic query enhancement using LLMs
- **Multi-Configuration Retrieval** – Query multiple knowledge bases at once
- **Reranking** – Advanced reranking for relevance
- **Security** – JWT authentication with metadata-based filtering
- **Knowledge Graphs** – Neo4j integration for semantic relationships
- **Document Processing** – PDF, DOCX, PPTX, TXT
- **Configurable Pipeline** – Chunking, embedding, generation fully configurable
- **Production Ready** – FastAPI with monitoring hooks

---

## Architecture

### System Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                      DSP AI RAG2                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  API Layer                                                  │
│    ├─ REST API Endpoints                                    │
│    ├─ OpenAI-Compatible Endpoints                           │
│    └─ MCP Server                                            │
│                                                             │
│  RAG Service Core                                           │
│    ├─ Query Expansion                                       │
│    ├─ Document Processor                                    │
│    └─ Configuration Manager                                 │
│                                                             │
│  Processing Pipeline                                        │
│    ├─ Chunking Service                                      │
│    ├─ Embedding Service                                     │
│    └─ Storage Service                                       │
│                                                             │
│  Vector Stores                                              │
│    ├─ FAISS  ├─ Redis ├─ Elasticsearch ├─ Neo4j ├─ BM25     │
│                                                             │
│  LLM Providers                                              │
│    ├─ Groq ├─ OpenAI-compatible ├─ Triton ├─ Custom         │
└─────────────────────────────────────────────────────────────┘
```

### Request Flow

1. **User Query → API Endpoint**  
2. **Authentication** (JWT validation, if enabled)  
3. **Query Expansion (optional)**  
   - Generate query variations  
   - Multi-query generation  
4. **Document Retrieval**  
   - Vector similarity search  
   - Metadata filtering  
   - Multi-store fusion (optional)  
5. **Reranking (optional)** – Score/model-based reranking  
6. **Context Assembly** – Combine retrieved docs and context  
7. **LLM Generation** – System prompt + context-aware response  
8. **Response → User**

---

## Getting Started

### Prerequisites

- Python 3.8+
- Groq API key (or other LLM provider key)
- Optional: Redis, Elasticsearch, Neo4j for advanced storage

### Installation

```bash
# Clone repository
git clone https://github.com/dsp/dsp_ai_rag2.git
cd dsp_ai_rag2

# Create virtual environment
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env and add your API keys

# Run the application
python -m app.main
```

> **Default Port**: RAG Service runs on **http://localhost:8000** by default.

### Environment Configuration

```bash
# .env (example)
GROQ_API_KEY=your_groq_api_key
OPENAI_API_KEY=your_openai_api_key
COHERE_API_KEY=your_cohere_api_key

STORAGE_PATH=./storage
MAX_FILE_SIZE=10485760
LOCAL_MODELS_PATH=./models

MODEL_SERVER_URL=http://localhost:9001
REDIS_URL=redis://localhost:6379
LOG_LEVEL=INFO
```

---

## Configuration Management

### Creating a Configuration

```bash
curl -X POST http://localhost:8000/api/v1/configurations \
  -H "Content-Type: application/json" \
  -d '{
    "configuration_name": "company_knowledge_base",
    "chunking": {
      "strategy": "recursive",
      "chunk_size": 1000,
      "chunk_overlap": 200
    },
    "vector_store": {
      "type": "faiss",
      "normalize_similarity_scores": true
    },
    "embedding": {
      "model": "sentence-transformers/all-MiniLM-L6-v2",
      "batch_size": 32
    },
    "generation": {
      "provider": "groq",
      "model": "llama3-8b-8192",
      "api_key": "${GROQ_API_KEY}",
      "temperature": 0.7,
      "max_tokens": 1024
    },
    "reranking": {
      "enabled": true,
      "model": "cohere",
      "top_n": 10
    },
    "retrieval_k": 5,
    "similarity_threshold": 0.7
  }'
```

### Configuration Components

| Component        | Options                                       | Description                     |
|------------------|-----------------------------------------------|---------------------------------|
| **Chunking**     | fixed, sentence, recursive, semantic          | Document splitting strategy     |
| **Vector Store** | faiss, redis, elasticsearch, neo4j, bm25      | Storage backend                 |
| **Embedding**    | sentence-transformers, openai, custom         | Text embedding model            |
| **Generation**   | groq, openai, triton                          | LLM provider for responses      |
| **Reranking**    | cohere, cross-encoder, custom                 | Result reranking model          |

---

## Document Management

### Uploading Documents

```bash
# Upload a PDF
auth_header="Authorization: Bearer $TOKEN"  # if security enabled

curl -X POST http://localhost:8000/api/v1/documents \
  -H "Content-Type: multipart/form-data" \
  -H "$auth_header" \
  -F "file=@company_manual.pdf" \
  -F "configuration_name=company_knowledge_base" \
  -F "metadata={\"department\":\"engineering\",\"year\":2024}"
```

### Supported File Formats

- **PDF** – PyPDF2
- **DOCX** – `python-docx`
- **PPTX** – `python-pptx`
- **TXT** – Plain text

### Document Processing Pipeline

```text
1. File Upload
2. Content Extraction
   - PDF, DOCX, PPTX, TXT
3. Text Chunking
   - Strategy-based splitting + overlap
4. Embedding Generation
   - Batch processing to vectors
5. Vector Storage
   - Index creation + metadata
6. Ready for Retrieval
```

---

## Related Documentation

- RAG Service Documentation Part 2 – Query Operations & API
- RAG Service Documentation Part 3 – Advanced Features & Best Practices
- Control Tower Documentation
- Front Door Documentation
- JWT Service Documentation
