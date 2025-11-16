# Front Door - Security Design & Implementation

> This guide describes the security architecture of the Front Door (FD2) service, including authentication, authorization, rate limiting, input validation, and threat protection.

---

## Security Architecture

### Defense in Depth

```text
┌─────────────────────────────────────────────────────────────┐
│                    Security Layers                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Layer 1: Network Security                                  │
│    ├─> TLS/SSL termination                                  │
│    ├─> IP whitelisting                                      │
│    └─> DDoS protection                                      │
│                                                             │
│  Layer 2: Authentication                                    │
│    ├─> JWT bearer tokens                                    │
│    ├─> API key validation                                   │
│    └─> OAuth 2.0 support                                    │
│                                                             │
│  Layer 3: Authorization                                     │
│    ├─> Role-based access control (RBAC)                     │
│    ├─> Resource-level permissions                           │
│    └─> Policy-based access (OPA)                            │
│                                                             │
│  Layer 4: Rate Limiting                                     │
│    ├─> Per-user limits                                      │
│    ├─> Per-IP limits                                        │
│    └─> Adaptive throttling                                  │
│                                                             │
│  Layer 5: Input Validation                                  │
│    ├─> Request schema validation                            │
│    ├─> SQL injection prevention                             │
│    └─> XSS protection                                       │
│                                                             │
│  Layer 6: Audit & Monitoring                                │
│    ├─> Request logging                                      │
│    ├─> Security event tracking                              │
│    └─> Anomaly detection                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Authentication

### JWT Bearer Token Authentication

```python
from fastapi import Request, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt
from typing import Optional, Dict, Any
import os

class JWTBearer(HTTPBearer):
    def __init__(self, auto_error: bool = True):
        super().__init__(auto_error=auto_error)
        self.secret_key = os.getenv("JWT_SECRET_KEY")
        self.algorithm = "HS256"
        self.issuer = "dsp-ai-platform"
        
    async def __call__(self, request: Request):
        credentials: HTTPAuthorizationCredentials = await super().__call__(request)

        if not credentials:
            raise HTTPException(status_code=403, detail="Invalid authorization code")

        if credentials.scheme != "Bearer":
            raise HTTPException(status_code=403, detail="Invalid authentication scheme")

        payload = self.verify_jwt(credentials.credentials)
        if not payload:
            raise HTTPException(status_code=403, detail="Invalid or expired token")

        # Attach user context to request
        request.state.user = payload
        return payload
    
    def verify_jwt(self, token: str) -> Optional[Dict[str, Any]]:
        try:
            payload = jwt.decode(
                token,
                self.secret_key,
                algorithms=[self.algorithm],
                issuer=self.issuer,
                options={"verify_exp": True},
            )

            if not self.validate_claims(payload):
                return None

            return payload
        except jwt.ExpiredSignatureError:
            return None
        except jwt.InvalidTokenError:
            return None

    def validate_claims(self, payload: Dict[str, Any]) -> bool:
        required_claims = ["sub", "iat", "exp", "roles"]
        return all(claim in payload for claim in required_claims)

jwt_bearer = JWTBearer()

@app.get("/protected", dependencies=[Depends(jwt_bearer)])
async def protected_route(request: Request):
    user = request.state.user
    return {"message": f"Hello {user['sub']}", "roles": user["roles"]}
```

### API Key Authentication (Pattern)

Front Door can also support header- or query-based API keys that are validated against a configuration store (e.g., Redis, Postgres, or JWT service). Typical checks include:

- Key format validation.
- Active/disabled status.
- Rate-limit metadata lookup.

---

## Authorization & RBAC

- Enforce **role-based access control** using claims in JWT (`roles`, `permissions`).
- Support **resource-level rules**, e.g. `resource:project:read`, `resource:manifest:write`.
- Integrate with **OPA** (Open Policy Agent) for complex policies:
  - Route-level allow/deny.
  - Attribute-based access control (ABAC) on metadata.

Policy decisions can be evaluated in a dedicated middleware or module before requests are forwarded downstream.

---

## Rate Limiting

Typical patterns:

- **Per-API-key** rate limits (RPM/TPM).
- **Per-user / per-tenant** limits from JWT claims.
- **Per-IP** fallback limits.
- Adaptive throttling during incidents.

Implementation options:

- APISIX / gateway plugins.
- Redis-backed token bucket.

---

## Input Validation & Sanitization

Example of Pydantic-based request validation:

```python
from pydantic import BaseModel, validator, Field
from typing import Optional
import re
import html
import os

class SecureRequestModel(BaseModel):
    class Config:
        anystr_strip_whitespace = True
        max_anystr_length = 10000

    @validator('*', pre=True)
    def sanitize_input(cls, v):
        if isinstance(v, str):
            v = v.replace('\x00', '')
            v = html.escape(v)
            v = ''.join(ch for ch in v if ord(ch) >= 32 or ch == '\n')
        return v

class QueryRequest(SecureRequestModel):
    query: str = Field(..., min_length=1, max_length=1000)
    context: Optional[str] = Field(None, max_length=5000)

    @validator('query')
    def validate_query(cls, v):
        sql_patterns = [
            r"(\b(SELECT|INSERT|UPDATE|DELETE|DROP|UNION|ALTER)\b)",
            r"(--|#|/\*|\*/)",
            r"(\bOR\b.*=.*)",
            r"(\bAND\b.*=.*)",
        ]
        for pattern in sql_patterns:
            if re.search(pattern, v, re.IGNORECASE):
                raise ValueError("Potentially malicious input detected")
        return v

class FileUploadRequest(SecureRequestModel):
    filename: str = Field(..., max_length=255)
    content_type: str = Field(..., regex="^[a-zA-Z0-9][a-zA-Z0-9/\-\+\.]*$")

    @validator('filename')
    def validate_filename(cls, v):
        v = os.path.basename(v)
        if '..' in v or '/' in v or '\\' in v:
            raise ValueError("Invalid filename")

        allowed_extensions = ['.pdf', '.txt', '.docx', '.json']
        if not any(v.endswith(ext) for ext in allowed_extensions):
            raise ValueError("File type not allowed")
        return v
```

---

## Security Headers

```python
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)

        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"
        response.headers["Content-Security-Policy"] = "default-src 'self'"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
        response.headers["Permissions-Policy"] = "geolocation=(), microphone=(), camera=()"

        # Strip server-identifying headers
        response.headers.pop("Server", None)
        response.headers.pop("X-Powered-By", None)

        return response

app.add_middleware(SecurityHeadersMiddleware)
```

---

## Audit Logging

```python
import json
import logging
from datetime import datetime
from typing import Dict, Any

class SecurityAuditLogger:
    def __init__(self):
        self.logger = logging.getLogger("security_audit")
        handler = logging.FileHandler("security_audit.log")
        handler.setFormatter(logging.Formatter(
            '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
        ))
        self.logger.addHandler(handler)
        self.logger.setLevel(logging.INFO)

    def log_authentication(self, event_type: str, user_id: str, ip_address: str, success: bool, details: Dict[str, Any] | None = None):
        event = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": f"auth_{event_type}",
            "user_id": user_id,
            "ip_address": ip_address,
            "success": success,
            "details": details or {},
        }
        self.logger.info(json.dumps(event))

    def log_authorization(self, user_id: str, resource: str, action: str, allowed: bool, reason: str | None = None):
        event = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": "authorization",
            "user_id": user_id,
            "resource": resource,
            "action": action,
            "allowed": allowed,
            "reason": reason,
        }
        self.logger.info(json.dumps(event))

    def log_security_violation(self, violation_type: str, ip_address: str, details: Dict[str, Any]):
        event = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": "security_violation",
            "violation_type": violation_type,
            "ip_address": ip_address,
            "details": details,
        }
        self.logger.warning(json.dumps(event))
```

---

## Security Best Practices

Security checklist for Front Door:

- Use HTTPS/TLS in production.
- Require authentication for all endpoints.
- Enforce rate limiting (per API key, user, IP).
- Validate and sanitize all inputs.
- Log security events and review regularly.
- Keep dependencies up to date.
- Set strict security headers.
- Configure CORS carefully.
- Run regular security assessments.
- Maintain an incident response plan.

---

## Related Documentation

- Front Door Documentation.
- FD-Scalability-Resilience.
- FD-APISIX-Langfuse.
