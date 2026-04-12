# Mealie Deployment Design

**Date**: 2026-04-12
**Status**: Approved, ready for implementation plan
**Author**: Cedric + Claude

## Context

Add a recipe management application to the homelab. Primary use case is a recipe library (browse, search, import from web, consult on mobile while cooking); meal planning and shopping lists are secondary nice-to-haves.

## Decision: Mealie

Chose **Mealie** over Tandoor after comparing both for the stated priority.

**Why Mealie:**
- Modern Vue 3 UI — better "library consultation" UX than Tandoor's Django interface
- Strong URL import via `recipe-scrapers`
- Single FastAPI process — lighter than Tandoor's Django + Celery worker + Redis stack
- Native OIDC support (stable in v2.x)
- French available: `fr-FR`, `fr-BE`, `fr-CA` locales shipped (set per-user in UI)

**Trade-off accepted:** Meal planning and shopping list features are more basic than Tandoor's (no supermarket aisle ordering, no pantry/stock tracking). Acceptable since library is the priority.

## Architecture

Follows the standard homelab application pattern (identical to Miniflux and Karakeep).

### Dependencies

- **`shared-services/postgres`**: new `mealie` database + dedicated user (created manually once via `kubectl exec`, since the init script only runs on first Postgres boot)
- **Authelia**: new OIDC client `mealie` + new group `mealie-users`. The existing global `admins` group is reused for admin rights.
- **Bitwarden Secrets Manager**: two new secrets — `mealie-db-password`, `mealie-oidc-client-secret`

No Redis required. No separate worker. No dedicated PostgreSQL.

### File layout

```
apps/mealie/
├── README.md                    # French doc, same style as other apps
└── base/
    ├── kustomization.yaml
    ├── namespace.yaml           # privileged PodSecurity labels
    ├── pvc.yaml                 # 10Gi local-path for /app/data
    ├── external-secrets.yaml    # db password + OIDC client secret
    ├── mealie-deployment.yaml   # ghcr.io/mealie-recipes/mealie:v2.8.0
    ├── mealie-service.yaml      # ClusterIP, port 9000
    └── tailscale-ingress.yaml   # mealie.tail<id>.ts.net
```

Plus:
- `apps/argocd/applications/mealie.yaml` — ArgoCD Application for GitOps sync
- `private/configmaps/mealie-urls-configmap.yaml` — `BASE_URL` with real Tailscale URL (gitignored)

## Deployment details

### Image and resources

```yaml
image: ghcr.io/mealie-recipes/mealie:v2.8.0
resources:
  requests: { cpu: 100m, memory: 256Mi }
  limits:   { cpu: 1000m, memory: 768Mi }
```

Renovate will track the image tag through the existing configuration.

### Environment variables

| Variable | Source | Value |
|---|---|---|
| `DB_ENGINE` | ConfigMap | `postgres` |
| `POSTGRES_SERVER` | ConfigMap | `postgres.shared-services.svc.cluster.local` |
| `POSTGRES_PORT` | ConfigMap | `5432` |
| `POSTGRES_DB` | ConfigMap | `mealie` |
| `POSTGRES_USER` | ConfigMap | `mealie` |
| `POSTGRES_PASSWORD` | Secret (ESO) | from Bitwarden `mealie-db-password` |
| `BASE_URL` | private ConfigMap | `https://mealie.tail<id>.ts.net` |
| `ALLOW_SIGNUP` | ConfigMap | `false` |
| `TZ` | ConfigMap | `Europe/Paris` |
| `OIDC_AUTH_ENABLED` | ConfigMap | `true` |
| `OIDC_CONFIGURATION_URL` | ConfigMap | `https://auth.tail<id>.ts.net/.well-known/openid-configuration` |
| `OIDC_CLIENT_ID` | ConfigMap | `mealie` |
| `OIDC_CLIENT_SECRET` | Secret (ESO) | from Bitwarden `mealie-oidc-client-secret` |
| `OIDC_AUTO_REDIRECT_LOGIN` | ConfigMap | `false` (keep local login as fallback) |
| `OIDC_PROVIDER_NAME` | ConfigMap | `Authelia` |
| `OIDC_USER_GROUP` | ConfigMap | `mealie-users` |
| `OIDC_ADMIN_GROUP` | ConfigMap | `admins` (global admin group, reused) |

### Storage

Single PVC `mealie-data`, 10Gi, `local-path` storage class, mounted at `/app/data`. Holds recipe images, user uploads, and backups.

### Health probes (port 9000, path `/api/app/about`)

- `startupProbe`: 5s period × 30 failures (~2.5 min budget for first boot)
- `livenessProbe`: 30s period
- `readinessProbe`: 10s period

### Ingress

Standard Tailscale ingress with short hostname annotation:

```yaml
annotations:
  tailscale.com/hostname: mealie
ingressClassName: tailscale
tls:
  - hosts: [mealie]
```

Accessible at `https://mealie.tail<id>.ts.net`.

## OIDC / Authelia configuration

### Authelia client

Added to Authelia's private config (outside this repo):

```yaml
- client_id: mealie
  client_name: Mealie
  client_secret: '<hashed>'
  public: false
  authorization_policy: two_factor
  redirect_uris:
    - https://mealie.tail<id>.ts.net/login
    - https://mealie.tail<id>.ts.net/api/auth/callback/oidc
  scopes: [openid, profile, email, groups]
  userinfo_signed_response_alg: none
```

### Authelia groups

- **New**: `mealie-users` — required for any access to Mealie
- **Reused**: `admins` — existing global admin group; members automatically get admin rights in Mealie via `OIDC_ADMIN_GROUP` mapping

This is simpler than Paperless's flow: Mealie's `OIDC_ADMIN_GROUP` handles admin promotion automatically at login, no manual `manage.py` step needed.

## Secrets

`ExternalSecret` `mealie-secrets` in namespace `mealie`:

```yaml
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: bitwarden-cluster-secretstore
    kind: ClusterSecretStore
  target:
    name: mealie-secrets
  data:
    - secretKey: POSTGRES_PASSWORD
      remoteRef: { key: mealie-db-password }
    - secretKey: OIDC_CLIENT_SECRET
      remoteRef: { key: mealie-oidc-client-secret }
```

Both secrets must exist in Bitwarden before deployment.

## ArgoCD Application

New file `apps/argocd/applications/mealie.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mealie
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/CedricLphn/homelab
    targetRevision: main
    path: apps/mealie/base
  destination:
    server: https://kubernetes.default.svc
    namespace: mealie
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [CreateNamespace=false]
```

## Deployment sequence (one-time setup)

1. Create both secrets in Bitwarden Secrets Manager
2. Create OIDC client `mealie` and group `mealie-users` in Authelia; assign user to `mealie-users` (and confirm `admins` membership for admin access)
3. Create the `mealie` database and user in shared Postgres:
   ```bash
   kubectl exec -n shared-services deployment/postgres -- psql -U postgres -c \
     "CREATE USER mealie WITH PASSWORD '<password-from-bitwarden>'; \
      CREATE DATABASE mealie OWNER mealie; \
      GRANT ALL PRIVILEGES ON DATABASE mealie TO mealie;"
   ```
4. Create `private/configmaps/mealie-urls-configmap.yaml` with the real Tailscale URL
5. Commit and push `apps/mealie/` and `apps/argocd/applications/mealie.yaml`
6. Wait for ArgoCD sync; watch `kubectl get pods -n mealie -w`
7. First login via Authelia; verify admin promotion is automatic
8. Set default site language to French in Mealie UI (Settings → Site Settings)

## Testing / validation

- `kubectl get externalsecret -n mealie` — should show `SecretSynced`
- `kubectl get pods -n mealie` — should reach `Running` / `Ready`
- `curl -sk https://mealie.tail<id>.ts.net/api/app/about` — returns version JSON
- Login via Authelia redirects back to Mealie dashboard
- User from `admins` group has admin menu available
- Import one recipe via URL to verify `recipe-scrapers` works
- Upload a recipe image to verify PVC write path

## Out of scope

- Backups (Mealie has built-in backup/restore in UI; automated cluster-level backups of the PVC are deferred)
- Meal plan / shopping list configuration (left to user)
- Migration from any existing recipe collection
