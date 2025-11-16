# JWT Service - JWE Encryption & Advanced Features

> This guide covers JSON Web Encryption (JWE) in the JWT Service, including encrypted tokens, secure payload handling, dynamic API key configurations, and advanced token types.

---

## JWE Overview

### What is JWE?

JSON Web Encryption (JWE) adds **confidentiality** to JWT tokens. The payload is encrypted so that even if a token is intercepted, its contents cannot be read without the decryption key.

### JWE Structure

```text
Standard JWT (JWS):
header.payload.signature
↓
Base64-encoded, readable payload

JWE Token:
header.encrypted_key.iv.ciphertext.tag
↓
Encrypted payload, unreadable without key
```

---

## JWE Implementation

### JWE Handler Class

```python
# jwe_handler.py
from jose import jwe
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2
import json
import base64
from typing import Dict, Any, Optional

class JWEHandler:
    def __init__(self, encryption_key: str):
        """Initialize JWE handler with encryption key"""
        self.algorithm = "dir"       # Direct key agreement
        self.encryption = "A256GCM"  # AES-256-GCM
        self.key = self._derive_key(encryption_key)
    
    def _derive_key(self, passphrase: str) -> str:
        """Derive 256-bit key from passphrase"""
        kdf = PBKDF2(
            algorithm=hashes.SHA256(),
            length=32,
            salt=b'dsp-ai-platform-salt',
            iterations=100000,
        )
        raw = kdf.derive(passphrase.encode())
        return base64.urlsafe_b64encode(raw).decode("utf-8")
    
    def encrypt_token(self, payload: Dict[str, Any]) -> str:
        """Encrypt JWT payload into a JWE token"""
        try:
            payload_json = json.dumps(payload)
            encrypted = jwe.encrypt(
                payload_json.encode("utf-8"),
                self.key,
                algorithm=self.algorithm,
                encryption=self.encryption,
            )
            return encrypted.decode("utf-8")
        except Exception as e:
            raise ValueError(f"JWE encryption failed: {e}")
    
    def decrypt_token(self, token: str) -> Dict[str, Any]:
        """Decrypt a JWE token back to a payload"""
        try:
            decrypted = jwe.decrypt(token.encode("utf-8"), self.key)
            return json.loads(decrypted.decode("utf-8"))
        except Exception as e:
            raise ValueError(f"JWE decryption failed: {e}")
    
    def create_encrypted_jwt(self, claims: Dict[str, Any], additional_headers: Optional[Dict[str, Any]] = None) -> str:
        """Create an encrypted JWT with standard and custom claims"""
        import time
        now = int(time.time())

        claims.update({
            "iat": now,
            "exp": now + 3600,
            "nbf": now,
            "jti": self._generate_jti(),
        })

        headers = {
            "typ": "JWE",
            "alg": self.algorithm,
            "enc": self.encryption,
        }
        if additional_headers:
            headers.update(additional_headers)

        # For this implementation headers are not explicitly encoded; they can be
        # included in a higher-level JWE envelope if needed.
        return self.encrypt_token(claims)
    
    def _generate_jti(self) -> str:
        import uuid
        return str(uuid.uuid4())
```

---

## API Key Configuration

### Dynamic API Key Configuration Model

```python
# api_key_config.py
from typing import Dict, Any, List, Optional
from pydantic import BaseModel
import yaml
import os

class ModelConfig(BaseModel):
    name: str
    endpoint: str
    api_key: str
    max_tokens: int = 1024
    temperature: float = 0.7

class RateLimitConfig(BaseModel):
    requests_per_minute: int = 60
    tokens_per_minute: int = 10000
    requests_per_hour: int = 1000
    concurrent_requests: int = 5

class APIKeyConfig(BaseModel):
    api_key: str
    name: str
    type: str  # 'tiered_model_exec', 'function_call', 'api_call'
    models: List[ModelConfig] = []
    rate_limits: RateLimitConfig
    metadata: Dict[str, Any] = {}
    permissions: List[str] = []
    expires_at: Optional[str] = None
    active: bool = True

class APIKeyManager:
    def __init__(self, config_path: str):
        self.config_path = config_path
        self.configs = self._load_configs()

    def _load_configs(self) -> Dict[str, APIKeyConfig]:
        configs: Dict[str, APIKeyConfig] = {}
        for filename in os.listdir(self.config_path):
            if filename.endswith(".yaml"):
                with open(os.path.join(self.config_path, filename)) as f:
                    data = yaml.safe_load(f)
                    cfg = APIKeyConfig(**data)
                    configs[cfg.api_key] = cfg
        return configs

    def get_config(self, api_key: str) -> Optional[APIKeyConfig]:
        return self.configs.get(api_key)

    def create_dynamic_claims(self, api_key: str, request_context: Dict[str, Any]) -> Dict[str, Any]:
        cfg = self.get_config(api_key)
        if not cfg:
            return {}

        claims: Dict[str, Any] = {
            "api_key_name": cfg.name,
            "type": cfg.type,
            "permissions": cfg.permissions,
            "metadata": cfg.metadata,
        }

        if cfg.type == "tiered_model_exec":
            claims["allowed_models"] = [m.name for m in cfg.models]
            claims["rate_limits"] = cfg.rate_limits.dict()

        # Additional types (function_call, api_call) could use
        # request_context to build richer, dynamic claims.
        return claims
```

### Example Enterprise API Key Configuration

```yaml
# api_key_enterprise.yaml
api_key: sk-enterprise-001
name: Enterprise Customer API Key
type: tiered_model_exec
active: true
expires_at: "2025-12-31T23:59:59Z"

models:
  - name: gpt-4
    endpoint: https://api.openai.com/v1/chat/completions
    api_key: ${OPENAI_API_KEY}
    max_tokens: 4096
    temperature: 0.7

  - name: claude-3
    endpoint: https://api.anthropic.com/v1/messages
    api_key: ${ANTHROPIC_API_KEY}
    max_tokens: 4096
    temperature: 0.7

rate_limits:
  requests_per_minute: 100
  tokens_per_minute: 50000
  requests_per_hour: 5000
  concurrent_requests: 10

permissions:
  - read:documents
  - write:documents
  - execute:models
  - admin:configurations

metadata:
  customer_id: cust_12345
  tier: enterprise
  department: engineering
  cost_center: AI-001
  billing_contact: billing@example.com
```

---

## Advanced Token Features

Typical advanced token patterns (as implemented in the HTML version):

- **Delegation tokens** – `delegator`, `delegated_to`, `permissions`, `expires_at`.
- **Refresh tokens** – long-lived tokens bound to `session_id` and optional `device_id`.
- **Service tokens** – `sub`/`aud` for service-to-service calls, short TTL.

These are built on top of the same `JWEHandler` and claims model, adding type-specific fields and validation.

---

## Token Validation

### Advanced Validation Logic

```python
# token_validator.py
from datetime import datetime
from typing import Dict, Any, List, Optional

class TokenValidator:
    def __init__(self, jwe_handler: JWEHandler):
        self.jwe = jwe_handler
        self.revoked_tokens: set[str] = set()  # In production, use Redis

    def validate_token(
        self,
        token: str,
        required_scopes: Optional[List[str]] = None,
        required_audience: Optional[str] = None,
    ) -> Dict[str, Any]:
        """Comprehensive token validation"""
        # Decrypt JWE
        claims = self.jwe.decrypt_token(token)

        # Revocation check
        jti = claims.get("jti")
        if jti and jti in self.revoked_tokens:
            raise ValueError("Token has been revoked")

        # Expiration
        if "exp" in claims and datetime.fromtimestamp(claims["exp"]) < datetime.utcnow():
            raise ValueError("Token has expired")

        # Not-before
        if "nbf" in claims and datetime.fromtimestamp(claims["nbf"]) > datetime.utcnow():
            raise ValueError("Token not yet valid")

        # Audience
        if required_audience:
            aud = claims.get("aud")
            aud_list = aud if isinstance(aud, list) else [aud] if aud else []
            if required_audience not in aud_list:
                raise ValueError("Invalid audience")

        # Scopes
        if required_scopes:
            token_scopes = claims.get("scopes", [])
            if not all(scope in token_scopes for scope in required_scopes):
                raise ValueError("Insufficient scopes")

        # Token type
        token_type = claims.get("type")
        if token_type == "refresh":
            raise ValueError("Refresh token cannot be used for API access")

        return claims

    def revoke_token(self, token: str) -> None:
        try:
            claims = self.jwe.decrypt_token(token)
            jti = claims.get("jti")
            if jti:
                self.revoked_tokens.add(jti)
        except Exception:
            # Ignore already-invalid tokens
            pass
```

---

## Security Best Practices for JWE

- Use strong encryption keys (256-bit minimum).
- Rotate encryption keys regularly.
- Store keys in a secure vault (Vault, KMS, etc.).
- Use distinct keys for different environments.
- Implement key versioning to support smooth rotation.
- Monitor for suspicious token usage patterns.
- Provide token revocation and blacklist support.
- Use short expiry for sensitive operations.
- Audit token generation and validation events.
- Rate-limit per API key and per user.

---

## Related Documentation

- JWT Service Documentation.
- JWT-API-Key-Management.
- JWT-LDAP-Integration.
