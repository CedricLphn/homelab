# ArgoCD

[English](#english) | [Français](#français)

---

## English

Kubernetes deployment of ArgoCD with Kustomize, integrating Bitwarden External Secrets Operator, Tailscale Operator, and OIDC authentication via Authelia.

### Architecture

- **ArgoCD v2.13.2**: GitOps continuous delivery tool
- **Authelia OIDC**: OpenID Connect authentication
- **Tailscale**: Private network ingress

### Prerequisites

- Kubernetes cluster (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configured with a `ClusterSecretStore` named `bitwarden-cluster-secretstore`
- Tailscale Operator installed
- Authelia configured with an OIDC client for ArgoCD

### Authelia Configuration

Add the following client to Authelia configuration (`identity_providers.oidc.clients`):

```yaml
- client_id: 'argocd'
  client_name: 'ArgoCD'
  client_secret: '<hashed-secret>'
  public: false
  authorization_policy: 'two_factor'
  redirect_uris:
    - 'https://argocd.<your-tailnet-id>.ts.net/auth/callback'
  scopes:
    - 'openid'
    - 'profile'
    - 'email'
    - 'groups'
  response_types:
    - 'code'
  grant_types:
    - 'authorization_code'
```

**RBAC Groups in Authelia:**

Create groups in Authelia for ArgoCD access control:
- `argocd-admins`: Full admin access to ArgoCD
- `argocd-users`: Read-only access to ArgoCD

### Bitwarden Secrets

Create the following secrets in Bitwarden Secret Manager (key: value):

#### argocd-admin-password
```
key: argocd-admin-password
value: <admin-password>
```

**Generate**: `openssl rand -base64 24`

**Note**: This password will be hashed with bcrypt automatically by the ExternalSecret.

#### argocd-oidc-client-id
```
key: argocd-oidc-client-id
value: argocd
```

#### argocd-oidc-client-secret
```
key: argocd-oidc-client-secret
value: <oidc-client-secret>
```

**Generate**: `openssl rand -base64 32`

**Important**: This is the **plaintext** secret. You must also hash it for Authelia:
```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'your-plaintext-secret'
```

Use the plaintext in Bitwarden, and the hashed version in Authelia configuration.

### ArgoCD Configuration

#### Main Environment Variables

Modify `base/argocd-cm-patch.yaml`:

- `url`: Tailscale URL of your instance
- `issuer`: Authelia OIDC issuer URL
- `repositories`: Add your Git repository URL

#### Repository Access

For private repositories, you'll need to configure credentials after installation:

```bash
argocd repo add https://github.com/your-username/homelab.git \
  --username your-username \
  --password your-token
```

Or use SSH keys for better security.

### Deployment

#### Step 1: Create Bitwarden Secrets

Generate and create the three required secrets in Bitwarden Secret Manager (see Bitwarden Secrets section above).

#### Step 2: Configure Authelia

Add the ArgoCD OIDC client to your Authelia configuration and restart Authelia:
```bash
kubectl rollout restart deployment/authelia -n authelia
```

#### Step 3: Update ArgoCD Configuration

Edit `base/argocd-cm-patch.yaml` and update:
- `url`: Your Tailscale URL (e.g., `https://argocd.<your-tailnet-id>.ts.net`)
- `issuer`: Your Authelia URL (e.g., `https://auth.example.com`)
- `repositories`: Your Git repository URL

#### Step 4: Deploy ArgoCD

```bash
kubectl apply -k argocd/base/
```

#### Step 5: Wait for Pods

```bash
kubectl get pods -n argocd -w
# Wait for all pods to be Running and Ready
```

#### Step 6: Verify ExternalSecrets

```bash
kubectl get externalsecret -n argocd
# Should show STATUS: SecretSynced for both secrets
```

### Access

The application is exposed via Tailscale Operator.

To get the Tailscale URL:
```bash
kubectl get ingress -n argocd
```

Access URL: `https://argocd.<your-tailnet-id>.ts.net`

### Initial Configuration

#### First Login

1. Access the Tailscale URL
2. Click "Log in via Authelia"
3. Authenticate with your Authelia account (must be in `argocd-admins` or `argocd-users` group)

**Alternative**: Login with admin account (local auth):
```bash
# Get initial admin password
kubectl -n argocd get secret argocd-secret -o jsonpath="{.data.admin\.password}" | base64 -d
```

#### Add Applications

1. In ArgoCD UI, click "New App"
2. Configure:
   - **Application Name**: `immich`
   - **Project**: `default`
   - **Sync Policy**: `Automatic` (optional)
   - **Repository URL**: `https://github.com/your-username/homelab.git`
   - **Path**: `apps/immich/base`
   - **Destination**: `https://kubernetes.default.svc`
   - **Namespace**: `immich`

Repeat for other applications: paperless-ngx, wallabag, miniflux, shared-services.

#### ArgoCD Self-Management

To make ArgoCD manage itself (GitOps):

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/homelab.git
    targetRevision: HEAD
    path: apps/argocd/base
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Features

- **GitOps**: Declarative continuous delivery
- **SSO**: Single Sign-On via Authelia
- **RBAC**: Role-based access control with Authelia groups
- **Multi-tenancy**: Project isolation
- **Sync Policies**: Manual or automatic
- **Health Assessment**: Application health monitoring
- **Rollback**: Easy rollback to previous versions
- **Webhooks**: Automatic sync on Git push
- **CLI**: `argocd` command-line tool

### Maintenance

#### CLI Installation

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.13.2/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

#### CLI Login

```bash
argocd login argocd.<your-tailnet-id>.ts.net --sso
```

Or with admin account:
```bash
argocd login argocd.<your-tailnet-id>.ts.net --username admin --password <password>
```

#### Update

To update ArgoCD version:

1. Update `argocd-install.yaml` URL in deployment notes
2. Download new version:
   ```bash
   curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/v2.X.X/manifests/install.yaml \
     -o apps/argocd/base/argocd-install.yaml
   ```
3. Apply:
   ```bash
   kubectl apply -k argocd/base/
   ```

#### Logs

```bash
kubectl logs -f deployment/argocd-server -n argocd
kubectl logs -f deployment/argocd-application-controller -n argocd
kubectl logs -f deployment/argocd-repo-server -n argocd
```

#### Health Check

```bash
kubectl exec -n argocd deployment/argocd-server -- argocd version
```

#### Sync Application

```bash
argocd app sync <app-name>
argocd app sync immich
```

#### List Applications

```bash
argocd app list
```

### Troubleshooting

#### OIDC Login Not Working

1. Check Authelia client configuration:
   ```bash
   kubectl logs -f deployment/authelia -n authelia
   ```

2. Verify redirect URI matches exactly:
   ```
   https://argocd.<your-tailnet-id>.ts.net/auth/callback
   ```

3. Check ArgoCD OIDC configuration:
   ```bash
   kubectl get configmap argocd-cm -n argocd -o yaml
   ```

4. Verify secret is synced:
   ```bash
   kubectl get externalsecret -n argocd
   kubectl get secret argocd-oidc-secret -n argocd -o yaml
   ```

#### Application Not Syncing

1. Check application status:
   ```bash
   argocd app get <app-name>
   ```

2. View sync errors:
   ```bash
   argocd app logs <app-name>
   ```

3. Manual sync:
   ```bash
   argocd app sync <app-name> --force
   ```

#### Repository Connection Issues

1. Test repository access:
   ```bash
   argocd repo list
   argocd repo get https://github.com/your-username/homelab.git
   ```

2. Update credentials:
   ```bash
   argocd repo add https://github.com/your-username/homelab.git \
     --username your-username \
     --password your-new-token \
     --upsert
   ```

### Support

Official documentation: https://argo-cd.readthedocs.io/

---

## Français

Déploiement Kubernetes d'ArgoCD avec Kustomize, intégrant Bitwarden External Secrets Operator, Tailscale Operator et authentification OIDC via Authelia.

### Architecture

- **ArgoCD v2.13.2**: Outil de livraison continue GitOps
- **Authelia OIDC**: Authentification OpenID Connect
- **Tailscale**: Ingress réseau privé

### Prérequis

- Cluster Kubernetes (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- Tailscale Operator installé
- Authelia configuré avec un client OIDC pour ArgoCD

### Configuration Authelia

Ajouter le client suivant dans la configuration Authelia (`identity_providers.oidc.clients`):

```yaml
- client_id: 'argocd'
  client_name: 'ArgoCD'
  client_secret: '<hashed-secret>'
  public: false
  authorization_policy: 'two_factor'
  redirect_uris:
    - 'https://argocd.<your-tailnet-id>.ts.net/auth/callback'
  scopes:
    - 'openid'
    - 'profile'
    - 'email'
    - 'groups'
  response_types:
    - 'code'
  grant_types:
    - 'authorization_code'
```

**Groupes RBAC dans Authelia:**

Créer des groupes dans Authelia pour le contrôle d'accès ArgoCD:
- `argocd-admins`: Accès admin complet à ArgoCD
- `argocd-users`: Accès lecture seule à ArgoCD

### Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secret Manager (key: value):

#### argocd-admin-password
```
key: argocd-admin-password
value: <admin-password>
```

**Générer**: `openssl rand -base64 24`

**Note**: Ce mot de passe sera hashé avec bcrypt automatiquement par l'ExternalSecret.

#### argocd-oidc-client-id
```
key: argocd-oidc-client-id
value: argocd
```

#### argocd-oidc-client-secret
```
key: argocd-oidc-client-secret
value: <oidc-client-secret>
```

**Générer**: `openssl rand -base64 32`

**Important**: Ceci est le secret en **clair**. Vous devez aussi le hasher pour Authelia:
```bash
docker run --rm authelia/authelia:latest \
  authelia crypto hash generate argon2 \
  --password 'votre-secret-en-clair'
```

Utiliser le secret en clair dans Bitwarden, et la version hashée dans la configuration Authelia.

### Configuration ArgoCD

#### Variables d'environnement principales

Modifier `base/argocd-cm-patch.yaml`:

- `url`: URL Tailscale de votre instance
- `issuer`: URL de l'émetteur OIDC Authelia
- `repositories`: Ajouter l'URL de votre dépôt Git

#### Accès au dépôt

Pour les dépôts privés, vous devrez configurer les credentials après l'installation:

```bash
argocd repo add https://github.com/your-username/homelab.git \
  --username your-username \
  --password your-token
```

Ou utiliser des clés SSH pour plus de sécurité.

### Déploiement

#### Étape 1: Créer les secrets Bitwarden

Générer et créer les trois secrets requis dans Bitwarden Secret Manager (voir section Secrets Bitwarden ci-dessus).

#### Étape 2: Configurer Authelia

Ajouter le client OIDC ArgoCD dans votre configuration Authelia et redémarrer Authelia:
```bash
kubectl rollout restart deployment/authelia -n authelia
```

#### Étape 3: Mettre à jour la configuration ArgoCD

Éditer `base/argocd-cm-patch.yaml` et mettre à jour:
- `url`: Votre URL Tailscale (ex: `https://argocd.<your-tailnet-id>.ts.net`)
- `issuer`: Votre URL Authelia (ex: `https://auth.example.com`)
- `repositories`: L'URL de votre dépôt Git

#### Étape 4: Déployer ArgoCD

```bash
kubectl apply -k argocd/base/
```

#### Étape 5: Attendre les pods

```bash
kubectl get pods -n argocd -w
# Attendre que tous les pods soient Running et Ready
```

#### Étape 6: Vérifier les ExternalSecrets

```bash
kubectl get externalsecret -n argocd
# Doit afficher STATUS: SecretSynced pour les deux secrets
```

### Accès

L'application est exposée via Tailscale Operator.

Pour obtenir l'URL Tailscale:
```bash
kubectl get ingress -n argocd
```

URL d'accès : `https://argocd.<your-tailnet-id>.ts.net`

### Configuration initiale

#### Première connexion

1. Accéder à l'URL Tailscale
2. Cliquer sur "Log in via Authelia"
3. S'authentifier avec votre compte Authelia (doit être dans le groupe `argocd-admins` ou `argocd-users`)

**Alternative**: Connexion avec le compte admin (auth locale):
```bash
# Obtenir le mot de passe admin initial
kubectl -n argocd get secret argocd-secret -o jsonpath="{.data.admin\.password}" | base64 -d
```

#### Ajouter des applications

1. Dans l'UI ArgoCD, cliquer sur "New App"
2. Configurer:
   - **Application Name**: `immich`
   - **Project**: `default`
   - **Sync Policy**: `Automatic` (optionnel)
   - **Repository URL**: `https://github.com/your-username/homelab.git`
   - **Path**: `apps/immich/base`
   - **Destination**: `https://kubernetes.default.svc`
   - **Namespace**: `immich`

Répéter pour les autres applications: paperless-ngx, wallabag, miniflux, shared-services.

#### Auto-gestion d'ArgoCD

Pour faire gérer ArgoCD par lui-même (GitOps):

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/homelab.git
    targetRevision: HEAD
    path: apps/argocd/base
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Fonctionnalités

- **GitOps**: Livraison continue déclarative
- **SSO**: Authentification unique via Authelia
- **RBAC**: Contrôle d'accès basé sur les rôles avec groupes Authelia
- **Multi-tenant**: Isolation par projet
- **Sync Policies**: Synchronisation manuelle ou automatique
- **Health Assessment**: Surveillance de la santé des applications
- **Rollback**: Retour facile aux versions précédentes
- **Webhooks**: Synchronisation automatique sur push Git
- **CLI**: Outil en ligne de commande `argocd`

### Maintenance

#### Installation CLI

```bash
# macOS
brew install argocd

# Linux
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/download/v2.13.2/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

#### Connexion CLI

```bash
argocd login argocd.<your-tailnet-id>.ts.net --sso
```

Ou avec le compte admin:
```bash
argocd login argocd.<your-tailnet-id>.ts.net --username admin --password <password>
```

#### Mise à jour

Pour mettre à jour la version d'ArgoCD:

1. Mettre à jour l'URL `argocd-install.yaml` dans les notes de déploiement
2. Télécharger la nouvelle version:
   ```bash
   curl -sSL https://raw.githubusercontent.com/argoproj/argo-cd/v2.X.X/manifests/install.yaml \
     -o apps/argocd/base/argocd-install.yaml
   ```
3. Appliquer:
   ```bash
   kubectl apply -k argocd/base/
   ```

#### Logs

```bash
kubectl logs -f deployment/argocd-server -n argocd
kubectl logs -f deployment/argocd-application-controller -n argocd
kubectl logs -f deployment/argocd-repo-server -n argocd
```

#### Vérification de santé

```bash
kubectl exec -n argocd deployment/argocd-server -- argocd version
```

#### Synchroniser une application

```bash
argocd app sync <app-name>
argocd app sync immich
```

#### Lister les applications

```bash
argocd app list
```

### Dépannage

#### Connexion OIDC ne fonctionne pas

1. Vérifier la configuration du client Authelia:
   ```bash
   kubectl logs -f deployment/authelia -n authelia
   ```

2. Vérifier que l'URI de redirection correspond exactement:
   ```
   https://argocd.<your-tailnet-id>.ts.net/auth/callback
   ```

3. Vérifier la configuration OIDC d'ArgoCD:
   ```bash
   kubectl get configmap argocd-cm -n argocd -o yaml
   ```

4. Vérifier que le secret est synchronisé:
   ```bash
   kubectl get externalsecret -n argocd
   kubectl get secret argocd-oidc-secret -n argocd -o yaml
   ```

#### Application ne se synchronise pas

1. Vérifier le statut de l'application:
   ```bash
   argocd app get <app-name>
   ```

2. Voir les erreurs de synchronisation:
   ```bash
   argocd app logs <app-name>
   ```

3. Synchronisation manuelle:
   ```bash
   argocd app sync <app-name> --force
   ```

#### Problèmes de connexion au dépôt

1. Tester l'accès au dépôt:
   ```bash
   argocd repo list
   argocd repo get https://github.com/your-username/homelab.git
   ```

2. Mettre à jour les credentials:
   ```bash
   argocd repo add https://github.com/your-username/homelab.git \
     --username your-username \
     --password your-new-token \
     --upsert
   ```

### Support

Documentation officielle: https://argo-cd.readthedocs.io/
