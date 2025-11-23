# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Personal homelab infrastructure-as-code running self-hosted applications on a bare-metal Kubernetes cluster powered by Talos Linux. All manifests use Kustomize for deployment. Repository is bilingual (EN/FR) and designed for public sharing with sensitive data protected in gitignored `private/` directory.

## Infrastructure Stack

- **OS**: Talos Linux v1.11.1 (immutable, API-driven, no SSH)
- **Kubernetes**: v1.34.0
- **Deployment**: Kustomize (no Helm, no overlays)
- **Storage**: local-path-provisioner (StorageClass: `local-path`)
- **Secrets**: Bitwarden Secrets Manager via External Secrets Operator
- **Ingress**: Tailscale Operator (private mesh VPN)
- **Authentication**: Authelia (OIDC/OAuth2)

## Repository Structure

```
homelab/
├── apps/                        # Application Kustomize deployments
│   ├── argocd/                 # GitOps continuous delivery
│   ├── stirling-pdf/           # PDF manipulation toolkit
│   ├── immich/                 # Photo/video management
│   ├── paperless-ngx/          # Document management
│   ├── wallabag/               # Read-it-later
│   ├── miniflux/               # RSS reader
│   └── shared-services/        # Shared PostgreSQL + Redis
├── infrastructure/
│   ├── talos/                  # Talos config templates (.example files)
│   └── kubernetes/
│       └── external-secrets/   # ESO + Bitwarden setup
├── docs/
│   └── architecture.md         # Infrastructure architecture (bilingual)
└── private/                    # ⚠️ GITIGNORED - actual Talos/K8s configs
```

**Key principle**: `apps/` was renamed from `charts/` during repository reorganization. All sensitive configs (Talos, Bitwarden org/project IDs) live in gitignored `private/` directory.

## Critical Architectural Constraints

### Talos Linux Requirements

**All namespaces** must have privileged PodSecurity labels (required for local-path-provisioner):
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: <namespace>
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

**All PVCs** must specify:
```yaml
storageClassName: local-path
```

### External Secrets Operator

- **API version**: `external-secrets.io/v1` (not v1beta1)
- **ClusterSecretStore name**: `bitwarden-cluster-secretstore`
- All application secrets sync from Bitwarden Secrets Manager with 1-hour refresh
- Secrets stored as individual items in Bitwarden (not as key-value pairs)

Secret naming convention: `<app>-<component>-<type>`
- Example: `immich-db-password`, `paperless-oauth-providers`

### Tailscale Ingress

Use standard Kubernetes Ingress with TLS for automatic HTTPS:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: <app>-ingress
  annotations:
    tailscale.com/hostname: <short-hostname>
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - <short-hostname>
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: <service>
                port:
                  number: <port>
```

Tailscale Operator generates URLs: `https://<hostname>.tail<tailnet-id>.ts.net`

**Important**: Use `tailscale.com/hostname` annotation to get short URLs instead of auto-generated long names.

### Shared Services Pattern

PostgreSQL 16 and Redis 7 run in `shared-services` namespace, providing databases for:
- PostgreSQL: `paperless`, `wallabag`, `miniflux` databases
- Redis: DB 0 (Immich), DB 1 (Paperless), DB 2 (Wallabag)

**Exception**: Immich uses dedicated PostgreSQL with VectorChord extension.

## Application Chart Structure

Every app follows this exact pattern:
```
<app-name>/
├── README.md              # Comprehensive French documentation
└── base/
    ├── kustomization.yaml # Lists all resources + namespace + labels
    ├── namespace.yaml     # With privileged PodSecurity labels
    ├── pvc.yaml          # All PVCs with local-path storage
    ├── external-secrets.yaml # Bitwarden secret references
    ├── <component>-deployment.yaml
    ├── <component>-service.yaml
    └── tailscale-ingress.yaml
```

**No overlays** - homelab has single environment, no staging/prod distinction.

## Code Style Conventions

- **No comments in YAML manifests** - keep manifests clean
- **No Helm** - pure Kustomize only
- **Documentation in French** - app READMEs are in French
- **Root documentation bilingual** - README.md and docs/ have EN/FR sections
- All sensitive data uses placeholders in committed files (e.g., `<REDACTED>`, `<YOUR-CONTROL-PLANE-IP>`)

## Common Development Commands

### Deploy Application
```bash
kubectl apply -k apps/<app-name>/base/
```

### Check Status
```bash
kubectl get pods -n <namespace>
kubectl get ingress -n <namespace>
kubectl get externalsecrets -n <namespace>  # Check secret sync status
```

### View Logs
```bash
kubectl logs -f deployment/<deployment-name> -n <namespace>
```

### Restart Deployment
```bash
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

### Test Application Health
```bash
kubectl exec -n <namespace> deployment/<deployment-name> -- \
  curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/ --max-time 5
```

### Talos Operations
```bash
# Bootstrap cluster (run once)
talosctl bootstrap --nodes <control-plane-ip>

# Get kubeconfig
talosctl kubeconfig --nodes <control-plane-ip>

# Check cluster health
talosctl health --nodes <control-plane-ip>

# View logs
talosctl logs --nodes <node-ip> kubelet

# Upgrade Talos
talosctl upgrade --nodes <node-ip> --image ghcr.io/siderolabs/installer:v1.11.1
```

## Application-Specific Constraints

### Immich
- **PostgreSQL image**: Must use `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0` (VectorChord extension required)
- **REDIS_PORT environment variable**: Must explicitly set `REDIS_PORT: "6379"` to avoid Kubernetes service env var conflicts causing `REDIS_PORT=NaN` error
- **Health endpoint**: `/api/server/ping`
- **Dependencies**: Dedicated PostgreSQL (VectorChord), Redis, Machine Learning service

### Paperless-ngx
- **Authentication**: OAuth only via Authelia - regular login disabled
- **OAuth provider**: django-allauth with OpenID Connect
- **OAuth users**: Auto-created but without admin rights - must promote manually:
  ```bash
  kubectl exec -n paperless-ngx deployment/paperless-ngx -- \
    python3 manage.py shell -c "from django.contrib.auth import get_user_model; \
    User = get_user_model(); u = User.objects.get(username='USERNAME'); \
    u.is_superuser = True; u.is_staff = True; u.save()"
  ```
- **CSRF protection**: Requires explicit Tailscale URL:
  ```yaml
  - name: PAPERLESS_URL
    value: https://<app>.tail<id>.ts.net
  - name: PAPERLESS_CSRF_TRUSTED_ORIGINS
    value: https://<app>.tail<id>.ts.net
  ```
- **OAuth secret**: `paperless-oauth-providers` contains full JSON config in Bitwarden
- **Dependencies**: Shared PostgreSQL, Redis, Apache Tika, Gotenberg

### Wallabag
- **NO OIDC support**: Cannot use OAuth for user authentication (API only)
- **Symfony framework**: Bitwarden secrets must avoid characters: `&`, `%`, `=`, `#` (at start)
  - Error `You have requested a non-existent parameter "&"` indicates secret contains forbidden character
  - Safe characters: alphanumeric, `-`, `_`, `~`, `@`, `!`, `*`, `/`, `+`
- **Database initialization**: May require manual trigger:
  ```bash
  kubectl exec -n wallabag deployment/wallabag -- \
    /var/www/wallabag/bin/console wallabag:install --env=prod --no-interaction
  ```
- **Default credentials**: `wallabag:wallabag` (change after first login)
- **Dependencies**: Shared PostgreSQL, Redis
- **Health endpoint**: `/`

### Miniflux
- **Authentication**: OIDC via Authelia + local admin account
- **Dependencies**: Shared PostgreSQL

### ArgoCD
- **Authentication**: OIDC via Authelia + local admin account
- **Insecure mode**: Runs without TLS internally (Tailscale handles TLS termination)
- **Configuration**: Use `server.insecure: "true"` in `argocd-cm` ConfigMap (NOT deployment args)
- **RBAC**: Groups mapped via `argocd-rbac-cm` ConfigMap (`argocd-admins` → `role:admin`)
- **Private overlay**: Real URLs in `private/argocd/` for deployment
- **Dependencies**: None (standalone)

### Stirling-PDF
- **Startup time**: First boot takes 2-3 minutes (font installation)
- **Probes**: Use `startupProbe` with generous timeouts (30s initial, 30 failures allowed)
- **Health endpoint**: `/api/v1/info/status`
- **No authentication**: Open access (protected by Tailscale VPN)
- **Dependencies**: None (standalone)

### Shared Services
- **PostgreSQL init**: Uses `postgres-init.sh` script with ConfigMap to create multiple databases
- **Redis**: Single instance with multiple DBs (0, 1, 2)
- **No Tailscale ingress**: Internal services only

## Known Issues & Solutions

### REDIS_PORT=NaN Error (Immich)
Kubernetes creates `REDIS_PORT` env var from Service, conflicting with Immich. Always explicitly set:
```yaml
- name: REDIS_PORT
  value: "6379"
```

### PostgreSQL Password Changes
External Secrets Operator cannot change PostgreSQL passwords post-initialization. PostgreSQL reads password only at first startup. To change:
1. Delete deployment: `kubectl delete deployment/<name> -n <namespace>`
2. Delete PVC: `kubectl delete pvc/<name> -n <namespace>`
3. Update secret in Bitwarden
4. Redeploy application

### Symfony Parameter Parsing (Wallabag)
When creating secrets in Bitwarden for Symfony apps, test with:
```bash
kubectl logs -f deployment/wallabag -n wallabag
```
If error contains `You have requested a non-existent parameter`, regenerate secret without `&`, `%`, `=`, `#`.

### External Secrets Not Syncing
Check SecretStore status:
```bash
kubectl get clustersecretstore bitwarden-cluster-secretstore
kubectl describe externalsecret <name> -n <namespace>
kubectl logs -n external-secrets-system deployment/external-secrets
```

### Tailscale Ingress Stuck or DNS Stale
When Tailscale hostname conflicts or DNS returns wrong IP:
1. Delete device from Tailscale admin console (https://login.tailscale.com/admin/machines)
2. Remove finalizer if ingress is stuck: `kubectl patch ingress <name> -n <namespace> -p '{"metadata":{"finalizers":null}}' --type=merge`
3. Delete ingress and recreate: `kubectl delete ingress <name> -n <namespace> && kubectl apply -k apps/<app>/base/`
4. Check operator logs: `kubectl logs -n tailscale deployment/operator --tail=50`

### Tailscale Operator Network Issues
If operator cannot reach `api.tailscale.com`:
```bash
kubectl rollout restart deployment/operator -n tailscale
```

## Security & Sensitive Data

**Never commit to Git**:
- Actual Talos configs: `private/talos/controlplane.yaml`, `private/talos/worker.yaml`, `private/talos/talosconfig`
- Bitwarden configs: `private/kubernetes/bitwarden-cluster-secretstore.yaml` (contains org/project IDs)
- Any file with real IPs, tokens, passwords, certificates, keys

**Safe to commit**:
- `infrastructure/talos/*.example` files with `<REDACTED>` placeholders
- `infrastructure/kubernetes/external-secrets/*.example` files
- Application manifests in `apps/` (they reference secret names, not values)
- ExternalSecret resources (contain Bitwarden secret IDs, which are UUIDs - not sensitive)

**IP address sanitization**:
- Use `<CONTROL-PLANE-IP>` or `<YOUR-CONTROL-PLANE-IP>` in documentation/examples
- Standard Kubernetes subnets are OK: `10.244.0.0/16` (pods), `10.96.0.0/12` (services)
- Generic examples are OK: `192.168.x.x`, `10.x.x.x`

## Documentation Conventions

- **Bilingual docs**: README.md and docs/architecture.md have English and French sections separated by `---`
- **App READMEs**: In French only (apps/*/README.md)
- **Format**: Use `[English](#english) | [Français](#français)` links at top
- **No setup-guide.md**: Removed during cleanup - refer users to official Talos/K8s docs
- **Migration notes**: MIGRATION.md documents the repository reorganization from flat structure

## Repository Migration Context

This repository was reorganized on 2025-11-16:
- `charts/` → `apps/`
- Root-level configs → `private/` (gitignored)
- Added bilingual documentation
- Created infrastructure templates with sanitized examples

Active cluster config lives in `private/talos/`, archived old config in `private/archive/`.
