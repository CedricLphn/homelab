# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

Personal homelab infrastructure-as-code running self-hosted applications on a bare-metal Kubernetes cluster powered by Talos Linux. All manifests use Kustomize for deployment. Repository is bilingual (EN/FR) and designed for public sharing with sensitive data protected in gitignored `private/` directory.

## Infrastructure Stack

- **OS**: Talos Linux v1.13.2 (immutable, API-driven, no SSH)
- **Kubernetes**: v1.36.0
- **Deployment**: Kustomize (no Helm for apps; infra components like Cilium/cert-manager/Traefik installed by Helm out-of-band, only homelab config GitOps-managed)
- **CNI / kube-proxy**: Cilium (eBPF kube-proxy-replacement). Talos machine config has `cluster.network.cni.name=none` and `cluster.proxy.disabled=true`. No LB-IPAM / no L2 announce — see ingress model below
- **Routing**: Traefik v3 implementing Kubernetes Gateway API v1.4 (`HTTPRoute`, not legacy `Ingress`). Runs in `hostNetwork: true`, binds directly to the node's ports 80/443
- **TLS**: cert-manager with self-signed Homelab Root CA (`homelab-ca-issuer` ClusterIssuer). Wildcard cert `*.homelab.lastsector.lan` on the Gateway listener
- **DNS**: the LAN router / DNS server resolves `*.homelab.lastsector.lan` → node IP
- **Remote access**: WireGuard on the LAN router (outside the cluster)
- **Storage**: local-path-provisioner (StorageClass: `local-path`)
- **Secrets**: Bitwarden Secrets Manager via External Secrets Operator
- **Authentication**: Authelia (OIDC/OAuth2, external to cluster)

## Repository Structure

```
homelab/
├── apps/                        # Application Kustomize deployments
│   ├── argocd/                 # GitOps continuous delivery
│   ├── stirling-pdf/           # PDF manipulation toolkit
│   ├── immich/                 # Photo/video management
│   ├── paperless-ngx/          # Document management
│   ├── miniflux/               # RSS reader
│   ├── karakeep/               # Bookmark manager (ex-Hoarder)
│   └── shared-services/        # Shared PostgreSQL + Redis
├── argocd-apps/                # ArgoCD Application CRDs (app-of-apps)
├── infrastructure/
│   ├── talos/                  # Talos config templates (.example files) + patches
│   └── kubernetes/
│       ├── cilium/             # Cilium Helm values (no in-cluster resources)
│       ├── gateway-api/        # Gateway API v1.4 CRDs
│       ├── cert-manager/       # Homelab Root CA + ClusterIssuer
│       ├── traefik/            # Traefik Gateway + wildcard cert
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

### Gateway API + Traefik Routing

Each application exposes itself via a single `HTTPRoute` attached to the cluster-wide `homelab-gateway` Gateway in the `traefik` namespace. The Gateway terminates TLS with a wildcard `*.homelab.lastsector.lan` cert issued by cert-manager from the Homelab Root CA.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: <app>
  namespace: <namespace>
spec:
  parentRefs:
    - name: homelab-gateway
      namespace: traefik
      sectionName: https
  hostnames:
    - <app>.homelab.lastsector.lan
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: <service>
          port: <port>
```

URLs: `https://<app>.homelab.lastsector.lan` (resolved by LAN DNS to the node IP).

**No `ReferenceGrant` needed**: the Gateway allows routes from all namespaces (`allowedRoutes.namespaces.from: All`) and backends live in the same namespace as the HTTPRoute.

**No `Ingress` resources**: legacy `networking.k8s.io/v1 Ingress` is not used. Traefik's Ingress provider is disabled in `infrastructure/kubernetes/traefik/values.yaml`.

### Shared Services Pattern

PostgreSQL 16 and Redis 7 run in `shared-services` namespace, providing databases for:
- PostgreSQL: `paperless`, `miniflux` databases
- Redis: DB 0 (Immich), DB 1 (Paperless)

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
    └── httproute.yaml         # Gateway API HTTPRoute attached to homelab-gateway
```

**No overlays** - homelab has single environment, no staging/prod distinction.

### Kustomize Labels

**Never use `commonLabels`** in kustomization.yaml - it injects labels into `selector.matchLabels` which are immutable, preventing updates to existing deployments. Use `labels` instead:
```yaml
labels:
- pairs:
    app.kubernetes.io/part-of: <app>
  includeSelectors: false
```

### ArgoCD GitOps

ArgoCD manages all applications with **autosync enabled**. Manual `kubectl` changes (patches, set image, etc.) will be reverted by ArgoCD. All changes must be committed and pushed to git. Use `argocd.argoproj.io/refresh: hard` annotation to force immediate sync after pushing.

## Code Style Conventions

- **No comments in YAML manifests** - keep manifests clean
- **Apps use pure Kustomize**, no Helm. Heavy infrastructure components (Cilium, cert-manager, Traefik) are installed by Helm out-of-band (matches the External Secrets pattern); only their homelab-specific config is GitOps-managed.
- **Documentation in French** - app READMEs are in French
- **Root documentation bilingual** - README.md and docs/ have EN/FR sections
- All sensitive data uses placeholders in committed files (e.g., `<REDACTED>`, `<YOUR-CONTROL-PLANE-IP>`)
- **Internal domain**: `*.homelab.lastsector.lan` is committed directly. `.lan` is a reserved non-routable TLD, so nothing about this domain is publicly exploitable.

## Common Development Commands

### Deploy Application
```bash
kubectl apply -k apps/<app-name>/base/
```

### Check Status
```bash
kubectl get pods -n <namespace>
kubectl get httproute -n <namespace>
kubectl get certificate -n <namespace>
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
talosctl upgrade --nodes <node-ip> --image ghcr.io/siderolabs/installer:v1.13.2
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
- **CSRF protection**: Requires explicit internal URL:
  ```yaml
  - name: PAPERLESS_URL
    value: https://paperless.homelab.lastsector.lan
  - name: PAPERLESS_CSRF_TRUSTED_ORIGINS
    value: https://paperless.homelab.lastsector.lan
  ```
- **OAuth secret**: `paperless-oauth-providers` contains full JSON config in Bitwarden
- **Dependencies**: Shared PostgreSQL, Redis, Apache Tika, Gotenberg

### Miniflux
- **Authentication**: OIDC via Authelia + local admin account
- **Dependencies**: Shared PostgreSQL

### ArgoCD
- **Authentication**: OIDC via Authelia + local admin account
- **Insecure mode**: Runs without TLS internally (Traefik Gateway terminates TLS at the edge)
- **Configuration**: Use `server.insecure: "true"` in `argocd-cm` ConfigMap (NOT deployment args)
- **RBAC**: Groups mapped via `argocd-rbac-cm` ConfigMap (`argocd-admins` → `role:admin`)
- **Private overlay**: Real URLs in `private/argocd/` for deployment
- **Dependencies**: None (standalone)

### Karakeep
- **Image**: `ghcr.io/karakeep-app/karakeep:0.31.0` (formerly Hoarder)
- **Database**: SQLite internal (WAL mode) — no external PostgreSQL/Redis
- **Dependencies**: Meilisearch (full-text search), Chrome headless (web scraping/screenshots)
- **Authentication**: OIDC via Authelia + local accounts
- **Health endpoint**: `/` (port 3000)
- **URL config**: `NEXTAUTH_URL` in private ConfigMap (`private/configmaps/karakeep-urls-configmap.yaml`)

### Stirling-PDF
- **Deployment strategy**: Must use `Recreate` (not `RollingUpdate`) - H2 database locks cause deadlock with rolling updates
- **Memory**: Requires 2Gi limit (OOM on Metaspace with 1Gi after v2.7.0)
- **Startup time**: First boot takes 2-3 minutes (font installation)
- **Probes**: Use `startupProbe` with generous timeouts (30s initial, 30 failures allowed)
- **Health endpoint**: `/api/v1/info/status`
- **No authentication**: Open on LAN (remote access goes through WireGuard on the LAN router)
- **Dependencies**: None (standalone)

### Shared Services
- **PostgreSQL init**: Uses `postgres-init.sh` script with ConfigMap to create multiple databases
- **Redis**: Single instance with multiple DBs (0, 1)
- **No HTTPRoute**: Internal services only (consumed from inside the cluster via ClusterIP)

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

### Immutable Selector Conflicts (Kustomize)
If `kubectl apply -k` or ArgoCD sync fails with `spec.selector: Invalid value: ... field is immutable`, it means a label was added/removed from `selector.matchLabels`. Fix: delete the affected deployment(s) and let ArgoCD recreate them. PVCs and data are preserved.

### External Secrets Not Syncing
Check SecretStore status:
```bash
kubectl get clustersecretstore bitwarden-cluster-secretstore
kubectl describe externalsecret <name> -n <namespace>
kubectl logs -n external-secrets-system deployment/external-secrets
```

### HTTPRoute Not Routing
If `https://<app>.homelab.lastsector.lan` returns 404 or never connects:
1. Check the HTTPRoute is Accepted:
   ```bash
   kubectl describe httproute <app> -n <namespace>     # look at Conditions
   ```
2. Check the Gateway is Programmed and has an address:
   ```bash
   kubectl describe gateway homelab-gateway -n traefik
   ```
3. Check Traefik picked it up: `kubectl logs -n traefik deploy/traefik | grep -i <app>`
4. DNS: `dig +short <app>.homelab.lastsector.lan` must return the node IP

### Certificate Not Ready
If the wildcard cert or any per-app Certificate is `Ready=False`:
```bash
kubectl describe certificate <name> -n <namespace>
kubectl describe certificaterequest -n <namespace>
kubectl logs -n cert-manager deploy/cert-manager
```
Common cause: missing or broken `homelab-ca-issuer` ClusterIssuer (check `homelab-root-ca` secret exists in `cert-manager` namespace).

### Traefik Not Reachable on Node IP
If `https://<app>.homelab.lastsector.lan` times out and DNS resolves correctly to the node IP:
```bash
kubectl get pods -n traefik -o wide                # IP should equal node IP (hostNetwork)
kubectl exec -n traefik deploy/traefik -- ss -tlnp  # ports 80 and 443 listening
cilium status                                       # Cilium agent healthy
```
Common cause: another host process bound to :80 or :443 on the node (Tailscale operator, NodePort range collision). Talos doesn't run extra services, so usually nothing else competes.

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
