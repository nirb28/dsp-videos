# DSP Front Door (FD2)

> Front Door is an enterprise-grade, modular API gateway that serves as the intelligent front door for dynamic service discovery and routing. It provides a unified entry point for all AI/ML services with automatic module loading, security enforcement, and comprehensive observability.

---

## Overview

### What is Front Door?

Front Door v2 (FD2) is a dynamic API gateway that intelligently routes requests to backend services based on manifests from Control Tower. It acts as the single entry point for all AI platform services.

### Key Features

- **Dynamic Module Loading** – Load modules on-demand from manifests
- **Intelligent Routing** – Path, header, and subdomain-based routing
- **Security Enforcement** – JWT validation and authorization
- **Service Discovery** – Automatic endpoint resolution via Control Tower/manifest data
- **Load Balancing** – Round-robin, least connections, and more
- **Observability** – Metrics, logging, and distributed tracing
- **APISIX Integration** – Enterprise API gateway capabilities

---

## Architecture

### Request Flow

```text
┌─────────┐
│ Client  │
└────┬────┘
     │ 1. Request + JWT Token
     ▼
┌─────────────────────────────────────┐
│         Front Door (FD2)            │
├─────────────────────────────────────┤
│  Security Layer (JWT Validation)    │ 2. Validate JWT
│  Routing Layer (Path/Headers)       │ 3. Route Request
│  Module Layer (Dynamic Modules)     │ 4. Invoke Module
└─────────────────────────────────────┘
     │
     ▼
┌───────────────────────┐
│   Backend Services    │
└───────────────────────┘
```

### Core Components

| Component           | Description                                         |
|---------------------|-----------------------------------------------------|
| **Security Layer**  | Validates JWT tokens, enforces authentication       |
| **Routing Layer**   | Matches incoming requests to routes/modules         |
| **Module Manager**  | Loads and manages modules defined in manifests      |
| **Control Tower Client** | Fetches manifests and module configuration     |
| **APISIX Gateway**  | Optional external gateway for advanced features     |

---

## Module System

FD2 uses a pluggable module system. Modules are defined in Control Tower manifests and loaded dynamically at runtime.

### Module Types

Common module types include:

- **Router modules** – Define routing logic and upstreams
- **Auth modules** – JWT validation, API key checks
- **Proxy modules** – Direct HTTP proxying to services
- **Custom logic modules** – Business logic or orchestration

### Example Module Definition

```json
{
  "name": "rag-gateway",
  "type": "router",
  "config": {
    "base_path": "/rag",
    "upstream_url": "http://rag-service:8000",
    "timeout_ms": 30000,
    "retries": 2
  },
  "security": {
    "require_jwt": true,
    "required_scopes": ["rag:query"]
  }
}
```

---

## APISIX Integration

FD2 can be deployed behind **Apache APISIX** to provide:

- Authentication (JWT, API key, Basic, OAuth2)
- Rate limiting (per IP, per user, per key)
- Request/response transformation
- Logging and tracing
- Load balancing across multiple FD2 instances

### Example APISIX Route

```json
{
  "uri": "/fd2/*",
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "fd2-1:8080": 1,
      "fd2-2:8080": 1
    }
  },
  "plugins": {
    "jwt-auth": {},
    "limit-count": {
      "count": 100,
      "time_window": 60,
      "rejected_code": 429
    }
  }
}
```

---

## Getting Started

### Prerequisites

- Control Tower running and accessible
- Optional: APISIX gateway
- JWT Service (for token issuance)

### Running Front Door

```bash
# Example: docker-compose service

docker-compose up -d front-door

# Environment variables (examples)
export CONTROL_TOWER_URL=http://control-tower:8000
export JWT_SERVICE_URL=http://jwt-service:5000
export LOG_LEVEL=INFO
```

### Basic Health Check

```bash
curl http://localhost:8080/health
```

---

## Routing Examples

### Simple Proxy Route

```bash
curl -X POST http://localhost:8080/projects/ai-customer-service/modules/rag-gateway/query \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "What is our refund policy?"}'
```

### Using Path and Headers

- **Path-based routing** – `/projects/{project}/modules/{module}/...`
- **Header-based routing** – Use headers like `X-DSP-Project`, `X-DSP-Module`

---

## Observability

FD2 exposes metrics and logs for monitoring.

### Logging

- Request/response logs
- Module loading failures
- JWT validation errors

### Metrics (examples)

- Request counts and latencies
- Error rates per module
- JWT validation failures

---

## Troubleshooting

### Module Not Loading

- Verify manifest in Control Tower
- Check module implementation path and class name
- Ensure module implements the expected base interface
- Check logs for import/initialization errors

### Routing Issues

- Confirm project and module names in URLs
- Ensure manifest has the correct module configuration
- Check routing logs for match/miss details
- Verify JWT contains required claims for protected routes

### Performance Degradation

- Check backend service health and latency
- Monitor CPU/memory utilization of FD2 and backends
- Tune connection pools and timeouts
- Enable caching or rate limiting as appropriate

---

## Related Documentation

- Control Tower Documentation
- JWT Service Documentation
- APISIX Integration Guide
- Module Development Guide

---

## Additional Help

- API docs: `http://localhost:8080/docs`
- Example modules in the Front Door repository
- DSP AI Platform team support channels
