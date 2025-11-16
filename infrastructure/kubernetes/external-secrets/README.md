# External Secrets Operator - Bitwarden Integration

This directory contains configuration for External Secrets Operator (ESO) with Bitwarden Secrets Manager.

## Overview

The homelab uses Bitwarden Secrets Manager as the secret backend for all Kubernetes secrets. This provides:
- Centralized secret management
- Automatic secret rotation (1-hour refresh interval)
- Secure secret storage outside the cluster
- Easy secret updates without kubectl

## Architecture

```
Bitwarden Secrets Manager
         ↓
External Secrets Operator
         ↓
Kubernetes Secrets (auto-synced)
         ↓
Application Pods
```

## Components

### 1. Bitwarden SDK Server
- Deployed in `external-secrets-system` namespace
- Runs as HTTPS server with self-signed certificate
- Acts as proxy between ESO and Bitwarden API

### 2. ClusterSecretStore
- Cluster-wide secret store configuration
- Points to Bitwarden organization and project
- Used by all namespaces

### 3. ExternalSecret Resources
- Per-app secret definitions
- Map Bitwarden secret IDs to Kubernetes secret keys
- Located in each app's Kustomize base directory

## Setup Instructions

### Prerequisites

1. **Bitwarden Secrets Manager Account**
   - Create organization
   - Create project for homelab secrets
   - Note the Organization ID and Project ID

2. **Access Token**
   - Generate machine account access token
   - Store in Kubernetes secret: `bitwarden-access-token`

### Installation

1. **Install External Secrets Operator**:
   ```bash
   helm repo add external-secrets https://charts.external-secrets.io
   helm install external-secrets external-secrets/external-secrets \
     -n external-secrets-system --create-namespace
   ```

2. **Deploy Bitwarden SDK Server**:
   ```bash
   # Create self-signed certificate
   kubectl apply -f bitwarden-cert.yaml.example

   # Wait for cert-manager to issue certificate
   kubectl wait --for=condition=Ready certificate/bitwarden-sdk-server \
     -n external-secrets-system
   ```

3. **Create Access Token Secret**:
   ```bash
   kubectl create secret generic bitwarden-access-token \
     -n external-secrets-system \
     --from-literal=token='<YOUR-ACCESS-TOKEN>'
   ```

4. **Apply SecretStore Configuration**:
   ```bash
   # Edit bitwarden-secretstore.yaml.example with your IDs
   # Then apply:
   kubectl apply -f bitwarden-cluster-secretstore.yaml
   ```

### Creating Secrets in Bitwarden

For each application, create secrets in Bitwarden Secrets Manager:

**Example for Paperless-ngx**:
- `paperless-db-username` → `paperless`
- `paperless-db-password` → `<strong-password>`
- `paperless-django-secret` → `<random-50-char-string>`
- `paperless-admin-password` → `<admin-password>`

**Important**: Note the secret ID (UUID) for each secret - you'll need these in the ExternalSecret manifests.

## Secret Naming Convention

Pattern: `<app>-<component>-<type>`

Examples:
- `immich-db-password`
- `wallabag-redis-password`
- `miniflux-oidc-client-secret`
- `shared-postgres-username`

## ExternalSecret Example

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: paperless-db-secret
  namespace: paperless-ngx
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: bitwarden-cluster-secretstore
    kind: ClusterSecretStore
  target:
    name: paperless-db-secret
    creationPolicy: Owner
  dataFrom:
    - extract:
        key: <SECRET-ID-FROM-BITWARDEN>
```

## Special Considerations

### Wallabag Secrets
Wallabag (Symfony) has issues parsing certain characters in environment variables:
- Avoid: `&`, `%`, `=`, `#`
- Use alphanumeric + special chars like `!@$*-_`

### OAuth/OIDC Secrets
For apps with Authelia integration:
- `<app>-oidc-client-id`
- `<app>-oidc-client-secret`
- `<app>-oauth-providers` (full YAML config)

## Troubleshooting

### Check ESO Status
```bash
kubectl get clustersecretstore
kubectl get externalsecrets -A
```

### View Secret Sync Logs
```bash
kubectl logs -n external-secrets-system \
  deployment/external-secrets
```

### Verify Secret Creation
```bash
kubectl get secret <secret-name> -n <namespace> -o yaml
```

### Common Issues

1. **SecretStore not ready**
   - Check Bitwarden access token is valid
   - Verify SDK server is running
   - Check network connectivity

2. **ExternalSecret failing**
   - Verify secret exists in Bitwarden
   - Check secret ID is correct
   - Ensure project permissions are set

3. **Certificate issues**
   - Verify cert-manager is installed
   - Check certificate resource status
   - Ensure CA bundle matches SDK server cert

## Security Best Practices

- ✅ Use separate Bitwarden project for homelab
- ✅ Rotate access tokens regularly
- ✅ Use least-privilege access tokens (read-only)
- ✅ Enable Bitwarden audit logs
- ✅ Monitor ExternalSecret sync failures
- ❌ Never commit access tokens to Git
- ❌ Never store secrets in plain text

## References

- [External Secrets Operator Docs](https://external-secrets.io/)
- [Bitwarden Secrets Manager](https://bitwarden.com/products/secrets-manager/)
- [ESO Bitwarden Provider](https://external-secrets.io/latest/provider/bitwarden-secrets-manager/)
