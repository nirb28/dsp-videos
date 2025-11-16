# DSP AI Control Tower

> Control Tower is the centralized configuration and policy management system for the DSP AI Platform. It provides manifest management, OPA policy enforcement, and serves as the single source of truth for all AI services.

---

## Overview

### What is Control Tower?

Control Tower is a FastAPI-based application that provides:

- **Project Manifest Management** – Centralized configuration storage
- **Module Configuration** – 12 different module types
- **Policy-Based Access Control** – OPA (Open Policy Agent) integration
- **Environment Management** – Multi-environment support with variable substitution
- **Cross-Service Dependencies** – Automated dependency tracking and validation

### Key Features

**Core Capabilities**

- RESTful API for all operations
- JSON-based manifest system
- OPA policy evaluation
- HPC job template generation
- Environment variable resolution
- Module cross-referencing
- Validation and dependency checking

---

## Architecture

### Component Diagram

```text
┌─────────────────────────────────────────────────────────┐
│                    Control Tower                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │   FastAPI    │  │   Manifest   │  │     OPA      │ │
│  │   Layer      │──│    System    │──│    Engine    │ │
│  └──────────────┘  └──────────────┘  └──────────────┘ │
│         │                  │                  │        │
│         └──────────────────┴──────────────────┘        │
│                           │                            │
│                  ┌────────┴────────┐                   │
│                  │  Storage Layer  │                   │
│                  └─────────────────┘                   │
└─────────────────────────────────────────────────────────┘
           │              │              │
           ▼              ▼              ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │  Front   │   │   JWT    │   │  APISIX  │
    │  Door    │   │ Service  │   │ Gateway  │
    └──────────┘   └──────────┘   └──────────┘
```

### Core Components

| Component              | Description                         | Technology          |
|------------------------|-------------------------------------|---------------------|
| **FastAPI Layer**      | RESTful API endpoints for all operations | Python FastAPI      |
| **Manifest System**    | JSON-based project configurations   | Pydantic models     |
| **OPA Integration**    | Policy engine for access control    | Open Policy Agent (Rego) |
| **Module Registry**    | Catalog of available modules        | File-based storage  |
| **Storage Layer**      | Persistent configuration storage    | File system / Database |

---

## Module Types

Control Tower supports **12 module types** for comprehensive AI platform configuration:

| Module Type           | Purpose                              | Key Configuration                        |
|-----------------------|--------------------------------------|------------------------------------------|
| **jwt_config**        | Authentication and authorization     | Secret key, algorithm, expiration        |
| **rag_config**        | Retrieval Augmented Generation       | Vector store, embeddings, reranking      |
| **api_gateway**       | Request routing and management       | Routes, plugins, upstreams               |
| **inference_endpoint**| LLM model serving                    | Model, endpoint, parameters              |
| **security**          | Security policies and compliance     | Encryption, access control, audit        |
| **monitoring**        | Observability and metrics            | Logging, metrics, tracing                |
| **model_registry**    | Model versioning and lifecycle       | Registry URL, storage, metadata          |
| **data_pipeline**     | ETL/ELT workflows                    | Sources, transformations, targets        |
| **deployment**        | Environment-specific deployment      | Replicas, resources, scaling             |
| **resource_management**| Compute and storage allocation      | CPU, memory, GPU, storage                |
| **notifications**     | Alerting and notification channels   | Email, Slack, webhooks                   |
| **backup_recovery**   | Data backup and disaster recovery    | Schedule, retention, restore             |

---

## Getting Started

### Installation

```bash
# Clone repository
git clone https://github.com/dsp/control-tower.git
cd control-tower

# Create virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

# Start Control Tower
python app.py
```

> **Default Port**  
> Control Tower runs on **http://localhost:8000** by default.

### Quick Start Example

```bash
# List all manifests
curl http://localhost:8000/manifests

# Get specific manifest
curl http://localhost:8000/manifests/ai-customer-service

# Create new manifest
curl -X POST http://localhost:8000/manifests \
  -H "Content-Type: application/json" \
  -d @manifest.json

# Evaluate policy
curl -X POST http://localhost:8000/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "input_data": {
      "user": {"role": "developer"},
      "action": "deploy"
    },
    "client_id": "my-client",
    "client_secret": "secret123"
  }'
```

---

## API Reference

### Manifest Endpoints

| Method | Endpoint                 | Description                               |
|--------|--------------------------|-------------------------------------------|
| `GET`  | `/manifests`             | List all project manifests                |
| `POST` | `/manifests`             | Create new manifest (requires superuser)  |
| `GET`  | `/manifests/{id}`        | Get specific manifest                     |
| `PUT`  | `/manifests/{id}`        | Update manifest (requires superuser)      |
| `DELETE` | `/manifests/{id}`      | Delete manifest (requires superuser)      |
| `POST` | `/manifests/validate`    | Validate manifest structure               |

### Module Endpoints

| Method | Endpoint                          | Description                       |
|--------|-----------------------------------|-----------------------------------|
| `GET`  | `/manifests/{id}/modules`         | Get all modules in manifest       |
| `GET`  | `/manifests/{id}/modules/{name}`  | Get specific module               |
| `GET`  | `/module-types`                   | List available module types       |

### Policy Endpoints

| Method | Endpoint            | Description                     |
|--------|---------------------|---------------------------------|
| `GET`  | `/policies`         | List all OPA policies          |
| `POST` | `/evaluate`         | Evaluate policy with input data |
| `POST` | `/batch-evaluate`   | Batch policy evaluation         |

---

## Manifest Configuration

### Basic Manifest Structure

```json
{
  "project_id": "ai-customer-service",
  "name": "AI Customer Service Platform",
  "version": "1.0.0",
  "description": "Customer service AI with LLM and RAG",
  "environment": "production",
  "environments": {
    "development": {
      "secrets": {
        "jwt_secret": "dev-secret-key"
      },
      "urls": {
        "api_base": "http://localhost:8000"
      }
    },
    "production": {
      "secrets": {
        "jwt_secret": "${PROD_JWT_SECRET}"
      },
      "urls": {
        "api_base": "https://api.production.com"
      }
    }
  },
  "modules": [
    {
      "name": "auth-service",
      "type": "jwt_config",
      "config": {
        "secret_key": "${environments.${environment}.secrets.jwt_secret}",
        "algorithm": "HS256",
        "expiration_minutes": 30,
        "refresh_token_enabled": true
      }
    },
    {
      "name": "llm-inference",
      "type": "inference_endpoint",
      "config": {
        "model": "gpt-4",
        "endpoint": "${LLM_ENDPOINT}",
        "max_tokens": 2000,
        "temperature": 0.7
      },
      "cross_references": {
        "authentication": {
          "module_name": "auth-service",
          "module_type": "jwt_config",
          "purpose": "JWT token validation",
          "required": true
        }
      }
    }
  ]
}
```

### Environment Variables

> **Variable Substitution**
>
> Control Tower supports three types of variable substitution:
>
> - `${VARIABLE}` – OS environment variables
> - `${environment}` – Current environment name
> - `${environments.${environment}.key}` – Environment-specific values

---

## OPA Policy Management

### Policy Structure

```rego
package customer_service

# Client authentication
client_secret := "customer_service_secret_123"

# Default deny
default allow = false

# Allow data scientists to perform inference
allow {
    input.user.role == "data_scientist"
    input.action == "infer"
    input.resource.type == "llm_model"
    input.resource.status == "approved"
}

# Allow developers to deploy models
allow {
    input.user.role == "developer"
    input.action == "deploy"
    input.resource.monitoring.active == true
}

# Deny if model is not approved
deny {
    input.resource.status != "approved"
}
```

### Adding a New Client

1. Create policy file: `policies/clients/client_name.rego`
2. Define client secret in the policy
3. Write authorization rules
4. Test with client credentials

---

## Best Practices

### Configuration Best Practices

- **Version Control** – Store manifests in Git
- **Environment Separation** – Use environment-specific configurations
- **Secret Management** – Never hardcode secrets, use environment variables
- **Validation** – Always validate manifests before deployment
- **Documentation** – Document module dependencies and purposes
- **Testing** – Test policies with various input scenarios

### Security Best Practices

- **Principle of Least Privilege** – Grant minimum required permissions
- **Policy Versioning** – Track policy changes in version control
- **Audit Logging** – Log all policy evaluations
- **Regular Reviews** – Periodically review and update policies
- **Secret Rotation** – Regularly rotate client secrets

---

## Troubleshooting

### Manifest Validation Errors

**Problem:** Manifest fails validation  
**Solutions:**

- Check JSON syntax with a validator
- Verify all required fields are present
- Ensure module types are valid
- Check cross-references point to existing modules

### Policy Evaluation Failures

**Problem:** Policy evaluation returns unexpected results  
**Solutions:**

- Verify `client_id` and `client_secret` are correct
- Check policy file exists in `policies/clients/`
- Review Rego syntax for errors
- Test policy with OPA playground

### Environment Variable Resolution

**Problem:** Variables not resolving correctly  
**Solutions:**

- Verify environment variables are set
- Check variable syntax: `${VAR_NAME}`
- Use `?resolve_env=true` query parameter
- Review environment configuration in manifest

---

## Related Documentation

- Front Door Documentation
- JWT Service Documentation
- Integration Guide

---

## Additional Help

For additional support:

- Check the API docs at `http://localhost:8000/docs`
- Review example manifests in the repository
- Contact the DSP AI Platform team
