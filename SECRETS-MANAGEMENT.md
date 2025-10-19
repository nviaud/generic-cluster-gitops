# Secrets Management Guide

This document explains how to manage secrets across different environments in this GitOps repository.

## Overview

Secrets are managed differently per environment:
- **Dev**: Plain Kubernetes Secrets (applied manually, not in git)
- **Staging**: SealedSecrets (encrypted, safe to commit)
- **Production**: ExternalSecrets (stored in external secret store)

## Development Environment

For development, create secrets manually:

```bash
kubectl create secret generic grafana-admin-credentials \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=admin \
  -n monitoring
```

**Never commit plain secrets to git!**

## Staging Environment (SealedSecrets)

### Setup

1. Install sealed-secrets controller:
```bash
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
```

2. Install kubeseal CLI:
```bash
# macOS
brew install kubeseal

# Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64 -O kubeseal
chmod +x kubeseal
sudo mv kubeseal /usr/local/bin/
```

### Creating SealedSecrets

1. Create a secret (don't save this file!):
```bash
kubectl create secret generic grafana-admin-credentials \
  --from-literal=admin-user=admin \
  --from-literal=admin-password=YOUR_SECURE_PASSWORD \
  --dry-run=client -o yaml > /tmp/secret.yaml
```

2. Encrypt it with kubeseal:
```bash
kubeseal -f /tmp/secret.yaml -o yaml > infrastructure/grafana/overlays/staging/sealed-secret-grafana.yaml
```

3. Commit the sealed secret:
```bash
git add infrastructure/grafana/overlays/staging/sealed-secret-grafana.yaml
git commit -m "Add grafana sealed secret for staging"
```

4. Clean up the temporary secret:
```bash
rm /tmp/secret.yaml
```

### Adding to Kustomization

Update `infrastructure/grafana/overlays/staging/kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: monitoring

resources:
  - ../../base
  - sealed-secret-grafana.yaml
```

## Production Environment (ExternalSecrets)

### Setup

1. Install external-secrets operator:
```bash
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets -n external-secrets-system --create-namespace
```

2. Configure your secret backend (example: AWS Secrets Manager):
```bash
# Create IAM policy and service account
# Configure IRSA (IAM Roles for Service Accounts)
```

3. Create a SecretStore:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: monitoring
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
```

### Creating ExternalSecrets

1. Store the secret in AWS Secrets Manager:
```bash
aws secretsmanager create-secret \
  --name production/grafana/admin \
  --secret-string '{"username":"admin","password":"YOUR_SECURE_PASSWORD"}' \
  --region us-east-1
```

2. Create an ExternalSecret resource (safe to commit):
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: grafana-admin-credentials
  namespace: monitoring
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: grafana-admin-credentials
  data:
    - secretKey: admin-user
      remoteRef:
        key: production/grafana/admin
        property: username
    - secretKey: admin-password
      remoteRef:
        key: production/grafana/admin
        property: password
```

3. Commit the ExternalSecret:
```bash
git add infrastructure/grafana/overlays/production/external-secret-grafana.yaml
git commit -m "Add grafana external secret for production"
```

## Alternative: SOPS (Secrets Operations)

Another popular option is Mozilla SOPS for encrypting secrets in git:

1. Install SOPS:
```bash
brew install sops  # macOS
```

2. Configure encryption (example with age):
```bash
age-keygen -o keys.txt
export SOPS_AGE_KEY_FILE=keys.txt
```

3. Encrypt a secret file:
```bash
sops -e secret.yaml > secret.enc.yaml
git add secret.enc.yaml
```

4. Use with ArgoCD via the SOPS plugin or Kustomize KSOPS.

## Best Practices

1. **Never commit plain secrets** to version control
2. **Rotate secrets regularly**, especially after team member changes
3. **Use strong passwords** - generate them with tools like `pwgen` or `openssl rand`
4. **Limit secret access** - use RBAC to restrict who can read secrets
5. **Audit secret access** - enable audit logging for secret operations
6. **Use different secrets per environment** - never reuse production secrets in dev/staging

## Verification

After applying secrets, verify they exist:

```bash
# Check if secret exists
kubectl get secret grafana-admin-credentials -n monitoring

# Verify secret keys (doesn't show values)
kubectl describe secret grafana-admin-credentials -n monitoring

# Decode and view secret (use carefully!)
kubectl get secret grafana-admin-credentials -n monitoring -o jsonpath='{.data.admin-user}' | base64 -d
```
