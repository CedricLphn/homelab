# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a homelab Kubernetes configuration repository using Kustomize for deploying self-hosted applications on Talos Linux.

## Infrastructure Stack

- **Kubernetes Distribution**: Talos Linux
- **Configuration Management**: Kustomize
- **Storage**: local-path StorageClass (local-path-provisioner)
- **Secrets Management**: Bitwarden External Secrets Operator with ClusterSecretStore named `bitwarden-cluster-secretstore`
- **Ingress**: Tailscale Operator (standard Kubernetes Ingress with `ingressClassName: tailscale`)
- **Authentication**: Authelia for OAuth/OIDC

## Key Architectural Constraints

### Talos Linux Specifics
- All namespaces require privileged PodSecurity labels to allow local-path provisioner:
```yaml
labels:
  pod-security.kubernetes.io/enforce: privileged
  pod-security.kubernetes.io/audit: privileged
  pod-security.kubernetes.io/warn: privileged
```
- All PVCs must use `storageClassName: local-path`

### External Secrets Operator
- API version: `external-secrets.io/v1` (not v1beta1)
- ClusterSecretStore name: `bitwarden-cluster-secretstore`
- All secrets are managed via Bitwarden Secret Manager

### Tailscale Ingress
- Use standard Kubernetes Ingress resource with `ingressClassName: tailscale`
- Do NOT use custom Tailscale CRDs
- Ingress creates Tailscale URLs in format: `https://<service>.tail<tailnet-id>.ts.net`

### Code Style
- No comments in YAML manifests
- Charts structured with `base/` directory containing all resources
- No overlays directory (homelab has no staging/production distinction)

## Chart Structure

Each application follows this pattern:
```
<app-name>/
├── README.md              # Comprehensive setup and configuration docs
└── base/
    ├── kustomization.yaml # Lists all resources
    ├── namespace.yaml     # With privileged PodSecurity labels
    ├── pvc.yaml          # All PVCs with local-path storage
    ├── external-secrets.yaml # Bitwarden secrets sync
    ├── <service>-deployment.yaml
    ├── <service>-service.yaml
    └── tailscale-ingress.yaml
```

## Common Commands

### Deploy an application
```bash
kubectl apply -k <app-name>/base/
```

### Check deployment status
```bash
kubectl get pods -n <namespace>
kubectl get ingress -n <namespace>
```

### View logs
```bash
kubectl logs -f deployment/<deployment-name> -n <namespace>
```

### Restart deployment
```bash
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

### Test application health
```bash
kubectl exec -n <namespace> deployment/<deployment-name> -- curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/ --max-time 5
```

### Execute commands in pods (useful for Django apps like Paperless)
```bash
kubectl exec -n <namespace> deployment/<deployment-name> -- <command>
```

## Application-Specific Notes

### Paperless-ngx
- Uses OAuth exclusively via Authelia (regular login disabled)
- Requires django-allauth with OpenID Connect provider
- OAuth users auto-created but need manual superuser promotion via Django shell
- Access control managed by Authelia rules, not Paperless itself
- Dependencies: PostgreSQL, Redis, Apache Tika, Gotenberg
- OAuth secret (`paperless-oauth-providers`) contains full JSON config in Bitwarden

### Immich
- PostgreSQL uses special VectorChord image: `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`
- Requires explicit `REDIS_PORT: "6379"` to avoid Kubernetes service env var conflicts
- Health check endpoint: `/api/server/ping`
- Dependencies: PostgreSQL (VectorChord), Redis, Machine Learning service

### Wallabag
- Read-it-later application using Symfony framework
- Does NOT support OIDC/OAuth for user authentication (only for API access)
- Bitwarden secrets must avoid special characters: `&`, `%`, `=`, and `#` at start (Symfony parameter parsing)
- Database initialization may require manual trigger: `kubectl exec -n wallabag deployment/wallabag -- /var/www/wallabag/bin/console wallabag:install --env=prod --no-interaction`
- Default credentials: `wallabag:wallabag` (change after first login)
- Dependencies: PostgreSQL, Redis
- Health check endpoint: `/`

## Known Issues and Solutions

### REDIS_PORT=NaN Error
When Redis service creates Kubernetes environment variables, they can conflict with Immich's expected format. Always explicitly set:
```yaml
- name: REDIS_PORT
  value: "6379"
```

### OAuth User Permissions
OAuth users in Paperless are created automatically but without admin rights. Promote via:
```bash
kubectl exec -n paperless-ngx deployment/paperless-ngx -- python3 manage.py shell -c "from django.contrib.auth import get_user_model; User = get_user_model(); u = User.objects.get(username='USERNAME'); u.is_superuser = True; u.is_staff = True; u.save()"
```

### CSRF Errors
For OAuth apps, always set:
```yaml
- name: PAPERLESS_URL
  value: https://<app>.tail<id>.ts.net
- name: PAPERLESS_CSRF_TRUSTED_ORIGINS
  value: https://<app>.tail<id>.ts.net
```

### Symfony Secret Character Restrictions
Wallabag (and other Symfony-based apps) cannot parse environment variables containing:
- `&` - Interpreted as parameter reference
- `%` - Interpreted as parameter placeholder
- `=` - Breaks parameter parsing
- `#` at start - Interpreted as comment

When creating Bitwarden secrets for Symfony apps, use only: alphanumeric, `-`, `_`, `~`, `@`, `!`, `*`, `/`, `+`, and `#` (not at start).

Error example: `You have requested a non-existent parameter "&"` indicates a secret contains `&`.

### PostgreSQL Password Changes
Changing a PostgreSQL password in Bitwarden requires complete reinitialization:
1. Delete deployment: `kubectl delete deployment/<name> -n <namespace>`
2. Delete PVC: `kubectl delete pvc/<name> -n <namespace>`
3. Redeploy with new password
PostgreSQL initializes with the password present at first startup and cannot be changed via External Secrets alone.
