# DSP AI Platform: Manifest Generation & Deployment (Part 1B)

> This section continues from Part 1, covering manifest generation with the Control Tower CLI, service configuration, deployment with Front Door and JWT, and integration testing.

---

## Step 3: Generate Manifest with Control Tower CLI

### Using the Manifestor Tool

```bash
# Navigate to Control Tower examples
cd dsp-ai-control-tower/examples/manifestor

# Run the manifest generator
python manifest_generator.py

# Or use the launcher script (Windows)
generate_manifest.bat

# Or use the launcher script (Linux/Mac)
./generate_manifest.sh
```

### Example: Creating a Customer Support Manifest

```text
=== Control Tower Manifest Generator ===

Main Menu:
1. Create new manifest
2. Load existing manifest
3. Add module
4. Remove module
5. List modules
6. Preview manifest
7. Save manifest
0. Exit

Select option: 1

=== Creating New Manifest ===
Enter project ID: customer-support-bot
Enter project name: Customer Support Bot
Enter owner: AI Team
Enter environment (development/staging/production) [development]: development

Manifest created successfully!

Select option: 3

=== Available Module Types ===
1. jwt_config - JWT Authentication & Authorization
2. rag_config - RAG Configuration
3. rag_service - RAG Service Module
4. model_server - Model Server
5. api_gateway - API Gateway (Generic)
6. api_gateway_apisix - API Gateway (APISIX)
7. inference_endpoint - LLM Inference Endpoint
8. security - Security & Compliance
9. monitoring - Monitoring & Observability
10. model_registry - Model Registry (MLflow, W&B)
11. data_pipeline - Data Pipeline (ETL/ELT)
12. deployment - Deployment Configuration
13. resource_management - Resource Management
14. notifications - Notifications & Alerts
15. backup_recovery - Backup & Recovery
16. vault - HashiCorp Vault Integration
17. langgraph_workflow - LangGraph Workflow

Select module type: 1

=== Configuring JWT Config Module ===
Enter module name [customer-support-bot-jwt]:
Enter JWT service URL [http://localhost:5000]:
Enter consumer key [default-consumer]: support-bot-consumer
Enter rate limit (requests/second) [10]: 20
Enter token expiration (seconds) [3600]: 7200
Enable JWE encryption? (yes/no) [no]: yes
Enter JWE encryption key [${JWE_KEY}]: ${SUPPORT_BOT_JWE_KEY}

Module 'customer-support-bot-jwt' added successfully!

Select option: 3

Select module type: 2

=== Configuring RAG Config Module ===
Enter module name [customer-support-bot-rag]:
Enter RAG service URL [http://localhost:8000]:
Enter configuration name: support-docs
Select vector store type:
  1. faiss
  2. redis
  3. elasticsearch
  4. neo4j
  5. bm25
Select option [1]: 1
Enter embedding model [sentence-transformers/all-MiniLM-L6-v2]:
Enter chunk size [1000]: 800
Enter chunk overlap [200]: 150
Enter top K results [5]: 10

Module 'customer-support-bot-rag' added successfully!

Select option: 3

Select module type: 7

=== Configuring Inference Endpoint Module ===
Enter module name [customer-support-bot-llm]:
Enter model name: gpt-4
Enter endpoint URL: https://api.openai.com/v1/chat/completions
Enter API key variable [${API_KEY}]: ${OPENAI_API_KEY}
Enter system prompt: You are a helpful customer support agent...
Enter max tokens [1024]: 500
Enter temperature [0.7]: 0.3

Does this module depend on other modules? (yes/no) [no]: yes
Available modules:
  1. customer-support-bot-jwt
  2. customer-support-bot-rag
Select dependencies (comma-separated numbers): 1,2

Module 'customer-support-bot-llm' added successfully!

Select option: 7

Enter filename (without .json): customer-support-manifest
Manifest saved to manifests/customer-support-manifest.json
```

### Generated Manifest Structure

```json
{
  "project_id": "customer-support-bot",
  "project_name": "Customer Support Bot",
  "owner": "AI Team",
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:35:00Z",
  "status": "active",
  "environment": "development",
  "modules": [
    {
      "name": "customer-support-bot-jwt",
      "type": "jwt_config",
      "config": {
        "jwt_service_url": "http://localhost:5000",
        "consumer_key": "support-bot-consumer",
        "rate_limit": 20,
        "token_expiration": 7200,
        "jwe_enabled": true,
        "jwe_encryption_key": "${SUPPORT_BOT_JWE_KEY}"
      },
      "dependencies": []
    },
    {
      "name": "customer-support-bot-rag",
      "type": "rag_config",
      "config": {
        "rag_service_url": "http://localhost:8000",
        "configuration_name": "support-docs",
        "vector_store": {
          "type": "faiss",
          "index_path": "./storage/support_faiss"
        },
        "embedding": {
          "model": "sentence-transformers/all-MiniLM-L6-v2",
          "batch_size": 32
        },
        "chunking": {
          "strategy": "recursive",
          "chunk_size": 800,
          "chunk_overlap": 150
        },
        "retrieval_k": 10
      },
      "dependencies": []
    },
    {
      "name": "customer-support-bot-llm",
      "type": "inference_endpoint",
      "config": {
        "model": "gpt-4",
        "endpoint": "https://api.openai.com/v1/chat/completions",
        "api_key": "${OPENAI_API_KEY}",
        "system_prompt": "You are a helpful customer support agent...",
        "max_tokens": 500,
        "temperature": 0.3,
        "timeout": 30
      },
      "dependencies": [
        "customer-support-bot-jwt",
        "customer-support-bot-rag"
      ]
    }
  ],
  "environments": {
    "development": {
      "urls": {
        "jwt_service_url": "http://localhost:5000",
        "rag_service_url": "http://localhost:8000"
      }
    },
    "production": {
      "urls": {
        "jwt_service_url": "https://jwt.prod.example.com",
        "rag_service_url": "https://rag.prod.example.com"
      }
    }
  }
}
```

---

## Step 4: Deploy with Front Door and JWT

### Upload Manifest to Control Tower

```bash
# Upload manifest to Control Tower
curl -X POST http://localhost:8000/manifests \
  -H "Content-Type: application/json" \
  -H "X-Superuser-Secret: your-superuser-secret" \
  -d @manifests/customer-support-manifest.json

# Response
{
  "message": "Manifest created successfully",
  "project_id": "customer-support-bot"
}

# Verify manifest was created
curl http://localhost:8000/manifests/customer-support-bot
```

### Configure JWT Service

```bash
# 1. Create API key configuration for the support bot
cat > jwt_config.yaml << EOF
api_key: support-bot-key-123
name: Customer Support Bot
type: tiered_model_exec
config:
  models:
    - name: gpt-4
      endpoint: https://api.openai.com/v1/chat/completions
      api_key: ${OPENAI_API_KEY}
  rate_limits:
    requests_per_minute: 20
    tokens_per_minute: 10000
  metadata:
    department: customer_service
    access_level: standard
EOF

# 2. Upload configuration to JWT service
curl -X POST http://localhost:5000/api/config/api-keys \
  -H "Content-Type: application/json" \
  -d '{
    "api_key": "support-bot-key-123",
    "config": {
      "name": "Customer Support Bot",
      "type": "tiered_model_exec",
      "models": ["gpt-4"],
      "rate_limits": {
        "rpm": 20,
        "tpm": 10000
      }
    }
  }'

# 3. Generate JWT token for testing
curl -X POST http://localhost:5000/api/token \
  -H "Content-Type: application/json" \
  -d '{
    "username": "support-bot",
    "password": "secure-password"
  }'

# Example response
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 7200
}
```

### Configure Front Door Routing

```bash
# 1. Front Door auto-discovers the manifest from Control Tower
# 2. Create routing configuration

curl -X POST http://localhost:8080/api/routes \
  -H "Content-Type: application/json" \
  -d '{
    "route_name": "customer-support",
    "path": "/api/support/*",
    "upstream": "customer-support-bot",
    "methods": ["POST", "GET"],
    "plugins": {
      "jwt-auth": {
        "enabled": true,
        "key": "support-bot-consumer"
      },
      "rate-limit": {
        "enabled": true,
        "rate": 20,
        "burst": 30
      },
      "prometheus": {
        "enabled": true
      }
    }
  }'

# 3. Verify route is active
curl http://localhost:8080/api/routes/customer-support

# 4. Test the complete flow
curl -X POST http://localhost:8080/api/support/chat \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIs..." \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How do I return a product?",
    "session_id": "user-123"
  }'
```

---

## Step 5: Complete Integration Testing

### End-to-End Test Script

```python
# integration_test.py
import requests
import json
import time
from typing import Dict, Any

class DSPIntegrationTest:
    def __init__(self):
        self.control_tower_url = "http://localhost:8000"
        self.jwt_service_url = "http://localhost:5000"
        self.front_door_url = "http://localhost:8080"
        self.rag_service_url = "http://localhost:8000"
        
    def test_jwt_authentication(self) -> str:
        """Test JWT token generation"""
        print("Testing JWT authentication...")
        response = requests.post(
            f"{self.jwt_service_url}/api/token",
            json={
                "username": "test-user",
                "password": "test-password"
            }
        )
        assert response.status_code == 200
        token = response.json()["access_token"]
        print("✓ JWT authentication successful")
        return token
    
    def test_rag_document_upload(self, token: str):
        """Test document upload to RAG"""
        print("Testing RAG document upload...")
        
        # Create a test document
        test_doc = """
        Return Policy:
        - 30-day return window for unopened items
        - 14-day return for defective items
        - Original receipt required
        """
        
        files = {
            'file': ('return_policy.txt', test_doc, 'text/plain')
        }
        
        response = requests.post(
            f"{self.rag_service_url}/api/v1/documents",
            headers={"Authorization": f"Bearer {token}"},
            files=files,
            data={"configuration_name": "support-docs"}
        )
        assert response.status_code == 200
        print("✓ Document uploaded successfully")
    
    def test_manifest_deployment(self, token: str):
        """Test manifest deployment to Control Tower"""
        print("Testing manifest deployment...")
        
        manifest = {
            "project_id": "test-integration",
            "project_name": "Integration Test",
            "owner": "Test Suite",
            "modules": [
                {
                    "name": "test-jwt",
                    "type": "jwt_config",
                    "config": {
                        "jwt_service_url": self.jwt_service_url
                    }
                }
            ]
        }
        
        response = requests.post(
            f"{self.control_tower_url}/manifests",
            headers={
                "Authorization": f"Bearer {token}",
                "X-Superuser-Secret": "test-secret"
            },
            json=manifest
        )
        assert response.status_code in [200, 201]
        print("✓ Manifest deployed successfully")
    
    def test_front_door_routing(self, token: str):
        """Test Front Door API routing"""
        print("Testing Front Door routing...")
        
        response = requests.post(
            f"{self.front_door_url}/api/support/chat",
            headers={"Authorization": f"Bearer {token}"},
            json={
                "query": "What is the return policy?",
                "configuration": "support-docs"
            }
        )
        assert response.status_code == 200
        result = response.json()
        assert "30-day" in result.get("response", "")
        print("✓ Front Door routing successful")
        print(f"  Response: {result['response'][:100]}...")
    
    def test_complete_workflow(self):
        """Run complete integration test"""
        print("\n" + "="*50)
        print("DSP AI Platform Integration Test")
        print("="*50 + "\n")
        
        try:
            # Step 1: Get JWT token
            token = self.test_jwt_authentication()
            
            # Step 2: Upload documents to RAG
            self.test_rag_document_upload(token)
            
            # Step 3: Deploy manifest
            self.test_manifest_deployment(token)
            
            # Step 4: Test routing
            time.sleep(2)  # Wait for services to sync
            self.test_front_door_routing(token)
            
            print("\n" + "="*50)
            print("✅ All integration tests passed!")
            print("="*50)
            
        except AssertionError as e:
            print(f"\n❌ Test failed: {e}")
            raise
        except Exception as e:
            print(f"\n❌ Unexpected error: {e}")
            raise

if __name__ == "__main__":
    tester = DSPIntegrationTest()
    tester.test_complete_workflow()
```

### Running Integration Tests

```bash
# Run integration tests
python integration_test.py

# Example expected output:
==================================================
DSP AI Platform Integration Test
==================================================

Testing JWT authentication...
✓ JWT authentication successful
Testing RAG document upload...
✓ Document uploaded successfully
Testing manifest deployment...
✓ Manifest deployed successfully
Testing Front Door routing...
✓ Front Door routing successful
  Response: Based on the return policy, you have a 30-day return window for unopened items...

==================================================
✅ All integration tests passed!
==================================================
```

---

## Summary

In Parts 1 and 1B you covered how to:

- Define AI use cases with clear requirements.
- Test prompts comprehensively using promptfoo.
- Generate project manifests with the Manifestor CLI.
- Configure JWT authentication and API keys.
- Deploy manifests to Control Tower.
- Set up Front Door routing.
- Run end-to-end integration tests.

---

## Related Documentation

- **End-to-End Workflow Part 1** – Use cases and prompt testing.
- **End-to-End Workflow Part 2** – Production deployment and advanced topics.
- **Workflow Quick Reference** – Quick-start commands.
- **Control Tower Documentation**
- **Front Door Documentation**
- **JWT Service Documentation**
- **RAG Service Documentation Part 1**

> Next, continue with **Part 2** for production Docker Compose deployment, monitoring with Prometheus and Grafana, advanced use cases (ensemble models, knowledge graphs), performance optimization, and troubleshooting.
