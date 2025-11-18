---
layout: default
title:  "Secrets Management: HashiCorp Vault and Kubernetes"
date:   2025-11-18 12:00:00
categories: DevOps Security Vault Kubernetes
---

Proper secrets management is critical for security. For ML systems, this includes API keys, model signing keys, and database credentials. Here's how to implement it with Vault.

## HashiCorp Vault Setup

### Installation

```bash
# Install Vault via Helm
helm repo add hashicorp https://helm.releases.hashicorp.com

helm install vault hashicorp/vault \
  --namespace vault \
  --create-namespace \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3 \
  --set injector.enabled=true

# Initialize Vault
kubectl exec -it vault-0 -n vault -- vault operator init
```

### Configuration

```hcl
# vault-config.hcl
storage "raft" {
  path    = "/vault/data"
  node_id = "vault-0"

  retry_join {
    leader_api_addr = "http://vault-0.vault-internal:8200"
  }
}

listener "tcp" {
  address         = "0.0.0.0:8200"
  cluster_address = "0.0.0.0:8201"
  tls_disable     = false
  tls_cert_file   = "/vault/tls/tls.crt"
  tls_key_file    = "/vault/tls/tls.key"
}

seal "awskms" {
  region     = "us-east-1"
  kms_key_id = "alias/vault-auto-unseal"
}

api_addr     = "https://vault.example.com"
cluster_addr = "https://vault-0.vault-internal:8201"
ui           = true
```

## Kubernetes Authentication

### Enable Kubernetes Auth

```bash
# Enable auth method
vault auth enable kubernetes

# Configure Kubernetes auth
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  token_reviewer_jwt=@/var/run/secrets/kubernetes.io/serviceaccount/token
```

### Create Roles

```bash
# Create policy for ML services
vault policy write ml-service - <<EOF
path "secret/data/ml/*" {
  capabilities = ["read"]
}

path "database/creds/ml-readonly" {
  capabilities = ["read"]
}

path "pki/issue/ml-service" {
  capabilities = ["create", "update"]
}
EOF

# Create Kubernetes role
vault write auth/kubernetes/role/ml-inference \
  bound_service_account_names=ml-inference \
  bound_service_account_namespaces=ml-platform \
  policies=ml-service \
  ttl=1h
```

## Vault Agent Injector

### Pod with Secrets Injection

```yaml
# ml-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-inference
  namespace: ml-platform
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "ml-inference"

        # Inject API keys
        vault.hashicorp.com/agent-inject-secret-api-keys: "secret/data/ml/api-keys"
        vault.hashicorp.com/agent-inject-template-api-keys: |
          {{- with secret "secret/data/ml/api-keys" -}}
          export OPENAI_API_KEY="{{ .Data.data.openai }}"
          export HF_TOKEN="{{ .Data.data.huggingface }}"
          {{- end }}

        # Inject database credentials
        vault.hashicorp.com/agent-inject-secret-db: "database/creds/ml-readonly"
        vault.hashicorp.com/agent-inject-template-db: |
          {{- with secret "database/creds/ml-readonly" -}}
          export DB_USER="{{ .Data.username }}"
          export DB_PASS="{{ .Data.password }}"
          {{- end }}

    spec:
      serviceAccountName: ml-inference
      containers:
        - name: ml-server
          image: myregistry/ml-server:v1.0
          command:
            - /bin/sh
            - -c
            - |
              source /vault/secrets/api-keys
              source /vault/secrets/db
              python serve.py
```

### CSI Driver Method

```yaml
# vault-csi-secrets.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-ml-secrets
  namespace: ml-platform
spec:
  provider: vault
  parameters:
    vaultAddress: "https://vault.vault:8200"
    roleName: "ml-inference"
    objects: |
      - objectName: "openai-key"
        secretPath: "secret/data/ml/api-keys"
        secretKey: "openai"
      - objectName: "db-password"
        secretPath: "database/creds/ml-readonly"
        secretKey: "password"

  # Sync to Kubernetes secret
  secretObjects:
    - secretName: ml-secrets
      type: Opaque
      data:
        - objectName: openai-key
          key: OPENAI_API_KEY
        - objectName: db-password
          key: DB_PASSWORD

---
apiVersion: v1
kind: Pod
metadata:
  name: ml-server
spec:
  containers:
    - name: ml-server
      image: myregistry/ml-server:v1.0
      volumeMounts:
        - name: secrets-store
          mountPath: "/mnt/secrets"
          readOnly: true
      env:
        - name: OPENAI_API_KEY
          valueFrom:
            secretKeyRef:
              name: ml-secrets
              key: OPENAI_API_KEY
  volumes:
    - name: secrets-store
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: vault-ml-secrets
```

## Dynamic Database Credentials

### Configure Database Engine

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/mldb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="ml-readonly,ml-readwrite" \
  connection_url="postgresql://{{username}}:{{password}}@postgres.ml-platform:5432/mldb?sslmode=require" \
  username="vault-admin" \
  password="admin-password"

# Create readonly role
vault write database/roles/ml-readonly \
  db_name=mldb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Create readwrite role
vault write database/roles/ml-readwrite \
  db_name=mldb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT ALL ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"
```

### Use Dynamic Credentials

```python
# db_client.py
import hvac
import psycopg2
from contextlib import contextmanager

class DynamicDBClient:
    def __init__(self, vault_addr: str, role: str):
        self.vault = hvac.Client(url=vault_addr)
        self._authenticate_kubernetes()
        self.role = role

    def _authenticate_kubernetes(self):
        """Authenticate using Kubernetes service account"""
        with open('/var/run/secrets/kubernetes.io/serviceaccount/token') as f:
            jwt = f.read()

        self.vault.auth.kubernetes.login(
            role="ml-inference",
            jwt=jwt
        )

    def get_credentials(self) -> dict:
        """Get dynamic database credentials"""
        creds = self.vault.secrets.database.generate_credentials(
            name=self.role
        )
        return {
            'username': creds['data']['username'],
            'password': creds['data']['password'],
            'ttl': creds['lease_duration'],
            'lease_id': creds['lease_id']
        }

    @contextmanager
    def connection(self):
        """Get database connection with dynamic credentials"""
        creds = self.get_credentials()

        conn = psycopg2.connect(
            host="postgres.ml-platform",
            database="mldb",
            user=creds['username'],
            password=creds['password']
        )

        try:
            yield conn
        finally:
            conn.close()
            # Revoke credentials
            self.vault.sys.revoke_lease(creds['lease_id'])
```

## PKI and Certificates

### Configure PKI Engine

```bash
# Enable PKI engine
vault secrets enable pki

# Configure CA
vault secrets tune -max-lease-ttl=87600h pki

vault write -format=json pki/root/generate/internal \
  common_name="ML Platform CA" \
  ttl=87600h > ca.json

# Create role for issuing certificates
vault write pki/roles/ml-service \
  allowed_domains="ml-platform.svc.cluster.local" \
  allow_subdomains=true \
  max_ttl="72h"
```

### Request Certificates

```python
# cert_manager.py
import hvac
from cryptography import x509
from cryptography.hazmat.backends import default_backend

class CertManager:
    def __init__(self, vault_client: hvac.Client):
        self.vault = vault_client

    def issue_certificate(self, common_name: str) -> dict:
        """Issue TLS certificate from Vault"""
        cert_response = self.vault.secrets.pki.generate_certificate(
            name="ml-service",
            common_name=common_name,
            ttl="24h"
        )

        return {
            'certificate': cert_response['data']['certificate'],
            'private_key': cert_response['data']['private_key'],
            'ca_chain': cert_response['data']['ca_chain'],
            'serial_number': cert_response['data']['serial_number'],
            'expiration': cert_response['data']['expiration']
        }

    def rotate_certificate(self, serial_number: str, common_name: str) -> dict:
        """Rotate certificate before expiration"""
        # Revoke old certificate
        self.vault.secrets.pki.revoke_certificate(
            serial_number=serial_number
        )

        # Issue new certificate
        return self.issue_certificate(common_name)
```

## External Secrets Operator

### AWS Secrets Manager Integration

```yaml
# external-secrets.yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ml-api-keys
  namespace: ml-platform
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore

  target:
    name: ml-api-keys
    creationPolicy: Owner
    template:
      type: Opaque
      data:
        config.json: |
          {
            "openai_key": "{{ .openai }}",
            "hf_token": "{{ .huggingface }}"
          }

  data:
    - secretKey: openai
      remoteRef:
        key: prod/ml/api-keys
        property: openai_api_key

    - secretKey: huggingface
      remoteRef:
        key: prod/ml/api-keys
        property: hf_token
```

## Secret Rotation

### Automated Rotation

```python
# secret_rotation.py
import boto3
import hvac
from datetime import datetime, timedelta

class SecretRotator:
    def __init__(self, vault_client: hvac.Client):
        self.vault = vault_client
        self.secrets_manager = boto3.client('secretsmanager')

    def rotate_api_key(self, secret_path: str, provider: str):
        """Rotate API key and update in Vault"""
        # Generate new API key from provider
        new_key = self._generate_new_key(provider)

        # Update in Vault
        self.vault.secrets.kv.v2.create_or_update_secret(
            path=secret_path,
            secret={provider: new_key},
            cas=None  # Don't check version
        )

        # Log rotation
        self._log_rotation(secret_path, provider)

        return True

    def check_expiring_secrets(self) -> list:
        """Find secrets expiring soon"""
        expiring = []

        # List all secrets
        secrets = self.vault.secrets.kv.v2.list_secrets(path="ml")

        for secret_name in secrets['data']['keys']:
            metadata = self.vault.secrets.kv.v2.read_secret_metadata(
                path=f"ml/{secret_name}"
            )

            # Check custom metadata for expiration
            if 'expiration' in metadata['data'].get('custom_metadata', {}):
                exp_date = datetime.fromisoformat(
                    metadata['data']['custom_metadata']['expiration']
                )

                if exp_date < datetime.now() + timedelta(days=7):
                    expiring.append({
                        'path': f"ml/{secret_name}",
                        'expiration': exp_date
                    })

        return expiring

    def _generate_new_key(self, provider: str) -> str:
        """Generate new API key from provider"""
        # Implementation depends on provider
        pass
```

## Audit and Compliance

### Enable Audit Logging

```bash
# Enable file audit
vault audit enable file file_path=/vault/logs/audit.log

# Enable syslog audit
vault audit enable syslog tag="vault" facility="AUTH"
```

### Audit Log Analysis

```python
# audit_analyzer.py
import json
from collections import defaultdict

class VaultAuditAnalyzer:
    def __init__(self, audit_log_path: str):
        self.log_path = audit_log_path

    def analyze_access_patterns(self) -> dict:
        """Analyze secret access patterns"""
        patterns = defaultdict(lambda: {
            'read_count': 0,
            'write_count': 0,
            'clients': set()
        })

        with open(self.log_path) as f:
            for line in f:
                entry = json.loads(line)

                if entry['type'] == 'response':
                    path = entry['request']['path']
                    operation = entry['request']['operation']
                    client = entry['auth'].get('display_name', 'unknown')

                    if operation == 'read':
                        patterns[path]['read_count'] += 1
                    elif operation in ['create', 'update']:
                        patterns[path]['write_count'] += 1

                    patterns[path]['clients'].add(client)

        return patterns

    def find_anomalies(self) -> list:
        """Detect unusual access patterns"""
        anomalies = []
        patterns = self.analyze_access_patterns()

        for path, data in patterns.items():
            # High read frequency
            if data['read_count'] > 1000:
                anomalies.append({
                    'type': 'high_read_frequency',
                    'path': path,
                    'count': data['read_count']
                })

            # Multiple clients accessing same secret
            if len(data['clients']) > 5:
                anomalies.append({
                    'type': 'multiple_clients',
                    'path': path,
                    'client_count': len(data['clients'])
                })

        return anomalies
```

## Best Practices

1. **Principle of least privilege**: Minimal permissions
2. **Dynamic secrets**: Short-lived credentials
3. **Encrypt in transit**: TLS everywhere
4. **Audit everything**: Log all access
5. **Rotate regularly**: Automate rotation
6. **No secrets in code**: Use environment variables
7. **Disaster recovery**: Backup and test restore
8. **Multi-layer security**: Defense in depth

## Security Checklist

- [ ] TLS enabled for Vault
- [ ] Auto-unseal configured
- [ ] Audit logging enabled
- [ ] Policies follow least privilege
- [ ] Secret rotation scheduled
- [ ] Kubernetes auth configured
- [ ] Dynamic database credentials
- [ ] Backup and recovery tested

## Resources

- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Vault on Kubernetes](https://developer.hashicorp.com/vault/tutorials/kubernetes)
- [External Secrets Operator](https://external-secrets.io/)
- [Vault Security Model](https://developer.hashicorp.com/vault/docs/internals/security)

---

*Questions about secrets management? [Let me know](mailto:jordan@jordananderson.us).*
