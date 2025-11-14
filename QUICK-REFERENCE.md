# DSP AI Platform - Quick Reference for Video Tutorials

## 🚀 Quick Setup Commands

### All-in-One Setup
```bash
# Clone all repositories
git clone https://github.com/dsp/control-tower.git
git clone https://github.com/dsp/front-door.git  
git clone https://github.com/dsp/jwt-service.git

# Start everything with Docker Compose
docker-compose -f docker-compose.all.yml up -d

# Verify all services are running
curl http://localhost:8000/health  # Control Tower
curl http://localhost:8080/health  # Front Door
curl http://localhost:5000/health  # JWT Service
```

---

## 📝 Sample Configurations

### Control Tower Manifest
```json
{
  "project_id": "demo-project",
  "name": "Demo AI Project",
  "version": "1.0.0",
  "environment": "development",
  "modules": [
    {
      "name": "auth",
      "type": "jwt_config",
      "config": {
        "secret_key": "${JWT_SECRET}",
        "algorithm": "HS256",
        "expiration_minutes": 30
      }
    },
    {
      "name": "inference",
      "type": "inference_endpoint",
      "config": {
        "model": "gpt-3.5-turbo",
        "endpoint": "http://localhost:8001",
        "max_tokens": 2000,
        "temperature": 0.7
      }
    }
  ]
}
```

### JWT API Key Configuration
```yaml
# config/api_keys/demo_key.yaml
id: demo-service
owner: Demo Team
provider_permissions:
  - openai
  - local
endpoint_permissions:
  - /v1/chat/completions
  - /v1/embeddings
claims:
  static:
    tier: basic
    rate_limit: 100
    models:
      - gpt-3.5-turbo
  dynamic:
    quota:
      type: function
      module: claims.quota
      function: get_quota
      args:
        user_id: "{user_id}"
```

### Front Door Module Configuration
```json
{
  "module_type": "custom_inference",
  "runtime": {
    "type": "python:3.11",
    "implementation": "src.modules.custom.CustomModule"
  },
  "endpoints": {
    "dev": {
      "primary": "http://localhost:8002"
    },
    "prod": {
      "primary": "https://api.production.com"
    }
  },
  "configuration_references": [
    {
      "name": "api_key",
      "source": "vault://secrets/api_key",
      "required": true
    }
  ]
}
```

### Docker Compose Complete Stack
```yaml
version: '3.8'

services:
  # Database
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: dsp_platform
      POSTGRES_USER: dsp_user
      POSTGRES_PASSWORD: dsp_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  # Cache
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  # Control Tower
  control-tower:
    build: ./control-tower
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://dsp_user:dsp_password@postgres/dsp_platform
      - REDIS_URL=redis://redis:6379
    depends_on:
      - postgres
      - redis

  # JWT Service
  jwt-service:
    build: ./jwt-service
    ports:
      - "5000:5000"
    environment:
      - JWT_SECRET_KEY=your-secret-key-at-least-32-chars
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - redis

  # Front Door
  front-door:
    build: ./front-door
    ports:
      - "8080:8080"
    environment:
      - CONTROL_TOWER_URL=http://control-tower:8000
      - JWT_SERVICE_URL=http://jwt-service:5000
      - REDIS_URL=redis://redis:6379
    depends_on:
      - control-tower
      - jwt-service
      - redis

  # APISIX
  apisix:
    image: apache/apisix:3.6.0-debian
    ports:
      - "9080:9080"
      - "9091:9091"
      - "9443:9443"
    volumes:
      - ./apisix/config.yaml:/usr/local/apisix/conf/config.yaml
      - ./apisix/apisix.yaml:/usr/local/apisix/conf/apisix.yaml
    depends_on:
      - etcd

  # etcd for APISIX
  etcd:
    image: bitnami/etcd:3.5
    environment:
      - ALLOW_NONE_AUTHENTICATION=yes
      - ETCD_ADVERTISE_CLIENT_URLS=http://etcd:2379
    ports:
      - "2379:2379"

volumes:
  postgres_data:
```

---

## 🔑 Common API Calls

### Control Tower
```bash
# List all manifests
curl http://localhost:8000/manifests

# Create new manifest
curl -X POST http://localhost:8000/manifests \
  -H "Content-Type: application/json" \
  -d @manifest.json

# Get specific manifest
curl http://localhost:8000/manifests/demo-project

# Evaluate policy
curl -X POST http://localhost:8000/evaluate \
  -H "Content-Type: application/json" \
  -d '{
    "input_data": {
      "user": {"role": "developer"},
      "action": "deploy",
      "resource": {"type": "model"}
    },
    "client_id": "demo",
    "client_secret": "demo-secret"
  }'
```

### JWT Service
```bash
# Generate token
curl -X POST http://localhost:5000/token \
  -H "Content-Type: application/json" \
  -d '{
    "username": "demo_user",
    "password": "demo_password",
    "api_key": "demo_key"
  }'

# Refresh token
curl -X POST http://localhost:5000/refresh \
  -H "Authorization: Bearer <refresh_token>"

# Decode token
curl -X POST http://localhost:5000/decode \
  -H "Content-Type: application/json" \
  -d '{"token": "<jwt_token>"}'
```

### Front Door
```bash
# Health check
curl http://localhost:8080/health

# Call through gateway (with auth)
curl http://localhost:8080/demo-project/inference/v1/chat/completions \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-3.5-turbo",
    "messages": [
      {"role": "user", "content": "Hello!"}
    ]
  }'

# Get metrics
curl http://localhost:8080/metrics
```

---

## 🐛 Troubleshooting Commands

### Docker Issues
```bash
# View logs
docker-compose logs -f [service-name]

# Restart service
docker-compose restart [service-name]

# Clean restart
docker-compose down
docker-compose up -d

# Remove all volumes (careful!)
docker-compose down -v

# Check container health
docker ps --format "table {{.Names}}\t{{.Status}}"
```

### Network Debugging
```bash
# Test connectivity between containers
docker exec front-door ping control-tower

# Check exposed ports
netstat -an | findstr "8000 8080 5000"

# Test from inside container
docker exec -it front-door /bin/bash
curl http://control-tower:8000/health
```

### Service Health Checks
```bash
# Check all services
for port in 8000 8080 5000; do
  echo "Checking port $port:"
  curl -s http://localhost:$port/health | jq .
done

# Monitor logs
docker-compose logs -f --tail=50
```

---

## 📊 Environment Variables

### Essential Variables
```env
# JWT Configuration
JWT_SECRET_KEY=your-very-secure-secret-key-at-least-32-characters
JWT_ALGORITHM=HS256
JWT_EXPIRATION_MINUTES=30

# Service URLs
CONTROL_TOWER_URL=http://control-tower:8000
JWT_SERVICE_URL=http://jwt-service:5000
FRONT_DOOR_URL=http://front-door:8080

# Database
DATABASE_URL=postgresql://user:password@postgres:5432/dbname

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_DB=0

# LDAP (optional)
LDAP_SERVER=ldap://ldap.company.com:389
LDAP_BASE_DN=dc=company,dc=com

# Monitoring
PROMETHEUS_ENABLED=true
JAEGER_ENABLED=true
LOG_LEVEL=INFO
```

---

## 🎯 Testing Scenarios

### Scenario 1: Basic Authentication Flow
```bash
# 1. Get token
TOKEN_RESPONSE=$(curl -s -X POST http://localhost:5000/token \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test","api_key":"test_key"}')

# 2. Extract token
ACCESS_TOKEN=$(echo $TOKEN_RESPONSE | jq -r '.access_token')

# 3. Use token
curl http://localhost:8080/test/inference/chat \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -d '{"prompt":"Test"}'
```

### Scenario 2: Multi-Tenant Access
```bash
# Tenant A token
TOKEN_A=$(curl -s -X POST http://localhost:5000/token \
  -d '{"username":"tenant_a","password":"pass","api_key":"tenant_a_key"}' \
  | jq -r '.access_token')

# Tenant B token  
TOKEN_B=$(curl -s -X POST http://localhost:5000/token \
  -d '{"username":"tenant_b","password":"pass","api_key":"tenant_b_key"}' \
  | jq -r '.access_token')

# Different access levels
curl http://localhost:8080/service/endpoint -H "Authorization: Bearer $TOKEN_A"
curl http://localhost:8080/service/endpoint -H "Authorization: Bearer $TOKEN_B"
```

---

## 📚 Useful Scripts

### Health Check Script
```bash
#!/bin/bash
# health_check.sh

services=("control-tower:8000" "jwt-service:5000" "front-door:8080")

for service in "${services[@]}"; do
  IFS=':' read -r name port <<< "$service"
  echo -n "Checking $name... "
  
  if curl -s http://localhost:$port/health > /dev/null; then
    echo "✅ OK"
  else
    echo "❌ Failed"
  fi
done
```

### Token Generator Script
```python
#!/usr/bin/env python3
# generate_token.py

import requests
import json
import sys

def get_token(username, password, api_key):
    response = requests.post(
        "http://localhost:5000/token",
        json={
            "username": username,
            "password": password,
            "api_key": api_key
        }
    )
    
    if response.status_code == 200:
        data = response.json()
        print(f"Access Token: {data['access_token']}")
        print(f"Refresh Token: {data['refresh_token']}")
    else:
        print(f"Error: {response.status_code}")
        print(response.text)

if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("Usage: python generate_token.py <username> <password> <api_key>")
        sys.exit(1)
    
    get_token(sys.argv[1], sys.argv[2], sys.argv[3])
```

### Log Aggregator
```bash
#!/bin/bash
# aggregate_logs.sh

# Create logs directory
mkdir -p ./logs

# Export logs from each service
docker-compose logs control-tower > logs/control-tower.log
docker-compose logs jwt-service > logs/jwt-service.log
docker-compose logs front-door > logs/front-door.log

echo "Logs exported to ./logs directory"
```

---

## 🔗 Important URLs

### Service Endpoints
- Control Tower: http://localhost:8000
- Front Door: http://localhost:8080
- JWT Service: http://localhost:5000
- APISIX Admin: http://localhost:9080
- APISIX Dashboard: http://localhost:9000

### Documentation
- Control Tower Docs: http://localhost:8000/docs
- Front Door Docs: http://localhost:8080/docs
- JWT Service Docs: http://localhost:5000/docs

### Monitoring
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000
- Jaeger: http://localhost:16686

---

## 💡 Pro Tips

1. **Always use environment variables** for sensitive data
2. **Check logs first** when troubleshooting
3. **Use health endpoints** to verify service status
4. **Keep JWT secrets synchronized** across services
5. **Monitor rate limits** to prevent service overload
6. **Use correlation IDs** for request tracing
7. **Implement circuit breakers** for resilience
8. **Cache tokens** to reduce JWT service load
9. **Use connection pooling** for database connections
10. **Regular backup** of configurations and data

---

**Quick Command Cheat Sheet**

```bash
# Start all: docker-compose up -d
# Stop all: docker-compose down
# View logs: docker-compose logs -f
# Restart: docker-compose restart
# Status: docker ps
# Clean: docker-compose down -v
```

---

*Last Updated: November 2024*
