# Control Tower – HashiCorp Vault Integration

> Integrate HashiCorp Vault with Control Tower for secure secret management, dynamic credentials, and encryption services across the DSP AI Platform.

---

## Overview

### What is Vault Integration?

Vault provides centralized secret management and encryption-as-a-service. Integration with Control Tower allows manifests to reference Vault-stored secrets instead of hardcoding them.

### Key Features

- **Secret Storage** – Encrypted key-value store
- **Dynamic Credentials** – Temporary DB/API credentials
- **Encryption as a Service** – Encrypt/decrypt without storing keys
- **Audit Logging** – Full trace of secret usage
- **Access Control** – Policies and roles
- **Secret Rotation** – Automatic key and credential rotation

---

## Configuration

### Vault Module in Manifest

```json
{
  "name": "project-vault",
  "type": "vault",
  "config": {
    "instance_name": "primary-vault",
    "vault_url": "https://vault.example.com:8200",
    "auth_method": "token",
    "token": "${VAULT_TOKEN}",
    "kv_mount_point": "secret",
    "transit_mount_point": "transit",
    "database_mount_point": "database",
    "pki_mount_point": "pki",
    "namespace": "ai-platform",
    "ssl_verify": true
  }
}
```

### Using Vault Secrets in Other Modules

```json
{
  "name": "rag-config-main",
  "type": "rag_config",
  "config": {
    "database_url": "${vault:secret/data/databases/rag#data.url}",
    "api_key": "${vault:secret/data/api-keys/groq#data.api_key}"
  }
}
```

The `vault:` syntax is resolved by Control Tower’s environment/secret resolver.

---

## Secret Engines

Commonly used Vault engines:

- **KV (Key/Value)** – Secrets like API keys, passwords
- **Database** – Dynamic DB credentials
- **Transit** – Encryption and decryption
- **PKI** – Certificates and TLS management

### Example: KV Engine

```bash
vault kv put secret/api-keys/groq api_key="sk-..." env="production"
```

### Example: Database Engine

```bash
vault write database/config/ai-postgres \
  plugin_name=postgresql-database-plugin \
  allowed_roles="ai-*" \
  connection_url="postgresql://{{username}}:{{password}}@postgres:5432/ai_db"

vault write database/roles/ai-rag-service \
  db_name=ai-postgres \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}';" \
  default_ttl="1h" \
  max_ttl="24h"
```

---

## Best Practices

### Security Guidelines

- Use **Kubernetes auth** or another robust method in production
- Enable **audit logging** for all secret access
- Apply **least-privilege** Vault policies
- Prefer **dynamic credentials** over static ones
- Set reasonable **TTLs** for all secrets
- Use **Transit** for encryption instead of storing encrypted data
- Rotate static secrets regularly

### Example Vault Policy

```hcl
# Read API keys
path "secret/data/api-keys/*" {
  capabilities = ["read", "list"]
}

# Generate database credentials
path "database/creds/ai-*" {
  capabilities = ["read"]
}

# Encrypt/decrypt data
path "transit/encrypt/ai-platform" {
  capabilities = ["update"]
}

path "transit/decrypt/ai-platform" {
  capabilities = ["update"]
}

# Renew leases
path "sys/leases/renew" {
  capabilities = ["update"]
}
```

---

## Monitoring & Audit

### Enable Audit Logging

```bash
# File audit device
vault audit enable file \
  file_path=/vault/logs/audit.log \
  log_raw=false

# Syslog audit device
vault audit enable syslog \
  tag="vault" \
  facility="LOCAL7"

vault audit list
```

Monitor:

- Secret access patterns
- Lease creation and renewal
- Policy changes and admin actions

---

## Related Documentation

- Control Tower Documentation
- CT-LangGraph-Integration
- CT-Policy-Management
