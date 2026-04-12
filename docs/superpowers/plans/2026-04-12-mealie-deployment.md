# Mealie Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deploy Mealie recipe manager on the homelab Kubernetes cluster, wired to shared PostgreSQL and Authelia OIDC, managed by ArgoCD.

**Architecture:** A single FastAPI Deployment in a dedicated `mealie` namespace, backed by shared PostgreSQL (new `mealie` DB), authenticated via Authelia OIDC (reusing the global `admins` group), exposed via Tailscale ingress. All manifests live under `apps/mealie/base/` and follow the exact patterns used by Karakeep and Miniflux. ArgoCD auto-syncs via an Application CR in `argocd-apps/`.

**Tech Stack:** Kustomize (no Helm), External Secrets Operator v1 + Bitwarden, Tailscale Ingress, ArgoCD, Talos Linux, local-path storage.

**Spec:** `docs/superpowers/specs/2026-04-12-mealie-design.md`

---

## Prerequisites (manual, outside the repo)

These must be done by a human with access to Bitwarden, Authelia, and the cluster before ArgoCD tries to deploy. They are NOT automated steps.

### Prerequisite A: Create Bitwarden secrets

In Bitwarden Secrets Manager, create four secrets in the same project the other homelab apps use:

| Secret name | Value |
|---|---|
| `mealie-db-password` | Generate a 32+ char password (will be the Postgres user password) |
| `mealie-oidc-client-id` | `mealie` |
| `mealie-oidc-client-secret` | Generate a 48+ char secret (will be given to Authelia in hashed form) |
| `mealie-oidc-wellknown-url` | `https://auth.tail<id>.ts.net/.well-known/openid-configuration` (use the actual Authelia Tailscale hostname) |

Write down the UUID of each secret — they are referenced by key (name) in the ExternalSecret, so you don't need them in the manifest, but keep them handy for debugging.

### Prerequisite B: Configure Authelia

In the private Authelia config (outside this repo), add the new OIDC client:

```yaml
- client_id: mealie
  client_name: Mealie
  client_secret: '<argon2id hash of mealie-oidc-client-secret>'
  public: false
  authorization_policy: two_factor
  redirect_uris:
    - https://mealie.tail<id>.ts.net/login
    - https://mealie.tail<id>.ts.net/login?direct=1
  scopes:
    - openid
    - profile
    - email
    - groups
  userinfo_signed_response_alg: none
```

Hash the client secret with: `docker run authelia/authelia:latest authelia crypto hash generate argon2 --password '<plain-secret>'`

Create the new group `mealie-users` in Authelia's users database and assign it to whoever should access Mealie. Confirm your own user already has `admins` in their group list (for automatic admin promotion in Mealie).

Reload Authelia (whatever mechanism your deployment uses — usually a `kubectl rollout restart` of the Authelia deployment).

### Prerequisite C: Create the Mealie database in shared PostgreSQL

The `postgres-init.sh` ConfigMap only runs on first-time Postgres init. Since Postgres is already running with existing data, create the DB manually:

```bash
# Get the Bitwarden password value for mealie-db-password, substitute below
MEALIE_DB_PASSWORD='<paste-from-bitwarden>'

kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c \
  "CREATE USER mealie WITH PASSWORD '${MEALIE_DB_PASSWORD}';"

kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c \
  "CREATE DATABASE mealie OWNER mealie;"

kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c \
  "GRANT ALL PRIVILEGES ON DATABASE mealie TO mealie;"
```

Verify: `kubectl exec -n shared-services deployment/postgres -- psql -U postgres -c "\l mealie"` should show the database.

### Prerequisite D: Create the private ConfigMap

The `BASE_URL` contains the private Tailscale hostname and stays out of the git repo. Create the file locally in the gitignored `private/` directory:

```bash
cat > /Users/cedric/IdeaProjects/homelab/private/configmaps/mealie-urls-configmap.yaml <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: mealie-urls
  namespace: mealie
data:
  BASE_URL: "https://mealie.tail<id>.ts.net"
EOF
```

Replace `<id>` with the actual Tailscale tailnet ID (same one used by `karakeep.tail<id>.ts.net`). You can find it by running `kubectl get ingress -n karakeep karakeep-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'`.

This ConfigMap must be applied manually AFTER the namespace exists (done in Task 7):
```bash
kubectl apply -f /Users/cedric/IdeaProjects/homelab/private/configmaps/mealie-urls-configmap.yaml
```

---

## File Structure

Files created by this plan:

- `apps/mealie/base/namespace.yaml` — Namespace with PodSecurity privileged labels
- `apps/mealie/base/pvc.yaml` — 10Gi `local-path` PVC for `/app/data`
- `apps/mealie/base/external-secrets.yaml` — Two ExternalSecrets (`mealie-secret`, `mealie-oidc-secret`)
- `apps/mealie/base/mealie-deployment.yaml` — Single Deployment, image `ghcr.io/mealie-recipes/mealie:v3.14.0`
- `apps/mealie/base/mealie-service.yaml` — ClusterIP on port 9000
- `apps/mealie/base/tailscale-ingress.yaml` — Ingress with `tailscale.com/hostname: mealie`
- `apps/mealie/base/kustomization.yaml` — Kustomize manifest listing all resources (uses `labels:` form, not `commonLabels`)
- `apps/mealie/README.md` — French documentation matching Karakeep style
- `argocd-apps/mealie.yaml` — ArgoCD Application CR (picked up by the `apps` app-of-apps)

File modified:

- `renovate.json` — add Mealie image tracking

Files created locally but NOT committed (already gitignored via `private/`):

- `private/configmaps/mealie-urls-configmap.yaml` (see Prerequisite D)

---

## Task 1: Base Kustomize scaffold and namespace

**Files:**
- Create: `apps/mealie/base/namespace.yaml`
- Create: `apps/mealie/base/kustomization.yaml`

- [ ] **Step 1.1: Create `apps/mealie/base/namespace.yaml`**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mealie
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

- [ ] **Step 1.2: Create `apps/mealie/base/kustomization.yaml`** (initial skeleton, will be expanded as tasks progress)

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: mealie
resources:
  - namespace.yaml
labels:
  - pairs:
      app.kubernetes.io/part-of: mealie
    includeSelectors: false
```

- [ ] **Step 1.3: Verify the kustomization builds**

Run:
```bash
kubectl kustomize apps/mealie/base/
```

Expected: YAML output containing only the Namespace, no errors. If `kubectl kustomize` is not available, use `kustomize build apps/mealie/base/` instead.

- [ ] **Step 1.4: Do NOT commit yet** — we'll commit the complete base manifests together at the end of Task 6 for a clean single commit.

---

## Task 2: PVC

**Files:**
- Create: `apps/mealie/base/pvc.yaml`
- Modify: `apps/mealie/base/kustomization.yaml`

- [ ] **Step 2.1: Create `apps/mealie/base/pvc.yaml`**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mealie-data
  namespace: mealie
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi
```

- [ ] **Step 2.2: Add `pvc.yaml` to `apps/mealie/base/kustomization.yaml`**

The `resources:` list becomes:
```yaml
resources:
  - namespace.yaml
  - pvc.yaml
```

- [ ] **Step 2.3: Verify build**

Run: `kubectl kustomize apps/mealie/base/`
Expected: output now contains both the Namespace and the PersistentVolumeClaim.

---

## Task 3: ExternalSecrets

**Files:**
- Create: `apps/mealie/base/external-secrets.yaml`
- Modify: `apps/mealie/base/kustomization.yaml`

- [ ] **Step 3.1: Create `apps/mealie/base/external-secrets.yaml`**

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mealie-secret
  namespace: mealie
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: bitwarden-cluster-secretstore
    kind: ClusterSecretStore
  target:
    name: mealie-secret
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: mealie-db-password
---
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: mealie-oidc-secret
  namespace: mealie
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: bitwarden-cluster-secretstore
    kind: ClusterSecretStore
  target:
    name: mealie-oidc-secret
    creationPolicy: Owner
  data:
    - secretKey: client-id
      remoteRef:
        key: mealie-oidc-client-id
    - secretKey: client-secret
      remoteRef:
        key: mealie-oidc-client-secret
    - secretKey: wellknown-url
      remoteRef:
        key: mealie-oidc-wellknown-url
```

- [ ] **Step 3.2: Add `external-secrets.yaml` to `apps/mealie/base/kustomization.yaml`**

```yaml
resources:
  - namespace.yaml
  - pvc.yaml
  - external-secrets.yaml
```

- [ ] **Step 3.3: Verify build**

Run: `kubectl kustomize apps/mealie/base/`
Expected: both ExternalSecret resources appear in the output, with `apiVersion: external-secrets.io/v1` (NOT v1beta1).

---

## Task 4: Deployment

**Files:**
- Create: `apps/mealie/base/mealie-deployment.yaml`
- Modify: `apps/mealie/base/kustomization.yaml`

- [ ] **Step 4.1: Create `apps/mealie/base/mealie-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mealie
  namespace: mealie
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: mealie
  template:
    metadata:
      labels:
        app: mealie
    spec:
      containers:
        - name: mealie
          image: ghcr.io/mealie-recipes/mealie:v3.14.0
          ports:
            - containerPort: 9000
              name: http
          env:
            - name: TZ
              value: Europe/Paris
            - name: ALLOW_SIGNUP
              value: "false"
            - name: DB_ENGINE
              value: postgres
            - name: POSTGRES_SERVER
              value: postgres.shared-services.svc.cluster.local
            - name: POSTGRES_PORT
              value: "5432"
            - name: POSTGRES_DB
              value: mealie
            - name: POSTGRES_USER
              value: mealie
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mealie-secret
                  key: db-password
            - name: BASE_URL
              valueFrom:
                configMapKeyRef:
                  name: mealie-urls
                  key: BASE_URL
            - name: OIDC_AUTH_ENABLED
              value: "true"
            - name: OIDC_SIGNUP_ENABLED
              value: "true"
            - name: OIDC_AUTO_REDIRECT
              value: "false"
            - name: OIDC_PROVIDER_NAME
              value: Authelia
            - name: OIDC_USER_GROUP
              value: mealie-users
            - name: OIDC_ADMIN_GROUP
              value: admins
            - name: OIDC_CONFIGURATION_URL
              valueFrom:
                secretKeyRef:
                  name: mealie-oidc-secret
                  key: wellknown-url
            - name: OIDC_CLIENT_ID
              valueFrom:
                secretKeyRef:
                  name: mealie-oidc-secret
                  key: client-id
            - name: OIDC_CLIENT_SECRET
              valueFrom:
                secretKeyRef:
                  name: mealie-oidc-secret
                  key: client-secret
          volumeMounts:
            - name: data
              mountPath: /app/data
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "768Mi"
              cpu: "1000m"
          startupProbe:
            httpGet:
              path: /api/app/about
              port: 9000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30
          livenessProbe:
            httpGet:
              path: /api/app/about
              port: 9000
            periodSeconds: 30
            timeoutSeconds: 10
          readinessProbe:
            httpGet:
              path: /api/app/about
              port: 9000
            periodSeconds: 10
            timeoutSeconds: 5
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: mealie-data
```

**Notes on choices:**
- `strategy: Recreate` avoids the situation where two pods hit the same SQLite-locked files or step on each other during rolling updates. Mealie stores some runtime state in `/app/data`.
- `/api/app/about` is Mealie's unauthenticated health/version endpoint — safe for probes.
- `startupProbe` gives 10s + (5s × 30) = 160s budget for first boot — generous enough for cold starts and DB migrations.

- [ ] **Step 4.2: Add `mealie-deployment.yaml` to kustomization resources**

```yaml
resources:
  - namespace.yaml
  - pvc.yaml
  - external-secrets.yaml
  - mealie-deployment.yaml
```

- [ ] **Step 4.3: Verify build**

Run: `kubectl kustomize apps/mealie/base/`
Expected: Deployment appears. Check that the `app.kubernetes.io/part-of: mealie` label is on the Deployment metadata but NOT on `spec.selector.matchLabels` (this is what the `includeSelectors: false` achieves — confirms we avoided the immutable-selector trap noted in CLAUDE.md).

---

## Task 5: Service

**Files:**
- Create: `apps/mealie/base/mealie-service.yaml`
- Modify: `apps/mealie/base/kustomization.yaml`

- [ ] **Step 5.1: Create `apps/mealie/base/mealie-service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mealie
  namespace: mealie
spec:
  selector:
    app: mealie
  ports:
    - port: 9000
      targetPort: 9000
      name: http
```

- [ ] **Step 5.2: Add to kustomization resources**

```yaml
resources:
  - namespace.yaml
  - pvc.yaml
  - external-secrets.yaml
  - mealie-deployment.yaml
  - mealie-service.yaml
```

- [ ] **Step 5.3: Verify build**

Run: `kubectl kustomize apps/mealie/base/`
Expected: Service with selector `app: mealie` and port 9000.

---

## Task 6: Tailscale Ingress

**Files:**
- Create: `apps/mealie/base/tailscale-ingress.yaml`
- Modify: `apps/mealie/base/kustomization.yaml`

- [ ] **Step 6.1: Create `apps/mealie/base/tailscale-ingress.yaml`**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mealie-ingress
  namespace: mealie
  annotations:
    tailscale.com/hostname: mealie
spec:
  ingressClassName: tailscale
  tls:
    - hosts:
        - mealie
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mealie
                port:
                  number: 9000
```

- [ ] **Step 6.2: Add to kustomization resources**

Final `apps/mealie/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: mealie
resources:
  - namespace.yaml
  - pvc.yaml
  - external-secrets.yaml
  - mealie-deployment.yaml
  - mealie-service.yaml
  - tailscale-ingress.yaml
labels:
  - pairs:
      app.kubernetes.io/part-of: mealie
    includeSelectors: false
```

- [ ] **Step 6.3: Verify full build**

Run: `kubectl kustomize apps/mealie/base/ > /tmp/mealie-built.yaml && wc -l /tmp/mealie-built.yaml`
Expected: full output with no errors, approximately 180-220 lines. Grep to confirm all resources present:
```bash
grep -E "^kind:" /tmp/mealie-built.yaml | sort -u
```
Expected output:
```
kind: Deployment
kind: ExternalSecret
kind: Ingress
kind: Namespace
kind: PersistentVolumeClaim
kind: Service
```

- [ ] **Step 6.4: Commit the base manifests**

```bash
git add apps/mealie/base/
git commit -m "$(cat <<'EOF'
feat(mealie): add base Kustomize manifests

Single Deployment backed by shared Postgres, OIDC via Authelia
(reuses global admins group), Tailscale ingress. Matches the
Karakeep/Miniflux pattern: ESO v1, labels (not commonLabels),
PodSecurity privileged.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: French README

**Files:**
- Create: `apps/mealie/README.md`

- [ ] **Step 7.1: Create `apps/mealie/README.md`**

Match the Karakeep README style (headings, tables, French). Content:

````markdown
# Mealie

Gestionnaire de recettes self-hosted avec import depuis le web, recherche, planification de repas et liste de courses.

## Composants

| Composant | Image | Port | Role |
|-----------|-------|------|------|
| Mealie | `ghcr.io/mealie-recipes/mealie:v3.14.0` | 9000 | Application FastAPI + frontend Nuxt |

## Architecture

- **Base de donnees** : PostgreSQL partagee (`shared-services/postgres`), DB `mealie`
- **Stockage** : PVC `mealie-data` 10Gi en `local-path` pour `/app/data` (recettes, images, backups)
- **Authentification** : Authelia via OIDC — groupe `mealie-users` obligatoire, groupe global `admins` pour les droits admin
- **Ingress** : Tailscale (`https://mealie.tail<ID>.ts.net`)
- **Langue** : francais (fr-FR) configurable par utilisateur en UI, et par defaut au niveau site dans Settings

## Secrets Bitwarden

| Secret | Usage |
|--------|-------|
| `mealie-db-password` | Mot de passe Postgres (user `mealie`) |
| `mealie-oidc-client-id` | Client ID Authelia |
| `mealie-oidc-client-secret` | Client secret Authelia |
| `mealie-oidc-wellknown-url` | URL discovery OIDC Authelia (contient l'URL Tailscale privee) |

## ConfigMap privee

Le `BASE_URL` est dans un ConfigMap `mealie-urls` non committe :
```
private/configmaps/mealie-urls-configmap.yaml
```

## Deploiement

```bash
# Premier deploiement (manuel, une fois)
kubectl apply -f private/configmaps/mealie-urls-configmap.yaml

# Ensuite ArgoCD sync depuis argocd-apps/mealie.yaml
```

## Creation DB (une fois)

Le script postgres-init ne tourne qu'au premier boot de Postgres. Pour ajouter la DB apres coup :

```bash
kubectl exec -n shared-services deployment/postgres -- \
  psql -U postgres -c "CREATE USER mealie WITH PASSWORD '<password>'; \
                       CREATE DATABASE mealie OWNER mealie; \
                       GRANT ALL PRIVILEGES ON DATABASE mealie TO mealie;"
```

## Validation

```bash
kubectl get pods -n mealie
kubectl get externalsecret -n mealie  # doit etre SecretSynced
kubectl get ingress -n mealie
curl -sk https://mealie.tail<ID>.ts.net/api/app/about
```

## OIDC Authelia

Le groupe `admins` existant est reutilise pour les droits admin — `OIDC_ADMIN_GROUP=admins`. Les utilisateurs membres de `mealie-users` peuvent se connecter, ceux membres de `admins` obtiennent automatiquement les droits admin au premier login (pas de promotion manuelle necessaire, contrairement a Paperless).
````

- [ ] **Step 7.2: Commit the README**

```bash
git add apps/mealie/README.md
git commit -m "$(cat <<'EOF'
docs(mealie): add French README

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: ArgoCD Application

**Files:**
- Create: `argocd-apps/mealie.yaml`

- [ ] **Step 8.1: Create `argocd-apps/mealie.yaml`**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mealie
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: homelab
  source:
    repoURL: https://github.com/CedricLphn/homelab.git
    targetRevision: main
    path: apps/mealie/base
  destination:
    server: https://kubernetes.default.svc
    namespace: mealie
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Important: `project: homelab` (matches existing apps), `finalizers:` for proper cleanup on deletion, `CreateNamespace=true` for safety even though we also ship a namespace manifest.

- [ ] **Step 8.2: Commit**

```bash
git add argocd-apps/mealie.yaml
git commit -m "$(cat <<'EOF'
feat(argocd): add Mealie Application

Auto-synced from apps/mealie/base via the app-of-apps.

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Renovate tracking

**Files:**
- Modify: `renovate.json`

- [ ] **Step 9.1: Add a Mealie tracking rule to `renovate.json`**

Insert a new entry in the `packageRules` array, after the existing "Track Miniflux (semver)" rule and before "Track gotenberg major version 8 only":

```json
{
  "description": "Track Mealie (semver)",
  "matchDatasources": ["docker"],
  "matchPackageNames": [
    "ghcr.io/mealie-recipes/mealie"
  ],
  "versioning": "semver"
},
```

The resulting block (showing context) looks like:

```json
{
  "description": "Track Miniflux (semver)",
  "matchDatasources": ["docker"],
  "matchPackageNames": [
    "miniflux/miniflux"
  ],
  "versioning": "semver"
},
{
  "description": "Track Mealie (semver)",
  "matchDatasources": ["docker"],
  "matchPackageNames": [
    "ghcr.io/mealie-recipes/mealie"
  ],
  "versioning": "semver"
},
{
  "description": "Track gotenberg major version 8 only",
  ...
}
```

- [ ] **Step 9.2: Validate JSON**

Run: `python3 -m json.tool renovate.json > /dev/null && echo OK`
Expected: `OK`. If it fails, there's a trailing comma or missing brace.

- [ ] **Step 9.3: Commit**

```bash
git add renovate.json
git commit -m "$(cat <<'EOF'
chore(renovate): track Mealie image updates

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Run the prerequisites and apply the private ConfigMap

This task is where the manual prerequisites (A, B, C, D from the top of the plan) actually get executed against the cluster. Before proceeding, the engineer MUST have completed Prerequisites A, B, C.

- [ ] **Step 10.1: Confirm Prerequisites A (Bitwarden), B (Authelia), C (Postgres DB) are done**

Ask yourself:
- Are the four Bitwarden secrets created? (check with Bitwarden CLI or UI)
- Is the Authelia OIDC client `mealie` created and Authelia restarted?
- Does `kubectl exec -n shared-services deployment/postgres -- psql -U postgres -c "\l mealie"` return a row?

If any answer is "no", go back and finish them before continuing.

- [ ] **Step 10.2: Create `private/configmaps/mealie-urls-configmap.yaml`**

```bash
# Find the tailnet ID from an existing ingress
TAILNET_ID=$(kubectl get ingress -n karakeep karakeep-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' \
  | sed -n 's/karakeep\.\(tail[^.]*\)\.ts\.net/\1/p')
echo "Tailnet ID: ${TAILNET_ID}"
```

Expected: something like `tail3161aa`.

Then create the file:

```bash
cat > /Users/cedric/IdeaProjects/homelab/private/configmaps/mealie-urls-configmap.yaml <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mealie-urls
  namespace: mealie
data:
  BASE_URL: "https://mealie.${TAILNET_ID}.ts.net"
EOF
```

Verify:
```bash
cat /Users/cedric/IdeaProjects/homelab/private/configmaps/mealie-urls-configmap.yaml
```

- [ ] **Step 10.3: Verify the file is NOT tracked by git**

Run: `git status private/`
Expected: `private/` is gitignored, the file should NOT appear in `git status`. If it does, check the `.gitignore` — it should already contain a `private/` entry.

---

## Task 11: Push and deploy

- [ ] **Step 11.1: Push all commits to main**

```bash
git push origin main
```

Expected: all four commits from tasks 6, 7, 8, 9 land on `main`.

- [ ] **Step 11.2: Wait for ArgoCD to detect the new Application**

ArgoCD's app-of-apps polls every few minutes. To trigger immediately:

```bash
kubectl annotate application apps -n argocd \
  argocd.argoproj.io/refresh=hard --overwrite
```

Verify the `mealie` Application appears:

```bash
kubectl get application -n argocd mealie
```

Expected: `NAME: mealie`, sync status transitions to `Synced`.

- [ ] **Step 11.3: Apply the private ConfigMap**

The namespace now exists (created by ArgoCD's sync). Apply the private ConfigMap:

```bash
kubectl apply -f /Users/cedric/IdeaProjects/homelab/private/configmaps/mealie-urls-configmap.yaml
```

Expected: `configmap/mealie-urls created`.

- [ ] **Step 11.4: Trigger a deployment rollout to pick up the ConfigMap**

The Deployment pod may have started before the ConfigMap existed and be stuck in `CreateContainerConfigError`. Force a rollout:

```bash
kubectl rollout restart deployment/mealie -n mealie
```

- [ ] **Step 11.5: Watch pod come up**

```bash
kubectl get pods -n mealie -w
```

Expected: `mealie-xxx` pod transitions `Pending` → `ContainerCreating` → `Running` → `Ready 1/1` within ~2 minutes. First boot is slower because of DB migrations.

If stuck in `CrashLoopBackOff`, run:
```bash
kubectl logs -n mealie deployment/mealie --tail=100
```
Most likely causes: wrong Postgres password, missing DB, OIDC URL wrong. Do NOT proceed to Task 12 until the pod is `Ready 1/1`.

---

## Task 12: Validate

- [ ] **Step 12.1: Verify ExternalSecrets are synced**

```bash
kubectl get externalsecret -n mealie
```

Expected:
```
NAME                  STORE                             REFRESH INTERVAL   STATUS         READY
mealie-secret         bitwarden-cluster-secretstore     1h                 SecretSynced   True
mealie-oidc-secret    bitwarden-cluster-secretstore     1h                 SecretSynced   True
```

If `STATUS` is `SecretSyncedError`, check:
```bash
kubectl describe externalsecret mealie-secret -n mealie
kubectl describe externalsecret mealie-oidc-secret -n mealie
```
Most common cause: secret name in Bitwarden doesn't match `remoteRef.key` exactly.

- [ ] **Step 12.2: Verify the Ingress has a hostname**

```bash
kubectl get ingress -n mealie mealie-ingress
```

Expected: `HOSTS` column shows something like `mealie.tail<id>.ts.net`. Can take 30-60s after pod comes up.

- [ ] **Step 12.3: Smoke-test the health endpoint**

```bash
TAILNET_URL=$(kubectl get ingress -n mealie mealie-ingress \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -sk "https://${TAILNET_URL}/api/app/about"
```

Expected: JSON response with `version`, `production`, `allowSignup`, `enableOidc: true`, `oidcRedirect: false`. If you don't see `enableOidc: true`, OIDC env vars aren't being read — check the Deployment.

- [ ] **Step 12.4: Open the browser and test the full OIDC flow**

1. Open `https://${TAILNET_URL}` in a browser on the Tailscale network.
2. You should see Mealie's login page with BOTH a local login form AND a "Login with Authelia" button.
3. Click "Login with Authelia" → you're redirected to Authelia → log in → back to Mealie.
4. You land on the Mealie dashboard as a logged-in user.
5. If your account is in the `admins` group, the admin panel icon/link should be visible (top-right user menu).

**If OIDC fails at the Authelia redirect**, the most common causes are:
- Redirect URIs in Authelia don't match — check Mealie's callback URL in the browser error or Authelia logs.
- The `mealie-oidc-wellknown-url` in Bitwarden doesn't point to the right Authelia instance.
- The user isn't in the `mealie-users` group (Mealie will reject the login).

---

## Task 13: Post-deployment configuration

- [ ] **Step 13.1: Set French as default site language**

In the Mealie UI as an admin user:
1. Click the user menu → Site Settings (or navigate to `/admin/site-settings`).
2. Find "Default Language" (or "Language" in newer versions).
3. Select `Français (France)`.
4. Save.

New users created via OIDC will default to French.

- [ ] **Step 13.2: Set your own language to French**

1. User menu → User Settings.
2. Change "Language" to `Français (France)`.
3. Save and refresh — the UI should now be in French.

- [ ] **Step 13.3: Import one test recipe to verify `recipe-scrapers` works**

Pick any recipe URL (e.g., marmiton.org or any food blog that supports schema.org Recipe), click "+ Create" → "Import from URL", paste the URL, and confirm the recipe is imported with title, ingredients, steps, and image.

- [ ] **Step 13.4: Final health check**

```bash
kubectl get pods -n mealie
kubectl get ingress -n mealie
kubectl get externalsecret -n mealie
kubectl get application -n argocd mealie
```

Expected: everything `Running` / `Synced` / `SecretSynced` / `Healthy`.

If all of the above are green, Mealie is deployed and the plan is complete.

---

## Out of scope (do NOT do in this plan)

- PVC backups (mentioned in spec as deferred)
- Meal plan / shopping list configuration (user-driven, not infra)
- Migration from any existing recipe source
- Adding Mealie to any shared service discovery (there isn't one)
