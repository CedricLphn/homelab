# Paperless-ngx Kustomize Chart

Déploiement Kubernetes de Paperless-ngx avec Kustomize, intégrant Bitwarden External Secrets Operator, Tailscale Operator et authentification OAuth via Authelia.

## Architecture

- **Paperless-ngx**: Application principale de gestion documentaire
- **PostgreSQL**: Base de données
- **Redis**: Cache et file d'attente de tâches
- **Apache Tika**: Parsing de documents Office et emails
- **Gotenberg**: Conversion de documents
- **OAuth Authelia**: Authentification unique (SSO)

## Prérequis

- Cluster Kubernetes (Talos Linux)
- Kustomize
- Bitwarden External Secrets Operator configuré avec un `ClusterSecretStore` nommé `bitwarden-cluster-secretstore`
- Tailscale Operator installé
- StorageClass `local-path` configurée
- Authelia configuré et opérationnel

## Secrets Bitwarden

Créer les secrets suivants dans Bitwarden Secret Manager (key: value):

### paperless-db-username
```
key: paperless-db-username
value: <db-username>
```

### paperless-db-password
```
key: paperless-db-password
value: <db-password>
```

### paperless-secret-key
```
key: paperless-secret-key
value: <django-secret-key-aleatoire>
```

### paperless-admin-username
```
key: paperless-admin-username
value: root
```

### paperless-admin-password
```
key: paperless-admin-password
value: <admin-password>
```

### paperless-oauth-providers
```json
key: paperless-oauth-providers
value: {
  "openid_connect": {
    "APPS": [
      {
        "provider_id": "authelia",
        "name": "Authelia",
        "client_id": "paperless",
        "secret": "YOUR_CLIENT_SECRET",
        "settings": {
          "server_url": "https://auth.example.com/.well-known/openid-configuration"
        }
      }
    ],
    "OAUTH_PKCE_ENABLED": true
  }
}
```

## Configuration Authelia

Dans Authelia, configurer le client OAuth pour Paperless-ngx :

```yaml
identity_providers:
  oidc:
    clients:
      - id: paperless
        description: Paperless-ngx
        secret: YOUR_CLIENT_SECRET
        public: false
        authorization_policy: two_factor
        redirect_uris:
          - https://paperless.tail3161aa.ts.net/accounts/openid_connect/authelia/login/callback/
        scopes:
          - openid
          - profile
          - email
          - groups
        grant_types:
          - authorization_code
        response_types:
          - code
        token_endpoint_auth_method: client_secret_post
```

Pour restreindre l'accès au groupe "paperless" uniquement :

```yaml
access_control:
  rules:
    - domain: paperless.tail3161aa.ts.net
      policy: two_factor
      subject:
        - "group:paperless"
```

## Configuration Paperless

### Variables d'environnement principales

Modifier `base/paperless-deployment.yaml`:

- `PAPERLESS_TIME_ZONE`: Fuseau horaire (défaut: Europe/Paris)
- `PAPERLESS_OCR_LANGUAGE`: Langues OCR (défaut: fra+eng)
- `PAPERLESS_URL`: URL Tailscale
- `PAPERLESS_DISABLE_REGULAR_LOGIN`: true (OAuth uniquement)
- `PAPERLESS_SOCIALACCOUNT_ALLOW_SIGNUPS`: true (création auto des comptes OAuth)

### Stockage

Modifier les tailles dans `base/pvc.yaml`:

- `paperless-data`: 10Gi (index, base SQLite si utilisée)
- `paperless-media`: 50Gi (documents et miniatures)
- `paperless-consume`: 10Gi (répertoire d'import)
- `paperless-export`: 10Gi (exports)
- `postgres-data`: 10Gi
- `redis-data`: 1Gi

## Déploiement

```bash
kubectl apply -k base/
```

## Accès

L'application est exposée via Tailscale Operator et accessible uniquement via OAuth Authelia.

Pour obtenir l'URL Tailscale:
```bash
kubectl get ingress -n paperless-ngx
```

URL d'accès : `https://paperless.tail3161aa.ts.net`

## Premier utilisateur OAuth

Lors de la première connexion via Authelia, un compte utilisateur est automatiquement créé dans Paperless. Pour lui donner les droits administrateur :

```bash
kubectl exec -n paperless-ngx deployment/paperless-ngx -- python3 manage.py shell -c "from django.contrib.auth import get_user_model; User = get_user_model(); u = User.objects.get(username='VOTRE_USERNAME'); u.is_superuser = True; u.is_staff = True; u.save(); print(f'User {u.username} is now superuser')"
```

Ou via l'interface avec le compte `root` (créé au démarrage avec les credentials de Bitwarden).

## Maintenance

### Sauvegardes

Sauvegarder les PVCs:
- `paperless-data`
- `paperless-media`
- `postgres-data`

### Mise à jour

```bash
kubectl rollout restart deployment/paperless-ngx -n paperless-ngx
```

### Logs

```bash
kubectl logs -f deployment/paperless-ngx -n paperless-ngx
```

## Support

Documentation officielle: https://docs.paperless-ngx.com/
