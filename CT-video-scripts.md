# Control Tower Video Tutorial Scripts

## Episode CT-01: Introduction & Architecture (8 minutes)

### Opening [00:00-00:20]
**Visual**: DSP AI Platform logo animation
**Script**: "Welcome to the DSP AI Platform tutorial series! I'm excited to introduce you to the Control Tower - the centralized brain of our AI platform. In this video, you'll learn what Control Tower is, why it's essential, and how its architecture enables intelligent AI service management."

### What is Control Tower? [00:20-01:00]
**Visual**: Animated diagram showing Control Tower at center
**Script**: "Control Tower is our centralized configuration and policy management system. Think of it as the command center for your entire AI infrastructure. It manages project manifests, enforces security policies through OPA, and provides a single source of truth for all your AI services."

**Key Points**:
- Project manifest management
- Module configuration storage
- Policy-based access control
- Environment-specific settings
- Cross-service dependency tracking

### Why Control Tower? [01:00-02:30]
**Visual**: Before/After comparison
**Script**: "Without Control Tower, teams face configuration sprawl, inconsistent policies, and manual dependency management. Control Tower solves these with centralized management, consistent enforcement, and automated resolution."

### Core Architecture [02:30-04:00]
**Visual**: Architecture diagram with components
**Script**: "Let's explore the architecture:
1. FastAPI Application Layer - RESTful APIs
2. Manifest System - JSON-based configurations
3. OPA Integration - Policy engine
4. Module Registry - Available modules catalog
5. Storage Layer - File or database storage"

### Module Types Overview [04:00-05:30]
**Visual**: Grid of 12 module types
**Script**: "Control Tower supports 12 module types including JWT Config, RAG Config, API Gateway, Inference Endpoint, Security, Monitoring, and more. Each has specific schemas and validation rules."

### Integration Points [05:30-06:30]
**Visual**: Integration animation
**Script**: "Control Tower integrates with Front Door for module configs, JWT Service for auth policies, APISIX for routing, and monitoring systems for metrics."

### Quick Demo [06:30-07:30]
**Visual**: Terminal demonstration
```bash
# Start Control Tower
python app.py

# List manifests
curl http://localhost:8000/manifests

# Get specific manifest
curl http://localhost:8000/manifests/ai-customer-service
```

### Closing [07:30-08:00]
**Visual**: Summary slide
**Script**: "You've learned that Control Tower is the centralized hub bringing order to AI infrastructure. Next episode: the manifest system in detail. Thanks for watching!"

---

## Episode CT-02: Manifest System & Module Configuration (12 minutes)

### Opening [00:00-00:20]
**Visual**: Recap of previous episode
**Script**: "Welcome back! In this episode, we'll dive deep into the manifest system - the heart of Control Tower's configuration management."

### Understanding Manifests [00:20-01:30]
**Visual**: JSON manifest structure
**Script**: "A manifest is a JSON document defining your entire project configuration. It includes project metadata, environment settings, and module definitions."

**Example Structure**:
```json
{
  "project_id": "ai-customer-service",
  "name": "AI Customer Service Platform",
  "version": "1.0.0",
  "environment": "production",
  "modules": [...]
}
```

### Creating Your First Manifest [01:30-03:30]
**Visual**: Live coding session
**Script**: "Let's create a manifest from scratch. We'll build a simple AI service with authentication and inference."

**Step-by-step**:
1. Define project metadata
2. Add JWT authentication module
3. Configure inference endpoint
4. Set environment variables

### Module Configuration Deep Dive [03:30-06:00]
**Visual**: Module configuration examples
**Script**: "Each module has a specific configuration schema. Let's explore key modules:"

**JWT Module**:
```json
{
  "name": "auth-service",
  "type": "jwt_config",
  "config": {
    "secret_key": "${JWT_SECRET}",
    "algorithm": "HS256",
    "expiration_minutes": 30
  }
}
```

**Inference Module**:
```json
{
  "name": "llm-inference",
  "type": "inference_endpoint",
  "config": {
    "model": "gpt-4",
    "endpoint": "${LLM_ENDPOINT}",
    "max_tokens": 2000
  }
}
```

### Environment Management [06:00-08:00]
**Visual**: Environment configuration flow
**Script**: "Control Tower supports multiple environments with variable substitution and overrides."

**Features**:
- Environment-specific values
- Variable substitution with ${VAR}
- Override mechanisms
- Secret management integration

### Cross-References & Dependencies [08:00-10:00]
**Visual**: Dependency graph
**Script**: "Modules can reference each other, creating a dependency graph. Control Tower validates these references and ensures consistency."

**Example**:
```json
"cross_references": {
  "authentication": {
    "module_name": "auth-service",
    "module_type": "jwt_config",
    "purpose": "JWT validation",
    "required": true
  }
}
```

### API Operations [10:00-11:30]
**Visual**: API calls demonstration
**Script**: "Let's see how to interact with manifests via API:"

```bash
# Create manifest
curl -X POST http://localhost:8000/manifests \
  -H "Content-Type: application/json" \
  -d @manifest.json

# Update manifest
curl -X PUT http://localhost:8000/manifests/project-id \
  -d @updated-manifest.json

# Validate manifest
curl -X POST http://localhost:8000/manifests/validate \
  -d @manifest.json
```

### Closing [11:30-12:00]
**Visual**: Key takeaways
**Script**: "You've mastered manifest creation, module configuration, and environment management. Next: Policy management with OPA!"

---

## Episode CT-03: Policy Management & OPA Integration (10 minutes)

### Opening [00:00-00:20]
**Visual**: OPA logo and integration diagram
**Script**: "Welcome to episode 3! Today we'll explore how Control Tower uses Open Policy Agent for powerful, flexible access control."

### Understanding OPA Policies [00:20-01:30]
**Visual**: Policy concept diagram
**Script**: "OPA policies define who can do what with which resources. They're written in Rego, a declarative language designed for policy decisions."

**Basic Policy Structure**:
```rego
package customer_service

default allow = false

allow {
    input.user.role == "data_scientist"
    input.action == "infer"
    input.resource.type == "llm_model"
}
```

### Client Authentication [01:30-03:00]
**Visual**: Authentication flow
**Script**: "Each client has a unique ID and secret, stored in their policy file. This enables secure, client-specific policy enforcement."

**Setup**:
1. Create policy file: `policies/clients/client_name.rego`
2. Define client secret
3. Write authorization rules
4. Test with client credentials

### Writing Custom Policies [03:00-05:00]
**Visual**: Live Rego coding
**Script**: "Let's write a real-world policy for role-based access control:"

```rego
# Define roles and permissions
roles_permissions := {
    "admin": ["read", "write", "delete", "infer"],
    "developer": ["read", "write", "infer"],
    "viewer": ["read"]
}

# Check permission
allow {
    role := input.user.role
    action := input.action
    action in roles_permissions[role]
}

# Resource-specific rules
allow {
    input.resource.status == "approved"
    input.resource.monitoring.active == true
}
```

### Policy Evaluation API [05:00-07:00]
**Visual**: API request/response flow
**Script**: "Control Tower provides endpoints for policy evaluation:"

```bash
# Evaluate single policy
curl -X POST http://localhost:8000/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "input_data": {
      "user": {"role": "developer"},
      "action": "infer",
      "resource": {"type": "llm_model", "status": "approved"}
    },
    "client_id": "customer_service",
    "client_secret": "secret123"
  }'

# Batch evaluation
curl -X POST http://localhost:8000/batch-evaluate \
  -d @batch-request.json
```

### HPC Template Generation [07:00-08:30]
**Visual**: Template generation flow
**Script**: "Control Tower can generate HPC job templates based on policies:"

```bash
# Jupyter Lab template
curl -X POST http://localhost:8000/templates/jupyter-lab \
  -H "X-DSPAI-Client-ID: customer_service" \
  -H "X-DSPAI-Client-Secret: password" \
  -d '{"env_type": "training_dev", "username": "user123"}'

# Model deployment template
curl -X POST http://localhost:8000/templates/model-deployment \
  -H "X-DSPAI-Client-ID: customer_service" \
  -d '{"model_name": "sentiment_analysis"}'
```

### Best Practices [08:30-09:30]
**Visual**: Best practices checklist
**Script**: "Follow these best practices for effective policy management:"

1. **Principle of Least Privilege**: Grant minimum required permissions
2. **Policy Versioning**: Track policy changes in git
3. **Testing**: Write unit tests for policies
4. **Documentation**: Comment complex rules
5. **Monitoring**: Log policy decisions for audit

### Closing [09:30-10:00]
**Visual**: Series summary
**Script**: "Congratulations! You've completed the Control Tower series. You can now manage configurations, create manifests, and enforce policies. Next series: Front Door!"

## Resources & Commands Reference

### Setup Commands
```bash
# Clone repository
git clone <repo-url>
cd dsp-ai-control-tower

# Create virtual environment
python -m venv venv
source venv/bin/activate  # or venv\Scripts\activate on Windows

# Install dependencies
pip install -r requirements.txt

# Start Control Tower
python app.py
```

### Key API Endpoints
- `GET /` - Welcome message
- `GET /policies` - List all policies
- `POST /evaluate` - Evaluate policy
- `GET /manifests` - List manifests
- `POST /manifests` - Create manifest
- `GET /manifests/{id}` - Get specific manifest
- `PUT /manifests/{id}` - Update manifest
- `DELETE /manifests/{id}` - Delete manifest

### Sample Files Location
- Manifests: `manifests/`
- Policies: `policies/clients/`
- Examples: `manifest_examples/`

### Documentation Links
- [Control Tower README](../dsp-ai-control-tower/README.md)
- [Manifest System Docs](../dsp-ai-control-tower/MANIFEST_SYSTEM.md)
- [OPA Documentation](https://www.openpolicyagent.org/docs/)
