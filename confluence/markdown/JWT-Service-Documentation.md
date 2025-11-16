# DSP AI JWT Service

> A Flask-based JWT token service designed for integration with APISIX and other gateways. It provides JWT token generation, validation, and dynamic claims based on API keys and authentication methods.

---

## Overview

### What is JWT Service?

The JWT Service is a centralized authentication service that generates and validates JSON Web Tokens (JWT) for the DSP AI Platform. It supports multiple authentication methods and flexible API key-based claims.

### Key Features

- **Multiple Authentication Methods** – LDAP and file-based user auth
- **Configurable JWT Tokens** – Shared secret with APISIX and services
- **Flexible API Key Configuration** – File-based and inline payload support
- **Dynamic Claims** – Function-based and API-based claim generation
- **Token Refresh** – Issue new tokens from refresh tokens
- **Token Decoding** – Verify and decode existing JWTs
- **Control Tower Integration** – Use manifest-driven configuration

---

## Architecture

### Token Generation Flow

```text
┌─────────┐
│ Client  │
└────┬────┘
     │ 1. POST /token {username, password, api_key}
     ▼
┌─────────────────────────────────────┐
│            JWT Service              │
├─────────────────────────────────────┤
│  Authentication Layer (LDAP/File)   │
│  API Key Resolution & Dynamic Claims│
│  JWT Encoding & Signing             │
└─────────────────────────────────────┘
     │ 2. JWT / Refresh Token
     ▼
┌─────────────────────────────────────┐
│    APISIX / Backend Services        │
└─────────────────────────────────────┘
```

---

## Configuration

### Environment Variables (Examples)

```bash
export JWT_SECRET_KEY="super-secret-key"
export JWT_ALGORITHM="HS256"
export ACCESS_TOKEN_EXPIRES_MIN=60
export REFRESH_TOKEN_EXPIRES_DAYS=7
export USERS_CONFIG_PATH="config/users.yaml"
export API_KEYS_PATH="config/api_keys"
```

### Users Configuration (File-Based Auth)

```yaml
# config/users.yaml
users:
  - username: alice
    password: "$2b$12$..."   # bcrypt hash
    roles: ["admin", "developer"]
  - username: bob
    password: "$2b$12$..."
    roles: ["viewer"]
```

### API Key Configuration

```yaml
# config/api_keys/example_key.yaml
api_key: sk-test-123
name: Example Client
active: true
claims:
  sub: example-client
  scopes:
    - rag:query
    - rag:documents
metadata:
  customer_id: cust_001
  plan: standard
```

---

## Endpoints

### Issue Token

```bash
curl -X POST http://localhost:5000/api/token \
  -H "Content-Type: application/json" \
  -d '{
    "username": "alice",
    "password": "secret",
    "api_key": "sk-test-123"
  }'
```

**Response (example)**

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "refresh_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 3600
}
```

### Refresh Token

```bash
curl -X POST http://localhost:5000/api/refresh \
  -H "Content-Type: application/json" \
  -d '{"refresh_token": "..."}'
```

### Decode / Introspect Token

```bash
curl -X POST http://localhost:5000/api/decode \
  -H "Content-Type: application/json" \
  -d '{"token": "eyJhbGciOiJIUzI1NiIs..."}'
```

---

## Dynamic Claims

JWT Service can enrich tokens with dynamic claims based on API key configuration.

### Function-Based Claims

```yaml
claims:
  dynamic:
    - type: function
      module: jwt_claims
      function: build_metadata_claims
      args:
        username: "{{username}}"
        roles: "{{roles}}"
```

### API-Based Claims

```yaml
claims:
  dynamic:
    - type: http
      url: https://internal-api/users/{{username}}/metadata
      method: GET
      headers:
        Authorization: "Bearer ${INTERNAL_API_TOKEN}"
      response_field: claims
```

---

## APISIX Integration

Configure APISIX to validate JWTs issued by the JWT Service.

```yaml
plugins:
  jwt-auth:
    key: ${JWT_SECRET_KEY}
    algorithm: HS256
    header: Authorization
    query: jwt
    cookie: jwt
```

In routes, you can require JWT auth and use claims for routing or authorization.

---

## Troubleshooting

### Authentication Failures

- Check user/password against `users.yaml` (file auth)
- Verify LDAP connectivity and credentials (LDAP auth)
- Inspect service logs for error details

### Token Rejected by Gateway

- Confirm `JWT_SECRET_KEY` matches on JWT Service and APISIX
- Check token expiry (`exp` claim)
- Verify algorithm (`alg`) matches configuration
- Validate issuer (`iss`) and audience (`aud`) if used

### Dynamic Claims Not Working

- Verify function module and function name
- Check external API endpoint availability
- Ensure placeholder substitution in args is correct
- Inspect logs for claim-generation errors

---

## Related Documentation

- Control Tower Documentation
- Front Door Documentation
- APISIX Integration Guide
- API Key Configuration Guide

---

## Additional Help

- API docs: `http://localhost:5000/docs`
- Example configurations in the JWT Service repository
- DSP AI Platform team support channels
