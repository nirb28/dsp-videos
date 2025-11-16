# DSP AI Platform Integration Guide

> This guide walks you through setting up the complete DSP AI Platform with Control Tower, Front Door, and JWT Service working together.

---

## System Overview

### Architecture Diagram

```text
┌─────────────────────────────────────────────────────────────┐
│                    DSP AI Platform                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐         │
│  │ Control  │◄────►│  Front   │◄────►│   JWT    │         │
│  │  Tower   │      │   Door   │      │ Service  │         │
│  └────┬─────┘      └────┬─────┘      └────┬─────┘         │
│       │                 │                  │               │
│       │ Manifests       │ Routing          │ Tokens        │
│       │                 │                  │               │
└───────┼─────────────────┼──────────────────┼───────────────┘
        │                 │                  │
        ▼                 ▼                  ▼
   ┌─────────┐      ┌──────────┐      ┌──────────┐
   │ Postgres│      │  APISIX  │      │  Redis   │
   └─────────┘      └──────────┘      └──────────┘
                          │
                          ▼
                   ┌──────────────┐
                   │   Backend    │
                   │   Services   │
                   └──────────────┘
```

### Component Responsibilities

| Component       | Port | Primary Function                                   |
|-----------------|------|----------------------------------------------------|
| **Control Tower** | 8000 | Configuration management and policy enforcement   |
| **Front Door**    | 8080 | API gateway and request routing                   |
| **JWT Service**   | 5000 | Authentication and token generation               |
| **APISIX**        | 9080 | Advanced gateway features (rate limiting, etc.)   |
| **PostgreSQL**    | 5432 | Persistent storage for configurations             |
| **Redis**         | 6379 | Caching and session storage                       |

---

## Prerequisites

**System requirements**

- **Operating system**: Linux, macOS, or Windows with WSL2
- **Docker**: 20.10+
- **Docker Compose**: 2.0+
- **Python**: 3.11+
- **Memory**: ≥ 8 GB RAM
- **Disk space**: ≥ 20 GB free

---

## Quick Start with Docker Compose

### Step 1: Clone Repositories

```bash
# Create workspace directory
mkdir dsp-platform && cd dsp-platform

# Clone all repositories
git clone https://github.com/dsp/control-tower.git
git clone https://github.com/dsp/front-door.git
git clone https://github.com/dsp/jwt-service.git
```

### Step 2: Create Environment Configuration

Create a `.env` file:

```bash
# JWT Configuration (MUST be identical across services)
JWT_SECRET_KEY=your-super-secure-secret-key-at-least-32-characters-long
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=30

# Service URLs
CONTROL_TOWER_URL=http://control-tower:8000
JWT_SERVICE_URL=http://jwt-service:5000
FRONT_DOOR_URL=http://front-door:8080

# Database Configuration
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=dsp_platform
POSTGRES_USER=dsp_user
POSTGRES_PASSWORD=secure_db_password_change_me

# Redis Configuration
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0

# Optional: LDAP Configuration
LDAP_SERVER=ldap://ldap.company.com:389
LDAP_BASE_DN=dc=company,dc=com

# Monitoring
PROMETHEUS_ENABLED=true
JAEGER_ENABLED=true
LOG_LEVEL=INFO
```

### Step 3: Create Docker Compose File

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: dsp-postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: dsp-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Control Tower
  control-tower:
    build: ./control-tower
    container_name: dsp-control-tower
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
      - REDIS_URL=redis://redis:6379
      - LOG_LEVEL=${LOG_LEVEL}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./control-tower/manifests:/app/manifests
      - ./control-tower/policies:/app/policies
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # JWT Service
  jwt-service:
    build: ./jwt-service
    container_name: dsp-jwt-service
    ports:
      - "5000:5000"
    environment:
      - JWT_SECRET_KEY=${JWT_SECRET_KEY}
      - JWT_ALGORITHM=${JWT_ALGORITHM}
      - JWT_EXPIRATION_MINUTES=${JWT_EXPIRATION_MINUTES}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - LDAP_SERVER=${LDAP_SERVER}
      - LDAP_BASE_DN=${LDAP_BASE_DN}
      - LOG_LEVEL=${LOG_LEVEL}
    depends_on:
      redis:
        condition: service_healthy
    volumes:
      - ./jwt-service/config:/app/config
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Front Door
  front-door:
    build: ./front-door
    container_name: dsp-front-door
    ports:
      - "8080:8080"
    environment:
      - CONTROL_TOWER_URL=http://control-tower:8000
      - JWT_SERVICE_URL=http://jwt-service:5000
      - REDIS_URL=redis://redis:6379
      - MODULE_POOL_SIZE=10
      - CACHE_TTL_SECONDS=300
      - LOG_LEVEL=${LOG_LEVEL}
    depends_on:
      control-tower:
        condition: service_healthy
      jwt-service:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # APISIX
  apisix:
    image: apache/apisix:3.6.0-debian
    container_name: dsp-apisix
    ports:
      - "9080:9080"
      - "9091:9091"
      - "9443:9443"
    environment:
      - JWT_SECRET=${JWT_SECRET_KEY}
    volumes:
      - ./apisix/config.yaml:/usr/local/apisix/conf/config.yaml
      - ./apisix/apisix.yaml:/usr/local/apisix/conf/apisix.yaml
    depends_on:
      - etcd
      - front-door

  # etcd for APISIX
  etcd:
    image: bitnami/etcd:3.5
    container_name: dsp-etcd
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
    ports:
      - "2379:2379"

volumes:
  postgres_data:
  redis_data:
```

### Step 4: Start the Platform

```bash
# Start all services
docker-compose up -d

# Wait for services to be ready (about 30 seconds)
sleep 30

# Verify all services are healthy
docker-compose ps
```

---

## Initial Configuration

### Step 1: Create Initial Manifest

```bash
# Create a basic manifest
curl -X POST http://localhost:8000/manifests \
  -H "Content-Type: application/json" \
  -d '{
    "project_id": "demo-project",
    "name": "Demo AI Project",
    "version": "1.0.0",
    "environment": "development",
    "modules": [
      {
        "name": "auth",
        "type": "jwt_config",
        "config": {
          "secret_key": "'${JWT_SECRET_KEY}'",
          "algorithm": "HS256",
          "expiration_minutes": 30
        }
      }
    ]
  }'
```

### Step 2: Create Test User

```bash
# Add user to jwt-service/config/users.yaml
cat > jwt-service/config/users.yaml << EOF
demo_user:
  password: 5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
  name: Demo User
  email: demo@example.com
  groups: [developers]
  roles: [developer]
EOF

# Restart JWT service to load new user
docker-compose restart jwt-service
```

### Step 3: Create API Key Configuration

```bash
# Create API key config
cat > jwt-service/config/api_keys/demo_key.yaml << EOF
id: demo-service
owner: Demo Team
provider_permissions:
  - openai
  - local
endpoint_permissions:
  - /v1/chat/completions
claims:
  static:
    tier: basic
    rate_limit: 100
    models: [gpt-3.5-turbo]
EOF

# Restart JWT service
docker-compose restart jwt-service
```

---

## Testing the Integration

### Test 1: Health Checks

```bash
# Check all services
echo "Control Tower:"
curl http://localhost:8000/health

echo "\nJWT Service:"
curl http://localhost:5000/health

echo "\nFront Door:"
curl http://localhost:8080/health
```

### Test 2: Authentication Flow

```bash
# Get JWT token
TOKEN_RESPONSE=$(curl -s -X POST http://localhost:5000/token \
  -H "Content-Type: application/json" \
  -d '{
    "username": "demo_user",
    "password": "password",
    "api_key": "demo_key"
  }')

# Extract access token
ACCESS_TOKEN=$(echo $TOKEN_RESPONSE | jq -r '.access_token')

echo "Token obtained: ${ACCESS_TOKEN:0:50}..."

# Decode token to verify claims
curl -X POST http://localhost:5000/decode \
  -H "Content-Type: application/json" \
  -d "{\"token\": \"$ACCESS_TOKEN\"}" | jq .
```

### Test 3: End-to-End Request

```bash
# Make request through Front Door
curl -X POST http://localhost:8080/demo-project/inference/v1/chat \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "user", "content": "Hello, DSP AI Platform!"}
    ]
  }'
```

---

## Monitoring and Observability

### Viewing Logs

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f control-tower
docker-compose logs -f jwt-service
docker-compose logs -f front-door

# View last 100 lines
docker-compose logs --tail=100
```

### Metrics Endpoints

| Service        | Metrics URL                                          | Format      |
|----------------|------------------------------------------------------|-------------|
| Control Tower  | `http://localhost:8000/metrics`                     | Prometheus  |
| Front Door     | `http://localhost:8080/metrics`                     | Prometheus  |
| APISIX         | `http://localhost:9091/apisix/prometheus/metrics`   | Prometheus  |

---

## Common Integration Patterns

### Pattern 1: Multi-Tenant Application

```yaml
# tenant_a_key.yaml
id: tenant-a
owner: Company A
claims:
  static:
    tenant_id: tenant-a
    tier: premium
    rate_limit: 1000
  dynamic:
    tenant_resources:
      type: api
      url: "http://tenant-service/api/resources/{tenant_id}"

# tenant_b_key.yaml
id: tenant-b
owner: Company B
claims:
  static:
    tenant_id: tenant-b
    tier: basic
    rate_limit: 100
```

### Pattern 2: Tiered Access Control

```json
{
  "modules": [
    {
      "name": "basic-inference",
      "type": "inference_endpoint",
      "config": {
        "models": ["gpt-3.5-turbo"],
        "rate_limit": 100
      }
    },
    {
      "name": "premium-inference",
      "type": "inference_endpoint",
      "config": {
        "models": ["gpt-4", "claude-3"],
        "rate_limit": 1000
      },
      "access_control": {
        "required_tier": "premium"
      }
    }
  ]
}
```

### Pattern 3: Environment-Specific Configuration

```json
{
  "project_id": "ai-service",
  "environment": "production",
  "environments": {
    "development": {
      "urls": {
        "llm_endpoint": "http://dev-llm:8000"
      },
      "config": {
        "rate_limit": 1000,
        "timeout": 60
      }
    },
    "production": {
      "urls": {
        "llm_endpoint": "https://prod-llm.company.com"
      },
      "config": {
        "rate_limit": 10000,
        "timeout": 30
      }
    }
  },
  "modules": [
    {
      "name": "inference",
      "config": {
        "endpoint": "${environments.${environment}.urls.llm_endpoint}",
        "rate_limit": "${environments.${environment}.config.rate_limit}"
      }
    }
  ]
}
```

---

## Troubleshooting

### Services Won't Start

- Check port conflicts: `netstat -an | findstr "8000 8080 5000"`
- Verify Docker resources (≥ 8 GB RAM)
- Check logs: `docker-compose logs`
- Ensure `.env` exists and is correctly formatted
- Try clean restart: `docker-compose down -v && docker-compose up -d`

### JWT Token Validation Fails

- Verify `JWT_SECRET_KEY` is identical in all services
- Check token expiration
- Ensure `JWT_ALGORITHM` matches (HS256)
- Review JWT Service logs
- Test token with the decode endpoint

### Routing Not Working

- Verify manifest exists in Control Tower
- Check module name in URL matches manifest
- Review Front Door logs for routing
- Ensure backend service is reachable from Front Door
- Test Control Tower connectivity from Front Door

---

## Production Deployment Checklist

**Pre-production checklist**

- ☐ Change all default passwords and secrets
- ☐ Enable HTTPS/TLS for all services
- ☐ Configure firewall rules
- ☐ Set up database backups
- ☐ Configure log aggregation
- ☐ Set up monitoring and alerting
- ☐ Implement rate limiting
- ☐ Configure auto-scaling policies
- ☐ Test disaster recovery procedures
- ☐ Document runbooks for common issues
- ☐ Set up CI/CD pipelines
- ☐ Perform security audit

---

## Related Documentation

- Control Tower Documentation
- Front Door Documentation
- JWT Service Documentation

> For additional support:
>
> - Review component-specific documentation.
> - Check example configurations in repositories.
> - Contact the DSP AI Platform team.
