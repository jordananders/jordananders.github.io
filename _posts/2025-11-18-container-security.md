---
layout: default
title:  "Container Security: Scanning, Runtime Protection, and AI Workloads"
date:   2025-11-18 12:00:00
categories: DevOps Security Containers Kubernetes
---

Container security requires defense in depth - from image scanning to runtime protection. For AI workloads, additional considerations around model security and GPU access control apply.

## Image Scanning

### Trivy Integration

```yaml
# .github/workflows/container-security.yml
name: Container Security

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'
```

### Custom Scanning Policy

```yaml
# trivy.yaml
severity:
  - CRITICAL
  - HIGH

vulnerability:
  ignore-unfixed: true

scan:
  file-patterns:
    - "Dockerfile"
    - "*.yaml"
    - "*.yml"

# Ignore specific CVEs
ignore:
  - CVE-2023-1234  # False positive for our use case
```

### Grype for SBOM Analysis

```bash
#!/bin/bash
# scan-sbom.sh

# Generate SBOM
syft myapp:latest -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json --fail-on high

# Output in multiple formats
grype sbom:sbom.json -o json > vulnerabilities.json
grype sbom:sbom.json -o table
```

## Secure Base Images

### Distroless Images

```dockerfile
# Build stage
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt

COPY . .

# Runtime stage - distroless
FROM gcr.io/distroless/python3-debian12

COPY --from=builder /app/deps /app/deps
COPY --from=builder /app /app

ENV PYTHONPATH=/app/deps
WORKDIR /app

USER nonroot:nonroot
ENTRYPOINT ["python", "main.py"]
```

### ML Model Image Security

```dockerfile
# Secure ML serving image
FROM nvidia/cuda:12.1-runtime-ubuntu22.04 AS base

# Create non-root user
RUN groupadd -r mluser && useradd -r -g mluser mluser

# Install dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# Security hardening
RUN chmod 755 /usr/bin/python3 && \
    rm -rf /usr/bin/apt* /usr/bin/dpkg*

FROM base AS builder
COPY requirements.txt .
RUN pip3 install --no-cache-dir --prefix=/install -r requirements.txt

FROM base
COPY --from=builder /install /usr/local
COPY --chown=mluser:mluser . /app

WORKDIR /app
USER mluser

# Don't run as root
HEALTHCHECK --interval=30s --timeout=3s \
    CMD python3 -c "import requests; requests.get('http://localhost:8080/health')"

ENTRYPOINT ["python3", "serve.py"]
```

## Runtime Security

### Falco Rules

```yaml
# falco-rules.yaml
- rule: Container Drift Detected
  desc: Detect file changes in running container
  condition: >
    evt.type in (open, openat) and
    container and
    fd.name startswith /app and
    evt.is_open_write=true
  output: >
    File modified in container
    (file=%fd.name container=%container.name image=%container.image.repository)
  priority: WARNING

- rule: Crypto Mining Detection
  desc: Detect cryptocurrency mining
  condition: >
    spawned_process and
    container and
    (proc.name in (xmrig, minerd, cpuminer) or
     proc.cmdline contains "stratum+tcp" or
     proc.cmdline contains "pool.minergate")
  output: >
    Crypto mining detected
    (process=%proc.name container=%container.name)
  priority: CRITICAL

- rule: ML Model Exfiltration Attempt
  desc: Detect attempts to copy model files
  condition: >
    evt.type in (open, openat) and
    container and
    fd.name contains "/models/" and
    evt.is_open_read=true and
    not proc.name in (python, python3, serve)
  output: >
    Suspicious model file access
    (file=%fd.name process=%proc.name container=%container.name)
  priority: HIGH
```

### Kubernetes Pod Security

```yaml
# pod-security-policy.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-ml-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault

  containers:
    - name: ml-server
      image: myregistry/ml-server:v1.0
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL

      resources:
        limits:
          memory: "4Gi"
          cpu: "2"
          nvidia.com/gpu: "1"
        requests:
          memory: "2Gi"
          cpu: "1"

      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: model-cache
          mountPath: /app/cache
          readOnly: false

  volumes:
    - name: tmp
      emptyDir: {}
    - name: model-cache
      emptyDir:
        sizeLimit: 1Gi
```

### Network Policies

```yaml
# network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ml-service-policy
  namespace: ml-platform
spec:
  podSelector:
    matchLabels:
      app: ml-inference
  policyTypes:
    - Ingress
    - Egress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: api-gateway
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080

  egress:
    # Allow DNS
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53

    # Allow model registry
    - to:
        - podSelector:
            matchLabels:
              app: model-registry
      ports:
        - protocol: TCP
          port: 5000

    # Deny internet access
    - to:
        - ipBlock:
            cidr: 10.0.0.0/8
```

## Secrets Management

### External Secrets Operator

```yaml
# external-secret.yaml
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

  data:
    - secretKey: OPENAI_API_KEY
      remoteRef:
        key: prod/ml/api-keys
        property: openai

    - secretKey: HF_TOKEN
      remoteRef:
        key: prod/ml/api-keys
        property: huggingface

---
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
```

### Vault Integration

```python
# vault_client.py
import hvac
from functools import lru_cache

class VaultClient:
    def __init__(self, vault_addr: str, role: str):
        self.client = hvac.Client(url=vault_addr)
        self._authenticate_kubernetes(role)

    def _authenticate_kubernetes(self, role: str):
        """Authenticate using Kubernetes service account"""
        with open('/var/run/secrets/kubernetes.io/serviceaccount/token') as f:
            jwt = f.read()

        self.client.auth.kubernetes.login(
            role=role,
            jwt=jwt,
            mount_point='kubernetes'
        )

    @lru_cache(maxsize=100)
    def get_secret(self, path: str, key: str) -> str:
        """Get secret from Vault"""
        secret = self.client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point='secret'
        )
        return secret['data']['data'][key]

    def get_database_credentials(self, role: str) -> dict:
        """Get dynamic database credentials"""
        creds = self.client.secrets.database.generate_credentials(
            name=role,
            mount_point='database'
        )
        return {
            'username': creds['data']['username'],
            'password': creds['data']['password'],
            'ttl': creds['lease_duration']
        }
```

## Supply Chain Security

### Sigstore Cosign

```bash
#!/bin/bash
# sign-and-verify.sh

# Generate key pair (one time)
cosign generate-key-pair

# Sign container image
cosign sign --key cosign.key myregistry/ml-model:v1.0

# Verify signature
cosign verify --key cosign.pub myregistry/ml-model:v1.0

# Sign with OIDC (keyless)
cosign sign myregistry/ml-model:v1.0

# Attach SBOM
cosign attach sbom --sbom sbom.json myregistry/ml-model:v1.0

# Verify SBOM
cosign verify-attestation --type spdx myregistry/ml-model:v1.0
```

### Admission Controller

```yaml
# kyverno-policy.yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: verify-image-signature
spec:
  validationFailureAction: Enforce
  background: false
  rules:
    - name: verify-signature
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "myregistry/*"
          attestors:
            - entries:
                - keyless:
                    subject: "https://github.com/myorg/*"
                    issuer: "https://token.actions.githubusercontent.com"
                    rekor:
                      url: https://rekor.sigstore.dev

    - name: require-sbom
      match:
        any:
          - resources:
              kinds:
                - Pod
      verifyImages:
        - imageReferences:
            - "myregistry/*"
          attestations:
            - type: https://spdx.dev/Document
              conditions:
                - all:
                    - key: "{{ packages[].name }}"
                    - operator: AllNotIn
                    - value: ["vulnerable-package"]
```

## AI/ML Specific Security

### Model Integrity Verification

```python
# model_security.py
import hashlib
import json
from pathlib import Path

class ModelSecurityManager:
    def __init__(self, model_registry_url: str):
        self.registry_url = model_registry_url

    def calculate_model_hash(self, model_path: str) -> str:
        """Calculate SHA256 hash of model file"""
        sha256_hash = hashlib.sha256()

        with open(model_path, "rb") as f:
            for byte_block in iter(lambda: f.read(4096), b""):
                sha256_hash.update(byte_block)

        return sha256_hash.hexdigest()

    def verify_model_integrity(self, model_path: str,
                               expected_hash: str) -> bool:
        """Verify model hasn't been tampered with"""
        actual_hash = self.calculate_model_hash(model_path)
        return actual_hash == expected_hash

    def generate_model_manifest(self, model_path: str,
                                metadata: dict) -> dict:
        """Generate signed manifest for model"""
        manifest = {
            'model_hash': self.calculate_model_hash(model_path),
            'model_size': Path(model_path).stat().st_size,
            'metadata': metadata,
            'created_at': datetime.utcnow().isoformat()
        }

        # Sign manifest
        manifest['signature'] = self._sign_manifest(manifest)

        return manifest

    def audit_model_access(self, model_name: str, action: str,
                          user: str, context: dict):
        """Log model access for audit trail"""
        audit_entry = {
            'timestamp': datetime.utcnow().isoformat(),
            'model_name': model_name,
            'action': action,
            'user': user,
            'context': context,
            'source_ip': context.get('source_ip')
        }

        # Send to audit log
        self._send_audit_log(audit_entry)
```

### GPU Isolation

```yaml
# gpu-resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: ml-platform
spec:
  hard:
    nvidia.com/gpu: "4"
    requests.nvidia.com/gpu: "4"
    limits.nvidia.com/gpu: "4"

---
apiVersion: v1
kind: LimitRange
metadata:
  name: gpu-limits
  namespace: ml-platform
spec:
  limits:
    - type: Container
      max:
        nvidia.com/gpu: "2"
      min:
        nvidia.com/gpu: "1"
      default:
        nvidia.com/gpu: "1"
```

## Security Scanning Pipeline

```python
# security_pipeline.py
from dataclasses import dataclass
from enum import Enum

class Severity(Enum):
    CRITICAL = "critical"
    HIGH = "high"
    MEDIUM = "medium"
    LOW = "low"

@dataclass
class ScanResult:
    tool: str
    passed: bool
    vulnerabilities: list
    recommendations: list

class SecurityPipeline:
    def __init__(self):
        self.scanners = []

    def add_scanner(self, scanner):
        self.scanners.append(scanner)

    async def run_pipeline(self, image: str) -> dict:
        results = []

        for scanner in self.scanners:
            result = await scanner.scan(image)
            results.append(result)

        # Aggregate results
        all_passed = all(r.passed for r in results)
        critical_vulns = sum(
            1 for r in results
            for v in r.vulnerabilities
            if v.severity == Severity.CRITICAL
        )

        return {
            'passed': all_passed and critical_vulns == 0,
            'results': results,
            'summary': {
                'critical': critical_vulns,
                'high': sum(1 for r in results for v in r.vulnerabilities
                           if v.severity == Severity.HIGH)
            }
        }
```

## Best Practices

1. **Shift left**: Scan in CI/CD before deployment
2. **Least privilege**: Run as non-root, drop capabilities
3. **Immutable infrastructure**: Read-only filesystems
4. **Network segmentation**: Default deny policies
5. **Secrets management**: Never embed in images
6. **Supply chain**: Sign and verify images
7. **Runtime protection**: Monitor with Falco
8. **Model security**: Verify integrity, audit access

## Resources

- [NIST Container Security Guide](https://csrc.nist.gov/publications/detail/sp/800-190/final)
- [Kubernetes Security Best Practices](https://kubernetes.io/docs/concepts/security/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Falco Rules](https://falco.org/docs/rules/)

---

*Questions about container security? [Let me know](mailto:jordan@jordananderson.us).*
