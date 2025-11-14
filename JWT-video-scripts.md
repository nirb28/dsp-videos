# JWT Service Video Tutorial Scripts

## Episode JWT-01: Introduction & Authentication Methods (8 minutes)

### Opening [00:00-00:20]
**Visual**: JWT token animation and service logo
**Script**: "Welcome to the JWT Service tutorial series! Today we'll explore how our JWT service provides secure, flexible authentication for the entire DSP AI platform. Let's unlock the power of modern token-based auth!"

### What is JWT Service? [00:20-01:30]
**Visual**: Service architecture diagram
**Script**: "Our JWT Service is a Flask-based authentication system designed for APISIX gateway integration. It provides token generation with multiple auth methods and dynamic claims based on API keys."

**Core Features**:
- Multiple authentication methods (LDAP, file-based)
- Configurable JWT token signing
- Dynamic claims generation
- API key-based permissions
- Token refresh mechanism
- Seamless APISIX integration

### Understanding JWT Tokens [01:30-02:30]
**Visual**: JWT structure breakdown
**Script**: "JWT (JSON Web Token) consists of three parts:

1. **Header**: Algorithm and token type
2. **Payload**: Claims (user data, permissions)
3. **Signature**: Ensures token integrity

Example:
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiJ1c2VyMTIzIiwicm9sZSI6ImRldmVsb3BlciJ9.
TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

Each part is Base64 encoded and separated by dots."

### Authentication Methods [02:30-04:30]
**Visual**: Auth flow diagrams
**Script**: "JWT Service supports two authentication methods:

**1. File-based Authentication**:
```yaml
# config/users.yaml
john_doe:
  password: <sha256_hash>
  name: John Doe
  email: john@example.com
  groups: [developers, ml_team]
  roles: [developer, ml_engineer]
```

**2. LDAP/Active Directory**:
```env
LDAP_SERVER=ldap://corp-ldap:389
LDAP_BASE_DN=dc=company,dc=com
LDAP_USER_DN=cn=users,dc=company,dc=com
```

Both methods provide secure, enterprise-ready authentication."

### Token Generation Flow [04:30-06:00]
**Visual**: Sequence diagram of token generation
**Script**: "Here's how tokens are generated:

1. Client sends credentials + API key
2. Service validates credentials
3. Loads API key configuration
4. Generates static claims
5. Resolves dynamic claims
6. Signs JWT with secret
7. Returns access & refresh tokens

```bash
curl -X POST http://localhost:5000/token \
  -H "Content-Type: application/json" \
  -d '{
    "username": "john_doe",
    "password": "secure_password",
    "api_key": "api_key_openai_1234"
  }'
```"

### Security Best Practices [06:00-07:00]
**Visual**: Security checklist
**Script**: "Follow these security practices:

1. **Strong Secrets**: Use cryptographically secure keys
2. **Short Expiration**: Tokens expire in 30-60 minutes
3. **Refresh Tokens**: Separate long-lived refresh tokens
4. **HTTPS Only**: Always use encrypted connections
5. **Secret Rotation**: Regularly rotate signing keys
6. **Audit Logging**: Track all authentication attempts"

### Quick Demo [07:00-07:40]
**Visual**: Terminal demonstration
```bash
# Start JWT Service
python app.py

# Generate token
TOKEN=$(curl -s -X POST http://localhost:5000/token \
  -d '{"username":"admin","password":"password","api_key":"test_key"}' \
  | jq -r '.access_token')

# Decode token
curl -X POST http://localhost:5000/decode \
  -d "{\"token\":\"$TOKEN\"}"
```

### Closing [07:40-08:00]
**Visual**: Next episode preview
**Script**: "You've learned JWT Service fundamentals and authentication methods. Next episode: API key configuration and dynamic claims - the real power of our system!"

---

## Episode JWT-02: API Keys & Dynamic Claims (10 minutes)

### Opening [00:00-00:20]
**Visual**: API key configuration overview
**Script**: "Welcome back! Today we'll explore the powerful API key system and dynamic claims - features that make our JWT Service incredibly flexible and intelligent."

### API Key Configuration [00:20-02:00]
**Visual**: Configuration file structure
**Script**: "Each API key has a configuration file defining permissions and claims:

```yaml
# config/api_keys/api_key_premium_tier.yaml
id: premium-service
owner: Premium Team
provider_permissions:
  - openai
  - anthropic
  - groq
endpoint_permissions:
  - /v1/chat/completions
  - /v1/embeddings
claims:
  static:
    tier: premium
    rate_limit: 1000
    models:
      - gpt-4
      - claude-3
  dynamic:
    # We'll explore these next
```

This configuration controls what the token holder can access."

### Static Claims [02:00-03:00]
**Visual**: Static claims examples
**Script**: "Static claims are fixed values included in every token:

```yaml
static:
  tier: premium
  rate_limit: 1000
  max_tokens: 4000
  models:
    - gpt-4
    - gpt-3.5-turbo
  features:
    - streaming
    - function_calling
  metadata_filter:
    department: engineering
    access_level: confidential
```

These become part of the JWT payload for authorization decisions."

### Dynamic Claims - Function-based [03:00-05:00]
**Visual**: Function-based claim flow
**Script**: "Dynamic claims are generated at runtime. Function-based claims call Python functions:

```yaml
dynamic:
  remaining_quota:
    type: function
    module: claims.quota
    function: get_remaining_quota
    args:
      user_id: "{user_id}"
      tier: "{tier}"
```

Implementation:
```python
# claims/quota.py
def get_remaining_quota(user_id, tier):
    # Check database for usage
    used = db.get_usage(user_id)
    limits = {'premium': 10000, 'basic': 1000}
    remaining = limits[tier] - used
    return {"quota": remaining, "reset_date": "2024-12-01"}
```

This enables real-time quota checking!"

### Dynamic Claims - API-based [05:00-07:00]
**Visual**: API-based claim flow
**Script**: "API-based claims fetch data from external services:

```yaml
dynamic:
  user_permissions:
    type: api
    url: "http://permission-service/api/permissions/{user_id}"
    method: GET
    headers:
      Authorization: "Bearer {internal_token}"
    response_field: data.permissions
  
  usage_stats:
    type: api
    url: "http://analytics/api/stats"
    method: POST
    body:
      user_id: "{user_id}"
      date_range: "last_30_days"
    response_field: summary
```

This integrates with existing services for centralized data."

### Inline Configuration [07:00-08:30]
**Visual**: Inline config example
**Script**: "New feature: pass configuration directly in the request:

```json
{
  "username": "john_doe",
  "password": "password",
  "api_key_config": {
    "id": "custom-key",
    "owner": "Development Team",
    "claims": {
      "static": {
        "key": "dev-key",
        "tier": "developer",
        "models": ["gpt-3.5-turbo"],
        "rate_limit": 100,
        "exp_hours": 2
      }
    }
  }
}
```

Perfect for dynamic environments and testing!"

### Practical Examples [08:30-09:30]
**Visual**: Real-world scenarios
**Script**: "Let's see practical use cases:

**1. Tiered Access**:
```yaml
# Basic tier
static:
  tier: basic
  models: [gpt-3.5-turbo]
  rate_limit: 100

# Premium tier
static:
  tier: premium
  models: [gpt-4, claude-3]
  rate_limit: 1000
```

**2. Department-based Access**:
```yaml
dynamic:
  department_resources:
    type: function
    module: claims.department
    function: get_resources
    args:
      dept: "{user_department}"
```

**3. Cost Tracking**:
```yaml
dynamic:
  budget_remaining:
    type: api
    url: "http://billing/api/budget/{team_id}"
```"

### Closing [09:30-10:00]
**Visual**: Configuration best practices
**Script**: "You've mastered API key configuration and dynamic claims! You can now create sophisticated authorization rules with real-time data. Next: Integration with APISIX and Front Door!"

---

## Episode JWT-03: Gateway Integration (7 minutes)

### Opening [00:00-00:20]
**Visual**: Integration architecture diagram
**Script**: "Welcome to our final JWT episode! Today we'll connect everything - integrating JWT Service with APISIX gateway and Front Door for complete authentication."

### APISIX Integration [00:20-02:00]
**Visual**: APISIX JWT plugin configuration
**Script**: "APISIX validates our JWT tokens using the jwt-auth plugin:

```json
{
  "plugins": {
    "jwt-auth": {
      "key": "user-key",
      "secret": "your-shared-secret-key",
      "algorithm": "HS256",
      "exp": 3600,
      "base64_secret": false
    }
  }
}
```

The secret must match JWT Service configuration:
```python
# JWT Service config
JWT_SECRET_KEY = "your-shared-secret-key"
JWT_ALGORITHM = "HS256"
```

Now APISIX can validate tokens without calling JWT Service!"

### Front Door Integration [02:00-03:30]
**Visual**: Request flow through all components
**Script**: "Here's the complete flow:

1. Client authenticates with JWT Service
2. Receives JWT token with claims
3. Sends request to Front Door with token
4. APISIX validates token
5. Front Door reads claims from token
6. Routes based on permissions
7. Backend service processes request

```bash
# Get token
TOKEN=$(curl -X POST http://jwt-service:5000/token \
  -d '{"username":"user","password":"pass","api_key":"key"}' \
  | jq -r '.access_token')

# Use token with Front Door
curl http://front-door:8080/ai-service/inference/chat \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"prompt":"Hello"}'
```"

### Claims-based Routing [03:30-05:00]
**Visual**: Routing decision tree
**Script**: "Front Door uses JWT claims for intelligent routing:

```python
# Front Door routing logic
def route_request(request, jwt_claims):
    # Check tier for model access
    if jwt_claims.get('tier') == 'premium':
        endpoints = self.premium_endpoints
    else:
        endpoints = self.basic_endpoints
    
    # Check rate limits
    rate_limit = jwt_claims.get('rate_limit', 100)
    if self.check_rate_limit(user, rate_limit):
        return self.rate_limit_exceeded()
    
    # Route based on permissions
    allowed_models = jwt_claims.get('models', [])
    if request.model not in allowed_models:
        return self.forbidden()
    
    return self.forward_to_endpoint(endpoints, request)
```

Claims enable fine-grained access control!"

### Token Refresh Flow [05:00-06:00]
**Visual**: Refresh token sequence
**Script**: "Implement seamless token refresh:

```javascript
// Client-side token management
async function makeRequest(url, data) {
  let response = await fetch(url, {
    headers: {'Authorization': `Bearer ${accessToken}`},
    body: JSON.stringify(data)
  });
  
  if (response.status === 401) {
    // Token expired, refresh
    const refreshResponse = await fetch('/refresh', {
      headers: {'Authorization': `Bearer ${refreshToken}`}
    });
    
    const {access_token} = await refreshResponse.json();
    accessToken = access_token;
    
    // Retry original request
    response = await fetch(url, {
      headers: {'Authorization': `Bearer ${accessToken}`},
      body: JSON.stringify(data)
    });
  }
  
  return response;
}
```"

### Production Deployment [06:00-06:40]
**Visual**: Docker Compose setup
**Script**: "Deploy the complete stack:

```yaml
version: '3.8'
services:
  jwt-service:
    image: dsp-jwt:latest
    environment:
      - JWT_SECRET_KEY=${JWT_SECRET}
      - LDAP_SERVER=${LDAP_SERVER}
    ports:
      - "5000:5000"
  
  apisix:
    image: apache/apisix
    volumes:
      - ./apisix.yaml:/usr/local/apisix/conf/apisix.yaml
    environment:
      - JWT_SECRET=${JWT_SECRET}
  
  front-door:
    image: dsp-fd2:latest
    depends_on:
      - jwt-service
      - apisix
```

All services share the same JWT secret for validation."

### Closing [06:40-07:00]
**Visual**: JWT series summary
**Script**: "Congratulations! You've mastered JWT Service from authentication to gateway integration. You can now implement secure, scalable authentication for your AI platform. Check out our integration tutorials for the complete system setup!"

---

## Integration Scripts

## Episode INT-01: Complete System Setup (10 minutes)

### Opening [00:00-00:20]
**Visual**: Complete architecture diagram
**Script**: "Welcome to the integration series! Today we'll set up the entire DSP AI platform - Control Tower, Front Door, and JWT Service working together seamlessly."

### Prerequisites [00:20-01:00]
**Visual**: Requirements checklist
**Script**: "Before we start, ensure you have:
- Docker and Docker Compose installed
- Python 3.11+
- 8GB RAM minimum
- Ports 5000, 8000, 8080 available
- Git for cloning repositories"

### System Architecture [01:00-02:00]
**Visual**: Component interaction diagram
**Script**: "Our integrated system:
1. Control Tower manages configurations
2. JWT Service handles authentication
3. Front Door routes requests
4. APISIX provides gateway features
5. Backend services process requests

All components communicate securely using JWT tokens and REST APIs."

### Step 1: Clone Repositories [02:00-02:30]
**Visual**: Terminal commands
```bash
# Clone all repositories
git clone https://github.com/dsp/control-tower.git
git clone https://github.com/dsp/front-door.git
git clone https://github.com/dsp/jwt-service.git

# Create workspace
mkdir dsp-platform && cd dsp-platform
mv ../control-tower ./
mv ../front-door ./
mv ../jwt-service ./
```

### Step 2: Configure Environment [02:30-04:00]
**Visual**: Configuration files
**Script**: "Create unified environment configuration:

```env
# .env
JWT_SECRET_KEY=your-secure-secret-key-min-32-chars
CONTROL_TOWER_URL=http://control-tower:8000
JWT_SERVICE_URL=http://jwt-service:5000
FRONT_DOOR_URL=http://front-door:8080

# LDAP Configuration (optional)
LDAP_SERVER=ldap://corporate-ldap:389
LDAP_BASE_DN=dc=company,dc=com

# Database
POSTGRES_HOST=postgres
POSTGRES_DB=dsp_platform
POSTGRES_USER=dsp_user
POSTGRES_PASSWORD=secure_password

# Redis Cache
REDIS_HOST=redis
REDIS_PORT=6379
```"

### Step 3: Docker Compose Setup [04:00-06:00]
**Visual**: Docker Compose file
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7
    ports:
      - "6379:6379"

  control-tower:
    build: ./control-tower
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres/${POSTGRES_DB}
    depends_on:
      - postgres

  jwt-service:
    build: ./jwt-service
    ports:
      - "5000:5000"
    environment:
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
      - REDIS_HOST=${REDIS_HOST}
    depends_on:
      - redis

  front-door:
    build: ./front-door
    ports:
      - "8080:8080"
    environment:
      - CONTROL_TOWER_URL=${CONTROL_TOWER_URL}
      - JWT_SERVICE_URL=${JWT_SERVICE_URL}
    depends_on:
      - control-tower
      - jwt-service

  apisix:
    image: apache/apisix:latest
    ports:
      - "9080:9080"
      - "9091:9091"
    volumes:
      - ./apisix_conf/apisix.yaml:/usr/local/apisix/conf/apisix.yaml
      - ./apisix_conf/config.yaml:/usr/local/apisix/conf/config.yaml
    depends_on:
      - etcd

  etcd:
    image: bitnami/etcd:latest
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes

volumes:
  postgres_data:
```

### Step 4: Initialize System [06:00-07:30]
**Visual**: Initialization commands
```bash
# Start all services
docker-compose up -d

# Wait for services to be ready
sleep 10

# Create initial manifest
curl -X POST http://localhost:8000/manifests \
  -H "Content-Type: application/json" \
  -d @initial-manifest.json

# Create admin user
curl -X POST http://localhost:5000/admin/users \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123","roles":["admin"]}'

# Configure APISIX routes
curl -X PUT http://localhost:9080/apisix/admin/routes/1 \
  -H "X-API-KEY: admin-key" \
  -d @apisix-route.json
```

### Step 5: Verification [07:30-09:00]
**Visual**: Testing the system
```bash
# Get JWT token
TOKEN=$(curl -s -X POST http://localhost:5000/token \
  -d '{"username":"admin","password":"admin123","api_key":"test_key"}' \
  | jq -r '.access_token')

# Test Control Tower
curl http://localhost:8000/manifests \
  -H "Authorization: Bearer $TOKEN"

# Test Front Door routing
curl http://localhost:8080/test-project/inference/v1/models \
  -H "Authorization: Bearer $TOKEN"

# Check system health
curl http://localhost:8080/health
curl http://localhost:8000/health
curl http://localhost:5000/health
```

### Troubleshooting [09:00-09:40]
**Visual**: Common issues and solutions
**Script**: "Common issues:
1. Port conflicts: Change ports in docker-compose.yml
2. Memory issues: Increase Docker memory to 4GB+
3. Network issues: Ensure containers are on same network
4. Token errors: Verify JWT_SECRET_KEY matches across services"

### Closing [09:40-10:00]
**Visual**: Success confirmation
**Script**: "Excellent! Your DSP AI platform is running. All components are integrated and ready. Next episode: implementing a real-world AI use case!"

---

## Episode INT-02: Real-World Use Case (10 minutes)

### Opening [00:00-00:20]
**Visual**: Use case overview
**Script**: "Let's implement a real-world scenario: a multi-tenant AI customer service platform with tiered access, using everything we've learned!"

### Use Case Requirements [00:20-01:00]
**Visual**: Requirements diagram
**Script**: "Our customer service platform needs:
- Multi-tenant isolation
- Tiered access (Basic, Premium, Enterprise)
- Multiple LLM models
- RAG for knowledge base
- Usage tracking and quotas
- Real-time monitoring"

### Creating the Manifest [01:00-03:00]
**Visual**: Manifest creation
```json
{
  "project_id": "ai-customer-service",
  "name": "AI Customer Service Platform",
  "version": "1.0.0",
  "environment": "production",
  "modules": [
    {
      "name": "auth",
      "type": "jwt_config",
      "config": {
        "secret_key": "${JWT_SECRET}",
        "expiration_minutes": 60,
        "refresh_enabled": true
      }
    },
    {
      "name": "llm-inference",
      "type": "inference_endpoint",
      "config": {
        "models": {
          "basic": ["gpt-3.5-turbo"],
          "premium": ["gpt-4", "claude-3"],
          "enterprise": ["gpt-4", "claude-3", "custom-model"]
        },
        "endpoints": {
          "openai": "${OPENAI_ENDPOINT}",
          "anthropic": "${ANTHROPIC_ENDPOINT}"
        }
      }
    },
    {
      "name": "knowledge-base",
      "type": "rag_config",
      "config": {
        "vector_store": "faiss",
        "embedding_model": "text-embedding-ada-002",
        "chunk_size": 1000,
        "overlap": 200
      }
    }
  ]
}
```

### Configuring API Keys [03:00-04:30]
**Visual**: API key configurations
```yaml
# Premium tier configuration
id: premium-customer
owner: Premium Customer Inc
provider_permissions:
  - openai
  - anthropic
claims:
  static:
    tier: premium
    rate_limit: 1000
    models: ["gpt-4", "claude-3"]
  dynamic:
    usage_quota:
      type: api
      url: "http://billing/api/quota/{customer_id}"
    knowledge_bases:
      type: function
      module: claims.customer
      function: get_knowledge_bases
      args:
        customer_id: "{customer_id}"
```

### Implementing the Flow [04:30-07:00]
**Visual**: Complete request flow
**Script**: "Let's trace a customer query:

1. **Customer authenticates**:
```bash
TOKEN=$(curl -X POST http://localhost:5000/token \
  -d '{"username":"customer1","password":"pass","api_key":"premium-customer"}' \
  | jq -r '.access_token')
```

2. **Query with context**:
```bash
curl -X POST http://localhost:8080/ai-customer-service/chat/completions \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "How do I reset my password?"}],
    "knowledge_base": "customer-docs",
    "include_context": true
  }'
```

3. **System processes**:
- Front Door validates token
- Checks tier permissions
- RAG retrieves relevant docs
- LLM generates response
- Usage tracked and logged"

### Monitoring & Observability [07:00-08:30]
**Visual**: Monitoring dashboards
**Script**: "Monitor your platform with:

**Prometheus metrics**:
```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'dsp-platform'
    static_configs:
      - targets: 
        - 'front-door:8080'
        - 'control-tower:8000'
        - 'jwt-service:5000'
```

**Grafana dashboards**:
- Request rates by tier
- Token generation metrics
- Model usage distribution
- Error rates and latencies

**Langfuse for LLM tracking**:
- Token usage per customer
- Response quality metrics
- Cost tracking by tier"

### Scaling Considerations [08:30-09:30]
**Visual**: Scaling architecture
**Script**: "Scale your platform:

1. **Horizontal scaling**:
```yaml
front-door:
  deploy:
    replicas: 3
  load_balancer:
    strategy: round-robin
```

2. **Caching strategy**:
- Redis for token caching
- Response caching for common queries
- Manifest caching in Front Door

3. **Database optimization**:
- Read replicas for Control Tower
- Connection pooling
- Query optimization"

### Closing [09:30-10:00]
**Visual**: Complete platform overview
**Script**: "Congratulations! You've built a production-ready AI platform with authentication, routing, and policy management. You understand the complete architecture and can customize it for any use case. Thank you for joining this tutorial series!"

## Resources & Commands Summary

### Quick Start Commands
```bash
# Clone and setup
git clone [repositories]
docker-compose up -d

# Generate token
TOKEN=$(curl -X POST http://localhost:5000/token -d @creds.json | jq -r '.access_token')

# Test system
curl http://localhost:8080/health -H "Authorization: Bearer $TOKEN"
```

### Configuration Files
- `.env` - Environment variables
- `docker-compose.yml` - Service orchestration
- `manifest.json` - Project configuration
- `api_keys/*.yaml` - API key configs

### Monitoring Endpoints
- `/health` - Service health
- `/metrics` - Prometheus metrics
- `/docs` - API documentation

### Documentation
- [Complete Documentation](../README.md)
- [API Reference](../docs/API.md)
- [Deployment Guide](../docs/DEPLOYMENT.md)
